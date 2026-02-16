# DDD Implement

Implement DDD-designed flows from the specs directory. Supports implementing the entire project, a specific domain, or a specific flow.

## Scope Resolution

Parse the argument to determine scope:

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | Whole project — all domains, all flows | `/ddd-implement --all` |
| `domain-name` | All flows in a domain | `/ddd-implement users` |
| `domain-name/flow-name` | Single flow | `/ddd-implement users/user-registration` |
| *(empty)* | Interactive — list available flows and ask | `/ddd-implement` |

## Instructions

1. **Find the DDD project**: Look for `ddd-project.json` in the current directory or parent directories. This file lists all domains and project configuration (tech stack, etc.).

2. **Resolve the scope from the argument**:

   **If no argument**: List all domains and their flows with implementation status (check `.ddd/mapping.yaml`). Show which are implemented (with date), which have drift, and which are new. Ask the user what to implement.

   **If `--all`**: Collect all flows across all domains. Implement them in dependency order (flows that publish events before flows that consume them).

   **If `domain-name`**: Read `specs/domains/{domain-name}/domain.yaml` to get the flow list. Implement all flows in that domain.

   **If `domain-name/flow-name`**: Read `specs/domains/{domain-name}/flows/{flow-name}.yaml`. Implement just that flow.

3. **Read the specs for each flow**:
   - `ddd-project.json` — project config, tech stack
   - `specs/system.yaml` — project identity, tech stack, environments (if exists)
   - `specs/architecture.yaml` — conventions, infrastructure, API design, **cross-cutting patterns** (if exists)
   - `specs/config.yaml` — environment variable schema (if exists)
   - `specs/shared/errors.yaml` — error codes with HTTP status mappings (if exists)
   - `specs/shared/types.yaml` — shared enum/type definitions (if exists)
   - `specs/schemas/*.yaml` — data model definitions referenced by data_store nodes in the flow (if exist)
   - Note: `system.yaml` may contain an `integrations:` section with external API configs
   - `specs/domains/{domain}/domain.yaml` — domain context, events, relationships
   - `specs/domains/{domain}/flows/{flow}.yaml` — the flow specification

   **IMPORTANT — Cross-cutting patterns**: If `specs/architecture.yaml` contains a `cross_cutting_patterns` section, read it carefully. These are project-wide conventions that apply to ALL flows, even if individual flow specs don't mention them. They MUST be applied during implementation. See step 7 for details.

4. **Fetch the DDD Usage Guide**: Run `gh api repos/mhcandan/DDD/contents/DDD-USAGE-GUIDE.md --jq '.content' | base64 -d` to get the latest version. This guide defines all node types, spec fields, connection patterns, and conventions. Use it as your reference for understanding node specs during implementation.

5. **Understand the flow spec**: Each flow YAML contains:
   - `flow:` — metadata (id, name, type, domain)
   - `trigger:` — what starts the flow (event, HTTP, scheduled, manual)
   - `nodes:` — ordered list of processing steps (see Usage Guide for all node types and their spec fields)
   - Each node has `connections:` listing target nodes with `sourceHandle` for branching nodes
   - Each node has `spec:` with type-specific configuration fields
   - Each node may have `observability:` (logging, metrics, tracing) and `security:` (auth, rate limiting, encryption, audit) configs

