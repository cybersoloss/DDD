# DDD End-to-End Test Infrastructure

## The Problem

DDD evolves constantly. Node types get added. Trigger patterns change. UI component types grow. Validation rules tighten. Schema features expand. The ddd-tool codebase refactors functions, renames parameters, moves files.

A static test written today is stale in a month.

## The Solution: Generated Tests

Instead of maintaining test artifacts by hand, we maintain a **generator prompt** that reads the current state of all three DDD repos and produces fresh test artifacts every time it runs.

```
┌────────────────────────────────────────────────────────────────────────┐
│                        generate-tests.md                               │
│                     (the only permanent file)                          │
└──────────────────────────────┬─────────────────────────────────────────┘
                               │
                     Claude Code session
                   reads current state of:
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        DDD repo         ddd-tool repo    ddd-commands repo
     (Usage Guide,      (validators,      (slash command
      templates,         parsers,          prompts)
      conventions)       types, stores)
              │                │                │
              └────────────────┼────────────────┘
                               │
                         Generates ▼
                               │
         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼
  product-features.md   auto-test-prompt.md   auto-test src/
  (exercises EVERY       (references ACTUAL    (6 TypeScript files
   current DDD feature)   code paths today)    with current types)
```

### What gets generated

| File | Purpose | Why it must be fresh |
|------|---------|---------------------|
| `product-features.md` | Product description that exercises every DDD feature | Feature catalog changes (new node types, new triggers, etc.) |
| `ddd-tool-auto-test-prompt.md` | Prompt to implement auto-test runner in ddd-tool | Code paths change (functions move, signatures change, files rename) |
| `ddd-tool/src/auto-test/*.ts` (6 files) | The actual auto-test runner code | Type definitions evolve, validator API changes, new validation rules |

### What stays permanent

Only two files in this directory are permanent:

| File | Purpose |
|------|---------|
| `README.md` | This file — explains the infrastructure |
| `generate-tests.md` | The generator prompt — the single source of truth |

Everything else is output. It can be deleted and regenerated.

## How to Run

### Full regeneration (recommended monthly, or after any DDD framework change)

```bash
# In a Claude Code session with access to all three repos:
cd ~/dev/DDD

# Give Claude the generator prompt
cat tests/e2e/generate-tests.md
# (or just tell Claude: "Run the e2e test generator from tests/e2e/generate-tests.md")
```

Claude will:
1. Read the current DDD Usage Guide → extract the complete feature catalog
2. Read the current ddd-tool codebase → find validators, parsers, types with current signatures
3. Read the current ddd-commands → understand /ddd-create input format
4. Generate `product-features.md` exercising every feature found
5. Generate `ddd-tool-auto-test-prompt.md` referencing actual code paths
6. Optionally: generate the 6 auto-test TypeScript files directly in ddd-tool

### Running the generated tests

After generation, the test cycle is:

```bash
# Step 1: Generate a DDD project from the product features
cd /tmp && mkdir nexus-test && cd nexus-test
# In a fresh Claude Code session:
/ddd-create --from ~/dev/DDD/tests/e2e/product-features.md

# Step 2: Run the auto-test against the generated project
cd ~/dev/ddd-tool
npm run auto-test -- /tmp/nexus-test

# Step 3: Read the reports
cat /tmp/nexus-test/.ddd/reports/tool-compatibility-report.yaml
cat /tmp/nexus-test/.ddd/reports/spec-quality-report.yaml
```

### When to regenerate

- **After adding a node type** — the product features need a flow using it
- **After adding a trigger type** — needs a flow with that trigger
- **After adding a UI component** — needs a page section using it
- **After refactoring ddd-tool** — code paths in the auto-test prompt go stale
- **After changing validation rules** — coverage checker catalogs need updating
- **Monthly** — as a hygiene practice, even if you think nothing changed

### Quick check without full regeneration

If you just want to verify the current test artifacts are still valid:

```bash
# Check if product-features.md still references all current node types
cd ~/dev/DDD
# Compare node types in Usage Guide vs product-features.md
grep -c "node type" tests/e2e/product-features.md
```

But if in doubt, just regenerate. It takes 5 minutes.

## Architecture Decisions

### Why a prompt, not a script?

A script would need to parse YAML templates, understand DDD semantics, and generate natural-language product descriptions. That's an LLM task, not a scripting task. The prompt IS the test generator.

### Why one product concept?

The generator creates a single rich product description rather than isolated test cases because:
1. `/ddd-create` is designed to process natural product descriptions, not test matrices
2. A coherent product ensures event wiring, cross-domain flows, and schema relationships make sense together
3. Real bugs happen at integration boundaries — isolated feature tests miss them

### Why three repos as input?

The test artifacts must stay in sync across:
- **DDD repo** — defines what features exist (the "what")
- **ddd-tool repo** — defines how features are validated/rendered (the "how")
- **ddd-commands repo** — defines how specs are generated from descriptions (the "interface")

A change in any one can break the test. The generator reads all three.

### Why not hardcode feature catalogs?

The generator prompt does NOT contain lists like "28 node types: trigger, input, process...". Instead, it tells Claude to **read the current Usage Guide and extract whatever is there**. If next month has 32 node types, the generator finds all 32 without any prompt changes.

## File Structure

```
tests/e2e/
  README.md                      ← This file (permanent)
  generate-tests.md              ← Generator prompt (permanent)
  product-features.md            ← Generated output (regenerate monthly)
  ddd-tool-auto-test-prompt.md   ← Generated output (regenerate monthly)
```
