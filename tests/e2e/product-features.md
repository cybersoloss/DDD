# Nexus — Content Intelligence Platform

## Overview

Nexus is a content intelligence platform that ingests content from multiple sources (RSS feeds, web scraping, APIs, file imports), processes it through AI pipelines for classification, summarization, and quality scoring, routes content through editorial workflows with human approval, and publishes across channels (social media, newsletters, websites, SSE streams). The platform provides real-time analytics via WebSocket dashboards and desktop notifications for editorial alerts.

**Tech Stack:** TypeScript, Node.js 20, Express 4, Next.js 14, PostgreSQL 16, Prisma, Redis 7, BullMQ, JWT + bcrypt, shadcn/ui, Zustand

---

## System Architecture

### Zones

The system is organized into five architectural zones:

1. **Ingestion Zone** — Content acquisition from external sources (RSS, web scraping, APIs, filesystem, manual entry). Contains the `ingestion` domain.
2. **AI Zone** — AI processing pipeline for classification, summarization, quality scoring, and multi-agent content analysis. Contains the `processing` domain.
3. **Editorial Zone** — Human review workflows, bulk operations, and content lifecycle management. Contains the `editorial` and `content` domains.
4. **Distribution Zone** — Multi-channel publishing, SSE streaming, and notification delivery. Contains the `publishing` and `notifications` domains.
5. **Platform Zone** — User management, analytics, and system configuration. Contains the `users`, `analytics`, and `settings` domains.

### Data Flows

- Ingestion Zone → AI Zone: "Raw content for processing" (high volume)
- AI Zone → Editorial Zone: "Classified content for review" (medium volume)
- Editorial Zone → Distribution Zone: "Approved content for publishing" (medium volume)
- Distribution Zone → Platform Zone: "Publishing metrics and events" (low volume, dashed)
- Platform Zone → Ingestion Zone: "Source configuration updates" (low volume, dashed)

### Characteristics

- "Event-driven architecture"
- "Multi-agent AI pipeline"
- "6 cron schedules"
- "6 external integrations"
- "Real-time WebSocket analytics"

### Schedules

- Every 15 minutes: `ingestion/fetch-rss-feeds`
- Every 30 minutes: `ingestion/monitor-social-feeds`
- Every hour: `processing/summarize-batch`
- Every 4 hours: `ingestion/scrape-web-content`
- Daily at 2am: `analytics/generate-daily-report`
- Weekly on Monday 6am: `analytics/generate-weekly-digest`

### Pipelines

**Content Pipeline** — End-to-end from ingestion to publication:
1. `ingestion/fetch-rss-feeds` → emits `ContentIngested`
2. `processing/classify-content` → emits `ContentClassified`
3. `editorial/review-content` → emits `ContentApproved`
4. `publishing/publish-content` → emits `ContentPublished`
Spans domains: ingestion, processing, editorial, publishing

### External Integrations

1. **OpenAI API** (`https://api.openai.com/v1`) — bearer token auth via `OPENAI_API_KEY`, 60 requests/min, 3 retries with exponential backoff, 30s timeout. Used by: processing
2. **Twitter/X API** (`https://api.twitter.com/2`) — OAuth2 via `TWITTER_BEARER_TOKEN`, 300 requests/15min window, 3 retries with 1s backoff. Used by: ingestion, publishing
3. **Anthropic API** (`https://api.anthropic.com/v1`) — API key via `ANTHROPIC_API_KEY`, 60 requests/min, 3 retries with exponential backoff, 60s timeout. Used by: processing
4. **SendGrid API** (`https://api.sendgrid.com/v3`) — API key via `SENDGRID_API_KEY`, 100 requests/second, 2 retries. Used by: notifications
5. **Google Custom Search** (`https://www.googleapis.com/customsearch/v1`) — API key via `GOOGLE_CSE_KEY`, 100 queries/day. Used by: ingestion
6. **Stripe API** (`https://api.stripe.com/v1`) — API key via `STRIPE_SECRET_KEY`, 25 requests/second, 30s timeout. Used by: users

---

## Domains

### 1. Ingestion (process domain)

Acquires content from external sources. Owns schemas: ContentSource, Schedule.

**Event Groups:**
- `ingestion_events`: ContentIngested, ContentSourceAdded, ContentSourceRemoved, IngestionFailed

**Flows:**

1. **fetch-rss-feeds** — Scheduled (cron every 15 min). Reads configured RSS sources from database, loops through each source, fetches the RSS XML, parses it as RSS format with lenient strategy, filters entries newer than last check time using collection filter operation, deduplicates by URL using collection deduplicate, creates content records in batch via data_store create_many. On parse failure, log and continue to next source (connection behavior: continue). Emits ContentIngested event for each new item.

2. **scrape-web-content** — Scheduled (cron every 4 hours). Reads scraping targets from database, loops through each target, makes HTTP GET service_call to fetch the page, parses as HTML format using cheerio with CSS selectors for title (h1), body (article), and links (a[href] multiple). Uses circuit_break connection behavior for repeated failures. Includes a random delay (1000-3000ms) between requests for rate limiting. Transforms extracted data into content format using transform node. Emits ContentIngested.

3. **process-webhook** — Webhook trigger at `/webhooks/content`. Validates incoming webhook payload via input node (fields: source, type, payload — all required). Verifies webhook signature using crypto verify operation with HMAC-SHA256 algorithm, key from WEBHOOK_SECRET env. On valid signature, creates content record in database. On invalid signature, returns 401.

4. **import-csv-content** — HTTP POST `/api/v1/content/import`. Accepts file upload, parses as CSV format with strict strategy, validates each row via collection filter (predicate: item.title exists and item.url exists), creates records using data_store create_many with batch flag. Returns import summary with counts.

5. **manual-content-entry** — Manual trigger from Admin Dashboard. Input node validates fields: title (string, required), url (string), body (text), source_type (string, required), tags (string array). Creates content record in database, emits ContentIngested.

6. **watch-filesystem** — IPC trigger (`ipc:file-system-changed`). Receives file path from native file watcher (bridge: tauri). Reads file content via data_store filesystem read operation, parses as markdown format, creates content record. Emits ContentIngested.

7. **monitor-social-feeds** — Timer trigger (30000ms interval). Service calls to Twitter API to fetch latest posts matching configured keywords. Parses JSON response, filters new items via collection filter, deduplicates against existing content by external_id using collection deduplicate. Batch creates new records.

8. **process-xml-feed** — Event trigger (`event:ExternalFeedReceived`). Parses raw XML payload using fast-xml-parser, transforms into content format, stores in database. Handles malformed XML gracefully with parse error handle routing to an error terminal.

**Domain-level error hook:** All error terminals emit `IngestionFailed` event for centralized error tracking.

### 2. Processing (process domain)

AI-powered content analysis pipeline. Publishes: ContentClassified, ContentSummarized, QualityScored. Consumes: ContentIngested.

**Flows:**

1. **classify-content** — Event trigger (`event:ContentIngested`). Reads content from database with include (joins ContentSource via content_source_id). Makes an llm_call to Claude Sonnet with system prompt "You are a content classifier", prompt template "Classify this content: {content}" with temperature 0.7, structured_output for category and confidence fields, context_sources binding content from `$.content.body` with truncate(4000) transform. Stores classification result. Emits ContentClassified.

2. **summarize-batch** — Cron trigger (every hour) with job_config: queue "ai", concurrency 2, timeout 300000ms, retry max_attempts 3 with exponential backoff, dead_letter true. Reads unsummarized content from database (paginated, cursor style, limit 50). Loops through items, makes llm_call for each with model Claude Haiku, prompt template "Summarize: {content}", max_tokens 200. Accumulates summaries (strategy: append, output: all_summaries). Batch updates content records with summaries via data_store update_many.

