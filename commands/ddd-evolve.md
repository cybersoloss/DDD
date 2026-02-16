# DDD Evolve

Analyze DDD shortfall reports from one or more projects, critically evaluate each gap, separate real framework limitations from vague improvements, and produce a prioritized recommendation plan for human decision-making. This command does NOT automatically apply changes — it advises.

## Usage

```
/ddd-evolve <path-to-shortfalls.yaml> [<path2> <path3> ...]
/ddd-evolve --dir <project-dir> [--dir <project-dir2> ...]
/ddd-evolve --review <path-to-evolution-plan.yaml>
/ddd-evolve --apply <path-to-evolution-plan.yaml>
```

## Modes

| Mode | What it does |
|------|-------------|
| *(default)* | Analyze shortfalls → produce `ddd-evolution-plan.yaml` with recommendations |
| `--review` | Walk human through each item interactively, collect approve/defer/reject decisions |
| `--apply` | Execute approved items from a reviewed plan |

**Options:**

| Flag | Purpose |
|------|---------|
| `--dir <path>` | Point to a DDD project directory. Auto-discovers `specs/shortfalls.yaml` inside it. Verifies `ddd-project.json` exists to confirm it's a valid DDD project. Can be specified multiple times. Can be mixed with direct file paths. |

**Flow:** analyze → `--review` → `--apply` (each step requires the previous)

## Instructions

### Default mode: Analyze shortfalls

1. **Fetch the DDD Usage Guide**: Run `gh api repos/mhcandan/DDD/contents/DDD-USAGE-GUIDE.md --jq '.content' | base64 -d` to get the latest version. You need this to evaluate whether shortfalls are genuine gaps or already addressable within the current spec.

2. **Resolve shortfall file paths**: Collect all shortfall files from `$ARGUMENTS`:
   - **Direct paths** — use as-is (e.g., `~/code/proj-a/specs/shortfalls.yaml`)
   - **`--dir` flags** — for each directory:
     a. Verify `{dir}/ddd-project.json` exists. If not, warn: "Not a DDD project: {dir}" and skip.
     b. Check for `{dir}/specs/shortfalls.yaml`. If not found, warn: "No shortfalls.yaml in {dir} — run /ddd-create --shortfalls first" and skip.
     c. Use `{dir}/specs/shortfalls.yaml` as the shortfall file.
   - If no valid shortfall files are found after resolution, show an error and exit.

3. **Read all shortfall files**: Parse each resolved `shortfalls.yaml`. For each file, record the project name, date, and all shortfall entries.

4. **Deduplicate across projects**: Merge shortfalls that describe the same underlying gap (even if worded differently). Track:
   - `frequency` — how many projects reported this gap
   - `projects` — which projects
   - `max_severity` — highest severity reported across projects
   - `consistent_severity` — whether all projects agree on severity (high signal) or vary widely (low signal)

5. **Critically evaluate each shortfall**: This is the core of the command. For EVERY shortfall, apply these filters:

   **Filter A — Already possible?**
   Check the DDD Usage Guide. Can this already be done with existing nodes, fields, `custom_fields`, or connection patterns? If yes, classify as `already_expressible` — the shortfall is a documentation/awareness problem, not a framework gap.

   **Filter B — Frequency test**
   - Appeared in 1 project → `single_occurrence` (weak signal, could be project-specific)
   - Appeared in 2+ projects → `recurring` (stronger signal)
   - Appeared in 3+ projects → `systemic` (very strong signal, likely a real gap)

   **Filter C — Specificity test**
   Is the shortfall specific enough to act on?
   - "Need better error handling" → VAGUE (reject or request clarification)
   - "service_call node needs a `retry_config` field with max_retries, backoff_ms, and retry_on status codes" → SPECIFIC (actionable)
   Rate each as `specific`, `moderate`, or `vague`.

   **Filter D — Scope test**
   Would addressing this shortfall:
   - Break existing specs? → `breaking` (high cost)
   - Require new node type? → `additive_major` (medium cost)
   - Require new field on existing type? → `additive_minor` (low cost)
   - Require only documentation update? → `docs_only` (minimal cost)

   **Filter E — Workaround quality test**
   How bad is the current workaround?
   - No workaround exists (blocked) → `critical_gap`
   - Workaround exists but loses information (e.g., process node with free-text) → `lossy_workaround`
   - Workaround exists and preserves intent (e.g., custom_fields) → `adequate_workaround`
   - Workaround is trivial → `non_issue`

   **Filter F — Design intent test**
   Is the gap intentional? Some limitations exist by design:
   - DDD is deliberately opinionated about node types (not everything needs a node)
   - L1/L2 are intentionally high-level (not everything should be visible)
   - `process` nodes are intentionally flexible (sometimes free-text IS the right answer)
   Rate as `likely_intentional` or `likely_unintentional`.

