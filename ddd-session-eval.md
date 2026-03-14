# DDD Session Evaluation

**Do not change any code, specs, commands, or files. This is a read-only analysis task.**

Reflect on your usage of DDD commands, the DDD Usage Guide, and the claude-commands ecosystem in THIS session. Produce a structured report using real examples — not hypothetical ones.

**Accuracy rules:**
- Only report what actually happened this session. Do not invent findings to fill sections.
- If a section has no relevant observations, write "N/A — not applicable this session." Empty sections are better than padded ones.
- Token counts and overhead percentages are estimates — label them as such (use "~" prefix). Do not fabricate precision.
- For the non-DDD command table: only fill rows where you genuinely used the command or genuinely wished you had it during a specific moment. Leave other rows blank.
- Speculative questions (marked with "↳") are clearly opinion — answer honestly, including "I don't know" if you don't have a basis for the answer.

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

Also check for prior eval reports (`docs/ddd-session-eval-*.md`) and shortfalls files (`docs/ddd-session-shortfalls-*.yaml`) in the project. Note their finding IDs for cross-referencing in Section 12.

## Finding IDs

Every finding in Sections 10a–10d and Section 11 must have a unique ID. Format: `{project-short}-{YYYYMMDD}-{seq}` where `{project-short}` is a short project identifier (e.g., `cc` for content-curator, `ddt` for ddd-tool) and `{seq}` is a zero-padded sequence number starting at `001`.

Examples: `cc-20260313-001`, `cc-20260313-002`, `ddt-20260315-001`

Use these IDs in:
- The markdown report (prefix each finding row/bullet with its ID)
- The YAML shortfalls (add an `eval_id` field to each entry that qualifies)
- Section 12 (reference prior findings by ID)

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

