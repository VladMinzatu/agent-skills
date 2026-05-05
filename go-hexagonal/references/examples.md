# Hexagonal Architecture Examples

Real-world code examples for common patterns and use cases.

## Example 1: User Signup Flow

Complete implementation of a user signup use case.

### Domain Entity

```go
// internal/domain/user/user.go
package user

import (
    "errors"
    "regexp"
    "time"
)

var (
    ErrInvalidName  = errors.New("name must be non-empty")
    ErrInvalidEmail = errors.New("invalid email format")
)

type User struct {
    id        string
    name      string
    email     string
    password  string // hashed
    createdAt time.Time
}

func NewUser(id, name, email, hashedPassword string) (*User, error) {
    if len(name) == 0 {
        return nil, ErrInvalidName
    }
    if !isValidEmail(email) {
        return nil, ErrInvalidEmail
    }
    return &User{
        id:        id,
        name:      name,
        email:     email,
        password:  hashedPassword,
        createdAt: time.Now(),
    }, nil
}

func (u *User) ID() string        { return u.id }
func (u *User) Name() string      { return u.name }
func (u *User) Email() string     { return u.email }
func (u *User) CreatedAt() time.Time { return u.createdAt }

func (u *User) MatchPassword(plain string) bool {
    // Use bcrypt comparison
    return true // simplified
}

func isValidEmail(email string) bool {
    re := regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)
    return re.MatchString(email)
}
```

### Ports

```go
// internal/domain/user/repository.go
package user

import "context"

type Repository interface {
    Save(ctx context.Context, user *User) error
    ByEmail(ctx context.Context, email string) (*User, error)
    ByID(ctx context.Context, id string) (*User, error)
}

// internal/domain/user/event_publisher.go
package user

type UserSignedUpEvent struct {
    UserID    string
    Email     string
    SignedUpAt time.Time
}

type EventPublisher interface {
    Publish(ctx context.Context, event interface{}) error
}
```

### Domain Service

```go
// internal/domain/user/signup_service.go
package user

import (
    "context"
    "crypto/rand"
    "encoding/hex"
    "errors"
    "golang.org/x/crypto/bcrypt"
)

var ErrEmailAlreadyTaken = errors.New("email already registered")

type SignUpService struct {
    repo      Repository
    publisher EventPublisher
}

func NewSignUpService(repo Repository, pub EventPublisher) *SignUpService {
    return &SignUpService{repo: repo, publisher: pub}
}

func (s *SignUpService) Execute(
    ctx context.Context,
    name, email, password string,
) (*User, error) {
    // Check if email already taken
    existing, err := s.repo.ByEmail(ctx, email)
    if err == nil && existing != nil {
        return nil, ErrEmailAlreadyTaken
    }
    
    // Create domain entity (validates invariants)
    hashed, _ := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    id := generateID()
    user, err := NewUser(id, name, email, string(hashed))
    if err != nil {
        return nil, err
    }
    
    // Persist
    if err := s.repo.Save(ctx, user); err != nil {
        return nil, err
    }
    
    // Publish event
    s.publisher.Publish(ctx, UserSignedUpEvent{
        UserID:     user.ID(),
        Email:      user.Email(),
        SignedUpAt: user.CreatedAt(),
    })
    
    return user, nil
}

func generateID() string {
    b := make([]byte, 16)
    rand.Read(b)
    return hex.EncodeToString(b)
}
```

### HTTP Handler (Primary Adapter)

