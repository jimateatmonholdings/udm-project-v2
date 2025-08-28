# Coding Standards

**Document Version:** 1.0  
**Date:** August 28, 2025  
**Audience:** Developers  
**Status:** Active Standards  

---

## Overview

This document establishes coding standards for the Universal Data Model (UDM) Backend System, a GraphQL-first, metadata-driven architecture built with Go and React. These standards ensure code quality, maintainability, and alignment with our performance targets (<200ms query response times).

---

## Go Coding Standards

### Language Standards

#### Go Version and Setup
- **Go Version:** Go 1.21+ minimum
- **Architecture:** Single binary deployment with zero external runtime dependencies
- **Concurrency:** Utilize goroutine-based concurrency for efficient multi-tenant request handling
- **Performance Target:** Maintain <200ms GraphQL query response times

#### Formatting and Style
- **Always use `gofmt`** - Run `gofmt` on all code to automatically fix mechanical style issues
- **Indentation:** Use tabs for indentation (gofmt default)
- **Line length:** No strict limit, but wrap long lines with extra tab indentation
- **Parentheses:** Go needs fewer parentheses than C/Java for control structures

```go
// Good - minimal parentheses, proper indentation
if user.IsActive && user.HasPermission("read") {
    return user.GetData(), nil
}

// Bad - unnecessary parentheses
if (user.IsActive) && (user.HasPermission("read")) {
    return user.GetData(), nil
}
```

### Naming Conventions

#### Variables and Functions
- **Exported:** Use PascalCase for exported identifiers
- **Unexported:** Use camelCase for unexported identifiers
- **Brevity:** Favor short, descriptive names (`i` for index, `err` for errors)
- **Acronyms:** Keep consistent case (URL not Url, ID not Id)

```go
// Good naming
type UserService struct {
    dbConn *sql.DB
    logger *log.Logger
}

func (us *UserService) GetUserByID(userID string) (*User, error) {
    // Implementation
}

// Bad naming
type userservice struct {
    DbConn *sql.DB
    Logger *log.Logger
}

func (us *userservice) getUserById(userId string) (*User, error) {
    // Implementation
}
```

### Pointers vs Values

#### When to Use Values
- **Small structs** (roughly up to 3 machine words/24 bytes)
- **Immutable operations** that don't modify the original
- **Built-in types** that are already references (slices, maps, channels)

```go
// Good - small struct, value receiver for read-only operations
type Point struct {
    X, Y int
}

func (p Point) Distance() float64 {
    return math.Sqrt(float64(p.X*p.X + p.Y*p.Y))
}

// Good - built-in reference types don't need pointers
func ProcessUsers(users []User) error {
    for _, user := range users {
        // Process each user
    }
    return nil
}
```

#### When to Use Pointers
- **Large structs** to avoid copying overhead
- **Need to modify** the original value
- **API consistency** (if any method needs a pointer, use pointers everywhere)
- **Interface implementations** that require pointer receivers

```go
// Good - large struct, pointer receiver for modifications
type DatabaseConnection struct {
    Host     string
    Port     int
    Username string
    Password string
    Pool     *ConnectionPool
    Config   *DatabaseConfig
}

func (db *DatabaseConnection) Connect() error {
    // Modifies the connection state
    db.Pool = NewConnectionPool()
    return nil
}

func (db *DatabaseConnection) GetStats() ConnectionStats {
    // Even read-only methods use pointer for consistency
    return db.Pool.GetStats()
}
```

### Error Handling

#### Idiomatic Error Handling
- **Always check errors explicitly** - never ignore with `_`
- **Return errors as last parameter**
- **Use `fmt.Errorf` with `%w` verb for error wrapping**
- **Handle errors at the right level** - don't pass up if you can handle locally

```go
// Good - explicit error checking with wrapping
func ReadUserConfig(filename string) (*Config, error) {
    data, err := os.ReadFile(filename)
    if err != nil {
        return nil, fmt.Errorf("failed to read config file %s: %w", filename, err)
    }
    
    var config Config
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, fmt.Errorf("failed to parse config: %w", err)
    }
    
    return &config, nil
}

// Bad - ignoring errors
func ReadUserConfig(filename string) *Config {
    data, _ := os.ReadFile(filename)
    var config Config
    json.Unmarshal(data, &config)
    return &config
}
```

