# DDD Implementation Guide for Claude Code

## Overview

You are building **DDD (Design Driven Development)** — a desktop app for visual flow editing that outputs YAML specs.

**Read the full specification:** `ddd-specification-complete.md` (in same directory)

---

## Phase 1: MVP Scope (Build This First)

### What MVP Includes
- Desktop app (Tauri + React)
- **Multi-level canvas** (System Map → Domain Map → Flow Sheet)
- **Breadcrumb navigation** between sheet levels
- **Auto-generated** System Map (L1) and Domain Map (L2) from specs
- **Portal nodes** for cross-domain navigation
- Canvas with 5 basic node types (trigger, input, process, decision, terminal) — extended to 19 traditional + 4 agent + 4 orchestration = 27 total in later sessions
- Right panel for editing node specs
- Save/load YAML files
- Basic Git status display
- Export to Mermaid diagram

### What MVP Excludes
- Code generation
- Reverse engineering (code → diagram)
- Expert agents
- Community library
- Real-time collaboration
- MCP server

---

## Phase 2: Project Setup

### Step 1: Initialize Tauri Project

```bash
# Create project
npm create tauri-app@latest ddd-tool -- --template react-ts
cd ddd-tool

# Install dependencies
npm install zustand @radix-ui/react-dialog @radix-ui/react-dropdown-menu
npm install @radix-ui/react-tooltip @radix-ui/react-tabs
npm install lucide-react clsx tailwindcss postcss autoprefixer
npm install yaml uuid nanoid
npm install @tldraw/tldraw  # Or: npm install reactflow

# Initialize Tailwind
npx tailwindcss init -p
```

### Step 2: Project Structure

```
ddd-tool/
├── src-tauri/
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs
│       ├── lib.rs
│       ├── commands/
│       │   ├── mod.rs
│       │   ├── file.rs        # File operations
│       │   ├── entity.rs      # Domain/flow CRUD (create, rename, delete, move, duplicate)
│       │   ├── git.rs         # Git operations
│       │   ├── project.rs     # Project management
│       │   ├── pty.rs         # PTY terminal for Claude Code
│       │   └── test_runner.rs # Run tests and parse output
│       └── services/
│           ├── mod.rs
│           ├── git_service.rs
│           ├── file_service.rs
│           ├── pty_service.rs  # PTY process management
│           └── test_service.rs # Test execution and parsing
│
├── src/
│   ├── main.tsx
│   ├── App.tsx
│   ├── components/
│   │   ├── Navigation/
│   │   │   ├── Breadcrumb.tsx       # Breadcrumb bar (System > domain > flow)
│   │   │   └── SheetTabs.tsx        # Optional tab bar for open sheets
│   │   ├── SystemMap/
│   │   │   ├── SystemMap.tsx          # Level 1: domain blocks + event arrows
│   │   │   ├── DomainBlock.tsx        # Clickable domain block with flow count
│   │   │   ├── DomainContextMenu.tsx  # Right-click menu: rename, delete, edit, add event
│   │   │   ├── CanvasContextMenu.tsx  # Right-click L1 background: add domain
│   │   │   └── EventArrow.tsx         # Arrow between domains (shared with DomainMap)
│   │   ├── DomainMap/
│   │   │   ├── DomainMap.tsx          # Level 2: flow blocks + portals
│   │   │   ├── FlowBlock.tsx          # Clickable flow block
│   │   │   ├── FlowContextMenu.tsx    # Right-click menu: rename, delete, duplicate, move, change type
│   │   │   ├── L2CanvasContextMenu.tsx # Right-click L2 background: add flow
│   │   │   └── PortalNode.tsx         # Cross-domain navigation node
│   │   ├── Canvas/
│   │   │   ├── Canvas.tsx           # Level 3: flow sheet (routes to traditional or agent)
│   │   │   ├── AgentCanvas.tsx      # Agent flow layout (agent loop + tools + guardrails)
│   │   │   ├── Node.tsx
│   │   │   ├── Connection.tsx
│   │   │   ├── nodes/              # Traditional flow nodes
│   │   │   │   ├── TriggerNode.tsx
│   │   │   │   ├── InputNode.tsx
│   │   │   │   ├── ProcessNode.tsx
│   │   │   │   ├── DecisionNode.tsx
│   │   │   │   ├── TerminalNode.tsx
│   │   │   │   └── SubFlowNode.tsx
│   │   │   ├── agent-nodes/        # Agent flow nodes
│   │   │   │   ├── AgentLoopBlock.tsx
│   │   │   │   ├── GuardrailBlock.tsx
│   │   │   │   ├── HumanGateBlock.tsx
│   │   │   │   ├── ToolPalette.tsx
│   │   │   │   └── MemoryBlock.tsx
│   │   │   └── orchestration-nodes/ # Orchestration nodes
│   │   │       ├── OrchestratorBlock.tsx
│   │   │       ├── SmartRouterBlock.tsx
│   │   │       ├── HandoffBlock.tsx
│   │   │       └── AgentGroupBoundary.tsx
│   │   ├── SpecPanel/
│   │   │   ├── SpecPanel.tsx
│   │   │   ├── TriggerSpec.tsx
│   │   │   ├── InputSpec.tsx
│   │   │   ├── ProcessSpec.tsx
│   │   │   ├── DecisionSpec.tsx
│   │   │   ├── TerminalSpec.tsx
│   │   │   ├── AgentLoopSpec.tsx    # Agent loop config editor
│   │   │   ├── ToolSpec.tsx         # Tool definition editor
│   │   │   ├── GuardrailSpec.tsx    # Guardrail checks editor
│   │   │   ├── HumanGateSpec.tsx    # Human gate config editor
│   │   │   ├── RouterSpec.tsx       # Basic router config editor
│   │   │   ├── LLMCallSpec.tsx      # LLM call config editor
│   │   │   ├── OrchestratorSpec.tsx # Orchestrator config (strategy, agents, supervision)
│   │   │   ├── SmartRouterSpec.tsx  # Smart router (rules, LLM, policies, A/B)
│   │   │   ├── HandoffSpec.tsx      # Handoff config (mode, context, failure)
│   │   │   └── AgentGroupSpec.tsx   # Agent group (members, shared memory, coordination)
│   │   ├── Sidebar/
│   │   │   ├── FlowList.tsx
│   │   │   ├── NodePalette.tsx
│   │   │   └── GitPanel.tsx
│   │   ├── FlowCanvas/
│   │   │   └── StaleBanner.tsx         # Stale detection warning banner (Accept + Dismiss)
│   │   ├── ProjectLauncher/
│   │   │   ├── ProjectLauncher.tsx    # Main launcher screen (recent projects + actions)
│   │   │   ├── NewProjectWizard.tsx   # 3-step project creation wizard
│   │   │   └── RecentProjects.tsx     # Recent projects list with open/remove
│   │   ├── Settings/
│   │   │   ├── SettingsDialog.tsx     # Modal settings with tab navigation
│   │   │   ├── ClaudeCodeSettings.tsx # CLI path, enabled toggle
│   │   │   ├── EditorSettings.tsx     # Grid snap, auto-save, theme, font size
│   │   │   └── GitSettings.tsx        # Commit messages, branch naming
│   │   ├── FirstRun/
│   │   │   └── FirstRunWizard.tsx     # First-time setup (Claude Code + Get Started)
│   │   ├── Validation/
│   │   │   ├── ValidationPanel.tsx    # Validation results (errors, warnings, info) per scope
│   │   │   ├── ValidationBadge.tsx    # Compact badge for canvas (✓/⚠/✗ + count)
│   │   │   └── ImplementGate.tsx      # Pre-implementation validation gate (validate → prompt → run)
│   │   └── shared/
│   │       ├── Button.tsx
│   │       ├── Input.tsx
│   │       ├── Select.tsx
│   │       ├── Modal.tsx
│   │       └── CopyButton.tsx          # Reusable copy-to-clipboard button for output areas
│   ├── stores/
│   │   ├── sheet-store.ts     # Active sheet, navigation history, breadcrumbs
│   │   ├── flow-store.ts      # Current flow state (Level 3)
│   │   ├── project-store.ts   # Project/file state, domain configs
│   │   ├── ui-store.ts        # UI state (minimap visibility)
│   │   ├── git-store.ts       # Git state
│   │   ├── implementation-store.ts  # Drift detection, mapping persistence
│   │   ├── app-store.ts         # App-level state: current view, recent projects, first-run
│   │   ├── undo-store.ts        # Per-flow undo/redo stacks
│   │   └── validation-store.ts  # Validation results per flow/domain/system, gate state
│   ├── types/
│   │   ├── sheet.ts           # Sheet levels, navigation, breadcrumb types
│   │   ├── domain.ts          # Domain config, event wiring, portal types
│   │   ├── flow.ts
│   │   ├── node.ts
│   │   ├── spec.ts
│   │   ├── project.ts
│   │   ├── implementation.ts  # Drift detection, mapping, reconciliation types
│   │   ├── test-generator.ts  # Derived test cases, test paths, boundary tests, spec compliance
│   │   ├── app.ts             # App shell types: recent projects, settings, first-run, undo
│   │   └── validation.ts     # Validation issue types, scopes, severity, gate state
│   ├── utils/
│   │   ├── yaml.ts
│   │   ├── domain-parser.ts   # Parse domain.yaml → SystemMap/DomainMap data
│   │   ├── prompt-builder.ts  # Build Claude Code prompts from specs
│   │   ├── claude-md-generator.ts  # Generate/update CLAUDE.md from project state
│   │   ├── test-case-deriver.ts   # Walk flow graphs, derive test paths + boundary tests
│   │   ├── flow-validator.ts      # Flow, domain, and system-level validation engine
│   │   ├── flow-templates.ts     # Pre-built flow templates (REST API, CRUD, Webhook, Agent, etc.)
│   │   ├── mermaid-export.ts    # Generate Mermaid flowchart diagrams from flow specs
│   │   └── validation.ts
│   └── styles/
│       └── globals.css
│
├── package.json
├── tailwind.config.js
├── tsconfig.json
└── vite.config.ts
```

---

## Phase 3: Implementation Order

### Week 1: Core Infrastructure

#### Day 1-2: Types and Stores

**File: `src/types/sheet.ts`**
```typescript
export type SheetLevel = 'system' | 'domain' | 'flow';

export interface SheetLocation {
  level: SheetLevel;
  domainId?: string;   // Set for domain and flow levels
  flowId?: string;     // Set for flow level only
}

export interface BreadcrumbSegment {
  label: string;
  location: SheetLocation;
}

export interface NavigationHistory {
  past: SheetLocation[];
  current: SheetLocation;
}
```

**File: `src/types/domain.ts`**
```typescript
import { Position } from './node';

export interface DomainConfig {
  name: string;
  description: string;
  flows: DomainFlowEntry[];
  publishes_events: EventWiring[];
  consumes_events: EventWiring[];
  layout: DomainLayout;
}

export interface DomainFlowEntry {
  id: string;
  name: string;
  description: string;
}

export interface EventWiring {
  event: string;
  from_flow?: string;       // For publishes_events
  handled_by_flow?: string; // For consumes_events
  description: string;
}

export interface DomainLayout {
  flows: Record<string, Position>;
  portals: Record<string, Position>;
}

export interface SystemLayout {
  domains: Record<string, Position>;
}

// Derived data for rendering System Map (Level 1)
export interface SystemMapData {
  domains: SystemMapDomain[];
  eventArrows: SystemMapArrow[];
}

export interface SystemMapDomain {
  id: string;
  name: string;
  description: string;
  flowCount: number;
  position: Position;
}

export interface SystemMapArrow {
  sourceDomain: string;
  targetDomain: string;
  events: string[];       // Event names flowing on this arrow
}

// Derived data for rendering Domain Map (Level 2)
export interface DomainMapData {
  domainId: string;
  flows: DomainMapFlow[];
  portals: DomainMapPortal[];
  eventArrows: DomainMapArrow[];
}

export interface DomainMapFlow {
  id: string;
  name: string;
  description: string;
  position: Position;
}

export interface DomainMapPortal {
  targetDomain: string;
  position: Position;
  events: string[];       // Events flowing through this portal
}

export interface DomainMapArrow {
  sourceFlowId: string;
  targetFlowId?: string;     // Within same domain
  targetPortal?: string;     // To another domain
  event: string;
}

// Entity management action types
export type FlowType = 'traditional' | 'agent' | 'orchestration';

export interface CreateDomainPayload {
  name: string;
  description: string;
}

export interface RenameDomainPayload {
  oldName: string;
  newName: string;
}

export interface CreateFlowPayload {
  domainId: string;
  name: string;
  flowType: FlowType;
}

export interface RenameFlowPayload {
  domainId: string;
  oldId: string;
  newName: string;
}

export interface MoveFlowPayload {
  sourceDomain: string;
  targetDomain: string;
  flowId: string;
}

export interface DuplicateFlowPayload {
  domainId: string;
  flowId: string;
  newName: string;       // defaults to "{name}-copy"
}
```

**File: `src/types/flow.ts`** (consolidated — types previously in node.ts are now here)

> **Note:** The actual implementation uses a single `flow.ts` file for all node types, spec shapes, and flow document types. The original `node.ts` types are superseded.

```typescript
import type { Position } from './sheet';
import type { ValidationIssue } from './validation';

// --- Node types (all 27) ---

export type DddNodeType =
  | 'trigger' | 'input' | 'process' | 'decision' | 'terminal'
  | 'data_store' | 'service_call' | 'event' | 'loop' | 'parallel' | 'sub_flow' | 'llm_call'
  | 'delay' | 'cache' | 'transform'
  | 'collection' | 'parse' | 'crypto' | 'batch' | 'transaction'
  | 'agent_loop' | 'guardrail' | 'human_gate'
  | 'orchestrator' | 'smart_router' | 'handoff' | 'agent_group';

// --- Traditional node spec shapes ---

export interface TriggerSpec {
  event?: string | string[];
  source?: string;
  filter?: Record<string, unknown>;
  description?: string;
  job_config?: { queue?: string; concurrency?: number; timeout?: number; retry?: number; dead_letter?: string };
  pattern?: Record<string, unknown>;
  [key: string]: unknown;
}

export interface InputSpec {
  fields?: Array<{ name: string; type: string; required?: boolean }>;
  validation?: string;
  description?: string;
  [key: string]: unknown;
}

export interface ProcessSpec {
  action?: string;
  service?: string;
  category?: 'security' | 'transform' | 'integration' | 'business_logic' | 'infrastructure';
  inputs?: string[];
  outputs?: string[];
  description?: string;
  [key: string]: unknown;
}

export interface DecisionSpec {
  condition?: string;
  trueLabel?: string;
  falseLabel?: string;
  description?: string;
  [key: string]: unknown;
}

export interface TerminalSpec {
  outcome?: string;
  description?: string;
  status?: number;                     // HTTP response status code
  body?: Record<string, unknown>;      // HTTP response body
  response_type?: 'json' | 'stream' | 'sse' | 'empty';
  headers?: Record<string, string>;
  [key: string]: unknown;
}

// --- Extended traditional node spec shapes ---

export interface DataStoreSpec {
  operation?: 'create' | 'read' | 'update' | 'delete' | 'upsert' | 'create_many' | 'update_many' | 'delete_many';
  model?: string;
  data?: Record<string, string>;
  query?: Record<string, string>;
  pagination?: Record<string, unknown>;  // { style, default_limit, max_limit }
  sort?: Record<string, unknown>;        // { default, allowed }
  batch?: boolean;
  upsert_key?: string[];
  include?: Array<{ relation: string; fields?: string[] }>;
  returning?: boolean;
  description?: string;
  [key: string]: unknown;
}

export interface ServiceCallSpec {
  method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  url?: string;
  headers?: Record<string, string>;
  body?: Record<string, unknown>;
  timeout_ms?: number;
  retry?: { max_attempts?: number; backoff_ms?: number; strategy?: 'fixed' | 'linear' | 'exponential'; jitter?: boolean };
  error_mapping?: Record<string, string>;
  request_config?: {
    user_agent?: 'rotate' | 'browser' | 'custom';
    delay?: { min_ms?: number; max_ms?: number; strategy?: 'random' | 'fixed' };
    cookie_jar?: 'per_domain' | 'shared' | 'none';
    proxy?: 'pool' | 'direct' | 'tor';
    tls_fingerprint?: 'randomize' | 'chrome' | 'firefox' | 'default';
    fallback?: 'headless_browser' | 'none';
  };
  integration?: string;
  description?: string;
  [key: string]: unknown;
}

export interface EventNodeSpec {
  direction?: 'emit' | 'consume';
  event_name?: string;
  payload?: Record<string, unknown>;
  payload_source?: string;
  async?: boolean;
  target_queue?: string;
  priority?: number;
  delay_ms?: number;
  dedup_key?: string;
  description?: string;
  [key: string]: unknown;
}

export interface LoopSpec {
  collection?: string;
  iterator?: string;
  break_condition?: string;
  on_error?: 'continue' | 'break' | 'fail';
  accumulate?: { field?: string; strategy?: 'append' | 'merge' | 'sum' | 'last'; output?: string };
  description?: string;
  [key: string]: unknown;
}

export interface ParallelSpec {
  branches?: (string | { label: string; condition?: string })[];
  join?: 'all' | 'any' | 'n_of';
  join_count?: number;
  timeout_ms?: number;
  description?: string;
  [key: string]: unknown;
}

export interface SubFlowSpec {
  flow_ref?: string;                    // "domain/flowId" format
  input_mapping?: Record<string, string>;
  output_mapping?: Record<string, string>;
  description?: string;
  [key: string]: unknown;
}

export interface LlmCallSpec {
  model?: string;
  system_prompt?: string;
  prompt_template?: string;
  temperature?: number;
  max_tokens?: number;
  structured_output?: Record<string, unknown>;
  context_sources?: Record<string, { from: string; transform?: string }>;
  retry?: { max_attempts?: number; backoff_ms?: number };
  description?: string;
  [key: string]: unknown;
}

// Union of all spec types (27 total)
export type NodeSpec = TriggerSpec | InputSpec | ProcessSpec | DecisionSpec | TerminalSpec
  | DataStoreSpec | ServiceCallSpec | EventNodeSpec | LoopSpec | ParallelSpec
  | SubFlowSpec | LlmCallSpec
  | DelaySpec | CacheSpec | TransformSpec
  | CollectionSpec | ParseSpec | CryptoSpec | BatchSpec | TransactionSpec
  | AgentLoopSpec | GuardrailSpec | HumanGateSpec
  | OrchestratorSpec | SmartRouterSpec | HandoffSpec | AgentGroupSpec;

// --- Flow node (persisted to YAML) ---

export interface DddFlowNode {
  id: string;
  type: DddNodeType;
  position: Position;
  connections: Array<{
    targetNodeId: string;
    sourceHandle?: string;    // e.g., "valid"/"invalid", "success"/"error", "body"/"done", "branch-0"
    targetHandle?: string;
  }>;
  spec: NodeSpec;
  label: string;
  parentId?: string;
}

// --- Flow document (YAML shape) ---

export interface FlowDocument {
  flow: { id: string; name: string; type: 'traditional' | 'agent'; domain: string; description?: string };
  trigger: DddFlowNode;
  nodes: DddFlowNode[];
  metadata: { created: string; modified: string };
}

// --- React Flow data prop ---

export interface DddNodeData extends Record<string, unknown> {
  label: string;
  spec: NodeSpec;
  dddType: DddNodeType;
  validationIssues?: ValidationIssue[];
}
```

**sourceHandle routing** — Nodes with multiple output paths use named `sourceHandle` values:

| Node Type | Output Handles | Visual Labels |
|-----------|---------------|---------------|
| `input` | `valid` / `invalid` | "Ok / Err" (green / red) |
| `decision` | `true` / `false` | "Yes / No" (green / red) |
| `data_store` | `success` / `error` | "Ok / Err" (green / red) |
| `service_call` | `success` / `error` | "Ok / Err" (green / red) |
| `loop` | `body` / `done` | "Body / Done" (teal / muted) |
| `parallel` | `branch-0`, `branch-1`, ... / `done` | Dynamic labels (pink / muted) |
| `smart_router` | Dynamic route names | Route labels (pink) |
| `cache` | `hit` / `miss` | "Hit / Miss" (amber / muted) |
| `collection` | `result` / `empty` | "Result / Empty" (cyan / muted) |
| `parse` | `success` / `error` | "Ok / Err" (lime / red) |
| `crypto` | `success` / `error` | "Ok / Err" (fuchsia / red) |
| `batch` | `done` / `error` | "Done / Err" (rose / red) |
| `transaction` | `committed` / `rolled_back` | "Ok / Rollback" (amber / red) |

**Handle aliases** — Some node types expose hidden alias handles at the same position as a visible handle. This lets external YAML use either name in `sourceHandle`. Hidden handles use `!w-0 !h-0 !bg-transparent !border-0` styling so they are invisible but connectable.

| Node Type | Visible Handles | Hidden Aliases |
|-----------|----------------|----------------|
| `guardrail` | `pass`, `block` | `valid`, `invalid` |
| `agent_loop` | `done`, `error` | (unnamed default) |
| `llm_call` | (unnamed default) | `success`, `error` |

// ─── Custom Fields ───
// All node spec interfaces support extensibility via index signatures:
//   [key: string]: unknown;
// This allows AI suggestions and user-defined fields beyond the typed schema.
// The Spec Panel renders these in a collapsible "Custom Fields" section below
// the typed fields, with add/edit/delete capabilities.
// Custom fields are persisted to YAML and survive round-trips.

// ─── Agent Node Types ───

export type AgentNodeType = 'llm_call' | 'agent_loop' | 'tool' | 'memory' | 'guardrail' | 'human_gate' | 'router';

export interface LLMCallNode extends BaseNode {
  type: 'llm_call';
  spec: {
    model: string;
    system_prompt: string;
    prompt_template: string;
    temperature?: number;
    max_tokens?: number;
    structured_output?: Record<string, any>;
    retry?: {
      max_attempts: number;
      fallback_model?: string;
    };
  };
}

export interface ToolDefinition {
  id: string;
  name: string;
  description: string;
  parameters: Record<string, {
    type: string;
    required?: boolean;
    description?: string;
    enum?: string[];
    default?: any;
  }>;
  implementation: {
    type: 'service_call' | 'data_store' | 'event' | 'sub_flow';
    [key: string]: any;
  };
  is_terminal?: boolean;
  requires_confirmation?: boolean;
}

export interface AgentLoopNode extends BaseNode {
  type: 'agent_loop';
  spec: {
    model: string;
    system_prompt: string;
    max_iterations: number;
    temperature?: number;
    stop_conditions: Array<{
      tool_called?: string;
      max_iterations_reached?: boolean;
    }>;
    tools: string[];              // refs to tool IDs
    memory?: {
      type: 'conversation' | 'vector_store' | 'key_value';
      max_tokens?: number;
      strategy?: 'sliding_window' | 'summarize' | 'truncate';
    };
    scratchpad?: boolean;
    on_max_iterations?: {
      action: 'escalate' | 'respond' | 'error';
      connection?: string;
    };
  };
}

export interface ToolNode extends BaseNode {
  type: 'tool';
  spec: ToolDefinition;
}

export interface MemoryNode extends BaseNode {
  type: 'memory';
  spec: {
    stores: Array<{
      name: string;
      type: 'conversation_history' | 'vector_store' | 'key_value';
      max_tokens?: number;
      strategy?: string;
      provider?: string;
      embedding_model?: string;
      top_k?: number;
      min_similarity?: number;
      ttl?: number;
      fields?: string[];
    }>;
  };
}

export interface GuardrailNode extends BaseNode {
  type: 'guardrail';
  spec: {
    position: 'input' | 'output';
    checks: Array<{
      type: 'content_filter' | 'pii_detection' | 'topic_restriction' | 'prompt_injection' | 'tone' | 'factuality' | 'schema_validation' | 'no_hallucinated_urls';
      action: 'block' | 'mask' | 'warn' | 'rewrite' | 'redirect' | 'log';
      [key: string]: any;
    }>;
    on_block?: {
      connection: string;
    };
  };
}

export interface HumanGateNode extends BaseNode {
  type: 'human_gate';
  spec: {
    notification: {
      channels: Array<{
        type: 'slack' | 'email' | 'webhook';
        [key: string]: any;
      }>;
    };
    approval_options: Array<{
      id: string;
      label: string;
      description: string;
      requires_input?: boolean;
    }>;
    timeout: {
      duration: number;
      action: 'auto_escalate' | 'auto_approve' | 'return_error';
      fallback_connection?: string;
    };
    context_for_human?: string[];
  };
}

export interface RouterNode extends BaseNode {
  type: 'router';
  spec: {
    model: string;
    routing_prompt: string;
    routes: Array<{
      id: string;
      description: string;
      connection: string;
    }>;
    fallback_route: string;
    confidence_threshold?: number;
  };
}

export type AgentNode = LLMCallNode | AgentLoopNode | ToolNode | MemoryNode | GuardrailNode | HumanGateNode | RouterNode;

// ─── Orchestration Node Types ───

export type OrchestrationNodeType = 'orchestrator' | 'smart_router' | 'handoff' | 'agent_group';

export interface OrchestratorAgent {
  id: string;
  flow: string;
  domain?: string;
  specialization: string;
  priority: number;
}

export interface SupervisionRule {
  condition: 'agent_iterations_exceeded' | 'confidence_below' | 'customer_sentiment' | 'agent_error' | 'timeout';
  threshold?: number;
  sentiment?: string;
  action: 'reassign' | 'add_instructions' | 'escalate_to_human' | 'retry_with_different_agent';
  target?: string;
  instructions_prompt?: string;
  max_retries?: number;
}

export interface OrchestratorNode extends BaseNode {
  type: 'orchestrator';
  spec: {
    strategy: 'supervisor' | 'round_robin' | 'broadcast' | 'consensus';
    model: string;
    supervisor_prompt: string;
    agents: OrchestratorAgent[];
    routing: {
      primary: string;               // Node ID of the Smart Router
      fallback_chain: string[];      // Agent IDs to try in order
    };
    shared_memory?: Array<{
      name: string;
      type: string;
      access: 'read_write' | 'read_only';
      fields?: string[];
    }>;
    supervision: {
      monitor_iterations: boolean;
      monitor_tool_calls?: boolean;
      monitor_confidence?: boolean;
      intervene_on: SupervisionRule[];
    };
    result_merge: {
      strategy: 'last_wins' | 'best_of' | 'combine' | 'supervisor_picks';
    };
  };
}

export interface SmartRouterRule {
  id: string;
  condition: string;                 // Expression evaluated against context
  route: string;                     // Agent ID or connection name
  priority: number;
}

export interface SmartRouterNode extends BaseNode {
  type: 'smart_router';
  spec: {
    rules: SmartRouterRule[];
    llm_routing: {
      enabled: boolean;
      model: string;
      routing_prompt: string;
      confidence_threshold: number;
      temperature?: number;
      routes: Record<string, string>;  // label → agent ID
    };
    fallback_chain: string[];
    policies: {
      retry?: {
        max_attempts: number;
        on_failure: 'next_in_fallback_chain' | 'error';
        delay_ms?: number;
      };
      timeout?: {
        per_route: number;
        total: number;
        action: 'next_in_fallback_chain' | 'error';
      };
      circuit_breaker?: {
        enabled: boolean;
        failure_threshold: number;
        recovery_time: number;
        half_open_requests?: number;
        on_open: 'next_in_fallback_chain' | 'error';
      };
    };
    ab_testing?: {
      enabled: boolean;
      experiments: Array<{
        name: string;
        route: string;
        percentage: number;
        original_route: string;
        metrics?: string[];
      }>;
    };
    context_routing?: {
      enabled: boolean;
      rules: Array<{
        condition: string;
        route: string;
        reason: string;
      }>;
    };
  };
}

export interface HandoffNode extends BaseNode {
  type: 'handoff';
  spec: {
    mode: 'transfer' | 'consult' | 'collaborate';
    target: {
      flow: string;
      domain: string;
    };
    context_transfer: {
      include: Array<{
        type: 'conversation_summary' | 'structured_data' | 'task_description';
        [key: string]: any;
      }>;
      exclude?: string[];
      max_context_tokens: number;
    };
    on_complete: {
      return_to: 'source_agent' | 'orchestrator' | 'terminal';
      merge_strategy: 'append' | 'replace' | 'summarize';
      summarize_prompt?: string;
    };
    on_failure: {
      action: 'return_with_error' | 'retry' | 'escalate';
      fallback?: string;
      timeout: number;
    };
    notify_customer?: boolean;
    notification_message?: string;
  };
}

export interface AgentGroupMember {
  flow: string;
  domain?: string;
}

export interface AgentGroupNode extends BaseNode {
  type: 'agent_group';
  spec: {
    name: string;
    description: string;
    members: AgentGroupMember[];
    shared_memory: Array<{
      name: string;
      type: string;
      access: 'read_write' | 'read_only';
      fields?: string[];
      provider?: string;
    }>;
    coordination: {
      communication: 'via_orchestrator' | 'direct' | 'blackboard';
      concurrency: {
        max_active_agents: number;
      };
      selection: {
        strategy: 'router_first' | 'round_robin' | 'least_busy' | 'random';
        sticky_session: boolean;
        sticky_timeout?: number;
      };
    };
    guardrails?: {
      input: Array<Record<string, any>>;
      output: Array<Record<string, any>>;
    };
    metrics?: {
      track: string[];
    };
  };
}

export type OrchestrationNode = OrchestratorNode | SmartRouterNode | HandoffNode | AgentGroupNode;

export type AnyNode = FlowNode | AgentNode | OrchestrationNode;
```

**Flow types** are now defined in `src/types/flow.ts` (see consolidated types section above). The `FlowDocument` interface is the YAML shape, and `DddFlowNode` is the node type with `sourceHandle` support in its `connections` array.

**`normalizeFlowDocument()` in `src/stores/flow-store.ts`** — Bridges external YAML formats (e.g., specs generated by `/ddd-create`) with the internal `FlowDocument` shape. Called during `loadFlow` before any store state is set. Normalization rules:

| External Format | Internal Format |
|-----------------|-----------------|
| `target` or `targetId` in connections | `targetNodeId` |
| `config` or `properties` on a node | `spec` |
| `sourceHandle: "default"` | `undefined` (stripped) |
| `name` on a node | `label` |
| `routes[].id` (smart_router) | `rules[].route` |
| Inlined spec fields on node (e.g., `method`, `path` at top level) | Extracted into `spec` object |
| Trigger node inside `nodes[]` array | Extracted to top-level `trigger` |
| `branches: 3` (number on parallel node) | `branches: ["Branch 0", "Branch 1", "Branch 2"]` |
| Missing node `id` | Auto-generated via `nanoid(8)` |
| Missing node `type` | Defaults to `'process'` |

This normalizer is idempotent — already-normalized documents pass through unchanged.

**File: `src/stores/flow-store.ts`**
```typescript
import { create } from 'zustand';
import { Flow, FlowNode, Position } from '../types';
import { nanoid } from 'nanoid';

interface FlowStore {
  // State
  currentFlow: Flow | null;
  selectedNodeId: string | null;
  isDirty: boolean;
  
  // Actions
  loadFlow: (flow: Flow) => void;
  createNewFlow: (name: string, domain: string) => void;
  
  // Node operations
  addNode: (type: NodeType, position: Position) => void;
  updateNode: (nodeId: string, updates: Partial<FlowNode>) => void;
  deleteNode: (nodeId: string) => void;
  moveNode: (nodeId: string, position: Position) => void;
  
  // Connection operations
  connect: (sourceId: string, outputName: string, targetId: string) => void;
  disconnect: (sourceId: string, outputName: string) => void;
  
  // Selection
  selectNode: (nodeId: string | null) => void;
  
  // Serialization
  toYaml: () => string;
  fromYaml: (yaml: string) => void;
}

export const useFlowStore = create<FlowStore>((set, get) => ({
  currentFlow: null,
  selectedNodeId: null,
  isDirty: false,
  
  loadFlow: (flow) => set({ currentFlow: flow, isDirty: false }),
  
  createNewFlow: (name, domain) => {
    const triggerId = nanoid();
    const terminalId = nanoid();
    
    const newFlow: Flow = {
      id: name.toLowerCase().replace(/\s+/g, '-'),
      name,
      domain,
      trigger: {
        id: triggerId,
        type: 'trigger',
        position: { x: 100, y: 200 },
        spec: { triggerType: 'http', method: 'POST', path: '/api/endpoint' },
        connections: { next: terminalId },
      },
      nodes: [
        {
          id: terminalId,
          type: 'terminal',
          position: { x: 500, y: 200 },
          spec: { status: 200, body: { message: 'Success' } },
          connections: {},
        },
      ],
      metadata: {
        createdAt: new Date().toISOString(),
        updatedAt: new Date().toISOString(),
        completeness: 10,
      },
    };
    
    set({ currentFlow: newFlow, isDirty: true });
  },
  
  addNode: (type, position) => {
    const { currentFlow } = get();
    if (!currentFlow) return;
    
    const newNode: FlowNode = {
      id: nanoid(),
      type,
      position,
      connections: {},
      spec: getDefaultSpec(type),
    };
    
    set({
      currentFlow: {
        ...currentFlow,
        nodes: [...currentFlow.nodes, newNode],
        metadata: { ...currentFlow.metadata, updatedAt: new Date().toISOString() },
      },
      isDirty: true,
    });
  },
  
  updateNode: (nodeId, updates) => {
    const { currentFlow } = get();
    if (!currentFlow) return;
    
    const updateNodeInList = (nodes: FlowNode[]) =>
      nodes.map(n => n.id === nodeId ? { ...n, ...updates } : n);
    
    set({
      currentFlow: {
        ...currentFlow,
        trigger: currentFlow.trigger.id === nodeId 
          ? { ...currentFlow.trigger, ...updates }
          : currentFlow.trigger,
        nodes: updateNodeInList(currentFlow.nodes),
        metadata: { ...currentFlow.metadata, updatedAt: new Date().toISOString() },
      },
      isDirty: true,
    });
  },
  
  deleteNode: (nodeId) => {
    const { currentFlow } = get();
    if (!currentFlow) return;
    if (currentFlow.trigger.id === nodeId) return; // Can't delete trigger
    
    // Remove node and any connections to it
    const nodes = currentFlow.nodes.filter(n => n.id !== nodeId);
    
    // Clean up connections pointing to deleted node
    const cleanConnections = (node: FlowNode) => ({
      ...node,
      connections: Object.fromEntries(
        Object.entries(node.connections).filter(([_, target]) => target !== nodeId)
      ),
    });
    
    set({
      currentFlow: {
        ...currentFlow,
        trigger: cleanConnections(currentFlow.trigger),
        nodes: nodes.map(cleanConnections),
      },
      isDirty: true,
      selectedNodeId: get().selectedNodeId === nodeId ? null : get().selectedNodeId,
    });
  },
  
  moveNode: (nodeId, position) => {
    get().updateNode(nodeId, { position });
  },
  
  connect: (sourceId, outputName, targetId) => {
    const { currentFlow } = get();
    if (!currentFlow) return;
    
    const updateConnections = (node: FlowNode) => {
      if (node.id !== sourceId) return node;
      return {
        ...node,
        connections: { ...node.connections, [outputName]: targetId },
      };
    };
    
    set({
      currentFlow: {
        ...currentFlow,
        trigger: updateConnections(currentFlow.trigger),
        nodes: currentFlow.nodes.map(updateConnections),
      },
      isDirty: true,
    });
  },
  
  disconnect: (sourceId, outputName) => {
    const { currentFlow } = get();
    if (!currentFlow) return;
    
    const updateConnections = (node: FlowNode) => {
      if (node.id !== sourceId) return node;
      const { [outputName]: _, ...rest } = node.connections;
      return { ...node, connections: rest };
    };
    
    set({
      currentFlow: {
        ...currentFlow,
        trigger: updateConnections(currentFlow.trigger),
        nodes: currentFlow.nodes.map(updateConnections),
      },
      isDirty: true,
    });
  },
  
  selectNode: (nodeId) => set({ selectedNodeId: nodeId }),
  
  toYaml: () => {
    const { currentFlow } = get();
    if (!currentFlow) return '';
    return flowToYaml(currentFlow);
  },
  
  fromYaml: (yamlString) => {
    const flow = yamlToFlow(yamlString);
    set({ currentFlow: flow, isDirty: false });
  },
}));

function getDefaultSpec(type: NodeType): any {
  switch (type) {
    case 'trigger':
      return { triggerType: 'http', method: 'POST', path: '/api/endpoint' };
    case 'input':
      return { fields: [] };
    case 'process':
      return { operation: '', description: '', inputs: {}, outputs: {} };
    case 'decision':
      return { condition: '', description: '', onTrue: {}, onFalse: {} };
    case 'terminal':
      return { status: 200, body: { message: 'Success' } };
  }
}
```

**File: `src/stores/sheet-store.ts`**
```typescript
import { create } from 'zustand';
import { SheetLevel, SheetLocation, BreadcrumbSegment } from '../types/sheet';

interface SheetStore {
  // State
  current: SheetLocation;
  history: SheetLocation[];   // Back-navigation stack

  // Navigation
  navigateTo: (location: SheetLocation) => void;
  navigateUp: () => void;
  navigateToSystem: () => void;
  navigateToDomain: (domainId: string) => void;
  navigateToFlow: (domainId: string, flowId: string) => void;

  // Breadcrumbs (derived)
  getBreadcrumbs: () => BreadcrumbSegment[];
}

