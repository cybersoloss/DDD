# Nexus — DDD Framework Benchmark Project

Nexus is a fictional **AI-powered content intelligence platform** used as the benchmark example for the DDD framework. It exists to validate that the spec format, node catalog, and DDD commands work correctly at real-world complexity.

> **Not a tutorial project.** For a simpler getting-started example, see [`examples/todo-app`](../todo-app/README.md).

---

## What Nexus Tests

| Coverage | Count |
|---|---|
| Node types used | **28/28** (100% of catalog) |
| Flows | **53** across 9 domains |
| Schemas | **17** |
| UI page specs | **13** |
| Spec files total | **99** (100% parse + normalize success) |

---

## Two Test Scenarios

### Scenario 1 — DDD Commands Effectiveness Test
*Validates that `/ddd-create` and the Usage Guide produce correct specs from a product brief.*

**Purpose:** Recreate Nexus from scratch using only the product definition and DDD commands. If the resulting specs score 98+/100 and cover 28/28 node types, the commands and Usage Guide are working correctly.

**Steps:**

```bash
# 1. Create a fresh project folder
mkdir ~/dev/nexus-test && cd ~/dev/nexus-test

# 2. Run ddd-create with the product definition
#    Open a Claude Code session here, then:
/ddd-create   # paste the contents of product-definition.md when prompted

# 3. Implement all flows
/ddd-implement

# 4. Run tests
/ddd-test

# 5. Run the automated benchmark (from ddd-tool repo)
cd ~/dev/ddd-tool
npm run test:specs -- ~/dev/nexus-test

# 6. Compare against Nexus benchmarks
#    Target: score ≥ 98/100, node type coverage = 28/28, 0 errors
```

**Pass criteria:**
- `spec-quality-report.yaml` score ≥ 98/100
- `tool-compatibility-report.yaml` success_rate_pct = 100
- Node type coverage = 28/28
- 0 parse or normalize failures

The product brief is in [`product-definition.md`](product-definition.md). It includes the full domain structure, flow descriptions, tech stack, and a node type coverage checklist to guide spec generation.

---

### Scenario 2 — DDD Tool Compatibility Test
*Validates that the ddd-tool app correctly handles all 28 node types and the full spec format.*

**Purpose:** Run the automated test runner against the committed Nexus specs. If all 99 files parse and normalize correctly, the tool is compatible with the current spec format.

**Steps:**

```bash
cd ~/dev/ddd-tool
npm run test:specs -- ~/dev/DDD/examples/nexus
```

**Expected output:**
```
  [1/4] Discovering YAML files...   Found 99 YAML files
  [2/4] Parsing YAML...             Parsed: 99 OK, 0 failed
  [3/4] Normalizing flow documents  Normalized: 53 OK, 0 failed
  [4/4] Running validation...       Issues: 0 errors, 12 warnings
                                    Node type coverage: 28/28

  Compatibility: FULLY_COMPATIBLE
  Quality:       EXCELLENT (score: 98/100)
  Completed in 0.17s
```

**When to run:** After any change to ddd-tool's parser, normalizer, or validator. If score drops or coverage falls below 28/28, something broke.

---

## App Description

**Nexus** — Content intelligence platform. Ingests content from RSS feeds, web scraping, CSV import, webhooks, and social feeds. Runs AI classification, summarization, fact-checking, and quality scoring. Routes content through editorial review, then publishes across channels.

**Tech stack:** TypeScript · Node.js 20 · Express 4 · Next.js 14 · PostgreSQL 16 · Prisma · Redis · BullMQ · shadcn/ui

---

## Domains

| Domain | Role | Flows |
|---|---|---|
| **Ingestion** | Content acquisition from external sources | 8 |
| **Processing** | AI-powered content analysis pipeline | 11 |
| **Editorial** | Human review and approval workflows | 7 |
| **Publishing** | Multi-channel content distribution | 6 |
| **Analytics** | Real-time and historical analytics | 5 |
| **Users** | User management and authentication | 6 |
| **Content** | Central content entity and event wiring hub | 4 |
| **Notifications** | Alert and notification delivery | 3 |
| **Settings** | System configuration and preferences | 3 |

---

## Node Type Coverage Map

Every node type in the 28-type catalog is exercised. Rare types and the flows that use them:

| Node Type | Flow Example |
|---|---|
| `agent_loop` | `processing/fact-check-agent`, `processing/bias-check-agent` |
| `agent_group` | `processing/collaborative-review`, `processing/analyze-with-agents` |
| `orchestrator` | `processing/orchestrate-analysis` |
| `smart_router` | `processing/route-to-specialist`, `publishing/publish-content` |
| `handoff` | `processing/route-to-specialist` |
| `guardrail` | `processing/classify-content`, `processing/score-quality` |
| `human_gate` | `editorial/review-content` |
| `llm_call` | `processing/classify-content`, `processing/summarize-batch` |
| `transaction` | `editorial/bulk-approve` |
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

Common types (`trigger`, `data_store`, `process`, `decision`, `input`, `event`, `service_call`, `terminal`) appear throughout all domains.

---

## Reports

Two auto-generated reports are committed alongside the specs:

| File | Purpose |
|---|---|
| `spec-quality-report.yaml` | Validates spec quality — event wiring, reference integrity, node coverage. Score: 98/100, 0 errors. |
| `tool-compatibility-report.yaml` | Validates DDD Tool compatibility — all 99 files parse and normalize successfully. |

Regenerate reports after any spec change:
```bash
cd ~/dev/ddd-tool && npm run test:specs -- ~/dev/DDD/examples/nexus
```

---

## Adding Coverage for New Node Types

When a new node type is added to the DDD catalog:

1. Identify which Nexus domain it fits naturally (e.g., new AI node type → `processing/`)
2. Add or update a flow spec to use the new node type
3. Update `product-definition.md` — add to the node type coverage checklist
4. Regenerate reports: `npm run test:specs -- ~/dev/DDD/examples/nexus`
5. Verify `coverage_pct` = 100 and commit updated flow + reports
