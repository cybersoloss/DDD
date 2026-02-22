# Nexus — Coverage-Driven Product Definition

This document is designed to exercise **every feature documented in the DDD Usage Guide** across all four pillars (Logic, Data, Interface, Infrastructure). Feed it to `/ddd-create` to produce a benchmark project. Every requirement is annotated with the DDD feature it exercises.

---

## App Concept

**Nexus** is an AI-powered content intelligence platform. It ingests content from multiple external sources, runs a multi-agent AI analysis pipeline, routes content through editorial review, then publishes approved content across channels. It includes real-time analytics, user management, notification delivery, and system configuration.

**Tech stack:**
- Language: TypeScript · Runtime: Node.js 20
- Backend: Express 4 · Frontend: Next.js 14
- Database: PostgreSQL 16 · ORM: Prisma
- Cache: Redis 7 · Queue: BullMQ
- Auth: JWT + bcrypt
- UI: shadcn/ui · State: Zustand
- AI providers: Anthropic API (primary), OpenAI API (secondary)
- External: Twitter API (OAuth2), SendGrid, Google Custom Search, Stripe

---

## DDD Feature Coverage Matrix

Every section below is mapped to Usage Guide features. The matrix is the test contract — if the generated specs contain all rows, the DDD methodology is fully validated.