#### Custom Error Types
```go
// Good - custom error type with context
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed for field %s: %s", e.Field, e.Message)
}

func ValidateUser(user *User) error {
    if user.Email == "" {
        return &ValidationError{Field: "email", Message: "email is required"}
    }
    return nil
}

// Usage with error type checking
if err := ValidateUser(user); err != nil {
    var validationErr *ValidationError
    if errors.As(err, &validationErr) {
        // Handle validation error specifically
        log.Printf("Field validation failed: %s", validationErr.Field)
    }
    return err
}
```

#### Error Checking Patterns
```go
// Good - early return pattern
func ProcessData(data []byte) (*Result, error) {
    if len(data) == 0 {
        return nil, fmt.Errorf("empty data provided")
    }
    
    parsed, err := parseData(data)
    if err != nil {
        return nil, fmt.Errorf("parsing failed: %w", err)
    }
    
    validated, err := validateData(parsed)
    if err != nil {
        return nil, fmt.Errorf("validation failed: %w", err)
    }
    
    return processValidatedData(validated)
}
```

### Interfaces and Methods

#### Interface Design
- **Keep interfaces small** - prefer many small interfaces over few large ones
- **Define interfaces in consumer packages**, not implementer packages
- **Use interface{} sparingly** - prefer typed interfaces

```go
// Good - small, focused interface
type UserReader interface {
    GetUser(id string) (*User, error)
}

type UserWriter interface {
    SaveUser(*User) error
    DeleteUser(id string) error
}

// Compose larger interfaces when needed
type UserRepository interface {
    UserReader
    UserWriter
}

// Bad - large, unfocused interface
type UserManager interface {
    GetUser(id string) (*User, error)
    SaveUser(*User) error
    DeleteUser(id string) error
    ValidateUser(*User) error
    SendEmail(user *User, message string) error
    LogActivity(user *User, action string) error
}
```

#### Method Receivers
```go
// Good - consistent pointer receivers for API consistency
type UserService struct {
    db     Database
    logger Logger
}

func (us *UserService) GetUser(id string) (*User, error) {
    return us.db.FindUser(id)
}

func (us *UserService) CreateUser(user *User) error {
    if err := us.validateUser(user); err != nil {
        return err
    }
    return us.db.SaveUser(user)
}

func (us *UserService) validateUser(user *User) error {
    // Private method also uses pointer for consistency
    if user.Email == "" {
        return fmt.Errorf("email is required")
    }
    return nil
}
```

### Concurrency Patterns

#### Goroutines and Channels
- **Prefer channels over mutexes** for communication
- **Use context.Context for cancellation** and timeouts
- **Don't add Context to structs** - pass as method parameters

```go
// Good - channel-based communication
func ProcessUsers(ctx context.Context, users <-chan User) <-chan Result {
    results := make(chan Result)
    
    go func() {
        defer close(results)
        for {
            select {
            case user, ok := <-users:
                if !ok {
                    return
                }
                result := processUser(user)
                select {
                case results <- result:
                case <-ctx.Done():
                    return
                }
            case <-ctx.Done():
                return
            }
        }
    }()
    
    return results
}

// Good - context as parameter, not struct field
type UserService struct {
    db Database
}

func (us *UserService) GetUser(ctx context.Context, id string) (*User, error) {
    // Use context for timeout/cancellation
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()
    
    return us.db.FindUserWithContext(ctx, id)
}
```

### Package Organization

#### Package Structure
```go
// Good - clear package organization
package user

import (
    "context"
    "fmt"
    
    "github.com/company/udm/internal/database"
    "github.com/company/udm/pkg/validation"
)

// Exported types
type Service struct {
    db       database.DB
    validator validation.Validator
}

type User struct {
    ID    string `json:"id"`
    Email string `json:"email"`
}

// Exported functions
func NewService(db database.DB) *Service {
    return &Service{
        db:       db,
        validator: validation.New(),
    }
}

func (s *Service) CreateUser(ctx context.Context, email string) (*User, error) {
    // Implementation
}
```