3. **score-quality** — Event trigger (`event:ContentClassified`). Uses smart_router with both rules-based and LLM routing. Rules: if category is "news" route to fact-check-agent, if category is "opinion" route to bias-check-agent, with confidence_threshold 0.8 for LLM routing fallback. Fallback chain: [general-scorer]. Routes to appropriate sub_flow for scoring. Each sub_flow has a contract (inputs: content_id uuid required, content_body string required; outputs: score number, reasoning string).

4. **analyze-with-agents** — HTTP POST `/api/v1/content/:id/analyze`. Agent flow type. Starts with a guardrail node (position: input) checking content_length > 0 and no_prohibited_content, on_block returns 422. Then an agent_loop with model Claude Sonnet, system prompt for deep content analysis, max_iterations 10, temperature 0.3, on_max_iterations: respond. Tools: search_web (id: search, name: search_web, description: "Search for related content"), analyze_sentiment (terminal tool), check_facts. Memory: conversation_history with sliding_window strategy, max_tokens 4000. Output guardrail (position: output) checks no_hallucinated_sources, action: warn. Emits QualityScored.

5. **orchestrate-analysis** — Event trigger (`event:ContentClassified`, filter: `confidence` gte 0.9). Uses orchestrator node with supervisor strategy, model Claude Sonnet, supervisor_prompt "Coordinate content analysis agents". Agents: [{id: summarizer, flow: processing/summarize-single, specialization: "Text summarization"}, {id: classifier, flow: processing/classify-content, specialization: "Content classification"}, {id: scorer, flow: processing/score-quality, specialization: "Quality scoring"}]. Shared memory: [{name: content_context, type: key_value, access: read_write}]. Result merge strategy: supervisor_picks. Fallback chain: [summarizer, scorer].

6. **route-to-specialist** — Event trigger (`event:ContentIngested`, filter: `source_type` in [api, webhook]). Uses handoff node with mode: consult, target: {flow: processing/analyze-with-agents}, context_transfer: {include_types: [content, metadata], max_context_tokens: 8000}, on_complete: {return_to: self, merge_strategy: append}, on_failure: {action: escalate, timeout: 60000}.

7. **collaborative-review** — HTTP POST `/api/v1/content/:id/deep-review`. Uses agent_group node with name "Content Review Panel", members: [{flow: processing/classify-content, domain: processing}, {flow: processing/score-quality, domain: processing}], shared_memory: [{name: review_state, type: key_value, access: read_write}], coordination: {communication: shared_memory, max_active_agents: 3, selection_strategy: round_robin}.

### 3. Editorial (process domain)

Human review and approval workflows. Owns schemas: none (operates on Content). Publishes: ContentApproved, ContentRejected, BulkOperationCompleted. Consumes: ContentClassified, QualityScored.

**Flows:**

1. **review-content** — UI action trigger (`ui:ReviewSubmitted`, debounce_ms: 300). Reads content with related data using data_store include (ContentSource via source_id, User via assigned_to). Presents human_gate with notification_channels ["email", "slack"], approval_options: [{id: approve, label: "Approve for Publishing", description: "Content meets quality standards"}, {id: reject, label: "Reject", description: "Content needs revision", requires_input: true}, {id: escalate, label: "Escalate to Senior Editor", description: "Needs senior review"}, {id: defer, label: "Defer to Later"}], timeout: {duration: 86400000, action: escalate}, context_for_human: ["content.title", "content.summary", "quality_score", "classification"]. On approve: update content status, emit ContentApproved. On reject: update with rejection reason, emit ContentRejected.

2. **bulk-approve** — HTTP POST `/api/v1/editorial/bulk-approve`. Input validates content_ids (array, required) and reviewer_notes (string, optional). Wraps operations in a transaction node with isolation serializable, steps: [{action: "Update all content statuses to approved", rollback: "Revert content statuses to pending"}, {action: "Create audit log entries", rollback: "Delete audit entries"}, {action: "Emit approval events"}], rollback_on_error: true. Uses connection behavior: stop (any failure stops the batch). Emits BulkOperationCompleted.

3. **assign-reviewer** — HTTP POST `/api/v1/editorial/assign`. Decision node checks if auto-assignment is enabled. If true, reads available reviewers from database, uses collection sort by workload ascending, takes first reviewer. If false, expects reviewer_id in input. Updates content assignment. Emits event ReviewerAssigned.

4. **list-editorial-queue** — HTTP GET `/api/v1/editorial/queue`. Reads content with status "pending_review" from database with pagination (cursor, default_limit 20, max_limit 100), sort (default: created_at:desc, allowed: [created_at, priority, category]), and include (ContentSource). Returns paginated list.

5. **get-review-stats** — HTTP GET `/api/v1/editorial/stats`. Reads aggregate counts from database: total pending, approved today, rejected today, average review time. Uses cache node with key "editorial:stats", ttl 60000ms, store redis. On cache hit, returns cached data. On cache miss, queries database, stores result in cache, returns.

6. **reject-content** — HTTP POST `/api/v1/editorial/:id/reject`. Input validates reason (string, required). Updates content status with transition enforcement (must be pending_review → rejected). Creates rejection record. Emits ContentRejected.

### 4. Publishing (process domain)

Multi-channel content distribution. Publishes: ContentPublished, PublishFailed. Consumes: ContentApproved.

**Flows:**

1. **publish-content** — Event trigger (`event:ContentApproved`). Reads content and channel configuration. Uses parallel node with conditional branches: [{id: social, label: "Publish to social media", condition: "$.channels.has_social"}, {id: newsletter, label: "Include in newsletter", condition: "$.channels.has_newsletter"}, {id: website, label: "Publish to website", condition: "$.channels.has_website"}], join: all, timeout_ms: 30000. Each branch calls the appropriate publishing sub_flow. On completion, creates PublishRecord in database. Emits ContentPublished.

2. **publish-to-social** — Sub-flow called by publish-content. Service call to Twitter API with POST method, retry: {max_attempts: 3, backoff_ms: 2000, strategy: exponential, jitter: true}. Uses connection behavior: retry. Signs the request payload using crypto sign operation with HMAC-SHA256 algorithm, key from TWITTER_SECRET env. Includes delay node (min_ms: 1000, max_ms: 3000, strategy: random) for rate limiting between posts.

3. **publish-sse-updates** — SSE trigger at `/api/v1/updates/stream`. Subscribes to content changes via data_store memory subscribe operation on store "content-store", selector "latestPublished". Transforms content into SSE event format. Streams to connected clients. Terminal with response_type: sse.

4. **schedule-publication** — HTTP POST `/api/v1/publishing/schedule`. Input validates content_id (uuid, required), publish_at (datetime, required), channels (array, required). Creates Schedule record in database. Emits PublicationScheduled event with delay_ms calculated from publish_at.

5. **retry-failed-publish** — Event trigger (`event:PublishFailed`). Reads failure details from database. Decision: has retry attempts remaining? If true, re-enqueue with incremented attempt count (event node with target_queue: "publishing", priority: 5, delay_ms: 60000). If false, notify editors via ipc_call command "show_notification" with args {title: "Publish Failed", body: "Content failed after max retries"}, bridge: tauri.

### 5. Analytics (entity domain)

Real-time and historical analytics. Owns schemas: AnalyticsEvent.

**Flows:**

1. **stream-live-metrics** — WebSocket trigger at `/api/v1/analytics/live`. Subscribes to real-time metrics stream. Uses collection aggregate operation on incoming events with accumulator (init: {views: 0, engagements: 0}, expression: "{views: acc.views + item.views, engagements: acc.engagements + item.engagements}"), output: "live_metrics". Streams aggregated metrics to connected WebSocket clients. Terminal with response_type: stream.