| Pillar | Feature | Exercised By |
|---|---|---|
| **Logic** | trigger: HTTP | All API flows |
| **Logic** | trigger: cron | 6 scheduled flows |
| **Logic** | trigger: event | All cross-domain event flows |
| **Logic** | trigger: webhook | `ingestion/process-webhook` |
| **Logic** | trigger: ipc | `ingestion/watch-filesystem`, `analytics/stream-live-metrics` |
| **Logic** | trigger: ws | `analytics/stream-live-metrics` (WebSocket push) |
| **Logic** | trigger: pattern | `monitoring/detect-anomaly` |
| **Logic** | trigger: manual | `editorial/trigger-bulk-reprocess` |
| **Logic** | trigger: interval | `monitoring/health-check` |
| **Logic** | trigger: event_group | `processing/handle-any-content-event` |
| **Logic** | node: process | All domains |
| **Logic** | node: decision | All domains |
| **Logic** | node: input | All HTTP flows |
| **Logic** | node: terminal (success/error/redirect) | All flows |
| **Logic** | node: data_store (read/write/update/delete) | All domains |
| **Logic** | node: data_store (aggregate) | `analytics/query-analytics` |
| **Logic** | node: data_store (update_where) | `editorial/bulk-approve` |
| **Logic** | node: event (emit/listen) | All cross-domain flows |
| **Logic** | node: service_call | `ingestion/monitor-social-feeds`, `notifications/send-batch-digest` |
| **Logic** | node: ipc_call | `ingestion/watch-filesystem`, `analytics/stream-live-metrics` |
| **Logic** | node: sub_flow (with input/output contract) | `publishing/publish-content` |
| **Logic** | node: loop (with break_condition) | `publishing/retry-failed-publish` |
| **Logic** | node: parallel (failure_policy: all_required) | `publishing/publish-to-social` |
| **Logic** | node: parallel (failure_policy: best_effort) | `notifications/send-realtime-alert` |
| **Logic** | node: collection (filter, sort, deduplicate) | `editorial/list-editorial-queue` |
| **Logic** | node: collection (merge, group_by, aggregate) | `analytics/query-analytics` |
| **Logic** | node: collection (reduce, flatten, first, last, join) | `processing/summarize-batch` |
| **Logic** | node: parse (RSS, HTML, XML, CSV) | `ingestion/fetch-rss-feeds`, `ingestion/process-xml-feed`, `ingestion/import-csv-content`, `ingestion/scrape-web-content` |
| **Logic** | node: transform (schema mode) | `publishing/publish-to-website` |
| **Logic** | node: transform (expression mode) | `analytics/generate-daily-report` |
| **Logic** | node: cache (check/set/invalidate) | `editorial/get-review-stats`, `content/get-content`, `settings/get-settings` |
| **Logic** | node: delay | `publishing/schedule-publication` |
| **Logic** | node: batch | `processing/summarize-batch` |
| **Logic** | node: transaction | `editorial/bulk-approve` |
| **Logic** | node: crypto (hash, encrypt, decrypt, generate_key) | `users/register-user`, `users/login-user`, `users/manage-api-keys` |
| **Logic** | node: llm_call (structured_output) | `processing/classify-content`, `processing/score-quality` |
| **Logic** | node: agent_loop (is_terminal tool) | `processing/fact-check-agent` |
| **Logic** | node: agent_loop (requires_confirmation tool) | `processing/resolve-conflict-agent` |
| **Logic** | node: agent_loop (vector_store memory) | `processing/bias-check-agent` |
| **Logic** | node: agent_loop (conversation_history memory) | `processing/fact-check-agent` |
| **Logic** | node: agent_loop (key_value memory) | `processing/orchestrate-analysis` |
| **Logic** | node: guardrail (input position) | `processing/classify-content` |
| **Logic** | node: guardrail (output position) | `processing/classify-content` |
| **Logic** | node: human_gate (with approval_options + timeout) | `editorial/review-content` |
| **Logic** | node: orchestrator (supervisor strategy) | `processing/orchestrate-analysis` |
| **Logic** | node: orchestrator (round_robin strategy) | `notifications/route-notification` |
| **Logic** | node: orchestrator (broadcast strategy) | `processing/analyze-with-agents` |
| **Logic** | node: orchestrator (consensus strategy) | `editorial/ai-consensus-review` |
| **Logic** | node: smart_router (rule-based) | `publishing/publish-content` |
| **Logic** | node: smart_router (llm_routing) | `processing/route-to-specialist` |
| **Logic** | node: handoff (transfer mode) | `editorial/escalate-to-senior` |
| **Logic** | node: handoff (consult mode) | `processing/route-to-specialist` |
| **Logic** | node: handoff (collaborate mode) | `editorial/co-author-review` |
| **Logic** | node: agent_group (broadcast coordination) | `processing/analyze-with-agents` |
| **Logic** | node: agent_group (round_robin coordination) | `processing/collaborative-review` |
| **Logic** | node: agent_group (sequential coordination) | `processing/pipeline-agents` |
| **Data** | schema: all field types (string, number, boolean, date, enum, json, uuid, text, decimal) | `content`, `user`, `analytics-event` schemas |
| **Data** | schema: relationships (has_many, belongs_to, has_one, many_to_many) | `content→content-version`, `user→role`, `content→channel` |
| **Data** | schema: indexes (unique, composite, btree, gin) | `content` (title full-text), `user` (email unique), `analytics-event` (composite) |
| **Data** | schema: seed data | `role`, `setting`, `channel` schemas |
| **Data** | schema: soft_delete (deleted_at) | `content`, `user` schemas |
| **Data** | schema: state machine transitions | `content` (draft→review→approved→published→archived) |
| **Interface** | ui: stat-card component | `dashboard` page |
| **Interface** | ui: item-list component | `content-feed`, `editorial-queue` pages |
| **Interface** | ui: card-grid component | `sources` page |
| **Interface** | ui: detail-card component | `content-detail` page |
| **Interface** | ui: chart component | `analytics` page |
| **Interface** | ui: filter-bar component | `content-feed`, `editorial-queue` pages |
| **Interface** | ui: form field types (all 14) | `manual-content-entry`, `general-settings`, `notification-settings` forms |
| **Interface** | ui: interaction: bulk-select | `content-feed`, `editorial-queue` pages |
| **Interface** | ui: interaction: inline-edit | `content-detail` page |
| **Interface** | ui: interaction: drag-drop | `sources` page (reorder priority) |
| **Interface** | ui: interaction: reorder | `publishing` queue page |
| **Interface** | ui: state: realtime (WebSocket) | `dashboard`, `analytics` pages |
| **Interface** | ui: state: initial_fetch | All data pages |
| **Interface** | ui: loading/error/empty states | All data pages |
| **Infrastructure** | service: web_server | Express API |
| **Infrastructure** | service: worker | BullMQ worker |
| **Infrastructure** | service: datastore (database) | PostgreSQL |
| **Infrastructure** | service: datastore (cache) | Redis |
| **Infrastructure** | service: proxy | Nginx (production) |
| **Infrastructure** | depends_on | worker→database, worker→cache, api→database, api→cache |
| **Infrastructure** | startup_order | database→cache→api→worker |
| **Infrastructure** | dev_command | Per-service dev commands |
| **Infrastructure** | setup | Database migration, Redis flush |
| **Cross-cutting** | pattern: stealth_http | `ingestion/scrape-web-content`, `ingestion/monitor-social-feeds` |
| **Cross-cutting** | pattern: api_key_resolution | `users/manage-api-keys`, all external API flows |
| **Cross-cutting** | pattern: encryption | `users/register-user`, `users/login-user` |
| **Cross-cutting** | pattern: soft_delete | `content/delete-content` |
| **Cross-cutting** | pattern: content_hashing | `ingestion/process-xml-feed` (deduplication) |
| **Cross-cutting** | pattern: error_handling | All external service_call flows |
| **Domain** | domain role: process | ingestion, processing, editorial, publishing, notifications |
| **Domain** | domain role: entity | content, users, analytics, settings |
| **Domain** | domain role: gateway | (external integrations domain) |
| **Domain** | publishes_events with payload schema | All cross-domain events |
| **Domain** | consumes_events with handled_by_flow | All event consumers |
| **Domain** | event_groups | `processing/handle-any-content-event` |
| **Domain** | stores (Zustand) | `dashboard`, `content-feed` pages |

