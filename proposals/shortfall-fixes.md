# DDD Shortfall Fixes — Complete Proposal

Based on the Content Curator 5 analysis (28 flows, ~200 nodes, 9 domains). Every shortfall is addressed.

---

## Part A: New Node Types (5 nodes)

### A1. `collection` node

Replaces: `filter`, `collection_operation`, `accumulator` (3 of 5 missing types — combined into one).

Handles all collection-level operations: filter, sort, deduplicate, merge, group_by, aggregate, reduce, flatten.

**Spec fields:**

```yaml
type: collection
spec:
  operation: filter | sort | deduplicate | merge | group_by | aggregate | reduce | flatten
  input: "$.sources"                    # input collection (single or array of collections for merge)
  predicate: "item.created_at > $.last_checked_at"  # for filter
  key: "item.url"                       # for deduplicate, sort, group_by
  direction: asc | desc                 # for sort
  accumulator:                          # for reduce/aggregate
    init: []                            # initial value
    expression: "acc.concat(item.new_posts)"  # reduce expression
  output: "filtered_posts"              # output variable name
```

**Connections:** `sourceHandle: "result"` (single output — the transformed collection) + `sourceHandle: "empty"` (collection is empty after operation).

**Would eliminate these workarounds:**
- `proc_extract_twitter` — "Extract new posts since last_checked_at" → `collection` with `operation: filter`
- `proc_extract_linkedin` — same
- `process-001` in keyword-search — "Merge and deduplicate results" → `collection` with `operation: merge` then `operation: deduplicate`
- All loop accumulation problems — `collection` with `operation: reduce` after loop

**Also solves:**
- Shortfall 1.3 (filter/collection_operation) — fully
- Shortfall 1.5 (accumulator/collector) — via `operation: reduce` and `operation: aggregate`
- Shortfall 2.5 (loop aggregation) — use `collection` node at loop's `done` handle
- Shortfall 2.8 (event batch emission) — `collection` node accumulates, then `event` node emits the result

---

### A2. `parse` node

Structured extraction from raw content formats.

**Spec fields:**

```yaml
type: parse
spec:
  format: rss | atom | html | xml | json | csv | markdown
  input: "$.raw_response"              # raw content to parse
  strategy:                            # format-specific extraction config
    selectors:                         # for HTML
      - { name: title, css: "h1.article-title" }
      - { name: body, css: "article.content", extract: text }
      - { name: links, css: "a[href]", extract: href, multiple: true }
    fields:                            # for RSS/Atom/XML
      - { name: title, path: "item.title" }
      - { name: link, path: "item.link" }
      - { name: published, path: "item.pubDate", transform: date }
    root_path: "rss.channel.item"      # for XML/JSON — path to array
  library: cheerio | rss-parser | fast-xml-parser  # hint for implementation
  output: "parsed_entries"             # output variable
```

**Connections:** `sourceHandle: "success"` + `sourceHandle: "error"` (malformed content).

**Would eliminate these workarounds:**
- `proc_parse_medium` — "Parse Medium RSS entries" → `parse` with `format: rss`
- `proc_parse_rss` — "Parse RSS/Atom feed entries" → `parse` with `format: atom`
- `proc_extract_web` — "Extract article content with cheerio" → `parse` with `format: html`

---

### A3. `crypto` node

Cryptographic operations: encrypt, decrypt, hash, sign, verify.

**Spec fields:**

```yaml
type: crypto
spec:
  operation: encrypt | decrypt | hash | sign | verify | generate_key
  algorithm: aes-256-gcm | aes-256-cbc | sha256 | sha512 | hmac-sha256 | rsa-oaep | ed25519
  key_source: { env: "ENCRYPTION_KEY" }  # where the key comes from
  input_fields: ["api_key"]              # field(s) to process
  output_field: "encrypted_key"          # result field name
  encoding: base64 | hex                 # output encoding
```

**Connections:** `sourceHandle: "success"` + `sourceHandle: "error"` (crypto failure — invalid key, corrupted data).