6. **Classify each shortfall** into one of these verdicts:

   | Verdict | Meaning | Criteria |
   |---------|---------|----------|
   | `REAL_GAP` | Genuine framework limitation worth fixing | Recurring + specific + lossy/no workaround + not intentional |
   | `ENHANCEMENT` | Valid improvement but not blocking anyone | Specific + adequate workaround exists |
   | `VAGUE` | Too unspecific to act on — needs refinement | Failed specificity test |
   | `ALREADY_POSSIBLE` | Can be done today, just not obvious | Passed Filter A |
   | `BY_DESIGN` | Limitation is intentional | Failed Filter F |
   | `PROJECT_SPECIFIC` | Only relevant to one unusual project | Single occurrence + not generalizable |

7. **Produce the evolution plan**: Write `ddd-evolution-plan.yaml` to the current directory. Structure:

   ```yaml
   # DDD Evolution Plan
   # Generated by: /ddd-evolve
   # Date: {ISO timestamp}
   # Input: {list of shortfall files analyzed}
   # Status: pending_review

   meta:
     projects_analyzed: {count}
     total_shortfalls_reviewed: {count}
     status: pending_review
     verdicts:
       REAL_GAP: {count}
       ENHANCEMENT: {count}
       VAGUE: {count}
       ALREADY_POSSIBLE: {count}
       BY_DESIGN: {count}
       PROJECT_SPECIFIC: {count}

   # ──────────────────────────────────────────────
   # TIER 1: Real gaps — recommended for action
   # These are genuine DDD framework limitations
   # ──────────────────────────────────────────────
   real_gaps:
     - id: "RG-001"
       shortfall: "{concise description}"
       verdict: REAL_GAP
       evidence:
         frequency: {n}
         projects: ["{proj1}", "{proj2}"]
         max_severity: critical|high|medium
         specificity: specific
         workaround_quality: critical_gap|lossy_workaround
       analysis: |
         {2-3 sentences explaining WHY this is a real gap, what design intent
         it blocks, and why the current workaround is insufficient}
       recommendation:
         action: ADD_NODE_TYPE|ADD_FIELD|ADD_LAYER_ELEMENT|ADD_CONNECTION_FEATURE|UPDATE_SPEC
         scope: "{what changes — e.g., 'Add retry_config field to service_call node'}"
         affects:
           spec: true|false        # DDD-USAGE-GUIDE.md
           commands: true|false     # ddd-create.md, ddd-implement.md, etc.
           tool: true|false         # DDD Tool components
           validator: true|false    # flow-validator.ts
         effort: small|medium|large
         risk: low|medium|high
         breaking: true|false
       decision: null  # ← Set during --review

   # ──────────────────────────────────────────────
   # TIER 2: Enhancements — nice to have
   # Valid improvements with adequate workarounds
   # ──────────────────────────────────────────────
   enhancements:
     - id: "EN-001"
       shortfall: "{concise description}"
       verdict: ENHANCEMENT
       evidence:
         frequency: {n}
         projects: ["{proj1}"]
         max_severity: medium|low
         specificity: specific|moderate
         workaround_quality: adequate_workaround
       analysis: |
         {Why this is an enhancement, not a gap. What workaround exists
         and why it's acceptable for now.}
       recommendation:
         action: "{proposed change}"
         effort: small|medium
       decision: null  # ← Set during --review

   # ──────────────────────────────────────────────
   # TIER 3: Not actionable
   # Shortfalls that don't warrant changes
   # ──────────────────────────────────────────────
   not_actionable:
     - id: "NA-001"
       shortfall: "{concise description}"
       verdict: VAGUE|ALREADY_POSSIBLE|BY_DESIGN|PROJECT_SPECIFIC
       reason: |
         {Why this doesn't warrant a change. If ALREADY_POSSIBLE, explain
         how to do it with current DDD. If VAGUE, explain what additional
         specificity is needed.}
       decision: null  # ← Set during --review (override allowed)

   # ──────────────────────────────────────────────
   # Execution order (for approved items only)
   # ──────────────────────────────────────────────
   recommended_order:
     - phase: 1
       ids: ["RG-001", "RG-003"]
       rationale: "Low effort, high impact, no dependencies"
     - phase: 2
       ids: ["RG-002", "EN-001"]
       rationale: "Depends on phase 1 types being added first"
   ```

8. **Present the plan to the human**: After writing the file, show a summary table:

   ```
   DDD Evolution Plan — {date}

   Analyzed {n} shortfalls from {n} projects

   TIER 1 — Real Gaps (recommended):
     RG-001  [critical]  Add retry_config to service_call        (small effort, affects: spec + tool)
     RG-002  [high]      New rate_limiter node type               (medium effort, affects: spec + commands + tool)
     RG-003  [high]      Connection priority/ordering             (small effort, affects: spec + tool)

   TIER 2 — Enhancements (nice to have):
     EN-001  [medium]    Webhook signature field on trigger       (small effort)

   TIER 3 — Not Actionable:
     NA-001  [VAGUE]       "Better error handling" — too unspecific
     NA-002  [BY_DESIGN]   Process node flexibility is intentional
     NA-003  [ALREADY_OK]  Can use custom_fields for this today

   Recommended execution: Phase 1 (RG-001, RG-003) → Phase 2 (RG-002, EN-001)

   Next step:
     Run /ddd-evolve --review ddd-evolution-plan.yaml to review and decide on each item
   ```

