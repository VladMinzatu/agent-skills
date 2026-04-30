---
name: go-hexagonal
description: "Scaffold a Go project using hexagonal (ports-and-adapters) architecture. Blends standard Go package structure with domain-driven design principles. Triggers on: creating a new Go service, hexagonal architecture, ports and adapters, domain-driven design, project structure, cleanly separating business logic from external dependencies."
---

# Go Hexagonal Architecture

Create a production-ready Go project structure that combines idiomatic Go package organization with hexagonal (ports-and-adapters) architecture.

**Supporting files (load when needed):**
- [reference.md](reference.md) - Architecture patterns, principles, and decision tree
- [examples.md](examples.md) - Real-world project examples and common patterns

## Overview

Hexagonal architecture (ports-and-adapters) isolates business logic from external dependencies. This skill helps you understand and implement this pattern in Go, with two main uses:

1. **Learn the principles** — Read this SKILL.md, [reference.md](reference.md), and [examples.md](examples.md) to understand the architecture
2. **Scaffold a new project** — Use the interactive setup to generate a working skeleton project tailored to your needs (see [setup-interactive.md](setup-interactive.md))

A hexagonal architecture project:

- **Protects domain from infrastructure** — business logic depends only on abstractions (interfaces)
- **Enables unit testing** — mock any external dependency via interfaces
- **Supports multiple transports** — swap HTTP for gRPC, PostgreSQL for MongoDB without touching domain code
- **Respects Go idioms** — uses standard project layout while maintaining clean architecture

## Architecture Principles

### Three Core Layers

1. **Domain (Business Logic)**
   - Pure Go, no external dependencies for input and output
   - Defines entities, value objects, aggregates and domain services
   - Encodes business rules and invariants
   - No infrastructure concerns (HTTP, databases, queues)

2. **Ports (Interfaces)**
   - Define contracts between domain and external systems
   - Defined by the domain package that needs them (consumer principle)
   - Implemented by adapters
   - Primary ports (input/driven): driven by external actors (HTTP handlers, event consumers)
   - Secondary ports (output/driver): driven by domain (database access, external API calls)

3. **Adapters (Implementation)**
   - Translate between domain and external systems
   - HTTP, gRPC, database drivers, message queues
   - Can be swapped without changing domain or ports
   - Often thin wrappers around external libraries

### Go Integration

Standard Go practices maintained:

```
myapp/
├── go.mod                    # Standard Go module
├── cmd/
│   ├── server/              # Binary entry points
│   └── cli/
├── internal/
│   ├── domain/              # Core business logic (pure Go, defines interfaces/ports)
│   └── adapters/            # External system implementations
├── pkg/                      # Optional: exported library code
└── test/                     # Integration and acceptance tests
```

