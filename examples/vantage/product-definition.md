# Vantage — AI-Powered Supply Chain Intelligence Platform

## Product Concept

Vantage is a supply chain intelligence platform for operations teams managing suppliers, purchase orders, inventory, logistics, and demand forecasting. It ingests data from ERP systems, carrier APIs, warehouse sensors, and EDI feeds. AI agents score supplier risk, generate demand forecasts, detect anomalies, and automate procurement decisions. Operations teams review flagged orders, approve high-value purchases, and monitor live shipment status.

**Core value:** Replace reactive supply chain management with predictive, AI-assisted decisions — reducing stockouts, late deliveries, and supplier risk exposure.

---

## Tech Stack

- **Runtime:** TypeScript, Node.js 20
- **API:** Express 4
- **Frontend:** Next.js 14, shadcn/ui
- **Database:** PostgreSQL 16, Prisma ORM
- **Queue:** BullMQ (Redis-backed)
- **Cache:** Redis
- **AI:** Anthropic Claude (llm_call, agent_loop, agent_group, orchestrator)
- **Realtime:** Socket.io (WebSocket)
- **File storage:** S3-compatible (MinIO in dev)

---

## DDD Feature Coverage Matrix

Every feature in the DDD Usage Guide is mapped to a specific flow, schema, or UI page.

### Trigger Types

| Trigger | Flow(s) |
|---|---|
| `HTTP` | suppliers/get-supplier, suppliers/list-suppliers, procurement/create-order, procurement/list-orders, inventory/get-stock, inventory/list-items, logistics/list-shipments, users/register, users/login, analytics/get-dashboard, settings/get-config |
| `cron` | forecasting/run-daily-forecast, alerts/send-daily-digest, analytics/generate-weekly-report |
| `event` | inventory/handle-stock-depleted, procurement/handle-order-approved, logistics/handle-shipment-delivered, alerts/handle-sla-breach |
| `webhook` | logistics/receive-carrier-update, suppliers/receive-edi-feed |
| `ipc` | inventory/watch-warehouse-sensors, logistics/stream-shipment-position |
| `ws` | analytics/stream-live-metrics, logistics/track-shipment-live |
| `pattern` | alerts/detect-inventory-anomaly |
| `manual` | procurement/emergency-reorder, suppliers/trigger-risk-review |
| `interval` | inventory/refresh-stock-cache, logistics/poll-carrier-status |
| `event_group` | alerts/handle-supply-chain-event |

### Node Types

| Node Type | Flow(s) | Variant / Feature Exercised |
|---|---|---|
| `trigger` | All entry flows | HTTP, cron, event, webhook, ipc, ws, pattern, manual, interval, event_group |
| `input` | procurement/create-order, users/register, suppliers/onboard-supplier | Input validation with field specs |
| `process` | All domains | Core computation steps |
| `decision` | procurement/approve-order, inventory/check-reorder, alerts/route-alert | Multi-branch routing |
| `terminal` | All flows | success, error, redirect |
| `data_store` | All domains | read, write, update, delete operations |
| `service_call` | logistics/notify-stakeholders, suppliers/enrich-supplier-data, alerts/send-email | External HTTP service integration |
| `ipc_call` | inventory/watch-warehouse-sensors, logistics/stream-shipment-position | IPC to local process |
| `event` | inventory/handle-stock-depleted, procurement/handle-order-approved, logistics/handle-shipment-delivered | Cross-domain events with payloads |
| `loop` | logistics/retry-carrier-api, alerts/escalate-unresolved | break_condition |
| `parallel` | logistics/notify-stakeholders, alerts/send-multi-channel-alert | failure_policy: all_required AND best_effort (both variants) |
| `sub_flow` | logistics/process-fulfillment-order | Nested flow invocation |
| `llm_call` | suppliers/score-supplier-risk, forecasting/generate-ai-forecast | Prompt + structured output |
| `delay` | alerts/send-scheduled-digest | Fixed delay before next node |
| `cache` | inventory/get-stock-levels, analytics/get-dashboard-data, settings/get-config | read-through cache |
| `transform` | logistics/normalize-carrier-event, inventory/format-stock-response | schema mode AND expression mode (both variants) |
| `collection` | inventory/list-stock-items, analytics/query-order-metrics | filter, sort, deduplicate, merge, group_by, aggregate, reduce, flatten, first, last, join (all 11 ops) |
| `parse` | logistics/parse-carrier-webhook-payload, suppliers/parse-edi-document | JSON and XML parsing |
| `crypto` | users/register-user, users/authenticate-user, users/rotate-api-key | hash, verify, encrypt, decrypt, generate |
| `batch` | forecasting/batch-generate-forecasts | Chunked processing |
| `transaction` | inventory/adjust-stock-quantity | Atomic multi-step DB operation |
| `agent_loop` | forecasting/ai-forecast-agent, suppliers/supplier-risk-agent | stop_conditions (max_iterations + goal_reached), is_terminal tool, requires_confirmation tool, all 3 memory types (conversation_history, vector_store, key_value) |
| `guardrail` | procurement/validate-order-safety, forecasting/validate-forecast-output | input position AND output position (both variants) |
| `human_gate` | procurement/approve-high-value-order | Approval with timeout + escalation |
| `orchestrator` | logistics/orchestrate-fulfillment | All 4 strategies: supervisor, round_robin, broadcast, consensus |
| `smart_router` | alerts/route-alert-to-channel, logistics/route-to-carrier | llm_routing with condition_field AND llm_routing: true (both modes) |
| `handoff` | procurement/escalate-to-finance | All 3 modes: transfer, consult, collaborate |
| `agent_group` | forecasting/collaborative-demand-forecast, procurement/multi-agent-procurement-review | All 3 coordination strategies: broadcast, round_robin, sequential |