---

### `--review` mode

When `$ARGUMENTS` contains `--review` and a path to an evolution plan:

1. **Read the evolution plan**: Parse the YAML file. Verify `meta.status` is `pending_review` or `reviewed` (allow re-review).

2. **Walk through each item interactively, tier by tier**:

   For each **TIER 1 (Real Gaps)** item, present:
   ```
   ── RG-001 ─────────────────────────────────────────────
   Shortfall:  Add retry_config to service_call
   Severity:   critical  |  Frequency: 3 projects
   Verdict:    REAL_GAP

   Analysis:
     service_call nodes have no way to configure retry behavior.
     Users must implement retries manually in code, which defeats
     the purpose of spec-driven development.

   Recommendation:
     Action: ADD_FIELD
     Scope:  Add retry_config field (max_retries, backoff_ms, retry_on)
     Effort: small  |  Risk: low  |  Breaking: no
     Affects: spec, commands, tool, validator
   ───────────────────────────────────────────────────────
   ```
   Then ask the human to decide using AskUserQuestion:
   - **Approve** — include in --apply execution
   - **Defer** — valid but not now, revisit later
   - **Reject** — disagree with the analysis
   - **Other** — human provides a custom note/modification

   If the human provides a note (via Other or alongside a decision), record it in `decision_notes` on the item.

   For each **TIER 2 (Enhancements)** item, present the same format but shorter analysis.

   For **TIER 3 (Not Actionable)** items, present them as a batch summary:
   ```
   ── Not Actionable (3 items) ───────────────────────────
   NA-001  [VAGUE]          "Better error handling" — too unspecific
   NA-002  [BY_DESIGN]      Process node flexibility is intentional
   NA-003  [ALREADY_POSSIBLE] Can use custom_fields for webhook signatures
   ───────────────────────────────────────────────────────
   ```
   Then ask: "Any of these you want to override? Select items to promote, or skip."
   - If the human selects items to override, ask what verdict/decision they want
   - If skip, mark all as `decision: accept_verdict` (agree with the classification)

3. **Show review summary**:
   ```
   Review Complete — {date}

   Decisions:
     Approved:  RG-001, RG-003, EN-001  (3 items)
     Deferred:  RG-002                   (1 item)
     Rejected:  —                        (0 items)
     Accepted:  NA-001, NA-002, NA-003   (3 not-actionable, verdicts accepted)

   Approved execution plan:
     Phase 1: RG-001 (Add retry_config), RG-003 (Connection priority)
     Phase 2: EN-001 (Webhook signature field)

   Next step:
     Run /ddd-evolve --apply ddd-evolution-plan.yaml to execute approved items
   ```

4. **Update the evolution plan file**:
   - Set `decision: approve|defer|reject|accept_verdict` on each item
   - Add `decision_notes` where the human provided comments
   - Add `reviewed_at: {ISO timestamp}`
   - Set `meta.status: reviewed`
   - Write the updated YAML back to the same file path

---

### `--apply` mode

When `$ARGUMENTS` contains `--apply` and a path to an evolution plan:

1. **Read the evolution plan**: Parse the YAML file. Verify `meta.status` is `reviewed`. If status is `pending_review`, refuse and tell the user to run `--review` first.

2. **Collect approved items**: Find all items where `decision: approve`. If none, inform the user and exit.

3. **Show the human what will change** before doing anything:
   ```
   Approved items to apply:

   RG-001: Add retry_config to service_call
     - DDD-USAGE-GUIDE.md: Add retry_config field definition under service_call node
     - ddd-create.md: Update service_call instructions to include retry_config
     - DDD Tool: Add retry_config editor to ServiceCallEditor component
     - Validator: Add checkServiceCallRetryConfig rule

   RG-003: Connection priority/ordering
     - DDD-USAGE-GUIDE.md: Add priority field to connection spec
     - ddd-create.md: Update connection wiring instructions

   Proceed? (The human must confirm)
   ```

4. After confirmation, execute changes in dependency order (phase 1 first, then phase 2).

5. For each approved item, make the actual changes:
   - **spec changes** → edit `~/code/DDD/DDD-USAGE-GUIDE.md`
   - **command changes** → edit `~/.claude/commands/ddd-create.md` (and other affected commands)
   - **tool changes** → edit files in `~/code/ddd-tool/src/`
   - **validator changes** → edit `~/code/ddd-tool/src/utils/flow-validator.ts`

6. After all changes:
   - Update `meta.status: applied` and `applied_at: {ISO timestamp}` in the plan file
   - Show what was modified and suggest committing

$ARGUMENTS