export const useSheetStore = create<SheetStore>((set, get) => ({
  current: { level: 'system' },
  history: [],

  navigateTo: (location) => {
    const { current } = get();
    set({
      current: location,
      history: [...get().history, current],
    });
  },

  navigateUp: () => {
    const { current, history } = get();
    if (current.level === 'system') return;

    if (current.level === 'flow') {
      // Go up to domain
      set({
        current: { level: 'domain', domainId: current.domainId },
        history: [...history, current],
      });
    } else if (current.level === 'domain') {
      // Go up to system
      set({
        current: { level: 'system' },
        history: [...history, current],
      });
    }
  },

  navigateToSystem: () => {
    get().navigateTo({ level: 'system' });
  },

  navigateToDomain: (domainId) => {
    get().navigateTo({ level: 'domain', domainId });
  },

  navigateToFlow: (domainId, flowId) => {
    get().navigateTo({ level: 'flow', domainId, flowId });
  },

  getBreadcrumbs: () => {
    const { current } = get();
    const crumbs: BreadcrumbSegment[] = [
      { label: 'System', location: { level: 'system' } },
    ];

    if (current.level === 'domain' || current.level === 'flow') {
      crumbs.push({
        label: current.domainId!,
        location: { level: 'domain', domainId: current.domainId },
      });
    }

    if (current.level === 'flow') {
      crumbs.push({
        label: current.flowId!,
        location: { level: 'flow', domainId: current.domainId, flowId: current.flowId },
      });
    }

    return crumbs;
  },
}));
```

#### Day 3-4: Multi-Level Canvas & Navigation

**File: `src/App.tsx`** (sheet-level routing)
```typescript
import React from 'react';
import { useSheetStore } from './stores/sheet-store';
import { SystemMap } from './components/SystemMap/SystemMap';
import { DomainMap } from './components/DomainMap/DomainMap';
import { Canvas } from './components/Canvas/Canvas';
import { Breadcrumb } from './components/Navigation/Breadcrumb';
import { SpecPanel } from './components/SpecPanel/SpecPanel';
import { Sidebar } from './components/Sidebar/Sidebar';

export function App() {
  const { current } = useSheetStore();

  return (
    <div className="app-layout">
      <Breadcrumb />

      <div className="main-area">
        <Sidebar />

        <div className="canvas-area">
          {current.level === 'system' && <SystemMap />}
          {current.level === 'domain' && <DomainMap domainId={current.domainId!} />}
          {current.level === 'flow' && <Canvas domainId={current.domainId!} flowId={current.flowId!} />}
        </div>

        {current.level === 'flow' && <SpecPanel />}
      </div>
    </div>
  );
}
```

**File: `src/components/Navigation/Breadcrumb.tsx`**
```typescript
import React from 'react';
import { useSheetStore } from '../../stores/sheet-store';
import { ChevronRight } from 'lucide-react';

export function Breadcrumb() {
  const { current, getBreadcrumbs, navigateTo, navigateUp } = useSheetStore();
  const crumbs = getBreadcrumbs();

  // Keyboard: Backspace/Esc to go up
  React.useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (e.key === 'Backspace' || e.key === 'Escape') {
        // Only if not focused on an input/textarea
        if (
          document.activeElement?.tagName !== 'INPUT' &&
          document.activeElement?.tagName !== 'TEXTAREA'
        ) {
          e.preventDefault();
          navigateUp();
        }
      }
    };
    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [navigateUp]);

  return (
    <nav className="breadcrumb-bar flex items-center gap-1 px-4 py-2 bg-gray-900 border-b border-gray-700 text-sm">
      {crumbs.map((crumb, i) => {
        const isLast = i === crumbs.length - 1;
        return (
          <React.Fragment key={i}>
            {i > 0 && <ChevronRight size={14} className="text-gray-500" />}
            <button
              onClick={() => !isLast && navigateTo(crumb.location)}
              className={`
                px-2 py-1 rounded
                ${isLast
                  ? 'text-white font-medium cursor-default'
                  : 'text-gray-400 hover:text-white hover:bg-gray-800 cursor-pointer'
                }
              `}
              disabled={isLast}
            >
              {crumb.label}
            </button>
          </React.Fragment>
        );
      })}
    </nav>
  );
}
```

**File: `src/components/SystemMap/SystemMap.tsx`**
```typescript
import React from 'react';
import { useSheetStore } from '../../stores/sheet-store';
import { useProjectStore } from '../../stores/project-store';
import { DomainBlock } from './DomainBlock';
import { EventArrow } from './EventArrow';
import { SystemMapData } from '../../types/domain';
import { buildSystemMapData } from '../../utils/domain-parser';

export function SystemMap() {
  const { navigateToDomain } = useSheetStore();
  const { domainConfigs, systemLayout } = useProjectStore();

  const mapData: SystemMapData = React.useMemo(
    () => buildSystemMapData(domainConfigs, systemLayout),
    [domainConfigs, systemLayout]
  );

  return (
    <div className="system-map relative w-full h-full overflow-auto">
      <svg className="absolute inset-0 w-full h-full pointer-events-none">
        {mapData.eventArrows.map((arrow, i) => (
          <EventArrow
            key={`${arrow.sourceDomain}-${arrow.targetDomain}-${i}`}
            sourcePos={mapData.domains.find(d => d.id === arrow.sourceDomain)!.position}
            targetPos={mapData.domains.find(d => d.id === arrow.targetDomain)!.position}
            events={arrow.events}
          />
        ))}
      </svg>

      {mapData.domains.map(domain => (
        <DomainBlock
          key={domain.id}
          domain={domain}
          onDoubleClick={() => navigateToDomain(domain.id)}
        />
      ))}
    </div>
  );
}
```

**File: `src/components/SystemMap/DomainBlock.tsx`**
```typescript
import React from 'react';
import { SystemMapDomain } from '../../types/domain';
import { Layers } from 'lucide-react';

interface DomainBlockProps {
  domain: SystemMapDomain;
  onDoubleClick: () => void;
}

export function DomainBlock({ domain, onDoubleClick }: DomainBlockProps) {
  return (
    <div
      className="domain-block absolute cursor-pointer select-none"
      style={{
        left: domain.position.x,
        top: domain.position.y,
        transform: 'translate(-50%, -50%)',
      }}
      onDoubleClick={onDoubleClick}
    >
      <div className="bg-gray-800 border-2 border-blue-500 rounded-xl px-6 py-4 shadow-lg hover:border-blue-400 transition-colors min-w-[180px]">
        <div className="flex items-center gap-2 mb-1">
          <Layers size={18} className="text-blue-400" />
          <span className="font-semibold text-white">{domain.name}</span>
        </div>
        <p className="text-gray-400 text-xs mb-2">{domain.description}</p>
        <span className="text-xs text-gray-500">{domain.flowCount} flows</span>
      </div>
    </div>
  );
}
```

**File: `src/components/SystemMap/DomainContextMenu.tsx`**
```typescript
import React from 'react';
import * as ContextMenu from '@radix-ui/react-context-menu';
import { Plus, Edit2, Trash2, FileText } from 'lucide-react';
import { useProjectStore } from '../../stores/project-store';

interface DomainContextMenuProps {
  domainId: string;
  children: React.ReactNode;
}

export function DomainContextMenu({ domainId, children }: DomainContextMenuProps) {
  const { renameDomain, deleteDomain, editDomainDescription, addDomainEvent } =
    useProjectStore();

  const [renaming, setRenaming] = React.useState(false);
  const [newName, setNewName] = React.useState('');

  const handleRename = () => {
    if (newName.trim()) {
      renameDomain(domainId, newName.trim());
      setRenaming(false);
    }
  };

  const handleDelete = () => {
    const confirmed = window.confirm(
      `Delete domain "${domainId}" and all its flows? This cannot be undone.`
    );
    if (confirmed) deleteDomain(domainId);
  };

  return (
    <ContextMenu.Root>
      <ContextMenu.Trigger asChild>{children}</ContextMenu.Trigger>
      <ContextMenu.Portal>
        <ContextMenu.Content className="bg-gray-800 border border-gray-600 rounded-lg shadow-xl p-1 min-w-[180px] z-50">
          <ContextMenu.Item
            className="flex items-center gap-2 px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
            onSelect={() => setRenaming(true)}
          >
            <Edit2 size={14} /> Rename
          </ContextMenu.Item>
          <ContextMenu.Item
            className="flex items-center gap-2 px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
            onSelect={() => editDomainDescription(domainId)}
          >
            <FileText size={14} /> Edit description
          </ContextMenu.Item>

          <ContextMenu.Separator className="h-px bg-gray-600 my-1" />

          <ContextMenu.Sub>
            <ContextMenu.SubTrigger className="flex items-center gap-2 px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer">
              <Plus size={14} /> Add event
            </ContextMenu.SubTrigger>
            <ContextMenu.SubContent className="bg-gray-800 border border-gray-600 rounded-lg shadow-xl p-1 min-w-[160px]">
              <ContextMenu.Item
                className="px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
                onSelect={() => addDomainEvent(domainId, 'publish')}
              >
                Add published event
              </ContextMenu.Item>
              <ContextMenu.Item
                className="px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
                onSelect={() => addDomainEvent(domainId, 'consume')}
              >
                Add consumed event
              </ContextMenu.Item>
            </ContextMenu.SubContent>
          </ContextMenu.Sub>

          <ContextMenu.Separator className="h-px bg-gray-600 my-1" />

          <ContextMenu.Item
            className="flex items-center gap-2 px-3 py-2 text-sm text-red-400 hover:bg-red-900/30 rounded cursor-pointer"
            onSelect={handleDelete}
          >
            <Trash2 size={14} /> Delete domain
          </ContextMenu.Item>
        </ContextMenu.Content>
      </ContextMenu.Portal>
    </ContextMenu.Root>
  );
}
```

**File: `src/components/SystemMap/CanvasContextMenu.tsx`** (right-click on L1 canvas background)
```typescript
import React, { useState } from 'react';
import * as ContextMenu from '@radix-ui/react-context-menu';
import * as Dialog from '@radix-ui/react-dialog';
import { Plus } from 'lucide-react';
import { useProjectStore } from '../../stores/project-store';

interface CanvasContextMenuProps {
  children: React.ReactNode;
}

export function L1CanvasContextMenu({ children }: CanvasContextMenuProps) {
  const { createDomain } = useProjectStore();
  const [showDialog, setShowDialog] = useState(false);
  const [name, setName] = useState('');
  const [description, setDescription] = useState('');

  const handleCreate = async () => {
    if (!name.trim()) return;
    await createDomain({ name: name.trim(), description: description.trim() });
    setShowDialog(false);
    setName('');
    setDescription('');
  };

  return (
    <>
      <ContextMenu.Root>
        <ContextMenu.Trigger asChild>{children}</ContextMenu.Trigger>
        <ContextMenu.Portal>
          <ContextMenu.Content className="bg-gray-800 border border-gray-600 rounded-lg shadow-xl p-1 min-w-[160px] z-50">
            <ContextMenu.Item
              className="flex items-center gap-2 px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
              onSelect={() => setShowDialog(true)}
            >
              <Plus size={14} /> Add domain
            </ContextMenu.Item>
          </ContextMenu.Content>
        </ContextMenu.Portal>
      </ContextMenu.Root>

      <Dialog.Root open={showDialog} onOpenChange={setShowDialog}>
        <Dialog.Portal>
          <Dialog.Overlay className="fixed inset-0 bg-black/50 z-40" />
          <Dialog.Content className="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-gray-800 border border-gray-600 rounded-xl p-6 w-[400px] z-50">
            <Dialog.Title className="text-lg font-semibold text-white mb-4">
              Add Domain
            </Dialog.Title>
            <div className="space-y-3">
              <input
                className="w-full bg-gray-700 border border-gray-600 rounded px-3 py-2 text-white"
                placeholder="Domain name (e.g. users)"
                value={name}
                onChange={e => setName(e.target.value)}
                autoFocus
              />
              <textarea
                className="w-full bg-gray-700 border border-gray-600 rounded px-3 py-2 text-white resize-none"
                placeholder="Description"
                rows={2}
                value={description}
                onChange={e => setDescription(e.target.value)}
              />
            </div>
            <div className="flex justify-end gap-2 mt-4">
              <button
                className="px-4 py-2 text-sm text-gray-400 hover:text-white"
                onClick={() => setShowDialog(false)}
              >
                Cancel
              </button>
              <button
                className="px-4 py-2 text-sm bg-blue-600 text-white rounded hover:bg-blue-500"
                onClick={handleCreate}
              >
                Create
              </button>
            </div>
          </Dialog.Content>
        </Dialog.Portal>
      </Dialog.Root>
    </>
  );
}
```

**File: `src/components/DomainMap/FlowContextMenu.tsx`**
```typescript
import React, { useState } from 'react';
import * as ContextMenu from '@radix-ui/react-context-menu';
import { Edit2, Trash2, Copy, ArrowRightLeft, RefreshCw } from 'lucide-react';
import { useProjectStore } from '../../stores/project-store';
import { FlowType } from '../../types/domain';

interface FlowContextMenuProps {
  domainId: string;
  flowId: string;
  flowName: string;
  children: React.ReactNode;
}

export function FlowContextMenu({
  domainId,
  flowId,
  flowName,
  children,
}: FlowContextMenuProps) {
  const {
    renameFlow,
    deleteFlow,
    duplicateFlow,
    moveFlow,
    changeFlowType,
    domainConfigs,
  } = useProjectStore();

  const otherDomains = Object.keys(domainConfigs).filter(d => d !== domainId);

  const handleDelete = () => {
    const confirmed = window.confirm(
      `Delete flow "${flowName}"? This cannot be undone.`
    );
    if (confirmed) deleteFlow(domainId, flowId);
  };

  return (
    <ContextMenu.Root>
      <ContextMenu.Trigger asChild>{children}</ContextMenu.Trigger>
      <ContextMenu.Portal>
        <ContextMenu.Content className="bg-gray-800 border border-gray-600 rounded-lg shadow-xl p-1 min-w-[180px] z-50">
          <ContextMenu.Item
            className="flex items-center gap-2 px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
            onSelect={() => renameFlow(domainId, flowId)}
          >
            <Edit2 size={14} /> Rename
          </ContextMenu.Item>
          <ContextMenu.Item
            className="flex items-center gap-2 px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
            onSelect={() => duplicateFlow(domainId, flowId)}
          >
            <Copy size={14} /> Duplicate
          </ContextMenu.Item>

          {otherDomains.length > 0 && (
            <ContextMenu.Sub>
              <ContextMenu.SubTrigger className="flex items-center gap-2 px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer">
                <ArrowRightLeft size={14} /> Move to...
              </ContextMenu.SubTrigger>
              <ContextMenu.SubContent className="bg-gray-800 border border-gray-600 rounded-lg shadow-xl p-1 min-w-[140px]">
                {otherDomains.map(d => (
                  <ContextMenu.Item
                    key={d}
                    className="px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
                    onSelect={() => moveFlow(domainId, d, flowId)}
                  >
                    {d}
                  </ContextMenu.Item>
                ))}
              </ContextMenu.SubContent>
            </ContextMenu.Sub>
          )}

          <ContextMenu.Sub>
            <ContextMenu.SubTrigger className="flex items-center gap-2 px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer">
              <RefreshCw size={14} /> Change type
            </ContextMenu.SubTrigger>
            <ContextMenu.SubContent className="bg-gray-800 border border-gray-600 rounded-lg shadow-xl p-1 min-w-[140px]">
              {(['traditional', 'agent', 'orchestration'] as FlowType[]).map(t => (
                <ContextMenu.Item
                  key={t}
                  className="px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
                  onSelect={() => changeFlowType(domainId, flowId, t)}
                >
                  {t.charAt(0).toUpperCase() + t.slice(1)}
                </ContextMenu.Item>
              ))}
            </ContextMenu.SubContent>
          </ContextMenu.Sub>

          <ContextMenu.Separator className="h-px bg-gray-600 my-1" />

          <ContextMenu.Item
            className="flex items-center gap-2 px-3 py-2 text-sm text-red-400 hover:bg-red-900/30 rounded cursor-pointer"
            onSelect={handleDelete}
          >
            <Trash2 size={14} /> Delete flow
          </ContextMenu.Item>
        </ContextMenu.Content>
      </ContextMenu.Portal>
    </ContextMenu.Root>
  );
}
```

**File: `src/components/DomainMap/L2CanvasContextMenu.tsx`** (right-click on L2 canvas background)
```typescript
import React, { useState } from 'react';
import * as ContextMenu from '@radix-ui/react-context-menu';
import * as Dialog from '@radix-ui/react-dialog';
import { Plus } from 'lucide-react';
import { useProjectStore } from '../../stores/project-store';
import { useSheetStore } from '../../stores/sheet-store';
import { FlowType } from '../../types/domain';

export function L2CanvasContextMenu({ children }: { children: React.ReactNode }) {
  const { createFlow } = useProjectStore();
  const domainId = useSheetStore(s => s.currentDomain);
  const [showDialog, setShowDialog] = useState(false);
  const [name, setName] = useState('');
  const [flowType, setFlowType] = useState<FlowType>('traditional');

  const handleCreate = async () => {
    if (!name.trim() || !domainId) return;
    await createFlow({ domainId, name: name.trim(), flowType });
    setShowDialog(false);
    setName('');
    setFlowType('traditional');
  };

  return (
    <>
      <ContextMenu.Root>
        <ContextMenu.Trigger asChild>{children}</ContextMenu.Trigger>
        <ContextMenu.Portal>
          <ContextMenu.Content className="bg-gray-800 border border-gray-600 rounded-lg shadow-xl p-1 min-w-[160px] z-50">
            <ContextMenu.Item
              className="flex items-center gap-2 px-3 py-2 text-sm text-gray-200 hover:bg-gray-700 rounded cursor-pointer"
              onSelect={() => setShowDialog(true)}
            >
              <Plus size={14} /> Add flow
            </ContextMenu.Item>
          </ContextMenu.Content>
        </ContextMenu.Portal>
      </ContextMenu.Root>

      <Dialog.Root open={showDialog} onOpenChange={setShowDialog}>
        <Dialog.Portal>
          <Dialog.Overlay className="fixed inset-0 bg-black/50 z-40" />
          <Dialog.Content className="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-gray-800 border border-gray-600 rounded-xl p-6 w-[400px] z-50">
            <Dialog.Title className="text-lg font-semibold text-white mb-4">
              Add Flow
            </Dialog.Title>
            <div className="space-y-3">
              <input
                className="w-full bg-gray-700 border border-gray-600 rounded px-3 py-2 text-white"
                placeholder="Flow name (e.g. user-register)"
                value={name}
                onChange={e => setName(e.target.value)}
                autoFocus
              />
              <div className="flex gap-2">
                {(['traditional', 'agent', 'orchestration'] as FlowType[]).map(t => (
                  <button
                    key={t}
                    className={`px-3 py-1.5 text-xs rounded border ${
                      flowType === t
                        ? 'bg-blue-600 border-blue-500 text-white'
                        : 'bg-gray-700 border-gray-600 text-gray-300 hover:border-gray-500'
                    }`}
                    onClick={() => setFlowType(t)}
                  >
                    {t.charAt(0).toUpperCase() + t.slice(1)}
                  </button>
                ))}
              </div>
            </div>
            <div className="flex justify-end gap-2 mt-4">
              <button
                className="px-4 py-2 text-sm text-gray-400 hover:text-white"
                onClick={() => setShowDialog(false)}
              >
                Cancel
              </button>
              <button
                className="px-4 py-2 text-sm bg-blue-600 text-white rounded hover:bg-blue-500"
                onClick={handleCreate}
              >
                Create
              </button>
            </div>
          </Dialog.Content>
        </Dialog.Portal>
      </Dialog.Root>
    </>
  );
}
```

**Wrap existing DomainBlock in DomainContextMenu — update `SystemMap.tsx`:**
```typescript
// In SystemMap.tsx render, wrap each DomainBlock:
import { DomainContextMenu } from './DomainContextMenu';
import { L1CanvasContextMenu } from './CanvasContextMenu';

// Wrap the canvas div:
<L1CanvasContextMenu>
  <div className="system-map relative w-full h-full overflow-auto">
    ...
    {mapData.domains.map(domain => (
      <DomainContextMenu key={domain.id} domainId={domain.id}>
        <DomainBlock
          domain={domain}
          onDoubleClick={() => navigateToDomain(domain.id)}
        />
      </DomainContextMenu>
    ))}
  </div>
</L1CanvasContextMenu>
```

**Wrap existing FlowBlock in FlowContextMenu — update `DomainMap.tsx`:**
```typescript
// In DomainMap.tsx render, wrap each FlowBlock:
import { FlowContextMenu } from './FlowContextMenu';
import { L2CanvasContextMenu } from './L2CanvasContextMenu';

// Wrap the canvas div:
<L2CanvasContextMenu>
  <div className="domain-map relative w-full h-full overflow-auto">
    ...
    {mapData.flows.map(flow => (
      <FlowContextMenu
        key={flow.id}
        domainId={domainId}
        flowId={flow.id}
        flowName={flow.name}
      >
        <FlowBlock
          flow={flow}
          onDoubleClick={() => navigateToFlow(domainId, flow.id)}
        />
      </FlowContextMenu>
    ))}
  </div>
</L2CanvasContextMenu>
```

**Entity management actions — add to `src/stores/project-store.ts`:**
```typescript
import { invoke } from '@tauri-apps/api/core';
import {
  CreateDomainPayload,
  RenameDomainPayload,
  CreateFlowPayload,
  RenameFlowPayload,
  MoveFlowPayload,
  FlowType,
} from '../types/domain';

// Add these actions to the project store:

createDomain: async (payload: CreateDomainPayload) => {
  const { projectPath } = get();
  await invoke('create_domain', {
    projectPath,
    name: payload.name,
    description: payload.description,
  });
  // Update system.yaml
  const systemYaml = get().systemConfig;
  systemYaml.domains.push({ name: payload.name, description: payload.description });
  await invoke('write_file', {
    path: `${projectPath}/specs/system.yaml`,
    content: YAML.stringify(systemYaml),
  });
  // Reload project to refresh domain configs
  await get().loadProject(projectPath);
},

renameDomain: async (oldName: string, newName: string) => {
  const { projectPath } = get();
  await invoke('rename_domain', { projectPath, oldName, newName });
  // Update system.yaml
  const systemYaml = get().systemConfig;
  const domainEntry = systemYaml.domains.find((d: any) => d.name === oldName);
  if (domainEntry) domainEntry.name = newName;
  await invoke('write_file', {
    path: `${projectPath}/specs/system.yaml`,
    content: YAML.stringify(systemYaml),
  });
  // Update cross-references in other domain.yaml files
  await get().updateCrossReferences(oldName, newName);
  await get().loadProject(projectPath);
},

deleteDomain: async (name: string) => {
  const { projectPath } = get();
  await invoke('delete_domain', { projectPath, name });
  // Remove from system.yaml
  const systemYaml = get().systemConfig;
  systemYaml.domains = systemYaml.domains.filter((d: any) => d.name !== name);
  await invoke('write_file', {
    path: `${projectPath}/specs/system.yaml`,
    content: YAML.stringify(systemYaml),
  });
  // Clean up event/portal references in other domains
  await get().removeOrphanedReferences(name);
  await get().loadProject(projectPath);
},

editDomainDescription: async (domainId: string) => {
  // Opens inline edit — actual UI handled by DomainContextMenu dialog
  // Writes updated description to domain.yaml
  const { projectPath, domainConfigs } = get();
  const config = domainConfigs[domainId];
  // ... prompt for new description, update domain.yaml
},

addDomainEvent: async (domainId: string, type: 'publish' | 'consume') => {
  // Opens dialog for event name + payload
  // Appends to publishes_events or consumes_events in domain.yaml
},

createFlow: async (payload: CreateFlowPayload) => {
  const { projectPath } = get();
  const flowId = await invoke<string>('create_flow', {
    projectPath,
    domain: payload.domainId,
    name: payload.name,
    flowType: payload.flowType,
  });
  await get().loadProject(projectPath);
  return flowId;
},

renameFlow: async (domainId: string, flowId: string) => {
  // Opens inline rename — actual name entry handled by UI
  // Calls invoke('rename_flow', ...) then reloads
},

deleteFlow: async (domainId: string, flowId: string) => {
  const { projectPath } = get();
  await invoke('delete_flow', { projectPath, domain: domainId, flowId });
  await get().loadProject(projectPath);
},

duplicateFlow: async (domainId: string, flowId: string) => {
  const { projectPath } = get();
  await invoke('duplicate_flow', {
    projectPath,
    domain: domainId,
    flowId,
    newName: `${flowId}-copy`,
  });
  await get().loadProject(projectPath);
},

moveFlow: async (sourceDomain: string, targetDomain: string, flowId: string) => {
  const { projectPath } = get();
  await invoke('move_flow', { projectPath, sourceDomain, targetDomain, flowId });
  await get().loadProject(projectPath);
},

changeFlowType: async (domainId: string, flowId: string, newType: FlowType) => {
  // Read existing flow, update type field, warn if agent/orch nodes will be lost
  const { projectPath } = get();
  const flowPath = `${projectPath}/specs/domains/${domainId}/flows/${flowId}.yaml`;
  const content = await invoke<string>('read_file', { path: flowPath });
  const flowYaml = YAML.parse(content);
  flowYaml.flow.type = newType;
  // Remove type-specific sections if switching away
  if (newType !== 'agent') delete flowYaml.agent_loop;
  if (newType !== 'orchestration') delete flowYaml.orchestrator;
  await invoke('write_file', { path: flowPath, content: YAML.stringify(flowYaml) });
  await get().loadProject(projectPath);
},

// Helper: update cross-domain references when a domain is renamed
updateCrossReferences: async (oldName: string, newName: string) => {
  const { projectPath, domainConfigs } = get();
  for (const [id, config] of Object.entries(domainConfigs)) {
    if (id === newName) continue; // Skip the renamed domain itself
    const domainYamlPath = `${projectPath}/specs/domains/${id}/domain.yaml`;
    let content = await invoke<string>('read_file', { path: domainYamlPath });
    if (content.includes(oldName)) {
      content = content.replaceAll(oldName, newName);
      await invoke('write_file', { path: domainYamlPath, content });
    }
  }
},

// Helper: remove orphaned references when a domain is deleted
removeOrphanedReferences: async (deletedDomain: string) => {
  const { projectPath, domainConfigs } = get();
  for (const [id, config] of Object.entries(domainConfigs)) {
    if (id === deletedDomain) continue;
    const domainYamlPath = `${projectPath}/specs/domains/${id}/domain.yaml`;
    let content = await invoke<string>('read_file', { path: domainYamlPath });
    // Remove event references pointing to deleted domain
    // This is a simplified version — production should parse YAML properly
    if (content.includes(deletedDomain)) {
      const yaml = YAML.parse(content);
      if (yaml.consumes_events) {
        yaml.consumes_events = yaml.consumes_events.filter(
          (e: any) => !e.from_domain || e.from_domain !== deletedDomain
        );
      }
      await invoke('write_file', { path: domainYamlPath, content: YAML.stringify(yaml) });
    }
  }
},
```

**File: `src/components/DomainMap/DomainMap.tsx`**
```typescript
import React from 'react';
import { useSheetStore } from '../../stores/sheet-store';
import { useProjectStore } from '../../stores/project-store';
import { FlowBlock } from './FlowBlock';
import { PortalNode } from './PortalNode';
import { EventArrow } from '../SystemMap/EventArrow';
import { DomainMapData } from '../../types/domain';
import { buildDomainMapData } from '../../utils/domain-parser';

interface DomainMapProps {
  domainId: string;
}

export function DomainMap({ domainId }: DomainMapProps) {
  const { navigateToFlow, navigateToDomain } = useSheetStore();
  const { domainConfigs } = useProjectStore();

  const domainConfig = domainConfigs[domainId];
  const mapData: DomainMapData = React.useMemo(
    () => buildDomainMapData(domainConfig, domainConfigs),
    [domainConfig, domainConfigs]
  );

  return (
    <div className="domain-map relative w-full h-full overflow-auto">
      <svg className="absolute inset-0 w-full h-full pointer-events-none">
        {mapData.eventArrows.map((arrow, i) => {
          const sourcePos = mapData.flows.find(f => f.id === arrow.sourceFlowId)?.position;
          const targetPos = arrow.targetFlowId
            ? mapData.flows.find(f => f.id === arrow.targetFlowId)?.position
            : mapData.portals.find(p => p.targetDomain === arrow.targetPortal)?.position;
          if (!sourcePos || !targetPos) return null;
          return (
            <EventArrow
              key={`${arrow.sourceFlowId}-${arrow.event}-${i}`}
              sourcePos={sourcePos}
              targetPos={targetPos}
              events={[arrow.event]}
            />
          );
        })}
      </svg>

      {mapData.flows.map(flow => (
        <FlowBlock
          key={flow.id}
          flow={flow}
          onDoubleClick={() => navigateToFlow(domainId, flow.id)}
        />
      ))}

      {mapData.portals.map(portal => (
        <PortalNode
          key={portal.targetDomain}
          portal={portal}
          onDoubleClick={() => navigateToDomain(portal.targetDomain)}
        />
      ))}
    </div>
  );
}
```

**File: `src/components/DomainMap/PortalNode.tsx`**
```typescript
import React from 'react';
import { DomainMapPortal } from '../../types/domain';
import { ExternalLink } from 'lucide-react';

interface PortalNodeProps {
  portal: DomainMapPortal;
  onDoubleClick: () => void;
}

export function PortalNode({ portal, onDoubleClick }: PortalNodeProps) {
  return (
    <div
      className="portal-node absolute cursor-pointer select-none"
      style={{
        left: portal.position.x,
        top: portal.position.y,
        transform: 'translate(-50%, -50%)',
      }}
      onDoubleClick={onDoubleClick}
    >
      <div className="bg-gray-900 border-2 border-dashed border-purple-500 rounded-lg px-4 py-3 shadow-lg hover:border-purple-400 transition-colors">
        <div className="flex items-center gap-2">
          <ExternalLink size={16} className="text-purple-400" />
          <span className="font-medium text-purple-300">{portal.targetDomain}</span>
        </div>
        <p className="text-gray-500 text-xs mt-1">
          {portal.events.join(', ')}
        </p>
      </div>
    </div>
  );
}
```

**File: `src/utils/domain-parser.ts`**
```typescript
import { DomainConfig, SystemLayout, SystemMapData, SystemMapArrow, DomainMapData, DomainMapArrow, DomainMapPortal } from '../types/domain';

/**
 * Build Level 1 (System Map) data from domain configs.
 * Derives inter-domain event arrows by matching publishes_events → consumes_events.
 */
export function buildSystemMapData(
  domainConfigs: Record<string, DomainConfig>,
  systemLayout: SystemLayout
): SystemMapData {
  const domains = Object.entries(domainConfigs).map(([id, config]) => ({
    id,
    name: config.name,
    description: config.description,
    flowCount: config.flows.length,
    position: systemLayout.domains[id] || { x: 0, y: 0 },
  }));

  // Build event arrows: for each published event, find which domain consumes it
  const arrowMap = new Map<string, string[]>(); // "source->target" → event names

  for (const [sourceId, sourceConfig] of Object.entries(domainConfigs)) {
    for (const pub of sourceConfig.publishes_events) {
      for (const [targetId, targetConfig] of Object.entries(domainConfigs)) {
        if (targetId === sourceId) continue;
        const consumed = targetConfig.consumes_events.find(c => c.event === pub.event);
        if (consumed) {
          const key = `${sourceId}->${targetId}`;
          const existing = arrowMap.get(key) || [];
          existing.push(pub.event);
          arrowMap.set(key, existing);
        }
      }
    }
  }

  const eventArrows: SystemMapArrow[] = Array.from(arrowMap.entries()).map(([key, events]) => {
    const [sourceDomain, targetDomain] = key.split('->');
    return { sourceDomain, targetDomain, events };
  });

  return { domains, eventArrows };
}

/**
 * Build Level 2 (Domain Map) data from a single domain config.
 * Shows flows, portals to other domains, and event arrows.
 */
export function buildDomainMapData(
  domainConfig: DomainConfig,
  allDomainConfigs: Record<string, DomainConfig>
): DomainMapData {
  const flows = domainConfig.flows.map(f => ({
    id: f.id,
    name: f.name,
    description: f.description,
    position: domainConfig.layout.flows[f.id] || { x: 0, y: 0 },
  }));

  // Build portals: other domains that consume our events or publish events we consume
  const portalDomains = new Map<string, string[]>();

  for (const pub of domainConfig.publishes_events) {
    for (const [targetId, targetConfig] of Object.entries(allDomainConfigs)) {
      if (targetId === domainConfig.name) continue;
      const consumed = targetConfig.consumes_events.find(c => c.event === pub.event);
      if (consumed) {
        const events = portalDomains.get(targetId) || [];
        events.push(pub.event);
        portalDomains.set(targetId, events);
      }
    }
  }

  const portals: DomainMapPortal[] = Array.from(portalDomains.entries()).map(([targetDomain, events]) => ({
    targetDomain,
    position: domainConfig.layout.portals[targetDomain] || { x: 0, y: 0 },
    events,
  }));

  // Build arrows from flows to portals (or other flows in same domain)
  const eventArrows: DomainMapArrow[] = domainConfig.publishes_events.map(pub => {
    // Does this event go to a portal or another flow in the same domain?
    const internalConsumer = domainConfig.consumes_events.find(c => c.event === pub.event);
    if (internalConsumer) {
      return {
        sourceFlowId: pub.from_flow!,
        targetFlowId: internalConsumer.handled_by_flow,
        event: pub.event,
      };
    }
    // Goes to a portal
    const targetDomain = Array.from(portalDomains.entries())
      .find(([_, events]) => events.includes(pub.event))?.[0];
    return {
      sourceFlowId: pub.from_flow!,
      targetPortal: targetDomain,
      event: pub.event,
    };
  });

  return {
    domainId: domainConfig.name,
    flows,
    portals,
    eventArrows,
  };
}
```

#### Day 5-6: Flow Canvas (Level 3)

**File: `src/components/Canvas/Canvas.tsx`**
```typescript
import React, { useCallback, useRef, useState } from 'react';
import { useFlowStore } from '../../stores/flow-store';
import { Node } from './Node';
import { Connection } from './Connection';
import { Position, NodeType } from '../../types';

export function Canvas() {
  const { currentFlow, addNode, moveNode, selectNode, selectedNodeId } = useFlowStore();
  const canvasRef = useRef<HTMLDivElement>(null);
  const [isPanning, setIsPanning] = useState(false);
  const [panOffset, setPanOffset] = useState({ x: 0, y: 0 });
  const [zoom, setZoom] = useState(1);
  const [draggedNode, setDraggedNode] = useState<string | null>(null);
  
  const handleCanvasClick = useCallback((e: React.MouseEvent) => {
    if (e.target === canvasRef.current) {
      selectNode(null);
    }
  }, [selectNode]);
  
  const handleDrop = useCallback((e: React.DragEvent) => {
    e.preventDefault();
    const nodeType = e.dataTransfer.getData('nodeType') as NodeType;
    if (!nodeType) return;
    
    const rect = canvasRef.current?.getBoundingClientRect();
    if (!rect) return;
    
    const position: Position = {
      x: (e.clientX - rect.left - panOffset.x) / zoom,
      y: (e.clientY - rect.top - panOffset.y) / zoom,
    };
    
    addNode(nodeType, position);
  }, [addNode, panOffset, zoom]);
  
  const handleDragOver = useCallback((e: React.DragEvent) => {
    e.preventDefault();
  }, []);
  
  const handleWheel = useCallback((e: React.WheelEvent) => {
    if (e.ctrlKey || e.metaKey) {
      e.preventDefault();
      const delta = e.deltaY > 0 ? 0.9 : 1.1;
      setZoom(z => Math.min(Math.max(z * delta, 0.25), 2));
    }
  }, []);
  
  if (!currentFlow) {
    return (
      <div className="canvas-empty">
        <p>No flow loaded</p>
        <p>Create a new flow or open an existing one</p>
      </div>
    );
  }
  
  const allNodes = [currentFlow.trigger, ...currentFlow.nodes];
  
  // Collect all connections
  const connections: Array<{
    sourceId: string;
    targetId: string;
    sourcePos: Position;
    targetPos: Position;
    label?: string;
  }> = [];
  
  for (const node of allNodes) {
    for (const [outputName, targetId] of Object.entries(node.connections)) {
      const targetNode = allNodes.find(n => n.id === targetId);
      if (targetNode) {
        connections.push({
          sourceId: node.id,
          targetId,
          sourcePos: node.position,
          targetPos: targetNode.position,
          label: outputName !== 'next' ? outputName : undefined,
        });
      }
    }
  }
  
  return (
    <div
      ref={canvasRef}
      className="canvas"
      onClick={handleCanvasClick}
      onDrop={handleDrop}
      onDragOver={handleDragOver}
      onWheel={handleWheel}
      style={{
        transform: `translate(${panOffset.x}px, ${panOffset.y}px) scale(${zoom})`,
        transformOrigin: '0 0',
      }}
    >
      {/* Connections layer */}
      <svg className="connections-layer">
        {connections.map((conn, i) => (
          <Connection
            key={`${conn.sourceId}-${conn.targetId}-${i}`}
            sourcePos={conn.sourcePos}
            targetPos={conn.targetPos}
            label={conn.label}
          />
        ))}
      </svg>
      
      {/* Nodes layer */}
      {allNodes.map(node => (
        <Node
          key={node.id}
          node={node}
          isSelected={selectedNodeId === node.id}
          onSelect={() => selectNode(node.id)}
          onMove={(pos) => moveNode(node.id, pos)}
        />
      ))}
    </div>
  );
}
```

**File: `src/components/Canvas/Node.tsx`**
```typescript
import React, { useState } from 'react';
import { FlowNode, Position } from '../../types';
import { 
  Hexagon, 
  Square, 
  Diamond, 
  Circle,
  Database 
} from 'lucide-react';

interface NodeProps {
  node: FlowNode;
  isSelected: boolean;
  onSelect: () => void;
  onMove: (position: Position) => void;
}

const nodeIcons: Record<string, React.ReactNode> = {
  trigger: <Hexagon size={20} />,
  input: <Square size={20} style={{ transform: 'skewX(-10deg)' }} />,
  process: <Square size={20} />,
  decision: <Diamond size={20} />,
  terminal: <Circle size={20} />,
};

const nodeColors: Record<string, string> = {
  trigger: 'bg-blue-500',
  input: 'bg-green-500',
  process: 'bg-gray-500',
  decision: 'bg-yellow-500',
  terminal: 'bg-red-500',
};

