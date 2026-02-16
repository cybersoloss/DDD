# DDD Reflect

Capture implementation wisdom — patterns and details that code has but specs don't describe. Creates annotation files in `.ddd/annotations/` for human review and later promotion into permanent specs via `/ddd-promote`.

## Scope Resolution

Parse `$ARGUMENTS` to determine scope:

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | Entire project | `/ddd-reflect --all` |
| `{domain}` | All flows in a domain | `/ddd-reflect monitoring` |
| `{domain}/{flow}` | Single flow | `/ddd-reflect monitoring/check-social-sources` |
| *(empty)* | Interactive — show flows, ask which to reflect on | `/ddd-reflect` |

## Instructions

1. **Find the DDD project**: Look for `ddd-project.json` in the current directory or parent directories.

2. **Read project context**:
   - `ddd-project.json` — domain list
   - `.ddd/mapping.yaml` — implementation tracking (files, hashes)
   - `specs/architecture.yaml` — especially `cross_cutting_patterns` section (reference for classifying findings)
   - `.ddd/annotations/` — existing annotations (to skip duplicates)

3. **Resolve scope**: Parse `$ARGUMENTS` using the table above.
   - If no argument, show all implemented flows (from mapping.yaml) and ask the user which to reflect on
   - If `--all`, process every flow that has an implementation in mapping.yaml
   - If `{domain}`, process all implemented flows in that domain
   - If `{domain}/{flow}`, process that single flow

4. **For each flow in scope**, perform wisdom capture:

   a. **Read the flow spec YAML** from the spec path in mapping.yaml

   b. **Read the implementation files** listed in mapping.yaml for this flow

   c. **Compare spec vs code** — identify what code does that spec doesn't describe:
      - **Utility usage**: Does the code import/use shared utilities (stealth HTTP, encryption, API key resolution) that the spec doesn't mention?
      - **Error handling patterns**: Does the code have retry logic, circuit breakers, graceful degradation, fallback behavior beyond what the spec describes?
      - **Data filtering**: Does the code apply filters (soft-delete, tenant isolation, status filters) that the spec doesn't capture?
      - **Security measures**: Does the code encrypt/decrypt fields, validate API keys, sanitize inputs beyond spec requirements?
      - **Performance patterns**: Does the code use caching, batching, connection pooling, or rate limiting not in the spec?
      - **Content processing**: Does the code hash content for deduplication, normalize data, or transform formats beyond spec?
      - **Infrastructure integration**: Does the code use queue-specific features, database-specific operations, or cache patterns not in spec?

   d. **Classify each finding** into a pattern category:
      | Category | Indicators |
      |----------|-----------|
      | `stealth_http` | User-agent rotation, proxy pools, cookie jars, headless browser fallback, anti-detection delays |
      | `api_key_resolution` | Multi-source key lookup (DB → env), key validation, key rotation |
      | `encryption` | Field-level encrypt/decrypt, key derivation, algorithm selection |
      | `soft_delete` | `deletedAt: null` or `deleted_at IS NULL` filters on reads |
      | `content_hashing` | SHA-256/MD5 hashing for deduplication, content fingerprinting |
      | `error_handling` | Retry with backoff, circuit breaker, graceful degradation, dead letter queues |
      | `custom` | Project-specific patterns not matching above categories |

      Use `architecture.yaml` → `cross_cutting_patterns` as reference — if a pattern is already documented there, note the match but still check if the code's implementation has details the pattern description doesn't capture.

   e. **Check for duplicates**: Read existing `.ddd/annotations/{domain}/{flow}.yaml` if it exists. Skip findings that match existing annotations (same type + same applies_to_nodes).

   f. **Check for already-specified patterns**: If the finding is already described in the flow spec (e.g., the spec already mentions encryption in a crypto node), skip it — it's not "wisdom" if the spec already knows.

5. **Write annotation files**: For each flow with new findings, write to `.ddd/annotations/{domain}/{flow}.yaml`:

   ```yaml
   flow: {domain}/{flow-id}
   captured_at: "{ISO timestamp}"
   captured_from: reflect
   patterns:
     - id: {nanoid(8)}
       type: {category}
       description: >
         {What the code does that the spec doesn't capture}
       applies_to_nodes: [{node-ids where this applies}]
       code_evidence:
         file: {relative path to source file}
         lines: "{start-end}"
         snippet: >
           {Brief code excerpt showing the pattern — max 5 lines}
       status: candidate
   implementation_details:
     - node_id: {node-id}
       detail: >
         {What the code adds beyond what the spec describes for this node}
       code_evidence:
         file: {relative path}
         lines: "{start-end}"
   ```

   If the annotation file already exists (from a previous reflect), merge new findings — append to `patterns` array, skip duplicates.

6. **Update mapping.yaml**: For each flow that got new annotations, update the `annotationCount` field in `.ddd/mapping.yaml`:
   ```yaml
   flows:
     {domain}/{flow}:
       annotationCount: {total pending annotations for this flow}
   ```

7. **Report summary**:
   ```
   Reflected on: {N} flows

   {domain}/{flow}:
     Patterns found: 3
       stealth_http: Uses stealth HTTP utility for external fetches (2 nodes)
       soft_delete: Applies deletedAt filter on all reads (1 node)
       error_handling: Retry with exponential backoff on API calls (1 node)
     New annotations: 2 (1 already captured)

   {domain}/{flow2}:
     Patterns found: 1
       api_key_resolution: DB-first key lookup with env fallback (1 node)
     New annotations: 1

   Summary:
     Total patterns found: 4
     New annotations created: 3
     Already captured: 1
     Annotation files: .ddd/annotations/{domain}/{flow}.yaml

   Next steps:
     - Review annotations in .ddd/annotations/
     - Run /ddd-promote --review to approve/dismiss findings
     - Approved patterns will be written to architecture.yaml or flow specs
   ```

## Pattern Detection Guidance

When comparing code to spec, focus on these signals:

**Import analysis**: Look at what the code imports beyond what the spec implies:
- `import { stealthFetch } from '../utils/stealth-http'` → stealth_http pattern
- `import { encrypt, decrypt } from '../utils/encryption'` → encryption pattern
- `import { resolveApiKey } from '../utils/api-keys'` → api_key_resolution pattern

**Filter analysis**: Look at database queries for filters not in the spec:
- `where: { deletedAt: null }` → soft_delete pattern
- `where: { tenantId: ctx.tenantId }` → tenant isolation (custom)

**Error handling analysis**: Look at try/catch blocks, retry logic, fallbacks:
- Retry loops with backoff → error_handling pattern
- Circuit breaker imports → error_handling pattern
- Fallback to alternative service → error_handling pattern

**Don't flag**:
- Standard framework boilerplate (Express middleware, Prisma client setup)
- Type definitions that match the spec's intent even if not explicitly specified
- Logging that's a standard part of the architecture
- Test-specific code patterns

$ARGUMENTS