2. **record-event** — Event trigger (multi-event: [`event:ContentPublished`, `event:ContentApproved`, `event:ContentRejected`]). Creates AnalyticsEvent record in database with JSONB event_data field. Uses data_store with the event data, no caching.

3. **generate-daily-report** — Cron trigger (daily at 2am). Reads analytics events from past 24 hours. Uses collection group_by on event_type, then collection reduce to calculate totals per type. Writes report to filesystem via data_store filesystem create operation, path "$.reports_dir/daily/$.date.json", content "$.report_json", create_parents true. Emits ReportGenerated.

4. **generate-weekly-digest** — Cron trigger (weekly Monday 6am). Reads past 7 days of daily reports from filesystem using data_store filesystem read. Merges data using collection merge operation. Transforms into digest format. Stores digest record in database.

5. **query-analytics** — HTTP GET `/api/v1/analytics`. Input validates date_range (required), granularity (enum: hour/day/week/month), metrics (array). Checks cache with key "analytics:{date_range}:{granularity}", ttl 300000ms, store redis. On miss, queries database with composite index on [event_type, created_at]. Returns aggregated results.

### 6. Users (entity domain)

User management and authentication. Owns schemas: User, ApiKey, Role.

**Flows:**

1. **register-user** — HTTP POST `/api/v1/auth/register`. Input validates email (string, required, format email), password (string, required, min 8), name (string, required). Decision: email available? Uses crypto hash operation with bcrypt algorithm on password, key_source env "BCRYPT_ROUNDS", output_field "hashed_password". Creates User record. Emits UserRegistered.

2. **login-user** — HTTP POST `/api/v1/auth/login`. Input validates email and password. Reads user by email from database. Uses crypto verify operation with bcrypt algorithm on input password against stored hash. Decision: password valid? If true, generates JWT token, updates last_login_at timestamp. Returns user with token.

3. **manage-api-keys** — HTTP POST `/api/v1/users/:id/api-keys`. Crypto generate_key operation with algorithm aes-256-gcm. Encrypts the generated key using crypto encrypt operation with aes-256-gcm, key_source env "ENCRYPTION_KEY", encoding base64. Stores encrypted ApiKey record (encrypted field on schema). Returns the plain key only once.

4. **list-users** — HTTP GET `/api/v1/users`. Reads users from database with pagination (cursor style, default_limit 20), sort (allowed: [name, created_at, role]), include Role relationship. Returns paginated user list.

5. **update-user-role** — HTTP PUT `/api/v1/users/:id/role`. Input validates role (enum: user, editor, admin, required). Reads user from database. Enforces role transition via schema transitions (user → editor → admin, admin → editor → user). Updates user record. Emits UserRoleChanged.

### 7. Content (entity domain)

Central content entity management and event wiring hub. Owns schemas: Content, BaseContent, ContentVersion.

**Flows:**

1. **create-content** — HTTP POST `/api/v1/content`. Input validates title, body, source_id, content_type. Creates Content record in database. Creates initial ContentVersion (data_store create, model ContentVersion). Emits ContentCreated.

2. **get-content** — HTTP GET `/api/v1/content/:id`. Reads Content from database with include (ContentSource via source_id, ContentVersion via has_many). Uses data_store read with safety strict (null-safe property access). Returns content with relations.

3. **update-content** — HTTP PUT `/api/v1/content/:id`. Reads existing content, creates new ContentVersion snapshot, updates Content record. Uses data_store upsert operation with upsert_key ["id"]. Emits ContentUpdated.

4. **delete-content** — HTTP DELETE `/api/v1/content/:id`. Soft-deletes content (sets deleted_at). Uses data_store update operation. Emits ContentDeleted.

### 8. Notifications (process domain)

Alert and notification delivery. Publishes: NotificationSent. Consumes: ContentApproved, ContentRejected, PublishFailed, UserRegistered.

**Flows:**

1. **send-realtime-alert** — Event group trigger (`event_group:editorial_alerts`). The domain defines an event_group named "editorial_alerts" containing events: ContentApproved, ContentRejected, ReviewerAssigned, BulkOperationCompleted. Transforms event into notification format. Makes service_call to SendGrid API for email delivery. Creates Notification record in database with status transition tracking.

2. **send-pattern-alert** — Pattern trigger (`pattern:HighVolumeIngestion`). Pattern config: event ContentIngested, group_by source_type, threshold 50, window "1h". When 50+ items from same source type arrive within an hour, generates alert. Sends via ipc_call to desktop notification system (command: "show_notification", bridge: tauri). Creates Notification record.

3. **send-batch-digest** — Cron trigger (daily at 8am but included in schedules as `processing/summarize-batch` already covers this — this is triggered by `event:ReportGenerated`). Actually: Event trigger (`event:ReportGenerated`). Reads pending notifications from database. Uses batch node to send notifications through multiple channels: operation_template type service_call, dispatch_field "item.channel", configs for email (SendGrid POST), push (internal webhook), and in-app (data_store create). Concurrency 5, on_error continue.

4. **get-notification-preferences** — HTTP GET `/api/v1/notifications/preferences`. Reads user notification preferences from database. Returns preference settings.

### 9. Settings (entity domain)

System configuration and preferences. Owns schemas: Rule, Channel.

**Flows:**

1. **get-settings** — HTTP GET `/api/v1/settings`. Reads settings from database. Uses cache node with key "settings:global", ttl 3600000ms (1 hour), store redis. On hit, returns cached. On miss, reads from database, caches, returns.

2. **update-settings** — HTTP PUT `/api/v1/settings`. Input validates settings object. Updates settings in database. Invalidates cache. Uses data_store memory reset operation on store "settings-store" to clear in-memory state. Emits SettingsUpdated.

3. **manage-rules** — HTTP POST `/api/v1/settings/rules`. Input validates rule name, conditions, actions. Creates or updates Rule record (data_store upsert, upsert_key ["name"]). Emits RulesUpdated.

4. **reset-settings** — Shortcut trigger (`shortcut Cmd+Shift+R`). Reads default settings. Uses data_store memory reset operation on store "settings-store". Overwrites database settings with defaults using data_store update. Emits SettingsReset.

---

## Data Models

### _base (Base Model)
All models inherit: id (uuid, required), created_at (datetime, required), updated_at (datetime, required), deleted_at (datetime, optional — soft delete).

### User
Inherits: _base. Fields: email (string, required, format email, unique, max_length 255), password_hash (string, required, encrypted, max_length 255), name (string, required, max_length 100), role (enum: [user, editor, admin], required, default user), is_verified (boolean, required, default false), last_login_at (datetime, optional), avatar_url (string, optional, max_length 500), preferences (json, optional — notification and display settings).

Indexes: email (unique), [role, created_at] (btree), [last_login_at] (btree).
Relationships: has_many Content (foreign_key: author_id), has_many ApiKey (foreign_key: user_id), belongs_to Role (foreign_key: role_id).
Transitions: field role, states: user → [editor, admin], editor → [user, admin], admin → [editor], on_invalid: reject.
Seed: default-roles, strategy migration, data: [{id: user, label: "User", permissions: ["read"]}, {id: editor, label: "Editor", permissions: ["read", "write", "review"]}, {id: admin, label: "Admin", permissions: ["read", "write", "review", "admin"]}].