---

## GraphQL Standards

### Schema-First Development
- **Framework:** Use gqlgen exclusively for GraphQL implementation
- **Type Safety:** Leverage gqlgen's type-safe resolver generation
- **Performance:** Implement query complexity analysis and optimization
- **API Design:** Maintain single, strongly-typed API endpoint

```go
// Good - resolver implementation
type Resolver struct {
    userService *user.Service
    logger      Logger
}

func (r *mutationResolver) CreateUser(ctx context.Context, input model.CreateUserInput) (*model.User, error) {
    if err := validateCreateUserInput(input); err != nil {
        return nil, fmt.Errorf("invalid input: %w", err)
    }
    
    user, err := r.userService.CreateUser(ctx, input.Email)
    if err != nil {
        r.logger.Error("failed to create user", "error", err, "email", input.Email)
        return nil, fmt.Errorf("failed to create user: %w", err)
    }
    
    return &model.User{
        ID:    user.ID,
        Email: user.Email,
    }, nil
}
```

---

## Database Standards

### PostgreSQL Integration
- **Database:** PostgreSQL 15+ as primary data store
- **Isolation:** Database-per-tenant isolation for compliance and security
- **Data Types:** Use JSONB for flexible UDM attribute storage
- **Connections:** Connection pooling with tenant-aware routing

```go
// Good - database layer implementation
type UserRepository struct {
    db *sql.DB
}

func (ur *UserRepository) CreateUser(ctx context.Context, user *User) error {
    query := `
        INSERT INTO users (id, email, metadata, created_at)
        VALUES ($1, $2, $3, $4)`
    
    _, err := ur.db.ExecContext(ctx, query,
        user.ID,
        user.Email,
        user.Metadata, // JSONB field
        time.Now(),
    )
    if err != nil {
        return fmt.Errorf("failed to insert user: %w", err)
    }
    
    return nil
}

func (ur *UserRepository) GetUser(ctx context.Context, id string) (*User, error) {
    query := `
        SELECT id, email, metadata, created_at
        FROM users
        WHERE id = $1`
    
    var user User
    err := ur.db.QueryRowContext(ctx, query, id).Scan(
        &user.ID,
        &user.Email,
        &user.Metadata,
        &user.CreatedAt,
    )
    if err != nil {
        if errors.Is(err, sql.ErrNoRows) {
            return nil, fmt.Errorf("user not found: %s", id)
        }
        return nil, fmt.Errorf("failed to query user: %w", err)
    }
    
    return &user, nil
}
```

---

## Frontend Standards (React)

### React Component Standards
- **Components:** Use PascalCase for component names
- **Props:** Use camelCase for props and state
- **Styling:** Tailwind CSS with utility-first approach
- **Components:** shadcn/ui for accessible, customizable components

```tsx
// Good - React component with proper TypeScript
interface UserProfileProps {
  userId: string;
  onUserUpdate: (user: User) => void;
}

export const UserProfile: React.FC<UserProfileProps> = ({
  userId,
  onUserUpdate
}) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchUser = async () => {
      try {
        setLoading(true);
        setError(null);
        const userData = await userService.getUser(userId);
        setUser(userData);
      } catch (err) {
        setError(err instanceof Error ? err.message : 'Unknown error');
      } finally {
        setLoading(false);
      }
    };

    fetchUser();
  }, [userId]);

  if (loading) return <div className="animate-pulse">Loading...</div>;
  if (error) return <div className="text-red-500">Error: {error}</div>;
  if (!user) return <div>User not found</div>;

  return (
    <div className="max-w-md mx-auto bg-white rounded-lg shadow-md p-6">
      <h2 className="text-xl font-semibold mb-4">{user.name}</h2>
      <p className="text-gray-600">{user.email}</p>
    </div>
  );
};
```

---

## Testing Standards

