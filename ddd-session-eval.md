# DDD Session Evaluation

**Do not change any code, specs, commands, or files. This is a read-only analysis task.**

Reflect on your usage of DDD commands, the DDD Usage Guide, and the claude-commands ecosystem in THIS session. Produce a structured report using real examples — not hypothetical ones.

**This evaluation produces two outputs:**
1. `docs/ddd-session-eval-{YYYY-MM-DD}.md` — full human-readable report (all findings)
2. `docs/ddd-session-shortfalls-{YYYY-MM-DD}.yaml` — machine-readable subset for `/ddd-evolve` (only qualifying spec/framework gaps)

**Do not modify any other files.**

## Before You Begin

Fetch the `/ddd-create` command to learn the `shortfalls.yaml` format:

```bash
gh api repos/cybersoloss/claude-commands/contents/ddd-create.md --jq '.content' | base64 -d
```

Find Step 16 (`--shortfalls` section) which defines the `shortfalls.yaml` YAML structure. This is the format you must use for the YAML output. Understand the sections (`missing_node_types`, `inadequate_existing_nodes`, `missing_spec_fields`, `connection_limitations`, `layer_gaps`, `workarounds`, `cross_cutting_gaps`, `ui_shortfalls`, `pillar_balance`, `summary`) and the content rules (shortfalls are DDD framework limitations, NOT project scope decisions or spec quality issues).

---

## 1. Session Profile

- **Project:** (name and brief description)
- **Date:**
- **Session type:** (new feature / bug fix / refactor / investigation / mixed)
- **Pillars touched:** (Logic / Data / Interface / Infrastructure)
- **DDD commands invoked** (list each with count):
- **Non-DDD commands invoked** (e.g., `/code-review`, `/security-scan`):
- **Total change-history entries in the project:**

## 2. Spec-Driven Coding Impact

The core question: did reading specs before coding improve outcomes?

For each significant task this session:

| Task | Read Spec First? | First-Try Correct? | Iterations | Root Cause of Iterations | Spec Would Have Helped? |
|------|-------------------|--------------------|-----------:|--------------------------|-------------------------|
| _example: Added retry logic to publish flow_ | _Yes_ | _Yes_ | _0_ | _—_ | _N/A — spec showed all error paths_ |
| _example: Fixed validation toast_ | _No, went to code_ | _No_ | _1_ | _Missing error branch_ | _Yes — spec had the branch_ |

**Summary:**
- Tasks where reading specs improved correctness or prevented rework:
- Tasks where skipping specs had no impact:
- Tasks where specs would have helped but were skipped:
- Edge cases or error paths that specs revealed but code alone wouldn't:

## 3. Error Prevention Analysis

List bugs, rework, or missed requirements from this session. For each:

| Issue | Caught By | Could Spec Have Prevented It? | How |
|-------|-----------|-------------------------------|-----|
| _example: Missing null check on API response_ | _Runtime error_ | _Yes_ | _Flow spec shows decision node for empty response_ |
| _example: Wrong env var name_ | _Test failure_ | _No_ | _Runtime config, not in spec_ |

## 4. Usage Guide Reference Patterns

For each DDD command that references the Usage Guide:

| Command Invocation | Fetched Guide? | Sections Actually Used | Could Have Skipped Fetch? | Reason |
|--------------------|----------------|------------------------|---------------------------|--------|
| _example: `/ddd-update` adding cache node_ | _Yes_ | _Section 6 (cache node spec)_ | _No — first time using cache_ |
| _example: `/ddd-update` adding process node_ | _No_ | _None_ | _Yes — familiar node type_ |

- **Total fetch events:**
- **Necessary fetches:**
- **Unnecessary fetches:**

## 5. Command Prompt Utilization

For each DDD command executed, which sections of the command prompt did you actually reference during execution?

| Command | Sections Referenced | Sections Ignored | Notes |
|---------|---------------------|------------------|-------|
| _example: `/ddd-update` single flow_ | _Scope resolution, change-history format_ | _Infrastructure scope, schema scope, UI spec integrity_ | _Simple flow change, most pillar-specific sections unused_ |

Focus on what you observably consulted, not what percentage was "overhead" — some sections influence behavior implicitly.

## 6. Spec Freshness Observations

Did you encounter specs that were out of sync with code?

| Spec File | Drift Type | Impact | How Discovered |
|-----------|-----------|--------|----------------|
| _example: auth/login.yaml_ | _Code ahead (new field added in code, not in spec)_ | _Low — cosmetic_ | _Noticed during `/ddd-sync`_ |
| _example: schemas/user.yaml_ | _Spec ahead (field defined but never implemented)_ | _High — generated wrong test_ | _Test failure_ |

## 7. Workflow: Planned vs Actual

**DDD lifecycle suggests:**
```
e.g. /ddd-update → /ddd-implement → /ddd-test
```

**What actually happened:**
```
e.g. User asked about a bug → read code directly → fixed it →
     committed → later ran /ddd-update to sync spec with fix
```

**Friction points:**
- Where did you break out of the DDD command flow? Why?
- Where did the command pipeline feel natural?
- Where did a command load context you didn't need?

## 8. Node Type Usage

- **Used this session:**
- **Needed reference for:**
- **Knew from prior context:**
- **Never used (of 29 available):**

## 9. Cross-Command Observations

**Duplication noticed** across commands or between commands and the Usage Guide:
- (list specific repeated content you encountered)

**Non-DDD command integration opportunities** — did you use or wish you could use non-DDD commands alongside DDD?

