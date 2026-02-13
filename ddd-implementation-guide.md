# DDD Implementation Guide for Claude Code

## Overview

You are building **DDD (Diagram-Driven Development)** — a desktop app for visual flow editing that outputs YAML specs.

**Read the full specification:** `ddd-specification-complete.md` (in same directory)

---

## Phase 1: MVP Scope (Build This First)

### What MVP Includes
- Desktop app (Tauri + React)
- **Multi-level canvas** (System Map → Domain Map → Flow Sheet)
- **Breadcrumb navigation** between sheet levels
- **Auto-generated** System Map (L1) and Domain Map (L2) from specs
- **Portal nodes** for cross-domain navigation
- Canvas with 5 node types (trigger, input, process, decision, terminal)
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
│       │   ├── git.rs         # Git operations
│       │   ├── project.rs     # Project management
│       │   ├── llm.rs         # LLM API proxy (Anthropic/OpenAI/Ollama)
│       │   ├── pty.rs         # PTY terminal for Claude Code
│       │   └── test_runner.rs # Run tests and parse output
│       └── services/
│           ├── mod.rs
│           ├── git_service.rs
│           ├── file_service.rs
│           ├── llm_service.rs  # HTTP client, streaming, provider abstraction
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
│   │   │   ├── SystemMap.tsx        # Level 1: domain blocks + event arrows
│   │   │   ├── DomainBlock.tsx      # Clickable domain block with flow count
│   │   │   └── EventArrow.tsx       # Arrow between domains (shared with DomainMap)
│   │   ├── DomainMap/
│   │   │   ├── DomainMap.tsx        # Level 2: flow blocks + portals
│   │   │   ├── FlowBlock.tsx        # Clickable flow block
│   │   │   └── PortalNode.tsx       # Cross-domain navigation node
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
│   │   ├── LLMAssistant/
│   │   │   ├── ChatPanel.tsx          # Toggleable chat sidebar
│   │   │   ├── ChatMessage.tsx        # Single message (user/assistant/system)
│   │   │   ├── ChatInput.tsx          # Input box with submit + context indicator
│   │   │   ├── GhostPreview.tsx       # Ghost nodes overlay + Apply/Discard bar
│   │   │   ├── InlineAssist.tsx       # Context menu with ✨ actions
│   │   │   ├── ModelPicker.tsx        # Model dropdown selector in chat header
│   │   │   ├── UsageBar.tsx           # Bottom status bar (model + cost tracking)
│   │   │   └── FlowPreview.tsx        # Mini flow diagram in chat for generated flows
│   │   ├── ImplementationPanel/
│   │   │   ├── ImplementationPanel.tsx # Main panel (prompt preview + terminal + test results)
│   │   │   ├── PromptPreview.tsx       # Editable prompt display with Run button
│   │   │   ├── TerminalEmbed.tsx       # Embedded PTY terminal for Claude Code
│   │   │   ├── TestResults.tsx         # Test output display with fix loop
│   │   │   ├── ImplementationQueue.tsx # Queue view for batch implementation
│   │   │   ├── StaleBanner.tsx         # Stale detection warning banner
│   │   │   ├── ReconciliationReport.tsx # Code→spec drift report with accept/remove/ignore
│   │   │   └── SyncBadge.tsx           # Sync score badge for flow blocks
│   │   ├── MemoryPanel/
│   │   │   ├── MemoryPanel.tsx        # Main memory panel (sidebar toggle)
│   │   │   ├── SummaryCard.tsx        # Project summary display
│   │   │   ├── StatusList.tsx         # Implementation status per flow
│   │   │   ├── DecisionList.tsx       # Design decisions with add/edit
│   │   │   ├── DecisionForm.tsx       # Add/edit decision dialog
│   │   │   └── FlowDependencies.tsx   # Flow map for current flow
│   │   └── shared/
│   │       ├── Button.tsx
│   │       ├── Input.tsx
│   │       ├── Select.tsx
│   │       └── Modal.tsx
│   ├── stores/
│   │   ├── sheet-store.ts     # Active sheet, navigation history, breadcrumbs
│   │   ├── flow-store.ts      # Current flow state (Level 3)
│   │   ├── project-store.ts   # Project/file state, domain configs
│   │   ├── ui-store.ts        # UI state (selection, panels)
│   │   ├── git-store.ts       # Git state
│   │   ├── llm-store.ts       # Chat state, ghost nodes, LLM config
│   │   ├── memory-store.ts    # Project memory layers, refresh triggers
│   │   └── implementation-store.ts  # Implementation panel state, queue, test results
│   ├── types/
│   │   ├── sheet.ts           # Sheet levels, navigation, breadcrumb types
│   │   ├── domain.ts          # Domain config, event wiring, portal types
│   │   ├── flow.ts
│   │   ├── node.ts
│   │   ├── spec.ts
│   │   ├── project.ts
│   │   ├── llm.ts             # Chat messages, LLM config, ghost node types
│   │   ├── memory.ts          # Project memory layer types
│   │   └── implementation.ts  # Implementation panel, prompt builder, test runner types
│   ├── utils/
│   │   ├── yaml.ts
│   │   ├── domain-parser.ts   # Parse domain.yaml → SystemMap/DomainMap data
│   │   ├── llm-context.ts     # Build context object for LLM requests (uses memory layers)
│   │   ├── memory-builder.ts  # Scan specs → generate memory layers
│   │   ├── prompt-builder.ts  # Build Claude Code prompts from specs
│   │   ├── claude-md-generator.ts  # Generate/update CLAUDE.md from project state
│   │   ├── openapi-generator.ts    # Generate OpenAPI 3.0 from flow specs
│   │   ├── cicd-generator.ts       # Generate CI/CD pipeline from architecture.yaml
│   │   ├── dockerfile-generator.ts # Generate Dockerfile + compose from deployment config
│   │   ├── migration-tracker.ts    # Track schema hashes, detect changes, build migration prompts
│   │   ├── mermaid.ts
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
```

**File: `src/types/node.ts`**
```typescript
export type NodeType = 'trigger' | 'input' | 'process' | 'decision' | 'terminal';

export interface Position {
  x: number;
  y: number;
}

export interface NodeConnection {
  targetNodeId: string;
  label?: string;  // For decision nodes: "valid", "invalid"
}

export interface BaseNode {
  id: string;
  type: NodeType;
  position: Position;
  connections: Record<string, string>;  // outputName -> targetNodeId
}

export interface TriggerNode extends BaseNode {
  type: 'trigger';
  spec: {
    triggerType: 'http' | 'webhook' | 'cron' | 'event';
    method?: 'GET' | 'POST' | 'PUT' | 'DELETE';
    path?: string;
    schedule?: string;
    event?: string;
  };
}

export interface InputNode extends BaseNode {
  type: 'input';
  spec: {
    fields: InputField[];
  };
}

export interface InputField {
  name: string;
  type: 'string' | 'number' | 'boolean' | 'array' | 'object';
  required: boolean;
  validations: Validation[];
}

export interface Validation {
  type: 'min_length' | 'max_length' | 'pattern' | 'format' | 'min' | 'max' | 'enum';
  value: any;
  error: string;
}

export interface ProcessNode extends BaseNode {
  type: 'process';
  spec: {
    operation: string;
    description: string;
    inputs: Record<string, string>;  // name -> jsonpath
    outputs: Record<string, string>;
  };
}

export interface DecisionNode extends BaseNode {
  type: 'decision';
  spec: {
    condition: string;
    description: string;
    onTrue: { status?: number; body?: any };
    onFalse: { status?: number; body?: any };
  };
}

export interface TerminalNode extends BaseNode {
  type: 'terminal';
  spec: {
    status: number;
    body: Record<string, any>;
  };
}

export type FlowNode = TriggerNode | InputNode | ProcessNode | DecisionNode | TerminalNode;

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

**File: `src/types/flow.ts`**
```typescript
import { FlowNode, AgentNode, OrchestrationNode, AnyNode, ToolDefinition } from './node';

export type FlowType = 'traditional' | 'agent';

export interface Flow {
  id: string;
  name: string;
  type: FlowType;
  description?: string;
  domain: string;
  trigger: FlowNode;  // First node, must be trigger type
  nodes: AnyNode[];   // Traditional or agent nodes
  metadata: {
    createdAt: string;
    updatedAt: string;
    createdBy?: string;
    completeness: number;  // 0-100
  };
}

// Agent-specific flow config (only present when type === 'agent')
export interface AgentFlowConfig {
  model: string;
  system_prompt: string;
  max_iterations: number;
  temperature?: number;
  memory: Array<{
    name: string;
    type: 'conversation_history' | 'vector_store' | 'key_value';
    max_tokens?: number;
    strategy?: string;
    [key: string]: any;
  }>;
  tools: ToolDefinition[];
  guardrails: {
    input: Array<Record<string, any>>;
    output: Array<Record<string, any>>;
  };
}

export interface AgentFlow extends Flow {
  type: 'agent';
  agent_config: AgentFlowConfig;
}

export function isAgentFlow(flow: Flow): flow is AgentFlow {
  return flow.type === 'agent';
}

export interface FlowFile {
  flow: {
    id: string;
    name: string;
    type?: FlowType;
    description?: string;
    domain: string;
  };
  agent_config?: any;  // Agent-specific config
  trigger: any;        // Trigger spec
  nodes: any[];        // Node specs
  tools?: any[];       // Tool definitions (agent flows only)
  memory?: any[];      // Memory config (agent flows only)
  guardrails?: any;    // Guardrail config (agent flows only)
  metadata: any;
}
```

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
```

#### Day 15-17: LLM Design Assistant

**File: `src/types/llm.ts`**
```typescript
// --- Multi-Model Architecture ---

export type LLMProvider = 'anthropic' | 'openai' | 'ollama' | 'openai_compatible';
export type ModelTier = 'fast' | 'standard' | 'powerful';

export interface ModelRegistryEntry {
  id: string;                    // Registry key (e.g., 'claude-sonnet')
  provider: LLMProvider;
  modelId: string;               // Provider model ID (e.g., 'claude-sonnet-4-5-20250929')
  apiKeyEnv?: string;            // Env var for API key (not needed for Ollama)
  baseUrl?: string;              // For ollama / openai_compatible endpoints
  label: string;                 // Display name in UI
  tier: ModelTier;
  maxTokens: number;
  temperature: number;
  costPer1kInput: number;        // USD per 1K input tokens (0 for local)
  costPer1kOutput: number;       // USD per 1K output tokens
  available?: boolean;           // Runtime: is this model reachable?
}

export type TaskType =
  // Inline assist
  | 'suggest_spec' | 'complete_spec' | 'explain_node' | 'add_error_handling'
  | 'generate_test_cases' | 'label_connection' | 'add_node_between'
  // Flow/domain/system generation
  | 'generate_flow' | 'generate_domain' | 'generate_from_description'
  | 'import_from_description'
  // Review
  | 'review_flow' | 'review_architecture' | 'suggest_wiring'
  | 'suggest_flows' | 'suggest_events' | 'suggest_domains'
  // Memory
  | 'generate_summary'
  // Chat
  | 'chat';

export interface TaskRouting {
  [task: string]: string;        // TaskType → model registry ID ('_active' = user's choice)
}

export interface CostTracking {
  enabled: boolean;
  resetPeriod: 'daily' | 'weekly' | 'monthly';
  budgetWarning: number;         // USD threshold for warning
  budgetLimit: number;           // USD threshold for blocking (0 = unlimited)
}

export interface LLMConfig {
  activeModel: string;           // Current model registry ID for chat
  models: Record<string, ModelRegistryEntry>;
  taskRouting: TaskRouting;
  fallbackChain: string[];       // Ordered list of model IDs to try
  costTracking: CostTracking;
}

// --- Cost / Usage Tracking ---

export interface UsagePeriod {
  start: string;                 // ISO date
  end: string;
  totalCost: number;
  totalInputTokens: number;
  totalOutputTokens: number;
  requests: number;
  byModel: Record<string, ModelUsage>;
  byTask: Record<string, TaskUsage>;
}

export interface ModelUsage {
  requests: number;
  inputTokens: number;
  outputTokens: number;
  cost: number;
}

export interface TaskUsage {
  requests: number;
  cost: number;
}

export interface UsageData {
  currentPeriod: UsagePeriod;
  history: UsagePeriod[];
}

// --- Chat Messages ---

export type ChatRole = 'user' | 'assistant' | 'system';

export interface ChatMessage {
  id: string;
  role: ChatRole;
  content: string;
  timestamp: number;
  modelId?: string;              // Which model generated this (for assistant messages)
  inputTokens?: number;          // Token usage for this message
  outputTokens?: number;
  // For assistant messages that generate nodes/flows
  generatedYaml?: string;
  previewState?: 'pending' | 'applied' | 'discarded';
}

export interface ChatThread {
  id: string;
  flowId?: string;             // Scoped to a flow (null = project-level)
  domainId?: string;
  messages: ChatMessage[];
  createdAt: number;
  updatedAt: number;
}

// --- Ghost Nodes (Preview before Apply) ---

export interface GhostPreview {
  id: string;
  sourceMessageId: string;     // Which chat message generated this
  yaml: string;                // The generated YAML
  nodes: GhostNode[];
  connections: GhostConnection[];
  state: 'previewing' | 'applied' | 'discarded';
}

export interface GhostNode {
  id: string;
  type: string;
  label: string;
  position: { x: number; y: number };
  spec: Record<string, any>;
}

export interface GhostConnection {
  from: string;
  to: string;
  label?: string;
}

// --- Inline Assist ---

export type InlineAssistAction =
  // Node-level
  | 'suggest_spec'
  | 'complete_spec'
  | 'explain_node'
  | 'add_error_handling'
  | 'generate_test_cases'
  // Connection-level
  | 'add_node_between'
  | 'label_connection'
  // Canvas-level
  | 'generate_flow'
  | 'review_flow'
  | 'suggest_wiring'
  | 'import_from_description'
  // Domain-level (Level 2)
  | 'suggest_flows'
  | 'suggest_events'
  | 'generate_domain'
  // System-level (Level 1)
  | 'suggest_domains'
  | 'review_architecture'
  | 'generate_from_description';

export interface InlineAssistRequest {
  action: InlineAssistAction;
  target: InlineAssistTarget;
}

export type InlineAssistTarget =
  | { type: 'node'; nodeId: string }
  | { type: 'connection'; fromId: string; toId: string }
  | { type: 'canvas'; flowId: string }
  | { type: 'domain'; domainId: string }
  | { type: 'system' };

// --- LLM Context (sent with every request) ---

export interface LLMContext {
  system: {
    name: string;
    techStack: Record<string, string>;
    domains: string[];
  };
  currentDomain?: {
    name: string;
    flows: string[];
    publishesEvents: string[];
    consumesEvents: string[];
  };
  currentFlow?: {
    id: string;
    type: 'traditional' | 'agent';
    yaml: string;
  };
  selectedNodes?: Array<{
    id: string;
    type: string;
    spec: Record<string, any>;
  }>;
  errorCodes?: string[];
  schemas?: Record<string, any>;
}
```

**File: `src/stores/llm-store.ts`**
```typescript
import { create } from 'zustand';
import type { ChatThread, ChatMessage, GhostPreview, LLMConfig, InlineAssistRequest } from '../types/llm';
import { buildLLMContext } from '../utils/llm-context';
import { invoke } from '@tauri-apps/api/core';
import { nanoid } from 'nanoid';

interface LLMState {
  // Config
  config: LLMConfig;

  // Usage tracking
  usage: UsageData | null;
  sessionTokens: number;
  sessionCost: number;

  // Chat panel
  chatOpen: boolean;
  activeThread: ChatThread | null;
  threads: ChatThread[];
  isStreaming: boolean;

  // Ghost preview
  ghostPreview: GhostPreview | null;

  // Actions — Model Management
  setActiveModel: (modelId: string) => void;
  resolveModelForTask: (taskType: TaskType) => ModelRegistryEntry;
  addModel: (model: ModelRegistryEntry) => void;
  removeModel: (modelId: string) => void;
  updateModel: (modelId: string, updates: Partial<ModelRegistryEntry>) => void;
  setTaskRouting: (task: TaskType, modelId: string) => void;
  checkModelAvailability: () => Promise<void>;

  // Actions — Chat
  toggleChat: () => void;
  openChat: () => void;
  closeChat: () => void;
  sendMessage: (content: string, modelOverride?: string) => Promise<void>;
  switchThread: (threadId: string) => void;
  startThreadForFlow: (flowId: string, domainId: string) => void;

  // Actions — Ghost Preview
  applyGhostPreview: () => void;
  discardGhostPreview: () => void;
  editGhostInChat: () => void;

  // Actions — Inline Assist
  executeInlineAssist: (request: InlineAssistRequest, modelOverride?: string) => Promise<string>;

  // Actions — Usage
  trackUsage: (modelId: string, taskType: TaskType, inputTokens: number, outputTokens: number) => void;
  loadUsage: (projectPath: string) => Promise<void>;
  isOverBudget: () => boolean;
  isNearBudget: () => boolean;

  // Actions — Config
  updateConfig: (config: Partial<LLMConfig>) => void;
}

const DEFAULT_MODELS: Record<string, ModelRegistryEntry> = {
  'claude-sonnet': {
    id: 'claude-sonnet',
    provider: 'anthropic',
    modelId: 'claude-sonnet-4-5-20250929',
    apiKeyEnv: 'ANTHROPIC_API_KEY',
    label: 'Claude Sonnet',
    tier: 'standard',
    maxTokens: 4096,
    temperature: 0.3,
    costPer1kInput: 0.003,
    costPer1kOutput: 0.015,
  },
  'claude-haiku': {
    id: 'claude-haiku',
    provider: 'anthropic',
    modelId: 'claude-haiku-4-5-20251001',
    apiKeyEnv: 'ANTHROPIC_API_KEY',
    label: 'Claude Haiku',
    tier: 'fast',
    maxTokens: 2048,
    temperature: 0.2,
    costPer1kInput: 0.0008,
    costPer1kOutput: 0.004,
  },
  'claude-opus': {
    id: 'claude-opus',
    provider: 'anthropic',
    modelId: 'claude-opus-4-6',
    apiKeyEnv: 'ANTHROPIC_API_KEY',
    label: 'Claude Opus',
    tier: 'powerful',
    maxTokens: 4096,
    temperature: 0.3,
    costPer1kInput: 0.015,
    costPer1kOutput: 0.075,
  },
};

const DEFAULT_TASK_ROUTING: TaskRouting = {
  // Fast model for inline assists
  suggest_spec: 'claude-haiku',
  complete_spec: 'claude-haiku',
  explain_node: 'claude-haiku',
  label_connection: 'claude-haiku',
  add_error_handling: 'claude-haiku',
  add_node_between: 'claude-haiku',
  // Standard model for generation
  generate_flow: 'claude-sonnet',
  generate_domain: 'claude-sonnet',
  generate_from_description: 'claude-sonnet',
  import_from_description: 'claude-sonnet',
  generate_test_cases: 'claude-sonnet',
  suggest_wiring: 'claude-sonnet',
  suggest_flows: 'claude-sonnet',
  suggest_events: 'claude-sonnet',
  suggest_domains: 'claude-sonnet',
  // Powerful model for review / summary
  review_flow: 'claude-sonnet',
  review_architecture: 'claude-opus',
  generate_summary: 'claude-opus',
  // Chat uses active model
  chat: '_active',
};

