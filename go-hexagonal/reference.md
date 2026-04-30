# Hexagonal Architecture Reference

## Architecture Decision Tree

Use this guide to understand when and how to apply hexagonal patterns.

### Is this pure business logic?

**YES → Domain Package**
- No external dependencies
- Pure Go, focused on correctness
- Encodes business rules and invariants
- Example: `user.NewUser()`, price calculations, discount logic

**NO → Adapter Package**
- Integrates with external systems
- Implements interfaces (ports) defined by domain
- Can be replaced without changing domain logic
- Example: PostgreSQL repository, HTTP handler, Kafka consumer

### Does this logic depend on external systems?

**NO → Keep in domain**
- Domain should remain pure and testable
- Example: User entity validation

**YES → Define a port**
- Create an interface (port) in domain package
- Adapter implements the port
- Domain uses the port (interface)
- Example: `user.Repository` interface used by domain services

### Can I swap this out for something else?

**YES → It's an adapter**
- Implement the port interface
- Multiple implementations allowed (PostgreSQL, MongoDB, in-memory)
- Example: HTTP handler could become gRPC handler

**NO → It might be a domain concept**
- Reconsider if it should be in domain
- Or define a more flexible port
- Example: User entity (core concept) vs. HTTP authentication (adapter concern)

## Package Organization

### Domain Packages

Located under `internal/domain/{entity}`:

```
internal/domain/user/
├── user.go           # Entity: aggregate root, value objects
├── repository.go     # Port: interface for persistence
├── event_publisher.go # Port: interface for events
├── service.go        # Domain service: use case logic
├── errors.go         # Domain-specific error types
└── user_test.go      # Unit tests (no dependencies)
```

**Characteristics:**
- Defines interfaces (ports) that domain needs
- Only interfaces as dependencies (to adapters)
- No concrete implementations (adapters) imported
- No external packages (except stdlib)
- Pure business logic, fully testable

**Key principle:** Interfaces are defined by their **consumer**. If `user` domain needs persistence, it defines `Repository` interface. Adapters then implement it.

### Adapter Packages

Located under `internal/adapters/{transport}`:

```
internal/adapters/
├── http/             # Primary: HTTP handlers
│   ├── handler/
│   │   ├── user_handler.go
│   │   └── article_handler.go
│   ├── middleware/
│   └── router.go
├── grpc/             # Primary: gRPC services
│   └── user_service.go
├── postgres/         # Secondary: database
│   ├── user_repository.go    # Implements domain/user.Repository
│   ├── article_repository.go # Implements domain/article.Repository
│   └── migrations/
├── kafka/            # Secondary: message queue
│   ├── event_publisher.go    # Implements domain interfaces for events
│   └── event_consumer.go
├── redis/            # Secondary: cache
│   └── cache_adapter.go
└── email/            # Secondary: external service
    └── smtp_service.go
```

**Characteristics:**
- Implements domain interfaces (ports)
- Contains external library code
- Can be replaced independently
- Thin translation layer
- One adapter can implement multiple domain interfaces

## Dependency Flow

```
HTTP Request
     ↓
[HTTP Handler]  (Adapter, primary)
     ↓
[Domain Service]  (Domain, core logic)
     ↓ (uses interfaces)
[Ports]  (Interfaces defined by domain)
     ↓ (implemented by)
[PostgreSQL Repository]  (Adapter, secondary)
     ↓
Database
```

**Key Rule:** Only flow is domain → ports → adapters. Never adapters → domain.

## Entity Design Patterns

### Aggregate Root

An entity that owns other entities (aggregates) and maintains invariants:

```go
package article

// Article is an aggregate root
type Article struct {
    id        string
    title     string
    content   string
    comments  []Comment  // Owned entities
    version   int        // Optimistic locking
}

// Constructor validates all invariants upfront
func NewArticle(id, title, content string) (*Article, error) {
    if len(title) == 0 {
        return nil, ErrTitleRequired
    }
    return &Article{
        id:      id,
        title:   title,
        content: content,
        version: 1,
    }, nil
}

// Always validate state changes
func (a *Article) UpdateTitle(newTitle string) error {
    if len(newTitle) == 0 {
        return ErrTitleRequired
    }
    a.title = newTitle
    a.version++
    return nil
}

// Expose only what's needed
func (a *Article) ID() string { return a.id }
func (a *Article) Title() string { return a.title }
```

