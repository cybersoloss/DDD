# Design Driven Development (DDD)

**Design software visually. Let AI implement it.**

DDD is a methodology for building software through visual flow diagrams that output YAML specs. Specs and code are both sources of truth at different levels — specs define what/why, code accumulates how. The Reflect phase feeds implementation wisdom back into specs.

## How It Works

```
 Phase 1: CREATE          Phase 2: DESIGN         Phase 3: BUILD          Phase 4: REFLECT

 Describe your software   Open DDD Tool           /ddd-scaffold           /ddd-sync
 /ddd-create generates    validate on canvas      set up skeleton         check alignment
 full YAML spec structure refine, adjust              │                       │
                                                      ▼                       ▼
                                                  /ddd-implement          /ddd-reflect
                                                  generate code + tests   capture wisdom
                                                      │                       │
                                                      ▼                       ▼
                                                  /ddd-test               /ddd-promote
                                                  verify tests pass       enrich specs
```

### The Workflow

**Phase 1 — Create:** Describe your software project to Claude. The `/ddd-create` command fetches the [DDD Usage Guide](DDD-USAGE-GUIDE.md) from this repo, then generates the complete spec structure: project config, tech stack, data models, error codes, domains, and flow graphs with all nodes wired. For existing codebases, use `/ddd-reverse` instead.

**Phase 2 — Design:** Open the generated specs in the [DDD Tool](https://github.com/mhcandan/ddd-tool) desktop app. Visualize the flow graphs on a 3-level canvas (System Map → Domain Map → Flow Canvas). Validate with 20+ built-in rules. Adjust nodes, connections, and specs directly on the canvas.

**Phase 3 — Build:** Run `/ddd-scaffold` to set up the project skeleton — package config, database schema, shared infrastructure, and test setup. Then `/ddd-implement` to generate working code from specs. Claude reads all YAML files, follows the node graph from trigger to terminal, generates code for each node, writes tests, runs them, and tracks everything in `.ddd/mapping.yaml`. Use `/ddd-test` to verify, `/ddd-update` to modify specs, and iterate.

**Phase 4 — Reflect:** After implementation stabilizes, run `/ddd-sync` to check alignment, `/ddd-reflect` to capture implementation wisdom (patterns the code uses that specs don't describe), and `/ddd-promote` to move approved patterns into permanent specs. This closes the loop — code wisdom feeds back into the design.

### Evolving DDD Itself

As you use DDD across projects, you'll encounter framework gaps — missing node types, inadequate fields, layer visibility issues. DDD has a built-in feedback loop:

```
/ddd-create projA --shortfalls  →  specs/shortfalls.yaml   ┐
/ddd-create projB --shortfalls  →  specs/shortfalls.yaml   ├→ /ddd-evolve → /ddd-evolve --review → /ddd-evolve --apply
/ddd-create projC --shortfalls  →  specs/shortfalls.yaml   ┘    (analyze)     (interactive decisions)   (execute approved)
```

The `--shortfalls` flag on `/ddd-create` generates a structured report of 7 gap categories (missing node types, inadequate nodes, missing fields, connection limitations, layer gaps, workarounds, cross-cutting gaps). `/ddd-evolve` then reads these reports, critically evaluates each gap through 6 filters (already possible? recurring? specific? breaking? adequate workaround? intentional?), and produces a tiered recommendation plan. Nothing changes until a human approves.

### Why Four Phases?

| Create + Design | Build + Reflect |
|---|---|
| Design context only | Implementation context only |
| Thinks about domains, flows, events | Thinks about code, tests, infrastructure |
| Outputs YAML specs | Outputs working code + annotations |
| No code noise | No design noise |

The **specs directory** is the contract between phases. The human bridges them — reviewing design decisions that AI can't make alone. The Reflect phase ensures implementation wisdom (cross-cutting patterns, conventions, edge cases) flows back into specs rather than living only in code.

> Legacy docs may reference "Session A" (= Phase 1+2) and "Session B" (= Phase 3+4).

## Commands

Eleven Claude Code slash commands power the workflow:

| Phase | Command | What it does |
|-------|---------|-------------|
| Create | `/ddd-create` | Describe a project in natural language → full DDD spec structure. Use `--shortfalls` for gap analysis. |
| Create | `/ddd-reverse` | Reverse-engineer existing code → DDD specs (6 strategies by codebase size) |
| Any | `/ddd-update` | Natural language change request → updated YAML specs |
| Build | `/ddd-scaffold` | Set up project skeleton from specs (Phase 3 first step) |
| Build | `/ddd-implement` | Read specs → generate flow code + tests, update mapping |
| Build | `/ddd-test` | Run tests for implemented flows without re-generating code |
| Reflect | `/ddd-sync` | Sync mapping, discover untracked code, fix drifted implementations |
| Reflect | `/ddd-reflect` | Capture implementation wisdom as annotations in `.ddd/annotations/` |
| Reflect | `/ddd-promote` | Move approved annotations into permanent specs |
| Any | `/ddd-status` | Quick read-only overview of project implementation state |
| Meta | `/ddd-evolve` | Analyze shortfall reports → `--review` for decisions → `--apply` to execute |

### Install Commands

```bash
# Clone this repo
git clone https://github.com/mhcandan/DDD.git

# Copy DDD commands into Claude Code
mkdir -p ~/.claude/commands
cp DDD/commands/*.md ~/.claude/commands/

# Restart Claude Code to load the commands
```

## Repository Structure

```
DDD/
├── DDD-USAGE-GUIDE.md              # How to write DDD specs (fetched by commands at runtime)
├── ddd-specification-complete.md    # Full spec: concepts, features, data model
├── ddd-implementation-guide.md      # Build instructions: types, stores, components
├── ddd-quickstart.md                # Quick start guide for using DDD
├── commands/                        # Claude Code slash commands
│   ├── DDD-commands.md              # Command reference
│   ├── ddd-create.md                # /ddd-create
│   ├── ddd-reverse.md               # /ddd-reverse
│   ├── ddd-update.md                # /ddd-update
│   ├── ddd-scaffold.md              # /ddd-scaffold
│   ├── ddd-implement.md             # /ddd-implement
│   ├── ddd-test.md                  # /ddd-test
│   ├── ddd-sync.md                  # /ddd-sync
│   ├── ddd-reflect.md               # /ddd-reflect
│   ├── ddd-promote.md               # /ddd-promote
│   ├── ddd-status.md                # /ddd-status
│   └── ddd-evolve.md                # /ddd-evolve
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
The definitive reference for writing DDD specs. Covers all 28 node types, YAML formats, connection patterns, supplementary spec files, validation rules, design patterns, and complete examples. This is what `/ddd-create` fetches at runtime to generate correct specs.

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

28 node types, 3-level canvas navigation, LLM design assistant, Git integration, validation engine, production generators (OpenAPI, Docker, K8s, CI/CD), project memory, drift detection, and reconciliation.