### Schema Features

| Feature | Schema(s) |
|---|---|
| Typed fields (string, number, boolean, date, enum, json) | All 17 schemas |
| `has_many` relationship | Supplier→PurchaseOrder, Supplier→SupplierContact, PurchaseOrder→OrderLine, Warehouse→StockItem, Shipment→ShipmentEvent |
| `belongs_to` relationship | PurchaseOrder→Supplier, StockItem→Warehouse, ShipmentEvent→Shipment |
| `has_one` relationship | User→UserProfile |
| `many_to_many` relationship | Carrier↔Warehouse (via CarrierWarehouse) |
| Indexes | Supplier(status), PurchaseOrder(status, created_at), StockItem(sku, warehouse_id), Shipment(tracking_number, status) |
| `soft_delete` | Supplier, PurchaseOrder, StockItem, User |
| Seed data | Supplier (3 rows), Warehouse (2 rows), Carrier (3 rows), User (admin + ops user), AlertRule (3 default rules) |
| State machine transitions | PurchaseOrder.status: draft→submitted→approved→fulfilled→cancelled; Shipment.status: pending→in_transit→delivered→failed; StockItem.status: active→low_stock→depleted→reordered |

### UI Component Types

| Component | Page(s) |
|---|---|
| `stat-card` | Dashboard, Inventory Overview, Analytics |
| `item-list` | Suppliers List, Purchase Orders, Inventory Items, Shipments, Alert Rules |
| `card-grid` | Supplier Cards, Warehouse Cards, Forecast Cards |
| `detail-card` | Supplier Detail, Purchase Order Detail, Shipment Detail |
| `chart` | Analytics Dashboard, Forecast Trends, Live Metrics |
| `filter-bar` | Suppliers List, Purchase Orders, Inventory Items, Shipments |
| `button-group` | Purchase Order actions, Bulk inventory actions |
| `page-header` | All 12 pages |
| `status-bar` | Live Tracking, Shipment Detail |

### Form Field Types (all 14)

All 14 field types used across the Purchase Order form and Supplier Onboarding form:

| Field Type | Used In |
|---|---|
| `text` | Supplier name, PO reference, notes |
| `number` | Quantity, unit price, lead time days |
| `select` | Supplier status, PO priority, currency |
| `multi-select` | Alert channels, product categories |
| `search-select` | Supplier lookup, warehouse lookup, carrier lookup |
| `date` | Expected delivery date, contract start date |
| `datetime` | Shipment departure time, alert trigger time |
| `date-range` | Analytics date range, forecast horizon |
| `textarea` | PO notes, supplier remarks, rejection reason |
| `toggle` | Auto-reorder enabled, email notifications enabled |
| `tag-input` | Product tags, supplier capability tags |
| `file` | PO attachment, EDI document upload |
| `color` | Alert rule color-coding |
| `slider` | Risk tolerance threshold, forecast confidence level |
| `markdown` | Supplier contract terms, internal notes |

### Interaction Types (all 4)

| Interaction | Page |
|---|---|
| `bulk-select` | Purchase Orders (bulk approve/reject), Inventory Items (bulk adjust) |
| `inline-edit` | Inventory Items (quantity, reorder threshold), Alert Rules (threshold values) |
| `drag-drop` | Alert Rules (priority ordering), Warehouse zones layout |
| `reorder` | Alert Rules list, Dashboard widgets |

### Real-time (WebSocket)

| Page | WebSocket Data |
|---|---|
| Live Shipment Tracking | Shipment GPS position, status updates, ETA |
| Analytics Dashboard | Live order volume, inventory change rate, alert counts |

### Infrastructure Service Types

| Type | Services |
|---|---|
| `server` | api-server (Express), frontend (Next.js) |
| `worker` | queue-worker (BullMQ jobs: forecasting, alerts, EDI processing) |
| `datastore` | postgres, redis |
| `proxy` | nginx (routes /api → api-server, / → frontend) |

Each service defines: `depends_on`, `startup_order`, `dev_command`, `setup`.

### Architecture Patterns (all 6)

| Pattern | Applied In |
|---|---|
| `stealth_http` | All supplier-facing and carrier-facing API calls |
| `api_key_resolution` | External EDI provider, carrier API authentication |
| `encryption` | PO attachment storage, supplier contract terms |
| `soft_delete` | Supplier, PurchaseOrder, StockItem, User |
| `content_hashing` | EDI document deduplication, carrier webhook deduplication |
| `error_handling` | All flows — standardized error shapes via shared/errors.yaml |

### Domain Events (all cross-domain)

