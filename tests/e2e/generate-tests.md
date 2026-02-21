# DDD E2E Test Generator

You are generating the DDD end-to-end test suite. This suite verifies that the three DDD repos (DDD guide, ddd-tool, ddd-commands) work together correctly. Your job is to read the **current state** of all three repos and produce test artifacts that exercise **every** DDD feature as it exists today.

**Do not use any hardcoded feature lists.** Extract everything from the source files. DDD evolves — what you find today is the truth.

---

## Phase 1: Extract the Current DDD Feature Catalog

Read these files and extract every feature, building a complete catalog.

### 1.1 Read the DDD Usage Guide

Read `~/dev/DDD/DDD-USAGE-GUIDE.md` in full. Extract:

**From Section 6 (All Node Types):**
- Every node type name and its spec fields
- Which node types have sourceHandle outputs (and what the handle names are)
- Which node types are traditional, extended-traditional, agent, or orchestration

**From Section 6 trigger conventions:**
- Every trigger type pattern (HTTP GET/POST/PUT/DELETE/PATCH, cron, event:, event_group:, webhook, manual, shortcut, timer, ui:, ipc:, sse, ws, pattern, and any others)
- Trigger-specific fields (job_config, pattern config, filter, debounce_ms, etc.)

**From Section 6 node details:**
- All `collection` operations (filter, sort, deduplicate, merge, group_by, aggregate, reduce, flatten, and any new ones)
- All `crypto` operations (encrypt, decrypt, hash, sign, verify, generate_key, and any new ones)
- All `parse` formats (rss, atom, html, xml, json, csv, markdown, and any new ones)
- All `data_store` store_types (database, filesystem, memory) and all operations per type
- All `orchestrator` strategies
- All `handoff` modes
- All `smart_router` routing modes
- All `parallel` branch/join options
- All `cache` store types
- All `human_gate` fields
- All `guardrail` positions and check types
- All `batch` operation_template fields

**From Section 8 (Connection Patterns):**
- All connection behavior values (continue, stop, retry, circuit_break, and any new ones)
- All sourceHandle values per node type

**From Section 9 (Validation Rules):**
- Every validation rule at flow, domain, and system level
- Error vs warning distinctions

**From Section 4.7 (UI Specs):**
- All component types (stat-card, item-list, card-grid, detail-card, button-group, page-header, status-bar, chart, filter-bar, and any new ones)
- All form field types (text, number, select, multi-select, search-select, date, datetime, date-range, textarea, toggle, tag-input, file, color, slider, and any new ones)
- All layout types, navigation types, loading/error/refresh states
- All page-level fields (state, forms, sections)
- Shared component pattern

**From Section 4.5 (Schemas):**
- All field types (string, number, decimal, boolean, uuid, datetime, enum, json, text, and any new ones)
- All index types (btree, hash, gin, gist, and any new ones)
- All seed strategies (migration, fixture, script, and any new ones)
- All relationship types (has_many, belongs_to, has_one, many_to_many, and any new ones)
- Transitions (field, states, on_invalid options)
- Encrypted field support
- Inherits pattern (including non-_base inheritance)

**From Section 4.1 (system.yaml):**
- Zones, data_flows, schedules, characteristics, pipelines, integrations structure

**From Section 4.8 (infrastructure.yaml):**
- All service types (server, datastore, worker, proxy, and any new ones)
- Deployment strategies

**From Section 3 (Domain YAML):**
- Event groups, stores, on_error hooks, flow groups, domain dependencies
- Flow entry fields (type, tags, criticality, throughput, template, parameters)

**From Section 7 (Cross-Cutting Concerns):**
- Observability fields
- Security fields

**From Sections 12-17 (Advanced Features):**
- mapping.yaml structure
- Flow templates / parameterized flows
- Design patterns
- Implementation patterns

After reading, output a structured summary:

```
=== DDD Feature Catalog (extracted from current Usage Guide) ===

Node Types (N total): [list]
Trigger Types (N total): [list]
Collection Operations (N total): [list]
Crypto Operations (N total): [list]
Parse Formats (N total): [list]
Data Store Types (N total): [list]
Data Store Operations - Database (N total): [list]
Data Store Operations - Memory (N total): [list]
Connection Behaviors (N total): [list]
UI Component Types (N total): [list]
Form Field Types (N total): [list]
Schema Field Types (N total): [list]
Schema Index Types (N total): [list]
Schema Seed Strategies (N total): [list]
Schema Relationship Types (N total): [list]
Orchestrator Strategies (N total): [list]
Handoff Modes (N total): [list]
Smart Router Modes (N total): [list]
Infrastructure Service Types (N total): [list]
Page Layout Types (N total): [list]
Navigation Types (N total): [list]
```

### 1.2 Read the ddd-tool Codebase

Read key files in `~/dev/ddd-tool/`:

1. **`src/utils/flow-validator.ts`** — Find `validateFlow()`, `validateDomain()`, `validateSystem()`. Record:
   - Exact function signatures (parameter names, types, return type)
   - Line numbers
   - What `specsContext` or other optional parameters accept
   - Any new validation functions added since last generation

2. **`src/stores/flow-store.ts`** — Find `normalizeFlowDocument()`. Record:
   - Exact function signature
   - Line number
   - Whether it's a standalone export or embedded in a Zustand store
   - Any Tauri-dependent imports in the file

3. **`src/utils/domain-parser.ts`** — Find `buildSystemMapData()`, `buildDomainMapData()`, `extractOrchestrationData()`. Record:
   - Exact function signatures and line numbers
   - Which functions call `invoke()` (Tauri IPC)
   - Which functions are pure (no Tauri dependency)

4. **`src/types/`** — Read the type definition files. Record:
   - `FlowDocument`, `DddFlowNode`, `NodeSpec` type shapes
   - `DomainConfig`, `SystemLayout`, `SystemMapData` type shapes
   - `ValidationResult`, `ValidationIssue` type shapes
   - Any new types relevant to validation or parsing

5. **`package.json`** — Record:
   - Current scripts
   - Current dependencies (especially: is `tsx` present? `js-yaml`?)
   - Project name and structure

6. **`tsconfig.json`** — Record target, module, strict settings

7. **Any existing `src/auto-test/` directory** — If it already exists from a previous generation, read what's there to understand current state.

Output a structured summary:

```
=== ddd-tool Codebase Snapshot ===

normalizeFlowDocument: line N, signature: (raw, domainId, flowId, flowType?) => FlowDocument
validateFlow: line N, signature: (flow, domainConfigs?) => ValidationResult
validateDomain: line N, signature: (domainId, domainConfig, allDomainConfigs, flowDocs?) => ValidationResult
validateSystem: line N, signature: (domainConfigs, specsContext?) => ValidationResult
buildSystemMapData: line N, signature: (domainConfigs, systemLayout) => SystemMapData
buildDomainMapData: line N, signature: (domainId, domainConfig, allDomainConfigs) => DomainMapData
extractOrchestrationData: line N, USES INVOKE — needs fs-adapter

Tauri-dependent functions: [list]
Pure functions (safe to import): [list]

ValidationResult shape: { issues: ValidationIssue[] }
ValidationIssue shape: { severity, message, rule?, ... }

Existing auto-test files: [list or "none"]
```

### 1.3 Read ddd-commands

Read `~/.claude/commands/ddd-create.md`. Record:
- What input formats `/ddd-create --from` accepts
- How it processes the input file
- What output files it generates
- The four-pillar coverage check it performs

This ensures the product features file is written in a format that `/ddd-create --from` processes well.

---

## Phase 2: Design the Product Concept

Using the complete feature catalog from Phase 1, design a product concept that **naturally exercises every feature**. The product must feel like a real application, not a test checklist.

### Design Principles

1. **Every feature must be traceable to a product need.** Don't say "uses agent_loop because we need to test it." Say "the AI analysis pipeline uses an agent loop because content analysis requires iterative tool use."

2. **Domains should have natural event wiring.** Content flows between domains via events, not just because events need testing.

3. **Schema relationships should reflect real data.** A User has_many Content because users author content, not because we need to test has_many.

4. **The product should have enough domains to justify all features.** Typically 7-12 domains, 40-60 flows.

### Feature-to-Product Mapping