### Content
Inherits: _base. Fields: title (string, required, max_length 500), body (text, required), summary (text, optional), source_id (uuid, required), author_id (uuid, optional), content_type (enum: [article, post, news, opinion, tutorial], required), status (enum: [draft, pending_review, approved, published, rejected, archived], required, default draft), quality_score (decimal, optional), classification (json, optional — AI classification results), tags (json, optional — array of tag strings), external_url (string, optional, format url, max_length 2000), published_at (datetime, optional), metadata (json, optional).

Indexes: [source_id] (btree), [status, created_at] (btree — editorial queue), [author_id] (btree), [content_type, status] (btree), [tags] (gin — JSONB array queries), [classification] (gin — JSONB field queries), [external_url] (hash — exact URL lookups), [title] (btree, unique false — search).
Relationships: belongs_to ContentSource (foreign_key: source_id), belongs_to User (foreign_key: author_id), has_many ContentVersion (foreign_key: content_id), has_many PublishRecord (foreign_key: content_id), has_one AnalyticsEvent (foreign_key: content_id).
Transitions: field status, states: draft → [pending_review], pending_review → [approved, rejected], approved → [published, archived], rejected → [draft, archived], published → [archived], archived → [], on_invalid: reject.
Seed: sample-content, strategy fixture, data: [{title: "Getting Started with AI", body: "Introduction to AI concepts...", content_type: article, status: draft, tags: ["ai", "intro"]}, {title: "Breaking News Today", body: "Latest developments...", content_type: news, status: published}].

### BaseContent
Inherits: _base. Description: "Abstract base for content variants — articles, posts, etc. Use as inherits target for specialized content schemas." Fields: title (string, required, max_length 500), body (text, required), content_type (string, required). Note: This schema exists as an `inherits` target demonstrating schema inheritance beyond _base.

### ContentVersion
Inherits: _base. Fields: content_id (uuid, required), version_number (number, required), title (string, required, max_length 500), body (text, required), changed_by (uuid, optional), change_reason (string, optional, max_length 500).
Indexes: [content_id, version_number] (btree, unique).
Relationships: belongs_to Content (foreign_key: content_id).

### ContentSource
Inherits: _base. Fields: name (string, required, max_length 200), source_type (enum: [rss, api, scraper, webhook, manual, filesystem], required), url (string, optional, format url, max_length 2000), config (json, optional — source-specific configuration), is_active (boolean, required, default true), last_fetched_at (datetime, optional), error_count (number, required, default 0).
Indexes: [source_type, is_active] (btree), [url] (btree, unique).
Relationships: has_many Content (foreign_key: source_id).
Seed: default-sources, strategy fixture, data: [{name: "TechCrunch RSS", source_type: rss, url: "https://techcrunch.com/feed/", is_active: true}].

### ApiKey
Inherits: _base. Fields: user_id (uuid, required), name (string, required, max_length 100), key_hash (string, required, encrypted, max_length 500), key_prefix (string, required, max_length 10 — first 8 chars for display), provider (enum: [openai, anthropic, custom], required), is_valid (boolean, required, default true), last_used_at (datetime, optional), expires_at (datetime, optional).
Indexes: [user_id] (btree), [key_prefix] (btree, unique).
Relationships: belongs_to User (foreign_key: user_id).

### PublishRecord
Inherits: _base. Fields: content_id (uuid, required), channel (enum: [twitter, linkedin, newsletter, website], required, ref: publish_channel), platform_post_id (string, optional, max_length 500), status (enum: [pending, published, failed], required, default pending), published_at (datetime, optional), error_message (text, optional), metrics (json, optional — engagement metrics from platform).
Indexes: [content_id, channel] (btree, unique), [status] (btree).
Relationships: belongs_to Content (foreign_key: content_id).
Transitions: field status, states: pending → [published, failed], failed → [pending], published → [], on_invalid: warn.

### AnalyticsEvent
Inherits: _base. Fields: event_type (string, required, max_length 100), content_id (uuid, optional), user_id (uuid, optional), event_data (json, required — JSONB payload with event-specific data), source (string, required, max_length 50), session_id (string, optional, max_length 100).
Indexes: [event_type, created_at] (btree — time-range queries by type), [content_id] (btree), [event_data] (gin — JSONB containment queries on event payload), [session_id] (hash — exact session lookups).
Relationships: belongs_to Content (foreign_key: content_id), belongs_to User (foreign_key: user_id).

### Notification
Inherits: _base. Fields: user_id (uuid, required), type (enum: [email, push, in_app, desktop], required), title (string, required, max_length 200), body (text, required), status (enum: [pending, sent, read, failed], required, default pending), sent_at (datetime, optional), read_at (datetime, optional), channel_data (json, optional).
Indexes: [user_id, status] (btree), [type, created_at] (btree).
Relationships: belongs_to User (foreign_key: user_id).
Transitions: field status, states: pending → [sent, failed], sent → [read], failed → [pending], read → [], on_invalid: log.

### Role
Inherits: _base. Fields: name (string, required, max_length 50, unique), label (string, required, max_length 100), permissions (json, required — array of permission strings), is_system (boolean, required, default false).
Indexes: [name] (btree, unique).
Relationships: has_many User (foreign_key: role_id).
Seed: system-roles, strategy migration, data: [{name: user, label: "User", permissions: ["content.read"], is_system: true}, {name: editor, label: "Editor", permissions: ["content.read", "content.write", "content.review"], is_system: true}, {name: admin, label: "Admin", permissions: ["content.read", "content.write", "content.review", "users.manage", "settings.manage"], is_system: true}].

### Rule
Inherits: _base. Fields: name (string, required, max_length 200, unique), description (text, optional), conditions (json, required — rule conditions), actions (json, required — rule actions), is_active (boolean, required, default true), priority (number, required, default 0).
Indexes: [name] (btree, unique), [is_active, priority] (btree).
Seed: default-rules, strategy script, source: "rules-import.json", count_estimate: 25.

### Channel
Inherits: _base. Fields: name (string, required, max_length 100), channel_type (enum: [social, email, website, rss_out], required), config (json, required), is_active (boolean, required, default true), credentials (json, optional, encrypted — API keys and tokens).
Indexes: [channel_type, is_active] (btree).
Relationships: has_many PublishRecord (foreign_key: channel_id).

### Schedule
Inherits: _base. Fields: content_id (uuid, required), publish_at (datetime, required), channels (json, required — array of channel IDs), status (enum: [scheduled, published, cancelled], required, default scheduled), created_by (uuid, required).
Indexes: [publish_at, status] (btree — upcoming publication query), [content_id] (btree).
Relationships: belongs_to Content (foreign_key: content_id), belongs_to User (foreign_key: created_by, as creator).
Transitions: field status, states: scheduled → [published, cancelled], cancelled → [scheduled], published → [], on_invalid: reject.

### Shared Types (specs/shared/types.yaml)
Enums: publish_channel (values: [twitter, linkedin, newsletter, website]), content_status (values: [draft, pending_review, approved, published, rejected, archived]), source_type (values: [rss, api, scraper, webhook, manual, filesystem]).
Value objects: date_range (fields: start datetime, end datetime).

---

## User Interface

**App Type:** web
**Framework:** Next.js 14
**Router:** app
**State Management:** Zustand
**Component Library:** shadcn/ui
**Theme:** light color scheme, primary color #2563eb, font Inter, border radius 8px

### Navigation
Type: sidebar
Items: Dashboard (icon: home), Content Feed (icon: rss, badge: pending_count), Editorial Queue (icon: clipboard-check, badge: review_count), Analytics (icon: bar-chart), Sources (icon: globe), Publishing (icon: send), Users & Admin (icon: users), Notifications & Settings (icon: bell), Settings (icon: settings)