| Event | Producer → Consumer | Payload Schema |
|---|---|---|
| `stock.depleted` | Inventory → Procurement, Alerts | { sku, warehouse_id, quantity, threshold } |
| `order.approved` | Procurement → Logistics, Inventory | { order_id, supplier_id, lines[], expected_date } |
| `order.cancelled` | Procurement → Inventory, Alerts | { order_id, reason, cancelled_by } |
| `shipment.in_transit` | Logistics → Alerts, Analytics | { shipment_id, carrier, tracking_number, eta } |
| `shipment.delivered` | Logistics → Inventory, Analytics, Procurement | { shipment_id, order_id, received_at, lines[] } |
| `supplier.risk_flagged` | Suppliers → Procurement, Alerts | { supplier_id, risk_score, risk_factors[] } |
| `forecast.generated` | Forecasting → Inventory, Procurement | { sku, warehouse_id, forecast_horizon_days, predicted_demand } |
| `alert.triggered` | Alerts → Notifications (all consumers) | { alert_id, rule_id, severity, context{} } |
| `supply_chain.event_group` | Multiple → Alerts | composite event trigger for correlated events |

---

## Logic Pillar — Domains and Flows

### Domain: Suppliers (role: entity)

**Description:** Manages supplier records, contacts, risk scoring, and onboarding.

**Flows:**

1. **list-suppliers** — HTTP GET → collection(filter, sort) → data_store(read) → transform(schema) → terminal
2. **get-supplier** — HTTP GET → cache(read) → data_store(read) → transform(schema) → terminal
3. **onboard-supplier** — HTTP POST → input(validation) → guardrail(input: validate business rules) → data_store(write) → service_call(enrich via external API) → event(supplier.created) → terminal
4. **update-supplier** — HTTP PUT → input → data_store(update) → cache(invalidate) → terminal
5. **score-supplier-risk** — manual trigger → llm_call(risk scoring prompt) → data_store(write risk_score) → decision(score threshold) → event(supplier.risk_flagged) → terminal
6. **trigger-risk-review** — manual trigger → supplier-risk-agent[agent_loop: stop_conditions(max_iterations=10, goal_reached), memory(vector_store for research, key_value for findings), tools(search_suppliers is_terminal=false, flag_supplier requires_confirmation=true)] → guardrail(output: validate risk assessment) → data_store(write) → event(supplier.risk_flagged) → terminal
7. **receive-edi-feed** — webhook trigger → parse(XML EDI document) → transform(expression: normalize to internal schema) → decision(document type) → sub_flow(process-edi-document) → terminal

### Domain: Procurement (role: process)

**Description:** Purchase order lifecycle from requisition through approval and fulfillment.

**Flows:**

1. **list-orders** — HTTP GET → collection(filter, sort, group_by) → data_store(read) → transform(schema) → terminal
2. **create-order** — HTTP POST → input(full PO form validation) → guardrail(input: budget limits, supplier status check) → data_store(write) → event(order.created) → terminal
3. **approve-high-value-order** — event trigger (order.created where total > threshold) → human_gate(approver assignment, timeout=24h, escalation→finance manager) → decision(approved/rejected) → data_store(update status) → event(order.approved OR order.cancelled) → terminal
4. **escalate-to-finance** — event trigger → handoff(mode: transfer → finance agent; consult → legal on terms; collaborate → CFO for strategic orders) → terminal
5. **emergency-reorder** — manual trigger → decision(stock level check) → multi-agent-procurement-review[agent_group: broadcast strategy → [demand-forecaster, supplier-scorer, risk-checker], then sequential for approval chain] → data_store(create emergency PO) → event(order.approved) → terminal
6. **validate-order-safety** — sub_flow invoked by create-order → guardrail(input: check sanctioned suppliers) → guardrail(output: validate line totals) → terminal
7. **bulk-approve-orders** — HTTP POST → transaction[begin → loop(order list, break_condition: error) → data_store(update each) → event(order.approved) each → commit OR rollback] → terminal

### Domain: Inventory (role: entity)

**Description:** Stock levels, warehouse management, replenishment triggers.

**Flows:**

1. **list-stock-items** — HTTP GET → collection(filter, sort, deduplicate, aggregate: sum by warehouse) → data_store(read) → transform(schema) → terminal
2. **get-stock-levels** — HTTP GET → cache(read-through) → data_store(read) → transform(schema) → cache(write) → terminal
3. **adjust-stock-quantity** — HTTP POST → input → transaction[begin → data_store(read current) → decision(sufficient stock) → data_store(update quantity) → decision(below threshold) → event(stock.depleted) → commit OR rollback on error] → terminal
4. **handle-stock-depleted** — event trigger (stock.depleted) → decision(auto-reorder enabled) → service_call(procurement API) → terminal
5. **watch-warehouse-sensors** — ipc trigger → loop(sensor stream, break_condition: disconnect) → decision(anomaly threshold) → event(sensor.alert) → terminal
6. **refresh-stock-cache** — interval trigger (every 5 min) → collection(filter: active items) → data_store(read) → cache(write bulk) → terminal

### Domain: Logistics (role: process)

**Description:** Shipment lifecycle, carrier integrations, fulfillment orchestration.

**Flows:**

