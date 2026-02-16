# Design Driven Development (DDD) - Complete Specification

## Document Purpose
This document captures the complete specification for the DDD tool, enabling continuation of development discussions in any Claude session. It includes core concepts, architectural decisions, technical details, and implementation guidance.

---

# Part 1: Core Concept

## What is DDD?

**Design Driven Development (DDD)** is a unified tool for bidirectional conversion between visual flow diagrams and code, enabling solopreneurs and small teams to architect systems visually while LLMs (like Claude Code) handle implementation.

### The Four-Phase Lifecycle

```
Phase 1: CREATE        Phase 2: DESIGN         Phase 3: BUILD          Phase 4: REFLECT
Human intent → Specs   Human reviews in Tool   Specs → Code            Code wisdom → Specs

/ddd-create            (DDD Tool)              /ddd-scaffold           /ddd-reverse
/ddd-update                                    /ddd-implement          /ddd-sync --discover
                                               /ddd-test               /ddd-reflect
                                                                       /ddd-promote

Cross-cutting (any phase): /ddd-status
Meta-level: /ddd-evolve
```

> **Note:** Legacy docs may reference "Session A" (= Phase 1+2) and "Session B" (= Phase 3+4).

### Core Insight

**Specs and code are both sources of truth at different levels.** Specs define the what and why, code accumulates the how. The Reflect phase (Phase 4) feeds implementation wisdom back into specs through annotations and promotion. Visual diagrams are a UI for editing YAML spec files. Everything lives in the Git repo.

---

# Part 2: Purpose & Value Proposition

## Primary Purpose

Enable non-developers (PMs, solopreneurs) to:
1. Design software architecture visually
2. Specify exact behavior (validations, error messages, business rules)
3. Have LLMs implement the code automatically
4. Maintain sync between design and implementation

## Target Users

| User | Technical Level | Primary Use |
|------|-----------------|-------------|
| Product Managers | Low-Medium | Flow design, business rules |
| Tech Leads | High | Spec detailing, architecture review |
| Developers | High | Code generation, implementation |
| Solopreneurs | Varies | End-to-end control without coding |

## Value Proposition

### Time Savings
| Task | Traditional | With DDD |
|------|-------------|----------|
| Design auth flow | 2 hours | 5 min (import template) |
| Stripe integration | 4 hours | 30 min (import + customize) |
| CRUD for 10 entities | 5 hours | 20 min (template) |
| New SaaS project | 2 days | 1 hour (starter) |
| Onboard contractor | 2-4 weeks | Day 1 |
| Feature: idea to code | 2-5 days | 2-4 hours |

### Solopreneur Superpower
- **Bottleneck shifts** from coding speed to thinking speed
- Visual design (10 min) + Fill specs (20 min) + Review (10 min) = LLM implements (2-4 hours automated)
- One person can build what previously required a team

---

# Part 3: Key Properties & Principles

## 1. Git as Single Source of Truth

**Decision:** Use Git for all sync between DDD Tool and Claude Code. No custom sync protocol.

```
obligo/
├── .git/                    # Git handles ALL versioning
├── specs/                   # DDD Tool reads/writes here
│   ├── system.yaml
│   ├── schemas/
│   └── domains/
│       └── {domain}/
│           └── flows/
│               └── {flow}.yaml
├── src/                     # Code generated from specs
├── .ddd/
│   ├── config.yaml          # Project settings
│   ├── mapping.yaml         # Spec-to-code mapping
│   └── annotations/         # Implementation wisdom from /ddd-reflect
└── CLAUDE.md                # Instructions for Claude Code
```

**Why Git wins over MCP-based sync:**
- Single source of truth (no state duplication)
- Built-in versioning, branching, merging, conflict resolution
- Works offline
- No extra infrastructure
- Familiar to developers
- Claude Code already knows Git

## 2. Hierarchical Spec Structure

```
System → Domain → Flow → Node
```

- **System Level:** Tech stack, domains, shared schemas, events, infrastructure
- **Domain Level:** Bounded contexts (ingestion, analysis, api, notification)
- **Flow Level:** Individual processes with triggers, nodes, connections
- **Node Level:** Single steps (trigger, input, process, decision, etc.)

## 3. Spec-Code Mapping

Every spec element maps to code:

```yaml
# .ddd/mapping.yaml
flows:
  webhook-ingestion:
    specHash: a1b2c3d4...          # SHA-256 of the flow YAML
    implementedAt: "2025-01-15T14:30:00Z"
    files:
      - domains/ingestion/src/router.py
      - domains/ingestion/src/middleware.py
    fileHashes:                     # Per-file hashes for reverse drift detection
      domains/ingestion/src/router.py: abc123...
    syncState: synced               # synced | spec_ahead | code_ahead | diverged
    annotationCount: 0              # Pending annotations from /ddd-reflect
```

## 4. Validation as First-Class Citizen

Specs capture exact validation rules:

```yaml
spec:
  fields:
    email:
      type: string
      validations:
        - format: email
          error: "Please enter a valid email address"
        - max_length: 255
          error: "Email must be less than 255 characters"
        - unique: true
          error: "This email is already registered"
```

These become Pydantic validators, Zod schemas, etc.

## 5. Error Messages in Specs

Error messages are specified, not generated:

```yaml
on_failure:
  status: 401
  body:
    error: "Signature validation failed"
    code: "INVALID_SIGNATURE"
```

Code must use these exact messages. Validation checks this.

---

# Part 4: Architecture

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ DDD TOOL (Desktop App - Tauri)                                  │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                │
│ │ Visual      │ │ Spec        │ │ Git         │                │
│ │ Editor      │ │ Panel       │ │ Panel       │                │
│ │ (Canvas)    │ │ (YAML UI)   │ │ (Status)    │                │
│ └─────────────┘ └─────────────┘ └─────────────┘                │
│                        │                                        │
│                        ▼                                        │
│              ┌─────────────────┐                               │
│              │ Spec Store      │                               │
│              │ (Zustand)       │                               │
│              └────────┬────────┘                               │
│                       │                                        │
└───────────────────────┼────────────────────────────────────────┘
                        │ Read/Write YAML files
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ FILE SYSTEM (Git Repo)                                          │
│                                                                 │
│ specs/*.yaml ◄──────────────────────► src/*.py                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                        ▲
                        │ Read specs, Write code
                        │
┌───────────────────────┼────────────────────────────────────────┐
│ CLAUDE CODE           │                                        │
│                       │                                        │
│ ┌─────────────────────┴─────────────────────┐                 │
│ │ 1. Read pending spec changes              │                 │
│ │ 2. Generate/update code                   │                 │
│ │ 3. Validate against specs                 │                 │
│ │ 4. Commit changes                         │                 │
│ └───────────────────────────────────────────┘                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## Multi-Level Canvas Architecture

A single canvas cannot scale to show an entire application. DDD uses a **3-level hierarchical sheet system** where each level provides the right abstraction for its scope. Users drill down by double-clicking, and navigate back via a breadcrumb bar.

```
┌─────────────────────────────────────────────────────────────────┐
│ LEVEL 1: SYSTEM MAP                                              │
│ One sheet. Auto-generated from system.yaml + domain.yaml files.  │
│ Shows domains as blocks, event arrows between them.              │
│                                                                   │
│   ┌──────────┐  contract.ingested  ┌──────────┐                 │
│   │ ingestion│────────────────────▶│ analysis │                 │
│   │ (4 flows)│                     │ (3 flows)│                 │
│   └──────────┘                     └──────────┘                 │
│        │                                │                        │
│        │ webhook.received               │ analysis.completed    │
│        ▼                                ▼                        │
│   ┌──────────┐                    ┌──────────┐                  │
│   │   api    │                    │notification│                 │
│   │ (5 flows)│                    │ (2 flows) │                 │
│   └──────────┘                    └──────────┘                  │
│                                                                   │
│   Double-click domain → Level 2                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ LEVEL 2: DOMAIN MAP                                              │
│ One sheet per domain. Shows flows as blocks within a domain,     │
│ event connections between flows, and portal nodes to other       │
│ domains.                                                         │
│                                                                   │
│   DOMAIN: ingestion                                              │
│                                                                   │
│   ┌──────────────┐     ┌───────────────┐                        │
│   │ webhook-     │────▶│ scheduled-    │                        │
│   │ ingestion    │event│ sync          │                        │
│   └──────────────┘     └───────────────┘                        │
│          │                                                       │
│          │ event: contract.ingested                              │
│          ▼                                                       │
│   ╔══════════════╗  ← portal to analysis domain                 │
│   ║ ➜ analysis   ║                                               │
│   ╚══════════════╝                                               │
│                                                                   │
│   Double-click flow → Level 3                                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ LEVEL 3: FLOW SHEET                                              │
│ One sheet per flow. Shows individual nodes and connections.      │
│ This is the existing detailed canvas.                            │
│                                                                   │
│   FLOW: webhook-ingestion                                        │
│                                                                   │
│   ⬡ trigger → ◇ validate_sig → ▱ validate_input                │
│                                    │                             │
│                              ⌗ store → ▭➤ event                 │
│                                           │                      │
│                                      ⬭ return                   │
│                                                                   │
│   Sub-flow nodes (▢) link to other flow sheets                  │
└─────────────────────────────────────────────────────────────────┘
```

### Navigation Model

| Action | Result |
|--------|--------|
| Double-click domain block (Level 1) | Drill into that domain's sheet (Level 2) |
| Double-click flow block (Level 2) | Drill into that flow's sheet (Level 3) |
| Double-click sub-flow node (Level 3) | Jump to referenced flow's sheet (Level 3) |
| Double-click portal node (Level 2) | Jump to target domain's sheet (Level 2) |
| Click breadcrumb segment | Navigate back to that level |
| Keyboard: Backspace / Esc | Navigate one level up |

**Breadcrumb bar** always visible at top of canvas:

```
System > ingestion > webhook-ingestion
  ^          ^              ^
  L1         L2             L3 (current)
```

### Sheet Data Sources

| Level | Auto-generated from | Editable? |
|-------|---------------------|-----------|
| System Map (L1) | `system.yaml` domains + `domain.yaml` event wiring | Layout only (positions) |
| Domain Map (L2) | `domain.yaml` flow list + inter-flow events | Layout only (positions) |
| Flow Sheet (L3) | `flows/*.yaml` | Fully editable (nodes, connections, specs) |

Levels 1 and 2 are **derived views** — their content comes from spec files. Users can reposition blocks but cannot add/remove domains or flows from these views (that happens via spec files or by creating new flow sheets). Level 3 is the primary editing surface.

## Node Types

### Flow-Level Nodes (Level 3) — Traditional Flows

| Type | Icon | Purpose |
|------|------|---------|
| Trigger | ⬡ | HTTP, webhook, cron, event |
| Input | ▱ | Form fields, API parameters |
| Process | ▭ | Transform, calculate, map |
| Decision | ◇ | Validation, business rules |
| Service Call | ▭▭ | External API, database, LLM |
| Data Store | ⌗ | Read/write operations |
| Event | ▭➤ | Queue publish, webhooks |
| Terminal | ⬭ | Response, end flow |
| Loop | ↺ | Iteration |
| Parallel | ═ | Concurrent execution |
| Sub-flow | ▢ | Call another flow (navigable link) |
| Delay | ⏱ | Wait/throttle (fixed or random) |
| Cache | ⊟ | Cache lookup with hit/miss branching |
| Transform | ⇌ | Field mapping between schemas |
| Collection | ⊞ | Filter, sort, deduplicate, merge, group, aggregate, reduce, flatten |
| Parse | ⊡ | Structured extraction (RSS, Atom, HTML, XML, JSON, CSV, Markdown) |
| Crypto | 🔒 | Encrypt, decrypt, hash, sign, verify, generate key |
| Batch | ▤ | Execute operation template against collection with concurrency control |
| Transaction | ⊛ | Atomic multi-step database operation with rollback on error |

### Agent Nodes (Level 3) — Agent Flows

| Type | Icon | Purpose |
|------|------|---------|
| LLM Call | ◆ | Call an LLM with prompt template, model config, structured output |
| Agent Loop | ↻ | Reason → select tool → execute → observe → repeat until done |
| Guardrail | ⛨ | Input/output content filter, PII detection, topic restriction |
| Human Gate | ✋ | Pause flow, notify human, await approval, timeout/escalation |

> **Note:** Tool definitions, memory stores, and semantic routing are configured as fields within the Agent Loop spec (tools[], memory[], and via Smart Router), not as standalone canvas nodes. The DDD Tool's `DddNodeType` union has 27 types: 19 traditional + 4 agent (llm_call, agent_loop, guardrail, human_gate) + 4 orchestration.

### Orchestration Nodes (Level 3) — Multi-Agent Flows

| Type | Icon | Purpose |
|------|------|---------|
| Orchestrator | ⊛ | Supervises multiple agents, monitors progress, merges results, intervenes |
| Smart Router | ◇◇+ | Rule-based + LLM hybrid routing with fallbacks, retries, A/B testing, circuit breaker |
| Handoff | ⇄ | Formal context transfer between agents (transfer/consult/collaborate modes) |
| Agent Group | [⊛] | Container for agents that share memory and coordination policy |

### Navigation Nodes (Level 2)

| Type | Icon | Purpose |
|------|------|---------|
| Flow Block | ▣ | Represents a traditional flow (double-click to drill in) |
| Agent Flow Block | ▣⊛ | Represents an agent flow (shows agent badge, double-click to drill in) |
| Agent Group Boundary | ╔[⊛]╗ | Visual boundary grouping agents that share memory/supervisor |
| Supervisor Arrow | ──⊛▶ | Orchestrator → managed agent relationship |
| Handoff Arrow | ⇄──▶ | Agent-to-agent handoff with mode label (transfer/consult/collaborate) |
| Portal | ╔══╗ | Link to another domain (double-click to jump) |
| Event Arrow | ──▶ | Async event connection between flows or to portals |

### Domain Blocks (Level 1)

| Type | Icon | Purpose |
|------|------|---------|
| Domain Block | ▣ | Represents a domain (shows flow count, double-click to drill in) |
| Event Arrow | ──▶ | Async event connection between domains |

## Node Output Handles (sourceHandle Routing)

Nodes with multiple output paths use named `sourceHandle` values to distinguish connections. This is critical for both the DDD Tool canvas rendering and for Claude Code implementation routing.

| Node Type | Output Handles | Visual | Color |
|-----------|---------------|--------|-------|
| Input | `valid` / `invalid` | "Ok / Err" labels | Green / Red |
| Decision | `true` / `false` | "Yes / No" labels | Green / Red |
| Data Store | `success` / `error` | "Ok / Err" labels | Green / Red |
| Service Call | `success` / `error` | "Ok / Err" labels | Green / Red |
| Loop | `body` / `done` | "Body / Done" labels | Teal / Muted |
| Parallel | `branch-0`, `branch-1`, ... / `done` | Dynamic branch labels | Pink / Muted |
| Smart Router | Dynamic route names | Route labels | Pink |
| Cache | `hit` / `miss` | "Hit / Miss" labels | Amber / Muted |
| Collection | `result` / `empty` | "Result / Empty" labels | Cyan / Muted |
| Parse | `success` / `error` | "Ok / Err" labels | Lime / Red |
| Crypto | `success` / `error` | "Ok / Err" labels | Fuchsia / Red |
| Batch | `done` / `error` | "Done / Err" labels | Rose / Red |
| Transaction | `committed` / `rolled_back` | "Ok / Rollback" labels | Amber / Red |

**Connection YAML format with sourceHandle:**

```yaml
connections:
  - from: input-xK9mR2vL
    to: process-aPq3nW8j
    sourceHandle: valid        # Valid input → continue
  - from: input-xK9mR2vL
    to: terminal-bR7nP4qL
    sourceHandle: invalid      # Invalid input → error response

  - from: data_store-mN3pK8vR
    to: process-cQ5tW2xJ
    sourceHandle: success      # DB operation succeeded
  - from: data_store-mN3pK8vR
    to: terminal-eR9nP4qL
    sourceHandle: error        # DB error → error terminal

  - from: loop-fT2mK7vN
    to: process-gP4nW8xJ
    sourceHandle: body         # Loop iteration body
  - from: loop-fT2mK7vN
    to: terminal-hR6nP4qL
    sourceHandle: done         # After loop completes

  - from: parallel-jK8mR2vL
    to: service_call-kP3nW8xJ
    sourceHandle: branch-0     # First parallel branch
  - from: parallel-jK8mR2vL
    to: service_call-lQ5tW2xJ
    sourceHandle: branch-1     # Second parallel branch
  - from: parallel-jK8mR2vL
    to: process-mR7nP4qL
    sourceHandle: done         # Join point after all branches
```

**DDD Tool canvas rendering:**
- Dual-handle nodes show two labeled output handles at the bottom (positioned at 33% and 66%)
- Parallel nodes show N+1 handles (one per branch + done), evenly spaced
- Handle colors match their semantic meaning (green=success, red=error, teal=body, pink=branch)

**Claude Code implementation routing:**
When implementing a flow, Claude Code uses `sourceHandle` values to determine control flow:
- `valid`/`success` paths → continue normal execution
- `invalid`/`error` paths → error handling, early return with error codes from `errors.yaml`
- `body` → loop iteration logic
- `done` → post-loop continuation
- `branch-N` → parallel execution branches (Promise.all, asyncio.gather, etc.)

---

## Connection Types Between Flows

1. **Event-Based (Async):** Publisher/subscriber via event bus — shown as arrows on Level 1 and Level 2
2. **Direct Call (Sync):** Sub-flow node with input/output mapping — shown as ▢ node on Level 3
3. **Shared Data Models:** `$ref:/schemas/SchemaName` references
4. **API Contracts:** Internal APIs connecting domains
5. **Agent Delegation:** Router or Agent Loop hands off to sub-agent flows
6. **Orchestrator → Agent:** Supervisor arrow on Level 2, orchestrator node on Level 3
7. **Agent Handoff:** Formal context transfer with handoff protocol (transfer/consult/collaborate)

---

## Node Type Spec Fields

This section defines the `spec` fields for each of the 27 node types. For connection patterns (sourceHandle values), see "Node Output Handles" above.

### Traditional Node Specs

#### trigger
The entry point of every flow. Exactly one per flow.

| Spec Field | Type | Description |
|------------|------|-------------|
| `event` | string \| string[] | What triggers the flow (see trigger conventions below) |
| `source` | string | Where the trigger comes from |
| `filter` | `Record<string, unknown>`? | Event payload filter — flow only triggers when filter matches |
| `job_config` | object? | Job queue config for cron triggers (see below) |
| `description` | string | Details |

**Trigger type conventions** — use these patterns in the `event` field:
- HTTP trigger: `"HTTP POST /api/users/register"`
- Scheduled trigger: `"cron */30 * * * *"`
- Event trigger: `"event:UserRegistered"`
- Webhook trigger: `"webhook /stripe/events"`
- Manual trigger: `"manual"`
- SSE trigger: `"sse /api/updates"`
- WebSocket trigger: `"ws /api/live"`
- Event pattern trigger: `"pattern:ContentAnalyzed"`
- Multi-event: string array `["event:ContentDiscovered", "event:SourceContentFound"]`

**job_config** — optional fields for cron triggers:

| Field | Type | Description |
|-------|------|-------------|
| `queue` | string? | Named queue for the job |
| `concurrency` | number? | Max concurrent executions (default: 1) |
| `timeout_ms` | number? | Job timeout in milliseconds |
| `retry` | object? | `{ max_attempts?, backoff_ms?, strategy?, jitter? }` |
| `dead_letter` | boolean? | Send failed jobs to dead letter queue |
| `lock_ttl_ms` | number? | Distributed lock TTL |
| `priority` | number? | Job priority (higher = processed first) |
| `dedup_key` | string? | Deduplication key template |

#### input
Validates incoming data. Has two output handles: `valid` and `invalid`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `fields` | `Array<{ name, type, required? }>` | Field definitions |
| `validation` | string | Validation rules/regex |
| `description` | string | What this input represents |

#### process
Business logic step.

| Spec Field | Type | Description |
|------------|------|-------------|
| `action` | string | What this step does |
| `service` | string | Service/function to call |
| `category` | string? | `'security' \| 'transform' \| 'integration' \| 'business_logic' \| 'infrastructure'` |
| `inputs` | string[]? | Explicit input fields (e.g., `["$.password"]`) |
| `outputs` | string[]? | Explicit output fields (e.g., `["$.hashed_password"]`) |
| `description` | string | Details |

#### decision
Branching point. Has two output handles: `true` and `false`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `condition` | string | The condition to evaluate |
| `trueLabel` | string | Label for the true branch |
| `falseLabel` | string | Label for the false branch |
| `description` | string | Details |

#### terminal
End state. Must have zero outgoing connections.

| Spec Field | Type | Description |
|------------|------|-------------|
| `outcome` | string | Result (e.g., "success", "error", "timeout") |
| `description` | string | What happens at this endpoint |
| `status` | number? | HTTP status code |
| `body` | object? | Response body shape |
| `response_type` | string? | `'json' \| 'stream' \| 'sse' \| 'empty'` (default: 'json') |
| `headers` | `Record<string, string>?` | Custom response headers |

#### data_store
Database operation. Has two output handles: `success` and `error`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `operation` | string | `'create' \| 'read' \| 'update' \| 'delete' \| 'upsert' \| 'create_many' \| 'update_many' \| 'delete_many'` |
| `model` | string | Entity/table name (e.g., "User") |
| `data` | `Record<string, string>` | Fields to write (for create/update) |
| `query` | `Record<string, string>` | Query conditions (for read/update/delete) |
| `description` | string | Details |
| `pagination` | object? | Pagination config (for list operations) |
| `sort` | object? | Sort config (for list operations) |
| `upsert_key` | string[]? | Fields for upsert conflict resolution |
| `include` | IncludeRelation[]? | Eager-load related records (joins) |
| `returning` | boolean? | Return affected records (for bulk operations) |

**IncludeRelation fields:**

| Field | Type | Description |
|-------|------|-------------|
| `model` | string | Related entity name |
| `via` | string | Foreign key or relation path (supports dot notation for nested) |
| `as` | string | Output field name |

#### service_call
External API call. Has two output handles: `success` and `error`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `method` | string | `'GET' \| 'POST' \| 'PUT' \| 'PATCH' \| 'DELETE'` |
| `url` | string | API endpoint URL |
| `headers` | `Record<string, string>` | HTTP headers |
| `body` | `Record<string, unknown>` | Request body |
| `timeout_ms` | number | Request timeout |
| `retry` | object | `{ max_attempts?, backoff_ms?, strategy? }` |
| `error_mapping` | `Record<string, string>` | Status code to error code mapping |
| `request_config` | RequestConfig? | Outbound request behavior configuration |
| `integration` | string? | Reference to integration defined in system.yaml |
| `description` | string | Details |

**RequestConfig fields:**

| Field | Type | Description |
|-------|------|-------------|
| `user_agent` | string? | `'rotate' \| 'browser' \| 'custom'` |
| `delay` | object? | `{ min_ms, max_ms, strategy }` |
| `cookie_jar` | string? | `'per_domain' \| 'shared' \| 'none'` |
| `proxy` | string? | `'pool' \| 'direct' \| 'tor'` |
| `tls_fingerprint` | string? | `'randomize' \| 'chrome' \| 'firefox' \| 'default'` |
| `fallback` | string? | `'headless_browser' \| 'none'` |

#### event
Publish or subscribe to an event.

| Spec Field | Type | Description |
|------------|------|-------------|
| `direction` | string | `'emit' \| 'consume'` |
| `event_name` | string | Event identifier |
| `payload` | `Record<string, unknown>` | Event data shape (static template) |
| `payload_source` | string? | Dynamic payload variable reference (overrides static `payload`) |
| `async` | boolean | Fire-and-forget? |
| `target_queue` | string? | Specific queue name |
| `priority` | number? | Job priority when enqueueing |
| `delay_ms` | number? | Delay before processing |
| `dedup_key` | string? | Deduplication key |
| `description` | string | Details |

#### loop
Iterate over a collection. Has two output handles: `body` and `done`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `collection` | string | What to iterate (e.g., "$.items") |
| `iterator` | string | Iterator variable name |
| `break_condition` | string | When to stop early |
| `on_error` | string? | `'continue' \| 'break' \| 'fail'` (default: 'fail') |
| `accumulate` | AccumulateConfig? | Collect results across iterations |
| `description` | string | Details |

**AccumulateConfig fields:**

| Field | Type | Description |
|-------|------|-------------|
| `field` | string | What to collect from each iteration's output |
| `strategy` | string | `'append' \| 'merge' \| 'sum' \| 'last'` |
| `output` | string | Variable name available at "done" handle |

#### parallel
Run branches concurrently. Has N+1 output handles: `branch-0`, `branch-1`, ... + `done`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `branches` | `ParallelBranch[] \| string[] \| number` | Branch definitions |
| `join` | string | `'all' \| 'any' \| 'n_of'` |
| `join_count` | number | Required if join is `n_of` |
| `timeout_ms` | number | Max wait time |
| `description` | string | Details |

**ParallelBranch fields:**

| Field | Type | Description |
|-------|------|-------------|
| `id` | string? | Branch identifier |
| `label` | string | Branch description |
| `condition` | string? | Skip branch if condition evaluates to false |

#### sub_flow
Reference another flow.

| Spec Field | Type | Description |
|------------|------|-------------|
| `flow_ref` | string | Target flow in `domain/flow-id` format |
| `input_mapping` | `Record<string, string>` | Input parameter mapping |
| `output_mapping` | `Record<string, string>` | Output parameter mapping |
| `description` | string | Details |

#### delay
Introduce a deliberate wait.

| Spec Field | Type | Description |
|------------|------|-------------|
| `min_ms` | number | Minimum delay in milliseconds |
| `max_ms` | number | Maximum delay (if strategy is random) |
| `strategy` | string | `'fixed' \| 'random'` |
| `description` | string | Why this delay exists |

#### cache
Check cache before expensive operations. Has two output handles: `hit` and `miss`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `key` | string | Cache key template (e.g., "search:{query}") |
| `ttl_ms` | number | Time-to-live in milliseconds |
| `store` | string | `'redis' \| 'memory'` |
| `description` | string | What is being cached |

#### transform
Structured data mapping between formats.

| Spec Field | Type | Description |
|------------|------|-------------|
| `input_schema` | string | Source data format reference |
| `output_schema` | string | Target data format reference |
| `field_mappings` | `Record<string, string>` | Output field to input field/expression mapping |
| `description` | string | What transformation is performed |

#### collection
Collection operations. Has two output handles: `result` and `empty`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `operation` | string | `'filter' \| 'sort' \| 'deduplicate' \| 'merge' \| 'group_by' \| 'aggregate' \| 'reduce' \| 'flatten'` |
| `input` | string | Input collection reference (e.g., `"$.sources"`) |
| `predicate` | string? | Filter expression (for `filter`) |
| `key` | string? | Field key (for `deduplicate`, `sort`, `group_by`) |
| `direction` | string? | `'asc' \| 'desc'` (for `sort`) |
| `accumulator` | object? | `{ init: any, expression: string }` (for `reduce`/`aggregate`) |
| `output` | string | Output variable name |
| `description` | string | Details |

#### parse
Structured extraction from raw content formats. Has two output handles: `success` and `error`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `format` | string | `'rss' \| 'atom' \| 'html' \| 'xml' \| 'json' \| 'csv' \| 'markdown'` |
| `input` | string | Raw content variable (e.g., `"$.raw_response"`) |
| `strategy` | string \| object | `'strict' \| 'lenient' \| 'streaming'` or `{ selectors: [...] }` for HTML scraping |
| `library` | string? | Implementation hint (e.g., `'cheerio'`, `'rss-parser'`) |
| `output` | string | Output variable name |
| `description` | string | Details |

#### crypto
Cryptographic operations. Has two output handles: `success` and `error`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `operation` | string | `'encrypt' \| 'decrypt' \| 'hash' \| 'sign' \| 'verify' \| 'generate_key'` |
| `algorithm` | string | `'aes-256-gcm' \| 'sha256' \| 'sha512' \| 'hmac-sha256' \| 'rsa-oaep' \| 'ed25519'` |
| `key_source` | object | Where the key comes from: `{ env: string }` |
| `input_fields` | string[] | Field(s) to process |
| `output_field` | string | Result field name |
| `encoding` | string? | `'base64' \| 'hex'` |
| `description` | string | Details |

