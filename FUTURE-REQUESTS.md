# DDD Future Requests

A lightweight backlog of known gaps, friction points, and improvement ideas that are not yet implemented. Feed high-signal items into `/ddd-evolve` when patterns emerge across multiple projects.

---

## change-history.yaml — Merge Conflict Resolution Tooling

**Context:** `change-history.yaml` is a tracked repo file and replicates on every push/pull. When two branches both append entries, git may produce a merge conflict inside the YAML.

**Current behavior (manual):**
1. Keep all entries from both sides of the conflict
2. Re-sequence `id` fields (`chg-0001`, `chg-0002`, ...) to eliminate duplicates
3. Sort by `timestamp` for readability (optional)

**Requested improvement:** Automate this with either:
- A `/ddd-merge-history` command that resolves conflicts in `change-history.yaml` automatically
- A git merge driver (`.gitattributes` + custom script) that handles the YAML append-only merge without conflict markers

**Priority:** Low — the manual step is straightforward, but becomes friction on active multi-developer teams.

---