**Methodology efficiency:**
- When was reading specs redundant or wasteful? (spec added nothing the code didn't already tell you)
- When were specs wrong or stale, and code was the actual truth?
- When did spec-driven workflow add overhead without improving the outcome?
- ↳ If you redid this session without DDD at all, what would you lose? What would be easier?

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
- **Estimated tokens wasted on unnecessary fetches:** (Usage Guide is ~5,200 lines / ~18,000 tokens per full fetch)
- **If you fetched the guide at least once but skipped it for later commands, why?** (e.g., internalized the relevant sections, node types were familiar, guide was too large for the value it provided, fetched section wasn't relevant to the task)
- **If you never fetched the guide this session, why not?** (e.g., all node types familiar, no command required it, consciously skipped to save tokens)

## 5. Command Prompt Utilization

For each DDD command executed, estimate how much of the command prompt was useful vs overhead:

| Command | Prompt Size (est. lines) | Sections Referenced | Sections Ignored | Est. Overhead % | What Was Overhead |
|---------|--------------------------|---------------------|------------------|-----------------|-------------------|
| _example: `/ddd-update` single flow_ | _~320_ | _Scope resolution, change-history format_ | _Infra, schema, UI integrity, add-domain_ | _~85%_ | _All non-Logic pillar instructions, Usage Guide fetch instructions, add-domain/add-page scope tables_ |
| _example: `/ddd-update` question_ | _~320_ | _None_ | _Entire prompt_ | _~98%_ | _User asked "why is X happening?" — entire spec-editing pipeline loaded for a question that needed zero command structure_ |

Some sections influence behavior implicitly — note those in the "Sections Referenced" column even if you didn't actively consult them.

**Session total:** Estimated tokens of command prompt overhead across all invocations: ___

**↳ What patterns do you see in the overhead?** Look across all rows. Is the overhead concentrated in specific commands? Specific types of invocations (e.g., questions vs changes)? Specific unused pillar instructions? What structural change to commands or the Usage Guide would eliminate the most waste?

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
- ↳ Was there a task that NO DDD command could help with? What would that command look like?
- Did the user invoke a DDD command for something it wasn't designed for? What was the user's actual intent vs the command's purpose?

**Human-AI collaboration through DDD:**

How well did DDD serve as a collaboration layer between you and the user this session? Cite specific moments.

- Did the user struggle to pick the right DDD command for their intent? What were they trying to do and what did they reach for?
- Did the user make a design decision during conversation (verbally or by approving a fix) that should have been captured in specs but wasn't? What fell through the cracks?
- Did DDD's pipeline (update spec → implement → test) add ceremony the user didn't need, or did it give useful structure they wouldn't have had otherwise?
- Were there moments where you and the user were aligned on what to do, but DDD's structure got in the way? Or moments where DDD helped you stay aligned?
- ↳ What changes to DDD commands, Usage Guide, or workflow would improve how the user communicates intent and reviews your work? (new commands, new modes, guide sections, scope changes)

## 8. Node Type Usage

- **Used this session:**
- **Needed reference for:**
- **Knew from prior context:**
- **Never used (of 30 available):**
- **Used `process` node as a workaround** for something a structured node type should handle? (If yes, describe what the process node does and what node type you wish existed — this feeds into the shortfalls YAML)

## 9. Cross-Command Observations

### 9a. Duplication Analysis

List specific content you **actually noticed** repeated across commands you executed this session. Only report duplication you encountered firsthand — do not read other command prompts just to find duplication:

| Duplicated Content | Where It Appears | Times Loaded This Session | Could Be Shared? |
|--------------------|------------------|---------------------------|-------------------|
| _example: "Read project context" file list_ | _`/ddd-update`, `/ddd-implement`, `/ddd-test`_ | _3_ | _Yes — identical across all commands_ |
| _example: Cross-cutting patterns explanation_ | _`/ddd-update`, `/ddd-implement`_ | _2_ | _Yes — same ~30 lines_ |

**↳ What would you extract or deduplicate?** If you could restructure the commands and Usage Guide to eliminate the repetition you observed, what would you change? (e.g., shared preamble file, reference-instead-of-reproduce, conditional loading)

### 9b. Non-DDD Command Integration

For each command below, note whether you used it or genuinely wished you had at a specific moment this session. Leave rows blank if you have no real observation — do not fill rows speculatively:

| Non-DDD Command | Used? | Relationship to DDD | Observation |
|-----------------|-------|---------------------|-------------|
| `/code-review` | | Could verify code matches spec | |
| `/security-scan` | | Overlaps with architecture.yaml security patterns | |
| `/test-coverage` | | Overlaps with `/ddd-test` | |
| `/spec-verify` | | Could use mapping.yaml for verification | |
| `/perf-review` | | Independent | |
| `/pre-deploy` | | Could include `/ddd-status` drift check | |
| `/simplify` | | Could check if simplifications break spec compliance | |
| `/trust-verify` | | Could use spec-to-code mapping as trust evidence | |
| _(other)_ | | | |

**↳ Integration architecture questions** (only answer if grounded in observations above):
- Which non-DDD commands should be DDD-aware (read specs, mapping, or architecture.yaml)?
- Which DDD commands could delegate work to non-DDD commands instead of reimplementing?
- Are there workflow combinations that should be a single command?

## 10. DDD Framework Feedback

Report problems, gaps, and inconsistencies you encountered in the DDD commands and Usage Guide during this session. Be specific — cite the command name, step number, or Usage Guide section.

### 10a. Command Problems

Issues encountered while executing DDD slash commands:

| ID | Command | Step/Section | Problem Type | Description |
|----|---------|-------------|--------------|-------------|
| _cc-20260313-001_ | _`/ddd-implement`_ | _Step 11 (cross-cutting)_ | _Unclear guidance_ | _Says "apply matching patterns" but doesn't say what to do when two patterns conflict_ |
| _cc-20260313-002_ | _`/ddd-sync`_ | _Step 6 (drift analysis)_ | _Missing instruction_ | _No guidance for handling a flow that exists in mapping.yaml but spec file was deleted_ |

**Problem types:** unclear guidance, missing instruction, wrong instruction, ambiguous step, silent failure, missing error handling, missing scope support

### 10b. Usage Guide Problems

Issues with the DDD Usage Guide content:

| ID | Section | Problem Type | Description |
|----|---------|--------------|-------------|
| _cc-20260313-003_ | _Section 6, `smart_router` node_ | _Incomplete spec_ | _Lists fields but no example YAML — unclear how to set `routes`_ |
| _cc-20260313-004_ | _Section 8, connection patterns_ | _Missing pattern_ | _No example for connecting `parallel` node outputs to a `collection` node_ |

**Problem types:** incomplete spec, wrong information, missing example, unclear explanation, outdated content, missing section

### 10c. Inconsistencies

Conflicting or mismatched content between commands, or between commands and the Usage Guide:

| ID | Location A | Location B | Inconsistency |
|----|------------|------------|---------------|
| _cc-20260313-005_ | _`/ddd-implement` step 5_ | _`/ddd-update` step 6_ | _Different node ID format conventions (one says 8-char, other says 6-char)_ |
| _cc-20260313-006_ | _`/ddd-sync` next steps_ | _Usage Guide Section 12.1_ | _Different remediation order for `diverged` findings_ |

### 10d. Missing Capabilities

Things you needed to express or do but DDD didn't support:

- _cc-20260313-007: "Needed to spec a WebSocket reconnection strategy but no node type or spec field supports it"_
- _cc-20260313-008: "Wanted to mark a flow as deprecated in the spec but there's no `status` or `lifecycle` field"_
- _cc-20260313-009: "Command doesn't support `--dry-run` — would have been useful to preview changes"_

## 11. Recommendations

### 11a. Tactical Fixes

Specific fixes for issues found in Sections 10a–10d:

| ID | What to Change | Category | Evidence From This Session | Estimated Impact | Priority |
|----|----------------|----------|---------------------------|------------------|----------|
| _cc-20260313-010_ | | _quality / efficiency / workflow / integration / safety_ | | _e.g., "~5,000 tokens saved per invocation" or "prevents class of bugs"_ | _high / medium / low_ |

### 11b. Systemic Improvements

Step back from individual findings. Based on the overhead data (Section 5), duplication analysis (Section 9a), workflow friction (Section 7), and integration gaps (Section 9b), what **structural changes** to the DDD framework would have the highest impact? Only propose changes grounded in your actual observations — do not generate generic optimization ideas.

Think about:
- How should the Usage Guide be structured for AI consumption? (monolithic file vs split sections, navigation aids, caching strategies)
- How should command prompts be organized to reduce overhead? (shared preambles, conditional loading, fast paths for simple operations)
- What recurring patterns across commands should be extracted or deduplicated?
- What new commands or command modes would eliminate friction you observed?
- How should DDD commands integrate with non-DDD commands in the ecosystem?

For each proposal:

| ID | Structural Change | What It Eliminates | Estimated Token/Workflow Impact |
|----|-------------------|--------------------|-------------------------------|
| _cc-20260313-015_ | _e.g., Split Usage Guide into fetchable sections by topic_ | _~15,000 tokens of unnecessary content per fetch_ | _60-80% reduction per Usage Guide reference_ |
| _cc-20260313-016_ | _e.g., Add fast-path mode to `/ddd-update` for single-flow changes_ | _Pillar instructions, cross-domain analysis, scope tables for unused scopes_ | _~1,500 tokens saved for simple changes_ |

## 12. Prior Reports

If prior eval reports exist in the project's `docs/` directory (`ddd-session-eval-*.md` or `ddd-session-shortfalls-*.yaml`), scan their finding IDs. Only report on prior findings whose scenario you **actually re-encountered** this session:

- **Confirmed:** You hit the same issue again this session (reference by ID, e.g., "Confirms `cc-20260310-003` — same inconsistency still present")
- **Fixed:** You encountered the same scenario but the problem was gone (e.g., "`cc-20260310-001` — I used `/ddd-update` for a question and it correctly detected non-spec intent")
- **Patterns:** Recurring themes you observe across reports you've read

Do NOT guess whether prior findings are fixed if you didn't encounter their scenario this session. Silence is honest; guessing is not. Do NOT copy prior findings into this report. Reference by ID only.

If no prior reports exist, skip this section.

## 13. DDD Usage Statistics

Provide project-level usage context that makes findings credible across reports:

- **Change-history entries by pillar:** Logic: ___ | Data: ___ | Interface: ___ | Infrastructure: ___
- **Most common DDD operations this session:** (e.g., `/ddd-update` single flow 45%, `/ddd-implement` pending 25%)
- **Node types actively used in project:** ___ of 30
- **Node types used this session:** ___ of 30
- **Usage Guide fetches this session vs total command invocations:** ___/___ (fetch rate: __%)

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

Add an `eval_id` field to each shortfall entry so it can be traced back to the markdown report:
```yaml
missing_node_types:
  - eval_id: "cc-20260313-007"  # links to markdown finding
    name: "reconnection-strategy"
    severity: medium
    # ... rest of standard shortfalls.yaml fields
```

**Comprehensive YAML requirement:** If you produce a YAML, it must include ALL standard sections — not just findings from this session. Do a lightweight project-level audit:

1. **`pillar_balance`** — count the project's flows, schemas, UI pages, and infrastructure services by scanning `specs/`. Include `pages_without_specs` and `imbalance_warnings`. This is fast (directory listing + file count).
2. **`summary.feature_coverage`** — compute the full Feature Usage Matrix (node_types, trigger_types, collection_operations, crypto_operations, parse_formats, data_store_types, connection_behaviors, ui_component_types, form_field_types, schema_index_types, etc.) by scanning existing flow specs. Report used/available ratios. Include `unused_but_applicable` for features the project could benefit from.
3. **`ui_shortfalls`** — if UI specs exist (`specs/ui/`), scan them for missing component types, inadequate components, form limitations, and interaction gaps — even if UI wasn't touched this session. Use empty arrays for sub-sections with no findings.
4. **`layer_gaps`** — populate `elements_used` for all layers where specs exist. For `missing_elements` and `invisible_information`, only report issues you can identify from the specs.
5. **Empty sections** — include all sections with `[]` when no issues found. An explicit empty array signals "audited, no gaps" vs an omitted section which signals "not checked."

This ensures every session-eval YAML is complete enough for `/ddd-evolve` to produce meaningful analysis, even when the session itself was narrow in scope.

### After writing

Tell the user the file path(s) and summarize:
- How many findings went into the markdown report (tactical fixes in 11a + systemic proposals in 11b)
- How many (if any) qualified as shortfalls for the YAML output
- Highlight the top systemic proposal from 11b if one stands out
- Suggested next step: if YAML was produced, suggest `/ddd-evolve docs/ddd-session-shortfalls-{date}.yaml`