#### batch
Execute operations against a collection with concurrency control. Has two output handles: `done` and `error`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `input` | string | Collection to iterate over |
| `operation_template` | object | Per-item operation config (type, dispatch_field, configs) |
| `concurrency` | number? | Max parallel operations |
| `on_error` | string? | `'continue' \| 'stop'` |
| `output` | string | Output variable (array of `{ item, result, success }`) |
| `description` | string | Details |

#### transaction
Atomic multi-step database operations. Has two output handles: `committed` and `rolled_back`.

| Spec Field | Type | Description |
|------------|------|-------------|
| `isolation` | string? | `'read_committed' \| 'serializable' \| 'repeatable_read'` |
| `steps` | array | Ordered operations: `[{ action, rollback? }]` |
| `rollback_on_error` | boolean? | Auto-rollback on any step failure (default: true) |
| `description` | string | Details |

### Cross-Cutting Concerns (Per-Node)

Every node can optionally have `observability` and `security` configs:

**Observability:**

```yaml
observability:
  logging:
    level: info          # debug | info | warn | error
    include_input: true
    include_output: false
  metrics:
    enabled: true
    custom_counters:
      - requests
      - errors
  tracing:
    enabled: true
    span_name: node.validate_input
```

**Security:**

```yaml
security:
  authentication:
    required: true
    methods: [jwt, api_key]
    roles: [admin, user]
  rate_limiting:
    enabled: true
    requests_per_minute: 60
  encryption:
    at_rest: true
    in_transit: true
    pii_fields: [email, ssn]
  audit:
    enabled: true
```

---

## Agent Infrastructure

### Flow Types

DDD supports two flow types, each with different canvas behavior:

| Flow Type | `type:` value | Canvas Behavior |
|-----------|---------------|-----------------|
| **Traditional** | `traditional` (default) | Fixed node-to-node paths, deterministic execution |
| **Agent** | `agent` | Dynamic tool selection, iterative reasoning loops, non-deterministic paths |

When a flow has `type: agent`, the Level 3 canvas shows an **agent-centric layout** — the Agent Loop is the central element, with tools, guardrails, and memory arranged around it.

### Agent Flow Canvas Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ FLOW: support-agent (type: agent)                                │
│                                                                   │
│   ⛨ Input                                                       │
│   Guardrail                                                       │
│      │                                                            │
│      ▼                                                            │
│   ┌─────────────────────────────────────────────┐                │
│   │            ↻ AGENT LOOP                      │                │
│   │                                              │                │
│   │  System Prompt: "You are a helpful..."       │                │
│   │  Model: claude-sonnet-4-5-20250929                    │                │
│   │  Max Iterations: 10                          │                │
│   │                                              │                │
│   │  ┌─────────────────────────────────┐         │                │
│   │  │  Reason → Select → Execute →    │         │                │
│   │  │  Observe → Repeat               │         │                │
│   │  └─────────────────────────────────┘         │                │
│   │                                              │                │
│   │  Available Tools:                            │                │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │                │
│   │  │🔧 search │ │🔧 create │ │🔧 escalate│    │                │
│   │  │   -kb    │ │  -ticket │ │          │    │                │
│   │  └──────────┘ └──────────┘ └──────────┘    │                │
│   │                                              │                │
│   │  Memory:                                     │                │
│   │  ◈ conversation (8000 tokens)               │                │
│   └─────────────────────────────────────────────┘                │
│      │                                                            │
│      ▼                                                            │
│   ⛨ Output          ✋ Human Gate                                │
│   Guardrail          (if escalation needed)                       │
│      │                    │                                       │
│      ▼                    ▼                                       │
│   ⬭ Response         ⬭ Escalated                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Agent Node Specifications

#### LLM Call Node (◆)

A single LLM invocation with a prompt template. Used in traditional flows when you need an LLM step without a full agent loop.

```yaml
- id: classify_intent
  type: llm_call
  spec:
    model: claude-haiku-4-5-20251001
    system_prompt: "Classify the user's intent into one of these categories."
    prompt_template: |
      User message: {{$.user_message}}

      Categories: billing, technical, general, escalation

      Respond with JSON: {"intent": "<category>", "confidence": <0-1>}
    temperature: 0.0
    max_tokens: 100
    structured_output:
      type: object
      properties:
        intent:
          type: string
          enum: [billing, technical, general, escalation]
        confidence:
          type: number
    retry:
      max_attempts: 2
      fallback_model: claude-sonnet-4-5-20250929
  connections:
    success: route_by_intent
    failure: return_error
```

#### Agent Loop Node (↻)

The core agent pattern — iterative reasoning with dynamic tool selection. This is the central node in agent flows.

```yaml
- id: support_agent_loop
  type: agent_loop
  spec:
    model: claude-sonnet-4-5-20250929
    system_prompt: |
      You are a customer support agent for Obligo.
      You help users with contract questions, obligation tracking,
      and account management.

      Guidelines:
      - Always search the knowledge base before answering
      - If you can't resolve the issue, create a ticket
      - Escalate to human if the customer is frustrated

    max_iterations: 10
    stop_conditions:
      - tool_called: respond_to_user    # Stop when agent calls this tool
      - tool_called: escalate_to_human
      - max_iterations_reached: true

    tools:
      - $ref: "#/tools/search_kb"
      - $ref: "#/tools/create_ticket"
      - $ref: "#/tools/get_account"
      - $ref: "#/tools/respond_to_user"
      - $ref: "#/tools/escalate_to_human"

    memory:
      type: conversation
      max_tokens: 8000
      include_tool_results: true

    scratchpad: true               # Agent can write internal notes

    on_max_iterations:
      action: escalate             # What to do if loop doesn't converge
      connection: escalation_gate

  connections:
    respond_to_user: output_guardrail
    escalate_to_human: escalation_gate
    error: return_error
```

#### Tool Node (🔧)

Defines a tool available to an agent. Tools are not hard-wired into the flow — the agent selects them dynamically.

```yaml
tools:
  - id: search_kb
    type: tool
    spec:
      name: search_kb
      description: "Search the knowledge base for articles matching a query"
      parameters:
        query:
          type: string
          required: true
          description: "Search query"
        limit:
          type: integer
          default: 5
          description: "Max results to return"
      implementation:
        type: service_call          # How the tool is actually executed
        service: knowledge_base
        method: search
      returns:
        type: array
        items:
          type: object
          properties:
            title: { type: string }
            content: { type: string }
            relevance: { type: number }

  - id: create_ticket
    type: tool
    spec:
      name: create_ticket
      description: "Create a support ticket for issues that need follow-up"
      parameters:
        subject:
          type: string
          required: true
        description:
          type: string
          required: true
        priority:
          type: string
          enum: [low, medium, high, urgent]
          default: medium
      implementation:
        type: data_store
        operation: create
        model: SupportTicket
      requires_confirmation: false   # Set true for destructive actions

  - id: escalate_to_human
    type: tool
    spec:
      name: escalate_to_human
      description: "Escalate to a human agent. Use when the customer is frustrated or the issue is too complex."
      parameters:
        reason:
          type: string
          required: true
        urgency:
          type: string
          enum: [normal, high, critical]
      implementation:
        type: event
        event: support.escalated
      is_terminal: true              # Calling this tool ends the agent loop

  - id: respond_to_user
    type: tool
    spec:
      name: respond_to_user
      description: "Send a response to the user. Call this when you have a complete answer."
      parameters:
        message:
          type: string
          required: true
      is_terminal: true              # Calling this tool ends the agent loop
```

#### Memory Node (◈)

Manages agent memory — conversation history, vector store, or shared context.

```yaml
- id: agent_memory
  type: memory
  spec:
    stores:
      - name: conversation
        type: conversation_history
        max_tokens: 8000
        strategy: sliding_window     # sliding_window | summarize | truncate
        include_system: true
        include_tool_results: true

      - name: knowledge
        type: vector_store
        provider: pgvector           # pgvector | pinecone | qdrant | chromadb
        embedding_model: text-embedding-3-small
        similarity_metric: cosine
        top_k: 5
        min_similarity: 0.7

      - name: user_context
        type: key_value
        storage: redis
        ttl: 3600
        fields:
          - user_id
          - account_tier
          - recent_tickets
```

#### Guardrail Node (⛨)

Content filtering and validation for LLM inputs/outputs.

```yaml
- id: input_guardrail
  type: guardrail
  spec:
    position: input                  # input | output
    checks:
      - type: content_filter
        block_categories:
          - hate_speech
          - self_harm
          - illegal_activity
        action: block
        message: "I'm unable to help with that request."

      - type: pii_detection
        detect: [email, phone, ssn, credit_card]
        action: mask                 # mask | block | log
        mask_format: "***"

      - type: topic_restriction
        allowed_topics:
          - contracts
          - obligations
          - account_management
          - billing
        action: redirect
        message: "I can only help with Obligo-related questions."

      - type: prompt_injection
        enabled: true
        action: block
        message: "I detected an unusual request pattern."

    on_block:
      connection: return_blocked_response

  connections:
    pass: agent_loop
    block: return_blocked_response
```

```yaml
- id: output_guardrail
  type: guardrail
  spec:
    position: output
    checks:
      - type: content_filter
        block_categories: [hate_speech]
        action: block

      - type: factuality
        enabled: true
        sources: [knowledge_base]     # Cross-check against these
        action: warn                  # warn | block

      - type: tone
        required: professional
        action: rewrite              # Ask LLM to rewrite if tone is off

      - type: no_hallucinated_urls
        action: block

      - type: schema_validation
        schema:
          type: object
          required: [message]
          properties:
            message: { type: string, max_length: 2000 }

    on_block:
      connection: return_fallback_response

  connections:
    pass: return_response
    block: return_fallback_response
```

#### Human Gate Node (✋)

Pauses execution and waits for human approval.

```yaml
- id: escalation_gate
  type: human_gate
  spec:
    notification:
      channels:
        - type: slack
          channel: "#support-escalations"
          message_template: |
            🚨 Escalation from agent
            Customer: {{$.user_id}}
            Reason: {{$.escalation_reason}}
            Conversation: {{$.conversation_summary}}
        - type: email
          to: "support-leads@obligo.io"

    approval_options:
      - id: approve
        label: "Take Over"
        description: "Human agent takes over the conversation"
      - id: reject
        label: "Send Back to AI"
        description: "Return to AI agent with instructions"
        requires_input: true         # Human can add instructions
      - id: resolve
        label: "Resolve"
        description: "Mark as resolved with a response"
        requires_input: true

    timeout:
      duration: 300                  # 5 minutes
      action: auto_escalate          # auto_escalate | auto_approve | return_error
      fallback_connection: timeout_response

    context_for_human:               # What the human sees
      - conversation_history
      - customer_account
      - agent_reasoning

  connections:
    approve: human_takeover
    reject: agent_loop               # Send back with human instructions
    resolve: return_resolved
    timeout: timeout_response
```

#### Router Node (◇◇)

Semantic routing — uses an LLM to classify and route to different sub-agents or flows.

```yaml
- id: intent_router
  type: router
  spec:
    model: claude-haiku-4-5-20251001
    routing_prompt: |
      Classify the user's message into one of these categories:
      - billing: Payment, invoices, subscription questions
      - technical: Contract analysis, obligation tracking, integrations
      - general: Account settings, feature questions, how-to
      - escalation: Angry customer, legal threats, data deletion requests

    routes:
      - id: billing
        description: "Billing and payment questions"
        connection: billing_agent
      - id: technical
        description: "Technical support"
        connection: technical_agent
      - id: general
        description: "General inquiries"
        connection: general_agent
      - id: escalation
        description: "Requires immediate human attention"
        connection: escalation_gate

    fallback_route: general          # If classification fails
    confidence_threshold: 0.7        # Below this → fallback

  connections:
    billing: billing_agent_flow
    technical: technical_agent_flow
    general: general_agent_flow
    escalation: escalation_gate
```

### Complete Agent Flow Example

```yaml
# specs/domains/support/flows/customer-support-agent.yaml

flow:
  id: customer-support-agent
  name: Customer Support Agent
  type: agent                        # ← Agent flow type
  domain: support
  description: AI-powered customer support with human escalation

trigger:
  type: http
  method: POST
  path: /api/v1/support/chat
  auth: required

agent_config:
  model: claude-sonnet-4-5-20250929
  system_prompt: |
    You are a customer support agent for Obligo, a cyber liability
    management platform. You help users with contract questions,
    obligation tracking, and account management.

    Guidelines:
    - Always greet the customer by name
    - Search the knowledge base before answering questions
    - If you can't resolve in 3 tool calls, offer to create a ticket
    - Escalate to human if customer expresses frustration
    - Never make up information about contracts or obligations

  max_iterations: 10
  temperature: 0.3

memory:
  - name: conversation
    type: conversation_history
    max_tokens: 8000
    strategy: sliding_window
  - name: user_context
    type: key_value
    load_on_start:
      - user_profile
      - recent_tickets
      - account_tier

tools:
  - id: search_kb
    name: search_kb
    description: "Search knowledge base for help articles"
    parameters:
      query: { type: string, required: true }
    implementation:
      type: service_call
      service: knowledge_base
      method: semantic_search

  - id: get_contracts
    name: get_contracts
    description: "Get user's contracts and their status"
    parameters:
      status_filter: { type: string, enum: [all, active, expiring] }
    implementation:
      type: data_store
      operation: list
      model: Contract
      filter: "tenant_id = $.tenant_id"

  - id: get_obligations
    name: get_obligations
    description: "Get obligations for a specific contract"
    parameters:
      contract_id: { type: string, required: true }
    implementation:
      type: data_store
      operation: list
      model: Obligation
      filter: "contract_id = $.contract_id"

  - id: create_ticket
    name: create_ticket
    description: "Create a support ticket for follow-up"
    parameters:
      subject: { type: string, required: true }
      description: { type: string, required: true }
      priority: { type: string, enum: [low, medium, high], default: medium }
    implementation:
      type: data_store
      operation: create
      model: SupportTicket
    requires_confirmation: false

  - id: respond_to_user
    name: respond_to_user
    description: "Send final response to the user"
    parameters:
      message: { type: string, required: true }
    is_terminal: true

  - id: escalate_to_human
    name: escalate_to_human
    description: "Escalate to human agent when AI cannot resolve"
    parameters:
      reason: { type: string, required: true }
      urgency: { type: string, enum: [normal, high, critical] }
    is_terminal: true
    implementation:
      type: event
      event: support.escalated

guardrails:
  input:
    - type: content_filter
      block_categories: [hate_speech, illegal_activity]
      action: block
    - type: pii_detection
      detect: [ssn, credit_card]
      action: mask
    - type: prompt_injection
      enabled: true
      action: block

  output:
    - type: tone
      required: professional
      action: rewrite
    - type: no_hallucinated_urls
      action: block
    - type: schema_validation
      schema:
        type: object
        required: [message]
        properties:
          message: { type: string, max_length: 2000 }

nodes:
  - id: input_guard
    type: guardrail
    spec:
      position: input
      checks: $ref: "#/guardrails/input"
    connections:
      pass: agent_loop
      block: blocked_response

  - id: agent_loop
    type: agent_loop
    spec:
      config: $ref: "#/agent_config"
      tools: $ref: "#/tools"
      memory: $ref: "#/memory"
    connections:
      respond_to_user: output_guard
      escalate_to_human: escalation_gate
      error: error_response

  - id: output_guard
    type: guardrail
    spec:
      position: output
      checks: $ref: "#/guardrails/output"
    connections:
      pass: return_response
      block: fallback_response

  - id: escalation_gate
    type: human_gate
    spec:
      notification:
        channels:
          - type: slack
            channel: "#support-escalations"
      timeout:
        duration: 300
        action: auto_escalate
    connections:
      approve: human_takeover
      reject: agent_loop
      resolve: return_resolved
      timeout: timeout_response

  - id: return_response
    type: terminal
    spec:
      status: 200
      body:
        message: "$.agent_response"
        conversation_id: "$.conversation_id"

  - id: blocked_response
    type: terminal
    spec:
      status: 400
      body:
        error: "Your message could not be processed"
        code: CONTENT_BLOCKED

  - id: error_response
    type: terminal
    spec:
      status: 500
      body:
        error: "An error occurred. Please try again."
        code: AGENT_ERROR

metadata:
  created_by: murat
  created_at: 2025-02-13
  completeness: 100
```

### Orchestration Node Specifications

#### Orchestrator Node (⊛)

A supervisor that manages multiple agents. It has its own reasoning model and decides which agent to activate, monitors their progress, and can intervene.

```yaml
- id: support_orchestrator
  type: orchestrator
  spec:
    strategy: supervisor             # supervisor | round_robin | broadcast | consensus

    # Supervisor's own reasoning model
    model: claude-sonnet-4-5-20250929
    supervisor_prompt: |
      You manage a team of support agents. Given the user's message
      and context, decide which specialist to route to. Monitor their
      progress and intervene if they get stuck.

    # Managed agents
    agents:
      - id: billing_agent
        flow: billing-agent
        domain: support
        specialization: "Billing, payments, invoices"
        priority: 1                  # Lower = higher priority when multiple match
      - id: technical_agent
        flow: technical-agent
        domain: support
        specialization: "Contract analysis, integrations, API questions"
        priority: 2
      - id: general_agent
        flow: general-agent
        domain: support
        specialization: "Account questions, how-to, general inquiries"
        priority: 3

    # How the orchestrator picks an agent
    routing:
      primary: smart_router          # Node ID of the Smart Router
      fallback_chain:                # If primary fails, try these in order
        - general_agent
        - human_escalation

    # Shared memory accessible by all managed agents
    shared_memory:
      - name: conversation
        type: conversation_history
        access: read_write           # All agents see same conversation
      - name: customer_context
        type: key_value
        access: read_only            # Agents can read, orchestrator writes
        fields: [customer_id, tier, recent_tickets, satisfaction_score]

    # Supervision rules
    supervision:
      monitor_iterations: true
      monitor_tool_calls: true
      monitor_confidence: true

      intervene_on:
        - condition: agent_iterations_exceeded
          threshold: 5
          action: reassign
          target: next_in_priority

        - condition: confidence_below
          threshold: 0.3
          action: add_instructions
          instructions_prompt: |
            The agent seems unsure. Provide additional context or
            suggest a different approach.

        - condition: customer_sentiment
          sentiment: frustrated
          action: escalate_to_human

        - condition: agent_error
          action: retry_with_different_agent
          max_retries: 1

        - condition: timeout
          threshold: 60              # seconds
          action: escalate_to_human

    # How to merge results when multiple agents contribute
    result_merge:
      strategy: last_wins            # last_wins | best_of | combine | supervisor_picks
      # best_of: supervisor evaluates all agent responses and picks best
      # combine: supervisor synthesizes a combined response
      # supervisor_picks: supervisor sees all results and writes final response

  connections:
    resolved: output_guardrail
    escalated: human_gate
    error: error_response
```

**Orchestration Strategies:**

| Strategy | How it works | Use when |
|----------|-------------|----------|
| **supervisor** | Orchestrator's LLM reasons about which agent to activate next, monitors, and can intervene | Complex routing with dynamic decisions |
| **round_robin** | Distribute requests evenly across agents | Load balancing homogeneous agents |
| **broadcast** | Send to all agents simultaneously, merge results | Need multiple perspectives (consensus, fact-checking) |
| **consensus** | All agents respond, orchestrator's LLM picks best or synthesizes | High-stakes decisions requiring agreement |

#### Smart Router Node (◇◇+)

Enhanced router with rule-based routing, LLM fallback, policies, and A/B testing. Replaces the basic Router node for orchestration scenarios.

```yaml
- id: smart_router
  type: smart_router
  spec:
    # ── Rule-based routes (evaluated first, fast, no LLM call) ──
    rules:
      - id: enterprise_route
        condition: "$.customer.tier == 'enterprise'"
        route: enterprise_agent
        priority: 1                  # Lower = evaluated first

      - id: cancellation_route
        condition: "$.intent == 'cancel' or $.message icontains 'cancel'"
        route: retention_agent
        priority: 2

      - id: billing_keywords
        condition: "$.message icontains 'invoice' or $.message icontains 'payment' or $.message icontains 'charge'"
        route: billing_agent
        priority: 3

      - id: urgent_route
        condition: "$.customer.open_tickets > 3 and $.customer.satisfaction_score < 3"
        route: human_escalation
        priority: 0                  # Highest priority

    # ── LLM-based classification (when no rule matches) ──
    llm_routing:
      enabled: true
      model: claude-haiku-4-5-20251001
      routing_prompt: |
        Classify the customer's intent into one of these categories:
        - billing: Payment, invoices, subscription, pricing
        - technical: Contract analysis, integrations, API, data
        - general: Account settings, features, how-to, feedback
        - escalation: Legal threats, data deletion, angry customer

        Customer tier: {{$.customer.tier}}
        Recent tickets: {{$.customer.open_tickets}}
        Message: {{$.message}}

        Respond with JSON: {"route": "<category>", "confidence": <0-1>}
      confidence_threshold: 0.7
      temperature: 0.0

      routes:
        billing: billing_agent
        technical: technical_agent
        general: general_agent
        escalation: human_escalation

    # ── Fallback chain (when LLM confidence < threshold or error) ──
    fallback_chain:
      - general_agent
      - human_escalation

    # ── Routing policies ──
    policies:
      retry:
        max_attempts: 2
        on_failure: next_in_fallback_chain
        delay_ms: 0

      timeout:
        per_route: 30                # seconds per agent attempt
        total: 120                   # seconds total across all retries
        action: next_in_fallback_chain

      circuit_breaker:
        enabled: true
        failure_threshold: 3         # Consecutive failures before opening
        recovery_time: 60            # Seconds before trying again
        half_open_requests: 1        # Test requests when recovering
        on_open: next_in_fallback_chain

    # ── A/B testing ──
    ab_testing:
      enabled: true
      experiments:
        - name: new_billing_agent_v2
          route: billing_agent_v2
          percentage: 20             # 20% of billing traffic
          original_route: billing_agent
          metrics:
            - resolution_rate
            - customer_satisfaction
            - average_iterations

        - name: fast_general_model
          route: general_agent_fast  # Same agent, different model
          percentage: 10
          original_route: general_agent

    # ── Context-aware routing ──
    context_routing:
      enabled: true
      rules:
        # If customer was recently talking to billing agent, route back there
        - condition: "$.session.last_agent == 'billing_agent' and $.session.turns_since < 3"
          route: billing_agent
          reason: "Continue with same agent for conversation continuity"

        # If this is a follow-up to an escalated conversation
        - condition: "$.session.was_escalated == true"
          route: human_escalation
          reason: "Previously escalated, go directly to human"

  connections:
    billing_agent: billing_agent_flow
    technical_agent: technical_agent_flow
    general_agent: general_agent_flow
    enterprise_agent: enterprise_agent_flow
    retention_agent: retention_agent_flow
    human_escalation: escalation_gate
    billing_agent_v2: billing_agent_v2_flow
    general_agent_fast: general_agent_fast_flow
```

#### Handoff Node (⇄)

Formal protocol for transferring context between agents.

```yaml
- id: handoff_to_specialist
  type: handoff
  spec:
    # ── Handoff Mode ──
    mode: consult                    # transfer | consult | collaborate

    # transfer: Source agent stops, target takes over completely
    # consult:  Source agent pauses, target answers, result returns to source
    # collaborate: Both agents active, shared context, orchestrator merges

    # ── Target ──
    target:
      flow: technical-agent
      domain: support

    # ── Context Transfer ──
    context_transfer:
      # What gets passed to the target agent
      include:
        - type: conversation_summary
          generator: llm              # LLM generates summary (not raw history)
          model: claude-haiku-4-5-20251001
          max_tokens: 500
          prompt: |
            Summarize this conversation for a specialist agent.
            Focus on: what the customer needs, what's been tried,
            and any relevant account details.

        - type: structured_data
          fields:
            customer_id: "$.customer.id"
            customer_tier: "$.customer.tier"
            original_intent: "$.routing.classified_intent"
            attempted_solutions: "$.agent.tool_calls_summary"

        - type: task_description
          content: |
            The customer needs help with {{$.routing.classified_intent}}.
            Previous agent could not resolve. Please assist.

      exclude:
        - internal_reasoning          # Don't share source agent's scratchpad
        - raw_tool_results            # Only send summaries
        - system_prompts              # Each agent has its own

      max_context_tokens: 2000       # Hard limit on transferred context

    # ── What happens when target finishes ──
    on_complete:
      return_to: source_agent        # source_agent | orchestrator | terminal
      merge_strategy: append         # append | replace | summarize

      # For 'summarize': LLM summarizes the specialist's work
      summarize_prompt: |
        Summarize what the specialist found and recommended.
        Keep it concise for the customer.

    # ── Failure handling ──
    on_failure:
      action: return_with_error      # return_with_error | retry | escalate
      fallback: escalate_to_human
      timeout: 60                    # seconds

    # ── Handoff notification (optional) ──
    notify_customer: true
    notification_message: |
      I'm connecting you with a specialist who can better help
      with your {{$.routing.classified_intent}} question.

  connections:
    complete: return_to_orchestrator
    failure: escalation_gate
    timeout: timeout_response
```

**Handoff Modes:**

| Mode | Source Agent | Target Agent | Context Flow | Use When |
|------|-------------|-------------|--------------|----------|
| **transfer** | Stops completely | Takes over | One-way: source → target | Routing to specialist, escalation |
| **consult** | Pauses, waits | Answers specific question | Round-trip: source → target → source | Need expert opinion, then continue |
| **collaborate** | Stays active | Also active | Bidirectional via shared memory | Complex tasks needing multiple specialists |

#### Agent Group Node ([⊛])

A container that groups agents sharing memory and coordination policy. On Level 2, rendered as a visual boundary around grouped agent flows.

```yaml
- id: support_team
  type: agent_group
  spec:
    name: Support Team
    description: Customer support agents working together

    # Agents in this group
    members:
      - flow: billing-agent
      - flow: technical-agent
      - flow: general-agent
      - flow: retention-agent

    # Shared resources within the group
    shared_memory:
      - name: conversation
        type: conversation_history
        access: read_write
        description: "All agents see the same conversation thread"
      - name: customer_context
        type: key_value
        access: read_only
        fields: [customer_id, tier, history_summary]
      - name: team_knowledge
        type: vector_store
        provider: pgvector
        access: read_only
        description: "Shared knowledge base for all support agents"

    # Coordination policy
    coordination:
      # How agents communicate within the group
      communication: via_orchestrator  # via_orchestrator | direct | blackboard

      # via_orchestrator: All inter-agent communication goes through the orchestrator
      # direct: Agents can call each other via handoff nodes
      # blackboard: Agents read/write to a shared blackboard (shared_memory)

      # Concurrency rules
      concurrency:
        max_active_agents: 1         # How many agents can be active at once
        # 1 = sequential (one at a time, typical for chat)
        # N = parallel (multiple working simultaneously, for batch/analysis)

      # Agent selection when multiple could handle a task
      selection:
        strategy: router_first       # router_first | round_robin | least_busy | random
        sticky_session: true         # Keep same agent for a conversation if possible
        sticky_timeout: 300          # Seconds before sticky session expires

    # Group-level guardrails (applied to all agents in group)
    guardrails:
      input:
        - type: content_filter
          block_categories: [hate_speech, illegal_activity]
      output:
        - type: tone
          required: professional

    # Group-level metrics (for A/B testing, monitoring)
    metrics:
      track:
        - resolution_rate
        - average_iterations
        - handoff_count
        - escalation_rate
        - customer_satisfaction
```

### Orchestration Visualization (Level 2)

When a domain contains orchestrated agents, the Level 2 Domain Map shows the relationships:

