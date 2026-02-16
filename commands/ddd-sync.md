# DDD Sync

Synchronize the DDD project specs with the current implementation state. This is the primary tool for resolving drift between specs and code.

**Core principle:** Specs and code are BOTH sources of truth at different levels. Specs describe *what* and *why*; code may include *how* details the spec doesn't capture. Sync must respect both directions.

## Instructions

1. **Find the DDD project**: Look for `ddd-project.json` in the current directory or parent directories.

2. **Scan the implementation**:
   - Read `.ddd/mapping.yaml` for currently tracked flow mappings
   - For each mapped flow, check if the implementation files still exist
   - Look for any new source files that aren't mapped yet

3. **Bidirectional drift analysis** (CRITICAL — do NOT skip):

   For each flow where the specHash doesn't match the current flow YAML:

   a. **Spec → Code check:** Read the current spec YAML. Identify what changed from the mapping's hash. Read the implementation code files. Does the code already implement the spec's intent?

   b. **Code → Spec check:** Does the implementation code contain patterns, error handling, integrations, or logic that the spec doesn't describe?

   c. **Classify the drift** (same categories as `/ddd-status`):
      - **Metadata-only** → Update hash, no further action
      - **Spec enriched, code covers it** → Update hash after verification
      - **Code ahead of spec** → Flag for `/ddd-reverse`, do NOT update hash yet
      - **New spec logic** → Flag for `/ddd-implement`, do NOT update hash yet

   **RULE: Never update a specHash unless both directions are verified.** If code has details the spec doesn't describe, updating the hash would falsely declare "in sync" and risk losing those details on future re-implementation.

4. **Update mapping.yaml** (only for verified-in-sync flows):
   - For flows that are genuinely in sync (metadata-only or spec-enriched with code coverage), compute and update the `specHash`
   - Update the `files` list with all source files that are part of the implementation
   - Update `implementedAt` timestamp only if implementation files have actually changed
   - Remove entries for flows that no longer have implementation files

5. **Detect new patterns** (if `--discover` or `--full` flag):

   This is a three-phase operation: **Analyze → Approve → Apply**

   **Phase 1 — Analyze (read-only):**
   - Scan `src/` for route handlers, services, models that don't have corresponding flow specs
   - Compare code against specs to find implementation details not captured in YAML
   - Check `architecture.yaml` for cross-cutting patterns that code uses but aren't documented
   - Generate a complete gap report

   **Phase 2 — Approve (human gate):**
   - Present the gap report to the user with proposed changes:
     - New flow specs to create
     - Existing specs to enrich
     - Architecture.yaml updates for cross-cutting patterns
   - User approves, rejects, or modifies each proposed change individually
   - Do NOT proceed to Phase 3 without explicit user approval

   **Phase 3 — Apply (write, only approved changes):**
   - Create new flow YAML specs under `specs/domains/` (approved only)
   - Update existing specs with approved enrichments
   - Update architecture.yaml with approved cross-cutting patterns
   - Update mapping.yaml hashes for all changes

6. **Fix drift** (if `--fix-drift` or `--full` flag):

   **IMPORTANT:** `--fix-drift` does NOT blindly re-implement. It follows this decision tree:

   For each drifted flow:
   - **Metadata-only drift** → Update hash (no code change)
   - **Code ahead of spec** → Run `/ddd-reverse` logic to enrich the spec first, then update hash
   - **New spec logic** → Re-implement:
     - Read the updated flow spec
     - Read the existing implementation files from mapping
     - Update the implementation to match the new spec while PRESERVING existing implementation patterns (stealth HTTP, encryption, error handling, etc.)
     - Run tests and fix until passing
     - Update mapping.yaml with new specHash and timestamp

7. **Report**:
   - Show a summary of what was synced:
     - Flows verified in sync (hash updated)
     - Flows with code ahead of spec (needs `/ddd-reverse`)
     - Flows with new spec logic (needs `/ddd-implement`)
     - Flows with missing implementation
     - (If --discover) Untracked code and proposed specs
     - (If --fix-drift) Flows that were re-implemented or enriched

## Usage

The user will say something like:
- `/ddd-sync` — bidirectional sync analysis, update hashes for verified flows only
- `/ddd-sync --discover` — also discover untracked code and propose new specs (analyze-approve-apply)
- `/ddd-sync --fix-drift` — resolve all drift using the decision tree (metadata→hash, code-ahead→reverse, new-logic→implement)
- `/ddd-sync --full` — do all of the above: sync, discover, and fix drift

$ARGUMENTS
