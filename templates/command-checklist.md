# DDD Command Quality Checklist

Use this checklist when creating new DDD commands or auditing existing ones. Every item marked **Required** must be present. Items marked **Conditional** apply when the stated condition is true.

Companion file: `command-template.md` provides the structural skeleton to copy-and-fill.

## 1. Purpose & Identity

- [ ] **Required** — First line is `# DDD {Name}` with a clear, one-sentence purpose
- [ ] **Required** — Purpose statement names which pillars the command operates on (Logic, Data, Interface, Infrastructure)
- [ ] **Required** — Purpose statement says what the command does (creates, reads, modifies, or deletes) and to what (specs, code, mapping, annotations)
- [ ] **Required** — States which lifecycle phase the command belongs to: **Create**, **Build**, **Reflect**, **Any** (cross-cutting), or **Meta**

## 2. Inputs

- [ ] **Conditional** (command supports selective targeting) — Has a `## Scope Resolution` section with an argument table
- [ ] **Conditional** (has scope table) — Scope table covers all four pillars where applicable: `{domain}`, `{domain}/{flow}`, `--ui`, `--ui {page-id}`, `--schema`, `--schema {model}`, `--infra`
- [ ] **Conditional** (has scope table) — Scope table includes `--all` and `*(empty)*` rows
- [ ] **Conditional** (has scope table) — Every scope argument has an Example column showing exact usage
- [ ] **Required** — Has a `**Files read:**` section listing all files the command reads, with bullet points explaining what each provides
- [ ] **Required** — Has a `**Files written:**` section listing all files the command creates or modifies. Read-only commands state `**Files written:** None ({reason})`
- [ ] **Required** — Files read and Files written sections appear BEFORE `## Instructions`, AFTER the scope/options section
- [ ] **Required** — Ends with `$ARGUMENTS` placeholder on the last line

## 3. Project Context

- [ ] **Required** (except create/reverse/evolve) — Has a "Find the DDD project" step that looks for `ddd-project.json`
- [ ] **Required** — Has a "Read project context" step listing all spec files it consumes
- [ ] **Conditional** (command creates or modifies nodes) — Fetches the DDD Usage Guide via `gh api repos/cybersoloss/DDD/contents/DDD-USAGE-GUIDE.md --jq '.content' | base64 -d`
- [ ] **Conditional** (fetches Usage Guide) — Usage Guide description matches canonical wording: "all YAML formats, node types, spec fields, connection patterns, UI spec format, infrastructure spec format, and conventions"

## 4. Processing Logic

- [ ] **Required** — Instructions are numbered sequentially with no gaps or duplicates
- [ ] **Required** — Each step has a bold title describing the action (e.g., `7. **Create schema specs**:`)
- [ ] **Conditional** (command writes to specs or code) — Has quality checks or validation before finishing
- [ ] **Conditional** (command works with flows) — References `architecture.yaml` `cross_cutting_patterns` and explains when/how to apply them

## 5. Safety & Idempotency

- [ ] **Required** — Command validates arguments before processing (invalid scope shows a clear error, not silent failure)
- [ ] **Conditional** (command is scoped) — A scoped invocation (e.g., `users/user-login`) only touches files within that scope, never modifies unrelated flows/pages/schemas
- [ ] **Conditional** (command can destroy user work) — Has a `**WARNING:**` block BEFORE the destructive action explaining what will be overwritten and suggesting `git commit` first
- [ ] **Conditional** (command modifies spec files) — Preserves existing content: node IDs, positions, connections, `metadata.created`. Only adds or updates, never removes unless explicitly requested
- [ ] **Conditional** (command modifies spec files) — Updates `metadata.modified` to current ISO timestamp on every changed spec file
- [ ] **Conditional** (command writes to change-history) — Deduplicates before appending: checks for existing entries with the same `spec_file` and matching `spec_checksum`, skips duplicates
- [ ] **Conditional** (command writes to mapping.yaml) — Running the command twice on the same input produces the same mapping state (no duplicate entries, no timestamp churn on unchanged files)