```
┌─────────────────────────────────────────────────────────────────┐
│ DOMAIN MAP: support                                              │
│                                                                   │
│   ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐         │
│   │  [⊛] Support Team (Agent Group)                   │         │
│   │                                                    │         │
│   │   ⊛ Support Orchestrator                          │         │
│   │   │ (supervisor)                                   │         │
│   │   │                                                │         │
│   │   ├──⊛▶── ┌──────────┐                           │         │
│   │   │        │▣⊛ billing │                           │         │
│   │   │        │   agent   │ ─── ⇄ consult ──┐       │         │
│   │   │        └──────────┘                   │       │         │
│   │   │                                       ▼       │         │
│   │   ├──⊛▶── ┌──────────┐           ┌──────────┐   │         │
│   │   │        │▣⊛ tech    │ ◀── ⇄ ──│▣⊛ general │   │         │
│   │   │        │   agent   │ consult  │   agent   │   │         │
│   │   │        └──────────┘           └──────────┘   │         │
│   │   │                                               │         │
│   │   └──⊛▶── ┌──────────┐                           │         │
│   │            │▣⊛retention│                           │         │
│   │            │   agent   │                           │         │
│   │            └──────────┘                           │         │
│   │                                                    │         │
│   │   ◈ Shared: conversation, customer_context        │         │
│   └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘         │
│                    │                                              │
│                    │ ⇄ transfer (escalation)                     │
│                    ▼                                              │
│              ✋ Human Gate                                        │
│                                                                   │
│   ╔════════════╗     ╔════════════╗                              │
│   ║ ➜ billing  ║     ║ ➜ analysis ║                              │
│   ╚════════════╝     ╚════════════╝                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Level 2 orchestration elements:**

| Element | Visual | Description |
|---------|--------|-------------|
| Agent Group boundary | Dashed rectangle | Groups agents sharing memory/supervisor |
| Orchestrator | ⊛ block at top of group | The supervisor agent managing the group |
| Supervisor arrow (⊛▶) | Solid arrow from orchestrator | Shows which agents are managed |
| Handoff arrow (⇄) | Bidirectional dashed arrow | Shows agent-to-agent handoff with mode label |
| Agent flow block (▣⊛) | Block with agent badge | Distinguishes agent flows from traditional flows |
| Shared memory indicator (◈) | Badge at bottom of group | Shows what memory is shared within the group |

### Complete Orchestration Flow Example

```yaml
# specs/domains/support/flows/support-orchestration.yaml

flow:
  id: support-orchestration
  name: Support Team Orchestration
  type: agent                        # Agent flow type
  domain: support
  description: >
    Orchestrated customer support with smart routing,
    specialist agents, handoffs, and human escalation.

trigger:
  type: http
  method: POST
  path: /api/v1/support/chat
  auth: required

# ── Agent Group Definition ──
agent_group:
  id: support_team
  name: Support Team
  shared_memory:
    - name: conversation
      type: conversation_history
      access: read_write
    - name: customer_context
      type: key_value
      access: read_only
      load_on_start: [customer_profile, recent_tickets, satisfaction_score]
  coordination:
    communication: via_orchestrator
    concurrency:
      max_active_agents: 1
    selection:
      sticky_session: true
      sticky_timeout: 300

# ── Nodes ──
nodes:
  # Input guardrail
  - id: input_guard
    type: guardrail
    spec:
      position: input
      checks:
        - type: content_filter
          block_categories: [hate_speech, illegal_activity]
          action: block
        - type: pii_detection
          detect: [ssn, credit_card]
          action: mask
        - type: prompt_injection
          enabled: true
          action: block
    connections:
      pass: orchestrator
      block: blocked_response

  # Orchestrator — the brain
  - id: orchestrator
    type: orchestrator
    spec:
      strategy: supervisor
      model: claude-sonnet-4-5-20250929
      supervisor_prompt: |
        You manage a customer support team. Route requests to the
        right specialist. Monitor their work. Intervene if needed.

      agents:
        - id: billing_agent
          flow: billing-agent
          specialization: "Billing, payments, invoices, subscriptions"
          priority: 1
        - id: technical_agent
          flow: technical-agent
          specialization: "Contracts, obligations, integrations, API"
          priority: 2
        - id: general_agent
          flow: general-agent
          specialization: "Account settings, features, how-to"
          priority: 3
        - id: retention_agent
          flow: retention-agent
          specialization: "Cancellations, complaints, win-back"
          priority: 1

      routing:
        primary: smart_router
        fallback_chain: [general_agent, human_gate]

      supervision:
        monitor_iterations: true
        intervene_on:
          - condition: agent_iterations_exceeded
            threshold: 5
            action: reassign
          - condition: customer_sentiment
            sentiment: frustrated
            action: escalate_to_human
          - condition: timeout
            threshold: 60
            action: escalate_to_human

      result_merge:
        strategy: last_wins

    connections:
      resolved: output_guard
      escalated: human_gate
      error: error_response

  # Smart Router — decides which agent handles the request
  - id: smart_router
    type: smart_router
    spec:
      rules:
        - id: urgent
          condition: "$.customer.open_tickets > 3 and $.customer.satisfaction_score < 3"
          route: human_gate
          priority: 0
        - id: enterprise
          condition: "$.customer.tier == 'enterprise'"
          route: technical_agent
          priority: 1
        - id: cancel_intent
          condition: "$.message icontains 'cancel' or $.message icontains 'downgrade'"
          route: retention_agent
          priority: 2
        - id: billing_keywords
          condition: "$.message icontains 'invoice' or $.message icontains 'payment'"
          route: billing_agent
          priority: 3

      llm_routing:
        enabled: true
        model: claude-haiku-4-5-20251001
        confidence_threshold: 0.7
        routes:
          billing: billing_agent
          technical: technical_agent
          general: general_agent
          escalation: human_gate

      fallback_chain: [general_agent]

      policies:
        timeout:
          per_route: 30
          total: 120
        circuit_breaker:
          enabled: true
          failure_threshold: 3
          recovery_time: 60

      context_routing:
        enabled: true
        rules:
          - condition: "$.session.last_agent != null and $.session.turns_since < 3"
            route: "$.session.last_agent"
            reason: "Conversation continuity"

      ab_testing:
        enabled: false

  # Handoff — when billing agent needs technical help
  - id: billing_to_tech_handoff
    type: handoff
    spec:
      mode: consult
      target:
        flow: technical-agent
        domain: support
      context_transfer:
        include:
          - type: conversation_summary
            generator: llm
            model: claude-haiku-4-5-20251001
            max_tokens: 500
          - type: structured_data
            fields:
              customer_id: "$.customer.id"
              billing_context: "$.agent.findings"
        max_context_tokens: 2000
      on_complete:
        return_to: source_agent
        merge_strategy: append
      on_failure:
        action: return_with_error
        timeout: 60

  # Human Gate
  - id: human_gate
    type: human_gate
    spec:
      notification:
        channels:
          - type: slack
            channel: "#support-escalations"
          - type: email
            to: "support-leads@obligo.io"
      approval_options:
        - id: take_over
          label: "Take Over"
          description: "Human agent takes the conversation"
        - id: send_back
          label: "Send Back"
          description: "Return to AI with instructions"
          requires_input: true
        - id: resolve
          label: "Resolve"
          description: "Mark as resolved"
          requires_input: true
      timeout:
        duration: 300
        action: auto_escalate
      context_for_human:
        - conversation_history
        - customer_account
        - agent_reasoning
        - routing_decisions
    connections:
      take_over: human_takeover_terminal
      send_back: orchestrator
      resolve: resolved_terminal
      timeout: timeout_terminal

  # Output guardrail
  - id: output_guard
    type: guardrail
    spec:
      position: output
      checks:
        - type: tone
          required: professional
          action: rewrite
        - type: no_hallucinated_urls
          action: block
        - type: schema_validation
          schema:
            type: object
            required: [message]
            properties:
              message: { type: string, max_length: 2000 }
    connections:
      pass: resolved_terminal
      block: fallback_terminal

  # Terminals
  - id: resolved_terminal
    type: terminal
    spec:
      status: 200
      body:
        message: "$.agent_response"
        conversation_id: "$.conversation_id"
        handled_by: "$.routing.agent_id"

  - id: blocked_response
    type: terminal
    spec:
      status: 400
      body: { error: "Message could not be processed", code: CONTENT_BLOCKED }

  - id: error_response
    type: terminal
    spec:
      status: 500
      body: { error: "An error occurred", code: AGENT_ERROR }

  - id: human_takeover_terminal
    type: terminal
    spec:
      status: 200
      body:
        message: "You've been connected with a human agent."
        conversation_id: "$.conversation_id"

  - id: timeout_terminal
    type: terminal
    spec:
      status: 504
      body: { error: "Request timed out", code: ESCALATION_TIMEOUT }

  - id: fallback_terminal
    type: terminal
    spec:
      status: 200
      body:
        message: "I apologize, let me connect you with a team member who can help."
        conversation_id: "$.conversation_id"

metadata:
  created_by: murat
  created_at: 2025-02-13
  completeness: 100
```

### Orchestration Patterns Summary

| Pattern | Nodes Used | Level 2 Visual | Description |
|---------|-----------|----------------|-------------|
| **Supervisor** | Orchestrator (⊛) + Agent Group ([⊛]) | Group boundary + supervisor arrows | One orchestrator monitors and manages multiple agents |
| **Router → Specialists** | Smart Router (◇◇+) → Agent flows | Router block → flow blocks | Classify then delegate to best agent |
| **Consult** | Handoff (⇄) in consult mode | Bidirectional dashed arrow | Agent A asks Agent B, gets answer back |
| **Transfer** | Handoff (⇄) in transfer mode | One-way dashed arrow | Agent A hands off completely to Agent B |
| **Collaborate** | Handoff (⇄) in collaborate mode + shared memory | Bidirectional arrow + shared memory badge | Both agents active with shared context |
| **Pipeline** | Event/sub-flow connections | Sequential flow blocks | Agent A → Agent B → Agent C |
| **Broadcast** | Orchestrator (strategy: broadcast) | Multiple supervisor arrows | Send to all, merge results |
| **Consensus** | Orchestrator (strategy: consensus) | Multiple supervisor arrows + merge indicator | All respond, supervisor picks best |
| **Fallback Chain** | Smart Router fallback_chain | Dashed arrows labeled "fallback" | Try A, if fails try B, then C |
| **A/B Testing** | Smart Router ab_testing | Split arrow with percentage labels | Route N% to experimental agent |
| **Human-in-the-Loop** | Human Gate (✋) | Gate block with approval options | Pause for human decision |
| **Circuit Breaker** | Smart Router policies.circuit_breaker | Status indicator on route arrows | Stop routing to failing agent |

### Agent & Orchestration Error Codes

```yaml
# Add to specs/shared/errors.yaml

agent:
  AGENT_ERROR:
    http_status: 500
    message_template: "Agent encountered an error"
    log_level: ERROR

  AGENT_MAX_ITERATIONS:
    http_status: 500
    message_template: "Agent did not converge within {max_iterations} iterations"
    log_level: WARNING

  AGENT_TOOL_ERROR:
    http_status: 500
    message_template: "Agent tool '{tool_name}' failed: {reason}"
    log_level: ERROR

  CONTENT_BLOCKED:
    http_status: 400
    message_template: "Content blocked by guardrail: {guardrail}"

  GUARDRAIL_FAILED:
    http_status: 422
    message_template: "Output did not pass guardrail check: {check}"

  ESCALATION_TIMEOUT:
    http_status: 504
    message_template: "Human escalation timed out after {timeout} seconds"

  ROUTING_FAILED:
    http_status: 500
    message_template: "Intent router could not classify request"
    log_level: WARNING

orchestration:
  ORCHESTRATOR_ERROR:
    http_status: 500
    message_template: "Orchestrator failed: {reason}"
    log_level: ERROR

  AGENT_REASSIGNED:
    http_status: 200
    message_template: "Request reassigned from {from_agent} to {to_agent}"
    log_level: INFO

  HANDOFF_FAILED:
    http_status: 500
    message_template: "Handoff to {target_agent} failed: {reason}"
    log_level: ERROR

  HANDOFF_TIMEOUT:
    http_status: 504
    message_template: "Handoff to {target_agent} timed out after {timeout}s"
    log_level: WARNING

  CIRCUIT_BREAKER_OPEN:
    http_status: 503
    message_template: "Route to {agent} is temporarily unavailable (circuit open)"
    log_level: WARNING

  ALL_ROUTES_EXHAUSTED:
    http_status: 503
    message_template: "All routes exhausted including fallback chain"
    log_level: ERROR

  CONTEXT_TRANSFER_FAILED:
    http_status: 500
    message_template: "Failed to transfer context to {target_agent}: {reason}"
    log_level: ERROR

  SUPERVISION_INTERVENTION:
    http_status: 200
    message_template: "Supervisor intervened: {intervention_reason}"
    log_level: INFO
```

---

# Part 5: DDD Format Specification v1.0

## Spec Hierarchy (Complete)

```
specs/
├── system.yaml              # Project identity, tech stack, zones, integrations, schedules, pipelines
├── system-layout.yaml       # System Map (L1) block positions (managed by DDD Tool)
├── architecture.yaml        # Project structure, infrastructure, cross-cutting concerns
├── config.yaml              # Environment variables, secrets schema
├── schemas/
│   ├── _base.yaml           # Base model (id, timestamps, soft delete)
│   ├── {model}.yaml         # Data model definitions (with optional transitions)
│   └── ...
├── shared/
│   ├── errors.yaml          # Error codes with HTTP status mappings (incl. agent errors)
│   ├── types.yaml           # Shared enums and value objects (optional)
│   └── layers.yaml          # Cross-cutting concern layers (retry, stealth, etc.) (optional)
└── domains/
    └── {domain}/
        ├── domain.yaml      # Domain config: flows, events, depends_on, groups, layout
        └── flows/
            ├── {flow}.yaml          # Traditional flow (type: traditional)
            └── {agent-flow}.yaml    # Agent flow (type: agent) — includes tools, memory, guardrails
```

> **Note:** `system.yaml`, `architecture.yaml`, `config.yaml`, `shared/errors.yaml`, `shared/types.yaml`, `shared/layers.yaml`, and schema files are supplementary spec files. They are NOT read by the DDD Tool UI directly — they are context files that `/ddd-implement` reads to generate correct implementation code.

## System Config

```yaml
# specs/system.yaml
name: obligo
version: 1.0.0
description: Cyber Liability Operating System

tech_stack:
  language: python
  language_version: "3.11"
  framework: fastapi
  orm: sqlalchemy
  database: postgresql
  cache: redis
  queue: rabbitmq

> **Note:** The `queue` field determines event infrastructure: when set (e.g., `BullMQ`, `rabbitmq`), `/ddd-scaffold` generates queue-based event infrastructure and `/ddd-implement` uses it for async events. When absent, use an in-process EventEmitter for domain events.

environments:
  - name: development
    url: http://localhost:8000
  - name: staging
    url: https://staging.obligo.io
  - name: production
    url: https://api.obligo.io

# Optional: Visual grouping of domains at L1
zones:
  - id: ingestion-pipeline
    name: Ingestion Pipeline
    domains: [ingestion, analysis]
  - id: user-facing
    name: User-Facing
    domains: [api, notification]

# Optional: External API integration configs
# service_call nodes can reference these by name
integrations:
  clm_provider_api:
    base_url: "https://api.clmprovider.com/v1"
    auth:
      method: api_key
      credentials_env: CLM_API_KEY
    rate_limits:
      requests_per_second: 10
    retry:
      max_attempts: 3
      backoff_ms: 1000
    timeout_ms: 30000
    used_by_domains: [ingestion]

# Optional: Inter-zone data flow arrows (shown at L1)
data_flows:
  - from: ingestion-pipeline
    to: user-facing
    label: "analysis.completed events"
    volume: high
  - from: user-facing
    to: ingestion-pipeline
    label: "Manual re-analysis requests"
    volume: low
    style: dashed

# Optional: Scheduling topology (surfaces cron schedules at L1)
schedules:
  - frequency: "*/30 * * * *"
    label: "Every 30 minutes"
    flows: [ingestion/scheduled-sync]
  - frequency: "0 2 * * *"
    label: "Daily at 2am"
    flows: [analysis/batch-reanalysis]

# Optional: System-level badges shown at L1
characteristics:
  - "Event-driven"
  - "2 cron schedules"
  - "1 external API"

# Optional: Cross-domain event chains
pipelines:
  - id: contract-processing
    name: Contract Processing Pipeline
    description: "End-to-end from ingestion to notification"
    steps:
      - { domain: ingestion, flow: webhook-ingestion, event_out: contract.ingested }
      - { domain: analysis, flow: extract-obligations, event_out: analysis.completed }
      - { domain: notification, flow: send-alert }
    spans_domains: [ingestion, analysis, notification]
```

> **Note:** The `integrations` section is optional. When present, `service_call` nodes can reference an integration by name, and `/ddd-implement` uses the integration's auth, retry, rate limit, and timeout config instead of repeating them per node.

### IntegrationConfig

| Field | Type | Description |
|-------|------|-------------|
| `base_url` | string | Base URL for the external API |
| `auth` | object | `{ method: 'api_key' \| 'oauth2' \| 'bearer', credentials_env: string }` |
| `rate_limits` | object? | `{ requests_per_second?, requests_per_window?, window_seconds? }` |
| `retry` | object? | `{ max_attempts?, backoff_ms?, strategy?: 'fixed' \| 'linear' \| 'exponential', jitter?: boolean }` |
| `timeout_ms` | number? | Request timeout in milliseconds |
| `headers` | `Record<string, string>?` | Default headers for all requests |
| `used_by_domains` | string[]? | Domains that use this integration (shown at L1) |

### SystemZone

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Zone ID |
| `name` | string | Display name |
| `domains` | string[] | Domain IDs in this zone |

### DataFlow

Inter-zone directed data flow arrows, shown at L1 to visualize high-level architectural data movement.

| Field | Type | Description |
|-------|------|-------------|
| `from` | string | Source zone or domain ID |
| `to` | string | Target zone or domain ID |
| `label` | string | Description of what flows |
| `volume` | string? | `'low' \| 'medium' \| 'high'` — visual thickness hint |
| `style` | string? | `'solid' \| 'dashed'` — visual distinction (default: solid) |

### Schedule

Scheduling topology — surfaces cron schedules at L1.

| Field | Type | Description |
|-------|------|-------------|
| `frequency` | string | Cron expression |
| `label` | string | Human-readable schedule description |
| `flows` | string[] | Flow references in `domain/flow-id` format |

### Pipeline

Cross-domain event chains that trace a complete pipeline across domains.

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Pipeline ID |
| `name` | string | Display name |
| `description` | string? | What this pipeline does end-to-end |
| `steps` | PipelineStep[] | Ordered steps across domains |
| `spans_domains` | string[] | All domains involved |

**PipelineStep fields:**

| Field | Type | Description |
|-------|------|-------------|
| `domain` | string | Domain ID |
| `flow` | string | Flow ID within the domain |
| `event_out` | string? | Event emitted to trigger the next step |

## Domain Config (Drives Level 1 & Level 2 Sheets)

Each domain has a `domain.yaml` that declares its flows, published events, and consumed events. This data drives the System Map (Level 1) and Domain Map (Level 2) visualizations automatically.

### Domain Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Display name |
| `description` | string? | What this domain handles |
| `flows` | DomainFlowEntry[] | List of flows in this domain |
| `publishes_events` | EventWiring[] | Events this domain emits |
| `consumes_events` | EventWiring[] | Events this domain listens to |
| `owns_schemas` | string[]? | Schema names this domain owns (e.g., ["User", "Session"]) |
| `depends_on` | DomainDependency[]? | Cross-domain data dependencies (beyond event wiring) |
| `groups` | FlowGroup[]? | Visual grouping of flows at L2 |
| `layout` | DomainLayout | Canvas positions (managed by DDD Tool) |

### DomainFlowEntry

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique flow ID (kebab-case) |
| `name` | string | Display name |
| `description` | string? | What this flow does |
| `type` | `'traditional' \| 'agent'` | Flow type |
| `tags` | string[]? | Custom tags (e.g., ["cron", "internal", "public-api"]) |
| `criticality` | string? | `'critical' \| 'high' \| 'normal' \| 'low'` — visual priority at L2 |
| `throughput` | string? | Expected volume (e.g., `"~500 items/day"`) — shown as subtitle at L2 |

### FlowGroup

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Group ID |
| `name` | string | Display name |
| `flows` | string[] | Flow IDs in this group |

### EventWiring

| Field | Type | Description |
|-------|------|-------------|
| `event` | string | Event name (e.g., UserRegistered) |
| `schema` | string? | Data schema reference |
| `from_flow` | string? | Which flow publishes it |
| `handled_by_flow` | string? | Which flow consumes it |
| `description` | string? | What this event means |
| `payload` | `Record<string, unknown>?` | Event data shape (field names and types) |

### DomainDependency

Cross-domain data dependencies (beyond event wiring). Shown as dashed arrows at L2.

| Field | Type | Description |
|-------|------|-------------|
| `domain` | string | Domain ID this domain depends on |
| `reason` | string | Why the dependency exists |
| `flows_affected` | string[]? | Which flows in this domain use the dependency |

```yaml
depends_on:
  - domain: settings
    reason: "Reads API keys and social account preferences"
    flows_affected: [generate-drafts, check-social-sources]
```

```yaml
# specs/domains/ingestion/domain.yaml
name: ingestion
description: Webhook and data ingestion from CLM providers

flows:
  - id: webhook-ingestion
    name: Webhook Ingestion
    description: Receives and processes webhooks from CLM providers
    tags: [webhook]
    criticality: critical
  - id: scheduled-sync
    name: Scheduled Sync
    description: Periodic sync with CLM provider APIs
    tags: [cron]
    throughput: "~200 syncs/day"

# Events this domain publishes (outgoing arrows on System Map)
publishes_events:
  - event: contract.ingested
    from_flow: webhook-ingestion
    description: Fired when a new contract is received via webhook
    payload:
      contract_id: uuid
      provider: string
      ingested_at: datetime
  - event: contract.synced
    from_flow: scheduled-sync
    description: Fired when a contract is synced from provider API
    payload:
      contract_id: uuid
      provider: string
      sync_type: string

# Events this domain consumes (incoming arrows on System Map)
consumes_events:
  - event: analysis.completed
    handled_by_flow: webhook-ingestion
    description: Notification that analysis of an ingested contract is done

# Optional: Cross-domain data dependencies
depends_on:
  - domain: api
    reason: "Reads provider credentials for API sync"
    flows_affected: [scheduled-sync]

# Layout positions for Domain Map (Level 2) — managed by DDD Tool
layout:
  flows:
    webhook-ingestion: { x: 100, y: 100 }
    scheduled-sync: { x: 400, y: 100 }
  portals:
    analysis: { x: 100, y: 300 }
```

```yaml
# specs/domains/analysis/domain.yaml
domain:
  name: analysis
  description: Contract analysis and obligation extraction

  flows:
    - id: extract-obligations
      name: Extract Obligations
      description: Extract obligations from contract using LLM
    - id: classify-risk
      name: Classify Risk
      description: Classify risk level of extracted obligations

  publishes_events:
    - event: analysis.completed
      from_flow: extract-obligations
      description: Fired when obligation extraction is complete
    - event: risk.classified
      from_flow: classify-risk
      description: Fired when risk classification is complete

  consumes_events:
    - event: contract.ingested
      handled_by_flow: extract-obligations
      description: Triggers obligation extraction for new contracts
    - event: contract.synced
      handled_by_flow: extract-obligations
      description: Triggers obligation extraction for synced contracts

  layout:
    flows:
      extract-obligations: { x: 100, y: 100 }
      classify-risk: { x: 400, y: 100 }
    portals:
      ingestion: { x: 100, y: -100 }
      notification: { x: 400, y: 300 }
```

### System Map Layout

The System Map (Level 1) also stores layout positions for domain blocks:

```yaml
# specs/system-layout.yaml (managed by DDD Tool)
system_layout:
  domains:
    ingestion: { x: 100, y: 200 }
    analysis: { x: 400, y: 200 }
    api: { x: 100, y: 500 }
    notification: { x: 400, y: 500 }
```

This file is auto-managed by the DDD Tool when users reposition domain blocks on the System Map canvas.

## Architecture Config (Critical for Code Generation)

```yaml
# specs/architecture.yaml

