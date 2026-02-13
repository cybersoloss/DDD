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

## Node Types

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
| Sub-flow | ▢ | Call another flow |

## Connection Types Between Flows

1. **Event-Based (Async):** Publisher/subscriber via event bus
2. **Direct Call (Sync):** Sub-flow node with input/output mapping
3. **Shared Data Models:** `$ref:/schemas/SchemaName` references
4. **API Contracts:** Internal APIs connecting domains

---

# Part 5: DDD Format Specification v1.0

## Spec Hierarchy (Complete)

```
specs/
├── system.yaml              # Project identity, tech stack, domains
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
│   ├── errors.yaml          # Error codes and response format
│   └── api.yaml             # API conventions (pagination, filtering)
└── domains/
    └── {domain}/
        ├── domain.yaml
        └── flows/
            └── {flow}.yaml
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

## Architecture Config (NEW - Critical for Code Generation)

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

### Visual Editor
- Canvas with drag-drop nodes
- Zoom/pan navigation
- Undo/redo history
- Keyboard shortcuts

### Spec Panel
- Right sidebar with type-specific fields
- Dropdown menus (not free text)
- Preset templates (username, email, password)
- Contextual tooltips
- Cmd+K command palette

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
- Canvas + 5 basic node types
- Spec panel (basic fields)
- YAML export
- Mermaid preview
- LLM prompt generation
- Single user, local storage

**Excluded (v0.2+):**
- Real-time collaboration
- Direct code generation
- Test generation
- Git integration in UI
- Reverse engineering
- Community library

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
│       │   ├── domain.yaml
│       │   └── flows/
│       │       ├── webhook-ingestion.yaml
│       │       └── scheduled-sync.yaml
│       ├── analysis/
│       │   ├── domain.yaml
│       │   └── flows/
│       │       ├── extract-obligations.yaml
│       │       └── classify-risk.yaml
│       ├── api/
│       │   ├── domain.yaml
│       │   └── flows/
│       │       ├── list-contracts.yaml
│       │       ├── get-contract.yaml
│       │       └── update-obligation.yaml
│       └── notification/
│           ├── domain.yaml
│           └── flows/
│               ├── send-email.yaml
│               └── send-slack.yaml
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
