# Design Driven Development (DDD)

**Design software visually. Let AI implement it.**

DDD is a methodology for building software through visual flow diagrams that output YAML specs. The specs are the single source of truth — they live in Git alongside code, and LLMs (like Claude Code) implement directly from them.

## How It Works

```
 Session A (Architect)          Human Review           Session B (Developer)

 Describe your software    ──→  Open DDD Tool     ──→  /ddd-implement --all
 /ddd-create generates          validate on canvas      generates code + tests
 full YAML spec structure       refine, adjust          from specs
                                                            │
                                                            ▼
                                                        /ddd-update
                                                        modify specs from
                                                        natural language
                                                            │
                                                            ▼
                                                        /ddd-sync
                                                        keep specs & code
                                                        aligned
```

### The Workflow

**Step 1 — Design (Session A):** Describe your software project to Claude. The `/ddd-create` command fetches the [DDD Usage Guide](DDD-USAGE-GUIDE.md) from this repo, then generates the complete spec structure: project config, tech stack, data models, error codes, domains, and flow graphs with all nodes wired.

**Step 2 — Review (Human):** Open the generated specs in the [DDD Tool](https://github.com/mhcandan/ddd-tool) desktop app. Visualize the flow graphs on a 3-level canvas (System Map → Domain Map → Flow Canvas). Validate with 20+ built-in rules. Adjust nodes, connections, and specs directly on the canvas.

**Step 3 — Implement (Session B):** In a fresh Claude session, run `/ddd-implement` to generate working code from the specs. Claude reads all YAML files (flows, schemas, architecture, error codes), follows the node graph from trigger to terminal, generates code for each node, writes tests, runs them, and tracks everything in `.ddd/mapping.yaml`.

**Step 4 — Iterate (Session B):** As requirements evolve, use `/ddd-update` to modify specs from natural language ("add rate limiting before login"), then `/ddd-implement` to update the code. Use `/ddd-sync` to detect drift between specs and implementation.

### Why Two Sessions?

| Session A (Architect) | Session B (Developer) |
|---|---|
| Design context only | Implementation context only |
| Thinks about domains, flows, events | Thinks about code, tests, infrastructure |
| Outputs YAML specs | Outputs working code |
| No code noise | No design noise |

The **specs directory** is the contract between sessions. Both Claude instances read and write the same YAML files, but neither needs the other's context. The human bridges them — reviewing design decisions that AI can't make alone.

## Commands

Four Claude Code slash commands power the workflow:

| Command | What it does |
|---------|-------------|
| `/ddd-create` | Describe a project in natural language → full DDD spec structure (domains, flows, schemas, config) |
| `/ddd-implement` | Read specs → generate code + tests, run tests, update mapping |
| `/ddd-update` | Natural language change request → updated YAML specs |
| `/ddd-sync` | Sync mapping, discover untracked code, fix drifted implementations |

Commands are in the [claude-commands](https://github.com/mhcandan/claude-commands) repo.

## Repository Structure

```
DDD/
├── DDD-USAGE-GUIDE.md              # How to write DDD specs (fetched by commands at runtime)
├── ddd-specification-complete.md    # Full spec: concepts, features, data model
├── ddd-implementation-guide.md      # Build instructions: types, stores, components
├── ddd-quickstart.md                # Quick start guide for using DDD
└── templates/
    ├── architecture-template.yaml   # Reusable: project structure & conventions
    ├── config-template.yaml         # Reusable: environment variables schema
    └── errors-template.yaml         # Reusable: standardized error codes
```

## What's Here vs What's Not

| This repo | The [ddd-tool](https://github.com/mhcandan/ddd-tool) repo |
|-----------|------------|
| What DDD is | The actual desktop app |
| How to build the tool | TypeScript, Rust, React code |
| Reusable templates | Build system, dependencies |
| Design decisions | CI/CD, releases |

## Key Documents

### [DDD Usage Guide](DDD-USAGE-GUIDE.md)
The definitive reference for writing DDD specs. Covers all 19 node types, YAML formats, connection patterns, supplementary spec files, validation rules, design patterns, and complete examples. This is what `/ddd-create` fetches at runtime to generate correct specs.

### Specification (~280 KB)
The complete spec covering: multi-level canvas, 20+ node types, agent & orchestration flows, LLM design assistant, Claude Code integration, bidirectional drift detection, diagram-derived test generation, design validation, production infrastructure generation, and more.

### Implementation Guide (~420 KB)
Step-by-step build instructions with near-complete code for every component: types, Zustand stores, React components, Tauri commands, utilities. Organized into phases and sessions.

### Templates
Copy these into any project's `specs/` folder to bootstrap a DDD project manually:

```bash
mkdir -p specs/shared
cp templates/architecture-template.yaml specs/architecture.yaml
cp templates/config-template.yaml specs/config.yaml
cp templates/errors-template.yaml specs/shared/errors.yaml
```

Or skip manual setup entirely — `/ddd-create` generates everything including these files.

## The DDD Tool

The desktop app (Tauri + React) is built in a separate repo: **[mhcandan/ddd-tool](https://github.com/mhcandan/ddd-tool)**

19 node types, 3-level canvas navigation, LLM design assistant, Git integration, validation engine, production generators (OpenAPI, Docker, K8s, CI/CD), project memory, drift detection, and reconciliation.
