# DDD Status

Show a quick read-only overview of the DDD project's implementation state. No files are modified — this is purely informational.

## Instructions

1. **Find the DDD project**: Look for `ddd-project.json` in the current directory or parent directories.

2. **Read project structure**:
   - `ddd-project.json` — domain list
   - For each domain: `specs/domains/{domain}/domain.yaml` — flow list
   - `.ddd/mapping.yaml` — implementation tracking (if exists)

3. **For each flow, determine status**:

   | Status | Condition |
   |--------|-----------|
   | **Not implemented** | No entry in mapping.yaml |
   | **Up to date** | Entry exists, specHash matches current flow YAML hash |
   | **Drifted** | Entry exists, specHash does NOT match (spec changed since implementation) |
   | **Stale** | Entry exists, but one or more implementation files are missing |
   | **Scaffolded** | Project has package.json/tsconfig but no flow implementations |

   To compute specHash: read the flow YAML file content and compute SHA-256.

4. **For drifted flows, classify the drift** (CRITICAL — do NOT skip this step):

   When a flow shows as "Drifted", you MUST analyze what actually changed before recommending any action:

   a. **Read the current spec YAML** and compare against the mapping's stored specHash context
   b. **Read the existing implementation code** listed in the mapping's `files` array
   c. **Classify the drift into one of these categories:**

   | Drift Type | How to Detect | Recommended Action |
   |------------|---------------|-------------------|
   | **Metadata-only** | Only `metadata.modified`, `metadata.updated`, `position` fields, or formatting changed. No nodes, connections, or spec logic changed. | `/ddd-sync` — safe to update hash only |
   | **Spec enriched, code already covers it** | Spec added detail (e.g., new field description) but the implementation code already handles it. | `/ddd-sync` — verify and update hash |
   | **Code has details spec doesn't** | Implementation has patterns (error handling, caching, stealth HTTP, encryption) that the spec doesn't describe. | `/ddd-reverse {domain/flow}` first to enrich specs, then `/ddd-sync` |
   | **Spec has new logic code doesn't** | New nodes, connections, tools, or business logic were added to the spec that the code does not implement. | `/ddd-implement {domain/flow}` — only case where re-implementation is appropriate |

   **WARNING:** Never recommend `/ddd-implement` without first confirming the drift is type 4 (new logic). Re-implementing a flow overwrites existing code, which can destroy working implementation details that the spec doesn't capture.

5. **Check scaffold state**:
   - Does `package.json` (or equivalent) exist?
   - Does the main entry point exist (e.g., `src/app.ts`)?
   - Does the database schema exist (e.g., `prisma/schema.prisma`)?
   - Does `.ddd/mapping.yaml` exist?

6. **Display the status report**:

   ```
   DDD Project: {project-name}
   Scaffold: {Yes / No / Partial}

   Domain          Flow                    Status          Implemented
   ─────────────── ─────────────────────── ─────────────── ──────────────
   users           user-register           Up to date      2025-12-15
   users           user-login              Drifted (meta)  2025-12-14
   users           reset-password          Not implemented —
   orders          create-order            Up to date      2025-12-15
   orders          process-payment         Stale           2025-12-13
   notifications   send-email              Not implemented —

   Summary:
     2 up to date
     1 drifted — metadata only (run /ddd-sync)
     1 stale (missing files — run /ddd-implement orders/process-payment)
     2 not implemented (run /ddd-scaffold then /ddd-implement)

   Event wiring:
     UserRegistered: users → notifications (consumer not implemented)
     OrderCreated: orders → notifications (consumer not implemented)
   ```

   **For drifted flows, always show the drift type in parentheses:**
   - `Drifted (metadata)` — only timestamps/positions changed
   - `Drifted (spec enriched)` — spec added detail, code already covers it
   - `Drifted (code ahead)` — code has details spec doesn't describe
   - `Drifted (new logic)` — spec has new logic code doesn't implement

7. **If `$ARGUMENTS` includes `--json`**, output the status as a JSON object instead of the table format. This is useful for scripting.

8. **Suggest next actions** based on what's found — using the SAFE recommendation rules:
   - If no scaffold: "Run `/ddd-scaffold` to set up the project"
   - If not-implemented flows exist: "Run `/ddd-implement {scope}` to generate code"
   - If drifted (metadata or spec enriched): "Run `/ddd-sync` to update hashes"
   - If drifted (code ahead): "Run `/ddd-reverse {domain/flow}` to capture implementation details into specs, then `/ddd-sync`"
   - If drifted (new logic): "Run `/ddd-implement {domain/flow}` to update code — WARNING: this will regenerate code, review the spec diff first"
   - If stale flows exist: "Run `/ddd-implement {domain/flow}` to regenerate missing files"
   - If everything is up to date: "All flows are implemented and in sync"

   **NEVER suggest `/ddd-implement` as the default action for drifted flows.** Always classify the drift first and recommend the least destructive action.

$ARGUMENTS
