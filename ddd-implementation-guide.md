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
│       │   └── llm.rs         # LLM API proxy (Anthropic/OpenAI/Ollama)
│       └── services/
│           ├── mod.rs
│           ├── git_service.rs
│           ├── file_service.rs
│           └── llm_service.rs  # HTTP client, streaming, provider abstraction
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
│   │   │   └── FlowPreview.tsx        # Mini flow diagram in chat for generated flows
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
│   │   └── llm-store.ts       # Chat state, ghost nodes, LLM config
│   ├── types/
│   │   ├── sheet.ts           # Sheet levels, navigation, breadcrumb types
│   │   ├── domain.ts          # Domain config, event wiring, portal types
│   │   ├── flow.ts
│   │   ├── node.ts
│   │   ├── spec.ts
│   │   ├── project.ts
│   │   └── llm.ts             # Chat messages, LLM config, ghost node types
│   ├── utils/
│   │   ├── yaml.ts
│   │   ├── domain-parser.ts   # Parse domain.yaml → SystemMap/DomainMap data
│   │   ├── llm-context.ts     # Build context object for LLM requests
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
// --- LLM Provider Configuration ---

export type LLMProvider = 'anthropic' | 'openai' | 'ollama' | 'custom';

export interface LLMConfig {
  provider: LLMProvider;
  model: string;
  apiKeyEnv: string;           // Environment variable name (never store key directly)
  maxTokens: number;
  temperature: number;
  fallback?: {
    provider: LLMProvider;
    model: string;
    baseUrl?: string;          // For Ollama / custom endpoints
  };
}

// --- Chat Messages ---

export type ChatRole = 'user' | 'assistant' | 'system';

export interface ChatMessage {
  id: string;
  role: ChatRole;
  content: string;
  timestamp: number;
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

  // Chat panel
  chatOpen: boolean;
  activeThread: ChatThread | null;
  threads: ChatThread[];
  isStreaming: boolean;

  // Ghost preview
  ghostPreview: GhostPreview | null;

  // Actions — Chat
  toggleChat: () => void;
  openChat: () => void;
  closeChat: () => void;
  sendMessage: (content: string) => Promise<void>;
  switchThread: (threadId: string) => void;
  startThreadForFlow: (flowId: string, domainId: string) => void;

  // Actions — Ghost Preview
  applyGhostPreview: () => void;
  discardGhostPreview: () => void;
  editGhostInChat: () => void;

  // Actions — Inline Assist
  executeInlineAssist: (request: InlineAssistRequest) => Promise<string>;

  // Actions — Config
  updateConfig: (config: Partial<LLMConfig>) => void;
}

