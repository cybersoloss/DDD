# Design Driven Development (DDD)

**Design software visually. Let AI implement it.**

Design Driven Development is a methodology for building software through visual flow diagrams that output YAML specs. Specs and code are both sources of truth at different levels — specs define what/why, code accumulates how. The Reflect phase feeds implementation wisdom back into specs.

> **Note:** The abbreviation "DDD" is used throughout this project for brevity. This is not related to Eric Evans' Domain-Driven Design, which is an entirely separate methodology. We use the full name "Design Driven Development" to avoid confusion.

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

**Create** → describe your project, `/ddd-create` generates full YAML specs. **Design** → review in the [Design Driven Development Tool](https://github.com/cybersoloss/ddd-tool) desktop app. **Build** → `/ddd-scaffold` + `/ddd-implement` + `/ddd-test`. **Reflect** → `/ddd-sync` + `/ddd-reflect` + `/ddd-promote` to feed code wisdom back into specs.

The **specs directory** is the contract between phases. The human bridges them — reviewing design decisions that AI can't make alone.

> Legacy docs may reference "Session A" (= Phase 1+2) and "Session B" (= Phase 3+4).

## Commands

Eleven Claude Code slash commands power the Design Driven Development workflow:

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

### Ongoing Development

```bash
# Change a flow, then re-implement
/ddd-update users/user-login "Add rate limiting"
/ddd-implement users/user-login

# Scope: --all (project), domain, domain/flow
/ddd-implement --all
```

### Evolving the Framework

```bash
# Track framework limitations while designing
/ddd-create My SaaS app with real-time collab. TypeScript, Hono. --shortfalls
# → generates specs + specs/shortfalls.yaml with gap analysis

# Accumulate shortfalls across projects, then analyze
/ddd-evolve --dir ~/code/projA --dir ~/code/projB

# Review each finding interactively (approve/defer/reject)
/ddd-evolve --review ddd-evolution-plan.yaml

# Apply approved changes to the DDD framework
/ddd-evolve --apply ddd-evolution-plan.yaml
```

See the [Usage Guide — Shortfalls & Evolve](DDD-USAGE-GUIDE.md#shortfalls--evolve-end-to-end-example) for a full walkthrough with example output.

### Install Commands

```bash
# Clone the commands repo and install
git clone https://github.com/cybersoloss/claude-commands.git
cd claude-commands && ./install.sh

# Restart Claude Code to load the commands
```

## Repository Structure

```
DDD/
├── DDD-USAGE-GUIDE.md              # How to write DDD specs (fetched by commands at runtime)
└── templates/
    ├── architecture-template.yaml   # Reusable: project structure & conventions
    ├── config-template.yaml         # Reusable: environment variables schema
    └── errors-template.yaml         # Reusable: standardized error codes
```

> **Commands** live in the [claude-commands](https://github.com/cybersoloss/claude-commands) repo.

## What's Here vs What's Not

| This repo | The [Design Driven Development Tool](https://github.com/cybersoloss/ddd-tool) repo |
|-----------|------------|
| What Design Driven Development is | The actual desktop app |
| Usage Guide (YAML format reference) | Specification + Implementation Guide |
| Reusable templates | TypeScript, Rust, React code |
| Design decisions | Build system, dependencies, CI/CD |

## Key Documents

### [DDD Usage Guide](DDD-USAGE-GUIDE.md)
The definitive reference for writing Design Driven Development specs. Covers all 28 node types, YAML formats, connection patterns, supplementary spec files, validation rules, design patterns, and complete examples. This is what `/ddd-create` fetches at runtime to generate correct specs.

> **Specification + Implementation Guide** have moved to the [Design Driven Development Tool](https://github.com/cybersoloss/ddd-tool) repo — they are the source of truth for building the DDD Tool app.

### Templates
Copy these into any project's `specs/` folder to bootstrap a Design Driven Development project manually:

```bash
mkdir -p specs/shared
cp templates/architecture-template.yaml specs/architecture.yaml
cp templates/config-template.yaml specs/config.yaml
cp templates/errors-template.yaml specs/shared/errors.yaml
```

Or skip manual setup entirely — `/ddd-create` generates everything including these files.