export const useLLMStore = create<LLMState>((set, get) => ({
  config: {
    activeModel: 'claude-sonnet',
    models: DEFAULT_MODELS,
    taskRouting: DEFAULT_TASK_ROUTING,
    fallbackChain: ['claude-sonnet', 'claude-haiku'],
    costTracking: {
      enabled: true,
      resetPeriod: 'monthly',
      budgetWarning: 10.0,
      budgetLimit: 50.0,
    },
  },

  usage: null,
  sessionTokens: 0,
  sessionCost: 0,
  chatOpen: false,
  activeThread: null,
  threads: [],
  isStreaming: false,
  ghostPreview: null,

  // --- Model Management ---

  setActiveModel: (modelId: string) => {
    set(s => ({ config: { ...s.config, activeModel: modelId } }));
  },

  resolveModelForTask: (taskType: TaskType): ModelRegistryEntry => {
    const { config } = get();
    const routedId = config.taskRouting[taskType] ?? config.activeModel;
    const modelId = routedId === '_active' ? config.activeModel : routedId;
    const model = config.models[modelId];
    if (!model) {
      // Fallback to first available model
      return Object.values(config.models)[0];
    }
    return model;
  },

  addModel: (model: ModelRegistryEntry) => {
    set(s => ({
      config: {
        ...s.config,
        models: { ...s.config.models, [model.id]: model },
      },
    }));
  },

  removeModel: (modelId: string) => {
    set(s => {
      const { [modelId]: _, ...rest } = s.config.models;
      return { config: { ...s.config, models: rest } };
    });
  },

  updateModel: (modelId: string, updates: Partial<ModelRegistryEntry>) => {
    set(s => ({
      config: {
        ...s.config,
        models: {
          ...s.config.models,
          [modelId]: { ...s.config.models[modelId], ...updates },
        },
      },
    }));
  },

  setTaskRouting: (task: TaskType, modelId: string) => {
    set(s => ({
      config: {
        ...s.config,
        taskRouting: { ...s.config.taskRouting, [task]: modelId },
      },
    }));
  },

  checkModelAvailability: async () => {
    const { config } = get();
    const results = await Promise.allSettled(
      Object.values(config.models).map(async (model) => {
        try {
          await invoke('llm_health_check', { model });
          return { id: model.id, available: true };
        } catch {
          return { id: model.id, available: false };
        }
      })
    );

    const updates: Record<string, ModelRegistryEntry> = { ...config.models };
    for (const result of results) {
      if (result.status === 'fulfilled') {
        const { id, available } = result.value;
        updates[id] = { ...updates[id], available };
      }
    }
    set(s => ({ config: { ...s.config, models: updates } }));
  },

  // --- Chat ---

  toggleChat: () => set(s => ({ chatOpen: !s.chatOpen })),
  openChat: () => set({ chatOpen: true }),
  closeChat: () => set({ chatOpen: false }),

  sendMessage: async (content: string, modelOverride?: string) => {
    const state = get();
    const thread = state.activeThread;
    if (!thread) return;

    // Budget check
    if (state.isOverBudget()) {
      // Add system message about budget
      return;
    }

    // Resolve which model to use
    const model = modelOverride
      ? state.config.models[modelOverride]
      : state.resolveModelForTask('chat');

    const userMsg: ChatMessage = {
      id: nanoid(),
      role: 'user',
      content,
      timestamp: Date.now(),
    };

    const updatedThread = {
      ...thread,
      messages: [...thread.messages, userMsg],
      updatedAt: Date.now(),
    };

    set({ activeThread: updatedThread, isStreaming: true });

    const context = buildLLMContext();

    // Try model with fallback chain
    let response: { content: string; inputTokens: number; outputTokens: number } | null = null;
    let usedModelId = model.id;

    const modelsToTry = [model.id, ...state.config.fallbackChain.filter(id => id !== model.id)];

    for (const modelId of modelsToTry) {
      const tryModel = state.config.models[modelId];
      if (!tryModel) continue;

      try {
        response = await invoke('llm_chat', {
          messages: updatedThread.messages.map(m => ({ role: m.role, content: m.content })),
          context,
          model: tryModel,
        });
        usedModelId = modelId;
        break;
      } catch (err) {
        console.warn(`Model ${modelId} failed, trying next...`, err);
      }
    }

    if (!response) {
      set({ isStreaming: false });
      return;
    }

    // Track usage
    state.trackUsage(usedModelId, 'chat', response.inputTokens, response.outputTokens);

    const assistantMsg: ChatMessage = {
      id: nanoid(),
      role: 'assistant',
      content: response.content,
      timestamp: Date.now(),
      modelId: usedModelId,
      inputTokens: response.inputTokens,
      outputTokens: response.outputTokens,
    };

    const yamlMatch = response.content.match(/```yaml\n([\s\S]*?)```/);
    if (yamlMatch) {
      assistantMsg.generatedYaml = yamlMatch[1];
      assistantMsg.previewState = 'pending';
    }

    const finalThread = {
      ...updatedThread,
      messages: [...updatedThread.messages, assistantMsg],
      updatedAt: Date.now(),
    };

    set({
      activeThread: finalThread,
      isStreaming: false,
      threads: state.threads.map(t => t.id === finalThread.id ? finalThread : t),
    });

    if (assistantMsg.generatedYaml) {
      set({ ghostPreview: parseYamlToGhostPreview(assistantMsg.id, assistantMsg.generatedYaml) });
    }
  },

  switchThread: (threadId: string) => {
    const thread = get().threads.find(t => t.id === threadId);
    if (thread) set({ activeThread: thread });
  },

  startThreadForFlow: (flowId: string, domainId: string) => {
    const existing = get().threads.find(t => t.flowId === flowId);
    if (existing) {
      set({ activeThread: existing, chatOpen: true });
      return;
    }

    const thread: ChatThread = {
      id: nanoid(),
      flowId,
      domainId,
      messages: [],
      createdAt: Date.now(),
      updatedAt: Date.now(),
    };

    set(s => ({
      threads: [...s.threads, thread],
      activeThread: thread,
      chatOpen: true,
    }));
  },

  // --- Ghost Preview ---

  applyGhostPreview: () => {
    const preview = get().ghostPreview;
    if (!preview) return;

    set({ ghostPreview: { ...preview, state: 'applied' } });

    const thread = get().activeThread;
    if (thread) {
      const updatedMessages = thread.messages.map(m =>
        m.id === preview.sourceMessageId ? { ...m, previewState: 'applied' as const } : m
      );
      set({ activeThread: { ...thread, messages: updatedMessages }, ghostPreview: null });
    }
  },

  discardGhostPreview: () => {
    const preview = get().ghostPreview;
    if (!preview) return;

    const thread = get().activeThread;
    if (thread) {
      const updatedMessages = thread.messages.map(m =>
        m.id === preview.sourceMessageId ? { ...m, previewState: 'discarded' as const } : m
      );
      set({ activeThread: { ...thread, messages: updatedMessages }, ghostPreview: null });
    }
  },

  editGhostInChat: () => {
    const preview = get().ghostPreview;
    if (!preview) return;
    set({ ghostPreview: null, chatOpen: true });
  },

  // --- Inline Assist ---

  executeInlineAssist: async (request: InlineAssistRequest, modelOverride?: string): Promise<string> => {
    const state = get();

    // Resolve model: override → task routing → fallback
    const model = modelOverride
      ? state.config.models[modelOverride]
      : state.resolveModelForTask(request.action as TaskType);

    const context = buildLLMContext();

    const response: { content: string; inputTokens: number; outputTokens: number } =
      await invoke('llm_inline_assist', {
        action: request.action,
        target: request.target,
        context,
        model,
      });

    // Track usage
    state.trackUsage(model.id, request.action as TaskType, response.inputTokens, response.outputTokens);

    return response.content;
  },

  // --- Usage Tracking ---

  trackUsage: (modelId: string, taskType: TaskType, inputTokens: number, outputTokens: number) => {
    const model = get().config.models[modelId];
    if (!model) return;

    const cost = (inputTokens / 1000) * model.costPer1kInput +
                 (outputTokens / 1000) * model.costPer1kOutput;

    set(s => ({
      sessionTokens: s.sessionTokens + inputTokens + outputTokens,
      sessionCost: s.sessionCost + cost,
      usage: s.usage ? {
        ...s.usage,
        currentPeriod: {
          ...s.usage.currentPeriod,
          totalCost: s.usage.currentPeriod.totalCost + cost,
          totalInputTokens: s.usage.currentPeriod.totalInputTokens + inputTokens,
          totalOutputTokens: s.usage.currentPeriod.totalOutputTokens + outputTokens,
          requests: s.usage.currentPeriod.requests + 1,
          byModel: {
            ...s.usage.currentPeriod.byModel,
            [modelId]: {
              requests: (s.usage.currentPeriod.byModel[modelId]?.requests ?? 0) + 1,
              inputTokens: (s.usage.currentPeriod.byModel[modelId]?.inputTokens ?? 0) + inputTokens,
              outputTokens: (s.usage.currentPeriod.byModel[modelId]?.outputTokens ?? 0) + outputTokens,
              cost: (s.usage.currentPeriod.byModel[modelId]?.cost ?? 0) + cost,
            },
          },
          byTask: {
            ...s.usage.currentPeriod.byTask,
            [taskType]: {
              requests: (s.usage.currentPeriod.byTask[taskType]?.requests ?? 0) + 1,
              cost: (s.usage.currentPeriod.byTask[taskType]?.cost ?? 0) + cost,
            },
          },
        },
      } : null,
    }));
  },

  loadUsage: async (projectPath: string) => {
    try {
      const usage = await invoke<UsageData>('load_usage', { projectPath });
      set({ usage });
    } catch {
      // Initialize empty usage
    }
  },

  isOverBudget: () => {
    const { usage, config } = get();
    if (!config.costTracking.enabled || config.costTracking.budgetLimit === 0) return false;
    return (usage?.currentPeriod.totalCost ?? 0) >= config.costTracking.budgetLimit;
  },

  isNearBudget: () => {
    const { usage, config } = get();
    if (!config.costTracking.enabled) return false;
    return (usage?.currentPeriod.totalCost ?? 0) >= config.costTracking.budgetWarning;
  },

  updateConfig: (partial: Partial<LLMConfig>) => {
    set(s => ({ config: { ...s.config, ...partial } }));
  },
}));

// Helper: parse generated YAML into ghost nodes for canvas preview
function parseYamlToGhostPreview(
  messageId: string,
  yaml: string
): GhostPreview {
  // Parse YAML, extract nodes[] and connections
  // Position nodes in a vertical layout starting from (400, 100)
  // Returns GhostPreview for canvas rendering
  return {
    id: nanoid(),
    sourceMessageId: messageId,
    yaml,
    nodes: [],       // Populated by YAML parser
    connections: [],  // Populated by YAML parser
    state: 'previewing',
  };
}
```

**File: `src/utils/llm-context.ts`**
```typescript
import { useProjectStore } from '../stores/project-store';
import { useSheetStore } from '../stores/sheet-store';
import { useFlowStore } from '../stores/flow-store';
import type { LLMContext } from '../types/llm';

/**
 * Build the context object sent with every LLM request.
 * Automatically scoped to the user's current location and selection.
 */
export function buildLLMContext(): LLMContext {
  const project = useProjectStore.getState();
  const sheet = useSheetStore.getState();
  const flow = useFlowStore.getState();

  const context: LLMContext = {
    system: {
      name: project.systemConfig?.name ?? '',
      techStack: project.systemConfig?.tech_stack ?? {},
      domains: project.domains?.map(d => d.name) ?? [],
    },
  };

  // Add domain context on Level 2 or 3
  if (sheet.current.level !== 'system' && sheet.current.domainId) {
    const domain = project.domainConfigs?.[sheet.current.domainId];
    if (domain) {
      context.currentDomain = {
        name: domain.name,
        flows: domain.flows.map(f => f.id),
        publishesEvents: domain.publishes_events.map(e => e.event),
        consumesEvents: domain.consumes_events.map(e => e.event),
      };
    }
  }

  // Add flow context on Level 3
  if (sheet.current.level === 'flow' && sheet.current.flowId) {
    context.currentFlow = {
      id: sheet.current.flowId,
      type: flow.flowType ?? 'traditional',
      yaml: flow.toYaml(),      // Serialize current flow state
    };
  }

  // Add selected nodes
  const selectedNodes = flow.getSelectedNodes?.();
  if (selectedNodes?.length) {
    context.selectedNodes = selectedNodes.map(n => ({
      id: n.id,
      type: n.type,
      spec: n.spec,
    }));
  }

  // Add error codes and schemas from project
  context.errorCodes = project.errorCodes ?? [];
  context.schemas = project.schemas ?? {};

  return context;
}
```

**File: `src-tauri/src/commands/llm.rs`**
```rust
use serde::{Deserialize, Serialize};
use tauri::command;

/// A single model from the registry — sent from the frontend per-request
#[derive(Deserialize, Clone)]
pub struct ModelEntry {
    id: String,
    provider: String,            // "anthropic" | "openai" | "ollama" | "openai_compatible"
    model_id: String,            // Provider-specific model ID
    api_key_env: Option<String>, // Env var name (None for Ollama)
    base_url: Option<String>,    // For ollama / openai_compatible
    max_tokens: u32,
    temperature: f32,
}

#[derive(Deserialize)]
pub struct ChatMsg {
    role: String,
    content: String,
}

#[derive(Serialize)]
pub struct LLMResult {
    content: String,
    input_tokens: u32,
    output_tokens: u32,
}

/// Main chat endpoint — the frontend resolves which model to use and passes it directly
#[command]
pub async fn llm_chat(
    messages: Vec<ChatMsg>,
    context: serde_json::Value,
    model: ModelEntry,
) -> Result<LLMResult, String> {
    let api_key = resolve_api_key(&model)?;
    let system_prompt = build_system_prompt(&context);

    call_model(&model, &api_key, &system_prompt, &messages).await
}

/// Inline assist endpoint
#[command]
pub async fn llm_inline_assist(
    action: String,
    target: serde_json::Value,
    context: serde_json::Value,
    model: ModelEntry,
) -> Result<LLMResult, String> {
    let api_key = resolve_api_key(&model)?;
    let system_prompt = build_inline_assist_prompt(&action, &target, &context);

    let messages = vec![ChatMsg {
        role: "user".to_string(),
        content: format_inline_assist_user_message(&action, &target),
    }];

    call_model(&model, &api_key, &system_prompt, &messages).await
}

/// Health check — test if a model endpoint is reachable
#[command]
pub async fn llm_health_check(model: ModelEntry) -> Result<bool, String> {
    let api_key = resolve_api_key(&model).unwrap_or_default();

    let messages = vec![ChatMsg {
        role: "user".to_string(),
        content: "ping".to_string(),
    }];

    match call_model(&model, &api_key, "Respond with 'pong'.", &messages).await {
        Ok(_) => Ok(true),
        Err(_) => Ok(false),
    }
}

/// Resolve API key from environment variable
fn resolve_api_key(model: &ModelEntry) -> Result<String, String> {
    match &model.api_key_env {
        Some(env_var) => std::env::var(env_var)
            .map_err(|_| format!("Environment variable {} not set", env_var)),
        None => Ok(String::new()), // Ollama doesn't need an API key
    }
}

/// Unified model caller — routes to the correct provider
async fn call_model(
    model: &ModelEntry,
    api_key: &str,
    system_prompt: &str,
    messages: &[ChatMsg],
) -> Result<LLMResult, String> {
    match model.provider.as_str() {
        "anthropic" => call_anthropic(api_key, &model.model_id, system_prompt, messages, model.max_tokens, model.temperature).await,
        "openai" => call_openai(api_key, &model.model_id, system_prompt, messages, model.max_tokens, model.temperature, None).await,
        "ollama" => call_ollama(&model.model_id, system_prompt, messages, model.base_url.as_deref()).await,
        "openai_compatible" => call_openai(api_key, &model.model_id, system_prompt, messages, model.max_tokens, model.temperature, model.base_url.as_deref()).await,
        _ => Err(format!("Unsupported provider: {}", model.provider)),
    }
}

fn build_system_prompt(context: &serde_json::Value) -> String {
    format!(
        r#"You are the DDD Design Assistant, embedded in the DDD (Diagram-Driven Development) desktop tool.
You help users design software flows visually.

When generating flows or nodes, output valid DDD YAML inside a ```yaml``` code block.
Always follow the DDD spec format with proper node types, connections, and specs.

Project context:
{}

When suggesting specs, use exact error codes from the project's errors.yaml.
When suggesting schemas, reference existing schemas with $ref.
Keep suggestions concise and actionable."#,
        serde_json::to_string_pretty(context).unwrap_or_default()
    )
}

fn build_inline_assist_prompt(
    action: &str,
    target: &serde_json::Value,
    context: &serde_json::Value,
) -> String {
    let action_instruction = match action {
        "suggest_spec" => "Suggest a complete spec for this node based on its name, type, and the flow context. Output the spec as YAML.",
        "complete_spec" => "The node has a partial spec. Fill in any missing fields with sensible defaults based on context.",
        "explain_node" => "Explain what this node does in plain language, based on its spec.",
        "add_error_handling" => "Add failure connections and error handling to this node. Use error codes from the project's errors.yaml.",
        "generate_test_cases" => "List test scenarios for this node: happy path, edge cases, and failure cases.",
        "add_node_between" => "Suggest a node to insert on this connection based on the source and target node semantics.",
        "label_connection" => "Suggest a label for this connection based on the source and target nodes.",
        "generate_flow" => "Generate a complete DDD flow with nodes and connections based on the user's description. Output as YAML.",
        "review_flow" => "Review this flow for completeness: missing error paths, unhandled edges, security gaps, missing validations.",
        "suggest_wiring" => "Look at unconnected nodes in this flow and suggest how to wire them together.",
        "import_from_description" => "Convert this feature description into a DDD flow with nodes and connections. Output as YAML.",
        "suggest_flows" => "Based on this domain's existing flows, suggest additional flows that are typically needed.",
        "suggest_events" => "Based on this domain's flows, suggest events to publish and consume.",
        "generate_domain" => "Generate a domain.yaml with flows, events, and wiring based on the description. Output as YAML.",
        "suggest_domains" => "Based on the system description and existing domains, suggest additional domains.",
        "review_architecture" => "Check domain boundaries, event wiring completeness, and circular dependencies.",
        "generate_from_description" => "Generate a system.yaml with domains based on the application description. Output as YAML.",
        _ => "Help the user with their DDD design.",
    };

    format!(
        r#"You are the DDD Inline Assistant. Perform exactly ONE action.

Action: {}
Instruction: {}

Target: {}

Project context:
{}

Be concise. If outputting YAML, use a ```yaml``` code block."#,
        action,
        action_instruction,
        serde_json::to_string_pretty(target).unwrap_or_default(),
        serde_json::to_string_pretty(context).unwrap_or_default()
    )
}

fn format_inline_assist_user_message(action: &str, target: &serde_json::Value) -> String {
    format!("Execute inline assist action '{}' on target: {}", action, target)
}

// Provider implementations — all return LLMResult with token counts

async fn call_anthropic(
    api_key: &str,
    model: &str,
    system_prompt: &str,
    messages: &[ChatMsg],
    max_tokens: u32,
    temperature: f32,
) -> Result<LLMResult, String> {
    let client = reqwest::Client::new();

    let api_messages: Vec<serde_json::Value> = messages.iter().map(|m| {
        serde_json::json!({ "role": m.role, "content": m.content })
    }).collect();

    let body = serde_json::json!({
        "model": model,
        "max_tokens": max_tokens,
        "temperature": temperature,
        "system": system_prompt,
        "messages": api_messages
    });

    let response = client.post("https://api.anthropic.com/v1/messages")
        .header("x-api-key", api_key)
        .header("anthropic-version", "2023-06-01")
        .header("content-type", "application/json")
        .json(&body)
        .send().await.map_err(|e| e.to_string())?;

    let json: serde_json::Value = response.json().await.map_err(|e| e.to_string())?;

    let content = json["content"][0]["text"].as_str()
        .ok_or_else(|| "Failed to parse Anthropic response".to_string())?
        .to_string();

    Ok(LLMResult {
        content,
        input_tokens: json["usage"]["input_tokens"].as_u64().unwrap_or(0) as u32,
        output_tokens: json["usage"]["output_tokens"].as_u64().unwrap_or(0) as u32,
    })
}

async fn call_openai(
    api_key: &str,
    model: &str,
    system_prompt: &str,
    messages: &[ChatMsg],
    max_tokens: u32,
    temperature: f32,
    base_url: Option<&str>,        // None = OpenAI, Some = Azure/Together/Groq/etc.
) -> Result<LLMResult, String> {
    let client = reqwest::Client::new();
    let url = format!("{}/chat/completions",
        base_url.unwrap_or("https://api.openai.com/v1"));

    let mut api_messages = vec![serde_json::json!({
        "role": "system", "content": system_prompt
    })];
    for m in messages {
        api_messages.push(serde_json::json!({ "role": m.role, "content": m.content }));
    }

    let body = serde_json::json!({
        "model": model,
        "max_tokens": max_tokens,
        "temperature": temperature,
        "messages": api_messages
    });

    let response = client.post(&url)
        .header("Authorization", format!("Bearer {}", api_key))
        .header("content-type", "application/json")
        .json(&body)
        .send().await.map_err(|e| e.to_string())?;

    let json: serde_json::Value = response.json().await.map_err(|e| e.to_string())?;

    let content = json["choices"][0]["message"]["content"].as_str()
        .ok_or_else(|| "Failed to parse OpenAI response".to_string())?
        .to_string();

    Ok(LLMResult {
        content,
        input_tokens: json["usage"]["prompt_tokens"].as_u64().unwrap_or(0) as u32,
        output_tokens: json["usage"]["completion_tokens"].as_u64().unwrap_or(0) as u32,
    })
}

async fn call_ollama(
    model: &str,
    system_prompt: &str,
    messages: &[ChatMsg],
    base_url: Option<&str>,
) -> Result<LLMResult, String> {
    let client = reqwest::Client::new();
    let url = format!("{}/api/chat", base_url.unwrap_or("http://localhost:11434"));

    let mut api_messages = vec![serde_json::json!({
        "role": "system", "content": system_prompt
    })];
    for m in messages {
        api_messages.push(serde_json::json!({ "role": m.role, "content": m.content }));
    }

    let body = serde_json::json!({
        "model": model,
        "messages": api_messages,
        "stream": false
    });

    let response = client.post(&url)
        .json(&body)
        .send().await.map_err(|e| e.to_string())?;

    let json: serde_json::Value = response.json().await.map_err(|e| e.to_string())?;

    let content = json["message"]["content"].as_str()
        .ok_or_else(|| "Failed to parse Ollama response".to_string())?
        .to_string();

    Ok(LLMResult {
        content,
        input_tokens: json["prompt_eval_count"].as_u64().unwrap_or(0) as u32,
        output_tokens: json["eval_count"].as_u64().unwrap_or(0) as u32,
    })
}
```

**File: `src/components/LLMAssistant/ChatPanel.tsx`**
```tsx
import { useEffect, useRef, useState } from 'react';
import { useLLMStore } from '../../stores/llm-store';
import { ChatMessage } from './ChatMessage';
import { ChatInput } from './ChatInput';
import { GhostPreview } from './GhostPreview';
import { X, MessageSquare, Sparkles } from 'lucide-react';

export function ChatPanel() {
  const {
    chatOpen, activeThread, isStreaming,
    sessionTokens, sessionCost,
    closeChat, sendMessage,
  } = useLLMStore();

  const messagesEndRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [activeThread?.messages.length]);

  if (!chatOpen) return null;

  return (
    <div className="w-96 border-l border-gray-200 bg-white flex flex-col h-full">
      {/* Header with Model Picker */}
      <div className="flex items-center justify-between px-4 py-3 border-b border-gray-200">
        <div className="flex items-center gap-2">
          <Sparkles className="w-4 h-4 text-purple-500" />
          <span className="font-medium text-sm">Design Assistant</span>
        </div>
        <div className="flex items-center gap-2">
          <ModelPicker />
          <button onClick={closeChat} className="text-gray-400 hover:text-gray-600">
            <X className="w-4 h-4" />
          </button>
        </div>
      </div>

      {/* Context indicator */}
      <div className="px-4 py-2 bg-gray-50 text-xs text-gray-500 border-b flex justify-between">
        <span>
          {activeThread?.flowId
            ? `Context: ${activeThread.domainId} / ${activeThread.flowId}`
            : 'Context: Project-level'}
        </span>
        <span className="text-gray-400">
          {sessionTokens > 0 && `${(sessionTokens / 1000).toFixed(1)}K tok · $${sessionCost.toFixed(2)}`}
        </span>
      </div>

      {/* Messages */}
      <div className="flex-1 overflow-y-auto px-4 py-3 space-y-4">
        {!activeThread?.messages.length && (
          <div className="text-center text-gray-400 text-sm mt-8">
            <MessageSquare className="w-8 h-8 mx-auto mb-2 opacity-50" />
            <p>Ask me to generate flows, fill specs,</p>
            <p>review your design, or suggest improvements.</p>
            <div className="mt-4 space-y-2">
              <SuggestionChip text="Generate a user registration flow" />
              <SuggestionChip text="Review this flow for missing error paths" />
              <SuggestionChip text="Suggest flows for this domain" />
            </div>
          </div>
        )}

        {activeThread?.messages.map(msg => (
          <ChatMessage key={msg.id} message={msg} />
        ))}

        {isStreaming && (
          <div className="flex gap-1 text-purple-400">
            <span className="animate-bounce">●</span>
            <span className="animate-bounce" style={{ animationDelay: '0.1s' }}>●</span>
            <span className="animate-bounce" style={{ animationDelay: '0.2s' }}>●</span>
          </div>
        )}

        <div ref={messagesEndRef} />
      </div>

      {/* Input */}
      <ChatInput onSend={sendMessage} disabled={isStreaming} />
    </div>
  );
}

function SuggestionChip({ text }: { text: string }) {
  const { sendMessage } = useLLMStore();
  return (
    <button
      onClick={() => sendMessage(text)}
      className="block w-full text-left px-3 py-2 text-xs bg-white border border-gray-200 rounded-lg hover:border-purple-300 hover:bg-purple-50 transition-colors"
    >
      ✨ {text}
    </button>
  );
}
```

**File: `src/components/LLMAssistant/ChatMessage.tsx`**
```tsx
import type { ChatMessage as ChatMessageType } from '../../types/llm';
import { useLLMStore } from '../../stores/llm-store';
import { User, Sparkles, Check, X } from 'lucide-react';

interface Props {
  message: ChatMessageType;
}

export function ChatMessage({ message }: Props) {
  const { applyGhostPreview, discardGhostPreview, editGhostInChat } = useLLMStore();
  const isUser = message.role === 'user';

  return (
    <div className={`flex gap-2 ${isUser ? 'justify-end' : 'justify-start'}`}>
      {!isUser && (
        <div className="w-6 h-6 rounded-full bg-purple-100 flex items-center justify-center flex-shrink-0">
          <Sparkles className="w-3 h-3 text-purple-600" />
        </div>
      )}

      <div
        className={`max-w-[85%] rounded-lg px-3 py-2 text-sm ${
          isUser
            ? 'bg-blue-500 text-white'
            : 'bg-gray-100 text-gray-800'
        }`}
      >
        {/* Render markdown content */}
        <div className="whitespace-pre-wrap">{message.content}</div>

        {/* Generated YAML actions */}
        {message.generatedYaml && message.previewState === 'pending' && (
          <div className="mt-2 pt-2 border-t border-gray-200 flex gap-2">
            <button
              onClick={applyGhostPreview}
              className="flex items-center gap-1 px-2 py-1 text-xs bg-green-500 text-white rounded hover:bg-green-600"
            >
              <Check className="w-3 h-3" /> Apply
            </button>
            <button
              onClick={editGhostInChat}
              className="px-2 py-1 text-xs bg-gray-200 text-gray-700 rounded hover:bg-gray-300"
            >
              Edit
            </button>
            <button
              onClick={discardGhostPreview}
              className="flex items-center gap-1 px-2 py-1 text-xs bg-red-100 text-red-600 rounded hover:bg-red-200"
            >
              <X className="w-3 h-3" /> Discard
            </button>
          </div>
        )}

        {message.previewState === 'applied' && (
          <div className="mt-2 pt-2 border-t border-gray-200 text-xs text-green-600">
            ✓ Applied to canvas
          </div>
        )}

        {message.previewState === 'discarded' && (
          <div className="mt-2 pt-2 border-t border-gray-200 text-xs text-gray-400">
            Discarded
          </div>
        )}
      </div>

      {isUser && (
        <div className="w-6 h-6 rounded-full bg-blue-100 flex items-center justify-center flex-shrink-0">
          <User className="w-3 h-3 text-blue-600" />
        </div>
      )}
    </div>
  );
}
```

**File: `src/components/LLMAssistant/ChatInput.tsx`**
```tsx
import { useState, useRef, useEffect } from 'react';
import { Send } from 'lucide-react';

