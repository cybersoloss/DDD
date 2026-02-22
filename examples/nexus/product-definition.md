# Nexus — Product Definition

This is the product brief used to generate the Nexus DDD specs via `/ddd-create`.
Feed this file to `/ddd-create` in a fresh project to reproduce equivalent specs.

---

## Product Brief

**Nexus** is an AI-powered content intelligence platform. It ingests content from multiple external sources, runs a multi-agent AI analysis pipeline, routes content through editorial review, then publishes approved content to multiple channels. It includes real-time analytics, user management, and system configuration.

---

## Tech Stack

- **Language:** TypeScript
- **Runtime:** Node.js 20
- **Backend framework:** Express 4
- **Frontend:** Next.js 14 with shadcn/ui and Zustand state management
- **Database:** PostgreSQL 16 with Prisma ORM
- **Cache:** Redis 7
- **Queue:** BullMQ (background jobs)
- **Auth:** JWT + bcrypt
- **AI providers:** Anthropic API (primary), OpenAI API (secondary)

---

## Domains

### 1. Ingestion
Acquires content from external sources.

**Flows:**
- `fetch-rss-feeds` — cron every 15min, parse RSS XML, store raw content, emit ContentIngested
- `scrape-web-content` — cron every 4hrs, scrape URLs via CSS selectors, respect robots.txt
- `import-csv-content` — HTTP upload, parse CSV rows, validate and batch-insert
- `process-webhook` — HTTP POST, validate HMAC signature, queue for processing
- `process-xml-feed` — event-triggered, parse XML with field mapping, deduplicate
- `monitor-social-feeds` — cron every 30min, fetch Twitter/LinkedIn via OAuth2, rate-limit aware
- `manual-content-entry` — HTTP POST form submission, validate fields, store draft
- `watch-filesystem` — IPC trigger, watch configured directory, detect new files

**External integrations:** Twitter API (OAuth2), Google Custom Search API, SendGrid

### 2. Processing
AI-powered content analysis. Uses agent flows extensively.

**Flows:**
- `classify-content` — event-triggered, guardrail input validation, LLM classification into categories (news/opinion/research/other), guardrail output, emit ContentClassified
- `summarize-batch` — cron every hour, batch collect unprocessed content, parallel LLM summarization, store results
- `fact-check-agent` — agent_loop with web search tool and submit_verdict terminal tool, max 8 iterations
- `bias-check-agent` — agent_loop detecting ideological or factual bias, structured output
- `score-quality` — guardrail + LLM quality scoring (0-100), store score
- `orchestrate-analysis` — supervisor orchestrator coordinating fact-checker, quality-scorer, bias-detector via shared key_value memory, result_merge_strategy: supervisor_picks
- `analyze-with-agents` — agent_group with broadcast coordination, all three specialists run in parallel
- `collaborative-review` — agent_group with shared memory for consensus building
- `route-to-specialist` — smart_router with LLM routing to specialist agents, handoff mode: consult
- `general-scorer` — guardrail + llm_call for generic scoring
- `bias-check-agent` — agent_loop for bias detection

**External integrations:** Anthropic API, OpenAI API

### 3. Editorial
Human review and approval workflows.

**Flows:**
- `review-content` — human_gate with approval_options (approve/reject/escalate/defer), Slack + email notification, 24hr timeout → escalate
- `assign-reviewer` — HTTP POST, decision on workload, assign to available editor, notify via event
- `list-editorial-queue` — HTTP GET with filters, collection sort/filter, paginate
- `reject-content` — HTTP POST, update status, emit ContentRejected, notify author
- `bulk-approve` — HTTP POST list of IDs, transaction wrapping batch updates, emit events
- `get-review-stats` — HTTP GET, aggregate stats by reviewer/period, cache 5min
- `edit-before-approve` — HTTP PATCH, validate changes, update with version tracking

### 4. Publishing
Distributes approved content to multiple channels.