```go
// internal/adapters/http/handler/signup_handler.go
package handler

import (
    "encoding/json"
    "errors"
    "net/http"
    "github.com/myapp/internal/domain/user"
)

type SignUpHandler struct {
    service *user.SignUpService
}

func NewSignUpHandler(svc *user.SignUpService) *SignUpHandler {
    return &SignUpHandler{service: svc}
}

type SignUpRequest struct {
    Name     string `json:"name"`
    Email    string `json:"email"`
    Password string `json:"password"`
}

type SignUpResponse struct {
    ID    string `json:"id"`
    Email string `json:"email"`
}

func (h *SignUpHandler) SignUp(w http.ResponseWriter, r *http.Request) {
    var req SignUpRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }
    
    u, err := h.service.Execute(r.Context(), req.Name, req.Email, req.Password)
    if err != nil {
        switch {
        case errors.Is(err, user.ErrEmailAlreadyTaken):
            http.Error(w, err.Error(), http.StatusConflict)
        case errors.Is(err, user.ErrInvalidEmail):
            http.Error(w, err.Error(), http.StatusBadRequest)
        default:
            http.Error(w, "internal error", http.StatusInternalServerError)
        }
        return
    }
    
    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(SignUpResponse{
        ID:    u.ID(),
        Email: u.Email(),
    })
}
```

### PostgreSQL Adapter (Secondary)

```go
// internal/adapters/postgres/user_repository.go
package postgres

import (
    "context"
    "database/sql"
    "errors"
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
        `INSERT INTO users (id, name, email, password, created_at)
         VALUES ($1, $2, $3, $4, $5)`,
        u.ID(), u.Name(), u.Email(), u.password, u.CreatedAt(),
    )
    return err
}

func (r *UserRepository) ByEmail(ctx context.Context, email string) (*user.User, error) {
    var id, name, password string
    var createdAt time.Time
    
    err := r.db.QueryRowContext(ctx,
        `SELECT id, name, password, created_at FROM users WHERE email = $1`,
        email,
    ).Scan(&id, &name, &password, &createdAt)
    
    if err == sql.ErrNoRows {
        return nil, nil
    }
    if err != nil {
        return nil, err
    }
    
    // Reconstruct entity (encapsulation)
    u := &user.User{
        id:        id,
        name:      name,
        email:     email,
        password:  password,
        createdAt: createdAt,
    }
    return u, nil
}

func (r *UserRepository) ByID(ctx context.Context, id string) (*user.User, error) {
    // Similar to ByEmail
    return nil, nil
}
```

### Kafka Event Publisher (Secondary)

```go
// internal/adapters/kafka/event_publisher.go
package kafka

import (
    "context"
    "encoding/json"
    "github.com/segmentio/kafka-go"
    "github.com/myapp/internal/domain/user"
)

type EventPublisher struct {
    writer *kafka.Writer
}

func NewEventPublisher(brokers []string) *EventPublisher {
    return &EventPublisher{
        writer: &kafka.Writer{
            Addr:                   kafka.TCP(brokers...),
            Topic:                  "events",
            Compression:            kafka.Snappy,
            AllowAutoTopicCreation: true,
        },
    }
}

func (p *EventPublisher) Publish(ctx context.Context, event interface{}) error {
    data, err := json.Marshal(event)
    if err != nil {
        return err
    }
    
    eventType := "UserSignedUp"
    if _, ok := event.(user.UserSignedUpEvent); !ok {
        eventType = "UnknownEvent"
    }
    
    return p.writer.WriteMessages(ctx, kafka.Message{
        Key:   []byte(eventType),
        Value: data,
    })
}

func (p *EventPublisher) Close() error {
    return p.writer.Close()
}
```

### Wiring (main.go)

```go
// cmd/server/main.go
package main

import (
    "database/sql"
    "net/http"
    "os"
    
    _ "github.com/lib/pq"
    "github.com/myapp/internal/adapters/http/handler"
    "github.com/myapp/internal/adapters/kafka"
    "github.com/myapp/internal/adapters/postgres"
    "github.com/myapp/internal/domain/user"
)

func main() {
    // Setup database
    db, err := sql.Open("postgres", os.Getenv("DATABASE_URL"))
    if err != nil {
        panic(err)
    }
    defer db.Close()
    
    // Setup message broker
    brokers := []string{"localhost:9092"}
    eventPub := kafka.NewEventPublisher(brokers)
    defer eventPub.Close()
    
    // Wire secondary adapters
    userRepo := postgres.NewUserRepository(db)
    
    // Wire domain services
    signUpSvc := user.NewSignUpService(userRepo, eventPub)
    
    // Wire primary adapters
    signUpHandler := handler.NewSignUpHandler(signUpSvc)
    
    // Setup HTTP router
    mux := http.NewServeMux()
    mux.HandleFunc("POST /signup", signUpHandler.SignUp)
    
    // Start server
    http.ListenAndServe(":8080", mux)
}
```