**Would eliminate:**
- `process-001` in save-api-keys — "Encrypt key value with AES-256" → `crypto` with `operation: encrypt`
- Any future decrypt/hash/sign needs

---

### A4. `batch` node

Execute a heterogeneous list of operations (typically service calls) against a collection where each item may target a different endpoint, auth method, and response format.

**Spec fields:**

```yaml
type: batch
spec:
  input: "$.api_keys"                   # collection to iterate over
  operation_template:                    # per-item operation config
    type: service_call                   # what kind of operation (service_call, data_store, etc.)
    dispatch_field: "item.provider"      # field that determines which config to use
    configs:
      openai:
        method: GET
        url: "https://api.openai.com/v1/models"
        headers: { Authorization: "Bearer {item.key}" }
        success_check: "response.status == 200"
      anthropic:
        method: GET
        url: "https://api.anthropic.com/v1/models"
        headers: { x-api-key: "{item.key}" }
        success_check: "response.status == 200"
      google:
        method: GET
        url: "https://generativelanguage.googleapis.com/v1/models?key={item.key}"
        success_check: "response.status == 200"
  concurrency: 3                        # max parallel operations
  on_error: continue | stop             # per-item error handling
  output: "validation_results"           # array of { item, result, success }
```

**Connections:** `sourceHandle: "done"` (all complete) + `sourceHandle: "error"` (if `on_error: stop` and one fails).

**Would eliminate:**
- `process-002` in save-api-keys — "Validate keys via lightweight test calls" → `batch` node

---

### A5. `transaction` node

Group multiple data operations into an atomic unit. If any step fails, all are rolled back.

**Spec fields:**

```yaml
type: transaction
spec:
  isolation: read_committed | serializable | repeatable_read  # DB isolation level
  steps:                                # ordered operations within the transaction
    - { operation: create, model: Source, data: "$.source_data" }
    - { operation: update, model: Creator, query: { id: "$.creator_id" }, data: { promoted: true } }
  rollback_on_error: true               # auto-rollback on any step failure
```

**Connections:** `sourceHandle: "committed"` + `sourceHandle: "rolled_back"`.

**Addresses:** Shortfall in Sources domain `review-source-suggestion` — two sequential data_store updates that need atomicity.

---

## Part B: Existing Node Extensions (9 fixes)

### B1. `trigger` — add `filter` field

**Current:** `event` and `source` fields only.

**Add:**

```yaml
trigger:
  spec:
    event: "event:DraftApproved"
    source: approval-domain
    filter:                             # NEW — event payload filter
      platform: twitter                 # only trigger when payload.platform == "twitter"
    # supports dot notation and operators:
    # filter:
    #   "payload.amount": { gte: 100 }
    #   "payload.status": [active, pending]  # in-list
```

**Impact:** Eliminates ~4 unnecessary decision nodes in Publishing domain. Cleaner flows.

**Implementation:** Filter is checked before the flow starts. If filter doesn't match, flow doesn't trigger. In DDD Tool, display the filter as a badge on the trigger node.

---

### B2. `data_store` — add `include` field for joins

**Current:** `operation`, `model`, `query`, `data`, `pagination`, `sort`.

**Add:**

```yaml
data_store:
  spec:
    operation: read
    model: Draft
    query: { id: "$.draft_id" }
    include:                            # NEW — eager load related records
      - model: ContentItem
        via: content_item_id            # foreign key on Draft
        as: content_item                # output field name
      - model: SocialAccount
        via: content_item.subject_id    # nested relation path
        as: social_account
```

**Impact:** Eliminates needing multiple sequential data_store reads for related data. Covers Approval domain `get-draft-detail` and Settings `get-settings`.

---

### B3. `data_store` — add `update_many` and `delete_many` operations

**Current:** `create`, `read`, `update`, `delete`, `upsert`.

**Add:** `create_many`, `update_many`, `delete_many` operations.