---

## Logic Pillar — Domains & Flows

### 1. Ingestion (role: process)
Acquires content from all external source types.

- `fetch-rss-feeds` — **trigger: cron** (*/15), parse RSS XML (`parse: rss`), deduplicate by hash (`collection: deduplicate`), emit ContentIngested
- `scrape-web-content` — **trigger: cron** (*/4h), stealth HTTP, `parse: html` with CSS selectors, `collection: filter` duplicates
- `import-csv-content` — **trigger: HTTP POST**, `parse: csv`, `batch` insert, validate with `input`
- `process-webhook` — **trigger: webhook** (HMAC verified), `decision` on payload type, `smart_router` to handler
- `process-xml-feed` — **trigger: event** (SourceUpdated), `parse: xml`, `collection: deduplicate`, `crypto: hash` for content fingerprint
- `monitor-social-feeds` — **trigger: cron** (*/30min), `service_call` Twitter OAuth2, `collection: merge` results
- `manual-content-entry` — **trigger: HTTP POST**, `input` validation, `data_store: write`
- `watch-filesystem` — **trigger: ipc** (file_watch), `ipc_call` to file reader, `parse: csv` or raw text, emit ContentIngested

### 2. Processing (role: process)
AI-powered analysis pipeline. Covers ALL agent and orchestration variants.

- `classify-content` — **trigger: event** (ContentIngested), `guardrail` (input), `llm_call` with `structured_output`, `guardrail` (output), emit ContentClassified
- `summarize-batch` — **trigger: cron** (hourly), `data_store: read` unprocessed batch, `collection: reduce` to batches, `batch` parallel LLM summarize, `collection: flatten` results
- `fact-check-agent` — **trigger: event**, `agent_loop` with web-search tool, submit-verdict `is_terminal` tool, `memory: conversation_history` + `memory: key_value`
- `bias-check-agent` — **trigger: event**, `agent_loop` with `memory: vector_store` (semantic search past cases), structured output
- `resolve-conflict-agent` — **trigger: event**, `agent_loop` with `requires_confirmation: true` tool (human approves before action)
- `score-quality` — **trigger: event**, `guardrail` (input), `llm_call`, `guardrail` (output), `data_store: update`
- `orchestrate-analysis` — **trigger: event**, `orchestrator` strategy: **supervisor**, shared `memory: key_value`, `result_merge_strategy: supervisor_picks`
- `analyze-with-agents` — **trigger: event**, `orchestrator` strategy: **broadcast**, all agents run in parallel
- `pipeline-agents` — **trigger: event**, `orchestrator` strategy: **round_robin**, sequential context passing
- `ai-consensus-review` — **trigger: event**, `orchestrator` strategy: **consensus**, supervisor picks final result
- `route-to-specialist` — **trigger: event**, `smart_router` with `llm_routing`, `handoff` mode: **consult**
- `collaborative-review` — **trigger: event**, `agent_group` coordination: **round_robin**, shared memory
- `handle-any-content-event` — **trigger: event_group** [ContentIngested, ContentUpdated, ContentRestored], routes to unified handler
- `pipeline-agents` — **trigger: event**, `agent_group` coordination: **sequential**, each member enriches result