export const useLLMStore = create<LLMState>((set, get) => ({
  config: {
    provider: 'anthropic',
    model: 'claude-sonnet-4-5-20250929',
    apiKeyEnv: 'ANTHROPIC_API_KEY',
    maxTokens: 4096,
    temperature: 0.3,
  },

  chatOpen: false,
  activeThread: null,
  threads: [],
  isStreaming: false,
  ghostPreview: null,

  toggleChat: () => set(s => ({ chatOpen: !s.chatOpen })),
  openChat: () => set({ chatOpen: true }),
  closeChat: () => set({ chatOpen: false }),

  sendMessage: async (content: string) => {
    const state = get();
    const thread = state.activeThread;
    if (!thread) return;

    // Add user message
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

    set({
      activeThread: updatedThread,
      isStreaming: true,
    });

    // Build context from current project state
    const context = buildLLMContext();

    // Call Tauri backend
    try {
      const response: string = await invoke('llm_chat', {
        messages: updatedThread.messages.map(m => ({
          role: m.role,
          content: m.content,
        })),
        context,
        config: state.config,
      });

      const assistantMsg: ChatMessage = {
        id: nanoid(),
        role: 'assistant',
        content: response,
        timestamp: Date.now(),
      };

      // Check if response contains YAML (flow/node generation)
      const yamlMatch = response.match(/```yaml\n([\s\S]*?)```/);
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
        threads: state.threads.map(t =>
          t.id === finalThread.id ? finalThread : t
        ),
      });

      // If YAML was generated, parse and create ghost preview
      if (assistantMsg.generatedYaml) {
        const preview = parseYamlToGhostPreview(
          assistantMsg.id,
          assistantMsg.generatedYaml
        );
        set({ ghostPreview: preview });
      }
    } catch (error) {
      set({ isStreaming: false });
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

  applyGhostPreview: () => {
    const preview = get().ghostPreview;
    if (!preview) return;

    // Convert ghost nodes to real nodes via flow-store
    // (implementation delegates to flow-store.addNodes)
    set({ ghostPreview: { ...preview, state: 'applied' } });

    // Update the message's previewState
    const thread = get().activeThread;
    if (thread) {
      const updatedMessages = thread.messages.map(m =>
        m.id === preview.sourceMessageId
          ? { ...m, previewState: 'applied' as const }
          : m
      );
      set({
        activeThread: { ...thread, messages: updatedMessages },
        ghostPreview: null,
      });
    }
  },

  discardGhostPreview: () => {
    const preview = get().ghostPreview;
    if (!preview) return;

    const thread = get().activeThread;
    if (thread) {
      const updatedMessages = thread.messages.map(m =>
        m.id === preview.sourceMessageId
          ? { ...m, previewState: 'discarded' as const }
          : m
      );
      set({
        activeThread: { ...thread, messages: updatedMessages },
        ghostPreview: null,
      });
    }
  },

  editGhostInChat: () => {
    const preview = get().ghostPreview;
    if (!preview) return;
    set({ ghostPreview: null, chatOpen: true });
    // Focus the chat input — the user can type refinements
  },

  executeInlineAssist: async (request: InlineAssistRequest): Promise<string> => {
    const context = buildLLMContext();
    const config = get().config;

    const response: string = await invoke('llm_inline_assist', {
      action: request.action,
      target: request.target,
      context,
      config,
    });

    return response;
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

#[derive(Deserialize)]
pub struct LLMConfig {
    provider: String,     // "anthropic" | "openai" | "ollama" | "custom"
    model: String,
    api_key_env: String,
    max_tokens: u32,
    temperature: f32,
    fallback: Option<FallbackConfig>,
}

#[derive(Deserialize)]
pub struct FallbackConfig {
    provider: String,
    model: String,
    base_url: Option<String>,
}

#[derive(Deserialize)]
pub struct ChatMsg {
    role: String,
    content: String,
}

#[derive(Serialize)]
pub struct LLMResponse {
    content: String,
}

/// Main chat endpoint — sends messages + context to LLM provider
#[command]
pub async fn llm_chat(
    messages: Vec<ChatMsg>,
    context: serde_json::Value,
    config: LLMConfig,
) -> Result<String, String> {
    let api_key = std::env::var(&config.api_key_env)
        .map_err(|_| format!("Environment variable {} not set", config.api_key_env))?;

    let system_prompt = build_system_prompt(&context);

    let response = match config.provider.as_str() {
        "anthropic" => call_anthropic(&api_key, &config.model, &system_prompt, &messages, config.max_tokens, config.temperature).await,
        "openai" => call_openai(&api_key, &config.model, &system_prompt, &messages, config.max_tokens, config.temperature).await,
        "ollama" => call_ollama(&config.model, &system_prompt, &messages, config.fallback.as_ref().and_then(|f| f.base_url.as_deref())).await,
        _ => Err("Unsupported provider".to_string()),
    };

    match response {
        Ok(content) => Ok(content),
        Err(e) => {
            // Try fallback if configured
            if let Some(fallback) = &config.fallback {
                let fb_result = match fallback.provider.as_str() {
                    "ollama" => call_ollama(&fallback.model, &system_prompt, &messages, fallback.base_url.as_deref()).await,
                    _ => Err("Unsupported fallback provider".to_string()),
                };
                fb_result
            } else {
                Err(e)
            }
        }
    }
}

/// Inline assist endpoint — targeted action on a specific element
#[command]
pub async fn llm_inline_assist(
    action: String,
    target: serde_json::Value,
    context: serde_json::Value,
    config: LLMConfig,
) -> Result<String, String> {
    let api_key = std::env::var(&config.api_key_env)
        .map_err(|_| format!("Environment variable {} not set", config.api_key_env))?;

    let system_prompt = build_inline_assist_prompt(&action, &target, &context);

    let messages = vec![ChatMsg {
        role: "user".to_string(),
        content: format_inline_assist_user_message(&action, &target),
    }];

    match config.provider.as_str() {
        "anthropic" => call_anthropic(&api_key, &config.model, &system_prompt, &messages, config.max_tokens, config.temperature).await,
        "openai" => call_openai(&api_key, &config.model, &system_prompt, &messages, config.max_tokens, config.temperature).await,
        _ => Err("Unsupported provider".to_string()),
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

// Provider implementations (simplified — real implementation uses reqwest)

async fn call_anthropic(
    api_key: &str,
    model: &str,
    system_prompt: &str,
    messages: &[ChatMsg],
    max_tokens: u32,
    temperature: f32,
) -> Result<String, String> {
    let client = reqwest::Client::new();

    let api_messages: Vec<serde_json::Value> = messages.iter().map(|m| {
        serde_json::json!({
            "role": m.role,
            "content": m.content
        })
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
        .send()
        .await
        .map_err(|e| e.to_string())?;

    let json: serde_json::Value = response.json().await.map_err(|e| e.to_string())?;

    json["content"][0]["text"]
        .as_str()
        .map(|s| s.to_string())
        .ok_or_else(|| "Failed to parse LLM response".to_string())
}

async fn call_openai(
    api_key: &str,
    model: &str,
    system_prompt: &str,
    messages: &[ChatMsg],
    max_tokens: u32,
    temperature: f32,
) -> Result<String, String> {
    let client = reqwest::Client::new();

    let mut api_messages = vec![serde_json::json!({
        "role": "system",
        "content": system_prompt
    })];

    for m in messages {
        api_messages.push(serde_json::json!({
            "role": m.role,
            "content": m.content
        }));
    }

    let body = serde_json::json!({
        "model": model,
        "max_tokens": max_tokens,
        "temperature": temperature,
        "messages": api_messages
    });

    let response = client.post("https://api.openai.com/v1/chat/completions")
        .header("Authorization", format!("Bearer {}", api_key))
        .header("content-type", "application/json")
        .json(&body)
        .send()
        .await
        .map_err(|e| e.to_string())?;

    let json: serde_json::Value = response.json().await.map_err(|e| e.to_string())?;

    json["choices"][0]["message"]["content"]
        .as_str()
        .map(|s| s.to_string())
        .ok_or_else(|| "Failed to parse LLM response".to_string())
}

async fn call_ollama(
    model: &str,
    system_prompt: &str,
    messages: &[ChatMsg],
    base_url: Option<&str>,
) -> Result<String, String> {
    let client = reqwest::Client::new();
    let url = format!("{}/api/chat", base_url.unwrap_or("http://localhost:11434"));

    let mut api_messages = vec![serde_json::json!({
        "role": "system",
        "content": system_prompt
    })];

    for m in messages {
        api_messages.push(serde_json::json!({
            "role": m.role,
            "content": m.content
        }));
    }

    let body = serde_json::json!({
        "model": model,
        "messages": api_messages,
        "stream": false
    });

    let response = client.post(&url)
        .json(&body)
        .send()
        .await
        .map_err(|e| e.to_string())?;

    let json: serde_json::Value = response.json().await.map_err(|e| e.to_string())?;

    json["message"]["content"]
        .as_str()
        .map(|s| s.to_string())
        .ok_or_else(|| "Failed to parse Ollama response".to_string())
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
    closeChat, sendMessage,
  } = useLLMStore();

  const messagesEndRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [activeThread?.messages.length]);

  if (!chatOpen) return null;

  return (
    <div className="w-96 border-l border-gray-200 bg-white flex flex-col h-full">
      {/* Header */}
      <div className="flex items-center justify-between px-4 py-3 border-b border-gray-200">
        <div className="flex items-center gap-2">
          <Sparkles className="w-4 h-4 text-purple-500" />
          <span className="font-medium text-sm">Design Assistant</span>
        </div>
        <button onClick={closeChat} className="text-gray-400 hover:text-gray-600">
          <X className="w-4 h-4" />
        </button>
      </div>

      {/* Context indicator */}
      <div className="px-4 py-2 bg-gray-50 text-xs text-gray-500 border-b">
        {activeThread?.flowId
          ? `Context: ${activeThread.domainId} / ${activeThread.flowId}`
          : 'Context: Project-level'}
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

**App layout update — ChatPanel sits alongside SpecPanel:**
```tsx
// In App.tsx render
function App() {
  const { current } = useSheetStore();
  const { chatOpen } = useLLMStore();

  return (
    <div className="flex h-screen">
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

      {/* Right panels — Spec Panel and/or Chat Panel */}
      <SpecPanel />
      {chatOpen && <ChatPanel />}
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

19. **LLM Provider Config**
    - [ ] Can configure provider (Anthropic / OpenAI / Ollama)
    - [ ] API key read from environment variable (not stored in config)
    - [ ] Fallback provider works when primary fails
    - [ ] Ollama works for offline use

---

## Phase 5: Key Implementation Notes

### Important Patterns

1. **State Management**
   - Use Zustand for all state
   - Keep stores focused (sheet, flow, project, ui, git, llm)
   - `sheet-store` owns navigation state (current level, breadcrumbs, history)
   - `project-store` owns domain configs parsed from domain.yaml files
   - `flow-store` owns current flow being edited (Level 3 only)
   - `llm-store` owns chat state, ghost previews, LLM config
   - Never mutate state directly

2. **Tauri Commands**
   - All file/git/LLM operations go through Tauri
   - Use async/await pattern
   - Handle errors gracefully

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
18. **Do** support provider fallback (e.g., Anthropic → Ollama for offline use)

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

# 3. Add git2 and reqwest to Cargo.toml
# In src-tauri/Cargo.toml, add:
# [dependencies]
# git2 = "0.18"
# reqwest = { version = "0.12", features = ["json"] }
# serde_json = "1.0"

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
- [ ] Provider fallback works (Anthropic → Ollama for offline)
- [ ] Chat threads scoped per flow, switchable
- [ ] API key read from env var, never stored in config files

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
