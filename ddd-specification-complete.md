# Diagram-Driven Development (DDD) - Complete Specification

## Document Purpose
This document captures the complete specification for the DDD tool, enabling continuation of development discussions in any Claude session. It includes core concepts, architectural decisions, technical details, and implementation guidance.

---

# Part 1: Core Concept

## What is DDD?

**Diagram-Driven Development (DDD)** is a unified tool for bidirectional conversion between visual flow diagrams and code, enabling solopreneurs and small teams to architect systems visually while LLMs (like Claude Code) handle implementation.

### The Two-Way Workflow

```
FORWARD (Design-First):
  Diagram + Spec → Code
  You design visually → LLM implements

REVERSE (Documentation/Analysis):
  Code/Repo → Diagram + Spec
  Import existing code → Visualize and document
```

### Core Insight

**Specs are the single source of truth.** They live in the Git repo alongside code. Visual diagrams are just a UI for editing YAML spec files.

---

# Part 2: Purpose & Value Proposition

## Primary Purpose

Enable non-developers (PMs, solopreneurs) to:
1. Design software architecture visually
2. Specify exact behavior (validations, error messages, business rules)
3. Have LLMs implement the code automatically
4. Maintain sync between design and implementation

## Target Users

| User | Technical Level | Primary Use |
|------|-----------------|-------------|
| Product Managers | Low-Medium | Flow design, business rules |
| Tech Leads | High | Spec detailing, architecture review |
| Developers | High | Code generation, implementation |
| Solopreneurs | Varies | End-to-end control without coding |

## Value Proposition

### Time Savings
| Task | Traditional | With DDD |
|------|-------------|----------|
| Design auth flow | 2 hours | 5 min (import template) |
| Stripe integration | 4 hours | 30 min (import + customize) |
| CRUD for 10 entities | 5 hours | 20 min (template) |
| New SaaS project | 2 days | 1 hour (starter) |
| Onboard contractor | 2-4 weeks | Day 1 |
| Feature: idea to code | 2-5 days | 2-4 hours |

### Solopreneur Superpower
- **Bottleneck shifts** from coding speed to thinking speed
- Visual design (10 min) + Fill specs (20 min) + Review (10 min) = LLM implements (2-4 hours automated)
- One person can build what previously required a team

---

# Part 3: Key Properties & Principles

## 1. Git as Single Source of Truth

**Decision:** Use Git for all sync between DDD Tool and Claude Code. No custom sync protocol.

```
obligo/
├── .git/                    # Git handles ALL versioning
├── specs/                   # DDD Tool reads/writes here
│   ├── system.yaml
│   ├── schemas/
│   └── domains/
│       └── {domain}/
│           └── flows/
│               └── {flow}.yaml
├── src/                     # Code generated from specs
├── .ddd/
│   ├── config.yaml          # Project settings
│   └── mapping.yaml         # Spec-to-code mapping
└── CLAUDE.md                # Instructions for Claude Code
```

**Why Git wins over MCP-based sync:**
- Single source of truth (no state duplication)
- Built-in versioning, branching, merging, conflict resolution
- Works offline
- No extra infrastructure
- Familiar to developers
- Claude Code already knows Git

## 2. Hierarchical Spec Structure

```
System → Domain → Flow → Node
```

- **System Level:** Tech stack, domains, shared schemas, events, infrastructure
- **Domain Level:** Bounded contexts (ingestion, analysis, api, notification)
- **Flow Level:** Individual processes with triggers, nodes, connections
- **Node Level:** Single steps (trigger, input, process, decision, etc.)

## 3. Spec-Code Mapping

Every spec element maps to code:

```yaml
# .ddd/mapping.yaml
flows:
  webhook-ingestion:
    file: domains/ingestion/src/router.py
    function: receive_webhook
    nodes:
      validate_signature:
        type: inline
        location: "router.py:45-52"
      rate_limit_check:
        type: middleware
        file: domains/ingestion/src/middleware.py
        function: rate_limit_check
```

## 4. Validation as First-Class Citizen

Specs capture exact validation rules:

```yaml
spec:
  fields:
    email:
      type: string
      validations:
        - format: email
          error: "Please enter a valid email address"
        - max_length: 255
          error: "Email must be less than 255 characters"
        - unique: true
          error: "This email is already registered"
```

These become Pydantic validators, Zod schemas, etc.

## 5. Error Messages in Specs

Error messages are specified, not generated:

```yaml
on_failure:
  status: 401
  body:
    error: "Signature validation failed"
    code: "INVALID_SIGNATURE"
```

Code must use these exact messages. Validation checks this.

---