### 3. Editorial (role: process)
Human review with async AI assistance.

- `review-content` — **trigger: event** (ContentClassified), `human_gate` with approval_options (approve/reject/escalate/defer), Slack + email, 24hr timeout → escalate
- `escalate-to-senior` — **trigger: event**, `handoff` mode: **transfer** (fire and forget to senior editor flow)
- `co-author-review` — **trigger: event**, `handoff` mode: **collaborate** (iterative back-and-forth)
- `assign-reviewer` — **trigger: HTTP POST**, `decision` on workload, `data_store: update_where`
- `list-editorial-queue` — **trigger: HTTP GET**, `collection: filter` + `sort` + `first`/`last` for pagination
- `bulk-approve` — **trigger: HTTP POST**, `transaction` wrapping batch `data_store: update_where`, emit events
- `get-review-stats` — **trigger: HTTP GET**, `data_store: aggregate`, `cache: check`/`set`, TTL 5min
- `trigger-bulk-reprocess` — **trigger: manual** (admin UI button), requeue selected content for processing
- `ai-consensus-review` — **trigger: event**, `orchestrator` strategy: **consensus**, 3 AI reviewers vote

### 4. Publishing (role: process)
Multi-channel content distribution.

- `publish-content` — **trigger: event** (ContentApproved), `smart_router` (rule-based) to channel flows, emit ContentPublished
- `publish-to-website` — called as `sub_flow` with `input`/`output` contracts, `transform` schema mode, HTTP POST to CMS
- `publish-to-social` — `parallel` with `failure_policy: all_required`, Twitter + LinkedIn `service_call`
- `publish-to-newsletter` — `batch` subscribers, `collection: group_by` subscription tier, SendGrid `service_call`
- `schedule-publication` — **trigger: HTTP POST**, `delay` until target datetime, then trigger publish
- `retry-failed-publish` — **trigger: event** (PublishFailed), `loop` with `break_condition: attempts >= 3`, exponential `delay`
- `publish-sse-updates` — **trigger: event**, SSE stream to connected clients via `ipc_call`

### 5. Analytics (role: entity)
Real-time and historical reporting.

- `record-event` — **trigger: event**, `input` validation, `data_store: write`, `data_store: update_where` counters
- `query-analytics` — **trigger: HTTP GET**, `collection: group_by` + `aggregate`, `data_store: aggregate`, `cache: check`/`set` 1min TTL
- `stream-live-metrics` — **trigger: ws** (WebSocket connection), `loop` with `break_condition: connection_closed`, emit metrics every 5s
- `detect-anomaly` — **trigger: pattern** (threshold breach detected), `llm_call` for explanation, emit AlertTriggered
- `generate-daily-report` — **trigger: cron** (daily 2am), `collection: reduce` day's events to stats, `transform` expression mode, `data_store: write`
- `generate-weekly-digest` — **trigger: cron** (Monday 6am), `collection: aggregate`, email via `service_call`

### 6. Users (role: entity)
Auth and user management.

- `register-user` — **trigger: HTTP POST**, `input` validation, `crypto: hash` password, `crypto: generate_key` for API key, `data_store: write`, emit UserRegistered
- `login-user` — **trigger: HTTP POST**, `crypto: hash` compare, issue JWT, `crypto: encrypt` refresh token, `cache: set`
- `manage-api-keys` — **trigger: HTTP CRUD**, `crypto: generate_key`, `crypto: encrypt` for storage, `cache: invalidate`
- `update-user-role` — **trigger: HTTP PATCH**, `decision` on permissions, `data_store: update`, emit RoleChanged
- `list-users` — **trigger: HTTP GET**, `collection: filter` + `sort` + `first`, admin only
- `refresh-token` — **trigger: HTTP POST**, `crypto: decrypt` stored token, validate, `crypto: generate_key` new JWT