interface Props {
  onSend: (message: string) => void;
  disabled: boolean;
}

export function ChatInput({ onSend, disabled }: Props) {
  const [value, setValue] = useState('');
  const textareaRef = useRef<HTMLTextAreaElement>(null);

  // Auto-resize textarea
  useEffect(() => {
    const el = textareaRef.current;
    if (el) {
      el.style.height = 'auto';
      el.style.height = `${Math.min(el.scrollHeight, 120)}px`;
    }
  }, [value]);

  const handleSubmit = () => {
    const trimmed = value.trim();
    if (!trimmed || disabled) return;
    onSend(trimmed);
    setValue('');
  };

  const handleKeyDown = (e: React.KeyboardEvent) => {
    if (e.key === 'Enter' && !e.shiftKey) {
      e.preventDefault();
      handleSubmit();
    }
  };

  return (
    <div className="border-t border-gray-200 px-4 py-3">
      <div className="flex items-end gap-2">
        <textarea
          ref={textareaRef}
          value={value}
          onChange={e => setValue(e.target.value)}
          onKeyDown={handleKeyDown}
          placeholder="Describe what you need…"
          disabled={disabled}
          rows={1}
          className="flex-1 resize-none border border-gray-300 rounded-lg px-3 py-2 text-sm focus:outline-none focus:ring-2 focus:ring-purple-300 disabled:opacity-50"
        />
        <button
          onClick={handleSubmit}
          disabled={disabled || !value.trim()}
          className="p-2 rounded-lg bg-purple-500 text-white hover:bg-purple-600 disabled:opacity-50 disabled:cursor-not-allowed"
        >
          <Send className="w-4 h-4" />
        </button>
      </div>
    </div>
  );
}
```

**File: `src/components/LLMAssistant/GhostPreview.tsx`**
```tsx
import { useLLMStore } from '../../stores/llm-store';
import { Check, Pencil, X, Sparkles } from 'lucide-react';

/**
 * Floating bar rendered at the bottom of the canvas when ghost nodes are visible.
 * Ghost nodes themselves are rendered by the Canvas component with dashed borders
 * and reduced opacity — this component just provides the Apply/Edit/Discard controls.
 */
export function GhostPreview() {
  const { ghostPreview, applyGhostPreview, editGhostInChat, discardGhostPreview } = useLLMStore();

  if (!ghostPreview || ghostPreview.state !== 'previewing') return null;

  const nodeCount = ghostPreview.nodes.length;
  const connectionCount = ghostPreview.connections.length;

  return (
    <div className="absolute bottom-6 left-1/2 -translate-x-1/2 z-50">
      <div className="bg-white border border-purple-200 rounded-xl shadow-lg px-5 py-3 flex items-center gap-4">
        <div className="flex items-center gap-2 text-sm text-purple-700">
          <Sparkles className="w-4 h-4" />
          <span>
            Generated {nodeCount} node{nodeCount !== 1 ? 's' : ''}
            {connectionCount > 0 && `, ${connectionCount} connection${connectionCount !== 1 ? 's' : ''}`}
          </span>
        </div>

        <div className="flex gap-2">
          <button
            onClick={applyGhostPreview}
            className="flex items-center gap-1 px-3 py-1.5 text-sm bg-green-500 text-white rounded-lg hover:bg-green-600 transition-colors"
          >
            <Check className="w-3.5 h-3.5" /> Apply
          </button>
          <button
            onClick={editGhostInChat}
            className="flex items-center gap-1 px-3 py-1.5 text-sm bg-gray-100 text-gray-700 rounded-lg hover:bg-gray-200 transition-colors"
          >
            <Pencil className="w-3.5 h-3.5" /> Edit in chat
          </button>
          <button
            onClick={discardGhostPreview}
            className="flex items-center gap-1 px-3 py-1.5 text-sm bg-red-50 text-red-600 rounded-lg hover:bg-red-100 transition-colors"
          >
            <X className="w-3.5 h-3.5" /> Discard
          </button>
        </div>
      </div>
    </div>
  );
}
```

**File: `src/components/LLMAssistant/InlineAssist.tsx`**
```tsx
import { useState } from 'react';
import { useLLMStore } from '../../stores/llm-store';
import { useSheetStore } from '../../stores/sheet-store';
import { Sparkles, Loader2 } from 'lucide-react';
import type { InlineAssistAction, InlineAssistTarget } from '../../types/llm';

interface Props {
  x: number;
  y: number;
  target: InlineAssistTarget;
  onClose: () => void;
}

interface MenuAction {
  action: InlineAssistAction;
  label: string;
}

const NODE_ACTIONS: MenuAction[] = [
  { action: 'suggest_spec', label: '✨ Suggest spec' },
  { action: 'complete_spec', label: '✨ Complete spec' },
  { action: 'explain_node', label: '✨ Explain this node' },
  { action: 'add_error_handling', label: '✨ Add error handling' },
  { action: 'generate_test_cases', label: '✨ Generate test cases' },
];

const CONNECTION_ACTIONS: MenuAction[] = [
  { action: 'add_node_between', label: '✨ Add node between' },
  { action: 'label_connection', label: '✨ Suggest label' },
];

const CANVAS_ACTIONS: MenuAction[] = [
  { action: 'generate_flow', label: '✨ Generate flow…' },
  { action: 'review_flow', label: '✨ Review this flow' },
  { action: 'suggest_wiring', label: '✨ Suggest wiring' },
  { action: 'import_from_description', label: '✨ Import from description…' },
];

const DOMAIN_ACTIONS: MenuAction[] = [
  { action: 'suggest_flows', label: '✨ Suggest flows' },
  { action: 'suggest_events', label: '✨ Suggest events' },
  { action: 'generate_domain', label: '✨ Generate domain…' },
];

const SYSTEM_ACTIONS: MenuAction[] = [
  { action: 'suggest_domains', label: '✨ Suggest domains' },
  { action: 'review_architecture', label: '✨ Review architecture' },
  { action: 'generate_from_description', label: '✨ Generate from description…' },
];

function getActionsForTarget(target: InlineAssistTarget): MenuAction[] {
  switch (target.type) {
    case 'node': return NODE_ACTIONS;
    case 'connection': return CONNECTION_ACTIONS;
    case 'canvas': return CANVAS_ACTIONS;
    case 'domain': return DOMAIN_ACTIONS;
    case 'system': return SYSTEM_ACTIONS;
  }
}

export function InlineAssist({ x, y, target, onClose }: Props) {
  const { executeInlineAssist, openChat } = useLLMStore();
  const [loading, setLoading] = useState<string | null>(null);
  const [result, setResult] = useState<string | null>(null);

  const actions = getActionsForTarget(target);

  const handleAction = async (menuAction: MenuAction) => {
    // Actions ending with '…' open the chat panel instead
    if (menuAction.label.endsWith('…')) {
      openChat();
      onClose();
      return;
    }

    setLoading(menuAction.action);
    try {
      const response = await executeInlineAssist({
        action: menuAction.action,
        target,
      });
      setResult(response);
    } catch (err) {
      setResult('Error: ' + String(err));
    } finally {
      setLoading(null);
    }
  };

  return (
    <div
      className="fixed z-50 bg-white rounded-lg shadow-xl border border-gray-200 min-w-[220px] py-1"
      style={{ left: x, top: y }}
    >
      {!result ? (
        // Action menu
        actions.map(a => (
          <button
            key={a.action}
            onClick={() => handleAction(a)}
            disabled={!!loading}
            className="w-full text-left px-4 py-2 text-sm hover:bg-purple-50 disabled:opacity-50 flex items-center gap-2"
          >
            {loading === a.action ? (
              <Loader2 className="w-3.5 h-3.5 animate-spin text-purple-500" />
            ) : null}
            {a.label}
          </button>
        ))
      ) : (
        // Result popover
        <div className="px-4 py-3 max-w-sm">
          <div className="text-xs text-gray-500 mb-1 flex items-center gap-1">
            <Sparkles className="w-3 h-3 text-purple-500" />
            Suggestion
          </div>
          <div className="text-sm whitespace-pre-wrap max-h-64 overflow-y-auto">
            {result}
          </div>
          <div className="mt-3 flex gap-2 justify-end">
            <button
              onClick={onClose}
              className="px-2 py-1 text-xs bg-gray-100 rounded hover:bg-gray-200"
            >
              Close
            </button>
          </div>
        </div>
      )}
    </div>
  );
}
```

**File: `src/components/LLMAssistant/ModelPicker.tsx`**
```tsx
import { useState, useRef, useEffect } from 'react';
import { useLLMStore } from '../../stores/llm-store';
import { ChevronDown, Circle, Settings } from 'lucide-react';

export function ModelPicker() {
  const { config, setActiveModel, isNearBudget, usage } = useLLMStore();
  const [open, setOpen] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  const activeModel = config.models[config.activeModel];

  // Close on outside click
  useEffect(() => {
    const handler = (e: MouseEvent) => {
      if (ref.current && !ref.current.contains(e.target as Node)) setOpen(false);
    };
    document.addEventListener('mousedown', handler);
    return () => document.removeEventListener('mousedown', handler);
  }, []);

  // Group models by provider
  const grouped = Object.values(config.models).reduce((acc, model) => {
    const group = model.provider === 'ollama' ? 'Local' :
                  model.provider === 'anthropic' ? 'Anthropic' :
                  model.provider === 'openai' ? 'OpenAI' : 'Other';
    if (!acc[group]) acc[group] = [];
    acc[group].push(model);
    return acc;
  }, {} as Record<string, typeof Object.values<any>>);

  return (
    <div className="relative" ref={ref}>
      <button
        onClick={() => setOpen(!open)}
        className="flex items-center gap-1.5 px-2 py-1 text-xs border border-gray-200 rounded-md hover:bg-gray-50"
      >
        <Circle
          className={`w-2 h-2 ${activeModel?.available !== false ? 'fill-green-500 text-green-500' : 'fill-red-400 text-red-400'}`}
        />
        <span className="max-w-[120px] truncate">{activeModel?.label ?? 'Select model'}</span>
        <ChevronDown className="w-3 h-3 text-gray-400" />
      </button>

      {open && (
        <div className="absolute right-0 top-full mt-1 w-64 bg-white border border-gray-200 rounded-lg shadow-xl z-50 py-1">
          {Object.entries(grouped).map(([group, models]) => (
            <div key={group}>
              <div className="px-3 py-1.5 text-xs font-medium text-gray-400 uppercase tracking-wide">
                {group}
              </div>
              {(models as any[]).map((model: any) => (
                <button
                  key={model.id}
                  onClick={() => { setActiveModel(model.id); setOpen(false); }}
                  className={`w-full flex items-center justify-between px-3 py-2 text-sm hover:bg-purple-50 ${
                    model.id === config.activeModel ? 'bg-purple-50 text-purple-700' : 'text-gray-700'
                  }`}
                >
                  <div className="flex items-center gap-2">
                    <Circle
                      className={`w-2 h-2 ${model.available !== false ? 'fill-green-500 text-green-500' : 'fill-red-400 text-red-400'}`}
                    />
                    <span>{model.label}</span>
                  </div>
                  <span className="text-xs text-gray-400">
                    {model.costPer1kInput === 0 ? 'Free' : `$${model.costPer1kInput}`}
                  </span>
                </button>
              ))}
            </div>
          ))}

          <div className="border-t border-gray-100 mt-1">
            <button className="w-full flex items-center gap-2 px-3 py-2 text-sm text-gray-500 hover:bg-gray-50">
              <Settings className="w-3.5 h-3.5" />
              Manage models…
            </button>
          </div>

          {/* Budget indicator */}
          {usage && config.costTracking.enabled && (
            <div className={`border-t px-3 py-2 text-xs ${
              isNearBudget() ? 'text-amber-600 bg-amber-50' : 'text-gray-400'
            }`}>
              Month: ${usage.currentPeriod.totalCost.toFixed(2)}
              {config.costTracking.budgetLimit > 0 && ` / $${config.costTracking.budgetLimit.toFixed(2)}`}
            </div>
          )}
        </div>
      )}
    </div>
  );
}
```

**File: `src/components/LLMAssistant/UsageBar.tsx`**
```tsx
import { useLLMStore } from '../../stores/llm-store';
import { Circle } from 'lucide-react';

/**
 * Status bar at the bottom of the app showing active model + usage.
 */
export function UsageBar() {
  const { config, usage, sessionTokens, sessionCost, isNearBudget, isOverBudget } = useLLMStore();

  const activeModel = config.models[config.activeModel];

  return (
    <div className="h-6 border-t border-gray-200 bg-gray-50 px-4 flex items-center justify-between text-xs text-gray-500">
      {/* Active model */}
      <div className="flex items-center gap-2">
        <Circle
          className={`w-2 h-2 ${activeModel?.available !== false ? 'fill-green-500 text-green-500' : 'fill-red-400 text-red-400'}`}
        />
        <span>{activeModel?.label ?? 'No model'}</span>
      </div>

      {/* Session + period usage */}
      <div className="flex items-center gap-4">
        {sessionTokens > 0 && (
          <span>Session: {(sessionTokens / 1000).toFixed(1)}K tok · ${sessionCost.toFixed(2)}</span>
        )}

        {usage && config.costTracking.enabled && (
          <span className={
            isOverBudget() ? 'text-red-600 font-medium' :
            isNearBudget() ? 'text-amber-600' : ''
          }>
            Month: ${usage.currentPeriod.totalCost.toFixed(2)}
            {config.costTracking.budgetLimit > 0 && `/$${config.costTracking.budgetLimit.toFixed(2)}`}
          </span>
        )}
      </div>
    </div>
  );
}
```

**Update the InlineAssist context menu to show routed model and allow override:**

Add to `src/components/LLMAssistant/InlineAssist.tsx` — update the action menu items:
```tsx
// Inside InlineAssist component, update the action rendering:
const { resolveModelForTask } = useLLMStore();

// In the menu render:
actions.map(a => {
  const routedModel = resolveModelForTask(a.action as TaskType);
  return (
    <button
      key={a.action}
      onClick={() => handleAction(a)}
      disabled={!!loading}
      className="w-full text-left px-4 py-2 text-sm hover:bg-purple-50 disabled:opacity-50 flex items-center justify-between"
    >
      <span className="flex items-center gap-2">
        {loading === a.action ? (
          <Loader2 className="w-3.5 h-3.5 animate-spin text-purple-500" />
        ) : null}
        {a.label}
      </span>
      <span className="text-xs text-gray-400">{routedModel.label}</span>
    </button>
  );
})
```

**Keyboard shortcuts — add to `src/App.tsx`:**
```typescript
// Inside App component useEffect for keyboard shortcuts
useEffect(() => {
  const handleKeyDown = (e: KeyboardEvent) => {
    const mod = e.metaKey || e.ctrlKey;

    // Cmd+L / Ctrl+L — Toggle Chat Panel
    if (mod && e.key === 'l') {
      e.preventDefault();
      useLLMStore.getState().toggleChat();
    }

    // Cmd+Shift+G — Generate flow from description
    if (mod && e.shiftKey && e.key === 'G') {
      e.preventDefault();
      useLLMStore.getState().openChat();
      // Pre-fill with generate prompt (handled by ChatPanel)
    }

    // Cmd+. — Inline assist on selected node
    if (mod && e.key === '.') {
      e.preventDefault();
      // Trigger inline assist for currently selected node
      // (handled by Canvas component)
    }

    // Escape — Discard ghost preview (if active)
    if (e.key === 'Escape') {
      const ghost = useLLMStore.getState().ghostPreview;
      if (ghost?.state === 'previewing') {
        e.preventDefault();
        useLLMStore.getState().discardGhostPreview();
      }
    }

    // Enter — Apply ghost preview (if active and no input focused)
    if (e.key === 'Enter' && !['INPUT', 'TEXTAREA', 'SELECT'].includes((e.target as HTMLElement).tagName)) {
      const ghost = useLLMStore.getState().ghostPreview;
      if (ghost?.state === 'previewing') {
        e.preventDefault();
        useLLMStore.getState().applyGhostPreview();
      }
    }
  };

  window.addEventListener('keydown', handleKeyDown);
  return () => window.removeEventListener('keydown', handleKeyDown);
}, []);
```

**App layout update — ChatPanel, MemoryPanel, and UsageBar:**
```tsx
// In App.tsx render
function App() {
  const { current } = useSheetStore();
  const { chatOpen } = useLLMStore();
  const { memoryPanelOpen } = useMemoryStore();

  return (
    <div className="flex flex-col h-screen">
      <div className="flex flex-1 overflow-hidden">
        {/* Left sidebar */}
        <Sidebar />

        {/* Main canvas area */}
        <div className="flex-1 flex flex-col">
          <Breadcrumb />
          <div className="flex-1 relative">
            {current.level === 'system' && <SystemMap />}
            {current.level === 'domain' && <DomainMap />}
            {current.level === 'flow' && <Canvas />}

            {/* Ghost preview bar (floats above canvas) */}
            <GhostPreview />
          </div>
        </div>

        {/* Right panels — Spec Panel, Chat Panel, and/or Memory Panel */}
        <SpecPanel />
        {chatOpen && <ChatPanel />}
        {memoryPanelOpen && <MemoryPanel />}
      </div>

      {/* Bottom status bar — active model + usage tracking */}
      <UsageBar />
    </div>
  );
}
```

#### Day 18-19: Project Memory System

**File: `src/types/memory.ts`**
```typescript
// --- Project Memory Layer Types ---

// Layer 1: Project Summary
export interface ProjectSummary {
  content: string;              // Markdown content
  generatedAt: number;          // Timestamp
  stale: boolean;               // True if specs changed since generation
}

// Layer 2: Spec Index
export interface SpecIndex {
  domains: Record<string, SpecIndexDomain>;
  schemas: Record<string, SpecIndexSchema>;
  events: SpecIndexEvent[];
  sharedErrorCodes: string[];
  generatedAt: number;
}

export interface SpecIndexDomain {
  description: string;
  flows: Record<string, SpecIndexFlow>;
}

export interface SpecIndexFlow {
  type: 'traditional' | 'agent';
  trigger: {
    type: string;
    method?: string;
    path?: string;
    schedule?: string;
    event?: string;
  };
  nodes: string[];               // Node IDs (condensed)
  publishesEvents: string[];
  consumesEvents: string[];
  schemasUsed: string[];
  errorCodes: string[];
  // Agent-specific
  agentModel?: string;
  tools?: string[];
  guardrails?: string[];
}

export interface SpecIndexSchema {
  fields: string[];              // Field names (condensed)
}

export interface SpecIndexEvent {
  name: string;
  publisher: string;             // domain name
  consumers: string[];           // domain names
}

// Layer 3: Decision Log
export interface Decision {
  id: string;
  date: string;                  // ISO date string
  title: string;
  rationale: string;
  affected: string[];            // Domain/flow/file references
  author: 'user' | 'llm';       // Who captured it
}

// Layer 4: Cross-Flow Map
export interface FlowMap {
  events: FlowMapEdge[];
  subFlows: FlowMapEdge[];
  sharedSchemas: FlowMapSchemaUsage[];
  agentConnections: FlowMapEdge[];
  flowDependencies: Record<string, FlowDependency>;
  generatedAt: number;
}

export interface FlowMapEdge {
  from: string;                  // "domain/flow"
  to: string;                   // "domain/flow"
  via?: string;                  // Event name or ref
  type: 'event' | 'sub_flow' | 'orchestrator_manages' | 'handoff';
  mode?: string;                 // For handoffs: transfer/consult/collaborate
}

export interface FlowMapSchemaUsage {
  schema: string;
  usedBy: string[];              // "domain/flow" paths
}

export interface FlowDependency {
  dependsOn: string[];           // "domain/flow" paths
  dependedOnBy: string[];
  schemas: string[];
  eventsIn: string[];
  eventsOut: string[];
}

// Layer 5: Implementation Status
export interface ImplementationStatus {
  overview: {
    totalFlows: number;
    implemented: number;
    pending: number;
    stale: number;
  };
  flows: Record<string, FlowStatus>;
  schemas: Record<string, SchemaStatus>;
  generatedAt: number;
}

export interface FlowStatus {
  status: 'implemented' | 'pending' | 'stale';
  codeFiles: string[];
  lastGenerated?: string;        // ISO timestamp
  specChangedSince: boolean;
  changesSince?: string[];       // Human-readable change descriptions
}

export interface SchemaStatus {
  status: 'implemented' | 'pending' | 'stale';
  migration?: string;
  specChangedSince: boolean;
}

// --- Memory Panel State ---

export interface MemoryState {
  summary: ProjectSummary | null;
  specIndex: SpecIndex | null;
  decisions: Decision[];
  flowMap: FlowMap | null;
  implementationStatus: ImplementationStatus | null;
  memoryPanelOpen: boolean;
  isRefreshing: boolean;
}
```

**File: `src/stores/memory-store.ts`**
```typescript
import { create } from 'zustand';
import type {
  MemoryState, ProjectSummary, SpecIndex, Decision,
  FlowMap, ImplementationStatus, FlowDependency,
} from '../types/memory';
import { invoke } from '@tauri-apps/api/core';
import { nanoid } from 'nanoid';

interface MemoryActions {
  // Panel
  toggleMemoryPanel: () => void;

  // Load all memory layers from .ddd/memory/
  loadMemory: (projectPath: string) => Promise<void>;

  // Refresh auto-generated layers (1, 2, 4, 5)
  refreshMemory: (projectPath: string) => Promise<void>;

  // Regenerate project summary using LLM
  regenerateSummary: (projectPath: string) => Promise<void>;

  // Decision log
  addDecision: (decision: Omit<Decision, 'id'>) => void;
  editDecision: (id: string, updates: Partial<Decision>) => void;
  removeDecision: (id: string) => void;
  saveDecisions: (projectPath: string) => Promise<void>;

  // Contextual getters
  getFlowDependencies: (flowId: string) => FlowDependency | null;
  getRelevantDecisions: (domainId?: string, flowId?: string) => Decision[];
  getFlowStatus: (flowId: string) => FlowStatus | null;
}

export const useMemoryStore = create<MemoryState & MemoryActions>((set, get) => ({
  summary: null,
  specIndex: null,
  decisions: [],
  flowMap: null,
  implementationStatus: null,
  memoryPanelOpen: false,
  isRefreshing: false,

  toggleMemoryPanel: () => set(s => ({ memoryPanelOpen: !s.memoryPanelOpen })),

  loadMemory: async (projectPath: string) => {
    try {
      const [summary, specIndex, decisions, flowMap, status] = await Promise.all([
        invoke<ProjectSummary | null>('load_memory_layer', { projectPath, layer: 'summary' }),
        invoke<SpecIndex | null>('load_memory_layer', { projectPath, layer: 'spec-index' }),
        invoke<Decision[]>('load_memory_layer', { projectPath, layer: 'decisions' }),
        invoke<FlowMap | null>('load_memory_layer', { projectPath, layer: 'flow-map' }),
        invoke<ImplementationStatus | null>('load_memory_layer', { projectPath, layer: 'status' }),
      ]);

      set({
        summary,
        specIndex,
        decisions: decisions ?? [],
        flowMap,
        implementationStatus: status,
      });
    } catch (err) {
      console.error('Failed to load project memory:', err);
    }
  },

  refreshMemory: async (projectPath: string) => {
    set({ isRefreshing: true });

    try {
      // Rebuild auto-generated layers from specs
      const [specIndex, flowMap, status] = await Promise.all([
        invoke<SpecIndex>('build_spec_index', { projectPath }),
        invoke<FlowMap>('build_flow_map', { projectPath }),
        invoke<ImplementationStatus>('build_implementation_status', { projectPath }),
      ]);

      set({
        specIndex,
        flowMap,
        implementationStatus: status,
        isRefreshing: false,
      });

      // Save to disk
      await Promise.all([
        invoke('save_memory_layer', { projectPath, layer: 'spec-index', data: specIndex }),
        invoke('save_memory_layer', { projectPath, layer: 'flow-map', data: flowMap }),
        invoke('save_memory_layer', { projectPath, layer: 'status', data: status }),
      ]);
    } catch (err) {
      console.error('Failed to refresh memory:', err);
      set({ isRefreshing: false });
    }
  },

  regenerateSummary: async (projectPath: string) => {
    set({ isRefreshing: true });

    try {
      // Send all specs to LLM to generate project summary
      const summary = await invoke<ProjectSummary>('generate_project_summary', { projectPath });
      set({ summary, isRefreshing: false });

      await invoke('save_memory_layer', { projectPath, layer: 'summary', data: summary });
    } catch (err) {
      console.error('Failed to regenerate summary:', err);
      set({ isRefreshing: false });
    }
  },

  addDecision: (decision) => {
    const newDecision: Decision = { ...decision, id: nanoid() };
    set(s => ({ decisions: [...s.decisions, newDecision] }));
  },

  editDecision: (id, updates) => {
    set(s => ({
      decisions: s.decisions.map(d => d.id === id ? { ...d, ...updates } : d),
    }));
  },

  removeDecision: (id) => {
    set(s => ({ decisions: s.decisions.filter(d => d.id !== id) }));
  },

  saveDecisions: async (projectPath: string) => {
    await invoke('save_memory_layer', {
      projectPath,
      layer: 'decisions',
      data: get().decisions,
    });
  },

  getFlowDependencies: (flowId: string) => {
    return get().flowMap?.flowDependencies[flowId] ?? null;
  },

  getRelevantDecisions: (domainId?: string, flowId?: string) => {
    const decisions = get().decisions;
    if (!domainId && !flowId) return decisions;

    return decisions.filter(d =>
      d.affected.some(ref =>
        (domainId && ref.includes(domainId)) ||
        (flowId && ref.includes(flowId))
      )
    );
  },

  getFlowStatus: (flowId: string) => {
    return get().implementationStatus?.flows[flowId] ?? null;
  },
}));
```

**File: `src/utils/memory-builder.ts`**
```typescript
import type { SpecIndex, SpecIndexDomain, SpecIndexFlow, FlowMap, FlowMapEdge, FlowDependency } from '../types/memory';