# Part 4: Architecture

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ DDD TOOL (Desktop App - Tauri)                                  │
│ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                │
│ │ Visual      │ │ Spec        │ │ Git         │                │
│ │ Editor      │ │ Panel       │ │ Panel       │                │
│ │ (Canvas)    │ │ (YAML UI)   │ │ (Status)    │                │
│ └─────────────┘ └─────────────┘ └─────────────┘                │
│                        │                                        │
│                        ▼                                        │
│              ┌─────────────────┐                               │
│              │ Spec Store      │                               │
│              │ (Zustand)       │                               │
│              └────────┬────────┘                               │
│                       │                                        │
└───────────────────────┼────────────────────────────────────────┘
                        │ Read/Write YAML files
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│ FILE SYSTEM (Git Repo)                                          │
│                                                                 │
│ specs/*.yaml ◄──────────────────────► src/*.py                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                        ▲
                        │ Read specs, Write code
                        │
┌───────────────────────┼────────────────────────────────────────┐
│ CLAUDE CODE           │                                        │
│                       │                                        │
│ ┌─────────────────────┴─────────────────────┐                 │
│ │ 1. Read pending spec changes              │                 │
│ │ 2. Generate/update code                   │                 │
│ │ 3. Validate against specs                 │                 │
│ │ 4. Commit changes                         │                 │
│ └───────────────────────────────────────────┘                 │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

## Multi-Level Canvas Architecture

A single canvas cannot scale to show an entire application. DDD uses a **3-level hierarchical sheet system** where each level provides the right abstraction for its scope. Users drill down by double-clicking, and navigate back via a breadcrumb bar.

```
┌─────────────────────────────────────────────────────────────────┐
│ LEVEL 1: SYSTEM MAP                                              │
│ One sheet. Auto-generated from system.yaml + domain.yaml files.  │
│ Shows domains as blocks, event arrows between them.              │
│                                                                   │
│   ┌──────────┐  contract.ingested  ┌──────────┐                 │
│   │ ingestion│────────────────────▶│ analysis │                 │
│   │ (4 flows)│                     │ (3 flows)│                 │
│   └──────────┘                     └──────────┘                 │
│        │                                │                        │
│        │ webhook.received               │ analysis.completed    │
│        ▼                                ▼                        │
│   ┌──────────┐                    ┌──────────┐                  │
│   │   api    │                    │notification│                 │
│   │ (5 flows)│                    │ (2 flows) │                 │
│   └──────────┘                    └──────────┘                  │
│                                                                   │
│   Double-click domain → Level 2                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ LEVEL 2: DOMAIN MAP                                              │
│ One sheet per domain. Shows flows as blocks within a domain,     │
│ event connections between flows, and portal nodes to other       │
│ domains.                                                         │
│                                                                   │
│   DOMAIN: ingestion                                              │
│                                                                   │
│   ┌──────────────┐     ┌───────────────┐                        │
│   │ webhook-     │────▶│ scheduled-    │                        │
│   │ ingestion    │event│ sync          │                        │
│   └──────────────┘     └───────────────┘                        │
│          │                                                       │
│          │ event: contract.ingested                              │
│          ▼                                                       │
│   ╔══════════════╗  ← portal to analysis domain                 │
│   ║ ➜ analysis   ║                                               │
│   ╚══════════════╝                                               │
│                                                                   │
│   Double-click flow → Level 3                                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ LEVEL 3: FLOW SHEET                                              │
│ One sheet per flow. Shows individual nodes and connections.      │
│ This is the existing detailed canvas.                            │
│                                                                   │
│   FLOW: webhook-ingestion                                        │
│                                                                   │
│   ⬡ trigger → ◇ validate_sig → ▱ validate_input                │
│                                    │                             │
│                              ⌗ store → ▭➤ event                 │
│                                           │                      │
│                                      ⬭ return                   │
│                                                                   │
│   Sub-flow nodes (▢) link to other flow sheets                  │
└─────────────────────────────────────────────────────────────────┘
```

### Navigation Model

| Action | Result |
|--------|--------|
| Double-click domain block (Level 1) | Drill into that domain's sheet (Level 2) |
| Double-click flow block (Level 2) | Drill into that flow's sheet (Level 3) |
| Double-click sub-flow node (Level 3) | Jump to referenced flow's sheet (Level 3) |
| Double-click portal node (Level 2) | Jump to target domain's sheet (Level 2) |
| Click breadcrumb segment | Navigate back to that level |
| Keyboard: Backspace / Esc | Navigate one level up |

**Breadcrumb bar** always visible at top of canvas:

```
System > ingestion > webhook-ingestion
  ^          ^              ^
  L1         L2             L3 (current)
```

### Sheet Data Sources

| Level | Auto-generated from | Editable? |
|-------|---------------------|-----------|
| System Map (L1) | `system.yaml` domains + `domain.yaml` event wiring | Layout only (positions) |
| Domain Map (L2) | `domain.yaml` flow list + inter-flow events | Layout only (positions) |
| Flow Sheet (L3) | `flows/*.yaml` | Fully editable (nodes, connections, specs) |

Levels 1 and 2 are **derived views** — their content comes from spec files. Users can reposition blocks but cannot add/remove domains or flows from these views (that happens via spec files or by creating new flow sheets). Level 3 is the primary editing surface.

## Node Types

### Flow-Level Nodes (Level 3) — Traditional Flows

| Type | Icon | Purpose |
|------|------|---------|
| Trigger | ⬡ | HTTP, webhook, cron, event |
| Input | ▱ | Form fields, API parameters |
| Process | ▭ | Transform, calculate, map |
| Decision | ◇ | Validation, business rules |
| Service Call | ▭▭ | External API, database, LLM |
| Data Store | ⌗ | Read/write operations |
| Event | ▭➤ | Queue publish, webhooks |
| Terminal | ⬭ | Response, end flow |
| Loop | ↺ | Iteration |
| Parallel | ═ | Concurrent execution |
| Sub-flow | ▢ | Call another flow (navigable link) |

### Agent Nodes (Level 3) — Agent Flows

| Type | Icon | Purpose |
|------|------|---------|
| LLM Call | ◆ | Call an LLM with prompt template, model config, structured output |
| Agent Loop | ↻ | Reason → select tool → execute → observe → repeat until done |
| Tool | 🔧 | Tool definition available to an agent (name, description, params) |
| Memory | ◈ | Vector store read/write, conversation history, context window |
| Guardrail | ⛨ | Input/output content filter, PII detection, topic restriction |
| Human Gate | ✋ | Pause flow, notify human, await approval, timeout/escalation |
| Router | ◇◇ | Semantic routing — LLM classifies intent, routes to sub-agents |

### Orchestration Nodes (Level 3) — Multi-Agent Flows

| Type | Icon | Purpose |
|------|------|---------|
| Orchestrator | ⊛ | Supervises multiple agents, monitors progress, merges results, intervenes |
| Smart Router | ◇◇+ | Rule-based + LLM hybrid routing with fallbacks, retries, A/B testing, circuit breaker |
| Handoff | ⇄ | Formal context transfer between agents (transfer/consult/collaborate modes) |
| Agent Group | [⊛] | Container for agents that share memory and coordination policy |

### Navigation Nodes (Level 2)

| Type | Icon | Purpose |
|------|------|---------|
| Flow Block | ▣ | Represents a traditional flow (double-click to drill in) |
| Agent Flow Block | ▣⊛ | Represents an agent flow (shows agent badge, double-click to drill in) |
| Agent Group Boundary | ╔[⊛]╗ | Visual boundary grouping agents that share memory/supervisor |
| Supervisor Arrow | ──⊛▶ | Orchestrator → managed agent relationship |
| Handoff Arrow | ⇄──▶ | Agent-to-agent handoff with mode label (transfer/consult/collaborate) |
| Portal | ╔══╗ | Link to another domain (double-click to jump) |
| Event Arrow | ──▶ | Async event connection between flows or to portals |

### Domain Blocks (Level 1)

| Type | Icon | Purpose |
|------|------|---------|
| Domain Block | ▣ | Represents a domain (shows flow count, double-click to drill in) |
| Event Arrow | ──▶ | Async event connection between domains |

## Connection Types Between Flows

1. **Event-Based (Async):** Publisher/subscriber via event bus — shown as arrows on Level 1 and Level 2
2. **Direct Call (Sync):** Sub-flow node with input/output mapping — shown as ▢ node on Level 3
3. **Shared Data Models:** `$ref:/schemas/SchemaName` references
4. **API Contracts:** Internal APIs connecting domains
5. **Agent Delegation:** Router or Agent Loop hands off to sub-agent flows
6. **Orchestrator → Agent:** Supervisor arrow on Level 2, orchestrator node on Level 3
7. **Agent Handoff:** Formal context transfer with handoff protocol (transfer/consult/collaborate)

---

## Agent Infrastructure

### Flow Types

DDD supports two flow types, each with different canvas behavior:

| Flow Type | `type:` value | Canvas Behavior |
|-----------|---------------|-----------------|
| **Traditional** | `traditional` (default) | Fixed node-to-node paths, deterministic execution |
| **Agent** | `agent` | Dynamic tool selection, iterative reasoning loops, non-deterministic paths |

When a flow has `type: agent`, the Level 3 canvas shows an **agent-centric layout** — the Agent Loop is the central element, with tools, guardrails, and memory arranged around it.

### Agent Flow Canvas Layout

```
┌─────────────────────────────────────────────────────────────────┐
│ FLOW: support-agent (type: agent)                                │
│                                                                   │
│   ⛨ Input                                                       │
│   Guardrail                                                       │
│      │                                                            │
│      ▼                                                            │
│   ┌─────────────────────────────────────────────┐                │
│   │            ↻ AGENT LOOP                      │                │
│   │                                              │                │
│   │  System Prompt: "You are a helpful..."       │                │
│   │  Model: claude-sonnet-4-5-20250929                    │                │
│   │  Max Iterations: 10                          │                │
│   │                                              │                │
│   │  ┌─────────────────────────────────┐         │                │
│   │  │  Reason → Select → Execute →    │         │                │
│   │  │  Observe → Repeat               │         │                │
│   │  └─────────────────────────────────┘         │                │
│   │                                              │                │
│   │  Available Tools:                            │                │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │                │
│   │  │🔧 search │ │🔧 create │ │🔧 escalate│    │                │
│   │  │   -kb    │ │  -ticket │ │          │    │                │
│   │  └──────────┘ └──────────┘ └──────────┘    │                │
│   │                                              │                │
│   │  Memory:                                     │                │
│   │  ◈ conversation (8000 tokens)               │                │
│   └─────────────────────────────────────────────┘                │
│      │                                                            │
│      ▼                                                            │
│   ⛨ Output          ✋ Human Gate                                │
│   Guardrail          (if escalation needed)                       │
│      │                    │                                       │
│      ▼                    ▼                                       │
│   ⬭ Response         ⬭ Escalated                                │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Agent Node Specifications

#### LLM Call Node (◆)

A single LLM invocation with a prompt template. Used in traditional flows when you need an LLM step without a full agent loop.

```yaml
- id: classify_intent
  type: llm_call
  spec:
    model: claude-haiku-4-5-20251001
    system_prompt: "Classify the user's intent into one of these categories."
    prompt_template: |
      User message: {{$.user_message}}

      Categories: billing, technical, general, escalation

      Respond with JSON: {"intent": "<category>", "confidence": <0-1>}
    temperature: 0.0
    max_tokens: 100
    structured_output:
      type: object
      properties:
        intent:
          type: string
          enum: [billing, technical, general, escalation]
        confidence:
          type: number
    retry:
      max_attempts: 2
      fallback_model: claude-sonnet-4-5-20250929
  connections:
    success: route_by_intent
    failure: return_error
```

#### Agent Loop Node (↻)

The core agent pattern — iterative reasoning with dynamic tool selection. This is the central node in agent flows.

```yaml
- id: support_agent_loop
  type: agent_loop
  spec:
    model: claude-sonnet-4-5-20250929
    system_prompt: |
      You are a customer support agent for Obligo.
      You help users with contract questions, obligation tracking,
      and account management.

      Guidelines:
      - Always search the knowledge base before answering
      - If you can't resolve the issue, create a ticket
      - Escalate to human if the customer is frustrated

    max_iterations: 10
    stop_conditions:
      - tool_called: respond_to_user    # Stop when agent calls this tool
      - tool_called: escalate_to_human
      - max_iterations_reached: true

    tools:
      - $ref: "#/tools/search_kb"
      - $ref: "#/tools/create_ticket"
      - $ref: "#/tools/get_account"
      - $ref: "#/tools/respond_to_user"
      - $ref: "#/tools/escalate_to_human"

    memory:
      type: conversation
      max_tokens: 8000
      include_tool_results: true

    scratchpad: true               # Agent can write internal notes

    on_max_iterations:
      action: escalate             # What to do if loop doesn't converge
      connection: escalation_gate

  connections:
    respond_to_user: output_guardrail
    escalate_to_human: escalation_gate
    error: return_error
```

#### Tool Node (🔧)

Defines a tool available to an agent. Tools are not hard-wired into the flow — the agent selects them dynamically.

```yaml
tools:
  - id: search_kb
    type: tool
    spec:
      name: search_kb
      description: "Search the knowledge base for articles matching a query"
      parameters:
        query:
          type: string
          required: true
          description: "Search query"
        limit:
          type: integer
          default: 5
          description: "Max results to return"
      implementation:
        type: service_call          # How the tool is actually executed
        service: knowledge_base
        method: search
      returns:
        type: array
        items:
          type: object
          properties:
            title: { type: string }
            content: { type: string }
            relevance: { type: number }

  - id: create_ticket
    type: tool
    spec:
      name: create_ticket
      description: "Create a support ticket for issues that need follow-up"
      parameters:
        subject:
          type: string
          required: true
        description:
          type: string
          required: true
        priority:
          type: string
          enum: [low, medium, high, urgent]
          default: medium
      implementation:
        type: data_store
        operation: create
        model: SupportTicket
      requires_confirmation: false   # Set true for destructive actions

  - id: escalate_to_human
    type: tool
    spec:
      name: escalate_to_human
      description: "Escalate to a human agent. Use when the customer is frustrated or the issue is too complex."
      parameters:
        reason:
          type: string
          required: true
        urgency:
          type: string
          enum: [normal, high, critical]
      implementation:
        type: event
        event: support.escalated
      is_terminal: true              # Calling this tool ends the agent loop

  - id: respond_to_user
    type: tool
    spec:
      name: respond_to_user
      description: "Send a response to the user. Call this when you have a complete answer."
      parameters:
        message:
          type: string
          required: true
      is_terminal: true              # Calling this tool ends the agent loop
```

#### Memory Node (◈)

Manages agent memory — conversation history, vector store, or shared context.

```yaml
- id: agent_memory
  type: memory
  spec:
    stores:
      - name: conversation
        type: conversation_history
        max_tokens: 8000
        strategy: sliding_window     # sliding_window | summarize | truncate
        include_system: true
        include_tool_results: true

      - name: knowledge
        type: vector_store
        provider: pgvector           # pgvector | pinecone | qdrant | chromadb
        embedding_model: text-embedding-3-small
        similarity_metric: cosine
        top_k: 5
        min_similarity: 0.7

      - name: user_context
        type: key_value
        storage: redis
        ttl: 3600
        fields:
          - user_id
          - account_tier
          - recent_tickets
```

#### Guardrail Node (⛨)

Content filtering and validation for LLM inputs/outputs.

```yaml
- id: input_guardrail
  type: guardrail
  spec:
    position: input                  # input | output
    checks:
      - type: content_filter
        block_categories:
          - hate_speech
          - self_harm
          - illegal_activity
        action: block
        message: "I'm unable to help with that request."

      - type: pii_detection
        detect: [email, phone, ssn, credit_card]
        action: mask                 # mask | block | log
        mask_format: "***"

      - type: topic_restriction
        allowed_topics:
          - contracts
          - obligations
          - account_management
          - billing
        action: redirect
        message: "I can only help with Obligo-related questions."

      - type: prompt_injection
        enabled: true
        action: block
        message: "I detected an unusual request pattern."

    on_block:
      connection: return_blocked_response

  connections:
    pass: agent_loop
    block: return_blocked_response
```

```yaml
- id: output_guardrail
  type: guardrail
  spec:
    position: output
    checks:
      - type: content_filter
        block_categories: [hate_speech]
        action: block

      - type: factuality
        enabled: true
        sources: [knowledge_base]     # Cross-check against these
        action: warn                  # warn | block

      - type: tone
        required: professional
        action: rewrite              # Ask LLM to rewrite if tone is off

      - type: no_hallucinated_urls
        action: block

      - type: schema_validation
        schema:
          type: object
          required: [message]
          properties:
            message: { type: string, max_length: 2000 }

    on_block:
      connection: return_fallback_response

  connections:
    pass: return_response
    block: return_fallback_response
```

#### Human Gate Node (✋)

Pauses execution and waits for human approval.

```yaml
- id: escalation_gate
  type: human_gate
  spec:
    notification:
      channels:
        - type: slack
          channel: "#support-escalations"
          message_template: |
            🚨 Escalation from agent
            Customer: {{$.user_id}}
            Reason: {{$.escalation_reason}}
            Conversation: {{$.conversation_summary}}
        - type: email
          to: "support-leads@obligo.io"

    approval_options:
      - id: approve
        label: "Take Over"
        description: "Human agent takes over the conversation"
      - id: reject
        label: "Send Back to AI"
        description: "Return to AI agent with instructions"
        requires_input: true         # Human can add instructions
      - id: resolve
        label: "Resolve"
        description: "Mark as resolved with a response"
        requires_input: true

    timeout:
      duration: 300                  # 5 minutes
      action: auto_escalate          # auto_escalate | auto_approve | return_error
      fallback_connection: timeout_response

    context_for_human:               # What the human sees
      - conversation_history
      - customer_account
      - agent_reasoning

  connections:
    approve: human_takeover
    reject: agent_loop               # Send back with human instructions
    resolve: return_resolved
    timeout: timeout_response
```

#### Router Node (◇◇)

Semantic routing — uses an LLM to classify and route to different sub-agents or flows.

```yaml
- id: intent_router
  type: router
  spec:
    model: claude-haiku-4-5-20251001
    routing_prompt: |
      Classify the user's message into one of these categories:
      - billing: Payment, invoices, subscription questions
      - technical: Contract analysis, obligation tracking, integrations
      - general: Account settings, feature questions, how-to
      - escalation: Angry customer, legal threats, data deletion requests

    routes:
      - id: billing
        description: "Billing and payment questions"
        connection: billing_agent
      - id: technical
        description: "Technical support"
        connection: technical_agent
      - id: general
        description: "General inquiries"
        connection: general_agent
      - id: escalation
        description: "Requires immediate human attention"
        connection: escalation_gate

    fallback_route: general          # If classification fails
    confidence_threshold: 0.7        # Below this → fallback

  connections:
    billing: billing_agent_flow
    technical: technical_agent_flow
    general: general_agent_flow
    escalation: escalation_gate
```

### Complete Agent Flow Example

```yaml
# specs/domains/support/flows/customer-support-agent.yaml

flow:
  id: customer-support-agent
  name: Customer Support Agent
  type: agent                        # ← Agent flow type
  domain: support
  description: AI-powered customer support with human escalation

trigger:
  type: http
  method: POST
  path: /api/v1/support/chat
  auth: required

agent_config:
  model: claude-sonnet-4-5-20250929
  system_prompt: |
    You are a customer support agent for Obligo, a cyber liability
    management platform. You help users with contract questions,
    obligation tracking, and account management.

    Guidelines:
    - Always greet the customer by name
    - Search the knowledge base before answering questions
    - If you can't resolve in 3 tool calls, offer to create a ticket
    - Escalate to human if customer expresses frustration
    - Never make up information about contracts or obligations

  max_iterations: 10
  temperature: 0.3

memory:
  - name: conversation
    type: conversation_history
    max_tokens: 8000
    strategy: sliding_window
  - name: user_context
    type: key_value
    load_on_start:
      - user_profile
      - recent_tickets
      - account_tier

tools:
  - id: search_kb
    name: search_kb
    description: "Search knowledge base for help articles"
    parameters:
      query: { type: string, required: true }
    implementation:
      type: service_call
      service: knowledge_base
      method: semantic_search

  - id: get_contracts
    name: get_contracts
    description: "Get user's contracts and their status"
    parameters:
      status_filter: { type: string, enum: [all, active, expiring] }
    implementation:
      type: data_store
      operation: list
      model: Contract
      filter: "tenant_id = $.tenant_id"

  - id: get_obligations
    name: get_obligations
    description: "Get obligations for a specific contract"
    parameters:
      contract_id: { type: string, required: true }
    implementation:
      type: data_store
      operation: list
      model: Obligation
      filter: "contract_id = $.contract_id"

  - id: create_ticket
    name: create_ticket
    description: "Create a support ticket for follow-up"
    parameters:
      subject: { type: string, required: true }
      description: { type: string, required: true }
      priority: { type: string, enum: [low, medium, high], default: medium }
    implementation:
      type: data_store
      operation: create
      model: SupportTicket
    requires_confirmation: false

  - id: respond_to_user
    name: respond_to_user
    description: "Send final response to the user"
    parameters:
      message: { type: string, required: true }
    is_terminal: true

  - id: escalate_to_human
    name: escalate_to_human
    description: "Escalate to human agent when AI cannot resolve"
    parameters:
      reason: { type: string, required: true }
      urgency: { type: string, enum: [normal, high, critical] }
    is_terminal: true
    implementation:
      type: event
      event: support.escalated

guardrails:
  input:
    - type: content_filter
      block_categories: [hate_speech, illegal_activity]
      action: block
    - type: pii_detection
      detect: [ssn, credit_card]
      action: mask
    - type: prompt_injection
      enabled: true
      action: block

  output:
    - type: tone
      required: professional
      action: rewrite
    - type: no_hallucinated_urls
      action: block
    - type: schema_validation
      schema:
        type: object
        required: [message]
        properties:
          message: { type: string, max_length: 2000 }

nodes:
  - id: input_guard
    type: guardrail
    spec:
      position: input
      checks: $ref: "#/guardrails/input"
    connections:
      pass: agent_loop
      block: blocked_response

  - id: agent_loop
    type: agent_loop
    spec:
      config: $ref: "#/agent_config"
      tools: $ref: "#/tools"
      memory: $ref: "#/memory"
    connections:
      respond_to_user: output_guard
      escalate_to_human: escalation_gate
      error: error_response

  - id: output_guard
    type: guardrail
    spec:
      position: output
      checks: $ref: "#/guardrails/output"
    connections:
      pass: return_response
      block: fallback_response

  - id: escalation_gate
    type: human_gate
    spec:
      notification:
        channels:
          - type: slack
            channel: "#support-escalations"
      timeout:
        duration: 300
        action: auto_escalate
    connections:
      approve: human_takeover
      reject: agent_loop
      resolve: return_resolved
      timeout: timeout_response

  - id: return_response
    type: terminal
    spec:
      status: 200
      body:
        message: "$.agent_response"
        conversation_id: "$.conversation_id"

  - id: blocked_response
    type: terminal
    spec:
      status: 400
      body:
        error: "Your message could not be processed"
        code: CONTENT_BLOCKED

  - id: error_response
    type: terminal
    spec:
      status: 500
      body:
        error: "An error occurred. Please try again."
        code: AGENT_ERROR

metadata:
  created_by: murat
  created_at: 2025-02-13
  completeness: 100
```

### Orchestration Node Specifications

#### Orchestrator Node (⊛)

A supervisor that manages multiple agents. It has its own reasoning model and decides which agent to activate, monitors their progress, and can intervene.

```yaml
- id: support_orchestrator
  type: orchestrator
  spec:
    strategy: supervisor             # supervisor | round_robin | broadcast | consensus

    # Supervisor's own reasoning model
    model: claude-sonnet-4-5-20250929
    supervisor_prompt: |
      You manage a team of support agents. Given the user's message
      and context, decide which specialist to route to. Monitor their
      progress and intervene if they get stuck.

    # Managed agents
    agents:
      - id: billing_agent
        flow: billing-agent
        domain: support
        specialization: "Billing, payments, invoices"
        priority: 1                  # Lower = higher priority when multiple match
      - id: technical_agent
        flow: technical-agent
        domain: support
        specialization: "Contract analysis, integrations, API questions"
        priority: 2
      - id: general_agent
        flow: general-agent
        domain: support
        specialization: "Account questions, how-to, general inquiries"
        priority: 3

    # How the orchestrator picks an agent
    routing:
      primary: smart_router          # Node ID of the Smart Router
      fallback_chain:                # If primary fails, try these in order
        - general_agent
        - human_escalation

    # Shared memory accessible by all managed agents
    shared_memory:
      - name: conversation
        type: conversation_history
        access: read_write           # All agents see same conversation
      - name: customer_context
        type: key_value
        access: read_only            # Agents can read, orchestrator writes
        fields: [customer_id, tier, recent_tickets, satisfaction_score]

    # Supervision rules
    supervision:
      monitor_iterations: true
      monitor_tool_calls: true
      monitor_confidence: true

      intervene_on:
        - condition: agent_iterations_exceeded
          threshold: 5
          action: reassign
          target: next_in_priority

        - condition: confidence_below
          threshold: 0.3
          action: add_instructions
          instructions_prompt: |
            The agent seems unsure. Provide additional context or
            suggest a different approach.

        - condition: customer_sentiment
          sentiment: frustrated
          action: escalate_to_human

        - condition: agent_error
          action: retry_with_different_agent
          max_retries: 1

        - condition: timeout
          threshold: 60              # seconds
          action: escalate_to_human

    # How to merge results when multiple agents contribute
    result_merge:
      strategy: last_wins            # last_wins | best_of | combine | supervisor_picks
      # best_of: supervisor evaluates all agent responses and picks best
      # combine: supervisor synthesizes a combined response
      # supervisor_picks: supervisor sees all results and writes final response

  connections:
    resolved: output_guardrail
    escalated: human_gate
    error: error_response
```

**Orchestration Strategies:**

| Strategy | How it works | Use when |
|----------|-------------|----------|
| **supervisor** | Orchestrator's LLM reasons about which agent to activate next, monitors, and can intervene | Complex routing with dynamic decisions |
| **round_robin** | Distribute requests evenly across agents | Load balancing homogeneous agents |
| **broadcast** | Send to all agents simultaneously, merge results | Need multiple perspectives (consensus, fact-checking) |
| **consensus** | All agents respond, orchestrator's LLM picks best or synthesizes | High-stakes decisions requiring agreement |

#### Smart Router Node (◇◇+)

Enhanced router with rule-based routing, LLM fallback, policies, and A/B testing. Replaces the basic Router node for orchestration scenarios.

```yaml
- id: smart_router
  type: smart_router
  spec:
    # ── Rule-based routes (evaluated first, fast, no LLM call) ──
    rules:
      - id: enterprise_route
        condition: "$.customer.tier == 'enterprise'"
        route: enterprise_agent
        priority: 1                  # Lower = evaluated first

      - id: cancellation_route
        condition: "$.intent == 'cancel' or $.message icontains 'cancel'"
        route: retention_agent
        priority: 2

      - id: billing_keywords
        condition: "$.message icontains 'invoice' or $.message icontains 'payment' or $.message icontains 'charge'"
        route: billing_agent
        priority: 3

      - id: urgent_route
        condition: "$.customer.open_tickets > 3 and $.customer.satisfaction_score < 3"
        route: human_escalation
        priority: 0                  # Highest priority

    # ── LLM-based classification (when no rule matches) ──
    llm_routing:
      enabled: true
      model: claude-haiku-4-5-20251001
      routing_prompt: |
        Classify the customer's intent into one of these categories:
        - billing: Payment, invoices, subscription, pricing
        - technical: Contract analysis, integrations, API, data
        - general: Account settings, features, how-to, feedback
        - escalation: Legal threats, data deletion, angry customer

        Customer tier: {{$.customer.tier}}
        Recent tickets: {{$.customer.open_tickets}}
        Message: {{$.message}}

        Respond with JSON: {"route": "<category>", "confidence": <0-1>}
      confidence_threshold: 0.7
      temperature: 0.0

      routes:
        billing: billing_agent
        technical: technical_agent
        general: general_agent
        escalation: human_escalation

    # ── Fallback chain (when LLM confidence < threshold or error) ──
    fallback_chain:
      - general_agent
      - human_escalation

    # ── Routing policies ──
    policies:
      retry:
        max_attempts: 2
        on_failure: next_in_fallback_chain
        delay_ms: 0

      timeout:
        per_route: 30                # seconds per agent attempt
        total: 120                   # seconds total across all retries
        action: next_in_fallback_chain

      circuit_breaker:
        enabled: true
        failure_threshold: 3         # Consecutive failures before opening
        recovery_time: 60            # Seconds before trying again
        half_open_requests: 1        # Test requests when recovering
        on_open: next_in_fallback_chain

    # ── A/B testing ──
    ab_testing:
      enabled: true
      experiments:
        - name: new_billing_agent_v2
          route: billing_agent_v2
          percentage: 20             # 20% of billing traffic
          original_route: billing_agent
          metrics:
            - resolution_rate
            - customer_satisfaction
            - average_iterations

        - name: fast_general_model
          route: general_agent_fast  # Same agent, different model
          percentage: 10
          original_route: general_agent

    # ── Context-aware routing ──
    context_routing:
      enabled: true
      rules:
        # If customer was recently talking to billing agent, route back there
        - condition: "$.session.last_agent == 'billing_agent' and $.session.turns_since < 3"
          route: billing_agent
          reason: "Continue with same agent for conversation continuity"

        # If this is a follow-up to an escalated conversation
        - condition: "$.session.was_escalated == true"
          route: human_escalation
          reason: "Previously escalated, go directly to human"

  connections:
    billing_agent: billing_agent_flow
    technical_agent: technical_agent_flow
    general_agent: general_agent_flow
    enterprise_agent: enterprise_agent_flow
    retention_agent: retention_agent_flow
    human_escalation: escalation_gate
    billing_agent_v2: billing_agent_v2_flow
    general_agent_fast: general_agent_fast_flow
```

#### Handoff Node (⇄)

Formal protocol for transferring context between agents.

```yaml
- id: handoff_to_specialist
  type: handoff
  spec:
    # ── Handoff Mode ──
    mode: consult                    # transfer | consult | collaborate

    # transfer: Source agent stops, target takes over completely
    # consult:  Source agent pauses, target answers, result returns to source
    # collaborate: Both agents active, shared context, orchestrator merges

    # ── Target ──
    target:
      flow: technical-agent
      domain: support

    # ── Context Transfer ──
    context_transfer:
      # What gets passed to the target agent
      include:
        - type: conversation_summary
          generator: llm              # LLM generates summary (not raw history)
          model: claude-haiku-4-5-20251001
          max_tokens: 500
          prompt: |
            Summarize this conversation for a specialist agent.
            Focus on: what the customer needs, what's been tried,
            and any relevant account details.

        - type: structured_data
          fields:
            customer_id: "$.customer.id"
            customer_tier: "$.customer.tier"
            original_intent: "$.routing.classified_intent"
            attempted_solutions: "$.agent.tool_calls_summary"

        - type: task_description
          content: |
            The customer needs help with {{$.routing.classified_intent}}.
            Previous agent could not resolve. Please assist.

      exclude:
        - internal_reasoning          # Don't share source agent's scratchpad
        - raw_tool_results            # Only send summaries
        - system_prompts              # Each agent has its own

      max_context_tokens: 2000       # Hard limit on transferred context

    # ── What happens when target finishes ──
    on_complete:
      return_to: source_agent        # source_agent | orchestrator | terminal
      merge_strategy: append         # append | replace | summarize

      # For 'summarize': LLM summarizes the specialist's work
      summarize_prompt: |
        Summarize what the specialist found and recommended.
        Keep it concise for the customer.

    # ── Failure handling ──
    on_failure:
      action: return_with_error      # return_with_error | retry | escalate
      fallback: escalate_to_human
      timeout: 60                    # seconds

    # ── Handoff notification (optional) ──
    notify_customer: true
    notification_message: |
      I'm connecting you with a specialist who can better help
      with your {{$.routing.classified_intent}} question.

  connections:
    complete: return_to_orchestrator
    failure: escalation_gate
    timeout: timeout_response
```

**Handoff Modes:**

| Mode | Source Agent | Target Agent | Context Flow | Use When |
|------|-------------|-------------|--------------|----------|
| **transfer** | Stops completely | Takes over | One-way: source → target | Routing to specialist, escalation |
| **consult** | Pauses, waits | Answers specific question | Round-trip: source → target → source | Need expert opinion, then continue |
| **collaborate** | Stays active | Also active | Bidirectional via shared memory | Complex tasks needing multiple specialists |

#### Agent Group Node ([⊛])

A container that groups agents sharing memory and coordination policy. On Level 2, rendered as a visual boundary around grouped agent flows.

```yaml
- id: support_team
  type: agent_group
  spec:
    name: Support Team
    description: Customer support agents working together

    # Agents in this group
    members:
      - flow: billing-agent
      - flow: technical-agent
      - flow: general-agent
      - flow: retention-agent

    # Shared resources within the group
    shared_memory:
      - name: conversation
        type: conversation_history
        access: read_write
        description: "All agents see the same conversation thread"
      - name: customer_context
        type: key_value
        access: read_only
        fields: [customer_id, tier, history_summary]
      - name: team_knowledge
        type: vector_store
        provider: pgvector
        access: read_only
        description: "Shared knowledge base for all support agents"

    # Coordination policy
    coordination:
      # How agents communicate within the group
      communication: via_orchestrator  # via_orchestrator | direct | blackboard

      # via_orchestrator: All inter-agent communication goes through the orchestrator
      # direct: Agents can call each other via handoff nodes
      # blackboard: Agents read/write to a shared blackboard (shared_memory)

      # Concurrency rules
      concurrency:
        max_active_agents: 1         # How many agents can be active at once
        # 1 = sequential (one at a time, typical for chat)
        # N = parallel (multiple working simultaneously, for batch/analysis)

      # Agent selection when multiple could handle a task
      selection:
        strategy: router_first       # router_first | round_robin | least_busy | random
        sticky_session: true         # Keep same agent for a conversation if possible
        sticky_timeout: 300          # Seconds before sticky session expires

    # Group-level guardrails (applied to all agents in group)
    guardrails:
      input:
        - type: content_filter
          block_categories: [hate_speech, illegal_activity]
      output:
        - type: tone
          required: professional

    # Group-level metrics (for A/B testing, monitoring)
    metrics:
      track:
        - resolution_rate
        - average_iterations
        - handoff_count
        - escalation_rate
        - customer_satisfaction
```

### Orchestration Visualization (Level 2)

When a domain contains orchestrated agents, the Level 2 Domain Map shows the relationships:

```
┌─────────────────────────────────────────────────────────────────┐
│ DOMAIN MAP: support                                              │
│                                                                   │
│   ┌─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐         │
│   │  [⊛] Support Team (Agent Group)                   │         │
│   │                                                    │         │
│   │   ⊛ Support Orchestrator                          │         │
│   │   │ (supervisor)                                   │         │
│   │   │                                                │         │
│   │   ├──⊛▶── ┌──────────┐                           │         │
│   │   │        │▣⊛ billing │                           │         │
│   │   │        │   agent   │ ─── ⇄ consult ──┐       │         │
│   │   │        └──────────┘                   │       │         │
│   │   │                                       ▼       │         │
│   │   ├──⊛▶── ┌──────────┐           ┌──────────┐   │         │
│   │   │        │▣⊛ tech    │ ◀── ⇄ ──│▣⊛ general │   │         │
│   │   │        │   agent   │ consult  │   agent   │   │         │
│   │   │        └──────────┘           └──────────┘   │         │
│   │   │                                               │         │
│   │   └──⊛▶── ┌──────────┐                           │         │
│   │            │▣⊛retention│                           │         │
│   │            │   agent   │                           │         │
│   │            └──────────┘                           │         │
│   │                                                    │         │
│   │   ◈ Shared: conversation, customer_context        │         │
│   └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘         │
│                    │                                              │
│                    │ ⇄ transfer (escalation)                     │
│                    ▼                                              │
│              ✋ Human Gate                                        │
│                                                                   │
│   ╔════════════╗     ╔════════════╗                              │
│   ║ ➜ billing  ║     ║ ➜ analysis ║                              │
│   ╚════════════╝     ╚════════════╝                              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Level 2 orchestration elements:**

| Element | Visual | Description |
|---------|--------|-------------|
| Agent Group boundary | Dashed rectangle | Groups agents sharing memory/supervisor |
| Orchestrator | ⊛ block at top of group | The supervisor agent managing the group |
| Supervisor arrow (⊛▶) | Solid arrow from orchestrator | Shows which agents are managed |
| Handoff arrow (⇄) | Bidirectional dashed arrow | Shows agent-to-agent handoff with mode label |
| Agent flow block (▣⊛) | Block with agent badge | Distinguishes agent flows from traditional flows |
| Shared memory indicator (◈) | Badge at bottom of group | Shows what memory is shared within the group |

### Complete Orchestration Flow Example

```yaml
# specs/domains/support/flows/support-orchestration.yaml

flow:
  id: support-orchestration
  name: Support Team Orchestration
  type: agent                        # Agent flow type
  domain: support
  description: >
    Orchestrated customer support with smart routing,
    specialist agents, handoffs, and human escalation.

trigger:
  type: http
  method: POST
  path: /api/v1/support/chat
  auth: required

# ── Agent Group Definition ──
agent_group:
  id: support_team
  name: Support Team
  shared_memory:
    - name: conversation
      type: conversation_history
      access: read_write
    - name: customer_context
      type: key_value
      access: read_only
      load_on_start: [customer_profile, recent_tickets, satisfaction_score]
  coordination:
    communication: via_orchestrator
    concurrency:
      max_active_agents: 1
    selection:
      sticky_session: true
      sticky_timeout: 300

# ── Nodes ──
nodes:
  # Input guardrail
  - id: input_guard
    type: guardrail
    spec:
      position: input
      checks:
        - type: content_filter
          block_categories: [hate_speech, illegal_activity]
          action: block
        - type: pii_detection
          detect: [ssn, credit_card]
          action: mask
        - type: prompt_injection
          enabled: true
          action: block
    connections:
      pass: orchestrator
      block: blocked_response

  # Orchestrator — the brain
  - id: orchestrator
    type: orchestrator
    spec:
      strategy: supervisor
      model: claude-sonnet-4-5-20250929
      supervisor_prompt: |
        You manage a customer support team. Route requests to the
        right specialist. Monitor their work. Intervene if needed.

      agents:
        - id: billing_agent
          flow: billing-agent
          specialization: "Billing, payments, invoices, subscriptions"
          priority: 1
        - id: technical_agent
          flow: technical-agent
          specialization: "Contracts, obligations, integrations, API"
          priority: 2
        - id: general_agent
          flow: general-agent
          specialization: "Account settings, features, how-to"
          priority: 3
        - id: retention_agent
          flow: retention-agent
          specialization: "Cancellations, complaints, win-back"
          priority: 1

      routing:
        primary: smart_router
        fallback_chain: [general_agent, human_gate]

      supervision:
        monitor_iterations: true
        intervene_on:
          - condition: agent_iterations_exceeded
            threshold: 5
            action: reassign
          - condition: customer_sentiment
            sentiment: frustrated
            action: escalate_to_human
          - condition: timeout
            threshold: 60
            action: escalate_to_human

      result_merge:
        strategy: last_wins

    connections:
      resolved: output_guard
      escalated: human_gate
      error: error_response

  # Smart Router — decides which agent handles the request
  - id: smart_router
    type: smart_router
    spec:
      rules:
        - id: urgent
          condition: "$.customer.open_tickets > 3 and $.customer.satisfaction_score < 3"
          route: human_gate
          priority: 0
        - id: enterprise
          condition: "$.customer.tier == 'enterprise'"
          route: technical_agent
          priority: 1
        - id: cancel_intent
          condition: "$.message icontains 'cancel' or $.message icontains 'downgrade'"
          route: retention_agent
          priority: 2
        - id: billing_keywords
          condition: "$.message icontains 'invoice' or $.message icontains 'payment'"
          route: billing_agent
          priority: 3

      llm_routing:
        enabled: true
        model: claude-haiku-4-5-20251001
        confidence_threshold: 0.7
        routes:
          billing: billing_agent
          technical: technical_agent
          general: general_agent
          escalation: human_gate

      fallback_chain: [general_agent]

      policies:
        timeout:
          per_route: 30
          total: 120
        circuit_breaker:
          enabled: true
          failure_threshold: 3
          recovery_time: 60

      context_routing:
        enabled: true
        rules:
          - condition: "$.session.last_agent != null and $.session.turns_since < 3"
            route: "$.session.last_agent"
            reason: "Conversation continuity"

      ab_testing:
        enabled: false

  # Handoff — when billing agent needs technical help
  - id: billing_to_tech_handoff
    type: handoff
    spec:
      mode: consult
      target:
        flow: technical-agent
        domain: support
      context_transfer:
        include:
          - type: conversation_summary
            generator: llm
            model: claude-haiku-4-5-20251001
            max_tokens: 500
          - type: structured_data
            fields:
              customer_id: "$.customer.id"
              billing_context: "$.agent.findings"
        max_context_tokens: 2000
      on_complete:
        return_to: source_agent
        merge_strategy: append
      on_failure:
        action: return_with_error
        timeout: 60

  # Human Gate
  - id: human_gate
    type: human_gate
    spec:
      notification:
        channels:
          - type: slack
            channel: "#support-escalations"
          - type: email
            to: "support-leads@obligo.io"
      approval_options:
        - id: take_over
          label: "Take Over"
          description: "Human agent takes the conversation"
        - id: send_back
          label: "Send Back"
          description: "Return to AI with instructions"
          requires_input: true
        - id: resolve
          label: "Resolve"
          description: "Mark as resolved"
          requires_input: true
      timeout:
        duration: 300
        action: auto_escalate
      context_for_human:
        - conversation_history
        - customer_account
        - agent_reasoning
        - routing_decisions
    connections:
      take_over: human_takeover_terminal
      send_back: orchestrator
      resolve: resolved_terminal
      timeout: timeout_terminal

  # Output guardrail
  - id: output_guard
    type: guardrail
    spec:
      position: output
      checks:
        - type: tone
          required: professional
          action: rewrite
        - type: no_hallucinated_urls
          action: block
        - type: schema_validation
          schema:
            type: object
            required: [message]
            properties:
              message: { type: string, max_length: 2000 }
    connections:
      pass: resolved_terminal
      block: fallback_terminal

  # Terminals
  - id: resolved_terminal
    type: terminal
    spec:
      status: 200
      body:
        message: "$.agent_response"
        conversation_id: "$.conversation_id"
        handled_by: "$.routing.agent_id"

  - id: blocked_response
    type: terminal
    spec:
      status: 400
      body: { error: "Message could not be processed", code: CONTENT_BLOCKED }

  - id: error_response
    type: terminal
    spec:
      status: 500
      body: { error: "An error occurred", code: AGENT_ERROR }

  - id: human_takeover_terminal
    type: terminal
    spec:
      status: 200
      body:
        message: "You've been connected with a human agent."
        conversation_id: "$.conversation_id"

  - id: timeout_terminal
    type: terminal
    spec:
      status: 504
      body: { error: "Request timed out", code: ESCALATION_TIMEOUT }

  - id: fallback_terminal
    type: terminal
    spec:
      status: 200
      body:
        message: "I apologize, let me connect you with a team member who can help."
        conversation_id: "$.conversation_id"

metadata:
  created_by: murat
  created_at: 2025-02-13
  completeness: 100
```

### Orchestration Patterns Summary

| Pattern | Nodes Used | Level 2 Visual | Description |
|---------|-----------|----------------|-------------|
| **Supervisor** | Orchestrator (⊛) + Agent Group ([⊛]) | Group boundary + supervisor arrows | One orchestrator monitors and manages multiple agents |
| **Router → Specialists** | Smart Router (◇◇+) → Agent flows | Router block → flow blocks | Classify then delegate to best agent |
| **Consult** | Handoff (⇄) in consult mode | Bidirectional dashed arrow | Agent A asks Agent B, gets answer back |
| **Transfer** | Handoff (⇄) in transfer mode | One-way dashed arrow | Agent A hands off completely to Agent B |
| **Collaborate** | Handoff (⇄) in collaborate mode + shared memory | Bidirectional arrow + shared memory badge | Both agents active with shared context |
| **Pipeline** | Event/sub-flow connections | Sequential flow blocks | Agent A → Agent B → Agent C |
| **Broadcast** | Orchestrator (strategy: broadcast) | Multiple supervisor arrows | Send to all, merge results |
| **Consensus** | Orchestrator (strategy: consensus) | Multiple supervisor arrows + merge indicator | All respond, supervisor picks best |
| **Fallback Chain** | Smart Router fallback_chain | Dashed arrows labeled "fallback" | Try A, if fails try B, then C |
| **A/B Testing** | Smart Router ab_testing | Split arrow with percentage labels | Route N% to experimental agent |
| **Human-in-the-Loop** | Human Gate (✋) | Gate block with approval options | Pause for human decision |
| **Circuit Breaker** | Smart Router policies.circuit_breaker | Status indicator on route arrows | Stop routing to failing agent |

### Agent & Orchestration Error Codes

```yaml
# Add to specs/shared/errors.yaml

agent:
  AGENT_ERROR:
    http_status: 500
    message_template: "Agent encountered an error"
    log_level: ERROR

  AGENT_MAX_ITERATIONS:
    http_status: 500
    message_template: "Agent did not converge within {max_iterations} iterations"
    log_level: WARNING

  AGENT_TOOL_ERROR:
    http_status: 500
    message_template: "Agent tool '{tool_name}' failed: {reason}"
    log_level: ERROR

  CONTENT_BLOCKED:
    http_status: 400
    message_template: "Content blocked by guardrail: {guardrail}"

  GUARDRAIL_FAILED:
    http_status: 422
    message_template: "Output did not pass guardrail check: {check}"

  ESCALATION_TIMEOUT:
    http_status: 504
    message_template: "Human escalation timed out after {timeout} seconds"

  ROUTING_FAILED:
    http_status: 500
    message_template: "Intent router could not classify request"
    log_level: WARNING

orchestration:
  ORCHESTRATOR_ERROR:
    http_status: 500
    message_template: "Orchestrator failed: {reason}"
    log_level: ERROR

  AGENT_REASSIGNED:
    http_status: 200
    message_template: "Request reassigned from {from_agent} to {to_agent}"
    log_level: INFO

  HANDOFF_FAILED:
    http_status: 500
    message_template: "Handoff to {target_agent} failed: {reason}"
    log_level: ERROR

  HANDOFF_TIMEOUT:
    http_status: 504
    message_template: "Handoff to {target_agent} timed out after {timeout}s"
    log_level: WARNING

  CIRCUIT_BREAKER_OPEN:
    http_status: 503
    message_template: "Route to {agent} is temporarily unavailable (circuit open)"
    log_level: WARNING

  ALL_ROUTES_EXHAUSTED:
    http_status: 503
    message_template: "All routes exhausted including fallback chain"
    log_level: ERROR

  CONTEXT_TRANSFER_FAILED:
    http_status: 500
    message_template: "Failed to transfer context to {target_agent}: {reason}"
    log_level: ERROR

  SUPERVISION_INTERVENTION:
    http_status: 200
    message_template: "Supervisor intervened: {intervention_reason}"
    log_level: INFO
```

---

# Part 5: DDD Format Specification v1.0

## Spec Hierarchy (Complete)

```
specs/
├── system.yaml              # Project identity, tech stack, domains
├── system-layout.yaml       # System Map (L1) block positions (managed by DDD Tool)
├── architecture.yaml        # Project structure, infrastructure, cross-cutting
├── config.yaml              # Environment variables, secrets schema
├── schemas/
│   ├── _base.yaml           # Base model (id, timestamps, soft delete)
│   ├── contract.yaml
│   ├── obligation.yaml
│   └── events/
│       └── contract-ingested.yaml
├── shared/
│   ├── auth.yaml            # Authentication/authorization spec
│   ├── middleware.yaml      # Middleware stack spec
│   ├── errors.yaml          # Error codes and response format (incl. agent errors)
│   └── api.yaml             # API conventions (pagination, filtering)
└── domains/
    └── {domain}/
        ├── domain.yaml      # Domain info, flows, events, L2 layout positions
        └── flows/
            ├── {flow}.yaml          # Traditional flow (type: traditional)
            └── {agent-flow}.yaml    # Agent flow (type: agent) — includes tools, memory, guardrails
```

## System Config

```yaml
# specs/system.yaml
system:
  name: obligo
  version: 1.0.0
  description: Cyber Liability Operating System
  
  tech_stack:
    language: python
    language_version: "3.11"
    framework: fastapi
    orm: sqlalchemy
    database: postgresql
    cache: redis
    queue: rabbitmq
    
  domains:
    - name: ingestion
      description: Webhook and data ingestion from CLM providers
    - name: analysis
      description: Contract analysis and obligation extraction
    - name: api
      description: REST API for frontend and integrations
    - name: notification
      description: Email, Slack, and webhook notifications
```

## Domain Config (Drives Level 1 & Level 2 Sheets)

Each domain has a `domain.yaml` that declares its flows, published events, and consumed events. This data drives the System Map (Level 1) and Domain Map (Level 2) visualizations automatically.

```yaml
# specs/domains/ingestion/domain.yaml
domain:
  name: ingestion
  description: Webhook and data ingestion from CLM providers

  flows:
    - id: webhook-ingestion
      name: Webhook Ingestion
      description: Receives and processes webhooks from CLM providers
    - id: scheduled-sync
      name: Scheduled Sync
      description: Periodic sync with CLM provider APIs

  # Events this domain publishes (outgoing arrows on System Map)
  publishes_events:
    - event: contract.ingested
      from_flow: webhook-ingestion
      description: Fired when a new contract is received via webhook
    - event: contract.synced
      from_flow: scheduled-sync
      description: Fired when a contract is synced from provider API

  # Events this domain consumes (incoming arrows on System Map)
  consumes_events:
    - event: analysis.completed
      handled_by_flow: webhook-ingestion  # or a dedicated flow
      description: Notification that analysis of an ingested contract is done

  # Layout positions for Domain Map (Level 2) — managed by DDD Tool
  layout:
    flows:
      webhook-ingestion: { x: 100, y: 100 }
      scheduled-sync: { x: 400, y: 100 }
    portals:
      analysis: { x: 100, y: 300 }
```

```yaml
# specs/domains/analysis/domain.yaml
domain:
  name: analysis
  description: Contract analysis and obligation extraction

  flows:
    - id: extract-obligations
      name: Extract Obligations
      description: Extract obligations from contract using LLM
    - id: classify-risk
      name: Classify Risk
      description: Classify risk level of extracted obligations

  publishes_events:
    - event: analysis.completed
      from_flow: extract-obligations
      description: Fired when obligation extraction is complete
    - event: risk.classified
      from_flow: classify-risk
      description: Fired when risk classification is complete

  consumes_events:
    - event: contract.ingested
      handled_by_flow: extract-obligations
      description: Triggers obligation extraction for new contracts
    - event: contract.synced
      handled_by_flow: extract-obligations
      description: Triggers obligation extraction for synced contracts

  layout:
    flows:
      extract-obligations: { x: 100, y: 100 }
      classify-risk: { x: 400, y: 100 }
    portals:
      ingestion: { x: 100, y: -100 }
      notification: { x: 400, y: 300 }
```

### System Map Layout

The System Map (Level 1) also stores layout positions for domain blocks:

```yaml
# specs/system-layout.yaml (managed by DDD Tool)
system_layout:
  domains:
    ingestion: { x: 100, y: 200 }
    analysis: { x: 400, y: 200 }
    api: { x: 100, y: 500 }
    notification: { x: 400, y: 500 }
```

This file is auto-managed by the DDD Tool when users reposition domain blocks on the System Map canvas.

## Architecture Config (Critical for Code Generation)

```yaml
# specs/architecture.yaml

architecture:
  version: 1.0.0
  
  # ═══════════════════════════════════════════════════════════════════════════
  # PROJECT STRUCTURE
  # ═══════════════════════════════════════════════════════════════════════════
  
  structure:
    type: domain-driven  # domain-driven | feature-based | layered
    
    layout:
      src:
        domains:
          "{domain}":
            router: router.py
            schemas: schemas.py
            services: services.py
            models: models.py
            events: events.py
            exceptions: exceptions.py
        shared:
          auth: auth/
          database: database/
          events: events/
          middleware: middleware/
          exceptions: exceptions/
          utils: utils/
        config:
          settings: settings.py
          logging: logging.py
      tests:
        unit:
          domains: "{domain}/"
        integration: integration/
        e2e: e2e/
        fixtures: fixtures/
        factories: factories/
      migrations:
        versions: versions/
      scripts: scripts/
      
    naming:
      files: snake_case
      classes: PascalCase
      functions: snake_case
      variables: snake_case
      constants: SCREAMING_SNAKE_CASE
      database_tables: plural_snake_case
      database_columns: snake_case
      api_endpoints: kebab-case
      
  # ═══════════════════════════════════════════════════════════════════════════
  # DEPENDENCIES
  # ═══════════════════════════════════════════════════════════════════════════
  
  dependencies:
    production:
      # Framework
      fastapi: "^0.109.0"
      uvicorn: "^0.27.0"
      
      # Data validation
      pydantic: "^2.5.0"
      pydantic-settings: "^2.1.0"
      
      # Database
      sqlalchemy: "^2.0.25"
      alembic: "^1.13.0"
      asyncpg: "^0.29.0"
      
      # Cache & Queue
      redis: "^5.0.0"
      aio-pika: "^9.3.0"  # RabbitMQ
      
      # HTTP client
      httpx: "^0.26.0"
      
      # Auth
      python-jose: "^3.3.0"
      passlib: "^1.7.4"
      bcrypt: "^4.1.0"
      
      # Observability
      structlog: "^24.1.0"
      opentelemetry-api: "^1.22.0"
      
    development:
      pytest: "^7.4.0"
      pytest-asyncio: "^0.23.0"
      pytest-cov: "^4.1.0"
      factory-boy: "^3.3.0"
      fakeredis: "^2.20.0"
      ruff: "^0.1.0"
      mypy: "^1.8.0"
      pre-commit: "^3.6.0"
      
  # ═══════════════════════════════════════════════════════════════════════════
  # INFRASTRUCTURE
  # ═══════════════════════════════════════════════════════════════════════════
  
  infrastructure:
    database:
      type: postgresql
      async: true
      pool:
        min_size: 5
        max_size: 20
        max_overflow: 10
        pool_timeout: 30
      conventions:
        primary_key: uuid
        timestamps: true        # created_at, updated_at on all
        soft_delete: true       # deleted_at instead of DELETE
        audit_log: false        # Per-table override available
        
    cache:
      type: redis
      default_ttl: 3600
      key_prefix: "{system_name}:"
      serializer: json
      
    queue:
      type: rabbitmq
      exchange: events
      exchange_type: topic
      durable: true
      retry:
        max_attempts: 3
        backoff_type: exponential
        initial_delay: 1
        max_delay: 60
        dead_letter: true
        
    storage:
      type: s3
      provider: aws           # aws | gcs | minio | cloudflare
      
  # ═══════════════════════════════════════════════════════════════════════════
  # CROSS-CUTTING CONCERNS
  # ═══════════════════════════════════════════════════════════════════════════
  
  cross_cutting:
    
    authentication:
      type: jwt
      algorithm: RS256
      access_token:
        ttl: 900              # 15 minutes
        location: header
        header_name: Authorization
        scheme: Bearer
      refresh_token:
        ttl: 604800           # 7 days
        location: cookie
        cookie_name: refresh_token
        http_only: true
        secure: true
        same_site: lax
        
    authorization:
      type: rbac
      roles:
        - name: admin
          description: Full system access
        - name: owner
          description: Tenant owner
        - name: member
          description: Regular team member
        - name: viewer
          description: Read-only access
      default_role: member
      super_admin_bypass: true
      
    multi_tenancy:
      enabled: true
      strategy: row_level     # row_level | schema | database
      identifier: tenant_id
      header: X-Tenant-ID
      enforce_on_all_queries: true
      
    logging:
      format: structured
      output: json
      include:
        always:
          - timestamp
          - level
          - message
          - logger
          - request_id
          - tenant_id
        on_request:
          - method
          - path
          - status_code
          - duration_ms
          - user_id
        on_error:
          - error_type
          - error_code
          - stack_trace
      exclude_paths:
        - /health
        - /ready
        - /metrics
      levels:
        root: INFO
        sqlalchemy.engine: WARNING
        httpx: WARNING
        uvicorn.access: WARNING
        
    error_handling:
      response_format:
        error:
          type: string
          description: Human-readable error message
        code:
          type: string
          description: Machine-readable error code
        details:
          type: object
          description: Additional error context
        request_id:
          type: string
          description: Request ID for support
      error_codes:
        VALIDATION_ERROR:
          http_status: 422
          description: Request validation failed
        NOT_FOUND:
          http_status: 404
          description: Resource not found
        UNAUTHORIZED:
          http_status: 401
          description: Authentication required
        FORBIDDEN:
          http_status: 403
          description: Permission denied
        CONFLICT:
          http_status: 409
          description: Resource conflict
        RATE_LIMITED:
          http_status: 429
          description: Rate limit exceeded
        INTERNAL_ERROR:
          http_status: 500
          description: Internal server error
      include_stack_trace:
        development: true
        staging: true
        production: false
        
    rate_limiting:
      enabled: true
      storage: redis
      default:
        requests: 100
        window_seconds: 60
      by_tier:
        free:
          requests: 100
          window_seconds: 60
        pro:
          requests: 1000
          window_seconds: 60
        enterprise:
          requests: 10000
          window_seconds: 60
      headers:
        limit: X-RateLimit-Limit
        remaining: X-RateLimit-Remaining
        reset: X-RateLimit-Reset
        
    request_context:
      fields:
        - request_id          # Auto-generated UUID
        - tenant_id           # From header or JWT
        - user_id             # From JWT
        - roles               # From JWT
        - trace_id            # For distributed tracing
        
    middleware_order:
      # Order matters - first to last
      - cors
      - request_id
      - logging_start
      - error_handler
      - rate_limiter
      - authentication
      - tenant_context
      - authorization
      - logging_end
      
  # ═══════════════════════════════════════════════════════════════════════════
  # API DESIGN
  # ═══════════════════════════════════════════════════════════════════════════
  
  api:
    base_path: /api
    versioning:
      strategy: url_prefix    # url_prefix | header | query_param
      current_version: v1
      header_name: X-API-Version
      
    pagination:
      style: cursor           # cursor | offset
      default_limit: 20
      max_limit: 100
      params:
        limit: limit
        cursor: cursor
        offset: offset        # For offset style
      response:
        items: items
        total: total          # Only for offset style
        next_cursor: next_cursor
        prev_cursor: prev_cursor
        has_more: has_more
        
    filtering:
      style: query_params
      operators:
        eq: ""                # ?status=active
        ne: __ne              # ?status__ne=deleted
        gt: __gt              # ?amount__gt=100
        gte: __gte
        lt: __lt
        lte: __lte
        in: __in              # ?status__in=active,pending
        nin: __nin
        contains: __contains
        icontains: __icontains
        starts_with: __startswith
        is_null: __isnull     # ?deleted_at__isnull=true
        
    sorting:
      param: sort
      format: field:direction  # ?sort=created_at:desc
      multi_sort: true         # ?sort=status:asc,created_at:desc
      max_sort_fields: 3
      default: created_at:desc
      
    response_envelope:
      success_single:
        data: object
      success_list:
        data: array
        meta:
          pagination: object
          filters: object
      error:
        error: string
        code: string
        details: object
        request_id: string
        
    cors:
      allowed_origins:
        development: ["http://localhost:3000", "http://localhost:5173"]
        production: ["https://app.obligo.io"]
      allowed_methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]
      allowed_headers: ["*"]
      allow_credentials: true
      max_age: 600
      
  # ═══════════════════════════════════════════════════════════════════════════
  # TESTING
  # ═══════════════════════════════════════════════════════════════════════════
  
  testing:
    framework: pytest
    async_mode: auto
    
    coverage:
      minimum: 80
      fail_under: true
      exclude:
        - "*/migrations/*"
        - "*/config/*"
        - "*/__init__.py"
        
    database:
      strategy: transaction_rollback  # transaction_rollback | truncate | recreate
      use_test_database: true
      
    fixtures:
      auto_use:
        - db_session
        - test_client
        - authenticated_client
        
    factories:
      library: factory_boy
      base_class: AsyncFactory
      
    mocks:
      external_apis: true
      redis: fakeredis
      s3: moto
      time: freezegun
      
    markers:
      - unit: Unit tests (no IO)
      - integration: Integration tests (with DB)
      - e2e: End-to-end tests
      - slow: Slow tests (skip in CI fast mode)
      
  # ═══════════════════════════════════════════════════════════════════════════
  # DEPLOYMENT
  # ═══════════════════════════════════════════════════════════════════════════
  
  deployment:
    containerization:
      enabled: true
      runtime: docker
      base_image: python:3.11-slim
      multi_stage: true
      
    environments:
      development:
        debug: true
        log_level: DEBUG
        replicas: 1
      staging:
        debug: false
        log_level: INFO
        replicas: 2
      production:
        debug: false
        log_level: INFO
        replicas: 3
        auto_scaling:
          min: 3
          max: 10
          target_cpu: 70
          
    health_checks:
      liveness:
        path: /health
        interval: 10
        timeout: 5
        failure_threshold: 3
      readiness:
        path: /ready
        interval: 5
        timeout: 3
        failure_threshold: 3
        
    ci_cd:
      provider: github_actions
      branches:
        main: production
        staging: staging
        develop: development
      stages:
        - name: lint
          run: ruff check .
        - name: type_check
          run: mypy src/
        - name: test
          run: pytest --cov
        - name: build
          run: docker build
        - name: deploy
          run: kubectl apply