### 7. Content (role: entity)
Central content entity and cross-domain event hub.

- `create-content` — **trigger: HTTP POST**, `data_store: write`, emit ContentCreated
- `get-content` — **trigger: HTTP GET**, `cache: check` → `data_store: read`, `transform` schema mode, `cache: set`
- `update-content` — **trigger: HTTP PATCH**, `data_store: update`, `cache: invalidate`, version `data_store: write`
- `delete-content` — **trigger: HTTP DELETE**, soft delete (`data_store: update` deleted_at), cascade emit ContentDeleted

### 8. Notifications (role: process)
Multi-channel alert delivery.

- `send-realtime-alert` — **trigger: event**, `parallel` with `failure_policy: best_effort`, email + Slack `service_call`
- `send-batch-digest` — **trigger: event**, `collection: filter` pending, `batch` format, `service_call` SendGrid
- `get-notification-preferences` — **trigger: HTTP GET**, `cache: check` preferences, `data_store: read`
- `route-notification` — **trigger: event**, `orchestrator` strategy: **round_robin**, cycles through delivery channels

### 9. Settings (role: entity)
System configuration.

- `get-settings` — **trigger: HTTP GET**, `cache: check` 10min TTL, `data_store: read`
- `update-settings` — **trigger: HTTP PATCH**, `input` validation, `data_store: update`, `cache: invalidate`, emit SettingsChanged
- `reset-settings` — **trigger: HTTP POST**, `transaction` wrapping defaults restore, emit SettingsReset
- `manage-rules` — **trigger: HTTP CRUD**, `input` validate rule syntax, `data_store` CRUD

### 10. Monitoring (role: gateway)
Health and anomaly detection — covers `pattern` and `interval` triggers.

- `health-check` — **trigger: interval** (every 60s), ping all services, `parallel` collect results, `decision` on health, emit HealthStatus
- `detect-anomaly` — **trigger: pattern** (metric threshold breach), `llm_call` explain anomaly, `handoff` mode: **transfer** to alerting

---

## Data Pillar — Schemas

Each schema must include: field types, relationships, indexes, soft_delete where applicable, seed data where applicable, and state machine transitions where applicable.

### content
- Fields: `id` (uuid), `title` (string), `body` (text), `summary` (text?), `category` (enum: news/opinion/research/other), `status` (enum: draft/review/approved/rejected/published/archived), `quality_score` (decimal?), `bias_score` (decimal?), `source_type` (string), `source_id` (string?), `metadata` (json?), `deleted_at` (date?), `created_at` (date), `updated_at` (date)
- Relationships: `has_many: content-version`, `has_many: content-rejection`, `belongs_to: content-source`, `many_to_many: channel`
- Indexes: title full-text (gin), composite (status, created_at), unique (source_type, source_id)
- Soft delete: deleted_at
- **State machine transitions:** draft→review, review→approved, review→rejected, approved→published, published→archived, rejected→draft (on_invalid: error)

### user
- Fields: `id` (uuid), `email` (string, unique), `password_hash` (string), `role_id` (uuid), `api_key_hash` (string?), `plan` (enum: free/pro/enterprise), `created_at` (date), `deleted_at` (date?)
- Relationships: `belongs_to: role`, `has_many: api-key`, `has_many: notification`
- Indexes: unique (email), btree (role_id)
- Soft delete: deleted_at

### role
- Fields: `id` (uuid), `name` (string), `permissions` (json)
- Seed data: admin, editor, reviewer, viewer roles with permission sets

### api-key
- Fields: `id` (uuid), `user_id` (uuid), `key_hash` (string), `name` (string), `last_used_at` (date?), `expires_at` (date?)
- Relationships: `belongs_to: user`
- Indexes: btree (user_id), btree (expires_at)

### content-source
- Fields: `id` (uuid), `type` (enum: rss/web/csv/webhook/social/filesystem), `name` (string), `url` (string?), `config` (json), `last_synced_at` (date?), `enabled` (boolean)
- Indexes: btree (type), btree (enabled)
- Seed data: 2 sample RSS sources, 1 webhook source

