# Vantage — DDD Framework Validation Project

Vantage is a second benchmark project for the DDD framework — a fictional **AI-powered supply chain intelligence platform**. It exists alongside [Nexus](../nexus/README.md) to validate `/ddd-create` across a different domain.

> **Not a tutorial project.** For a simpler getting-started example, see [`examples/todo-app`](../todo-app/README.md).

---

## What Vantage Tests

Vantage exercises every DDD feature via a completely different domain (supply chain) than Nexus (content intelligence). The same full coverage is required:

| Coverage | Target |
|---|---|
| Node types | **28/28** (100% of catalog) |
| Trigger types | **10/10** |
| Schemas | **17** with all relationship/index/seed/state machine features |
| UI pages | **12** with all 9 component types, all 14 form field types, all 4 interaction types |
| Infrastructure services | **6** (server ×2, worker ×1, datastore ×2, proxy ×1) |
| Architecture patterns | **6/6** |

---

## Scenario 1 — DDD Commands Effectiveness Test

*Validates that `/ddd-create` produces correct specs from a product brief in a new domain.*

**Purpose:** Run `/ddd-create` from scratch using `product-definition.md`. If the resulting specs score 98+/100, cover 28/28 node types, and the shortfalls report is clean, the command and Usage Guide are working correctly.

**Steps:**

```bash
# 1. Create a fresh project folder
mkdir ~/dev/vantage-test && cd ~/dev/vantage-test

# 2. Open a Claude Code session in ~/dev/vantage-test, then run:
/ddd-create --from ~/dev/DDD/examples/vantage/product-definition.md --shortfalls

# 3. Run the automated benchmark (from ddd-tool repo)
cd ~/dev/ddd-tool
npm run test:specs -- ~/dev/vantage-test
```

**Pass criteria:**
- `spec-quality-report.yaml` score ≥ 98/100
- `tool-compatibility-report.yaml` success_rate_pct = 100
- Node type coverage = 28/28
- 0 parse or normalize failures
- `specs/shortfalls.yaml` — review for any DDD framework limitations hit during generation

**When to run:** After any change to `/ddd-create` or `DDD-USAGE-GUIDE.md`. Use alongside Nexus — if both score 98+, confidence is high across domains. Shortfalls report feeds into `/ddd-evolve`.

The product brief is in [`product-definition.md`](product-definition.md). It contains a DDD Feature Coverage Matrix and a node type + trigger + schema + UI checklist at the end to verify all features were generated.

---

## App Description

**Vantage** — Supply chain intelligence platform. Manages suppliers, purchase orders, inventory, and logistics. AI agents score supplier risk, generate demand forecasts, detect anomalies, and automate procurement decisions. Operations teams approve high-value orders, monitor live shipment tracking, and review AI-generated forecasts.

**Tech stack:** TypeScript · Node.js 20 · Express 4 · Next.js 14 · PostgreSQL 16 · Prisma · Redis · BullMQ · shadcn/ui

---

## Domains

| Domain | Role | Key Flows |
|---|---|---|
| **Suppliers** | entity | list, get, onboard, risk scoring, EDI feed |
| **Procurement** | process | create order, approve (human_gate), escalate (handoff), bulk approve (transaction) |
| **Inventory** | entity | stock levels, adjust (transaction), sensor watch (ipc), auto-reorder |
| **Logistics** | process | carrier webhooks, fulfillment orchestration, live tracking (ws), batch polling |
| **Forecasting** | process | daily cron, AI agent loop, collaborative agent_group, batch generation |
| **Analytics** | process | dashboard, query metrics, weekly report, live stream (ws), anomaly detection |
| **Alerts** | gateway | event_group trigger, smart_router, multi-channel parallel, escalation loop |
| **Users** | entity | register, authenticate, API key rotation (crypto) |
| **Settings** | entity | config read/write, SLA breach handler |

---

## Why Two Benchmarks

| | Nexus | Vantage |
|---|---|---|
| Domain | Content intelligence | Supply chain |
| Primary AI pattern | Content classification + fact-checking | Demand forecasting + risk scoring |
| Key real-time feature | Live metrics + editorial queue | Live shipment tracking + sensor stream |
| Human review flow | Editorial approval | Purchase order approval |
| Key orchestration | Multi-agent content analysis | Fulfillment orchestration |

Running both catches domain-specific regressions that a single benchmark might miss.