export function Node({ node, isSelected, onSelect, onMove }: NodeProps) {
  const [isDragging, setIsDragging] = useState(false);
  const [dragStart, setDragStart] = useState<Position | null>(null);
  
  const handleMouseDown = (e: React.MouseEvent) => {
    e.stopPropagation();
    setIsDragging(true);
    setDragStart({ x: e.clientX - node.position.x, y: e.clientY - node.position.y });
    onSelect();
  };
  
  const handleMouseMove = (e: MouseEvent) => {
    if (!isDragging || !dragStart) return;
    onMove({
      x: e.clientX - dragStart.x,
      y: e.clientY - dragStart.y,
    });
  };
  
  const handleMouseUp = () => {
    setIsDragging(false);
    setDragStart(null);
  };
  
  React.useEffect(() => {
    if (isDragging) {
      window.addEventListener('mousemove', handleMouseMove);
      window.addEventListener('mouseup', handleMouseUp);
      return () => {
        window.removeEventListener('mousemove', handleMouseMove);
        window.removeEventListener('mouseup', handleMouseUp);
      };
    }
  }, [isDragging, dragStart]);
  
  return (
    <div
      className={`
        node absolute cursor-move select-none
        ${isSelected ? 'ring-2 ring-blue-400' : ''}
        ${isDragging ? 'opacity-75' : ''}
      `}
      style={{
        left: node.position.x,
        top: node.position.y,
        transform: 'translate(-50%, -50%)',
      }}
      onMouseDown={handleMouseDown}
    >
      <div className={`
        flex items-center gap-2 px-4 py-2 rounded-lg
        ${nodeColors[node.type]} text-white
        shadow-lg
      `}>
        {nodeIcons[node.type]}
        <span className="font-medium">{node.id}</span>
      </div>
      
      {/* Connection points */}
      <div className="connection-point input" />
      <div className="connection-point output" />
      
      {/* Extra outputs for decision nodes */}
      {node.type === 'decision' && (
        <>
          <div className="connection-point output-true" title="true" />
          <div className="connection-point output-false" title="false" />
        </>
      )}
    </div>
  );
}
```

#### Day 5-7: Spec Panel

**File: `src/components/SpecPanel/SpecPanel.tsx`**
```typescript
import React from 'react';
import { useFlowStore } from '../../stores/flow-store';
import { TriggerSpec } from './TriggerSpec';
import { InputSpec } from './InputSpec';
import { ProcessSpec } from './ProcessSpec';
import { DecisionSpec } from './DecisionSpec';
import { TerminalSpec } from './TerminalSpec';

export function SpecPanel() {
  const { currentFlow, selectedNodeId, updateNode } = useFlowStore();
  
  if (!currentFlow || !selectedNodeId) {
    return (
      <div className="spec-panel empty">
        <p>Select a node to edit its specification</p>
      </div>
    );
  }
  
  const allNodes = [currentFlow.trigger, ...currentFlow.nodes];
  const selectedNode = allNodes.find(n => n.id === selectedNodeId);
  
  if (!selectedNode) {
    return (
      <div className="spec-panel empty">
        <p>Node not found</p>
      </div>
    );
  }
  
  const handleSpecChange = (spec: any) => {
    updateNode(selectedNodeId, { spec });
  };
  
  return (
    <div className="spec-panel">
      <div className="spec-panel-header">
        <h3>{selectedNode.type.toUpperCase()}</h3>
        <span className="node-id">{selectedNode.id}</span>
      </div>
      
      <div className="spec-panel-content">
        {selectedNode.type === 'trigger' && (
          <TriggerSpec spec={selectedNode.spec} onChange={handleSpecChange} />
        )}
        {selectedNode.type === 'input' && (
          <InputSpec spec={selectedNode.spec} onChange={handleSpecChange} />
        )}
        {selectedNode.type === 'process' && (
          <ProcessSpec spec={selectedNode.spec} onChange={handleSpecChange} />
        )}
        {selectedNode.type === 'decision' && (
          <DecisionSpec spec={selectedNode.spec} onChange={handleSpecChange} />
        )}
        {selectedNode.type === 'terminal' && (
          <TerminalSpec spec={selectedNode.spec} onChange={handleSpecChange} />
        )}
      </div>
    </div>
  );
}
```

**File: `src/components/SpecPanel/InputSpec.tsx`**
```typescript
import React from 'react';
import { InputField, Validation } from '../../types';
import { Plus, Trash2, ChevronDown, ChevronUp } from 'lucide-react';

interface InputSpecProps {
  spec: { fields: InputField[] };
  onChange: (spec: { fields: InputField[] }) => void;
}

export function InputSpec({ spec, onChange }: InputSpecProps) {
  const addField = () => {
    onChange({
      fields: [
        ...spec.fields,
        {
          name: `field_${spec.fields.length + 1}`,
          type: 'string',
          required: true,
          validations: [],
        },
      ],
    });
  };
  
  const updateField = (index: number, updates: Partial<InputField>) => {
    const newFields = [...spec.fields];
    newFields[index] = { ...newFields[index], ...updates };
    onChange({ fields: newFields });
  };
  
  const removeField = (index: number) => {
    onChange({ fields: spec.fields.filter((_, i) => i !== index) });
  };
  
  const addValidation = (fieldIndex: number) => {
    const field = spec.fields[fieldIndex];
    updateField(fieldIndex, {
      validations: [
        ...field.validations,
        { type: 'min_length', value: 1, error: 'Field is required' },
      ],
    });
  };
  
  const updateValidation = (fieldIndex: number, valIndex: number, updates: Partial<Validation>) => {
    const field = spec.fields[fieldIndex];
    const newValidations = [...field.validations];
    newValidations[valIndex] = { ...newValidations[valIndex], ...updates };
    updateField(fieldIndex, { validations: newValidations });
  };
  
  const removeValidation = (fieldIndex: number, valIndex: number) => {
    const field = spec.fields[fieldIndex];
    updateField(fieldIndex, {
      validations: field.validations.filter((_, i) => i !== valIndex),
    });
  };
  
  return (
    <div className="input-spec">
      <div className="spec-section">
        <div className="section-header">
          <h4>Fields</h4>
          <button onClick={addField} className="btn-icon">
            <Plus size={16} />
          </button>
        </div>
        
        {spec.fields.map((field, fieldIndex) => (
          <FieldEditor
            key={fieldIndex}
            field={field}
            onChange={(updates) => updateField(fieldIndex, updates)}
            onRemove={() => removeField(fieldIndex)}
            onAddValidation={() => addValidation(fieldIndex)}
            onUpdateValidation={(vi, updates) => updateValidation(fieldIndex, vi, updates)}
            onRemoveValidation={(vi) => removeValidation(fieldIndex, vi)}
          />
        ))}
        
        {spec.fields.length === 0 && (
          <p className="empty-message">No fields. Click + to add.</p>
        )}
      </div>
    </div>
  );
}

interface FieldEditorProps {
  field: InputField;
  onChange: (updates: Partial<InputField>) => void;
  onRemove: () => void;
  onAddValidation: () => void;
  onUpdateValidation: (index: number, updates: Partial<Validation>) => void;
  onRemoveValidation: (index: number) => void;
}

function FieldEditor({
  field,
  onChange,
  onRemove,
  onAddValidation,
  onUpdateValidation,
  onRemoveValidation,
}: FieldEditorProps) {
  const [expanded, setExpanded] = React.useState(true);
  
  return (
    <div className="field-editor">
      <div className="field-header">
        <button onClick={() => setExpanded(!expanded)} className="btn-icon">
          {expanded ? <ChevronUp size={14} /> : <ChevronDown size={14} />}
        </button>
        <input
          type="text"
          value={field.name}
          onChange={(e) => onChange({ name: e.target.value })}
          className="field-name-input"
        />
        <select
          value={field.type}
          onChange={(e) => onChange({ type: e.target.value as any })}
          className="field-type-select"
        >
          <option value="string">string</option>
          <option value="number">number</option>
          <option value="boolean">boolean</option>
          <option value="array">array</option>
          <option value="object">object</option>
        </select>
        <label className="required-checkbox">
          <input
            type="checkbox"
            checked={field.required}
            onChange={(e) => onChange({ required: e.target.checked })}
          />
          Required
        </label>
        <button onClick={onRemove} className="btn-icon danger">
          <Trash2 size={14} />
        </button>
      </div>
      
      {expanded && (
        <div className="field-validations">
          <div className="validations-header">
            <span>Validations</span>
            <button onClick={onAddValidation} className="btn-icon small">
              <Plus size={12} />
            </button>
          </div>
          
          {field.validations.map((val, valIndex) => (
            <ValidationEditor
              key={valIndex}
              validation={val}
              fieldType={field.type}
              onChange={(updates) => onUpdateValidation(valIndex, updates)}
              onRemove={() => onRemoveValidation(valIndex)}
            />
          ))}
        </div>
      )}
    </div>
  );
}

interface ValidationEditorProps {
  validation: Validation;
  fieldType: string;
  onChange: (updates: Partial<Validation>) => void;
  onRemove: () => void;
}

function ValidationEditor({ validation, fieldType, onChange, onRemove }: ValidationEditorProps) {
  const validationTypes = getValidationTypesForFieldType(fieldType);
  
  return (
    <div className="validation-editor">
      <select
        value={validation.type}
        onChange={(e) => onChange({ type: e.target.value as any })}
        className="validation-type-select"
      >
        {validationTypes.map((vt) => (
          <option key={vt} value={vt}>{vt}</option>
        ))}
      </select>
      
      <input
        type="text"
        value={validation.value}
        onChange={(e) => onChange({ value: e.target.value })}
        placeholder="Value"
        className="validation-value-input"
      />
      
      <input
        type="text"
        value={validation.error}
        onChange={(e) => onChange({ error: e.target.value })}
        placeholder="Error message"
        className="validation-error-input"
      />
      
      <button onClick={onRemove} className="btn-icon danger small">
        <Trash2 size={12} />
      </button>
    </div>
  );
}

function getValidationTypesForFieldType(fieldType: string): string[] {
  switch (fieldType) {
    case 'string':
      return ['min_length', 'max_length', 'pattern', 'format', 'enum'];
    case 'number':
      return ['min', 'max', 'integer'];
    case 'array':
      return ['min_items', 'max_items'];
    default:
      return ['custom'];
  }
}
```

### Week 1.5: Agent Flow Support

#### Agent Canvas Layout

When a flow has `type: 'agent'`, the Canvas renders an **agent-centric layout** instead of the traditional node graph. The App.tsx routing already handles this based on the flow's type.

**File: `src/components/Canvas/Canvas.tsx`** (update to support both flow types)
```typescript
// In Canvas.tsx, add flow type detection:
import { isAgentFlow } from '../../types/flow';
import { AgentCanvas } from './AgentCanvas';

export function Canvas({ domainId, flowId }: CanvasProps) {
  const { currentFlow } = useFlowStore();

  if (!currentFlow) return <EmptyCanvas />;

  // Route to agent-specific canvas if it's an agent flow
  if (isAgentFlow(currentFlow)) {
    return <AgentCanvas flow={currentFlow} />;
  }

  // ... existing traditional canvas code
}
```

**File: `src/components/Canvas/AgentCanvas.tsx`**
```typescript
import React from 'react';
import { AgentFlow } from '../../types/flow';
import { useFlowStore } from '../../stores/flow-store';
import { AgentLoopBlock } from './agent-nodes/AgentLoopBlock';
import { ToolPalette } from './agent-nodes/ToolPalette';
import { GuardrailBlock } from './agent-nodes/GuardrailBlock';
import { MemoryBlock } from './agent-nodes/MemoryBlock';
import { HumanGateBlock } from './agent-nodes/HumanGateBlock';
import { Connection } from './Connection';

interface AgentCanvasProps {
  flow: AgentFlow;
}