### content-version
- Fields: `id` (uuid), `content_id` (uuid), `body_snapshot` (text), `changed_by` (uuid), `change_summary` (string?), `changed_at` (date)
- Relationships: `belongs_to: content`, `belongs_to: user` (as changed_by)
- Indexes: composite (content_id, changed_at)

### content-rejection
- Fields: `id` (uuid), `content_id` (uuid), `reason` (text), `rejected_by` (uuid), `rejected_at` (date)
- Relationships: `belongs_to: content`, `belongs_to: user`

### analytics-event
- Fields: `id` (uuid), `event_type` (string), `entity_id` (uuid?), `entity_type` (string?), `actor_id` (uuid?), `metadata` (json?), `occurred_at` (date)
- Indexes: composite (event_type, occurred_at) btree, btree (entity_id), gin (metadata)

### publish-record
- Fields: `id` (uuid), `content_id` (uuid), `channel_id` (uuid), `status` (enum: pending/success/failed), `published_at` (date?), `error` (string?), `attempts` (number)
- Relationships: `belongs_to: content`, `belongs_to: channel`
- **State machine transitions:** pending→success, pending→failed, failed→pending (retry)

### channel
- Fields: `id` (uuid), `name` (string), `type` (enum: website/twitter/linkedin/newsletter/webhook), `config` (json), `enabled` (boolean)
- Indexes: unique (name)
- Seed data: website, newsletter, twitter channels

### schedule
- Fields: `id` (uuid), `content_id` (uuid), `publish_at` (date), `channel_id` (uuid), `created_by` (uuid), `status` (enum: pending/executed/cancelled)
- Relationships: `belongs_to: content`, `belongs_to: channel`
- Indexes: btree (publish_at), composite (status, publish_at)
- **State machine transitions:** pending→executed, pending→cancelled

### notification
- Fields: `id` (uuid), `user_id` (uuid), `type` (string), `title` (string), `body` (text), `read` (boolean), `created_at` (date)
- Relationships: `belongs_to: user`
- Indexes: composite (user_id, read), btree (created_at)

### setting
- Fields: `id` (uuid), `key` (string, unique), `value` (json), `updated_at` (date)
- Indexes: unique (key)
- Seed data: default system settings

### rule
- Fields: `id` (uuid), `type` (enum: ingestion/routing/publishing), `condition` (json), `action` (json), `priority` (number), `enabled` (boolean)
- Indexes: composite (type, enabled, priority)

### weekly-digest
- Fields: `id` (uuid), `period_start` (date), `period_end` (date), `stats` (json), `generated_at` (date)

---

## Interface Pillar — UI Pages

Every page must specify sections (with component types), forms (with field types), state (store/initial_fetch/realtime), and interactions.

### dashboard
- Components: `stat-card` (content ingested/processed/published today), `chart` (hourly ingestion rate), `item-list` (recent alerts), `status-bar` (pipeline health)
- State: `realtime: true` (WebSocket updates), `initial_fetch: [analytics/query-analytics, monitoring/health-check]`
- Interactions: none
- Stores: dashboard Zustand store (metrics, alerts)

### content-feed
- Components: `filter-bar` (status/category/date-range), `item-list` (content cards with status badges), `button-group` (bulk actions)
- Form fields (filter form): `search-select` (category), `multi-select` (status), `date-range` (created between), `toggle` (show deleted)
- Interactions: `bulk-select` (select all/some), `inline-edit` (quick title edit)
- State: `initial_fetch: [content/list-content]`, store for selection state

### content-detail
- Components: `detail-card` (content body + metadata), `item-list` (version history), `button-group` (publish/reject/edit actions)
- Form fields (edit form): `text` (title), `textarea` (body), `markdown` (rich body), `select` (category), `date` (schedule date), `datetime` (publish at), `file` (attach media)
- Interactions: `inline-edit` (title, category)
- State: `initial_fetch: [content/get-content]`

### editorial-queue
- Components: `filter-bar` (reviewer/status/priority), `item-list` (queued items with AI scores), `page-header` (queue stats)
- Form fields (filter): `search-select` (reviewer), `select` (priority), `date` (submitted after), `toggle` (unassigned only)
- Interactions: `bulk-select` (bulk approve/reject), `drag-drop` (reorder priority)
- State: `initial_fetch: [editorial/list-editorial-queue]`, `realtime: true` (new items pushed)