/**
 * Build SpecIndex (Layer 2) by scanning all parsed domain configs and flow files.
 * Called on project load and when specs change.
 */
export function buildSpecIndex(
  systemConfig: any,
  domainConfigs: Record<string, any>,
  flowFiles: Record<string, any>,
  schemas: Record<string, any>,
  errorCodes: string[]
): SpecIndex {
  const domains: Record<string, SpecIndexDomain> = {};

  for (const [domainId, domain] of Object.entries(domainConfigs)) {
    const flows: Record<string, SpecIndexFlow> = {};

    for (const flowEntry of (domain as any).flows ?? []) {
      const flowYaml = flowFiles[`${domainId}/${flowEntry.id}`];
      if (!flowYaml) continue;

      const flow: SpecIndexFlow = {
        type: flowYaml.flow?.type ?? 'traditional',
        trigger: {
          type: flowYaml.trigger?.type ?? 'unknown',
          method: flowYaml.trigger?.method,
          path: flowYaml.trigger?.path,
          schedule: flowYaml.trigger?.schedule,
          event: flowYaml.trigger?.event,
        },
        nodes: (flowYaml.nodes ?? []).map((n: any) => n.id),
        publishesEvents: extractPublishedEvents(flowYaml),
        consumesEvents: flowYaml.trigger?.type === 'event' ? [flowYaml.trigger.event] : [],
        schemasUsed: extractSchemaRefs(flowYaml),
        errorCodes: extractErrorCodes(flowYaml),
      };

      // Agent-specific fields
      if (flow.type === 'agent') {
        flow.agentModel = flowYaml.agent_config?.model;
        flow.tools = (flowYaml.tools ?? []).map((t: any) => t.id);
        flow.guardrails = (flowYaml.guardrails ?? []).map((g: any) => g.id);
      }

      flows[flowEntry.id] = flow;
    }

    domains[domainId] = {
      description: (domain as any).description ?? '',
      flows,
    };
  }

  // Build event map
  const events = buildEventList(domains);

  return {
    domains,
    schemas: Object.fromEntries(
      Object.entries(schemas).map(([name, schema]) => [
        name,
        { fields: Object.keys((schema as any).fields ?? (schema as any).properties ?? {}) },
      ])
    ),
    events,
    sharedErrorCodes: errorCodes,
    generatedAt: Date.now(),
  };
}

/**
 * Build FlowMap (Layer 4) from SpecIndex — derive the dependency graph.
 */
export function buildFlowMap(specIndex: SpecIndex): FlowMap {
  const events: FlowMapEdge[] = [];
  const subFlows: FlowMapEdge[] = [];
  const agentConnections: FlowMapEdge[] = [];
  const sharedSchemas: { schema: string; usedBy: string[] }[] = [];
  const flowDependencies: Record<string, FlowDependency> = {};

  // Initialize dependencies for all flows
  for (const [domainId, domain] of Object.entries(specIndex.domains)) {
    for (const flowId of Object.keys(domain.flows)) {
      const fullId = `${domainId}/${flowId}`;
      flowDependencies[fullId] = {
        dependsOn: [],
        dependedOnBy: [],
        schemas: domain.flows[flowId].schemasUsed,
        eventsIn: domain.flows[flowId].consumesEvents,
        eventsOut: domain.flows[flowId].publishesEvents,
      };
    }
  }

  // Build event edges
  for (const event of specIndex.events) {
    for (const [domainId, domain] of Object.entries(specIndex.domains)) {
      for (const [flowId, flow] of Object.entries(domain.flows)) {
        const fullId = `${domainId}/${flowId}`;

        if (flow.publishesEvents.includes(event.name)) {
          // Find consumers
          for (const [cDomainId, cDomain] of Object.entries(specIndex.domains)) {
            for (const [cFlowId, cFlow] of Object.entries(cDomain.flows)) {
              if (cFlow.consumesEvents.includes(event.name)) {
                const cFullId = `${cDomainId}/${cFlowId}`;
                events.push({
                  from: fullId,
                  to: cFullId,
                  via: event.name,
                  type: 'event',
                });
                flowDependencies[cFullId]?.dependsOn.push(fullId);
                flowDependencies[fullId]?.dependedOnBy.push(cFullId);
              }
            }
          }
        }
      }
    }
  }

  // Build shared schema map
  const schemaUsage: Record<string, string[]> = {};
  for (const [domainId, domain] of Object.entries(specIndex.domains)) {
    for (const [flowId, flow] of Object.entries(domain.flows)) {
      for (const schema of flow.schemasUsed) {
        if (!schemaUsage[schema]) schemaUsage[schema] = [];
        schemaUsage[schema].push(`${domainId}/${flowId}`);
      }
    }
  }
  for (const [schema, usedBy] of Object.entries(schemaUsage)) {
    if (usedBy.length > 1) {
      sharedSchemas.push({ schema, usedBy });
    }
  }

  return {
    events,
    subFlows,
    sharedSchemas,
    agentConnections,
    flowDependencies,
    generatedAt: Date.now(),
  };
}

// --- Helpers ---

function extractPublishedEvents(flowYaml: any): string[] {
  const events: string[] = [];
  for (const node of flowYaml.nodes ?? []) {
    if (node.type === 'event' && node.spec?.event) {
      events.push(node.spec.event);
    }
  }
  return events;
}

function extractSchemaRefs(flowYaml: any): string[] {
  const refs: string[] = [];
  const yamlStr = JSON.stringify(flowYaml);
  const matches = yamlStr.matchAll(/\$ref:\/schemas\/(\w+)/g);
  for (const match of matches) {
    refs.push(match[1]);
  }
  // Also check model references in data_store nodes
  for (const node of flowYaml.nodes ?? []) {
    if (node.spec?.model && typeof node.spec.model === 'string') {
      refs.push(node.spec.model);
    }
  }
  return [...new Set(refs)];
}

function extractErrorCodes(flowYaml: any): string[] {
  const codes: string[] = [];
  const yamlStr = JSON.stringify(flowYaml);
  const matches = yamlStr.matchAll(/"error_code"\s*:\s*"(\w+)"/g);
  for (const match of matches) {
    codes.push(match[1]);
  }
  return [...new Set(codes)];
}

function buildEventList(domains: Record<string, SpecIndexDomain>) {
  const eventMap: Record<string, { publisher: string; consumers: string[] }> = {};

  for (const [domainId, domain] of Object.entries(domains)) {
    for (const flow of Object.values(domain.flows)) {
      for (const event of flow.publishesEvents) {
        if (!eventMap[event]) eventMap[event] = { publisher: domainId, consumers: [] };
        eventMap[event].publisher = domainId;
      }
      for (const event of flow.consumesEvents) {
        if (!eventMap[event]) eventMap[event] = { publisher: '', consumers: [] };
        if (!eventMap[event].consumers.includes(domainId)) {
          eventMap[event].consumers.push(domainId);
        }
      }
    }
  }

  return Object.entries(eventMap).map(([name, info]) => ({
    name,
    publisher: info.publisher,
    consumers: info.consumers,
  }));
}
```

**Update `src/utils/llm-context.ts` — integrate memory layers:**
```typescript
import { useProjectStore } from '../stores/project-store';
import { useSheetStore } from '../stores/sheet-store';
import { useFlowStore } from '../stores/flow-store';
import { useMemoryStore } from '../stores/memory-store';
import type { LLMContext } from '../types/llm';

/**
 * Build the context object sent with every LLM request.
 * Integrates all 5 Project Memory layers with priority-based budgeting.
 */
export function buildLLMContext(): LLMContext {
  const project = useProjectStore.getState();
  const sheet = useSheetStore.getState();
  const flow = useFlowStore.getState();
  const memory = useMemoryStore.getState();

  const context: LLMContext = {
    // Priority 1: Project Summary (Layer 1) — always included
    projectSummary: memory.summary?.content ?? '',

    system: {
      name: project.systemConfig?.name ?? '',
      techStack: project.systemConfig?.tech_stack ?? {},
      domains: project.domains?.map(d => d.name) ?? [],
    },
  };

  // Priority 2: Current domain + related domains (Layer 4)
  if (sheet.current.level !== 'system' && sheet.current.domainId) {
    const domain = project.domainConfigs?.[sheet.current.domainId];
    if (domain) {
      context.currentDomain = {
        name: domain.name,
        flows: domain.flows.map(f => f.id),
        publishesEvents: domain.publishes_events.map(e => e.event),
        consumesEvents: domain.consumes_events.map(e => e.event),
      };
    }
  }

  // Priority 3: Current flow + connected flows (Layer 4)
  if (sheet.current.level === 'flow' && sheet.current.flowId) {
    const flowId = sheet.current.flowId;
    const domainId = sheet.current.domainId;
    const fullFlowId = domainId ? `${domainId}/${flowId}` : flowId;

    context.currentFlow = {
      id: flowId,
      type: flow.flowType ?? 'traditional',
      yaml: flow.toYaml(),
    };

    // Connected flows from Flow Map (Layer 4)
    const deps = memory.getFlowDependencies(fullFlowId);
    if (deps) {
      context.connectedFlows = {
        upstream: deps.dependsOn,
        downstream: deps.dependedOnBy,
        sharedSchemas: deps.schemas,
        eventsIn: deps.eventsIn,
        eventsOut: deps.eventsOut,
      };
    }

    // Implementation status (Layer 5)
    const status = memory.getFlowStatus(fullFlowId);
    if (status) {
      context.implementationStatus = {
        status: status.status,
        specChangedSince: status.specChangedSince,
        changesSince: status.changesSince,
      };
    }
  }

  // Selected nodes
  const selectedNodes = flow.getSelectedNodes?.();
  if (selectedNodes?.length) {
    context.selectedNodes = selectedNodes.map(n => ({
      id: n.id,
      type: n.type,
      spec: n.spec,
    }));
  }

  // Priority 4: Relevant decisions (Layer 3)
  const relevantDecisions = memory.getRelevantDecisions(
    sheet.current.domainId,
    sheet.current.flowId
  );
  if (relevantDecisions.length) {
    context.relevantDecisions = relevantDecisions.map(d =>
      `${d.title}: ${d.rationale}`
    );
  }

  // Error codes and schemas
  context.errorCodes = project.errorCodes ?? [];
  context.schemas = project.schemas ?? {};

  return context;
}
```

**File: `src/components/MemoryPanel/MemoryPanel.tsx`**
```tsx
import { useMemoryStore } from '../../stores/memory-store';
import { useProjectStore } from '../../stores/project-store';
import { useSheetStore } from '../../stores/sheet-store';
import { SummaryCard } from './SummaryCard';
import { StatusList } from './StatusList';
import { DecisionList } from './DecisionList';
import { FlowDependencies } from './FlowDependencies';
import { X, Brain, RefreshCw } from 'lucide-react';

export function MemoryPanel() {
  const {
    summary, implementationStatus, decisions, flowMap,
    memoryPanelOpen, isRefreshing,
    toggleMemoryPanel, refreshMemory, regenerateSummary,
  } = useMemoryStore();
  const { projectPath } = useProjectStore();
  const { current } = useSheetStore();

  if (!memoryPanelOpen) return null;

  return (
    <div className="w-80 border-l border-gray-200 bg-white flex flex-col h-full overflow-hidden">
      {/* Header */}
      <div className="flex items-center justify-between px-4 py-3 border-b border-gray-200">
        <div className="flex items-center gap-2">
          <Brain className="w-4 h-4 text-indigo-500" />
          <span className="font-medium text-sm">Project Memory</span>
        </div>
        <div className="flex items-center gap-1">
          <button
            onClick={() => refreshMemory(projectPath)}
            disabled={isRefreshing}
            className="p-1 text-gray-400 hover:text-gray-600 disabled:animate-spin"
            title="Refresh memory"
          >
            <RefreshCw className="w-3.5 h-3.5" />
          </button>
          <button onClick={toggleMemoryPanel} className="p-1 text-gray-400 hover:text-gray-600">
            <X className="w-4 h-4" />
          </button>
        </div>
      </div>

      {/* Scrollable content */}
      <div className="flex-1 overflow-y-auto px-4 py-3 space-y-4">
        {/* Layer 1: Summary */}
        <SummaryCard
          summary={summary}
          status={implementationStatus?.overview}
          onRegenerate={() => regenerateSummary(projectPath)}
          isRefreshing={isRefreshing}
        />

        {/* Layer 5: Implementation Status */}
        <StatusList status={implementationStatus} />

        {/* Layer 3: Decisions */}
        <DecisionList decisions={decisions} />

        {/* Layer 4: Flow Dependencies (shown on L3) */}
        {current.level === 'flow' && current.flowId && current.domainId && (
          <FlowDependencies
            flowId={`${current.domainId}/${current.flowId}`}
            flowMap={flowMap}
          />
        )}
      </div>
    </div>
  );
}
```

**File: `src/components/MemoryPanel/SummaryCard.tsx`**
```tsx
import type { ProjectSummary } from '../../types/memory';
import { FileText, RefreshCw } from 'lucide-react';

interface Props {
  summary: ProjectSummary | null;
  status?: { totalFlows: number; implemented: number; pending: number; stale: number };
  onRegenerate: () => void;
  isRefreshing: boolean;
}

export function SummaryCard({ summary, status, onRegenerate, isRefreshing }: Props) {
  return (
    <div className="border border-gray-200 rounded-lg p-3">
      <div className="flex items-center justify-between mb-2">
        <div className="flex items-center gap-1.5 text-xs font-medium text-gray-500">
          <FileText className="w-3 h-3" />
          Summary
        </div>
        <button
          onClick={onRegenerate}
          disabled={isRefreshing}
          className="text-xs text-indigo-500 hover:text-indigo-700 flex items-center gap-1"
        >
          <RefreshCw className={`w-3 h-3 ${isRefreshing ? 'animate-spin' : ''}`} />
          Regenerate
        </button>
      </div>

      {summary ? (
        <>
          <div className="text-sm text-gray-700 line-clamp-4 whitespace-pre-wrap">
            {summary.content.split('\n').slice(0, 4).join('\n')}
          </div>
          {summary.stale && (
            <div className="mt-2 text-xs text-amber-600 bg-amber-50 px-2 py-1 rounded">
              Summary may be outdated — specs changed since generation
            </div>
          )}
        </>
      ) : (
        <div className="text-xs text-gray-400">
          No summary generated yet. Click Regenerate.
        </div>
      )}

      {status && (
        <div className="mt-2 flex gap-3 text-xs">
          <span className="text-gray-500">{status.totalFlows} flows</span>
          <span className="text-green-600">✓ {status.implemented}</span>
          <span className="text-gray-400">◌ {status.pending}</span>
          {status.stale > 0 && (
            <span className="text-amber-600">⚠ {status.stale} stale</span>
          )}
        </div>
      )}
    </div>
  );
}
```

**File: `src/components/MemoryPanel/StatusList.tsx`**
```tsx
import type { ImplementationStatus } from '../../types/memory';
import { CheckCircle, Circle, AlertTriangle } from 'lucide-react';

interface Props {
  status: ImplementationStatus | null;
}

export function StatusList({ status }: Props) {
  if (!status) return null;

  const entries = Object.entries(status.flows).sort(([, a], [, b]) => {
    const order = { stale: 0, pending: 1, implemented: 2 };
    return order[a.status] - order[b.status];
  });

  return (
    <div className="border border-gray-200 rounded-lg p-3">
      <div className="text-xs font-medium text-gray-500 mb-2">Implementation Status</div>
      <div className="space-y-1.5 max-h-48 overflow-y-auto">
        {entries.map(([flowId, flowStatus]) => (
          <div key={flowId} className="flex items-center gap-2 text-xs">
            {flowStatus.status === 'implemented' && (
              <CheckCircle className="w-3.5 h-3.5 text-green-500 flex-shrink-0" />
            )}
            {flowStatus.status === 'pending' && (
              <Circle className="w-3.5 h-3.5 text-gray-300 flex-shrink-0" />
            )}
            {flowStatus.status === 'stale' && (
              <AlertTriangle className="w-3.5 h-3.5 text-amber-500 flex-shrink-0" />
            )}
            <span className="text-gray-700 truncate flex-1" title={flowId}>
              {flowId}
            </span>
            {flowStatus.status === 'stale' && flowStatus.changesSince && (
              <span className="text-amber-500" title={flowStatus.changesSince.join(', ')}>
                {flowStatus.changesSince.length} changes
              </span>
            )}
          </div>
        ))}
      </div>
    </div>
  );
}
```

**File: `src/components/MemoryPanel/DecisionList.tsx`**
```tsx
import { useState } from 'react';
import { useMemoryStore } from '../../stores/memory-store';
import { useProjectStore } from '../../stores/project-store';
import { DecisionForm } from './DecisionForm';
import type { Decision } from '../../types/memory';
import { Plus, Pencil, Trash2, BookOpen } from 'lucide-react';

interface Props {
  decisions: Decision[];
}

export function DecisionList({ decisions }: Props) {
  const { addDecision, removeDecision, saveDecisions } = useMemoryStore();
  const { projectPath } = useProjectStore();
  const [showForm, setShowForm] = useState(false);

  const handleAdd = (decision: Omit<Decision, 'id'>) => {
    addDecision(decision);
    saveDecisions(projectPath);
    setShowForm(false);
  };

  const handleRemove = (id: string) => {
    removeDecision(id);
    saveDecisions(projectPath);
  };

  return (
    <div className="border border-gray-200 rounded-lg p-3">
      <div className="flex items-center justify-between mb-2">
        <div className="flex items-center gap-1.5 text-xs font-medium text-gray-500">
          <BookOpen className="w-3 h-3" />
          Decisions ({decisions.length})
        </div>
        <button
          onClick={() => setShowForm(true)}
          className="text-xs text-indigo-500 hover:text-indigo-700 flex items-center gap-1"
        >
          <Plus className="w-3 h-3" /> Add
        </button>
      </div>

      <div className="space-y-2 max-h-48 overflow-y-auto">
        {decisions.map(d => (
          <div key={d.id} className="bg-gray-50 rounded px-2 py-1.5 group">
            <div className="flex items-start justify-between">
              <div className="text-xs font-medium text-gray-700">{d.title}</div>
              <button
                onClick={() => handleRemove(d.id)}
                className="opacity-0 group-hover:opacity-100 text-gray-400 hover:text-red-500 transition-opacity"
              >
                <Trash2 className="w-3 h-3" />
              </button>
            </div>
            <div className="text-xs text-gray-500 mt-0.5 line-clamp-2">{d.rationale}</div>
            <div className="text-xs text-gray-400 mt-0.5">{d.date}</div>
          </div>
        ))}
      </div>

      {showForm && (
        <DecisionForm onSave={handleAdd} onCancel={() => setShowForm(false)} />
      )}
    </div>
  );
}
```

**File: `src/components/MemoryPanel/DecisionForm.tsx`**
```tsx
import { useState } from 'react';
import type { Decision } from '../../types/memory';

interface Props {
  onSave: (decision: Omit<Decision, 'id'>) => void;
  onCancel: () => void;
}

export function DecisionForm({ onSave, onCancel }: Props) {
  const [title, setTitle] = useState('');
  const [rationale, setRationale] = useState('');
  const [affected, setAffected] = useState('');

  const handleSubmit = () => {
    if (!title.trim() || !rationale.trim()) return;
    onSave({
      date: new Date().toISOString().slice(0, 10),
      title: title.trim(),
      rationale: rationale.trim(),
      affected: affected.split(',').map(s => s.trim()).filter(Boolean),
      author: 'user',
    });
  };

  return (
    <div className="mt-2 p-2 bg-indigo-50 rounded-lg border border-indigo-200 space-y-2">
      <input
        value={title}
        onChange={e => setTitle(e.target.value)}
        placeholder="Decision title"
        className="w-full text-xs px-2 py-1 border rounded"
      />
      <textarea
        value={rationale}
        onChange={e => setRationale(e.target.value)}
        placeholder="Why this decision was made…"
        rows={3}
        className="w-full text-xs px-2 py-1 border rounded resize-none"
      />
      <input
        value={affected}
        onChange={e => setAffected(e.target.value)}
        placeholder="Affected (comma-separated: domain, flow, file)"
        className="w-full text-xs px-2 py-1 border rounded"
      />
      <div className="flex gap-2 justify-end">
        <button onClick={onCancel} className="text-xs px-2 py-1 text-gray-500 hover:text-gray-700">
          Cancel
        </button>
        <button
          onClick={handleSubmit}
          disabled={!title.trim() || !rationale.trim()}
          className="text-xs px-2 py-1 bg-indigo-500 text-white rounded hover:bg-indigo-600 disabled:opacity-50"
        >
          Save
        </button>
      </div>
    </div>
  );
}
```

**File: `src/components/MemoryPanel/FlowDependencies.tsx`**
```tsx
import type { FlowMap } from '../../types/memory';
import { ArrowDown, ArrowUp, Layers, Zap } from 'lucide-react';

interface Props {
  flowId: string;
  flowMap: FlowMap | null;
}

export function FlowDependencies({ flowId, flowMap }: Props) {
  if (!flowMap) return null;

  const deps = flowMap.flowDependencies[flowId];
  if (!deps) return null;

  return (
    <div className="border border-gray-200 rounded-lg p-3">
      <div className="text-xs font-medium text-gray-500 mb-2">Flow Dependencies</div>

      {/* Upstream */}
      {deps.dependsOn.length > 0 && (
        <div className="mb-2">
          <div className="flex items-center gap-1 text-xs text-gray-400 mb-1">
            <ArrowDown className="w-3 h-3" /> Depends on
          </div>
          {deps.dependsOn.map(id => (
            <div key={id} className="text-xs text-gray-700 ml-4">{id}</div>
          ))}
        </div>
      )}

      {/* Downstream */}
      {deps.dependedOnBy.length > 0 && (
        <div className="mb-2">
          <div className="flex items-center gap-1 text-xs text-gray-400 mb-1">
            <ArrowUp className="w-3 h-3" /> Depended on by
          </div>
          {deps.dependedOnBy.map(id => (
            <div key={id} className="text-xs text-gray-700 ml-4">{id}</div>
          ))}
        </div>
      )}

      {/* Schemas */}
      {deps.schemas.length > 0 && (
        <div className="mb-2">
          <div className="flex items-center gap-1 text-xs text-gray-400 mb-1">
            <Layers className="w-3 h-3" /> Schemas
          </div>
          <div className="flex flex-wrap gap-1 ml-4">
            {deps.schemas.map(s => (
              <span key={s} className="text-xs bg-gray-100 px-1.5 py-0.5 rounded">{s}</span>
            ))}
          </div>
        </div>
      )}

      {/* Events */}
      {(deps.eventsOut.length > 0 || deps.eventsIn.length > 0) && (
        <div>
          <div className="flex items-center gap-1 text-xs text-gray-400 mb-1">
            <Zap className="w-3 h-3" /> Events
          </div>
          {deps.eventsOut.map(e => (
            <div key={e} className="text-xs text-gray-700 ml-4">→ {e} (publishes)</div>
          ))}
          {deps.eventsIn.map(e => (
            <div key={e} className="text-xs text-gray-700 ml-4">← {e} (consumes)</div>
          ))}
        </div>
      )}
    </div>
  );
}
```

**Add keyboard shortcut for Memory Panel in `src/App.tsx`:**
```typescript
// Add to the existing keyboard handler useEffect
// Cmd+M / Ctrl+M — Toggle Memory Panel
if (mod && e.key === 'm') {
  e.preventDefault();
  useMemoryStore.getState().toggleMemoryPanel();
}
```

**Trigger memory refresh on spec changes — add to `src/stores/project-store.ts`:**
```typescript
// After loading or saving any spec file, trigger memory refresh
import { useMemoryStore } from './memory-store';

