# DDD Tool Auto-Test Feature — Implementation Prompt

## Context

Give this prompt to a Claude Code session working on the **ddd-tool** repo (`~/dev/ddd-tool`). It describes a standalone Node.js CLI runner that validates DDD projects and produces two YAML reports.

---

## What to Build

A **Node.js CLI auto-test runner** that loads a DDD project, runs the existing validation and parsing logic against it, checks feature coverage, and produces two YAML report files. This is NOT a Tauri feature — it's a standalone Node.js script that imports the same TypeScript modules the Tauri app uses.

### Why a Node.js runner (not Tauri)

The validation logic is all TypeScript with no Tauri dependency:
- `normalizeFlowDocument()` in `src/stores/flow-store.ts` (line ~407) — pure function, no Tauri calls
- `validateFlow()`, `validateDomain()`, `validateSystem()` in `src/utils/flow-validator.ts` (line ~1083, ~1229, ~1486) — pure functions
- `buildSystemMapData()`, `buildDomainMapData()` in `src/utils/domain-parser.ts` (line ~40, ~128) — pure functions, no Tauri calls

The only function that calls `invoke()` (Tauri IPC) is `extractOrchestrationData()` in `src/utils/domain-parser.ts` (line ~307), which reads flow YAML files via `invoke('read_file', ...)`. The auto-test runner replaces this with a Node.js filesystem adapter.

A Tauri Rust command would mean reimplementing 1,600+ lines of TypeScript validation logic. Instead, run the same code via `npx tsx`.

---

## Files to Create

```
src/auto-test/
  runner.ts              — CLI entry point
  fs-adapter.ts          — Node.js filesystem adapter (replaces Tauri invoke() calls)
  project-loader.ts      — Loads all specs into memory
  report-generator.ts    — Categorizes issues into two YAML reports
  coverage-checker.ts    — Feature coverage analysis
  types.ts               — AutoTestResult, report interfaces
```

### Add npm script to package.json

```json
{
  "scripts": {
    "auto-test": "tsx src/auto-test/runner.ts"
  }
}
```

Add `tsx` and `js-yaml` as dev dependencies if not already present.

Usage: `npm run auto-test -- /path/to/ddd-project`

---

## Implementation Details

### 1. `types.ts` — Shared Types

```typescript
export interface AutoTestConfig {
  projectPath: string;
  outputDir: string;       // defaults to projectPath/.ddd/reports/
  verbose: boolean;
}

export interface AutoTestResult {
  projectPath: string;
  timestamp: string;
  toolIssues: ToolIssue[];
  specIssues: SpecIssue[];
  coverage: CoverageReport;
}

export interface ToolIssue {
  category: 'parse_error' | 'normalize_error' | 'unknown_node_type' | 'view_builder_error' | 'unrecognized_feature';
  severity: 'error' | 'warning';
  file: string;
  message: string;
  stackTrace?: string;
}

export interface SpecIssue {
  category: 'validation_error' | 'validation_warning' | 'missing_coverage' | 'broken_reference' | 'event_wiring';
  severity: 'error' | 'warning';
  scope: 'flow' | 'domain' | 'system';
  target: string;          // e.g., "ingestion/fetch-rss-feeds" or "ingestion" or "system"
  message: string;
  rule?: string;           // validation rule ID if applicable
}

export interface CoverageReport {
  nodeTypes: CoverageItem;
  triggerTypes: CoverageItem;
  collectionOps: CoverageItem;
  cryptoOps: CoverageItem;
  parseFormats: CoverageItem;
  dataStoreTypes: CoverageItem;
  connectionBehaviors: CoverageItem;
  uiComponents: CoverageItem;
  formFieldTypes: CoverageItem;
  schemaFeatures: CoverageItem;
  orchestratorStrategies: CoverageItem;
  handoffModes: CoverageItem;
  smartRouterModes: CoverageItem;
}

export interface CoverageItem {
  total: number;
  covered: number;
  percentage: number;
  missing: string[];
  found: string[];
}
```