export function AgentCanvas({ flow }: AgentCanvasProps) {
  const { selectedNodeId, selectNode } = useFlowStore();
  const config = flow.agent_config;

  // Find nodes by type
  const agentLoop = flow.nodes.find(n => n.type === 'agent_loop');
  const guardrails = flow.nodes.filter(n => n.type === 'guardrail');
  const humanGates = flow.nodes.filter(n => n.type === 'human_gate');
  const terminals = flow.nodes.filter(n => n.type === 'terminal');

  const inputGuardrail = guardrails.find(n => n.spec.position === 'input');
  const outputGuardrail = guardrails.find(n => n.spec.position === 'output');

  return (
    <div className="agent-canvas relative w-full h-full overflow-auto p-8">
      {/* Vertical flow layout */}
      <div className="flex flex-col items-center gap-6">

        {/* Input Guardrail */}
        {inputGuardrail && (
          <GuardrailBlock
            node={inputGuardrail}
            isSelected={selectedNodeId === inputGuardrail.id}
            onSelect={() => selectNode(inputGuardrail.id)}
          />
        )}

        {/* Agent Loop — the main block */}
        {agentLoop && (
          <AgentLoopBlock
            node={agentLoop}
            tools={config.tools}
            memory={config.memory}
            isSelected={selectedNodeId === agentLoop.id}
            onSelect={() => selectNode(agentLoop.id)}
          />
        )}

        {/* Output paths */}
        <div className="flex gap-8">
          {/* Output Guardrail → Response */}
          {outputGuardrail && (
            <div className="flex flex-col items-center gap-4">
              <GuardrailBlock
                node={outputGuardrail}
                isSelected={selectedNodeId === outputGuardrail.id}
                onSelect={() => selectNode(outputGuardrail.id)}
              />
              {terminals.filter(t => t.id === 'return_response').map(t => (
                <div key={t.id} className="terminal-node bg-green-900 border border-green-500 rounded-lg px-4 py-2 text-green-300 text-sm cursor-pointer"
                     onClick={() => selectNode(t.id)}>
                  ⬭ {t.id}
                </div>
              ))}
            </div>
          )}

          {/* Human Gate → Escalation */}
          {humanGates.map(gate => (
            <div key={gate.id} className="flex flex-col items-center gap-4">
              <HumanGateBlock
                node={gate}
                isSelected={selectedNodeId === gate.id}
                onSelect={() => selectNode(gate.id)}
              />
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
```

**File: `src/components/Canvas/agent-nodes/AgentLoopBlock.tsx`**
```typescript
import React from 'react';
import { AgentLoopNode, ToolDefinition } from '../../../types/node';
import { RotateCw, Wrench, Brain } from 'lucide-react';

interface AgentLoopBlockProps {
  node: AgentLoopNode;
  tools: ToolDefinition[];
  memory: Array<Record<string, any>>;
  isSelected: boolean;
  onSelect: () => void;
}

export function AgentLoopBlock({ node, tools, memory, isSelected, onSelect }: AgentLoopBlockProps) {
  return (
    <div
      className={`
        agent-loop-block border-2 rounded-xl p-6 min-w-[400px] cursor-pointer
        ${isSelected ? 'border-blue-400 bg-gray-800' : 'border-gray-600 bg-gray-900'}
      `}
      onClick={onSelect}
    >
      {/* Header */}
      <div className="flex items-center gap-2 mb-4">
        <RotateCw size={20} className="text-blue-400" />
        <span className="font-semibold text-white text-lg">Agent Loop</span>
        <span className="text-gray-500 text-sm ml-auto">max {node.spec.max_iterations} iterations</span>
      </div>

      {/* Model + Prompt preview */}
      <div className="mb-4 text-sm">
        <div className="text-gray-400">Model: <span className="text-white">{node.spec.model}</span></div>
        <div className="text-gray-400 mt-1">
          System Prompt:
          <p className="text-gray-300 mt-1 line-clamp-3 italic">
            {node.spec.system_prompt.slice(0, 150)}...
          </p>
        </div>
      </div>

      {/* Loop visualization */}
      <div className="border border-gray-700 rounded-lg p-3 mb-4 bg-gray-950">
        <div className="text-xs text-gray-500 mb-2">Reasoning Cycle</div>
        <div className="flex items-center gap-2 text-sm text-gray-300">
          <span>Reason</span> <span className="text-gray-600">→</span>
          <span>Select Tool</span> <span className="text-gray-600">→</span>
          <span>Execute</span> <span className="text-gray-600">→</span>
          <span>Observe</span> <span className="text-gray-600">→</span>
          <span className="text-blue-400">Repeat</span>
        </div>
      </div>

      {/* Tools */}
      <div className="mb-4">
        <div className="text-xs text-gray-500 mb-2 flex items-center gap-1">
          <Wrench size={12} /> Available Tools ({tools.length})
        </div>
        <div className="flex flex-wrap gap-2">
          {tools.map(tool => (
            <div
              key={tool.id}
              className={`
                px-3 py-1 rounded-md text-xs border cursor-pointer
                ${tool.is_terminal
                  ? 'border-orange-600 text-orange-300 bg-orange-950'
                  : 'border-gray-600 text-gray-300 bg-gray-800'
                }
              `}
              title={tool.description}
            >
              🔧 {tool.name}
              {tool.is_terminal && <span className="ml-1 text-orange-500">⏹</span>}
            </div>
          ))}
        </div>
      </div>

      {/* Memory */}
      {memory.length > 0 && (
        <div>
          <div className="text-xs text-gray-500 mb-2 flex items-center gap-1">
            <Brain size={12} /> Memory
          </div>
          <div className="flex flex-wrap gap-2">
            {memory.map((mem, i) => (
              <div key={i} className="px-3 py-1 rounded-md text-xs border border-purple-700 text-purple-300 bg-purple-950">
                ◈ {mem.name} ({mem.type})
              </div>
            ))}
          </div>
        </div>
      )}
    </div>
  );
}
```

**File: `src/components/Canvas/agent-nodes/GuardrailBlock.tsx`**
```typescript
import React from 'react';
import { GuardrailNode } from '../../../types/node';
import { Shield } from 'lucide-react';

interface GuardrailBlockProps {
  node: GuardrailNode;
  isSelected: boolean;
  onSelect: () => void;
}

export function GuardrailBlock({ node, isSelected, onSelect }: GuardrailBlockProps) {
  const checkCount = node.spec.checks.length;
  const isInput = node.spec.position === 'input';

  return (
    <div
      className={`
        guardrail-block border-2 rounded-lg px-4 py-3 cursor-pointer min-w-[200px]
        ${isSelected ? 'border-yellow-400' : 'border-yellow-700'}
        ${isInput ? 'bg-yellow-950' : 'bg-yellow-950'}
      `}
      onClick={onSelect}
    >
      <div className="flex items-center gap-2">
        <Shield size={16} className="text-yellow-400" />
        <span className="text-yellow-300 font-medium text-sm">
          {isInput ? 'Input' : 'Output'} Guardrail
        </span>
      </div>
      <div className="text-xs text-yellow-600 mt-1">
        {checkCount} check{checkCount !== 1 ? 's' : ''}:
        {node.spec.checks.map(c => c.type).join(', ')}
      </div>
    </div>
  );
}
```

**File: `src/components/Canvas/agent-nodes/HumanGateBlock.tsx`**
```typescript
import React from 'react';
import { HumanGateNode } from '../../../types/node';
import { Hand } from 'lucide-react';

interface HumanGateBlockProps {
  node: HumanGateNode;
  isSelected: boolean;
  onSelect: () => void;
}

export function HumanGateBlock({ node, isSelected, onSelect }: HumanGateBlockProps) {
  return (
    <div
      className={`
        human-gate-block border-2 rounded-lg px-4 py-3 cursor-pointer min-w-[200px]
        ${isSelected ? 'border-red-400' : 'border-red-700'}
        bg-red-950
      `}
      onClick={onSelect}
    >
      <div className="flex items-center gap-2">
        <Hand size={16} className="text-red-400" />
        <span className="text-red-300 font-medium text-sm">Human Gate</span>
      </div>
      <div className="text-xs text-red-600 mt-1">
        Timeout: {node.spec.timeout.duration}s → {node.spec.timeout.action}
      </div>
      <div className="flex gap-1 mt-2">
        {node.spec.approval_options.map(opt => (
          <span key={opt.id} className="px-2 py-0.5 bg-red-900 border border-red-800 rounded text-xs text-red-300">
            {opt.label}
          </span>
        ))}
      </div>
    </div>
  );
}
```

#### Agent Spec Panel Components

The SpecPanel detects the selected node type and renders the appropriate editor.

**File: `src/components/SpecPanel/SpecPanel.tsx`** (updated for agent nodes)
```typescript
// Add agent spec panels to the existing SpecPanel:
import { AgentLoopSpec } from './AgentLoopSpec';
import { ToolSpec } from './ToolSpec';
import { GuardrailSpec } from './GuardrailSpec';
import { HumanGateSpec } from './HumanGateSpec';
import { RouterSpec } from './RouterSpec';
import { LLMCallSpec } from './LLMCallSpec';

// In the render:
{selectedNode.type === 'agent_loop' && (
  <AgentLoopSpec spec={selectedNode.spec} onChange={handleSpecChange} />
)}
{selectedNode.type === 'guardrail' && (
  <GuardrailSpec spec={selectedNode.spec} onChange={handleSpecChange} />
)}
{selectedNode.type === 'human_gate' && (
  <HumanGateSpec spec={selectedNode.spec} onChange={handleSpecChange} />
)}
{selectedNode.type === 'router' && (
  <RouterSpec spec={selectedNode.spec} onChange={handleSpecChange} />
)}
{selectedNode.type === 'llm_call' && (
  <LLMCallSpec spec={selectedNode.spec} onChange={handleSpecChange} />
)}
```

**File: `src/components/SpecPanel/AgentLoopSpec.tsx`**
```typescript
import React from 'react';
import { AgentLoopNode } from '../../types/node';

interface AgentLoopSpecProps {
  spec: AgentLoopNode['spec'];
  onChange: (spec: AgentLoopNode['spec']) => void;
}

export function AgentLoopSpec({ spec, onChange }: AgentLoopSpecProps) {
  return (
    <div className="agent-loop-spec space-y-4">
      <div className="spec-section">
        <h4>Model</h4>
        <select
          value={spec.model}
          onChange={(e) => onChange({ ...spec, model: e.target.value })}
          className="select-field"
        >
          <option value="claude-opus-4-6">Claude Opus 4.6</option>
          <option value="claude-sonnet-4-5-20250929">Claude Sonnet 4.5</option>
          <option value="claude-haiku-4-5-20251001">Claude Haiku 4.5</option>
          <option value="gpt-4o">GPT-4o</option>
          <option value="custom">Custom</option>
        </select>
      </div>

      <div className="spec-section">
        <h4>System Prompt</h4>
        <textarea
          value={spec.system_prompt}
          onChange={(e) => onChange({ ...spec, system_prompt: e.target.value })}
          className="textarea-field"
          rows={8}
        />
      </div>

      <div className="spec-section">
        <h4>Max Iterations</h4>
        <input
          type="number"
          value={spec.max_iterations}
          onChange={(e) => onChange({ ...spec, max_iterations: parseInt(e.target.value) })}
          className="input-field"
          min={1}
          max={50}
        />
      </div>

      <div className="spec-section">
        <h4>Temperature</h4>
        <input
          type="range"
          value={spec.temperature || 0.3}
          onChange={(e) => onChange({ ...spec, temperature: parseFloat(e.target.value) })}
          min={0}
          max={1}
          step={0.1}
          className="range-field"
        />
        <span className="text-xs text-gray-400">{spec.temperature || 0.3}</span>
      </div>

      <div className="spec-section">
        <h4>Stop Conditions</h4>
        {spec.stop_conditions.map((cond, i) => (
          <div key={i} className="flex gap-2 items-center text-sm">
            {cond.tool_called && <span>Tool called: <code>{cond.tool_called}</code></span>}
            {cond.max_iterations_reached && <span>Max iterations reached</span>}
          </div>
        ))}
      </div>

      <div className="spec-section">
        <h4>On Max Iterations</h4>
        <select
          value={spec.on_max_iterations?.action || 'error'}
          onChange={(e) => onChange({
            ...spec,
            on_max_iterations: { ...spec.on_max_iterations, action: e.target.value as any }
          })}
          className="select-field"
        >
          <option value="escalate">Escalate to human</option>
          <option value="respond">Send best response</option>
          <option value="error">Return error</option>
        </select>
      </div>
    </div>
  );
}
```

### Week 1.75: Orchestration Components

#### Orchestration Canvas

The orchestration nodes extend the AgentCanvas with supervisor topology rendering. The `AgentCanvas` detects orchestration nodes and renders them accordingly.

**File: `src/components/Canvas/orchestration-nodes/OrchestratorBlock.tsx`**
```typescript
import React from 'react';
import { OrchestratorNode } from '../../../types/node';
import { Network, Eye } from 'lucide-react';

interface OrchestratorBlockProps {
  node: OrchestratorNode;
  isSelected: boolean;
  onSelect: () => void;
}

export function OrchestratorBlock({ node, isSelected, onSelect }: OrchestratorBlockProps) {
  const { spec } = node;
  const strategyLabels = {
    supervisor: 'Supervisor',
    round_robin: 'Round Robin',
    broadcast: 'Broadcast',
    consensus: 'Consensus',
  };

  return (
    <div
      className={`
        orchestrator-block border-2 rounded-xl p-6 min-w-[450px] cursor-pointer
        ${isSelected ? 'border-indigo-400 bg-gray-800' : 'border-indigo-700 bg-gray-900'}
      `}
      onClick={onSelect}
    >
      {/* Header */}
      <div className="flex items-center gap-2 mb-4">
        <Network size={20} className="text-indigo-400" />
        <span className="font-semibold text-white text-lg">Orchestrator</span>
        <span className="ml-auto px-2 py-0.5 rounded bg-indigo-900 text-indigo-300 text-xs">
          {strategyLabels[spec.strategy]}
        </span>
      </div>

      {/* Model + Prompt preview */}
      <div className="mb-4 text-sm">
        <div className="text-gray-400">Model: <span className="text-white">{spec.model}</span></div>
        <div className="text-gray-400 mt-1 line-clamp-2 italic">
          {spec.supervisor_prompt.slice(0, 120)}...
        </div>
      </div>

      {/* Managed Agents */}
      <div className="mb-4">
        <div className="text-xs text-gray-500 mb-2">Managed Agents ({spec.agents.length})</div>
        <div className="flex flex-col gap-1">
          {spec.agents
            .sort((a, b) => a.priority - b.priority)
            .map(agent => (
            <div key={agent.id} className="flex items-center gap-2 px-3 py-1.5 bg-gray-800 rounded border border-gray-700 text-sm">
              <span className="text-indigo-400 font-mono text-xs">P{agent.priority}</span>
              <span className="text-white font-medium">{agent.id}</span>
              <span className="text-gray-500 text-xs ml-auto">{agent.specialization}</span>
            </div>
          ))}
        </div>
      </div>

      {/* Supervision Rules */}
      <div className="mb-4">
        <div className="text-xs text-gray-500 mb-2 flex items-center gap-1">
          <Eye size={12} /> Supervision Rules ({spec.supervision.intervene_on.length})
        </div>
        <div className="flex flex-wrap gap-1">
          {spec.supervision.intervene_on.map((rule, i) => (
            <span key={i} className="px-2 py-0.5 bg-orange-950 border border-orange-800 rounded text-xs text-orange-300">
              {rule.condition}{rule.threshold ? ` > ${rule.threshold}` : ''} → {rule.action}
            </span>
          ))}
        </div>
      </div>

      {/* Shared Memory */}
      {spec.shared_memory && spec.shared_memory.length > 0 && (
        <div>
          <div className="text-xs text-gray-500 mb-1">Shared Memory</div>
          <div className="flex gap-2">
            {spec.shared_memory.map((mem, i) => (
              <span key={i} className="px-2 py-0.5 bg-purple-950 border border-purple-800 rounded text-xs text-purple-300">
                ◈ {mem.name} ({mem.access})
              </span>
            ))}
          </div>
        </div>
      )}

      {/* Result Merge */}
      <div className="mt-3 text-xs text-gray-500">
        Merge: <span className="text-gray-300">{spec.result_merge.strategy}</span>
      </div>
    </div>
  );
}
```

**File: `src/components/Canvas/orchestration-nodes/SmartRouterBlock.tsx`**
```typescript
import React from 'react';
import { SmartRouterNode } from '../../../types/node';
import { GitBranch, Zap, FlaskConical, ShieldOff } from 'lucide-react';

interface SmartRouterBlockProps {
  node: SmartRouterNode;
  isSelected: boolean;
  onSelect: () => void;
}

export function SmartRouterBlock({ node, isSelected, onSelect }: SmartRouterBlockProps) {
  const { spec } = node;
  const hasCircuitBreaker = spec.policies.circuit_breaker?.enabled;
  const hasABTesting = spec.ab_testing?.enabled;

  return (
    <div
      className={`
        smart-router-block border-2 rounded-xl p-5 min-w-[380px] cursor-pointer
        ${isSelected ? 'border-cyan-400 bg-gray-800' : 'border-cyan-700 bg-gray-900'}
      `}
      onClick={onSelect}
    >
      {/* Header */}
      <div className="flex items-center gap-2 mb-3">
        <GitBranch size={18} className="text-cyan-400" />
        <span className="font-semibold text-white">Smart Router</span>
        <div className="ml-auto flex gap-1">
          {hasCircuitBreaker && (
            <span title="Circuit breaker enabled" className="px-1.5 py-0.5 bg-red-950 border border-red-800 rounded text-xs text-red-400">
              <ShieldOff size={10} className="inline" /> CB
            </span>
          )}
          {hasABTesting && (
            <span title="A/B testing enabled" className="px-1.5 py-0.5 bg-green-950 border border-green-800 rounded text-xs text-green-400">
              <FlaskConical size={10} className="inline" /> A/B
            </span>
          )}
        </div>
      </div>

      {/* Rule-based routes */}
      {spec.rules.length > 0 && (
        <div className="mb-3">
          <div className="text-xs text-gray-500 mb-1 flex items-center gap-1">
            <Zap size={10} /> Rules ({spec.rules.length})
          </div>
          {spec.rules
            .sort((a, b) => a.priority - b.priority)
            .map(rule => (
            <div key={rule.id} className="flex items-center gap-2 text-xs py-0.5">
              <span className="text-cyan-600 font-mono">P{rule.priority}</span>
              <span className="text-gray-400 truncate max-w-[200px]">{rule.condition}</span>
              <span className="text-gray-600">→</span>
              <span className="text-cyan-300">{rule.route}</span>
            </div>
          ))}
        </div>
      )}

      {/* LLM routing */}
      {spec.llm_routing.enabled && (
        <div className="mb-3">
          <div className="text-xs text-gray-500 mb-1">LLM Fallback: {spec.llm_routing.model}</div>
          <div className="text-xs text-gray-400">
            Confidence threshold: {spec.llm_routing.confidence_threshold}
          </div>
        </div>
      )}

      {/* Fallback chain */}
      <div className="text-xs text-gray-500">
        Fallback: {spec.fallback_chain.join(' → ')}
      </div>
    </div>
  );
}
```

**File: `src/components/Canvas/orchestration-nodes/HandoffBlock.tsx`**
```typescript
import React from 'react';
import { HandoffNode } from '../../../types/node';
import { ArrowLeftRight, ArrowRight, Users } from 'lucide-react';

interface HandoffBlockProps {
  node: HandoffNode;
  isSelected: boolean;
  onSelect: () => void;
}

const modeIcons = {
  transfer: ArrowRight,
  consult: ArrowLeftRight,
  collaborate: Users,
};

const modeColors = {
  transfer: 'border-amber-700 bg-amber-950 text-amber-300',
  consult: 'border-teal-700 bg-teal-950 text-teal-300',
  collaborate: 'border-pink-700 bg-pink-950 text-pink-300',
};

export function HandoffBlock({ node, isSelected, onSelect }: HandoffBlockProps) {
  const { spec } = node;
  const ModeIcon = modeIcons[spec.mode];

  return (
    <div
      className={`
        handoff-block border-2 rounded-lg px-4 py-3 cursor-pointer min-w-[220px]
        ${isSelected ? 'border-white' : modeColors[spec.mode].split(' ')[0]}
        ${modeColors[spec.mode].split(' ').slice(1).join(' ')}
      `}
      onClick={onSelect}
    >
      <div className="flex items-center gap-2 mb-1">
        <ModeIcon size={16} />
        <span className="font-medium text-sm capitalize">{spec.mode} Handoff</span>
      </div>
      <div className="text-xs opacity-70">
        → {spec.target.flow}
        {spec.target.domain && <span className="ml-1">({spec.target.domain})</span>}
      </div>
      <div className="text-xs opacity-50 mt-1">
        Context: {spec.context_transfer.max_context_tokens} tokens
        | Timeout: {spec.on_failure.timeout}s
      </div>
      {spec.notify_customer && (
        <div className="text-xs opacity-50 mt-1">Notifies customer</div>
      )}
    </div>
  );
}
```

**File: `src/components/Canvas/orchestration-nodes/AgentGroupBoundary.tsx`**
```typescript
import React from 'react';
import { AgentGroupNode } from '../../../types/node';
import { Box } from 'lucide-react';

interface AgentGroupBoundaryProps {
  node: AgentGroupNode;
  isSelected: boolean;
  onSelect: () => void;
  children: React.ReactNode;        // The orchestrator + agents rendered inside
}

export function AgentGroupBoundary({ node, isSelected, onSelect, children }: AgentGroupBoundaryProps) {
  const { spec } = node;

  return (
    <div
      className={`
        agent-group-boundary border-2 border-dashed rounded-2xl p-6 relative
        ${isSelected ? 'border-gray-300' : 'border-gray-700'}
      `}
      onClick={(e) => { e.stopPropagation(); onSelect(); }}
    >
      {/* Group header */}
      <div className="absolute -top-3 left-4 bg-gray-900 px-3 py-0.5 rounded flex items-center gap-2">
        <Box size={14} className="text-gray-400" />
        <span className="text-sm text-gray-300 font-medium">{spec.name}</span>
        <span className="text-xs text-gray-600">
          {spec.members.length} agents | {spec.coordination.communication}
        </span>
      </div>

      {/* Children (orchestrator + agent blocks) */}
      <div className="mt-2">
        {children}
      </div>

      {/* Shared memory footer */}
      {spec.shared_memory.length > 0 && (
        <div className="mt-4 flex gap-2 items-center">
          <span className="text-xs text-gray-500">Shared:</span>
          {spec.shared_memory.map((mem, i) => (
            <span key={i} className="px-2 py-0.5 bg-purple-950 border border-purple-800 rounded text-xs text-purple-300">
              ◈ {mem.name}
            </span>
          ))}
        </div>
      )}

      {/* Metrics */}
      {spec.metrics && (
        <div className="mt-2 text-xs text-gray-600">
          Tracking: {spec.metrics.track.join(', ')}
        </div>
      )}
    </div>
  );
}
```

#### Orchestration Spec Panels

**File: `src/components/SpecPanel/OrchestratorSpec.tsx`**
```typescript
import React from 'react';
import { OrchestratorNode, OrchestratorAgent, SupervisionRule } from '../../types/node';
import { Plus, Trash2 } from 'lucide-react';

interface OrchestratorSpecProps {
  spec: OrchestratorNode['spec'];
  onChange: (spec: OrchestratorNode['spec']) => void;
}

export function OrchestratorSpec({ spec, onChange }: OrchestratorSpecProps) {
  const addAgent = () => {
    const newAgent: OrchestratorAgent = {
      id: `agent_${spec.agents.length + 1}`,
      flow: '',
      specialization: '',
      priority: spec.agents.length + 1,
    };
    onChange({ ...spec, agents: [...spec.agents, newAgent] });
  };

  const updateAgent = (index: number, updates: Partial<OrchestratorAgent>) => {
    const agents = [...spec.agents];
    agents[index] = { ...agents[index], ...updates };
    onChange({ ...spec, agents });
  };

  const removeAgent = (index: number) => {
    onChange({ ...spec, agents: spec.agents.filter((_, i) => i !== index) });
  };

  const addSupervisionRule = () => {
    const rule: SupervisionRule = {
      condition: 'agent_iterations_exceeded',
      threshold: 5,
      action: 'reassign',
    };
    onChange({
      ...spec,
      supervision: {
        ...spec.supervision,
        intervene_on: [...spec.supervision.intervene_on, rule],
      },
    });
  };

  return (
    <div className="orchestrator-spec space-y-4">
      {/* Strategy */}
      <div className="spec-section">
        <h4>Strategy</h4>
        <select
          value={spec.strategy}
          onChange={(e) => onChange({ ...spec, strategy: e.target.value as any })}
          className="select-field"
        >
          <option value="supervisor">Supervisor (LLM decides)</option>
          <option value="round_robin">Round Robin (even distribution)</option>
          <option value="broadcast">Broadcast (all agents, merge results)</option>
          <option value="consensus">Consensus (all respond, pick best)</option>
        </select>
      </div>

      {/* Model */}
      <div className="spec-section">
        <h4>Supervisor Model</h4>
        <select
          value={spec.model}
          onChange={(e) => onChange({ ...spec, model: e.target.value })}
          className="select-field"
        >
          <option value="claude-opus-4-6">Claude Opus 4.6</option>
          <option value="claude-sonnet-4-5-20250929">Claude Sonnet 4.5</option>
          <option value="claude-haiku-4-5-20251001">Claude Haiku 4.5</option>
        </select>
      </div>

      {/* Supervisor Prompt */}
      <div className="spec-section">
        <h4>Supervisor Prompt</h4>
        <textarea
          value={spec.supervisor_prompt}
          onChange={(e) => onChange({ ...spec, supervisor_prompt: e.target.value })}
          className="textarea-field"
          rows={6}
        />
      </div>

      {/* Managed Agents */}
      <div className="spec-section">
        <div className="section-header">
          <h4>Managed Agents</h4>
          <button onClick={addAgent} className="btn-icon"><Plus size={16} /></button>
        </div>
        {spec.agents.map((agent, i) => (
          <div key={i} className="flex gap-2 items-center mb-2 p-2 bg-gray-800 rounded">
            <input type="text" value={agent.id} placeholder="Agent ID"
              onChange={(e) => updateAgent(i, { id: e.target.value })}
              className="input-field flex-1" />
            <input type="text" value={agent.flow} placeholder="Flow ID"
              onChange={(e) => updateAgent(i, { flow: e.target.value })}
              className="input-field flex-1" />
            <input type="number" value={agent.priority} min={0}
              onChange={(e) => updateAgent(i, { priority: parseInt(e.target.value) })}
              className="input-field w-16" title="Priority" />
            <button onClick={() => removeAgent(i)} className="btn-icon danger">
              <Trash2 size={14} />
            </button>
          </div>
        ))}
      </div>

      {/* Supervision Rules */}
      <div className="spec-section">
        <div className="section-header">
          <h4>Supervision Rules</h4>
          <button onClick={addSupervisionRule} className="btn-icon"><Plus size={16} /></button>
        </div>
        {spec.supervision.intervene_on.map((rule, i) => (
          <div key={i} className="flex gap-2 items-center mb-1 text-sm">
            <select value={rule.condition}
              onChange={(e) => {
                const rules = [...spec.supervision.intervene_on];
                rules[i] = { ...rules[i], condition: e.target.value as any };
                onChange({ ...spec, supervision: { ...spec.supervision, intervene_on: rules } });
              }}
              className="select-field flex-1">
              <option value="agent_iterations_exceeded">Iterations exceeded</option>
              <option value="confidence_below">Confidence below</option>
              <option value="customer_sentiment">Customer sentiment</option>
              <option value="agent_error">Agent error</option>
              <option value="timeout">Timeout</option>
            </select>
            <select value={rule.action}
              onChange={(e) => {
                const rules = [...spec.supervision.intervene_on];
                rules[i] = { ...rules[i], action: e.target.value as any };
                onChange({ ...spec, supervision: { ...spec.supervision, intervene_on: rules } });
              }}
              className="select-field flex-1">
              <option value="reassign">Reassign</option>
              <option value="add_instructions">Add instructions</option>
              <option value="escalate_to_human">Escalate to human</option>
              <option value="retry_with_different_agent">Retry different agent</option>
            </select>
          </div>
        ))}
      </div>

      {/* Result Merge */}
      <div className="spec-section">
        <h4>Result Merge Strategy</h4>
        <select
          value={spec.result_merge.strategy}
          onChange={(e) => onChange({ ...spec, result_merge: { strategy: e.target.value as any } })}
          className="select-field"
        >
          <option value="last_wins">Last wins</option>
          <option value="best_of">Best of (supervisor evaluates)</option>
          <option value="combine">Combine (supervisor synthesizes)</option>
          <option value="supervisor_picks">Supervisor picks</option>
        </select>
      </div>
    </div>
  );
}
```

**File: `src/components/SpecPanel/HandoffSpec.tsx`**
```typescript
import React from 'react';
import { HandoffNode } from '../../types/node';

interface HandoffSpecProps {
  spec: HandoffNode['spec'];
  onChange: (spec: HandoffNode['spec']) => void;
}

export function HandoffSpec({ spec, onChange }: HandoffSpecProps) {
  return (
    <div className="handoff-spec space-y-4">
      {/* Mode */}
      <div className="spec-section">
        <h4>Handoff Mode</h4>
        <select
          value={spec.mode}
          onChange={(e) => onChange({ ...spec, mode: e.target.value as any })}
          className="select-field"
        >
          <option value="transfer">Transfer (source stops, target takes over)</option>
          <option value="consult">Consult (source waits, target answers, result returns)</option>
          <option value="collaborate">Collaborate (both active, shared context)</option>
        </select>
      </div>

      {/* Target */}
      <div className="spec-section">
        <h4>Target</h4>
        <div className="flex gap-2">
          <input type="text" value={spec.target.flow} placeholder="Flow ID"
            onChange={(e) => onChange({ ...spec, target: { ...spec.target, flow: e.target.value } })}
            className="input-field flex-1" />
          <input type="text" value={spec.target.domain} placeholder="Domain"
            onChange={(e) => onChange({ ...spec, target: { ...spec.target, domain: e.target.value } })}
            className="input-field flex-1" />
        </div>
      </div>

      {/* Context Transfer */}
      <div className="spec-section">
        <h4>Max Context Tokens</h4>
        <input type="number" value={spec.context_transfer.max_context_tokens}
          onChange={(e) => onChange({
            ...spec,
            context_transfer: { ...spec.context_transfer, max_context_tokens: parseInt(e.target.value) }
          })}
          className="input-field" min={100} max={10000} />
      </div>

      {/* On Complete */}
      <div className="spec-section">
        <h4>On Complete</h4>
        <div className="flex gap-2">
          <select value={spec.on_complete.return_to}
            onChange={(e) => onChange({ ...spec, on_complete: { ...spec.on_complete, return_to: e.target.value as any } })}
            className="select-field flex-1">
            <option value="source_agent">Return to source agent</option>
            <option value="orchestrator">Return to orchestrator</option>
            <option value="terminal">End flow</option>
          </select>
          <select value={spec.on_complete.merge_strategy}
            onChange={(e) => onChange({ ...spec, on_complete: { ...spec.on_complete, merge_strategy: e.target.value as any } })}
            className="select-field flex-1">
            <option value="append">Append result</option>
            <option value="replace">Replace context</option>
            <option value="summarize">Summarize</option>
          </select>
        </div>
      </div>

      {/* Failure Handling */}
      <div className="spec-section">
        <h4>Timeout (seconds)</h4>
        <input type="number" value={spec.on_failure.timeout}
          onChange={(e) => onChange({
            ...spec,
            on_failure: { ...spec.on_failure, timeout: parseInt(e.target.value) }
          })}
          className="input-field" min={5} max={600} />
      </div>

      {/* Customer Notification */}
      <div className="spec-section">
        <label className="flex items-center gap-2">
          <input type="checkbox" checked={spec.notify_customer || false}
            onChange={(e) => onChange({ ...spec, notify_customer: e.target.checked })} />
          <span className="text-sm">Notify customer about handoff</span>
        </label>
      </div>
    </div>
  );
}
```

### Week 2: File Operations & Git

#### Day 8-9: Tauri File Commands

**File: `src-tauri/src/commands/file.rs`**
```rust
use std::fs;
use std::path::PathBuf;
use tauri::command;

#[command]
pub async fn read_file(path: String) -> Result<String, String> {
    fs::read_to_string(&path).map_err(|e| e.to_string())
}

#[command]
pub async fn write_file(path: String, content: String) -> Result<(), String> {
    // Create parent directories if needed
    if let Some(parent) = PathBuf::from(&path).parent() {
        fs::create_dir_all(parent).map_err(|e| e.to_string())?;
    }
    fs::write(&path, content).map_err(|e| e.to_string())
}

#[command]
pub async fn list_directory(path: String) -> Result<Vec<FileEntry>, String> {
    let entries = fs::read_dir(&path).map_err(|e| e.to_string())?;
    
    let mut result = Vec::new();
    for entry in entries {
        let entry = entry.map_err(|e| e.to_string())?;
        let metadata = entry.metadata().map_err(|e| e.to_string())?;
        
        result.push(FileEntry {
            name: entry.file_name().to_string_lossy().to_string(),
            path: entry.path().to_string_lossy().to_string(),
            is_directory: metadata.is_dir(),
        });
    }
    
    Ok(result)
}

#[command]
pub async fn file_exists(path: String) -> bool {
    PathBuf::from(&path).exists()
}

#[derive(serde::Serialize)]
pub struct FileEntry {
    name: String,
    path: String,
    is_directory: bool,
}
```

**File: `src-tauri/src/commands/entity.rs`**
```rust
use std::fs;
use std::path::PathBuf;
use tauri::command;

/// Create a new domain: create directory + domain.yaml + flows/ subdirectory
#[command]
pub async fn create_domain(
    project_path: String,
    name: String,
    description: String,
) -> Result<(), String> {
    let domain_dir = PathBuf::from(&project_path)
        .join("specs/domains")
        .join(&name);

    if domain_dir.exists() {
        return Err(format!("Domain '{}' already exists", name));
    }

    // Create directory structure
    let flows_dir = domain_dir.join("flows");
    fs::create_dir_all(&flows_dir).map_err(|e| e.to_string())?;

    // Write domain.yaml
    let domain_yaml = format!(
        "domain:\n  name: {}\n  description: {}\n\npublishes_events: []\nconsumes_events: []\n",
        name, description
    );
    fs::write(domain_dir.join("domain.yaml"), domain_yaml)
        .map_err(|e| e.to_string())?;

    Ok(())
}

/// Rename domain directory and update all cross-references
#[command]
pub async fn rename_domain(
    project_path: String,
    old_name: String,
    new_name: String,
) -> Result<(), String> {
    let specs = PathBuf::from(&project_path).join("specs");
    let old_dir = specs.join("domains").join(&old_name);
    let new_dir = specs.join("domains").join(&new_name);

    if !old_dir.exists() {
        return Err(format!("Domain '{}' not found", old_name));
    }
    if new_dir.exists() {
        return Err(format!("Domain '{}' already exists", new_name));
    }

    fs::rename(&old_dir, &new_dir).map_err(|e| e.to_string())?;

    // Update domain.yaml inside the renamed directory
    let domain_yaml_path = new_dir.join("domain.yaml");
    if domain_yaml_path.exists() {
        let content = fs::read_to_string(&domain_yaml_path)
            .map_err(|e| e.to_string())?;
        let updated = content.replace(
            &format!("name: {}", old_name),
            &format!("name: {}", new_name),
        );
        fs::write(&domain_yaml_path, updated).map_err(|e| e.to_string())?;
    }

    Ok(())
}

/// Delete a domain directory (after confirmation in UI)
#[command]
pub async fn delete_domain(
    project_path: String,
    name: String,
) -> Result<u32, String> {
    let domain_dir = PathBuf::from(&project_path)
        .join("specs/domains")
        .join(&name);

    if !domain_dir.exists() {
        return Err(format!("Domain '{}' not found", name));
    }

    // Count flows for confirmation
    let flows_dir = domain_dir.join("flows");
    let flow_count = if flows_dir.exists() {
        fs::read_dir(&flows_dir)
            .map(|entries| entries.filter_map(|e| e.ok()).count() as u32)
            .unwrap_or(0)
    } else {
        0
    };

    fs::remove_dir_all(&domain_dir).map_err(|e| e.to_string())?;
    Ok(flow_count)
}

/// Create a new flow YAML file with starter template
#[command]
pub async fn create_flow(
    project_path: String,
    domain: String,
    name: String,
    flow_type: String,  // "traditional" | "agent" | "orchestration"
) -> Result<String, String> {
    let flow_id = name.to_lowercase().replace(' ', "-");
    let flow_path = PathBuf::from(&project_path)
        .join("specs/domains")
        .join(&domain)
        .join("flows")
        .join(format!("{}.yaml", flow_id));

    if flow_path.exists() {
        return Err(format!("Flow '{}' already exists in domain '{}'", flow_id, domain));
    }

    let yaml = match flow_type.as_str() {
        "agent" => format!(
            "flow:\n  id: {id}\n  name: {name}\n  domain: {domain}\n  type: agent\n  description: \"\"\n\ntrigger:\n  type: http\n  method: POST\n  path: /api/{domain}/{id}\n\nagent_loop:\n  model: default\n  max_iterations: 10\n  system_prompt: \"\"\n\nnodes: []\n",
            id = flow_id, name = name, domain = domain
        ),
        "orchestration" => format!(
            "flow:\n  id: {id}\n  name: {name}\n  domain: {domain}\n  type: orchestration\n  description: \"\"\n\ntrigger:\n  type: http\n  method: POST\n  path: /api/{domain}/{id}\n\norchestrator:\n  strategy: supervisor\n  agents: []\n\nnodes: []\n",
            id = flow_id, name = name, domain = domain
        ),
        _ => format!(
            "flow:\n  id: {id}\n  name: {name}\n  domain: {domain}\n  type: traditional\n  description: \"\"\n\ntrigger:\n  type: http\n  method: POST\n  path: /api/{domain}/{id}\n\nnodes:\n  - id: done\n    type: terminal\n    spec:\n      status: 200\n      body:\n        message: \"Success\"\n",
            id = flow_id, name = name, domain = domain
        ),
    };

    // Ensure parent directory exists
    if let Some(parent) = flow_path.parent() {
        fs::create_dir_all(parent).map_err(|e| e.to_string())?;
    }
    fs::write(&flow_path, yaml).map_err(|e| e.to_string())?;

    Ok(flow_id)
}

/// Rename a flow file and update its internal id field
#[command]
pub async fn rename_flow(
    project_path: String,
    domain: String,
    old_id: String,
    new_name: String,
) -> Result<String, String> {
    let new_id = new_name.to_lowercase().replace(' ', "-");
    let flows_dir = PathBuf::from(&project_path)
        .join("specs/domains")
        .join(&domain)
        .join("flows");

    let old_path = flows_dir.join(format!("{}.yaml", old_id));
    let new_path = flows_dir.join(format!("{}.yaml", new_id));

    if !old_path.exists() {
        return Err(format!("Flow '{}' not found", old_id));
    }
    if new_path.exists() && new_id != old_id {
        return Err(format!("Flow '{}' already exists", new_id));
    }

    // Update YAML content
    let content = fs::read_to_string(&old_path).map_err(|e| e.to_string())?;
    let updated = content
        .replace(&format!("id: {}", old_id), &format!("id: {}", new_id))
        .replace(&format!("name: {}", old_id), &format!("name: {}", new_name));
    fs::write(&old_path, updated).map_err(|e| e.to_string())?;

    // Rename file
    if new_id != old_id {
        fs::rename(&old_path, &new_path).map_err(|e| e.to_string())?;
    }

    Ok(new_id)
}

/// Delete a flow YAML file
#[command]
pub async fn delete_flow(
    project_path: String,
    domain: String,
    flow_id: String,
) -> Result<(), String> {
    let flow_path = PathBuf::from(&project_path)
        .join("specs/domains")
        .join(&domain)
        .join("flows")
        .join(format!("{}.yaml", flow_id));

    if !flow_path.exists() {
        return Err(format!("Flow '{}' not found", flow_id));
    }

    fs::remove_file(&flow_path).map_err(|e| e.to_string())?;
    Ok(())
}

/// Duplicate a flow YAML file with a new id
#[command]
pub async fn duplicate_flow(
    project_path: String,
    domain: String,
    flow_id: String,
    new_name: String,
) -> Result<String, String> {
    let new_id = new_name.to_lowercase().replace(' ', "-");
    let flows_dir = PathBuf::from(&project_path)
        .join("specs/domains")
        .join(&domain)
        .join("flows");

    let source_path = flows_dir.join(format!("{}.yaml", flow_id));
    let target_path = flows_dir.join(format!("{}.yaml", new_id));

    if !source_path.exists() {
        return Err(format!("Flow '{}' not found", flow_id));
    }
    if target_path.exists() {
        return Err(format!("Flow '{}' already exists", new_id));
    }

    let content = fs::read_to_string(&source_path).map_err(|e| e.to_string())?;
    let updated = content
        .replace(&format!("id: {}", flow_id), &format!("id: {}", new_id))
        .replace(&format!("name: {}", flow_id), &format!("name: {}", new_name));
    fs::write(&target_path, updated).map_err(|e| e.to_string())?;

    Ok(new_id)
}

/// Move a flow from one domain to another
#[command]
pub async fn move_flow(
    project_path: String,
    source_domain: String,
    target_domain: String,
    flow_id: String,
) -> Result<(), String> {
    let specs = PathBuf::from(&project_path).join("specs/domains");
    let source = specs.join(&source_domain).join("flows").join(format!("{}.yaml", flow_id));
    let target_dir = specs.join(&target_domain).join("flows");
    let target = target_dir.join(format!("{}.yaml", flow_id));

    if !source.exists() {
        return Err(format!("Flow '{}' not found in '{}'", flow_id, source_domain));
    }
    if target.exists() {
        return Err(format!("Flow '{}' already exists in '{}'", flow_id, target_domain));
    }

    // Update domain field in YAML
    let content = fs::read_to_string(&source).map_err(|e| e.to_string())?;
    let updated = content.replace(
        &format!("domain: {}", source_domain),
        &format!("domain: {}", target_domain),
    );

    fs::create_dir_all(&target_dir).map_err(|e| e.to_string())?;
    fs::write(&target, updated).map_err(|e| e.to_string())?;
    fs::remove_file(&source).map_err(|e| e.to_string())?;

    Ok(())
}
```

#### Day 10-11: YAML Serialization

**File: `src/utils/yaml.ts`**
```typescript
import * as YAML from 'yaml';
import { Flow, FlowNode } from '../types';

export function flowToYaml(flow: Flow): string {
  const doc = {
    flow: {
      id: flow.id,
      name: flow.name,
      description: flow.description,
      domain: flow.domain,
    },
    trigger: nodeToYaml(flow.trigger),
    nodes: flow.nodes.map(nodeToYaml),
    metadata: flow.metadata,
  };
  
  return YAML.stringify(doc, {
    indent: 2,
    lineWidth: 120,
  });
}

export function yamlToFlow(yamlString: string): Flow {
  const doc = YAML.parse(yamlString);
  
  return {
    id: doc.flow.id,
    name: doc.flow.name,
    description: doc.flow.description,
    domain: doc.flow.domain,
    trigger: yamlToNode(doc.trigger),
    nodes: doc.nodes.map(yamlToNode),
    metadata: doc.metadata,
  };
}

function nodeToYaml(node: FlowNode): any {
  return {
    id: node.id,
    type: node.type,
    position: node.position,
    spec: node.spec,
    connections: node.connections,
  };
}

function yamlToNode(yaml: any): FlowNode {
  return {
    id: yaml.id,
    type: yaml.type,
    position: yaml.position,
    spec: yaml.spec,
    connections: yaml.connections || {},
  };
}
```

#### Day 12-14: Git Integration

**File: `src-tauri/src/commands/git.rs`**
```rust
use git2::{Repository, Status, StatusOptions};
use serde::{Deserialize, Serialize};
use tauri::command;

#[derive(Serialize)]
pub struct GitStatus {
    branch: String,
    staged: Vec<String>,
    unstaged: Vec<String>,
    untracked: Vec<String>,
}

#[command]
pub async fn git_status(project_path: String) -> Result<GitStatus, String> {
    let repo = Repository::open(&project_path).map_err(|e| e.to_string())?;
    
    let head = repo.head().map_err(|e| e.to_string())?;
    let branch = head.shorthand().unwrap_or("detached").to_string();
    
    let mut opts = StatusOptions::new();
    opts.include_untracked(true);
    
    let statuses = repo.statuses(Some(&mut opts)).map_err(|e| e.to_string())?;
    
    let mut staged = Vec::new();
    let mut unstaged = Vec::new();
    let mut untracked = Vec::new();
    
    for entry in statuses.iter() {
        let path = entry.path().unwrap_or("").to_string();
        let status = entry.status();
        
        if status.contains(Status::INDEX_NEW) || 
           status.contains(Status::INDEX_MODIFIED) ||
           status.contains(Status::INDEX_DELETED) {
            staged.push(path.clone());
        }
        
        if status.contains(Status::WT_MODIFIED) ||
           status.contains(Status::WT_DELETED) {
            unstaged.push(path.clone());
        }
        
        if status.contains(Status::WT_NEW) {
            untracked.push(path);
        }
    }
    
    Ok(GitStatus {
        branch,
        staged,
        unstaged,
        untracked,
    })
}

#[command]
pub async fn git_stage(project_path: String, file_path: String) -> Result<(), String> {
    let repo = Repository::open(&project_path).map_err(|e| e.to_string())?;
    let mut index = repo.index().map_err(|e| e.to_string())?;
    index.add_path(std::path::Path::new(&file_path)).map_err(|e| e.to_string())?;
    index.write().map_err(|e| e.to_string())?;
    Ok(())
}

#[command]
pub async fn git_commit(project_path: String, message: String) -> Result<String, String> {
    let repo = Repository::open(&project_path).map_err(|e| e.to_string())?;
    
    let mut index = repo.index().map_err(|e| e.to_string())?;
    let oid = index.write_tree().map_err(|e| e.to_string())?;
    let tree = repo.find_tree(oid).map_err(|e| e.to_string())?;
    
    let signature = repo.signature().map_err(|e| e.to_string())?;
    let parent = repo.head().map_err(|e| e.to_string())?
        .peel_to_commit().map_err(|e| e.to_string())?;
    
    let commit_oid = repo.commit(
        Some("HEAD"),
        &signature,
        &signature,
        &message,
        &tree,
        &[&parent]
    ).map_err(|e| e.to_string())?;
    
    Ok(commit_oid.to_string())
}

/// Clone a repository using the system git binary.
/// Uses system git to inherit all authentication (macOS Keychain, SSH agent, credential helpers).
#[tauri::command]
pub fn git_clone(url: String, path: String) -> Result<(), String> {
    let output = std::process::Command::new("git")
        .args(["clone", &url, &path])
        .output()
        .map_err(|e| format!("Failed to run git: {}. Is git installed?", e))?;
    if output.status.success() {
        Ok(())
    } else {
        let stderr = String::from_utf8_lossy(&output.stderr);
        Err(format!("Clone failed: {}", stderr.trim()))
    }
}
```


### Claude Code Integration

#### Cowork Workflow — DDD Tool ↔ Claude Code Terminal

The recommended workflow uses Claude Code's own terminal with custom slash commands rather than embedding a full terminal:

1. **Design** in DDD Tool — create/edit flows, add nodes, set specs
2. **Save** — specs are written to `specs/` as YAML
3. **Copy command** from `ClaudeCommandBox` in DDD → paste into Claude Code terminal
4. **Implement** — Claude Code reads specs, generates code, runs tests (`/ddd-implement`)
5. **Fix interactively** — fix runtime errors, test failures directly in Claude Code
6. **Sync** — run `/ddd-sync` to update `.ddd/mapping.yaml`
7. **Reload** in DDD Tool to see updated implementation status

**Custom Claude Code commands** (installed at `~/.claude/commands/`, repo: [mhcandan/claude-commands](https://github.com/mhcandan/claude-commands)):

| Command | What it does |
|---------|-------------|
| `/ddd-create` | Describe project in natural language → generates full spec structure (domains, flows, schemas, config). Fetches the DDD Usage Guide from GitHub at runtime. |
| `/ddd-implement --all` | Implement all domains and flows |
| `/ddd-implement {domain}` | Implement all flows in one domain |
| `/ddd-implement {domain}/{flow}` | Implement a single flow |
| `/ddd-implement` | Interactive: lists flows with status, asks what to implement |
| `/ddd-update` | Natural language change request → updated YAML specs |
| `/ddd-update --add-flow {domain}` | Add a new flow to a domain from description |
| `/ddd-update --add-domain` | Add a new domain with flows from description |
| `/ddd-sync` | Update `.ddd/mapping.yaml` with current implementation state |
| `/ddd-sync --discover` | Also find untracked code and suggest new flow specs |
| `/ddd-sync --fix-drift` | Re-implement flows where specs have drifted |
| `/ddd-sync --full` | All of the above |

**Workflow with sessions:**

```
Session A (Architect):          Human Review:           Session B (Developer):
  /ddd-create                    Open DDD Tool            /ddd-implement --all
  generates full spec            validate on canvas       generates code + tests
  structure                      refine, adjust           from specs

                                                         /ddd-update
                                                         modify specs from
                                                         natural language

                                                         /ddd-sync
                                                         keep specs & code aligned
```

---

**File: `src/types/implementation.ts`**
```typescript
export interface FlowMapping {
  spec: string;
  specHash: string;
  files: string[];
  fileHashes?: Record<string, string>;
  implementedAt: string;
  mode: 'new' | 'update';
}

export interface DriftInfo {
  flowKey: string;
  flowName: string;
  domainId: string;
  specPath: string;
  previousHash: string;
  currentHash: string;
  implementedAt: string;
  detectedAt: string;
  direction: 'forward' | 'reverse';
}

export type ReconciliationAction = 'accept' | 'reimpl' | 'ignore';

export interface ReconciliationEntry {
  flowKey: string;
  action: ReconciliationAction;
  previousHash: string;
  newHash: string;
  resolvedAt: string;
}

export interface ReconciliationReport {
  id: string;
  timestamp: string;
  entries: ReconciliationEntry[];
  syncScoreBefore: number;
  syncScoreAfter: number;
}

export interface SyncScore {
  total: number;
  implemented: number;
  stale: number;
  pending: number;
  score: number;
}
```

**File: `src/stores/implementation-store.ts`**

The implementation store has been simplified to focus on drift detection and mapping persistence. The in-app Implementation Panel, PTY terminal, test runner, and queue UI have been removed. Implementation is now done via Claude Code CLI (`/ddd-implement`).

**State:**
- `mappings: Record<string, FlowMapping>` — flow-to-file mappings from `.ddd/mapping.yaml`
- `driftItems: DriftInfo[]` — detected spec/code drift items
- `syncScore: SyncScore` — overall project sync score
- `ignoredDrifts: Set<string>` — flow keys whose drift has been dismissed

**Actions:**
- `loadMappings(projectPath)` — reads `.ddd/mapping.yaml` and populates mappings
- `saveMappings(projectPath)` — writes current mappings back to `.ddd/mapping.yaml`
- `detectDrift(projectPath)` — compares spec hashes against stored hashes, populates driftItems
- `computeSyncScore()` — calculates total/implemented/stale/pending/score from mappings and drift
- `resolveFlow(flowKey, action)` — handles drift resolution: `'accept'` updates the stored hash, `'reimpl'` copies `/ddd-implement {flowKey}` to clipboard, `'ignore'` adds to ignoredDrifts
- `resolveAll(action)` — applies an action to all current drift items
- `saveReconciliationReport(projectPath, report)` — writes a reconciliation report to `.ddd/reconciliations/`
- `reset()` — clears all state
```

**File: `src/utils/prompt-builder.ts`**
```typescript
import { invoke } from '@tauri-apps/api/core';
import type { BuiltPrompt } from '../types/implementation';
import { useImplementationStore } from '../stores/implementation-store';

/**
 * Build an optimal Claude Code prompt for implementing a flow.
 * Resolves referenced schemas, detects agent flows, handles update mode.
 */
export async function buildPrompt(flowId: string, domain: string): Promise<BuiltPrompt> {
  const projectPath = await invoke<string>('get_project_path');
  const flowFileName = flowId.split('/').pop() ?? flowId;

  // Read the flow spec
  const flowSpecPath = `specs/domains/${domain}/flows/${flowFileName}.yaml`;
  const flowYaml = await invoke<string>('read_file', { path: `${projectPath}/${flowSpecPath}` });
  const flowSpec = (await import('yaml')).parse(flowYaml);

  // Detect flow type
  const isAgent = flowSpec.flow?.type === 'agent';

  // Find referenced schemas
  const schemas = extractReferencedSchemas(flowSpec);

  // Check if this is an update (existing mapping)
  const { mappings, staleResults } = useImplementationStore.getState();
  const existingMapping = mappings[flowId];
  const stale = staleResults[flowId];
  const isUpdate = !!existingMapping;

  // Build spec file list
  const specFiles: string[] = [
    'specs/architecture.yaml',
    'specs/shared/errors.yaml',
    ...schemas.map(s => `specs/schemas/${s}.yaml`),
    `specs/domains/${domain}/flows/${flowFileName}.yaml`,
  ];

  // Build prompt content
  let content = `# ${isUpdate ? 'Update' : 'Implement'}: ${flowId}\n\n`;
  content += `Read these spec files in order:\n\n`;
  content += `1. specs/architecture.yaml — Project structure, conventions, dependencies\n`;
  content += `2. specs/shared/errors.yaml — Error codes and messages\n`;
  schemas.forEach((s, i) => {
    content += `${i + 3}. specs/schemas/${s}.yaml — Data model\n`;
  });
  content += `${schemas.length + 3}. specs/domains/${domain}/flows/${flowFileName}.yaml — The flow to implement\n\n`;

  content += `## Instructions\n\n`;

  if (isUpdate && stale) {
    content += `This flow was previously implemented. The spec has changed:\n`;
    stale.changes.forEach(c => { content += `- ${c}\n`; });
    content += `\nUpdate the existing code to match the new spec.\n`;
    content += `Do NOT rewrite files from scratch — modify the existing implementation.\n`;
    content += `Update affected tests.\n\n`;
  } else {
    content += `Implement the ${flowId} flow following architecture.yaml exactly:\n\n`;
    content += `- Create the endpoint matching the trigger spec`;
    if (flowSpec.trigger?.method && flowSpec.trigger?.path) {
      content += ` (method: ${flowSpec.trigger.method}, path: ${flowSpec.trigger.path})`;
    }
    content += `\n`;
    content += `- Create request/response schemas matching the input node validations\n`;
    content += `- Implement each node as described in the spec\n`;
    content += `- Use EXACT error codes from errors.yaml (do not invent new ones)\n`;
    content += `- Use EXACT validation messages from the flow spec\n`;
    content += `- Create unit tests covering: happy path, each validation failure, each error path\n\n`;
  }

  content += `## File locations\n\n`;
  content += `- Implementation: src/domains/${domain}/\n`;
  content += `- Tests: tests/unit/domains/${domain}/\n\n`;

  content += `## After implementation\n\n`;
  content += `Update .ddd/mapping.yaml with the flow mapping.\n\n`;

  // Implementation Report instruction for reverse drift detection
  content += `## Implementation Report (required)\n\n`;
  content += `After implementing, output a section titled \`## Implementation Notes\` with:\n`;
  content += `1. **Deviations** — Anything you did differently from the spec\n`;
  content += `2. **Additions** — Anything you added that the spec didn't mention\n`;
  content += `3. **Ambiguities resolved** — Anything the spec was unclear about and how you decided\n`;
  content += `4. **Schema changes** — Any new fields, changed types, or migration implications\n\n`;
  content += `If you followed the spec exactly with no changes, write: "No deviations."\n`;

  if (isAgent) {
    content += `\n## Agent-specific\n\n`;
    content += `This is an agent flow. Implement:\n`;
    content += `- Agent runner with the agent loop configuration\n`;
    content += `- Tool implementations for each tool defined in the spec\n`;
    content += `- Guardrail middleware for input/output filtering\n`;
    content += `- Memory management per the memory spec\n`;
    content += `- Use mocked LLM responses in tests\n`;
  }

  return {
    flowId,
    domain,
    content,
    specFiles,
    mode: isUpdate ? 'update' : 'new',
    agentFlow: isAgent,
    editedByUser: false,
  };
}

/** Extract schema names referenced by a flow spec via $ref and data_store model fields. */
function extractReferencedSchemas(flowSpec: any): string[] {
  const refs: string[] = [];
  const yamlStr = JSON.stringify(flowSpec);

  // $ref patterns
  const refMatches = yamlStr.matchAll(/\$ref[":]*\/?schemas\/(\w+)/g);
  for (const match of refMatches) refs.push(match[1]);

  // data_store model references
  for (const node of flowSpec.nodes ?? []) {
    if (node.spec?.model && typeof node.spec.model === 'string') {
      refs.push(node.spec.model);
    }
  }

  return [...new Set(refs)];
}
```

**File: `src/utils/claude-md-generator.ts`**
```typescript
import { invoke } from '@tauri-apps/api/core';
import { useImplementationStore } from '../stores/implementation-store';
import { useProjectStore } from '../stores/project-store';

/**
 * Generate CLAUDE.md content from project state.
 * Preserves any custom section the user has added.
 */
export async function generateClaudeMd(projectPath: string): Promise<string> {
  const project = useProjectStore.getState();
  const { mappings, staleResults } = useImplementationStore.getState();

  const systemConfig = project.systemConfig;
  const domains = project.domains ?? [];

  // Read existing CLAUDE.md to preserve custom section
  let customSection = '';
  try {
    const existing = await invoke<string>('read_file', { path: `${projectPath}/CLAUDE.md` });
    const customMarker = '<!-- CUSTOM: Add your own instructions below this line. They won\'t be overwritten. -->';
    const customIndex = existing.indexOf(customMarker);
    if (customIndex !== -1) {
      customSection = existing.slice(customIndex);
    }
  } catch {
    // No existing CLAUDE.md
  }

  // Read architecture.yaml for tech stack and folder structure
  let techStack: any = systemConfig?.tech_stack ?? {};
  let architectureYaml: any = {};
  try {
    const archRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/architecture.yaml` });
    architectureYaml = (await import('yaml')).parse(archRaw);
  } catch { /* ignore */ }

  // Count error codes
  let errorCodeCount = 0;
  try {
    const errRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/shared/errors.yaml` });
    const errYaml = (await import('yaml')).parse(errRaw);
    errorCodeCount = Object.values(errYaml).reduce((sum: number, cat: any) =>
      sum + (typeof cat === 'object' ? Object.keys(cat).length : 0), 0);
  } catch { /* ignore */ }

  // Count schemas
  let schemaNames: string[] = [];
  try {
    const schemaFiles = await invoke<string[]>('list_files', { dir: `${projectPath}/specs/schemas`, pattern: '*.yaml' });
    schemaNames = schemaFiles.map(f => f.replace('.yaml', '').replace('_base', 'Base'));
  } catch { /* ignore */ }

  // Build domain table
  const domainRows = domains.map(d => {
    const domainFlows = (d as any).flows ?? [];
    const implemented = domainFlows.filter((f: any) => {
      const fullId = `${d.name}/${f.id}`;
      return mappings[fullId] && !staleResults[fullId]?.isStale;
    }).length;
    const pending = domainFlows.length - implemented;
    const flowNames = domainFlows.map((f: any) => {
      const type = f.type === 'agent' ? ' (agent)' : '';
      return `${f.id}${type}`;
    }).join(', ');
    return `| ${d.name} | ${flowNames} | ${implemented} implemented, ${pending} pending |`;
  });

  // Build commands from architecture
  const commands = architectureYaml.testing?.commands ?? {};

  let md = `<!-- Auto-generated by DDD Tool. Manual edits below the CUSTOM section are preserved. -->\n\n`;
  md += `# Project: ${systemConfig?.name ?? 'Unknown'}\n\n`;
  md += `## Spec-Driven Development\n\n`;
  md += `This project uses Design Driven Development (DDD). All business logic\n`;
  md += `is specified in YAML files under \`specs/\`. Code MUST match specs exactly.\n\n`;

  md += `## Spec Files\n\n`;
  md += `- \`specs/system.yaml\` — Project identity, tech stack, ${domains.length} domains\n`;
  md += `- \`specs/architecture.yaml\` — Folder structure, conventions, dependencies\n`;
  md += `- \`specs/config.yaml\` — Environment variables schema\n`;
  md += `- \`specs/shared/errors.yaml\` — ${errorCodeCount} error codes\n`;
  if (schemaNames.length) {
    md += `- \`specs/schemas/*.yaml\` — ${schemaNames.length} data models (${schemaNames.join(', ')})\n`;
  }
  md += `\n`;

  md += `## Domains\n\n`;
  md += `| Domain | Flows | Status |\n`;
  md += `|--------|-------|--------|\n`;
  domainRows.forEach(row => { md += `${row}\n`; });
  md += `\n`;

  md += `## Implementation Rules\n\n`;
  md += `1. **Read architecture.yaml first** — it defines folder structure and conventions\n`;
  md += `2. **Follow the folder layout** — put files where architecture.yaml specifies\n`;
  md += `3. **Use EXACT error codes** — from specs/shared/errors.yaml, do not invent new ones\n`;
  md += `4. **Use EXACT validation messages** — from the flow spec, do not rephrase\n`;
  md += `5. **Match field types exactly** — spec field types map to language types\n`;
  md += `6. **Update .ddd/mapping.yaml** — after implementing a flow, record the mapping\n\n`;

  md += `## Tech Stack\n\n`;
  md += `- Language: ${techStack.language ?? 'unknown'} ${techStack.language_version ?? ''}\n`;
  md += `- Framework: ${techStack.framework ?? 'unknown'}\n`;
  if (techStack.orm) md += `- ORM: ${techStack.orm}\n`;
  if (techStack.database) md += `- Database: ${techStack.database}\n`;
  if (techStack.cache) md += `- Cache: ${techStack.cache}\n`;
  if (techStack.queue) md += `- Queue: ${techStack.queue}\n`;
  md += `\n`;

  md += `## Commands\n\n\`\`\`bash\n`;
  md += `${commands.test ?? 'pytest'}                    # Run tests\n`;
  md += `${commands.typecheck ?? 'mypy src/'}             # Type check\n`;
  md += `${commands.lint ?? 'ruff check .'}               # Lint\n`;
  md += `\`\`\`\n\n`;

  if (!customSection) {
    customSection = `<!-- CUSTOM: Add your own instructions below this line. They won't be overwritten. -->\n`;
  }
  md += customSection;

  return md;
}
```

**File: `src-tauri/src/commands/pty.rs`**
```rust
use std::collections::HashMap;
use std::sync::Mutex;
use tauri::State;
use portable_pty::{native_pty_system, PtySize, CommandBuilder};
use std::io::{Read, Write};
use std::sync::Arc;

struct PtyEntry {
    writer: Arc<Mutex<Box<dyn Write + Send>>>,
    reader: Arc<Mutex<Box<dyn Read + Send>>>,
    child: Arc<Mutex<Box<dyn portable_pty::Child + Send>>>,
}

pub struct PtyManager {
    sessions: Mutex<HashMap<String, PtyEntry>>,
}

impl PtyManager {
    pub fn new() -> Self {
        Self { sessions: Mutex::new(HashMap::new()) }
    }
}

#[tauri::command]
pub async fn pty_spawn(
    command: String,
    args: Vec<String>,
    cwd: String,
    manager: State<'_, PtyManager>,
) -> Result<String, String> {
    let pty_system = native_pty_system();
    let pair = pty_system.openpty(PtySize {
        rows: 24,
        cols: 80,
        pixel_width: 0,
        pixel_height: 0,
    }).map_err(|e| e.to_string())?;

    let mut cmd = CommandBuilder::new(&command);
    for arg in &args {
        cmd.arg(arg);
    }
    cmd.cwd(&cwd);

    let child = pair.slave.spawn_command(cmd).map_err(|e| e.to_string())?;
    let reader = pair.master.try_clone_reader().map_err(|e| e.to_string())?;
    let writer = pair.master.take_writer().map_err(|e| e.to_string())?;

    let session_id = nanoid::nanoid!(10);

    let entry = PtyEntry {
        writer: Arc::new(Mutex::new(writer)),
        reader: Arc::new(Mutex::new(reader)),
        child: Arc::new(Mutex::new(child)),
    };

    manager.sessions.lock().unwrap().insert(session_id.clone(), entry);
    Ok(session_id)
}

#[tauri::command]
pub async fn pty_read(
    session_id: String,
    manager: State<'_, PtyManager>,
) -> Result<String, String> {
    let sessions = manager.sessions.lock().unwrap();
    let entry = sessions.get(&session_id).ok_or("Session not found")?;
    let mut buf = [0u8; 4096];
    let reader = entry.reader.clone();
    drop(sessions);

    let n = reader.lock().unwrap().read(&mut buf).map_err(|e| e.to_string())?;
    Ok(String::from_utf8_lossy(&buf[..n]).to_string())
}

#[tauri::command]
pub async fn pty_write(
    session_id: String,
    data: String,
    manager: State<'_, PtyManager>,
) -> Result<(), String> {
    let sessions = manager.sessions.lock().unwrap();
    let entry = sessions.get(&session_id).ok_or("Session not found")?;
    let writer = entry.writer.clone();
    drop(sessions);

    writer.lock().unwrap().write_all(data.as_bytes()).map_err(|e| e.to_string())?;
    Ok(())
}

#[tauri::command]
pub async fn pty_wait(
    session_id: String,
    manager: State<'_, PtyManager>,
) -> Result<i32, String> {
    let sessions = manager.sessions.lock().unwrap();
    let entry = sessions.get(&session_id).ok_or("Session not found")?;
    let child = entry.child.clone();
    drop(sessions);

    let status = child.lock().unwrap().wait().map_err(|e| e.to_string())?;
    // Clean up
    manager.sessions.lock().unwrap().remove(&session_id);
    Ok(status.exit_code() as i32)
}

#[tauri::command]
pub async fn pty_kill(
    session_id: String,
    manager: State<'_, PtyManager>,
) -> Result<(), String> {
    let sessions = manager.sessions.lock().unwrap();
    let entry = sessions.get(&session_id).ok_or("Session not found")?;
    let child = entry.child.clone();
    drop(sessions);

    child.lock().unwrap().kill().map_err(|e| e.to_string())?;
    manager.sessions.lock().unwrap().remove(&session_id);
    Ok(())
}
```

**File: `src-tauri/src/commands/test_runner.rs`**
```rust
use std::process::Command;
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Clone)]
pub struct TestCase {
    pub name: String,
    pub status: String,       // "passed" | "failed" | "error" | "skipped"
    pub duration_ms: f64,
    pub error_message: Option<String>,
    pub error_detail: Option<String>,
}

#[derive(Serialize, Deserialize)]
pub struct TestSummary {
    pub total: usize,
    pub passed: usize,
    pub failed: usize,
    pub errors: usize,
    pub duration_ms: f64,
    pub tests: Vec<TestCase>,
}

#[tauri::command]
pub async fn run_tests(
    command: String,
    args: Vec<String>,
    scoped: bool,
    scope_pattern: Option<String>,
    cwd: String,
) -> Result<TestSummary, String> {
    let mut cmd_args = args.clone();

    // Add scope filter if configured
    if scoped {
        if let Some(pattern) = &scope_pattern {
            cmd_args.push(pattern.clone());
        }
    }

    // Add JSON output for parseable results (pytest-specific)
    if command.contains("pytest") {
        cmd_args.push("--json-report".to_string());
        cmd_args.push("--json-report-file=-".to_string());
    }

    let output = Command::new(&command)
        .args(&cmd_args)
        .current_dir(&cwd)
        .output()
        .map_err(|e| format!("Failed to run tests: {}", e))?;

    let stdout = String::from_utf8_lossy(&output.stdout).to_string();
    let stderr = String::from_utf8_lossy(&output.stderr).to_string();

    // Parse test output (simplified — extend for different test frameworks)
    parse_test_output(&command, &stdout, &stderr)
}

fn parse_test_output(command: &str, stdout: &str, stderr: &str) -> Result<TestSummary, String> {
    // Basic parsing — counts passed/failed from output
    // In production, use JSON reporter output for accurate parsing
    let mut tests = Vec::new();
    let mut passed = 0;
    let mut failed = 0;
    let mut errors = 0;

    for line in stdout.lines().chain(stderr.lines()) {
        if line.contains("PASSED") || line.contains("passed") {
            passed += 1;
            let name = line.split_whitespace().next().unwrap_or("unknown").to_string();
            tests.push(TestCase {
                name,
                status: "passed".to_string(),
                duration_ms: 0.0,
                error_message: None,
                error_detail: None,
            });
        } else if line.contains("FAILED") || line.contains("failed") {
            failed += 1;
            let name = line.split_whitespace().next().unwrap_or("unknown").to_string();
            tests.push(TestCase {
                name,
                status: "failed".to_string(),
                duration_ms: 0.0,
                error_message: Some(line.to_string()),
                error_detail: None,
            });
        } else if line.contains("ERROR") {
            errors += 1;
        }
    }

    Ok(TestSummary {
        total: passed + failed + errors,
        passed,
        failed,
        errors,
        duration_ms: 0.0,
        tests,
    })
}

#[tauri::command]
pub async fn hash_file(path: String) -> Result<String, String> {
    use sha2::{Sha256, Digest};
    let content = std::fs::read(&path).map_err(|e| format!("Failed to read {}: {}", path, e))?;
    let hash = Sha256::digest(&content);
    Ok(format!("{:x}", hash))
}

#[tauri::command]
pub async fn cache_spec(flow_id: String, spec_path: String) -> Result<(), String> {
    let project_path = std::env::current_dir().map_err(|e| e.to_string())?;
    let cache_dir = project_path.join(".ddd/cache/specs-at-implementation");
    std::fs::create_dir_all(&cache_dir).map_err(|e| e.to_string())?;

    let cache_name = flow_id.replace('/', "--") + ".yaml";
    std::fs::copy(&spec_path, cache_dir.join(&cache_name))
        .map_err(|e| format!("Failed to cache spec: {}", e))?;
    Ok(())
}

#[tauri::command]
pub async fn diff_specs(cached_path: String, current_path: String) -> Result<Vec<String>, String> {
    let cached = std::fs::read_to_string(&cached_path).unwrap_or_default();
    let current = std::fs::read_to_string(&current_path)
        .map_err(|e| format!("Failed to read current spec: {}", e))?;

    // Simple line-by-line diff — returns human-readable change descriptions
    let mut changes = Vec::new();
    let cached_lines: Vec<&str> = cached.lines().collect();
    let current_lines: Vec<&str> = current.lines().collect();

    for line in &current_lines {
        if !cached_lines.contains(line) && !line.trim().is_empty() {
            changes.push(format!("Added: {}", line.trim()));
        }
    }
    for line in &cached_lines {
        if !current_lines.contains(line) && !line.trim().is_empty() {
            changes.push(format!("Removed: {}", line.trim()));
        }
    }

    Ok(changes)
}
```

**File: `src/components/shared/CopyButton.tsx`**

Reusable copy-to-clipboard button used across all output text areas (terminal output, prompt preview, test errors). Uses the browser Clipboard API with visual feedback.

```tsx
import { useState, useCallback } from 'react';
import { Copy, Check } from 'lucide-react';

interface Props {
  text: string;        // Content to copy
  className?: string;  // Additional CSS classes for positioning
}

export function CopyButton({ text, className = '' }: Props) {
  const [copied, setCopied] = useState(false);

  const handleCopy = useCallback(async () => {
    try {
      await navigator.clipboard.writeText(text);
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    } catch {
      // Clipboard API may fail in some contexts
    }
  }, [text]);

  return (
    <button
      className={`flex items-center gap-1 px-1.5 py-0.5 rounded text-[10px] transition-colors ${
        copied
          ? 'text-success'
          : 'text-text-muted hover:text-text-primary hover:bg-bg-hover'
      } ${className}`}
      onClick={handleCopy}
      title="Copy to clipboard"
    >
      {copied ? <Check className="w-3 h-3" /> : <Copy className="w-3 h-3" />}
      {copied ? 'Copied' : 'Copy'}
    </button>
  );
}
```

**Usage patterns:**

1. **In a header bar**: place alongside other actions
   ```tsx
   <CopyButton text={content} />
   ```

2. **Absolute-positioned over a `<pre>` block**:
   ```tsx
   <div className="relative">
     <CopyButton text={content} className="absolute top-1 right-1" />
     <pre>...</pre>
   </div>
   ```

3. **Inline with a label**:
   ```tsx
   <div className="flex items-center justify-between">
     <p className="text-[10px] font-medium">Output</p>
     <CopyButton text={JSON.stringify(data, null, 2)} />
   </div>
   ```

**File: `src/components/shared/ClaudeCommandBox.tsx`**

Auto-generates the correct Claude Code slash command based on the current navigation scope. Shows a copyable command box with Terminal icon.

```tsx
import { Terminal } from 'lucide-react';
import { useSheetStore } from '../../stores/sheet-store';
import { CopyButton } from '../shared/CopyButton';

interface Props {
  /** Override the auto-detected scope (e.g. for fix commands) */
  command?: string;
}

export function ClaudeCommandBox({ command: commandOverride }: Props) {
  const current = useSheetStore((s) => s.current);

  let command: string;
  if (commandOverride) {
    command = commandOverride;
  } else if (current.level === 'flow' && current.domainId && current.flowId) {
    command = `/ddd-implement ${current.domainId}/${current.flowId}`;
  } else if (current.level === 'domain' && current.domainId) {
    command = `/ddd-implement ${current.domainId}`;
  } else {
    command = `/ddd-implement --all`;
  }

  return (
    <div className="border border-border rounded bg-bg-primary">
      <div className="flex items-center gap-1.5 px-3 py-1.5 border-b border-border">
        <Terminal className="w-3 h-3 text-text-muted" />
        <span className="text-[10px] text-text-muted flex-1">Claude Code command</span>
        <CopyButton text={command} />
      </div>
      <div className="px-3 py-2">
        <code className="text-xs font-mono text-accent select-all">{command}</code>
      </div>
    </div>
  );
}
```

---

**File: `src/components/ProjectLauncher/CloneDialog.tsx`**

Modal dialog for cloning Git repositories with optional token authentication for private repos.

```tsx
// Key features:
// - Repository URL input
// - Optional Personal Access Token field (password type)
// - Destination folder with Browse button (uses @tauri-apps/plugin-dialog open())
// - Token injection: embeds token into HTTPS URL as https://<token>@host/repo
// - Uses system git binary via invoke('git_clone', { url, path })
// - Error display and loading state
```

**Token handling:**
```tsx
let cloneUrl = url.trim();
if (token.trim()) {
  try {
    const parsed = new URL(cloneUrl);
    if (parsed.protocol === 'https:' || parsed.protocol === 'http:') {
      parsed.username = token.trim();
      parsed.password = '';
      cloneUrl = parsed.toString();
    }
  } catch { /* ignore parse errors */ }
}
await invoke('git_clone', { url: cloneUrl, path: destination });
```

### Diagram-Derived Test Generation

**File: `src/types/test-generator.ts`**
```typescript
/** A path through the flow graph from trigger to terminal */
export interface TestPath {
  path_id: string;
  path_type: 'happy_path' | 'error_path' | 'edge_case' | 'agent_loop';
  nodes: string[];                  // ordered node IDs traversed
  description: string;              // human-readable description
  expected_outcome: {
    status?: number;                // HTTP status code (for HTTP flows)
    error_code?: string;            // from errors.yaml
    response_fields?: string[];     // expected fields in response body
    message?: string;               // expected message/error text
  };
}

/** A boundary test derived from validation rules */
export interface BoundaryTest {
  field: string;
  test_type: 'valid' | 'invalid_missing' | 'invalid_format' | 'invalid_type'
    | 'boundary_min_below' | 'boundary_min_exact' | 'boundary_max_exact' | 'boundary_max_above';
  input_value: any;                 // the test value (or null for missing)
  expected_success: boolean;
  expected_error?: string;          // exact error message from spec
  expected_status?: number;
}

/** A derived test case (path test or boundary test) */
export interface DerivedTestCase {
  id: string;
  source: 'path' | 'boundary' | 'agent' | 'orchestration';
  path?: TestPath;
  boundary?: BoundaryTest;
  description: string;
  priority: 'critical' | 'important' | 'nice_to_have';
}

/** Complete test specification for a flow */
export interface DerivedTestSpec {
  flow_id: string;
  generated_at: string;
  paths: TestPath[];
  boundary_tests: BoundaryTest[];
  test_cases: DerivedTestCase[];
  total_count: number;
  coverage: {
    paths_covered: number;
    total_paths: number;
    boundary_fields_covered: number;
    total_fields: number;
  };
}

/** Generated test code (actual code string) */
export interface GeneratedTestCode {
  flow_id: string;
  framework: string;               // pytest, jest, go_test, etc.
  code: string;                     // the generated test file content
  test_count: number;
  generated_at: string;
}

/** Spec compliance result for a single test case */
export interface ComplianceResult {
  test_id: string;
  description: string;
  compliant: boolean;
  expected: { status?: number; message?: string; error_code?: string };
  actual?: { status?: number; message?: string; error_code?: string };
  diff?: string;                    // human-readable diff
}

/** Full spec compliance report */
export interface SpecComplianceReport {
  flow_id: string;
  checked_at: string;
  score: number;                    // 0-100
  total: number;
  compliant: number;
  non_compliant: number;
  results: ComplianceResult[];
}

/** Test generation state in the implementation store */
export interface TestGenerationState {
  derivedSpec: DerivedTestSpec | null;
  generatedCode: GeneratedTestCode | null;
  complianceReport: SpecComplianceReport | null;
  isDeriving: boolean;
  isGeneratingCode: boolean;
  isCheckingCompliance: boolean;
}
```

**File: `src/utils/test-case-deriver.ts`**
```typescript
import { DddNode, DddFlow } from '../types/flow';
import { TestPath, BoundaryTest, DerivedTestCase, DerivedTestSpec } from '../types/test-generator';

/**
 * Walk a flow graph and enumerate all paths from trigger to terminal nodes.
 * Uses DFS with path tracking to find every reachable terminal.
 */
export function deriveTestPaths(flow: DddFlow): TestPath[] {
  const paths: TestPath[] = [];
  const nodeMap = new Map(flow.nodes.map(n => [n.id, n]));

  // Find trigger node
  const trigger = flow.nodes.find(n => n.type === 'trigger');
  if (!trigger) return paths;

  // DFS to find all paths from trigger to terminal nodes
  function dfs(nodeId: string, visited: string[], pathNodes: string[]) {
    const node = nodeMap.get(nodeId);
    if (!node) return;

    const currentPath = [...pathNodes, nodeId];

    // If terminal node, record this path
    if (node.type === 'terminal') {
      const pathType = determinePathType(currentPath, nodeMap);
      paths.push({
        path_id: `path_${paths.length + 1}`,
        path_type: pathType,
        nodes: currentPath,
        description: describePathHumanReadable(currentPath, nodeMap),
        expected_outcome: extractExpectedOutcome(node),
      });
      return;
    }

    // Prevent cycles
    if (visited.includes(nodeId)) return;
    const newVisited = [...visited, nodeId];

    // Follow all connections from this node
    const connections = node.connections || {};
    for (const [branch, targetId] of Object.entries(connections)) {
      if (typeof targetId === 'string') {
        dfs(targetId, newVisited, currentPath);
      }
    }
  }

  // Start DFS from trigger's first connection
  const triggerConnections = trigger.connections || {};
  for (const targetId of Object.values(triggerConnections)) {
    if (typeof targetId === 'string') {
      dfs(targetId, [trigger.id], [trigger.id]);
    }
  }

  return paths;
}

/**
 * Determine if a path is happy_path, error_path, or edge_case
 * based on the terminal node and intermediate decision branches taken.
 */
function determinePathType(
  path: string[],
  nodeMap: Map<string, DddNode>
): TestPath['path_type'] {
  const terminal = nodeMap.get(path[path.length - 1]);
  if (!terminal) return 'edge_case';

  const status = terminal.spec?.status;
  if (status && status >= 200 && status < 300) return 'happy_path';
  if (status && status >= 400) return 'error_path';

  // Check if path goes through any error-producing decision branches
  for (const nodeId of path) {
    const node = nodeMap.get(nodeId);
    if (node?.type === 'decision' && node.spec?.on_true?.error_code) {
      // This path took the error branch of a decision
      return 'error_path';
    }
  }

  return 'happy_path';
}

/**
 * Generate human-readable description of a path.
 */
function describePathHumanReadable(
  path: string[],
  nodeMap: Map<string, DddNode>
): string {
  return path
    .map(id => {
      const node = nodeMap.get(id);
      return node ? `${node.id} (${node.type})` : id;
    })
    .join(' → ');
}

/**
 * Extract expected outcome from a terminal node.
 */
function extractExpectedOutcome(node: DddNode) {
  return {
    status: node.spec?.status,
    error_code: node.spec?.error_code,
    response_fields: node.spec?.body ? Object.keys(node.spec.body) : undefined,
    message: node.spec?.body?.message,
  };
}

/**
 * Derive boundary tests from all input nodes in a flow.
 * For each field with validation rules, generates min/max/missing/format tests.
 */
export function deriveBoundaryTests(flow: DddFlow): BoundaryTest[] {
  const tests: BoundaryTest[] = [];

  const inputNodes = flow.nodes.filter(n => n.type === 'input');
  for (const node of inputNodes) {
    const fields = node.spec?.fields || {};
    for (const [fieldName, rules] of Object.entries(fields)) {
      const r = rules as any;

      // Valid case
      tests.push({
        field: fieldName,
        test_type: 'valid',
        input_value: generateValidValue(r),
        expected_success: true,
      });

      // Missing (if required)
      if (r.required) {
        tests.push({
          field: fieldName,
          test_type: 'invalid_missing',
          input_value: null,
          expected_success: false,
          expected_error: r.error,
          expected_status: 422,
        });
      }

      // Format validation
      if (r.format) {
        tests.push({
          field: fieldName,
          test_type: 'invalid_format',
          input_value: generateInvalidFormat(r.format),
          expected_success: false,
          expected_error: r.error,
          expected_status: 422,
        });
      }

      // Min length boundary
      if (r.min_length !== undefined) {
        // Below minimum
        tests.push({
          field: fieldName,
          test_type: 'boundary_min_below',
          input_value: 'x'.repeat(r.min_length - 1),
          expected_success: false,
          expected_error: r.error,
          expected_status: 422,
        });
        // At exact minimum
        tests.push({
          field: fieldName,
          test_type: 'boundary_min_exact',
          input_value: 'x'.repeat(r.min_length),
          expected_success: true,
        });
      }

      // Max length boundary
      if (r.max_length !== undefined) {
        // At exact maximum
        tests.push({
          field: fieldName,
          test_type: 'boundary_max_exact',
          input_value: 'x'.repeat(r.max_length),
          expected_success: true,
        });
        // Above maximum
        tests.push({
          field: fieldName,
          test_type: 'boundary_max_above',
          input_value: 'x'.repeat(r.max_length + 1),
          expected_success: false,
          expected_error: r.error,
          expected_status: 422,
        });
      }
    }
  }

  return tests;
}

/** Generate a valid value for a field based on its type and format */
function generateValidValue(rules: any): any {
  if (rules.format === 'email') return 'user@example.com';
  if (rules.type === 'number' || rules.type === 'integer') return 42;
  if (rules.type === 'boolean') return true;
  if (rules.min_length) return 'x'.repeat(rules.min_length + 2);
  return 'test_value';
}

/** Generate an invalid value for a given format */
function generateInvalidFormat(format: string): any {
  switch (format) {
    case 'email': return 'not-an-email';
    case 'url': return 'not-a-url';
    case 'uuid': return 'not-a-uuid';
    case 'date': return 'not-a-date';
    default: return 'invalid';
  }
}

/**
 * Derive agent-specific test cases from an agent flow.
 */
export function deriveAgentTests(flow: DddFlow): DerivedTestCase[] {
  const tests: DerivedTestCase[] = [];
  if (flow.flow_type !== 'agent') return tests;

  const agent = flow.nodes.find(n => n.type === 'agent_loop');
  if (!agent) return tests;

  // Tool success/failure for each tool
  const tools = flow.nodes.filter(n => n.type === 'tool');
  for (const tool of tools) {
    tests.push({
      id: `agent_tool_success_${tool.id}`,
      source: 'agent',
      description: `Agent calls ${tool.id} → returns success`,
      priority: 'critical',
    });
    tests.push({
      id: `agent_tool_failure_${tool.id}`,
      source: 'agent',
      description: `Agent calls ${tool.id} → tool returns error → agent handles gracefully`,
      priority: 'important',
    });
  }

  // Guardrail tests
  const guardrails = flow.nodes.filter(n => n.type === 'guardrail');
  for (const guard of guardrails) {
    tests.push({
      id: `agent_guardrail_block_${guard.id}`,
      source: 'agent',
      description: `Agent output violates ${guard.id} → output blocked, agent retries`,
      priority: 'critical',
    });
  }

  // Max iterations test
  if (agent.spec?.max_iterations) {
    tests.push({
      id: 'agent_max_iterations',
      source: 'agent',
      description: `Agent reaches max_iterations (${agent.spec.max_iterations}) → returns partial/error`,
      priority: 'important',
    });
  }

  // Human gate tests
  const humanGates = flow.nodes.filter(n => n.type === 'human_gate');
  for (const gate of humanGates) {
    tests.push({
      id: `agent_human_gate_approve_${gate.id}`,
      source: 'agent',
      description: `Agent reaches ${gate.id} → user approves → continues`,
      priority: 'critical',
    });
    tests.push({
      id: `agent_human_gate_reject_${gate.id}`,
      source: 'agent',
      description: `Agent reaches ${gate.id} → user rejects → handles rejection`,
      priority: 'important',
    });
  }

  return tests;
}

/**
 * Derive orchestration-specific test cases from an orchestration flow.
 */
export function deriveOrchestrationTests(flow: DddFlow): DerivedTestCase[] {
  const tests: DerivedTestCase[] = [];
  if (flow.flow_type !== 'orchestration') return tests;

  const orchestrators = flow.nodes.filter(n => n.type === 'orchestrator');
  const routers = flow.nodes.filter(n => n.type === 'smart_router');
  const handoffs = flow.nodes.filter(n => n.type === 'handoff');

  // Routing tests per router
  for (const router of routers) {
    const rules = router.spec?.rules || [];
    for (const rule of rules) {
      tests.push({
        id: `routing_${router.id}_${rule.intent || rule.condition}`,
        source: 'orchestration',
        description: `Input matches "${rule.intent || rule.condition}" → routed to ${rule.target}`,
        priority: 'critical',
      });
    }
    // Fallback test
    if (router.spec?.fallback) {
      tests.push({
        id: `routing_fallback_${router.id}`,
        source: 'orchestration',
        description: `Input matches no rule → fallback to ${router.spec.fallback}`,
        priority: 'important',
      });
    }
    // Circuit breaker test
    if (router.spec?.circuit_breaker) {
      tests.push({
        id: `circuit_breaker_${router.id}`,
        source: 'orchestration',
        description: `Agent fails ${router.spec.circuit_breaker.failure_threshold} times → circuit opens → fallback`,
        priority: 'important',
      });
    }
  }

  // Handoff tests
  for (const handoff of handoffs) {
    tests.push({
      id: `handoff_${handoff.id}`,
      source: 'orchestration',
      description: `${handoff.spec?.mode || 'transfer'} handoff: context passed correctly to target agent`,
      priority: 'critical',
    });
  }

  // Supervisor override test
  for (const orch of orchestrators) {
    if (orch.spec?.strategy === 'supervisor') {
      tests.push({
        id: `supervisor_override_${orch.id}`,
        source: 'orchestration',
        description: `Supervisor detects stuck agent → intervenes → reassigns`,
        priority: 'nice_to_have',
      });
    }
  }

  return tests;
}

/**
 * Main entry: derive all test cases for a flow.
 */
export function deriveAllTests(flow: DddFlow): DerivedTestSpec {
  const paths = deriveTestPaths(flow);
  const boundaryTests = deriveBoundaryTests(flow);
  const agentTests = deriveAgentTests(flow);
  const orchestrationTests = deriveOrchestrationTests(flow);

  // Convert paths to test cases
  const pathCases: DerivedTestCase[] = paths.map((p, i) => ({
    id: `path_${i + 1}`,
    source: 'path' as const,
    path: p,
    description: p.description,
    priority: p.path_type === 'happy_path' ? 'critical' as const : 'important' as const,
  }));

  // Convert boundary tests to test cases
  const boundaryCases: DerivedTestCase[] = boundaryTests.map((b, i) => ({
    id: `boundary_${i + 1}`,
    source: 'boundary' as const,
    boundary: b,
    description: `${b.field}: ${b.test_type} → ${b.expected_success ? 'success' : 'error'}`,
    priority: b.test_type === 'invalid_missing' ? 'critical' as const : 'important' as const,
  }));

  const allCases = [...pathCases, ...boundaryCases, ...agentTests, ...orchestrationTests];

  const inputNodes = flow.nodes.filter(n => n.type === 'input');
  const totalFields = inputNodes.reduce(
    (sum, n) => sum + Object.keys(n.spec?.fields || {}).length, 0
  );

  return {
    flow_id: flow.id,
    generated_at: new Date().toISOString(),
    paths,
    boundary_tests: boundaryTests,
    test_cases: allCases,
    total_count: allCases.length,
    coverage: {
      paths_covered: paths.length,
      total_paths: paths.length,
      boundary_fields_covered: new Set(boundaryTests.map(b => b.field)).size,
      total_fields: totalFields,
    },
  };
}
```

**Store updates (`src/stores/implementation-store.ts` additions):**

Add to the implementation store state:

```typescript
// Test generation state
derivedSpec: DerivedTestSpec | null;
generatedTestCode: GeneratedTestCode | null;
complianceReport: SpecComplianceReport | null;
isDeriving: boolean;
isGeneratingCode: boolean;
isCheckingCompliance: boolean;

// Test generation actions
deriveTests: (flowId: string) => void;
generateTestCode: (flowId: string) => Promise<void>;
checkSpecCompliance: (flowId: string) => Promise<void>;
```

Implementation:

```typescript
deriveTests: (flowId) => {
  set({ isDeriving: true });
  const flow = useFlowStore.getState().getFlow(flowId);
  if (!flow) { set({ isDeriving: false }); return; }

  const spec = deriveAllTests(flow);
  set({ derivedSpec: spec, isDeriving: false });

  // Persist to mapping
  const mapping = get().mapping;
  if (mapping.flows[flowId]) {
    mapping.flows[flowId].test_generation = {
      derived_test_count: spec.total_count,
      paths_found: spec.paths.length,
      boundary_tests: spec.boundary_tests.length,
      last_generated_at: spec.generated_at,
    };
    get().saveMapping();
  }
},

generateTestCode: async (flowId) => {
  set({ isGeneratingCode: true });
  const derivedSpec = get().derivedSpec;
  if (!derivedSpec) { set({ isGeneratingCode: false }); return; }

  const flow = useFlowStore.getState().getFlow(flowId);
  const project = useProjectStore.getState();
  const architecture = project.architecture;

  // Build prompt for Claude Code to generate test code
  const testFramework = architecture?.testing?.framework || 'pytest';
  const prompt = buildTestCodePrompt(derivedSpec, flow, testFramework);

  // Write prompt to .ddd/.test-gen-prompt.md and execute via Claude Code
  await invoke('write_file', { path: `${projectPath}/.ddd/.test-gen-prompt.md`, content: prompt });
  const response = await invoke<string>('run_claude_code', { promptPath: `${projectPath}/.ddd/.test-gen-prompt.md` });

  const code = extractCodeBlock(response);
  set({
    generatedTestCode: {
      flow_id: flowId,
      framework: testFramework,
      code,
      test_count: derivedSpec.total_count,
      generated_at: new Date().toISOString(),
    },
    isGeneratingCode: false,
  });
},

checkSpecCompliance: async (flowId) => {
  set({ isCheckingCompliance: true });
  const derivedSpec = get().derivedSpec;
  const testResults = get().testResults;
  if (!derivedSpec || !testResults) { set({ isCheckingCompliance: false }); return; }

  const flow = useFlowStore.getState().getFlow(flowId);

  // Build prompt comparing expected outcomes against test results
  const prompt = buildCompliancePrompt(derivedSpec, testResults, flow);

  // Write prompt and execute via Claude Code
  await invoke('write_file', { path: `${projectPath}/.ddd/.compliance-prompt.md`, content: prompt });
  const response = await invoke<string>('run_claude_code', { promptPath: `${projectPath}/.ddd/.compliance-prompt.md` });

  const report = parseComplianceResponse(response, flowId);
  set({ complianceReport: report, isCheckingCompliance: false });

  // Persist to mapping
  const mapping = get().mapping;
  if (mapping.flows[flowId]) {
    mapping.flows[flowId].spec_compliance = {
      score: report.score,
      compliant: report.compliant,
      non_compliant: report.non_compliant,
      last_checked_at: report.checked_at,
      issues: report.results
        .filter(r => !r.compliant)
        .map(r => ({
          description: r.description,
          expected: r.expected,
          actual: r.actual,
          resolved: false,
        })),
    };
    get().saveMapping();
  }
},
```

**Helper functions:**

```typescript
/** Build prompt for Claude Code to generate test code from derived spec */
function buildTestCodePrompt(
  spec: DerivedTestSpec,
  flow: DddFlow,
  framework: string
): string {
  return `Generate ${framework} test code for the following flow.

## Flow Spec
\`\`\`yaml
${flowToYaml(flow)}
\`\`\`

## Derived Test Cases (${spec.total_count} total)

### Paths (${spec.paths.length}):
${spec.paths.map(p => `- ${p.path_id} (${p.path_type}): ${p.description}
  Expected: status=${p.expected_outcome.status}, ${p.expected_outcome.error_code || 'success'}`).join('\n')}

### Boundary Tests (${spec.boundary_tests.length}):
${spec.boundary_tests.map(b => `- ${b.field} ${b.test_type}: input=${JSON.stringify(b.input_value)} → ${b.expected_success ? 'success' : `error: "${b.expected_error}"`}`).join('\n')}

## Requirements
- Use ${framework} with async test client
- Test EXACT error messages from the spec (copy them verbatim)
- Test EXACT status codes from the spec
- Group tests by path (happy path, validation errors, business errors)
- Include boundary tests for all validation fields
- Use descriptive test names that reference the spec path
- Do NOT add tests beyond what the spec defines`;
}

/** Build prompt for compliance checking */
function buildCompliancePrompt(
  spec: DerivedTestSpec,
  testResults: TestSummary,
  flow: DddFlow
): string {
  return `Compare these test results against the flow spec expectations.

## Expected (from spec):
${spec.test_cases.map(tc => `- ${tc.id}: ${tc.description}
  Expected: ${JSON.stringify(tc.path?.expected_outcome || tc.boundary)}`).join('\n')}

## Actual Test Results:
${testResults.cases.map(tc => `- ${tc.name}: ${tc.passed ? 'PASS' : 'FAIL'}
  ${tc.error || 'no error'}`).join('\n')}

For each expected test case, determine if the actual result is compliant (matches the spec exactly) or non-compliant (differs in status code, error message, response shape, etc.).

Output JSON array:
[{ "test_id": "...", "description": "...", "compliant": true/false, "expected": {...}, "actual": {...}, "diff": "..." }]`;
}

/** Parse compliance response into structured report */
function parseComplianceResponse(content: string, flowId: string): SpecComplianceReport {
  const results = JSON.parse(extractJsonBlock(content)) as ComplianceResult[];
  const compliant = results.filter(r => r.compliant).length;
  return {
    flow_id: flowId,
    checked_at: new Date().toISOString(),
    score: Math.round((compliant / results.length) * 100),
    total: results.length,
    compliant,
    non_compliant: results.length - compliant,
    results,
  };
}
```

**Component: `src/components/ImplementationPanel/TestGenerator.tsx`**

```tsx
import { useImplementationStore } from '../../stores/implementation-store';

export function TestGenerator({ flowId }: { flowId: string }) {
  const {
    derivedSpec, generatedTestCode, isDeriving, isGeneratingCode,
    deriveTests, generateTestCode,
  } = useImplementationStore();

  return (
    <div className="test-generator">
      <div className="flex items-center justify-between mb-3">
        <h3 className="font-semibold">Test Specification</h3>
        <button
          onClick={() => deriveTests(flowId)}
          disabled={isDeriving}
          className="btn btn-sm"
        >
          {isDeriving ? 'Analyzing flow...' : '✨ Derive tests'}
        </button>
      </div>

      {derivedSpec && (
        <>
          <div className="stats mb-3 text-sm text-gray-600">
            {derivedSpec.total_count} test cases from {derivedSpec.paths.length} paths
            + {derivedSpec.boundary_tests.length} boundary tests
          </div>

          {/* Paths section */}
          <div className="mb-4">
            <h4 className="text-sm font-medium mb-2">Paths</h4>
            {derivedSpec.paths.map(path => (
              <div key={path.path_id} className="flex items-center gap-2 py-1 text-sm">
                <span className={
                  path.path_type === 'happy_path' ? 'text-green-600' :
                  path.path_type === 'error_path' ? 'text-red-600' : 'text-yellow-600'
                }>
                  {path.path_type === 'happy_path' ? '✓' : path.path_type === 'error_path' ? '✗' : '~'}
                </span>
                <span>{path.description}</span>
                <span className="text-gray-400 ml-auto">
                  → {path.expected_outcome.status || '?'}
                </span>
              </div>
            ))}
          </div>

          {/* Boundary tests section */}
          <div className="mb-4">
            <h4 className="text-sm font-medium mb-2">Boundary Tests</h4>
            {derivedSpec.boundary_tests.map((bt, i) => (
              <div key={i} className="flex items-center gap-2 py-1 text-sm">
                <span className={bt.expected_success ? 'text-green-600' : 'text-red-600'}>
                  {bt.expected_success ? '✓' : '✗'}
                </span>
                <span>{bt.field}: {bt.test_type}</span>
              </div>
            ))}
          </div>

          {/* Generate code button */}
          <div className="flex gap-2">
            <button
              onClick={() => generateTestCode(flowId)}
              disabled={isGeneratingCode}
              className="btn btn-primary btn-sm"
            >
              {isGeneratingCode ? 'Generating...' : 'Generate test code'}
            </button>
          </div>

          {/* Generated code preview */}
          {generatedTestCode && (
            <div className="mt-3">
              <h4 className="text-sm font-medium mb-2">
                Generated Test Code ({generatedTestCode.framework})
              </h4>
              <pre className="bg-gray-50 p-3 rounded text-xs overflow-auto max-h-64">
                {generatedTestCode.code}
              </pre>
              <div className="flex gap-2 mt-2">
                <button
                  onClick={() => {
                    // Include in prompt builder
                    useImplementationStore.getState().includeTestsInPrompt(generatedTestCode.code);
                  }}
                  className="btn btn-sm"
                >
                  Include in prompt
                </button>
              </div>
            </div>
          )}
        </>
      )}
    </div>
  );
}
```

**Component: `src/components/ImplementationPanel/CoverageBadge.tsx`**

```tsx
interface CoverageBadgeProps {
  specTestCount: number;   // tests derived from spec
  actualTestCount: number; // tests actually passing
  onClick?: () => void;
}

export function CoverageBadge({ specTestCount, actualTestCount, onClick }: CoverageBadgeProps) {
  const ratio = actualTestCount / specTestCount;
  const color = ratio >= 1 ? 'bg-green-100 text-green-800'
    : ratio >= 0.7 ? 'bg-yellow-100 text-yellow-800'
    : 'bg-red-100 text-red-800';
  const icon = ratio >= 1 ? '📋' : '⚠';

  return (
    <button onClick={onClick} className={`inline-flex items-center gap-1 px-2 py-0.5 rounded text-xs ${color}`}>
      {icon} {actualTestCount}/{specTestCount} spec tests
    </button>
  );
}
```

**Component: `src/components/ImplementationPanel/SpecComplianceTab.tsx`**

```tsx
import { useImplementationStore } from '../../stores/implementation-store';

export function SpecComplianceTab({ flowId }: { flowId: string }) {
  const {
    complianceReport, isCheckingCompliance,
    checkSpecCompliance,
  } = useImplementationStore();

  return (
    <div className="spec-compliance">
      <div className="flex items-center justify-between mb-3">
        <h3 className="font-semibold">Spec Compliance</h3>
        <button
          onClick={() => checkSpecCompliance(flowId)}
          disabled={isCheckingCompliance}
          className="btn btn-sm"
        >
          {isCheckingCompliance ? 'Checking...' : 'Check compliance'}
        </button>
      </div>

      {complianceReport && (
        <>
          <div className="mb-3 text-lg font-bold">
            Compliance: {complianceReport.compliant}/{complianceReport.total} ({complianceReport.score}%)
          </div>

          {/* Compliant items */}
          <div className="mb-4">
            <h4 className="text-sm font-medium text-green-700 mb-2">Compliant</h4>
            {complianceReport.results.filter(r => r.compliant).map(r => (
              <div key={r.test_id} className="flex items-center gap-2 py-1 text-sm">
                <span className="text-green-600">✓</span>
                <span>{r.description}</span>
              </div>
            ))}
          </div>

          {/* Non-compliant items */}
          {complianceReport.non_compliant > 0 && (
            <div className="mb-4">
              <h4 className="text-sm font-medium text-red-700 mb-2">Non-compliant</h4>
              {complianceReport.results.filter(r => !r.compliant).map(r => (
                <div key={r.test_id} className="border border-red-200 rounded p-2 mb-2">
                  <div className="flex items-center gap-2 text-sm">
                    <span className="text-red-600">⚠</span>
                    <span className="font-medium">{r.description}</span>
                  </div>
                  {r.diff && (
                    <div className="text-xs text-gray-600 mt-1">{r.diff}</div>
                  )}
                  <div className="text-xs mt-1">
                    Expected: {JSON.stringify(r.expected)} | Actual: {JSON.stringify(r.actual)}
                  </div>
                  <button
                    onClick={() => {
                      // Build fix prompt targeting this specific issue
                      useImplementationStore.getState().fixComplianceIssue(r);
                    }}
                    className="btn btn-xs btn-danger mt-1"
                  >
                    Fix via Claude Code
                  </button>
                </div>
              ))}

              <button
                onClick={() => {
                  const issues = complianceReport.results.filter(r => !r.compliant);
                  useImplementationStore.getState().fixAllComplianceIssues(issues);
                }}
                className="btn btn-sm btn-danger mt-2"
              >
                Fix all non-compliant
              </button>
            </div>
          )}
        </>
      )}
    </div>
  );
}
```

**Note:** The TestGenerator, CoverageBadge, and SpecComplianceTab components were previously housed in the ImplementationPanel directory. With the removal of the in-app Implementation Panel, test generation and compliance checking are now done via the Claude Code CLI (`/ddd-implement` includes derived test specs when available).

---

### App Shell & UX Fundamentals

**File: `src/types/app.ts`**
```typescript
/** Recent project entry */
export interface RecentProject {
  name: string;
  path: string;
  lastOpenedAt: string;
  description?: string;
}

/** New project wizard state */
export interface NewProjectConfig {
  name: string;
  location: string;
  description: string;
  initGit: boolean;
  techStack: {
    language: string;
    languageVersion: string;
    framework: string;
    database: string;
    orm: string;
    cache?: string;
  };
  domains: Array<{ name: string; description: string }>;
}

/** App-level view state */
export type AppView = 'launcher' | 'first-run' | 'project';

/** Global settings (stored in ~/.ddd-tool/settings.json) */
export interface GlobalSettings {
  claudeCode: {
    enabled: boolean;
    command: string;                        // CLI path
  };
  editor: {
    gridSnap: boolean;
    autoSaveInterval: number;               // seconds, 0 = disabled
    theme: 'light' | 'dark' | 'system';
    fontSize: number;
  };
  git: {
    autoCommitMessage: string;              // template with {flow_id}, {action}
    branchNaming: string;                   // template
  };
}

/** Undo/redo snapshot for a flow */
export interface FlowSnapshot {
  nodes: any[];                             // deep copy of DddNode[]
  connections: any[];                       // deep copy of Connection[]
  specValues: Record<string, any>;          // spec field values per node
  timestamp: number;
  description: string;                      // "Added process node", "Moved node", etc.
}

/** Undo/redo state per flow */
export interface UndoState {
  undoStack: FlowSnapshot[];
  redoStack: FlowSnapshot[];
  maxHistory: number;                       // default 100
}

/** Error notification */
export interface AppError {
  id: string;
  severity: 'info' | 'warning' | 'error' | 'fatal';
  component: string;                        // which subsystem: 'file', 'git', 'pty', 'canvas'
  message: string;
  detail?: string;
  recoveryAction?: {
    label: string;
    action: () => void;
  };
  timestamp: number;
  dismissed: boolean;
}
```

**File: `src/stores/app-store.ts`**
```typescript
import { create } from 'zustand';
import { invoke } from '@tauri-apps/api/core';
import {
  AppView, RecentProject, NewProjectConfig,
  GlobalSettings, AppError,
} from '../types/app';

interface AppStore {
  // View state
  view: AppView;
  setView: (view: AppView) => void;

  // Recent projects
  recentProjects: RecentProject[];
  loadRecentProjects: () => Promise<void>;
  addRecentProject: (project: RecentProject) => void;
  removeRecentProject: (path: string) => void;

  // First-run
  isFirstRun: boolean;
  checkFirstRun: () => Promise<void>;
  completeFirstRun: () => void;

  // Settings
  globalSettings: GlobalSettings;
  loadGlobalSettings: () => Promise<void>;
  saveGlobalSettings: (settings: Partial<GlobalSettings>) => Promise<void>;

  // Project creation
  createProject: (config: NewProjectConfig) => Promise<string>;

  // Error handling
  errors: AppError[];
  pushError: (error: Omit<AppError, 'id' | 'timestamp' | 'dismissed'>) => void;
  dismissError: (id: string) => void;
  clearErrors: () => void;

  // Auto-save
  autoSaveTimer: ReturnType<typeof setInterval> | null;
  startAutoSave: (intervalSeconds: number) => void;
  stopAutoSave: () => void;
}

export const useAppStore = create<AppStore>((set, get) => ({
  view: 'launcher',
  setView: (view) => set({ view }),

  recentProjects: [],
  loadRecentProjects: async () => {
    try {
      const data = await invoke<string>('read_file', {
        path: '~/.ddd-tool/recent-projects.json',
      });
      const projects: RecentProject[] = JSON.parse(data);
      // Prune entries where folder no longer exists
      const valid: RecentProject[] = [];
      for (const p of projects) {
        const exists = await invoke<boolean>('path_exists', { path: p.path });
        if (exists) valid.push(p);
      }
      set({ recentProjects: valid.slice(0, 20) });
    } catch {
      set({ recentProjects: [] });
    }
  },

  addRecentProject: (project) => {
    const current = get().recentProjects.filter(p => p.path !== project.path);
    const updated = [project, ...current].slice(0, 20);
    set({ recentProjects: updated });
    // Persist
    invoke('write_file', {
      path: '~/.ddd-tool/recent-projects.json',
      content: JSON.stringify(updated, null, 2),
    });
  },

  removeRecentProject: (path) => {
    const updated = get().recentProjects.filter(p => p.path !== path);
    set({ recentProjects: updated });
    invoke('write_file', {
      path: '~/.ddd-tool/recent-projects.json',
      content: JSON.stringify(updated, null, 2),
    });
  },

  isFirstRun: false,
  checkFirstRun: async () => {
    try {
      const exists = await invoke<boolean>('path_exists', {
        path: '~/.ddd-tool/settings.json',
      });
      set({ isFirstRun: !exists, view: exists ? 'launcher' : 'first-run' });
    } catch {
      set({ isFirstRun: true, view: 'first-run' });
    }
  },

  completeFirstRun: () => {
    set({ isFirstRun: false, view: 'launcher' });
  },

  globalSettings: {} as GlobalSettings,
  loadGlobalSettings: async () => {
    try {
      const data = await invoke<string>('read_file', {
        path: '~/.ddd-tool/settings.json',
      });
      set({ globalSettings: JSON.parse(data) });
    } catch {
      // Use defaults
      set({ globalSettings: getDefaultSettings() });
    }
  },

  saveGlobalSettings: async (partial) => {
    const merged = { ...get().globalSettings, ...partial };
    set({ globalSettings: merged });
    await invoke('write_file', {
      path: '~/.ddd-tool/settings.json',
      content: JSON.stringify(merged, null, 2),
    });
  },

  createProject: async (config) => {
    // 1. Create directory
    await invoke('create_directory', { path: config.location });

    // 2. Create specs/ structure
    const specsDir = `${config.location}/specs`;
    await invoke('create_directory', { path: specsDir });
    await invoke('create_directory', { path: `${specsDir}/domains` });
    await invoke('create_directory', { path: `${specsDir}/schemas` });
    await invoke('create_directory', { path: `${specsDir}/shared` });

    // 3. Generate system.yaml
    const systemYaml = generateSystemYaml(config);
    await invoke('write_file', { path: `${specsDir}/system.yaml`, content: systemYaml });

    // 4. Copy + customize templates (architecture, config, errors)
    const archYaml = generateArchitectureYaml(config.techStack);
    await invoke('write_file', { path: `${specsDir}/architecture.yaml`, content: archYaml });
    await invoke('write_file', { path: `${specsDir}/config.yaml`, content: generateConfigYaml(config) });
    await invoke('write_file', { path: `${specsDir}/shared/errors.yaml`, content: getErrorsTemplate() });

    // 5. Create domain directories + domain.yaml per domain
    for (const domain of config.domains) {
      const domainDir = `${specsDir}/domains/${domain.name}`;
      await invoke('create_directory', { path: domainDir });
      await invoke('create_directory', { path: `${domainDir}/flows` });
      await invoke('write_file', {
        path: `${domainDir}/domain.yaml`,
        content: generateDomainYaml(domain),
      });
    }

    // 6. Create .ddd/ directory
    await invoke('create_directory', { path: `${config.location}/.ddd` });
    await invoke('write_file', {
      path: `${config.location}/.ddd/config.yaml`,
      content: getDefaultDddConfig(),
    });
    await invoke('write_file', {
      path: `${config.location}/.ddd/mapping.yaml`,
      content: 'flows: {}\nschemas: {}\n',
    });

    // 7. Init Git + initial commit
    if (config.initGit) {
      await invoke('git_init', { path: config.location });
      await invoke('git_add_all', { path: config.location });
      await invoke('git_commit', {
        path: config.location,
        message: 'Initialize DDD project',
      });
    }

    // 8. Add to recent projects
    get().addRecentProject({
      name: config.name,
      path: config.location,
      lastOpenedAt: new Date().toISOString(),
      description: config.description,
    });

    return config.location;
  },

  errors: [],
  pushError: (error) => {
    const full: AppError = {
      ...error,
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      dismissed: false,
    };
    set({ errors: [...get().errors, full] });
    // Auto-dismiss info after 5s
    if (error.severity === 'info') {
      setTimeout(() => get().dismissError(full.id), 5000);
    }
  },

  dismissError: (id) => {
    set({ errors: get().errors.map(e => e.id === id ? { ...e, dismissed: true } : e) });
  },

  clearErrors: () => set({ errors: [] }),

  autoSaveTimer: null,
  startAutoSave: (intervalSeconds) => {
    get().stopAutoSave();
    if (intervalSeconds <= 0) return;
    const timer = setInterval(async () => {
      // Save current flow to .ddd/autosave/{flow_id}.yaml
      const flowStore = (await import('./flow-store')).useFlowStore.getState();
      const flow = flowStore.currentFlow;
      if (!flow) return;
      try {
        await invoke('write_file', {
          path: `.ddd/autosave/${flow.id}.yaml`,
          content: flowStore.serializeToYaml(flow),
        });
      } catch {
        // Silent fail — autosave should never interrupt
      }
    }, intervalSeconds * 1000);
    set({ autoSaveTimer: timer });
  },

  stopAutoSave: () => {
    const timer = get().autoSaveTimer;
    if (timer) clearInterval(timer);
    set({ autoSaveTimer: null });
  },
}));

function getDefaultSettings(): GlobalSettings {
  return {
    claudeCode: { enabled: false, command: 'claude' },
    editor: { gridSnap: true, autoSaveInterval: 30, theme: 'system', fontSize: 14 },
    git: { autoCommitMessage: 'Update {flow_id}', branchNaming: 'feature/{flow_id}' },
  };
}
```

**File: `src/stores/undo-store.ts`**
```typescript
import { create } from 'zustand';
import { FlowSnapshot, UndoState } from '../types/app';
import { useFlowStore } from './flow-store';

interface UndoStore {
  // Per-flow undo states
  flowStates: Record<string, UndoState>;

  // Actions
  pushSnapshot: (flowId: string, description: string) => void;
  undo: (flowId: string) => void;
  redo: (flowId: string) => void;
  canUndo: (flowId: string) => boolean;
  canRedo: (flowId: string) => boolean;
  getLastDescription: (flowId: string, direction: 'undo' | 'redo') => string | null;
  clearHistory: (flowId: string) => void;
}

const MAX_HISTORY = 100;
const COALESCE_MS = 500;

export const useUndoStore = create<UndoStore>((set, get) => ({
  flowStates: {},

  pushSnapshot: (flowId, description) => {
    const states = { ...get().flowStates };
    const state = states[flowId] || { undoStack: [], redoStack: [], maxHistory: MAX_HISTORY };

    // Take snapshot of current flow state
    const flowStore = useFlowStore.getState();
    const snapshot: FlowSnapshot = {
      nodes: structuredClone(flowStore.currentFlow?.nodes || []),
      connections: structuredClone(flowStore.currentFlow?.connections || []),
      specValues: structuredClone(flowStore.getSpecValues?.() || {}),
      timestamp: Date.now(),
      description,
    };

    // Coalesce rapid changes to the same description
    const last = state.undoStack[state.undoStack.length - 1];
    if (last && last.description === description && snapshot.timestamp - last.timestamp < COALESCE_MS) {
      // Overwrite the last snapshot instead of pushing a new one
      state.undoStack[state.undoStack.length - 1] = snapshot;
    } else {
      state.undoStack.push(snapshot);
      // Trim to max history
      if (state.undoStack.length > state.maxHistory) {
        state.undoStack.shift();
      }
    }

    // New action clears redo stack
    state.redoStack = [];
    states[flowId] = state;
    set({ flowStates: states });
  },

  undo: (flowId) => {
    const states = { ...get().flowStates };
    const state = states[flowId];
    if (!state || state.undoStack.length === 0) return;

    // Save current state to redo stack
    const flowStore = useFlowStore.getState();
    const currentSnapshot: FlowSnapshot = {
      nodes: structuredClone(flowStore.currentFlow?.nodes || []),
      connections: structuredClone(flowStore.currentFlow?.connections || []),
      specValues: structuredClone(flowStore.getSpecValues?.() || {}),
      timestamp: Date.now(),
      description: 'current',
    };
    state.redoStack.push(currentSnapshot);

    // Pop from undo stack and restore
    const snapshot = state.undoStack.pop()!;
    flowStore.restoreFromSnapshot(snapshot);

    states[flowId] = state;
    set({ flowStates: states });
  },

  redo: (flowId) => {
    const states = { ...get().flowStates };
    const state = states[flowId];
    if (!state || state.redoStack.length === 0) return;

    // Save current to undo stack
    const flowStore = useFlowStore.getState();
    const currentSnapshot: FlowSnapshot = {
      nodes: structuredClone(flowStore.currentFlow?.nodes || []),
      connections: structuredClone(flowStore.currentFlow?.connections || []),
      specValues: structuredClone(flowStore.getSpecValues?.() || {}),
      timestamp: Date.now(),
      description: 'current',
    };
    state.undoStack.push(currentSnapshot);

    // Pop from redo stack and restore
    const snapshot = state.redoStack.pop()!;
    flowStore.restoreFromSnapshot(snapshot);

    states[flowId] = state;
    set({ flowStates: states });
  },

  canUndo: (flowId) => {
    const state = get().flowStates[flowId];
    return !!state && state.undoStack.length > 0;
  },

  canRedo: (flowId) => {
    const state = get().flowStates[flowId];
    return !!state && state.redoStack.length > 0;
  },

  getLastDescription: (flowId, direction) => {
    const state = get().flowStates[flowId];
    if (!state) return null;
    const stack = direction === 'undo' ? state.undoStack : state.redoStack;
    return stack.length > 0 ? stack[stack.length - 1].description : null;
  },

  clearHistory: (flowId) => {
    const states = { ...get().flowStates };
    states[flowId] = { undoStack: [], redoStack: [], maxHistory: MAX_HISTORY };
    set({ flowStates: states });
  },
}));
```

**Component: `src/components/ProjectLauncher/ProjectLauncher.tsx`**
```tsx
import { useAppStore } from '../../stores/app-store';
import { RecentProjects } from './RecentProjects';
import { NewProjectWizard } from './NewProjectWizard';
import { useState } from 'react';

export function ProjectLauncher() {
  const { recentProjects, removeRecentProject } = useAppStore();
  const [showWizard, setShowWizard] = useState(false);

  if (showWizard) {
    return <NewProjectWizard onCancel={() => setShowWizard(false)} />;
  }

  const openExisting = async () => {
    const { open } = await import('@tauri-apps/plugin-dialog');
    const selected = await open({ directory: true, title: 'Open DDD Project' });
    if (selected) {
      await openProject(selected as string);
    }
  };

  return (
    <div className="flex flex-col items-center justify-center h-screen bg-gray-50">
      <h1 className="text-3xl font-bold mb-2">DDD Tool</h1>
      <p className="text-gray-500 mb-8">Design Driven Development</p>

      {recentProjects.length > 0 && (
        <RecentProjects
          projects={recentProjects}
          onOpen={openProject}
          onRemove={removeRecentProject}
        />
      )}

      <div className="flex gap-3 mt-6">
        <button onClick={() => setShowWizard(true)} className="btn btn-primary">
          + New Project
        </button>
        <button onClick={openExisting} className="btn btn-secondary">
          Open Existing
        </button>
      </div>
    </div>
  );
}

async function openProject(path: string) {
  const { invoke } = await import('@tauri-apps/api/core');
  // Check if this is a DDD project
  const hasSystem = await invoke<boolean>('path_exists', {
    path: `${path}/specs/system.yaml`,
  });
  const hasDdd = await invoke<boolean>('path_exists', {
    path: `${path}/.ddd/config.yaml`,
  });

  if (!hasSystem && !hasDdd) {
    useAppStore.getState().pushError({
      severity: 'error',
      component: 'file',
      message: "This folder doesn't appear to be a DDD project.",
      recoveryAction: {
        label: 'Initialize as DDD project',
        action: () => { /* open wizard with path pre-filled */ },
      },
    });
    return;
  }

  // Load project
  useAppStore.getState().addRecentProject({
    name: path.split('/').pop() || 'project',
    path,
    lastOpenedAt: new Date().toISOString(),
  });

  const projectStore = (await import('../../stores/project-store')).useProjectStore.getState();
  await projectStore.loadProject(path);
  useAppStore.getState().setView('project');
}
```

**Component: `src/components/ProjectLauncher/NewProjectWizard.tsx`**
```tsx
import { useState } from 'react';
import { useAppStore } from '../../stores/app-store';
import { NewProjectConfig } from '../../types/app';

export function NewProjectWizard({ onCancel }: { onCancel: () => void }) {
  const [step, setStep] = useState(1);
  const [config, setConfig] = useState<NewProjectConfig>({
    name: '', location: '', description: '', initGit: true,
    techStack: { language: 'python', languageVersion: '3.11', framework: 'fastapi', database: 'postgresql', orm: 'sqlalchemy' },
    domains: [{ name: '', description: '' }],
  });
  const { createProject, setView } = useAppStore();

  const handleCreate = async () => {
    // Filter out empty domains
    const finalConfig = {
      ...config,
      domains: config.domains.filter(d => d.name.trim()),
    };
    const path = await createProject(finalConfig);

    // Load and navigate
    const projectStore = (await import('../../stores/project-store')).useProjectStore.getState();
    await projectStore.loadProject(path);
    setView('project');
  };

  return (
    <div className="max-w-lg mx-auto mt-16 p-6">
      <h2 className="text-xl font-bold mb-4">New Project — Step {step} of 3</h2>

      {step === 1 && (
        <div className="space-y-4">
          <label>Project Name
            <input value={config.name} onChange={e => setConfig({ ...config, name: e.target.value })}
              className="input w-full" placeholder="my-project" />
          </label>
          <label>Location
            <div className="flex gap-2">
              <input value={config.location} onChange={e => setConfig({ ...config, location: e.target.value })}
                className="input flex-1" placeholder="~/code/my-project" />
              <button onClick={async () => {
                const { open } = await import('@tauri-apps/plugin-dialog');
                const dir = await open({ directory: true });
                if (dir) setConfig({ ...config, location: `${dir}/${config.name}` });
              }} className="btn btn-sm">Browse</button>
            </div>
          </label>
          <label>Description
            <input value={config.description} onChange={e => setConfig({ ...config, description: e.target.value })}
              className="input w-full" placeholder="My awesome project" />
          </label>
          <label className="flex items-center gap-2">
            <input type="checkbox" checked={config.initGit}
              onChange={e => setConfig({ ...config, initGit: e.target.checked })} />
            Initialize Git repository
          </label>
        </div>
      )}

      {step === 2 && (
        <div className="space-y-4">
          <label>Language
            <select value={config.techStack.language}
              onChange={e => setConfig({ ...config, techStack: { ...config.techStack, language: e.target.value } })}
              className="select w-full">
              <option value="python">Python</option>
              <option value="typescript">TypeScript</option>
              <option value="go">Go</option>
            </select>
          </label>
          <label>Framework
            <select value={config.techStack.framework}
              onChange={e => setConfig({ ...config, techStack: { ...config.techStack, framework: e.target.value } })}
              className="select w-full">
              <option value="fastapi">FastAPI</option>
              <option value="django">Django</option>
              <option value="nestjs">NestJS</option>
              <option value="express">Express</option>
              <option value="gin">Gin</option>
            </select>
          </label>
          <label>Database
            <select value={config.techStack.database}
              onChange={e => setConfig({ ...config, techStack: { ...config.techStack, database: e.target.value } })}
              className="select w-full">
              <option value="postgresql">PostgreSQL</option>
              <option value="mysql">MySQL</option>
              <option value="sqlite">SQLite</option>
              <option value="mongodb">MongoDB</option>
            </select>
          </label>
        </div>
      )}

      {step === 3 && (
        <div className="space-y-4">
          <p className="text-sm text-gray-600">Define your initial domains:</p>
          {config.domains.map((d, i) => (
            <div key={i} className="flex gap-2">
              <input value={d.name} placeholder="users"
                onChange={e => {
                  const domains = [...config.domains];
                  domains[i] = { ...d, name: e.target.value };
                  setConfig({ ...config, domains });
                }} className="input flex-1" />
              <input value={d.description} placeholder="User management"
                onChange={e => {
                  const domains = [...config.domains];
                  domains[i] = { ...d, description: e.target.value };
                  setConfig({ ...config, domains });
                }} className="input flex-1" />
              <button onClick={() => {
                setConfig({ ...config, domains: config.domains.filter((_, j) => j !== i) });
              }} className="btn btn-sm text-red-500">x</button>
            </div>
          ))}
          <button onClick={() => {
            setConfig({ ...config, domains: [...config.domains, { name: '', description: '' }] });
          }} className="btn btn-sm">+ Add domain</button>
        </div>
      )}

      <div className="flex justify-between mt-6">
        <button onClick={step === 1 ? onCancel : () => setStep(step - 1)} className="btn btn-secondary">
          {step === 1 ? 'Cancel' : '← Back'}
        </button>
        <button onClick={step === 3 ? handleCreate : () => setStep(step + 1)} className="btn btn-primary">
          {step === 3 ? 'Create' : 'Next →'}
        </button>
      </div>
    </div>
  );
}
```

**Component: `src/components/Settings/SettingsDialog.tsx`**
```tsx
import { useState } from 'react';
import { useAppStore } from '../../stores/app-store';
import { ClaudeCodeSettings } from './ClaudeCodeSettings';
import { EditorSettings } from './EditorSettings';
import { GitSettings } from './GitSettings';

const TABS = ['Editor', 'Claude Code', 'Git'] as const;
type Tab = typeof TABS[number];

export function SettingsDialog({ onClose }: { onClose: () => void }) {
  const [activeTab, setActiveTab] = useState<Tab>('Editor');
  const [scope, setScope] = useState<'global' | 'project'>('global');
  const { globalSettings, saveGlobalSettings } = useAppStore();

  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div className="bg-white rounded-lg shadow-xl w-[700px] h-[500px] flex">
        {/* Sidebar */}
        <div className="w-40 border-r p-3 space-y-1">
          <select value={scope} onChange={e => setScope(e.target.value as any)}
            className="select w-full text-sm mb-3">
            <option value="global">Global</option>
            <option value="project">Project</option>
          </select>
          {TABS.map(tab => (
            <button key={tab}
              onClick={() => setActiveTab(tab)}
              className={`block w-full text-left px-3 py-1.5 rounded text-sm ${
                activeTab === tab ? 'bg-blue-50 text-blue-700 font-medium' : 'text-gray-700'
              }`}>
              {tab}
            </button>
          ))}
        </div>

        {/* Content */}
        <div className="flex-1 p-4 overflow-auto">
          <div className="flex justify-between items-center mb-4">
            <h2 className="text-lg font-bold">{activeTab}</h2>
            <button onClick={onClose} className="text-gray-400 hover:text-gray-600">✕</button>
          </div>

          {activeTab === 'Claude Code' && <ClaudeCodeSettings settings={globalSettings} onSave={saveGlobalSettings} />}
          {activeTab === 'Editor' && <EditorSettings settings={globalSettings} onSave={saveGlobalSettings} />}
          {activeTab === 'Git' && <GitSettings settings={globalSettings} onSave={saveGlobalSettings} />}
        </div>
      </div>
    </div>
  );
}
```

**Update `src/App.tsx` — route between views:**
```tsx
import { useEffect } from 'react';
import { useAppStore } from './stores/app-store';
import { ProjectLauncher } from './components/ProjectLauncher/ProjectLauncher';
import { FirstRunWizard } from './components/FirstRun/FirstRunWizard';
import { AppShell } from './AppShell'; // existing canvas + panels layout

export default function App() {
  const { view, checkFirstRun, loadRecentProjects, loadGlobalSettings, startAutoSave } = useAppStore();

  useEffect(() => {
    // Boot sequence
    checkFirstRun().then(() => {
      loadGlobalSettings();
      loadRecentProjects();
    });
  }, []);

  useEffect(() => {
    // Start auto-save when in project view
    const settings = useAppStore.getState().globalSettings;
    if (view === 'project' && settings.editor?.autoSaveInterval) {
      startAutoSave(settings.editor.autoSaveInterval);
    }
    return () => useAppStore.getState().stopAutoSave();
  }, [view]);

  switch (view) {
    case 'first-run':
      return <FirstRunWizard />;
    case 'launcher':
      return <ProjectLauncher />;
    case 'project':
      return <AppShell />;
  }
}
```

**Error notification component (add to AppShell):**
```tsx
import { useAppStore } from '../../stores/app-store';

export function ErrorToasts() {
  const { errors, dismissError } = useAppStore();
  const visible = errors.filter(e => !e.dismissed);

  return (
    <div className="fixed bottom-4 right-4 space-y-2 z-50">
      {visible.map(error => (
        <div key={error.id} className={`rounded-lg shadow-lg p-3 max-w-sm flex gap-3 ${
          error.severity === 'info' ? 'bg-blue-50 border-blue-200' :
          error.severity === 'warning' ? 'bg-yellow-50 border-yellow-200' :
          error.severity === 'error' ? 'bg-red-50 border-red-200' :
          'bg-red-100 border-red-300'
        } border`}>
          <div className="flex-1">
            <p className="text-sm font-medium">{error.message}</p>
            {error.detail && <p className="text-xs text-gray-500 mt-1">{error.detail}</p>}
            {error.recoveryAction && (
              <button onClick={error.recoveryAction.action}
                className="text-xs text-blue-600 underline mt-1">
                {error.recoveryAction.label}
              </button>
            )}
          </div>
          <button onClick={() => dismissError(error.id)}
            className="text-gray-400 hover:text-gray-600 text-sm">✕</button>
        </div>
      ))}
    </div>
  );
}
```

**Undo/redo toolbar buttons (add to Canvas toolbar):**
```tsx
import { useUndoStore } from '../../stores/undo-store';

export function UndoRedoButtons({ flowId }: { flowId: string }) {
  const { canUndo, canRedo, undo, redo, getLastDescription } = useUndoStore();

  return (
    <div className="flex items-center gap-1">
      <button
        onClick={() => undo(flowId)}
        disabled={!canUndo(flowId)}
        title={canUndo(flowId) ? `Undo: ${getLastDescription(flowId, 'undo')}` : 'Nothing to undo'}
        className="btn btn-sm disabled:opacity-30"
      >
        ↩ Undo
      </button>
      <button
        onClick={() => redo(flowId)}
        disabled={!canRedo(flowId)}
        title={canRedo(flowId) ? `Redo: ${getLastDescription(flowId, 'redo')}` : 'Nothing to redo'}
        className="btn btn-sm disabled:opacity-30"
      >
        Redo ↪
      </button>
    </div>
  );
}
```

**Keyboard shortcut registration (add to Canvas or App level):**
```typescript
// Register global undo/redo shortcuts
useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    const mod = e.metaKey || e.ctrlKey;
    if (!mod) return;

    const flowId = useFlowStore.getState().currentFlow?.id;
    if (!flowId) return;

    if (e.key === 'z' && !e.shiftKey) {
      e.preventDefault();
      useUndoStore.getState().undo(flowId);
    }
    if ((e.key === 'z' && e.shiftKey) || e.key === 'y') {
      e.preventDefault();
      useUndoStore.getState().redo(flowId);
    }
    if (e.key === ',') {
      e.preventDefault();
      setShowSettings(true);
    }
  };
  window.addEventListener('keydown', handler);
  return () => window.removeEventListener('keydown', handler);
}, []);
```

**Integration: push undo snapshots before mutations in flow-store:**
```typescript
// In flow-store.ts, wrap every mutation that should be undoable:
addNode: (node) => {
  useUndoStore.getState().pushSnapshot(get().currentFlow!.id, `Add ${node.type} node`);
  // ... existing add logic
},