// In loadProject action:
async loadProject(path: string) {
  // ... existing project loading logic ...

  // After all specs are loaded, build memory layers
  const memoryStore = useMemoryStore.getState();
  await memoryStore.loadMemory(path);

  // Check if auto-generated layers need refresh
  if (!memoryStore.specIndex || memoryStore.summary?.stale) {
    await memoryStore.refreshMemory(path);
  }
}

// In saveFlow action:
async saveFlow(flowId: string) {
  // ... existing save logic ...

  // Mark memory as potentially stale and refresh
  const memoryStore = useMemoryStore.getState();
  await memoryStore.refreshMemory(get().projectPath);
}
```

**Decision capture in Chat — update `src/stores/llm-store.ts`:**
```typescript
// After receiving an LLM response, check if it detected a design decision
// Pattern: LLM includes "💡 Decision detected:" in response

sendMessage: async (content: string) => {
  // ... existing send logic ...

  // After response received, check for decision capture
  if (response.includes('💡 Decision detected:')) {
    const decisionMatch = response.match(
      /💡 Decision detected:\n- Title: (.+)\n- Rationale: (.+)\n- Affected: (.+)/
    );
    if (decisionMatch) {
      // Add to decision log automatically
      useMemoryStore.getState().addDecision({
        date: new Date().toISOString().slice(0, 10),
        title: decisionMatch[1],
        rationale: decisionMatch[2],
        affected: decisionMatch[3].split(',').map(s => s.trim()),
        author: 'llm',
      });
    }
  }
}
```

---

### Claude Code Integration

**File: `src/types/implementation.ts`**
```typescript
/** Panel states */
export type ImplementationPanelState =
  | 'idle'
  | 'prompt_ready'
  | 'running'
  | 'done'
  | 'failed';

/** A single flow's implementation status in the queue */
export interface QueueItem {
  flowId: string;          // e.g. "api/user-register"
  domain: string;
  flowName: string;
  status: 'pending' | 'stale' | 'implemented';
  testResult?: TestSummary;
  staleChanges?: string[]; // Human-readable list of spec changes
  selected: boolean;       // For batch operations
}

/** Prompt built for Claude Code */
export interface BuiltPrompt {
  flowId: string;
  domain: string;
  content: string;         // The full prompt text
  specFiles: string[];     // Files referenced in the prompt
  mode: 'new' | 'update';  // New implementation or update to existing
  agentFlow: boolean;       // Whether this is an agent flow
  editedByUser: boolean;    // Whether user has modified the prompt
}

/** Mapping entry stored in .ddd/mapping.yaml */
export interface FlowMapping {
  spec: string;            // Path to spec file
  spec_hash: string;       // SHA-256 of spec at implementation time
  files: string[];         // Generated code files
  implemented_at: string;  // ISO timestamp
  test_results?: TestSummary;
}

/** Test execution results */
export interface TestSummary {
  total: number;
  passed: number;
  failed: number;
  errors: number;
  duration_ms: number;
  tests: TestCase[];
}

export interface TestCase {
  name: string;
  status: 'passed' | 'failed' | 'error' | 'skipped';
  duration_ms: number;
  error_message?: string;
  error_detail?: string;
}

/** PTY session for embedded terminal */
export interface PtySession {
  id: string;
  running: boolean;
  exitCode?: number;
}

/** Stale detection result */
export interface StaleResult {
  flowId: string;
  currentHash: string;
  storedHash: string;
  isStale: boolean;
  changes: string[];       // Human-readable change descriptions
  cachedSpecPath?: string;  // Path to spec at implementation time
}

/** Implementation config from .ddd/config.yaml */
export interface ImplementationConfig {
  claude_code: {
    enabled: boolean;
    command: string;
    post_implement: {
      run_tests: boolean;
      run_lint: boolean;
      auto_commit: boolean;
      regenerate_claude_md: boolean;
    };
    prompt: {
      include_architecture: boolean;
      include_errors: boolean;
      include_schemas: 'all' | 'auto' | 'none';
    };
  };
  testing: {
    command: string;
    args: string[];
    scoped: boolean;
    scope_pattern: string;
    auto_run: boolean;
  };
  reconciliation: {
    auto_run: boolean;
    auto_accept_matching: boolean;
    notify_on_drift: boolean;
  };
}

/** Reconciliation report comparing code vs spec */
export interface ReconciliationReport {
  flowId: string;
  syncScore: number;           // 0-100
  reconciledAt: string;        // ISO timestamp
  matching: ReconciliationItem[];    // Spec = Code
  codeOnly: ReconciliationItem[];    // Code has, spec doesn't
  specOnly: ReconciliationItem[];    // Spec has, code doesn't
}

export interface ReconciliationItem {
  id: string;                  // Unique ID for this item
  description: string;         // Human-readable description
  category: 'endpoint' | 'validation' | 'error_handling' | 'middleware' | 'schema' | 'response' | 'logic' | 'other';
  codeLocation?: string;       // e.g. "src/domains/api/services.py:23"
  specLocation?: string;       // e.g. "nodes[2].spec.fields.email"
  severity: 'minor' | 'moderate' | 'significant';
  resolution?: 'accepted' | 'removed' | 'ignored';
  resolvedAt?: string;
}

/** Accepted deviation stored in mapping.yaml */
export interface AcceptedDeviation {
  description: string;
  codeLocation: string;
  acceptedAt: string;
}

/** Extended flow mapping with reconciliation data */
export interface FlowMappingExtended extends FlowMapping {
  sync_score?: number;
  last_reconciled_at?: string;
  accepted_deviations?: AcceptedDeviation[];
}
```

**File: `src/stores/implementation-store.ts`**
```typescript
import { create } from 'zustand';
import { invoke } from '@tauri-apps/api/core';
import type {
  ImplementationPanelState, QueueItem, BuiltPrompt, FlowMapping,
  TestSummary, PtySession, StaleResult, ImplementationConfig,
} from '../types/implementation';
import { buildPrompt } from '../utils/prompt-builder';
import { generateClaudeMd } from '../utils/claude-md-generator';

interface ImplementationState {
  // Panel
  panelOpen: boolean;
  panelState: ImplementationPanelState;
  togglePanel: () => void;

  // Prompt
  currentPrompt: BuiltPrompt | null;
  buildPromptForFlow: (flowId: string, domain: string) => Promise<void>;
  updatePromptContent: (content: string) => void;

  // Terminal (PTY)
  ptySession: PtySession | null;
  runClaudeCode: () => Promise<void>;
  stopClaudeCode: () => Promise<void>;
  sendToPty: (input: string) => Promise<void>;

  // Test runner
  testResults: TestSummary | null;
  isRunningTests: boolean;
  runTests: (flowId: string) => Promise<void>;
  fixFailingTest: (flowId: string, errorOutput: string) => Promise<void>;

  // Queue
  queue: QueueItem[];
  loadQueue: (projectPath: string) => Promise<void>;
  toggleQueueItem: (flowId: string) => void;
  selectAllPending: () => void;
  implementSelected: () => Promise<void>;

  // Stale detection
  staleResults: Record<string, StaleResult>;
  checkAllStale: (projectPath: string) => Promise<void>;
  checkFlowStale: (flowId: string, projectPath: string) => Promise<StaleResult>;

  // Mapping
  mappings: Record<string, FlowMapping>;
  loadMappings: (projectPath: string) => Promise<void>;
  updateMapping: (flowId: string, mapping: FlowMapping, projectPath: string) => Promise<void>;

  // Config
  config: ImplementationConfig | null;
  loadConfig: (projectPath: string) => Promise<void>;

  // CLAUDE.md
  regenerateClaudeMd: (projectPath: string) => Promise<void>;

  // Reconciliation
  reconciliationReport: ReconciliationReport | null;
  isReconciling: boolean;
  reconcileFlow: (flowId: string, projectPath: string) => Promise<void>;
  resolveReconciliationItem: (flowId: string, itemId: string, resolution: 'accepted' | 'removed' | 'ignored', projectPath: string) => Promise<void>;
  parseImplementationNotes: (terminalOutput: string) => string[];
}

