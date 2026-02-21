# DDD Command Quality Checklist

Use this checklist when creating new DDD commands or auditing existing ones. Every item marked **Required** must be present. Items marked **Conditional** apply when the stated condition is true.

## 1. Purpose & Identity

- [ ] **Required** — First line is `# DDD {Name}` with a clear, one-sentence purpose
- [ ] **Required** — Purpose statement names which pillars the command operates on (Logic, Data, Interface, Infrastructure)
- [ ] **Required** — Purpose statement says what the command does (creates, reads, modifies, or deletes) and to what (specs, code, mapping, annotations)
- [ ] **Required** — States which lifecycle phase the command belongs to (Create, Build, Reflect, Any, Meta)

## 2. Inputs

- [ ] **Conditional** (command supports selective targeting) — Has a `## Scope Resolution` section with an argument table
- [ ] **Conditional** (has scope table) — Scope table covers all four pillars where applicable: `{domain}`, `{domain}/{flow}`, `--ui`, `--ui {page-id}`, `--schema`, `--schema {model}`, `--infra`
- [ ] **Conditional** (has scope table) — Scope table includes `--all` and `*(empty)*` (interactive) rows
- [ ] **Conditional** (has scope table) — Every scope argument has an Example column showing exact usage
- [ ] **Required** — Lists all files the command reads, with bullet points explaining what each provides
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
- [ ] **Conditional** (command modifies spec files) — Preserves existing content — only adds, never removes unless explicitly requested
- [ ] **Conditional** (command modifies spec files) — Updates `metadata.modified` timestamp on changed specs

## 5. Outputs

- [ ] **Required** — Has a Report or Summary section with a formatted example showing exact output structure
- [ ] **Required** — Report covers all four pillars the command touches (not just Logic)
- [ ] **Conditional** (command writes implementation) — Updates `.ddd/mapping.yaml` with `specHash`, `implementedAt`, `files`, `syncState`
- [ ] **Conditional** (command creates annotations) — Writes to `.ddd/annotations/` with correct paths: `{domain}/{flow}.yaml`, `ui/{page-id}.yaml`, `schemas/{model}.yaml`, `infrastructure.yaml`

## 6. Guidance

- [ ] **Required** — Has a "Next steps" section recommending the follow-up command(s)
- [ ] **Required** — Next steps reference only commands that actually exist and use correct syntax
- [ ] **Required** — Next steps cover the complete workflow path (e.g., implement recommends test, test recommends reflect)
- [ ] **Conditional** (command can cause data loss) — Has explicit warnings about destructive actions (e.g., re-implementation overwrites code)

## 7. Cross-Command Consistency

- [ ] **Required** — Scope arguments use the same syntax as peer commands (see Table 2 below)
- [ ] **Required** — File paths match the canonical project structure: `specs/domains/*/flows/*.yaml`, `specs/ui/{page-id}.yaml`, `specs/schemas/{model}.yaml`, `specs/infrastructure.yaml`, `.ddd/mapping.yaml`, `.ddd/annotations/`
- [ ] **Required** — mapping.yaml field names are consistent: `specHash`, `syncState`, `annotationCount`, `implementedAt`, `files`, `mode`
- [ ] **Required** — Pillar names are consistent: Logic, Data, Interface, Infrastructure (never "Frontend", "Backend", "Database")

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

## Reference: Lifecycle Phases

Each command should know its place in the workflow and recommend the correct next command:

```
Phase 1 CREATE ──→ Phase 2 DESIGN ──→ Phase 3 BUILD ──→ Phase 4 REFLECT
/ddd-create         (DDD Tool)         /ddd-scaffold      /ddd-sync
/ddd-reverse                           /ddd-implement      /ddd-reflect
                                       /ddd-test           /ddd-promote

Cross-cutting: /ddd-status, /ddd-update
Meta-level: /ddd-evolve
```

| After this command... | ...recommend this |
|----------------------|-------------------|
| `/ddd-create` | Open in DDD Tool, then `/ddd-scaffold` |
| `/ddd-reverse` | Review in DDD Tool, then `/ddd-implement --all` |
| `/ddd-scaffold` | `/ddd-implement --all` |
| `/ddd-implement` | `/ddd-test --all` |
| `/ddd-test` | Fix failures or proceed to `/ddd-sync` |
| `/ddd-update` | `/ddd-implement {scope}` or `/ddd-scaffold` |
| `/ddd-sync` | `/ddd-reflect` for code-ahead, `/ddd-implement` for new-logic |
| `/ddd-reflect` | `/ddd-promote --review` |
| `/ddd-promote` | `/ddd-sync` to update hashes |
| `/ddd-evolve` | `/ddd-evolve --review` then `--apply` |
