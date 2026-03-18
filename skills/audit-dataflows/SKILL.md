---
name: audit-dataflows
description: >
  Audit your data flow manifest for stale entries and unmapped cross-service routes.
  Reports coverage stats without modifying files. Run periodically to catch manifest drift.
user-invocable: true
argument-hint: ""
---

# audit-dataflows

Audit a dataflow manifest for stale entries, unmapped routes, and coverage gaps.

## Manifest Discovery

Find the manifest by searching for `.claude/dataflows/manifest.yaml` starting from the current working directory, walking up parent directories until found or reaching `/`.

**Search algorithm:**
1. Check `{cwd}/.claude/dataflows/manifest.yaml`
2. Check `{cwd}/../.claude/dataflows/manifest.yaml`
3. Continue walking up until found or root reached

If not found, print this error and stop:
```
No dataflow manifest found. Searched for .claude/dataflows/manifest.yaml from cwd upward.

Create one using the example at:
  https://github.com/adamwdennis/agent-skills/blob/main/examples/manifest.yaml

Place it at: {closest_repo_root}/.claude/dataflows/manifest.yaml
```

Set `MANIFEST_DIR` to the directory containing the found manifest.

## Parse Manifest

Read and parse the YAML manifest. Required fields:
- `repo_root` — resolve `~` to home dir. If relative, resolve relative to `MANIFEST_DIR`.
- `domains` — map of domain name → domain config

Required for audit:
- `route_patterns` — list of `{glob, type}` for finding route files
- `cross_service_indicators` — list of string patterns indicating cross-service calls

If `route_patterns` or `cross_service_indicators` are missing, warn that audit will be limited to stale checks only.

## Step 1: Stale Entry Check

For every flow across all domains, verify:
- `entry` file exists at `{repo_root}/{flow.entry}`
- `method` name is defined in that file (grep for function/method/route definition)

Collect stale entries (file missing or method missing).

## Step 2: Route Discovery

For each pattern in `route_patterns`:
1. Glob for matching files under `repo_root` using the `glob` field
2. For each matched file, extract all public route definitions:
   - **Python** (`type: python`): methods decorated with `@app.get`, `@app.post`, `@app.put`, `@app.delete`, `@app.patch`, `@router.get`, etc.
   - **TypeScript** (`type: typescript`): methods decorated with `@Get()`, `@Post()`, `@Put()`, `@Delete()`, `@Patch()`

Build a list of all discovered routes: `{file, method_name, line_number, http_method, path}`.

## Step 3: Identify Mapped vs Unmapped

Compare discovered routes against all flows in the manifest. A route is "mapped" if its file + method name matches any flow's `entry` + `method`.

Separate unmapped routes for classification.

## Step 4: Deterministic Classification

For each unmapped route, read the method body and grep for `cross_service_indicators`:
- If ANY indicator is found → classify as **cross-service** (should be mapped)
- If NO indicators found → classify as **simple** (likely CRUD, no action needed)

## Step 5: LLM Classification (Ambiguous Only)

Some routes may be ambiguous — they don't contain literal indicator strings but may still involve cross-service logic (e.g., via abstracted service calls, event dispatching).

For routes where the deterministic pass is inconclusive (e.g., the method calls another service method that internally makes cross-service calls), do a quick read of the method and its immediate callees to classify:
- **cross-service** — involves calls to other services
- **simple** — self-contained CRUD or utility

Keep LLM passes minimal — only for genuinely ambiguous cases. If the deterministic pass clearly classifies a route, skip the LLM pass.

## Step 6: Generate Report

Print a structured report:

```
Dataflow Manifest Audit
========================
Manifest: {MANIFEST_DIR}/manifest.yaml
Repo root: {repo_root}
Scanned: {timestamp}

Stale Entries (file/method no longer exists):
  ✗ billing.create_subscription — method removed from cp_billing/routes.py
  ✗ org-lifecycle.setup_bhip_organization — file renamed

  (or: ✓ All entries valid)

Unmapped Cross-Service Routes:
  + POST /billing/webhooks/stripe (cp_billing/routes.py:34)
    → calls httpService.post (cross-service)
  + POST /integrations/sync (cp_integrations/routes.py:89)
    → calls private_web_client.post (cross-service)

  (or: ✓ No unmapped cross-service routes)

Simple Routes (not flagged): {count} routes
  These are self-contained CRUD/utility routes that don't need flow documentation.

Coverage:
  Mapped flows: {mapped_count}
  Cross-service routes total: {mapped_count + unmapped_cross_service_count}
  Coverage: {mapped_count}/{total} ({percentage}%)

Domains:
  org-lifecycle: {flow_count} flows, {stale_count} stale
  billing: {flow_count} flows, {stale_count} stale
```

## Important Notes

- This skill is **read-only** — it never modifies the manifest or any files
- Route discovery depends on `route_patterns` being comprehensive — missing patterns mean missing coverage
- The LLM pass should be conservative — when in doubt, flag as cross-service (false positives are cheaper than false negatives)
- Run this periodically (e.g., before sprint planning) to catch manifest drift
