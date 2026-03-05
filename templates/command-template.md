# DDD Command Template

Structural skeleton for DDD slash commands. Copy this template when creating a new command, fill in the placeholders, and remove sections that don't apply. Validate the result against `command-checklist.md`.

## How to use

1. Copy this file to `~/.claude/commands/ddd-{name}.md`
2. Replace all `{placeholders}` with actual values
3. Remove sections marked `<!-- SKIP IF ... -->` when the condition applies
4. Validate against `command-checklist.md` before finalizing

## Who uses this

- `/ddd-evolve --apply` — when creating new commands as part of framework evolution
- Framework maintainers — when manually adding a command
- Auditors — compare existing commands against this skeleton to find structural drift

---

<!-- ═══════════════════════════════════════════════════════════════════ -->
<!-- TEMPLATE STARTS BELOW — everything above this line is guidance    -->
<!-- ═══════════════════════════════════════════════════════════════════ -->

# DDD {Name}

{One-sentence description: (1) what the command does, (2) which pillars it operates on — Logic, Data, Interface, Infrastructure — and (3) what artifacts it creates/reads/modifies (specs, code, mapping, annotations).} **Lifecycle phase: {Create | Build | Reflect | Any | Meta}.**

<!-- SKIP IF the command has a single mode with no explanation needed -->
**Core principle:** {State the fundamental design principle — e.g., "Specs and code are BOTH sources of truth at different levels."}

<!-- ─── INPUT SECTION ─── Choose ONE of the following two patterns ─── -->

<!-- PATTERN A: Named flags (use for: ddd-create, ddd-reverse, ddd-evolve) -->

## Options

- `--flag-name` — {Description. Include default behavior if not specified.}
- `--another-flag <value>` — {Description.}

<!-- PATTERN B: Scope resolution table (use for: ddd-implement, ddd-test, ddd-update, ddd-reflect, ddd-promote) -->

## Scope Resolution

Parse `$ARGUMENTS` to determine scope:

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | All {items} across all domains and pillars | `/ddd-{name} --all` |
| `{domain}` | All flows in a domain | `/ddd-{name} users` |
| `{domain}/{flow}` | Single flow | `/ddd-{name} users/user-register` |
| `--ui` | All UI pages | `/ddd-{name} --ui` |
| `--ui {page-id}` | Single page | `/ddd-{name} --ui dashboard` |
| `--schema` | All schemas | `/ddd-{name} --schema` |
| `--schema {model}` | Single schema model | `/ddd-{name} --schema user` |
| `--infra` | Infrastructure | `/ddd-{name} --infra` |
| *(empty)* | {Default: change-history driven, interactive, or error} | `/ddd-{name}` |

<!-- Remove rows that don't apply. See command-checklist.md Scope Argument Matrix for which rows are required per command. -->

<!-- ─── FILE DECLARATIONS ─── Both sections are REQUIRED ─── -->

**Files read:**
- `ddd-project.json` — project config, domain list
- `.ddd/mapping.yaml` — implementation tracking (specHash, syncState, files, fileHashes, implementedAt, annotationCount, mode)
- `.ddd/change-history.yaml` — pending and recent change entries
- `specs/architecture.yaml` — conventions, cross-cutting patterns
- `specs/domains/*/domain.yaml` — domain configs, event definitions
- `specs/domains/*/flows/*.yaml` — flow specs with node graphs
- `specs/schemas/*.yaml` — data model definitions
- `specs/ui/pages.yaml` — page registry, navigation, theme
- `specs/ui/*.yaml` — per-page specs
- `specs/infrastructure.yaml` — services, ports, deployment
- DDD Usage Guide (fetched via `gh api`) — YAML formats, node types, spec fields reference
<!-- Remove items this command doesn't read. Keep descriptions consistent with other commands. -->

**Files written:**
- {path-or-pattern} — {what is written and when}
- {path-or-pattern} — {what is written and when}
<!-- OR for read-only commands: -->
<!-- **Files written:** None ({reason — e.g., "read-only command", "runs tests only"}) -->

## Instructions

1. **Find the DDD project**: Look for `ddd-project.json` in the current directory or parent directories.

2. **Read project context**:
   - Read `ddd-project.json` for domain list and project config
   - Read `.ddd/mapping.yaml` for implementation tracking
   - {List all other files to read — must match the Files read section above}

<!-- SKIP IF command doesn't create or modify node-level specs -->
3. **Fetch the DDD Usage Guide**: Run `gh api repos/cybersoloss/DDD/contents/DDD-USAGE-GUIDE.md --jq '.content' | base64 -d` to get the latest version. This guide defines all YAML formats, node types, spec fields, connection patterns, UI spec format, infrastructure spec format, and conventions.