```yaml
data_store:
  spec:
    operation: update_many              # NEW
    model: ApiKey
    query: { user_id: "$.user_id" }     # match criteria
    data: "$.validation_results"        # array of { id, updates } or bulk update object
    returning: true                     # return updated records
```

---

### B4. `loop` — add `accumulate` field

**Current:** `collection`, `iterator`, `break_condition`, `on_error`.

**Add:**

```yaml
loop:
  spec:
    collection: "$.sources"
    iterator: source
    on_error: continue
    accumulate:                         # NEW — collect results across iterations
      field: new_posts                  # what to collect from each iteration's output
      strategy: append | merge | count | sum | concat
      output: "all_new_posts"           # variable name available at "done" handle
```

**Impact:** Makes loop result collection explicit instead of relying on implicit variables. Solves the fundamental problem of "what happens to results produced during each iteration."

---

### B5. `parallel` — add `condition` per branch

**Current:** `branches` (array of items), `join` (all/any/n_of).

**Add:**

```yaml
parallel:
  spec:
    branches:
      - id: twitter-draft
        label: "Generate Twitter draft"
        condition: "$.social_accounts.has_twitter"   # NEW — skip branch if false
      - id: linkedin-draft
        label: "Generate LinkedIn draft"
        condition: "$.social_accounts.has_linkedin"  # NEW
    join: all
```

**Impact:** Eliminates decision nodes before parallel branches. Content domain `generate-drafts` no longer needs to check account existence separately.

---

### B6. `event` — add `payload_source` for dynamic/accumulated payloads

**Current:** `direction`, `event_name`, `payload` (static template).

**Add:**

```yaml
event:
  spec:
    direction: emit
    event_name: NewContentDiscovered
    payload_source: "$.all_new_posts"   # NEW — reference to dynamic/accumulated variable
    # payload_source takes precedence over payload when present
    # payload remains for static payloads
    payload:                            # used as schema documentation even when payload_source is set
      posts: "array of ContentItem"
      source_id: string
      checked_at: datetime
```

**Impact:** Enables clean event emission after loops with accumulated results. The static `payload` field still serves as documentation of the event shape.

---

### B7. `llm_call` — add `context_sources` and template functions

**Current:** `model`, `system_prompt`, `prompt_template` with `{variable}` placeholders.

**Add:**

```yaml
llm_call:
  spec:
    model: claude-sonnet
    system_prompt: "You are a social media expert."
    prompt_template: "Write a {platform} post about: {content}"
    context_sources:                    # NEW — formal variable bindings
      platform:
        from: "$.social_account.platform"
      content:
        from: "$.content_item.raw_content"
        transform: truncate(4000)       # NEW — template functions
      tone:
        from: "$.social_account.preferences.tone"
      keywords:
        from: "$.subject.keywords"
        transform: join(", ")           # array to comma-separated string
    # supported transforms: truncate(n), join(sep), lowercase, uppercase,
    # first(n), last(n), json_stringify, strip_html, summarize(n)
```

**Impact:** Makes LLM prompt construction explicit and auditable. No more implicit `{variable}` references with unclear sources. The `transform` functions handle common data preparation needs.

---

### B8. `smart_router` — clarify for traditional flow use

**Documentation fix** (not a spec change). In the DDD Usage Guide:

- Move `smart_router` to a shared section "Routing Nodes" visible to both traditional and agent flows
- Add explicit examples of `smart_router` in traditional flows (e.g., platform-based routing, role-based routing)
- Add a note: "smart_router is not agent-only — use it in any flow where you need multi-way branching based on rules or data values"

---

### B9. `process` — add `category` field to reduce catch-all ambiguity

**Current:** `action`, `service`, `description`.

**Add:**

```yaml
process:
  spec:
    action: hashPassword
    service: AuthService
    category: security | transform | integration | business_logic | infrastructure  # NEW
    inputs: ["$.password"]                # NEW — explicit input fields
    outputs: ["$.hashed_password"]        # NEW — explicit output fields
```

