# {Domain Name}

> {Domain description from manifest}
>
> Generated: {YYYY-MM-DD}

## Overview

```mermaid
flowchart TD
    subgraph External
        EXT1["{external_integration.source}"]
    end

    subgraph ServiceA["Service A"]
        A1["{flow_name}"]
        A2["{flow_name}"]
    end

    subgraph ServiceB["Service B"]
        B1["{flow_name}"]
    end

    EXT1 -->|"{mechanism}"| A1
    A1 -->|"HTTP"| B1
    A2 -.->|"async"| B1
```

The overview flowchart is auto-generated from:
- Cross-flow call references discovered during tracing
- External integrations from the manifest
- Use subgraphs for service boundaries
- Solid arrows for synchronous calls, dashed for async

---

## {flow_name}

| Field | Value |
|-------|-------|
| **Route** | `{HTTP_METHOD} {path}` |
| **Auth** | `{auth_requirement}` |
| **Entry** | `{entry_file}:{line}` |
| **Source chain** | `{file}:{line}` → `{file}:{line}` → ... |
| **Frontend callers** | `{component_file}:{line}` — {description} |
| **Internal callers** | `{caller_file}:{line}` — {description} |

```mermaid
sequenceDiagram
    participant FE as Frontend
    participant SvcA as Service A
    participant SvcB as Service B
    participant DB as Database

    FE->>SvcA: POST /endpoint
    activate SvcA
    note over SvcA: validate input, check auth

    SvcA->>SvcB: POST /internal/endpoint
    activate SvcB
    SvcB->>DB: INSERT entity
    DB-->>SvcB: ok
    SvcB-->>SvcA: 200 response
    deactivate SvcB

    SvcA-->>FE: 200 response
    deactivate SvcA

    SvcA-->>Worker: dispatch async_task
    note over Worker: dashed = async, not traced
```

<!-- traced: {ISO_DATE} from {entry}:{method} -->

---

## External Integrations

| Source | Target Flow | Mechanism | Owner |
|--------|-------------|-----------|-------|
| {source} | {target} | {mechanism} | {owner} |