deleteNode: (nodeId) => {
  useUndoStore.getState().pushSnapshot(get().currentFlow!.id, `Delete node`);
  // ... existing delete logic
},

moveNode: (nodeId, position) => {
  useUndoStore.getState().pushSnapshot(get().currentFlow!.id, `Move node`);
  // ... existing move logic
},

addConnection: (from, to) => {
  useUndoStore.getState().pushSnapshot(get().currentFlow!.id, `Connect nodes`);
  // ... existing connect logic
},

updateSpec: (nodeId, field, value) => {
  useUndoStore.getState().pushSnapshot(get().currentFlow!.id, `Edit ${field}`);
  // ... existing update logic
},
```

---

### Design Validation

**File: `src/types/validation.ts`**
```typescript
export type ValidationSeverity = 'error' | 'warning' | 'info';
export type ValidationScope = 'flow' | 'domain' | 'system';
export type ValidationCategory =
  | 'graph_completeness'
  | 'spec_completeness'
  | 'reference_integrity'
  | 'agent_validation'
  | 'orchestration_validation'
  | 'domain_consistency'
  | 'event_wiring'
  | 'cross_domain_data'
  | 'portal_wiring'
  | 'orchestration_wiring';

export interface ValidationIssue {
  id: string;
  scope: ValidationScope;
  severity: ValidationSeverity;
  category: ValidationCategory;
  message: string;
  detail?: string;                     // Longer explanation
  suggestion?: string;                 // How to fix
  nodeId?: string;                     // Which node (for flow-level)
  flowId?: string;                     // Which flow
  domainId?: string;                   // Which domain
  relatedFlowId?: string;             // For cross-domain issues, the other flow
  relatedDomainId?: string;           // For cross-domain issues, the other domain
}