### Value Objects

Immutable objects with no identity, defined by their attributes:

```go
package user

// Email is a value object
type Email struct {
    value string
}

func NewEmail(s string) (Email, error) {
    if !isValid(s) {
        return Email{}, ErrInvalidEmail
    }
    return Email{value: s}, nil
}

func (e Email) String() string { return e.value }

// Value objects are compared by value
func (e Email) Equal(other Email) bool {
    return e.value == other.value
}
```

### Domain Events

Immutable records of something that happened:

```go
package user

type UserCreatedEvent struct {
    UserID    string
    Email     string
    CreatedAt time.Time
}

type UserEmailChangedEvent struct {
    UserID   string
    OldEmail string
    NewEmail string
    ChangedAt time.Time
}
```

## Repository Pattern

Repositories abstract persistence and present data as if in memory:

```go
package user

// Repository defines the persistence contract
type Repository interface {
    // Save persists a new or updated user
    Save(ctx context.Context, user *User) error
    
    // ByID retrieves a user (or error if not found)
    ByID(ctx context.Context, id string) (*User, error)
    
    // ByEmail finds a user by email (domain-specific query)
    ByEmail(ctx context.Context, email string) (*User, error)
    
    // Delete removes a user
    Delete(ctx context.Context, id string) error
}
```

**Key principles:**
- Repository returns domain entities, not DTOs
- Queries are domain-specific, not generic
- Repository abstracts the technology (SQL, NoSQL, etc.)
- Multiple implementations allowed

## Service Layer

Services orchestrate domain logic and coordinate between repositories:

```go
package user

type SignUpService struct {
    repo      Repository
    publisher EventPublisher
}

func NewSignUpService(repo Repository, pub EventPublisher) *SignUpService {
    return &SignUpService{repo: repo, publisher: pub}
}

// Execute runs the use case
func (s *SignUpService) Execute(
    ctx context.Context,
    name string,
    email string,
) (*User, error) {
    // Validate domain rules
    user, err := NewUser(uuid.New().String(), name, email)
    if err != nil {
        return nil, err
    }
    
    // Check uniqueness (cross-entity invariant)
    existing, _ := s.repo.ByEmail(ctx, email)
    if existing != nil {
        return nil, ErrEmailTaken
    }
    
    // Persist
    if err := s.repo.Save(ctx, user); err != nil {
        return nil, err
    }
    
    // Publish event
    s.publisher.Publish(ctx, UserSignedUpEvent{
        UserID: user.ID(),
        Email:  email,
    })
    
    return user, nil
}
```

## Error Handling

Define domain-specific errors:

```go
package user

var (
    ErrNotFound       = errors.New("user not found")
    ErrInvalidName    = errors.New("name must not be empty")
    ErrInvalidEmail   = errors.New("invalid email format")
    ErrEmailTaken     = errors.New("email already registered")
    ErrDuplicateEmail = errors.New("email already in use")
)

// Custom error type for domain errors
type DomainError struct {
    code    string
    message string
}

func (e *DomainError) Error() string {
    return e.message
}

func (e *DomainError) Code() string {
    return e.code
}
```

Adapters translate domain errors to HTTP status codes:

```go
func handleError(w http.ResponseWriter, err error) {
    switch err {
    case user.ErrNotFound:
        http.Error(w, err.Error(), http.StatusNotFound)
    case user.ErrInvalidEmail:
        http.Error(w, err.Error(), http.StatusBadRequest)
    case user.ErrEmailTaken:
        http.Error(w, err.Error(), http.StatusConflict)
    default:
        http.Error(w, "internal error", http.StatusInternalServerError)
    }
}
```

## Mocking and Testing

### Generate mocks automatically

Use mockgen to create mocks from interfaces:

```bash
# In domain/user/repository.go
//go:generate mockgen -destination ../../adapters/mocks/mock_user_repository.go . Repository
```

Run: `go generate ./...`

### Unit tests (domain logic)

No setup needed, test in isolation:

