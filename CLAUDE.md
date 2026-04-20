# DDD — Development Context

## What This Repo Is

The DDD specification and guide repository. Primary artifacts:
- `DDD-USAGE-GUIDE.md` — source of truth for all YAML spec formats, node types, spec fields, connection patterns, trigger conventions, and validation rules
- `templates/` — YAML templates for each spec type (architecture, config, errors, infrastructure, schema, ui-page)

## Before Editing

**Before editing the Usage Guide or any template**, scan all `##` headings in `DDD-USAGE-GUIDE.md` to understand the full section structure. The guide is the source of truth — all DDD commands, the DDD Tool, and ddd-tool source code derive from it. Changes here propagate everywhere.

## Git Remotes (Dual Remote Setup)
- `origin` → github.com/mhcandan/DDD (private, master repo)
- `public` → github.com/cybersoloss/DDD (public mirror)

**Push to both:** `git push-all` — pushes to origin normally, then pushes to public with `CLAUDE.md` and `.claude/` excluded. These stay on mhcandan only.

**IMPORTANT — Always use `git push-all` instead of `git push`.** Never use `git push origin` — it skips the public mirror.

## Change Authoring Principles

Changes to the guide and templates must be **domain-agnostic**. The guide defines mechanisms (node types, spec fields, connection patterns) — not domain-specific implementations. When examples are needed, use diverse domains rather than examples from any one project.

After any guide change, perform a DDD-PCCA (see below).

## DDD-FFR (Fresh Framework Read)

The source of truth for guide and template work is `DDD-USAGE-GUIDE.md` itself. All other files (command summaries in `DDD-commands.md`, CLAUDE.md descriptions, ddd-tool architecture notes) derive from it and may lag behind.

**DDD-FFR is read-only. Do NOT edit files, run commands, or make any changes during FFR — regardless of what inconsistencies or gaps are found. FFR only loads context.**

**Before any guide or template work:**

1. Scan all `##` headings in the guide to build a complete section map:
   ```bash
   grep "^## " ~/dev/DDD/DDD-USAGE-GUIDE.md
   ```
   Note every section number and title. This is your table of contents — do not rely on any summary of what sections exist.

2. For your specific task, identify which sections are relevant from the heading scan, then read those sections in full from the guide itself. If no specific task is given, read all sections sequentially.

3. If the task touches commands or tool code, read the relevant guide section first, then the command file or source file. Never edit a command based on another file's description of what the guide says.

**Rules:**
- Do NOT use `DDD-commands.md`, CLAUDE.md session notes, or any other derivative file to decide what the guide says. Use derivative files only after forming your own understanding from the guide, and only to find what else needs updating.
- Do NOT trust section counts, node type counts, or feature lists from any file other than the guide itself. If a file says "30 node types", verify by reading Section 6 of the guide.
- When editing a template, read the corresponding guide section for that spec type first. The template must match the guide exactly.
- After any guide change, perform a DDD-PCCA (see below).

## DDD-PCCA (Post-Change Consistency Audit)

After ANY change to DDD artifacts in any of the 3 repos (DDD, ddd-tool, claude-commands), audit all dependent artifacts for consistency. This is mandatory — not optional. The DDD ecosystem has multi-directional dependencies:

```
Usage Guide changes → commands, tool (validator, normalizer, editors, nodes, types), templates, vocabulary gate
Command changes → other commands, Usage Guide (if gap found), tool normalizer (if new format)
Tool changes → Usage Guide (if new validation), commands (if format dropped)
```

### Audit Checklist by Change Type

| What Changed | Where | What to Audit |
|---|---|---|
| Node type added/removed | Guide | All 11 `ddd-*.md` commands, tool `nodeTypes` registry + `defaultSpec` + `defaultLabel` + toolbar + validator + normalizer, vocabulary gate in 6 commands, sourceHandle Reference + Spec Field Reference tables |
| Spec field added/renamed/removed | Guide | Validator field checks, normalizer aliases, spec editor UI, commands' required-fields tables, vocabulary gate, Spec Field Reference table |
| Handle name changed | Guide | sourceHandle Reference table, validator handle checks, normalizer remappings, commands' handle lists, vocabulary gate, node components (Handle elements) |
| Connection format changed | Guide | Normalizer `normalizeNode`, commands' connection instructions, Connection Field Reference table |
| Trigger convention changed | Guide | Commands' trigger lists, normalizer trigger synthesis, validator trigger checks |
| Validation rule added/changed | Guide or Tool | Validator, commands' integrity check lists, ddd-tool-architecture.md |
| Template changed | Guide | Corresponding Usage Guide section for that spec type |
| Command node instructions | Commands | Usage Guide matches, other commands referencing same node, tool handles the output |
| Command field/handle name | Commands | Usage Guide Reference tables match, vocabulary gate matches across all 6 commands, tool validator expects it |
| Vocabulary gate modified | Commands | Same gate text across all 6 YAML-writing commands (create, update, reverse, sync, evolve, promote) |
| New YAML pattern in command | Commands | Tool normalizer can parse it, validator doesn't reject it |
| Validator new check | Tool | Usage Guide documents field as required, Spec Field Reference includes it, commands' tables include it |
| Normalizer alias added/removed | Tool | Commands still produce accepted format, Connection Field Reference reflects aliases |
| Spec editor new field | Tool | Usage Guide documents field, `flow.ts` type includes it |
| Node component added | Tool | Registered in `nodes/index.ts`, in `NodeToolbar.tsx`, in `flow.ts`, guide documents type |

### How to Audit

```bash
# Find all references to a changed term across the DDD ecosystem
grep -rn "{term}" ~/dev/DDD/DDD-USAGE-GUIDE.md ~/dev/DDD/templates/
grep -rn "{term}" ~/.claude/commands/ddd-*.md
grep -rn "{term}" ~/dev/ddd-tool/src/

# Verify vocabulary gate is identical across all 6 YAML-writing commands
for cmd in ddd-create ddd-update ddd-reverse ddd-sync ddd-evolve ddd-promote; do
  echo "=== $cmd ===" && grep -A12 "VOCABULARY GATE" ~/.claude/commands/$cmd.md
done
```

### Key Tool Files to Cross-Reference

| DDD Tool File | Must Match |
|---|---|
| `src/utils/flow-validator.ts` | Usage Guide § Validation Rules + § DDD Vocabulary Reference |
| `src/utils/flow-normalizer.ts` | Usage Guide § Connection Field Reference (accepted aliases) |
| `src/types/flow.ts` | Usage Guide § node type spec field definitions |
| `src/components/FlowCanvas/nodes/index.ts` | All node types registered |
| `src/components/FlowCanvas/NodeToolbar.tsx` | All node types in toolbar |
| `src/components/SpecPanel/editors/` | One editor per node type, fields matching Usage Guide |
