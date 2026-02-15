# Design Driven Development (DDD)

**Design software visually. Let AI implement it.**

DDD is a methodology and desktop tool for building software through visual flow diagrams that output YAML specs. The specs are the single source of truth — they live in Git alongside code, and LLMs (like Claude Code) implement directly from them.

## Repository Structure

```
DDD/
├── ddd-specification-complete.md    # Full spec: concepts, features, data model
├── ddd-implementation-guide.md      # Build instructions: types, stores, components
├── ddd-quickstart.md                # Quick start guide for using DDD
└── templates/
    ├── architecture-template.yaml   # Reusable: project structure & conventions
    ├── config-template.yaml         # Reusable: environment variables schema
    └── errors-template.yaml         # Reusable: standardized error codes
```

## What's Here vs What's Not

| This repo | The [ddd-tool](https://github.com/mhcandan/ddd-tool) repo |
|-----------|------------|
| What DDD is | The actual desktop app |
| How to build the tool | TypeScript, Rust, React code |
| Reusable templates | Build system, dependencies |
| Design decisions | CI/CD, releases |

## Documents

### Specification (~280 KB)
The complete spec covering: multi-level canvas, 20+ node types, agent & orchestration flows, LLM design assistant, Claude Code integration, bidirectional drift detection, diagram-derived test generation, design validation, production infrastructure generation, and more.

### Implementation Guide (~420 KB)
Step-by-step build instructions with near-complete code for every component: types, Zustand stores, React components, Tauri commands, utilities. Organized into phases and sessions.

### Templates
Copy these into any project's `specs/` folder to use DDD for code generation:

```bash
mkdir -p specs/shared
cp templates/architecture-template.yaml specs/architecture.yaml
cp templates/config-template.yaml specs/config.yaml
cp templates/errors-template.yaml specs/shared/errors.yaml
```

Then customize for your tech stack and let Claude Code generate from specs.

## The DDD Tool

The desktop app (Tauri + React) is built in a separate repo: **[mhcandan/ddd-tool](https://github.com/mhcandan/ddd-tool)**
