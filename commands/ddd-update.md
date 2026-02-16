# DDD Update

Update DDD project specs (YAML files) to reflect design changes requested during development. This is the reverse of `/ddd-implement` — instead of generating code from specs, you update specs to match new requirements, then optionally re-implement.

## Scope Resolution

Parse the argument to determine what to update:

| Argument | Scope | Example |
|----------|-------|---------|
| `domain-name/flow-name` | Update a specific flow spec | `/ddd-update users/user-register` |
| `domain-name` | Update domain config and/or its flows | `/ddd-update users` |
| `--add-flow domain-name` | Add a new flow to a domain | `/ddd-update --add-flow users` |
| `--add-domain` | Add a new domain to the project | `/ddd-update --add-domain` |
| *(empty)* | Interactive — ask what to update | `/ddd-update` |

## Instructions

1. **Find the DDD project**: Look for `ddd-project.json` in the current directory or parent directories.

2. **Read the current specs**: Load the relevant YAML files to understand the current state:
   - `ddd-project.json` — project config, domain list
   - `specs/domains/{domain}/domain.yaml` — domain config, flow list, events
   - `specs/domains/{domain}/flows/{flow}.yaml` — flow spec (if updating a specific flow)
   - `specs/schemas/*.yaml` — data model definitions (reference when adding data_store nodes to use correct model names)
   - `specs/shared/errors.yaml` — error codes (reference when adding terminal error nodes to use correct error codes)
   - `specs/system.yaml` — tech stack context (reference when choosing patterns for new nodes)
   - `specs/architecture.yaml` — especially the `cross_cutting_patterns` section, which defines project-specific conventions to apply to new nodes

3. **Understand the user's request**: The user will describe what they want to change in natural language. Examples:
   - "Add a rate limiting step before the process node"
   - "Add an email notification after user registration"
   - "Split the payment flow into two: authorize and capture"
   - "Add a new search-products flow to the products domain"
   - "Add a notifications domain with an email flow"
   - "Change the login flow to support OAuth"
   - "Add a caching layer with Redis to the product listing"

4. **Resolve the scope from the argument**:

   **If no argument**: Show the current project structure (domains and flows) and ask the user what they want to update.

   **If `--add-domain`**: Ask the user for the domain name, description, and initial flows. Create:
   - Add the domain entry to `ddd-project.json`
   - Create `specs/domains/{domain-id}/domain.yaml` with the domain config
   - Create flow YAML files for any initial flows

   **If `--add-flow domain-name`**: Ask the user for the flow details. Create:
   - Add the flow entry to `specs/domains/{domain}/domain.yaml`
   - Create `specs/domains/{domain}/flows/{flow-id}.yaml` with the flow spec

   **If `domain-name`**: Update the domain config and/or modify flows within it.

   **If `domain-name/flow-name`**: Update the specific flow spec.

5. **Apply the changes to the YAML specs**:

   When **modifying an existing flow**, read the current flow YAML and update it:
   - **Adding a node**: Create a new node entry with proper `id`, `type`, `position`, `spec`, `label`, and `connections`. Update the upstream node's connections to include the new node. Position it logically on the canvas (below the node it follows, with ~130px vertical spacing).
   - **Removing a node**: Remove the node entry, rewire upstream connections to skip it (connect to the removed node's targets instead).
   - **Modifying a node**: Update the `spec` fields, `label`, or `connections` as needed.
   - **Adding a decision branch**: Add a new connection with `sourceHandle` and create the target node(s).
   - **Applying cross-cutting patterns**: When adding new nodes, check if any `cross_cutting_patterns` from `architecture.yaml` apply:
     - New `service_call` node fetching external content → check if `stealth_http` pattern applies (is the domain in `used_by_domains`?)
     - New `data_store` node with `operation: read` → apply `soft_delete` pattern (add `deletedAt: null` to filters) if applicable
     - New `data_store` node writing credentials → apply `encryption` pattern if applicable
     - Any flow needing API keys → note `api_key_resolution` convention from patterns
   - **Changing flow type**: Update `flow.type` and add/remove agent-specific nodes as needed.

   When **modifying a domain**, update `specs/domains/{domain}/domain.yaml`:
   - **Adding/removing flows**: Update the `flows` array and create/delete flow YAML files.
   - **Updating events**: Update `publishes_events` or `consumes_events` arrays. If the event is cross-domain, remind the user to update the consuming/publishing domain too.

   When **adding a domain**, create all required files and update `ddd-project.json`.

6. **Maintain spec integrity**: After making changes, verify:
   - Every flow still has exactly one trigger
   - All paths from trigger reach a terminal (no dead ends)
   - No orphaned nodes (all reachable from trigger)
   - Decision nodes have both true and false branches
   - Node IDs are unique within the flow
   - All connections reference valid node IDs
   - Event wiring is consistent (published events match consumed events across domains)
   - Agent flows have at least one agent_loop with tools and a terminal tool

7. **Preserve existing data**: When updating a flow:
   - Keep all nodes that aren't being changed — don't regenerate the entire flow
   - Preserve node IDs — changing IDs would break `.ddd/mapping.yaml` references
   - Preserve positions of unchanged nodes
   - Preserve `metadata.created`, update `metadata.modified` to current ISO timestamp
   - Preserve `observability` and `security` configs on unchanged nodes

8. **Handle cross-domain impacts**: If the change affects event wiring:
   - If adding a new event publication, check if any domain consumes it
   - If removing an event, warn about domains that consume it
   - If renaming an event, update all references across domains
   - List all affected files after making cross-domain changes

9. **Report what changed**: After updating, show a clear summary:
   ```
   Updated specs:
     specs/domains/users/flows/user-register.yaml
       + Added node: rate-limiter (process) after trigger
       ~ Modified node: input-001 connections (rewired through rate-limiter)
       ~ Updated metadata.modified

     specs/domains/users/domain.yaml
       + Added event: UserRateLimited to publishes_events

   Affected domains: users

     Cross-cutting patterns applied: stealth_http (to service_call nodes)

   Next steps:
     - Reload the DDD Tool to see changes (Cmd+R)
     - Run /ddd-implement users/user-register to update the implementation
   ```

10. **Suggest next steps**: After updating specs, tell the user:
    - "Reload the DDD Tool to see the updated flow graph (Cmd+R)"
    - "Run `/ddd-implement {scope}` to update the implementation to match"
    - If cross-domain changes were made, list which other flows may need re-implementation

## Node Type Reference

When creating new nodes, **fetch the DDD Usage Guide** for the full node type reference:

```bash
gh api repos/mhcandan/DDD/contents/DDD-USAGE-GUIDE.md --jq '.content' | base64 -d
```

This guide defines all node types, their required spec fields, connection patterns (sourceHandle values), and conventions. Always refer to it when adding or modifying nodes.

## Node ID Convention

When creating new nodes, use the format `{type}-{nanoid(8)}` where nanoid is an 8-character random string. Examples:
- `process-xK9mR2vL`
- `decision-aPq3nW8j`
- `data_store-bY7cT4hK`

$ARGUMENTS
