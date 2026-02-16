# DDD Promote

Move approved annotations into permanent specs. This is how implementation wisdom — captured by `/ddd-reflect` — becomes part of the design. Promoted patterns are written to `architecture.yaml`, flow specs, or shared spec files.

## Scope Resolution

Parse `$ARGUMENTS` to determine scope:

| Argument | Scope | Example |
|----------|-------|---------|
| `--all` | Promote all approved annotations | `/ddd-promote --all` |
| `--review` | Interactive review — present each candidate for approval | `/ddd-promote --review` |
| `{domain}/{flow}` | Scope to specific flow's annotations | `/ddd-promote monitoring/check-social-sources` |
| *(empty)* | Interactive — same as `--review` | `/ddd-promote` |

## Instructions

1. **Find the DDD project**: Look for `ddd-project.json` in the current directory or parent directories.

2. **Read project context**:
   - `ddd-project.json` — domain list
   - `.ddd/mapping.yaml` — implementation tracking
   - `specs/architecture.yaml` — current `cross_cutting_patterns` section
   - `specs/shared/errors.yaml` — current error codes
   - `specs/shared/types.yaml` — current shared types (if exists)
   - `.ddd/annotations/` — all annotation files

3. **Load all annotations**: Read every `.yaml` file in `.ddd/annotations/` (recursively by domain subdirectories). Group annotations by status:
   - `candidate` — awaiting review
   - `approved` — reviewed and approved, ready to promote
   - `promoted` — already promoted into specs
   - `dismissed` — reviewed and rejected

4. **Determine mode from arguments**:

   **If `--review` or no argument**: Interactive review mode.
   - Present each `candidate` annotation to the user with:
     - Pattern type and description
     - Which nodes it applies to
     - Code evidence (file, lines, snippet)
     - What the current spec says vs what the code does
   - Ask the user to **approve** or **dismiss** each candidate
   - For approved candidates, ask the user to confirm the promotion target:
     - **Cross-cutting pattern** → will be added to `architecture.yaml` `cross_cutting_patterns`
     - **Flow-specific detail** → will be added to the flow spec YAML (node spec enrichment, observability, or security section)
     - **Shared type/error** → will be added to `shared/types.yaml` or `shared/errors.yaml`
   - Update annotation status to `approved` or `dismissed`
   - After review, proceed to promotion of approved items

   **If `--all`**: Promote all `approved` annotations without interactive review.
   - Skip `candidate` annotations (they need review first)
   - Process only annotations with status `approved`

   **If `{domain}/{flow}`**: Scope to that flow's annotations.
   - Read only `.ddd/annotations/{domain}/{flow}.yaml`
   - If `--review` is also present (or no other flag), enter interactive review for that flow
   - If `--all` is also present, promote all approved annotations for that flow

5. **Promote approved annotations**: For each annotation with status `approved`:

   **Cross-cutting patterns** (patterns that appear across multiple flows):

   Check if the pattern type already exists in `architecture.yaml` → `cross_cutting_patterns`:
   - If YES: Update the existing pattern — add new `used_by_domains` entries, expand the `description` or `convention` if the annotation adds new details
   - If NO: Add a new entry to `cross_cutting_patterns`:
     ```yaml
     cross_cutting_patterns:
       {pattern_type}:
         description: >
           {From annotation description}
         utility: {From code_evidence.file if it's a utility file}
         config: {}
         used_by_domains: [{domains that use this pattern}]
         convention: >
           {When and how to apply this pattern, derived from annotation}
     ```

   **Flow-specific details** (patterns unique to one flow):

   Read the flow spec YAML. Enrich the appropriate node(s):
   - If the detail is about security (encryption, auth, rate limiting) → add/update the `security` section on the node
   - If the detail is about observability (logging, metrics, tracing) → add/update the `observability` section on the node
   - If the detail is about implementation behavior → add detail to the node's `spec.description` or add a `spec.implementation` field
   - Preserve all existing node fields — only add, never remove

   **Shared types/errors**:

   - If the annotation describes a new error code → add to `specs/shared/errors.yaml`
   - If the annotation describes a new shared enum or value object → add to `specs/shared/types.yaml`

6. **Update annotation status**: After promoting, update each annotation's status:
   - `approved` → `promoted` (if successfully written to specs)
   - Write the updated annotation file back to `.ddd/annotations/{domain}/{flow}.yaml`

7. **Update mapping.yaml**: Since spec files have changed:
   - Recompute `specHash` for any flow specs that were modified
   - Update `annotationCount` (decrement by promoted count)
   - Update `syncState` to `synced` if the promotion brought spec in line with code

8. **Report what was promoted**:
   ```
   Promoted annotations:

   Cross-cutting patterns → architecture.yaml:
     + stealth_http: Added new pattern — rotate user agents, proxy pools for external fetches
       used_by_domains: [monitoring, discovery]
     ~ api_key_resolution: Updated — added key validation detail
       used_by_domains: [settings, monitoring, discovery, publishing]

   Flow-specific details:
     monitoring/check-social-sources:
       ~ node service_call-abc123: Added security.encryption config
       ~ node data_store-def456: Added spec.description detail about soft-delete filter

   Dismissed:
     2 annotations dismissed (not useful)

   Spec files modified:
     specs/architecture.yaml
     specs/domains/monitoring/flows/check-social-sources.yaml

   Mapping updated:
     monitoring/check-social-sources: specHash updated, annotationCount: 0

   Summary:
     Promoted: 4 patterns (2 cross-cutting, 2 flow-specific)
     Dismissed: 2
     Remaining candidates: 0
   ```

## Promotion Rules

1. **Never overwrite existing spec content**: Only add or enrich. If a node already has a `security` section, merge new fields into it — don't replace the existing section.

2. **Preserve node IDs and positions**: When editing flow specs, never change node IDs, positions, connections, or other structural elements. Only add descriptive/behavioral fields.

3. **Cross-cutting threshold**: If the same pattern type appears in annotations for 2+ flows across different domains, strongly recommend promoting it as a cross-cutting pattern in `architecture.yaml` rather than as flow-specific details.

4. **Update metadata**: When modifying a flow spec, update `metadata.modified` to the current ISO timestamp.

5. **Validate after promotion**: After writing changes, verify:
   - The flow spec YAML is still valid (proper structure, no broken references)
   - The architecture.yaml is still valid YAML
   - No duplicate pattern IDs in cross_cutting_patterns

$ARGUMENTS