export const useImplementationStore = create<ImplementationState>((set, get) => ({
  panelOpen: false,
  panelState: 'idle',
  currentPrompt: null,
  ptySession: null,
  testResults: null,
  reconciliationReport: null,
  isReconciling: false,
  isRunningTests: false,
  queue: [],
  staleResults: {},
  mappings: {},
  config: null,

  togglePanel: () => set(s => ({ panelOpen: !s.panelOpen })),

  buildPromptForFlow: async (flowId, domain) => {
    const prompt = await buildPrompt(flowId, domain);
    set({ currentPrompt: prompt, panelState: 'prompt_ready', panelOpen: true });
  },

  updatePromptContent: (content) => {
    const current = get().currentPrompt;
    if (current) {
      set({ currentPrompt: { ...current, content, editedByUser: true } });
    }
  },

  runClaudeCode: async () => {
    const prompt = get().currentPrompt;
    if (!prompt) return;

    set({ panelState: 'running' });

    try {
      // Spawn PTY with claude CLI
      const sessionId = await invoke<string>('pty_spawn', {
        command: get().config?.claude_code.command ?? 'claude',
        args: [prompt.content],
        cwd: await invoke<string>('get_project_path'),
      });

      set({ ptySession: { id: sessionId, running: true } });

      // Listen for PTY exit
      const exitCode = await invoke<number>('pty_wait', { sessionId });
      set(s => ({
        ptySession: s.ptySession ? { ...s.ptySession, running: false, exitCode } : null,
        panelState: exitCode === 0 ? 'done' : 'failed',
      }));

      // Post-implementation actions
      if (exitCode === 0) {
        const config = get().config;
        const flowId = prompt.flowId;
        const projectPath = await invoke<string>('get_project_path');

        // Update mapping
        const specHash = await invoke<string>('hash_file', { path: prompt.specFiles[prompt.specFiles.length - 1] });
        await get().updateMapping(flowId, {
          spec: prompt.specFiles[prompt.specFiles.length - 1],
          spec_hash: specHash,
          files: [], // Will be populated by git diff
          implemented_at: new Date().toISOString(),
        }, projectPath);

        // Cache spec at implementation time
        await invoke('cache_spec', { flowId, specPath: prompt.specFiles[prompt.specFiles.length - 1] });

        // Auto-run tests
        if (config?.claude_code.post_implement.run_tests) {
          await get().runTests(flowId);
        }

        // Regenerate CLAUDE.md
        if (config?.claude_code.post_implement.regenerate_claude_md) {
          await get().regenerateClaudeMd(projectPath);
        }

        // Auto-reconcile (reverse drift detection)
        if (config?.reconciliation?.auto_run) {
          await get().reconcileFlow(flowId, projectPath);
        }
      }
    } catch (err) {
      set({ panelState: 'failed' });
      console.error('Claude Code execution failed:', err);
    }
  },

  stopClaudeCode: async () => {
    const session = get().ptySession;
    if (session?.running) {
      await invoke('pty_kill', { sessionId: session.id });
      set(s => ({
        ptySession: s.ptySession ? { ...s.ptySession, running: false } : null,
        panelState: 'failed',
      }));
    }
  },

  sendToPty: async (input) => {
    const session = get().ptySession;
    if (session?.running) {
      await invoke('pty_write', { sessionId: session.id, data: input });
    }
  },

  runTests: async (flowId) => {
    set({ isRunningTests: true, testResults: null });
    try {
      const config = get().config;
      const projectPath = await invoke<string>('get_project_path');
      const results = await invoke<TestSummary>('run_tests', {
        command: config?.testing.command ?? 'pytest',
        args: config?.testing.args ?? [],
        scoped: config?.testing.scoped ?? true,
        scopePattern: config?.testing.scope_pattern?.replace('{flow_id}', flowId.split('/').pop() ?? ''),
        cwd: projectPath,
      });
      set({ testResults: results, isRunningTests: false });
    } catch (err) {
      set({ isRunningTests: false });
      console.error('Test execution failed:', err);
    }
  },

  fixFailingTest: async (flowId, errorOutput) => {
    // Build a fix prompt and send to Claude Code
    const fixPrompt = `This test is failing after implementing ${flowId}. Fix the implementation to match the spec.\n\nError output:\n${errorOutput}`;
    set({
      currentPrompt: {
        flowId,
        domain: flowId.split('/')[0],
        content: fixPrompt,
        specFiles: [],
        mode: 'update',
        agentFlow: false,
        editedByUser: false,
      },
      panelState: 'prompt_ready',
    });
  },

  loadQueue: async (projectPath) => {
    const mappings = get().mappings;
    const staleResults = get().staleResults;

    // Build queue from project flows + mapping status
    const allFlows = await invoke<Array<{ flowId: string; domain: string; name: string }>>('list_all_flows', { projectPath });

    const queue: QueueItem[] = allFlows.map(f => {
      const mapping = mappings[f.flowId];
      const stale = staleResults[f.flowId];

      let status: QueueItem['status'] = 'pending';
      if (mapping && !stale?.isStale) status = 'implemented';
      else if (mapping && stale?.isStale) status = 'stale';

      return {
        flowId: f.flowId,
        domain: f.domain,
        flowName: f.name,
        status,
        staleChanges: stale?.changes,
        selected: false,
      };
    });

    set({ queue });
  },

  toggleQueueItem: (flowId) =>
    set(s => ({
      queue: s.queue.map(q => q.flowId === flowId ? { ...q, selected: !q.selected } : q),
    })),

  selectAllPending: () =>
    set(s => ({
      queue: s.queue.map(q => ({
        ...q,
        selected: q.status === 'pending' || q.status === 'stale',
      })),
    })),

  implementSelected: async () => {
    const selected = get().queue.filter(q => q.selected);
    for (const item of selected) {
      await get().buildPromptForFlow(item.flowId, item.domain);
      await get().runClaudeCode();
    }
    // Reload queue after batch
    const projectPath = await invoke<string>('get_project_path');
    await get().loadQueue(projectPath);
  },

  checkAllStale: async (projectPath) => {
    const mappings = get().mappings;
    const results: Record<string, StaleResult> = {};

    for (const [flowId, mapping] of Object.entries(mappings)) {
      const currentHash = await invoke<string>('hash_file', { path: `${projectPath}/${mapping.spec}` });
      const isStale = currentHash !== mapping.spec_hash;

      let changes: string[] = [];
      if (isStale) {
        changes = await invoke<string[]>('diff_specs', {
          cachedPath: `${projectPath}/.ddd/cache/specs-at-implementation/${flowId.replace('/', '--')}.yaml`,
          currentPath: `${projectPath}/${mapping.spec}`,
        });
      }

      results[flowId] = {
        flowId,
        currentHash,
        storedHash: mapping.spec_hash,
        isStale,
        changes,
      };
    }

    set({ staleResults: results });
  },

  checkFlowStale: async (flowId, projectPath) => {
    const mapping = get().mappings[flowId];
    if (!mapping) return { flowId, currentHash: '', storedHash: '', isStale: false, changes: [] };

    const currentHash = await invoke<string>('hash_file', { path: `${projectPath}/${mapping.spec}` });
    const isStale = currentHash !== mapping.spec_hash;

    let changes: string[] = [];
    if (isStale) {
      changes = await invoke<string[]>('diff_specs', {
        cachedPath: `${projectPath}/.ddd/cache/specs-at-implementation/${flowId.replace('/', '--')}.yaml`,
        currentPath: `${projectPath}/${mapping.spec}`,
      });
    }

    const result = { flowId, currentHash, storedHash: mapping.spec_hash, isStale, changes };
    set(s => ({ staleResults: { ...s.staleResults, [flowId]: result } }));
    return result;
  },

  loadMappings: async (projectPath) => {
    try {
      const raw = await invoke<string>('read_file', { path: `${projectPath}/.ddd/mapping.yaml` });
      const parsed = (await import('yaml')).parse(raw);
      set({ mappings: parsed.flows ?? {} });
    } catch {
      set({ mappings: {} });
    }
  },

  updateMapping: async (flowId, mapping, projectPath) => {
    const mappings = { ...get().mappings, [flowId]: mapping };
    set({ mappings });
    const yaml = (await import('yaml')).stringify({ flows: mappings });
    await invoke('write_file', { path: `${projectPath}/.ddd/mapping.yaml`, content: yaml });
  },

  loadConfig: async (projectPath) => {
    try {
      const raw = await invoke<string>('read_file', { path: `${projectPath}/.ddd/config.yaml` });
      const parsed = (await import('yaml')).parse(raw);
      set({ config: parsed });
    } catch {
      // Use defaults
      set({
        config: {
          claude_code: {
            enabled: true,
            command: 'claude',
            post_implement: { run_tests: true, run_lint: false, auto_commit: false, regenerate_claude_md: true },
            prompt: { include_architecture: true, include_errors: true, include_schemas: 'auto' },
          },
          testing: {
            command: 'pytest',
            args: ['--tb=short', '-q'],
            scoped: true,
            scope_pattern: 'tests/**/test_{flow_id}*',
            auto_run: true,
          },
          reconciliation: {
            auto_run: true,
            auto_accept_matching: true,
            notify_on_drift: true,
          },
        },
      });
    }
  },

  regenerateClaudeMd: async (projectPath) => {
    const content = await generateClaudeMd(projectPath);
    await invoke('write_file', { path: `${projectPath}/CLAUDE.md`, content });
  },

  reconcileFlow: async (flowId, projectPath) => {
    set({ isReconciling: true, reconciliationReport: null });

    try {
      const mapping = get().mappings[flowId];
      if (!mapping) throw new Error(`No mapping found for ${flowId}`);

      // Read the spec and all implementation files
      const specContent = await invoke<string>('read_file', { path: `${projectPath}/${mapping.spec}` });
      const codeContents: Record<string, string> = {};
      for (const file of mapping.files) {
        try {
          codeContents[file] = await invoke<string>('read_file', { path: `${projectPath}/${file}` });
        } catch { /* file may have been deleted */ }
      }

      // Send to Design Assistant LLM for comparison
      const { useLLMStore } = await import('./llm-store');
      const reconciliationPrompt = buildReconciliationPrompt(flowId, specContent, codeContents);
      const response = await useLLMStore.getState().sendRawMessage(reconciliationPrompt, 'review_flow');

      // Parse LLM response into structured report
      const report = parseReconciliationResponse(flowId, response);
      set({ reconciliationReport: report, isReconciling: false });

      // Update mapping with sync score
      const updatedMapping = { ...mapping, sync_score: report.syncScore, last_reconciled_at: new Date().toISOString() };
      await get().updateMapping(flowId, updatedMapping, projectPath);
    } catch (err) {
      set({ isReconciling: false });
      console.error('Reconciliation failed:', err);
    }
  },

  resolveReconciliationItem: async (flowId, itemId, resolution, projectPath) => {
    const report = get().reconciliationReport;
    if (!report) return;

    // Find and update the item
    const allItems = [...report.codeOnly, ...report.specOnly];
    const item = allItems.find(i => i.id === itemId);
    if (!item) return;

    item.resolution = resolution;
    item.resolvedAt = new Date().toISOString();

    if (resolution === 'accepted' && report.codeOnly.some(i => i.id === itemId)) {
      // Accept into spec — use LLM to generate spec update
      const { useLLMStore } = await import('./llm-store');
      const mapping = get().mappings[flowId];
      const specContent = await invoke<string>('read_file', { path: `${projectPath}/${mapping.spec}` });
      const codeFile = item.codeLocation?.split(':')[0] ?? '';
      const codeContent = codeFile ? await invoke<string>('read_file', { path: `${projectPath}/${codeFile}` }) : '';

      const updatePrompt = `The following item exists in code but not in the spec. Generate an updated spec YAML that includes this item.\n\nItem: ${item.description}\nCode location: ${item.codeLocation}\nCode:\n${codeContent}\n\nCurrent spec:\n${specContent}\n\nReturn only the updated YAML.`;
      const updatedSpec = await useLLMStore.getState().sendRawMessage(updatePrompt, 'suggest_spec');
      await invoke('write_file', { path: `${projectPath}/${mapping.spec}`, content: updatedSpec });
    } else if (resolution === 'removed') {
      // Remove from code — build targeted Claude Code prompt
      const removePrompt = `Remove this from the implementation — it's not in the spec:\n${item.description}\nLocation: ${item.codeLocation}`;
      set({
        currentPrompt: {
          flowId,
          domain: flowId.split('/')[0],
          content: removePrompt,
          specFiles: [],
          mode: 'update',
          agentFlow: false,
          editedByUser: false,
        },
        panelState: 'prompt_ready',
      });
    } else if (resolution === 'ignored') {
      // Store as accepted deviation in mapping
      const mapping = get().mappings[flowId];
      const deviations = (mapping as any).accepted_deviations ?? [];
      deviations.push({
        description: item.description,
        codeLocation: item.codeLocation ?? '',
        acceptedAt: new Date().toISOString(),
      });
      await get().updateMapping(flowId, { ...mapping, accepted_deviations: deviations } as any, projectPath);
    }

    // Recalculate sync score
    const unresolved = [...report.codeOnly, ...report.specOnly].filter(i => !i.resolution);
    const total = report.matching.length + report.codeOnly.length + report.specOnly.length;
    const resolved = report.matching.length + (report.codeOnly.length + report.specOnly.length - unresolved.length);
    report.syncScore = total > 0 ? Math.round((resolved / total) * 100) : 100;

    set({ reconciliationReport: { ...report } });
  },

  parseImplementationNotes: (terminalOutput) => {
    // Extract "## Implementation Notes" section from Claude Code output
    const notesMatch = terminalOutput.match(/## Implementation Notes\n([\s\S]*?)(?=\n##|\n---|\Z)/);
    if (!notesMatch) return [];

    const notes = notesMatch[1];
    if (notes.includes('No deviations')) return [];

    // Extract bullet points
    return notes
      .split('\n')
      .filter(line => line.trim().startsWith('-') || line.trim().startsWith('*'))
      .map(line => line.trim().replace(/^[-*]\s*/, ''));
  },
}));

// --- Helper functions for reconciliation ---

function buildReconciliationPrompt(
  flowId: string,
  specContent: string,
  codeContents: Record<string, string>
): string {
  let prompt = `Compare this flow spec against its implementation code. For each item, determine if the code matches the spec, if the code has something the spec doesn't, or if the spec has something the code doesn't.\n\n`;
  prompt += `## Flow Spec (${flowId})\n\`\`\`yaml\n${specContent}\n\`\`\`\n\n`;
  prompt += `## Implementation Code\n`;
  for (const [file, content] of Object.entries(codeContents)) {
    prompt += `### ${file}\n\`\`\`\n${content}\n\`\`\`\n\n`;
  }
  prompt += `## Response Format\nRespond with JSON only:\n\`\`\`json\n{\n  "matching": [{"description": "...", "category": "..."}],\n  "codeOnly": [{"description": "...", "category": "...", "codeLocation": "file:line", "severity": "minor|moderate|significant"}],\n  "specOnly": [{"description": "...", "category": "...", "specLocation": "...", "severity": "minor|moderate|significant"}]\n}\n\`\`\``;
  return prompt;
}

function parseReconciliationResponse(flowId: string, response: string): ReconciliationReport {
  // Extract JSON from response
  const jsonMatch = response.match(/```json\n([\s\S]*?)\n```/) ?? response.match(/\{[\s\S]*\}/);
  const parsed = JSON.parse(jsonMatch?.[1] ?? jsonMatch?.[0] ?? '{}');

  const matching = (parsed.matching ?? []).map((item: any, i: number) => ({
    id: `match-${i}`,
    description: item.description,
    category: item.category ?? 'other',
    severity: 'minor' as const,
  }));

  const codeOnly = (parsed.codeOnly ?? []).map((item: any, i: number) => ({
    id: `code-${i}`,
    description: item.description,
    category: item.category ?? 'other',
    codeLocation: item.codeLocation,
    severity: item.severity ?? 'moderate',
  }));

  const specOnly = (parsed.specOnly ?? []).map((item: any, i: number) => ({
    id: `spec-${i}`,
    description: item.description,
    category: item.category ?? 'other',
    specLocation: item.specLocation,
    severity: item.severity ?? 'moderate',
  }));

  const total = matching.length + codeOnly.length + specOnly.length;
  const syncScore = total > 0 ? Math.round((matching.length / total) * 100) : 100;

  return {
    flowId,
    syncScore,
    reconciledAt: new Date().toISOString(),
    matching,
    codeOnly,
    specOnly,
  };
}
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
  md += `This project uses Diagram-Driven Development (DDD). All business logic\n`;
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

**File: `src/components/ImplementationPanel/ImplementationPanel.tsx`**
```tsx
import { useImplementationStore } from '../../stores/implementation-store';
import { PromptPreview } from './PromptPreview';
import { TerminalEmbed } from './TerminalEmbed';
import { TestResults } from './TestResults';
import { ImplementationQueue } from './ImplementationQueue';
import { StaleBanner } from './StaleBanner';
import { X, Terminal, Play, Square } from 'lucide-react';

export function ImplementationPanel() {
  const {
    panelOpen, panelState, currentPrompt, ptySession,
    testResults, isRunningTests, staleResults,
    togglePanel, runClaudeCode, stopClaudeCode, runTests,
  } = useImplementationStore();

  if (!panelOpen) return null;

  return (
    <div className="w-[480px] border-l border-gray-200 bg-white flex flex-col h-full">
      {/* Header */}
      <div className="flex items-center justify-between px-4 py-3 border-b border-gray-200">
        <div className="flex items-center gap-2">
          <Terminal className="w-4 h-4 text-blue-500" />
          <span className="font-medium text-sm">Implementation</span>
          {panelState === 'running' && (
            <span className="text-xs bg-blue-100 text-blue-700 px-1.5 py-0.5 rounded">Running</span>
          )}
          {panelState === 'done' && (
            <span className="text-xs bg-green-100 text-green-700 px-1.5 py-0.5 rounded">Done</span>
          )}
          {panelState === 'failed' && (
            <span className="text-xs bg-red-100 text-red-700 px-1.5 py-0.5 rounded">Failed</span>
          )}
        </div>
        <button onClick={togglePanel} className="p-1 text-gray-400 hover:text-gray-600">
          <X className="w-4 h-4" />
        </button>
      </div>

      {/* Content */}
      <div className="flex-1 overflow-y-auto flex flex-col">
        {/* Stale banner */}
        {currentPrompt && staleResults[currentPrompt.flowId]?.isStale && (
          <StaleBanner staleResult={staleResults[currentPrompt.flowId]} />
        )}

        {/* Idle state — show queue */}
        {panelState === 'idle' && <ImplementationQueue />}

        {/* Prompt preview */}
        {panelState === 'prompt_ready' && currentPrompt && (
          <PromptPreview prompt={currentPrompt} onRun={runClaudeCode} />
        )}

        {/* Running terminal */}
        {panelState === 'running' && ptySession && (
          <div className="flex-1 flex flex-col">
            <TerminalEmbed sessionId={ptySession.id} />
            <div className="p-2 border-t border-gray-200">
              <button
                onClick={stopClaudeCode}
                className="flex items-center gap-1.5 px-3 py-1.5 text-xs bg-red-50 text-red-600 rounded hover:bg-red-100"
              >
                <Square className="w-3 h-3" /> Stop
              </button>
            </div>
          </div>
        )}

        {/* Done / Failed — show summary + test results */}
        {(panelState === 'done' || panelState === 'failed') && (
          <div className="flex-1 flex flex-col">
            {panelState === 'done' && (
              <div className="p-4 bg-green-50 border-b border-green-100 text-sm text-green-700">
                Implementation complete. {testResults ? '' : 'Running tests...'}
              </div>
            )}
            {panelState === 'failed' && (
              <div className="p-4 bg-red-50 border-b border-red-100 text-sm text-red-700">
                Implementation failed. Check terminal output above.
                <div className="mt-2 flex gap-2">
                  <button
                    onClick={runClaudeCode}
                    className="text-xs px-2 py-1 bg-red-100 rounded hover:bg-red-200"
                  >
                    Retry
                  </button>
                </div>
              </div>
            )}
            {(testResults || isRunningTests) && currentPrompt && (
              <TestResults
                results={testResults}
                isRunning={isRunningTests}
                flowId={currentPrompt.flowId}
              />
            )}
          </div>
        )}
      </div>
    </div>
  );
}
```

**File: `src/components/ImplementationPanel/PromptPreview.tsx`**
```tsx
import { useState } from 'react';
import { useImplementationStore } from '../../stores/implementation-store';
import type { BuiltPrompt } from '../../types/implementation';
import { Play, Pencil, FileText } from 'lucide-react';

interface Props {
  prompt: BuiltPrompt;
  onRun: () => void;
}

export function PromptPreview({ prompt, onRun }: Props) {
  const { updatePromptContent } = useImplementationStore();
  const [isEditing, setIsEditing] = useState(false);

  return (
    <div className="flex-1 flex flex-col">
      {/* Header */}
      <div className="px-4 py-2 border-b border-gray-100 flex items-center justify-between">
        <div className="flex items-center gap-2 text-xs text-gray-500">
          <FileText className="w-3 h-3" />
          <span>{prompt.mode === 'update' ? 'Update' : 'New'}: {prompt.flowId}</span>
          {prompt.agentFlow && (
            <span className="bg-purple-100 text-purple-700 px-1.5 py-0.5 rounded text-[10px]">agent</span>
          )}
        </div>
        <div className="flex items-center gap-1">
          <button
            onClick={() => setIsEditing(!isEditing)}
            className="flex items-center gap-1 text-xs px-2 py-1 text-gray-500 hover:text-gray-700 rounded hover:bg-gray-100"
          >
            <Pencil className="w-3 h-3" /> {isEditing ? 'Preview' : 'Edit'}
          </button>
          <button
            onClick={onRun}
            className="flex items-center gap-1.5 px-3 py-1.5 text-xs bg-blue-500 text-white rounded hover:bg-blue-600"
          >
            <Play className="w-3 h-3" /> Run
          </button>
        </div>
      </div>

      {/* Prompt content */}
      <div className="flex-1 overflow-y-auto p-4">
        {isEditing ? (
          <textarea
            value={prompt.content}
            onChange={(e) => updatePromptContent(e.target.value)}
            className="w-full h-full font-mono text-xs bg-gray-50 border border-gray-200 rounded p-3 resize-none focus:outline-none focus:border-blue-300"
          />
        ) : (
          <pre className="text-xs font-mono text-gray-700 whitespace-pre-wrap">
            {prompt.content}
          </pre>
        )}
      </div>

      {/* Spec files referenced */}
      <div className="px-4 py-2 border-t border-gray-100">
        <div className="text-[10px] text-gray-400 mb-1">Referenced specs:</div>
        <div className="flex flex-wrap gap-1">
          {prompt.specFiles.map(f => (
            <span key={f} className="text-[10px] bg-gray-100 text-gray-500 px-1.5 py-0.5 rounded">
              {f.split('/').pop()}
            </span>
          ))}
        </div>
      </div>
    </div>
  );
}
```

**File: `src/components/ImplementationPanel/TerminalEmbed.tsx`**
```tsx
import { useEffect, useRef, useState, useCallback } from 'react';
import { invoke } from '@tauri-apps/api/core';
import { useImplementationStore } from '../../stores/implementation-store';

interface Props {
  sessionId: string;
}

export function TerminalEmbed({ sessionId }: Props) {
  const termRef = useRef<HTMLDivElement>(null);
  const [output, setOutput] = useState('');
  const { sendToPty } = useImplementationStore();

  // Poll PTY for output
  useEffect(() => {
    let active = true;

    const poll = async () => {
      while (active) {
        try {
          const data = await invoke<string>('pty_read', { sessionId });
          if (data) {
            setOutput(prev => prev + data);
          }
        } catch {
          break; // Session ended
        }
        await new Promise(r => setTimeout(r, 100));
      }
    };

    poll();
    return () => { active = false; };
  }, [sessionId]);

  // Auto-scroll to bottom
  useEffect(() => {
    if (termRef.current) {
      termRef.current.scrollTop = termRef.current.scrollHeight;
    }
  }, [output]);

  // Handle keyboard input
  const handleKeyDown = useCallback(async (e: React.KeyboardEvent) => {
    if (e.key === 'Enter') {
      await sendToPty('\n');
    } else if (e.key === 'Backspace') {
      await sendToPty('\x7f');
    } else if (e.key.length === 1) {
      await sendToPty(e.key);
    }
  }, [sendToPty]);

  return (
    <div
      ref={termRef}
      className="flex-1 bg-gray-900 text-green-400 font-mono text-xs p-3 overflow-y-auto whitespace-pre-wrap focus:outline-none"
      tabIndex={0}
      onKeyDown={handleKeyDown}
    >
      {output || 'Starting Claude Code...'}
    </div>
  );
}
```

**File: `src/components/ImplementationPanel/TestResults.tsx`**
```tsx
import { useImplementationStore } from '../../stores/implementation-store';
import type { TestSummary } from '../../types/implementation';
import { CheckCircle, XCircle, Loader2, RotateCw, Wrench } from 'lucide-react';

interface Props {
  results: TestSummary | null;
  isRunning: boolean;
  flowId: string;
}

export function TestResults({ results, isRunning, flowId }: Props) {
  const { runTests, fixFailingTest } = useImplementationStore();

  if (isRunning) {
    return (
      <div className="p-4 flex items-center gap-2 text-sm text-gray-500">
        <Loader2 className="w-4 h-4 animate-spin" />
        Running tests...
      </div>
    );
  }

  if (!results) return null;

  const failedTests = results.tests.filter(t => t.status === 'failed');

  return (
    <div className="border-t border-gray-200">
      {/* Summary bar */}
      <div className="px-4 py-2 flex items-center justify-between bg-gray-50 border-b border-gray-100">
        <div className="text-xs font-medium">
          {results.passed}/{results.total} passing
          {results.failed > 0 && <span className="text-red-500 ml-2">{results.failed} failing</span>}
        </div>
        <div className="flex gap-1">
          <button
            onClick={() => runTests(flowId)}
            className="flex items-center gap-1 text-xs px-2 py-1 text-gray-500 hover:text-gray-700 rounded hover:bg-gray-100"
          >
            <RotateCw className="w-3 h-3" /> Re-run
          </button>
          {failedTests.length > 0 && (
            <button
              onClick={() => {
                const errorOutput = failedTests
                  .map(t => `${t.name}: ${t.error_message ?? 'failed'}`)
                  .join('\n');
                fixFailingTest(flowId, errorOutput);
              }}
              className="flex items-center gap-1 text-xs px-2 py-1 text-blue-500 hover:text-blue-700 rounded hover:bg-blue-50"
            >
              <Wrench className="w-3 h-3" /> Fix failing
            </button>
          )}
        </div>
      </div>

      {/* Test list */}
      <div className="max-h-60 overflow-y-auto">
        {results.tests.map(test => (
          <div key={test.name} className="px-4 py-1.5 flex items-start gap-2 text-xs border-b border-gray-50">
            {test.status === 'passed' ? (
              <CheckCircle className="w-3.5 h-3.5 text-green-500 mt-0.5 flex-shrink-0" />
            ) : (
              <XCircle className="w-3.5 h-3.5 text-red-500 mt-0.5 flex-shrink-0" />
            )}
            <div className="flex-1 min-w-0">
              <div className="text-gray-700 truncate">{test.name}</div>
              {test.error_message && (
                <div className="text-red-400 mt-0.5 text-[10px] font-mono truncate">
                  {test.error_message}
                </div>
              )}
            </div>
            <span className="text-gray-400 text-[10px]">{test.duration_ms.toFixed(0)}ms</span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

**File: `src/components/ImplementationPanel/ImplementationQueue.tsx`**
```tsx
import { useImplementationStore } from '../../stores/implementation-store';
import { CheckCircle, Circle, AlertTriangle, Play } from 'lucide-react';

export function ImplementationQueue() {
  const {
    queue, toggleQueueItem, selectAllPending, implementSelected, buildPromptForFlow,
  } = useImplementationStore();

  const pending = queue.filter(q => q.status === 'pending');
  const stale = queue.filter(q => q.status === 'stale');
  const implemented = queue.filter(q => q.status === 'implemented');
  const hasSelected = queue.some(q => q.selected);

  const renderItem = (item: typeof queue[0]) => (
    <div key={item.flowId} className="flex items-center gap-2 px-4 py-1.5 hover:bg-gray-50">
      <input
        type="checkbox"
        checked={item.selected}
        onChange={() => toggleQueueItem(item.flowId)}
        className="w-3.5 h-3.5 rounded border-gray-300"
        disabled={item.status === 'implemented'}
      />
      {item.status === 'implemented' && <CheckCircle className="w-3.5 h-3.5 text-green-500" />}
      {item.status === 'pending' && <Circle className="w-3.5 h-3.5 text-gray-300" />}
      {item.status === 'stale' && <AlertTriangle className="w-3.5 h-3.5 text-amber-500" />}
      <button
        onClick={() => buildPromptForFlow(item.flowId, item.domain)}
        className="text-xs text-gray-700 hover:text-blue-600 truncate flex-1 text-left"
      >
        {item.flowId}
      </button>
      {item.staleChanges && (
        <span className="text-[10px] text-amber-500">{item.staleChanges.length} changes</span>
      )}
      {item.testResult && (
        <span className={`text-[10px] ${item.testResult.failed > 0 ? 'text-red-500' : 'text-green-500'}`}>
          {item.testResult.passed}/{item.testResult.total}
        </span>
      )}
    </div>
  );

  return (
    <div className="flex-1 overflow-y-auto">
      <div className="px-4 py-3 text-xs font-medium text-gray-500 border-b border-gray-100">
        Implementation Queue
      </div>

      {pending.length > 0 && (
        <div className="border-b border-gray-100">
          <div className="px-4 py-1.5 text-[10px] font-medium text-gray-400 uppercase">Pending</div>
          {pending.map(renderItem)}
        </div>
      )}

      {stale.length > 0 && (
        <div className="border-b border-gray-100">
          <div className="px-4 py-1.5 text-[10px] font-medium text-amber-400 uppercase">Stale</div>
          {stale.map(renderItem)}
        </div>
      )}

      {implemented.length > 0 && (
        <div className="border-b border-gray-100">
          <div className="px-4 py-1.5 text-[10px] font-medium text-green-400 uppercase">Implemented</div>
          {implemented.map(renderItem)}
        </div>
      )}

      {/* Actions */}
      <div className="px-4 py-3 flex gap-2 border-t border-gray-200">
        <button
          onClick={selectAllPending}
          className="text-xs px-2 py-1 text-gray-500 hover:text-gray-700 rounded hover:bg-gray-100"
        >
          Select all pending
        </button>
        {hasSelected && (
          <button
            onClick={implementSelected}
            className="flex items-center gap-1 text-xs px-3 py-1 bg-blue-500 text-white rounded hover:bg-blue-600"
          >
            <Play className="w-3 h-3" /> Implement selected
          </button>
        )}
      </div>
    </div>
  );
}
```

**File: `src/components/ImplementationPanel/StaleBanner.tsx`**
```tsx
import type { StaleResult } from '../../types/implementation';
import { useImplementationStore } from '../../stores/implementation-store';
import { AlertTriangle } from 'lucide-react';

interface Props {
  staleResult: StaleResult;
}

export function StaleBanner({ staleResult }: Props) {
  const { buildPromptForFlow } = useImplementationStore();

  return (
    <div className="px-4 py-3 bg-amber-50 border-b border-amber-100">
      <div className="flex items-start gap-2">
        <AlertTriangle className="w-4 h-4 text-amber-500 mt-0.5 flex-shrink-0" />
        <div className="flex-1">
          <div className="text-sm font-medium text-amber-800">
            Spec changed since code was generated
          </div>
          <ul className="mt-1 text-xs text-amber-700 space-y-0.5">
            {staleResult.changes.slice(0, 5).map((change, i) => (
              <li key={i}>{change}</li>
            ))}
            {staleResult.changes.length > 5 && (
              <li className="text-amber-500">...and {staleResult.changes.length - 5} more</li>
            )}
          </ul>
          <div className="mt-2 flex gap-2">
            <button
              onClick={() => buildPromptForFlow(
                staleResult.flowId,
                staleResult.flowId.split('/')[0]
              )}
              className="text-xs px-2 py-1 bg-amber-100 text-amber-700 rounded hover:bg-amber-200"
            >
              Update code
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}
```

**File: `src/components/ImplementationPanel/ReconciliationReport.tsx`**
```tsx
import { useImplementationStore } from '../../stores/implementation-store';
import { useProjectStore } from '../../stores/project-store';
import type { ReconciliationItem } from '../../types/implementation';
import { CheckCircle, AlertTriangle, XCircle, Loader2, ArrowDownToLine, Trash2, EyeOff } from 'lucide-react';

export function ReconciliationReport() {
  const { reconciliationReport: report, isReconciling, resolveReconciliationItem } = useImplementationStore();
  const { projectPath } = useProjectStore();

  if (isReconciling) {
    return (
      <div className="p-4 flex items-center gap-2 text-sm text-gray-500">
        <Loader2 className="w-4 h-4 animate-spin" />
        Reconciling spec vs code...
      </div>
    );
  }

  if (!report) return null;

  const scoreColor = report.syncScore >= 95 ? 'text-green-600' :
    report.syncScore >= 80 ? 'text-yellow-600' :
    report.syncScore >= 50 ? 'text-amber-600' : 'text-red-600';

  const scoreBg = report.syncScore >= 95 ? 'bg-green-50' :
    report.syncScore >= 80 ? 'bg-yellow-50' :
    report.syncScore >= 50 ? 'bg-amber-50' : 'bg-red-50';

  const handleResolve = (itemId: string, resolution: 'accepted' | 'removed' | 'ignored') => {
    resolveReconciliationItem(report.flowId, itemId, resolution, projectPath);
  };

  const renderItem = (item: ReconciliationItem, type: 'codeOnly' | 'specOnly') => (
    <div key={item.id} className="px-4 py-2 border-b border-gray-50">
      <div className="flex items-start gap-2">
        {type === 'codeOnly' ? (
          <AlertTriangle className="w-3.5 h-3.5 text-amber-500 mt-0.5 flex-shrink-0" />
        ) : (
          <XCircle className="w-3.5 h-3.5 text-red-400 mt-0.5 flex-shrink-0" />
        )}
        <div className="flex-1 min-w-0">
          <div className="text-xs text-gray-700">{item.description}</div>
          {item.codeLocation && (
            <div className="text-[10px] text-gray-400 mt-0.5 font-mono">{item.codeLocation}</div>
          )}
          <div className="text-[10px] text-gray-400 mt-0.5">
            <span className={`px-1 py-0.5 rounded ${
              item.severity === 'significant' ? 'bg-red-50 text-red-500' :
              item.severity === 'moderate' ? 'bg-amber-50 text-amber-500' :
              'bg-gray-50 text-gray-400'
            }`}>
              {item.severity}
            </span>
            <span className="ml-1">{item.category}</span>
          </div>
          {!item.resolution && (
            <div className="mt-1.5 flex gap-1">
              {type === 'codeOnly' && (
                <>
                  <button
                    onClick={() => handleResolve(item.id, 'accepted')}
                    className="flex items-center gap-0.5 text-[10px] px-1.5 py-0.5 bg-green-50 text-green-600 rounded hover:bg-green-100"
                  >
                    <ArrowDownToLine className="w-2.5 h-2.5" /> Accept into spec
                  </button>
                  <button
                    onClick={() => handleResolve(item.id, 'removed')}
                    className="flex items-center gap-0.5 text-[10px] px-1.5 py-0.5 bg-red-50 text-red-500 rounded hover:bg-red-100"
                  >
                    <Trash2 className="w-2.5 h-2.5" /> Remove from code
                  </button>
                </>
              )}
              <button
                onClick={() => handleResolve(item.id, 'ignored')}
                className="flex items-center gap-0.5 text-[10px] px-1.5 py-0.5 bg-gray-50 text-gray-500 rounded hover:bg-gray-100"
              >
                <EyeOff className="w-2.5 h-2.5" /> Ignore
              </button>
            </div>
          )}
          {item.resolution && (
            <div className="mt-1 text-[10px] text-gray-400 italic">
              {item.resolution === 'accepted' ? 'Accepted into spec' :
               item.resolution === 'removed' ? 'Queued for removal' : 'Ignored'}
            </div>
          )}
        </div>
      </div>
    </div>
  );

  return (
    <div className="border-t border-gray-200">
      {/* Sync score header */}
      <div className={`px-4 py-2 ${scoreBg} border-b border-gray-100 flex items-center justify-between`}>
        <div className="text-xs font-medium">
          <span className={scoreColor}>Sync: {report.syncScore}%</span>
          <span className="text-gray-400 ml-2">
            {report.matching.length} matching, {report.codeOnly.length} code-only, {report.specOnly.length} spec-only
          </span>
        </div>
      </div>

      {/* Matching items (collapsed by default) */}
      {report.matching.length > 0 && (
        <details className="border-b border-gray-100">
          <summary className="px-4 py-1.5 text-[10px] font-medium text-green-500 cursor-pointer hover:bg-gray-50">
            Matching ({report.matching.length})
          </summary>
          {report.matching.map(item => (
            <div key={item.id} className="px-4 py-1 flex items-center gap-2 text-xs text-gray-500">
              <CheckCircle className="w-3 h-3 text-green-400 flex-shrink-0" />
              {item.description}
            </div>
          ))}
        </details>
      )}

      {/* Code-only items (drift) */}
      {report.codeOnly.length > 0 && (
        <div className="border-b border-gray-100">
          <div className="px-4 py-1.5 text-[10px] font-medium text-amber-500 uppercase">
            Code has, spec doesn't ({report.codeOnly.length})
          </div>
          {report.codeOnly.map(item => renderItem(item, 'codeOnly'))}
        </div>
      )}

      {/* Spec-only items (missing implementation) */}
      {report.specOnly.length > 0 && (
        <div className="border-b border-gray-100">
          <div className="px-4 py-1.5 text-[10px] font-medium text-red-400 uppercase">
            Spec has, code doesn't ({report.specOnly.length})
          </div>
          {report.specOnly.map(item => renderItem(item, 'specOnly'))}
        </div>
      )}

      {report.syncScore === 100 && (
        <div className="px-4 py-3 text-xs text-green-600 text-center">
          Spec and code are fully in sync.
        </div>
      )}
    </div>
  );
}
```

**File: `src/components/ImplementationPanel/SyncBadge.tsx`**
```tsx
interface Props {
  syncScore: number | undefined;
  onClick?: () => void;
}

export function SyncBadge({ syncScore, onClick }: Props) {
  if (syncScore === undefined) return null;

  const color = syncScore >= 95 ? 'text-green-600 bg-green-50' :
    syncScore >= 80 ? 'text-yellow-600 bg-yellow-50' :
    syncScore >= 50 ? 'text-amber-600 bg-amber-50' : 'text-red-600 bg-red-50';

  const icon = syncScore >= 95 ? '✓' :
    syncScore >= 80 ? '~' :
    syncScore >= 50 ? '⚠' : '✗';

  return (
    <button
      onClick={onClick}
      className={`text-[10px] px-1.5 py-0.5 rounded ${color} hover:opacity-80`}
      title={`Sync score: ${syncScore}% — click to view reconciliation report`}
    >
      {icon} {syncScore}% synced
    </button>
  );
}
```

**Update `src/components/ImplementationPanel/ImplementationPanel.tsx` — add reconciliation:**
```tsx
// Add to imports
import { ReconciliationReport } from './ReconciliationReport';

// Add after test results in the done/failed state:
{(panelState === 'done' || panelState === 'failed') && (
  <div className="flex-1 flex flex-col">
    {/* ... existing done/failed UI ... */}
    {(testResults || isRunningTests) && currentPrompt && (
      <TestResults results={testResults} isRunning={isRunningTests} flowId={currentPrompt.flowId} />
    )}
    {/* Reconciliation report — shown after tests */}
    <ReconciliationReport />
  </div>
)}
```

**Update `src/components/DomainMap/FlowBlock.tsx` — add sync badge:**
```tsx
// Add to imports
import { SyncBadge } from '../ImplementationPanel/SyncBadge';
import { useImplementationStore } from '../../stores/implementation-store';

// Inside FlowBlock render, after test badge:
const { mappings, togglePanel, reconcileFlow } = useImplementationStore();
const mapping = mappings[`${domainId}/${flowId}`];

// In the flow block JSX:
{mapping?.sync_score !== undefined && (
  <SyncBadge
    syncScore={mapping.sync_score}
    onClick={() => {
      togglePanel();
      reconcileFlow(`${domainId}/${flowId}`, projectPath);
    }}
  />
)}
```

**Update `src/App.tsx` — add Implementation Panel:**
```tsx
// Add to imports
import { ImplementationPanel } from './components/ImplementationPanel/ImplementationPanel';

// Add keyboard shortcut in useEffect:
// Cmd+I / Ctrl+I → toggle implementation panel
if ((e.metaKey || e.ctrlKey) && e.key === 'i') {
  e.preventDefault();
  useImplementationStore.getState().togglePanel();
}

// Add to layout — Implementation Panel goes rightmost:
<div className="flex h-screen">
  {/* ... existing Sidebar, Canvas, SpecPanel, ChatPanel, MemoryPanel ... */}
  <ImplementationPanel />
  <UsageBar /> {/* Bottom bar */}
</div>
```

**Update `src-tauri/Cargo.toml` — add dependencies:**
```toml
[dependencies]
# ... existing deps ...
portable-pty = "0.8"
sha2 = "0.10"
nanoid = "0.4"
```

---

### Production Infrastructure Generators

**File: `src/types/production.ts`**
```typescript
/** OpenAPI generation */
export interface OpenAPISpec {
  openapi: string;
  info: { title: string; version: string; description: string };
  paths: Record<string, Record<string, OpenAPIOperation>>;
  components: { schemas: Record<string, any> };
}

export interface OpenAPIOperation {
  operationId: string;
  summary: string;
  tags: string[];
  requestBody?: any;
  responses: Record<string, { description: string; content?: any }>;
}

/** Schema migration tracking */
export interface SchemaMapping {
  spec: string;
  spec_hash: string;
  migration_history: MigrationEntry[];
  last_synced_at: string;
}

export interface MigrationEntry {
  version: string;
  description: string;
  generated_at: string;
}

export interface SchemaChange {
  schemaName: string;
  changes: string[];        // Human-readable changes
  currentHash: string;
  storedHash: string;
}

/** Generated artifact tracking */
export interface GeneratedArtifact {
  type: 'openapi' | 'ci_pipeline' | 'dockerfile' | 'compose' | 'k8s' | 'migration';
  path: string;
  generated_at: string;
  source_hash: string;      // Hash of source config that generated this
  stale: boolean;
}
```

**File: `src/utils/openapi-generator.ts`**
```typescript
import { invoke } from '@tauri-apps/api/core';
import type { OpenAPISpec } from '../types/production';

/**
 * Generate OpenAPI 3.0 spec from all HTTP flow specs.
 * Reads flow YAMLs, errors.yaml, and schemas to produce a complete API spec.
 */
export async function generateOpenAPI(projectPath: string): Promise<string> {
  const systemRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/system.yaml` });
  const system = (await import('yaml')).parse(systemRaw);

  const spec: OpenAPISpec = {
    openapi: '3.0.3',
    info: {
      title: system.system?.name ?? 'API',
      version: system.system?.version ?? '1.0.0',
      description: system.system?.description ?? '',
    },
    paths: {},
    components: { schemas: {} },
  };

  // Read error codes for response mapping
  let errorCodes: Record<string, any> = {};
  try {
    const errRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/shared/errors.yaml` });
    errorCodes = (await import('yaml')).parse(errRaw);
  } catch { /* ignore */ }

  // Read schemas
  try {
    const schemaFiles = await invoke<string[]>('list_files', { dir: `${projectPath}/specs/schemas`, pattern: '*.yaml' });
    for (const file of schemaFiles) {
      const raw = await invoke<string>('read_file', { path: `${projectPath}/specs/schemas/${file}` });
      const schema = (await import('yaml')).parse(raw);
      const name = file.replace('.yaml', '');
      spec.components.schemas[name] = schemaToOpenAPI(schema);
    }
  } catch { /* no schemas dir */ }

  // Read all flows and extract HTTP endpoints
  const domains = system.system?.domains ?? [];
  for (const domain of domains) {
    try {
      const flowFiles = await invoke<string[]>('list_files', {
        dir: `${projectPath}/specs/domains/${domain.name}/flows`,
        pattern: '*.yaml',
      });

      for (const file of flowFiles) {
        const raw = await invoke<string>('read_file', {
          path: `${projectPath}/specs/domains/${domain.name}/flows/${file}`,
        });
        const flow = (await import('yaml')).parse(raw);

        // Only include HTTP-triggered flows
        if (flow.trigger?.type !== 'http') continue;

        const path = flow.trigger.path;
        const method = flow.trigger.method.toLowerCase();

        if (!spec.paths[path]) spec.paths[path] = {};

        spec.paths[path][method] = {
          operationId: flow.flow?.id ?? file.replace('.yaml', ''),
          summary: flow.flow?.name ?? '',
          tags: [domain.name],
          ...buildRequestBody(flow),
          responses: buildResponses(flow, errorCodes),
        };

        // Add request/response schemas
        const reqSchema = buildInputSchema(flow);
        if (reqSchema) {
          const schemaName = `${capitalize(flow.flow.id.replace(/-/g, ''))}Request`;
          spec.components.schemas[schemaName] = reqSchema;
        }
      }
    } catch { /* domain has no flows */ }
  }

  return (await import('yaml')).stringify(spec);
}