### Unit Tests

```go
// internal/domain/user/signup_service_test.go
package user

import (
    "context"
    "testing"
    "github.com/golang/mock/gomock"
    "github.com/myapp/internal/adapters/mocks"
    "github.com/stretchr/testify/assert"
)

func TestSignUpService_Success(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()
    
    mockRepo := mocks.NewMockRepository(ctrl)
    mockPub := mocks.NewMockEventPublisher(ctrl)
    
    mockRepo.EXPECT().
        ByEmail(gomock.Any(), "test@example.com").
        Return(nil, nil)  // Email not taken
    
    mockRepo.EXPECT().
        Save(gomock.Any(), gomock.Any()).
        Return(nil)
    
    mockPub.EXPECT().
        Publish(gomock.Any(), gomock.Any()).
        Return(nil)
    
    svc := NewSignUpService(mockRepo, mockPub)
    
    u, err := svc.Execute(context.Background(), "John", "test@example.com", "password123")
    
    assert.NoError(t, err)
    assert.NotNil(t, u)
    assert.Equal(t, "John", u.Name())
}

func TestSignUpService_EmailTaken(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()
    
    mockRepo := mocks.NewMockRepository(ctrl)
    mockPub := mocks.NewMockEventPublisher(ctrl)
    
    existingUser := &User{
        id:    "123",
        name:  "Existing",
        email: "test@example.com",
    }
    
    mockRepo.EXPECT().
        ByEmail(gomock.Any(), "test@example.com").
        Return(existingUser, nil)  // Email already taken
    
    svc := NewSignUpService(mockRepo, mockPub)
    
    _, err := svc.Execute(context.Background(), "John", "test@example.com", "password123")
    
    assert.Equal(t, ErrEmailAlreadyTaken, err)
}
```

### Integration Tests

```go
// test/integration/postgres/user_repository_test.go
package postgres

import (
    "context"
    "database/sql"
    "testing"
    "github.com/myapp/internal/adapters/postgres"
    "github.com/myapp/internal/domain/user"
    "github.com/stretchr/testify/assert"
)

func TestUserRepository_Save_and_ByEmail(t *testing.T) {
    db := setupTestDB()
    defer db.Close()
    
    repo := postgres.NewUserRepository(db)
    ctx := context.Background()
    
    // Create and save user
    u, _ := user.NewUser("id1", "John", "john@example.com", "hashedpass")
    err := repo.Save(ctx, u)
    assert.NoError(t, err)
    
    // Retrieve by email
    retrieved, err := repo.ByEmail(ctx, "john@example.com")
    assert.NoError(t, err)
    assert.NotNil(t, retrieved)
    assert.Equal(t, "id1", retrieved.ID())
    assert.Equal(t, "John", retrieved.Name())
}

func setupTestDB() *sql.DB {
    // Use test database or in-memory database
    db, _ := sql.Open("postgres", "postgres://test:test@localhost/test_db")
    // Run migrations
    return db
}
```

## Example 2: Event-Driven Architecture

Handling events published by domain services.

### Domain Event

```go
// internal/domain/article/events.go
package article

import "time"

type ArticlePublishedEvent struct {
    ArticleID string
    Title     string
    AuthorID  string
    PublishedAt time.Time
}

type ArticleCommentedEvent struct {
    ArticleID string
    CommentID string
    AuthorID  string
    CommentedAt time.Time
}
```

### Event Consumer (Primary Adapter)

