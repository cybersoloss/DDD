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
│       │   └── project.rs     # Project management
│       └── services/
│           ├── mod.rs
│           ├── git_service.rs
│           └── file_service.rs
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
│   │   │   ├── Canvas.tsx           # Level 3: flow sheet (node editing)
│   │   │   ├── Node.tsx
│   │   │   ├── Connection.tsx
│   │   │   └── nodes/
│   │   │       ├── TriggerNode.tsx
│   │   │       ├── InputNode.tsx
│   │   │       ├── ProcessNode.tsx
│   │   │       ├── DecisionNode.tsx
│   │   │       ├── TerminalNode.tsx
│   │   │       └── SubFlowNode.tsx  # Navigable link to another flow sheet
│   │   ├── SpecPanel/
│   │   │   ├── SpecPanel.tsx
│   │   │   ├── TriggerSpec.tsx
│   │   │   ├── InputSpec.tsx
│   │   │   ├── ProcessSpec.tsx
│   │   │   ├── DecisionSpec.tsx
│   │   │   └── TerminalSpec.tsx
│   │   ├── Sidebar/
│   │   │   ├── FlowList.tsx
│   │   │   ├── NodePalette.tsx
│   │   │   └── GitPanel.tsx
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
│   │   └── git-store.ts       # Git state
│   ├── types/
│   │   ├── sheet.ts           # Sheet levels, navigation, breadcrumb types
│   │   ├── domain.ts          # Domain config, event wiring, portal types
│   │   ├── flow.ts
│   │   ├── node.ts
│   │   ├── spec.ts
│   │   └── project.ts
│   ├── utils/
│   │   ├── yaml.ts
│   │   ├── domain-parser.ts   # Parse domain.yaml → SystemMap/DomainMap data
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
```

**File: `src/types/flow.ts`**
```typescript
import { FlowNode } from './node';

export interface Flow {
  id: string;
  name: string;
  description?: string;
  domain: string;
  trigger: FlowNode;  // First node, must be trigger type
  nodes: FlowNode[];
  metadata: {
    createdAt: string;
    updatedAt: string;
    createdBy?: string;
    completeness: number;  // 0-100
  };
}

export interface FlowFile {
  flow: {
    id: string;
    name: string;
    description?: string;
    domain: string;
  };
  trigger: any;  // Trigger spec
  nodes: any[];  // Node specs
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

9. **Save flow**
   - [ ] Click Save
   - [ ] YAML file created
   - [ ] File content matches flow

10. **Load flow**
    - [ ] Open existing YAML
    - [ ] Canvas shows correct nodes
    - [ ] Spec panel shows correct data

11. **Git status**
    - [ ] Panel shows current branch
    - [ ] Changed files listed
    - [ ] Stage/commit works

---

## Phase 5: Key Implementation Notes

### Important Patterns

1. **State Management**
   - Use Zustand for all state
   - Keep stores focused (sheet, flow, project, ui, git)
   - `sheet-store` owns navigation state (current level, breadcrumbs, history)
   - `project-store` owns domain configs parsed from domain.yaml files
   - `flow-store` owns current flow being edited (Level 3 only)
   - Never mutate state directly

2. **Tauri Commands**
   - All file/git operations go through Tauri
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

### Common Pitfalls

1. **Don't** build code generation in MVP
2. **Don't** build MCP server
3. **Don't** build reverse engineering
4. **Don't** make L1/L2 editable beyond repositioning — they are derived views
5. **Do** focus on visual editing → YAML output
6. **Do** build multi-level navigation early — it's the core UX
7. **Do** parse domain.yaml files on project load to populate L1/L2
8. **Do** make sure Git integration works
9. **Do** validate YAML output format

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

# 3. Add git2 to Cargo.toml
# In src-tauri/Cargo.toml, add:
# [dependencies]
# git2 = "0.18"

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

### Flow Sheet (L3)
- [ ] Can create a new flow with name and domain
- [ ] Can add 5 node types to canvas
- [ ] Can edit node specs in right panel
- [ ] Can connect nodes together
- [ ] Sub-flow nodes navigate to referenced flow sheet on double-click

### File Operations
- [ ] Can save flow as YAML file
- [ ] Can load flow from YAML file
- [ ] Can see Git status
- [ ] Can stage and commit changes
- [ ] YAML format matches specification
- [ ] Domain configs (domain.yaml) are parsed to populate L1/L2

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

---

## Reference

Full specification: `ddd-specification-complete.md`