architecture:
  version: 1.0.0
  
  # ═══════════════════════════════════════════════════════════════════════════
  # PROJECT STRUCTURE
  # ═══════════════════════════════════════════════════════════════════════════
  
  structure:
    type: domain-driven  # domain-driven | feature-based | layered
    
    layout:
      src:
        domains:
          "{domain}":
            router: router.py
            schemas: schemas.py
            services: services.py
            models: models.py
            events: events.py
            exceptions: exceptions.py
        shared:
          auth: auth/
          database: database/
          events: events/
          middleware: middleware/
          exceptions: exceptions/
          utils: utils/
        config:
          settings: settings.py
          logging: logging.py
      tests:
        unit:
          domains: "{domain}/"
        integration: integration/
        e2e: e2e/
        fixtures: fixtures/
        factories: factories/
      migrations:
        versions: versions/
      scripts: scripts/
      
    naming:
      files: snake_case
      classes: PascalCase
      functions: snake_case
      variables: snake_case
      constants: SCREAMING_SNAKE_CASE
      database_tables: plural_snake_case
      database_columns: snake_case
      api_endpoints: kebab-case
      
  # ═══════════════════════════════════════════════════════════════════════════
  # DEPENDENCIES
  # ═══════════════════════════════════════════════════════════════════════════
  
  dependencies:
    production:
      # Framework
      fastapi: "^0.109.0"
      uvicorn: "^0.27.0"
      
      # Data validation
      pydantic: "^2.5.0"
      pydantic-settings: "^2.1.0"
      
      # Database
      sqlalchemy: "^2.0.25"
      alembic: "^1.13.0"
      asyncpg: "^0.29.0"
      
      # Cache & Queue
      redis: "^5.0.0"
      aio-pika: "^9.3.0"  # RabbitMQ
      
      # HTTP client
      httpx: "^0.26.0"
      
      # Auth
      python-jose: "^3.3.0"
      passlib: "^1.7.4"
      bcrypt: "^4.1.0"
      
      # Observability
      structlog: "^24.1.0"
      opentelemetry-api: "^1.22.0"
      
    development:
      pytest: "^7.4.0"
      pytest-asyncio: "^0.23.0"
      pytest-cov: "^4.1.0"
      factory-boy: "^3.3.0"
      fakeredis: "^2.20.0"
      ruff: "^0.1.0"
      mypy: "^1.8.0"
      pre-commit: "^3.6.0"
      
  # ═══════════════════════════════════════════════════════════════════════════
  # INFRASTRUCTURE
  # ═══════════════════════════════════════════════════════════════════════════
  
  infrastructure:
    database:
      type: postgresql
      async: true
      pool:
        min_size: 5
        max_size: 20
        max_overflow: 10
        pool_timeout: 30
      conventions:
        primary_key: uuid
        timestamps: true        # created_at, updated_at on all
        soft_delete: true       # deleted_at instead of DELETE
        audit_log: false        # Per-table override available
        
    cache:
      type: redis
      default_ttl: 3600
      key_prefix: "{system_name}:"
      serializer: json
      
    queue:
      type: rabbitmq
      exchange: events
      exchange_type: topic
      durable: true
      retry:
        max_attempts: 3
        backoff_type: exponential
        initial_delay: 1
        max_delay: 60
        dead_letter: true
        
    storage:
      type: s3
      provider: aws           # aws | gcs | minio | cloudflare
      
  # ═══════════════════════════════════════════════════════════════════════════
  # CROSS-CUTTING CONCERNS
  # ═══════════════════════════════════════════════════════════════════════════
  
  cross_cutting:
    
    authentication:
      type: jwt
      algorithm: RS256
      access_token:
        ttl: 900              # 15 minutes
        location: header
        header_name: Authorization
        scheme: Bearer
      refresh_token:
        ttl: 604800           # 7 days
        location: cookie
        cookie_name: refresh_token
        http_only: true
        secure: true
        same_site: lax
        
    authorization:
      type: rbac
      roles:
        - name: admin
          description: Full system access
        - name: owner
          description: Tenant owner
        - name: member
          description: Regular team member
        - name: viewer
          description: Read-only access
      default_role: member
      super_admin_bypass: true
      
    multi_tenancy:
      enabled: true
      strategy: row_level     # row_level | schema | database
      identifier: tenant_id
      header: X-Tenant-ID
      enforce_on_all_queries: true
      
    logging:
      format: structured
      output: json
      include:
        always:
          - timestamp
          - level
          - message
          - logger
          - request_id
          - tenant_id
        on_request:
          - method
          - path
          - status_code
          - duration_ms
          - user_id
        on_error:
          - error_type
          - error_code
          - stack_trace
      exclude_paths:
        - /health
        - /ready
        - /metrics
      levels:
        root: INFO
        sqlalchemy.engine: WARNING
        httpx: WARNING
        uvicorn.access: WARNING
        
    error_handling:
      response_format:
        error:
          type: string
          description: Human-readable error message
        code:
          type: string
          description: Machine-readable error code
        details:
          type: object
          description: Additional error context
        request_id:
          type: string
          description: Request ID for support
      error_codes:
        VALIDATION_ERROR:
          http_status: 422
          description: Request validation failed
        NOT_FOUND:
          http_status: 404
          description: Resource not found
        UNAUTHORIZED:
          http_status: 401
          description: Authentication required
        FORBIDDEN:
          http_status: 403
          description: Permission denied
        CONFLICT:
          http_status: 409
          description: Resource conflict
        RATE_LIMITED:
          http_status: 429
          description: Rate limit exceeded
        INTERNAL_ERROR:
          http_status: 500
          description: Internal server error
      include_stack_trace:
        development: true
        staging: true
        production: false
        
    rate_limiting:
      enabled: true
      storage: redis
      default:
        requests: 100
        window_seconds: 60
      by_tier:
        free:
          requests: 100
          window_seconds: 60
        pro:
          requests: 1000
          window_seconds: 60
        enterprise:
          requests: 10000
          window_seconds: 60
      headers:
        limit: X-RateLimit-Limit
        remaining: X-RateLimit-Remaining
        reset: X-RateLimit-Reset
        
    request_context:
      fields:
        - request_id          # Auto-generated UUID
        - tenant_id           # From header or JWT
        - user_id             # From JWT
        - roles               # From JWT
        - trace_id            # For distributed tracing
        
    middleware_order:
      # Order matters - first to last
      - cors
      - request_id
      - logging_start
      - error_handler
      - rate_limiter
      - authentication
      - tenant_context
      - authorization
      - logging_end
      
  # ═══════════════════════════════════════════════════════════════════════════
  # API DESIGN
  # ═══════════════════════════════════════════════════════════════════════════
  
  api:
    base_path: /api
    versioning:
      strategy: url_prefix    # url_prefix | header | query_param
      current_version: v1
      header_name: X-API-Version
      
    pagination:
      style: cursor           # cursor | offset
      default_limit: 20
      max_limit: 100
      params:
        limit: limit
        cursor: cursor
        offset: offset        # For offset style
      response:
        items: items
        total: total          # Only for offset style
        next_cursor: next_cursor
        prev_cursor: prev_cursor
        has_more: has_more
        
    filtering:
      style: query_params
      operators:
        eq: ""                # ?status=active
        ne: __ne              # ?status__ne=deleted
        gt: __gt              # ?amount__gt=100
        gte: __gte
        lt: __lt
        lte: __lte
        in: __in              # ?status__in=active,pending
        nin: __nin
        contains: __contains
        icontains: __icontains
        starts_with: __startswith
        is_null: __isnull     # ?deleted_at__isnull=true
        
    sorting:
      param: sort
      format: field:direction  # ?sort=created_at:desc
      multi_sort: true         # ?sort=status:asc,created_at:desc
      max_sort_fields: 3
      default: created_at:desc
      
    response_envelope:
      success_single:
        data: object
      success_list:
        data: array
        meta:
          pagination: object
          filters: object
      error:
        error: string
        code: string
        details: object
        request_id: string
        
    cors:
      allowed_origins:
        development: ["http://localhost:3000", "http://localhost:5173"]
        production: ["https://app.obligo.io"]
      allowed_methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]
      allowed_headers: ["*"]
      allow_credentials: true
      max_age: 600
      
  # ═══════════════════════════════════════════════════════════════════════════
  # TESTING
  # ═══════════════════════════════════════════════════════════════════════════
  
  testing:
    framework: pytest
    async_mode: auto
    
    coverage:
      minimum: 80
      fail_under: true
      exclude:
        - "*/migrations/*"
        - "*/config/*"
        - "*/__init__.py"
        
    database:
      strategy: transaction_rollback  # transaction_rollback | truncate | recreate
      use_test_database: true
      
    fixtures:
      auto_use:
        - db_session
        - test_client
        - authenticated_client
        
    factories:
      library: factory_boy
      base_class: AsyncFactory
      
    mocks:
      external_apis: true
      redis: fakeredis
      s3: moto
      time: freezegun
      
    markers:
      - unit: Unit tests (no IO)
      - integration: Integration tests (with DB)
      - e2e: End-to-end tests
      - slow: Slow tests (skip in CI fast mode)
      
  # ═══════════════════════════════════════════════════════════════════════════
  # DEPLOYMENT
  # ═══════════════════════════════════════════════════════════════════════════
  
  deployment:
    containerization:
      enabled: true
      runtime: docker
      base_image: python:3.11-slim
      multi_stage: true
      
    environments:
      development:
        debug: true
        log_level: DEBUG
        replicas: 1
      staging:
        debug: false
        log_level: INFO
        replicas: 2
      production:
        debug: false
        log_level: INFO
        replicas: 3
        auto_scaling:
          min: 3
          max: 10
          target_cpu: 70
          
    health_checks:
      liveness:
        path: /health
        interval: 10
        timeout: 5
        failure_threshold: 3
      readiness:
        path: /ready
        interval: 5
        timeout: 3
        failure_threshold: 3
        
    ci_cd:
      provider: github_actions
      branches:
        main: production
        staging: staging
        develop: development
      stages:
        - name: lint
          run: ruff check .
        - name: type_check
          run: mypy src/
        - name: test
          run: pytest --cov
        - name: build
          run: docker build
        - name: deploy
          run: kubectl apply
```

#### Cross-Cutting Patterns

The `cross_cutting_patterns` section captures project-specific implementation conventions discovered during Phase 3 (Build) and Phase 4 (Reflect). Each pattern describes a recurring approach that `/ddd-implement` applies automatically to matching flows.

```yaml
cross_cutting_patterns:
  stealth_http:
    description: >
      Rotate user agents, use proxy pools, maintain cookie jars per domain
    utility: src/utils/stealth-http.ts
    config:
      user_agent_pool: 50
      proxy_rotation: per_request
    used_by_domains: [monitoring, discovery]
    convention: >
      All service_call nodes fetching external content must use stealth HTTP
```

| Field | Type | Description |
|-------|------|-------------|
| `description` | string | What this pattern does and why |
| `utility` | string? | Path to the utility file implementing it |
| `config` | `Record<string, unknown>?` | Pattern-specific configuration |
| `used_by_domains` | string[] | Which domains use this pattern |
| `convention` | string | When and how to apply this pattern |

## Config Schema (Environment Variables)

```yaml
# specs/config.yaml

config:
  # Required environment variables
  required:
    DATABASE_URL:
      type: string
      format: "postgresql+asyncpg://{user}:{password}@{host}:{port}/{database}"
      description: PostgreSQL connection string
      example: "postgresql+asyncpg://user:pass@localhost:5432/obligo"
      
    REDIS_URL:
      type: string
      format: "redis://{host}:{port}/{db}"
      description: Redis connection string
      example: "redis://localhost:6379/0"
      
    RABBITMQ_URL:
      type: string
      format: "amqp://{user}:{password}@{host}:{port}/{vhost}"
      description: RabbitMQ connection string
      example: "amqp://guest:guest@localhost:5672/"
      
    JWT_PRIVATE_KEY:
      type: string
      sensitive: true
      description: RSA private key for signing JWTs (PEM format)
      
    JWT_PUBLIC_KEY:
      type: string
      sensitive: true
      description: RSA public key for verifying JWTs (PEM format)
      
  # Optional with defaults
  optional:
    ENVIRONMENT:
      type: string
      default: development
      enum: [development, staging, production]
      
    LOG_LEVEL:
      type: string
      default: INFO
      enum: [DEBUG, INFO, WARNING, ERROR, CRITICAL]
      
    LOG_FORMAT:
      type: string
      default: json
      enum: [json, text]
      
    PORT:
      type: integer
      default: 8000
      
    CORS_ORIGINS:
      type: array
      default: ["http://localhost:3000"]
      description: Allowed CORS origins (comma-separated in env)
      
    RATE_LIMIT_ENABLED:
      type: boolean
      default: true
      
    SENTRY_DSN:
      type: string
      sensitive: true
      description: Sentry DSN for error tracking (optional)
      
    AWS_ACCESS_KEY_ID:
      type: string
      sensitive: true
      description: AWS access key for S3
      
    AWS_SECRET_ACCESS_KEY:
      type: string
      sensitive: true
      description: AWS secret key for S3
      
    S3_BUCKET:
      type: string
      default: obligo-uploads
```

## Base Model Spec

```yaml
# specs/schemas/_base.yaml

base_model:
  name: BaseModel
  description: Base model inherited by all database models
  
  fields:
    id:
      type: uuid
      primary_key: true
      default: uuid4
      description: Unique identifier
      
    created_at:
      type: datetime
      default: now
      nullable: false
      index: true
      description: Record creation timestamp
      
    updated_at:
      type: datetime
      default: now
      on_update: now
      nullable: false
      description: Last update timestamp
      
    deleted_at:
      type: datetime
      nullable: true
      index: true
      description: Soft delete timestamp (null = not deleted)
      
  behaviors:
    soft_delete:
      enabled: true
      field: deleted_at
      
    timestamps:
      created_field: created_at
      updated_field: updated_at
      
    default_query_filter: "deleted_at IS NULL"
    
  mixins:
    - TimestampMixin
    - SoftDeleteMixin
```

## Shared Error Codes

```yaml
# specs/shared/errors.yaml

errors:
  # Validation errors (422)
  VALIDATION_ERROR:
    http_status: 422
    message_template: "Validation failed"
    
  INVALID_INPUT:
    http_status: 422
    message_template: "Invalid input: {details}"
    
  MISSING_FIELD:
    http_status: 422
    message_template: "Missing required field: {field}"
    
  INVALID_FORMAT:
    http_status: 422
    message_template: "Invalid format for {field}: {reason}"
    
  # Authentication errors (401)
  UNAUTHORIZED:
    http_status: 401
    message_template: "Authentication required"
    
  INVALID_TOKEN:
    http_status: 401
    message_template: "Invalid or expired token"
    
  TOKEN_EXPIRED:
    http_status: 401
    message_template: "Token has expired"
    
  # Authorization errors (403)
  FORBIDDEN:
    http_status: 403
    message_template: "You don't have permission to perform this action"
    
  INSUFFICIENT_ROLE:
    http_status: 403
    message_template: "Required role: {required_role}"
    
  TENANT_MISMATCH:
    http_status: 403
    message_template: "Access denied to this tenant's resources"
    
  # Not found errors (404)
  NOT_FOUND:
    http_status: 404
    message_template: "Resource not found"
    
  ENTITY_NOT_FOUND:
    http_status: 404
    message_template: "{entity} with id {id} not found"
    
  # Conflict errors (409)
  CONFLICT:
    http_status: 409
    message_template: "Resource conflict"
    
  DUPLICATE_ENTRY:
    http_status: 409
    message_template: "{entity} with {field}={value} already exists"
    
  # Rate limiting (429)
  RATE_LIMITED:
    http_status: 429
    message_template: "Rate limit exceeded. Retry after {retry_after} seconds"
    headers:
      Retry-After: "{retry_after}"
      
  # Server errors (500)
  INTERNAL_ERROR:
    http_status: 500
    message_template: "An unexpected error occurred"
    log_level: ERROR
    
  DATABASE_ERROR:
    http_status: 500
    message_template: "Database operation failed"
    log_level: ERROR
    
  EXTERNAL_SERVICE_ERROR:
    http_status: 502
    message_template: "External service unavailable: {service}"
    log_level: WARNING
```

## Flow Spec

### FlowDocument Structure

| Field | Type | Description |
|-------|------|-------------|
| `flow` | object | Flow metadata |
| `flow.id` | string | Unique flow ID |
| `flow.name` | string | Display name |
| `flow.type` | `'traditional' \| 'agent'` | Flow category |
| `flow.domain` | string | Parent domain ID |
| `flow.description` | string? | What this flow does |
| `flow.template` | boolean? | When `true`, this flow is a template (see Parameterized Flow pattern) |
| `flow.parameters` | `Record<string, FlowParameter>`? | Template parameters (required when `template: true`) |
| `flow.contract` | object? | Sub-flow input/output contract (see below) |
| `trigger` | DddFlowNode | The entry point node |
| `nodes` | DddFlowNode[] | All other nodes |
| `metadata` | object | `{ created, modified }` ISO timestamps |

### DddFlowNode (Common Fields)

Every node has:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique node ID |
| `type` | DddNodeType | One of the 27 node types |
| `position` | `{ x, y }` | Canvas position |
| `connections` | Array | List of `{ targetNodeId, sourceHandle?, targetHandle?, data?, behavior? }` |
| `spec` | object | Type-specific configuration |
| `label` | string | Display name on canvas |
| `observability` | object? | Logging/metrics/tracing config (see Cross-Cutting Concerns) |
| `security` | object? | Auth/rate-limiting/encryption config (see Cross-Cutting Concerns) |

### Connection Data Annotations

Connections can optionally include a `data` field to document what data flows between nodes. This is purely descriptive — it doesn't affect behavior but enables data shape hover in the DDD Tool.

```yaml
connections:
  - targetNodeId: data_store-abc123
    sourceHandle: valid
    data:
      - { name: email, type: string }
      - { name: password, type: string }
```

### Connection Behavior

Connections can optionally include a `behavior` field to distinguish error handling strategies:

| Value | Description |
|-------|-------------|
| `continue` | Log and continue (soft fail) |
| `stop` | Stop the flow (hard fail) |
| `retry` | Retry before failing (retry count configured at node level) |
| `circuit_break` | Use circuit breaker pattern |

```yaml
connections:
  - targetNodeId: terminal-error-001
    sourceHandle: error
    behavior: continue
    label: "Log and skip"
```

### Connection Format Compatibility

The DDD Tool normalizes several alternative field names on import:

| You write | Normalized to |
|-----------|---------------|
| `target`, `targetId`, or `targetNodeId` | `targetNodeId` |
| `sourceHandle: "default"` | unnamed handle (omitted) |
| `spec`, `properties`, or `config` | `spec` |
| `label` or `name` | `label` |

### Variable Scope and Data Flow

DDD specs use `$.variable_name` syntax to reference data flowing through a flow:

- `$` represents the flow context — a mutable object created when the trigger fires and passed through every node.
- Each node reads inputs from `$` and writes its output back to `$` under a named key.
- Scope is **flow-scoped** and **accumulative**: later nodes can read anything written by earlier nodes.

**Naming conventions:**
- `$.raw_*` — unprocessed content (e.g., `$.raw_feed`, `$.raw_html`)
- `$.model_name` — data model instances (e.g., `$.order`, `$.user`)
- `$.model_name_list` — arrays of models (e.g., `$.order_list`)
- `$.is_*` or `$.has_*` — boolean flags for decision nodes

**Code generation rule:** When generating code, replace each `$.X` reference with the corresponding local variable or parameter. The `$` object is a design-time abstraction — it does not need to exist as a literal runtime object.

### FlowParameter (for parameterized flows)

| Field | Type | Description |
|-------|------|-------------|
| `type` | string | Parameter type (`'string'`, `'integration_ref'`, `'number'`, `'boolean'`) |
| `values` | array? | Allowed values (for string type) |

### Sub-Flow Contract

Flows called as sub-flows can define a `contract` in their flow metadata:

```yaml
flow:
  id: normalize-content
  name: Normalize Content
  type: traditional
  domain: ingestion
  contract:
    inputs:
      - name: raw_content
        type: string
        required: true
      - name: platform
        type: enum
        ref: platform
        required: true
    outputs:
      - name: content_item_id
        type: uuid
      - name: is_duplicate
        type: boolean
```

| Contract Field | Type | Description |
|----------------|------|-------------|
| `inputs` | `Array<{ name, type, required?, ref? }>` | Expected input parameters |
| `outputs` | `Array<{ name, type }>` | Returned output values |

### Flow Spec Example

```yaml
# specs/domains/ingestion/flows/webhook-ingestion.yaml
flow:
  id: webhook-ingestion
  name: Webhook Ingestion
  type: traditional
  domain: ingestion
  description: Receives and processes webhooks from CLM providers

trigger:
  id: trigger-001
  type: trigger
  position: { x: 100, y: 50 }
  connections:
    - targetNodeId: validate_signature
  spec:
    event: "webhook /webhooks/{provider}"
    source: CLM Providers
    description: Webhook callback from CLM provider
  
nodes:
  - id: validate_signature
    type: decision
    position: {x: 100, y: 50}
    spec:
      check: webhook_signature
      algorithm: hmac-sha256
      header: X-Webhook-Signature
      on_failure:
        status: 401
        body:
          error: "Signature validation failed"
    connections:
      valid: validate_payload
      invalid: reject_unauthorized

  - id: validate_payload
    type: input
    position: {x: 200, y: 50}
    spec:
      fields:
        contract_id:
          type: string
          required: true
          format: uuid
          error: "contract_id must be a valid UUID"
        document_url:
          type: string
          required: true
          format: url
          error: "document_url must be a valid URL"
        event_type:
          type: string
          enum: [created, updated, signed, executed]
          error: "Invalid event type"
    connections:
      valid: store_contract
      invalid: return_validation_error

  - id: store_contract
    type: data_store
    position: {x: 300, y: 50}
    spec:
      operation: upsert
      model: Contract
      data:
        id: "$.contract_id"
        document_url: "$.document_url"
        status: "$.event_type"
        provider: "$.path.provider"
    connections:
      success: publish_event
      failure: return_error

  - id: publish_event
    type: event
    position: {x: 400, y: 50}
    spec:
      event: contract.ingested
      payload:
        contract_id: "$.contract_id"
        provider: "$.path.provider"
    connections:
      done: return_success

  - id: return_success
    type: terminal
    position: {x: 500, y: 50}
    spec:
      status: 202
      body:
        message: "Webhook processed"
        contract_id: "$.contract_id"

metadata:
  created_by: murat
  created_at: 2025-02-04
  completeness: 100
```

## Schema Spec

```yaml
# specs/schemas/contract.yaml
schema:
  name: Contract
  version: 1.0.0
  description: Represents a contract document from CLM
  
  fields:
    id:
      type: uuid
      primary_key: true
    document_url:
      type: string
      format: url
      max_length: 2048
    status:
      type: string
      enum: [created, updated, signed, executed]
    provider:
      type: string
      enum: [ironclad, icertis, docusign]
    created_at:
      type: datetime
      default: now
    updated_at:
      type: datetime
      auto_update: true

  indexes:
    - fields: [provider, status]
    - fields: [created_at]

  # Optional: State machine transitions for lifecycle fields
  transitions:
    field: status
    states:
      - from: created
        to: [updated, signed]
      - from: updated
        to: [signed]
      - from: signed
        to: [executed]
    on_invalid: reject   # reject | warn | log

  used_by:
    - webhook-ingestion
    - extract-obligations
```

> **Note:** The `transitions` section is optional. When present, it defines valid state machine transitions for enum fields with lifecycle semantics. `/ddd-implement` generates validation logic that enforces valid state changes.

> **Note:** Schemas can reference shared enums from `specs/shared/types.yaml` using `type: enum, ref: {enum_name}` instead of duplicating `enum:` arrays.

## Shared Types

**Path:** `specs/shared/types.yaml`

Shared enums and value objects used across multiple schemas:

```yaml
enums:
  platform:
    values: [twitter, linkedin, medium, rss, web]
    description: Supported content platforms

  content_status:
    values: [pending, analyzed, relevant, irrelevant, error]
    description: Content processing lifecycle

value_objects:
  money:
    fields:
      - name: amount
        type: decimal
      - name: currency
        type: string
    description: Monetary amount with currency
```

### Enum Definition

| Field | Type | Description |
|-------|------|-------------|
| `values` | string[] | Allowed values |
| `description` | string? | What this enum represents |

### Value Object Definition

| Field | Type | Description |
|-------|------|-------------|
| `fields` | `Array<{ name, type }>` | Component fields |
| `description` | string? | What this value object represents |

> **Note:** Reference shared enums in schemas with `type: enum, ref: platform` instead of duplicating `values:` arrays. `/ddd-implement` reads this file to generate shared type definitions.

## Shared Layers

**Path:** `specs/shared/layers.yaml`

Cross-cutting concern layers that span multiple nodes and domains. Layers define infrastructure behavior (retry policies, stealth config, circuit breakers) that applies to specific node types across specific domains.

```yaml
layers:
  - id: stealth
    name: Stealth / Anti-Detection
    description: "Rotating user agents, proxy pools, cookie jars"
    applies_to:
      nodes: [service_call, parse]
      domains: [monitoring, discovery]
    config:
      user_agent_pool: 50
      proxy_rotation: per_request
      cookie_jar: per_domain

  - id: retry-policy
    name: Retry Policy
    description: "Exponential backoff with circuit breaker"
    applies_to:
      nodes: [service_call, batch]
      domains: [monitoring, discovery, publishing]
    config:
      max_retries: 3
      backoff: exponential
      circuit_breaker:
        failure_threshold: 5
        reset_timeout_ms: 60000
```

### Layer

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique layer ID |
| `name` | string | Display name |
| `description` | string? | What this layer does |
| `applies_to` | object | `{ nodes: string[], domains: string[] }` — which node types and domains this layer affects |
| `config` | `Record<string, unknown>` | Layer-specific configuration |

> **Note:** Layers don't change node specs — they're read by `/ddd-implement` to generate infrastructure code (retry wrappers, proxy config, rate limiters). In the DDD Tool, affected nodes show a layer badge.

---

# Part 6: DDD Slash Commands (Claude Code Integration)

DDD uses Claude Code slash commands as the implementation interface. These commands are invoked in a Claude Code terminal session and operate on the spec files.

## Commands

### /ddd-create

Generates a complete DDD project from a natural-language description.

```
/ddd-create {description}
```

**Example:**
```
/ddd-create An e-commerce platform with user auth, product catalog,
            shopping cart, order processing, and email notifications