1. **list-shipments** — HTTP GET → collection(filter, sort, join: carrier data) → data_store(read) → transform(schema) → terminal
2. **receive-carrier-update** — webhook trigger → parse(JSON carrier payload) → transform(expression: map to ShipmentEvent schema) → data_store(write ShipmentEvent) → decision(terminal status) → event(shipment.delivered OR shipment.in_transit) → terminal
3. **process-fulfillment-order** — event trigger (order.approved) → orchestrate-fulfillment[orchestrator: supervisor strategy → agents[carrier-selector, route-optimizer, customs-checker]; round_robin for load balancing; broadcast for notifications; consensus for carrier selection] → data_store(write Shipment) → event(shipment.in_transit) → terminal
4. **route-to-carrier** — sub_flow invoked by process-fulfillment-order → smart_router(condition_field: shipment.type → [express_carrier, standard_carrier, freight_carrier]; llm_routing: true for international routing decisions) → service_call(carrier booking API) → terminal
5. **notify-stakeholders** — event trigger → parallel(failure_policy: best_effort → [service_call(email), service_call(SMS), service_call(Slack), ipc_call(internal notify)]) → terminal
6. **retry-carrier-api** — sub_flow invoked on carrier API failure → loop(max 3 retries, break_condition: success OR max_retries) → delay(exponential: 2^attempt seconds) → service_call(carrier API) → terminal
7. **stream-shipment-position** — ipc trigger → ws broadcast → loop(position stream, break_condition: delivered OR disconnect) → data_store(write ShipmentEvent) → terminal
8. **track-shipment-live** — ws trigger → cache(read shipment) → terminal (real-time position feed)
9. **poll-carrier-status** — interval trigger (every 2 min) → collection(filter: in_transit shipments) → batch(chunk=20 → service_call(carrier status API)) → data_store(bulk update) → terminal

### Domain: Forecasting (role: process)

**Description:** AI-powered demand forecasting, trend analysis, batch forecast generation.

**Flows:**

1. **run-daily-forecast** — cron trigger (daily 02:00) → collection(filter: active SKUs) → batch(chunk=50 → batch-generate-forecasts sub_flow) → event(forecast.generated bulk) → terminal
2. **batch-generate-forecasts** — sub_flow invoked by run-daily-forecast → batch(process SKU chunk → llm_call(demand forecast prompt with historical data)) → guardrail(output: validate forecast ranges) → data_store(write ForecastData) → terminal
3. **generate-ai-forecast** — HTTP POST (on-demand) → ai-forecast-agent[agent_loop: stop_conditions(max_iterations=15, goal_reached: confidence>0.85), tools(query_historical_data is_terminal=false, search_market_trends is_terminal=false, finalize_forecast is_terminal=true requires_confirmation=false), memory(conversation_history: full context, vector_store: historical patterns, key_value: intermediate calculations)] → guardrail(output: confidence ≥ 0.7, values > 0) → data_store(write Forecast) → event(forecast.generated) → terminal
4. **collaborative-demand-forecast** — HTTP POST → collaborative-forecast[agent_group: broadcast → [statistical-forecaster, ml-forecaster, market-analyst]; round_robin for tie-breaking; sequential for final review chain] → transform(schema: merge forecasts) → data_store(write consensus Forecast) → terminal
5. **validate-forecast-output** — sub_flow → guardrail(input: required fields present) → guardrail(output: values within acceptable bounds, no negative demand) → terminal

### Domain: Analytics (role: process)

**Description:** Dashboards, KPI reports, real-time metrics streaming.

**Flows:**

1. **get-dashboard-data** — HTTP GET → cache(read-through, TTL=60s) → collection(aggregate: orders by status, inventory by warehouse, shipments by carrier) → transform(schema: dashboard DTO) → terminal
2. **query-order-metrics** — HTTP POST → collection(filter, group_by: date/supplier/category, aggregate: sum/avg/count, sort, first, last) → data_store(read) → transform(schema) → terminal
3. **generate-weekly-report** — cron trigger (Monday 06:00) → collection(filter: last 7 days) → batch(generate PDF sections) → service_call(email report) → terminal
4. **stream-live-metrics** — ws trigger → loop(ws stream, break_condition: disconnect) → collection(aggregate: real-time counts) → data_store(read) → terminal
5. **detect-inventory-anomaly** — pattern trigger (rolling window: 3 stockouts in 1h) → decision(severity level) → event(alert.triggered) → terminal

### Domain: Alerts (role: gateway)

**Description:** Alert rules, multi-channel notifications, escalation, anomaly detection.

**Flows:**

1. **handle-supply-chain-event** — event_group trigger (stock.depleted + order.cancelled + shipment.delivered composite) → decision(correlated event pattern) → decision(severity classification) → route-alert-to-channel sub_flow → terminal
2. **route-alert-to-channel** — sub_flow → smart_router(condition_field: alert.severity → [critical_pager, high_slack, medium_email, low_log]; llm_routing: false — rule-based routing) → parallel(failure_policy: all_required → [service_call(primary channel), data_store(write AlertEvent)]) → terminal
3. **send-multi-channel-alert** — event trigger (alert.triggered) → parallel(failure_policy: best_effort → [service_call(email), service_call(Slack), service_call(PagerDuty), service_call(webhook)]) → terminal
4. **escalate-unresolved** — interval trigger (every 30 min) → collection(filter: unresolved alerts older than SLA) → loop(alert list, break_condition: list empty) → decision(escalation level) → service_call(escalate) → data_store(update alert) → terminal
5. **send-daily-digest** — cron trigger (daily 08:00) → delay(randomize dispatch: 0-5 min to avoid thundering herd) → collection(filter: last 24h alerts, group_by: severity) → service_call(send digest email) → terminal