| Non-DDD Command | Used? | Observation |
|-----------------|-------|-------------|
| _example: `/code-review`_ | _No_ | _Would have been useful to check code against spec after manual fix_ |
| _example: `/pre-deploy`_ | _Yes_ | _Didn't check spec drift — could have caught stale mapping_ |

## 10. DDD Framework Feedback

Report problems, gaps, and inconsistencies you encountered in the DDD commands and Usage Guide during this session. Be specific — cite the command name, step number, or Usage Guide section.

### 10a. Command Problems

Issues encountered while executing DDD slash commands:

| Command | Step/Section | Problem Type | Description |
|---------|-------------|--------------|-------------|
| _example: `/ddd-implement`_ | _Step 11 (cross-cutting)_ | _Unclear guidance_ | _Says "apply matching patterns" but doesn't say what to do when two patterns conflict_ |
| _example: `/ddd-sync`_ | _Step 6 (drift analysis)_ | _Missing instruction_ | _No guidance for handling a flow that exists in mapping.yaml but spec file was deleted_ |

**Problem types:** unclear guidance, missing instruction, wrong instruction, ambiguous step, silent failure, missing error handling, missing scope support

### 10b. Usage Guide Problems

Issues with the DDD Usage Guide content:

| Section | Problem Type | Description |
|---------|--------------|-------------|
| _example: Section 6, `smart_router` node_ | _Incomplete spec_ | _Lists fields but no example YAML — unclear how to set `routes`_ |
| _example: Section 8, connection patterns_ | _Missing pattern_ | _No example for connecting `parallel` node outputs to a `collection` node_ |

**Problem types:** incomplete spec, wrong information, missing example, unclear explanation, outdated content, missing section

### 10c. Inconsistencies

Conflicting or mismatched content between commands, or between commands and the Usage Guide:

| Location A | Location B | Inconsistency |
|------------|------------|---------------|
| _example: `/ddd-implement` step 5_ | _`/ddd-update` step 6_ | _Different node ID format conventions (one says 8-char, other says 6-char)_ |
| _example: `/ddd-sync` next steps_ | _Usage Guide Section 12.1_ | _Different remediation order for `diverged` findings_ |

### 10d. Missing Capabilities

Things you needed to express or do but DDD didn't support:

- _example: "Needed to spec a WebSocket reconnection strategy but no node type or spec field supports it"_
- _example: "Wanted to mark a flow as deprecated in the spec but there's no `status` or `lifecycle` field"_
- _example: "Command doesn't support `--dry-run` — would have been useful to preview changes"_

## 11. Recommendations

For each recommendation, provide:

| # | What to Change | Category | Evidence From This Session | Priority |
|---|----------------|----------|---------------------------|----------|
| | | _quality / efficiency / workflow / integration / safety_ | | _high / medium / low_ |

## 12. Prior Reports

If prior evaluation reports exist in this project's `docs/` directory (`ddd-session-eval-*.md`), briefly note:
- Which prior recommendations you can now confirm or refute based on this session
- Any patterns emerging across multiple reports

If no prior reports exist, skip this section.

---

## Output

### Output 1: Human-readable report

Write the full report (all sections above) to `docs/ddd-session-eval-{YYYY-MM-DD}.md`.

### Output 2: Machine-readable shortfalls for `/ddd-evolve`

Review your findings from Sections 8, 10b, and 10d. If ANY qualify as DDD framework gaps (not project decisions, not spec quality issues, not command/workflow problems), write them to `docs/ddd-session-shortfalls-{YYYY-MM-DD}.yaml` using the exact `shortfalls.yaml` format from `/ddd-create` Step 16 (fetched in the "Before You Begin" step).

**Mapping guide — which eval findings go into the YAML:**

| Eval Finding | shortfalls.yaml Section | Condition |
|-------------|------------------------|-----------|
| 10d: Missing node type | `missing_node_types` | Needed a node type that doesn't exist |
| 10d: Missing spec field | `missing_spec_fields` | Needed a field on a node/connection/flow that doesn't exist |
| 10d: Connection limitation | `connection_limitations` | Couldn't express an edge behavior or routing pattern |
| 10d: Missing UI capability | `ui_shortfalls.*` | Couldn't express a UI component, interaction, or form pattern |
| 10b: Inadequate node spec | `inadequate_existing_nodes` | A node type exists but lacks needed capabilities (not just missing docs) |
| 8: Used process node as workaround | `workarounds` | Used a generic node because no structured type fit |
| 8: Node type usage stats | `summary.feature_coverage` | Used vs available counts |

**What does NOT go into the YAML:**
- Command instruction problems (10a) — these are command quality issues, not spec gaps
- Inconsistencies between commands/guide (10c) — these are documentation issues
- Workflow friction (Section 7) — these are process observations
- Usage Guide documentation issues (10b, when the node spec is adequate but docs are unclear)
- Recommendations (Section 11) — these are general suggestions

**If no findings qualify for YAML, skip the YAML output entirely.** An absent file is better than an empty one.

Use the `shortfalls.yaml` header fields:
```yaml
project: "{project-name}"
generated: "{ISO timestamp}"
ddd_version: "1.0"
source: "ddd-session-eval"  # distinguishes from /ddd-create --shortfalls output
```

### After writing

Tell the user the file path(s) and summarize:
- How many findings went into the markdown report
- How many (if any) qualified as shortfalls for the YAML output
- Suggested next step: if YAML was produced, suggest `/ddd-evolve docs/ddd-session-shortfalls-{date}.yaml`