```

**What it does:**
1. Fetches the latest DDD Usage Guide for spec format reference
2. Analyzes the description (asks clarifying questions if brief)
3. Creates the full project structure: `ddd-project.json`, supplementary specs, schemas, domain YAML, and flow YAML
4. Validates all flows (trigger → terminals, wired branches, event matching)
5. Shows summary with domain counts, file list, event wiring, and next steps

### /ddd-scaffold

Sets up the project skeleton and shared infrastructure from specs. First step of Phase 3 (Build).

```
/ddd-scaffold
```

**What it does:**
1. Reads system.yaml, architecture.yaml, config.yaml, errors.yaml, types.yaml, and all schema files
2. Initializes the project (package.json, tsconfig, dependencies, directory structure)
3. Generates shared infrastructure: config loader, error handler, database schema, app entry point, integration clients, event bus, test setup
4. Creates environment files (.env.example, .gitignore)
5. Verifies build compiles and example test passes
6. Initializes `.ddd/mapping.yaml`

### /ddd-implement

Generates implementation code from DDD specs.

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | Entire project | `/ddd-implement --all` |
| `{domain}` | All flows in a domain | `/ddd-implement users` |
| `{domain}/{flow}` | Single flow | `/ddd-implement users/user-register` |
| *(empty)* | Interactive mode | `/ddd-implement` |

**What it does:**
1. Reads `ddd-project.json` and flow YAML specs
2. Reads supplementary specs for implementation context
3. Checks `.ddd/mapping.yaml` for existing implementations
4. Follows the node graph: trigger → nodes → terminals
5. Generates code for each node (route handlers, services, DB queries, etc.)
6. Generates tests (happy path, decision branches, error states, input validation)
7. Runs tests and fixes until passing
8. Updates `.ddd/mapping.yaml` with specHash and file list

### /ddd-update

Updates DDD project specs (YAML files) to reflect design changes.

| Argument | Scope | Example |
|----------|-------|---------|
| `{domain}/{flow}` | Update a specific flow spec | `/ddd-update users/user-register` |
| `{domain}` | Update domain config and/or its flows | `/ddd-update users` |
| `--add-flow {domain}` | Add a new flow to a domain | `/ddd-update --add-flow users` |
| `--add-domain` | Add a new domain to the project | `/ddd-update --add-domain` |
| *(empty)* | Interactive mode | `/ddd-update` |

### /ddd-sync

Synchronizes specs with implementation state.

| Argument | What it does |
|----------|-------------|
| *(default)* | Sync mapping.yaml with current implementation state |
| `--discover` | Also discover untracked code and suggest new flow specs |
| `--fix-drift` | Re-implement flows where specs have drifted |
| `--full` | All of the above |

### /ddd-status

Quick read-only overview of the project's implementation state.

```
/ddd-status [--json]
```

### /ddd-test

Run tests for implemented flows without re-generating code.

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | All implemented flows | `/ddd-test --all` |
| `{domain}` | All flows in a domain | `/ddd-test users` |
| `{domain}/{flow}` | Single flow | `/ddd-test users/user-register` |

### /ddd-reverse

Reverse-engineers DDD specs from an existing codebase.

| Argument | Scope | Example |
|----------|-------|---------|
| `--output <path>` | Where to write specs | `/ddd-reverse --output ./specs` |
| `--domains <d1,d2>` | Only reverse specific domains | `/ddd-reverse --domains users,orders` |
| `--merge` | Merge with existing specs | `/ddd-reverse --merge` |
| `--strategy <name>` | Override auto-selected strategy | `/ddd-reverse --strategy compiler` |
| *(empty)* | Interactive mode (auto-selects strategy by file count) | `/ddd-reverse` |

**Strategies** (auto-selected by source file count): `baseline` (<30), `index` (30-80), `swap` (80-150), `bottom-up` (150-300), `compiler` (300-500), `codex` (500+)

### /ddd-evolve

Analyzes shortfall reports and applies framework improvements.

```
/ddd-evolve [--dir {path}]
```

### /ddd-reflect

Captures implementation wisdom by comparing code against specs. Writes annotation files to `.ddd/annotations/{domain}/{flow}.yaml`.

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | All implemented flows | `/ddd-reflect --all` |
| `{domain}` | All flows in a domain | `/ddd-reflect users` |
| `{domain}/{flow}` | Single flow | `/ddd-reflect users/user-register` |
| *(empty)* | Interactive mode | `/ddd-reflect` |

### /ddd-promote

Moves approved annotations into permanent specs (architecture.yaml cross_cutting_patterns, flow specs, or shared types/errors).

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | Promote all approved annotations | `/ddd-promote --all` |
| `--review` | Interactive review of candidates | `/ddd-promote --review` |
| `{domain}/{flow}` | Scope to a specific flow | `/ddd-promote users/user-register` |

## Workflow

**Four-Phase Lifecycle:**

1. **Phase 1 — Create** — Run `/ddd-create` with a project description, or `/ddd-reverse` for existing codebases
2. **Phase 2 — Design** — Open the project in DDD Tool to visualize, validate, and refine specs on the canvas
3. **Phase 3 — Build** — Run `/ddd-scaffold` → `/ddd-implement --all` → `/ddd-test --all`
4. **Phase 4 — Reflect** — Run `/ddd-sync` to check alignment, `/ddd-reflect` to capture implementation wisdom, `/ddd-promote` to move approved patterns into specs
5. **Iterate** — Use `/ddd-status` to check state, `/ddd-update` to modify specs, `/ddd-implement` to update code

> Legacy docs may reference "Session A" (= Phase 1+2) and "Session B" (= Phase 3+4).

---

# Part 7: DDD Tool (Desktop App)

## Tech Stack

- **Framework:** Tauri (Rust backend, React frontend)
- **Canvas:** tldraw SDK or React Flow
- **State:** Zustand
- **UI:** Tailwind + Radix
- **Git:** libgit2 (via git2 crate)

## Key Components

### Multi-Level Canvas (see Part 4: Multi-Level Canvas Architecture)
- **3-level hierarchy:** System Map → Domain Map → Flow Sheet
- **Breadcrumb navigation** at top of canvas
- **Double-click** to drill into domains, flows, sub-flows
- **Portal nodes** for cross-domain navigation
- Levels 1-2 are auto-derived from specs; Level 3 is the editing surface
- Zoom/pan navigation on all levels
- Undo/redo history
- Keyboard shortcuts (Backspace/Esc to navigate up)

### Visual Editor (Level 3 — Flow Sheet)
- Canvas with drag-drop nodes
- 19 node types (trigger, input, process, decision, service call, data store, event, terminal, loop, parallel, sub-flow, delay, cache, transform, collection, parse, crypto, batch, transaction)
- Connection drawing between nodes
- Sub-flow nodes are navigable links to other flow sheets

### System Map (Level 1)
- Auto-generated from `system.yaml` + `domain.yaml` files
- Domain blocks with flow count badges
- Event arrows between domains (from `publishes_events`/`consumes_events`)
- Repositionable blocks (positions saved to `system-layout.yaml`)

**Domain management (right-click context menu on L1):**

| Action | What happens |
|--------|-------------|
| **Add domain** | Right-click canvas → "Add domain" → enter name + description → creates `specs/domains/{name}/domain.yaml` + `flows/` directory, updates `system.yaml`, adds block to canvas |
| **Rename domain** | Right-click domain block → "Rename" → enter new name → renames `specs/domains/{old}/` to `specs/domains/{new}/`, updates `system.yaml`, updates all cross-domain references (events, portals, orchestration agent refs) |
| **Delete domain** | Right-click domain block → "Delete" → confirmation dialog ("This will delete {n} flows. Are you sure?") → removes `specs/domains/{name}/` directory, updates `system.yaml`, removes event/portal references from other domains |
| **Edit description** | Right-click domain block → "Edit description" → inline edit → updates `domain.yaml` |
| **Add event** | Right-click domain block → "Add published event" / "Add consumed event" → enter event name + payload → updates `domain.yaml`, draws new event arrow on canvas |

### Domain Map (Level 2)
- Auto-generated from `domain.yaml` + flow files
- Flow blocks within a domain
- Inter-flow event connections
- Portal nodes linking to other domains
- Repositionable blocks (positions saved to `domain.yaml` layout section)

**Flow management (right-click context menu on L2):**

| Action | What happens |
|--------|-------------|
| **Add flow** | Right-click canvas → "Add flow" → enter name, select type (traditional/agent/orchestration) → creates `specs/domains/{domain}/flows/{name}.yaml` with trigger + terminal, adds block to canvas, opens L3 |
| **Rename flow** | Right-click flow block → "Rename" → enter new name → renames flow file, updates `flow.id`, updates all cross-references (sub-flow nodes, orchestration agent refs, portal references, mapping.yaml) |
| **Delete flow** | Right-click flow block → "Delete" → confirmation dialog → removes flow YAML file, removes from mapping.yaml, removes portal/event references |
| **Duplicate flow** | Right-click flow block → "Duplicate" → creates copy with `{name}-copy` → new flow file with all nodes/connections copied |
| **Move to domain** | Right-click flow block → "Move to..." → select target domain → moves flow file to target domain's `flows/` directory, updates all references |
| **Change flow type** | Right-click flow block → "Change type" → traditional/agent/orchestration → warns if nodes will be lost ("Agent nodes will be removed. Continue?"), restructures flow |

### Flow Sheet (Level 3)

**Flow-level actions (toolbar or right-click canvas background):**

| Action | What happens |
|--------|-------------|
| **Rename flow** | Click flow name in toolbar → inline edit → updates flow.id and filename |
| **Change trigger type** | Click trigger node → spec panel → change type (http/event/scheduled/manual) |
| **Delete all nodes** | Right-click canvas → "Clear canvas" → confirmation → removes all nodes except trigger |
| **Import from template** | Right-click canvas → "Import template" → select a template flow → merges nodes into current canvas |

### Flow Templates

The DDD Tool includes pre-built flow templates that create fully wired node graphs with one click. Templates are available when creating a new flow or importing from the canvas context menu.

**Traditional Flow Templates:**

| Template | Nodes | Description |
|----------|-------|-------------|
| REST API Endpoint | 5 | Trigger → Input → Process → Terminal (success) + Terminal (error) |
| CRUD Entity | 6 | Trigger → Input → Decision → Data Store → Terminal |
| Webhook Handler | 5 | Trigger → Input → Process → Service Call → Terminal |
| Event Processor | 5 | Trigger → Event (consume) → Process → Event (emit) → Terminal |

**Agent Flow Templates:**

| Template | Nodes | Description |
|----------|-------|-------------|
| RAG Agent | 5 | Guardrail (input) → Agent Loop (retrieval + answer tools) → Guardrail (output) → Terminal |
| Customer Support Agent | 5 | Guardrail → Agent Loop (ticket tools) → Human Gate → Terminal |
| Code Review Agent | 3 | Trigger → Agent Loop (code analysis + diff tools) → Terminal |
| Data Pipeline Agent | 3 | Trigger → Agent Loop (ETL tools) → Terminal |

Each template creates a complete `FlowDocument` with trigger, nodes, connections, and spec defaults pre-wired using the `{type}-{nanoid(8)}` ID convention.

### Spec Panel
- Right sidebar with type-specific fields
- Dropdown menus (not free text)
- Preset templates (username, email, password)
- Contextual tooltips
- Cmd+K command palette
- Context-aware: shows domain info on Level 2, flow spec on Level 3
- **Custom Fields:** All node specs are extensible — beyond the typed fields (e.g. event, source, description for Trigger), users can add arbitrary key-value custom fields. These appear in a collapsible "Custom Fields" section below the typed fields. AI suggestions that include non-standard fields (e.g. ip_address, session_id, payload_schema) are preserved when applied and visible/editable in this section. Custom fields are persisted to YAML alongside typed fields.

### Copy to Clipboard

All read-only output areas across the tool include a **Copy** button for one-click clipboard copying. This is implemented via a shared `CopyButton` component (`src/components/shared/CopyButton.tsx`) that uses the browser Clipboard API.

**Integration points:**

| Panel | What's copied |
|-------|--------------|
| **TerminalOutput** (Implementation) | Running/completed Claude Code output (ANSI-stripped) |
| **DoneView** (Implementation) | Completed implementation output |
| **FailedView** (Implementation) | Failed implementation error output |
| **PromptPreview** (Implementation) | Generated implementation prompt text |
| **TestResults** (Implementation) | Individual test error messages |

**Behavior:**
- Small "Copy" label with copy icon, positioned in the header or top-right corner of each output area
- On click: copies full text content to system clipboard
- Visual feedback: label changes to "Copied" with a checkmark for 2 seconds
- Non-intrusive: uses muted text color, only highlights on hover

### Git Panel
- Branch selector
- Staged/unstaged changes
- Commit box
- Pull/push buttons
- History view

#### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Cmd+Shift+M` / `Ctrl+Shift+M` | Toggle Minimap |
| `Cmd+Z` / `Ctrl+Z` | Undo |
| `Cmd+Shift+Z` / `Ctrl+Shift+Z` | Redo |

### App Shell & User Experience

The DDD Tool needs a cohesive app shell that handles project management, settings, onboarding, error recovery, and undo/redo. These are the "glue" between features.

#### 1. Project Management (Create / Open / Recent)

The app launches to a **Project Launcher** screen — not directly into a canvas. This is the entry point for all sessions.

**Project Launcher screen:**

```
┌──────────────────────────────────────────────────────────────┐
│                                                                │
│                        DDD Tool                               │
│                   Design Driven Development                  │
│                                                                │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │  Recent Projects                                         │  │
│  │                                                           │  │
│  │  📁 Obligo         ~/code/obligo        2 min ago       │  │
│  │  📁 SaaS Starter   ~/code/saas-kit      3 days ago     │  │
│  │  📁 Invoice App    ~/code/invoicely     1 week ago     │  │
│  │                                                           │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  [+ New Project]    [Open Existing]    [Import from Git]     │
│                                                                │
│                                          ⚙ Settings          │
└──────────────────────────────────────────────────────────────┘
```

**"New Project" wizard (3 steps):**

```
Step 1: Basics
┌──────────────────────────────────────────────────────────────┐
│  Project Name:    [my-project          ]                     │
│  Location:        [~/code/my-project   ] [Browse]           │
│  Description:     [My awesome project  ]                     │
│                                                                │
│  Initialize Git:  [✓]  (checked by default)                 │
│                                         [Cancel] [Next →]   │
└──────────────────────────────────────────────────────────────┘

Step 2: Tech Stack
┌──────────────────────────────────────────────────────────────┐
│  Language:    [Python ▾]     Version:  [3.11 ▾]             │
│  Framework:   [FastAPI ▾]                                    │
│  Database:    [PostgreSQL ▾]                                 │
│  ORM:         [SQLAlchemy ▾]                                 │
│  Cache:       [Redis ▾]  (optional)                          │
│                                                                │
│  Or: [Import from existing architecture.yaml]                │
│                                         [← Back] [Next →]   │
└──────────────────────────────────────────────────────────────┘

Step 3: Domains
┌──────────────────────────────────────────────────────────────┐
│  Define your initial domains:                                 │
│                                                                │
│  [users        ] User management                    [✕]     │
│  [billing      ] Subscription and payments          [✕]     │
│  [+ Add domain]                                              │
│                                                                │
│  Or: [Start blank — add domains later]                       │
│                                         [← Back] [Create]   │
└──────────────────────────────────────────────────────────────┘
```

**What "Create" does:**
1. Creates the project directory
2. Initializes Git (if checked)
3. Creates `specs/system.yaml` from wizard inputs
4. Creates `specs/architecture.yaml` from template + tech stack choices
5. Creates `specs/config.yaml` from template
6. Creates `specs/shared/errors.yaml` from template
7. Creates `specs/domains/{name}/domain.yaml` for each domain
8. Creates `.ddd/config.yaml` with defaults
9. Creates `.ddd/mapping.yaml` (empty)
10. Generates `CLAUDE.md` from project state
11. Makes initial Git commit: "Initialize DDD project"
12. Opens the project on the System Map (Level 1)

**"Open Existing" flow:**
- Standard OS file picker → select a folder
- DDD Tool looks for `specs/system.yaml` or `.ddd/config.yaml`
- If found: opens project, parses all specs, populates canvas
- If not found: shows error "This folder doesn't appear to be a DDD project. [Initialize as DDD project?]"

**"Import from Git" flow:**
- User enters a Git URL → DDD Tool clones into selected location
- Optional Personal Access Token field for private repositories (injected into HTTPS URL as `https://<token>@host/repo`)
- Uses system `git` binary (inherits macOS Keychain, SSH agent, credential helpers)
- Same detection as "Open Existing" after clone
- If `ddd-project.json` not found: auto-creates one with empty domains (allows opening any cloned project)

**Recent Projects:**
- Stored in app-level config: `~/.ddd-tool/recent-projects.json`
- Tracks: path, name, last opened timestamp
- Max 20 entries, pruned on load (remove entries where folder no longer exists)
- Click to open, right-click for "Remove from recent" or "Open in terminal"

#### 2. Settings Screen

Accessible from Project Launcher (⚙) and from within any project (menu bar → Settings or `Cmd+,`).

**Two levels: Global settings (apply to all projects) and Project settings (per-project overrides).**

```
┌──────────────────────────────────────────────────────────────┐
│ Settings                                     [Global ▾]      │
│──────────────────────────────────────────────────────────────│
│                                                                │
│ ┌──────────┐                                                  │
│ │ Editor   │  Editor                                          │
│ │ Claude   │                                                  │
│ │ Testing  │  ┌─────────────────────────────────────────┐    │
│ │ Git      │  │ Theme:     [Dark ▾]                      │    │
│ │          │  │ Font Size: [14   ]                        │    │
│ │          │  │ Auto-save: [30   ] seconds (0=disable)   │    │
│ │          │  │ [✓] Snap nodes to grid                   │    │
│ │          │  └─────────────────────────────────────────┘    │
│ └──────────┘                                                  │
│                                           [Cancel] [Save]    │
└──────────────────────────────────────────────────────────────┘
```

**Settings tabs:**

| Tab | What it configures |
|-----|-------------------|
| **Editor** | Canvas grid snap, auto-save interval, theme (light/dark), font size |
| **Claude Code** | CLI command path, post-implementation actions (run tests, run lint, auto-commit), prompt options |
| **Testing** | Test command, args, scoped test pattern, auto-run toggle |
| **Git** | Auto-commit messages, branch naming, commit signing |

**Project-level overrides:**
- Toggle dropdown: "Global" ↔ "Project: {name}"
- When on "Project," settings are saved to `.ddd/config.yaml` and override global

**Global settings location:** `~/.ddd-tool/settings.json`

#### 3. First-Run Experience

The very first time a user opens the DDD Tool (no `~/.ddd-tool/` directory exists), they see a guided setup:

```
Step 1 of 2: Claude Code
┌──────────────────────────────────────────────────────────────┐
│  Claude Code enables automated implementation from your       │
│  flow designs.                                                │
│                                                                │
│  Claude Code CLI:  [✓ Enable]                                │
│  Path: [claude                    ] [Detect]                 │
│  ✓ Found at /usr/local/bin/claude                            │
│                                                                │
│                          [Skip Setup]   [Next →]             │
└──────────────────────────────────────────────────────────────┘

Step 2 of 2: Get Started
┌──────────────────────────────────────────────────────────────┐
│  How would you like to start?                                 │
│                                                                │
│  ● New Project                                               │
│    Create a new DDD project from scratch                     │
│                                                                │
│  ○ Open Existing                                             │
│    Open an existing DDD project folder                       │
│                                                                │
│  ○ Explore Sample                                            │
│    Open a pre-built sample project with 3 domains and 5 flows│
│                                                                │
│                                   [← Back]   [Get Started]  │
└──────────────────────────────────────────────────────────────┘
```

**What first-run creates:**
- `~/.ddd-tool/` directory
- `~/.ddd-tool/settings.json` with defaults
- `~/.ddd-tool/recent-projects.json` (empty)
- `~/.ddd-tool/first-run-completed: true` flag

**Sample project:** A bundled read-only project (embedded in the app binary) with:
- 2 domains (users, billing)
- 3 flows (user-register, user-login, create-subscription)
- Pre-filled specs with all node types demonstrated
- User can browse all 3 canvas levels, open spec panel, see how everything connects
- Cannot be modified — a banner says "This is a sample project. Create your own to start editing."

#### 4. Error States and Recovery

Every operation in the DDD Tool can fail. Each failure type has a defined behavior:

**File operation errors:**

| Error | User sees | Recovery |
|-------|-----------|----------|
| YAML parse error | Red banner on spec panel: "Invalid YAML at line 23: unexpected mapping" + highlight line | Auto-revert to last valid state. User fixes in spec panel or text editor. |
| File not found | Toast: "Flow spec not found: {path}. Was it deleted outside DDD?" | Remove from project index. Offer to recreate from last known state if cached. |
| Permission denied | Toast: "Cannot write to {path}. Check file permissions." | No auto-retry. User must fix permissions. |
| Disk full | Modal: "Cannot save — disk full. Free up space and try again." | Block further saves until resolved. |

**Git operation errors:**

| Error | User sees | Recovery |
|-------|-----------|----------|
| Merge conflict | Modal: "Merge conflict in {files}. Resolve in terminal or editor." + show conflicting files | Open files in external editor. After resolution, user clicks "Mark resolved." |
| Uncommitted changes on pull | Toast: "Cannot pull — uncommitted changes. Commit or stash first." | Offer [Commit now] or [Stash & pull] buttons. |
| Push rejected | Toast: "Push rejected — remote has new changes. Pull first." | Offer [Pull & retry] button. |
| Auth failed | Toast: "Git authentication failed. Check your credentials." | Link to Git settings. |

**Claude Code / PTY errors:**

| Error | User sees | Recovery |
|-------|-----------|----------|
| Claude Code not found | Implementation panel: "Claude Code CLI not found. [Install] [Configure path]" | Link to install instructions + settings. |
| PTY spawn failed | Implementation panel: "Failed to start terminal session." | [Retry] button. Check if another PTY session is stuck — offer to kill it. |
| PTY disconnected | Terminal shows: "Session disconnected." | [Reconnect] or [New session] buttons. Terminal output preserved. |
| Claude Code error exit | Red banner: "Claude Code exited with error. See terminal output." | Terminal scrolls to error. [Re-run] button with same prompt. |

**Canvas / UI errors:**

| Error | User sees | Recovery |
|-------|-----------|----------|
| Invalid connection | Node connection rejected (snap-back animation) | Toast: "Cannot connect {type} to {type} — incompatible." |
| Circular dependency | Connection rejected | Toast: "This connection would create a cycle." |
| Orphaned nodes | Yellow dot on disconnected nodes | Warning in validation panel: "3 nodes are not connected to any path." |

**Global error handling pattern:**

```
Severity levels:
  INFO    → Toast notification (auto-dismiss 5s)
  WARNING → Toast notification (manual dismiss) + yellow indicator
  ERROR   → Red banner in relevant panel + blocks related actions
  FATAL   → Modal dialog + blocks all actions until resolved

All errors are logged to: ~/.ddd-tool/logs/ddd-tool.log
Log rotation: 10MB max, keep 5 files
Logs include: timestamp, severity, component, message, stack trace (if error)
```

**Auto-save and crash recovery:**
- Auto-save every 30 seconds (configurable in settings)
- On crash, next launch detects unsaved state in `.ddd/autosave/`
- Recovery dialog: "DDD Tool didn't shut down cleanly. Recover unsaved changes?" → [Recover] / [Discard]
- Auto-save writes to `.ddd/autosave/{flow_id}.yaml` (not the real spec file) to avoid corrupting specs

#### 5. Undo/Redo System

The DDD Tool supports full undo/redo for all canvas and spec operations. This is essential for a design tool — users must be able to experiment freely.

**Scope:** Undo/redo is **per-flow** (each Level 3 flow sheet has its own history). Level 1 and Level 2 are derived views with no editable state, so they don't need undo.

**What is undoable:**

| Action | Undo behavior |
|--------|--------------|
| Add node | Remove the node |
| Delete node | Restore node + its connections |
| Move node | Return to previous position |
| Connect nodes | Remove the connection |
| Disconnect nodes | Restore the connection |
| Edit spec field | Revert to previous value |
| Accept reconciliation item | Revert spec to pre-accept state |
| Bulk operations (paste, duplicate) | Remove all pasted/duplicated items |

**What is NOT undoable (side-effects outside the flow):**

| Action | Why not undoable |
|--------|-----------------|
| Git commit | Use `git revert` — Git's own undo |
| Claude Code implementation | Code is in files — use Git to revert |
| File save (YAML write) | Previous version in Git history |
| Settings changes | Not flow-level state |

**Implementation approach: Command pattern with immutable snapshots**

```
Each flow has an undo stack:

  undoStack: FlowSnapshot[]    ← past states (max 100)
  redoStack: FlowSnapshot[]    ← future states (cleared on new action)
  current:   FlowSnapshot      ← the live state

FlowSnapshot = {
  nodes: DddNode[]             ← deep copy of all nodes
  connections: Connection[]    ← deep copy of all connections
  specValues: Record<string, any>  ← spec field values for all nodes
  timestamp: number
  description: string          ← "Added process node", "Moved decision node", etc.
}
```

**How it works:**

1. Before any mutation, push `current` onto `undoStack`
2. Apply the mutation to `current`
3. Clear `redoStack` (new action invalidates redo history)
4. On **Undo** (`Cmd+Z`): push `current` onto `redoStack`, pop `undoStack` into `current`
5. On **Redo** (`Cmd+Shift+Z`): push `current` onto `undoStack`, pop `redoStack` into `current`

**Coalescing rapid changes:**
- Typing in a spec field doesn't create a snapshot per keystroke
- Instead, coalesce: if the last snapshot was the same field and < 500ms ago, overwrite it
- Drag operations: only create snapshot on mouse-up, not during drag

**Undo stack limits:**
- Max 100 snapshots per flow (configurable)
- When limit reached, oldest snapshot is dropped
- Stack is in-memory only — lost on app close (Git is the persistent undo)

**UI indicators:**

```
┌──────────────────────────────────────────────────────────────┐
│ [← ↩ Undo] [Redo ↪ →]    "Added process node"              │
│              ↑ grayed out when empty                         │
│              ↑ tooltip shows description of what will be     │
│                undone/redone                                  │
└──────────────────────────────────────────────────────────────┘
```

**Keyboard shortcuts:**

| Shortcut | Action |
|----------|--------|
| `Cmd+Z` / `Ctrl+Z` | Undo |
| `Cmd+Shift+Z` / `Ctrl+Shift+Z` | Redo |
| `Cmd+Y` / `Ctrl+Y` | Redo (alternative) |

### Design Validation

Before a flow reaches Claude Code for implementation, the DDD Tool validates the design for completeness, correctness, and cross-domain integrity. Validation runs at three scopes: **Flow-level** (single flow graph), **Domain-level** (flows within a domain), and **System-level** (cross-domain wiring). The "Implement" button is blocked until all critical issues are resolved.

#### Validation Severity

| Severity | Icon | Meaning | Blocks implementation? |
|----------|------|---------|----------------------|
| **Error** | ✗ (red) | Design is broken — will produce incorrect or unimplementable code | Yes |
| **Warning** | ⚠ (amber) | Design is technically valid but likely incomplete or suspicious | No (user can override) |
| **Info** | ℹ (blue) | Suggestion for improvement — doesn't affect correctness | No |

#### 1. Flow-Level Validation (Single Flow)

Runs whenever a flow is saved, and always before implementation.

**Graph completeness:**

| Check | Severity | Description |
|-------|----------|-------------|
| Trigger exists | Error | Every flow must have exactly one trigger node |
| All paths reach terminal | Error | Starting from trigger, every possible path through the graph must end at a terminal node. Dead-end nodes (non-terminal with no outgoing connections) are errors. |
| No orphaned nodes | Error | Every node must be reachable from the trigger. Unreachable nodes are flagged. |
| No circular paths (non-agent) | Error | Traditional flows must be acyclic (DAGs). Agent flows are excluded — their loop is intentional. |
| Decision branches complete | Error | Every decision node must have both `true` and `false` connections |
| Input node has connections | Error | Every input node must connect to at least one downstream node for both `valid` and `invalid` branches |
| Terminal nodes have no outgoing | Warning | Terminal nodes should not connect to anything |

**Spec completeness:**

| Check | Severity | Description |
|-------|----------|-------------|
| Trigger type defined | Error | Trigger must have `type` (http, event, scheduled, manual) |
| HTTP trigger has method + path | Error | If trigger type is `http`, both `method` and `path` are required |
| Input fields have types | Error | Every field in an input node must have a `type` |
| Required fields have error messages | Error | If a field is `required: true`, it must have an `error` message |
| Validation rules have error messages | Warning | Fields with `min_length`, `max_length`, `format` etc. should have `error` messages |
| Decision has check defined | Error | Decision node must have `check` or `condition` specified |
| Decision has error_code on true | Warning | Decision error branches should reference an error code from errors.yaml |
| Terminal has status code | Warning | HTTP flow terminal nodes should have `status` defined |
| Terminal has response body | Info | Terminal nodes should have `body` for non-empty responses |
| Data store has operation + model | Error | Data store nodes must have `operation` (create/read/update/delete) and `model` |
| Process node has description | Warning | Process nodes should describe what they do |

**Reference integrity:**

| Check | Severity | Description |
|-------|----------|-------------|
| Error codes exist | Error | Every error code referenced in the flow (e.g., `DUPLICATE_ENTRY`) must exist in `specs/shared/errors.yaml` |
| Schema models exist | Error | Every model referenced in data_store nodes (e.g., `User`) must have a corresponding `specs/schemas/{model}.yaml` |
| Sub-flow references exist | Error | Sub-flow nodes must reference an existing flow in the same or another domain |
| Event references exist | Warning | Event publish/consume references should match defined events in domain.yaml |
| Field references resolve | Warning | Data expressions like `$.email` or `$.user.id` should reference fields that exist in the flow's data context |

**Agent flow validation (when `flow_type: agent`):**

| Check | Severity | Description |
|-------|----------|-------------|
| Agent loop exists | Error | Agent flows must have exactly one `agent_loop` node |
| At least one tool | Error | Agent must have at least one tool node connected |
| At least one terminal tool | Error | At least one tool must be `terminal: true` (can end the loop) |
| Max iterations defined | Warning | Agent loop should have `max_iterations` to prevent infinite loops |
| LLM model specified | Warning | Agent loop should specify which model to use |
| Guardrail has checks | Warning | Guardrail nodes should have at least one check defined |
| Human gate has description | Warning | Human gate should describe what the user is approving |

**Orchestration flow validation (when `flow_type: orchestration`):**

| Check | Severity | Description |
|-------|----------|-------------|
| Orchestrator has agents | Error | Orchestrator must reference at least 2 agents |
| Orchestrator has strategy | Error | Must specify strategy (supervisor, round_robin, broadcast, consensus) |
| Router has rules | Error | Smart router must have at least one routing rule |
| Router has fallback | Warning | Smart router should define a fallback agent |
| Handoff has target | Error | Handoff node must specify target agent |
| Handoff has mode | Warning | Handoff should specify mode (transfer, consult, collaborate) |
| Agent group has members | Error | Agent group must list at least 2 member agents |
| Referenced agents exist | Error | All agent IDs referenced in orchestrator/router/handoff must correspond to actual agent flows in the project |
| Circuit breaker thresholds | Warning | If circuit breaker is configured, `failure_threshold` and `timeout` should be specified |

#### 2. Domain-Level Validation (Flows Within a Domain)

Runs when any flow in the domain is saved, or when the domain map (Level 2) is viewed.

| Check | Severity | Description |
|-------|----------|-------------|
| No duplicate flow IDs | Error | Every flow within a domain must have a unique `flow.id` |
| No duplicate HTTP paths | Error | Two flows in the same domain cannot have the same HTTP method + path combination |
| Internal event consumers matched | Warning | If flow A publishes event X and flow B in the same domain consumes event X, verify the event payload shapes match |
| Shared schema consistency | Warning | If two flows in the same domain use the same model (e.g., `User`), they should reference the same schema file |
| Domain.yaml matches flows | Info | `domain.yaml` flow list should match the actual flow files on disk |

#### 3. System-Level Validation (Cross-Domain Wiring)

Runs on project load, after git pull, and before batch implementation. This is the critical cross-domain check.

**Event wiring:**

| Check | Severity | Description |
|-------|----------|-------------|
| Published events have consumers | Warning | If domain A publishes event `contract.ingested`, at least one other domain should consume it. An event published to nobody is suspicious. |
| Consumed events have publishers | Error | If domain B consumes event `contract.ingested`, some domain must publish it. Consuming a non-existent event is broken. |
| Event payload shapes match | Error | The publisher's event payload shape must match what the consumer expects. If publisher sends `{ contract_id, source }` but consumer expects `{ document_id, origin }`, this is a field mismatch. |
| Event names are consistent | Warning | Event names should follow a consistent pattern (e.g., `{entity}.{action}`). `contract.ingested` vs `contractIngested` vs `CONTRACT_INGESTED` flags an inconsistency. |

**Cross-domain data contracts:**

| Check | Severity | Description |
|-------|----------|-------------|
| Shared schemas are identical | Error | If domain A and domain B both reference `specs/schemas/user.yaml`, they get the same schema. But if domain A defines `User` inline while domain B uses the schema file, flag the divergence. |
| Cross-domain field references | Warning | If flow in domain A passes `$.user.id` to an event consumed by domain B, verify `user.id` exists in the User schema |
| API dependencies are valid | Warning | If domain A's flow calls domain B's API (via service_call node), verify the target endpoint exists as a flow in domain B |

**Portal wiring:**

| Check | Severity | Description |
|-------|----------|-------------|
| Portal targets exist | Error | Every portal node on Level 2 must point to an existing domain |
| Portal flow references exist | Error | If a portal says "api/user-register triggers ingestion/webhook-ingestion", both flows must exist |
| Bidirectional portals match | Warning | If domain A has a portal to domain B, domain B should have a portal back to domain A (unless the relationship is one-way) |

**Orchestration wiring:**

| Check | Severity | Description |
|-------|----------|-------------|
| Orchestrated agents exist | Error | Agents referenced in orchestration flows must exist as actual agent flows somewhere in the project |
| Agent domains match | Warning | If an orchestrator in domain A references an agent in domain B, verify the cross-domain dependency is intentional (show info noting the cross-domain reference) |
| No circular orchestration | Error | Agent A orchestrates Agent B which orchestrates Agent A → infinite loop. Detect cycles in the orchestration graph. |
| Router targets reachable | Error | Every agent targeted by a smart router must exist and be a valid agent flow |