**Impact:** When a process node IS the right choice (genuine business logic), the `category` field helps implementers understand what kind of code to write. The `inputs`/`outputs` fields make data flow explicit. Doesn't fix the catch-all problem entirely (new node types do that), but makes remaining process nodes more structured.

---

## Part C: New Spec Concepts (3 additions)

### C1. Parameterized flows (flow templates)

For flows that share identical structure but differ by configuration (e.g., `publish-to-twitter` and `publish-to-linkedin`).

**In flow YAML:**

```yaml
flow:
  id: publish-to-platform
  name: Publish to Platform
  type: traditional
  domain: publishing
  template: true                        # NEW — this is a template
  parameters:                           # NEW — template parameters
    platform:
      type: string
      values: [twitter, linkedin, facebook, instagram]
    api_integration:
      type: integration_ref             # references system.yaml integrations
    publish_endpoint:
      type: string

# In domain.yaml, instantiate:
flows:
  - id: publish-to-twitter
    template: publish-to-platform
    parameters:
      platform: twitter
      api_integration: twitter-api-v2
      publish_endpoint: /2/tweets
  - id: publish-to-linkedin
    template: publish-to-platform
    parameters:
      platform: linkedin
      api_integration: linkedin-api
      publish_endpoint: /v2/ugcPosts
```

**Impact:** Eliminates duplicate flow definitions. Publishing domain goes from 2 near-identical flows to 1 template + 2 instantiations. Future platforms (Facebook, Instagram) are one-line additions.

---

### C2. Cross-cutting concern layers

For infrastructure that spans multiple nodes/flows but isn't a node itself (e.g., stealth/anti-detection layer, retry policies, circuit breakers).

**New file:** `specs/shared/layers.yaml`

```yaml
layers:
  - id: stealth
    name: Stealth / Anti-Detection
    description: "Rotating user agents, proxy pools, cookie jars, headless browser fallback"
    applies_to:
      nodes: [service_call, parse]      # node types this layer wraps
      domains: [monitoring, discovery]  # domains where it's active
    config:
      user_agent_pool: 50
      proxy_rotation: per_request
      cookie_jar: per_domain
      headless_fallback: true
      rate_limits:
        twitter: { rpm: 30, daily: 500 }
        google: { rpm: 10, daily: 100 }

  - id: retry-policy
    name: Retry Policy
    description: "Exponential backoff with circuit breaker"
    applies_to:
      nodes: [service_call, batch]
      domains: [monitoring, discovery, publishing]
    config:
      max_retries: 3
      backoff: exponential
      initial_delay_ms: 1000
      circuit_breaker:
        failure_threshold: 5
        reset_timeout_ms: 60000
```

**In DDD Tool:** At L3, nodes affected by a layer show a small badge/icon. Clicking the badge shows the layer config. At L1, layers are shown as overlays on affected domains.

---

### C3. Connection data annotations

Make data flowing between nodes visible at L3.

**In flow YAML connections:**

```yaml
connections:
  - target: data_store-abc123
    sourceHandle: valid
    data:                               # NEW — what data this connection carries
      - { name: email, type: string }
      - { name: password, type: string }
      - { name: name, type: string }
```

**In DDD Tool:** Hover over a connection arrow to see the data shape. Optional — not required for every connection, but available when explicit data documentation is useful.

---

## Part D: Layer Visibility Enhancements

### D1. L1 — Inter-zone data flow arrows

**Current:** L1 shows domain blocks with flow count badges and event arrows between domains.

**Add:** Directional data flow arrows between zones (or domain groups). These are higher-level than event arrows — they show the architectural flow.

**In `system.yaml`:**

```yaml
data_flows:                             # NEW section
  - from: first-loop                    # zone or domain ID
    to: shared-pipeline
    label: "ContentDiscovered events"
    volume: high                        # low, medium, high — for visual thickness
  - from: shared-pipeline
    to: second-loop
    label: "DraftApproved events"
    volume: medium
  - from: first-loop
    to: second-loop
    label: "Source suggestions (flywheel)"
    volume: low
    style: dashed                       # visual distinction for indirect/suggestion flows
```