### Domain: Users (role: entity)

**Description:** Authentication, authorization, API key management.

**Flows:**

1. **register-user** — HTTP POST → input(registration form) → crypto(hash password: bcrypt) → data_store(write User) → service_call(send welcome email) → terminal
2. **authenticate-user** — HTTP POST → input → data_store(read User by email) → crypto(verify password) → decision(valid) → crypto(generate JWT) → terminal
3. **rotate-api-key** — HTTP POST → crypto(generate: secure random key) → crypto(hash: store hashed) → data_store(update User.api_key_hash) → terminal
4. **list-users** — HTTP GET → collection(filter, sort) → data_store(read) → transform(schema: redact sensitive fields) → terminal
5. **get-user-profile** — HTTP GET → cache(read) → data_store(read) → transform(schema) → terminal

### Domain: Settings (role: entity)

**Description:** System configuration, integration credentials, preferences.

**Flows:**

1. **get-config** — HTTP GET → cache(read-through) → data_store(read SystemConfig) → transform(schema: redact secrets) → terminal
2. **update-config** — HTTP PUT → input → data_store(update) → cache(invalidate) → event(config.updated) → terminal
3. **handle-sla-breach** — event trigger (alert.triggered where type=sla_breach) → decision(breach severity) → service_call(notify ops lead) → data_store(write SLABreachRecord) → terminal

---

## Data Pillar — Schemas

### 1. Supplier
**Fields:** id (uuid), name (string, required), code (string, unique), status (enum: active|suspended|inactive|onboarding), risk_score (number, nullable), risk_factors (json[]), country (string), currency (string, default: USD), lead_time_days (number), auto_approve_below (number), notes (text, nullable), deleted_at (datetime, nullable)
**Relationships:** has_many PurchaseOrder, has_many SupplierContact
**Indexes:** status, country, risk_score
**Soft delete:** yes (deleted_at)
**State machine:** status: onboarding→active, active→suspended, suspended→active, active→inactive
**Seed data:** 3 suppliers (2 active, 1 onboarding)

### 2. SupplierContact
**Fields:** id (uuid), supplier_id (uuid, FK), name (string), role (string), email (string), phone (string, nullable), is_primary (boolean, default: false)
**Relationships:** belongs_to Supplier
**Indexes:** supplier_id, is_primary

### 3. PurchaseOrder
**Fields:** id (uuid), reference (string, unique), supplier_id (uuid, FK), status (enum: draft|submitted|approved|fulfilled|cancelled), priority (enum: low|standard|high|emergency), total_amount (number), currency (string), expected_date (date), approved_by (uuid, nullable), approved_at (datetime, nullable), notes (text, nullable), attachment_url (string, nullable), deleted_at (datetime, nullable)
**Relationships:** belongs_to Supplier, has_many OrderLine
**Indexes:** status, supplier_id, created_at, priority
**Soft delete:** yes
**State machine:** draft→submitted, submitted→approved, submitted→cancelled, approved→fulfilled, approved→cancelled

### 4. OrderLine
**Fields:** id (uuid), order_id (uuid, FK), sku (string), description (string), quantity (number), unit_price (number), total (number, computed)
**Relationships:** belongs_to PurchaseOrder
**Indexes:** order_id, sku

### 5. Warehouse
**Fields:** id (uuid), name (string), code (string, unique), location (string), capacity (number), is_active (boolean, default: true)
**Relationships:** has_many StockItem
**Seed data:** 2 warehouses (primary, overflow)

### 6. StockItem
**Fields:** id (uuid), sku (string), name (string), warehouse_id (uuid, FK), quantity (number, default: 0), reorder_threshold (number), auto_reorder (boolean, default: false), unit_cost (number), status (enum: active|low_stock|depleted|reordered), deleted_at (datetime, nullable)
**Relationships:** belongs_to Warehouse
**Indexes:** sku, warehouse_id, status
**Soft delete:** yes
**State machine:** active→low_stock (quantity ≤ threshold), low_stock→depleted (quantity = 0), depleted→reordered (PO created), reordered→active (shipment delivered)

### 7. Carrier
**Fields:** id (uuid), name (string), code (string, unique), api_url (string), supported_regions (string[]), service_types (enum[]: standard|express|freight), is_active (boolean)
**Relationships:** many_to_many Warehouse (via CarrierWarehouse)
**Seed data:** 3 carriers (standard, express, freight)

### 8. CarrierWarehouse (junction)
**Fields:** id (uuid), carrier_id (uuid, FK), warehouse_id (uuid, FK), is_preferred (boolean)
**Relationships:** belongs_to Carrier, belongs_to Warehouse
**Indexes:** carrier_id, warehouse_id (unique composite)