Thus, we follow the [standard Go project layout](https://github.com/golang-standards/project-layout) while enforcing hexagonal boundaries via package visibility (`internal/` prevents external imports).

**Key principle**: Interfaces are defined by their **consumer** in domain packages (e.g., `domain/user/repository.go`), not in a separate `/ports` directory. Adapters implement these interfaces — and one adapter can implement multiple interfaces.

## Project Structure

### Full Template

```
myapp/
├── go.mod
├── go.sum
├── CLAUDE.md                   # Project-specific guidance
├── Makefile                    # Build targets
│
├── cmd/
│   ├── server/
│   │   └── main.go            # HTTP server entry point
│   └── cli/
│       └── main.go            # CLI entry point (optional)
│
├── internal/
│   ├── domain/                # Core business logic
│   │   ├── user/
│   │   │   ├── user.go        # Entity: User aggregate root
│   │   │   ├── repository.go  # Port: repository interface
│   │   │   ├── service.go     # Domain service
│   │   │   └── errors.go      # Domain errors
│   │   └── article/
│   │       ├── article.go
│   │       ├── repository.go  # Port: repository interface
│   │       └── errors.go
│   │
│   └── adapters/              # External system implementations
│       ├── http/              # HTTP handlers (primary adapter)
│       │   ├── middleware/
│       │   ├── handler/
│       │   │   ├── user_handler.go
│       │   │   └── article_handler.go
│       │   ├── response.go
│       │   └── router.go
│       ├── postgres/          # PostgreSQL adapter (secondary)
│       │   ├── migrations/
│       │   ├── user_repository.go    # Implements domain/user.Repository
│       │   └── article_repository.go # Implements domain/article.Repository
│       ├── kafka/             # Kafka adapter (secondary)
│       │   └── event_publisher.go    # Implements domain interfaces
│       └── email/             # Email service adapter
│           └── smtp_service.go
│
├── pkg/                       # Optional: reusable libraries
│   └── logger/
│       └── logger.go
│
├── test/
│   ├── integration/           # Tests hitting real dependencies
│   │   └── user_repository_test.go
│   └── acceptance/            # End-to-end tests
│       └── create_user_test.go
│
└── docs/
    ├── architecture.md
    └── api.md
```

## Key Patterns

### 1. Domain Entities

Entities encapsulate business logic and maintain invariants:

```go
// internal/domain/user/user.go
package user

type User struct {
    id    string
    name  string
    email string
}

// Constructor validates invariants
func NewUser(id, name, email string) (*User, error) {
    if len(name) == 0 {
        return nil, ErrInvalidName
    }
    return &User{id: id, name: name, email: email}, nil
}

// Methods encode business rules
func (u *User) UpdateEmail(email string) error {
    if !isValidEmail(email) {
        return ErrInvalidEmail
    }
    u.email = email
    return nil
}

func (u *User) ID() string { return u.id }
```

### 2. Ports (Interfaces)

Ports define contracts; domain code depends only on ports:

```go
// internal/domain/user/repository.go
package user

//go:generate mockgen -destination ../../adapters/mocks/mock_user_repository.go . Repository

type Repository interface {
    Save(ctx context.Context, user *User) error
    ByID(ctx context.Context, id string) (*User, error)
    ByEmail(ctx context.Context, email string) (*User, error)
}

type EventPublisher interface {
    Publish(ctx context.Context, event interface{}) error
}
```

### 3. Domain Services

Encapsulate cross-entity business logic:

```go
// internal/domain/user/service.go
package user

type CreateUserService struct {
    repo      Repository
    publisher EventPublisher
}

func NewCreateUserService(repo Repository, pub EventPublisher) *CreateUserService {
    return &CreateUserService{repo: repo, publisher: pub}
}

func (s *CreateUserService) Execute(ctx context.Context, name, email string) (*User, error) {
    user, err := NewUser(uuid.New().String(), name, email)
    if err != nil {
        return nil, err
    }
    
    if err := s.repo.Save(ctx, user); err != nil {
        return nil, err
    }
    
    s.publisher.Publish(ctx, UserCreatedEvent{UserID: user.ID()})
    return user, nil
}
```

### 4. HTTP Adapter (Primary/Driven)

Handlers translate HTTP to domain operations:

```go
// internal/adapters/http/handler/user_handler.go
package handler

import (
    "encoding/json"
    "net/http"
    "github.com/myapp/internal/domain/user"
)

type UserHandler struct {
    createService *user.CreateUserService
}

func NewUserHandler(svc *user.CreateUserService) *UserHandler {
    return &UserHandler{createService: svc}
}

func (h *UserHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Name  string `json:"name"`
        Email string `json:"email"`
    }
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    u, err := h.createService.Execute(r.Context(), req.Name, req.Email)
    if err != nil {
        // Handle domain errors
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(u)
}
```

### 5. Database Adapter (Secondary/Driver)

Repositories implement ports:

```go
// internal/adapters/postgres/user_repository.go
package postgres

import (
    "context"
    "database/sql"
    "github.com/myapp/internal/domain/user"
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Save(ctx context.Context, u *user.User) error {
    _, err := r.db.ExecContext(ctx,
        "INSERT INTO users (id, name, email) VALUES ($1, $2, $3)",
        u.ID(), u.Name(), u.Email(),
    )
    return err
}

func (r *UserRepository) ByID(ctx context.Context, id string) (*user.User, error) {
    // Query and reconstruct entity
}
```

## Dependency Injection

Wire dependencies at the application boundary (main.go):

```go
// cmd/server/main.go
package main

import (
    "github.com/myapp/internal/adapters/http"
    "github.com/myapp/internal/adapters/postgres"
    "github.com/myapp/internal/domain/user"
)

func main() {
    db := setupDB()
    defer db.Close()
    
    // Wire secondary adapters (repositories, event publishers)
    userRepo := postgres.NewUserRepository(db)
    eventPub := kafka.NewEventPublisher()
    
    // Wire domain services
    createUserSvc := user.NewCreateUserService(userRepo, eventPub)
    
    // Wire primary adapters (HTTP handlers)
    userHandler := handler.NewUserHandler(createUserSvc)
    
    // Start server
    router := http.NewRouter(userHandler)
    http.ListenAndServe(":8080", router)
}
```

## Testing Strategy

### Unit Tests (Domain Logic)

No external dependencies needed:

```go
// internal/domain/user/user_test.go
func TestNewUser_InvalidEmail(t *testing.T) {
    u, err := user.NewUser("id1", "John", "invalid-email")
    assert.Error(t, err)
    assert.Nil(t, u)
}
```

### Integration Tests (Adapters)

Test adapters against real dependencies (test containers, in-memory databases):

```go
// test/integration/postgres/user_repository_test.go
func TestSave_Success(t *testing.T) {
    db := setupTestDB()
    repo := postgres.NewUserRepository(db)
    
    user := &user.User{...}
    err := repo.Save(context.Background(), user)
    assert.NoError(t, err)
}
```

### Acceptance Tests (End-to-End)

Test full flow via HTTP API or CLI:

```go
// test/acceptance/create_user_test.go
func TestCreateUserFlow(t *testing.T) {
    server := startTestServer()
    
    resp, _ := http.Post(server.URL+"/users", "application/json", ...)
    assert.Equal(t, http.StatusCreated, resp.StatusCode)
}
```

## Common Mistakes to Avoid

1. **Business logic in adapters** — Don't leak any logic into adapters - they should be integration only
2. **Adapters depending on each other** — Route through ports only
3. **Domain packages importing from adapters** — Reverse the dependency
4. **Exposing domain internals** — Use value objects, hide mutation
5. **Mixing concerns in handlers** — Handlers should be thin (translate, validate, respond) - avoid leaking business logic here.
6. **Circular imports** — Use interfaces to break cycles
7. **No error types** — Define domain-specific error types

## Go-Specific Considerations

- **Package structure enforces boundaries** — Use `internal/` to prevent accidental imports
- **Interfaces are structural** — No explicit "implements" keyword; implement all methods to satisfy
- **Receiver methods** — Use value receivers for immutable logic, pointer receivers for mutation
- **Context propagation** — Always pass context.Context as first parameter in adapters
- **Error handling** — Use sentinel errors or custom error types in domain packages

## File Naming Conventions

- **Entities** — `{noun}.go` (e.g., `user.go`, `article.go`)
- **Repositories** — `{noun}_repository.go`
- **Services** — `{verb}_{noun}_service.go` or `service.go`
- **Handlers** — `{noun}_handler.go`
- **Errors** — `errors.go`
- **Tests** — `{noun}_test.go`

## Getting Started

### Option 1: Learn the Architecture

1. Read [reference.md](reference.md) for detailed architecture decisions and patterns
2. Review [examples.md](examples.md) for real-world code examples
3. Apply these patterns to your project

### Option 2: Scaffold a New Project

For a hands-on approach with minimal groundwork, see [setup-interactive.md](setup-interactive.md). The interactive setup will:
- Ask you about your project (domain, storage, adapters)
- Generate a working skeleton project
- Create a project-specific CLAUDE.md with guidance tailored to your choices
- You own the development from there

After scaffolding:
1. Run the generated project to verify it builds
2. Add your first endpoint or feature
3. Refer to the generated CLAUDE.md for architecture decisions
4. Grow the codebase incrementally, following the principles in reference.md and examples.md