## 6. Outputs

- [ ] **Required** — Has a Report or Summary section with a formatted example showing exact output structure
- [ ] **Required** — Report covers all four pillars the command touches (not just Logic)
- [ ] **Required** — Report uses consistent pillar section headers: `── Logic (Flows) ──`, `── Interface (Pages) ──`, `── Data (Schemas) ──`, `── Infrastructure ──`
- [ ] **Conditional** (command generates across multiple pillars) — Report includes a `Pillar balance:` line showing counts per pillar so imbalances are visually obvious
- [ ] **Conditional** (command writes implementation) — Updates `.ddd/mapping.yaml` with `specHash`, `implementedAt`, `files`, `fileHashes`, `syncState`
- [ ] **Conditional** (command creates annotations) — Writes to `.ddd/annotations/` with correct paths: `{domain}/{flow}.yaml`, `ui/{page-id}.yaml`, `schemas/{model}.yaml`, `infrastructure.yaml`

## 7. Guidance

- [ ] **Required** — Has a "Next steps" section recommending follow-up command(s)
- [ ] **Required** — Next steps reference only commands that actually exist and use correct syntax
- [ ] **Required** — Next steps match scope: don't suggest `--all` unless the user's original scope was `--all`
- [ ] **Required** — Next steps follow the lifecycle successor table (see Reference section below)
- [ ] **Conditional** (Build phase command) — Does NOT suggest `/ddd-reflect` or `/ddd-promote` as immediate next steps after freshly generated code. Reflect captures wisdom from *manual* edits, not fresh generation.
- [ ] **Conditional** (command detects drift) — Does NOT suggest `/ddd-implement` for metadata-only or spec-enriched drift. Recommends `/ddd-sync` to update hash instead.
- [ ] **Conditional** (command detects code-ahead drift) — Recommends `/ddd-reflect` first (capture wisdom), then `/ddd-promote --review`, then `/ddd-sync`. Never skips reflect to go straight to implement.
- [ ] **Conditional** (all tests pass) — Suggests "Build loop complete — continue with your next change" rather than escalating to Reflect phase commands.

## 8. Change-History Pipeline

These items ensure the change-history.yaml handoff chain works correctly across commands.

- [ ] **Conditional** (command creates or modifies specs) — Appends `pending_implement` entries to `.ddd/change-history.yaml` for each changed spec file
- [ ] **Conditional** (command writes change-history entries) — Entry format includes all required fields: `id`, `timestamp`, `source`, `change_type`, `scope` (with `level`, `domain`, `flow`, `pillar`), `spec_file`, `spec_checksum`, `status`, `implemented_at`, `code_files`
- [ ] **Conditional** (command writes change-history entries) — `source` field correctly identifies the command (e.g., `ddd-update`, `ddd-sync`, `ddd-promote`)
- [ ] **Conditional** (command implements code from specs) — Reads `pending_implement` entries from change-history and processes them
- [ ] **Conditional** (command implements code from specs) — Marks processed entries as `implemented` with `implemented_at` timestamp and populated `code_files` list
- [ ] **Conditional** (command reads change-history for scoping) — Skips entries with `status: pending_implement` when looking for recently implemented items (those have no code yet)
- [ ] **Conditional** (command does NOT interact with change-history) — Does not accidentally read or write change-history (e.g., ddd-scaffold, ddd-reflect, ddd-evolve should not touch it)

**Pipeline map** — which commands write vs. read change-history:

| Role | Commands |
|------|----------|
| **Writers** (append `pending_implement`) | ddd-create, ddd-reverse, ddd-update, ddd-sync, ddd-promote |
| **Processors** (read pending → mark `implemented`) | ddd-implement |
| **Readers** (scope to recent entries, never write) | ddd-test, ddd-status |
| **Non-participants** (do not interact) | ddd-scaffold, ddd-reflect, ddd-evolve |

## 9. Cross-Command Consistency