6. **Check existing implementation**: Read `.ddd/mapping.yaml` to see if this flow was previously implemented.
   - If yes and spec hasn't changed → skip (tell user it's up to date)
   - If yes and spec changed → update mode (modify existing files, don't recreate)
   - If no → new implementation

7. **Implement**:

   **Entry point — determine from trigger convention**:
   - `HTTP {METHOD} {path}` → route handler (e.g., Express route, FastAPI endpoint)
   - `cron {expression}` → scheduled job (e.g., node-cron, BullMQ repeatable). If the trigger has `job_config`, configure the job queue (e.g., BullMQ) with the specified concurrency, timeout, retry, and dead_letter settings
   - `event:{EventName}` → event listener/consumer (e.g., message queue subscriber)
   - `webhook {path}` → webhook handler route
   - `manual` → CLI command or admin endpoint

   **Follow the node graph** from trigger through all paths to terminal nodes. Each node becomes real code:
   - `process` → service function call
   - `decision` → if/else or switch using the `condition` field
   - `data_store` → database query (ORM call matching `operation`: create/read/update/delete on `model`)
   - `service_call` → HTTP client call (method, url, headers, body, timeout, retry). If `system.yaml` has an `integrations:` section and the service_call URL matches an integration's `base_url`, use the integration's auth, retry, and rate limit config
   - `event` → event emission (emit) or subscription handler (consume)
   - `loop` → for/forEach over `collection` with `iterator` variable, optional `break_condition`
   - `parallel` → Promise.all / concurrent execution of branches
   - `sub_flow` → call to the referenced flow's entry function. If the target flow has a `contract` section, validate that `input_mapping` keys match contract inputs and `output_mapping` keys match contract outputs
   - `llm_call` → LLM API call with model, prompt, temperature, structured output
   - `agent_loop` → agent loop with tool dispatch, memory management, stop conditions
   - `guardrail` → validation middleware (inline, sequential — input guard before agent, output guard after)
   - `human_gate` → async approval workflow (pause, notify, wait for human decision)
   - `orchestrator` → multi-agent coordination per `strategy` (supervisor, round_robin, broadcast, consensus)
   - `smart_router` → routing logic from `rules` and/or `llm_routing`
   - `handoff` → agent transfer with context passing per `mode` (transfer/consult/collaborate)
   - `agent_group` → agent team coordination with shared memory
   - `collection` → collection operation (filter, sort, deduplicate, merge, group_by, aggregate, reduce, flatten) on input
   - `parse` → structured extraction from raw format (rss, atom, html, xml, json, csv, markdown)
   - `crypto` → cryptographic operation (encrypt, decrypt, hash, sign, verify, generate_key)
   - `batch` → execute operation template against each item in input collection with concurrency control
   - `transaction` → atomic multi-step database operation with rollback on error

   **Wire connections using `sourceHandle`** — nodes with multiple output paths use `sourceHandle` values:
   - `input` → `"valid"` path (continue) / `"invalid"` path (validation error terminal)
   - `decision` → `"true"` path / `"false"` path
   - `data_store` → `"success"` path (continue) / `"error"` path (DB error terminal)
   - `service_call` → `"success"` path (continue) / `"error"` path (use `error_mapping` to map HTTP status codes to error codes)
   - `loop` → `"body"` path (loop body) / `"done"` path (after loop completes)
   - `parallel` → `"branch-0"`, `"branch-1"`, etc. (parallel branches) / `"done"` path (join point)
   - `smart_router` → dynamic route names as handle IDs
   - `collection` → `"result"` path / `"empty"` path
   - `parse` → `"success"` path / `"error"` path
   - `crypto` → `"success"` path / `"error"` path
   - `batch` → `"done"` path / `"error"` path
   - `transaction` → `"committed"` path / `"rolled_back"` path
   - All other nodes (delay, transform, sub_flow, orchestrator, handoff, agent_group) → single unnamed output

   **Terminal nodes → HTTP responses**: Use `status` and `body` fields from terminal spec:
   - `status` → HTTP status code (e.g., 201, 400, 409)
   - `body` → response body shape (values starting with `$.` are variable references)
   - Map `body.error` values to error codes defined in `specs/shared/errors.yaml` for consistent error responses

   **Data store pagination/sort**: When `data_store` has `operation: read` with `pagination` and/or `sort` fields:
   - Implement cursor-based or offset pagination per `pagination.style`
   - Apply sort ordering per `sort.default` and `sort.allowed`
   - Accept pagination/sort query parameters in the route handler

   **Cross-cutting concerns**: If a node has `observability` or `security` config:
   - `observability.logging` → add structured logging at the specified level, include/exclude input/output
   - `observability.metrics` → instrument with counters/histograms for the custom counters listed
   - `observability.tracing` → add tracing spans with the specified span name
   - `security.authentication` → add auth middleware (methods, required roles)
   - `security.rate_limiting` → add rate limiting middleware (requests per minute)
   - `security.encryption` → encrypt PII fields at rest, ensure TLS in transit
   - `security.audit` → add audit logging for the operation

   **Follow architecture conventions**: Use `specs/architecture.yaml` to shape generated code:
   - `project_structure` → place files in the correct directories
   - `naming_conventions` → use the right casing for files, classes, functions, variables, tables, endpoints
   - `dependencies` → use the specified libraries and versions
   - `api_design` → follow versioning, pagination format, filtering style, error format
   - `testing` → use the specified framework, runner, and patterns

   **Apply cross-cutting patterns** (CRITICAL): If `architecture.yaml` has a `cross_cutting_patterns` section, each pattern defines a project-wide convention that MUST be applied to every flow in matching domains. These patterns exist because implementation experience proved they're necessary — skipping them will produce code that breaks in production.

   For each pattern, check:
   1. Does this flow's domain appear in the pattern's `used_by_domains` list?
   2. Does this flow contain node types that the pattern's `convention` addresses?

   If yes, apply the pattern. Common cross-cutting patterns and how to apply them:

   - **stealth_http** — Any `service_call` node or agent tool that fetches external web content: use the stealth HTTP utility (e.g., `stealthFetch`) instead of plain HTTP clients. Apply the configured `rotateUserAgent`, `delayMin`, `delayMax` settings. Only skip for trusted first-party APIs listed in `system.yaml` integrations.

   - **api_key_resolution** — Any flow that needs API keys: use the key resolution utility (e.g., `requireApiKey`/`getApiKey`) instead of reading `process.env` directly. This checks the database first (user may have configured keys via UI), caches lookups, and falls back to env vars.

   - **encryption** — Any `data_store` node that writes credentials, API keys, or tokens: encrypt before write using the encryption utility. Any read that returns credentials: decrypt after read. Display to users: use masking.

   - **soft_delete** — ALL `data_store` read operations: include `deletedAt: null` in filters unless the flow explicitly queries deleted records. This prevents returning soft-deleted records.

   - **content_hashing** — Any flow that stores content items: compute a content hash for deduplication before storing.

   - **error_handling** — Any `loop` node with `on_error: continue`: wrap each iteration in try/catch so one item's failure doesn't stop the batch. Log per-item errors but continue processing.

   If a flow spec already specifies a cross-cutting concern at the node level (e.g., `request_config` on a tool, `deletedAt: null` in filters), use the flow-level spec. The flow spec takes precedence over architecture defaults, but architecture defaults fill in anything the flow spec doesn't mention.

   - Match the project's existing code style and conventions
   - For multi-flow implementations, share common infrastructure (DB, config, middleware)

8. **Write tests**: Create tests covering:
   - Happy path through the flow
   - Each decision branch
   - Error/terminal states
   - Input validation rules from input node specs

9. **Run tests and fix**: Run the test suite. If tests fail, fix the implementation. Keep iterating until all tests pass.

10. **Update mapping**: After each flow is successfully implemented, update `.ddd/mapping.yaml`:
   ```yaml
   flows:
     domain-id/flow-id:
       specHash: (sha256 of the flow YAML content)
       implementedAt: (current ISO timestamp)
       files:
         - src/path/to/file1.ts
         - src/path/to/file2.ts
   ```

11. **Summary**: After all flows are done, show a summary table:
    ```
    Domain/Flow                  Status    Files  Tests
    users/user-registration      ✓ done    5      12/12
    users/user-login             ✓ done    3      8/8
    billing/create-subscription  ✓ done    4      6/6
    ```

$ARGUMENTS