export interface ValidationResult {
  scope: ValidationScope;
  targetId: string;                    // flow ID, domain name, or 'system'
  issues: ValidationIssue[];
  errorCount: number;
  warningCount: number;
  infoCount: number;
  isValid: boolean;                    // true if errorCount === 0
  validatedAt: string;
}

export interface ImplementGateState {
  flowValidation: ValidationResult | null;
  domainValidation: ValidationResult | null;
  systemValidation: ValidationResult | null;
  canImplement: boolean;               // true if no errors at any level
  hasWarnings: boolean;
}
```

**File: `src/utils/flow-validator.ts`**
```typescript
import { DddFlow, DddNode } from '../types/flow';
import { ValidationIssue, ValidationResult } from '../types/validation';

// ============================================================
// FLOW-LEVEL VALIDATION
// ============================================================

export function validateFlow(
  flow: DddFlow,
  errorCodes: string[],
  schemaNames: string[],
  allFlowIds: string[]
): ValidationResult {
  const issues: ValidationIssue[] = [
    ...validateGraphCompleteness(flow),
    ...validateSpecCompleteness(flow),
    ...validateReferenceIntegrity(flow, errorCodes, schemaNames, allFlowIds),
    ...(flow.flow_type === 'agent' ? validateAgentFlow(flow) : []),
    ...(flow.flow_type === 'orchestration' ? validateOrchestrationFlow(flow, allFlowIds) : []),
  ];

  return buildResult('flow', flow.id, issues);
}

function validateGraphCompleteness(flow: DddFlow): ValidationIssue[] {
  const issues: ValidationIssue[] = [];
  const nodeMap = new Map(flow.nodes.map(n => [n.id, n]));

  // Trigger exists
  const triggers = flow.nodes.filter(n => n.type === 'trigger');
  if (triggers.length === 0) {
    issues.push(issue('error', 'graph_completeness', 'Flow has no trigger node', flow.id));
  } else if (triggers.length > 1) {
    issues.push(issue('error', 'graph_completeness', 'Flow has multiple trigger nodes — only one allowed', flow.id));
  }

  // Find all reachable nodes from trigger via BFS
  const reachable = new Set<string>();
  if (triggers[0]) {
    const queue = [triggers[0].id];
    while (queue.length > 0) {
      const current = queue.shift()!;
      if (reachable.has(current)) continue;
      reachable.add(current);
      const node = nodeMap.get(current);
      if (!node) continue;
      const connections = node.connections || {};
      for (const targetId of Object.values(connections)) {
        if (typeof targetId === 'string' && !reachable.has(targetId)) {
          queue.push(targetId);
        }
      }
    }
  }

  // Orphaned nodes
  for (const node of flow.nodes) {
    if (!reachable.has(node.id) && node.type !== 'trigger') {
      issues.push(issue('error', 'graph_completeness',
        `Node "${node.id}" is unreachable from trigger`, flow.id, node.id));
    }
  }

  // All paths reach terminal (check for dead-ends)
  for (const node of flow.nodes) {
    if (!reachable.has(node.id)) continue;
    if (node.type === 'terminal') continue;

    const connections = node.connections || {};
    const targets = Object.values(connections).filter(v => typeof v === 'string');

    if (targets.length === 0 && node.type !== 'trigger') {
      issues.push(issue('error', 'graph_completeness',
        `Node "${node.id}" (${node.type}) has no outgoing connections — dead end`,
        flow.id, node.id,
        'Connect this node to a downstream node or terminal'));
    }
  }

  // Decision branches complete
  for (const node of flow.nodes) {
    if (node.type === 'decision') {
      const conn = node.connections || {};
      if (!conn['true'] && !conn['false']) {
        issues.push(issue('error', 'graph_completeness',
          `Decision "${node.id}" has no branches connected`, flow.id, node.id));
      } else if (!conn['true']) {
        issues.push(issue('error', 'graph_completeness',
          `Decision "${node.id}" missing true branch`, flow.id, node.id));
      } else if (!conn['false']) {
        issues.push(issue('error', 'graph_completeness',
          `Decision "${node.id}" missing false branch`, flow.id, node.id));
      }
    }
  }

  // Input node branches
  for (const node of flow.nodes) {
    if (node.type === 'input') {
      const conn = node.connections || {};
      if (!conn['valid']) {
        issues.push(issue('error', 'graph_completeness',
          `Input "${node.id}" missing valid branch`, flow.id, node.id));
      }
      if (!conn['invalid']) {
        issues.push(issue('warning', 'graph_completeness',
          `Input "${node.id}" missing invalid branch — validation errors won't be handled`,
          flow.id, node.id));
      }
    }
  }

  // Cycle detection for traditional flows
  if (flow.flow_type !== 'agent') {
    if (hasCycle(flow)) {
      issues.push(issue('error', 'graph_completeness',
        'Flow contains a cycle — traditional flows must be acyclic (DAG)', flow.id));
    }
  }

  return issues;
}

function validateSpecCompleteness(flow: DddFlow): ValidationIssue[] {
  const issues: ValidationIssue[] = [];

  for (const node of flow.nodes) {
    const spec = node.spec || {};

    if (node.type === 'trigger') {
      if (!spec.type) {
        issues.push(issue('error', 'spec_completeness',
          `Trigger "${node.id}" has no type defined`, flow.id, node.id,
          'Set trigger type (http, event, scheduled, manual)'));
      }
      if (spec.type === 'http') {
        if (!spec.method) issues.push(issue('error', 'spec_completeness',
          `HTTP trigger "${node.id}" missing method`, flow.id, node.id));
        if (!spec.path) issues.push(issue('error', 'spec_completeness',
          `HTTP trigger "${node.id}" missing path`, flow.id, node.id));
      }
    }

    if (node.type === 'input') {
      const fields = spec.fields || {};
      for (const [name, rules] of Object.entries(fields)) {
        const r = rules as any;
        if (!r.type) {
          issues.push(issue('error', 'spec_completeness',
            `Input "${node.id}" field "${name}" has no type`, flow.id, node.id));
        }
        if (r.required && !r.error) {
          issues.push(issue('error', 'spec_completeness',
            `Input "${node.id}" required field "${name}" has no error message`,
            flow.id, node.id, 'Add error message for validation feedback'));
        }
        if ((r.min_length !== undefined || r.max_length !== undefined || r.format) && !r.error) {
          issues.push(issue('warning', 'spec_completeness',
            `Input "${node.id}" field "${name}" has validation rules but no error message`,
            flow.id, node.id));
        }
      }
    }

    if (node.type === 'decision') {
      if (!spec.check && !spec.condition) {
        issues.push(issue('error', 'spec_completeness',
          `Decision "${node.id}" has no check or condition defined`, flow.id, node.id));
      }
      if (spec.on_true && !spec.on_true.error_code) {
        issues.push(issue('warning', 'spec_completeness',
          `Decision "${node.id}" error branch has no error_code`, flow.id, node.id));
      }
    }

    if (node.type === 'terminal') {
      if (!spec.status && flow.nodes.find(n => n.type === 'trigger')?.spec?.type === 'http') {
        issues.push(issue('warning', 'spec_completeness',
          `Terminal "${node.id}" has no HTTP status code`, flow.id, node.id));
      }
      if (!spec.body) {
        issues.push(issue('info', 'spec_completeness',
          `Terminal "${node.id}" has no response body defined`, flow.id, node.id));
      }
    }

    if (node.type === 'data_store') {
      if (!spec.operation) {
        issues.push(issue('error', 'spec_completeness',
          `Data store "${node.id}" has no operation (create/read/update/delete)`, flow.id, node.id));
      }
      if (!spec.model) {
        issues.push(issue('error', 'spec_completeness',
          `Data store "${node.id}" has no model specified`, flow.id, node.id));
      }
    }

    if (node.type === 'process' && !spec.description) {
      issues.push(issue('warning', 'spec_completeness',
        `Process "${node.id}" has no description`, flow.id, node.id));
    }
  }

  return issues;
}

function validateReferenceIntegrity(
  flow: DddFlow,
  errorCodes: string[],
  schemaNames: string[],
  allFlowIds: string[]
): ValidationIssue[] {
  const issues: ValidationIssue[] = [];

  for (const node of flow.nodes) {
    const spec = node.spec || {};

    // Error code references
    const codes = extractErrorCodes(node);
    for (const code of codes) {
      if (!errorCodes.includes(code)) {
        issues.push(issue('error', 'reference_integrity',
          `Error code "${code}" not found in errors.yaml`, flow.id, node.id,
          `Add "${code}" to specs/shared/errors.yaml or use an existing code`));
      }
    }

    // Schema model references
    if (node.type === 'data_store' && spec.model) {
      if (!schemaNames.includes(spec.model.toLowerCase())) {
        issues.push(issue('error', 'reference_integrity',
          `Model "${spec.model}" not found in specs/schemas/`, flow.id, node.id,
          `Create specs/schemas/${spec.model.toLowerCase()}.yaml`));
      }
    }

    // Sub-flow references
    if (node.type === 'sub_flow' && spec.flow_ref) {
      if (!allFlowIds.includes(spec.flow_ref)) {
        issues.push(issue('error', 'reference_integrity',
          `Sub-flow "${spec.flow_ref}" does not exist`, flow.id, node.id));
      }
    }
  }

  return issues;
}