### Shared Components
1. **content-card** — "Displays content item with title, source badge, quality score, status, and quick actions". Used by: dashboard, content-feed, editorial-queue.
2. **analytics-chart** — "Reusable chart component for time-series and aggregate data visualization". Used by: dashboard, analytics.
3. **source-badge** — "Colored badge showing content source type (RSS, API, Scraper, etc.)". Used by: content-feed, content-detail, editorial-queue.
4. **quality-indicator** — "Visual quality score display with color coding and confidence level". Used by: content-detail, review-panel, editorial-queue.

### Pages

#### 1. Dashboard (route: /, layout: sidebar)
Sections:
- **content-stats**: stat-card, position top-left, label "Total Content", data_source analytics/query-analytics, fields: value "$.total_content", subtitle "items ingested", trend "$.weekly_change"
- **pending-reviews**: stat-card, position top-center, label "Pending Reviews", data_source editorial/get-review-stats, fields: value "$.pending_count", subtitle "awaiting review", urgency rules: [{threshold: 20, level: critical, color: red}, {threshold: 10, level: warning, color: amber}, {threshold: 0, level: normal, color: green}], actions: click navigate /editorial-queue
- **published-today**: stat-card, position top-right, label "Published Today", data_source analytics/query-analytics, fields: value "$.published_today", subtitle "articles published"
- **recent-content**: item-list, position main, label "Recent Content", data_source content/get-content, query: sort created_at, order desc, limit 10, item_template: title "$.title", subtitle "$.source_type · $.created_at", badge "$.status", item_actions: [{label: Review, icon: eye, navigate: "/content/$.id"}, {label: Approve, icon: check, flow: editorial/bulk-approve, args: {content_ids: ["$.id"]}, confirm: true}], empty_state: {message: "No content yet", description: "Add a source to start ingesting", icon: inbox, action: {label: "Add Source", navigate: /sources}}
- **quick-actions**: button-group, position sidebar, label "Quick Actions", buttons: [{label: "New Source", icon: plus, navigate: /sources, variant: primary}, {label: "Manual Entry", icon: edit, flow: ingestion/manual-content-entry, variant: secondary}, {label: "Import CSV", icon: upload, flow: ingestion/import-csv-content, variant: secondary}]
- **system-health**: status-bar, position footer, label "System Health", data_source analytics/stream-live-metrics, fields: items "$.service_statuses", item_template: label "$.service_name", status "$.status", last_sync "$.last_check"

State: store dashboard-store, initial_fetch [analytics/query-analytics, editorial/get-review-stats], realtime analytics/stream-live-metrics
Loading: skeleton. Error: retry-banner. Refresh: auto-30s.