```

## Config Schema (Environment Variables)

```yaml
# specs/config.yaml

config:
  # Required environment variables
  required:
    DATABASE_URL:
      type: string
      format: "postgresql+asyncpg://{user}:{password}@{host}:{port}/{database}"
      description: PostgreSQL connection string
      example: "postgresql+asyncpg://user:pass@localhost:5432/obligo"
      
    REDIS_URL:
      type: string
      format: "redis://{host}:{port}/{db}"
      description: Redis connection string
      example: "redis://localhost:6379/0"
      
    RABBITMQ_URL:
      type: string
      format: "amqp://{user}:{password}@{host}:{port}/{vhost}"
      description: RabbitMQ connection string
      example: "amqp://guest:guest@localhost:5672/"
      
    JWT_PRIVATE_KEY:
      type: string
      sensitive: true
      description: RSA private key for signing JWTs (PEM format)
      
    JWT_PUBLIC_KEY:
      type: string
      sensitive: true
      description: RSA public key for verifying JWTs (PEM format)
      
  # Optional with defaults
  optional:
    ENVIRONMENT:
      type: string
      default: development
      enum: [development, staging, production]
      
    LOG_LEVEL:
      type: string
      default: INFO
      enum: [DEBUG, INFO, WARNING, ERROR, CRITICAL]
      
    LOG_FORMAT:
      type: string
      default: json
      enum: [json, text]
      
    PORT:
      type: integer
      default: 8000
      
    CORS_ORIGINS:
      type: array
      default: ["http://localhost:3000"]
      description: Allowed CORS origins (comma-separated in env)
      
    RATE_LIMIT_ENABLED:
      type: boolean
      default: true
      
    SENTRY_DSN:
      type: string
      sensitive: true
      description: Sentry DSN for error tracking (optional)
      
    AWS_ACCESS_KEY_ID:
      type: string
      sensitive: true
      description: AWS access key for S3
      
    AWS_SECRET_ACCESS_KEY:
      type: string
      sensitive: true
      description: AWS secret key for S3
      
    S3_BUCKET:
      type: string
      default: obligo-uploads
