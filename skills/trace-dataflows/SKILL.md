---
name: trace-dataflows
description: >
  Trace data flows through your codebase and generate Mermaid sequence diagrams.
  Dispatches parallel agents to follow call chains from manifest entry points,
  presents findings for review, then generates grouped diagrams per domain.
user-invocable: true
argument-hint: "<domain> | --all"
---

# trace-dataflows

Trace cross-service data flows and generate Mermaid sequence diagrams from a project manifest.

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

Read and parse the YAML manifest. Required top-level fields:
- `repo_root` — resolve `~` to home dir. If relative, resolve relative to `MANIFEST_DIR`.
- `domains` — map of domain name → domain config

Optional fields:
- `architecture_docs` — list of paths relative to `repo_root`
- `openapi_specs` — list of paths relative to `repo_root`
- `route_patterns` — list of `{glob, type}` for audit scanning
- `cross_service_indicators` — list of string patterns indicating cross-service calls

If `repo_root` doesn't exist or `domains` is missing, report the error and stop.

## Mode: No Arguments

If invoked without arguments (`/trace-dataflows`), list available domains:

```
Dataflow domains ({MANIFEST_DIR}/manifest.yaml):

  org-lifecycle — Organization creation, setup, and user onboarding
    Flows: 3 | Last traced: 2026-03-15

  billing — Subscription and payment processing
    Flows: 5 | Last traced: never
```

For each domain, show: description, flow count, and the most recent `last_traced` date (or "never").

Then stop.

## Mode: Trace (`<domain>` or `--all`)

If `--all`, iterate domains sequentially. Otherwise, look up the named domain.

If domain not found, list available domains and stop.

### Step 1: Pre-Trace Audit

For the target domain, validate all flow entries:

**Stale check:** For each flow, verify:
- `entry` file exists at `{repo_root}/{flow.entry}`
- `method` name exists in that file (grep for function/method/route definition)

**Unmapped check:** For each unique entry file in the domain, scan for public route methods (decorated with `@app.post`, `@app.get`, `@Post()`, `@Get()`, etc.) that are NOT listed in any flow's `method` field.

Surface results:
```
Pre-trace audit for org-lifecycle:

  Stale entries:
    ✗ setup_bhip_organization — method not found in cp_organization/routes.py

  Unmapped routes in traced files:
    + archive_organization (cp_organization/routes.py:89) — not in any flow

  3 flows valid, 1 stale, 1 unmapped
```

If stale entries exist, ask the user:
> Stale entries found. Continue tracing valid flows only, or fix manifest first? [continue/fix]

If user says "fix", stop and let them edit the manifest.

### Step 2: Read Context

1. Read each file listed in `architecture_docs` (paths relative to `repo_root`). Concatenate contents as architecture context.
2. Read each file listed in `openapi_specs` (paths relative to `repo_root`). Extract route metadata (paths, methods, descriptions) as API context.

If any file doesn't exist, warn but continue.

### Step 3: Group Flows by Entry File

Group the domain's valid flows by their `entry` file path. Each group will be traced by one agent.

### Step 4: Dispatch Parallel Trace Agents

For each entry file group, dispatch one **Explore** subagent (in parallel via multiple Agent tool calls) with this prompt:

```
You are tracing data flows through a codebase. Your task is to follow call chains
from specific entry points and document what you find.

## Architecture Context
{architecture_doc_contents}

## API Context (OpenAPI)
{relevant_openapi_metadata}

## Entry File
{repo_root}/{entry_file}

## Flows to Trace
{for each flow in group:}
- {flow.name}: method `{flow.method}` — trace the full call chain

## Instructions

For each flow:
1. Read the entry file and locate the method
2. Identify the route decorator (path, HTTP method, auth requirements)
3. Follow the call chain inward:
   - Intra-service calls (same service dir) — follow fully
   - Cross-service HTTP calls — follow up to 2 hops, note the target
   - Async dispatches (Celery tasks, message queues) — note as async, do NOT trace into
   - Database operations — note the entity/table and operation type
4. Search for frontend callers:
   - Grep for the route path in React/TypeScript files
   - Note the component and how it calls the endpoint
5. Search for internal callers:
   - Grep for the method name being called from other backend code
   - Note the caller location and context

## Output Format

For each flow, return:

### {flow_name}
- **Route:** {HTTP_METHOD} {path} ({auth_requirement})
- **Source:** {entry_file}:{line} → {next_file}:{line} → ...
- **Frontend callers:** {component_file}:{line} — {description}
- **Internal callers:** {caller_file}:{line} — {description}
- **Async dispatches:** {task_name} via {mechanism}
- **Cross-service calls:**
  1. {source_service} → {target_service}: {endpoint} ({purpose})
  2. ...
- **DB operations:** {entity} — {operation} (via {method})

Be thorough but concise. Include file paths and line numbers.
```

### Step 5: Present Findings for Review

Collect results from all agents. Present a summary per flow:

```
## Trace Results: org-lifecycle

### setup_eve_organization
Route: POST /cp/organizations/setup-eve (EVE_ADMIN)
Source: cp_organization/routes.py:51 → service.py:92
Frontend callers: AdminOrgSetup.tsx:34 — setup button handler
Internal callers: CPPilotService.create_pilot_org (step 3 of pilot creation)

Chain:
  1. FastAPI → NestJS: POST /internal/organizations (create org + feature toggles)
  2. FastAPI → NestJS: POST /internal/users/support (Auth0 user + DB record)
  3. FastAPI → DB: OrgSettings insert

Async: notify_ops_channel via Celery

---

### completeOnboarding
Route: POST /internal/users/complete-onboarding (INTERNAL)
Source: UsersInternalController.ts:45 → UsersService.ts:120
...

---

Approve these findings? You can:
- Type "approve" to generate diagrams
- Provide corrections (e.g. "setup_eve_organization also calls billing service")
- Add file paths to investigate (e.g. "also check src/web_server/feature/billing/")
```

If the user provides corrections, update the findings summary directly based on their feedback — do NOT re-dispatch agents. Then re-present for approval.

### Step 6: Generate Output

On approval, generate `{MANIFEST_DIR}/{domain}.md` using the output template structure:

1. **Header** — domain name, description, generation date
2. **Overview Flowchart** — Mermaid flowchart showing:
   - Cross-flow call references discovered during tracing
   - External integrations from manifest
   - Service boundaries as subgraphs
3. **Per-Flow Sections** — for each flow:
   - Metadata table (route, auth, source files, callers)
   - Mermaid sequence diagram
   - `<!-- traced: {ISO_DATE} from {entry}:{method} -->`
4. **External Integrations Table** — from manifest `external_integrations`

Refer to `templates/output-template.md` for the exact structure.

### Step 7: Update Manifest

Update `last_traced` to the current ISO date for each successfully traced flow in the manifest YAML file.

## Mermaid Diagram Guidelines

- Use `participant` aliases for readability (e.g. `participant FE as Frontend`)
- Dashed arrows (`-->>`) for async operations (Celery, webhooks)
- Solid arrows (`->>`) for synchronous calls
- `activate`/`deactivate` for request-response pairs
- `note over` for important context (auth, validation)
- `rect` blocks for logical groupings
- Keep diagrams under 40 lines — split complex flows into sub-diagrams if needed

## Error Handling

- If a flow can't be traced (file missing, method not found), report the failure and continue with remaining flows
- Partial results are valuable — 3 of 4 flows traced is still useful
- Malformed YAML stops execution before any tracing
- Agent failures are reported per-flow, not as a global failure