#### Validation Panel UI

The validation panel is accessible from:
- **Level 3 toolbar:** ✓ button shows flow-level issues
- **Level 2 toolbar:** ✓ button shows domain-level issues
- **Level 1 toolbar:** ✓ button shows system-level issues

```
┌──────────────────────────────────────────────────────────────┐
│ VALIDATION — api/user-register                                │
│                                                                │
│ ✗ 2 errors · ⚠ 3 warnings · ℹ 1 info                       │
│                                                                │
│ ✗ Errors (must fix before implementation):                    │
│   • Node "check_duplicate" decision missing false branch     │
│     → connect false branch to a target node                  │
│     [Select node]                                             │
│   • Error code "USER_EXISTS" not found in errors.yaml        │
│     → add it to specs/shared/errors.yaml or use existing     │
│     [Open errors.yaml]  [Show similar: DUPLICATE_ENTRY]      │
│                                                                │
│ ⚠ Warnings:                                                   │
│   • Terminal "return_success" has no status code              │
│     → add status: 201 to the terminal spec                   │
│     [Select node]                                             │
│   • Input field "name" has max_length but no error message   │
│     → add error message for better user experience           │
│     [Select node]                                             │
│   • Process node "hash_password" has no description          │
│     [Select node]                                             │
│                                                                │
│ ℹ Info:                                                       │
│   • Terminal "return_error" has no response body defined      │
│     [Select node]                                             │
│                                                                │
│ [Dismiss warnings]                                            │
└──────────────────────────────────────────────────────────────┘
```

**System-level validation panel (Level 1):**

```
┌──────────────────────────────────────────────────────────────┐
│ SYSTEM VALIDATION                                              │
│                                                                │
│ ✗ 1 error · ⚠ 2 warnings                                    │
│                                                                │
│ Cross-Domain Wiring:                                          │
│ ✗ Event "contract.analyzed" consumed by notification domain   │
│   but no domain publishes it                                  │
│   → analysis domain should publish this event                │
│   [Go to analysis domain]                                     │
│                                                                │
│ ⚠ Event "contract.ingested" payload mismatch:                │
│   Publisher (ingestion): { contract_id: string, source: str } │
│   Consumer (analysis):   { document_id: string, origin: str } │
│   → field names don't match — update one side                │
│   [Go to publisher]  [Go to consumer]                        │
│                                                                │
│ ⚠ Portal ingestion → analysis exists but no reverse portal   │
│   → add portal from analysis back to ingestion if needed     │
│   [Add reverse portal]  [Dismiss]                            │
│                                                                │
│ Orchestration Wiring:                                         │
│ ✓ All orchestrated agents exist and are reachable            │
│ ✓ No circular orchestration dependencies                     │
│                                                                │
│ Schema Integrity:                                              │
│ ✓ All referenced schemas exist                                │
│ ✓ No inline/file schema divergence                           │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

#### Validation on Canvas (Real-Time Indicators)

Validation runs continuously as the user designs. Invalid nodes show visual indicators directly on the canvas:

```
┌────────────────────┐     ┌────────────────────┐
│ ✓ validate_input   │────▶│ ⚠ check_duplicate  │──── ?
│   3 fields defined │     │   missing false     │
│                    │     │   branch            │
└────────────────────┘     └────────────────────┘
     │ (invalid)
     ▼
┌────────────────────┐
│ ✗ return_error     │
│   no error_code    │
│   referenced       │
└────────────────────┘
```

- **✓ (green border):** Node passes all validation checks
- **⚠ (amber border + dot):** Node has warnings
- **✗ (red border + dot):** Node has errors
- Hovering the indicator shows a tooltip with the issue(s)
- Clicking the indicator selects the node and opens the validation panel filtered to that node

#### Implementation Gate

The validation gate checks flow quality before implementation proceeds (used by the CLI's `/ddd-implement` command):

```
┌──────────────────────────────────────────────────────────────┐
│ ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐   │
│ │ ✓ Validate  │→ │ Build Prompt │→ │ ▶ Run Claude Code  │   │
│ └─────────────┘  └──────────────┘  └────────────────────┘   │
│                                                                │
│ Step 1: Validate                                              │
│   Flow validation:     ✓ passed (0 errors, 1 warning)       │
│   Domain validation:   ✓ passed                              │
│   System validation:   ✓ passed                              │
│   References:          ✓ all schemas + error codes exist     │
│                                                                │
│ Step 2: Build Prompt                                          │
│   [Preview prompt]                                            │
│                                                                │
│ Step 3: Implement                                             │
│   [▶ Run Claude Code]                                        │
│                                                                │
│ ⚠ 1 warning: Terminal "return_success" has no status code    │
│   [Implement anyway]  [Fix first]                            │
└──────────────────────────────────────────────────────────────┘
```

**Gate rules:**
- **Errors present:** "Implement" button is disabled. Must fix all errors first.
- **Warnings only:** "Implement" button shows "Implement anyway" with warning count. User can proceed.
- **Clean (no issues):** "Implement" button is green and enabled.

#### Batch Implementation Validation

When implementing multiple flows from the queue, validation runs for ALL selected flows before any implementation starts:

```
Validating 5 selected flows...
  ✓ api/user-register        — clean
  ✓ api/user-login           — clean
  ⚠ api/user-reset-password  — 2 warnings
  ✗ api/get-contracts        — 1 error (missing schema: Contract)
  ✓ notification/deadline-alert — clean

Cannot start batch: 1 flow has errors.
[Fix api/get-contracts]  [Skip and implement 4 clean flows]
```

#### When Validation Runs

| Trigger | Scope | Behavior |
|---------|-------|----------|
| Node added/edited/connected | Flow-level | Immediate (debounced 500ms) — updates canvas indicators |
| Flow saved | Flow-level + domain-level | Full validation, updates validation panel |
| "Implement" clicked | All three levels | Blocks if errors exist |
| Level 2 viewed | Domain-level | Background check, badge on domain map |
| Level 1 viewed / project load | System-level | Background check, badge on system map |
| Git pull completed | System-level | Re-check cross-domain wiring (specs may have changed) |
| Batch implement selected | All three levels for all selected flows | Pre-flight check before queue starts |

#### Validation Badges on Canvas

Level 1 (System Map) — domain blocks show validation status:
```
┌──────────────────┐     ┌──────────────────┐
│ ingestion         │     │ analysis          │
│ 4 flows           │     │ 3 flows           │
│ ✓ valid          │     │ ⚠ 2 warnings     │
└──────────────────┘     └──────────────────┘
```

Level 2 (Domain Map) — flow blocks show validation status alongside test and sync badges:
```
┌──────────────────┐
│ user-register     │
│ ✓ valid          │    ← validation badge
│ ✓ 14/14 tests   │    ← test badge
│ ~ 85% synced    │    ← sync badge
└──────────────────┘
```

### Claude Code Integration

The DDD Tool integrates with Claude Code CLI to turn specs into running code. Three components work together: a **Prompt Builder** that constructs optimal prompts from specs (copied to clipboard via "Copy Command"), **Stale Detection** that tracks spec-vs-code drift with visual banners, and **CLAUDE.md Auto-Generation** that keeps Claude Code instructions in sync with the project. Implementation is driven from the CLI via `/ddd-implement`.

#### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ DDD TOOL                                                         │
│                                                                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────────────────────┐│
│  │ Canvas  │ │ Spec    │ │ Git     │ │ Validation Panel       ││
│  │         │ │ Panel   │ │ Panel   │ │                        ││
│  │         │ │         │ │         │ │ Flow/Domain/System     ││
│  │         │ │         │ │         │ │ scope validation       ││
│  └─────────┘ └─────────┘ └─────────┘ └────────────────────────┘│
│                                                                   │
│  ┌──────────────────────────────────────────────────────────────┐│
│  │ Stale Banners (L2 FlowBlock / L3 FlowCanvas)                ││
│  │ Copy Command → /ddd-implement {domain}/{flow}                ││
│  └──────────────────────────────────────────────────────────────┘│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**ClaudeCommandBox:** A reusable UI component that auto-generates the correct Claude Code slash command based on the current navigation scope. Shows a copyable command in a bordered box with Terminal icon:

| Navigation Level | Generated Command |
|-----------------|------------------|
| Flow (L3) | `/ddd-implement {domainId}/{flowId}` |
| Domain (L2) | `/ddd-implement {domainId}` |
| System (L1) | `/ddd-implement --all` |
| Override | Any custom command (e.g. `/ddd-sync`) |

**Fix Runtime Error:** After implementation completes (Done state), users can report runtime errors encountered while testing. Clicking "Fix Runtime Error" reveals a text area to paste the error message. Submitting builds a targeted fix prompt that:
- References the original flow and domain
- Wraps the error in a code block
- Instructs Claude to fix existing code (not rewrite from scratch)
- Runs tests after fixing
- Sets up local defaults for missing infrastructure

#### 2. Prompt Builder

The DDD Tool auto-constructs an optimal prompt for Claude Code based on the flow being implemented. The user never has to remember which files to reference or in what order.

**Prompt template:**

```markdown
# Auto-generated prompt for Claude Code
# Flow: {flow_id} | Domain: {domain_id}

Read these spec files in order:

1. specs/architecture.yaml — Project structure, conventions, dependencies
2. specs/shared/errors.yaml — Error codes and messages
{for each schema referenced by the flow}
3. specs/schemas/{schema}.yaml — Data model
{end for}
4. specs/domains/{domain}/flows/{flow}.yaml — The flow to implement

## Instructions

Implement the {flow_id} flow following architecture.yaml exactly:

- Create the endpoint matching the trigger spec (method: {method}, path: {path})
- Create request/response schemas matching the input node validations
- Implement each node as described in the spec
- Use EXACT error codes from errors.yaml (do not invent new ones)
- Use EXACT validation messages from the flow spec
- Create unit tests covering: happy path, each validation failure, each error path

## File locations

- Implementation: src/domains/{domain}/
- Tests: tests/unit/domains/{domain}/

## After implementation

Update .ddd/mapping.yaml with:
```yaml
flows:
  {flow_id}:
    spec: specs/domains/{domain}/flows/{flow}.yaml
    spec_hash: {sha256 of spec file}
    files: [list of files you created]
    implemented_at: {ISO timestamp}
```

{if flow type is agent}
## Agent-specific

This is an agent flow. Implement:
- Agent runner with the agent loop configuration
- Tool implementations for each tool defined in the spec
- Guardrail middleware for input/output filtering
- Memory management per the memory spec
- Use mocked LLM responses in tests
{end if}

{if this is an update to existing code}
## Update mode

This flow was previously implemented. The spec has changed:
{list of changes}

Update the existing code to match the new spec.
Do NOT rewrite files from scratch — modify the existing implementation.
Update affected tests.
{end if}
```

**What makes the prompt smart:**
- Automatically resolves which schemas the flow references (via `$ref` and data_store model fields)
- Includes only relevant error codes (not the full errors.yaml if the flow only uses 3 codes)
- Detects if this is a new implementation or an update
- For updates, includes a diff summary of what changed in the spec
- For agent flows, adds agent-specific instructions
- Includes architecture.yaml conventions so Claude Code knows naming patterns, folder structure, etc.

**The user can edit the prompt** before running. The prompt preview has an "Edit" button that opens it as editable text. Useful for adding specific instructions like "use bcrypt for password hashing" or "skip the email sending for now."

#### 3. Stale Detection

When a flow has existing code (recorded in `.ddd/mapping.yaml`) but the spec has changed since code generation, the flow is **stale**. DDD Tool detects this by comparing the spec file's SHA-256 hash against the `spec_hash` stored at implementation time.

**Stale detection triggers:**
- On project load (compare all hashes)
- On flow save (compare saved flow's hash)
- On git pull (any spec files changed)

**What the user sees:**

On Level 2 (Domain Map), stale flow blocks show a warning badge:
```
┌──────────────────┐
│ webhook-ingestion │
│ ⚠ spec changed    │
│   2 updates       │
└──────────────────┘
```

On Level 3 (Flow Sheet), a banner appears at the top:
```
┌──────────────────────────────────────────────────────────────┐
│ ⚠ This flow's spec changed since code was generated           │
│                                                                │
│ Changes:                                                      │
│ • Added send_verification_email node                          │
│ • Changed return message to "Check your email..."             │
│                                                                │
│ [▶ Update code]  [View diff]  [Dismiss]                      │
└──────────────────────────────────────────────────────────────┘
```

**Change detection:** The DDD Tool computes a human-readable diff between the old spec (at implementation time, cached in `.ddd/cache/`) and the current spec. This diff is included in the update prompt so Claude Code knows exactly what to change.

**Spec cache:** When a flow is implemented, the DDD Tool saves a copy of the spec at that point:
```
.ddd/
├── cache/
│   └── specs-at-implementation/
│       ├── api--user-register.yaml      # Spec as it was when code was generated
│       ├── ingestion--webhook-ingestion.yaml
│       └── ...
```

#### 4. Test Runner

After Claude Code finishes implementing a flow, the project's test suite runs via the CLI workflow (`/ddd-implement` handles test execution natively).

**Configuration:**

```yaml
# .ddd/config.yaml
testing:
  # Test commands per language/framework (auto-detected from architecture.yaml)
  command: pytest                    # Or: jest, go test, cargo test
  args: ["--tb=short", "-q"]        # Additional arguments

  # Run only tests related to the implemented flow
  scoped: true                      # If true, runs only relevant tests
  scope_pattern: "tests/**/test_{flow_id}*"  # Glob for finding flow tests

  # Auto-run after implementation
  auto_run: true                    # Run tests automatically after Claude Code finishes
```

**Test result display:**

```
┌──────────────────────────────────────────────────────────────┐
│ TEST RESULTS — api/user-register                              │
│                                                                │
│ ✓ test_register_success                              0.12s   │
│ ✓ test_register_duplicate_email                      0.08s   │
│ ✓ test_register_invalid_email_format                 0.03s   │
│ ✗ test_register_short_password                       0.04s   │
│   AssertionError: Expected status 422, got 400               │
│   > assert response.status_code == 422                       │
│                                                                │
│ 3/4 passing · 1 failing · 0.27s                              │
│                                                                │
│ [Re-run tests]  [Fix failing test]                           │
└──────────────────────────────────────────────────────────────┘
```

**"Fix failing test" button:** Sends the failing test output back to Claude Code with a prompt: "This test is failing after implementing {flow_id}. Fix the implementation to match the spec." This creates a fix-and-retest loop without the user having to copy-paste error messages.

**Test results on canvas:** After tests run, flow blocks on Level 2 show a test badge:
```
┌──────────────────┐
│ user-register     │
│ ✓ 4/4 tests      │    ← green badge
└──────────────────┘

┌──────────────────┐
│ webhook-ingestion │
│ ✗ 3/4 tests      │    ← red badge
└──────────────────┘
```

#### 5. CLAUDE.md Auto-Generation

The DDD Tool generates and maintains a `CLAUDE.md` file in the project root. This file tells Claude Code how the project is structured and how to work with DDD specs. It's regenerated whenever specs change.

**Generated CLAUDE.md:**

```markdown
<!-- Auto-generated by DDD Tool. Manual edits below the CUSTOM section are preserved. -->

# Project: Obligo

## Spec-Driven Development

This project uses Design Driven Development (DDD). All business logic
is specified in YAML files under `specs/`. Code MUST match specs exactly.

## Spec Files

- `specs/system.yaml` — Project identity, tech stack, 4 domains
- `specs/architecture.yaml` — Folder structure, conventions, dependencies
- `specs/config.yaml` — Environment variables schema
- `specs/shared/errors.yaml` — 12 error codes (VALIDATION_ERROR, NOT_FOUND, ...)
- `specs/schemas/*.yaml` — 6 data models (User, Document, Contract, ...)

## Domains

| Domain | Flows | Status |
|--------|-------|--------|
| ingestion | webhook-ingestion, scheduled-sync, manual-upload, email-intake | 3 implemented, 1 pending |
| analysis | extract-obligations (agent), classify-risk, generate-summary | 1 implemented, 2 pending |
| api | user-register, user-login, user-reset-password, get-contracts, get-obligations | 2 implemented, 3 pending |
| notification | deadline-alert (agent), compliance-report | 0 implemented, 2 pending |

## Implementation Rules

1. **Read architecture.yaml first** — it defines folder structure and conventions
2. **Follow the folder layout** — put files where architecture.yaml specifies
3. **Use EXACT error codes** — from specs/shared/errors.yaml, do not invent new ones
4. **Use EXACT validation messages** — from the flow spec, do not rephrase
5. **Match field types exactly** — spec field types map to language types
6. **Update .ddd/mapping.yaml** — after implementing a flow, record the mapping

## Tech Stack

- Language: Python 3.11
- Framework: FastAPI
- ORM: SQLAlchemy
- Database: PostgreSQL
- Cache: Redis
- Queue: Celery

## Commands

```bash
pytest                    # Run tests
mypy src/                 # Type check
ruff check .              # Lint
alembic upgrade head      # Run migrations
uvicorn src.main:app      # Start server
```

## Folder Structure

```
src/
├── main.py
├── config/
├── shared/
│   ├── database.py
│   ├── exceptions.py
│   └── middleware.py
├── models/
└── domains/
    └── {domain}/
        ├── router.py
        ├── schemas.py
        └── services.py
```

<!-- CUSTOM: Add your own instructions below this line. They won't be overwritten. -->

```

**What triggers regeneration:**
- New flow created or deleted
- Domain added or removed
- Architecture.yaml changed
- Implementation status changed (flow implemented or became stale)
- User clicks "Regenerate CLAUDE.md" in settings

**Custom section preserved:** Everything below `<!-- CUSTOM -->` is never overwritten. Users can add their own conventions, workarounds, or preferences there.

#### 6. Cowork Workflow — DDD Tool ↔ Claude Code Terminal

The DDD Tool and Claude Code work together through a design-implement-sync loop. Rather than embedding a full terminal in DDD, the recommended workflow uses Claude Code's own terminal with custom slash commands:

```
┌─────────────┐    save specs     ┌──────────────┐
│   DDD Tool   │ ──────────────► │  YAML files   │
│  (design)    │                  │  on disk      │
└─────────────┘                  └──────┬───────┘
       ▲                                │
       │  /ddd-sync                     │  /ddd-implement
       │  (reconcile)                   │  (generate code)
       │                                ▼
┌─────────────┐                  ┌──────────────┐
│   DDD Tool   │ ◄────────────── │  Claude Code  │
│  (reload)    │   mapping.yaml  │  Terminal     │
└─────────────┘                  └──────────────┘
```

**The workflow:**

1. **Design** in DDD Tool — create/edit flows, add nodes, set specs
2. **Save** — specs are written to `specs/` as YAML
3. **Copy command** from ClaudeCommandBox in DDD → paste into Claude Code terminal
4. **Implement** — Claude Code reads specs, generates code, runs tests
5. **Fix interactively** — fix runtime errors, test failures in Claude Code
6. **Sync** — run `/ddd-sync` to update `.ddd/mapping.yaml`
7. **Reload** in DDD Tool to see updated implementation status

**Custom Claude Code commands** (installed at `~/.claude/commands/`, repo: [mhcandan/claude-commands](https://github.com/mhcandan/claude-commands)):

| Command | Scope | What it does |
|---------|-------|-------------|
| `/ddd-create` | New project | Describe project in natural language → generates full spec structure (domains, flows, schemas, config) |
| `/ddd-implement --all` | Whole project | Implements all domains and flows |
| `/ddd-implement {domain}` | Single domain | `/ddd-implement users` |
| `/ddd-implement {domain}/{flow}` | Single flow | `/ddd-implement users/user-registration` |
| `/ddd-implement` | Interactive | Lists flows with status, asks what to implement |
| `/ddd-update` | Spec modification | Natural language change request → updated YAML specs |
| `/ddd-update --add-flow {domain}` | Add flow | "Add an order cancellation flow" → new flow spec |
| `/ddd-update --add-domain` | Add domain | "Add a notifications domain" → new domain + flows |
| `/ddd-sync` | Sync mapping | Updates `.ddd/mapping.yaml` with implementation state |
| `/ddd-sync --discover` | Discover | Also finds untracked code and suggests new flow specs |
| `/ddd-sync --fix-drift` | Fix drift | Re-implements flows where specs have drifted |
| `/ddd-sync --full` | Full sync | All of the above |

**The `/ddd-create` command** (Phase 1 — Create):
1. Fetches the DDD Usage Guide from GitHub at runtime
2. Reads the user's project description
3. Asks clarifying questions if the description is brief
4. Generates complete spec structure: `ddd-project.json`, `system.yaml`, `architecture.yaml`, `config.yaml`, `errors.yaml`, `schemas/*.yaml`, `domain.yaml` for each domain, and flow YAML files with full node graphs
5. Wires all connections with proper `sourceHandle` values
6. Runs quality checks (all paths reach terminal, decision branches wired, etc.)

**The `/ddd-update` command** (Phase 3 — Build/Iterate):
1. Reads the user's natural language change request
2. Reads existing specs (system, architecture, errors, schemas, domain, flow)
3. Modifies the YAML spec to reflect the change
4. Supports: adding/removing/modifying nodes, changing connections, updating specs, adding new flows or domains
5. Preserves all existing `sourceHandle` wiring and node IDs

The `/ddd-implement` command:
1. Finds `ddd-project.json` in the project
2. Resolves scope from the argument
3. Reads flow specs (`specs/domains/{domain}/flows/{flow}.yaml`)
4. Checks `.ddd/mapping.yaml` for existing implementation (skip if up-to-date, update if changed)
5. Implements following the node graph (trigger → terminals)
6. Writes tests covering happy path, decision branches, error states, validation
7. Runs tests and fixes until passing
8. Updates `.ddd/mapping.yaml` with spec hash, timestamp, and file list

#### Configuration

```yaml
# .ddd/config.yaml — Claude Code integration
claude_code:
  enabled: true
  command: claude                  # CLI command (must be in PATH)

  post_implement:
    run_tests: true                # Auto-run tests after code gen
    run_lint: false                # Auto-run linter after code gen
    auto_commit: false             # Never auto-commit (user reviews first)
    regenerate_claude_md: true     # Update CLAUDE.md after implementation

  prompt:
    include_architecture: true     # Always include architecture.yaml
    include_errors: true           # Always include errors.yaml
    include_schemas: auto          # Include only referenced schemas

testing:
  command: pytest
  args: ["--tb=short", "-q"]
  scoped: true
  scope_pattern: "tests/**/test_{flow_id}*"
  auto_run: true

reconciliation:
  auto_run: true                 # Auto-reconcile after implementation
  auto_accept_matching: true     # Auto-accept items where code matches spec
  notify_on_drift: true          # Show notification when drift detected
```

#### 7. Reverse Drift Detection

Stale Detection (section 3 above) catches when **specs change but code hasn't been updated**. Reverse Drift Detection catches the opposite: when **code changes but specs haven't been updated**. Together they form a bidirectional sync loop.

**The problem:** Claude Code follows the spec, but it also makes practical decisions — adding error handling the spec didn't mention, splitting a node into multiple steps, using a slightly different response shape, or adding middleware. These deviations are invisible to the DDD Tool. Over time, the flow diagrams show the *original design* but not what the code *actually does*.

**Three layers work together to detect and resolve this:**

##### Layer 1: Implementation Report

The Prompt Builder (section 2 above) includes an instruction asking Claude Code to report what it actually did:

```markdown
## Implementation Report (required)

After implementing, output a section titled `## Implementation Notes` with:

1. **Deviations** — Anything you did differently from the spec (different field names,
   changed error codes, reordered steps, etc.)
2. **Additions** — Anything you added that the spec didn't mention (middleware,
   validation, caching, extra error handling, helper functions, etc.)
3. **Ambiguities resolved** — Anything the spec was unclear about and how you decided
4. **Schema changes** — Any new fields, changed types, or migration implications

If you followed the spec exactly with no changes, write: "No deviations."
```

The DDD Tool parses this structured output from the terminal after Claude Code finishes. If anything other than "No deviations" is found, it triggers reconciliation.

##### Layer 2: Code → Spec Reconciliation

After implementation, the DDD Tool reads the generated code files and compares them against the flow spec. It produces a **reconciliation report**:

```
┌──────────────────────────────────────────────────────────────┐
│ RECONCILIATION — api/user-register                            │
│                                                                │
│ Sync Score: 85%  (3 items need attention)                     │
│                                                                │
│ ✓ Matching (spec = code):                                     │
│   • POST /auth/register endpoint                             │
│   • validate_input → 3 field validations                     │
│   • check_duplicate → DUPLICATE_ENTRY error                  │
│   • create_user → User model insert                          │
│   • return_success → 201 response                            │
│                                                                │
│ ⚠ Code has but spec doesn't:                                  │
│   • rate_limit middleware (10 req/min per IP)                 │
│     [Accept into spec]  [Remove from code]  [Ignore]         │
│   • password_hash uses bcrypt (spec says "hash()")           │
│     [Accept into spec]  [Ignore]                             │
│   • response includes "token" field (spec only has user)     │
│     [Accept into spec]  [Remove from code]  [Ignore]         │
│                                                                │
│ ⚠ Spec has but code doesn't:                                  │
│   (none — all spec nodes are implemented)                     │
│                                                                │
│ [Accept all matching]  [Dismiss]                              │
└──────────────────────────────────────────────────────────────┘
```

**How reconciliation works:**

1. The DDD Tool reads all files listed in `.ddd/mapping.yaml` for the flow
2. It compares the flow spec YAML against the implementation code (via hash comparison and structural analysis)
3. Differences are identified and categorized (matching, code-only, spec-only)
4. Results are saved to `.ddd/reconciliations/` for review

**Reconciliation actions per item:**

| Action | What happens |
|--------|-------------|
| **Accept into spec** | DDD Tool updates the flow spec YAML to include the addition (adds a node, updates a field, etc.). The canvas reflects the change immediately. |
| **Remove from code** | Builds a targeted prompt for Claude Code: "Remove {item} from the implementation — it's not in the spec." |
| **Ignore** | Marks this item as a known deviation. Stored in `.ddd/mapping.yaml` under `accepted_deviations`. Won't trigger drift warnings again. |

**Accept into spec** is the most powerful action — it reverse-engineers code changes back into the flow diagram. For example, if Claude Code added a `rate_limit` middleware, accepting it adds a process node to the flow with the rate limiting spec filled in.

##### Layer 3: Sync Score

Each implemented flow gets a **sync score** indicating how closely the code matches the spec:

| Score | Meaning | Badge |
|-------|---------|-------|
| **100%** | Code matches spec exactly | ✓ (green) |
| **80-99%** | Minor additions (extra validation, logging) | ~ (yellow) |
| **50-79%** | Significant divergence (extra endpoints, changed schemas) | ⚠ (amber) |
| **< 50%** | Code barely resembles spec | ✗ (red) |

**Where sync scores appear:**

On Level 2 (Domain Map), flow blocks show the sync badge alongside test badges:
```
┌──────────────────┐
│ user-register     │
│ ✓ 4/4 tests      │    ← test badge
│ ~ 85% synced     │    ← sync badge
└──────────────────┘
```

On Level 3 (Flow Sheet), a sync indicator appears in the toolbar:
```
[Flow: user-register]  [✓ Tests: 4/4]  [~ Sync: 85%]  [▶ Implement]
```

Clicking the sync badge opens the reconciliation report.

##### The Full Bidirectional Sync Cycle

```
┌──────────────┐   Stale Detection    ┌───────────────┐
│  SPEC        │ ────────────────────▶ │  CODE         │
│  (YAML)      │   "Spec changed,     │  (src/)       │
│              │    code is stale"     │               │
│              │                       │               │
│              │   Reverse Drift       │               │
│              │ ◀──────────────────── │               │
│              │   "Code diverged,     │               │
│              │    spec is outdated"  │               │
└──────┬───────┘                       └───────┬───────┘
       │                                       │
       │         ┌──────────────┐              │
       └────────▶│ RECONCILE    │◀─────────────┘
                 │              │
                 │ • Compare    │
                 │ • Score      │
                 │ • Accept /   │
                 │   Remove /   │
                 │   Ignore     │
                 └──────────────┘