### 2. `fs-adapter.ts` — Node.js Filesystem Adapter

Replace Tauri's `invoke('read_file', { path })` with Node.js `fs.readFileSync()`. This module provides a filesystem interface that `extractOrchestrationData()` and the project loader can use.

```typescript
import { readFileSync, existsSync, readdirSync, statSync } from 'fs';
import { join } from 'path';

export interface FileSystem {
  readFile(path: string): string;
  pathExists(path: string): boolean;
  listDirectory(path: string): string[];
  isDirectory(path: string): boolean;
}

export const nodeFs: FileSystem = {
  readFile: (path: string) => readFileSync(path, 'utf-8'),
  pathExists: (path: string) => existsSync(path),
  listDirectory: (path: string) => readdirSync(path),
  isDirectory: (path: string) => existsSync(path) && statSync(path).isDirectory(),
};
```

**Important:** The existing `extractOrchestrationData()` in `domain-parser.ts` calls `invoke('read_file', ...)`. For the auto-test runner, you need to either:
- (a) Monkey-patch the `invoke` function before importing, or
- (b) Create a thin wrapper that calls the same logic but uses `nodeFs.readFile()` instead, or
- (c) Accept that orchestration data extraction won't run in auto-test mode (skip it and note the gap).

Approach (b) is recommended. Copy the relevant logic from `extractOrchestrationData()` into the project loader.

### 3. `project-loader.ts` — Load All Specs

Mirrors the loading logic from the Tauri app's stores. Reads:

1. `ddd-project.json` — parse domain list
2. `specs/system.yaml` — system config
3. `specs/domains/*/domain.yaml` — domain configs
4. `specs/domains/*/flows/*.yaml` — all flow YAML files
5. `specs/schemas/*.yaml` — schema definitions
6. `specs/ui/pages.yaml` + `specs/ui/*.yaml` — UI specs
7. `specs/infrastructure.yaml` — infrastructure spec
8. `specs/architecture.yaml` — architecture spec
9. `specs/config.yaml` — config spec
10. `specs/shared/errors.yaml` — error codes
11. `specs/shared/types.yaml` — shared types

For each flow YAML, call `normalizeFlowDocument()` from `src/stores/flow-store.ts` to get the canonical `FlowDocument` shape.

**Loading pattern:**

```typescript
import { parse as parseYaml } from 'js-yaml';
// Import normalizeFlowDocument from flow-store
// This may require extracting the function or importing the module

export interface LoadedProject {
  projectConfig: any;            // ddd-project.json
  systemConfig: any;             // system.yaml
  domainConfigs: Record<string, DomainConfig>;
  flowDocuments: Record<string, FlowDocument>;  // keyed by "domain/flow-id"
  schemas: Record<string, any>;
  uiPages: any;
  uiPageSpecs: Record<string, any>;
  infrastructure: any;
  architecture: any;
  config: any;
  errors: any;
  sharedTypes: any;
  loadErrors: ToolIssue[];       // any parse/normalize failures
}
```

**Handling failures:** If a YAML file fails to parse or `normalizeFlowDocument()` throws, catch the error and add it as a `ToolIssue` (category: `parse_error` or `normalize_error`). Continue loading other files — don't abort on first failure.

### 4. `coverage-checker.ts` — Feature Coverage Analysis

Walks the loaded project and counts which DDD features are actually used. Compares against the full catalog.

**Full Feature Catalogs to Check Against:**

