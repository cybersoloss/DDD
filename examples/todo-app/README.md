# Todo App — DDD Example Project

A minimal but complete DDD project demonstrating Design Driven Development with a simple todo application.

## What This Is

This is a "try it in 5 minutes" example of DDD. It shows:

- **Project structure** — how to organize specs
- **Two domains** — Users (authentication) and Todos (task management)
- **Four flows** — user registration, user login, create todo, list todos
- **Data models** — User and Todo schemas with relationships
- **Event wiring** — UserRegistered and UserLoggedIn events, with Todos listening to UserRegistered

## Files Overview

```
todo-app/
  ddd-project.json                    # Project config with 2 domains
  specs/
    system.yaml                        # Tech stack (Node.js, Express, PostgreSQL)
    architecture.yaml                  # Conventions and project structure
    config.yaml                        # Environment variables (DATABASE_URL, JWT_SECRET, PORT)
    shared/
      errors.yaml                      # Error codes (VALIDATION_ERROR, UNAUTHORIZED, etc.)
    schemas/
      _base.yaml                       # Base model (id, timestamps)
      user.yaml                        # User model (email, password_hash, name)
      todo.yaml                        # Todo model (title, completed, user_id)
    domains/
      users/
        domain.yaml                    # Users domain: 2 flows, publishes UserRegistered + UserLoggedIn
        flows/
          user-register.yaml           # Full flow: trigger → validate → check email → create → emit → terminal
          user-login.yaml              # Full flow: trigger → validate → find user → verify password → generate JWT → emit → terminal
      todos/
        domain.yaml                    # Todos domain: 2 flows, consumes UserRegistered
        flows/
          create-todo.yaml             # Full flow: trigger → validate → auth check → create → emit → terminal
          list-todos.yaml              # Full flow: trigger → auth check → query → terminal
```

## How to Use

1. **Scaffold the project** (creates boilerplate code from specs):
   ```bash
   /ddd-scaffold
   ```

2. **Implement flows** (generates service code, routes, tests):
   ```bash
   /ddd-implement --all
   ```

3. **Run tests** (validates generated code):
   ```bash
   /ddd-test --all
   ```

4. **Check status** (see what's been implemented):
   ```bash
   /ddd-status
   ```

## Key Design Patterns

### User Registration Flow
- **Trigger:** HTTP POST `/api/users/register`
- **Input validation:** email, password, name
- **Decision:** Check if email already exists
- **Process:** Hash password, create user
- **Event:** Emit UserRegistered
- **Terminal:** Return 201 with user + JWT

### User Login Flow
- **Trigger:** HTTP POST `/api/users/login`
- **Input validation:** email, password
- **Data store:** Query user by email
- **Decision:** Verify password hash
- **Process:** Generate JWT token
- **Event:** Emit UserLoggedIn
- **Terminal:** Return 200 with user + JWT

### Create Todo Flow
- **Trigger:** HTTP POST `/api/todos`
- **Input validation:** title, optional description
- **Decision:** Check user authentication
- **Process:** Create todo record
- **Event:** Emit TodoCreated
- **Terminal:** Return 201 with todo

### List Todos Flow
- **Trigger:** HTTP GET `/api/todos`
- **Decision:** Check user authentication
- **Data store:** Query todos filtered by user_id
- **Terminal:** Return 200 with array of todos

## Tech Stack

- **Language:** TypeScript
- **Runtime:** Node.js 20
- **Framework:** Express 4
- **Database:** PostgreSQL 16
- **ORM:** Prisma
- **Auth:** JWT + bcrypt

## Learn More

For detailed DDD specification docs, see:
- `~/dev/DDD/DDD-USAGE-GUIDE.md` — complete reference for writing specs
- `~/dev/DDD/templates/` — reusable YAML templates for common patterns

## Next Steps

- Modify flows by editing `.yaml` files and re-running `/ddd-implement`
- Add new flows by creating new `.yaml` files in `flows/` subdirectories
- Add cross-domain dependencies in domain YAML
- Run `/ddd-reflect` to capture implementation wisdom back into specs