```

## Base Model Spec

```yaml
# specs/schemas/_base.yaml

base_model:
  name: BaseModel
  description: Base model inherited by all database models
  
  fields:
    id:
      type: uuid
      primary_key: true
      default: uuid4
      description: Unique identifier
      
    created_at:
      type: datetime
      default: now
      nullable: false
      index: true
      description: Record creation timestamp
      
    updated_at:
      type: datetime
      default: now
      on_update: now
      nullable: false
      description: Last update timestamp
      
    deleted_at:
      type: datetime
      nullable: true
      index: true
      description: Soft delete timestamp (null = not deleted)
      
  behaviors:
    soft_delete:
      enabled: true
      field: deleted_at
      
    timestamps:
      created_field: created_at
      updated_field: updated_at
      
    default_query_filter: "deleted_at IS NULL"
    
  mixins:
    - TimestampMixin
    - SoftDeleteMixin
```

## Shared Error Codes

```yaml
# specs/shared/errors.yaml

errors:
  # Validation errors (422)
  VALIDATION_ERROR:
    http_status: 422
    message_template: "Validation failed"
    
  INVALID_INPUT:
    http_status: 422
    message_template: "Invalid input: {details}"
    
  MISSING_FIELD:
    http_status: 422
    message_template: "Missing required field: {field}"
    
  INVALID_FORMAT:
    http_status: 422
    message_template: "Invalid format for {field}: {reason}"
    
  # Authentication errors (401)
  UNAUTHORIZED:
    http_status: 401
    message_template: "Authentication required"
    
  INVALID_TOKEN:
    http_status: 401
    message_template: "Invalid or expired token"
    
  TOKEN_EXPIRED:
    http_status: 401
    message_template: "Token has expired"
    
  # Authorization errors (403)
  FORBIDDEN:
    http_status: 403
    message_template: "You don't have permission to perform this action"
    
  INSUFFICIENT_ROLE:
    http_status: 403
    message_template: "Required role: {required_role}"
    
  TENANT_MISMATCH:
    http_status: 403
    message_template: "Access denied to this tenant's resources"
    
  # Not found errors (404)
  NOT_FOUND:
    http_status: 404
    message_template: "Resource not found"
    
  ENTITY_NOT_FOUND:
    http_status: 404
    message_template: "{entity} with id {id} not found"
    
  # Conflict errors (409)
  CONFLICT:
    http_status: 409
    message_template: "Resource conflict"
    
  DUPLICATE_ENTRY:
    http_status: 409
    message_template: "{entity} with {field}={value} already exists"
    
  # Rate limiting (429)
  RATE_LIMITED:
    http_status: 429
    message_template: "Rate limit exceeded. Retry after {retry_after} seconds"
    headers:
      Retry-After: "{retry_after}"
      
  # Server errors (500)
  INTERNAL_ERROR:
    http_status: 500
    message_template: "An unexpected error occurred"
    log_level: ERROR
    
  DATABASE_ERROR:
    http_status: 500
    message_template: "Database operation failed"
    log_level: ERROR
    
  EXTERNAL_SERVICE_ERROR:
    http_status: 502
    message_template: "External service unavailable: {service}"
    log_level: WARNING
```

## Flow Spec

```yaml
# specs/domains/ingestion/flows/webhook-ingestion.yaml
flow:
  id: webhook-ingestion
  name: Webhook Ingestion
  domain: ingestion
  description: Receives and processes webhooks from CLM providers
  
trigger:
  type: http
  method: POST
  path: /webhooks/{provider}
  
nodes:
  - id: validate_signature
    type: decision
    position: {x: 100, y: 50}
    spec:
      check: webhook_signature
      algorithm: hmac-sha256
      header: X-Webhook-Signature
      on_failure:
        status: 401
        body:
          error: "Signature validation failed"
    connections:
      valid: validate_payload
      invalid: reject_unauthorized

  - id: validate_payload
    type: input
    position: {x: 200, y: 50}
    spec:
      fields:
        contract_id:
          type: string
          required: true
          format: uuid
          error: "contract_id must be a valid UUID"
        document_url:
          type: string
          required: true
          format: url
          error: "document_url must be a valid URL"
        event_type:
          type: string
          enum: [created, updated, signed, executed]
          error: "Invalid event type"
    connections:
      valid: store_contract
      invalid: return_validation_error

  - id: store_contract
    type: data_store
    position: {x: 300, y: 50}
    spec:
      operation: upsert
      model: Contract
      data:
        id: "$.contract_id"
        document_url: "$.document_url"
        status: "$.event_type"
        provider: "$.path.provider"
    connections:
      success: publish_event
      failure: return_error

  - id: publish_event
    type: event
    position: {x: 400, y: 50}
    spec:
      event: contract.ingested
      payload:
        contract_id: "$.contract_id"
        provider: "$.path.provider"
    connections:
      done: return_success

  - id: return_success
    type: terminal
    position: {x: 500, y: 50}
    spec:
      status: 202
      body:
        message: "Webhook processed"
        contract_id: "$.contract_id"

metadata:
  created_by: murat
  created_at: 2025-02-04
  completeness: 100
```

## Schema Spec

```yaml
# specs/schemas/contract.yaml
schema:
  name: Contract
  version: 1.0.0
  description: Represents a contract document from CLM
  
  fields:
    id:
      type: uuid
      primary_key: true
    document_url:
      type: string
      format: url
      max_length: 2048
    status:
      type: string
      enum: [created, updated, signed, executed]
    provider:
      type: string
      enum: [ironclad, icertis, docusign]
    created_at:
      type: datetime
      default: now
    updated_at:
      type: datetime
      auto_update: true

  indexes:
    - fields: [provider, status]
    - fields: [created_at]

  used_by:
    - webhook-ingestion
    - extract-obligations
```

---

# Part 6: DDD CLI Tool

## Commands

```bash
# Initialize DDD in project
ddd init --language python --framework fastapi

# Check sync status
ddd status                    # All flows
ddd status -f webhook-ingestion  # Specific flow

# Validate spec-code sync
ddd validate                  # All flows
ddd validate -f webhook-ingestion --strict

# List pending implementations
ddd pending
ddd pending --since HEAD~5

# Generate code from specs
ddd generate webhook-ingestion
ddd generate webhook-ingestion -o schemas --dry-run

# Lint spec files
ddd lint
ddd lint --fix

# Show spec-code diff
ddd diff webhook-ingestion

# Manage mapping
ddd map show
ddd map update -f webhook-ingestion
ddd map verify