function buildInputSchema(flow: any): any | null {
  const inputNode = (flow.nodes ?? []).find((n: any) => n.type === 'input');
  if (!inputNode?.spec?.fields) return null;

  const properties: Record<string, any> = {};
  const required: string[] = [];

  for (const [name, field] of Object.entries(inputNode.spec.fields)) {
    const f = field as any;
    properties[name] = { type: mapType(f.type) };
    if (f.format) properties[name].format = f.format;
    if (f.min_length) properties[name].minLength = f.min_length;
    if (f.max_length) properties[name].maxLength = f.max_length;
    if (f.required) required.push(name);
  }

  return { type: 'object', required, properties };
}

function buildRequestBody(flow: any): any {
  const inputNode = (flow.nodes ?? []).find((n: any) => n.type === 'input');
  if (!inputNode) return {};
  return {
    requestBody: {
      required: true,
      content: { 'application/json': { schema: buildInputSchema(flow) } },
    },
  };
}

function buildResponses(flow: any, errorCodes: any): Record<string, any> {
  const responses: Record<string, any> = {};

  // Success responses from terminal nodes
  for (const node of flow.nodes ?? []) {
    if (node.type === 'terminal' && node.spec?.status) {
      const status = String(node.spec.status);
      responses[status] = {
        description: node.spec.body?.message ?? 'Success',
      };
    }
  }

  // Error responses from error codes used in the flow
  const usedCodes = extractFlowErrorCodes(flow);
  for (const code of usedCodes) {
    const errorDef = findErrorDef(code, errorCodes);
    if (errorDef) {
      const status = String(errorDef.http_status);
      if (!responses[status]) {
        responses[status] = { description: errorDef.message ?? code };
      }
    }
  }

  return responses;
}

function extractFlowErrorCodes(flow: any): string[] {
  const codes: string[] = [];
  const str = JSON.stringify(flow);
  const matches = str.matchAll(/"error_code"\s*:\s*"(\w+)"/g);
  for (const m of matches) codes.push(m[1]);
  return [...new Set(codes)];
}

function findErrorDef(code: string, errorCodes: any): any {
  for (const category of Object.values(errorCodes)) {
    if (typeof category === 'object' && (category as any)[code]) {
      return (category as any)[code];
    }
  }
  return null;
}

function schemaToOpenAPI(schema: any): any {
  const properties: Record<string, any> = {};
  for (const [name, field] of Object.entries(schema.fields ?? schema.properties ?? {})) {
    properties[name] = { type: mapType((field as any).type) };
  }
  return { type: 'object', properties };
}

function mapType(t: string): string {
  const map: Record<string, string> = { string: 'string', int: 'integer', integer: 'integer', float: 'number', boolean: 'boolean', date: 'string', datetime: 'string', uuid: 'string' };
  return map[t] ?? 'string';
}

function capitalize(s: string): string { return s.charAt(0).toUpperCase() + s.slice(1); }
```

**File: `src/utils/cicd-generator.ts`**
```typescript
import { invoke } from '@tauri-apps/api/core';

/**
 * Generate GitHub Actions CI/CD pipeline from architecture.yaml and system.yaml.
 */
export async function generateCICD(projectPath: string): Promise<string> {
  const yaml = await import('yaml');
  const systemRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/system.yaml` });
  const system = yaml.parse(systemRaw);

  let arch: any = {};
  try {
    const archRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/architecture.yaml` });
    arch = yaml.parse(archRaw);
  } catch { /* use defaults */ }

  const techStack = system.system?.tech_stack ?? {};
  const lang = techStack.language ?? 'python';
  const langVersion = techStack.language_version ?? '3.11';
  const db = techStack.database;
  const cache = techStack.cache;

  const testing = arch.testing ?? {};
  const security = arch.cross_cutting?.security ?? {};

  let pipeline: any = {
    name: 'CI',
    on: {
      push: { branches: ['main', 'develop'] },
      pull_request: { branches: ['main'] },
    },
    jobs: {
      test: {
        'runs-on': 'ubuntu-latest',
        services: {},
        steps: [
          { uses: 'actions/checkout@v4' },
        ],
      },
    },
  };

  const steps = pipeline.jobs.test.steps;

  // Add database service
  if (db === 'postgresql') {
    pipeline.jobs.test.services.postgres = {
      image: 'postgres:15',
      env: { POSTGRES_USER: 'test', POSTGRES_PASSWORD: 'test', POSTGRES_DB: 'test' },
      ports: ['5432:5432'],
      options: '--health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5',
    };
  }

  // Add cache service
  if (cache === 'redis') {
    pipeline.jobs.test.services.redis = {
      image: 'redis:7',
      ports: ['6379:6379'],
    };
  }

  // Language setup
  if (lang === 'python') {
    steps.push({
      name: `Set up Python ${langVersion}`,
      uses: 'actions/setup-python@v5',
      with: { 'python-version': langVersion },
    });
    steps.push({ name: 'Install dependencies', run: 'pip install -e ".[dev]"' });
  } else if (lang === 'typescript' || lang === 'javascript') {
    steps.push({
      name: `Set up Node ${langVersion}`,
      uses: 'actions/setup-node@v4',
      with: { 'node-version': langVersion },
    });
    steps.push({ name: 'Install dependencies', run: 'npm ci' });
  }

  // Migration step
  if (arch.database?.migrations) {
    steps.push({ name: 'Run migrations', run: arch.database.migrations.command ?? 'alembic upgrade head' });
  }

  // Lint
  const lintCmd = testing.commands?.lint ?? (lang === 'python' ? 'ruff check .' : 'npm run lint');
  steps.push({ name: 'Lint', run: lintCmd });

  // Type check
  const typeCmd = testing.commands?.typecheck ?? (lang === 'python' ? 'mypy src/' : 'npx tsc --noEmit');
  steps.push({ name: 'Type check', run: typeCmd });

  // Test
  const testCmd = testing.commands?.test ?? (lang === 'python' ? 'pytest --tb=short -q' : 'npm test');
  steps.push({ name: 'Test', run: testCmd });

  // Dependency scanning
  if (security.dependency_scanning?.ci_step) {
    const scanTool = security.dependency_scanning.tool ?? 'safety';
    steps.push({ name: 'Dependency scan', run: scanTool === 'safety' ? 'pip install safety && safety check' : 'npm audit' });
  }

  // Spec-code sync check
  steps.push({
    name: 'Spec-code sync check',
    run: lang === 'python'
      ? `python -c "\nimport yaml, hashlib, sys\nmapping = yaml.safe_load(open('.ddd/mapping.yaml'))\nstale = [fid for fid, m in mapping.get('flows', {}).items() if hashlib.sha256(open(m['spec'], 'rb').read()).hexdigest() != m['spec_hash']]\nif stale: print(f'Stale: {stale}'); sys.exit(1)\nprint('All in sync')\n"`
      : 'node -e "/* spec sync check */"',
  });

  return yaml.stringify(pipeline);
}
```

**File: `src/utils/dockerfile-generator.ts`**
```typescript
import { invoke } from '@tauri-apps/api/core';

/**
 * Generate Dockerfile and docker-compose.yaml from architecture.yaml deployment config.
 */
export async function generateDockerfile(projectPath: string): Promise<string> {
  const yaml = await import('yaml');
  const systemRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/system.yaml` });
  const system = yaml.parse(systemRaw);

  let arch: any = {};
  try {
    const archRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/architecture.yaml` });
    arch = yaml.parse(archRaw);
  } catch { /* defaults */ }

  const techStack = system.system?.tech_stack ?? {};
  const deployment = arch.deployment?.docker ?? {};
  const health = arch.cross_cutting?.observability?.health ?? {};
  const lang = techStack.language ?? 'python';
  const langVersion = techStack.language_version ?? '3.11';
  const port = deployment.port ?? 8000;
  const multiStage = deployment.multi_stage ?? true;
  const baseImage = deployment.base_image ?? `${lang}:${langVersion}-slim`;
  const healthPath = health.liveness?.path ?? '/health/live';

  if (lang === 'python') {
    const framework = techStack.framework ?? 'fastapi';
    const startCmd = framework === 'fastapi'
      ? `uvicorn src.main:app --host 0.0.0.0 --port ${port}`
      : `python -m src.main`;

    if (multiStage) {
      return `# Generated by DDD Tool
FROM ${baseImage} AS builder
WORKDIR /app
COPY pyproject.toml .
RUN pip install --no-cache-dir .

FROM ${baseImage}
WORKDIR /app
COPY --from=builder /usr/local/lib/python${langVersion}/site-packages /usr/local/lib/python${langVersion}/site-packages
COPY --from=builder /usr/local/bin /usr/local/bin
COPY src/ src/
COPY alembic/ alembic/
COPY alembic.ini .
EXPOSE ${port}
HEALTHCHECK --interval=30s --timeout=5s CMD curl -f http://localhost:${port}${healthPath} || exit 1
CMD ["${startCmd.split(' ')[0]}", ${startCmd.split(' ').slice(1).map(a => `"${a}"`).join(', ')}]
`;
    }
  }

  // Fallback generic Dockerfile
  return `# Generated by DDD Tool
FROM ${baseImage}
WORKDIR /app
COPY . .
EXPOSE ${port}
CMD ["start"]
`;
}

export async function generateDockerCompose(projectPath: string): Promise<string> {
  const yaml = await import('yaml');
  const systemRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/system.yaml` });
  const system = yaml.parse(systemRaw);

  let arch: any = {};
  try {
    const archRaw = await invoke<string>('read_file', { path: `${projectPath}/specs/architecture.yaml` });
    arch = yaml.parse(archRaw);
  } catch { /* defaults */ }

  const techStack = system.system?.tech_stack ?? {};
  const port = arch.deployment?.docker?.port ?? 8000;

  const services: any = {
    app: {
      build: '.',
      ports: [`${port}:${port}`],
      env_file: '.env',
      depends_on: [],
    },
  };

  if (techStack.database === 'postgresql') {
    services.postgres = {
      image: 'postgres:15',
      volumes: ['pgdata:/var/lib/postgresql/data'],
      environment: { POSTGRES_DB: system.system?.name ?? 'app', POSTGRES_USER: 'app', POSTGRES_PASSWORD: '${DB_PASSWORD}' },
      ports: ['5432:5432'],
    };
    services.app.depends_on.push('postgres');
  }

  if (techStack.cache === 'redis') {
    services.redis = { image: 'redis:7-alpine', ports: ['6379:6379'] };
    services.app.depends_on.push('redis');
  }

  const compose: any = { version: '3.8', services };
  if (techStack.database === 'postgresql') {
    compose.volumes = { pgdata: {} };
  }

  return yaml.stringify(compose);
}
```

**File: `src/utils/migration-tracker.ts`**
```typescript
import { invoke } from '@tauri-apps/api/core';
import type { SchemaMapping, SchemaChange } from '../types/production';

/**
 * Track schema hashes and detect changes for migration generation.
 */
export async function checkSchemaChanges(projectPath: string): Promise<SchemaChange[]> {
  const yaml = await import('yaml');
  const changes: SchemaChange[] = [];

  // Load existing schema mappings
  let mappings: Record<string, SchemaMapping> = {};
  try {
    const raw = await invoke<string>('read_file', { path: `${projectPath}/.ddd/mapping.yaml` });
    const parsed = yaml.parse(raw);
    mappings = parsed.schemas ?? {};
  } catch { /* no mapping file */ }

  // Scan all schemas
  let schemaFiles: string[] = [];
  try {
    schemaFiles = await invoke<string[]>('list_files', { dir: `${projectPath}/specs/schemas`, pattern: '*.yaml' });
  } catch { return []; }

  for (const file of schemaFiles) {
    const schemaName = file.replace('.yaml', '').replace('_base', 'Base');
    const specPath = `${projectPath}/specs/schemas/${file}`;
    const currentHash = await invoke<string>('hash_file', { path: specPath });
    const stored = mappings[schemaName];

    if (stored && stored.spec_hash !== currentHash) {
      // Schema changed — compute diff
      const cachedPath = `${projectPath}/.ddd/cache/schemas-at-migration/${file}`;
      let diffChanges: string[] = [];
      try {
        diffChanges = await invoke<string[]>('diff_specs', { cachedPath, currentPath: specPath });
      } catch {
        diffChanges = ['Schema file changed (no cached version for diff)'];
      }

      changes.push({
        schemaName,
        changes: diffChanges,
        currentHash,
        storedHash: stored.spec_hash,
      });
    } else if (!stored) {
      // New schema — not yet tracked
      changes.push({
        schemaName,
        changes: ['New schema (not yet in database)'],
        currentHash,
        storedHash: '',
      });
    }
  }

  return changes;
}

/**
 * Build a migration prompt for Claude Code based on detected schema changes.
 */
export function buildMigrationPrompt(changes: SchemaChange[], ormTool: string): string {
  let prompt = `## Database Migration Required\n\n`;
  prompt += `The following schema changes need database migrations:\n\n`;

  for (const change of changes) {
    prompt += `### ${change.schemaName}\n`;
    for (const c of change.changes) {
      prompt += `- ${c}\n`;
    }
    prompt += `\n`;
  }

  prompt += `Generate a database migration using ${ormTool}:\n`;
  prompt += `- Migration should be reversible (include downgrade)\n`;
  prompt += `- Do NOT modify existing data destructively\n`;
  prompt += `- Add new columns as nullable or with defaults\n`;
  prompt += `- For renamed columns, create new + copy + drop old\n`;

  return prompt;
}

/**
 * Cache a schema at migration time for future diffing.
 */
export async function cacheSchemaAtMigration(projectPath: string, schemaFile: string): Promise<void> {
  const cacheDir = `${projectPath}/.ddd/cache/schemas-at-migration`;
  await invoke('ensure_dir', { path: cacheDir });
  await invoke('copy_file', {
    src: `${projectPath}/specs/schemas/${schemaFile}`,
    dst: `${cacheDir}/${schemaFile}`,
  });
}
```

**Update `src/stores/implementation-store.ts` — add artifact generation methods:**

Add to the ImplementationState interface:
```typescript
  // Production infrastructure generation
  generateOpenAPI: (projectPath: string) => Promise<void>;
  generateCICD: (projectPath: string) => Promise<void>;
  generateDockerfiles: (projectPath: string) => Promise<void>;
  checkSchemaChanges: (projectPath: string) => Promise<SchemaChange[]>;
  schemaChanges: SchemaChange[];
```

Add to the store implementation:
```typescript
  schemaChanges: [],

  generateOpenAPI: async (projectPath) => {
    const { generateOpenAPI } = await import('../utils/openapi-generator');
    const content = await generateOpenAPI(projectPath);
    await invoke('write_file', { path: `${projectPath}/openapi.yaml`, content });
  },

  generateCICD: async (projectPath) => {
    const { generateCICD } = await import('../utils/cicd-generator');
    const content = await generateCICD(projectPath);
    await invoke('ensure_dir', { path: `${projectPath}/.github/workflows` });
    await invoke('write_file', { path: `${projectPath}/.github/workflows/ci.yaml`, content });
  },

  generateDockerfiles: async (projectPath) => {
    const { generateDockerfile, generateDockerCompose } = await import('../utils/dockerfile-generator');
    const dockerfile = await generateDockerfile(projectPath);
    const compose = await generateDockerCompose(projectPath);
    await invoke('write_file', { path: `${projectPath}/Dockerfile`, content: dockerfile });
    await invoke('write_file', { path: `${projectPath}/docker-compose.yaml`, content: compose });
  },

  checkSchemaChanges: async (projectPath) => {
    const { checkSchemaChanges } = await import('../utils/migration-tracker');
    const changes = await checkSchemaChanges(projectPath);
    set({ schemaChanges: changes });
    return changes;
  },
```

**Update prompt builder — add migration prompt when schemas changed:**

In `runClaudeCode` post-implementation, after test runner:
```typescript
  // Check for schema changes needing migrations
  const schemaChanges = await get().checkSchemaChanges(projectPath);
  if (schemaChanges.length > 0) {
    const { buildMigrationPrompt } = await import('../utils/migration-tracker');
    const ormTool = config?.claude_code?.orm_migration_tool ?? 'alembic';
    const migrationPrompt = buildMigrationPrompt(schemaChanges, ormTool);
    // Show migration prompt in panel for user to review and run
    set({
      currentPrompt: {
        flowId: `schema-migration`,
        domain: '_schemas',
        content: migrationPrompt,
        specFiles: schemaChanges.map(c => `specs/schemas/${c.schemaName}.yaml`),
        mode: 'update',
        agentFlow: false,
        editedByUser: false,
      },
      panelState: 'prompt_ready',
    });
  }
```

**Sidebar addition — Production tab:**

Add to `src/components/Sidebar/`:
```tsx
// ProductionTab.tsx — shows generated artifacts status
import { useImplementationStore } from '../../stores/implementation-store';
import { useProjectStore } from '../../stores/project-store';
import { FileCode, GitBranch, Container, Database, Shield, Activity, RefreshCw } from 'lucide-react';