```

**When reconciliation runs:**
- Automatically after every Claude Code implementation (if `reconciliation.auto_run: true`)
- On project load, for any flow that was implemented outside the DDD Tool (e.g., manually edited code)
- After git pull, if code files in mapping changed but spec didn't

**Reconciliation data storage:**

```yaml
# .ddd/mapping.yaml — extended with reconciliation data
flows:
  api/user-register:
    spec: specs/domains/api/flows/user-register.yaml
    spec_hash: abc123...
    files: [src/domains/api/router.py, src/domains/api/schemas.py, src/domains/api/services.py]
    implemented_at: "2025-01-15T10:30:00Z"
    sync_score: 85
    last_reconciled_at: "2025-01-15T10:32:00Z"
    accepted_deviations:
      - description: "password_hash uses bcrypt"
        code_location: "src/domains/api/services.py:23"
        accepted_at: "2025-01-15T10:33:00Z"
```

#### 8. Diagram-Derived Test Generation

Flow diagrams already contain everything needed to derive comprehensive test cases: every path through the graph, every validation rule with boundaries, every error code, every decision branch, and every terminal state. The DDD Tool extracts this information and generates test specifications, test code, and spec compliance validation — all without the user writing a single test manually.

**Three levels of test generation:**

| Level | What it does | When it runs |
|-------|-------------|--------------|
| **Test Specification** | Walks the flow graph, enumerates all paths, derives test cases | On demand (right-click → "Generate tests") or when flow is saved |
| **Test Code Generation** | Generates actual test code (pytest/jest/etc.) from derived test cases | Before implementation — included in the Claude Code prompt |
| **Spec Compliance Validation** | After implementation, compares test results against what the spec says should happen | After tests run — shown in Implementation Panel |

##### Level 1: Test Specification (Path Analysis)

The DDD Tool treats every flow as a directed graph and walks it to find all possible paths from trigger to terminal nodes.

**Path enumeration algorithm:**

```
Given: Flow graph G with trigger T and terminal nodes {T1, T2, ...}

1. Find all paths from T to each terminal node
2. For each decision node, enumerate both true/false branches
3. For each validation node, enumerate valid/invalid branches
4. Mark each path as: happy_path, error_path, edge_case, or agent_loop

Output: List of TestPath objects, each with:
  - path_id: unique identifier
  - path_type: happy_path | error_path | edge_case | agent_loop
  - nodes: ordered list of node IDs traversed
  - description: human-readable description of what this path tests
  - expected_outcome: what the terminal node produces
```

**Example — user-register flow:**

```
Flow graph:
  trigger → validate_input → check_duplicate → create_user → return_success
                ↓ (invalid)       ↓ (true)
          return_validation   return_duplicate
              _error              _error

Derived paths:
  Path 1 (happy_path): trigger → validate_input ✓ → check_duplicate (false) → create_user → return_success
    Expected: 201, user object with id/email/name

  Path 2 (error_path): trigger → validate_input ✗ → return_validation_error
    Expected: 422, validation error for invalid field

  Path 3 (error_path): trigger → validate_input ✓ → check_duplicate (true) → return_duplicate_error
    Expected: 409, DUPLICATE_ENTRY error code
```

**Boundary test derivation:**

For every `input` node with validation rules, the tool derives boundary test cases:

```
Field: email (type: string, format: email, required: true)
  → valid: "user@example.com"
  → invalid_missing: (omit field) → expects error "Please enter a valid email address"
  → invalid_format: "not-an-email" → expects error "Please enter a valid email address"
  → invalid_type: 12345 → expects type error

Field: password (type: string, min_length: 8, required: true)
  → valid: "password123"
  → boundary_min: "1234567" (7 chars) → expects error "Password must be at least 8 characters"
  → boundary_exact: "12345678" (8 chars) → expects success
  → invalid_missing: (omit field) → expects error

Field: name (type: string, min_length: 2, max_length: 100, required: true)
  → valid: "Alice"
  → boundary_min_below: "A" (1 char) → expects error
  → boundary_min_exact: "Al" (2 chars) → expects success
  → boundary_max_exact: "A" × 100 → expects success
  → boundary_max_above: "A" × 101 → expects error
```

**Agent flow test derivation:**

For agent flows, the tool derives test cases from the agent's tools, guardrails, memory, and loop behavior:

```
Agent: extract-obligations
  Test 1 (tool_success): Agent calls extract_text tool → returns structured output
  Test 2 (tool_failure): Agent calls extract_text tool → tool returns error → agent retries or falls back
  Test 3 (guardrail_block): Agent generates output that violates guardrail → output is blocked, agent retries
  Test 4 (max_iterations): Agent reaches max_iterations without completing → returns partial result or error
  Test 5 (memory_persistence): Agent stores result in memory → subsequent call retrieves it
  Test 6 (human_gate): Agent reaches human gate → pauses for approval → continues after approval
  Test 7 (human_gate_reject): Agent reaches human gate → user rejects → agent handles rejection
```

**Orchestration flow test derivation:**

For orchestration flows, test cases come from routing rules, handoff behavior, and supervision strategies:

```
Orchestrator: support-pipeline (supervisor strategy)
  Test 1 (routing_intent_A): Input matches intent A → routed to Agent A → response returned
  Test 2 (routing_intent_B): Input matches intent B → routed to Agent B → response returned
  Test 3 (routing_fallback): Input matches no intent → fallback agent handles it
  Test 4 (handoff_transfer): Agent A transfers to Agent B → context is passed → Agent B continues
  Test 5 (handoff_consult): Agent A consults Agent B → gets response → Agent A continues with advice
  Test 6 (circuit_breaker): Agent fails repeatedly → circuit breaker opens → fallback activates
  Test 7 (supervisor_override): Supervisor detects stuck agent → intervenes → reassigns to different agent
```

**Test specification display:**

```
┌──────────────────────────────────────────────────────────────┐
│ TEST SPECIFICATION — api/user-register                        │
│                                                                │
│ Derived: 14 test cases from 3 paths + 11 boundary tests      │
│                                                                │
│ Paths:                                                        │
│   Path 1: Happy path — register new user           3 tests   │
│     • valid registration → 201 + user object                 │
│     • valid with min-length name → 201                       │
│     • valid with max-length name → 201                       │
│                                                                │
│   Path 2: Validation errors                         8 tests   │
│     • missing email → 422 + "Please enter a valid..."       │
│     • invalid email format → 422 + "Please enter..."        │
│     • missing password → 422 + "Password must be..."        │
│     • short password (7 chars) → 422 + "Password must..."   │
│     • exact min password (8 chars) → success (Path 1)        │
│     • missing name → 422 + "Name must be..."                │
│     • short name (1 char) → 422 + "Name must be..."         │
│     • long name (101 chars) → 422 + "Name must be..."       │
│                                                                │
│   Path 3: Duplicate email                           1 test    │
│     • register with existing email → 409 + DUPLICATE_ENTRY  │
│                                                                │
│   Boundary tests:                                    2 tests   │
│     • name exact min (2 chars) → success                     │
│     • name exact max (100 chars) → success                   │
│                                                                │
│ [Generate test code]  [Include in prompt]  [Export]          │
└──────────────────────────────────────────────────────────────┘
```

##### Level 2: Test Code Generation

The DDD Tool generates actual test code from the derived test specification. The generated code uses the project's testing framework (detected from `architecture.yaml`) and follows its test conventions.

**How it works:**

1. DDD Tool reads the test specification (Level 1) for the flow
2. It detects the test framework from architecture.yaml (pytest, jest, go test, etc.)
3. It generates test code using Claude Code, providing:
   - The test specification (paths + boundary tests)
   - The flow spec YAML (for exact error messages, status codes, field names)
   - The architecture.yaml testing section (for conventions, fixtures, patterns)
4. Generated tests are shown as a preview — user can edit before including in prompt

**Generated test code example (pytest):**

```python
# Auto-generated by DDD Tool from spec: api/user-register
# 14 test cases derived from 3 paths + 11 boundary tests

import pytest
from httpx import AsyncClient

class TestUserRegisterHappyPath:
    """Path 1: trigger → validate_input ✓ → check_duplicate (false) → create_user → return_success"""

    async def test_register_success(self, client: AsyncClient):
        response = await client.post("/auth/register", json={
            "email": "user@example.com",
            "password": "password123",
            "name": "Alice"
        })
        assert response.status_code == 201
        data = response.json()
        assert data["message"] == "User registered successfully"
        assert "id" in data["user"]
        assert data["user"]["email"] == "user@example.com"
        assert data["user"]["name"] == "Alice"

    async def test_register_min_length_name(self, client: AsyncClient):
        response = await client.post("/auth/register", json={
            "email": "user2@example.com",
            "password": "password123",
            "name": "Al"  # exact min_length boundary
        })
        assert response.status_code == 201

class TestUserRegisterValidation:
    """Path 2: trigger → validate_input ✗ → return_validation_error"""

    async def test_missing_email(self, client: AsyncClient):
        response = await client.post("/auth/register", json={
            "password": "password123",
            "name": "Alice"
        })
        assert response.status_code == 422
        # Exact error message from spec
        assert "Please enter a valid email address" in response.text

    async def test_short_password(self, client: AsyncClient):
        response = await client.post("/auth/register", json={
            "email": "user@example.com",
            "password": "1234567",  # 7 chars, min is 8
            "name": "Alice"
        })
        assert response.status_code == 422
        assert "Password must be at least 8 characters" in response.text

    # ... more validation tests

class TestUserRegisterDuplicate:
    """Path 3: trigger → validate_input ✓ → check_duplicate (true) → return_duplicate_error"""

    async def test_duplicate_email(self, client: AsyncClient, existing_user):
        response = await client.post("/auth/register", json={
            "email": existing_user.email,
            "password": "password123",
            "name": "Bob"
        })
        assert response.status_code == 409
        assert response.json()["error_code"] == "DUPLICATE_ENTRY"
        assert "A user with this email already exists" in response.text
```

**Including generated tests in the Claude Code prompt:**

When the user clicks "Include in prompt" or starts implementation, the generated test code is appended to the Claude Code prompt:

```markdown
## Pre-Generated Tests

The following tests were derived from the flow specification.
Implement the flow so that ALL these tests pass.
Do NOT modify the test assertions — they reflect the exact spec requirements.

```python
{generated test code}
`` `
```

This ensures Claude Code implements to match the spec exactly — the tests become executable spec requirements.

**Coverage badges on Level 2:**

After test generation, flow blocks show a spec coverage badge:

```
┌──────────────────────�
│ user-register        │
│ ✓ 14/14 tests       │    ← test badge (from test runner)
│ ~ 85% synced        │    ← sync badge (from reconciliation)
│ 📋 14 spec tests    │    ← spec test badge (from test generator)
└──────────────────────┘
```

The spec test badge shows how many test cases were derived from the diagram. If actual tests (from the test runner) cover fewer cases than the spec tests, the badge turns amber to indicate untested spec paths.

##### Level 3: Spec Compliance Validation

After implementation and test execution, the DDD Tool compares actual test results against what the flow spec says should happen. This catches subtle mismatches where tests pass but the behavior doesn't match the spec exactly.

**How it works:**

1. Tests run (from the Test Runner, section 4)
2. DDD Tool reads the test output and the derived test specification
3. The DDD Tool compares:
   - Expected outcomes from the spec (status codes, error messages, response shapes)
   - Actual test results (pass/fail, response bodies, error messages)
4. Produces a spec compliance report

**Spec compliance report:**

```
┌──────────────────────────────────────────────────────────────┐
│ SPEC COMPLIANCE — api/user-register                           │
│                                                                │
│ Compliance: 12/14 (86%)                                       │
│                                                                │
│ ✓ Compliant:                                                  │
│   • POST /auth/register returns 201 on success               │
│   • Response includes id, email, name fields                 │
│   • Missing email → 422 + correct message                    │
│   • Invalid email → 422 + correct message                    │
│   • Short password → 422 + correct message                   │
│   • Duplicate email → 409 + DUPLICATE_ENTRY                  │
│   • ... 6 more                                                │
│                                                                │
│ ⚠ Non-compliant:                                              │
│   • Spec says status 422 for short name, code returns 400    │
│     Expected: 422  Actual: 400                                │
│     [Fix via Claude Code]                                     │
│   • Spec says max name 100 chars, code allows 255            │
│     Expected: error at 101 chars  Actual: accepts 255        │
│     [Fix via Claude Code]                                     │
│                                                                │
│ [Fix all non-compliant]  [Dismiss]                           │
└──────────────────────────────────────────────────────────────┘
```

**"Fix via Claude Code" action:** Generates a targeted prompt:

```markdown
The following spec compliance issue was found in the user-register flow:

**Issue:** Spec requires status 422 for names shorter than 2 characters,
but the implementation returns status 400.

**Spec says:**
  Field: name, min_length: 2, error: "Name must be between 2 and 100 characters"
  Expected HTTP status: 422 (from errors.yaml VALIDATION_ERROR mapping)

**Code location:** src/domains/api/schemas.py (likely validation config)

Fix the implementation to return 422 instead of 400 for this validation error.
```

**Compliance history:**

Each compliance check is timestamped and stored in `.ddd/mapping.yaml`:

```yaml
flows:
  api/user-register:
    # ... existing fields ...
    test_generation:
      derived_test_count: 14
      paths_found: 3
      boundary_tests: 11
      last_generated_at: "2025-01-15T10:25:00Z"
    spec_compliance:
      score: 86
      compliant: 12
      non_compliant: 2
      last_checked_at: "2025-01-15T10:35:00Z"
      issues:
        - field: name
          expected_status: 422
          actual_status: 400
          resolved: false
```

##### Test Generation Configuration

```yaml
# .ddd/config.yaml — test generation settings
test_generation:
  auto_derive: true                # Auto-derive test spec when flow is saved
  include_in_prompt: true          # Include generated tests in Claude Code prompt
  compliance_check: true           # Run compliance check after tests pass

  boundary_tests:
    enabled: true                  # Generate boundary tests from validation rules
    include_type_errors: true      # Test wrong types (string where int expected)
    include_null_tests: true       # Test null/missing for required fields

  agent_tests:
    tool_failure: true             # Test tool failure scenarios
    guardrail_violation: true      # Test guardrail blocking
    max_iterations: true           # Test iteration limits
    memory_persistence: false      # Skip memory tests (complex setup)

  orchestration_tests:
    routing: true                  # Test routing rules
    handoff: true                  # Test handoff behavior
    circuit_breaker: true          # Test circuit breaker
    supervisor: false              # Skip supervisor tests (complex setup)
```

---

# Part 7b: Validation Rules

The DDD Tool enforces these validation rules. Specs should pass all of them.

## Flow-Level (Error)
- Every flow must have exactly one trigger node
- All paths from trigger must reach a terminal node
- No orphaned (unreachable) nodes
- No circular paths in traditional flows (agents may have cycles)
- Decision nodes must have both true and false branch connections
- Trigger must have an `event` defined
- Input fields must have `type` defined
- Decision must have a `condition` defined

## Flow-Level (Warning)
- Terminal nodes should not have outgoing connections
- Process nodes should have a description or action
- Agent loops should have `max_iterations` and `model` set
- Sub-flow `input_mapping` keys should match target flow's contract inputs (if contract defined)
- Sub-flow `output_mapping` keys should match target flow's contract outputs (if contract defined)

## Agent-Specific (Error)
- Agent flow must have at least one `agent_loop` node
- Agent loop must have at least one tool
- Agent loop must have at least one terminal tool (`is_terminal: true`)

## Orchestration (Error)
- Orchestrator must have 2+ agents and a strategy
- Smart Router must have rules defined or LLM routing enabled
- Handoff must have a target flow
- Agent Group must have 2+ members

## Extended Nodes (Error)
- Data Store must have operation and model
- Service Call must have method and URL
- Event must have direction and event_name
- Loop must have collection and iterator
- Parallel must have 2+ branches
- Sub-flow must have flow_ref
- LLM Call must have model
- Cache must have key and store
- Transform must have input_schema and output_schema
- Delay must have min_ms
- Collection must have operation and input
- Parse must have format and input
- Crypto must have operation, algorithm, and key_source
- Batch must have input and operation_template
- Transaction must have steps with at least 2 entries

## Domain-Level
- No duplicate flow IDs within a domain
- Multiple flows may publish the same event name with different `from_flow` values — this is valid

## System-Level
- Events consumed by a domain must be published by some domain
- Events published should be consumed by at least one domain (warning)
- Event `payload` fields should match between publisher and consumer across domains (warning)

---

# Part 7c: Design Patterns

Common patterns and conventions for DDD specs.

## HTTP Request/Response Approval

For synchronous approval flows (e.g., "admin approves a pending request"), use an HTTP trigger + decision node — **not** a human_gate. Human gates are for async agent workflows.

## Guardrail Execution Model

Guardrails are **inline and sequential** — they sit in the connection chain and data flows through them in order. They are NOT sidecars or parallel watchers.

```
trigger -> guardrail(input) -> agent_loop -> guardrail(output) -> terminal
```

## Error Routing Convention

For nodes with dual output paths, the convention is:
- `"success"` / `"error"` for data_store and service_call
- `"valid"` / `"invalid"` for input
- `"true"` / `"false"` for decision

## Multi-Way Routing in Traditional Flows

Decision nodes are binary (true/false). When you need 3+ branches, use `smart_router` — it works in traditional flows too:

```yaml
- id: router-001
  type: smart_router
  spec:
    rules:
      - id: twitter
        condition: "platform === 'twitter'"
        route: twitter
      - id: linkedin
        condition: "platform === 'linkedin'"
        route: linkedin
    fallback_chain: [linkedin]
```

## Parameterized Flow

Define a flow template once, instantiate per-configuration in domain.yaml:

```yaml
# Flow YAML with template: true
flow:
  id: publish-to-platform
  template: true
  parameters:
    platform:
      type: string
      values: [twitter, linkedin]
    api_integration:
      type: integration_ref

# In domain.yaml — instantiate:
flows:
  - id: publish-to-twitter
    template: publish-to-platform
    parameters:
      platform: twitter
      api_integration: twitter-api-v2
```

## Collection Pipeline

Process a collection through multiple stages: fetch → parse → filter → iterate → reduce → emit. Use `collection`, `parse`, and `loop` nodes instead of generic `process` nodes.

## Trigger Event Filtering

Use the `filter` field on trigger nodes to filter incoming events by payload fields, eliminating unnecessary decision nodes:

```yaml
trigger:
  spec:
    event: "event:DraftApproved"
    filter:
      platform: twitter
    description: Only triggers when payload.platform is "twitter"
```

## Cross-Cutting Layers

Define infrastructure layers (retry policies, stealth config, circuit breakers) in `specs/shared/layers.yaml` and reference them from nodes. See Usage Guide Section 4.7 for the full layers.yaml format.

---

# Part 7d: Implementation Patterns

This section maps DDD spec constructs to implementation code artifacts.

## Node Type to Code Artifact Mapping

| Node Type | Primary Artifact | Secondary Artifacts |
|-----------|-----------------|---------------------|
| `trigger` (http) | Route handler / controller | Auth middleware, rate limiter |
| `trigger` (cron) | Scheduled job definition | — |
| `trigger` (event) | Event listener / subscriber | — |
| `input` | Zod/Pydantic validation schema | TypeScript request type |
| `process` | Service function | — |
| `decision` | Conditional branch in service | — |
| `terminal` | Response formatter / return | Error response variant |
| `data_store` | Repository / query function | ORM query |
| `collection` | Array utility in service | — |
| `service_call` | HTTP client call / SDK wrapper | Retry/timeout config |
| `event` (emit) | Event emitter call | Event type definition |
| `loop` | `for`/`while` block in service | — |
| `parallel` | `Promise.all` / `asyncio.gather` | — |
| `sub_flow` | Imported service function call | — |
| `parse` | Parser utility function | — |
| `crypto` | Crypto utility function | Key management helper |
| `transform` | Data mapping function | — |
| `delay` | `setTimeout` / queue delay | — |
| `llm_call` | LLM client call | Prompt template |
| `agent_loop` | Agentic tool-use loop | Tool definitions |
| `guardrail` | Check middleware / validator | — |
| `human_gate` | Async checkpoint + notification | Approval state persistence |
| `orchestrator` | Strategy dispatcher | Per-strategy handler |
| `smart_router` | Routing function | Confidence scorer |
| `handoff` | Context transfer + agent call | — |
| `agent_group` | Parallel/sequential agent runner | — |

## Service Layer Pattern

Each flow maps to one **service file**. The service exports a main function named after the flow. Nodes become sequential or branching calls within that function.

**Key rules:**
- One service file per flow, one exported function per flow
- Internal helper functions for process/decision nodes stay private
- Repository functions are shared across flows via the `repositories/` directory
- When multiple flows share a domain, group their HTTP routes into a single route file per resource

## Error Handling

Connection `sourceHandle` values map to control flow:

| Source Handle | Implementation |
|--------------|----------------|
| (unnamed) | Normal sequential call |
| `"success"` | Success branch (equivalent to unnamed) |
| `"error"` | `catch` block / error handler |
| `"true"` / `"false"` | `if`/`else` branches |

Connection `behavior` maps to error handling strategy:

| Behavior | Implementation |
|----------|----------------|
| `continue` | `try { ... } catch(e) { logger.warn(e); }` — log and proceed |
| `stop` | `throw e;` — propagate to flow-level error handler |
| `retry` | Wrap call in retry loop |
| `circuit_break` | Use circuit breaker with threshold from node spec |

## Test Generation

Generate tests based on flow structure:

| Flow Element | Test Case |
|-------------|-----------|
| Happy path (trigger → terminal success) | End-to-end success test |
| Each `decision` node | One test per branch |
| Each `error` handle connection | Error path test |
| `input` node with `required` fields | Validation rejection test per field |
| `service_call` node | Mock external call, test success + failure |
| `data_store` node | Mock repository, verify query/mutation |
| `guardrail` node | Test pass and block outcomes |
| `human_gate` node | Test approve and reject paths |

---

# Part 8: Workflows

## Complete Workflow: Idea to Deployment

### Phase 1: Ideation (Claude Chat)
```
Input: Natural language app idea
Process: Claude asks clarifying questions, outputs DDD specs
Output: Folder structure with YAML specs (completeness: partial)
```

### Phase 2: Visual Refinement (DDD Tool)
```
Input: Claude's specs
Process: Import, visualize, fill missing details via UI, run expert agents
Output: Refined specs with 100% completeness
```

### Phase 3: Implementation (Claude Code)
```
Input: Refined specs
Process: Read specs, implement flows, validate, run tests
Output: Working code matching specs exactly
```

### Phase 4: Reflect
```
Input: Implemented code + specs
Process: /ddd-sync (alignment check), /ddd-reflect (capture wisdom),
         /ddd-promote (move patterns into specs), then validate & deploy