# Generate Claude Code instructions
ddd instructions                     # All pending
ddd instructions webhook-ingestion   # Specific flow
ddd instructions -o /tmp/impl.md     # Output to file
```

## Validation Checks

The validator checks:
1. **Existence:** Node in spec → code exists at mapped location
2. **Validations:** Spec validation rules → Pydantic/Zod validators match
3. **Error Messages:** Spec error messages → code error messages match exactly
4. **Connections:** Spec flow → code function call order matches
5. **Types:** Spec field types → code types match

---

# Part 7: DDD Tool (Desktop App)

## Tech Stack

- **Framework:** Tauri (Rust backend, React frontend)
- **Canvas:** tldraw SDK or React Flow
- **State:** Zustand
- **UI:** Tailwind + Radix
- **Git:** libgit2 (via git2 crate)

## Key Components

### Multi-Level Canvas (see Part 4: Multi-Level Canvas Architecture)
- **3-level hierarchy:** System Map → Domain Map → Flow Sheet
- **Breadcrumb navigation** at top of canvas
- **Double-click** to drill into domains, flows, sub-flows
- **Portal nodes** for cross-domain navigation
- Levels 1-2 are auto-derived from specs; Level 3 is the editing surface
- Zoom/pan navigation on all levels
- Undo/redo history
- Keyboard shortcuts (Backspace/Esc to navigate up)

### Visual Editor (Level 3 — Flow Sheet)
- Canvas with drag-drop nodes
- 11 node types (trigger, input, process, decision, service call, data store, event, terminal, loop, parallel, sub-flow)
- Connection drawing between nodes
- Sub-flow nodes are navigable links to other flow sheets

### System Map (Level 1 — Read-Only)
- Auto-generated from `system.yaml` + `domain.yaml` files
- Domain blocks with flow count badges
- Event arrows between domains (from `publishes_events`/`consumes_events`)
- Repositionable blocks (positions saved to `system-layout.yaml`)

### Domain Map (Level 2 — Read-Only)
- Auto-generated from `domain.yaml` + flow files
- Flow blocks within a domain
- Inter-flow event connections
- Portal nodes linking to other domains
- Repositionable blocks (positions saved to `domain.yaml` layout section)

### Spec Panel
- Right sidebar with type-specific fields
- Dropdown menus (not free text)
- Preset templates (username, email, password)
- Contextual tooltips
- Cmd+K command palette
- Context-aware: shows domain info on Level 2, flow spec on Level 3

### Git Panel
- Branch selector
- Staged/unstaged changes
- Commit box
- Pull/push buttons
- History view

### Expert Agents
- Database Designer
- Application Tester
- Security Expert
- DevOps Engineer
- Performance Engineer

### LLM Design Assistant

The DDD Tool embeds an LLM to help users design flows, fill specs, and get contextual suggestions — without leaving the editor. Two surfaces expose this capability: a **Chat Panel** and **Inline Assist** (context menu).

#### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ DDD TOOL                                                         │
│                                                                   │
│  ┌──────────────────────────────┐  ┌──────────────────────────┐ │
│  │ CANVAS / SPEC PANEL          │  │ CHAT PANEL (toggleable)  │ │
│  │                              │  │                          │ │
│  │  Right-click node            │  │  "Generate a user        │ │
│  │  ┌──────────────────────┐    │  │   registration flow"     │ │
│  │  │ ✨ Suggest spec      │    │  │                          │ │
│  │  │ ✨ Explain this node │    │  │  ┌──────────────────┐   │ │
│  │  │ ✨ Add error paths   │    │  │  │ Preview:          │   │ │
│  │  │ ✨ Generate tests    │    │  │  │ ⬡ → ▱ → ◇ → ⌗   │   │ │
│  │  └──────────────────────┘    │  │  │ → ⬭              │   │ │
│  │                              │  │  │                    │   │ │
│  │  Right-click canvas          │  │  │ [Apply] [Edit]    │   │ │
│  │  ┌──────────────────────┐    │  │  └──────────────────┘   │ │
│  │  │ ✨ Generate flow…    │    │  │                          │ │
│  │  │ ✨ Review this flow  │    │  │  Conversation history    │ │
│  │  │ ✨ Suggest wiring    │    │  │  with context awareness  │ │
│  │  └──────────────────────┘    │  │                          │ │
│  └──────────────────────────────┘  └──────────────────────────┘ │
│                        │                      │                   │
│                        ▼                      ▼                   │
│              ┌──────────────────────────────────────┐            │
│              │ LLM Context Builder                    │            │
│              │                                        │            │
│              │ Assembles context from:                │            │
│              │ • Current sheet level + location       │            │
│              │ • Selected node(s) + their specs       │            │
│              │ • Current flow YAML                    │            │
│              │ • Domain config + event wiring         │            │
│              │ • system.yaml + architecture.yaml      │            │
│              │ • Error codes from errors.yaml         │            │
│              │ • Schema definitions                   │            │
│              └──────────────┬───────────────────────┘            │
│                              │                                    │
│                              ▼                                    │
│              ┌──────────────────────────────────────┐            │
│              │ Tauri Backend (Rust)                    │            │
│              │ LLM API call (Anthropic / OpenAI /     │            │
│              │ local model via Ollama)                 │            │
│              └──────────────────────────────────────┘            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

#### Chat Panel

A toggleable right-side panel (alongside or replacing the Spec Panel) for open-ended conversations with the LLM about the current project.

**Capabilities:**

| Action | User says | LLM does |
|--------|-----------|----------|
| **Generate flow** | "Create a user registration flow with email verification" | Generates complete flow YAML, previews it, user clicks Apply to add to canvas |
| **Design agent** | "I need a support chatbot that searches KB and escalates" | Generates agent flow with tools, guardrails, memory, and human gate |
| **Design orchestration** | "Route tickets to billing or technical agents with a supervisor" | Generates orchestrator + smart router + agent flows |
| **Explain** | "What does this flow do?" | Reads current flow YAML and explains in plain language |
| **Review** | "Is this flow production-ready?" | Checks for missing error paths, unhandled edges, missing validations, security gaps |
| **Suggest improvements** | "How can I make this more robust?" | Suggests error handling, retry logic, rate limiting, caching |
| **Ask architecture** | "Should users and auth be separate domains?" | Advises based on domain boundaries, event patterns, team structure |
| **Debug** | "This flow fails when email is blank" | Traces through flow nodes, identifies missing validation |

**Context awareness:** The chat panel always knows:
- Which sheet level you're on (System / Domain / Flow)
- Which domain and flow are active
- The full spec of selected nodes
- The project's tech stack, error codes, schemas

**Output format:** When the LLM generates nodes or flows, it outputs DDD YAML that the tool can parse directly. Users see a preview on canvas (ghost nodes with dashed borders) and click **Apply** to commit or **Discard** to cancel.

**Conversation persistence:** Chat history is saved per-project in `.ddd/chat-history.json`. Context is scoped — starting a chat on a different flow begins a new thread (previous threads are accessible).

#### Inline Assist (Context Menu)

Right-click actions that provide targeted LLM help without opening the chat panel. Results appear as inline popovers or are applied directly.

**Node-level actions (right-click a node):**

| Action | What it does |
|--------|-------------|
| **✨ Suggest spec** | Auto-fills the node's spec based on its name and context. E.g., a node named `validate_input` in a registration flow → suggests email, password, name fields with standard validations and error messages |
| **✨ Complete spec** | Fills in empty/missing fields in an already-partially-filled spec |
| **✨ Explain node** | Shows a popover explaining what this node does based on its spec |
| **✨ Add error handling** | Adds failure connections with appropriate error codes from `errors.yaml` |
| **✨ Generate test cases** | Lists test scenarios for this node (happy path, edge cases, failures) |

**Connection-level actions (right-click a connection):**

| Action | What it does |
|--------|-------------|
| **✨ Add node between** | Suggests a node to insert on this connection (e.g., add guardrail before terminal) |
| **✨ Label connection** | Suggests a label based on source/target node semantics |

**Canvas-level actions (right-click empty canvas):**

| Action | What it does |
|--------|-------------|
| **✨ Generate flow…** | Opens a mini-prompt: "Describe what this flow should do" → generates nodes + connections |
| **✨ Review this flow** | Analyzes the current flow for completeness, missing paths, and best practices |
| **✨ Suggest wiring** | Looks at unconnected nodes and suggests how to wire them |
| **✨ Import from description** | Paste a feature description (Jira ticket, user story) → generates flow |

**Domain-level actions (right-click on Level 2):**

| Action | What it does |
|--------|-------------|
| **✨ Suggest flows** | Based on domain name and existing flows, suggests missing flows (e.g., "users" domain has register but no login, reset-password, delete-account) |
| **✨ Suggest events** | Based on domain flows, suggests events to publish/consume |
| **✨ Generate domain** | "Describe this domain" → generates domain.yaml with flows, events, wiring |

**System-level actions (right-click on Level 1):**

| Action | What it does |
|--------|-------------|
| **✨ Suggest domains** | Based on system description and existing domains, suggests missing domains |
| **✨ Review architecture** | Checks domain boundaries, event wiring completeness, circular dependencies |
| **✨ Generate from description** | "Describe your application" → generates system.yaml with domains |

#### Project Memory System

The LLM needs more than just what's on screen — it needs cumulative understanding of the entire project. The **Project Memory** system provides five layers of persistent, auto-maintained context that give the LLM full awareness of what's being built, why certain decisions were made, and how everything connects.

```
┌─────────────────────────────────────────────────────────────────┐
│ PROJECT MEMORY LAYERS                                            │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 1. PROJECT SUMMARY          (.ddd/memory/summary.md)      │  │
│  │    Auto-maintained plain-language description of the        │  │
│  │    entire project: what it does, all domains, key patterns  │  │
│  │    Regenerated when specs change.                           │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 2. SPEC INDEX               (.ddd/memory/spec-index.yaml) │  │
│  │    Condensed index of ALL flows, ALL domains, ALL schemas  │  │
│  │    — not full YAML, but enough for the LLM to understand   │  │
│  │    what exists and how things connect.                      │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 3. DECISION LOG             (.ddd/memory/decisions.md)     │  │
│  │    User-authored + LLM-captured design rationale.          │  │
│  │    "Why did we split auth from users?" stored here.        │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 4. CROSS-FLOW MAP           (.ddd/memory/flow-map.yaml)   │  │
│  │    Derived graph: which flows call which, event wiring,    │  │
│  │    shared schemas, agent delegations, handoffs.            │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ 5. IMPLEMENTATION STATUS    (.ddd/memory/status.yaml)      │  │
│  │    Which specs have code, which are pending, which changed │  │
│  │    since last code gen. From .ddd/mapping.yaml + git diff. │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

##### Layer 1: Project Summary

Auto-generated plain-language description of the project. Regenerated whenever domain configs or system.yaml change. The LLM can read this to instantly understand the project without parsing every YAML file.

```markdown
<!-- .ddd/memory/summary.md — auto-generated, do not edit manually -->

# Obligo — Cyber Liability Operating System

A SaaS platform that creates an "Operational Obligations Graph" from contracts.
Users upload contracts → system extracts obligations → tracks compliance → sends notifications.

## Tech Stack
Python 3.11 / FastAPI / SQLAlchemy / PostgreSQL / Redis / Celery

## Domains (4)

### ingestion (4 flows)
Handles contract document intake from multiple sources.
- **webhook-ingestion**: Receives documents via partner webhooks (POST /webhooks/{source})
- **scheduled-sync**: Polls external systems on cron schedule
- **manual-upload**: User uploads documents via UI (POST /api/documents/upload)
- **email-intake**: Processes documents from email attachments
- Publishes: contract.ingested, webhook.received, document.uploaded

### analysis (3 flows)
Extracts obligations and clauses from ingested contracts using LLM.
- **extract-obligations**: Agent flow — LLM reads contract, extracts structured obligations
- **classify-risk**: Scores obligations by risk level
- **generate-summary**: Creates human-readable contract summary
- Consumes: contract.ingested
- Publishes: analysis.completed, obligations.extracted

### api (5 flows)
REST API for frontend and integrations.
- **user-register**, **user-login**, **user-reset-password**, **get-contracts**, **get-obligations**
- Consumes: webhook.received

### notification (2 flows)
Sends alerts when obligations approach deadlines.
- **deadline-alert**: Agent flow — checks upcoming deadlines, drafts notifications
- **compliance-report**: Generates periodic compliance reports
- Consumes: analysis.completed, obligation.deadline_approaching

## Key Patterns
- Auth: JWT with refresh tokens, RBAC (admin, manager, viewer)
- Multi-tenancy: organization_id on all models
- Soft delete: all entities use deleted_at timestamp
- Event-driven: domains communicate via async events
```

**When it regenerates:** On project load, and whenever system.yaml, any domain.yaml, or architecture.yaml changes. The LLM itself generates this summary by reading all specs — so it understands the project in its own words.

##### Layer 2: Spec Index

A condensed index of every flow, schema, and domain — not the full YAML, but enough structure for the LLM to know what exists, what each flow does, and how they connect.

```yaml
# .ddd/memory/spec-index.yaml — auto-generated from specs/

domains:
  ingestion:
    description: "Contract document intake from multiple sources"
    flows:
      webhook-ingestion:
        type: traditional
        trigger: { type: http, method: POST, path: "/webhooks/{source}" }
        nodes: [validate_signature, rate_limit_check, validate_payload, normalize, store_document, publish_event]
        publishes: [contract.ingested]
        schemas_used: [Document, WebhookPayload]
        error_codes: [INVALID_SIGNATURE, RATE_LIMIT_EXCEEDED, VALIDATION_ERROR]

      scheduled-sync:
        type: traditional
        trigger: { type: cron, schedule: "*/15 * * * *" }
        nodes: [fetch_sources, diff_check, process_new, store_documents, publish_events]
        publishes: [contract.ingested]

      manual-upload:
        type: traditional
        trigger: { type: http, method: POST, path: "/api/documents/upload" }
        nodes: [auth_check, validate_file, extract_metadata, store_document, publish_event]
        publishes: [document.uploaded]
        schemas_used: [Document, UploadRequest]

      email-intake:
        type: traditional
        trigger: { type: event, event: email.received }
        nodes: [parse_email, extract_attachments, validate_documents, store_documents]
        publishes: [contract.ingested]

  analysis:
    description: "Extract obligations and clauses from contracts using LLM"
    flows:
      extract-obligations:
        type: agent
        trigger: { type: event, event: contract.ingested }
        agent_model: claude-sonnet-4-5-20250929
        tools: [read_document, extract_clause, classify_obligation, save_obligation]
        guardrails: [pii_filter, hallucination_check]
        publishes: [obligations.extracted]
        schemas_used: [Contract, Obligation, Clause]

      classify-risk:
        type: traditional
        trigger: { type: event, event: obligations.extracted }
        nodes: [load_obligations, score_risk, update_risk_levels, publish_results]
        publishes: [analysis.completed]

      generate-summary:
        type: traditional
        trigger: { type: event, event: obligations.extracted }
        nodes: [load_contract, llm_summarize, store_summary]
        schemas_used: [Contract, ContractSummary]

  # ... (api, notification domains similarly indexed)

schemas:
  Document: { fields: [id, organization_id, filename, content_type, size_bytes, uploaded_by, status, created_at] }
  Contract: { fields: [id, organization_id, document_id, title, parties, effective_date, expiry_date, status] }
  Obligation: { fields: [id, contract_id, clause_text, obligation_type, deadline, risk_score, status] }
  User: { fields: [id, organization_id, email, name, role, created_at] }
  # ...

events:
  - contract.ingested: { publisher: ingestion, consumers: [analysis] }
  - obligations.extracted: { publisher: analysis, consumers: [analysis, notification] }
  - analysis.completed: { publisher: analysis, consumers: [notification] }
  - webhook.received: { publisher: ingestion, consumers: [api] }
  - document.uploaded: { publisher: ingestion, consumers: [] }
  - obligation.deadline_approaching: { publisher: cron, consumers: [notification] }

shared_error_codes: [VALIDATION_ERROR, NOT_FOUND, UNAUTHORIZED, DUPLICATE_ENTRY, RATE_LIMIT_EXCEEDED, INTERNAL_ERROR]
```

**When it regenerates:** On project load, and whenever any flow YAML, domain.yaml, schema, or errors.yaml changes. Built by scanning all spec files and extracting the key fields.

##### Layer 3: Decision Log

User-authored and LLM-captured design rationale. When the user explains *why* they made a choice in the chat, the LLM asks if it should save it as a decision. Users can also add decisions manually.

```markdown
<!-- .ddd/memory/decisions.md -->

## Design Decisions

### 2025-01-15: Separate ingestion from analysis
**Decision:** Keep document ingestion and obligation extraction as separate domains.
**Rationale:** Ingestion is I/O-heavy (webhooks, file uploads, email parsing) while analysis is compute-heavy (LLM calls). Different scaling profiles. Also, ingestion should work even if analysis is down — we buffer via events.
**Affected:** ingestion domain, analysis domain, contract.ingested event

### 2025-01-16: Agent flow for obligation extraction
**Decision:** Use an agent flow (not traditional) for extract-obligations.
**Rationale:** Contract formats vary wildly — fixed logic can't handle all cases. The LLM needs to reason about document structure, iterate over clauses, and decide when it's found all obligations. Agent loop with tools is the right pattern.
**Affected:** analysis/flows/extract-obligations.yaml

### 2025-01-18: JWT over sessions
**Decision:** Use JWT with refresh tokens for authentication, not server-side sessions.
**Rationale:** Stateless — works with horizontal scaling without sticky sessions or shared session store. Refresh tokens handle long-lived access.
**Affected:** architecture.yaml auth section, api domain

### 2025-01-20: Soft delete everywhere
**Decision:** All entities use soft delete (deleted_at timestamp) instead of hard delete.
**Rationale:** Compliance requirement — need audit trail. Also allows undo.
**Affected:** architecture.yaml database section, all schemas
```

**How decisions are captured:**
1. **Manually:** User clicks "Add Decision" in the Memory panel or types `/decide` in chat
2. **LLM-prompted:** When the user explains reasoning in chat (e.g., "I'm splitting these because…"), the LLM detects rationale and asks: *"Should I save this as a design decision?"*
3. **Inline:** Right-click a domain/flow → "✨ Add design note" → enters decision with context pre-filled

##### Layer 4: Cross-Flow Map

A derived dependency graph showing how every flow connects to every other flow — via events, sub-flow calls, shared schemas, agent delegations, and handoffs.

```yaml
# .ddd/memory/flow-map.yaml — auto-derived from all flow specs

graph:
  # Event-based connections (async)
  events:
    - from: ingestion/webhook-ingestion
      to: analysis/extract-obligations
      via: contract.ingested
      type: event

    - from: ingestion/manual-upload
      to: analysis/extract-obligations
      via: contract.ingested
      type: event

    - from: analysis/extract-obligations
      to: analysis/classify-risk
      via: obligations.extracted
      type: event

    - from: analysis/extract-obligations
      to: analysis/generate-summary
      via: obligations.extracted
      type: event

    - from: analysis/classify-risk
      to: notification/deadline-alert
      via: analysis.completed
      type: event

  # Direct call connections (sync)
  sub_flows:
    - from: api/user-register
      to: notification/send-welcome-email
      type: sub_flow

  # Shared schema connections
  shared_schemas:
    - schema: Contract
      used_by: [ingestion/webhook-ingestion, analysis/extract-obligations, analysis/generate-summary]
    - schema: Obligation
      used_by: [analysis/extract-obligations, analysis/classify-risk, notification/deadline-alert]
    - schema: User
      used_by: [api/user-register, api/user-login, api/user-reset-password]

  # Agent delegations / handoffs
  agent_connections:
    - from: support/orchestrator
      to: support/billing-agent
      type: orchestrator_manages
    - from: support/billing-agent
      to: support/technical-agent
      type: handoff
      mode: consult

# Dependency summary per flow
flow_dependencies:
  ingestion/webhook-ingestion:
    depends_on: []
    depended_on_by: [analysis/extract-obligations]
    schemas: [Document, WebhookPayload]

  analysis/extract-obligations:
    depends_on: [ingestion/webhook-ingestion, ingestion/manual-upload, ingestion/email-intake]
    depended_on_by: [analysis/classify-risk, analysis/generate-summary]
    schemas: [Contract, Obligation, Clause]

  # ...
```

**When it regenerates:** On project load, and whenever any flow YAML or domain.yaml changes. Built by scanning all flows for event publications/consumptions, sub-flow references, schema `$ref`s, and orchestrator/handoff nodes.

##### Layer 5: Implementation Status

Tracks which specs have been implemented as code, which are pending, and which changed since last code generation.

```yaml
# .ddd/memory/status.yaml — derived from .ddd/mapping.yaml + git status

overview:
  total_flows: 14
  implemented: 9
  pending: 3
  stale: 2          # Spec changed since code was generated

flows:
  ingestion/webhook-ingestion:
    status: implemented
    code_files:
      - src/domains/ingestion/router.py
      - src/domains/ingestion/services/webhook.py
      - tests/unit/domains/ingestion/test_webhook.py
    last_generated: "2025-01-18T14:30:00Z"
    spec_changed_since: false

  ingestion/scheduled-sync:
    status: implemented
    code_files:
      - src/domains/ingestion/tasks/sync.py
    last_generated: "2025-01-18T15:00:00Z"
    spec_changed_since: true     # ← STALE: spec was modified after code gen
    changes_since:
      - "Added rate_limit_check node"
      - "Changed cron schedule from */30 to */15"

  analysis/extract-obligations:
    status: implemented
    code_files:
      - src/domains/analysis/agents/extract.py
      - src/domains/analysis/tools/document_tools.py
    last_generated: "2025-01-19T10:00:00Z"
    spec_changed_since: false

  notification/compliance-report:
    status: pending    # ← Not yet implemented
    code_files: []

  api/get-obligations:
    status: pending
    code_files: []

  # ...

schemas:
  Document:
    status: implemented
    migration: "migrations/001_create_documents.py"
    spec_changed_since: false
  Obligation:
    status: implemented
    migration: "migrations/003_create_obligations.py"
    spec_changed_since: true    # ← Schema changed, migration needed
```