For each feature in the catalog, assign it to a product domain and flow. Create a mapping table:

```
=== Feature-to-Product Mapping ===

Node Type: trigger → every flow (inherent)
Node Type: agent_loop → [domain]/[flow] — [product reason]
Node Type: orchestrator → [domain]/[flow] — [product reason]
...
Trigger: cron → [domain]/[flow] — [product reason]
Trigger: sse → [domain]/[flow] — [product reason]
...
```

**Check for gaps:** Any feature that doesn't have a natural product home is a red flag. Either:
- Expand the product concept to include a use case for it
- Or note it as "contrived" in the traceability checklist (acceptable for rare features like `gist` index)

### Domain Structure

Design the domain structure as a table:

```
=== Domain Structure ===

| Domain | Role | Flows | Key Features Exercised |
|--------|------|-------|----------------------|
| ...    | ...  | N     | [list]               |
```

---

## Phase 3: Generate product-features.md

Write `~/dev/DDD/tests/e2e/product-features.md` — a natural-language product description.

### Structure

The file must follow this exact structure (because `/ddd-create --from` expects a product description):

```markdown
# [Product Name] — [Tagline]

## Overview
[2-3 paragraph product description. Include tech stack.]

## System Architecture
### Zones
[List all zones with domains]

### Data Flows
[Zone-to-zone data flows]

### Characteristics
[System-level badges]

### Schedules
[All cron flows]

### Pipelines
[Cross-domain pipelines]

### External Integrations
[All external APIs with auth, rate limits, retry, timeout]

## Domains
### 1. [Domain Name] ([role] domain)
[For each domain: description, owned schemas, event groups, then each flow
with enough detail that /ddd-create generates the right node types.]

## Data Models
[Every schema with fields, indexes, relationships, transitions, seed data.
Must exercise all index types, seed strategies, relationship types, etc.]

## User Interface
[App config, navigation, shared components, then each page with sections,
forms, state management. Must exercise all UI components and form field types.]

## Infrastructure
[Services, startup order, deployment. Must exercise all service types.]

## Environment Variables
[Required and optional config vars.]

## Error Codes
[Error codes with HTTP status mappings.]

## DDD Feature Coverage Checklist
[Traceability table mapping EVERY feature to the flow/schema/page that exercises it.
This is how we verify completeness.]
```

### Writing Guidelines for Each Flow

When describing flows, include enough detail that `/ddd-create` generates the correct node types. Specifically:

- **Name the node type explicitly** when it's a specialized type: "parses as RSS format" (not just "processes the feed"), "uses crypto hash operation" (not just "hashes the password"), "checks cache with key template" (not just "caches the result").
- **Name the trigger type** in a recognizable pattern: "Cron trigger (every 15 min)", "Event trigger (`event:ContentIngested`)", "WebSocket trigger at `/api/v1/live`".
- **Name connection behaviors** when non-default: "Uses connection behavior: circuit_break for repeated failures."
- **Name data_store operations and store_types**: "data_store filesystem read", "data_store memory reset", "data_store upsert with upsert_key".
- **For collection operations**, be specific: "collection filter operation", "collection group_by", "collection reduce with accumulator".
- **For UI sections**, name the component type: "stat-card component", "chart component with chart_type: line".
- **For forms**, name the field type: "search-select field with search_source", "slider field with min/max/step".

### Coverage Checklist

The final section must be a complete traceability matrix. For each feature category, list every feature and the flow/schema/page that exercises it. Flag any features marked "contrived" or missing.

---

## Phase 4: Generate ddd-tool-auto-test-prompt.md

Write `~/dev/DDD/tests/e2e/ddd-tool-auto-test-prompt.md` — a prompt for implementing the auto-test runner in ddd-tool.

### Key Principle

This prompt must reference **actual code paths from today's ddd-tool codebase**, not assumed paths. Use the exact function signatures, line numbers, type names, and file paths you found in Phase 1.2.

### Structure

