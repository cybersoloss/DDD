# DDD Quick Start Guide

## Files in This Package

| File | Purpose |
|------|---------|
| `ddd-specification-complete.md` | Full spec explaining DDD concepts, properties, workflows |
| `ddd-implementation-guide.md` | Instructions for building the DDD Tool itself |
| `architecture-template.yaml` | **Reusable** - Copy to any project for full code generation |
| `config-template.yaml` | **Reusable** - Environment variables schema template |
| `errors-template.yaml` | **Reusable** - Standardized error codes template |

---

## Two Different Use Cases

### Use Case A: Build the DDD Tool Itself

**Goal:** Create the desktop app that edits flow diagrams

**Files to use:**
- `ddd-specification-complete.md`
- `ddd-implementation-guide.md`

**Prompt for Claude Code:**
```
Read ddd-specification-complete.md and ddd-implementation-guide.md.
Build the DDD Tool following the implementation guide.
Start with Phase 2: Project Setup.
```

---

### Use Case B: Generate a Complete Application

**Goal:** Use DDD specs to generate a full codebase

**Files to use:**
- `architecture-template.yaml` → Copy to `specs/architecture.yaml`
- `config-template.yaml` → Copy to `specs/config.yaml`
- `errors-template.yaml` → Copy to `specs/shared/errors.yaml`
- Your flow specs in `specs/domains/*/flows/*.yaml`

**Setup Steps:**