- [ ] **Required** — Scope arguments use the same syntax as peer commands (see Scope Argument Matrix below)
- [ ] **Required** — File paths match the canonical project structure: `specs/domains/*/flows/*.yaml`, `specs/ui/{page-id}.yaml`, `specs/schemas/{model}.yaml`, `specs/infrastructure.yaml`, `.ddd/mapping.yaml`, `.ddd/annotations/`
- [ ] **Required** — mapping.yaml field names are consistent: `specHash`, `syncState`, `annotationCount`, `implementedAt`, `files`, `fileHashes`, `mode`
- [ ] **Required** — Pillar names are consistent: Logic, Data, Interface, Infrastructure (never "Frontend", "Backend", "Database")
- [ ] **Required** — Drift classification uses the same 4 categories everywhere: metadata-only, spec enriched (code covers it), code ahead of spec, new spec logic
- [ ] **Required** — SHA-256 hash computation is described consistently (specHash = hash of spec YAML content, fileHashes = hash of each implementation file)

## 10. Usage Guide Alignment

These items verify the command's behavior matches what the DDD Usage Guide (DDD-USAGE-GUIDE.md) says about it.

- [ ] **Required** — Command's scope table matches the Usage Guide's command reference table for that command (same flags, same empty-scope behavior)
- [ ] **Required** — Command's lifecycle phase matches the Usage Guide's lifecycle diagram
- [ ] **Required** — Drift resolution recommendations match the Usage Guide's drift decision tree (Section 12.3)
- [ ] **Required** — Node type references match the Usage Guide's 29 node types (no invented or missing types)
- [ ] **Conditional** (command creates flow specs) — Node ID convention matches the Usage Guide's format: `{type}-{6char-hash}` (e.g., `process-a1b2c3`)
- [ ] **Conditional** (command creates UI specs) — Page spec format matches the Usage Guide's PageSpec structure (sections, forms, data_source, interactions)
- [ ] **Conditional** (command references `/ddd-evolve` filters) — Filter count matches the Usage Guide (currently 7 filters: A-G)

## 11. Pillar Starvation Prevention

These items apply to commands that generate or process across multiple pillars — primarily `ddd-create`, `ddd-implement`, `ddd-scaffold`, `ddd-reverse`, and `ddd-sync`. They prevent the LLM from "forgetting" a pillar mid-execution.

- [ ] **Conditional** (multi-pillar generation) — **Enumerate-before-generate**: the command lists all items it will produce per pillar (plan table) before generating any output, creating a countable commitment that can be verified later
- [ ] **Conditional** (multi-pillar generation) — **Generation ordering**: detail-heavy pillars (typically Logic/flows) are generated last so lighter pillars (Interface, Data, Infrastructure) are not starved by context exhaustion
- [ ] **Conditional** (multi-pillar generation) — **Per-pillar checkpoints**: the command outputs a progress line after completing each pillar (e.g., "Interface complete: 5/5 pages generated") so gaps are visible mid-execution
- [ ] **Conditional** (multi-pillar generation) — **Hard gates between pillars**: after generating a pillar, the command compares actual count to the planned count and stops with an error if there is a mismatch — does not defer validation to the end-of-command quality check
- [ ] **Conditional** (multi-pillar generation) — **Concept disambiguation**: when a concept has dual-pillar representation (e.g., "Dashboard" as both a backend domain and a frontend page), the command includes an explicit warning that both representations must exist
- [ ] **Conditional** (multi-pillar generation) — **Skip-prone pillar enforcement**: the command identifies which pillar is most likely to be omitted (typically Interface for backend-heavy projects) and adds extra gates or enforcement specifically for it

---

## Reference: Scope Argument Matrix

Commands with `## Scope Resolution` should support arguments appropriate to their role. Use this matrix to check coverage:

| Argument | implement | test | update | reflect | promote |
|----------|-----------|------|--------|---------|---------|
| `--all` | Required | Required | — | Required | Required |
| `{domain}` | Required | Required | Required | Required | Required |
| `{domain}/{flow}` | Required | Required | Required | Required | Required |
| `--ui` | Required | Required | Required | Required | Required |
| `--ui {page-id}` | Required | Required | Required | Required | Required |
| `--schema` | Required | Required | — | Required | Required |
| `--schema {model}` | Required | — | Required | Required | Required |
| `--infra` | Required | Required | Required | Required | Required |
| *(empty)* | Required | Required | Required | Required | Required |

Notes:
- `ddd-update` uses `--add-flow`, `--add-domain`, `--add-page` instead of `--all` (creation, not batch processing)
- `ddd-test --schema` runs schema validation, not per-model tests (no `--schema {model}`)
- Commands without scope tables (scaffold, status, create, reverse, sync, evolve) have their own argument conventions

## Reference: Lifecycle Successor Table

Each command should know its place in the workflow and recommend the correct next command:

```
Phase 1 CREATE ──→ Phase 2 DESIGN ──→ Phase 3 BUILD ──→ Phase 4 REFLECT
/ddd-create         (DDD Tool)         /ddd-scaffold      /ddd-sync
/ddd-reverse                           /ddd-implement      /ddd-reflect
                                       /ddd-test           /ddd-promote

Cross-cutting: /ddd-status, /ddd-update
Meta-level: /ddd-evolve
```

| After this command... | ...recommend this | Anti-pattern (never suggest) |
|----------------------|-------------------|------------------------------|
| `/ddd-create` | Open in DDD Tool, then `/ddd-scaffold` | — |
| `/ddd-reverse` | Review in DDD Tool, then `/ddd-sync --verify` (code already exists — never implement) | `/ddd-implement` (overwrites existing working code) |
| `/ddd-scaffold` | `/ddd-implement --all` | — |
| `/ddd-implement` | `/ddd-test {scope}` | `/ddd-reflect` (code is fresh, no wisdom) |
| `/ddd-test` (pass) | "Build complete — continue with next change" | `/ddd-reflect`, `/ddd-sync` (not needed yet) |
| `/ddd-test` (fail) | Fix code or `/ddd-implement {scope}` if spec drift confirmed | `/ddd-implement --all` (over-broad) |
| `/ddd-update` | `/ddd-implement {scope}` for changed specs | `/ddd-implement --all` (only changed scope) |
| `/ddd-sync` (code-ahead) | `/ddd-reflect {scope}` → `/ddd-promote` → `/ddd-sync` | `/ddd-implement` (destroys code wisdom) |
| `/ddd-sync` (new-logic) | `/ddd-implement {scope}` | — |
| `/ddd-sync` (metadata) | Hash updated automatically | `/ddd-implement` (nothing to implement) |
| `/ddd-reflect` | `/ddd-promote --review` | — |
| `/ddd-promote` | `/ddd-sync` to update hashes | — |
| `/ddd-status` | Context-dependent (see status command for rules) | `/ddd-implement` for metadata-only drift |
| `/ddd-evolve` | `--review` then `--apply` (sequential) | — |

## Reference: Cross-Cutting Patterns Lifecycle

Commands interact with `architecture.yaml` `cross_cutting_patterns` at different stages:

| Stage | Command | Interaction |
|-------|---------|-------------|
| **Generate** | ddd-scaffold | Creates utility files for patterns with a `utility` field |
| **Apply** | ddd-implement | Applies matching patterns to every flow during code generation |
| **Reference** | ddd-update | Checks patterns when adding new nodes (awareness) |
| **Detect** | ddd-sync | Identifies code using patterns the spec doesn't mention (code-ahead classification) |
| **Classify** | ddd-reflect | Uses patterns to categorize implementation findings (avoid flagging known patterns) |
| **Analyze** | ddd-test | Considers patterns when analyzing test failures (pattern-related vs bug) |
| **Consolidate** | ddd-promote | Promotes per-flow patterns appearing in 2+ flows to `cross_cutting_patterns` |
| **Display** | ddd-status | Uses patterns for drift classification context |