**In DDD Tool:** At L1, render these as thick directional arrows between zone groupings. The flywheel becomes visible as a feedback loop arrow.

---

### D2. L1 — Scheduling topology overlay

**Current:** L1 has no information about when things run.

**Add:** A toggle-able overlay that shows cron schedules grouped by frequency.

**In `system.yaml`:**

```yaml
schedules:                              # NEW section — derived from flows but surfaced at L1
  - frequency: "*/15 * * * *"
    label: "Every 15 minutes"
    flows: [monitoring/check-social-sources]
  - frequency: "0 */4 * * *"
    label: "Every 4 hours"
    flows: [discovery/keyword-search]
  - frequency: "0 */6 * * *"
    label: "Every 6 hours"
    flows: [discovery/creator-discovery]
  - frequency: "0 2 * * *"
    label: "Daily at 2am"
    flows: [discovery/suggest-source, monitoring/full-source-check]
```

**In DDD Tool:** At L1, a "Schedules" toggle shows timeline lanes or badges on domains indicating when their cron flows run. Makes it immediately visible that monitoring runs every 15 minutes while discovery runs every 4-6 hours.

---

### D3. L1 — Integration dependency map

**Current:** `system.yaml` has `integrations` with `used_by_domains`, but no visual representation.

**Add:** At L1, render external integrations as special blocks on the edge of the canvas (different shape — e.g., cloud icon). Draw connection arrows from integration blocks to the domains that use them.

**Already in spec** — just needs DDD Tool visualization. The `used_by_domains` field in `system.yaml` integrations provides the data. No spec change needed.

---

### D4. L1 — System characteristic badges

**Current:** L1 shows domains and events. No way to see system-level characteristics.

**Add:** Badges on the L1 canvas showing key system properties:

```yaml
# In system.yaml
characteristics:                        # NEW section
  - "Event-driven"                      # primary architecture pattern
  - "6 cron schedules"                  # scheduling summary
  - "6 external APIs"                   # integration count
  - "2-loop flywheel"                   # custom architectural concept
```

**In DDD Tool:** Render as small info chips at the top of the L1 canvas.

---

### D5. L2 — Cross-domain event pipeline view

**Current:** L2 shows a single domain's flows and its published/consumed events as lists.

**Add:** A toggle-able "Pipeline View" at L2 that traces a complete event chain across domains.

**In `system.yaml`:**

```yaml
pipelines:                              # NEW section
  - id: content-pipeline
    name: Content Processing Pipeline
    description: "End-to-end from discovery to publication"
    steps:
      - { domain: monitoring, flow: check-social-sources, event_out: NewContentDiscovered }
      - { domain: content, flow: analyze-content, event_out: DraftGenerated }
      - { domain: approval, flow: review-draft, event_out: DraftApproved }
      - { domain: publishing, flow: publish-to-twitter, event_out: PostPublished }
    spans_domains: [monitoring, content, approval, publishing]
```

**In DDD Tool:** At L2, a "Pipelines" tab shows the pipeline as a horizontal sequence of steps, each linking to its flow. Clicking a step navigates to that flow's L3 canvas.

---

### D6. L2 — Cron schedule badges on flows

**Current:** Flow entries show name, description, type, tags. No schedule info.

**Add:** For flows with cron triggers, display the schedule as a badge on the flow block at L2.

**No spec change needed** — read the trigger node from the flow spec and display its `event` value (e.g., `*/15 * * * *`) as a small badge on the flow block in the DDD Tool.

---

### D7. L2 — Cross-domain data dependencies

**Current:** Only event wiring (publishes/consumes) is shown.

**Add:** A `depends_on` section in `domain.yaml`:

```yaml
depends_on:                             # NEW section
  - domain: settings
    reason: "Reads API keys and social account preferences"
    flows_affected: [generate-drafts, check-social-sources]
  - domain: subjects
    reason: "Reads subject keywords for content scoring"
    flows_affected: [analyze-content]
```

**In DDD Tool:** At L2, render data dependency arrows (dashed, different color from event arrows) between domains.

---

### D8. L2 — Flow criticality / priority marking

**Current:** All flows look equal at L2.

**Add:** `criticality` field on flow entries in `domain.yaml`:

```yaml
flows:
  - id: analyze-content
    name: Analyze Content
    type: traditional
    criticality: critical               # NEW — critical | high | normal | low
    throughput: "~500 items/day"         # NEW — expected volume
```

**In DDD Tool:** At L2, critical flows have a different border color or thickness. Throughput shown as a subtle label.

---

### D9. L2 — Sub-flow call relationships

**Current:** Sub-flow references are only visible inside L3 flow specs.

**Add:** At L2, draw arrows between flows that call each other via `sub_flow` nodes.

**No spec change needed** — the DDD Tool can derive these from `sub_flow` nodes' `flow_ref` fields and render them as arrows between flow blocks at L2.

---

### D10. L3 — Data shape annotations on connections

**Covered by C3 above.** Hover over connection arrows to see what data they carry.

---

### D11. L3 — Variable scope panel

**Current:** Variables are implicit — referenced via `$.node_id.result` syntax but never formally declared.

**Add:** A "Variables" panel in the DDD Tool L3 view (alongside the existing Spec Panel):

Shows for the selected node:
- **Available variables**: all variables in scope at this point in the flow
- **Produced variables**: what this node outputs
- **Consumed variables**: what this node reads

**No spec change needed** — derivable from node connections and spec fields. The DDD Tool traces the graph and builds the scope.

---

### D12. L3 — Connection behavior annotations

**Current:** Connections are just arrows. No way to distinguish "hard fail → stop" from "soft fail → continue next iteration."

**Add:** Optional `behavior` field on connections:

```yaml
connections:
  - target: terminal-error-001
    sourceHandle: error
    behavior: continue                  # NEW — continue | stop | retry(3) | circuit_break
    label: "Log and skip"
```

**In DDD Tool:** Connections with `behavior: continue` render as dashed lines. `retry(n)` shows a retry icon. `circuit_break` shows a circuit breaker icon.

---

### D13. L3 — Cross-cutting concern visibility

**Covered by C2 (layers).** Nodes affected by a layer show a badge. Also:

**Add to DDD Tool:** A "Layers" toggle at L3 that highlights all nodes affected by each layer (e.g., toggle "Stealth" and all service_call nodes in monitoring/discovery get highlighted).

---

## Implementation Priority

### Tier 1 — Highest impact, eliminates most workarounds

| # | Change | Type | Workarounds eliminated |
|---|--------|------|----------------------|
| A1 | `collection` node | New node | 5 of 8 |
| A2 | `parse` node | New node | 3 of 8 |
| B1 | `trigger` filter field | Field extension | ~4 extra decision nodes |
| B2 | `data_store` include/join | Field extension | Multiple flows |

**Combined:** eliminates 100% of process-node workarounds + ~4 unnecessary decision nodes.

### Tier 2 — Important structural improvements

| # | Change | Type |
|---|--------|------|
| A3 | `crypto` node | New node |
| A4 | `batch` node | New node |
| B3 | `data_store` update_many | Field extension |
| B4 | `loop` accumulate field | Field extension |
| B5 | `parallel` conditional branches | Field extension |
| B6 | `event` payload_source | Field extension |
| B7 | `llm_call` context_sources | Field extension |
| B9 | `process` category/inputs/outputs | Field extension |

### Tier 3 — Spec-level concepts

| # | Change | Type |
|---|--------|------|
| A5 | `transaction` node | New node |
| C1 | Parameterized flows | New concept |
| C2 | Cross-cutting layers | New concept |
| C3 | Connection data annotations | New concept |

