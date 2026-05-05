# Interactive Hexagonal Project Setup

This guide walks you through setting up a new Go hexagonal architecture project from scratch.

## How to Use This Skill

**Option A: Via Claude Code CLI**
```bash
claude code --skill go-hexagonal-scaffold --args "setup"
```

**Option B: In Claude Code IDE**
Type `/go-hexagonal-scaffold setup` (or similar, depending on how skills are invoked in your environment)

The skill will ask you questions, then generate a working skeleton project with:
- Directory structure (`cmd/`, `internal/domain/`, `internal/adapters/`)
- One complete domain concept with basic service
- One adapter (storage or primary)
- Project-specific `CLAUDE.md` with guidance tailored to your choices
- Minimal but runnable code

From there, **you own the development process**—the CLAUDE.md will guide decisions as you add features.

---

## Setup Questions

The skill will ask:

### 1. Project name
```
What's your project name? (e.g., "user-api", "article-service")
```
Used for `go.mod` and directory structure.

### 2. First domain concept
```
What's your first domain entity? (e.g., "User", "Article", "Order")
```
This becomes the first domain package. The skill generates a minimal entity, repository interface, and domain service for this.

### 3. Primary storage backend
```
What's your primary storage? (PostgreSQL, MongoDB, in-memory)
```
Generates a basic adapter. The skill creates `internal/adapters/{storage}/` with a repository implementation.

### 4. Other adapters (optional)
```
Do you need event publishing? (y/n)
Do you need gRPC? (y/n)
Do you need caching? (y/n)
```
Generates minimal adapter stubs. The skill won't fully implement these—it just creates the package structure and a placeholder, so you know where to add them.

---

## What Gets Generated

### Directory Structure
```
myproject/
├── go.mod
├── go.sum (initially empty)
├── CLAUDE.md                    # Project-specific guidance
├── Makefile                     # Build targets
├── cmd/
│   └── server/
│       └── main.go              # Entry point with basic wiring
├── internal/
│   ├── domain/
│   │   └── {entity}/
│   │       ├── {entity}.go      # Minimal entity
│   │       ├── repository.go    # Port interface
│   │       ├── service.go       # Domain service
│   │       └── errors.go        # Domain errors
│   └── adapters/
│       ├── {storage}/           # Repository implementation
│       └── http/                # Basic HTTP handler
└── test/
    └── .gitkeep
```

### Minimal Code Examples

**Domain Entity** (`internal/domain/{entity}/{entity}.go`)
- Basic struct with constructor
- One or two validation rules (e.g., non-empty name)
- Simple getters
- Encapsulates invariants

**Repository Interface** (`internal/domain/{entity}/repository.go`)
- Save, ByID, maybe one domain-specific query
- Not exhaustive—you'll add queries as needed

**Storage Adapter** (`internal/adapters/{storage}/{entity}_repository.go`)
- Basic implementation showing pattern
- You'll flesh this out with real schema, migrations, etc.

**Main.go** (`cmd/server/main.go`)
- Simple wiring of storage adapter + domain service + HTTP handler
- Shows dependency injection pattern
- Starts HTTP server on port 8080

**CLAUDE.md**
- Project-specific rules based on your choices
- Guidance on where to add the next domain, adapter, or test
- Decision tree for "is this domain or adapter?"
- Naming conventions for this project

---

## After Setup

The skill creates a **skeleton**, not a finished project. Next steps (guided by your project's CLAUDE.md):

1. **Run it** — `go run cmd/server/main.go` (should start without errors)
2. **Add the first route** — POST /users or similar
3. **Add tests** — Unit test the domain service
4. **Iterate** — Add another domain concept, another adapter, as needed

The generated CLAUDE.md will remind you of these patterns and keep you consistent.

---

## Customization

The skill respects your choices:
- **PostgreSQL** → uses `database/sql` (no ORM)
- **MongoDB** → uses official driver
- **Kafka events** → generates event publisher adapter stub
- **gRPC** → generates service stub (you define .proto)

But it doesn't try to template your actual code. It shows the structure and one example; you write the rest to fit your domain.

---

## Example: User Authentication Service

If you answer:
- Project: `auth-service`
- First domain: `User`
- Storage: PostgreSQL
- Other adapters: Kafka (for audit events)

You get:
```
auth-service/
├── go.mod (with postgres driver)
├── cmd/server/main.go
├── internal/domain/user/
│   ├── user.go (id, email, password_hash)
│   ├── repository.go (Save, ByEmail, ByID)
│   ├── service.go (SignUpService example)
│   └── errors.go (domain errors)
├── internal/adapters/
│   ├── postgres/
│   │   ├── user_repository.go
│   │   └── migrations/
│   ├── kafka/
│   │   └── audit_publisher.go (stub)
│   └── http/
│       └── user_handler.go
└── CLAUDE.md (tailored to auth service, postgres, kafka)
```

From there, you own adding signup logic, authentication, more routes, etc.