```bash
# 1. Create project and specs folder
mkdir my-project && cd my-project
mkdir -p specs/domains specs/schemas specs/shared

# 2. Copy templates
cp ~/Downloads/architecture-template.yaml specs/architecture.yaml
cp ~/Downloads/config-template.yaml specs/config.yaml
cp ~/Downloads/errors-template.yaml specs/shared/errors.yaml

# 3. Create system.yaml
cat > specs/system.yaml << 'EOF'
system:
  name: my-project
  version: 1.0.0
  description: My awesome project
  
  tech_stack:
    language: python
    language_version: "3.11"
    framework: fastapi
    orm: sqlalchemy
    database: postgresql
    cache: redis
    
  domains:
    - name: users
      description: User management
    - name: billing
      description: Subscription and payments
EOF

# 4. Customize architecture.yaml
# Edit specs/architecture.yaml:
#   - Update {placeholders} with your values
#   - Remove sections you don't need
#   - Adjust settings for your requirements

# 5. Create your first flow
mkdir -p specs/domains/users/flows
cat > specs/domains/users/flows/register.yaml << 'EOF'
flow:
  id: user-register
  name: User Registration
  domain: users
  description: Register a new user account
  
trigger:
  type: http
  method: POST
  path: /auth/register
  
nodes:
  - id: validate_input
    type: input
    spec:
      fields:
        email:
          type: string
          required: true
          format: email
          error: "Please enter a valid email address"
        password:
          type: string
          required: true
          min_length: 8
          error: "Password must be at least 8 characters"
        name:
          type: string
          required: true
          min_length: 2
          max_length: 100
          error: "Name must be between 2 and 100 characters"
    connections:
      valid: check_duplicate
      invalid: return_validation_error

  - id: check_duplicate
    type: decision
    spec:
      check: user_exists_by_email
      on_true:
        error_code: DUPLICATE_ENTRY
        error: "A user with this email already exists"
    connections:
      false: create_user
      true: return_duplicate_error

  - id: create_user
    type: data_store
    spec:
      operation: create
      model: User
      data:
        email: "$.email"
        password_hash: "hash($.password)"
        name: "$.name"
    connections:
      success: return_success
      failure: return_error

  - id: return_success
    type: terminal
    spec:
      status: 201
      body:
        message: "User registered successfully"
        user:
          id: "$.user.id"
          email: "$.user.email"
          name: "$.user.name"
EOF

# 6. Create CLAUDE.md instructions
cat > CLAUDE.md << 'EOF'
# Project Instructions

This project uses Diagram-Driven Development (DDD).

## Spec Files

- `specs/system.yaml` - Project identity and tech stack
- `specs/architecture.yaml` - Project structure, infrastructure, cross-cutting
- `specs/config.yaml` - Environment variables schema
- `specs/shared/errors.yaml` - Error codes
- `specs/domains/*/flows/*.yaml` - Flow specifications

## Implementation Rules

1. **Read architecture.yaml first** - Understand project structure, conventions
2. **Follow the folder layout** - Put files where architecture.yaml says
3. **Use exact error codes** - From specs/shared/errors.yaml
4. **Match validation rules** - Implement exactly as spec defines
5. **Use error messages from spec** - Don't invent new messages

## Commands

```bash
# After implementing
pytest                    # Run tests
mypy src/                 # Type check
ruff check .              # Lint
```
EOF
```

**Prompt for Claude Code:**
```
I have a DDD project with specs in the specs/ folder.

Read these files in order:
1. specs/system.yaml - Project overview
2. specs/architecture.yaml - How to structure code
3. specs/config.yaml - Environment variables
4. specs/shared/errors.yaml - Error codes
5. specs/domains/users/flows/register.yaml - First flow to implement

Generate the complete project following architecture.yaml exactly:
1. Create folder structure per architecture.yaml
2. Create shared infrastructure (database, auth, middleware, exceptions)
3. Implement the user-register flow
4. Create tests using the testing strategy in architecture.yaml

Start with the project scaffold, then implement shared infrastructure,
then the flow.
```

---

## Spec Hierarchy

```
specs/
├── system.yaml              # WHO: Project name, tech stack, domains
├── architecture.yaml        # HOW: Structure, infrastructure, conventions
├── config.yaml              # WHAT: Environment variables needed
│
├── schemas/                 # DATA MODELS
│   ├── _base.yaml           # Base model (id, timestamps)
│   ├── user.yaml
│   └── ...
│
├── shared/                  # CROSS-CUTTING SPECS
│   ├── errors.yaml          # Error codes
│   ├── auth.yaml            # Auth flow details (optional)
│   └── events.yaml          # Event definitions (optional)
│
└── domains/                 # BUSINESS LOGIC
    └── {domain}/
        ├── domain.yaml      # Domain description
        └── flows/
            └── {flow}.yaml  # Individual flows
```

---

## What Each Spec Controls

### system.yaml
- Project name and version
- Language and framework choice
- List of domains

### architecture.yaml
- Folder structure (where every file goes)
- Package versions (exact dependencies)
- Database conventions (naming, timestamps, soft delete)
- Cache configuration
- Queue setup
- Auth strategy (JWT, sessions)
- Authorization model (RBAC roles)
- Multi-tenancy settings
- Logging format and fields
- Error handling format
- Rate limiting rules
- Middleware order
- API conventions (pagination, filtering, sorting)
- Testing strategy
- Deployment configuration

### config.yaml
- Required environment variables
- Optional variables with defaults
- Validation rules for production

### errors.yaml
- Error code definitions
- HTTP status mappings
- Message templates
- Response format

### Flow specs
- API endpoints
- Request/response schemas
- Validation rules
- Business logic steps
- Error handling per step

---

## Claude Code Generation Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  1. CLAUDE READS SPECS                                          │
│     ├── system.yaml        → Knows tech stack                  │
│     ├── architecture.yaml  → Knows how to structure code       │
│     ├── config.yaml        → Knows what env vars needed        │
│     └── errors.yaml        → Knows error format                │
│                                                                 │
│  2. CLAUDE GENERATES SCAFFOLD                                   │
│     ├── pyproject.toml     → From dependencies section         │
│     ├── src/config/        → From config.yaml                  │
│     ├── src/shared/        → From cross_cutting section        │
│     └── src/domains/       → One per domain in system.yaml     │
│                                                                 │
│  3. CLAUDE READS FLOW SPECS                                     │
│     └── specs/domains/users/flows/register.yaml                │
│                                                                 │
│  4. CLAUDE IMPLEMENTS FLOW                                      │
│     ├── src/domains/users/router.py                            │
│     ├── src/domains/users/schemas.py                           │
│     ├── src/domains/users/services.py                          │
│     └── tests/unit/domains/users/test_register.py              │
│                                                                 │
│  5. RESULT: Complete, consistent, production-ready code        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Minimal Viable Spec Set

For a simple project, you need at minimum:

```
specs/
├── system.yaml          # Required
├── architecture.yaml    # Required (can use template defaults)
└── domains/
    └── {domain}/
        └── flows/
            └── {flow}.yaml   # At least one flow
```

The templates provide sensible defaults. Just customize what you need.

---

## Tips for Claude Code

1. **Always read architecture.yaml first** - It defines everything
2. **Generate scaffold before flows** - Infrastructure must exist
3. **One domain at a time** - Don't generate everything at once
4. **Validate as you go** - Run tests after each flow
5. **Use exact error codes** - From errors.yaml, no inventing

---

## Common Customizations

### Change language to TypeScript
In `architecture.yaml`:
```yaml
dependencies:
  node: "20"
  production:
    "@nestjs/core": "^10.0.0"
    # ... TypeScript packages
```

### Disable multi-tenancy
In `architecture.yaml`:
```yaml
cross_cutting:
  multi_tenancy:
    enabled: false
```

### Use offset pagination instead of cursor
In `architecture.yaml`:
```yaml
api:
  pagination:
    style: offset
```

### Add custom error codes
In `specs/shared/errors.yaml`:
```yaml
business:
  CONTRACT_EXPIRED:
    http_status: 422
    message: "Contract has expired"
```