function validateAgentFlow(flow: DddFlow): ValidationIssue[] {
  const issues: ValidationIssue[] = [];

  const agentLoops = flow.nodes.filter(n => n.type === 'agent_loop');
  if (agentLoops.length === 0) {
    issues.push(issue('error', 'agent_validation', 'Agent flow has no agent_loop node', flow.id));
    return issues;
  }
  if (agentLoops.length > 1) {
    issues.push(issue('error', 'agent_validation', 'Agent flow has multiple agent_loop nodes', flow.id));
  }

  const agent = agentLoops[0];
  const tools = flow.nodes.filter(n => n.type === 'tool');
  if (tools.length === 0) {
    issues.push(issue('error', 'agent_validation', 'Agent has no tools connected', flow.id, agent.id));
  }

  const terminalTools = tools.filter(t => t.spec?.terminal);
  if (terminalTools.length === 0) {
    issues.push(issue('error', 'agent_validation',
      'Agent has no terminal tool — loop has no way to end', flow.id, agent.id,
      'Mark at least one tool as terminal: true'));
  }

  if (!agent.spec?.max_iterations) {
    issues.push(issue('warning', 'agent_validation',
      'Agent loop has no max_iterations — could run indefinitely', flow.id, agent.id));
  }

  if (!agent.spec?.model) {
    issues.push(issue('warning', 'agent_validation',
      'Agent loop has no LLM model specified', flow.id, agent.id));
  }

  const guardrails = flow.nodes.filter(n => n.type === 'guardrail');
  for (const g of guardrails) {
    if (!g.spec?.checks || g.spec.checks.length === 0) {
      issues.push(issue('warning', 'agent_validation',
        `Guardrail "${g.id}" has no checks defined`, flow.id, g.id));
    }
  }

  const humanGates = flow.nodes.filter(n => n.type === 'human_gate');
  for (const h of humanGates) {
    if (!h.spec?.description) {
      issues.push(issue('warning', 'agent_validation',
        `Human gate "${h.id}" has no description — user won't know what they're approving`,
        flow.id, h.id));
    }
  }

  return issues;
}

function validateOrchestrationFlow(flow: DddFlow, allFlowIds: string[]): ValidationIssue[] {
  const issues: ValidationIssue[] = [];

  // Orchestrator checks
  for (const node of flow.nodes.filter(n => n.type === 'orchestrator')) {
    const agents = node.spec?.agents || [];
    if (agents.length < 2) {
      issues.push(issue('error', 'orchestration_validation',
        `Orchestrator "${node.id}" needs at least 2 agents`, flow.id, node.id));
    }
    if (!node.spec?.strategy) {
      issues.push(issue('error', 'orchestration_validation',
        `Orchestrator "${node.id}" has no strategy defined`, flow.id, node.id));
    }
    // Verify referenced agents exist
    for (const agentId of agents) {
      if (!allFlowIds.includes(agentId)) {
        issues.push(issue('error', 'orchestration_validation',
          `Orchestrator "${node.id}" references agent "${agentId}" which doesn't exist`,
          flow.id, node.id));
      }
    }
  }

  // Router checks
  for (const node of flow.nodes.filter(n => n.type === 'smart_router')) {
    const rules = node.spec?.rules || [];
    if (rules.length === 0) {
      issues.push(issue('error', 'orchestration_validation',
        `Router "${node.id}" has no routing rules`, flow.id, node.id));
    }
    if (!node.spec?.fallback) {
      issues.push(issue('warning', 'orchestration_validation',
        `Router "${node.id}" has no fallback agent`, flow.id, node.id));
    }
    // Verify rule targets exist
    for (const rule of rules) {
      if (rule.target && !allFlowIds.includes(rule.target)) {
        issues.push(issue('error', 'orchestration_validation',
          `Router "${node.id}" rule targets "${rule.target}" which doesn't exist`,
          flow.id, node.id));
      }
    }
    if (node.spec?.circuit_breaker) {
      if (!node.spec.circuit_breaker.failure_threshold) {
        issues.push(issue('warning', 'orchestration_validation',
          `Router "${node.id}" circuit breaker missing failure_threshold`, flow.id, node.id));
      }
    }
  }

  // Handoff checks
  for (const node of flow.nodes.filter(n => n.type === 'handoff')) {
    if (!node.spec?.target) {
      issues.push(issue('error', 'orchestration_validation',
        `Handoff "${node.id}" has no target agent`, flow.id, node.id));
    } else if (!allFlowIds.includes(node.spec.target)) {
      issues.push(issue('error', 'orchestration_validation',
        `Handoff "${node.id}" targets "${node.spec.target}" which doesn't exist`,
        flow.id, node.id));
    }
    if (!node.spec?.mode) {
      issues.push(issue('warning', 'orchestration_validation',
        `Handoff "${node.id}" has no mode (transfer/consult/collaborate)`, flow.id, node.id));
    }
  }

  // Agent group checks
  for (const node of flow.nodes.filter(n => n.type === 'agent_group')) {
    const members = node.spec?.members || [];
    if (members.length < 2) {
      issues.push(issue('error', 'orchestration_validation',
        `Agent group "${node.id}" needs at least 2 members`, flow.id, node.id));
    }
    for (const memberId of members) {
      if (!allFlowIds.includes(memberId)) {
        issues.push(issue('error', 'orchestration_validation',
          `Agent group "${node.id}" member "${memberId}" doesn't exist`, flow.id, node.id));
      }
    }
  }

  return issues;
}

// ============================================================
// DOMAIN-LEVEL VALIDATION
// ============================================================

export function validateDomain(
  domainName: string,
  flows: DddFlow[],
  domainYaml: any
): ValidationResult {
  const issues: ValidationIssue[] = [];

  // Duplicate flow IDs
  const flowIds = flows.map(f => f.id);
  const seen = new Set<string>();
  for (const id of flowIds) {
    if (seen.has(id)) {
      issues.push({
        id: crypto.randomUUID(), scope: 'domain', severity: 'error',
        category: 'domain_consistency', domainId: domainName,
        message: `Duplicate flow ID "${id}" in domain "${domainName}"`,
      });
    }
    seen.add(id);
  }

  // Duplicate HTTP paths
  const httpPaths = new Map<string, string>();
  for (const flow of flows) {
    const trigger = flow.nodes.find(n => n.type === 'trigger');
    if (trigger?.spec?.type === 'http' && trigger.spec.method && trigger.spec.path) {
      const key = `${trigger.spec.method} ${trigger.spec.path}`;
      if (httpPaths.has(key)) {
        issues.push({
          id: crypto.randomUUID(), scope: 'domain', severity: 'error',
          category: 'domain_consistency', domainId: domainName, flowId: flow.id,
          message: `Duplicate HTTP endpoint "${key}" — also used by flow "${httpPaths.get(key)}"`,
        });
      }
      httpPaths.set(key, flow.id);
    }
  }

  // Domain.yaml flow list matches actual files
  if (domainYaml?.flows) {
    const declared = new Set(domainYaml.flows.map((f: any) => f.id || f.name || f));
    for (const flow of flows) {
      if (!declared.has(flow.id)) {
        issues.push({
          id: crypto.randomUUID(), scope: 'domain', severity: 'info',
          category: 'domain_consistency', domainId: domainName, flowId: flow.id,
          message: `Flow "${flow.id}" exists on disk but not listed in domain.yaml`,
        });
      }
    }
  }

  return buildResult('domain', domainName, issues);
}

// ============================================================
// SYSTEM-LEVEL VALIDATION (CROSS-DOMAIN WIRING)
// ============================================================

export function validateSystem(
  domains: Array<{ name: string; config: any; flows: DddFlow[] }>,
  schemas: string[],
  allFlowIds: string[]
): ValidationResult {
  const issues: ValidationIssue[] = [
    ...validateEventWiring(domains),
    ...validatePortalWiring(domains),
    ...validateOrchestrationWiring(domains, allFlowIds),
    ...validateCrossDomainData(domains, schemas),
  ];

  return buildResult('system', 'system', issues);
}

function validateEventWiring(
  domains: Array<{ name: string; config: any; flows: DddFlow[] }>
): ValidationIssue[] {
  const issues: ValidationIssue[] = [];

  // Collect all published and consumed events
  const published: Array<{ event: string; domain: string; payload?: any }> = [];
  const consumed: Array<{ event: string; domain: string; expectedPayload?: any }> = [];

  for (const domain of domains) {
    const config = domain.config || {};
    for (const event of config.publishes_events || []) {
      published.push({
        event: typeof event === 'string' ? event : event.name,
        domain: domain.name,
        payload: typeof event === 'object' ? event.payload : undefined,
      });
    }
    for (const event of config.consumes_events || []) {
      consumed.push({
        event: typeof event === 'string' ? event : event.name,
        domain: domain.name,
        expectedPayload: typeof event === 'object' ? event.payload : undefined,
      });
    }
  }

  // Consumed events must have a publisher
  for (const c of consumed) {
    const publisher = published.find(p => p.event === c.event);
    if (!publisher) {
      issues.push({
        id: crypto.randomUUID(), scope: 'system', severity: 'error',
        category: 'event_wiring', domainId: c.domain,
        message: `Domain "${c.domain}" consumes event "${c.event}" but no domain publishes it`,
        suggestion: `Add "${c.event}" to publishes_events in the producing domain`,
      });
    }
  }

  // Published events should have at least one consumer
  for (const p of published) {
    const consumer = consumed.find(c => c.event === p.event);
    if (!consumer) {
      issues.push({
        id: crypto.randomUUID(), scope: 'system', severity: 'warning',
        category: 'event_wiring', domainId: p.domain,
        message: `Domain "${p.domain}" publishes event "${p.event}" but no domain consumes it`,
        suggestion: 'Either add a consumer or remove the published event',
      });
    }
  }

  // Event payload shape matching
  for (const c of consumed) {
    const publisher = published.find(p => p.event === c.event);
    if (publisher && publisher.payload && c.expectedPayload) {
      const pubFields = Object.keys(publisher.payload);
      const conFields = Object.keys(c.expectedPayload);
      const missingInPub = conFields.filter(f => !pubFields.includes(f));
      const extraInPub = pubFields.filter(f => !conFields.includes(f));

      if (missingInPub.length > 0) {
        issues.push({
          id: crypto.randomUUID(), scope: 'system', severity: 'error',
          category: 'event_wiring', domainId: c.domain,
          relatedDomainId: publisher.domain,
          message: `Event "${c.event}" payload mismatch: consumer "${c.domain}" expects fields [${missingInPub.join(', ')}] but publisher "${publisher.domain}" doesn't provide them`,
          suggestion: 'Update publisher payload or consumer expectations to match',
        });
      }
      if (extraInPub.length > 0) {
        issues.push({
          id: crypto.randomUUID(), scope: 'system', severity: 'info',
          category: 'event_wiring', domainId: publisher.domain,
          relatedDomainId: c.domain,
          message: `Event "${c.event}": publisher "${publisher.domain}" sends extra fields [${extraInPub.join(', ')}] not consumed by "${c.domain}"`,
        });
      }
    }
  }

  // Event naming consistency check
  const allEventNames = [...published, ...consumed].map(e => e.event);
  const patterns = {
    dotCase: /^[a-z]+\.[a-z_]+$/,          // contract.ingested
    camelCase: /^[a-z]+[A-Z][a-zA-Z]+$/,   // contractIngested
    snakeCase: /^[a-z]+_[a-z_]+$/,          // contract_ingested
  };
  const detected = new Set<string>();
  for (const name of allEventNames) {
    if (patterns.dotCase.test(name)) detected.add('dot.case');
    else if (patterns.camelCase.test(name)) detected.add('camelCase');
    else if (patterns.snakeCase.test(name)) detected.add('snake_case');
  }
  if (detected.size > 1) {
    issues.push({
      id: crypto.randomUUID(), scope: 'system', severity: 'warning',
      category: 'event_wiring',
      message: `Inconsistent event naming: mixing ${[...detected].join(', ')} conventions`,
      suggestion: 'Standardize on one naming convention (recommended: entity.action)',
    });
  }

  return issues;
}

function validatePortalWiring(
  domains: Array<{ name: string; config: any; flows: DddFlow[] }>
): ValidationIssue[] {
  const issues: ValidationIssue[] = [];
  const domainNames = new Set(domains.map(d => d.name));

  for (const domain of domains) {
    const portals = domain.config?.portals || [];
    for (const portal of portals) {
      const targetDomain = typeof portal === 'string' ? portal : portal.target;
      if (!domainNames.has(targetDomain)) {
        issues.push({
          id: crypto.randomUUID(), scope: 'system', severity: 'error',
          category: 'portal_wiring', domainId: domain.name,
          message: `Portal in "${domain.name}" points to domain "${targetDomain}" which doesn't exist`,
        });
      }
    }
  }

  return issues;
}

function validateOrchestrationWiring(
  domains: Array<{ name: string; config: any; flows: DddFlow[] }>,
  allFlowIds: string[]
): ValidationIssue[] {
  const issues: ValidationIssue[] = [];

  // Build orchestration dependency graph for cycle detection
  const orchGraph = new Map<string, string[]>();

  for (const domain of domains) {
    for (const flow of domain.flows) {
      if (flow.flow_type !== 'orchestration') continue;

      const agentRefs: string[] = [];
      for (const node of flow.nodes) {
        if (node.type === 'orchestrator') {
          agentRefs.push(...(node.spec?.agents || []));
        }
        if (node.type === 'smart_router') {
          for (const rule of node.spec?.rules || []) {
            if (rule.target) agentRefs.push(rule.target);
          }
          if (node.spec?.fallback) agentRefs.push(node.spec.fallback);
        }
        if (node.type === 'handoff' && node.spec?.target) {
          agentRefs.push(node.spec.target);
        }
      }
      orchGraph.set(flow.id, agentRefs);
    }
  }

  // Detect cycles in orchestration graph
  function detectCycle(start: string, visited: Set<string>, path: string[]): string[] | null {
    if (path.includes(start)) return [...path, start];
    if (visited.has(start)) return null;
    visited.add(start);
    const deps = orchGraph.get(start) || [];
    for (const dep of deps) {
      if (orchGraph.has(dep)) {  // only follow orchestration flows
        const cycle = detectCycle(dep, visited, [...path, start]);
        if (cycle) return cycle;
      }
    }
    return null;
  }

  const visited = new Set<string>();
  for (const flowId of orchGraph.keys()) {
    const cycle = detectCycle(flowId, visited, []);
    if (cycle) {
      issues.push({
        id: crypto.randomUUID(), scope: 'system', severity: 'error',
        category: 'orchestration_wiring',
        message: `Circular orchestration dependency: ${cycle.join(' → ')}`,
        suggestion: 'Break the cycle by removing one orchestration reference',
      });
      break; // Report first cycle only
    }
  }

  return issues;
}

function validateCrossDomainData(
  domains: Array<{ name: string; config: any; flows: DddFlow[] }>,
  schemas: string[]
): ValidationIssue[] {
  const issues: ValidationIssue[] = [];

  // Check cross-domain API dependencies (service_call nodes targeting other domains)
  const httpEndpoints = new Map<string, string>(); // "METHOD /path" → flow.id
  for (const domain of domains) {
    for (const flow of domain.flows) {
      const trigger = flow.nodes.find(n => n.type === 'trigger');
      if (trigger?.spec?.type === 'http' && trigger.spec.method && trigger.spec.path) {
        httpEndpoints.set(`${trigger.spec.method} ${trigger.spec.path}`, flow.id);
      }
    }
  }

  for (const domain of domains) {
    for (const flow of domain.flows) {
      for (const node of flow.nodes) {
        if (node.type === 'service_call' && node.spec?.method && node.spec?.path) {
          const key = `${node.spec.method} ${node.spec.path}`;
          if (!httpEndpoints.has(key)) {
            issues.push({
              id: crypto.randomUUID(), scope: 'system', severity: 'warning',
              category: 'cross_domain_data', domainId: domain.name, flowId: flow.id,
              message: `Service call in "${flow.id}" targets "${key}" which doesn't match any flow endpoint`,
              suggestion: 'Verify the target endpoint exists or create the flow',
            });
          }
        }
      }
    }
  }

  return issues;
}

// ============================================================
// HELPERS
// ============================================================

function issue(
  severity: ValidationIssue['severity'],
  category: ValidationIssue['category'],
  message: string,
  flowId?: string,
  nodeId?: string,
  suggestion?: string
): ValidationIssue {
  return {
    id: crypto.randomUUID(),
    scope: 'flow',
    severity,
    category,
    message,
    flowId,
    nodeId,
    suggestion,
  };
}

function buildResult(scope: ValidationResult['scope'], targetId: string, issues: ValidationIssue[]): ValidationResult {
  return {
    scope,
    targetId,
    issues,
    errorCount: issues.filter(i => i.severity === 'error').length,
    warningCount: issues.filter(i => i.severity === 'warning').length,
    infoCount: issues.filter(i => i.severity === 'info').length,
    isValid: issues.filter(i => i.severity === 'error').length === 0,
    validatedAt: new Date().toISOString(),
  };
}

function extractErrorCodes(node: DddNode): string[] {
  const codes: string[] = [];
  const spec = node.spec || {};
  if (spec.error_code) codes.push(spec.error_code);
  if (spec.on_true?.error_code) codes.push(spec.on_true.error_code);
  if (spec.on_false?.error_code) codes.push(spec.on_false.error_code);
  return codes;
}

function hasCycle(flow: DddFlow): boolean {
  const nodeMap = new Map(flow.nodes.map(n => [n.id, n]));
  const visiting = new Set<string>();
  const visited = new Set<string>();

  function dfs(nodeId: string): boolean {
    if (visiting.has(nodeId)) return true;
    if (visited.has(nodeId)) return false;
    visiting.add(nodeId);
    const node = nodeMap.get(nodeId);
    if (node) {
      for (const target of Object.values(node.connections || {})) {
        if (typeof target === 'string' && dfs(target)) return true;
      }
    }
    visiting.delete(nodeId);
    visited.add(nodeId);
    return false;
  }

  for (const node of flow.nodes) {
    if (dfs(node.id)) return true;
  }
  return false;
}
```

**File: `src/stores/validation-store.ts`**
```typescript
import { create } from 'zustand';
import { ValidationResult, ImplementGateState } from '../types/validation';
import { validateFlow, validateDomain, validateSystem } from '../utils/flow-validator';
import { useFlowStore } from './flow-store';
import { useProjectStore } from './project-store';

interface ValidationStore {
  // Results cache
  flowResults: Record<string, ValidationResult>;
  domainResults: Record<string, ValidationResult>;
  systemResult: ValidationResult | null;

  // Actions
  validateCurrentFlow: () => void;
  validateDomain: (domainName: string) => void;
  validateSystem: () => void;
  validateAll: () => void;

  // Implementation gate
  checkImplementGate: (flowId: string) => ImplementGateState;

  // Node-level queries
  getNodeIssues: (flowId: string, nodeId: string) => ValidationResult['issues'];
}

export const useValidationStore = create<ValidationStore>((set, get) => ({
  flowResults: {},
  domainResults: {},
  systemResult: null,

  validateCurrentFlow: () => {
    const flow = useFlowStore.getState().currentFlow;
    if (!flow) return;

    const project = useProjectStore.getState();
    const errorCodes = project.errorCodes || [];
    const schemaNames = project.schemaNames || [];
    const allFlowIds = project.allFlowIds || [];

    const result = validateFlow(flow, errorCodes, schemaNames, allFlowIds);
    set({ flowResults: { ...get().flowResults, [flow.id]: result } });
  },

  validateDomain: (domainName) => {
    const project = useProjectStore.getState();
    const domainFlows = project.getFlowsForDomain(domainName);
    const domainConfig = project.getDomainConfig(domainName);

    const result = validateDomain(domainName, domainFlows, domainConfig);
    set({ domainResults: { ...get().domainResults, [domainName]: result } });
  },

  validateSystem: () => {
    const project = useProjectStore.getState();
    const domains = project.domains.map(d => ({
      name: d.name,
      config: project.getDomainConfig(d.name),
      flows: project.getFlowsForDomain(d.name),
    }));
    const schemas = project.schemaNames || [];
    const allFlowIds = project.allFlowIds || [];

    const result = validateSystem(domains, schemas, allFlowIds);
    set({ systemResult: result });
  },

  validateAll: () => {
    const project = useProjectStore.getState();
    // Validate all flows
    for (const flowId of project.allFlowIds) {
      const flow = project.getFlow(flowId);
      if (flow) {
        const result = validateFlow(flow, project.errorCodes, project.schemaNames, project.allFlowIds);
        set(state => ({
          flowResults: { ...state.flowResults, [flowId]: result },
        }));
      }
    }
    // Validate all domains
    for (const domain of project.domains) {
      get().validateDomain(domain.name);
    }
    // Validate system
    get().validateSystem();
  },

  checkImplementGate: (flowId) => {
    // Ensure fresh validation
    get().validateCurrentFlow();
    const flow = useFlowStore.getState().currentFlow;
    if (flow) {
      const domainName = flow.domain;
      if (domainName) get().validateDomain(domainName);
    }
    get().validateSystem();

    const flowResult = get().flowResults[flowId] || null;
    const domainResult = flow?.domain ? get().domainResults[flow.domain] || null : null;
    const systemResult = get().systemResult;

    const hasErrors = [flowResult, domainResult, systemResult]
      .some(r => r && r.errorCount > 0);
    const hasWarnings = [flowResult, domainResult, systemResult]
      .some(r => r && r.warningCount > 0);

    return {
      flowValidation: flowResult,
      domainValidation: domainResult,
      systemValidation: systemResult,
      canImplement: !hasErrors,
      hasWarnings,
    };
  },

  getNodeIssues: (flowId, nodeId) => {
    const result = get().flowResults[flowId];
    if (!result) return [];
    return result.issues.filter(i => i.nodeId === nodeId);
  },
}));
```

**Component: `src/components/Validation/ValidationPanel.tsx`**
```tsx
import { ValidationResult, ValidationIssue } from '../../types/validation';

interface Props {
  result: ValidationResult;
  title: string;
  onSelectNode?: (nodeId: string) => void;
}

export function ValidationPanel({ result, title, onSelectNode }: Props) {
  const errors = result.issues.filter(i => i.severity === 'error');
  const warnings = result.issues.filter(i => i.severity === 'warning');
  const infos = result.issues.filter(i => i.severity === 'info');

  return (
    <div className="validation-panel p-3">
      <h3 className="font-semibold mb-2">Validation — {title}</h3>
      <div className="text-sm mb-3">
        {result.errorCount > 0 && <span className="text-red-600 mr-3">✗ {result.errorCount} errors</span>}
        {result.warningCount > 0 && <span className="text-amber-600 mr-3">⚠ {result.warningCount} warnings</span>}
        {result.infoCount > 0 && <span className="text-blue-600">ℹ {result.infoCount} info</span>}
        {result.isValid && result.warningCount === 0 && (
          <span className="text-green-600">✓ All checks passed</span>
        )}
      </div>

      {errors.length > 0 && (
        <IssueSection title="Errors (must fix)" issues={errors}
          color="red" onSelectNode={onSelectNode} />
      )}
      {warnings.length > 0 && (
        <IssueSection title="Warnings" issues={warnings}
          color="amber" onSelectNode={onSelectNode} />
      )}
      {infos.length > 0 && (
        <IssueSection title="Info" issues={infos}
          color="blue" onSelectNode={onSelectNode} />
      )}

    </div>
  );
}

function IssueSection({ title, issues, color, onSelectNode }: {
  title: string; issues: ValidationIssue[]; color: string;
  onSelectNode?: (nodeId: string) => void;
}) {
  return (
    <div className="mb-3">
      <h4 className={`text-sm font-medium text-${color}-700 mb-1`}>{title}</h4>
      {issues.map(issue => (
        <div key={issue.id} className={`border-l-2 border-${color}-300 pl-3 py-1.5 mb-1 text-sm`}>
          <div>{issue.message}</div>
          {issue.suggestion && (
            <div className="text-xs text-gray-500 mt-0.5">→ {issue.suggestion}</div>
          )}
          {issue.nodeId && onSelectNode && (
            <button onClick={() => onSelectNode(issue.nodeId!)}
              className="text-xs text-blue-600 underline mt-0.5">
              Select node
            </button>
          )}
        </div>
      ))}
    </div>
  );
}
```

**Component: `src/components/Validation/ValidationBadge.tsx`**
```tsx
interface Props {
  errorCount: number;
  warningCount: number;
  onClick?: () => void;
}

export function ValidationBadge({ errorCount, warningCount, onClick }: Props) {
  if (errorCount > 0) {
    return (
      <button onClick={onClick} className="inline-flex items-center gap-1 px-2 py-0.5 rounded text-xs bg-red-100 text-red-800">
        ✗ {errorCount} errors
      </button>
    );
  }
  if (warningCount > 0) {
    return (
      <button onClick={onClick} className="inline-flex items-center gap-1 px-2 py-0.5 rounded text-xs bg-yellow-100 text-yellow-800">
        ⚠ {warningCount} warnings
      </button>
    );
  }
  return (
    <span className="inline-flex items-center gap-1 px-2 py-0.5 rounded text-xs bg-green-100 text-green-800">
      ✓ valid
    </span>
  );
}
```

**Component: `src/components/Validation/ImplementGate.tsx`**
```tsx
import { useValidationStore } from '../../stores/validation-store';
import { ValidationPanel } from './ValidationPanel';

export function ImplementGate({ flowId, onProceed }: { flowId: string; onProceed: () => void }) {
  const { checkImplementGate } = useValidationStore();
  const gate = checkImplementGate(flowId);

  return (
    <div className="implement-gate p-3 border rounded">
      <h3 className="font-semibold mb-3">Pre-Implementation Check</h3>

      <div className="space-y-2 mb-4">
        <GateStep label="Flow validation" result={gate.flowValidation} />
        <GateStep label="Domain validation" result={gate.domainValidation} />
        <GateStep label="System validation" result={gate.systemValidation} />
      </div>

      {!gate.canImplement ? (
        <div className="text-red-600 text-sm mb-3">
          ✗ Cannot implement — fix all errors first
        </div>
      ) : gate.hasWarnings ? (
        <div className="flex gap-2">
          <button onClick={onProceed} className="btn btn-primary btn-sm">
            Implement anyway ({gate.flowValidation?.warningCount || 0} warnings)
          </button>
          <button className="btn btn-sm">Fix first</button>
        </div>
      ) : (
        <button onClick={onProceed} className="btn btn-primary btn-sm bg-green-600">
          ▶ Implement (all checks passed)
        </button>
      )}
    </div>
  );
}

function GateStep({ label, result }: { label: string; result: any }) {
  if (!result) return <div className="text-sm text-gray-400">⏳ {label}: checking...</div>;
  if (result.errorCount > 0) return <div className="text-sm text-red-600">✗ {label}: {result.errorCount} errors</div>;
  if (result.warningCount > 0) return <div className="text-sm text-amber-600">⚠ {label}: {result.warningCount} warnings</div>;
  return <div className="text-sm text-green-600">✓ {label}: passed</div>;
}
```

**Integration: auto-validate on flow changes (debounced):**
```typescript
// In flow-store.ts or a useEffect in Canvas.tsx
import { debounce } from '../utils/debounce';

const debouncedValidate = debounce(() => {
  useValidationStore.getState().validateCurrentFlow();
}, 500);

// Call after every mutation:
addNode: (node) => {
  // ... existing logic
  debouncedValidate();
},
deleteNode: (nodeId) => {
  // ... existing logic
  debouncedValidate();
},
// ... same for moveNode, addConnection, updateSpec, etc.
```

**Integration: node border color from validation:**
```tsx
// In Node.tsx, determine border color:
const nodeIssues = useValidationStore.getState().getNodeIssues(flowId, node.id);
const hasErrors = nodeIssues.some(i => i.severity === 'error');
const hasWarnings = nodeIssues.some(i => i.severity === 'warning');

const borderClass = hasErrors ? 'border-red-500 border-2'
  : hasWarnings ? 'border-amber-400 border-2'
  : 'border-gray-300';

// Tooltip on hover showing issues
{nodeIssues.length > 0 && (
  <div className="absolute -top-1 -right-1 w-3 h-3 rounded-full bg-red-500" title={
    nodeIssues.map(i => i.message).join('\n')
  } />
)}
```

---

## Phase 4: Testing the MVP

### Manual Test Checklist

1. **System Map (Level 1)**
   - [ ] App starts on System Map
   - [ ] All domains from system.yaml shown as blocks
   - [ ] Flow count badge visible on each domain block
   - [ ] Event arrows drawn between connected domains
   - [ ] Domain blocks are draggable (positions persist)
   - [ ] Double-click domain → navigates to Domain Map

2. **Domain Map (Level 2)**
   - [ ] Flow blocks shown for all flows in domain
   - [ ] Portal nodes shown for cross-domain connections
   - [ ] Event arrows connect flows to portals
   - [ ] Double-click flow block → navigates to Flow Sheet
   - [ ] Double-click portal → navigates to target domain's map
   - [ ] Flow blocks are draggable (positions persist)

3. **Breadcrumb Navigation**
   - [ ] Breadcrumb shows "System" on Level 1
   - [ ] Breadcrumb shows "System > domain" on Level 2
   - [ ] Breadcrumb shows "System > domain > flow" on Level 3
   - [ ] Click earlier breadcrumb segment → navigates to that level
   - [ ] Backspace/Esc → navigates one level up
   - [ ] Backspace on System Map → no-op

4. **Create new flow**
   - [ ] Click "New Flow" button
   - [ ] Enter name and domain
   - [ ] Canvas shows trigger + terminal nodes

5. **Add nodes**
   - [ ] Drag node from palette to canvas
   - [ ] Node appears at drop position
   - [ ] Node is selectable

6. **Edit node spec**
   - [ ] Select node
   - [ ] Spec panel shows correct form
   - [ ] Changes update node data

7. **Connect nodes**
   - [ ] Drag from output to input
   - [ ] Connection line appears
   - [ ] Connection saved in flow data

8. **Sub-flow navigation**
   - [ ] Double-click sub-flow node → navigates to referenced flow sheet
   - [ ] Breadcrumb updates to show new flow
   - [ ] Backspace returns to previous flow

9. **Agent flow canvas**
   - [ ] Create new agent flow (type: agent)
   - [ ] Agent canvas renders vertical layout (not traditional graph)
   - [ ] Agent Loop block shows model, system prompt, reasoning cycle
   - [ ] Tools displayed inside Agent Loop block as palette
   - [ ] Terminal tools have stop indicator
   - [ ] Input guardrail shows above Agent Loop
   - [ ] Output guardrail shows below Agent Loop
   - [ ] Human Gate shows as separate escalation path
   - [ ] Selecting Agent Loop → spec panel shows agent config editor
   - [ ] Can edit model, system prompt, max iterations, temperature
   - [ ] Can add/remove/edit tools
   - [ ] Can add/remove guardrail checks

10. **Orchestration canvas**
    - [ ] Orchestrator block shows strategy, managed agents, supervision rules
    - [ ] Smart Router block shows rules, LLM fallback, circuit breaker badge, A/B badge
    - [ ] Handoff block shows mode (transfer/consult/collaborate) and target
    - [ ] Agent Group boundary wraps grouped agents with dashed border
    - [ ] Agent Group shows shared memory badges at bottom
    - [ ] Selecting orchestrator → spec panel shows orchestrator config
    - [ ] Can add/remove managed agents and supervision rules
    - [ ] Selecting smart router → spec panel shows rules, policies, A/B config
    - [ ] Selecting handoff → spec panel shows mode, context, failure config

11. **Orchestration on Level 2**
    - [ ] Agent flow blocks show agent badge (▣⊛)
    - [ ] Supervisor arrows (⊛▶) connect orchestrator to managed agents
    - [ ] Handoff arrows (⇄) shown between agents with mode label
    - [ ] Agent Group appears as dashed boundary on Domain Map

12. **Save flow**
    - [ ] Click Save on traditional flow → correct YAML
    - [ ] Click Save on agent flow → YAML includes agent_config, tools, memory, guardrails
    - [ ] Click Save on orchestration flow → YAML includes agent_group, orchestrator, smart_router, handoff
    - [ ] File content matches flow

13. **Load flow**
    - [ ] Load traditional YAML → traditional canvas
    - [ ] Load agent YAML → agent canvas
    - [ ] Load orchestration YAML → orchestration canvas with group boundaries
    - [ ] Spec panel shows correct data for all types

14. **Git status**
    - [ ] Panel shows current branch
    - [ ] Changed files listed
    - [ ] Stage/commit works

15. **Drift Detection**
    - [ ] On project load, all flow hashes compared against mapping.yaml
    - [ ] Stale flows show warning badge on L2 (Domain Map)
    - [ ] Stale flow sheet shows banner with Accept + Dismiss buttons
    - [ ] Spec cached at implementation time in .ddd/cache/specs-at-implementation/
    - [ ] Sync score computed from mappings (total/implemented/stale/pending)
    - [ ] `resolveFlow('reimpl')` copies `/ddd-implement {flowKey}` to clipboard
    - [ ] `resolveFlow('accept')` updates stored hash to current
    - [ ] `resolveFlow('ignore')` adds to ignoredDrifts set
    - [ ] Reconciliation reports saved to .ddd/reconciliations/

16. **Prompt Builder**
    - [ ] Prompt includes architecture.yaml, errors.yaml, referenced schemas, flow spec
    - [ ] Schema resolution: only schemas referenced by $ref or data_store model are included
    - [ ] Agent flows include agent-specific instructions (tools, guardrails, memory)
    - [ ] Update mode: prompt includes list of spec changes and targets existing code

17. **CLAUDE.md Auto-Generation**
    - [ ] CLAUDE.md generated on first implementation
    - [ ] Regenerated when implementation status changes
    - [ ] Includes project name, spec files, domain table, rules, tech stack, commands
    - [ ] Custom section (below `<!-- CUSTOM -->` marker) preserved on regeneration
    - [ ] Domain table shows implementation status (N implemented, M pending)

24. **Test Case Derivation**
    - [ ] "Derive tests" button walks flow graph and enumerates all paths (trigger → terminal)
    - [ ] Happy path, error path, and edge case paths correctly identified
    - [ ] Boundary tests derived from input node validation rules (min/max/missing/format)
    - [ ] Agent test cases derived: tool success/failure, guardrail block, max iterations, human gate approve/reject
    - [ ] Orchestration test cases derived: routing per rule, fallback, circuit breaker, handoff, supervisor override
    - [ ] Derived test spec shows path count, boundary count, and total test case count

25. **Test Code Generation**
    - [ ] "Generate test code" uses Claude Code with derived spec + flow YAML
    - [ ] Generated code uses project's test framework (from architecture.yaml)
    - [ ] Generated tests use exact error messages and status codes from spec
    - [ ] "Include in prompt" appends generated tests to the Claude Code prompt
    - [ ] Coverage badge appears on Level 2 flow blocks after generation

26. **Spec Compliance Validation**
    - [ ] "Check compliance" compares test results against derived spec expectations
    - [ ] Compliance report shows compliant/non-compliant items with score percentage
    - [ ] Non-compliant items show expected vs actual diff
    - [ ] "Fix via Claude Code" generates targeted fix prompt for each non-compliant item
    - [ ] "Fix all non-compliant" batches all issues into one prompt
    - [ ] Compliance data persisted in mapping.yaml (score, issues, timestamps)

27. **Project Launcher**
    - [ ] App launches to Project Launcher (not directly into canvas)
    - [ ] Recent projects listed with name, path, last opened timestamp
    - [ ] Click recent project → loads project and navigates to System Map
    - [ ] Right-click recent project → "Remove from recent"
    - [ ] "New Project" opens 3-step wizard (basics, tech stack, domains)
    - [ ] Wizard creates specs/ directory, system.yaml, architecture.yaml, domain.yaml files
    - [ ] Wizard initializes Git repo if checkbox checked
    - [ ] "Open Existing" shows OS file picker, validates folder is a DDD project
    - [ ] Non-DDD folder shows error with "Initialize as DDD project?" recovery action
    - [ ] Recent projects pruned on load (remove entries where folder no longer exists)

28. **Settings Screen**
    - [ ] Accessible via menu bar → Settings or Cmd+, shortcut
    - [ ] Tab navigation: Editor, Claude Code, Git
    - [ ] Global vs Project scope toggle
    - [ ] Editor tab: grid snap, auto-save interval, theme (light/dark/system), font size
    - [ ] Settings persist to ~/.ddd-tool/settings.json (global) and .ddd/config.yaml (project)

29. **First-Run Experience**
    - [ ] First-run detected when ~/.ddd-tool/ directory doesn't exist
    - [ ] 2-step wizard: Claude Code detection → Get Started
    - [ ] Claude Code auto-detection: checks if `claude` is in PATH
    - [ ] "Explore with sample project" option opens bundled read-only example
    - [ ] Creates ~/.ddd-tool/ with settings.json and recent-projects.json
    - [ ] Subsequent launches go straight to Project Launcher

30. **Error Handling**
    - [ ] Error toasts appear in bottom-right corner
    - [ ] Info severity auto-dismisses after 5 seconds
    - [ ] Warning/error severity requires manual dismiss
    - [ ] Fatal severity shows modal blocking all actions
    - [ ] YAML parse errors show line number and revert to last valid state
    - [ ] Recovery actions available in error toasts (e.g., "Retry", "Revert")
    - [ ] Auto-save writes to .ddd/autosave/ (not real spec files)
    - [ ] Crash recovery dialog on next launch if autosave data exists
    - [ ] All errors logged to ~/.ddd-tool/logs/ddd-tool.log

31. **Undo/Redo**
    - [ ] Cmd+Z undoes last canvas action, Cmd+Shift+Z redoes
    - [ ] Undo/redo is per-flow (each flow has its own history stack)
    - [ ] Undoable: add/delete/move node, connect/disconnect, edit spec field
    - [ ] NOT undoable: git commit, claude code implementation, file save
    - [ ] History coalesces rapid changes (typing in same field < 500ms apart)
    - [ ] Max 100 snapshots per flow (oldest dropped when exceeded)
    - [ ] Undo/redo buttons in toolbar with tooltip showing what will be undone/redone
    - [ ] Buttons grayed out when stack is empty
    - [ ] Stack cleared when flow is closed

32. **Flow-Level Validation**
    - [ ] Graph completeness: trigger exists, all paths reach terminal, no orphaned nodes
    - [ ] Dead-end detection: non-terminal nodes with no outgoing connections flagged as error
    - [ ] Decision branches: both true and false must be connected
    - [ ] Input branches: valid branch required, invalid branch warned
    - [ ] Cycle detection for traditional flows (agent flows excluded)
    - [ ] Spec completeness: trigger type, HTTP method/path, input field types, error messages
    - [ ] Reference integrity: error codes exist in errors.yaml, models exist in schemas/
    - [ ] Agent validation: agent_loop exists, tools connected, terminal tool exists, max_iterations
    - [ ] Orchestration validation: agents assigned, strategy defined, rules defined, targets exist

33. **Domain-Level Validation**
    - [ ] Duplicate flow IDs detected within a domain
    - [ ] Duplicate HTTP endpoints detected within a domain
    - [ ] Domain.yaml flow list matches actual flow files on disk

34. **System-Level Validation (Cross-Domain Wiring)**
    - [ ] Consumed events have at least one publisher (error if missing)
    - [ ] Published events have at least one consumer (warning if unused)
    - [ ] Event payload shapes match between publisher and consumer
    - [ ] Event naming consistency checked (dot.case vs camelCase vs snake_case)
    - [ ] Portal targets point to existing domains
    - [ ] Circular orchestration dependencies detected (A→B→A)
    - [ ] Cross-domain API dependencies verified (service_call targets exist)

35. **Validation UI and Gate**
    - [ ] Real-time canvas indicators: green/amber/red borders on nodes, red dot for errors
    - [ ] Validation panel accessible from toolbar (Level 1/2/3)
    - [ ] Issues show message, suggestion, and "Select node" link
    - [ ] Issues show actionable suggestions for manual fixing
    - [ ] Implementation gate: 3-step (validate → prompt → run)
    - [ ] Errors block implementation — button disabled
    - [ ] Warnings allow "Implement anyway" with warning count
    - [ ] Batch implementation pre-validates all selected flows before starting
    - [ ] Validation badges on Level 1 (domain blocks) and Level 2 (flow blocks)

---

## Phase 5: Key Implementation Notes

### Important Patterns

1. **State Management**
   - Use Zustand for all state
   - Keep stores focused (sheet, flow, project, ui, git, implementation, app, undo, validation)
   - No LLM or memory stores — AI assistance is provided via Claude Code CLI
   - `sheet-store` owns navigation state (current level, breadcrumbs, history)
   - `project-store` owns domain configs parsed from domain.yaml files
   - `flow-store` owns current flow being edited (Level 3 only)
   - `implementation-store` owns drift detection, mapping persistence
   - `app-store` owns app-level view state, recent projects, global settings, errors, auto-save
   - `undo-store` owns per-flow undo/redo stacks with immutable snapshots
   - `validation-store` owns per-flow/domain/system validation results and implementation gate
   - Never mutate state directly

2. **Tauri Commands**
   - All file/git/PTY/test operations go through Tauri
   - Use async/await pattern
   - Handle errors gracefully
   - PTY commands manage Claude Code terminal sessions (spawn, read, write, wait, kill)
   - Test runner commands execute test commands and parse output

3. **YAML Format**
   - Follow spec exactly (see ddd-specification-complete.md)
   - Use $ref for shared schemas
   - Include metadata (createdAt, updatedAt, completeness)

4. **Canvas Performance**
   - Use React.memo for nodes
   - Batch position updates
   - Debounce auto-save

5. **Agent vs Traditional Flows**
   - Detect flow type from YAML (`type: agent` vs `type: traditional` or absent)
   - Route to `AgentCanvas` or traditional `Canvas` based on flow type
   - Agent flows use a vertical layout (guardrail → loop → outputs), not free-form node graph
   - Tools live inside the agent loop spec, not as separate canvas nodes
   - Agent spec panels are distinct from traditional spec panels

6. **Orchestration Rendering**
   - Orchestration nodes (Orchestrator, Smart Router, Handoff, Agent Group) extend the AgentCanvas
   - Agent Group renders as a wrapper/boundary around its child components
   - Orchestrator sits at the top of the group, with managed agents below
   - Smart Router renders within the orchestrator's routing section
   - Handoff arrows are bidirectional for consult/collaborate, one-way for transfer
   - Level 2 Domain Map derives orchestration topology from flow YAML (agent_group, orchestrator nodes)
   - Level 2 uses special arrow types: supervisor (⊛▶), handoff (⇄), event (──▶)

### Common Pitfalls

1. **Don't** build code generation in MVP
2. **Don't** build MCP server
3. **Don't** build reverse engineering
4. **Do** support entity management on L1/L2 (add/rename/delete domains and flows) via right-click context menus — all operations must update YAML files and cross-references atomically
5. **Don't** try to run/test agents within the DDD Tool in MVP — just design them
6. **Don't** try to execute routing rules or circuit breakers — DDD is a design tool, not a runtime
7. **Do** focus on visual editing → YAML output
8. **Do** build multi-level navigation early — it's the core UX
9. **Do** parse domain.yaml files on project load to populate L1/L2
10. **Do** support all three flow types (traditional, agent, orchestration) in canvas routing
11. **Do** derive Level 2 orchestration visuals from flow YAML automatically
12. **Do** make sure Git integration works
13. **Do** validate YAML output format (traditional, agent, and orchestration)
14. **Don't** auto-commit after Claude Code finishes — always let the user review generated code first
15. **Don't** run Claude Code in headless/non-interactive mode — the user must be able to approve file operations
16. **Do** cache spec files at implementation time in `.ddd/cache/` for accurate stale diffs later
17. **Do** include only referenced schemas in prompts (resolve from `$ref` and `data_store` model), not all schemas
18. **Do** preserve the `<!-- CUSTOM -->` section when regenerating CLAUDE.md — users add project-specific instructions there
19. **Don't** parse test output from raw text in production — use JSON reporters (`--json-report` for pytest, `--json` for jest) for accurate test results
20. **Do** always include the Implementation Report instruction in prompts — without it, reverse drift detection has no structured data to parse
21. **Don't** auto-accept reconciliation items without user review — always show the report and let the user decide accept/remove/ignore
22. **Do** store accepted deviations in mapping.yaml so they don't re-trigger drift warnings on every reconciliation
23. **Don't** run reconciliation with the full codebase — only send the files listed in the flow's mapping, not the entire project
24. **Do** derive tests deterministically from the graph (DFS path enumeration) — don't use Claude Code for path analysis, only for test code generation
25. **Don't** modify derived test assertions when including in Claude Code prompt — they are the executable spec, changing them defeats the purpose
26. **Do** generate boundary tests for every validation field, not just required fields — min/max boundaries catch off-by-one errors that manual testing misses
27. **Don't** run spec compliance check before tests pass — compliance compares expected vs actual, which is meaningless if tests are failing for other reasons
28. **Do** persist derived test counts in mapping.yaml — the coverage badge needs this data without re-deriving on every load
29. **Do** launch to Project Launcher, not directly into canvas — users need to choose/create a project first
30. **Don't** store raw API keys anywhere on disk — store only env var names in config, read from `process.env` at runtime, or use OS keychain
31. **Do** validate DDD project folders on open — check for `specs/system.yaml` or `.ddd/config.yaml` before attempting to load
32. **Don't** auto-save directly to spec files — auto-save writes to `.ddd/autosave/` to prevent corrupting specs on crash
33. **Do** use `structuredClone` for undo snapshots — shallow copies will share references and corrupt history
34. **Do** coalesce undo snapshots for rapid keystrokes (< 500ms same field) — otherwise typing a word creates 5 undo entries
35. **Don't** include undo/redo for side-effects (git, file writes) — only canvas and spec mutations are undoable
36. **Do** prune recent projects on load — removing entries where the folder no longer exists prevents confusing dead links
37. **Do** show the first-run wizard only once — set a flag in `~/.ddd-tool/settings.json` after completion
38. **Do** debounce flow validation (500ms) — validating on every keystroke will lag the canvas
39. **Don't** block the canvas while validation runs — validate async and update indicators when done
40. **Do** validate before implementation, not just during editing — the gate is the last line of defense
41. **Do** validate cross-domain event payloads structurally (field names + types), not just by event name — two domains consuming the same event name but expecting different shapes is a common bug
42. **Do** re-run system validation after git pull — specs may have changed on disk
43. **Don't** validate deleted/orphaned spec files — only validate flows that exist in the current project index
44. **Do** update system.yaml atomically with domain directory operations — a domain directory without a system.yaml entry (or vice versa) is a corrupt state
45. **Don't** allow renaming a domain/flow to an existing name — check for duplicates before invoking the Tauri command
46. **Do** update all cross-domain references when renaming a domain — event wiring, portal targets, orchestration agent refs, and mapping.yaml entries all contain domain names
47. **Don't** silently delete domains with flows — always show a confirmation dialog with flow count before deleting
48. **Do** reload the full project after any entity CRUD operation — partial state updates are error-prone; a full reload from disk is safer and simpler
49. **Don't** allow moving a flow to its current domain — filter out the source domain from the "Move to..." submenu
50. **Do** warn users when changing flow type will lose type-specific data (agent_loop, orchestrator sections) — show a confirmation before destructive type conversion

### Null-Safety Conventions

- Always use `(data.spec ?? {}) as SomeSpec` — never bare `data.spec as SomeSpec` (spec can be undefined when a node is first created)
- Always use `node.connections ?? []` — never bare `node.connections` (connections array may be absent in external YAML)
- Always use `currentFlow.flow?.id` — never `currentFlow.flow.id` (flow can be null between loads)
- When fixing a cross-cutting null-safety pattern, audit the entire codebase first and fix all instances in one pass — piecemeal fixes leave inconsistent guard coverage

---

## Commands to Start

```bash
# 1. Create project
npm create tauri-app@latest ddd-tool -- --template react-ts
cd ddd-tool