Output: Enriched specs + deployed application
```

## Git-Based Sync Workflow

```
1. YOU EDIT SPEC IN DDD TOOL
   └─► DDD Tool writes to specs/*.yaml

2. COMMIT SPEC CHANGES
   └─► git add specs/
   └─► git commit -m "Design: Add rate limiting"

3. CLAUDE CODE IMPLEMENTS
   └─► claude "Implement pending spec changes"
   └─► Claude reads specs/, generates code in src/

4. VALIDATE
   └─► ddd validate
   └─► pytest

5. COMMIT CODE CHANGES
   └─► git add src/ tests/
   └─► git commit -m "Implement: Add rate limiting"

6. PUSH
   └─► git push
   └─► CI validates spec-code sync
```

---

# Part 9: Reusability System

## Levels of Reuse

### Level 1: Sub-Flows (Within Project)
Reuse entire flows as callable units:
- `send-email` called from multiple flows
- Interface defines inputs/outputs

### Level 2: Shared Components (Within Project)
- **Node Templates:** `validate_webhook_signature`, `retry_with_backoff`
- **Validation Rules:** Standard email, password, phone
- **Schemas:** `PaginatedResponse<T>`, `ApiError`, `AuditLog`

### Level 3: Project Templates (Within Project)
Parameterized flow generators:
- `CRUD_API<Entity>` → 5 flows: list, get, create, update, delete
- `WEBHOOK_RECEIVER<Provider>` → Signature validation, parsing, dedup
- `EVENT_WORKER<Event>` → Subscription, retry, DLQ, idempotency

### Level 4: Personal Library (Across Projects)
Save patterns for reuse:
- `my-jwt-auth`, `my-stripe-checkout`, `my-s3-upload`
- Parameterization for customization
- Linked (get updates) or Copied (full control)

### Level 5: Community Library (Global)
Public marketplace:
- Pre-built integrations (Stripe, Auth0, SendGrid)
- Starter templates (SaaS, E-Commerce, API-First)

---

# Part 10: Production Infrastructure

The DDD Tool auto-generates production infrastructure artifacts from the specs that already exist. Flow specs contain endpoint definitions, schemas, error codes, and validation rules — enough to derive OpenAPI docs, CI pipelines, migration scripts, observability config, security middleware, and deployment manifests. These artifacts regenerate when specs change, keeping infrastructure in sync with design.

## 10.1 OpenAPI Generation

Flow specs already define: HTTP method, path, request fields with types and validation, response shapes, and error codes with HTTP status mappings. The DDD Tool maps these directly to an OpenAPI 3.0 specification.

**What gets generated:**

```yaml
# Generated: openapi.yaml (project root)
openapi: 3.0.3
info:
  title: Obligo API
  version: 1.0.0
  description: Cyber Liability Operating System

paths:
  /auth/register:                    # ← from trigger.path
    post:                            # ← from trigger.method
      operationId: user-register     # ← from flow.id
      summary: User Registration     # ← from flow.name
      tags: [api]                    # ← from flow.domain
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/UserRegisterRequest'
      responses:
        '201':                       # ← from terminal node status
          description: User registered successfully
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/UserRegisterResponse'
        '409':                       # ← from errors.yaml DUPLICATE_ENTRY
          description: A user with this email already exists
        '422':                       # ← from errors.yaml VALIDATION_ERROR
          description: Validation error

components:
  schemas:
    UserRegisterRequest:             # ← from input node fields
      type: object
      required: [email, password, name]
      properties:
        email:
          type: string
          format: email
        password:
          type: string
          minLength: 8
        name:
          type: string
          minLength: 2
          maxLength: 100
    UserRegisterResponse:            # ← from terminal node body
      type: object
      properties:
        message:
          type: string
        user:
          $ref: '#/components/schemas/User'
```

**Generation rules:**
- One path entry per flow with an HTTP trigger
- Request schemas derived from `input` node fields (type, required, min/max/format)
- Response schemas derived from `terminal` node body shapes
- Error responses derived from error codes used in the flow, mapped via `errors.yaml`
- Schema `$ref` links to `specs/schemas/*.yaml` definitions
- Non-HTTP flows (event, scheduled) are excluded from OpenAPI
- Agent flows with HTTP triggers are included (the trigger is still HTTP)

**When it regenerates:**
- Any flow with HTTP trigger is created, modified, or deleted
- Error codes change in `errors.yaml`
- Schemas change in `specs/schemas/`

**Preview:** The generated OpenAPI spec can be viewed with any OpenAPI renderer (e.g., Swagger UI, Redoc) or served alongside the application.

---

## 10.2 CI/CD Pipeline Generation

Architecture.yaml defines the tech stack, test commands, lint commands, and deployment config. The DDD Tool generates a GitHub Actions workflow (with templates for GitLab CI and others).

**What gets generated:**

```yaml
# Generated: .github/workflows/ci.yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:                                  # ← from specs/config.yaml (non-secret defaults)
  DATABASE_URL: postgresql://test:test@localhost:5432/test
  REDIS_URL: redis://localhost:6379

jobs:
  test:
    runs-on: ubuntu-latest

    services:                         # ← from architecture.yaml database/cache config
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
          POSTGRES_DB: test
        ports: [5432:5432]
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7
        ports: [6379:6379]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.11     # ← from system.yaml language + version
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: pip install -e ".[dev]"

      - name: Run migrations
        run: alembic upgrade head     # ← from architecture.yaml migration command

      - name: Lint
        run: ruff check .             # ← from architecture.yaml lint command

      - name: Type check
        run: mypy src/                # ← from architecture.yaml typecheck command

      - name: Test
        run: pytest --tb=short -q     # ← from architecture.yaml test command

      - name: Spec-code sync check    # ← DDD-specific validation
        run: |
          python -c "
          import yaml, hashlib, sys
          mapping = yaml.safe_load(open('.ddd/mapping.yaml'))
          stale = []
          for flow_id, m in mapping.get('flows', {}).items():
              current = hashlib.sha256(open(m['spec'], 'rb').read()).hexdigest()
              if current != m['spec_hash']:
                  stale.append(flow_id)
          if stale:
              print(f'Stale flows: {stale}')
              sys.exit(1)
          print('All flows in sync')
          "
```

**Generation rules:**
- Language setup step derived from `system.yaml` tech stack
- Service containers derived from `architecture.yaml` database/cache/queue config
- Commands derived from `architecture.yaml` testing/linting sections
- Spec-code sync validation step always included (checks mapping hashes)
- Deployment steps added if `architecture.yaml` has deployment config

**When it regenerates:**
- Tech stack changes in `system.yaml`
- Test/lint commands change in `architecture.yaml`
- Database/cache config changes in `architecture.yaml`

---

## 10.3 Database Migration Tracking

Schema specs (`specs/schemas/*.yaml`) define data models. When a schema changes, the DDD Tool detects it (like flow stale detection) and includes migration instructions in the Claude Code prompt.

**How it works:**

Schema hashes tracked in `.ddd/mapping.yaml`:

```yaml
schemas:
  User:
    spec: specs/schemas/user.yaml
    spec_hash: def456...
    migration_history:
      - version: "001"
        description: "Initial User table"
        generated_at: "2025-01-10T10:00:00Z"
      - version: "002"
        description: "Add email_verified column"
        generated_at: "2025-01-20T14:00:00Z"
    last_synced_at: "2025-01-20T14:00:00Z"
```

**Schema change detection:**

On Level 2 (Domain Map), when a schema used by flows in this domain has changed:

```
┌──────────────────────────────────┐
│ ⚠ Schema changed: User           │
│ • Added: email_verified (boolean) │
│ • Changed: name max_length 100→200│
│                                    │
│ [Generate migration]  [Dismiss]   │
└──────────────────────────────────┘
```

**Migration prompt addition:** When "Generate migration" is clicked, the Prompt Builder adds:

```markdown
## Migration Required

The User schema has changed since last migration:
- Added field: email_verified (boolean, default: false)
- Changed field: name max_length from 100 to 200

Generate a database migration:
- Use alembic (from architecture.yaml ORM config)
- Migration should be reversible (include downgrade)
- Do NOT modify existing data — add columns as nullable or with defaults
```

**Schema diff:** Like flow stale detection, schema specs are cached at implementation time. The DDD Tool computes a human-readable diff showing added/removed/changed fields.

---

## 10.4 Observability

A new `cross_cutting.observability` section in `architecture.yaml` defines the logging, tracing, metrics, and health check strategy. Claude Code reads this and generates the observability infrastructure alongside business logic.

**Architecture.yaml section:**

```yaml
cross_cutting:
  observability:
    logging:
      format: json                    # json | text
      level: info                     # debug | info | warn | error
      fields:                         # Fields included in every log line
        - timestamp
        - level
        - message
        - request_id
        - user_id
        - domain
        - flow_id
        - duration_ms
      sensitive_fields:               # Fields to redact in logs
        - password
        - token
        - api_key
        - credit_card
      library: structlog              # structlog | loguru | stdlib (Python)

    tracing:
      enabled: true
      provider: opentelemetry
      exporter: otlp                  # otlp | jaeger | zipkin
      endpoint: ${OTEL_EXPORTER_ENDPOINT}
      sample_rate: 1.0                # 1.0 = trace everything, 0.1 = 10%
      propagation: w3c                # w3c | b3
      auto_instrument:                # Auto-instrument these libraries
        - fastapi
        - sqlalchemy
        - httpx
        - redis

    metrics:
      enabled: true
      provider: prometheus
      endpoint: /metrics              # Prometheus scrape endpoint
      custom_metrics:                 # Business metrics defined per flow
        - name: user_registrations_total
          type: counter
          description: Total user registrations
          labels: [status, domain]
        - name: document_processing_duration_seconds
          type: histogram
          description: Time to process a document
          labels: [document_type]
          buckets: [0.1, 0.5, 1, 5, 10, 30, 60]

    health:
      readiness:
        path: /health/ready
        checks:                       # What to check for readiness
          - database
          - redis
          - migrations
      liveness:
        path: /health/live
        checks:
          - process
```

**What Claude Code generates from this:**
- A `src/shared/logging.py` module with structured logging configured per the spec
- Middleware that adds `request_id`, `user_id`, `domain`, `flow_id` to every log line
- Sensitive field redaction in log output
- OpenTelemetry setup with auto-instrumentation for configured libraries
- Trace context propagation through async calls
- Prometheus metrics endpoint and custom metric definitions
- Health check endpoints (readiness checks database/cache, liveness checks process)
- Per-flow logging: every flow node logs entry/exit with duration

**Flow-level observability:** Each flow node gets automatic logging:

```python
# Auto-generated for every node in a flow
logger.info("node.start", node_id="validate_input", flow_id="user-register")
# ... node logic ...
logger.info("node.complete", node_id="validate_input", flow_id="user-register", duration_ms=12)
```

This means flow execution is traceable through logs without the developer adding any logging code.

**Custom metrics per flow:** Flow specs can optionally define metrics:

```yaml
# In a flow spec
flow:
  id: user-register
  metrics:
    - name: user_registrations_total
      on: terminal          # Increment when terminal node reached
      labels:
        status: "$.response.status"
```

**DDD Tool integration:** Observability configuration is defined in architecture.yaml and applied by Claude Code during implementation.

---

## 10.5 Security Layer

A new `cross_cutting.security` section in `architecture.yaml` defines security policies that Claude Code applies to every generated endpoint.

**Architecture.yaml section:**

```yaml
cross_cutting:
  security:
    rate_limiting:
      enabled: true
      default: 100/minute             # Default for all endpoints
      per_endpoint:                    # Override per flow
        user-register: 5/minute       # Tight limit on registration
        user-login: 10/minute         # Tight limit on login
      storage: redis                   # Where to store counters
      response:
        status: 429
        body:
          error_code: RATE_LIMITED
          message: "Too many requests. Try again later."

    cors:
      allowed_origins:
        - ${CORS_ORIGIN}              # From environment
      allowed_methods: [GET, POST, PUT, DELETE, OPTIONS]
      allowed_headers: [Authorization, Content-Type]
      max_age: 3600

    headers:
      strict_transport_security: "max-age=31536000; includeSubDomains"
      content_security_policy: "default-src 'self'"
      x_content_type_options: nosniff
      x_frame_options: DENY
      x_xss_protection: "1; mode=block"

    input_sanitization:
      trim_strings: true              # Trim whitespace from all string inputs
      strip_html: true                # Remove HTML tags from string inputs
      max_string_length: 10000        # Reject strings longer than this

    audit_logging:
      enabled: true
      events:                         # What actions to audit
        - user.login
        - user.login_failed
        - user.register
        - user.password_reset
        - data.create
        - data.update
        - data.delete
        - admin.*
      storage: database               # database | file | external
      retention_days: 90
      fields:
        - timestamp
        - actor_id
        - actor_ip
        - action
        - resource_type
        - resource_id
        - changes                     # What changed (for updates)
        - result                      # success | failure

    dependency_scanning:
      enabled: true
      tool: safety                    # safety (Python) | npm audit (Node) | cargo audit (Rust)
      ci_step: true                   # Add to CI pipeline
      fail_on: high                   # Fail CI on: critical | high | medium | low
```

**What Claude Code generates from this:**
- Rate limiting middleware using configured storage and limits
- Per-endpoint rate limit overrides matching flow IDs
- CORS middleware with configured origins/methods/headers
- Security headers middleware
- Input sanitization middleware (trim, strip HTML, max length)
- Audit log model (database table or file logger)
- Audit log middleware that records configured events
- Dependency scanning step in CI pipeline

**Flow-level security:** Flow specs can reference security policies:

```yaml
# In a flow spec
trigger:
  type: http
  method: POST
  path: /auth/register
  security:
    rate_limit: 5/minute             # Override default
    auth: false                       # Public endpoint (no JWT required)
    audit: user.register              # Audit event name
```

**DDD Tool integration:** Security configuration is defined in architecture.yaml and applied by Claude Code during implementation. Flow-level security overrides (rate limits, auth requirements, audit events) are specified in the trigger node's spec.

---

## 10.6 Deployment / Infrastructure as Code

A new `deployment` section in `architecture.yaml` defines how the application is deployed. The DDD Tool generates Dockerfile, docker-compose, and optionally Kubernetes manifests.

**Architecture.yaml section:**

```yaml
deployment:
  docker:
    base_image: python:3.11-slim     # From system.yaml language + version
    port: 8000
    env_file: .env
    multi_stage: true                 # Use multi-stage build for smaller image
    healthcheck:
      path: /health/live             # From observability.health config
      interval: 30s
      timeout: 5s

  compose:
    services:
      app:
        build: .
        ports: ["8000:8000"]
        depends_on: [postgres, redis]
      postgres:
        image: postgres:15
        volumes: ["pgdata:/var/lib/postgresql/data"]
        environment:
          POSTGRES_DB: obligo
          POSTGRES_USER: obligo
          POSTGRES_PASSWORD: ${DB_PASSWORD}
      redis:
        image: redis:7-alpine

  kubernetes:
    enabled: true
    namespace: obligo
    replicas: 2
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
    ingress:
      host: api.obligo.com
      tls: true
      cert_issuer: letsencrypt
    hpa:
      min_replicas: 2
      max_replicas: 10
      target_cpu: 70
```

**What gets generated:**

```dockerfile
# Generated: Dockerfile
FROM python:3.11-slim AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install --no-cache-dir .

FROM python:3.11-slim
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.11/site-packages /usr/local/lib/python3.11/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY src/ src/
COPY alembic/ alembic/
COPY alembic.ini .
EXPOSE 8000
HEALTHCHECK --interval=30s --timeout=5s CMD curl -f http://localhost:8000/health/live || exit 1
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Plus `docker-compose.yaml`, and if Kubernetes is enabled: `k8s/deployment.yaml`, `k8s/service.yaml`, `k8s/ingress.yaml`, `k8s/hpa.yaml`.

**When they regenerate:**
- Tech stack changes in `system.yaml`
- Deployment config changes in `architecture.yaml`
- New services added (database, cache, queue)
- Health check config changes in observability section

**DDD Tool integration:** A "Deployment" tab in the Sidebar shows the generated artifacts with a preview. The user can click "Regenerate" after changing architecture.yaml.

---

## Architecture.yaml Cross-Cutting Summary

After adding observability, security, and deployment, the `cross_cutting` section of architecture.yaml controls:

```yaml
cross_cutting:
  auth:            # JWT/sessions, RBAC roles (existing)
  multi_tenancy:   # Tenant isolation (existing)
  observability:   # Logging, tracing, metrics, health (new)
  security:        # Rate limiting, CORS, headers, audit, scanning (new)

deployment:        # Docker, compose, Kubernetes (new)
```

Claude Code reads all of these sections and generates the corresponding infrastructure code. The DDD Tool tracks these as generated artifacts (like flows) with hash-based change detection.

---

## Generated Artifacts Summary

| Artifact | Generated From | File(s) | Regenerates When |
|----------|---------------|---------|-----------------|
| OpenAPI spec | Flow specs + errors.yaml + schemas | `openapi.yaml` | Flow HTTP triggers change |
| CI/CD pipeline | architecture.yaml + system.yaml | `.github/workflows/ci.yaml` | Tech stack or commands change |
| Database migrations | Schema specs | `alembic/versions/*.py` | Schema fields change |
| Logging config | architecture.yaml observability | `src/shared/logging.py` | Logging config changes |
| Tracing setup | architecture.yaml observability | `src/shared/tracing.py` | Tracing config changes |
| Metrics definitions | architecture.yaml + flow specs | `src/shared/metrics.py` | Metrics config changes |
| Health checks | architecture.yaml observability | `src/shared/health.py` | Health config changes |
| Rate limiting | architecture.yaml security | `src/shared/rate_limit.py` | Rate limit rules change |
| Security middleware | architecture.yaml security | `src/shared/security.py` | Security config changes |
| Audit logging | architecture.yaml security | `src/shared/audit.py` | Audit config changes |
| Dockerfile | architecture.yaml deployment | `Dockerfile` | Deployment config changes |
| Docker Compose | architecture.yaml deployment | `docker-compose.yaml` | Services change |
| K8s manifests | architecture.yaml deployment | `k8s/*.yaml` | K8s config changes |
| CLAUDE.md | All specs + implementation status | `CLAUDE.md` | Any spec or status changes |
| Mermaid diagrams | Flow specs (nodes + connections) | `generated/mermaid/{domain}/{flow}.md` | Flow nodes or connections change |

### Mermaid Generator

The DDD Tool generates Mermaid flowchart diagrams from flow specs. Each flow produces a markdown file containing a Mermaid code block with the flow's node graph.

**Generated format:**

```markdown
# user-register

```mermaid
flowchart TD
  user_register_trigger(("HTTP POST /users"))
  user_register_input["Input: email, password, name"]
  user_register_trigger --> user_register_input
  user_register_process_0["Hash Password"]
  user_register_input --> user_register_process_0
  user_register_terminal_0[/"Success"/]
  user_register_process_0 --> user_register_terminal_0
`` `
```

**Output location:** `generated/mermaid/{domain}/{flow}.md`

The Mermaid export utility generates diagrams from flow specs and can be triggered from the DDD Tool or via Claude Code commands.

---

# Part 11: Technical Decisions

## Decision: Git over MCP for Sync

**Chosen:** Git-based sync
**Rejected:** MCP-based real-time sync

**Reasoning:**
- MCP would duplicate state that Git already tracks
- Git provides versioning, branching, merging for free
- No extra infrastructure to maintain
- Claude Code already understands Git
- Simpler architecture

## Decision: YAML for Specs

**Chosen:** YAML
**Alternatives:** JSON, custom DSL

**Reasoning:**
- Human-readable and editable
- Git-friendly (meaningful diffs)
- Supports comments
- Well-supported in all languages
- $ref resolution straightforward

## Decision: Tauri for Desktop App

**Chosen:** Tauri
**Alternative:** Electron

**Reasoning:**
- Smaller bundle size
- Better performance
- Rust backend for file system access
- Native Git integration via libgit2

## Decision: Spec-to-Code Mapping

**Chosen:** Explicit mapping file (.ddd/mapping.yaml)
**Alternative:** Convention-based (infer from names)

**Reasoning:**
- Allows restructuring code without breaking sync
- Explicit > implicit for validation
- Supports incremental adoption

---

# Part 12: Open Questions / Future Work

## MVP Scope (v0.1)
**Included:**
- Multi-level canvas (System Map → Domain Map → Flow Sheet)
- Breadcrumb navigation between levels
- Canvas + 19 traditional node types for traditional flows (Level 3)
- Agent flow support: Agent Loop, Tool, LLM Call, Memory, Guardrail, Human Gate, Router nodes
- Orchestration support: Orchestrator, Smart Router, Handoff, Agent Group nodes
- Agent-centric canvas layout for agent flows
- Orchestration visualization on Level 2 (supervisor arrows, handoff arrows, group boundaries)
- Auto-generated System Map and Domain Map (Levels 1-2)
- Portal nodes for cross-domain navigation
- Spec panel (basic fields + agent-specific + orchestration panels)
- YAML export (traditional, agent, and orchestration flow formats)
- **Claude Code Integration:** Drift detection with stale banners, "Copy Command" for CLI-driven implementation via `/ddd-implement`
- **Prompt Builder:** Auto-constructs optimal prompts from specs (schema resolution, agent detection, update mode)
- **Stale Detection:** SHA-256 hash comparison to detect spec-vs-code drift with human-readable change summaries
- **Test Runner:** Auto-runs tests after implementation, displays results linked to flows, fix-and-retest loop
- **CLAUDE.md Auto-Generation:** Maintains CLAUDE.md with project structure, spec files, domains, rules (preserves custom section)
- **Implementation Queue:** Batch processing of pending/stale flows with sequential Claude Code invocation
- **Reverse Drift Detection:** Implementation Report parsing, code→spec reconciliation, sync scores, accept/remove/ignore actions
- **Design Validation:** Flow-level (graph completeness, spec completeness, reference integrity), domain-level (duplicate detection, internal event matching), system-level (cross-domain event wiring, payload shape matching, portal integrity, orchestration cycle detection), real-time canvas indicators, implementation gate
- **Entity Management:** Add/rename/delete domains (L1), add/rename/delete/duplicate/move flows (L2), rename/clear flows (L3) — all via right-click context menu with file operations and cross-reference updates
- **Extended Node Types:** 6 additional traditional flow nodes (data_store, service_call, event, loop, parallel, sub_flow) with spec editors, dual output handles (sourceHandle routing), and validation rules
- **Flow Templates:** Pre-built flow templates for common patterns — 4 traditional (REST API Endpoint, CRUD Entity, Webhook Handler, Event Processor) + 4 agent (RAG Agent, Customer Support Agent, Code Review Agent, Data Pipeline Agent) — insert fully wired node graphs with one click
- **Mermaid Export:** Generate Mermaid flowchart diagrams from flow specs, output as markdown files with embedded Mermaid code blocks
- **Minimap Toggle:** Cmd+Shift+M to toggle React Flow minimap on the flow canvas
- **Validation Presets:** InputNode spec editor includes presets for common field patterns (username, email, password)
- **Claude Code Commands:** 4 slash commands (`/ddd-create`, `/ddd-implement`, `/ddd-update`, `/ddd-sync`) for the full design-implement-iterate-sync workflow
- **Project Launcher:** Create new project, open existing, import from Git, recent projects list
- **Settings Screen:** Global + per-project settings for Claude Code, testing, editor preferences, Git
- **First-Run Experience:** Guided setup wizard (detect Claude Code, create/open/sample project)
- **Error Handling:** Defined recovery for file, Git, PTY, and canvas errors with auto-retry, fallback, and crash recovery
- **Undo/Redo:** Per-flow command pattern with immutable snapshots (Cmd+Z / Cmd+Shift+Z), 100-level history
- Single user, local storage

**Excluded (v0.2+):**
- Real-time collaboration
- Reverse engineering
- Community library

## Potential Extensions

1. **VS Code Extension:** Spec editing without full app
2. **GitHub Action:** Spec-code sync validation in CI
3. **MCP Server:** For Claude Code convenience tools (not sync)
4. **Web Version:** For quick viewing/editing without desktop app
5. **API:** For programmatic spec manipulation

## Open Technical Questions

1. How to handle partial implementations?
2. Schema migration generation from spec changes?
3. Multi-language support (Python + TypeScript in same project)?
4. Spec inheritance for similar flows?

---

# Part 13: Key Files Reference

## Project Structure (Complete)

```
obligo/
├── .git/
├── .ddd/
│   ├── config.yaml           # DDD tool config
│   ├── mapping.yaml          # Spec-to-code mapping
│   └── templates/            # Project-specific templates
│
├── specs/                    # ═══ ALL SPECS LIVE HERE ═══
│   │
│   ├── system.yaml           # Project identity, tech stack, domains
│   ├── system-layout.yaml    # System Map (L1) positions (managed by DDD Tool)
│   ├── architecture.yaml     # Structure, infrastructure, cross-cutting
│   ├── config.yaml           # Environment variables schema
│   │
│   ├── schemas/
│   │   ├── _base.yaml        # Base model (inherited by all)
│   │   ├── contract.yaml
│   │   ├── obligation.yaml
│   │   ├── user.yaml
│   │   ├── tenant.yaml
│   │   └── events/
│   │       ├── contract-ingested.yaml
│   │       └── obligation-extracted.yaml
│   │
│   ├── shared/
│   │   ├── auth.yaml         # Auth flow specification
│   │   ├── errors.yaml       # Error codes and formats
│   │   ├── middleware.yaml   # Middleware stack
│   │   └── api.yaml          # API conventions
│   │
│   └── domains/
│       ├── ingestion/
│       │   ├── domain.yaml      # Flows, events, L2 layout
│       │   └── flows/
│       │       ├── webhook-ingestion.yaml
│       │       └── scheduled-sync.yaml
│       ├── analysis/
│       │   ├── domain.yaml      # Flows, events, L2 layout
│       │   └── flows/
│       │       ├── extract-obligations.yaml
│       │       └── classify-risk.yaml
│       ├── api/
│       │   ├── domain.yaml      # Flows, events, L2 layout
│       │   └── flows/
│       │       ├── list-contracts.yaml
│       │       ├── get-contract.yaml
│       │       └── update-obligation.yaml
│       ├── notification/
│       │   ├── domain.yaml      # Flows, events, L2 layout
│       │   └── flows/
│       │       ├── send-email.yaml
│       │       └── send-slack.yaml
│       └── support/
│           ├── domain.yaml      # Flows, events, L2 layout
│           └── flows/
│               ├── customer-support-agent.yaml    # type: agent
│               ├── billing-agent.yaml             # type: agent
│               └── technical-agent.yaml           # type: agent
│
├── src/                      # ═══ GENERATED CODE ═══
│   ├── main.py               # FastAPI app entry point
│   ├── config/
│   │   ├── __init__.py
│   │   ├── settings.py       # Pydantic settings
│   │   └── logging.py        # Logging configuration
│   │
│   ├── shared/
│   │   ├── __init__.py
│   │   ├── auth/
│   │   │   ├── __init__.py
│   │   │   ├── jwt.py
│   │   │   ├── dependencies.py
│   │   │   └── permissions.py
│   │   ├── database/
│   │   │   ├── __init__.py
│   │   │   ├── connection.py
│   │   │   ├── base_model.py
│   │   │   └── mixins.py
│   │   ├── middleware/
│   │   │   ├── __init__.py
│   │   │   ├── request_id.py
│   │   │   ├── logging.py
│   │   │   ├── error_handler.py
│   │   │   ├── rate_limiter.py
│   │   │   └── tenant_context.py
│   │   ├── exceptions/
│   │   │   ├── __init__.py
│   │   │   ├── base.py
│   │   │   └── handlers.py
│   │   ├── events/
│   │   │   ├── __init__.py
│   │   │   ├── bus.py
│   │   │   └── handlers.py
│   │   └── utils/
│   │       ├── __init__.py
│   │       ├── pagination.py
│   │       └── filtering.py
│   │
│   └── domains/
│       ├── ingestion/
│       │   ├── __init__.py
│       │   ├── router.py
│       │   ├── schemas.py
│       │   ├── services.py
│       │   ├── models.py
│       │   ├── events.py
│       │   └── exceptions.py
│       ├── analysis/
│       │   └── ...
│       ├── api/
│       │   └── ...
│       └── notification/
│           └── ...
│
├── tests/
│   ├── conftest.py           # Shared fixtures
│   ├── factories/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── contract.py
│   │   └── obligation.py
│   ├── unit/
│   │   └── domains/
│   │       ├── ingestion/
│   │       └── ...
│   ├── integration/
│   │   └── ...
│   └── e2e/
│       └── ...
│
├── migrations/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
│       └── ...
│
├── scripts/
│   ├── seed.py
│   └── migrate.py
│
├── docker/
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   └── docker-compose.yml
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
├── pyproject.toml            # Dependencies and tool config
├── CLAUDE.md                 # Instructions for Claude Code
└── README.md
```

## CLAUDE.md Template

```markdown
# Spec-Driven Development

This project uses Design Driven Development. All specifications live in `specs/`.

## Spec Files (Read These First)

1. **specs/system.yaml** - Project identity, tech stack, domains
2. **specs/architecture.yaml** - Project structure, infrastructure, cross-cutting concerns
3. **specs/config.yaml** - Environment variables schema
4. **specs/schemas/_base.yaml** - Base model inherited by all entities
5. **specs/shared/errors.yaml** - Error codes and response formats

## When implementing a new flow:

1. Read the flow spec:
   ```bash
   cat specs/domains/{domain}/flows/{flow}.yaml
   ```

2. Read related schemas:
   ```bash
   cat specs/schemas/{entity}.yaml
   ```

3. Check architecture for conventions:
   ```bash
   # Check where files should go
   cat specs/architecture.yaml | grep -A 50 "structure:"
   
   # Check middleware order
   cat specs/architecture.yaml | grep -A 20 "middleware_order:"
   
   # Check error handling
   cat specs/architecture.yaml | grep -A 30 "error_handling:"
   ```

4. Implement to match spec exactly:
   - Validation rules from spec → Pydantic validators
   - Error messages from spec → Use exact text
   - Error codes from errors.yaml → Use defined codes
   - API conventions from architecture → Follow pagination, filtering, response format

5. Validate implementation:
   ```bash
   ddd validate
   ```

6. Commit changes:
   ```bash
   git add src/ tests/
   git commit -m "Implement: {flow description}"
   ```

## Key Architecture Decisions

- **Multi-tenancy**: Row-level, tenant_id on all queries
- **Soft delete**: Use deleted_at, never DELETE
- **Auth**: JWT with RS256, access + refresh tokens
- **Errors**: Always use codes from specs/shared/errors.yaml
- **Logging**: Structured JSON, include request_id and tenant_id
- **Pagination**: Cursor-based by default
- **Testing**: Transaction rollback strategy

## File Locations

| Spec | Code Location |
|------|---------------|
| specs/domains/{domain}/flows/*.yaml | src/domains/{domain}/router.py |
| specs/schemas/{entity}.yaml | src/domains/{domain}/models.py + schemas.py |
| specs/shared/errors.yaml | src/shared/exceptions/*.py |
| specs/architecture.yaml → middleware | src/shared/middleware/*.py |
| specs/architecture.yaml → auth | src/shared/auth/*.py |

## Validation Commands

```bash
# Check spec-code sync
ddd validate

# Check specific flow
ddd validate -f webhook-ingestion

# See what needs implementation
ddd pending

# Run tests
pytest

# Type check
mypy src/
```
```

---

# Part 13: Glossary

| Term | Definition |
|------|------------|
| **Flow** | A complete process from trigger to terminal (e.g., webhook-ingestion) |
| **Node** | A single step in a flow (trigger, input, process, decision, etc.) |
| **Spec** | YAML file defining a flow, schema, or system configuration |
| **Mapping** | Connection between spec elements and code locations |
| **Validation** | Checking that code matches spec exactly |
| **Sync** | State where code and specs match |
| **Changeset** | A set of spec changes to be implemented |
| **Domain** | Bounded context grouping related flows |
| **Schema** | Data model definition (reusable across flows) |
| **Event** | Async message published by one flow, consumed by others |
| **Sub-flow** | Flow called synchronously from another flow |
| **sourceHandle** | Named output handle on a branching node (e.g., `valid`/`invalid` on input, `success`/`error` on data_store) |
| **Flow Template** | Pre-built flow graph that can be inserted as a starting point (e.g., REST API, CRUD Entity, RAG Agent) |
| **Generator** | Tool that produces production artifacts (OpenAPI, Dockerfile, Mermaid) from specs |

---

# Part 14: Context for OBLIGO

The DDD tool is being designed with OBLIGO as the primary use case:

**OBLIGO:** A "Cyber Liability Operating System" SaaS platform that creates an "Operational Obligations Graph" from contracts by integrating with CLM, CRM, and threat intelligence systems.

**Key Domains:**
- **Ingestion:** Webhooks from CLM providers (Ironclad, Icertis, DocuSign)
- **Analysis:** Extract obligations from contracts using LLM
- **API:** REST API for frontend and integrations
- **Notification:** Email/Slack alerts for obligations

This context helps inform design decisions around:
- Webhook handling patterns
- LLM integration nodes
- Multi-tenant architecture
- Security requirements

---

# Part 15: Continuation Prompt

To continue this conversation in another Claude session, start with:

```
I'm building DDD (Design Driven Development), a tool for bidirectional 
conversion between visual flow diagrams and code. I have a detailed 
specification document I'd like to share. The key points are:

1. Specs (YAML files) are the single source of truth, stored in Git
2. DDD Tool is a desktop app (Tauri) for visual editing of specs
3. Claude Code reads specs and implements code
4. Git handles all sync (no custom protocol)
5. DDD CLI provides validation, generation, and instructions

I'd like to continue discussing [specific topic].
```

Then share the relevant section(s) of this document.

---

# Document Version

**Version:** 1.1.0
**Created:** 2025-02-04
**Updated:** 2026-02-15
**Author:** Murat (with Claude)
**Context:** Full specification developed over extended conversation, updated to reflect current DDD Tool implementation (Sessions 1-17 complete)

---

# End of Document
