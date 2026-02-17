# DDD Quick Start Guide

## What You Need

| Component | Purpose |
|-----------|---------|
| [DDD repo](https://github.com/cybersoloss/DDD) | Spec, guide, templates |
| [DDD Tool](https://github.com/cybersoloss/ddd-tool) | Desktop app for visual design |
| [Claude Commands](https://github.com/cybersoloss/claude-commands) | 12 slash commands for the full lifecycle |
| Claude Code | AI that reads specs and generates code |

---

## Quick Start: Create a Project in 4 Phases

### Phase 1 — Create specs

Open Claude Code and describe your project:

```
/ddd-create An e-commerce platform with users, products, orders, and notifications.
TypeScript + Express + PostgreSQL + Redis. JWT auth. REST API.
```

Claude fetches the [DDD Usage Guide](DDD-USAGE-GUIDE.md), asks any clarifying questions, then generates:

```
my-project/
  ddd-project.json                    # Project config
  specs/
    system.yaml                       # Tech stack
    architecture.yaml                 # Conventions, project structure
    config.yaml                       # Environment variables
    shared/errors.yaml                # Error codes
    schemas/_base.yaml                # Base model
    schemas/user.yaml                 # Data models
    schemas/product.yaml
    schemas/order.yaml
    domains/users/domain.yaml         # Domain config + events
    domains/users/flows/
      user-register.yaml              # Full flow graphs
      user-login.yaml
    domains/products/domain.yaml
    domains/products/flows/
      list-products.yaml
    domains/orders/domain.yaml
    domains/orders/flows/
      create-order.yaml
      process-payment.yaml
    domains/notifications/domain.yaml
    domains/notifications/flows/
      send-email.yaml
```

### Phase 2 — Design in DDD Tool

Open the project in the DDD Tool desktop app:

- **L1 System Map** — see all domains and event wiring
- **L2 Domain Map** — see flows within each domain
- **L3 Flow Canvas** — see the full node graph for each flow

Validate with 20+ built-in rules. Adjust nodes, connections, and specs on the canvas. Save (Cmd+S) writes changes back to the YAML files.

### Phase 3 — Build

Open Claude Code in the project directory:

```
/ddd-implement --all
```

Claude reads all specs, generates code for each flow, writes tests, runs them, and tracks everything in `.ddd/mapping.yaml`.

### Phase 4 — Reflect (after implementation stabilizes)

Capture implementation wisdom and feed it back into specs:

```
/ddd-sync              # Check alignment between specs and code
/ddd-reflect           # Capture patterns code has that specs don't describe
/ddd-promote           # Move approved patterns into permanent specs
```

This closes the loop — code wisdom feeds back into specs, making future implementations smarter.

---

## Ongoing Development

### Change a flow

```
/ddd-update users/user-login
> "Add rate limiting before the login process"
```

Claude updates the YAML spec. Reload DDD Tool (Cmd+R) to see the change. Then:

```
/ddd-implement users/user-login
```

### Add a new flow

```
/ddd-update --add-flow orders
> "Add an order cancellation flow with refund processing"
```

### Add a new domain

```
/ddd-update --add-domain
> "Add a notifications domain with email and push notification flows"
```

### Sync after manual code changes

```
/ddd-sync              # Update mapping.yaml
/ddd-sync --discover   # Find untracked code, suggest specs
/ddd-sync --fix-drift  # Re-implement drifted flows
/ddd-sync --full       # All of the above
```

---

## Commands Reference

| Phase | Command | What it does |
|-------|---------|-------------|
| 1 Create | `/ddd-create` | Describe project → full spec structure |
| 2 Design | *(DDD Tool)* | Visual review and refinement on canvas |
| 3 Build | `/ddd-scaffold` | Set up project skeleton from specs |
| 3 Build | `/ddd-implement` | Specs → code + tests |
| 3 Build | `/ddd-test` | Run tests for implemented flows |
| 4 Reflect | `/ddd-reverse` | Existing code → DDD specs |
| 4 Reflect | `/ddd-reflect` | Capture implementation wisdom as annotations |
| 4 Reflect | `/ddd-promote` | Move approved annotations into specs |
| Any | `/ddd-status` | Quick read-only project overview |
| Any | `/ddd-update` | Natural language → updated specs |
| Any | `/ddd-sync` | Keep specs and code aligned |
| Meta | `/ddd-evolve` | Analyze shortfalls → evolve DDD framework |

### Scope arguments

```
/ddd-implement --all                   # Whole project
/ddd-implement users                   # All flows in a domain
/ddd-implement users/user-register     # Single flow
/ddd-implement                         # Interactive — pick from list
```

---

## Files Reference

| File | What it controls |
|------|-----------------|
| `ddd-project.json` | Domain list |
| `specs/system.yaml` | Tech stack, environments |
| `specs/architecture.yaml` | Project structure, conventions, API design, testing |
| `specs/config.yaml` | Environment variables |
| `specs/shared/errors.yaml` | Error codes with HTTP status mappings |
| `specs/schemas/*.yaml` | Data models (fields, indexes, relationships) |
| `specs/domains/*/domain.yaml` | Domain config, flow list, event wiring |
| `specs/domains/*/flows/*.yaml` | Flow graphs (trigger, nodes, connections) |
| `.ddd/mapping.yaml` | Implementation tracking (spec hash, file list) |
| `.ddd/annotations/{domain}/{flow}.yaml` | Implementation wisdom captured by /ddd-reflect |

---

## Key Concepts

- **Specs and code are both sources of truth at different levels** — specs define what/why, code accumulates how. The Reflect phase feeds code wisdom back into specs.
- **Four-phase lifecycle** — Create → Design → Build → Reflect. Each phase has dedicated commands.
- **28 node types** — trigger, input, process, decision, terminal, data_store, service_call, ipc_call, event, loop, parallel, sub_flow, llm_call, delay, cache, transform, collection, parse, crypto, batch, transaction, agent_loop, guardrail, human_gate, orchestrator, smart_router, handoff, agent_group
- **Branching nodes use `sourceHandle`** — input (valid/invalid), decision (true/false), data_store (success/error), service_call (success/error), loop (body/done), parallel (branch-N/done)
- **Human bridges phases** — reviews specs in DDD Tool between Create and Build, reviews annotations between Build and Reflect

> **Note:** Legacy docs may reference "Session A" (= Phase 1+2) and "Session B" (= Phase 3+4).

See the [DDD Usage Guide](DDD-USAGE-GUIDE.md) for complete YAML formats, all node specs, and examples.