### review-panel
- Components: `detail-card` (content + AI analysis side-by-side), `button-group` (approve/reject/escalate/defer), `item-list` (AI findings)
- Form fields: `textarea` (rejection reason), `search-select` (escalate to reviewer), `toggle` (notify author)
- State: `initial_fetch: [content/get-content, editorial/get-review-stats]`

### sources
- Components: `card-grid` (source cards with status), `button-group` (add/edit/disable), `filter-bar` (type/enabled)
- Form fields (add source): `text` (name, url), `select` (type), `toggle` (enabled), `tag-input` (allowed domains), `number` (rate limit), `color` (label color)
- Interactions: `drag-drop` (reorder fetch priority), `inline-edit` (enable/disable toggle)
- State: `initial_fetch: [ingestion/list-sources]`

### publishing
- Components: `item-list` (publishing queue with channel badges), `status-bar` (channel health), `filter-bar` (channel/status)
- Form fields (schedule): `datetime` (publish at), `multi-select` (channels)
- Interactions: `reorder` (publication queue priority), `bulk-select` (retry failed)
- State: `initial_fetch: [publishing/list-queue]`, `realtime: true`

### analytics
- Components: `chart` (ingestion/processing/publishing time series), `filter-bar` (date-range/event-type), `stat-card` (totals), `item-list` (anomaly alerts)
- Form fields: `date-range` (period), `multi-select` (event types), `select` (granularity: hour/day/week)
- State: `initial_fetch: [analytics/query-analytics]`, `realtime: true` (WebSocket live metrics)
- Interactions: none

### users-admin
- Components: `item-list` (users with role badges), `filter-bar` (role/plan), `page-header` (user counts)
- Form fields (invite user): `text` (email), `select` (role), `select` (plan), `date` (access expires)
- Interactions: `bulk-select` (bulk role change), `inline-edit` (role dropdown)
- State: `initial_fetch: [users/list-users]`

### general-settings
- Components: `detail-card` (settings form sections)
- Form fields: `text` (app name, support email), `number` (rate limits, batch sizes), `toggle` (feature flags), `select` (timezone, language), `slider` (quality threshold 0-100), `color` (brand color), `markdown` (terms of service)
- State: `initial_fetch: [settings/get-settings]`

### notification-settings
- Components: `detail-card` (channel preferences), `item-list` (notification history)
- Form fields: `toggle` (email on/off, Slack on/off), `multi-select` (event types to notify), `select` (digest frequency), `text` (Slack webhook URL)
- State: `initial_fetch: [notifications/get-notification-preferences]`

### api-key-management
- Components: `item-list` (active keys with last-used), `button-group` (create/revoke)
- Form fields (create key): `text` (name), `date` (expires at), `multi-select` (allowed scopes), `tag-input` (IP allowlist)
- State: `initial_fetch: [users/manage-api-keys]`

---

## Infrastructure Pillar

### Services

- `api` — type: server, framework: Express 4, port: 3001, dev_command: `npm run dev:api`, depends_on: [postgres, redis]
- `worker` — type: worker, framework: BullMQ, dev_command: `npm run dev:worker`, depends_on: [postgres, redis]
- `frontend` — type: server, framework: Next.js 14, port: 3000, dev_command: `npm run dev:frontend`, depends_on: [api]
- `postgres` — type: datastore, engine: PostgreSQL 16, port: 5432, dev_command: `brew services start postgresql@16`, setup: `npx prisma migrate dev`
- `redis` — type: datastore, engine: Redis 7, port: 6379, dev_command: `brew services start redis`, setup: `redis-cli flushall`
- `nginx` — type: proxy, port: 80, dev_command: none (production only), depends_on: [api, frontend]

### startup_order
postgres → redis → api → worker → frontend

---

## Cross-Cutting Patterns (architecture.yaml)

All 6 patterns must be defined in `architecture.yaml` with `used_by_domains` and `convention`:

- `stealth_http` — rotate User-Agent, randomized delays, respect robots.txt — used_by: [ingestion]
- `api_key_resolution` — check header → query param → env var → error — used_by: [users, ingestion, processing]
- `encryption` — AES-256 for tokens, bcrypt for passwords, SHA-256 for content hashes — used_by: [users]
- `soft_delete` — filter `deletedAt: null` on all reads — used_by: [content, users]
- `content_hashing` — SHA-256 of (source_id + source_type) for deduplication — used_by: [ingestion]
- `error_handling` — retry 3x exponential, circuit breaker at 5 failures/minute — used_by: [ingestion, publishing, notifications]

---

## Domain Events (with payload schemas)

| Event | Published By | Consumed By | Key Payload Fields |
|---|---|---|---|
| ContentIngested | ingestion | processing | content_id, source_type, raw_url, ingested_at |
| ContentClassified | processing | editorial | content_id, category, confidence, quality_score |
| ContentApproved | editorial | publishing, analytics | content_id, approved_by, approved_at |
| ContentRejected | editorial | content, analytics, notifications | content_id, reason, rejected_by |
| ContentPublished | publishing | analytics, notifications | content_id, channel_ids, published_at |
| PublishFailed | publishing | publishing | content_id, channel_id, error, attempts |
| UserRegistered | users | notifications | user_id, email, plan |
| RoleChanged | users | analytics | user_id, old_role, new_role, changed_by |
| SettingsChanged | settings | all domains | key, old_value, new_value |
| AlertTriggered | monitoring | notifications | alert_type, metric, threshold, current_value |
| HealthStatus | monitoring | analytics | status, services, checked_at |

---

## Node Type + Trigger Type Checklist

Use this as the test contract when scoring generated specs.

**Trigger types (10):**
- [ ] HTTP · [ ] cron · [ ] event · [ ] webhook · [ ] ipc · [ ] ws · [ ] pattern · [ ] manual · [ ] interval · [ ] event_group

**Node types (28):**
- [ ] trigger · [ ] input · [ ] process · [ ] decision · [ ] terminal
- [ ] data_store (read/write/update/delete/aggregate/update_where)
- [ ] service_call · [ ] ipc_call · [ ] event · [ ] sub_flow (with contract)
- [ ] loop (with break_condition) · [ ] parallel (all_required + best_effort)
- [ ] collection (filter/sort/deduplicate/merge/group_by/aggregate/reduce/flatten/first/last/join)
- [ ] parse (rss/html/xml/csv) · [ ] transform (schema + expression modes)
- [ ] cache (check/set/invalidate) · [ ] delay · [ ] batch · [ ] transaction · [ ] crypto
- [ ] llm_call (structured_output) · [ ] agent_loop (is_terminal + requires_confirmation + all memory types)
- [ ] guardrail (input + output positions)
- [ ] human_gate (with approval_options + timeout action)
- [ ] orchestrator (supervisor + round_robin + broadcast + consensus)
- [ ] smart_router (rule-based + llm_routing)
- [ ] handoff (transfer + consult + collaborate)
- [ ] agent_group (broadcast + round_robin + sequential)

**Schema features:**
- [ ] All field types · [ ] has_many · [ ] belongs_to · [ ] has_one · [ ] many_to_many
- [ ] Indexes (unique/composite/gin) · [ ] Seed data · [ ] Soft delete · [ ] State machine transitions

**UI features:**
- [ ] All 9 component types · [ ] All 14 form field types
- [ ] bulk-select · [ ] inline-edit · [ ] drag-drop · [ ] reorder
- [ ] realtime (WebSocket) · [ ] initial_fetch · [ ] loading/error/empty states

**Infrastructure:**
- [ ] server · [ ] worker · [ ] datastore (db + cache) · [ ] proxy
- [ ] depends_on · [ ] startup_order · [ ] dev_command · [ ] setup

**Cross-cutting patterns:**
- [ ] stealth_http · [ ] api_key_resolution · [ ] encryption · [ ] soft_delete · [ ] content_hashing · [ ] error_handling

**Domain features:**
- [ ] process role · [ ] entity role · [ ] gateway role
- [ ] publishes_events with payload · [ ] consumes_events with handled_by_flow · [ ] event_groups