**When it updates:** From `.ddd/mapping.yaml` (spec-to-code mapping file) combined with `git diff` to detect spec changes since last code generation timestamp.

##### How Memory Feeds Into LLM Context

The Project Memory layers are integrated into the LLM Context Builder. Not everything is sent with every request — the context is **budgeted** to stay within token limits while maximizing relevance.

```
┌─────────────────────────────────────────────────────────────────┐
│ LLM CONTEXT BUDGET                                                │
│                                                                   │
│ Priority 1 (always included):                         ~1,500 tok │
│   ├── Project Summary (summary.md)                               │
│   └── System config (name, tech stack, domains)                  │
│                                                                   │
│ Priority 2 (included on L2/L3):                       ~1,000 tok │
│   ├── Current domain (flows, events)                             │
│   └── Related domains (via Cross-Flow Map)                       │
│                                                                   │
│ Priority 3 (included on L3):                          ~2,000 tok │
│   ├── Current flow (full YAML)                                   │
│   ├── Connected flows (via Cross-Flow Map, condensed)            │
│   └── Selected node specs                                        │
│                                                                   │
│ Priority 4 (included when relevant):                    ~500 tok │
│   ├── Relevant decisions (from Decision Log)                     │
│   ├── Implementation status of current flow                      │
│   └── Error codes + schema definitions                           │
│                                                                   │
│ Priority 5 (included on demand):                        ~500 tok │
│   └── Spec Index (condensed, for cross-project questions)        │
│                                                                   │
│ TOTAL BUDGET:                                    ~4,000-5,500 tok │
│ (leaves room for user message + LLM response within context)     │
└─────────────────────────────────────────────────────────────────┘
```

**Key rules:**
- **Project Summary** always included — it's the LLM's "big picture" understanding
- **Cross-Flow Map** tells the LLM what connects to the current flow, so it can warn about breaking changes
- **Decision Log** filtered to decisions relevant to the current domain/flow
- **Implementation Status** tells the LLM whether this flow has code, whether the code is stale
- **Spec Index** only sent for system-level or cross-domain questions

##### Memory Panel UI

A dedicated panel (accessible via sidebar icon or `Cmd+M` / `Ctrl+M`) for viewing and managing Project Memory.

```
┌─────────────────────────────────────────────────────────────────┐
│ 🧠 PROJECT MEMORY                                      [Refresh] │
│                                                                   │
│ ┌─ Summary ────────────────────────────────────────────────────┐ │
│ │ Obligo — Cyber Liability Operating System                    │ │
│ │ 4 domains · 14 flows · 9 implemented · 2 stale              │ │
│ │ [View full summary]                                          │ │
│ └──────────────────────────────────────────────────────────────┘ │
│                                                                   │
│ ┌─ Implementation Status ──────────────────────────────────────┐ │
│ │ ● ingestion/webhook-ingestion          ✓ implemented         │ │
│ │ ● ingestion/scheduled-sync             ⚠ stale (2 changes)  │ │
│ │ ● analysis/extract-obligations         ✓ implemented         │ │
│ │ ○ notification/compliance-report       ◌ pending             │ │
│ │ ○ api/get-obligations                  ◌ pending             │ │
│ └──────────────────────────────────────────────────────────────┘ │
│                                                                   │
│ ┌─ Decisions (4) ─────────────────────────────────────[+ Add]──┐ │
│ │ • Separate ingestion from analysis       2025-01-15          │ │
│ │ • Agent flow for obligation extraction   2025-01-16          │ │
│ │ • JWT over sessions                      2025-01-18          │ │
│ │ • Soft delete everywhere                 2025-01-20          │ │
│ └──────────────────────────────────────────────────────────────┘ │
│                                                                   │
│ ┌─ Flow Map (current: webhook-ingestion) ──────────────────────┐ │
│ │ Depends on: (none)                                           │ │
│ │ Depended on by: analysis/extract-obligations                 │ │
│ │ Schemas: Document, WebhookPayload                            │ │
│ │ Events out: contract.ingested                                │ │
│ └──────────────────────────────────────────────────────────────┘ │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

##### Memory File Storage

All memory files live in `.ddd/memory/` within the project directory:

```
.ddd/
├── config.yaml              # DDD Tool settings (LLM provider, etc.)
├── mapping.yaml             # Spec-to-code mapping
├── chat-history.json        # Chat threads
└── memory/
    ├── summary.md           # Layer 1: Project summary (auto-generated)
    ├── spec-index.yaml      # Layer 2: Condensed spec index (auto-generated)
    ├── decisions.md         # Layer 3: Design decisions (user + LLM authored)
    ├── flow-map.yaml        # Layer 4: Cross-flow dependency graph (auto-generated)
    └── status.yaml          # Layer 5: Implementation status (auto-generated)
```

**Git behavior:** Memory files are auto-generated (except `decisions.md`). The `.gitignore` should include auto-generated memory files but NOT `decisions.md` — design decisions are valuable to the team and should be version-controlled.

```gitignore
# .gitignore
.ddd/memory/summary.md
.ddd/memory/spec-index.yaml
.ddd/memory/flow-map.yaml
.ddd/memory/status.yaml
.ddd/chat-history.json

# DO commit: .ddd/memory/decisions.md (design rationale is team knowledge)
# DO commit: .ddd/config.yaml (shared tool settings, no secrets)
# DO commit: .ddd/mapping.yaml (spec-to-code mapping)
```

#### LLM Context Builder

Every LLM request includes structured context assembled from the user's current location, selection, and **Project Memory layers**. The context is automatically budgeted to stay within token limits.

```yaml
# Context sent with every LLM request
context:
  # From Project Memory Layer 1 (always included)
  project_summary: |
    Obligo — Cyber Liability Operating System.
    SaaS for creating Operational Obligations Graph from contracts.
    4 domains: ingestion, analysis, api, notification.
    Python 3.11 / FastAPI / PostgreSQL / Redis.

  # From system.yaml (always included)
  system:
    name: obligo
    tech_stack: { language: python, framework: fastapi }
    domains: [ingestion, analysis, api, notification]

  # Included when on Level 2 or 3
  current_domain:
    name: ingestion
    flows: [webhook-ingestion, scheduled-sync, manual-upload]
    publishes_events: [contract.ingested, webhook.received]
    consumes_events: []

  # Included when on Level 3
  current_flow:
    id: webhook-ingestion
    type: traditional
    yaml: |
      # Full flow YAML here

  # From Project Memory Layer 4 — connected flows (condensed)
  connected_flows:
    downstream:
      - id: analysis/extract-obligations
        type: agent
        via_event: contract.ingested
        summary: "LLM agent that reads contracts and extracts structured obligations"
    upstream: []

  # Included when nodes are selected
  selected_nodes:
    - id: validate_input
      type: input
      spec: { ... }

  # From Project Memory Layer 3 — relevant decisions
  relevant_decisions:
    - "Separate ingestion from analysis: different scaling profiles, buffer via events"

  # From Project Memory Layer 5 — implementation status
  implementation_status:
    current_flow: implemented
    spec_changed_since_codegen: false

  # Included from project specs
  error_codes:
    - VALIDATION_ERROR
    - DUPLICATE_ENTRY
    - NOT_FOUND
    # ... from errors.yaml

  schemas:
    - User: { fields: [id, email, name, ...] }
    # ... from schemas/*.yaml
```

#### Multi-Model Architecture

DDD supports multiple LLM providers and models simultaneously. Different tasks route to different models — fast/cheap models for quick suggestions, powerful models for complex flow generation. Users can switch models on the fly from the UI.

##### Model Registry

All configured models live in a **model registry**. Each entry defines a provider, model ID, capabilities, and cost tier.

```yaml
# .ddd/config.yaml
llm:
  # Active model for chat (user can switch via UI)
  active_model: claude-sonnet

  # Model registry — all available models
  models:
    claude-sonnet:
      provider: anthropic
      model_id: claude-sonnet-4-5-20250929
      api_key_env: ANTHROPIC_API_KEY
      label: "Claude Sonnet"               # Display name in UI
      tier: standard                       # fast | standard | powerful
      max_tokens: 4096
      temperature: 0.3
      cost_per_1k_input: 0.003             # USD per 1K input tokens
      cost_per_1k_output: 0.015            # USD per 1K output tokens

    claude-opus:
      provider: anthropic
      model_id: claude-opus-4-6
      api_key_env: ANTHROPIC_API_KEY
      label: "Claude Opus"
      tier: powerful
      max_tokens: 4096
      temperature: 0.3
      cost_per_1k_input: 0.015
      cost_per_1k_output: 0.075

    claude-haiku:
      provider: anthropic
      model_id: claude-haiku-4-5-20251001
      api_key_env: ANTHROPIC_API_KEY
      label: "Claude Haiku"
      tier: fast
      max_tokens: 2048
      temperature: 0.2
      cost_per_1k_input: 0.0008
      cost_per_1k_output: 0.004

    gpt-4o:
      provider: openai
      model_id: gpt-4o
      api_key_env: OPENAI_API_KEY
      label: "GPT-4o"
      tier: standard
      max_tokens: 4096
      temperature: 0.3
      cost_per_1k_input: 0.0025
      cost_per_1k_output: 0.01

    gpt-4o-mini:
      provider: openai
      model_id: gpt-4o-mini
      api_key_env: OPENAI_API_KEY
      label: "GPT-4o Mini"
      tier: fast
      max_tokens: 2048
      temperature: 0.2
      cost_per_1k_input: 0.00015
      cost_per_1k_output: 0.0006

    llama-local:
      provider: ollama
      model_id: llama3.1
      base_url: http://localhost:11434
      label: "Llama 3.1 (Local)"
      tier: standard
      max_tokens: 4096
      temperature: 0.3
      cost_per_1k_input: 0                 # Free (local)
      cost_per_1k_output: 0

    deepseek-r1:
      provider: openai_compatible
      model_id: deepseek-reasoner
      base_url: https://api.deepseek.com/v1
      api_key_env: DEEPSEEK_API_KEY
      label: "DeepSeek R1"
      tier: powerful
      max_tokens: 8192
      temperature: 0.3
      cost_per_1k_input: 0.00055
      cost_per_1k_output: 0.0022

    # Any OpenAI-compatible endpoint works:
    # Azure OpenAI, Together AI, Groq, Fireworks, Mistral, etc.
    custom-endpoint:
      provider: openai_compatible
      model_id: your-model-id
      base_url: https://your-endpoint.com/v1
      api_key_env: CUSTOM_API_KEY
      label: "Custom Model"
      tier: standard
      max_tokens: 4096
      temperature: 0.3
      cost_per_1k_input: 0
      cost_per_1k_output: 0

  # Task-to-model routing — which model handles which task
  task_routing:
    # Inline assist (quick, frequent) → fast model
    suggest_spec: claude-haiku
    complete_spec: claude-haiku
    explain_node: claude-haiku
    label_connection: claude-haiku
    add_error_handling: claude-haiku

    # Flow generation (complex) → powerful model
    generate_flow: claude-sonnet
    generate_domain: claude-sonnet
    generate_from_description: claude-sonnet
    import_from_description: claude-sonnet

    # Review / architecture (needs deep reasoning) → powerful model
    review_flow: claude-sonnet
    review_architecture: claude-opus
    suggest_wiring: claude-sonnet
    suggest_flows: claude-sonnet
    suggest_domains: claude-sonnet

    # Project summary generation → powerful model
    generate_summary: claude-opus

    # Chat (uses active_model by default) → user's choice
    chat: _active                          # Special value: uses active_model

    # Test case generation → standard model
    generate_test_cases: claude-sonnet

  # Fallback chain — if primary model fails, try next
  fallback_chain:
    - claude-sonnet                        # Try first
    - gpt-4o                               # Then try
    - llama-local                          # Last resort (offline)

  # Cost tracking
  cost_tracking:
    enabled: true
    reset_period: monthly                  # daily | weekly | monthly
    budget_warning: 10.00                  # USD — show warning at this threshold
    budget_limit: 50.00                    # USD — block requests above this (0 = unlimited)