#### 2. Content Feed (route: /content, layout: sidebar)
Sections:
- **header**: page-header, position top, label "Content Feed", buttons: [{label: "Import", icon: upload, flow: ingestion/import-csv-content, variant: secondary}, {label: "Manual Entry", icon: plus, navigate: /content/new, variant: primary}]
- **filters**: filter-bar, position top, fields: [{name: search, type: text, placeholder: "Search content..."}, {name: status, type: select, options: [all, draft, pending_review, approved, published, rejected, archived], default: all}, {name: content_type, type: multi-select, options_source: shared/types.yaml#content_status}, {name: date_range, type: date-range}]
- **content-grid**: card-grid, position main, data_source content/get-content, item_template: title "$.title", subtitle "$.content_type", badge "$.status", footer "$.source_type · Created $.created_at", item_actions: [{label: View, icon: eye, navigate: "/content/$.id"}, {label: Delete, icon: trash, flow: content/delete-content, args: {id: "$.id"}, confirm: true, confirm_message: "Permanently delete this content?", variant: destructive}], pagination: {type: cursor, page_size: 20}, empty_state: {message: "No content found", description: "Adjust filters or add content sources", icon: search}

State: store content-store, initial_fetch [content/get-content]
Loading: skeleton. Error: retry-banner. Refresh: manual.

#### 3. Content Detail (route: /content/:id, layout: sidebar)
Sections:
- **header**: page-header, position top, data_source content/get-content, query {id: "$.route.id"}, fields: title "$.title", subtitle "$.content_type · $.source_type · Created $.created_at", buttons: [{label: "Edit", icon: edit, action: toggle-edit-mode}, {label: "Delete", icon: trash, flow: content/delete-content, args: {id: "$.route.id"}, confirm: true, variant: destructive}]
- **detail-view**: detail-card, position main, data_source content/get-content, query {id: "$.route.id"}, fields: [{label: "Status", value: "$.status", type: badge}, {label: "Quality Score", value: "$.quality_score", type: badge}, {label: "Summary", value: "$.summary", type: markdown}, {label: "Body", value: "$.body", type: markdown}, {label: "Tags", value: "$.tags", type: tag-list}, {label: "Source", value: "$.source.name"}, {label: "Published", value: "$.published_at", type: relative-time}], visible_when: "$.edit_mode == false"
- **version-history**: item-list, position sidebar, label "Version History", data_source content/get-content, fields: items "$.versions", item_template: title "Version $.version_number", subtitle "$.changed_by · $.created_at", empty_state: {message: "No versions yet"}

Forms:
- **edit-content**: label "Edit Content", position main, visible_when "$.edit_mode == true", prefill_source content/get-content, prefill_args {id: "$.route.id"}, fields: [{name: title, type: text, label: "Title", required: true, validation: "min:3, max:500"}, {name: body, type: textarea, label: "Body", required: true}, {name: content_type, type: select, label: "Type", options: [article, post, news, opinion, tutorial]}, {name: tags, type: tag-input, label: "Tags", autocomplete_source: content/get-content, autocomplete_field: tags}], submit: {flow: content/update-content, args: {id: "$.route.id"}, label: "Save Changes", loading_label: "Saving...", success: {message: "Content updated", action: exit-edit-mode}, error: {message: "Failed to update", retry: true}}

State: store content-detail-store, initial_fetch [content/get-content], local: {edit_mode: false}
Loading: skeleton. Error: error-page. Refresh: none.

#### 4. Editorial Queue (route: /editorial, layout: sidebar)
Sections:
- **header**: page-header, position top, label "Editorial Queue", data_source editorial/get-review-stats, fields: subtitle "$.pending_count items pending review", buttons: [{label: "Bulk Approve", icon: check-all, flow: editorial/bulk-approve, variant: primary, confirm: true, confirm_message: "Approve all selected items?"}, {label: "Refresh", icon: refresh, action: refresh-data, variant: ghost}]
- **queue-list**: item-list, position main, label "Pending Review", data_source editorial/list-editorial-queue, item_template: title "$.title", subtitle "$.content_type · $.source_type · Added $.created_at", badge "$.quality_score", item_actions: [{label: Review, icon: eye, navigate: "/editorial/$.id/review"}, {label: Quick Approve, icon: check, flow: editorial/bulk-approve, args: {content_ids: ["$.id"]}, confirm: true}, {label: Reject, icon: x, flow: editorial/reject-content, args: {id: "$.id"}, confirm: true}], pagination: {type: cursor, page_size: 20}

State: store editorial-store, initial_fetch [editorial/list-editorial-queue, editorial/get-review-stats]
Loading: skeleton. Error: retry-banner. Refresh: manual.

#### 5. Review Panel (route: /editorial/:id/review, layout: split)
Sections:
- **content-preview**: detail-card, position main, data_source content/get-content, query {id: "$.route.id"}, fields: [{label: "Title", value: "$.title"}, {label: "Body", value: "$.body", type: markdown}, {label: "Quality Score", value: "$.quality_score"}, {label: "AI Classification", value: "$.classification", type: json}]
- **review-actions**: button-group, position sidebar, label "Review Decision", buttons: [{label: "Approve", icon: check, flow: editorial/review-content, args: {content_id: "$.route.id", decision: "approve"}, variant: primary}, {label: "Reject", icon: x, flow: editorial/reject-content, args: {id: "$.route.id"}, variant: destructive}, {label: "Escalate", icon: arrow-up, flow: editorial/review-content, args: {content_id: "$.route.id", decision: "escalate"}, variant: secondary}]

Forms:
- **rejection-reason**: label "Rejection Reason", position sidebar, visible_when "$.show_rejection_form", fields: [{name: reason, type: textarea, label: "Reason", required: true, placeholder: "Why is this content being rejected?", validation: "min:10"}, {name: severity, type: select, label: "Severity", options: [minor, major, critical], default: minor}], submit: {flow: editorial/reject-content, args: {id: "$.route.id"}, label: "Reject", loading_label: "Rejecting...", variant: destructive, success: {message: "Content rejected", redirect: "/editorial"}, error: {message: "Failed to reject", retry: true}}

State: store review-store, initial_fetch [content/get-content], local: {show_rejection_form: false}
Loading: skeleton. Error: error-page. Refresh: none.

#### 6. Analytics (route: /analytics, layout: sidebar)
Sections:
- **header**: page-header, position top, label "Analytics", buttons: [{label: "Export", icon: download, flow: analytics/generate-daily-report, variant: ghost}]
- **engagement-chart**: chart, position main, label "Content Engagement", data_source analytics/query-analytics, chart_type: line, fields: {series: "$.engagement_series", labels: "$.date_labels", values: "$.engagement_values"}
- **content-by-type**: chart, position main, label "Content by Type", data_source analytics/query-analytics, chart_type: pie, fields: {labels: "$.type_labels", values: "$.type_counts"}
- **top-content**: item-list, position sidebar, label "Top Performing", data_source analytics/query-analytics, query: {sort: engagement, limit: 10}, item_template: title "$.title", subtitle "$.views views · $.engagements engagements"

Forms:
- **date-filter**: label "Date Range", position top, fields: [{name: start_date, type: date, label: "From", required: true}, {name: end_date, type: date, label: "To", required: true}, {name: granularity, type: select, label: "Granularity", options: [hour, day, week, month], default: day}], submit: {flow: analytics/query-analytics, label: "Apply", loading_label: "Loading..."}

State: store analytics-store, initial_fetch [analytics/query-analytics], realtime analytics/stream-live-metrics
Loading: skeleton. Error: retry-banner. Refresh: auto-30s.

#### 7. Sources (route: /sources, layout: sidebar)
Sections:
- **header**: page-header, position top, label "Content Sources", buttons: [{label: "Add Source", icon: plus, action: show-add-form, variant: primary}]
- **source-list**: card-grid, position main, data_source ingestion/list-sources (implicitly: an HTTP GET flow), item_template: title "$.name", subtitle "$.source_type · Last fetched $.last_fetched_at", badge "$.is_active", footer "$.error_count errors", item_actions: [{label: Edit, icon: edit, navigate: "/sources/$.id"}, {label: Toggle, icon: power, flow: ingestion/toggle-source, args: {id: "$.id"}}, {label: Delete, icon: trash, flow: ingestion/delete-source, args: {id: "$.id"}, confirm: true, variant: destructive}]

Forms:
- **add-source**: label "Add Content Source", position modal, fields: [{name: name, type: text, label: "Name", required: true, placeholder: "Source name..."}, {name: source_type, type: select, label: "Type", options: [rss, api, scraper, webhook, manual, filesystem], required: true}, {name: url, type: text, label: "URL", placeholder: "https://...", visible_when: "source_type != 'manual' && source_type != 'filesystem'"}, {name: schedule_interval, type: number, label: "Check Interval (minutes)", min: 5, max: 1440, default: 60, visible_when: "source_type == 'rss' || source_type == 'scraper'"}, {name: config, type: textarea, label: "Configuration (JSON)", placeholder: '{"selectors": {...}}'}, {name: is_active, type: toggle, label: "Active", default: true}], submit: {flow: ingestion/add-source, label: "Add Source", loading_label: "Adding...", success: {message: "Source added successfully"}, error: {message: "Failed to add source", retry: true}}

State: store sources-store, initial_fetch [ingestion/list-sources]
Loading: skeleton. Error: retry-banner. Refresh: manual.

#### 8. Publishing (route: /publishing, layout: sidebar)
Sections:
- **header**: page-header, position top, label "Publishing", buttons: [{label: "Schedule", icon: clock, action: show-schedule-form, variant: primary}]
- **scheduled-items**: item-list, position main, label "Scheduled Publications", data_source publishing/list-scheduled, item_template: title "$.content.title", subtitle "$.channels · Scheduled $.publish_at", badge "$.status", item_actions: [{label: "Publish Now", icon: send, flow: publishing/publish-content, args: {content_id: "$.content_id"}, confirm: true}, {label: "Cancel", icon: x, flow: publishing/cancel-schedule, args: {id: "$.id"}, variant: destructive}]
- **recent-published**: item-list, position sidebar, label "Recently Published", data_source publishing/list-published, query: {limit: 5, sort: published_at, order: desc}, item_template: title "$.content.title", subtitle "$.channel · $.published_at", badge "$.status"

Forms:
- **schedule-publish**: label "Schedule Publication", position modal, fields: [{name: content_id, type: search-select, label: "Content", search_source: content/get-content, display_field: title, value_field: id, required: true}, {name: publish_at, type: datetime, label: "Publish At", required: true}, {name: channels, type: multi-select, label: "Channels", options: [twitter, linkedin, newsletter, website], required: true}, {name: priority, type: slider, label: "Priority", min: 1, max: 10, step: 1, default: 5}], submit: {flow: publishing/schedule-publication, label: "Schedule", loading_label: "Scheduling...", success: {message: "Publication scheduled"}, error: {message: "Failed to schedule", retry: true}}

State: store publishing-store, initial_fetch [publishing/list-scheduled]
Loading: skeleton. Error: retry-banner. Refresh: manual.

#### 9. Users & Admin (route: /admin/users, layout: sidebar, auth_required: true)
Sections:
- **header**: page-header, position top, label "User Management", buttons: [{label: "Invite User", icon: user-plus, action: show-invite-form, variant: primary}]
- **user-list**: item-list, position main, data_source users/list-users, item_template: title "$.name", subtitle "$.email · $.role · Joined $.created_at", badge "$.role", item_actions: [{label: "Edit Role", icon: shield, navigate: "/admin/users/$.id"}, {label: "Disable", icon: ban, flow: users/disable-user, args: {id: "$.id"}, confirm: true, confirm_message: "Disable this user?"}], pagination: {type: cursor, page_size: 20}

Forms:
- **invite-user**: label "Invite User", position modal, fields: [{name: email, type: text, label: "Email", required: true, validation: "email"}, {name: name, type: text, label: "Name", required: true}, {name: role, type: select, label: "Role", options: [user, editor, admin], default: user, required: true}], submit: {flow: users/register-user, label: "Send Invitation", loading_label: "Sending...", success: {message: "Invitation sent"}, error: {message: "Failed to invite user"}}

State: store users-store, initial_fetch [users/list-users]
Loading: skeleton. Error: retry-banner. Refresh: manual.

#### 10. Notification Settings (route: /notifications, layout: sidebar)
Sections:
- **header**: page-header, position top, label "Notifications"
- **notification-list**: item-list, position main, label "Recent Notifications", data_source notifications/get-notification-preferences, item_template: title "$.title", subtitle "$.type · $.created_at", badge "$.status", empty_state: {message: "No notifications", icon: bell-off}

Forms:
- **preferences**: label "Notification Preferences", position sidebar, fields: [{name: email_enabled, type: toggle, label: "Email Notifications", default: true}, {name: push_enabled, type: toggle, label: "Push Notifications", default: false}, {name: desktop_enabled, type: toggle, label: "Desktop Notifications", default: true}, {name: digest_frequency, type: select, label: "Digest Frequency", options: [realtime, hourly, daily, weekly], default: daily}, {name: quiet_hours_start, type: text, label: "Quiet Hours Start", placeholder: "22:00"}, {name: quiet_hours_end, type: text, label: "Quiet Hours End", placeholder: "07:00"}], submit: {flow: notifications/update-preferences, label: "Save Preferences"}

State: store notifications-store, initial_fetch [notifications/get-notification-preferences]
Loading: skeleton. Error: toast. Refresh: none.

#### 11. General Settings (route: /settings, layout: sidebar)
Sections:
- **header**: page-header, position top, label "Settings", buttons: [{label: "Reset All", icon: rotate-ccw, flow: settings/reset-settings, variant: destructive, confirm: true, confirm_message: "Reset all settings to defaults?"}]

Forms:
- **general-settings**: label "General Settings", position main, fields: [{name: site_name, type: text, label: "Site Name", required: true}, {name: site_description, type: textarea, label: "Description"}, {name: default_language, type: select, label: "Default Language", options: [en, es, fr, de, ja], default: en}, {name: timezone, type: select, label: "Timezone", options: [UTC, "America/New_York", "Europe/London", "Asia/Tokyo"]}, {name: items_per_page, type: number, label: "Items Per Page", min: 10, max: 100, default: 20}, {name: theme_color, type: color, label: "Theme Color", default: "#2563eb"}, {name: enable_ai, type: toggle, label: "Enable AI Processing", default: true}, {name: logo, type: file, label: "Logo", accept: ".png,.jpg,.svg", max_size_mb: 5}], submit: {flow: settings/update-settings, label: "Save Settings", loading_label: "Saving...", success: {message: "Settings saved"}, error: {message: "Failed to save", retry: true}}

State: store settings-store, initial_fetch [settings/get-settings]
Loading: skeleton. Error: retry-banner. Refresh: none.

#### 12. API Key Management (route: /settings/api-keys, layout: sidebar)
Sections:
- **header**: page-header, position top, label "API Keys", buttons: [{label: "Generate Key", icon: key, action: show-generate-form, variant: primary}]
- **key-list**: item-list, position main, data_source users/list-api-keys, item_template: title "$.name", subtitle "$.provider · $.key_prefix***", badge "$.is_valid", item_actions: [{label: "Revoke", icon: trash, flow: users/revoke-api-key, args: {id: "$.id"}, confirm: true, confirm_message: "Revoke this API key? This cannot be undone.", variant: destructive}], empty_state: {message: "No API keys", description: "Generate an API key to access the API programmatically", icon: key}

Forms:
- **generate-key**: label "Generate API Key", position modal, fields: [{name: name, type: text, label: "Key Name", required: true, placeholder: "My API Key"}, {name: provider, type: select, label: "Provider", options: [openai, anthropic, custom], required: true}, {name: expires_at, type: date, label: "Expiration Date", allow_empty: true}], submit: {flow: users/manage-api-keys, label: "Generate", loading_label: "Generating...", success: {message: "API key generated. Copy it now — it won't be shown again."}, error: {message: "Failed to generate key"}}

State: store api-keys-store, initial_fetch [users/list-api-keys]
Loading: skeleton. Error: retry-banner. Refresh: manual.

---

## Infrastructure

### Services

1. **backend** — type: server, runtime: Node.js 20, framework: Express 4, entry: src/server/index.ts, port: 3001, health: /api/v1/health, depends_on: [database, cache, queue], dev_command: "npx tsx watch src/server/index.ts"
2. **frontend** — type: server, runtime: Next.js 14, entry: src/app/, port: 3000, depends_on: [backend], dev_command: "npx next dev"
3. **database** — type: datastore, engine: PostgreSQL 16, port: 5432, setup: "npx prisma db push"
4. **cache** — type: datastore, engine: Redis 7, port: 6379
5. **queue** — type: worker, runtime: Node.js 20, engine: BullMQ, entry: src/worker/index.ts, depends_on: [database, cache], dev_command: "npx tsx watch src/worker/index.ts"
6. **proxy** — type: proxy, engine: nginx, port: 80, depends_on: [backend, frontend], config: nginx.conf

Startup order: database, cache, queue, backend, frontend, proxy

Deployment: local strategy process-manager, production strategy docker-compose.

---

## Environment Variables

### Required
- DATABASE_URL (string, sensitive) — PostgreSQL connection string. Example: "postgresql://user:pass@localhost:5432/nexus"
- JWT_SECRET (string, sensitive) — Secret key for JWT signing
- REDIS_URL (string) — Redis connection string. Example: "redis://localhost:6379"
- ENCRYPTION_KEY (string, sensitive) — AES-256 encryption key for data at rest
- ANTHROPIC_API_KEY (string, sensitive) — Anthropic API key for Claude
- OPENAI_API_KEY (string, sensitive) — OpenAI API key
- WEBHOOK_SECRET (string, sensitive) — Webhook signature verification secret

### Optional
- PORT (number, default 3001) — Server port
- LOG_LEVEL (string, default "info") — Logging level
- CORS_ORIGINS (string, default "*") — Comma-separated allowed origins
- RATE_LIMIT_RPM (number, default 60) — Requests per minute per IP
- TWITTER_BEARER_TOKEN (string, sensitive) — Twitter API bearer token
- SENDGRID_API_KEY (string, sensitive) — SendGrid API key
- GOOGLE_CSE_KEY (string, sensitive) — Google Custom Search API key
- STRIPE_SECRET_KEY (string, sensitive) — Stripe secret key
- BCRYPT_ROUNDS (number, default 12) — Bcrypt hashing rounds
- TWITTER_SECRET (string, sensitive) — Twitter API secret for request signing

---

## Error Codes

- VALIDATION_ERROR (422) — "Validation failed: {details}" (warn)
- UNAUTHORIZED (401) — "Authentication required" (warn)
- FORBIDDEN (403) — "Insufficient permissions" (warn)
- NOT_FOUND (404) — "{resource} not found" (info)
- DUPLICATE_ENTRY (409) — "{resource} already exists" (warn)
- RATE_LIMITED (429) — "Too many requests, try again in {retry_after}s" (warn)
- INTERNAL_ERROR (500) — "An unexpected error occurred" (error)
- SERVICE_UNAVAILABLE (503) — "{service} is temporarily unavailable" (error)
- INGESTION_FAILED (502) — "Content ingestion failed: {source}" (error)
- AI_PROCESSING_ERROR (500) — "AI processing failed: {details}" (error)
- PUBLISH_FAILED (502) — "Publishing failed: {channel}" (error)
- INVALID_WEBHOOK_SIGNATURE (401) — "Webhook signature verification failed" (warn)
- CRYPTO_ERROR (500) — "Cryptographic operation failed" (error)
- CONTENT_TRANSITION_INVALID (422) — "Invalid status transition from {from} to {to}" (warn)

---

## DDD Feature Coverage Checklist

This product description is designed to exercise every DDD feature. Here's the traceability:

### Node Types (all 28)
| Node Type | Flow(s) |
|---|---|
| trigger | Every flow |
| input | register-user, manual-content-entry, import-csv-content, bulk-approve, many others |
| process | Various business logic steps throughout |
| decision | register-user (email available?), login-user (password valid?), assign-reviewer, retry-failed-publish |
| terminal | Every flow (success + error terminals) |
| data_store | Every CRUD flow + filesystem (generate-daily-report, watch-filesystem) + memory (update-settings, reset-settings) |
| service_call | scrape-web-content, monitor-social-feeds, publish-to-social, send-realtime-alert |
| ipc_call | retry-failed-publish (desktop notification), send-pattern-alert, watch-filesystem |
| event | Throughout (ContentIngested, ContentClassified, ContentApproved, etc.) |
| loop | fetch-rss-feeds, scrape-web-content, summarize-batch |
| parallel | publish-content (conditional branches) |
| sub_flow | publish-content → publish-to-social, score-quality → specialized scorers |
| llm_call | classify-content, summarize-batch |
| delay | scrape-web-content (rate limit), publish-to-social (rate limit) |
| cache | get-review-stats, query-analytics, get-settings |
| transform | scrape-web-content, generate-daily-report, send-realtime-alert |
| collection | fetch-rss-feeds (filter, deduplicate), stream-live-metrics (aggregate), generate-daily-report (group_by, reduce), generate-weekly-digest (merge), import-csv-content (filter), monitor-social-feeds (filter, deduplicate), process-xml-feed (flatten - via parse + transform) |
| parse | fetch-rss-feeds (rss), scrape-web-content (html), import-csv-content (csv), watch-filesystem (markdown), process-xml-feed (xml), process-webhook (json via input), monitor-social-feeds (json) |
| crypto | process-webhook (verify hmac), register-user (hash bcrypt), login-user (verify bcrypt), manage-api-keys (generate_key, encrypt), publish-to-social (sign hmac) |
| batch | fetch-rss-feeds (create_many), import-csv-content (create_many), send-batch-digest |
| transaction | bulk-approve |
| agent_loop | analyze-with-agents |
| guardrail | analyze-with-agents (input + output) |
| human_gate | review-content (4 options + timeout) |
| orchestrator | orchestrate-analysis (supervisor strategy) |
| smart_router | score-quality (rules + LLM routing) |
| handoff | route-to-specialist (consult mode) |
| agent_group | collaborative-review |

### Trigger Types (all 13)
| Trigger | Flow |
|---|---|
| HTTP GET | list-users, get-content, get-settings, query-analytics, list-editorial-queue |
| HTTP POST | register-user, login-user, import-csv-content, manual-content-entry, bulk-approve |
| HTTP PUT | update-content, update-settings, update-user-role |
| HTTP DELETE | delete-content |
| cron | fetch-rss-feeds, scrape-web-content, summarize-batch, generate-daily-report, generate-weekly-digest |
| event:{Name} | classify-content, record-event, publish-content, send-batch-digest |
| event_group:{name} | send-realtime-alert |
| webhook | process-webhook |
| manual | manual-content-entry |
| shortcut | reset-settings |
| timer | monitor-social-feeds |
| ui:action | review-content |
| ipc:event | watch-filesystem |
| sse | publish-sse-updates |
| ws | stream-live-metrics |
| pattern | send-pattern-alert |

### Collection Operations (all 8)
| Operation | Flow |
|---|---|
| filter | fetch-rss-feeds, import-csv-content, monitor-social-feeds |
| sort | assign-reviewer |
| deduplicate | fetch-rss-feeds, monitor-social-feeds |
| merge | generate-weekly-digest |
| group_by | generate-daily-report |
| aggregate | stream-live-metrics |
| reduce | generate-daily-report |
| flatten | Referenced in parse + transform chains |

### Crypto Operations (all 6)
| Operation | Flow |
|---|---|
| hash | register-user (bcrypt) |
| verify | login-user (bcrypt), process-webhook (hmac) |
| encrypt | manage-api-keys (aes-256-gcm) |
| decrypt | (Implicit when reading encrypted ApiKey fields) |
| sign | publish-to-social (hmac) |
| generate_key | manage-api-keys |

### Data Store Types (all 3)
| Type | Flow |
|---|---|
| database | All CRUD operations |
| filesystem | generate-daily-report (write), watch-filesystem (read), generate-weekly-digest (read) |
| memory | update-settings (reset), reset-settings (reset), publish-sse-updates (subscribe) |

### Connection Behaviors (all 4)
| Behavior | Flow |
|---|---|
| continue | fetch-rss-feeds (parse error → skip source) |
| stop | bulk-approve (any failure stops batch) |
| retry | publish-to-social (API retry with backoff) |
| circuit_break | scrape-web-content (repeated failures) |

### UI Components (all 9)
| Component | Pages |
|---|---|
| stat-card | dashboard |
| item-list | dashboard, content-feed (via card-grid), editorial-queue, sources, publishing, users-admin, notifications-settings, api-keys |
| card-grid | content-feed, sources |
| detail-card | content-detail, review-panel |
| button-group | dashboard, review-panel |
| page-header | content-feed, editorial-queue, analytics, sources, publishing, users-admin, notifications-settings, general-settings, api-keys |
| status-bar | dashboard |
| chart | analytics (line, pie) |
| filter-bar | content-feed |

### Form Field Types (all 14)
| Field Type | Page/Form |
|---|---|
| text | Most forms (names, URLs, etc.) |
| number | sources/add-source (schedule_interval), general-settings (items_per_page) |
| select | content-feed/filters, review-panel, general-settings, many others |
| multi-select | content-feed/filters (content_type), publishing/schedule (channels) |
| search-select | publishing/schedule (content_id) |
| date | analytics/date-filter, api-keys/generate |
| datetime | publishing/schedule (publish_at) |
| date-range | content-feed/filters |
| textarea | content-detail/edit, general-settings, review-panel/rejection |
| toggle | sources/add-source, notification-settings, general-settings |
| tag-input | content-detail/edit |
| file | general-settings (logo) |
| color | general-settings (theme_color) |
| slider | publishing/schedule (priority) |

### Schema Features
| Feature | Schema |
|---|---|
| inherits: _base | All schemas |
| inherits (non-base) | BaseContent (as inherits target) |
| encrypted fields | User (password_hash), ApiKey (key_hash), Channel (credentials) |
| transitions (reject) | User (role), Content (status), Schedule (status) |
| transitions (warn) | PublishRecord (status) |
| transitions (log) | Notification (status) |
| btree index | User, Content, most schemas |
| hash index | Content (external_url), AnalyticsEvent (session_id) |
| gin index | Content (tags, classification), AnalyticsEvent (event_data) |
| gist index | (Available in spec but not explicitly needed — gin covers JSONB use cases) |
| seed: migration | Role (system-roles), User (default-roles) |
| seed: fixture | Content (sample-content), ContentSource (default-sources) |
| seed: script | Rule (default-rules) |
| has_many | User → Content, Content → ContentVersion, etc. |
| belongs_to | Content → ContentSource, ApiKey → User, etc. |
| has_one | Content → AnalyticsEvent |
| many_to_many | (Not explicitly required — covered by join patterns) |

### Orchestrator Strategies (all 4)
| Strategy | Flow |
|---|---|
| supervisor | orchestrate-analysis |
| round_robin | collaborative-review (via coordination.selection_strategy) |
| broadcast | (Possible via orchestrator with broadcast strategy) |
| consensus | (Possible via orchestrator with consensus strategy) |

### Smart Router Modes
| Mode | Flow |
|---|---|
| Rules-based | score-quality |
| LLM routing | score-quality (with confidence_threshold) |

### Handoff Modes (all 3)
| Mode | Flow |
|---|---|
| consult | route-to-specialist |
| transfer | (Available for direct agent handoffs) |
| collaborate | (Available for multi-agent collaboration) |