<!-- SKIP IF command processes a single item (not multi-pillar) -->
4. **Create plan**: Before generating anything, enumerate every item to process across all four pillars:

   | Pillar | Items | Count |
   |--------|-------|-------|
   | Data | {schemas, migrations, seeds to process} | {N} |
   | Interface | {pages, components, stores to process} | {N} |
   | Infrastructure | {services, configs, scripts to process} | {N} |
   | Logic | {flows, routes, services to process} | {N} |

   This plan is your commitment — every item listed must be processed.

   **Processing order:** Data → Interface → Infrastructure → Logic. Detail-heavy pillars last to prevent context exhaustion starving lighter pillars.

<!-- ─── PER-PILLAR PROCESSING ─── Repeat this block for each pillar ─── -->

5. **Process Data**:
   - {Data-specific instructions}

   **Checkpoint:** Output "Data complete: {N}/{N} items processed"

   **GATE:** Compare actual count to plan. If any planned item is missing, STOP and process it now before proceeding.

6. **Process Interface**:

   **Interface is the most commonly skipped pillar.** If the plan includes ANY pages, all must be processed. Zero tolerance.

   - {Interface-specific instructions}

   **Checkpoint:** Output "Interface complete: {N}/{N} items processed"

   **GATE:** Compare actual count to plan. If any planned item is missing, STOP and process it now before proceeding.

7. **Process Infrastructure**:
   - {Infrastructure-specific instructions}

   **Checkpoint:** Output "Infrastructure complete: {N}/{N} items processed"

   **GATE:** Compare actual count to plan. If any planned item is missing, STOP and process it now before proceeding.

8. **Process Logic**:
   - {Logic-specific instructions — this is typically the largest section}

   **Checkpoint:** Output "Logic complete: {N}/{N} items processed"

   **GATE:** Compare actual count to plan. If any planned item is missing, STOP and process it now before proceeding.

   **Concept disambiguation:** When a concept appears in multiple pillars (e.g., "Dashboard" as both a backend domain and a frontend page), both representations MUST be processed. Do not assume one covers the other.

<!-- ─── END PER-PILLAR PROCESSING ─── -->

<!-- SKIP IF command doesn't modify specs -->
9. **Preserve spec integrity**:
   - Never remove existing fields unless explicitly requested
   - Preserve node IDs, positions, connections
   - Update `metadata.modified` to current ISO timestamp on any changed spec

<!-- SKIP IF command doesn't interact with change-history -->
10. **Update change-history**: Append entries to `.ddd/change-history.yaml`:
    - Check for existing entries with the same `spec_file` and matching `spec_checksum` — skip duplicates
    - Use the format:
    ```yaml
    - id: "chg-{next 4-digit sequential id}"
      timestamp: "{ISO 8601}"
      source: ddd-{name}
      change_type: {added | modified | enriched}
      scope:
        level: L3
        domain: "{domain}"
        flow: "{flow}"
        pillar: "{pillar}"
      spec_file: "{relative path to spec file}"
      spec_checksum: "{SHA-256 first 12 chars}"
      status: pending_implement
      implemented_at: null
      code_files: []
    ```

<!-- SKIP IF command doesn't update mapping -->
11. **Update mapping.yaml**: For each processed entry:
    - Compute and set `specHash` (SHA-256 of spec file content)
    - Set `files` list with all implementation source files
    - Compute and set `fileHashes` (SHA-256 of each implementation file)
    - Set `syncState`: `in_sync` | `spec_ahead` | `code_ahead` | `diverged` | `new_logic`
    - Set `implementedAt` timestamp
    - Set `mode`: `new` | `update`
    - Set `annotationCount` if annotations exist

12. **Display results**:

    ```
    {Command Name} — {project-name}

    ── Logic (Flows) ──────────────────────────────────────────────────────
    {Domain}        {Flow}                  {Status/Result}      {Detail}
    ─────────────── ─────────────────────── ──────────────────── ──────────
    ...

    ── Interface (Pages) ──────────────────────────────────────────────────
    ...

    ── Data (Schemas) ─────────────────────────────────────────────────────
    ...

    ── Infrastructure ─────────────────────────────────────────────────────
    ...

    Pillar balance: Logic {N}, Interface {N}, Data {N}, Infrastructure {N}
    ```

13. **Next steps**: Based on results, suggest the appropriate follow-up command(s).

    - {condition}: "Run `/ddd-{next} {scope}`"
    - {condition}: "Run `/ddd-{alt} {scope}`"

    **Anti-patterns — never suggest these:**
    - `/ddd-reflect` immediately after `/ddd-implement` — code is freshly generated from specs, there is no manual wisdom to capture
    - `/ddd-implement` for metadata-only drift — use `/ddd-sync` to update the hash instead
    - `--all` scope unless the user's original scope was `--all` — match the scope
    - `/ddd-sync` or `/ddd-reflect` immediately after passing tests — Build phase is complete, the user moves on

$ARGUMENTS