```

##### Provider Types

| Provider | `provider` value | Auth | Notes |
|----------|------------------|------|-------|
| **Anthropic** | `anthropic` | API key via env var | Claude models |
| **OpenAI** | `openai` | API key via env var | GPT models |
| **Ollama** | `ollama` | None (local) | Any model pulled locally, offline capable |
| **OpenAI-Compatible** | `openai_compatible` | API key via env var | Azure OpenAI, Together AI, Groq, Fireworks, DeepSeek, Mistral, or any OpenAI-compatible endpoint |

The `openai_compatible` provider works with any endpoint that follows the OpenAI `/v1/chat/completions` API format. This covers most hosted LLM services.

##### Task-to-Model Routing

Every LLM action in DDD has a **task type**. The task router looks up which model handles that task type from `task_routing` config. This lets users optimize for cost vs. quality per action.

```
┌─────────────────────────────────────────────────────────────────┐
│ TASK-TO-MODEL ROUTING                                            │
│                                                                   │
│  User Action            Task Type            Routed Model        │
│  ─────────────          ──────────           ──────────          │
│  Right-click →          suggest_spec    ──→  claude-haiku (fast) │
│  "Suggest spec"                                                   │
│                                                                   │
│  Right-click →          generate_flow   ──→  claude-sonnet       │
│  "Generate flow…"                            (standard)          │
│                                                                   │
│  Right-click →          review_          ──→  claude-opus         │
│  "Review architecture"  architecture         (powerful)          │
│                                                                   │
│  Chat panel →           chat            ──→  active_model        │
│  free-form message                           (user's choice)     │
│                                                                   │
│  Memory →               generate_       ──→  claude-opus         │
│  regenerate summary     summary              (powerful)          │
│                                                                   │
│         If model fails → try fallback_chain                      │
│         claude-sonnet → gpt-4o → llama-local                     │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

**Model tiers guide routing defaults:**

| Tier | Use for | Token budget | Examples |
|------|---------|-------------|----------|
| **fast** | Inline assists, explanations, labels, completions | ~500-1,000 | Haiku, GPT-4o Mini |
| **standard** | Flow generation, suggestions, chat, test cases | ~2,000-4,000 | Sonnet, GPT-4o, Llama 3.1 |
| **powerful** | Architecture review, project summary, complex reasoning | ~4,000-8,000 | Opus, DeepSeek R1 |

##### Model Picker UI

A dropdown in the Chat Panel header and a status bar indicator for quickly switching the active model.

```
┌─────────────────────────────────────────────────────────────────┐
│ ✨ Design Assistant          [Claude Sonnet ▾]          [✕]     │
│                               ┌──────────────────────────┐      │
│ Context: ingestion /          │ ● Claude Sonnet   $0.003 │      │
│ webhook-ingestion             │ ● Claude Opus     $0.015 │      │
│                               │ ● Claude Haiku    $0.001 │      │
│                               │ ─────────────────────── │      │
│                               │ ● GPT-4o          $0.003 │      │
│                               │ ● GPT-4o Mini     $0.000 │      │
│                               │ ─────────────────────── │      │
│                               │ ● Llama 3.1 (Local) Free│      │
│                               │ ● DeepSeek R1     $0.001 │      │
│                               │ ─────────────────────── │      │
│                               │ ⚙ Manage models…        │      │
│                               └──────────────────────────┘      │
│                                                                   │
│ ...chat messages...                                              │
│                                                                   │
│ ┌─────────────────────────────────────────────────────────────┐ │
│ │ This session: 12.4K tokens · $0.04    [Month: $3.82/$50]  │ │
│ └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**Model picker features:**
- Dropdown shows all registered models with cost-per-1K indicator
- Green dot = connected, red dot = unavailable
- "Manage models…" opens settings to add/remove/configure models
- Switching model applies immediately to the chat thread
- Inline assist uses the routed model (not the active model), but user can override per-action via right-click → submenu

**Status bar (bottom of app):**
```
┌──────────────────────────────────────────────────────────────────────┐
│ 🟢 Claude Sonnet │ Session: 12.4K tok · $0.04 │ Month: $3.82/$50.00 │
└──────────────────────────────────────────────────────────────────────┘
```

##### Cost Tracking

Every LLM request tracks input/output tokens and calculates cost based on the model's configured rates. Costs are accumulated per session and per billing period.

```yaml
# .ddd/memory/usage.yaml — auto-maintained
usage:
  current_period:
    start: "2025-01-01"
    end: "2025-01-31"
    total_cost: 3.82
    total_input_tokens: 245000
    total_output_tokens: 89000
    requests: 142
    by_model:
      claude-sonnet:
        requests: 45
        input_tokens: 120000
        output_tokens: 52000
        cost: 2.14
      claude-haiku:
        requests: 89
        input_tokens: 110000
        output_tokens: 32000
        cost: 0.22
      claude-opus:
        requests: 8
        input_tokens: 15000
        output_tokens: 5000
        cost: 1.46
    by_task:
      suggest_spec: { requests: 52, cost: 0.12 }
      generate_flow: { requests: 18, cost: 1.05 }
      chat: { requests: 32, cost: 1.44 }
      review_flow: { requests: 12, cost: 0.48 }
      generate_summary: { requests: 3, cost: 0.73 }

  history:
    - period: "2024-12-01/2024-12-31"
      total_cost: 5.21
      requests: 198
```

**Budget enforcement:**
- At `budget_warning` threshold → yellow banner: "You've used $10 of your $50 monthly budget"
- At `budget_limit` threshold → requests blocked with option to increase limit or switch to free local model
- Budget = 0 means unlimited

##### Inline Assist Model Override

When right-clicking for inline assist, the context menu shows which model will handle the action. Users can override via a submenu.

```
┌──────────────────────────────────────┐
│ ✨ Suggest spec        (Haiku)       │
│ ✨ Complete spec       (Haiku)       │
│ ✨ Explain this node   (Haiku)       │
│ ✨ Add error handling  (Haiku)       │
│ ✨ Generate test cases (Sonnet)      │
│ ──────────────────────────────────── │
│ ▶ Use different model…               │
│   ┌────────────────────────────┐     │
│   │ ● Claude Sonnet            │     │
│   │ ● Claude Opus              │     │
│   │ ● GPT-4o                   │     │
│   └────────────────────────────┘     │
└──────────────────────────────────────┘
```

#### Ghost Preview (Apply/Discard Pattern)

When the LLM generates nodes or flows, they appear as **ghost nodes** — visually distinct (dashed borders, reduced opacity) — so the user can review before committing.

```
┌─────────────────────────────────────────────────────────────┐
│ Canvas                                                       │
│                                                               │
│   ⬡ trigger    →    ▱ validate ─ ─ ─ ▱ validate_input ─ ┐  │
│   (existing)        (existing)        (ghost - dashed)    │  │
│                                                            │  │
│                                       ◇ check_duplicate ─ ┘  │
│                                       (ghost - dashed)        │
│                                            │                  │
│                                       ⌗ create_user          │
│                                       (ghost - dashed)        │
│                                            │                  │
│                                       ⬭ return_success       │
│                                       (ghost - dashed)        │
│                                                               │
│   ┌──────────────────────────────────────────┐               │
│   │ ✨ LLM generated 4 nodes                 │               │
│   │                                           │               │
│   │ [Apply to canvas]  [Edit in chat]  [✕]   │               │
│   └──────────────────────────────────────────┘               │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

Ghost nodes become real nodes on Apply. The user can also click **Edit in chat** to refine the generation before applying.

#### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Cmd+L` / `Ctrl+L` | Toggle Chat Panel |
| `Cmd+M` / `Ctrl+M` | Toggle Memory Panel |
| `Cmd+Shift+G` / `Ctrl+Shift+G` | Generate flow from description (canvas-level) |
| `Cmd+.` / `Ctrl+.` | Inline assist on selected node (suggest spec) |
| `Cmd+I` / `Ctrl+I` | Toggle Implementation Panel |
| `Escape` | Discard ghost preview |
| `Enter` (in ghost preview) | Apply ghost preview to canvas |

### Claude Code Integration

The DDD Tool integrates with Claude Code CLI to turn specs into running code without leaving the application. Five components work together: an **Implementation Panel** with interactive terminal, a **Prompt Builder** that constructs optimal prompts from specs, **Stale Detection** that tracks spec-vs-code drift, a **Test Runner** that shows results linked to flows, and **CLAUDE.md Auto-Generation** that keeps Claude Code instructions in sync with the project.

#### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ DDD TOOL                                                         │
│                                                                   │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌────────────────────────┐│
│  │ Canvas  │ │ Spec    │ │ Chat    │ │ Implementation Panel   ││
│  │         │ │ Panel   │ │ Panel   │ │                        ││
│  │         │ │         │ │         │ │ ┌────────────────────┐ ││
│  │         │ │         │ │         │ │ │ Prompt Preview     │ ││
│  │         │ │         │ │         │ │ │ "Implement user-   │ ││
│  │         │ │         │ │         │ │ │  register flow..." │ ││
│  │         │ │         │ │         │ │ │ [▶ Run] [Edit]     │ ││
│  │         │ │         │ │         │ │ └────────────────────┘ ││
│  │         │ │         │ │         │ │ ┌────────────────────┐ ││
│  │         │ │         │ │         │ │ │ Terminal (PTY)     │ ││
│  │         │ │         │ │         │ │ │                    │ ││
│  │         │ │         │ │         │ │ │ claude> Creating   │ ││
│  │         │ │         │ │         │ │ │ router.py...       │ ││
│  │         │ │         │ │         │ │ │ Allow? (y/n) █     │ ││
│  │         │ │         │ │         │ │ └────────────────────┘ ││
│  │         │ │         │ │         │ │ ┌────────────────────┐ ││
│  │         │ │         │ │         │ │ │ Test Results       │ ││
│  │         │ │         │ │         │ │ │ ✓ register_success │ ││
│  │         │ │         │ │         │ │ │ ✓ duplicate_email  │ ││
│  │         │ │         │ │         │ │ │ ✗ short_password   │ ││
│  │         │ │         │ │         │ │ │ 2/3 passing        │ ││
│  │         │ │         │ │         │ │ └────────────────────┘ ││
│  └─────────┘ └─────────┘ └─────────┘ └────────────────────────┘│
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

#### 1. Implementation Panel

A toggleable right-side panel (`Cmd+I` / `Ctrl+I`) with three sections: prompt preview, interactive terminal, and test results.

**Prompt Preview:** Before running Claude Code, the panel shows the auto-generated prompt so the user can review and edit it. The prompt is built by the Prompt Builder (see below).

**Interactive Terminal:** A full pseudo-terminal (PTY) embedded in the DDD Tool. Claude Code runs here interactively — the user can approve file operations, answer questions, and see progress in real-time. This is not a "fire and forget" — the user stays in control.

**Test Results:** After Claude Code finishes, the DDD Tool runs the project's test command and displays results linked to the implemented flow.

**Panel states:**

| State | What the panel shows |
|-------|---------------------|
| **Idle** | "Select a flow and click Implement" + implementation queue |
| **Prompt ready** | Prompt preview + [Run] button |
| **Running** | Terminal with live Claude Code output |
| **Done** | Summary (files created/modified) + test results |
| **Failed** | Error output + [Retry] [Edit prompt] buttons |

**Entry points — how to start implementation:**

| Action | Where | What happens |
|--------|-------|-------------|
| Click "▶ Implement" button | Flow Sheet (L3) toolbar | Opens Implementation Panel, builds prompt for current flow |
| Right-click flow block → "Implement" | Domain Map (L2) | Opens Implementation Panel for that flow |
| Click "Update code" on stale warning | Stale notification | Opens Implementation Panel with targeted update prompt |
| Click "▶ Implement selected" | Implementation Queue | Processes selected flows sequentially |

#### 2. Prompt Builder

The DDD Tool auto-constructs an optimal prompt for Claude Code based on the flow being implemented. The user never has to remember which files to reference or in what order.

**Prompt template:**

```markdown
# Auto-generated prompt for Claude Code
# Flow: {flow_id} | Domain: {domain_id}

Read these spec files in order:

1. specs/architecture.yaml — Project structure, conventions, dependencies
2. specs/shared/errors.yaml — Error codes and messages
{for each schema referenced by the flow}
3. specs/schemas/{schema}.yaml — Data model
{end for}
4. specs/domains/{domain}/flows/{flow}.yaml — The flow to implement

## Instructions

Implement the {flow_id} flow following architecture.yaml exactly:

- Create the endpoint matching the trigger spec (method: {method}, path: {path})
- Create request/response schemas matching the input node validations
- Implement each node as described in the spec
- Use EXACT error codes from errors.yaml (do not invent new ones)
- Use EXACT validation messages from the flow spec
- Create unit tests covering: happy path, each validation failure, each error path

## File locations

- Implementation: src/domains/{domain}/
- Tests: tests/unit/domains/{domain}/

## After implementation

Update .ddd/mapping.yaml with:
```yaml
flows:
  {flow_id}:
    spec: specs/domains/{domain}/flows/{flow}.yaml
    spec_hash: {sha256 of spec file}
    files: [list of files you created]
    implemented_at: {ISO timestamp}
```

{if flow type is agent}
## Agent-specific

This is an agent flow. Implement:
- Agent runner with the agent loop configuration
- Tool implementations for each tool defined in the spec
- Guardrail middleware for input/output filtering
- Memory management per the memory spec
- Use mocked LLM responses in tests
{end if}

{if this is an update to existing code}
## Update mode

This flow was previously implemented. The spec has changed:
{list of changes}

Update the existing code to match the new spec.
Do NOT rewrite files from scratch — modify the existing implementation.
Update affected tests.
{end if}
```

**What makes the prompt smart:**
- Automatically resolves which schemas the flow references (via `$ref` and data_store model fields)
- Includes only relevant error codes (not the full errors.yaml if the flow only uses 3 codes)
- Detects if this is a new implementation or an update
- For updates, includes a diff summary of what changed in the spec
- For agent flows, adds agent-specific instructions
- Includes architecture.yaml conventions so Claude Code knows naming patterns, folder structure, etc.

**The user can edit the prompt** before running. The prompt preview has an "Edit" button that opens it as editable text. Useful for adding specific instructions like "use bcrypt for password hashing" or "skip the email sending for now."

#### 3. Stale Detection

When a flow has existing code (recorded in `.ddd/mapping.yaml`) but the spec has changed since code generation, the flow is **stale**. DDD Tool detects this by comparing the spec file's SHA-256 hash against the `spec_hash` stored at implementation time.

**Stale detection triggers:**
- On project load (compare all hashes)
- On flow save (compare saved flow's hash)
- On git pull (any spec files changed)

**What the user sees:**

On Level 2 (Domain Map), stale flow blocks show a warning badge:
```
┌──────────────────┐
│ webhook-ingestion │
│ ⚠ spec changed    │
│   2 updates       │
└──────────────────┘
```

On Level 3 (Flow Sheet), a banner appears at the top:
```
┌──────────────────────────────────────────────────────────────┐
│ ⚠ This flow's spec changed since code was generated           │
│                                                                │
│ Changes:                                                      │
│ • Added send_verification_email node                          │
│ • Changed return message to "Check your email..."             │
│                                                                │
│ [▶ Update code]  [View diff]  [Dismiss]                      │
└──────────────────────────────────────────────────────────────┘
```

**Change detection:** The DDD Tool computes a human-readable diff between the old spec (at implementation time, cached in `.ddd/cache/`) and the current spec. This diff is included in the update prompt so Claude Code knows exactly what to change.

**Spec cache:** When a flow is implemented, the DDD Tool saves a copy of the spec at that point:
```
.ddd/
├── cache/
│   └── specs-at-implementation/
│       ├── api--user-register.yaml      # Spec as it was when code was generated
│       ├── ingestion--webhook-ingestion.yaml
│       └── ...
```

#### 4. Test Runner

After Claude Code finishes implementing a flow, the DDD Tool runs the project's test suite and displays results in the Implementation Panel.

**Configuration:**

```yaml
# .ddd/config.yaml
testing:
  # Test commands per language/framework (auto-detected from architecture.yaml)
  command: pytest                    # Or: jest, go test, cargo test
  args: ["--tb=short", "-q"]        # Additional arguments

  # Run only tests related to the implemented flow
  scoped: true                      # If true, runs only relevant tests
  scope_pattern: "tests/**/test_{flow_id}*"  # Glob for finding flow tests

  # Auto-run after implementation
  auto_run: true                    # Run tests automatically after Claude Code finishes
```

**Test result display:**

```
┌──────────────────────────────────────────────────────────────┐
│ TEST RESULTS — api/user-register                              │
│                                                                │
│ ✓ test_register_success                              0.12s   │
│ ✓ test_register_duplicate_email                      0.08s   │
│ ✓ test_register_invalid_email_format                 0.03s   │
│ ✗ test_register_short_password                       0.04s   │
│   AssertionError: Expected status 422, got 400               │
│   > assert response.status_code == 422                       │
│                                                                │
│ 3/4 passing · 1 failing · 0.27s                              │
│                                                                │
│ [Re-run tests]  [Fix failing test]                           │
└──────────────────────────────────────────────────────────────┘
```

**"Fix failing test" button:** Sends the failing test output back to Claude Code with a prompt: "This test is failing after implementing {flow_id}. Fix the implementation to match the spec." This creates a fix-and-retest loop without the user having to copy-paste error messages.

**Test results on canvas:** After tests run, flow blocks on Level 2 show a test badge:
```
┌──────────────────┐
│ user-register     │
│ ✓ 4/4 tests      │    ← green badge
└──────────────────┘

┌──────────────────┐
│ webhook-ingestion │
│ ✗ 3/4 tests      │    ← red badge
└──────────────────┘
```

#### 5. CLAUDE.md Auto-Generation

The DDD Tool generates and maintains a `CLAUDE.md` file in the project root. This file tells Claude Code how the project is structured and how to work with DDD specs. It's regenerated whenever specs change.

**Generated CLAUDE.md:**

```markdown
<!-- Auto-generated by DDD Tool. Manual edits below the CUSTOM section are preserved. -->

# Project: Obligo

## Spec-Driven Development

This project uses Diagram-Driven Development (DDD). All business logic
is specified in YAML files under `specs/`. Code MUST match specs exactly.

## Spec Files

- `specs/system.yaml` — Project identity, tech stack, 4 domains
- `specs/architecture.yaml` — Folder structure, conventions, dependencies
- `specs/config.yaml` — Environment variables schema
- `specs/shared/errors.yaml` — 12 error codes (VALIDATION_ERROR, NOT_FOUND, ...)
- `specs/schemas/*.yaml` — 6 data models (User, Document, Contract, ...)

## Domains

| Domain | Flows | Status |
|--------|-------|--------|
| ingestion | webhook-ingestion, scheduled-sync, manual-upload, email-intake | 3 implemented, 1 pending |
| analysis | extract-obligations (agent), classify-risk, generate-summary | 1 implemented, 2 pending |
| api | user-register, user-login, user-reset-password, get-contracts, get-obligations | 2 implemented, 3 pending |
| notification | deadline-alert (agent), compliance-report | 0 implemented, 2 pending |

## Implementation Rules

1. **Read architecture.yaml first** — it defines folder structure and conventions
2. **Follow the folder layout** — put files where architecture.yaml specifies
3. **Use EXACT error codes** — from specs/shared/errors.yaml, do not invent new ones
4. **Use EXACT validation messages** — from the flow spec, do not rephrase
5. **Match field types exactly** — spec field types map to language types
6. **Update .ddd/mapping.yaml** — after implementing a flow, record the mapping

## Tech Stack

- Language: Python 3.11
- Framework: FastAPI
- ORM: SQLAlchemy
- Database: PostgreSQL
- Cache: Redis
- Queue: Celery

## Commands

```bash
pytest                    # Run tests
mypy src/                 # Type check
ruff check .              # Lint
alembic upgrade head      # Run migrations
uvicorn src.main:app      # Start server
```

## Folder Structure

```
src/
├── main.py
├── config/
├── shared/
│   ├── database.py
│   ├── exceptions.py
│   └── middleware.py
├── models/
└── domains/
    └── {domain}/
        ├── router.py
        ├── schemas.py
        └── services.py
```

<!-- CUSTOM: Add your own instructions below this line. They won't be overwritten. -->

```

**What triggers regeneration:**
- New flow created or deleted
- Domain added or removed
- Architecture.yaml changed
- Implementation status changed (flow implemented or became stale)
- User clicks "Regenerate CLAUDE.md" in settings

**Custom section preserved:** Everything below `<!-- CUSTOM -->` is never overwritten. Users can add their own conventions, workarounds, or preferences there.

#### Implementation Queue

The Implementation Panel includes a queue view showing all flows and their status.

```
┌──────────────────────────────────────────────────────────────┐
│ IMPLEMENTATION QUEUE                                          │
│                                                                │
│ Pending (not yet implemented):                                │
│ ☐ api/user-reset-password                                    │
│ ☐ api/get-contracts                                          │
│ ☐ api/get-obligations                                        │
│ ☐ notification/deadline-alert (agent)                        │
│ ☐ notification/compliance-report                             │
│                                                                │
│ Stale (spec changed since code gen):                          │
│ ☐ ingestion/scheduled-sync (2 changes)                       │
│                                                                │
│ Implemented (up to date):                                     │
│ ✓ api/user-register              4/4 tests ✓                │
│ ✓ api/user-login                 3/3 tests ✓                │
│ ✓ ingestion/webhook-ingestion    5/5 tests ✓                │
│                                                                │
│ [Select all pending]  [▶ Implement selected]                 │
└──────────────────────────────────────────────────────────────┘
```

**Batch implementation:** User selects multiple flows, clicks "Implement selected." The tool processes them sequentially — builds prompt, runs Claude Code, runs tests, updates status, moves to next. The user can monitor progress and intervene at any step.

#### Configuration

```yaml
# .ddd/config.yaml — Claude Code integration
claude_code:
  enabled: true
  command: claude                  # CLI command (must be in PATH)

  post_implement:
    run_tests: true                # Auto-run tests after code gen
    run_lint: false                # Auto-run linter after code gen
    auto_commit: false             # Never auto-commit (user reviews first)
    regenerate_claude_md: true     # Update CLAUDE.md after implementation

  prompt:
    include_architecture: true     # Always include architecture.yaml
    include_errors: true           # Always include errors.yaml
    include_schemas: auto          # Include only referenced schemas

testing:
  command: pytest
  args: ["--tb=short", "-q"]
  scoped: true
  scope_pattern: "tests/**/test_{flow_id}*"
  auto_run: true

reconciliation:
  auto_run: true                 # Auto-reconcile after implementation
  auto_accept_matching: true     # Auto-accept items where code matches spec
  notify_on_drift: true          # Show notification when drift detected
```

#### 7. Reverse Drift Detection

Stale Detection (section 3 above) catches when **specs change but code hasn't been updated**. Reverse Drift Detection catches the opposite: when **code changes but specs haven't been updated**. Together they form a bidirectional sync loop.

**The problem:** Claude Code follows the spec, but it also makes practical decisions — adding error handling the spec didn't mention, splitting a node into multiple steps, using a slightly different response shape, or adding middleware. These deviations are invisible to the DDD Tool. Over time, the flow diagrams show the *original design* but not what the code *actually does*.

**Three layers work together to detect and resolve this:**

##### Layer 1: Implementation Report

The Prompt Builder (section 2 above) includes an instruction asking Claude Code to report what it actually did:

```markdown
## Implementation Report (required)

After implementing, output a section titled `## Implementation Notes` with:

1. **Deviations** — Anything you did differently from the spec (different field names,
   changed error codes, reordered steps, etc.)
2. **Additions** — Anything you added that the spec didn't mention (middleware,
   validation, caching, extra error handling, helper functions, etc.)
3. **Ambiguities resolved** — Anything the spec was unclear about and how you decided
4. **Schema changes** — Any new fields, changed types, or migration implications

If you followed the spec exactly with no changes, write: "No deviations."
```

The DDD Tool parses this structured output from the terminal after Claude Code finishes. If anything other than "No deviations" is found, it triggers reconciliation.

##### Layer 2: Code → Spec Reconciliation

After implementation, the DDD Tool's Design Assistant (the embedded LLM) reads the generated code files and compares them against the flow spec. It produces a **reconciliation report**:

```
┌──────────────────────────────────────────────────────────────┐
│ RECONCILIATION — api/user-register                            │
│                                                                │
│ Sync Score: 85%  (3 items need attention)                     │
│                                                                │
│ ✓ Matching (spec = code):                                     │
│   • POST /auth/register endpoint                             │
│   • validate_input → 3 field validations                     │
│   • check_duplicate → DUPLICATE_ENTRY error                  │
│   • create_user → User model insert                          │
│   • return_success → 201 response                            │
│                                                                │
│ ⚠ Code has but spec doesn't:                                  │
│   • rate_limit middleware (10 req/min per IP)                 │
│     [Accept into spec]  [Remove from code]  [Ignore]         │
│   • password_hash uses bcrypt (spec says "hash()")           │
│     [Accept into spec]  [Ignore]                             │
│   • response includes "token" field (spec only has user)     │
│     [Accept into spec]  [Remove from code]  [Ignore]         │
│                                                                │
│ ⚠ Spec has but code doesn't:                                  │
│   (none — all spec nodes are implemented)                     │
│                                                                │
│ [Accept all matching]  [Dismiss]                              │
└──────────────────────────────────────────────────────────────┘
```

**How reconciliation works:**

1. The DDD Tool reads all files listed in `.ddd/mapping.yaml` for the flow
2. It sends both the flow spec YAML and the implementation code to the Design Assistant LLM
3. The LLM compares them and outputs a structured JSON reconciliation report
4. The DDD Tool displays the report in the Implementation Panel

**Reconciliation actions per item:**

| Action | What happens |
|--------|-------------|
| **Accept into spec** | DDD Tool updates the flow spec YAML to include the addition (adds a node, updates a field, etc.). The canvas reflects the change immediately. |
| **Remove from code** | Builds a targeted prompt for Claude Code: "Remove {item} from the implementation — it's not in the spec." |
| **Ignore** | Marks this item as a known deviation. Stored in `.ddd/mapping.yaml` under `accepted_deviations`. Won't trigger drift warnings again. |

**Accept into spec** is the most powerful action — it reverse-engineers code changes back into the flow diagram. For example, if Claude Code added a `rate_limit` middleware, accepting it adds a process node to the flow with the rate limiting spec filled in. The LLM generates the node spec based on what the code actually does.

##### Layer 3: Sync Score

Each implemented flow gets a **sync score** indicating how closely the code matches the spec:

| Score | Meaning | Badge |
|-------|---------|-------|
| **100%** | Code matches spec exactly | ✓ (green) |
| **80-99%** | Minor additions (extra validation, logging) | ~ (yellow) |
| **50-79%** | Significant divergence (extra endpoints, changed schemas) | ⚠ (amber) |
| **< 50%** | Code barely resembles spec | ✗ (red) |

**Where sync scores appear:**

On Level 2 (Domain Map), flow blocks show the sync badge alongside test badges:
```
┌──────────────────┐
│ user-register     │
│ ✓ 4/4 tests      │    ← test badge
│ ~ 85% synced     │    ← sync badge
└──────────────────┘
```

On Level 3 (Flow Sheet), a sync indicator appears in the toolbar:
```
[Flow: user-register]  [✓ Tests: 4/4]  [~ Sync: 85%]  [▶ Implement]
```

Clicking the sync badge opens the reconciliation report.

##### The Full Bidirectional Sync Cycle

```
┌──────────────┐   Stale Detection    ┌───────────────┐
│  SPEC        │ ────────────────────▶ │  CODE         │
│  (YAML)      │   "Spec changed,     │  (src/)       │
│              │    code is stale"     │               │
│              │                       │               │
│              │   Reverse Drift       │               │
│              │ ◀──────────────────── │               │
│              │   "Code diverged,     │               │
│              │    spec is outdated"  │               │
└──────┬───────┘                       └───────┬───────┘
       │                                       │
       │         ┌──────────────┐              │
       └────────▶│ RECONCILE    │◀─────────────┘
                 │              │
                 │ • Compare    │
                 │ • Score      │
                 │ • Accept /   │
                 │   Remove /   │
                 │   Ignore     │
                 └──────────────┘
```

**When reconciliation runs:**
- Automatically after every Claude Code implementation (if `reconciliation.auto_run: true`)
- Manually via "Reconcile" button in Implementation Panel
- On project load, for any flow that was implemented outside the DDD Tool (e.g., manually edited code)
- After git pull, if code files in mapping changed but spec didn't

**Reconciliation data storage:**

```yaml
# .ddd/mapping.yaml — extended with reconciliation data
flows:
  api/user-register:
    spec: specs/domains/api/flows/user-register.yaml
    spec_hash: abc123...
    files: [src/domains/api/router.py, src/domains/api/schemas.py, src/domains/api/services.py]
    implemented_at: "2025-01-15T10:30:00Z"
    sync_score: 85
    last_reconciled_at: "2025-01-15T10:32:00Z"
    accepted_deviations:
      - description: "password_hash uses bcrypt"
        code_location: "src/domains/api/services.py:23"
        accepted_at: "2025-01-15T10:33:00Z"
```

---

# Part 8: Workflows

## Complete Workflow: Idea to Deployment

### Phase 1: Ideation (Claude Chat)
```
Input: Natural language app idea
Process: Claude asks clarifying questions, outputs DDD specs
Output: Folder structure with YAML specs (completeness: partial)
```

### Phase 2: Visual Refinement (DDD Tool)
```
Input: Claude's specs
Process: Import, visualize, fill missing details via UI, run expert agents
Output: Refined specs with 100% completeness
```

### Phase 3: Implementation (Claude Code)
```
Input: Refined specs
Process: Read specs, implement flows, validate, run tests
Output: Working code matching specs exactly
```

### Phase 4: Validation & Deploy
```
Input: Implemented code
Process: Spec-code sync check, tests, security scan, deploy
Output: Deployed application
```

## Git-Based Sync Workflow

```
1. YOU EDIT SPEC IN DDD TOOL
   └─► DDD Tool writes to specs/*.yaml

2. COMMIT SPEC CHANGES
   └─► git add specs/
   └─► git commit -m "Design: Add rate limiting"

3. CLAUDE CODE IMPLEMENTS
   └─► claude "Implement pending spec changes"
   └─► Claude reads specs/, generates code in src/

4. VALIDATE
   └─► ddd validate
   └─► pytest

5. COMMIT CODE CHANGES
   └─► git add src/ tests/
   └─► git commit -m "Implement: Add rate limiting"

6. PUSH
   └─► git push
   └─► CI validates spec-code sync
```

---

# Part 9: Reusability System

## Levels of Reuse

### Level 1: Sub-Flows (Within Project)
Reuse entire flows as callable units:
- `send-email` called from multiple flows
- Interface defines inputs/outputs

### Level 2: Shared Components (Within Project)
- **Node Templates:** `validate_webhook_signature`, `retry_with_backoff`
- **Validation Rules:** Standard email, password, phone
- **Schemas:** `PaginatedResponse<T>`, `ApiError`, `AuditLog`

### Level 3: Project Templates (Within Project)
Parameterized flow generators:
- `CRUD_API<Entity>` → 5 flows: list, get, create, update, delete
- `WEBHOOK_RECEIVER<Provider>` → Signature validation, parsing, dedup
- `EVENT_WORKER<Event>` → Subscription, retry, DLQ, idempotency

### Level 4: Personal Library (Across Projects)
Save patterns for reuse:
- `my-jwt-auth`, `my-stripe-checkout`, `my-s3-upload`
- Parameterization for customization
- Linked (get updates) or Copied (full control)

### Level 5: Community Library (Global)
Public marketplace:
- Pre-built integrations (Stripe, Auth0, SendGrid)
- Starter templates (SaaS, E-Commerce, API-First)

---

# Part 10: Technical Decisions

## Decision: Git over MCP for Sync

**Chosen:** Git-based sync
**Rejected:** MCP-based real-time sync

**Reasoning:**
- MCP would duplicate state that Git already tracks
- Git provides versioning, branching, merging for free
- No extra infrastructure to maintain
- Claude Code already understands Git
- Simpler architecture

## Decision: YAML for Specs

**Chosen:** YAML
**Alternatives:** JSON, custom DSL

**Reasoning:**
- Human-readable and editable
- Git-friendly (meaningful diffs)
- Supports comments
- Well-supported in all languages
- $ref resolution straightforward

## Decision: Tauri for Desktop App

**Chosen:** Tauri
**Alternative:** Electron

**Reasoning:**
- Smaller bundle size
- Better performance
- Rust backend for file system access
- Native Git integration via libgit2

## Decision: Spec-to-Code Mapping

**Chosen:** Explicit mapping file (.ddd/mapping.yaml)
**Alternative:** Convention-based (infer from names)

**Reasoning:**
- Allows restructuring code without breaking sync
- Explicit > implicit for validation
- Supports incremental adoption

---

# Part 11: Open Questions / Future Work

## MVP Scope (v0.1)
**Included:**
- Multi-level canvas (System Map → Domain Map → Flow Sheet)
- Breadcrumb navigation between levels
- Canvas + 5 basic node types for traditional flows (Level 3)
- Agent flow support: Agent Loop, Tool, LLM Call, Memory, Guardrail, Human Gate, Router nodes
- Orchestration support: Orchestrator, Smart Router, Handoff, Agent Group nodes
- Agent-centric canvas layout for agent flows
- Orchestration visualization on Level 2 (supervisor arrows, handoff arrows, group boundaries)
- Auto-generated System Map and Domain Map (Levels 1-2)
- Portal nodes for cross-domain navigation
- Spec panel (basic fields + agent-specific + orchestration panels)
- YAML export (traditional, agent, and orchestration flow formats)
- **LLM Design Assistant:** Chat Panel + Inline Assist (context menu) for flow generation, spec auto-fill, review, and suggestions
- **Ghost Preview:** LLM-generated nodes appear as ghost nodes (dashed borders) with Apply/Discard before committing
- **Multi-Model Architecture:** Model registry with multiple providers (Anthropic, OpenAI, Ollama, any OpenAI-compatible), task-to-model routing, model picker UI, cost tracking
- **LLM Context Builder:** Automatically assembles project context (system, domain, flow, schemas, error codes) for every LLM request
- **Project Memory:** 5-layer persistent memory (project summary, spec index, decision log, cross-flow map, implementation status) with context budgeting
- **Memory Panel:** UI for viewing project summary, implementation status, design decisions, and flow dependencies
- **Claude Code Integration:** Implementation Panel with embedded PTY terminal for interactive Claude Code sessions
- **Prompt Builder:** Auto-constructs optimal prompts from specs (schema resolution, agent detection, update mode)
- **Stale Detection:** SHA-256 hash comparison to detect spec-vs-code drift with human-readable change summaries
- **Test Runner:** Auto-runs tests after implementation, displays results linked to flows, fix-and-retest loop
- **CLAUDE.md Auto-Generation:** Maintains CLAUDE.md with project structure, spec files, domains, rules (preserves custom section)
- **Implementation Queue:** Batch processing of pending/stale flows with sequential Claude Code invocation
- **Reverse Drift Detection:** Implementation Report parsing, LLM-powered code→spec reconciliation, sync scores, accept/remove/ignore actions
- Mermaid preview
- LLM prompt generation
- Single user, local storage

**Excluded (v0.2+):**
- Real-time collaboration
- Reverse engineering
- Community library
- Live agent testing/debugging (run agent from within DDD Tool)

## Potential Extensions

1. **VS Code Extension:** Spec editing without full app
2. **GitHub Action:** Spec-code sync validation in CI
3. **MCP Server:** For Claude Code convenience tools (not sync)
4. **Web Version:** For quick viewing/editing without desktop app
5. **API:** For programmatic spec manipulation

## Open Technical Questions

1. How to handle partial implementations?
2. Schema migration generation from spec changes?
3. Multi-language support (Python + TypeScript in same project)?
4. Spec inheritance for similar flows?

---

# Part 12: Key Files Reference

## Project Structure (Complete)

```
obligo/
├── .git/
├── .ddd/
│   ├── config.yaml           # DDD tool config
│   ├── mapping.yaml          # Spec-to-code mapping
│   └── templates/            # Project-specific templates
│
├── specs/                    # ═══ ALL SPECS LIVE HERE ═══
│   │
│   ├── system.yaml           # Project identity, tech stack, domains
│   ├── system-layout.yaml    # System Map (L1) positions (managed by DDD Tool)
│   ├── architecture.yaml     # Structure, infrastructure, cross-cutting
│   ├── config.yaml           # Environment variables schema
│   │
│   ├── schemas/
│   │   ├── _base.yaml        # Base model (inherited by all)
│   │   ├── contract.yaml
│   │   ├── obligation.yaml
│   │   ├── user.yaml
│   │   ├── tenant.yaml
│   │   └── events/
│   │       ├── contract-ingested.yaml
│   │       └── obligation-extracted.yaml
│   │
│   ├── shared/
│   │   ├── auth.yaml         # Auth flow specification
│   │   ├── errors.yaml       # Error codes and formats
│   │   ├── middleware.yaml   # Middleware stack
│   │   └── api.yaml          # API conventions
│   │
│   └── domains/
│       ├── ingestion/
│       │   ├── domain.yaml      # Flows, events, L2 layout
│       │   └── flows/
│       │       ├── webhook-ingestion.yaml
│       │       └── scheduled-sync.yaml
│       ├── analysis/
│       │   ├── domain.yaml      # Flows, events, L2 layout
│       │   └── flows/
│       │       ├── extract-obligations.yaml
│       │       └── classify-risk.yaml
│       ├── api/
│       │   ├── domain.yaml      # Flows, events, L2 layout
│       │   └── flows/
│       │       ├── list-contracts.yaml
│       │       ├── get-contract.yaml
│       │       └── update-obligation.yaml
│       ├── notification/
│       │   ├── domain.yaml      # Flows, events, L2 layout
│       │   └── flows/
│       │       ├── send-email.yaml
│       │       └── send-slack.yaml
│       └── support/
│           ├── domain.yaml      # Flows, events, L2 layout
│           └── flows/
│               ├── customer-support-agent.yaml    # type: agent
│               ├── billing-agent.yaml             # type: agent
│               └── technical-agent.yaml           # type: agent
│
├── src/                      # ═══ GENERATED CODE ═══
│   ├── main.py               # FastAPI app entry point
│   ├── config/
│   │   ├── __init__.py
│   │   ├── settings.py       # Pydantic settings
│   │   └── logging.py        # Logging configuration
│   │
│   ├── shared/
│   │   ├── __init__.py
│   │   ├── auth/
│   │   │   ├── __init__.py
│   │   │   ├── jwt.py
│   │   │   ├── dependencies.py
│   │   │   └── permissions.py
│   │   ├── database/
│   │   │   ├── __init__.py
│   │   │   ├── connection.py
│   │   │   ├── base_model.py
│   │   │   └── mixins.py
│   │   ├── middleware/
│   │   │   ├── __init__.py
│   │   │   ├── request_id.py
│   │   │   ├── logging.py
│   │   │   ├── error_handler.py
│   │   │   ├── rate_limiter.py
│   │   │   └── tenant_context.py
│   │   ├── exceptions/
│   │   │   ├── __init__.py
│   │   │   ├── base.py
│   │   │   └── handlers.py
│   │   ├── events/
│   │   │   ├── __init__.py
│   │   │   ├── bus.py
│   │   │   └── handlers.py
│   │   └── utils/
│   │       ├── __init__.py
│   │       ├── pagination.py
│   │       └── filtering.py
│   │
│   └── domains/
│       ├── ingestion/
│       │   ├── __init__.py
│       │   ├── router.py
│       │   ├── schemas.py
│       │   ├── services.py
│       │   ├── models.py
│       │   ├── events.py
│       │   └── exceptions.py
│       ├── analysis/
│       │   └── ...
│       ├── api/
│       │   └── ...
│       └── notification/
│           └── ...
│
├── tests/
│   ├── conftest.py           # Shared fixtures
│   ├── factories/
│   │   ├── __init__.py
│   │   ├── user.py
│   │   ├── contract.py
│   │   └── obligation.py
│   ├── unit/
│   │   └── domains/
│   │       ├── ingestion/
│   │       └── ...
│   ├── integration/
│   │   └── ...
│   └── e2e/
│       └── ...
│
├── migrations/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
│       └── ...
│
├── scripts/
│   ├── seed.py
│   └── migrate.py
│
├── docker/
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   └── docker-compose.yml
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
│
├── pyproject.toml            # Dependencies and tool config
├── CLAUDE.md                 # Instructions for Claude Code
└── README.md
```

## CLAUDE.md Template

```markdown
# Spec-Driven Development

This project uses Diagram-Driven Development. All specifications live in `specs/`.

## Spec Files (Read These First)

1. **specs/system.yaml** - Project identity, tech stack, domains
2. **specs/architecture.yaml** - Project structure, infrastructure, cross-cutting concerns
3. **specs/config.yaml** - Environment variables schema
4. **specs/schemas/_base.yaml** - Base model inherited by all entities
5. **specs/shared/errors.yaml** - Error codes and response formats

## When implementing a new flow:

1. Read the flow spec:
   ```bash
   cat specs/domains/{domain}/flows/{flow}.yaml
   ```

2. Read related schemas:
   ```bash
   cat specs/schemas/{entity}.yaml
   ```

3. Check architecture for conventions:
   ```bash
   # Check where files should go
   cat specs/architecture.yaml | grep -A 50 "structure:"
   
   # Check middleware order
   cat specs/architecture.yaml | grep -A 20 "middleware_order:"
   
   # Check error handling
   cat specs/architecture.yaml | grep -A 30 "error_handling:"
   ```

4. Implement to match spec exactly:
   - Validation rules from spec → Pydantic validators
   - Error messages from spec → Use exact text
   - Error codes from errors.yaml → Use defined codes
   - API conventions from architecture → Follow pagination, filtering, response format

5. Validate implementation:
   ```bash
   ddd validate
   ```

6. Commit changes:
   ```bash
   git add src/ tests/
   git commit -m "Implement: {flow description}"
   ```

## Key Architecture Decisions

- **Multi-tenancy**: Row-level, tenant_id on all queries
- **Soft delete**: Use deleted_at, never DELETE
- **Auth**: JWT with RS256, access + refresh tokens
- **Errors**: Always use codes from specs/shared/errors.yaml
- **Logging**: Structured JSON, include request_id and tenant_id
- **Pagination**: Cursor-based by default
- **Testing**: Transaction rollback strategy

## File Locations

| Spec | Code Location |
|------|---------------|
| specs/domains/{domain}/flows/*.yaml | src/domains/{domain}/router.py |
| specs/schemas/{entity}.yaml | src/domains/{domain}/models.py + schemas.py |
| specs/shared/errors.yaml | src/shared/exceptions/*.py |
| specs/architecture.yaml → middleware | src/shared/middleware/*.py |
| specs/architecture.yaml → auth | src/shared/auth/*.py |

## Validation Commands

```bash
# Check spec-code sync
ddd validate

# Check specific flow
ddd validate -f webhook-ingestion

# See what needs implementation
ddd pending

# Run tests
pytest

# Type check
mypy src/
```
```

---

# Part 13: Glossary

| Term | Definition |
|------|------------|
| **Flow** | A complete process from trigger to terminal (e.g., webhook-ingestion) |
| **Node** | A single step in a flow (trigger, input, process, decision, etc.) |
| **Spec** | YAML file defining a flow, schema, or system configuration |
| **Mapping** | Connection between spec elements and code locations |
| **Validation** | Checking that code matches spec exactly |
| **Sync** | State where code and specs match |
| **Changeset** | A set of spec changes to be implemented |
| **Domain** | Bounded context grouping related flows |
| **Schema** | Data model definition (reusable across flows) |
| **Event** | Async message published by one flow, consumed by others |
| **Sub-flow** | Flow called synchronously from another flow |

---

# Part 14: Context for OBLIGO

The DDD tool is being designed with OBLIGO as the primary use case:

**OBLIGO:** A "Cyber Liability Operating System" SaaS platform that creates an "Operational Obligations Graph" from contracts by integrating with CLM, CRM, and threat intelligence systems.

**Key Domains:**
- **Ingestion:** Webhooks from CLM providers (Ironclad, Icertis, DocuSign)
- **Analysis:** Extract obligations from contracts using LLM
- **API:** REST API for frontend and integrations
- **Notification:** Email/Slack alerts for obligations

This context helps inform design decisions around:
- Webhook handling patterns
- LLM integration nodes
- Multi-tenant architecture
- Security requirements

---

# Part 15: Continuation Prompt

To continue this conversation in another Claude session, start with:

```
I'm building DDD (Diagram-Driven Development), a tool for bidirectional 
conversion between visual flow diagrams and code. I have a detailed 
specification document I'd like to share. The key points are:

1. Specs (YAML files) are the single source of truth, stored in Git
2. DDD Tool is a desktop app (Tauri) for visual editing of specs
3. Claude Code reads specs and implements code
4. Git handles all sync (no custom protocol)
5. DDD CLI provides validation, generation, and instructions

I'd like to continue discussing [specific topic].
```

Then share the relevant section(s) of this document.

---

# Document Version

**Version:** 1.0.0
**Created:** 2025-02-04
**Author:** Murat (with Claude)
**Context:** Full specification developed over extended conversation

---

# End of Document