### Tier 4 — Layer visibility (DDD Tool sessions)

| # | Change | Layer |
|---|--------|-------|
| D1 | Inter-zone data flow arrows | L1 |
| D2 | Scheduling topology overlay | L1 |
| D3 | Integration dependency map | L1 |
| D4 | System characteristic badges | L1 |
| D5 | Cross-domain pipeline view | L2 |
| D6 | Cron schedule badges | L2 |
| D7 | Cross-domain data dependencies | L2 |
| D8 | Flow criticality marking | L2 |
| D9 | Sub-flow call relationships | L2 |
| D10 | Data shape on connections | L3 |
| D11 | Variable scope panel | L3 |
| D12 | Connection behavior annotations | L3 |
| D13 | Layer visibility toggle | L3 |

### Implementation effort estimate

- **Tier 1 (spec + tool):** Update DDD Usage Guide + add 2 node types + 2 field extensions to DDD Tool → 1 DDD Tool session
- **Tier 2 (spec + tool):** Update DDD Usage Guide + add 2 node types + 5 field extensions → 1 DDD Tool session
- **Tier 3 (spec + tool):** New YAML formats + DDD Tool support → 1 DDD Tool session
- **Tier 4 (tool only):** L1/L2/L3 visual enhancements → 2-3 DDD Tool sessions

**Total: 5-6 DDD Tool sessions to address every shortfall.**

---

## Shortfall Coverage Matrix

Every shortfall from the analysis mapped to its fix:

| Shortfall | Fix | Tier |
|---|---|---|
| 1.1 encrypt/decrypt node | A3 crypto node | 2 |
| 1.2 batch_service_call node | A4 batch node | 2 |
| 1.3 filter/collection_operation node | A1 collection node | 1 |
| 1.4 parse node | A2 parse node | 1 |
| 1.5 accumulator/collector node | A1 collection node (reduce op) | 1 |
| 2.1 process too unstructured | A1+A2+A3+A4 (new nodes) + B9 (category) | 1+2 |
| 2.2 trigger no event filtering | B1 trigger filter field | 1 |
| 2.3 data_store no join | B2 data_store include field | 1 |
| 2.4 data_store no update_many | B3 data_store update_many | 2 |
| 2.5 loop no aggregation | B4 loop accumulate field + A1 collection | 2 |
| 2.6 parallel no conditional branches | B5 parallel condition per branch | 2 |
| 2.7 smart_router docs unclear | B8 documentation fix | 2 |
| 2.8 event no batch emission | B6 event payload_source + A1 collection | 2 |
| 2.9 llm_call no dynamic context | B7 llm_call context_sources | 2 |
| L1 no inter-zone data flow | D1 data flow arrows | 4 |
| L1 no scheduling topology | D2 scheduling overlay | 4 |
| L1 no integration dependency map | D3 integration map | 4 |
| L1 no cross-cutting visibility | C2 layers + D4 badges | 3+4 |
| L1 can't see event-driven nature | D4 system characteristic badges | 4 |
| L2 no event flow direction | D5 pipeline view | 4 |
| L2 no cross-domain data deps | D7 depends_on section | 4 |
| L2 no flow criticality | D8 criticality field | 4 |
| L2 no cron schedule visibility | D6 cron badges | 4 |
| L2 no sub-flow relationships | D9 sub-flow arrows | 4 |
| L3 no stealth config visibility | C2 layers + D13 layer toggle | 3+4 |
| L3 no error recovery distinction | D12 connection behavior | 4 |
| L3 no conditional node execution | B5 parallel conditions (covers main case) | 2 |
| L3 no data flow visibility | C3 connection data + D10 + D11 variable scope | 3+4 |
| Self-review: no transaction/saga | A5 transaction node | 3 |
| Self-review: no parameterized flows | C1 flow templates | 3 |
| Self-review: service domain deps invisible | D7 cross-domain data dependencies | 4 |

**Coverage: 30/30 shortfalls addressed. Nothing left behind.**
