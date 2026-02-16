# DDD — Development Context

## What This Repo Is
The methodology, specification, and reference documentation for **Design Driven Development** — design software visually as flow graphs, let AI implement from YAML specs.

## Git Remotes (Dual Remote Setup)
- `origin` → github.com/mhcandan/DDD (private, master repo)
- `public` → github.com/cybersoloss/DDD (public mirror)

**Push to both:** `git push-all` — pushes to origin normally, then pushes to public with CLAUDE.md excluded (force-push). CLAUDE.md stays on mhcandan only, never on cybersoloss.

**GitHub accounts:** `mhcandan` (primary dev), `cybersoloss` (public). mhcandan is collaborator on cybersoloss repos — no account switching needed.

**Handle community PRs:**
```bash
gh pr checkout <PR#> --repo cybersoloss/DDD
# review and test locally
git push-all
```

**New Mac setup:** See full instructions in `~/.claude/CLAUDE.md` → "New Mac Setup (complete)". Quick version for this repo only:
```bash
git clone https://github.com/mhcandan/DDD.git
cd DDD && git remote add public https://github.com/cybersoloss/DDD.git
```

## Key Files
| File | Purpose |
|------|---------|
| `DDD-USAGE-GUIDE.md` | Primary reference — all YAML formats, node types, spec fields, conventions |
| `ddd-specification-complete.md` | Full specification — architecture, lifecycle, command specs |
| `ddd-implementation-guide.md` | Implementation patterns and code generation rules |
| `ddd-quickstart.md` | Four-step quickstart for new users |
| [claude-commands](https://github.com/mhcandan/claude-commands) | All DDD slash commands (install via claude-commands repo) |
| `templates/` | Architecture, config, and errors YAML templates |

## Four-Phase Lifecycle
```
Phase 1: CREATE        Phase 2: DESIGN        Phase 3: BUILD          Phase 4: REFLECT
Human intent → Specs   Human reviews in Tool   Specs → Code            Code wisdom → Specs

/ddd-create            (DDD Tool)             /ddd-scaffold            /ddd-reverse
/ddd-update (any)                             /ddd-implement           /ddd-sync
                                              /ddd-test                /ddd-reflect
                                                                       /ddd-promote

Cross-cutting: /ddd-status    Meta: /ddd-evolve
```

## Related Repos
| Repo | What | Private | Public |
|------|------|---------|--------|
| **DDD** (this repo) | Methodology, specs, templates | mhcandan/DDD | cybersoloss/DDD |
| **ddd-tool** | Desktop app (Tauri 2.0 + React) | mhcandan/ddd-tool | cybersoloss/ddd-tool |
| **claude-commands** | All slash commands — DDD + general dev (code-review, security-scan, etc.) | mhcandan/claude-commands | — |

## Conventions
- `DDD-USAGE-GUIDE.md` is the single source of truth for YAML formats — other docs reference it, not duplicate it
- Node type proposals require: description, spec fields, sourceHandle values, and at least two real-world use cases
- Spec format changes require discussion first (open an issue before PR)
- 28 node types: trigger, input, process, decision, terminal, data_store, service_call, ipc_call, event, loop, parallel, sub_flow, llm_call, delay, cache, transform, collection, parse, crypto, batch, transaction, agent_loop, guardrail, human_gate, orchestrator, smart_router, handoff, agent_group