```go
func TestSignUpService_DuplicateEmail(t *testing.T) {
    mockRepo := mocks.NewMockRepository(ctrl)
    mockPub := mocks.NewMockEventPublisher(ctrl)
    
    mockRepo.EXPECT().
        ByEmail(gomock.Any(), "test@example.com").
        Return(&user.User{}, nil)  // Email already exists
    
    svc := user.NewSignUpService(mockRepo, mockPub)
    
    _, err := svc.Execute(context.Background(), "John", "test@example.com")
    assert.Equal(t, user.ErrEmailTaken, err)
}
```

### Integration tests (adapters)

Test against real dependencies:

```go
func TestPostgresRepository_Save(t *testing.T) {
    db := setupTestDB()
    defer db.Close()
    
    repo := postgres.NewUserRepository(db)
    user := createTestUser()
    
    err := repo.Save(context.Background(), user)
    assert.NoError(t, err)
    
    // Verify in database
    retrieved, err := repo.ByID(context.Background(), user.ID())
    assert.NoError(t, err)
    assert.Equal(t, user.ID(), retrieved.ID())
}
```

## Configuration and Dependency Injection

### Manual wiring (small projects)

Wire in main.go:

```go
func main() {
    db := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    userRepo := postgres.NewUserRepository(db)
    eventPub := kafka.NewEventPublisher()
    signUpSvc := user.NewSignUpService(userRepo, eventPub)
    handler := http.NewSignUpHandler(signUpSvc)
    
    router := chi.NewRouter()
    router.Post("/signup", handler.SignUp)
    http.ListenAndServe(":8080", router)
}
```

### Auto-wiring with wire

For larger projects:

```go
// wire.go
//go:build wireinject
// +build wireinject

package main

import "github.com/google/wire"

func initializeApp() *App {
    wire.Build(
        provideDB,
        postgres.NewUserRepository,
        kafka.NewEventPublisher,
        user.NewSignUpService,
        http.NewSignUpHandler,
        newApp,
    )
    return nil
}

func provideDB() *sql.DB {
    return sql.Open("postgres", os.Getenv("DATABASE_URL"))
}
```

## Anti-Patterns to Avoid

### 1. Anemic Domain

Domain objects with no logic, just data:

```go
// BAD: Anemic domain
type User struct {
    ID    string
    Name  string
    Email string
}

// All logic in service
func (s *Service) CreateUser(name, email string) error {
    if len(name) == 0 {  // ← Business rule in service
        return fmt.Errorf("name required")
    }
}

// GOOD: Business logic in domain
func NewUser(id, name, email string) (*User, error) {
    if len(name) == 0 {  // ← Business rule in entity
        return nil, ErrNameRequired
    }
    return &User{...}, nil
}
```

### 2. Leaky Abstractions

Adapters exposing internal details:

```go
// BAD: Leaky abstraction
type User struct {
    ID        int64     // Database ID
    CreatedAt time.Time // Database timestamp
}

// GOOD: Hide infrastructure details
func (r *Repository) ByID(ctx context.Context, id string) (*user.User, error) {
    // Query database internally
    // Return domain entity, not row
}
```

### 3. God Objects

Entities with too many responsibilities:

```go
// BAD: Too much
type User struct {
    // Identity
    id string
    
    // Relationships
    articles []Article
    followers []User
    
    // Behavior
    func (u *User) SendEmail() {}
    func (u *User) PostArticle() {}
}

// GOOD: Separate concerns
type User struct {
    id string
    // Only user identity
}

type ArticleService struct {
    // Publishing articles
}

type FollowService struct {
    // Managing relationships
}
```

### 4. Repository Queries

Repositories with generic query methods:

```go
// BAD: Generic, not domain-driven
type Repository interface {
    Query(query string, args ...interface{}) ([]User, error)
}

// GOOD: Domain-specific queries
type Repository interface {
    ByID(ctx context.Context, id string) (*User, error)
    ByEmail(ctx context.Context, email string) (*User, error)
    Active(ctx context.Context) ([]*User, error)
}
```

### 5. Circular Dependencies

Adapters importing each other:

```go
// BAD: http imports kafka
import "github.com/myapp/internal/adapters/kafka"

func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
    // ... call kafka directly
}

// GOOD: Route through domain
import "github.com/myapp/internal/domain/user"

func (h *Handler) Create(w http.ResponseWriter, r *http.Request) {
    u, err := h.service.Execute(r.Context(), ...)  // Service handles events
}
```