**Flows:**
- `publish-content` — event-triggered (ContentApproved), smart_router to channel-specific flows, emit ContentPublished
- `publish-to-website` — sub_flow call, transform to page format, HTTP POST to CMS
- `publish-to-social` — parallel publish to Twitter + LinkedIn, collect results
- `publish-to-newsletter` — batch subscribers, SendGrid API, track delivery
- `schedule-publication` — HTTP POST with future datetime, delay node, then trigger publish
- `retry-failed-publish` — event-triggered (PublishFailed), exponential backoff loop, max 3 retries
- `publish-sse-updates` — event-triggered, Server-Sent Events stream to connected clients

### 5. Analytics
Real-time and historical reporting.

**Flows:**
- `record-event` — event-triggered, validate event type, store to analytics table, update counters
- `query-analytics` — HTTP GET with date range, collection aggregate, cache result 1min
- `stream-live-metrics` — IPC trigger, loop emitting current metrics every 5s via SSE
- `generate-daily-report` — cron daily 2am, collect previous day's events, transform to report, store
- `generate-weekly-digest` — cron Monday 6am, aggregate weekly data, email digest to admins

### 6. Users
Authentication and user management.

**Flows:**
- `register-user` — HTTP POST, validate input, hash password (crypto), create user, generate JWT, send welcome email
- `login-user` — HTTP POST, verify password (crypto), issue JWT + refresh token, encrypt refresh token (crypto)
- `manage-api-keys` — HTTP CRUD, generate API key (crypto), encrypt for storage, cache active keys
- `update-user-role` — HTTP PATCH, decision on permissions, update role, emit RoleChanged
- `list-users` — HTTP GET, admin-only, collection filter/sort/paginate
- `refresh-token` — HTTP POST, decrypt stored token (crypto), validate, issue new JWT

### 7. Content
Central content entity management and cross-domain event hub.

**Flows:**
- `create-content` — HTTP POST, validate, store, emit ContentCreated → triggers processing pipeline
- `get-content` — HTTP GET, cache check → DB fallback, transform to response format
- `update-content` — HTTP PATCH, validate diff, update with version increment, invalidate cache
- `delete-content` — HTTP DELETE, soft delete, cascade emit ContentDeleted to all domains

### 8. Notifications
Alert delivery across channels.

**Flows:**
- `send-realtime-alert` — event-triggered, decision on channel preference, parallel email + Slack
- `send-batch-digest` — event-triggered, collect pending alerts, batch format, SendGrid delivery
- `get-notification-preferences` — HTTP GET, cache preferences, return user settings
- `send-pattern-alert` — event-triggered, detect alert patterns, deduplicate, smart_router to channel

### 9. Settings
System configuration.

**Flows:**
- `get-settings` — HTTP GET, cache 10min, return current config
- `update-settings` — HTTP PATCH, validate schema, update, invalidate cache, emit SettingsChanged
- `reset-settings` — HTTP POST, transaction wrapping defaults restore, emit SettingsReset
- `manage-rules` — HTTP CRUD for ingestion/routing rules, validate rule syntax

---

## Data Schemas

- `content` — id, title, body, summary, category, status, quality_score, source_type, source_id, created_at, updated_at, deleted_at
- `base-content` — shared base fields
- `content-source` — id, type (rss/web/csv/webhook/social/filesystem), url, config, last_synced_at, enabled
- `content-version` — id, content_id, body_snapshot, changed_by, changed_at
- `content-rejection` — id, content_id, reason, rejected_by, rejected_at
- `user` — id, email, password_hash, role, api_key_hash, created_at
- `role` — id, name, permissions[]
- `api-key` — id, user_id, key_hash, name, last_used_at, expires_at
- `publish-record` — id, content_id, channel, status, published_at, error
- `schedule` — id, content_id, publish_at, channel, created_by
- `notification` — id, user_id, type, title, body, read, created_at
- `analytics-event` — id, event_type, entity_id, actor_id, metadata, occurred_at
- `weekly-digest` — id, period_start, period_end, stats, generated_at
- `setting` — id, key, value, updated_at
- `rule` — id, type, condition, action, priority, enabled
- `channel` — id, name, type, config, enabled
- `_base` — shared timestamp fields