# 2. Install dependencies
npm install zustand yaml nanoid lucide-react
npm install -D tailwindcss postcss autoprefixer @types/node
npx tailwindcss init -p

# 3. Add Rust dependencies to Cargo.toml
# In src-tauri/Cargo.toml, add:
# [dependencies]
# git2 = "0.18"
# serde_json = "1.0"
# portable-pty = "0.8"
# sha2 = "0.10"
# nanoid = "0.4"

# 4. Start development
npm run tauri dev
```

---

## Success Criteria for MVP

### Multi-Level Navigation
- [ ] System Map (L1) shows all domains as blocks with flow counts
- [ ] Event arrows render between connected domains on System Map
- [ ] Double-click domain block → drills into Domain Map (L2)
- [ ] Domain Map shows flow blocks and portal nodes
- [ ] Double-click flow block → drills into Flow Sheet (L3)
- [ ] Double-click portal node → jumps to target domain's map
- [ ] Breadcrumb bar shows current location (System > domain > flow)
- [ ] Clicking breadcrumb segment navigates to that level
- [ ] Backspace/Esc navigates one level up
- [ ] Block positions on L1/L2 are draggable and persisted

### Traditional Flow Sheet (L3)
- [ ] Can create a new traditional flow with name and domain
- [ ] Can add 5 node types to canvas
- [ ] Can edit node specs in right panel
- [ ] Can connect nodes together
- [ ] Sub-flow nodes navigate to referenced flow sheet on double-click

### Agent Flow Sheet (L3)
- [ ] Can create a new agent flow (type: agent)
- [ ] Agent canvas shows agent-centric layout (not traditional node graph)
- [ ] Agent Loop block displays model, system prompt, reasoning cycle
- [ ] Tools rendered as palette within Agent Loop block
- [ ] Terminal tools marked with stop indicator
- [ ] Input/output guardrails shown above/below agent loop
- [ ] Human Gate shown as escalation path
- [ ] Spec panel shows agent-specific editors (model, prompt, tools, etc.)
- [ ] Can add/remove/edit tools in the tool palette
- [ ] Can configure guardrail checks
- [ ] Can configure human gate options and timeout
- [ ] Agent flow YAML includes agent_config, tools, memory, guardrails sections

### Orchestration (L3 + L2)
- [ ] Orchestrator block renders with strategy badge, managed agents list, supervision rules
- [ ] Can add/remove/edit managed agents in orchestrator spec panel
- [ ] Can configure supervision rules (conditions, thresholds, actions)
- [ ] Can select orchestration strategy (supervisor, round_robin, broadcast, consensus)
- [ ] Can set result merge strategy
- [ ] Smart Router block renders with rule-based routes, LLM fallback, policies
- [ ] Can add/remove/edit routing rules with conditions and priorities
- [ ] Can configure routing policies (retry, timeout, circuit breaker)
- [ ] Can configure A/B testing experiments
- [ ] Can configure context-aware routing rules
- [ ] Handoff block renders with mode indicator (transfer/consult/collaborate)
- [ ] Can configure context transfer (include/exclude, max tokens)
- [ ] Can configure on_complete behavior (return_to, merge strategy)
- [ ] Agent Group renders as dashed boundary around grouped agents
- [ ] Agent Group shows shared memory indicators and coordination settings
- [ ] Level 2 Domain Map shows supervisor arrows (⊛▶) from orchestrator to agents
- [ ] Level 2 Domain Map shows handoff arrows (⇄) between agents with mode label
- [ ] Level 2 Domain Map shows agent group boundaries as dashed rectangles
- [ ] Level 2 distinguishes agent flow blocks (▣⊛) from traditional flow blocks (▣)
- [ ] Orchestration flow YAML includes agent_group, orchestrator, smart_router, handoff nodes

### File Operations
- [ ] Can save flow as YAML file (traditional, agent, and orchestration formats)
- [ ] Can load flow from YAML file (detects type automatically)
- [ ] Can see Git status
- [ ] Can stage and commit changes
- [ ] YAML format matches specification
- [ ] Domain configs (domain.yaml) are parsed to populate L1/L2

### Claude Code Integration (Drift Detection + CLI)
- [ ] Spec hash saved to .ddd/mapping.yaml after implementation via CLI
- [ ] Spec file cached in .ddd/cache/specs-at-implementation/ for future diff
- [ ] Stale detection compares SHA-256 hashes on project load, flow save, and git pull
- [ ] Stale flows show warning badge on L2 and banner on L3 with Accept + Dismiss buttons
- [ ] `resolveFlow('reimpl')` copies `/ddd-implement {flowKey}` to clipboard
- [ ] `resolveFlow('accept')` updates stored hash to current
- [ ] `resolveFlow('ignore')` adds to ignoredDrifts set
- [ ] Sync score computed from mappings (total/implemented/stale/pending)
- [ ] Reconciliation reports saved to .ddd/reconciliations/
- [ ] CLAUDE.md auto-generated with project info, domain table, rules, tech stack
- [ ] CLAUDE.md custom section preserved on regeneration
- [ ] Configuration loaded from .ddd/config.yaml with sensible defaults

### Diagram-Derived Test Generation
- [ ] "Derive tests" walks flow graph and enumerates all paths from trigger to terminal
- [ ] Path types correctly classified (happy_path, error_path, edge_case)
- [ ] Boundary tests derived from every input node field with validation rules
- [ ] Agent tests derived (tool success/failure, guardrail, max iterations, human gate)
- [ ] Orchestration tests derived (routing rules, fallback, circuit breaker, handoff)
- [ ] Test code generation uses Claude Code CLI with derived spec + flow YAML
- [ ] Generated test code uses project's test framework from architecture.yaml
- [ ] Generated tests reference exact error messages and status codes from spec

### App Shell & UX
- [ ] App launches to Project Launcher screen (not canvas)
- [ ] Recent projects load from ~/.ddd-tool/recent-projects.json with pruning
- [ ] New Project wizard creates specs/, system.yaml, architecture.yaml, domain.yaml, .ddd/, Git init
- [ ] Open Existing validates folder is a DDD project (checks specs/system.yaml or .ddd/config.yaml)
- [ ] Settings dialog opens via Cmd+, with tab navigation (Editor, Claude Code, Git)
- [ ] Global vs project scope toggle in settings
- [ ] First-run wizard detects missing ~/.ddd-tool/ and guides through Claude Code + Get Started
- [ ] Sample project available as read-only exploration (bundled in app)
- [ ] Error toasts with severity-based auto-dismiss (info=5s, warning/error=manual, fatal=modal)
- [ ] Auto-save to .ddd/autosave/ every 30s (configurable), crash recovery dialog on relaunch
- [ ] Undo/redo per-flow with Cmd+Z / Cmd+Shift+Z, max 100 snapshots, coalescing rapid changes
- [ ] Undo/redo toolbar buttons with tooltips, grayed when empty
- [ ] All errors logged to ~/.ddd-tool/logs/ddd-tool.log with rotation

### Design Validation
- [ ] Flow-level: graph completeness (trigger, reachability, dead-ends, branches), spec completeness, reference integrity
- [ ] Agent-level: agent_loop present, tools connected, terminal tool, max_iterations, guardrail checks
- [ ] Orchestration-level: agents assigned, strategy defined, rules+targets exist, handoff targets exist
- [ ] Domain-level: no duplicate flow IDs, no duplicate HTTP endpoints, domain.yaml consistency
- [ ] System-level: consumed events have publishers, payload shapes match, naming consistency
- [ ] System-level: portal targets exist, no circular orchestration, cross-domain API targets verified
- [ ] Canvas real-time indicators: node borders (green/amber/red), error dots, hover tooltips
- [ ] Validation panel per scope (flow/domain/system) with grouped issues + "Select node"
- [ ] Implementation gate blocks on errors, warns on warnings, green on clean
- [ ] Batch implementation pre-validates all selected flows
- [ ] Validation badges on Level 1 domain blocks and Level 2 flow blocks

### Entity Management
- [ ] Right-click L1 canvas background → "Add domain" dialog → creates directory + domain.yaml + updates system.yaml
- [ ] Right-click domain block → "Rename" → renames directory, updates domain.yaml, system.yaml, and cross-references
- [ ] Right-click domain block → "Delete" → confirmation with flow count → removes directory and system.yaml entry
- [ ] Right-click domain block → "Edit description" → inline edit → updates domain.yaml
- [ ] Right-click domain block → "Add published/consumed event" → updates domain.yaml, renders new arrow on L1
- [ ] Right-click L2 canvas background → "Add flow" dialog (name + type selector) → creates flow YAML, opens L3
- [ ] Right-click flow block → "Rename" → renames file, updates flow.id, updates cross-references
- [ ] Right-click flow block → "Delete" → confirmation → removes flow file
- [ ] Right-click flow block → "Duplicate" → creates copy with "-copy" suffix
- [ ] Right-click flow block → "Move to..." → shows other domains → moves file and updates domain field
- [ ] Right-click flow block → "Change type" → traditional/agent/orchestration with warning if data will be lost
- [ ] L3 toolbar → click flow name → inline rename → updates file and id
- [ ] L3 right-click canvas → "Clear canvas" → confirmation → removes all nodes except trigger
- [ ] All entity operations update disk files atomically (no partial states)
- [ ] Project reloads from disk after every entity CRUD to ensure consistency

---

## Session 16: First-Run, Settings, Polish

Session 16 adds first-run experience, settings persistence, undo/redo, auto-save, and crash recovery.

### Key Components

**First-Run Wizard:** `src/components/FirstRun/FirstRunWizard.tsx`
- 2-step setup: Detect Claude Code CLI → Get Started
- Detected when `~/.ddd-tool/` directory doesn't exist
- "Skip for now" and "Explore with sample project" options

**Settings Dialog:** `src/components/Settings/SettingsDialog.tsx`
- Tab navigation: Editor, Claude Code, Git
- Global scope (`~/.ddd-tool/settings.json`) vs project scope (`.ddd/config.yaml`)
- `GitSettings.tsx` — commit message templates, branch naming conventions

**Undo/Redo Store:** `src/stores/undo-store.ts`
- Per-flow undo/redo stacks with immutable snapshots
- `Cmd+Z` / `Cmd+Shift+Z` keyboard shortcuts
- Max 100 snapshots per flow, coalescing rapid changes (<500ms apart)
- Undoable: add/delete/move node, connect/disconnect, edit spec
- NOT undoable: git commit, implementation, file save

**Auto-Save:** Writes to `.ddd/autosave/` every 30s (configurable). Crash recovery dialog on relaunch detects autosave data and offers restore.

**Error Handling:** Error toasts with severity-based auto-dismiss (info=5s, warning/error=manual, fatal=modal).

---

## Session 17: Extended Nodes + Enhancements

Session 17 is a post-MVP enhancement session that adds missing node types and quality-of-life features.

### Extended Node Types

Add 6 traditional flow node types toward the spec's full set of 19:

| Node | Symbol | Purpose | Spec Panel Fields |
|------|--------|---------|-------------------|
| `data_store` | ▣ | Database CRUD operations | operation (create/read/update/delete), model, data mapping, query filters |
| `service_call` | ⇥ | External API/service calls | method, url/service, headers, body, timeout, retry, error mapping |
| `event` | ⚡ | Emit or consume domain events | event_name, payload mapping, async (fire-and-forget) vs sync |
| `loop` | ↻ | Iterate over collections | collection expression, iterator variable, body nodes, break condition |
| `parallel` | ═ | Concurrent execution branches | branches (list of node chains), join strategy (all/any/n-of), timeout |
| `sub_flow` | ⊞ | Call another flow as subroutine | flow_ref (domain/flow-id), input mapping, output mapping |

**Types** — See the consolidated `src/types/flow.ts` section in Phase 3 (Day 1-2). All 27 node types and their spec interfaces are defined there, including:
- `DddNodeType` union with all 27 types (19 traditional + 4 agent + 4 orchestration)
- `DataStoreSpec` with `pagination` and `sort` fields (for read operations)
- `ServiceCallSpec` with `error_mapping` for HTTP status → error code mapping
- `TerminalSpec` with `status` (HTTP code) and `body` (response body) fields
- `LoopSpec`, `ParallelSpec`, `EventNodeSpec`, `SubFlowSpec`, `LlmCallSpec`
- All spec interfaces use `[key: string]: unknown` for custom field extensibility

**New node components** — one file per node in `src/components/FlowCanvas/nodes/`:
- `DataStoreNode.tsx` — Database icon (emerald), model name, operation badge, **dual handles: success (green, 33%) / error (red, 66%)** with "Ok / Err" labels
- `ServiceCallNode.tsx` — ExternalLink icon (orange), URL preview, method badge, **dual handles: success (green, 33%) / error (red, 66%)** with "Ok / Err" labels
- `EventNode.tsx` — Zap icon (purple), event name, direction badge (emit/consume), single output handle
- `LoopNode.tsx` — Repeat icon (teal), large container with dashed border, collection expression, **dual handles: body (teal, 33%) / done (muted, 66%)** with "Body / Done" labels
- `ParallelNode.tsx` — Columns icon (pink), branch count, join strategy badge, **dynamic handles: branch-0, branch-1, ... (pink, evenly spaced) + done (muted, rightmost)** — follows SmartRouterNode pattern with `useMemo` for handle array
- `SubFlowNode.tsx` — GitMerge icon (violet), flow reference (domain/flowId), double-click navigates to referenced flow, ExternalLink indicator when navigatable

**Update `src/components/FlowCanvas/nodes/index.ts`:**
```typescript
import { DataStoreNode } from './DataStoreNode';
import { ServiceCallNode } from './ServiceCallNode';
import { EventNode } from './EventNode';
import { LoopNode } from './LoopNode';
import { ParallelNode } from './ParallelNode';
import { SubFlowNode } from './SubFlowNode';
import { DelayNode } from './DelayNode';
import { CacheNode } from './CacheNode';
import { TransformNode } from './TransformNode';
import { CollectionNode } from './CollectionNode';
import { ParseNode } from './ParseNode';
import { CryptoNode } from './CryptoNode';
import { BatchNode } from './BatchNode';
import { TransactionNode } from './TransactionNode';

export const nodeTypes: NodeTypes = {
  // ... existing 12 nodes ...
  data_store: DataStoreNode,
  service_call: ServiceCallNode,
  event: EventNode,
  loop: LoopNode,
  parallel: ParallelNode,
  sub_flow: SubFlowNode,
  delay: DelayNode,
  cache: CacheNode,
  transform: TransformNode,
  collection: CollectionNode,
  parse: ParseNode,
  crypto: CryptoNode,
  batch: BatchNode,
  transaction: TransactionNode,
};
```

**Update `NodeToolbar.tsx`** — add to traditional flow palette:
```typescript
const extendedTraditionalNodes = [
  { type: 'input', label: 'Input', icon: FormInput },
  { type: 'process', label: 'Process', icon: Cog },
  { type: 'decision', label: 'Decision', icon: GitFork },
  { type: 'data_store', label: 'Data Store', icon: Database },
  { type: 'service_call', label: 'Service Call', icon: ExternalLink },
  { type: 'event', label: 'Event', icon: Zap },
  { type: 'loop', label: 'Loop', icon: Repeat },
  { type: 'parallel', label: 'Parallel', icon: Columns },
  { type: 'sub_flow', label: 'Sub-Flow', icon: GitMerge },
  { type: 'delay', label: 'Delay', icon: Clock },
  { type: 'cache', label: 'Cache', icon: HardDrive },
  { type: 'transform', label: 'Transform', icon: Shuffle },
  { type: 'collection', label: 'Collection', icon: Filter },
  { type: 'parse', label: 'Parse', icon: FileText },
  { type: 'crypto', label: 'Crypto', icon: Lock },
  { type: 'batch', label: 'Batch', icon: Layers },
  { type: 'transaction', label: 'Transaction', icon: ShieldCheck },
  { type: 'terminal', label: 'Terminal', icon: Square },
];
```

**Spec panel editors** — in `src/components/SpecPanel/editors/`:
- `DataStoreSpecEditor.tsx` — operation dropdown (CRUD), model field with autocomplete, data/query JSON textareas, **conditional pagination/sort fields** (shown only when `operation === 'read'`): pagination JSON textarea (placeholder: `{ "style": "cursor", "default_limit": 20, "max_limit": 100 }`), sort JSON textarea (placeholder: `{ "default": "created_at:desc", "allowed": [] }`)
- `ServiceCallSpecEditor.tsx` — method dropdown, URL input, headers/body JSON, timeout, retry config, error_mapping JSON
- `EventSpecEditor.tsx` — direction toggle (emit/consume), event name (autocomplete from domain events), payload mapping, async checkbox
- `LoopSpecEditor.tsx` — collection expression, iterator name, break condition
- `ParallelSpecEditor.tsx` — branch list, join strategy dropdown, timeout
- `SubFlowSpecEditor.tsx` — flow reference picker (browse domains/flows), input/output mapping tables
- `TerminalSpecEditor.tsx` — outcome field, description, **status** (number input, placeholder "e.g. 200, 201, 400"), **response body** (JSON textarea with try/catch parse)

**ExtraFieldsEditor** (`src/components/SpecPanel/editors/ExtraFieldsEditor.tsx`) — handles custom fields for all node types. Uses a `KNOWN_KEYS` map per node type to identify which fields are "standard" (rendered by the typed editor) vs. "custom" (rendered in a collapsible section):

```typescript
const KNOWN_KEYS: Record<string, Set<string>> = {
  trigger: new Set(['event', 'source', 'filter', 'description']),
  input: new Set(['fields', 'validation', 'description']),
  process: new Set(['action', 'service', 'category', 'inputs', 'outputs', 'description']),
  decision: new Set(['condition', 'trueLabel', 'falseLabel', 'description']),
  terminal: new Set(['outcome', 'description', 'status', 'body', 'response_type', 'headers']),
  data_store: new Set(['operation', 'model', 'data', 'query', 'description', 'pagination', 'sort', 'batch', 'upsert_key', 'include', 'returning']),
  service_call: new Set(['method', 'url', 'headers', 'body', 'timeout_ms', 'retry', 'error_mapping', 'request_config', 'integration', 'description']),
  event: new Set(['direction', 'event_name', 'payload', 'payload_source', 'async', 'target_queue', 'priority', 'delay_ms', 'dedup_key', 'description']),
  loop: new Set(['collection', 'iterator', 'break_condition', 'on_error', 'accumulate', 'description']),
  parallel: new Set(['branches', 'join', 'join_count', 'timeout_ms', 'description']),
  sub_flow: new Set(['flow_ref', 'input_mapping', 'output_mapping', 'description']),
  llm_call: new Set(['model', 'system_prompt', 'prompt_template', 'temperature', 'max_tokens', 'structured_output', 'context_sources', 'retry', 'description']),
  delay: new Set(['min_ms', 'max_ms', 'strategy', 'description']),
  cache: new Set(['key', 'ttl_ms', 'store', 'description']),
  transform: new Set(['input_schema', 'output_schema', 'field_mappings', 'description']),
  collection: new Set(['operation', 'input', 'predicate', 'key', 'direction', 'accumulator', 'output', 'description']),
  parse: new Set(['format', 'input', 'strategy', 'library', 'output', 'description']),
  crypto: new Set(['operation', 'algorithm', 'key_source', 'input_fields', 'output_field', 'encoding', 'description']),
  batch: new Set(['input', 'operation_template', 'concurrency', 'on_error', 'output', 'description']),
  transaction: new Set(['isolation', 'steps', 'rollback_on_error', 'description']),
  // ... agent and orchestration types similarly
};
```

### Other Enhancements

| Feature | Description |
|---------|-------------|
| **Validation presets** | Reusable input validation patterns (email, phone, password, URL, UUID) — dropdown in InputNode spec panel |
| **Mermaid generator** | Generate Mermaid flowchart diagrams from flow specs. Output: `generated/mermaid/{domain}/{flow}.md` with embedded Mermaid code blocks. Implementation: `src/utils/generators/mermaid.ts` |
| **Minimap toggle** | React Flow minimap on the flow canvas (L3) — toggle with `Cmd+Shift+M`. State managed by `src/stores/ui-store.ts` (Zustand store with `showMinimap` boolean and `toggleMinimap` action) |

### Flow Templates

Pre-built flow templates for common patterns. Available when creating a new flow via the Add Flow dialog.

**Implementation:** `src/utils/flow-templates.ts`

```typescript
export interface FlowTemplate {
  id: string;
  name: string;
  description: string;
  type: 'traditional' | 'agent';
  nodeCount: number;
  create: (flowId: string, flowName: string, domainId: string) => FlowDocument;
}

export const FLOW_TEMPLATES: FlowTemplate[] = [
  // Traditional templates
  { id: 'rest-api', name: 'REST API Endpoint', nodeCount: 5, ... },
  { id: 'crud-entity', name: 'CRUD Entity', nodeCount: 6, ... },
  { id: 'webhook-handler', name: 'Webhook Handler', nodeCount: 5, ... },
  { id: 'event-processor', name: 'Event Processor', nodeCount: 5, ... },
  // Agent templates
  { id: 'rag-agent', name: 'RAG Agent', nodeCount: 5, ... },
  { id: 'support-agent', name: 'Customer Support Agent', nodeCount: 5, ... },
  { id: 'code-review-agent', name: 'Code Review Agent', nodeCount: 3, ... },
  { id: 'data-pipeline-agent', name: 'Data Pipeline Agent', nodeCount: 3, ... },
];
```

Each template's `create()` function returns a complete `FlowDocument` with trigger, nodes, connections (including proper `sourceHandle` values), and spec defaults — using `nanoid(8)` for node IDs.

### UI Store

**File:** `src/stores/ui-store.ts`

Simple Zustand store for UI state:

```typescript
interface UiStore {
  showMinimap: boolean;
  toggleMinimap: () => void;
}
```

Toggled with `Cmd+Shift+M` keyboard shortcut.

### Future Ideas

| Feature | Description |
|---------|-------------|
| **Expert agents** | Pre-built agent archetypes (researcher, coder, reviewer) as templates |
| **Templates/library** | Reusable flow templates (auth, CRUD, webhook, scheduled job) — "Import template" in Add Flow dialog |
| **Router node** | Multi-output classification visual for agent routing (extends SmartRouter with visual branching) |
| **Memory node** | Vector store / conversation config visual for agent memory management |
| **Circuit breaker indicators** | Visual status on routing arrows (closed/open/half-open) |

### Session 17 Success Criteria

- [ ] All 6 new node types render on canvas with correct icons and badges (Session 17)
- [ ] All 8 additional node types render correctly (Session 18: delay, cache, transform, collection, parse, crypto, batch, transaction)
- [ ] All 14 new node types have spec panel editors with appropriate fields
- [ ] DataStoreNode model selector shows schemas from project
- [ ] ServiceCallNode supports timeout and retry configuration
- [ ] EventNode autocompletes event names from domain.yaml
- [ ] LoopNode visually contains its body nodes
- [ ] ParallelNode shows branch count and join strategy
- [ ] SubFlowNode links to referenced flow (clickable navigation)
- [ ] All 6 nodes serialize to/from YAML correctly
- [ ] NodeToolbar shows extended palette for traditional flows
- [ ] Validation rules added for new node types (e.g. data_store must have model)

---

## Session 18: Extended Node Types + Field Extensions

Session 18 adds 8 more traditional node types (completing the full set of 19 traditional + 4 agent + 4 orchestration = 27 total) and extends 7 existing node spec interfaces with additional fields from the DDD Usage Guide.

### New Node Types (8)

| Node | Icon | Color | Purpose | Output Handles |
|------|------|-------|---------|----------------|
| `delay` | Clock | blue-600 | Wait/throttle with fixed or random strategy | Single output |
| `cache` | HardDrive | amber-500 | Cache lookup with hit/miss branching | `hit` / `miss` |
| `transform` | Shuffle | indigo-500 | Field mapping between schemas | Single output |
| `collection` | Filter | cyan-500 | Filter, sort, deduplicate, merge, group_by, aggregate, reduce, flatten | `result` / `empty` |
| `parse` | FileText | lime-500 | Structured extraction (RSS, Atom, HTML, XML, JSON, CSV, Markdown) | `success` / `error` |
| `crypto` | Lock | fuchsia-500 | Encrypt, decrypt, hash, sign, verify, generate key | `success` / `error` |
| `batch` | Layers | rose-500 | Execute operation template against collection with concurrency control | `done` / `error` |
| `transaction` | ShieldCheck | amber-600 | Atomic multi-step database operation with rollback on error | `committed` / `rolled_back` |

### Spec Interfaces (new)

```typescript
export interface DelaySpec {
  min_ms?: number; max_ms?: number;
  strategy?: 'random' | 'fixed';
  description?: string;
  [key: string]: unknown;
}

export interface CacheSpec {
  key?: string; ttl_ms?: number;
  store?: 'redis' | 'memory';
  description?: string;
  [key: string]: unknown;
}

export interface TransformSpec {
  input_schema?: string; output_schema?: string;
  field_mappings?: Record<string, string>;
  description?: string;
  [key: string]: unknown;
}

export interface CollectionSpec {
  operation?: 'filter' | 'sort' | 'deduplicate' | 'merge' | 'group_by' | 'aggregate' | 'reduce' | 'flatten';
  input?: string; predicate?: string; key?: string;
  direction?: 'asc' | 'desc'; accumulator?: string; output?: string;
  description?: string;
  [key: string]: unknown;
}

export interface ParseSpec {
  format?: 'rss' | 'atom' | 'html' | 'xml' | 'json' | 'csv' | 'markdown';
  input?: string; strategy?: 'strict' | 'lenient' | 'streaming';
  library?: string; output?: string;
  description?: string;
  [key: string]: unknown;
}

export interface CryptoSpec {
  operation?: 'encrypt' | 'decrypt' | 'hash' | 'sign' | 'verify' | 'generate_key';
  algorithm?: string; key_source?: { env?: string; vault?: string };
  input_fields?: string[]; output_field?: string;
  encoding?: 'base64' | 'hex';
  description?: string;
  [key: string]: unknown;
}

export interface BatchSpec {
  input?: string;
  operation_template?: { type?: string; dispatch_field?: string; configs?: Record<string, unknown> };
  concurrency?: number; on_error?: 'continue' | 'stop';
  output?: string;
  description?: string;
  [key: string]: unknown;
}

export interface TransactionSpec {
  isolation?: 'read_committed' | 'repeatable_read' | 'serializable';
  steps?: Array<{ action: string; rollback?: string }>;
  rollback_on_error?: boolean;
  description?: string;
  [key: string]: unknown;
}
```

### Field Extensions (9 existing specs)

| Spec | New Fields |
|------|-----------|
| `TriggerSpec` | `filter?: Record<string, unknown>` |
| `ProcessSpec` | `category?: 'security' \| 'transform' \| 'integration' \| 'business_logic' \| 'infrastructure'`, `inputs?: string[]`, `outputs?: string[]` |
| `DataStoreSpec` | Extended `operation` union with `'upsert' \| 'create_many' \| 'update_many' \| 'delete_many'`, `include?: Record<string, unknown>`, `returning?: boolean` |
| `ServiceCallSpec` | `request_config` updated to rich enums (`user_agent`, `delay`, `cookie_jar`, `proxy`, `tls_fingerprint`, `fallback`), `integration?: string` |
| `EventNodeSpec` | `payload_source?: string`, `target_queue?: string`, `priority?: number`, `delay_ms?: number`, `dedup_key?: string` |
| `LoopSpec` | `accumulate?: { field?: string; strategy?: 'append' \| 'merge' \| 'sum' \| 'last'; output?: string }` |
| `ParallelSpec` | `branches` broadened to `(string \| { label: string; condition?: string })[]` |
| `LlmCallSpec` | `context_sources?: Record<string, { from: string; transform?: string }>` |
| `TerminalSpec` | `response_type` and `headers` added to KNOWN_KEYS |

### Validation Rules (new)

Added to `checkExtendedNodes()` in `src/utils/flow-validator.ts`:
- `collection`: error if no operation, error if no input
- `parse`: error if no format, error if no input
- `crypto`: error if no operation, error if no algorithm, error if no key_source
- `batch`: error if no input, error if no operation_template
- `transaction`: error if steps < 2
- `cache`: error if no key, error if no store
- `transform`: error if no input_schema, error if no output_schema
- `delay`: error if no min_ms

### Updated Registrations

- `nodes/index.ts`: 27 entries in `nodeTypes` object
- `editors/index.ts`: 27 entries in `specEditors` object
- `ExtraFieldsEditor.tsx`: 27 entries in `KNOWN_KEYS` map (existing entries extended with new field names)
- `NodeToolbar.tsx`: 8 new items in `TRADITIONAL_ITEMS` palette
- `flow-store.ts`: `defaultSpec()` and `defaultLabel()` handle all 27 types

---

## Reference

Full specification: `ddd-specification-complete.md`