### 9. Shipment
**Fields:** id (uuid), order_id (uuid, FK), carrier_id (uuid, FK), tracking_number (string, unique, nullable), status (enum: pending|in_transit|delivered|failed), origin_warehouse_id (uuid, FK), destination (string), departed_at (datetime, nullable), delivered_at (datetime, nullable), eta (datetime, nullable)
**Relationships:** belongs_to PurchaseOrder, belongs_to Carrier, has_many ShipmentEvent
**Indexes:** tracking_number, status, order_id
**State machine:** pending→in_transit, in_transit→delivered, in_transit→failed, failed→in_transit (retry)

### 10. ShipmentEvent
**Fields:** id (uuid), shipment_id (uuid, FK), type (enum: departed|in_transit|out_for_delivery|delivered|exception), location (string, nullable), lat (number, nullable), lng (number, nullable), carrier_status_code (string), occurred_at (datetime)
**Relationships:** belongs_to Shipment
**Indexes:** shipment_id, occurred_at

### 11. Forecast
**Fields:** id (uuid), sku (string), warehouse_id (uuid, FK), forecast_horizon_days (number), predicted_demand (number), confidence (number), model_version (string), generated_at (datetime)
**Relationships:** belongs_to Warehouse, has_many ForecastDataPoint
**Indexes:** sku, warehouse_id, generated_at

### 12. ForecastDataPoint
**Fields:** id (uuid), forecast_id (uuid, FK), date (date), value (number), lower_bound (number), upper_bound (number)
**Relationships:** belongs_to Forecast
**Indexes:** forecast_id, date

### 13. AlertRule
**Fields:** id (uuid), name (string), type (enum: stock_level|sla_breach|supplier_risk|shipment_delay|anomaly), severity (enum: low|medium|high|critical), threshold (number, nullable), channels (string[]), is_active (boolean), color (string), priority_order (number)
**Seed data:** 3 default rules (low stock, shipment delay, supplier risk)
**Indexes:** type, severity, is_active

### 14. AlertEvent
**Fields:** id (uuid), rule_id (uuid, FK), type (string), severity (enum), context (json), triggered_at (datetime), resolved_at (datetime, nullable), acknowledged_by (uuid, nullable)
**Relationships:** belongs_to AlertRule
**Indexes:** rule_id, severity, triggered_at, resolved_at

### 15. User
**Fields:** id (uuid), email (string, unique), name (string), role (enum: admin|ops_manager|analyst|viewer), password_hash (string), api_key_hash (string, nullable), last_login_at (datetime, nullable), deleted_at (datetime, nullable)
**Relationships:** has_one UserProfile
**Indexes:** email, role
**Soft delete:** yes
**Seed data:** admin user + ops_manager user

### 16. UserProfile
**Fields:** id (uuid), user_id (uuid, FK, unique), avatar_url (string, nullable), timezone (string, default: UTC), notification_prefs (json)
**Relationships:** belongs_to User
**Indexes:** user_id

### 17. SystemConfig
**Fields:** id (uuid), key (string, unique), value (text), is_secret (boolean), description (string, nullable), updated_by (uuid, FK, nullable)
**Indexes:** key, is_secret

---

## Interface Pillar — UI Pages

### 1. Dashboard (/)
**Components:** page-header, stat-card (×4: open orders, low stock alerts, active shipments, forecast accuracy), chart (weekly order volume bar chart, inventory trend line chart), filter-bar (date-range, warehouse select), button-group (refresh, export)
**Realtime:** WebSocket — live alert count badge, live shipment status counter
**Interactions:** — (read-only overview)

### 2. Suppliers List (/suppliers)
**Components:** page-header (+ Onboard Supplier button), filter-bar (status multi-select, country select, risk_score slider), item-list (name, code, status, risk_score, last_order_date), button-group (export CSV)
**Interactions:** bulk-select (bulk suspend), inline-edit (risk_score override)
**Form fields used:** search (filter text), multi-select (status), select (country), slider (risk threshold)

### 3. Supplier Detail (/suppliers/:id)
**Components:** page-header (breadcrumb + actions), detail-card (supplier info, contacts), item-list (purchase orders for this supplier), chart (order history bar chart), status-bar (risk score gauge)
**Interactions:** inline-edit (notes, lead_time_days)

### 4. Onboard Supplier (/suppliers/new)
**Form fields:** text (name, code), select (country, currency), number (lead_time_days, auto_approve_below), toggle (active on creation), tag-input (capabilities), textarea (notes), markdown (contract terms), file (initial contract upload)
**Interactions:** — (form submit)

### 5. Purchase Orders (/orders)
**Components:** page-header (+ Create Order button), filter-bar (status multi-select, priority select, date-range, search-select supplier), item-list (reference, supplier, status, total, expected_date), button-group (bulk approve, bulk export)
**Interactions:** bulk-select (bulk approve/reject/cancel), inline-edit (expected_date, notes)

### 6. Create / Edit Purchase Order (/orders/new, /orders/:id/edit)
**Form fields:** text (reference, notes), search-select (supplier lookup), select (priority, currency), number (quantities, unit prices), date (expected_date), datetime (requested_by time), date-range (delivery window), textarea (notes), toggle (attach to existing contract), tag-input (internal tags), file (attachment), color (urgency color tag), slider (budget flexibility %), markdown (internal notes)
**All 14 field types used on this page + order lines sub-table with inline-edit**