export function ProductionTab() {
  const { generateOpenAPI, generateCICD, generateDockerfiles, checkSchemaChanges, schemaChanges } = useImplementationStore();
  const { projectPath } = useProjectStore();

  const artifacts = [
    { icon: FileCode, label: 'OpenAPI Spec', file: 'openapi.yaml', generate: () => generateOpenAPI(projectPath) },
    { icon: GitBranch, label: 'CI/CD Pipeline', file: '.github/workflows/ci.yaml', generate: () => generateCICD(projectPath) },
    { icon: Container, label: 'Dockerfile', file: 'Dockerfile', generate: () => generateDockerfiles(projectPath) },
  ];

  return (
    <div className="p-3 space-y-3">
      <div className="text-xs font-medium text-gray-500 uppercase">Generated Artifacts</div>

      {artifacts.map(({ icon: Icon, label, generate }) => (
        <div key={label} className="flex items-center justify-between text-xs">
          <div className="flex items-center gap-2 text-gray-700">
            <Icon className="w-3.5 h-3.5" />
            {label}
          </div>
          <button
            onClick={generate}
            className="flex items-center gap-1 text-blue-500 hover:text-blue-700"
          >
            <RefreshCw className="w-3 h-3" /> Generate
          </button>
        </div>
      ))}

      {schemaChanges.length > 0 && (
        <div className="border border-amber-200 rounded p-2 bg-amber-50">
          <div className="flex items-center gap-1.5 text-xs font-medium text-amber-700 mb-1">
            <Database className="w-3.5 h-3.5" />
            Schema changes detected
          </div>
          {schemaChanges.map(c => (
            <div key={c.schemaName} className="text-[10px] text-amber-600">
              {c.schemaName}: {c.changes.length} change(s)
            </div>
          ))}
          <button
            onClick={() => checkSchemaChanges(projectPath)}
            className="text-[10px] text-amber-700 underline mt-1"
          >
            Generate migration prompt
          </button>
        </div>
      )}
    </div>
  );
}
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

15. **LLM Chat Panel**
    - [ ] Cmd+L / Ctrl+L toggles chat panel open/closed
    - [ ] Chat panel shows context indicator (project-level / domain / flow)
    - [ ] Suggestion chips appear in empty chat
    - [ ] Can send a message and receive LLM response
    - [ ] Streaming indicator shows while LLM is responding
    - [ ] Generated YAML in response shows Apply / Edit / Discard buttons
    - [ ] Apply adds ghost nodes to canvas
    - [ ] Discard removes ghost preview
    - [ ] "Edit in chat" closes preview and focuses chat input
    - [ ] Chat history persists per-flow thread
    - [ ] Switching flows switches chat thread

16. **Ghost Preview**
    - [ ] Ghost nodes appear with dashed borders and reduced opacity
    - [ ] Ghost preview bar appears at bottom of canvas with node count
    - [ ] Apply converts ghost nodes to real nodes
    - [ ] Escape discards ghost preview
    - [ ] Enter (when no input focused) applies ghost preview

17. **Inline Assist**
    - [ ] Right-click node shows ✨ actions (Suggest spec, Explain, Add error handling, etc.)
    - [ ] Right-click connection shows ✨ actions (Add node between, Suggest label)
    - [ ] Right-click empty canvas shows ✨ actions (Generate flow, Review, Suggest wiring)
    - [ ] Right-click on Level 2 shows ✨ actions (Suggest flows, Suggest events, Generate domain)
    - [ ] Right-click on Level 1 shows ✨ actions (Suggest domains, Review architecture, Generate from description)
    - [ ] Inline assist shows loading spinner while LLM processes
    - [ ] Result appears as popover with suggestion text
    - [ ] Cmd+. / Ctrl+. triggers inline assist on selected node

18. **LLM Context**
    - [ ] System context always included in LLM requests
    - [ ] Domain context included on Level 2 and 3
    - [ ] Flow context (full YAML) included on Level 3
    - [ ] Selected node specs included when nodes are selected
    - [ ] Error codes from project included
    - [ ] Schemas from project included

19. **Multi-Model Architecture**
    - [ ] Model registry shows all configured models with availability indicators
    - [ ] Model picker dropdown in chat header shows all models grouped by provider
    - [ ] Switching active model applies to subsequent chat messages
    - [ ] Task routing routes inline assist actions to configured models (fast for suggest, powerful for review)
    - [ ] Inline assist context menu shows which model will handle each action
    - [ ] Inline assist model can be overridden via "Use different model…" submenu
    - [ ] Fallback chain tries next model when primary fails
    - [ ] OpenAI-compatible endpoint works (custom base_url)
    - [ ] Ollama works for offline use (no API key required)
    - [ ] API keys read from environment variables (never stored in config)
    - [ ] Health check detects which models are reachable (green/red dots)

20. **Cost Tracking**
    - [ ] Usage bar at bottom of app shows active model + session cost
    - [ ] Monthly usage tracked per model and per task type
    - [ ] Budget warning banner appears at configured threshold
    - [ ] Budget limit blocks requests with option to switch to free model
    - [ ] Usage data persisted to .ddd/memory/usage.yaml
    - [ ] Chat messages show which model generated them

20. **Project Memory — Summary (Layer 1)**
    - [ ] Project summary generated on first load (LLM call)
    - [ ] Summary regenerates when specs change
    - [ ] Stale indicator shows when summary is outdated
    - [ ] Summary included in every LLM request context

21. **Project Memory — Spec Index (Layer 2)**
    - [ ] Spec index built from all flow YAMLs, domain configs, schemas
    - [ ] Index includes condensed flow info (trigger, nodes, events, schemas, error codes)
    - [ ] Index auto-refreshes when spec files change

22. **Project Memory — Decision Log (Layer 3)**
    - [ ] Can add design decisions via Memory Panel
    - [ ] Can edit and remove decisions
    - [ ] Decisions saved to .ddd/memory/decisions.md
    - [ ] decisions.md is NOT in .gitignore (committed to repo)
    - [ ] LLM auto-detects rationale in chat and offers to save as decision
    - [ ] Relevant decisions filtered by current domain/flow in LLM context

23. **Project Memory — Flow Map (Layer 4)**
    - [ ] Cross-flow dependency graph derived from events, sub-flows, schemas
    - [ ] Flow dependencies shown in Memory Panel when on Level 3
    - [ ] Connected flows (upstream/downstream) included in LLM context
    - [ ] Auto-refreshes when flow specs change

24. **Project Memory — Implementation Status (Layer 5)**
    - [ ] Status list shows implemented/pending/stale per flow
    - [ ] Stale flows show count of changes since last code generation
    - [ ] Status derived from .ddd/mapping.yaml + git diff
    - [ ] Implementation status of current flow included in LLM context

25. **Memory Panel**
    - [ ] Cmd+M / Ctrl+M toggles memory panel
    - [ ] Shows summary card with project overview + flow counts
    - [ ] Shows implementation status list (sorted: stale first)
    - [ ] Shows decision list with add/edit/remove
    - [ ] Shows flow dependencies card when on Level 3
    - [ ] Refresh button triggers memory rebuild

26. **Implementation Panel**
    - [ ] Cmd+I / Ctrl+I toggles implementation panel
    - [ ] Panel shows queue in idle state (pending, stale, implemented flows)
    - [ ] Click flow in queue → builds prompt and shows prompt preview
    - [ ] Prompt preview shows auto-generated prompt with spec file references
    - [ ] Edit button toggles prompt into editable textarea
    - [ ] Run button spawns Claude Code in embedded PTY terminal
    - [ ] Terminal shows live Claude Code output with scrolling
    - [ ] Can type in terminal to respond to Claude Code prompts (y/n, etc.)
    - [ ] Stop button kills running Claude Code process
    - [ ] After completion, panel shows Done/Failed state
    - [ ] "Implement" button on L3 toolbar opens panel with prompt for current flow
    - [ ] Right-click flow block on L2 → "Implement" opens panel for that flow

27. **Prompt Builder**
    - [ ] Prompt includes architecture.yaml, errors.yaml, referenced schemas, flow spec
    - [ ] Schema resolution: only schemas referenced by $ref or data_store model are included
    - [ ] Agent flows include agent-specific instructions (tools, guardrails, memory)
    - [ ] Update mode: prompt includes list of spec changes and targets existing code
    - [ ] User edits to prompt are preserved until a new prompt is built

28. **Stale Detection**
    - [ ] On project load, all flow hashes compared against mapping.yaml
    - [ ] Stale flows show warning badge on L2 (Domain Map)
    - [ ] Stale flow sheet shows banner with list of changes
    - [ ] "Update code" button builds update prompt automatically
    - [ ] Spec cached at implementation time in .ddd/cache/specs-at-implementation/
    - [ ] Human-readable diff computed between cached and current spec

29. **Test Runner**
    - [ ] Tests auto-run after successful Claude Code completion (if configured)
    - [ ] Test results displayed with pass/fail per test case
    - [ ] "Re-run tests" button re-executes test command
    - [ ] "Fix failing test" sends error output to Claude Code with fix prompt
    - [ ] Test badges shown on L2 flow blocks (green ✓ or red ✗ with counts)
    - [ ] Scoped test execution: only runs tests matching the implemented flow

30. **CLAUDE.md Auto-Generation**
    - [ ] CLAUDE.md generated on first implementation
    - [ ] Regenerated when implementation status changes
    - [ ] Includes project name, spec files, domain table, rules, tech stack, commands
    - [ ] Custom section (below `<!-- CUSTOM -->` marker) preserved on regeneration
    - [ ] Domain table shows implementation status (N implemented, M pending)

31. **Implementation Queue**
    - [ ] Queue shows all flows grouped by status (pending, stale, implemented)
    - [ ] Can select multiple flows with checkboxes
    - [ ] "Select all pending" selects all pending + stale flows
    - [ ] "Implement selected" processes flows sequentially (prompt → run → test → next)
    - [ ] Implemented flows show test result counts

32. **Reverse Drift Detection — Implementation Report**
    - [ ] Prompt includes "Implementation Report" instruction asking Claude Code to list deviations
    - [ ] Terminal output is parsed for `## Implementation Notes` section after Claude Code finishes
    - [ ] "No deviations" is detected and skips reconciliation
    - [ ] Non-empty notes trigger auto-reconciliation (if configured)

33. **Reverse Drift Detection — Reconciliation**
    - [ ] After implementation, reconciliation runs automatically (configurable)
    - [ ] Reconciliation reads spec YAML and generated code files
    - [ ] LLM compares spec vs code and produces structured report
    - [ ] Report shows matching items, code-only items, and spec-only items
    - [ ] Sync score calculated and displayed (100% = perfect match)
    - [ ] "Accept into spec" updates flow YAML to include code addition
    - [ ] "Remove from code" builds targeted Claude Code prompt
    - [ ] "Ignore" stores accepted deviation in mapping.yaml
    - [ ] Accepted deviations don't trigger drift warnings again
    - [ ] Matching items collapsed by default (expandable)
    - [ ] Each item shows severity (minor/moderate/significant) and category

34. **Reverse Drift Detection — Sync Score**
    - [ ] Sync score badge shown on L2 flow blocks alongside test badge
    - [ ] Green (≥95%), yellow (80-94%), amber (50-79%), red (<50%)
    - [ ] Clicking sync badge opens reconciliation report
    - [ ] Sync score in toolbar on L3 flow sheet
    - [ ] Sync score stored in .ddd/mapping.yaml per flow
    - [ ] Sync score recalculates when items are resolved

35. **OpenAPI Generation**
    - [ ] "Generate" button produces openapi.yaml from all HTTP flow specs
    - [ ] Paths derived from trigger method + path
    - [ ] Request schemas derived from input node fields (type, format, min/max)
    - [ ] Response schemas derived from terminal node body shapes
    - [ ] Error responses mapped from error codes used in each flow
    - [ ] Schema $ref links to specs/schemas definitions
    - [ ] Non-HTTP flows (event, scheduled) excluded
    - [ ] Regenerates when flows with HTTP triggers change

36. **CI/CD Pipeline Generation**
    - [ ] "Generate" button produces .github/workflows/ci.yaml
    - [ ] Language setup step matches system.yaml tech stack
    - [ ] Service containers generated for database/cache (postgres, redis)
    - [ ] Lint, type-check, test commands from architecture.yaml
    - [ ] Dependency scanning step if security.dependency_scanning.ci_step enabled
    - [ ] Spec-code sync validation step always included
    - [ ] Regenerates when tech stack or commands change

37. **Database Migration Tracking**
    - [ ] Schema hashes tracked in .ddd/mapping.yaml under schemas key
    - [ ] Schema changes detected on project load and schema save
    - [ ] Changed schemas show warning in Production tab
    - [ ] "Generate migration prompt" builds Claude Code prompt with change details
    - [ ] Migration prompt specifies ORM tool from architecture.yaml
    - [ ] Schemas cached at migration time in .ddd/cache/schemas-at-migration/

38. **Observability Config**
    - [ ] Logging config in architecture.yaml generates structured logging setup
    - [ ] Tracing config generates OpenTelemetry initialization
    - [ ] Metrics config generates Prometheus endpoint and custom metric definitions
    - [ ] Health check config generates readiness and liveness endpoints
    - [ ] Sensitive fields redacted in log output
    - [ ] Per-flow node logging auto-generated (node.start, node.complete with duration)

39. **Security Layer**
    - [ ] Rate limiting config generates middleware with per-endpoint overrides
    - [ ] CORS config generates middleware
    - [ ] Security headers config generates middleware
    - [ ] Input sanitization config generates middleware (trim, strip HTML, max length)
    - [ ] Audit logging config generates audit model and middleware
    - [ ] Dependency scanning added to CI pipeline when configured

40. **Deployment Generation**
    - [ ] "Generate" button produces Dockerfile from deployment config
    - [ ] Multi-stage build when configured
    - [ ] Health check from observability config
    - [ ] docker-compose.yaml generated with all services (app, database, cache)
    - [ ] Kubernetes manifests generated when k8s enabled (deployment, service, ingress, hpa)
    - [ ] Regenerates when deployment or tech stack config changes

---

## Phase 5: Key Implementation Notes

### Important Patterns

1. **State Management**
   - Use Zustand for all state
   - Keep stores focused (sheet, flow, project, ui, git, llm, implementation)
   - `sheet-store` owns navigation state (current level, breadcrumbs, history)
   - `project-store` owns domain configs parsed from domain.yaml files
   - `flow-store` owns current flow being edited (Level 3 only)
   - `llm-store` owns chat state, ghost previews, LLM config
   - `memory-store` owns project memory layers, refresh triggers, decisions
   - `implementation-store` owns panel state, PTY session, queue, test results, mappings
   - Never mutate state directly

2. **Tauri Commands**
   - All file/git/LLM/memory/PTY/test operations go through Tauri
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
4. **Don't** make L1/L2 editable beyond repositioning — they are derived views
5. **Don't** try to run/test agents within the DDD Tool in MVP — just design them
6. **Don't** try to execute routing rules or circuit breakers — DDD is a design tool, not a runtime
7. **Do** focus on visual editing → YAML output
8. **Do** build multi-level navigation early — it's the core UX
9. **Do** parse domain.yaml files on project load to populate L1/L2
10. **Do** support all three flow types (traditional, agent, orchestration) in canvas routing
11. **Do** derive Level 2 orchestration visuals from flow YAML automatically
12. **Do** make sure Git integration works
13. **Do** validate YAML output format (traditional, agent, and orchestration)
14. **Don't** store API keys in `.ddd/config.yaml` — only store the env var name
15. **Don't** auto-apply LLM suggestions — always show ghost preview first
16. **Do** scope chat threads per flow — switching flows should switch threads
17. **Do** include full project context in every LLM request (system, domain, flow, schemas, error codes)
18. **Do** support fallback chain across models (e.g., Sonnet → GPT-4o → Llama local)
19. **Do** route fast tasks (suggest_spec, explain) to cheap/fast models and complex tasks (review, generate) to powerful models
20. **Don't** hard-code any provider — use the unified `call_model` dispatcher that routes by provider type
21. **Do** return token counts from every LLM call for accurate cost tracking
22. **Do** refresh memory layers when specs change (project load + save)
23. **Do** keep context budget under ~5,500 tokens — summarize, don't dump raw YAML
24. **Don't** gitignore `decisions.md` — design decisions are team knowledge
25. **Do** include connected flows from Flow Map in LLM context so it knows about upstream/downstream impact
26. **Don't** auto-commit after Claude Code finishes — always let the user review generated code first
27. **Don't** run Claude Code in headless/non-interactive mode — the user must be able to approve file operations
28. **Do** cache spec files at implementation time in `.ddd/cache/` for accurate stale diffs later
29. **Do** include only referenced schemas in prompts (resolve from `$ref` and `data_store` model), not all schemas
30. **Do** preserve the `<!-- CUSTOM -->` section when regenerating CLAUDE.md — users add project-specific instructions there
31. **Don't** parse test output from raw text in production — use JSON reporters (`--json-report` for pytest, `--json` for jest) for accurate test results
32. **Do** always include the Implementation Report instruction in prompts — without it, reverse drift detection has no structured data to parse
33. **Don't** auto-accept reconciliation items without user review — always show the report and let the user decide accept/remove/ignore
34. **Do** store accepted deviations in mapping.yaml so they don't re-trigger drift warnings on every reconciliation
35. **Don't** run reconciliation with the full codebase — only send the files listed in the flow's mapping, not the entire project
36. **Do** regenerate OpenAPI when any HTTP flow trigger changes — stale API docs are worse than no docs
37. **Don't** include non-HTTP flows (event, scheduled) in OpenAPI — they don't have REST endpoints
38. **Do** always include the spec-code sync check step in CI — it catches drift before deployment
39. **Do** cache schemas at migration time (like flow specs) so future diffs are accurate
40. **Don't** generate destructive migrations (DROP COLUMN) without user confirmation — add nullable columns, copy data, then drop
41. **Do** include sensitive_fields in logging config — accidentally logging passwords or tokens is a security incident
42. **Do** generate observability infrastructure before business logic — logging and health checks should exist from the first flow implementation

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
# reqwest = { version = "0.12", features = ["json"] }
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

### LLM Design Assistant
- [ ] Chat Panel opens/closes with Cmd+L and shows conversation
- [ ] Chat sends messages to configured LLM provider and displays responses
- [ ] LLM context includes system, domain, flow, and selected nodes automatically
- [ ] Generated YAML appears as ghost nodes on canvas with Apply/Discard
- [ ] Ghost nodes visually distinct (dashed borders, reduced opacity)
- [ ] Apply converts ghost nodes to real nodes in flow store
- [ ] Inline assist context menu appears on right-click with level-appropriate actions
- [ ] Node-level: Suggest spec, Complete spec, Explain, Add error handling, Generate tests
- [ ] Canvas-level: Generate flow, Review flow, Suggest wiring, Import from description
- [ ] Domain-level (L2): Suggest flows, Suggest events, Generate domain
- [ ] System-level (L1): Suggest domains, Review architecture, Generate from description
- [ ] Model registry supports Anthropic, OpenAI, Ollama, and any OpenAI-compatible endpoint
- [ ] Task-to-model routing sends fast tasks to cheap models and complex tasks to powerful models
- [ ] Model picker in chat header switches active model instantly
- [ ] Fallback chain tries next model when primary fails
- [ ] Cost tracking shows per-session and per-period usage with budget warnings
- [ ] Usage bar visible at bottom of app with active model indicator
- [ ] Chat threads scoped per flow, switchable
- [ ] API key read from env var, never stored in config files

### Project Memory
- [ ] 5 memory layers stored in .ddd/memory/ (summary.md, spec-index.yaml, decisions.md, flow-map.yaml, status.yaml)
- [ ] Auto-generated layers (1, 2, 4, 5) rebuild when specs change
- [ ] Decision log (Layer 3) is user-editable and version-controlled
- [ ] Project Summary always included in LLM context (~1,500 tokens)
- [ ] Connected flows from Flow Map included in LLM context on Level 3
- [ ] Relevant decisions filtered and included in LLM context
- [ ] Implementation status of current flow included in LLM context
- [ ] Memory Panel toggles with Cmd+M and shows all layers
- [ ] LLM auto-detects design rationale in chat and offers to save as decision
- [ ] Context budget stays within ~4,000-5,500 tokens total

### Claude Code Integration
- [ ] Implementation Panel opens/closes with Cmd+I and shows current state (idle/prompt/running/done/failed)
- [ ] Prompt Builder auto-generates prompt from flow spec with correct file references
- [ ] Prompt Builder resolves only referenced schemas (via $ref and data_store model)
- [ ] Prompt Builder detects agent flows and includes agent-specific instructions
- [ ] Prompt Builder detects update mode (existing mapping) and includes change list
- [ ] Prompt is editable before running — user can add custom instructions
- [ ] Embedded PTY terminal runs Claude Code interactively (user can approve/deny)
- [ ] Terminal output scrolls automatically and accepts keyboard input
- [ ] After Claude Code finishes, spec hash saved to .ddd/mapping.yaml
- [ ] Spec file cached in .ddd/cache/specs-at-implementation/ for future diff
- [ ] Tests auto-run after successful implementation (configurable)
- [ ] Test results displayed with per-test pass/fail and error messages
- [ ] "Fix failing test" button sends error to Claude Code for automatic fix
- [ ] Stale detection compares SHA-256 hashes on project load, flow save, and git pull
- [ ] Stale flows show warning badge on L2 and banner on L3 with change list
- [ ] CLAUDE.md auto-generated with project info, domain table, rules, tech stack
- [ ] CLAUDE.md custom section preserved on regeneration
- [ ] Implementation Queue shows all flows grouped by status
- [ ] Batch implementation processes selected flows sequentially
- [ ] Configuration loaded from .ddd/config.yaml with sensible defaults

### Reverse Drift Detection
- [ ] Prompt includes Implementation Report instruction requesting structured deviation notes
- [ ] Terminal output parsed for `## Implementation Notes` section
- [ ] Auto-reconciliation triggers after implementation (configurable via reconciliation.auto_run)
- [ ] Reconciliation sends spec + code to Design Assistant LLM for structured comparison
- [ ] Reconciliation report shows matching, code-only, and spec-only items
- [ ] Sync score (0-100%) calculated from matching vs total items
- [ ] "Accept into spec" uses LLM to generate updated spec YAML including the addition
- [ ] "Remove from code" builds targeted Claude Code prompt to remove the item
- [ ] "Ignore" stores accepted deviation in mapping.yaml (won't warn again)
- [ ] Sync score badge displayed on L2 flow blocks (green/yellow/amber/red)
- [ ] Clicking sync badge opens reconciliation report in Implementation Panel
- [ ] Sync score persisted in .ddd/mapping.yaml per flow
- [ ] Accepted deviations tracked and excluded from future reconciliation warnings
- [ ] Reconciliation available manually via "Reconcile" button in Implementation Panel

### Production Infrastructure
- [ ] OpenAPI generator produces valid OpenAPI 3.0 YAML from HTTP flow specs
- [ ] OpenAPI includes all paths, request/response schemas, error responses
- [ ] CI/CD generator produces GitHub Actions workflow with correct language setup and services
- [ ] CI pipeline includes spec-code sync validation step
- [ ] CI pipeline includes dependency scanning when security config enables it
- [ ] Schema migration tracker detects schema hash changes
- [ ] Schema migration prompt includes change details and ORM-specific instructions
- [ ] Schemas cached at migration time for future diffing
- [ ] Observability section in architecture.yaml generates logging/tracing/metrics/health infrastructure
- [ ] Logging includes sensitive field redaction
- [ ] Per-flow node logging added automatically (node.start, node.complete)
- [ ] Security section in architecture.yaml generates rate limiting, CORS, headers, sanitization, audit middleware
- [ ] Audit logging tracks configured events with actor, resource, and changes
- [ ] Dockerfile generated with multi-stage build and health check
- [ ] docker-compose generated with all service dependencies
- [ ] Kubernetes manifests generated when enabled (deployment, service, ingress, HPA)
- [ ] Production tab in Sidebar shows all generated artifacts with Generate buttons
- [ ] Schema changes notification shown in Production tab

---

## Next Steps After MVP

1. Add more node types (data_store, event, loop, parallel, sub_flow)
2. Add validation presets (email, phone, password)
3. Add Mermaid export
4. Add undo/redo
5. Add keyboard shortcuts
6. Add minimap showing position within hierarchy
7. Add expert agents
8. Add templates/library
9. Add live agent testing/debugging (run agent from within DDD Tool)
10. Add Router node visual (multi-output classification)
11. Add Memory node visual (vector store / conversation config)
12. Add orchestration observability dashboard (which agent handled what, latency, errors)
13. Add circuit breaker status indicators on routing arrows

---

## Reference

Full specification: `ddd-specification-complete.md`