```go
// internal/adapters/kafka/article_event_consumer.go
package kafka

import (
    "context"
    "encoding/json"
    "log"
    "github.com/segmentio/kafka-go"
    "github.com/myapp/internal/domain/article"
)

type ArticleEventConsumer struct {
    reader   *kafka.Reader
    handlers map[string]func(context.Context, []byte) error
}

func NewArticleEventConsumer(brokers []string) *ArticleEventConsumer {
    return &ArticleEventConsumer{
        reader: kafka.NewReader(kafka.ReaderConfig{
            Brokers: brokers,
            Topic:   "events",
            GroupID: "article-service",
        }),
        handlers: make(map[string]func(context.Context, []byte) error),
    }
}

func (c *ArticleEventConsumer) RegisterHandler(
    eventType string,
    handler func(context.Context, []byte) error,
) {
    c.handlers[eventType] = handler
}

func (c *ArticleEventConsumer) Start(ctx context.Context) {
    for {
        msg, err := c.reader.ReadMessage(ctx)
        if err != nil {
            log.Printf("read error: %v", err)
            continue
        }
        
        if handler, ok := c.handlers[string(msg.Key)]; ok {
            if err := handler(ctx, msg.Value); err != nil {
                log.Printf("handle error: %v", err)
            }
        }
    }
}

func (c *ArticleEventConsumer) Close() error {
    return c.reader.Close()
}
```

### Event Handler (Adapter)

```go
// internal/adapters/event_handlers/article_published_handler.go
package handlers

import (
    "context"
    "encoding/json"
    "github.com/myapp/internal/domain/article"
    "github.com/myapp/internal/domain/notification"
)

type ArticlePublishedHandler struct {
    notificationService *notification.SendNotificationService
}

func NewArticlePublishedHandler(svc *notification.SendNotificationService) *ArticlePublishedHandler {
    return &ArticlePublishedHandler{notificationService: svc}
}

func (h *ArticlePublishedHandler) Handle(ctx context.Context, data []byte) error {
    var event article.ArticlePublishedEvent
    if err := json.Unmarshal(data, &event); err != nil {
        return err
    }
    
    // Use domain service to send notification
    return h.notificationService.NotifyArticlePublished(ctx, event.ArticleID, event.AuthorID)
}
```

## Example 3: Multiple Transports

Same business logic served via HTTP and gRPC.

### gRPC Adapter (Primary)

```go
// internal/adapters/grpc/article_service.go
package grpc

import (
    "context"
    "github.com/myapp/internal/domain/article"
    pb "github.com/myapp/proto"
)

type ArticleService struct {
    pb.UnimplementedArticleServiceServer
    publishService *article.PublishArticleService
}

func NewArticleService(svc *article.PublishArticleService) *ArticleService {
    return &ArticleService{publishService: svc}
}

func (s *ArticleService) PublishArticle(
    ctx context.Context,
    req *pb.PublishArticleRequest,
) (*pb.PublishArticleResponse, error) {
    a, err := s.publishService.Execute(
        ctx,
        req.Title,
        req.Content,
        req.AuthorId,
    )
    if err != nil {
        return nil, err
    }
    
    return &pb.PublishArticleResponse{
        ArticleId: a.ID(),
        Title:     a.Title(),
    }, nil
}
```

### Wiring Both Transports

```go
// cmd/server/main.go
package main

import (
    "net"
    "net/http"
    
    grpc_adapter "github.com/myapp/internal/adapters/grpc"
    http_adapter "github.com/myapp/internal/adapters/http"
    "github.com/myapp/internal/adapters/postgres"
    "github.com/myapp/internal/domain/article"
    pb "github.com/myapp/proto"
    "google.golang.org/grpc"
)

func main() {
    // Setup dependencies...
    articleRepo := postgres.NewArticleRepository(db)
    publishSvc := article.NewPublishArticleService(articleRepo)
    
    // HTTP transport
    go func() {
        httpHandler := http_adapter.NewArticleHandler(publishSvc)
        mux := http.NewServeMux()
        mux.HandleFunc("POST /articles", httpHandler.Publish)
        http.ListenAndServe(":8080", mux)
    }()
    
    // gRPC transport
    go func() {
        listener, _ := net.Listen("tcp", ":50051")
        grpcServer := grpc.NewServer()
        
        grpcHandler := grpc_adapter.NewArticleService(publishSvc)
        pb.RegisterArticleServiceServer(grpcServer, grpcHandler)
        
        grpcServer.Serve(listener)
    }()
    
    select {}
}
```

Both transports use the same domain service — business logic unchanged.