### 7. Inventory (/inventory)
**Components:** page-header, filter-bar (warehouse select, status multi-select, search text), item-list (sku, name, warehouse, quantity, status, reorder_threshold), stat-card (×3: total SKUs, low stock count, depleted count), button-group (bulk adjust, export)
**Interactions:** bulk-select (bulk adjust quantities), inline-edit (reorder_threshold, auto_reorder toggle)

### 8. Warehouse Map (/warehouses)
**Components:** page-header, card-grid (warehouse cards with capacity gauge), filter-bar (is_active toggle)
**Interactions:** drag-drop (assign carrier to warehouse), reorder (warehouse priority order)

### 9. Shipments (/shipments)
**Components:** page-header, filter-bar (status multi-select, carrier select, date-range), item-list (tracking_number, carrier, status, origin, destination, eta), button-group (export)
**Interactions:** bulk-select (bulk mark received)
**Realtime:** WebSocket — live status updates in item-list rows

### 10. Live Shipment Tracking (/shipments/:id/track)
**Components:** page-header, detail-card (shipment info), status-bar (timeline: departed→in_transit→delivered), chart (GPS position map placeholder), item-list (ShipmentEvent history)
**Realtime:** WebSocket — live GPS position, status push, ETA recalculation

### 11. Forecasts (/forecasts)
**Components:** page-header (+ Generate Forecast button), filter-bar (sku search-select, warehouse select, date-range), card-grid (forecast cards: sku, predicted demand, confidence), chart (forecast vs actual line chart)
**Interactions:** drag-drop (reorder forecast priority queue)

### 12. Analytics (/analytics)
**Components:** page-header, filter-bar (date-range, warehouse multi-select, supplier search-select), stat-card (×4), chart (×3: order volume, fulfillment rate, supplier performance), button-group (export PDF, export CSV)
**Realtime:** WebSocket — live order count, live alert count

---

## Infrastructure Pillar

### Services

**api-server** (type: server)
- `dev_command`: `npm run dev:api` (port 3001)
- `depends_on`: [postgres, redis]
- `startup_order`: 3
- `setup`: `npx prisma migrate dev && npx prisma db seed`

**frontend** (type: server)
- `dev_command`: `npm run dev:frontend` (port 3000)
- `depends_on`: [api-server]
- `startup_order`: 4
- `setup`: —

**queue-worker** (type: worker)
- `dev_command`: `npm run dev:worker`
- `depends_on`: [postgres, redis]
- `startup_order`: 3
- `setup`: —
- Processes jobs: forecast-generation, alert-dispatch, edi-processing, report-generation

**postgres** (type: datastore)
- `dev_command`: `docker-compose up postgres`
- `startup_order`: 1
- `setup`: `createdb vantage_dev`

**redis** (type: datastore)
- `dev_command`: `docker-compose up redis`
- `startup_order`: 2
- `setup`: —

**nginx** (type: proxy)
- `dev_command`: — (not used in dev; only in staging/prod docker-compose)
- `startup_order`: 5
- Routes: `/api/*` → api-server:3001, `/*` → frontend:3000

---

## Cross-Cutting Patterns

All 6 patterns defined in `architecture.yaml`:

1. **stealth_http** — All outbound calls to carrier APIs and EDI providers use a shared HTTP client that adds X-Request-ID, sets Accept-Encoding, and strips identifying headers
2. **api_key_resolution** — External service credentials (carrier API keys, EDI provider tokens) resolved from SystemConfig at request time, never hardcoded
3. **encryption** — PO attachment S3 URLs and supplier contract terms encrypted at rest using AES-256 via crypto node
4. **soft_delete** — Supplier, PurchaseOrder, StockItem, User — deleted_at timestamp set; all queries include WHERE deleted_at IS NULL
5. **content_hashing** — EDI documents and carrier webhook payloads hashed (SHA-256) on receipt; duplicates rejected before processing
6. **error_handling** — All flows terminate with standardized error shape: { code, message, context{} } defined in shared/errors.yaml

---

## Domain Events with Payload Schemas

Cross-domain events that flow through the event bus. Every event includes a typed payload schema.

| Event | Payload Fields |
|---|---|
| `stock.depleted` | sku: string, warehouse_id: uuid, quantity: number, threshold: number, auto_reorder: boolean |
| `order.created` | order_id: uuid, supplier_id: uuid, total_amount: number, priority: string, lines: OrderLine[] |
| `order.approved` | order_id: uuid, supplier_id: uuid, approved_by: uuid, lines: OrderLine[], expected_date: date |
| `order.cancelled` | order_id: uuid, reason: string, cancelled_by: uuid, refund_required: boolean |
| `shipment.in_transit` | shipment_id: uuid, carrier_id: uuid, tracking_number: string, eta: datetime, order_id: uuid |
| `shipment.delivered` | shipment_id: uuid, order_id: uuid, received_at: datetime, lines: OrderLine[], carrier_id: uuid |
| `supplier.risk_flagged` | supplier_id: uuid, risk_score: number, risk_factors: string[], flagged_by: string, severity: string |
| `forecast.generated` | sku: string, warehouse_id: uuid, forecast_horizon_days: number, predicted_demand: number, confidence: number, model_version: string |
| `alert.triggered` | alert_id: uuid, rule_id: uuid, type: string, severity: string, context: object, triggered_at: datetime |
| `config.updated` | key: string, updated_by: uuid, old_value_hash: string |
| `supply_chain.event_group` | composite: [stock.depleted, order.cancelled, shipment.delivered] — correlates by sku or order_id |

