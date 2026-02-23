# awesome-claude-code Submission Answers

Submission URL: https://github.com/hesreallyhim/awesome-claude-code/issues/new?template=resource_submission.yml

Repo to submit: https://github.com/cybersoloss/claude-commands

**Submit one at a time.** Wait for bot validation to pass before submitting the next repo.
Resubmit eligibility: **Feb 28, 2026** (all three cybersoloss repos will have 7+ days of public history).

---

## Field Answers

### Validate Claims

```
Commands are plain .md prompt files — no code execution, no binaries, no scripts. Copy one to ~/.claude/commands/: cp ddd-create.md ~/.claude/commands/. Then in any project directory, run /ddd-create A simple todo app with users and tasks in Claude Code. It will generate a complete specs/ directory with YAML flow files, schemas, UI pages, and infrastructure config.
```

### Specific Task(s)

```
Give Claude Code a one-line product description and have it generate a full spec structure using /ddd-create.
```

### Specific Prompt(s)

```
/ddd-create A simple todo app with users, projects, and tasks. Node.js, PostgreSQL.
```

### Additional Comments

```
DDD is a four-phase methodology (Create → Design → Build → Reflect). The commands cover the full lifecycle — each is a standalone .md file. At runtime, commands fetch the DDD Usage Guide (github.com/cybersoloss/DDD) from GitHub via gh CLI — this is the source of truth for all spec formats, node types, and validation rules, so no local copy is needed. Generated specs can then be reviewed and edited visually on a canvas in the DDD Tool (github.com/cybersoloss/ddd-tool).
```

### Recommendation Checklist

All 5 boxes checked:
- [x] This resource hasn't already been submitted
- [x] It has been over one week since the first public commit
- [x] All provided links are working and publicly accessible
- [x] I do NOT have any other open issues in this repository
- [x] I am primarily composed of human-y stuff and not electrical circuits

**"Create more" checkbox: leave unchecked** — submit one at a time.

---

## Notes

- The security concern framing in Validate Claims ("no code execution, no binaries, no scripts") is intentional — addresses the maintainer's stated concern upfront before he raises it.
- Previous submission (Issue #823) failed with `enforce-cooldown` because force-push on push-all was resetting cybersoloss repo history. Fixed Feb 23 — push-all now uses commit-tree to preserve history.
- Once claude-commands passes, submit DDD guide next, then ddd-tool last.
