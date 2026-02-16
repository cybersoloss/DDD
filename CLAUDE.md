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

**New Mac setup:**
```bash
# 1. Install push-all script
mkdir -p ~/bin
cat > ~/bin/git-push-all << 'SCRIPT'
#!/bin/bash
set -e
CURRENT_BRANCH=$(git branch --show-current)
echo "Pushing to origin..."
git push origin "$CURRENT_BRANCH"
if ! git remote get-url public &>/dev/null; then
  echo "No 'public' remote — skipping public push"
  exit 0
fi
TMPBRANCH="_public_mirror_$$"
git checkout -b "$TMPBRANCH" "$CURRENT_BRANCH" --quiet
if git ls-files --error-unmatch CLAUDE.md &>/dev/null 2>&1; then
  git rm --cached CLAUDE.md --quiet
  git commit --amend --no-edit --allow-empty --quiet
  echo "Pushing to public (CLAUDE.md excluded)..."
else
  echo "Pushing to public..."
fi
git push public "$TMPBRANCH:$CURRENT_BRANCH" --force
git checkout "$CURRENT_BRANCH" --force --quiet
git branch -D "$TMPBRANCH" --quiet
echo "Done — pushed to origin and public"
SCRIPT
chmod +x ~/bin/git-push-all

# 2. Git alias + auth
git config --global alias.push-all '!~/bin/git-push-all'
gh auth setup-git

# 3. Add public remote to each repo clone
cd ~/dev/DDD && git remote add public https://github.com/cybersoloss/DDD.git
cd ~/dev/ddd-tool && git remote add public https://github.com/cybersoloss/ddd-tool.git
```

## Key Files
| File | Purpose |
|------|---------|
| `DDD-USAGE-GUIDE.md` | Primary reference — all YAML formats, node types, spec fields, conventions |
| `ddd-specification-complete.md` | Full specification — architecture, lifecycle, command specs |
| `ddd-implementation-guide.md` | Implementation patterns and code generation rules |
| `ddd-quickstart.md` | Four-step quickstart for new users |
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
| **claude-commands** | Slash commands for Claude Code | mhcandan/claude-commands | — |

## Conventions
- `DDD-USAGE-GUIDE.md` is the single source of truth for YAML formats — other docs reference it, not duplicate it
- Node type proposals require: description, spec fields, sourceHandle values, and at least two real-world use cases
- Spec format changes require discussion first (open an issue before PR)
- 28 node types: trigger, input, process, decision, terminal, data_store, service_call, ipc_call, event, loop, parallel, sub_flow, llm_call, delay, cache, transform, collection, parse, crypto, batch, transaction, agent_loop, guardrail, human_gate, orchestrator, smart_router, handoff, agent_group