---

## Checklist — Required DDD Feature Coverage

Use this checklist after running `/ddd-create` and `npm run test:specs` to verify all features were generated.

### Trigger Types (10/10)
- [ ] HTTP — GET and POST flows in ≥3 domains
- [ ] cron — forecasting/run-daily-forecast, alerts/send-daily-digest, analytics/generate-weekly-report
- [ ] event — inventory/handle-stock-depleted, procurement/handle-order-approved, logistics/handle-shipment-delivered
- [ ] webhook — logistics/receive-carrier-update, suppliers/receive-edi-feed
- [ ] ipc — inventory/watch-warehouse-sensors, logistics/stream-shipment-position
- [ ] ws — analytics/stream-live-metrics, logistics/track-shipment-live
- [ ] pattern — alerts/detect-inventory-anomaly
- [ ] manual — procurement/emergency-reorder, suppliers/trigger-risk-review
- [ ] interval — inventory/refresh-stock-cache, logistics/poll-carrier-status, alerts/escalate-unresolved
- [ ] event_group — alerts/handle-supply-chain-event

### Node Types (28/28)
- [ ] trigger, input, process, decision, terminal, data_store, service_call, ipc_call, event
- [ ] loop (with break_condition)
- [ ] parallel (failure_policy: all_required) — logistics/notify-stakeholders
- [ ] parallel (failure_policy: best_effort) — alerts/send-multi-channel-alert
- [ ] sub_flow — logistics/process-fulfillment-order
- [ ] llm_call — suppliers/score-supplier-risk, forecasting/generate-ai-forecast
- [ ] delay — alerts/send-daily-digest
- [ ] cache — inventory/get-stock-levels, analytics/get-dashboard-data, settings/get-config
- [ ] transform (schema mode) — list and get flows
- [ ] transform (expression mode) — logistics/normalize-carrier-event
- [ ] collection (all 11 ops covered across flows)
- [ ] parse — logistics/parse-carrier-webhook-payload, suppliers/parse-edi-document
- [ ] crypto (hash, verify, encrypt, decrypt, generate) — users domain
- [ ] batch — forecasting/batch-generate-forecasts
- [ ] transaction — inventory/adjust-stock-quantity
- [ ] agent_loop (stop_conditions, is_terminal, requires_confirmation, all 3 memory types)
- [ ] guardrail (input position) — procurement/validate-order-safety
- [ ] guardrail (output position) — forecasting/validate-forecast-output
- [ ] human_gate — procurement/approve-high-value-order
- [ ] orchestrator (supervisor) — logistics/orchestrate-fulfillment
- [ ] orchestrator (round_robin, broadcast, consensus variants)
- [ ] smart_router (condition_field routing) — alerts/route-alert-to-channel
- [ ] smart_router (llm_routing: true) — logistics/route-to-carrier
- [ ] handoff (transfer mode)
- [ ] handoff (consult mode)
- [ ] handoff (collaborate mode)
- [ ] agent_group (broadcast) — forecasting/collaborative-demand-forecast
- [ ] agent_group (round_robin)
- [ ] agent_group (sequential)

### Schemas (17)
- [ ] Supplier, SupplierContact, PurchaseOrder, OrderLine, Warehouse, StockItem, Carrier, CarrierWarehouse, Shipment, ShipmentEvent, Forecast, ForecastDataPoint, AlertRule, AlertEvent, User, UserProfile, SystemConfig
- [ ] soft_delete on Supplier, PurchaseOrder, StockItem, User
- [ ] state machines on PurchaseOrder.status, Shipment.status, StockItem.status, Supplier.status
- [ ] seed data on Supplier, Warehouse, Carrier, AlertRule, User
- [ ] all relationship types: has_many, belongs_to, has_one, many_to_many

### UI Pages (12)
- [ ] All 9 component types used across page set
- [ ] All 14 form field types used (on Create PO page)
- [ ] All 4 interaction types: bulk-select, inline-edit, drag-drop, reorder
- [ ] WebSocket realtime on Dashboard and Live Shipment Tracking

### Infrastructure
- [ ] 4 service types: server (api-server, frontend), worker (queue-worker), datastore (postgres, redis), proxy (nginx)
- [ ] All services have depends_on, startup_order, dev_command, setup

### Patterns
- [ ] All 6 patterns in architecture.yaml: stealth_http, api_key_resolution, encryption, soft_delete, content_hashing, error_handling

### Domains
- [ ] All 3 domain roles: process (Procurement, Logistics, Forecasting, Analytics, Alerts), entity (Suppliers, Inventory, Users, Settings), gateway (Alerts)
- [ ] All 11 cross-domain events with payload schemas
- [ ] event_group trigger uses composite of multiple domain events