```markdown
# DDD Tool Auto-Test Feature — Implementation Prompt

## Context
[Brief explanation of what to build and why]

## What to Build
[Node.js CLI runner description]

### Why a Node.js runner (not Tauri)
[List pure functions vs Tauri-dependent functions from Phase 1.2]

## Files to Create
[6 files in src/auto-test/]

## Implementation Details

### 1. types.ts
[Type definitions for AutoTestResult, ToolIssue, SpecIssue, CoverageReport.
CoverageReport must have a field for EVERY feature category found in Phase 1.1.]

### 2. fs-adapter.ts
[Node.js filesystem adapter. Reference ACTUAL invoke() calls found in Phase 1.2.]

### 3. project-loader.ts
[Loads all specs. Reference ACTUAL normalizeFlowDocument signature from Phase 1.2.]

### 4. coverage-checker.ts
[Feature catalogs — use the ACTUAL lists from Phase 1.1, not hardcoded.
Include detection logic for each feature category.]

### 5. report-generator.ts
[Two YAML reports with format examples.]

### 6. runner.ts
[CLI entry point. Reference ACTUAL validator signatures from Phase 1.2.]

## Key Codebase References
[Table of files with ACTUAL line numbers and functions from Phase 1.2.]

## Import Strategy
[Based on ACTUAL tsconfig and module system from Phase 1.2.]

## Testing
[Baseline test with examples/todo-app, full test with generated project.]

## Acceptance Criteria
[Exit codes, report format, coverage checker accuracy.]
```

### Coverage Checker Catalogs

The `coverage-checker.ts` section must include const arrays for every feature category. **These arrays must match exactly what you extracted in Phase 1.1.** Example:

```typescript
// Extracted from DDD-USAGE-GUIDE.md Section 6, [today's date]
const ALL_NODE_TYPES = [
  // ... every node type found in the guide
];
```

---

## Phase 5: Verify Completeness

Before finishing, run these checks:

### Check 1: Feature Coverage

For every feature in the Phase 1.1 catalog, verify it appears in:
- [ ] The product-features.md (in a flow, schema, or page)
- [ ] The coverage checklist at the bottom of product-features.md
- [ ] The ALL_* arrays in the coverage-checker.ts section of the auto-test prompt

### Check 2: Code Path Accuracy

For every code reference in ddd-tool-auto-test-prompt.md, verify:
- [ ] The file exists at the stated path
- [ ] The function exists at approximately the stated line number
- [ ] The function signature matches what you read

### Check 3: /ddd-create Compatibility

Verify the product-features.md:
- [ ] Reads like a natural product description (not a test matrix)
- [ ] Includes explicit tech stack mention
- [ ] Describes each domain with enough detail for flow generation
- [ ] Describes schemas with full field definitions
- [ ] Describes UI pages with sections and forms
- [ ] Describes infrastructure with all service types

### Check 4: Feature Delta

If previous test artifacts exist (`tests/e2e/product-features.md`, `tests/e2e/ddd-tool-auto-test-prompt.md`), compare what changed:

```
=== Feature Delta Since Last Generation ===

Added features: [list any features in current catalog not in previous artifacts]
Removed features: [list any features in previous artifacts not in current catalog]
Changed code paths: [list any ddd-tool functions that moved or changed signature]
```

This delta helps the user understand what evolved.

---

## Output

When complete, you should have written exactly two files:

1. `~/dev/DDD/tests/e2e/product-features.md` — fresh product description
2. `~/dev/DDD/tests/e2e/ddd-tool-auto-test-prompt.md` — fresh auto-test prompt

And displayed:
- The feature catalog summary (Phase 1.1)
- The ddd-tool codebase snapshot (Phase 1.2)
- The feature-to-product mapping (Phase 2)
- The completeness verification results (Phase 5)
- The feature delta from previous artifacts if they existed (Phase 5, Check 4)

---

## Usage

To run this generator:

```bash
cd ~/dev/DDD
# In a Claude Code session:
# "Read tests/e2e/generate-tests.md and execute it"
```

The generator needs access to:
- `~/dev/DDD/` — DDD repo (Usage Guide, templates, examples)
- `~/dev/ddd-tool/` — ddd-tool repo (validators, parsers, types)
- `~/.claude/commands/` — ddd-commands repo (slash command prompts)

Estimated time: 5-10 minutes for a full generation.
Recommended frequency: Monthly, or after any framework change.