```typescript
const ALL_NODE_TYPES = [
  'trigger', 'input', 'process', 'decision', 'terminal',
  'data_store', 'service_call', 'ipc_call', 'event', 'loop',
  'parallel', 'sub_flow', 'llm_call', 'delay', 'cache',
  'transform', 'collection', 'parse', 'crypto', 'batch',
  'transaction', 'agent_loop', 'guardrail', 'human_gate',
  'orchestrator', 'smart_router', 'handoff', 'agent_group',
];

const ALL_TRIGGER_TYPES = [
  'HTTP GET', 'HTTP POST', 'HTTP PUT', 'HTTP DELETE',
  'cron', 'event:', 'event_group:', 'webhook', 'manual',
  'shortcut', 'timer', 'ui:', 'ipc:', 'sse', 'ws', 'pattern',
];

const ALL_COLLECTION_OPS = [
  'filter', 'sort', 'deduplicate', 'merge',
  'group_by', 'aggregate', 'reduce', 'flatten',
];

const ALL_CRYPTO_OPS = [
  'encrypt', 'decrypt', 'hash', 'sign', 'verify', 'generate_key',
];

const ALL_PARSE_FORMATS = [
  'rss', 'atom', 'html', 'xml', 'json', 'csv', 'markdown',
];

const ALL_DATA_STORE_TYPES = ['database', 'filesystem', 'memory'];

const ALL_CONNECTION_BEHAVIORS = ['continue', 'stop', 'retry', 'circuit_break'];

const ALL_UI_COMPONENTS = [
  'stat-card', 'item-list', 'card-grid', 'detail-card',
  'button-group', 'page-header', 'status-bar', 'chart', 'filter-bar',
];

const ALL_FORM_FIELD_TYPES = [
  'text', 'number', 'select', 'multi-select', 'search-select',
  'date', 'datetime', 'date-range', 'textarea', 'toggle',
  'tag-input', 'file', 'color', 'slider',
];

const ALL_SCHEMA_FEATURES = [
  'inherits', 'encrypted', 'transitions',
  'btree_index', 'hash_index', 'gin_index', 'gist_index',
  'seed_migration', 'seed_fixture', 'seed_script',
  'has_many', 'belongs_to', 'has_one', 'many_to_many',
];

const ALL_ORCHESTRATOR_STRATEGIES = [
  'supervisor', 'round_robin', 'broadcast', 'consensus',
];

const ALL_HANDOFF_MODES = ['transfer', 'consult', 'collaborate'];

const ALL_SMART_ROUTER_MODES = ['rules', 'llm_routing'];
```

**How to detect each feature:**

- **Node types:** Iterate all flow documents, collect `trigger.type` and `nodes[].type`
- **Trigger types:** Parse `trigger.spec.event` string — match against patterns (starts with "HTTP", "cron", "event:", etc.)
- **Collection ops:** Find nodes where `type === 'collection'` and read `spec.operation`
- **Crypto ops:** Find nodes where `type === 'crypto'` and read `spec.operation`
- **Parse formats:** Find nodes where `type === 'parse'` and read `spec.format`
- **Data store types:** Find nodes where `type === 'data_store'` and read `spec.store_type` (default: `'database'`)
- **Connection behaviors:** Iterate all connections across all flows, check for `behavior` field
- **UI components:** Parse `specs/ui/*.yaml` page specs, collect `sections[].component` values
- **Form field types:** Parse form specs, collect `fields[].type` values
- **Schema features:** Parse `specs/schemas/*.yaml`, check for `inherits`, `encrypted`, `transitions`, index types, seed strategies, relationship types
- **Orchestrator strategies:** Find `orchestrator` nodes, read `spec.strategy`
- **Handoff modes:** Find `handoff` nodes, read `spec.mode`
- **Smart router modes:** Find `smart_router` nodes, check if `spec.rules` exists (rules mode) and `spec.llm_routing.enabled` (LLM mode)

### 5. `report-generator.ts` — Produce Two YAML Reports

Splits issues into two report files based on this heuristic:

**tool-compatibility-report.yaml** — "The tool broke." Issues where the DDD Tool itself can't handle the spec:
- Parse/normalize errors (YAML syntax, unknown fields that crash normalization)
- Unknown node types (not in the tool's type registry)
- View builder failures (buildSystemMapData/buildDomainMapData crash)
- Unrecognized features (fields the tool doesn't understand)

**spec-quality-report.yaml** — "The specs are wrong." Issues with the spec content:
- Validation errors/warnings from `validateFlow()`, `validateDomain()`, `validateSystem()`
- Missing coverage (DDD features not exercised)
- Broken references (event wiring, sub_flow refs, schema refs)
- Event wiring problems

**Report format:**

```yaml
# tool-compatibility-report.yaml
report: tool-compatibility
generated: "2025-01-15T10:30:00Z"
project: /path/to/project
summary:
  total_issues: 5
  errors: 3
  warnings: 2
issues:
  - category: parse_error
    severity: error
    file: specs/domains/ingestion/flows/fetch-rss-feeds.yaml
    message: "YAML parse error at line 45: unexpected indent"
  - category: unknown_node_type
    severity: error
    file: specs/domains/processing/flows/analyze.yaml
    message: "Unknown node type 'custom_processor' at node custom-001"
```

```yaml
# spec-quality-report.yaml
report: spec-quality
generated: "2025-01-15T10:30:00Z"
project: /path/to/project
summary:
  total_issues: 12
  errors: 4
  warnings: 8
  coverage:
    node_types: "25/28 (89%)"
    trigger_types: "13/16 (81%)"
    collection_ops: "8/8 (100%)"
issues:
  - category: validation_error
    severity: error
    scope: flow
    target: ingestion/fetch-rss-feeds
    message: "Decision node 'decision-001' missing false branch connection"
    rule: branch_completeness
  - category: event_wiring
    severity: warning
    scope: system
    target: system
    message: "Event 'ContentArchived' is published but never consumed"
coverage:
  node_types:
    total: 28
    covered: 25
    percentage: 89
    missing: [agent_group, smart_router, handoff]
    found: [trigger, input, process, ...]
  # ... other coverage sections
```

### 6. `runner.ts` — CLI Entry Point

```typescript
#!/usr/bin/env node
import { resolve } from 'path';
// Import other modules

async function main() {
  const args = process.argv.slice(2);

  if (args.length === 0 || args.includes('--help')) {
    console.log('Usage: npm run auto-test -- <project-path> [--output <dir>] [--verbose]');
    console.log('');
    console.log('Validates a DDD project and generates compatibility and quality reports.');
    console.log('');
    console.log('Options:');
    console.log('  --output <dir>   Output directory for reports (default: <project>/.ddd/reports/)');
    console.log('  --verbose        Show detailed progress');
    process.exit(0);
  }

  const projectPath = resolve(args[0]);
  const outputIdx = args.indexOf('--output');
  const outputDir = outputIdx >= 0 ? resolve(args[outputIdx + 1]) : resolve(projectPath, '.ddd/reports');
  const verbose = args.includes('--verbose');

  console.log(`DDD Auto-Test Runner`);
  console.log(`Project: ${projectPath}`);
  console.log(`Output:  ${outputDir}`);
  console.log('');

  // 1. Load project
  console.log('Loading project...');
  const project = loadProject(projectPath);
  console.log(`  Loaded ${Object.keys(project.domainConfigs).length} domains, ${Object.keys(project.flowDocuments).length} flows`);
  if (project.loadErrors.length > 0) {
    console.log(`  ${project.loadErrors.length} load errors (see tool-compatibility-report)`);
  }

  // 2. Run validation
  console.log('Running validation...');
  const validationIssues = runValidation(project);
  console.log(`  ${validationIssues.length} validation issues`);

  // 3. Run coverage checks
  console.log('Checking feature coverage...');
  const coverage = checkCoverage(project);

  // 4. Run renderability checks
  console.log('Checking renderability...');
  const renderIssues = checkRenderability(project);

  // 5. Generate reports
  console.log('Generating reports...');
  const result: AutoTestResult = {
    projectPath,
    timestamp: new Date().toISOString(),
    toolIssues: [...project.loadErrors, ...renderIssues],
    specIssues: validationIssues,
    coverage,
  };
  generateReports(result, outputDir);

  // 6. Print summary
  console.log('');
  console.log('=== Results ===');
  console.log(`Tool compatibility issues: ${result.toolIssues.length}`);
  console.log(`Spec quality issues:       ${result.specIssues.length}`);
  console.log(`Feature coverage:`);
  for (const [name, item] of Object.entries(coverage)) {
    const pct = item.percentage.toFixed(0);
    const status = item.percentage === 100 ? '✓' : item.percentage >= 80 ? '~' : '✗';
    console.log(`  ${status} ${name}: ${item.covered}/${item.total} (${pct}%)`);
  }
  console.log('');
  console.log(`Reports written to: ${outputDir}/`);

  // Exit with error code if there are errors
  const hasErrors = result.toolIssues.some(i => i.severity === 'error') ||
                    result.specIssues.some(i => i.severity === 'error');
  process.exit(hasErrors ? 1 : 0);
}

main().catch(err => {
  console.error('Fatal error:', err);
  process.exit(2);
});
```

### Renderability Checks

After loading and normalizing flows, attempt to call:
- `buildSystemMapData()` with the loaded domain configs and a dummy system layout
- `buildDomainMapData()` for each domain

Catch any errors and record them as `ToolIssue` entries (category: `view_builder_error`). These represent cases where the DDD Tool's canvas rendering would fail.

**Dummy system layout:** Create a minimal `SystemLayout` with auto-positioned domains (spread across a grid). The exact positions don't matter — we're testing that the builder doesn't crash, not that the layout looks good.

### Validation Integration

Call the existing validators directly:

```typescript
import { validateFlow, validateDomain, validateSystem } from '../utils/flow-validator';

function runValidation(project: LoadedProject): SpecIssue[] {
  const issues: SpecIssue[] = [];

  // Validate each flow
  for (const [key, flowDoc] of Object.entries(project.flowDocuments)) {
    const result = validateFlow(flowDoc, project.domainConfigs);
    for (const issue of result.issues) {
      issues.push({
        category: issue.severity === 'error' ? 'validation_error' : 'validation_warning',
        severity: issue.severity,
        scope: 'flow',
        target: key,
        message: issue.message,
        rule: issue.rule,
      });
    }
  }

  // Validate each domain
  for (const [domainId, domainConfig] of Object.entries(project.domainConfigs)) {
    const domainFlows = Object.entries(project.flowDocuments)
      .filter(([key]) => key.startsWith(domainId + '/'))
      .map(([_, doc]) => doc);
    const result = validateDomain(domainId, domainConfig, project.domainConfigs, domainFlows);
    for (const issue of result.issues) {
      issues.push({
        category: issue.severity === 'error' ? 'validation_error' : 'validation_warning',
        severity: issue.severity,
        scope: 'domain',
        target: domainId,
        message: issue.message,
        rule: issue.rule,
      });
    }
  }

  // Validate system
  const specsContext = {
    schemas: project.schemas,
    uiPages: project.uiPages,
    infrastructure: project.infrastructure,
  };
  const result = validateSystem(project.domainConfigs, specsContext);
  for (const issue of result.issues) {
    issues.push({
      category: issue.severity === 'error' ? 'validation_error' : 'validation_warning',
      severity: issue.severity,
      scope: 'system',
      target: 'system',
      message: issue.message,
      rule: issue.rule,
    });
  }

  return issues;
}
```

**Note:** The `validateFlow`, `validateDomain`, and `validateSystem` function signatures should be verified against the actual code. Check `src/utils/flow-validator.ts` for exact parameter shapes and return types. The `ValidationResult` type likely has an `issues` array where each issue has `severity`, `message`, and optionally `rule` fields — verify this.

---

## Key Codebase References

When implementing, read these files carefully:

| File | What to Look For |
|------|-----------------|
| `src/stores/flow-store.ts` ~line 407 | `normalizeFlowDocument()` — understand what transformations it applies to raw YAML |
| `src/utils/flow-validator.ts` ~line 1083 | `validateFlow()` — parameters, return type, what `domainConfigs` it expects |
| `src/utils/flow-validator.ts` ~line 1229 | `validateDomain()` — parameters, how it uses `flowDocs` |
| `src/utils/flow-validator.ts` ~line 1486 | `validateSystem()` — parameters, what `specsContext` shape it expects |
| `src/utils/domain-parser.ts` ~line 40 | `buildSystemMapData()` — parameters, what `SystemLayout` shape it expects |
| `src/utils/domain-parser.ts` ~line 128 | `buildDomainMapData()` — parameters and return type |
| `src/utils/domain-parser.ts` ~line 307 | `extractOrchestrationData()` — understand the `invoke()` calls to replace |
| `src/types/flow.ts` | `FlowDocument`, `DddFlowNode`, `NodeSpec` types |
| `src/types/domain.ts` | `DomainConfig`, `SystemLayout`, `SystemMapData` types |
| `src/types/validation.ts` | `ValidationResult`, `ValidationIssue` types |
| `package.json` | Current scripts and dependencies |
| `tsconfig.json` | TypeScript config (target ES2020, module ESNext, strict) |

## Import Strategy

The existing code uses ESM-style imports (`import ... from '...'`). Since the project uses Vite + TypeScript with `tsx` for execution, imports should work as-is. However:

1. **Zustand store imports:** `normalizeFlowDocument` is defined inside `flow-store.ts` which also creates a Zustand store. The function itself is a standalone export, but importing the file may trigger Zustand initialization. If this causes issues, extract `normalizeFlowDocument` into a separate utility file (e.g., `src/utils/normalize-flow.ts`) and re-export from `flow-store.ts`.

2. **Type imports:** Use `import type { ... }` for type-only imports to avoid runtime issues.

3. **js-yaml:** The existing codebase may already use `js-yaml` or another YAML parser. Check `package.json` and match what's already there. The Tauri app likely uses its own YAML parsing — if there's no `js-yaml` dependency, add it.

---

## Testing the Runner

### Baseline test: examples/todo-app

The DDD repo contains `examples/todo-app/` — a simple DDD project. Run the auto-test against it first:

```bash
npm run auto-test -- ~/dev/DDD/examples/todo-app
```

This should produce reports with:
- Zero or few tool compatibility issues (it's a known-good project)
- Some spec quality warnings (it's a simple project, won't cover all features)
- Low feature coverage (it's intentionally simple)

### Full test: Generated Nexus project

After generating a Nexus project with `/ddd-create --from ~/dev/DDD/tests/e2e/product-features.md`:

```bash
npm run auto-test -- /path/to/nexus-project
```

This should produce reports with:
- High feature coverage (90%+ across all categories)
- Validation results that reveal gaps in either the specs or the tool

---

## Acceptance Criteria

1. `npm run auto-test -- <path>` runs without crashing on any valid DDD project
2. Produces `tool-compatibility-report.yaml` and `spec-quality-report.yaml` in the output directory
3. Reports are valid YAML that can be parsed by any YAML library
4. Coverage checker correctly identifies all 28 node types, all 16 trigger types, all 8 collection ops, etc.
5. Validation results match what the DDD Tool app would show for the same project
6. Parse/normalize errors are caught and reported (not thrown as uncaught exceptions)
7. The runner exits with code 0 if no errors, 1 if errors found, 2 if fatal crash

---

## Non-Goals

- No Tauri integration — this is a standalone Node.js script
- No UI — command-line only
- No watching/live reload — single run, produce reports, exit
- No fixing — report issues only, don't attempt to fix them
- No internet access — everything is local file operations