---

## UI Pages

- `dashboard` — stats cards (content ingested/processed/published today), live metrics chart, recent activity feed
- `content-feed` — filterable list of all content with status badges, bulk actions
- `content-detail` — full content view with analysis results, version history, publish actions
- `editorial-queue` — reviewer queue with filter/sort, assign and review actions
- `review-panel` — side-by-side content + AI analysis, approve/reject/escalate/defer buttons
- `sources` — manage content sources (add/edit/disable), last sync status
- `publishing` — publishing queue and channel status, retry failed
- `analytics` — charts for ingestion/processing/publishing rates, date range picker
- `users-admin` — user list with role management, API key management
- `general-settings` — system config form
- `notification-settings` — per-user notification preferences
- `api-key-management` — create/revoke API keys with expiry

---

## Key Architecture Requirements

- **Event-driven:** All domain boundaries crossed via events (ContentIngested, ContentClassified, ContentApproved, ContentPublished, ContentRejected, RoleChanged, SettingsChanged)
- **6 cron schedules:** RSS (15min), social (30min), scraping (4hr), batch summarize (1hr), daily report (2am), weekly digest (Monday 6am)
- **Multi-agent AI pipeline:** orchestrator (supervisor) → agent_group (broadcast) → individual agent_loop flows with tools
- **Cross-cutting patterns:** encryption (JWT/API keys/refresh tokens), soft_delete (content), stealth_http (external APIs), api_key_resolution
- **All 28 DDD node types must be exercised** — ensure flows include: transaction, human_gate, orchestrator, agent_group, smart_router, handoff, agent_loop, guardrail, llm_call, batch, ipc_call, sub_flow, parallel, cache, crypto, parse, collection, transform, loop, delay, data_store, process, decision, input, event, service_call, terminal, trigger

---

## Node Type Coverage Checklist

When generating specs, verify every node type is present:

| Node Type | Target Flow |
|---|---|
| `transaction` | `editorial/bulk-approve` |
| `human_gate` | `editorial/review-content` |
| `orchestrator` | `processing/orchestrate-analysis` |
| `agent_group` | `processing/analyze-with-agents`, `processing/collaborative-review` |
| `smart_router` | `processing/route-to-specialist`, `publishing/publish-content` |
| `handoff` | `processing/route-to-specialist` |
| `agent_loop` | `processing/fact-check-agent`, `processing/bias-check-agent` |
| `guardrail` | `processing/classify-content`, `processing/score-quality` |
| `llm_call` | `processing/classify-content`, `processing/summarize-batch` |
| `batch` | `processing/summarize-batch` |
| `ipc_call` | `ingestion/watch-filesystem`, `analytics/stream-live-metrics` |
| `sub_flow` | `publishing/publish-content` |
| `parallel` | `publishing/publish-to-social`, `notifications/send-realtime-alert` |
| `cache` | `editorial/get-review-stats`, `content/get-content`, `settings/get-settings` |
| `crypto` | `users/register-user`, `users/login-user`, `users/manage-api-keys` |
| `parse` | `ingestion/fetch-rss-feeds`, `ingestion/process-xml-feed` |
| `collection` | `editorial/list-editorial-queue`, `analytics/query-analytics` |
| `transform` | `publishing/publish-to-website`, `content/get-content` |
| `loop` | `publishing/retry-failed-publish`, `analytics/stream-live-metrics` |
| `delay` | `publishing/schedule-publication` |
| `data_store` | Throughout all domains |
| `process` | Throughout all domains |
| `decision` | Throughout all domains |
| `input` | All HTTP-triggered flows |
| `event` | All cross-domain boundary flows |
| `service_call` | `ingestion/monitor-social-feeds`, `notifications/send-batch-digest` |
| `terminal` | All flows |
| `trigger` | All flows |
