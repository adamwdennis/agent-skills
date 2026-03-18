# agent-skills

Open-source agent skills for AI coding assistants. Cross-agent compatible (Claude Code, Cursor, Codex, etc). Git-based distribution.

## Install

```bash
# All skills
npx skills add adamwdennis/agent-skills

# Single skill
npx skills add adamwdennis/agent-skills --skill trace-dataflows
```

## Skills

### trace-dataflows

Trace cross-service data flows through your codebase and generate Mermaid sequence diagrams.

```
/trace-dataflows                    # List available domains
/trace-dataflows org-lifecycle      # Trace a specific domain
/trace-dataflows --all              # Trace all domains
```

Dispatches parallel agents to follow call chains from manifest-defined entry points, presents findings for human review, then generates grouped Mermaid diagrams per domain.

### audit-dataflows

Audit your data flow manifest for stale entries and unmapped cross-service routes.

```
/audit-dataflows
```

Reports coverage stats without modifying files. Run periodically to catch manifest drift.

## Setup

Both skills use a **manifest file** to know what to trace and audit.

1. Create `.claude/dataflows/manifest.yaml` in your project (or a shared parent directory)
2. Define your `repo_root`, domains, flows, and entry points
3. See [`examples/manifest.yaml`](examples/manifest.yaml) for all available fields

The skills auto-discover the manifest by walking up from your current directory — no configuration needed.

### Minimal manifest

```yaml
repo_root: .

domains:
  my-domain:
    description: What this domain covers
    flows:
      - name: my_flow
        entry: src/api/routes.py
        method: handle_request
        last_traced: null
```

### For audit support, add route patterns

```yaml
route_patterns:
  - glob: "**/routes.py"
    type: python
  - glob: "**/*Controller.ts"
    type: typescript

cross_service_indicators:
  - "httpService."
  - "axios."
  - "fetch("
```

## How It Works

**trace-dataflows:**
1. Finds manifest via convention-based directory walk
2. Validates entry points (pre-trace audit)
3. Reads architecture docs and OpenAPI specs for context
4. Dispatches parallel Explore agents per entry file group
5. Presents findings for human review
6. On approval, generates Mermaid sequence diagrams
7. Updates `last_traced` timestamps

**audit-dataflows:**
1. Finds manifest via same discovery
2. Checks all flow entries for staleness
3. Scans route files for unmapped cross-service routes
4. Reports coverage stats

## License

MIT
