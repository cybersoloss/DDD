# DDD Quick Start Guide

## What You Need

| Component | Purpose |
|-----------|---------|
| [DDD repo](https://github.com/mhcandan/DDD) | Spec, guide, templates |
| [DDD Tool](https://github.com/mhcandan/ddd-tool) | Desktop app for visual design |
| [Claude Commands](https://github.com/mhcandan/claude-commands) | `/ddd-create`, `/ddd-implement`, `/ddd-update`, `/ddd-sync` |
| Claude Code | AI that reads specs and generates code |

---

## Quick Start: Create a Project in 3 Steps

### Step 1 — Generate specs (Session A)

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

### Step 2 — Review in DDD Tool

Open the project in the DDD Tool desktop app:

- **L1 System Map** — see all domains and event wiring
- **L2 Domain Map** — see flows within each domain
- **L3 Flow Canvas** — see the full node graph for each flow

Validate with 20+ built-in rules. Adjust nodes, connections, and specs on the canvas. Save (Cmd+S) writes changes back to the YAML files.

### Step 3 — Implement (Session B)

Open a fresh Claude Code session in the project directory:

```
/ddd-implement --all
```

Claude reads all specs, generates code for each flow, writes tests, runs them, and tracks everything in `.ddd/mapping.yaml`.

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

| Command | What it does |
|---------|-------------|
| `/ddd-create` | Describe project → full spec structure |
| `/ddd-implement` | Specs → code + tests |
| `/ddd-update` | Natural language → updated specs |
| `/ddd-sync` | Keep specs and code aligned |

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

---

## Key Concepts

- **Specs are the source of truth** — code is derived from specs, not the other way around
- **19 node types** — trigger, input, process, decision, terminal, data_store, service_call, event, loop, parallel, sub_flow, llm_call, agent_loop, guardrail, human_gate, orchestrator, smart_router, handoff, agent_group
- **Branching nodes use `sourceHandle`** — input (valid/invalid), decision (true/false), data_store (success/error), service_call (success/error), loop (body/done), parallel (branch-N/done)
- **Session separation** — Session A designs (no code noise), Session B implements (no design noise)
- **Human bridges sessions** — reviews specs in DDD Tool before implementation

See the [DDD Usage Guide](DDD-USAGE-GUIDE.md) for complete YAML formats, all node specs, and examples.
