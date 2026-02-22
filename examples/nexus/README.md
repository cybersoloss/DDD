# Nexus — DDD Framework Benchmark Project

Nexus is a fictional **AI-powered content intelligence platform** used as the benchmark example for the DDD framework. It exists to validate that the spec format, node catalog, and DDD commands work correctly at real-world complexity.

> **Not a tutorial project.** For a simpler getting-started example, see [`examples/todo-app`](../todo-app/README.md).

## What Nexus Tests

| Coverage | Count |
|---|---|
| Node types used | **28/28** (100% of catalog) |
| Flows | **53** across 9 domains |
| Schemas | **17** |
| UI page specs | **13** |
| Spec files total | **99** (100% parse + normalize success) |

Running the DDD tool compatibility check and spec quality report against Nexus confirms that every node type in the catalog is exercised and every file parses correctly.

## App Description

**Nexus** — Content intelligence platform. Ingests content from RSS feeds, web scraping, CSV import, webhooks, and social feeds. Runs AI classification, summarization, fact-checking, and quality scoring. Routes content through editorial review, then publishes across channels.

**Tech stack:** TypeScript · Node.js 20 · Express 4 · Next.js 14 · PostgreSQL 16 · Prisma · Redis · BullMQ · shadcn/ui

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

## Node Type Coverage Map

Every node type in the 28-type catalog is exercised. Rare types and the flows that use them:

| Node Type | Flow Example |
|---|---|
| `agent_loop` | `processing/fact-check-agent`, `processing/bias-check-agent` |
| `agent_group` | `processing/collaborative-review` |
| `orchestrator` | `processing/orchestrate-analysis` |
| `smart_router` | `processing/route-to-specialist` |
| `handoff` | `processing/route-to-specialist` |
| `guardrail` | `processing/classify-content`, `processing/score-quality` |
| `human_gate` | `editorial/review-content` |
| `llm_call` | `processing/classify-content`, `processing/summarize-batch` |
| `transaction` | `editorial/bulk-approve` |
| `batch` | `processing/summarize-batch` |
| `sub_flow` | Multiple flows |
| `parallel` | Multiple flows |
| `ipc_call` | Multiple flows |
| `cache` | Multiple flows |
| `crypto` | `users/` flows (token encryption) |

Common types (`trigger`, `data_store`, `process`, `decision`, `event`, `terminal`, `input`, `transform`, `collection`, `parse`, `loop`, `delay`, `service_call`) appear throughout all domains.

## Reports

Two auto-generated reports are committed alongside the specs:

| File | Purpose |
|---|---|
| `spec-quality-report.yaml` | Validates spec quality — event wiring, reference integrity, node coverage. Score: 98/100, 0 errors. |
| `tool-compatibility-report.yaml` | Validates DDD Tool compatibility — all 99 files parse and normalize successfully. |

## Regenerating Reports

When the DDD framework changes (new node types, new spec conventions), regenerate reports by running the DDD Tool's validation against this project:

1. Open Nexus in DDD Tool: point it at this directory
2. Run validation — reports are written to `spec-quality-report.yaml` and `tool-compatibility-report.yaml`
3. Commit updated reports to track framework changes over time

## Adding Coverage for New Node Types

When a new node type is added to the DDD catalog:

1. Identify which Nexus domain it fits naturally (e.g., a new AI node type → `processing/`)
2. Add or update a flow spec to use the new node type
3. Regenerate reports and verify `coverage_pct` remains 100%
4. Commit the updated flow + reports
