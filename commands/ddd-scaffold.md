# DDD Scaffold

Set up the project skeleton and shared infrastructure from DDD specs before implementing flows. This is the first step of Phase 3 (Build) — it creates the project foundation that `/ddd-implement` builds on.

## Instructions

1. **Find the DDD project**: Look for `ddd-project.json` in the current directory or parent directories.

2. **Read the specs**:
   - `ddd-project.json` — project config, domain list
   - `specs/system.yaml` — project identity, tech stack, environments, integrations
   - `specs/architecture.yaml` — project structure, naming conventions, dependencies, infrastructure, API design, testing, deployment
   - `specs/config.yaml` — required and optional environment variables
   - `specs/shared/errors.yaml` — error codes with HTTP status mappings
   - `specs/shared/types.yaml` — shared enums and value objects (if exists)
   - `specs/schemas/_base.yaml` — base model fields
   - `specs/schemas/*.yaml` — all data model definitions

3. **Check for existing scaffold**: If the project already has a `package.json` (or equivalent for the tech stack), `tsconfig.json`, and a `src/` directory with middleware/config files, tell the user the project appears already scaffolded. Show what exists and ask if they want to re-scaffold (overwrite) or skip.

4. **Initialize the project** based on `specs/system.yaml` tech stack:

   **Package setup:**
   - Create `package.json` (or equivalent) with project name, version, scripts (dev, build, start, test, lint)
   - Create `tsconfig.json` / language config matching the framework
   - Install dependencies listed in `specs/architecture.yaml` → `dependencies`
   - Install dev dependencies (test framework, linter, type checker)

   **Project structure** based on `specs/architecture.yaml` → `project_structure`:
   - Create all directories (e.g., `src/routes/`, `src/services/`, `src/repositories/`, `src/middleware/`, `src/utils/`, `src/types/`)
   - Follow the naming conventions from architecture spec

5. **Generate shared infrastructure:**

   **Config loader** from `specs/config.yaml`:
   - Config file that reads environment variables
   - Validation for required variables (fail-fast on missing)
   - Type-safe config object

   **Error handling** from `specs/shared/errors.yaml`:
   - Error class/factory with code, message, HTTP status
   - Error handler middleware (catches errors, formats responses)
   - All error codes from the errors spec

   **Shared types** from `specs/shared/types.yaml` (if exists):
   - Enum definitions
   - Value object types

   **Database setup** from `specs/schemas/`:
   - ORM schema/models from all schema YAML files (e.g., Prisma schema, TypeORM entities, Drizzle schema)
   - Base model fields from `_base.yaml` applied to all models
   - Relationships, indexes, and constraints from schema specs
   - Migration or sync command in package.json scripts
   - If schemas have `transitions:`, generate state machine validation helpers

   **App entry point:**
   - Main application file (e.g., `src/app.ts`, `src/index.ts`)
   - Server setup with middleware stack (CORS, body parsing, auth, rate limiting, error handler)
   - Route registration placeholder (empty, will be filled by `/ddd-implement`)
   - Graceful shutdown handler

   **Integration clients** from `specs/system.yaml` → `integrations` (if exists):
   - HTTP client wrapper per integration with base URL, auth, retry, rate limiting
   - Typed client interface

   **Event infrastructure** (if any domain has `publishes_events` or `consumes_events`):
   - Event bus setup (in-memory for dev, or queue-based per architecture spec)
   - Event type definitions from domain event specs

   **Testing setup:**
   - Test configuration file (jest.config.ts, vitest.config.ts, etc.)
   - Test utilities (factory functions for models, mock helpers)
   - Example test to verify setup works

   **Cross-cutting utilities** from `specs/architecture.yaml` → `cross_cutting_patterns` (if exists):
   - For each pattern that has a `utility` field:
     - Generate the utility file at the specified path (e.g., `src/utils/stealth-http.ts`)
     - Use the pattern's `config`, and `convention` to shape the utility implementation
     - Add exports to a barrel file if the project uses one
   - If no `cross_cutting_patterns` section exists, skip this step

6. **Create environment files:**
   - `.env.example` from `specs/config.yaml` (all variables with placeholder values)
   - `.env` with development defaults (if safe — no real secrets)
   - `.gitignore` (node_modules, dist, .env, .ddd/autosave, etc.)

7. **Create Docker setup** (if `specs/architecture.yaml` mentions Docker or containerization):
   - `Dockerfile` with multi-stage build
   - `docker-compose.yaml` with app + database + cache services

8. **Verify the scaffold:**
   - Run the build command — should compile without errors
   - Run the test command — example test should pass
   - If either fails, fix and retry

9. **Initialize `.ddd/` tracking:**
   - Create `.ddd/mapping.yaml` with empty `flows:` section (populated by `/ddd-implement`)
   - Create `.ddd/annotations/` directory with `.gitkeep` (populated by `/ddd-reflect`)

10. **Summary**: After scaffolding, show:
    ```
    Scaffolded: {project-name}
    Tech stack: {language} / {framework} / {database}

    Created:
      package.json              (dependencies + scripts)
      tsconfig.json             (TypeScript config)
      src/app.ts                (entry point + middleware)
      src/config/index.ts       (env config loader)
      src/errors/index.ts       (error codes + handler)
      src/types/shared.ts       (shared enums)
      src/db/schema.prisma      (database schema)
      src/middleware/            (auth, rate-limit, error-handler)
      src/utils/                (test helpers)
      .env.example              (environment template)
      .gitignore
      docker-compose.yaml
      jest.config.ts

    Models: user, order, payment (3 schemas)
    Error codes: 8 defined
    Integrations: stripe, sendgrid (2 clients)

    Build: OK
    Tests: 1/1 passing

    Next steps:
      1. Copy .env.example to .env and fill in values
      2. Run database migration/sync
      3. Run /ddd-implement --all to generate flow code
    ```

$ARGUMENTS