### Test Structure and Coverage
- **Coverage:** Maintain >90% test coverage for critical business logic
- **Test Types:** Unit, integration, and performance tests
- **Naming:** Use descriptive test names that explain the scenario

```go
// Good - comprehensive test structure
func TestUserService_CreateUser(t *testing.T) {
    tests := []struct {
        name        string
        email       string
        setupMock   func(*MockDB)
        wantErr     bool
        errContains string
    }{
        {
            name:  "successful user creation",
            email: "test@example.com",
            setupMock: func(mockDB *MockDB) {
                mockDB.EXPECT().SaveUser(gomock.Any()).Return(nil)
            },
            wantErr: false,
        },
        {
            name:  "invalid email format",
            email: "invalid-email",
            setupMock: func(mockDB *MockDB) {
                // No database call expected
            },
            wantErr:     true,
            errContains: "invalid email format",
        },
        {
            name:  "database error",
            email: "test@example.com",
            setupMock: func(mockDB *MockDB) {
                mockDB.EXPECT().SaveUser(gomock.Any()).
                    Return(fmt.Errorf("database connection failed"))
            },
            wantErr:     true,
            errContains: "failed to save user",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            ctrl := gomock.NewController(t)
            defer ctrl.Finish()

            mockDB := NewMockDB(ctrl)
            tt.setupMock(mockDB)

            service := &UserService{db: mockDB}
            
            user, err := service.CreateUser(context.Background(), tt.email)

            if tt.wantErr {
                assert.Error(t, err)
                if tt.errContains != "" {
                    assert.Contains(t, err.Error(), tt.errContains)
                }
                assert.Nil(t, user)
            } else {
                assert.NoError(t, err)
                assert.NotNil(t, user)
                assert.Equal(t, tt.email, user.Email)
            }
        })
    }
}
```

---

## Performance Standards

### Backend Performance
- **Query Response:** <200ms for GraphQL queries
- **Entity Hydration:** <500ms for multi-level relationships
- **Concurrency:** Optimize for multi-tenant request handling
- **Resource Usage:** Monitor memory and CPU usage

### Performance Monitoring
```go
// Good - performance monitoring middleware
func (s *Server) performanceMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        next.ServeHTTP(w, r)
        
        duration := time.Since(start)
        if duration > 200*time.Millisecond {
            s.logger.Warn("slow request detected",
                "path", r.URL.Path,
                "duration", duration,
                "method", r.Method,
            )
        }
        
        s.metrics.RequestDuration.Observe(duration.Seconds())
    })
}
```

---

## Development Workflow

### Version Control Standards
- **Git Workflow:** Commit directly to main branch
- **Commit Messages:** Use conventional format: `type(scope): description`
- **Atomic Commits:** Single logical changes per commit
- **Testing:** Run tests and linting before commits

```bash
# Good commit messages
feat(user): add user creation endpoint with validation
fix(auth): resolve JWT token expiration handling
refactor(db): optimize connection pooling for multi-tenant access
test(user): add comprehensive user service test coverage
docs(api): update GraphQL schema documentation

# Bad commit messages
fix stuff
update code
changes
wip
```

### Build and Quality Standards
- **Linting:** Run `go vet` and `golint` before commits
- **Formatting:** Use `gofmt` and `goimports`
- **Testing:** Execute full test suite before commits
- **Performance:** Validate response time requirements

---

## Code Quality Tools

### Required Tools
- **Go:** `go vet`, `golint`, `goimports`, `staticcheck`
- **Testing:** `go test`, `gotestsum`, `go-mockgen`
- **Performance:** `go tool pprof`, benchmarking tools
- **Frontend:** ESLint, Prettier, TypeScript compiler

### Pre-commit Checklist
```bash
# Run these commands before each commit
go fmt ./...
go vet ./...
golint ./...
go test ./...
staticcheck ./...
```

---

## Conclusion

These coding standards ensure the UDM System maintains high code quality, performance, and maintainability while following Go idioms and best practices. Regular adherence to these standards supports our objectives of rapid development cycles and enterprise-grade reliability.