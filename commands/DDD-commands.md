# DDD Commands Reference

Eleven Claude Code slash commands for the [Design Driven Development](https://github.com/mhcandan/DDD) four-phase lifecycle.

```
Phase 1: CREATE        Phase 2: DESIGN         Phase 3: BUILD          Phase 4: REFLECT
Human intent → Specs   Human reviews in Tool   Specs → Code            Code wisdom → Specs

/ddd-create            (DDD Tool)              /ddd-scaffold           /ddd-reverse
/ddd-update                                    /ddd-implement          /ddd-sync --discover
                                               /ddd-test               /ddd-reflect
                                                                       /ddd-promote

Cross-cutting (any phase): /ddd-status
Meta-level: /ddd-evolve
```

## Overview

| Phase | Command | Options | Description |
|-------|---------|---------|-------------|
| 1 Create | `/ddd-create` | `--from`, `--shortfalls` | Design a new project from description or design file → YAML specs |
| 3 Build | `/ddd-scaffold` | — | Set up project skeleton from specs (first step of Phase 3) |
| 3 Build | `/ddd-implement` | `--all`, `domain`, `domain/flow` | Read specs → generate code + tests |
| 3 Build | `/ddd-test` | `--all`, `--coverage`, `domain`, `domain/flow` | Run tests for implemented flows |
| 4 Reflect | `/ddd-reverse` | `--output`, `--domains`, `--merge`, `--strategy` | Reverse-engineer existing code → YAML specs |
| 4 Reflect | `/ddd-reflect` | `--all`, `domain`, `domain/flow` | Capture implementation wisdom as annotations |
| 4 Reflect | `/ddd-promote` | `--all`, `--review`, `domain/flow` | Move approved annotations into permanent specs |
| Any | `/ddd-status` | `--json` | Quick read-only project overview |
| Any | `/ddd-update` | `--add-flow`, `--add-domain`, `domain/flow` | Natural language → updated specs |
| Any | `/ddd-sync` | `--discover`, `--fix-drift`, `--full` | Keep specs and code aligned |
| Meta | `/ddd-evolve` | `--dir`, `--review`, `--apply` | Analyze shortfall reports → review → apply approved changes |

### Workflow

```
New project:      /ddd-create → DDD Tool → /ddd-scaffold → /ddd-implement → /ddd-test → /ddd-reflect → /ddd-promote
Existing project: /ddd-reverse → DDD Tool → /ddd-scaffold → /ddd-implement → /ddd-test → /ddd-reflect → /ddd-promote
Iterate:          /ddd-status → /ddd-update → /ddd-implement → /ddd-test → /ddd-sync
Reflect:          /ddd-reflect → /ddd-promote --review → specs updated with implementation wisdom
Evolve DDD:       /ddd-create --shortfalls → /ddd-evolve → /ddd-evolve --review → /ddd-evolve --apply
```

---

## /ddd-create

Generate a complete DDD spec structure from a project description.

### Usage

```
/ddd-create <description> [--from <path-or-url>] [--shortfalls]
```

### Flags

| Flag | Purpose |
|------|---------|
| `--from <path-or-url>` | Use a design file as reference input. Supports images (PNG, JPG), PDFs, markdown, text, YAML, and URLs (Figma, Miro, web pages). Extracts domains, flows, data models, events, and architecture from the design. Combine with a text description for additional context. |
| `--shortfalls` | Generate `specs/shortfalls.yaml` — a structured report of DDD framework gaps encountered during design (7 categories). Feed into `/ddd-evolve` for analysis. |

### Examples

```
# From a text description
/ddd-create A SaaS platform for managing restaurant orders. Node.js, Express, PostgreSQL.

# From a design file (wireframes, mockups, architecture diagram)
/ddd-create --from ~/designs/app-wireframes.png E-commerce platform

# From a PDF requirements doc
/ddd-create --from ~/docs/requirements.pdf

# From a Figma URL
/ddd-create --from https://figma.com/file/abc123 Social media dashboard

# Combine design file with description and shortfall analysis
/ddd-create --from ~/designs/system-arch.pdf AI content moderation service. TypeScript, Hono. --shortfalls
```

### What it does

1. Fetches the [DDD Usage Guide](https://github.com/mhcandan/DDD/blob/main/DDD-USAGE-GUIDE.md) from GitHub
2. Parses the description (asks clarifying questions if brief)
3. Creates full spec structure:
   - `ddd-project.json` — project config, domain list
   - `specs/system.yaml` — tech stack, environments
   - `specs/architecture.yaml` — conventions, infrastructure
   - `specs/config.yaml` — environment variables
   - `specs/shared/errors.yaml` — error codes
   - `specs/shared/types.yaml` — shared enums (if needed)
   - `specs/schemas/_base.yaml` — base model fields
   - `specs/schemas/{model}.yaml` — one per data model
   - `specs/domains/{domain}/domain.yaml` — domain config, event wiring
   - `specs/domains/{domain}/flows/{flow}.yaml` — flow graphs with nodes and connections

### Output

Shows a summary with domains, flow counts, files created, event wiring, and next steps.

---

## /ddd-reverse

Reverse-engineer an existing codebase into DDD specs.

### Usage

```
/ddd-reverse <project-path> [flags]
```

### Flags

| Flag | Purpose | Default |
|---|---|---|
| `--output <path>` | Where to write specs | Same as project path |
| `--domains <d1,d2>` | Only reverse specific domains | All domains |
| `--merge` | Merge with existing specs (don't overwrite) | Overwrite |
| `--strategy <name>` | Override auto-selected strategy | Auto by file count |

### Strategies

Auto-selected based on source file count (override with `--strategy`):

| Strategy | Files | Approach |
|---|---|---|
| `baseline` | < 30 | Read code directly in context |
| `index` | 30–80 | Build in-context index, process per-domain |
| `swap` | 80–150 | Write index to `.ddd/reverse/` on disk, read selectively |
| `bottom-up` | 150–300 | Grep entry points (L3), extract each independently, group into domains (L2→L1) |
| `compiler` | 300–500 | 6-pass pipeline: scan → extract → resolve → IR → link → emit |
| `codex` | 500+ | Compress codebase to ref code vocabulary + one-line call chains |

### Examples

```
# Small project (auto-selects baseline)
/ddd-reverse ~/code/my-api

# Output specs to separate folder
/ddd-reverse ~/code/my-app --output ~/code/my-app-specs

# Only reverse users and orders domains
/ddd-reverse ~/code/my-app --domains users,orders

# Add to existing specs without overwriting
/ddd-reverse ~/code/my-app --merge

# Force compiler strategy
/ddd-reverse ~/code/my-app --strategy compiler
```

### What it does

1. Detects tech stack from config files (package.json, Cargo.toml, go.mod, etc.)
2. Scans project structure, infers domain boundaries
3. Extracts data models from ORM/schema definitions
4. Extracts flows from routes, handlers, event listeners, cron jobs
5. Extracts cross-cutting concerns (errors, shared types, config, architecture)
6. Wires events across domains
7. Runs quality checks and coverage verification

### Output

- Complete DDD spec structure (same as `/ddd-create`)
- Coverage report at `.ddd/reverse/coverage.yaml` (file, entry point, model, event, function coverage)
- Intermediate files at `.ddd/reverse/` (resumable for swap/bottom-up/compiler/codex strategies)
- Summary with coverage percentage, warnings, and next steps

---

## /ddd-implement

Generate working code and tests from DDD specs.

### Usage

```
/ddd-implement [scope]
```

### Scope

| Argument | Scope | Example |
|---|---|---|
| `--all` | Whole project — all domains, all flows | `/ddd-implement --all` |
| `domain-name` | All flows in a domain | `/ddd-implement users` |
| `domain-name/flow-name` | Single flow | `/ddd-implement users/user-register` |
| *(empty)* | Interactive — shows flows and asks | `/ddd-implement` |

### Examples

```
# Implement everything
/ddd-implement --all

# Implement all flows in the orders domain
/ddd-implement orders

# Implement a single flow
/ddd-implement users/user-register

# Interactive mode — pick what to implement
/ddd-implement
```

### What it does

1. Finds `ddd-project.json` in current or parent directory
2. Reads all relevant specs (system, architecture, config, errors, schemas, domain, flow)
3. Checks `.ddd/mapping.yaml` for existing implementations (skip if up-to-date, update if drifted)
4. Generates code following the node graph from trigger to terminal:
   - Routes/handlers from trigger type
   - Validation from input nodes
   - DB operations from data_store nodes
   - API calls from service_call nodes
   - Branching from decision nodes
   - Event emission/consumption from event nodes
   - Agent/orchestration logic from agent nodes
5. Writes tests (happy path, branches, errors, validation)
6. Runs tests, fixes until passing
7. Updates `.ddd/mapping.yaml` with spec hash, timestamp, and file list

### Output

Summary table with domain/flow, status, file count, and test results.

---

## /ddd-update

Update DDD specs from natural language change requests.

### Usage

```
/ddd-update [scope] <description of change>
```

### Scope

| Argument | Scope | Example |
|---|---|---|
| `domain/flow` | Update a specific flow | `/ddd-update users/user-register add rate limiting` |
| `domain` | Update domain config or flows | `/ddd-update users add email verification flow` |
| `--add-flow domain` | Add a new flow to a domain | `/ddd-update --add-flow users` |
| `--add-domain` | Add a new domain | `/ddd-update --add-domain` |
| *(empty)* | Interactive — shows structure and asks | `/ddd-update` |

### Examples

```
# Modify a specific flow
/ddd-update users/user-register add rate limiting before the input node

# Add a new flow to a domain
/ddd-update --add-flow orders add a refund-order flow

# Add a new domain
/ddd-update --add-domain add a notifications domain with email and push flows

# Change a flow's behavior
/ddd-update orders/create-order add a coupon validation step before calculating total

# Interactive mode
/ddd-update
```

### What it does

1. Finds `ddd-project.json` and reads current specs
2. Parses the natural language change request
3. Modifies YAML spec files:
   - Adding/removing/modifying nodes in flows
   - Rewiring connections
   - Adding flows to domains
   - Adding domains to project
   - Updating event wiring
4. Preserves existing node IDs, positions, metadata, observability, and security configs
5. Handles cross-domain impacts (event renaming, removal warnings)
6. Validates spec integrity after changes

### Output

Shows which files changed, what was added/modified/removed in each, affected domains, and next steps (`/ddd-implement` to update code).

---

## /ddd-sync

Synchronize specs and implementation state.

### Usage

```
/ddd-sync [flags]
```

### Flags

| Flag | What it does |
|---|---|
| *(no flag)* | Sync `.ddd/mapping.yaml` with current implementation state |
| `--discover` | Also scan for untracked code and suggest new flow specs |
| `--fix-drift` | Re-implement flows where specs drifted from code |
| `--full` | All of the above: sync + discover + fix drift |

### Examples

```
# Basic sync — update mapping.yaml
/ddd-sync

# Find code that doesn't have specs yet
/ddd-sync --discover

# Re-implement drifted flows
/ddd-sync --fix-drift

# Everything — sync, discover, and fix
/ddd-sync --full
```

### What it does

1. Reads `.ddd/mapping.yaml` for tracked flow mappings
2. For each mapped flow:
   - Checks if implementation files still exist
   - Computes current spec hash, compares to stored hash
   - Updates file lists and timestamps
3. With `--discover`: scans `src/` for routes, services, models not mapped to any flow spec. Suggests new flow specs.
4. With `--fix-drift`: for each drifted flow (spec changed since implementation), re-implements from updated spec, runs tests, updates mapping.

### Output

Summary showing:
- Flows with up-to-date specs
- Flows with spec drift (spec changed since implementation)
- Flows with missing implementation
- (with `--discover`) Untracked code that should become flows
- (with `--fix-drift`) Flows that were re-implemented

---

## /ddd-scaffold

Set up project skeleton and shared infrastructure from specs. Run this as the first step of Phase 3 (Build), before `/ddd-implement`.

### Usage

```
/ddd-scaffold
```

### What it does

1. Reads `system.yaml`, `architecture.yaml`, `config.yaml`, `errors.yaml`, `types.yaml`, and all schema files
2. Initializes the project:
   - Package config (package.json, tsconfig, dependencies)
   - Project directory structure from architecture spec
3. Generates shared infrastructure:
   - Config loader from config.yaml
   - Error classes/handler from errors.yaml
   - Shared types from types.yaml
   - Database schema/models from schemas/
   - App entry point with middleware stack
   - Integration clients from system.yaml integrations
   - Event infrastructure (if domains use events)
   - Test configuration and utilities
4. Creates environment files (.env.example, .gitignore)
5. Verifies build and tests pass
6. Initializes `.ddd/mapping.yaml`

### Output

Summary of created files, models, error codes, integrations, and build/test status.

---

## /ddd-status

Quick read-only overview of project implementation state.

### Usage

```
/ddd-status [--json]
```

### Examples

```
# Table view
/ddd-status

# Machine-readable output
/ddd-status --json
```

### What it does

1. Reads `ddd-project.json`, all domain.yaml files, and `.ddd/mapping.yaml`
2. For each flow, computes status: **Up to date**, **Drifted**, **Stale**, or **Not implemented**
3. Checks scaffold state (package.json, entry point, database schema)
4. Shows a table with domain, flow, status, and implementation date
5. Suggests next actions based on what it finds

### Output

```
Domain          Flow                    Status          Implemented
users           user-register           Up to date      2025-12-15
users           user-login              Drifted         2025-12-14
orders          create-order            Not implemented —

Summary: 1 up to date, 1 drifted, 1 not implemented
```

---

## /ddd-test

Run tests for implemented flows without re-generating code.

### Usage

```
/ddd-test [scope] [--coverage]
```

### Scope

| Argument | Scope | Example |
|---|---|---|
| `--all` | All implemented flows | `/ddd-test --all` |
| `domain-name` | All flows in a domain | `/ddd-test users` |
| `domain-name/flow-name` | Single flow | `/ddd-test users/user-register` |
| *(empty)* | Interactive — shows flows and asks | `/ddd-test` |

### Examples

```
# Test everything
/ddd-test --all

# Test one domain
/ddd-test users

# Test with coverage
/ddd-test --all --coverage

# Test a single flow
/ddd-test users/user-register
```

### What it does

1. Reads `.ddd/mapping.yaml` to find test files for the scoped flows
2. Runs the test runner (auto-detected from config files)
3. Reports pass/fail per flow with detailed failure analysis
4. Identifies likely failure cause: spec drift, manual code change, environment issue, or dependency issue
5. Suggests fix actions (re-implement, manual fix, or environment fix)

### Output

Table with domain/flow, test counts, pass/fail, and failure analysis with suggestions.

---

## /ddd-evolve

Analyze DDD shortfall reports, critically evaluate each gap, and produce a prioritized recommendation plan for human decision-making.

### Usage

```
/ddd-evolve <shortfalls.yaml> [<shortfalls2.yaml> ...]
/ddd-evolve --dir <project-dir> [--dir <project-dir2> ...]
/ddd-evolve --review <evolution-plan.yaml>
/ddd-evolve --apply <evolution-plan.yaml>
```

### Modes

| Mode | What it does |
|------|-------------|
| *(default)* | Analyze shortfalls → produce `ddd-evolution-plan.yaml` |
| `--review` | Walk through each item interactively, collect approve/defer/reject decisions |
| `--apply` | Execute approved items from a reviewed plan |

### Options

| Flag | Purpose |
|------|---------|
| `--dir <path>` | Point to a DDD project directory. Auto-discovers `specs/shortfalls.yaml` inside it. Can be specified multiple times. Can be mixed with direct file paths. |

### Examples

```
# Analyze shortfalls from one project
/ddd-evolve ~/code/my-app/specs/shortfalls.yaml

# Analyze across multiple projects for stronger signal
/ddd-evolve ~/code/proj-a/specs/shortfalls.yaml ~/code/proj-b/specs/shortfalls.yaml

# Point to project directories (auto-discovers shortfalls.yaml)
/ddd-evolve --dir ~/code/proj-a --dir ~/code/proj-b

# Mix direct paths and --dir
/ddd-evolve ~/code/proj-a/specs/shortfalls.yaml --dir ~/code/proj-b

# Interactively review and decide on each item
/ddd-evolve --review ddd-evolution-plan.yaml

# Apply approved changes from reviewed plan
/ddd-evolve --apply ddd-evolution-plan.yaml
```

### What it does

1. **Analyze** — Reads shortfall files, deduplicates, evaluates through 6 filters (already possible? recurring? specific? breaking? adequate workaround? intentional?), classifies as `REAL_GAP`/`ENHANCEMENT`/`VAGUE`/`ALREADY_POSSIBLE`/`BY_DESIGN`/`PROJECT_SPECIFIC`, writes `ddd-evolution-plan.yaml`
2. **Review** — Presents each item with analysis and evidence, asks human to approve/defer/reject via interactive prompts, records decisions and notes back to the plan file
3. **Apply** — Requires reviewed plan. Shows what will change, asks confirmation, executes approved items (updates spec, commands, tool, validator)

### Output

- `ddd-evolution-plan.yaml` with tiered recommendations and decisions
- Interactive review walkthrough (with `--review`)
- Code/spec changes across DDD repos (with `--apply`)

---

## /ddd-reflect

Capture implementation wisdom — patterns and details that code has but specs don't describe.

### Usage

```
/ddd-reflect [scope]
```

### Scope

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | Entire project | `/ddd-reflect --all` |
| `domain-name` | All flows in a domain | `/ddd-reflect monitoring` |
| `domain-name/flow-name` | Single flow | `/ddd-reflect monitoring/check-social-sources` |
| *(empty)* | Interactive — shows flows and asks | `/ddd-reflect` |

### Examples

```
# Reflect on entire project
/ddd-reflect --all

# Reflect on a single domain
/ddd-reflect monitoring

# Reflect on a specific flow
/ddd-reflect monitoring/check-social-sources
```

### What it does

1. Reads flow spec YAML and implementation files from mapping.yaml
2. Compares what code does vs what spec describes
3. Classifies findings into pattern categories using `architecture.yaml` cross_cutting_patterns as reference
4. Writes annotations to `.ddd/annotations/{domain}/{flow}.yaml`
5. Updates mapping.yaml `annotationCount` for each flow
6. Reports: N patterns found, M new annotations, K already captured

### Output

Summary of discovered patterns with code evidence, written as annotation files for human review.

---

## /ddd-promote

Move approved annotations into permanent specs. This is how implementation wisdom becomes part of the design.

### Usage

```
/ddd-promote [scope]
```

### Scope

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | Promote all approved annotations | `/ddd-promote --all` |
| `--review` | Interactive review of candidates | `/ddd-promote --review` |
| `domain-name/flow-name` | Scope to specific flow | `/ddd-promote monitoring/check-social-sources` |

### Examples

```
# Interactive review of all annotations
/ddd-promote --review

# Promote all approved annotations
/ddd-promote --all

# Promote annotations for a specific flow
/ddd-promote monitoring/check-social-sources
```

### What it does

1. Reads `.ddd/annotations/` files
2. Groups by status: candidate, approved, dismissed
3. Presents candidates with code evidence for review
4. For approved patterns:
   - Cross-cutting → add to `architecture.yaml` cross_cutting_patterns
   - Flow-specific → enrich the flow spec YAML
   - Shared type/error → update shared/types.yaml or errors.yaml
5. Updates annotation status and mapping.yaml

### Output

Report of what was promoted and where (which spec files were updated).

---

## Typical Workflows

### New project from scratch

```
/ddd-create A task management app with users, projects, and tasks...
# Review in DDD Tool, adjust flows
/ddd-scaffold
/ddd-implement --all
/ddd-test --all
# After implementation stabilizes:
/ddd-reflect --all
/ddd-promote --review
```

### Existing codebase, no specs

```
/ddd-reverse ~/code/my-existing-app
# Review generated specs in DDD Tool
/ddd-scaffold
/ddd-implement --all
/ddd-test --all
/ddd-reflect --all       # Capture what reverse missed
/ddd-promote --review    # Feed back into specs
```

### Add a feature

```
/ddd-status                   # see what's implemented
/ddd-update users/user-register add email verification step after creation
/ddd-implement users/user-register
/ddd-test users/user-register
```

### Add a new domain

```
/ddd-update --add-domain add analytics domain with page-view tracking and dashboard flows
/ddd-implement analytics
/ddd-test analytics
```

### Code drifted from specs

```
/ddd-status                   # see which flows drifted
/ddd-sync --full              # classify and resolve drift
/ddd-reflect --all            # capture any code-ahead patterns
/ddd-promote --review         # promote approved patterns to specs
/ddd-test --all
```

### Ongoing maintenance

```
/ddd-status            # quick overview
/ddd-update ...        # make changes to specs
/ddd-implement ...     # update code
/ddd-test ...          # verify tests pass
/ddd-sync              # verify alignment
```

### Capture implementation wisdom

```
/ddd-reflect --all            # scan all flows for code patterns not in specs
/ddd-promote --review         # interactively approve/dismiss each finding
# Approved patterns are written to architecture.yaml or flow specs
# Future /ddd-implement runs will apply these patterns automatically
```
