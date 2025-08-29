# Go Technical Specification

This is the Go implementation technical specification for the spec detailed in @.agent-os/specs/2025-08-28-thing-class-entity/spec.md

> Created: 2025-08-28
> Updated: 2025-08-29
> Version: 2.0.0
> Language: Go + gqlgen + PostgreSQL

## Go Struct Definitions

### Core Thing Class Domain Model

```go
package domain

import (
	"time"
	"encoding/json"
	"github.com/google/uuid"
)

// ThingClass represents the core Thing Class entity
type ThingClass struct {
	ID              uuid.UUID              `json:"id" db:"id"`
	Name            string                 `json:"name" db:"name"`
	Description     *string                `json:"description" db:"description"`
	ValidationRules json.RawMessage        `json:"validationRules" db:"validation_rules"`
	Permissions     json.RawMessage        `json:"permissions" db:"permissions"`
	CreatedAt       time.Time              `json:"createdAt" db:"created_at"`
	UpdatedAt       time.Time              `json:"updatedAt" db:"updated_at"`
	CreatedBy       uuid.UUID              `json:"createdBy" db:"created_by"`
	IsActive        bool                   `json:"isActive" db:"is_active"`
}

// ValidationRule represents individual validation rules within JSON-B
type ValidationRule struct {
	Type        string                 `json:"type"`
	Required    bool                   `json:"required"`
	MinLength   *int                   `json:"minLength,omitempty"`
	MaxLength   *int                   `json:"maxLength,omitempty"`
	Pattern     *string                `json:"pattern,omitempty"`
	CustomRules map[string]interface{} `json:"customRules,omitempty"`
}

// TenantContext carries tenant-specific information through request lifecycle
type TenantContext struct {
	TenantID     uuid.UUID `json:"tenantId"`
	SchemaName   string    `json:"schemaName"`
	UserID       uuid.UUID `json:"userId"`
	Permissions  []string  `json:"permissions"`
}
```

### GraphQL Input/Output Types

```go
package graph

import (
	"github.com/google/uuid"
	"encoding/json"
	"time"
)

// CreateThingClassInput represents the input for creating a Thing Class
type CreateThingClassInput struct {
	Name            string          `json:"name"`
	Description     *string         `json:"description"`
	ValidationRules json.RawMessage `json:"validationRules"`
}

// UpdateThingClassInput represents the input for updating a Thing Class
type UpdateThingClassInput struct {
	Name            *string         `json:"name"`
	Description     *string         `json:"description"`
	ValidationRules json.RawMessage `json:"validationRules"`
}

// ThingClassFilter represents filtering options for Thing Class queries
type ThingClassFilter struct {
	Name          *string    `json:"name"`
	Description   *string    `json:"description"`
	CreatedAfter  *time.Time `json:"createdAfter"`
	CreatedBefore *time.Time `json:"createdBefore"`
}
```

## gqlgen Resolver Implementation

### Thing Class Resolvers

```go
package graph

import (
	"context"
	"fmt"
	"github.com/google/uuid"
)

// ThingClass resolver implements GraphQL Thing Class operations
func (r *mutationResolver) CreateThingClass(ctx context.Context, input CreateThingClassInput) (*ThingClassPayload, error) {
	// Extract tenant context from GraphQL context
	tenantCtx, err := r.extractTenantContext(ctx)
	if err != nil {
		return &ThingClassPayload{
			Errors: []*UserError{{
				Message: "Invalid tenant context",
				Code:    "UNAUTHORIZED",
			}},
		}, nil
	}

	// Validate permissions
	if !r.hasPermission(tenantCtx, "CREATE_THING_CLASS") {
		return &ThingClassPayload{
			Errors: []*UserError{{
				Message: "Insufficient permissions",
				Code:    "FORBIDDEN",
			}},
		}, nil
	}

	// Validate input
	if err := r.validateThingClassInput(input); err != nil {
		return &ThingClassPayload{
			Errors: []*UserError{{
				Message: err.Error(),
				Code:    "VALIDATION_ERROR",
			}},
		}, nil
	}

	// Create Thing Class using repository
	thingClass, err := r.thingClassRepo.Create(ctx, tenantCtx, &domain.ThingClass{
		ID:              uuid.New(),
		Name:            input.Name,
		Description:     input.Description,
		ValidationRules: input.ValidationRules,
		CreatedBy:       tenantCtx.UserID,
		IsActive:        true,
	})
	
	if err != nil {
		return &ThingClassPayload{
			Errors: []*UserError{{
				Message: "Failed to create Thing Class",
				Code:    "INTERNAL_ERROR",
			}},
		}, nil
	}

	return &ThingClassPayload{
		ThingClass: thingClass,
		Errors:     []*UserError{},
	}, nil
}

func (r *queryResolver) ThingClass(ctx context.Context, id string) (*domain.ThingClass, error) {
	tenantCtx, err := r.extractTenantContext(ctx)
	if err != nil {
		return nil, fmt.Errorf("invalid tenant context: %w", err)
	}

	thingClassID, err := uuid.Parse(id)
	if err != nil {
		return nil, fmt.Errorf("invalid thing class ID: %w", err)
	}

	return r.thingClassRepo.GetByID(ctx, tenantCtx, thingClassID)
}

func (r *queryResolver) ThingClasses(ctx context.Context, first *int, after *string, last *int, before *string, filter *ThingClassFilter) (*ThingClassConnection, error) {
	tenantCtx, err := r.extractTenantContext(ctx)
	if err != nil {
		return nil, fmt.Errorf("invalid tenant context: %w", err)
	}

	// Build pagination parameters
	params := &repository.PaginationParams{
		First:  first,
		After:  after,
		Last:   last,
		Before: before,
	}

	return r.thingClassRepo.List(ctx, tenantCtx, filter, params)
}
```

## Repository Pattern Implementation

### Thing Class Repository Interface

```go
package repository

import (
	"context"
	"github.com/google/uuid"
)

type ThingClassRepository interface {
	Create(ctx context.Context, tenant *domain.TenantContext, thingClass *domain.ThingClass) (*domain.ThingClass, error)
	GetByID(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.ThingClass, error)
	Update(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID, updates *domain.ThingClass) (*domain.ThingClass, error)
	Delete(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) error
	List(ctx context.Context, tenant *domain.TenantContext, filter *ThingClassFilter, params *PaginationParams) (*ThingClassConnection, error)
	ExistsByName(ctx context.Context, tenant *domain.TenantContext, name string) (bool, error)
}

type PaginationParams struct {
	First  *int
	After  *string
	Last   *int
	Before *string
}
```

### PostgreSQL Repository Implementation

```go
package repository

import (
	"context"
	"database/sql"
	"fmt"
	"github.com/jmoiron/sqlx"
	"github.com/lib/pq"
)

type postgresThingClassRepo struct {
	db *sqlx.DB
}

func NewPostgresThingClassRepository(db *sqlx.DB) ThingClassRepository {
	return &postgresThingClassRepo{db: db}
}

func (r *postgresThingClassRepo) Create(ctx context.Context, tenant *domain.TenantContext, thingClass *domain.ThingClass) (*domain.ThingClass, error) {
	// Set tenant-specific search path
	if err := r.setTenantSchema(ctx, tenant.SchemaName); err != nil {
		return nil, fmt.Errorf("failed to set tenant schema: %w", err)
	}

	query := `
		INSERT INTO thing_classes (id, name, description, validation_rules, permissions, created_by, is_active)
		VALUES ($1, $2, $3, $4, $5, $6, $7)
		RETURNING id, name, description, validation_rules, permissions, created_at, updated_at, created_by, is_active
	`

	var result domain.ThingClass
	err := r.db.QueryRowxContext(ctx, query,
		thingClass.ID,
		thingClass.Name,
		thingClass.Description,
		thingClass.ValidationRules,
		thingClass.Permissions,
		thingClass.CreatedBy,
		thingClass.IsActive,
	).StructScan(&result)

	if err != nil {
		if pqErr, ok := err.(*pq.Error); ok {
			if pqErr.Code == "23505" { // unique_violation
				return nil, fmt.Errorf("thing class name already exists: %w", err)
			}
		}
		return nil, fmt.Errorf("failed to create thing class: %w", err)
	}

	return &result, nil
}

func (r *postgresThingClassRepo) GetByID(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.ThingClass, error) {
	if err := r.setTenantSchema(ctx, tenant.SchemaName); err != nil {
		return nil, fmt.Errorf("failed to set tenant schema: %w", err)
	}

	query := `
		SELECT id, name, description, validation_rules, permissions, created_at, updated_at, created_by, is_active
		FROM thing_classes 
		WHERE id = $1 AND is_active = true
	`

	var result domain.ThingClass
	err := r.db.GetContext(ctx, &result, query, id)
	if err != nil {
		if err == sql.ErrNoRows {
			return nil, nil
		}
		return nil, fmt.Errorf("failed to get thing class: %w", err)
	}

	return &result, nil
}

func (r *postgresThingClassRepo) setTenantSchema(ctx context.Context, schemaName string) error {
	query := fmt.Sprintf("SET search_path TO %s, public", pq.QuoteIdentifier(schemaName))
	_, err := r.db.ExecContext(ctx, query)
	return err
}
```

## Tenant Isolation Middleware

### Tenant Context Middleware

```go
package middleware

import (
	"context"
	"fmt"
	"net/http"
	"strings"
	
	"github.com/golang-jwt/jwt/v4"
	"github.com/google/uuid"
)

type TenantMiddleware struct {
	jwtSecret []byte
}

func NewTenantMiddleware(jwtSecret string) *TenantMiddleware {
	return &TenantMiddleware{
		jwtSecret: []byte(jwtSecret),
	}
}

func (m *TenantMiddleware) Handler(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Extract JWT token from Authorization header
		authHeader := r.Header.Get("Authorization")
		if authHeader == "" {
			http.Error(w, "Authorization header required", http.StatusUnauthorized)
			return
		}

		tokenString := strings.TrimPrefix(authHeader, "Bearer ")
		if tokenString == authHeader {
			http.Error(w, "Bearer token required", http.StatusUnauthorized)
			return
		}

		// Parse and validate JWT token
		token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
			if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
				return nil, fmt.Errorf("unexpected signing method: %v", token.Header["alg"])
			}
			return m.jwtSecret, nil
		})

		if err != nil || !token.Valid {
			http.Error(w, "Invalid token", http.StatusUnauthorized)
			return
		}

		// Extract tenant context from JWT claims
		claims, ok := token.Claims.(jwt.MapClaims)
		if !ok {
			http.Error(w, "Invalid token claims", http.StatusUnauthorized)
			return
		}

		tenantCtx, err := m.extractTenantContext(claims)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}

		// Add tenant context to request context
		ctx := context.WithValue(r.Context(), "tenant", tenantCtx)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}

func (m *TenantMiddleware) extractTenantContext(claims jwt.MapClaims) (*domain.TenantContext, error) {
	tenantID, err := uuid.Parse(claims["tenant_id"].(string))
	if err != nil {
		return nil, fmt.Errorf("invalid tenant ID: %w", err)
	}

	userID, err := uuid.Parse(claims["user_id"].(string))
	if err != nil {
		return nil, fmt.Errorf("invalid user ID: %w", err)
	}

	schemaName := claims["schema_name"].(string)
	if schemaName == "" {
		return nil, fmt.Errorf("schema name required")
	}

	permissions, ok := claims["permissions"].([]interface{})
	if !ok {
		permissions = []interface{}{}
	}

	permStrings := make([]string, len(permissions))
	for i, perm := range permissions {
		permStrings[i] = perm.(string)
	}

	return &domain.TenantContext{
		TenantID:    tenantID,
		SchemaName:  schemaName,
		UserID:      userID,
		Permissions: permStrings,
	}, nil
}
```

## Performance Optimization Patterns

### Connection Pool Management

```go
package database

import (
	"context"
	"fmt"
	"sync"
	"time"
	
	"github.com/jmoiron/sqlx"
	_ "github.com/lib/pq"
)

type TenantConnectionManager struct {
	pools    map[string]*sqlx.DB
	mu       sync.RWMutex
	config   *ConnectionConfig
}

type ConnectionConfig struct {
	MaxOpenConns    int
	MaxIdleConns    int
	ConnMaxLifetime time.Duration
	ConnMaxIdleTime time.Duration
}

func NewTenantConnectionManager(config *ConnectionConfig) *TenantConnectionManager {
	return &TenantConnectionManager{
		pools:  make(map[string]*sqlx.DB),
		config: config,
	}
}

func (m *TenantConnectionManager) GetConnection(tenantSchema string) (*sqlx.DB, error) {
	m.mu.RLock()
	if pool, exists := m.pools[tenantSchema]; exists {
		m.mu.RUnlock()
		return pool, nil
	}
	m.mu.RUnlock()

	m.mu.Lock()
	defer m.mu.Unlock()

	// Double-check pattern
	if pool, exists := m.pools[tenantSchema]; exists {
		return pool, nil
	}

	// Create new connection pool for tenant
	dsn := fmt.Sprintf("postgres://user:password@localhost/database?search_path=%s", tenantSchema)
	db, err := sqlx.Connect("postgres", dsn)
	if err != nil {
		return nil, fmt.Errorf("failed to create connection pool for tenant %s: %w", tenantSchema, err)
	}

	// Configure connection pool
	db.SetMaxOpenConns(m.config.MaxOpenConns)
	db.SetMaxIdleConns(m.config.MaxIdleConns)
	db.SetConnMaxLifetime(m.config.ConnMaxLifetime)
	db.SetConnMaxIdleTime(m.config.ConnMaxIdleTime)

	m.pools[tenantSchema] = db
	return db, nil
}

func (m *TenantConnectionManager) CloseAll() error {
	m.mu.Lock()
	defer m.mu.Unlock()

	var errs []error
	for schema, pool := range m.pools {
		if err := pool.Close(); err != nil {
			errs = append(errs, fmt.Errorf("failed to close pool for %s: %w", schema, err))
		}
	}

	if len(errs) > 0 {
		return fmt.Errorf("errors closing connection pools: %v", errs)
	}
	return nil
}
```

### Concurrent Processing with Goroutines

```go
package service

import (
	"context"
	"fmt"
	"sync"
)

type ThingClassService struct {
	repo       repository.ThingClassRepository
	validator  *ValidationService
	cacheService *CacheService
}

// BulkCreateThingClasses demonstrates concurrent processing pattern
func (s *ThingClassService) BulkCreateThingClasses(ctx context.Context, tenant *domain.TenantContext, inputs []CreateThingClassInput) ([]*domain.ThingClass, []error) {
	type result struct {
		thingClass *domain.ThingClass
		err        error
		index      int
	}

	results := make(chan result, len(inputs))
	var wg sync.WaitGroup

	// Process Thing Classes concurrently
	for i, input := range inputs {
		wg.Add(1)
		go func(idx int, inp CreateThingClassInput) {
			defer wg.Done()
			
			// Validate input
			if err := s.validator.ValidateCreateInput(inp); err != nil {
				results <- result{err: err, index: idx}
				return
			}

			// Create Thing Class
			thingClass := &domain.ThingClass{
				ID:              uuid.New(),
				Name:            inp.Name,
				Description:     inp.Description,
				ValidationRules: inp.ValidationRules,
				CreatedBy:       tenant.UserID,
				IsActive:        true,
			}

			created, err := s.repo.Create(ctx, tenant, thingClass)
			results <- result{thingClass: created, err: err, index: idx}
		}(i, input)
	}

	// Close results channel when all goroutines complete
	go func() {
		wg.Wait()
		close(results)
	}()

	// Collect results in order
	thingClasses := make([]*domain.ThingClass, len(inputs))
	errors := make([]error, len(inputs))

	for res := range results {
		thingClasses[res.index] = res.thingClass
		errors[res.index] = res.err
	}

	return thingClasses, errors
}
```

## Error Handling Patterns

### Go-idiomatic Error Handling

```go
package errors

import (
	"fmt"
	"errors"
)

var (
	ErrThingClassNotFound     = errors.New("thing class not found")
	ErrThingClassExists       = errors.New("thing class already exists")
	ErrInvalidValidationRules = errors.New("invalid validation rules")
	ErrUnauthorized          = errors.New("unauthorized")
	ErrForbidden             = errors.New("forbidden")
)

type ValidationError struct {
	Field   string
	Message string
	Code    string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("validation error on field %s: %s", e.Field, e.Message)
}

type BusinessError struct {
	Code    string
	Message string
	Cause   error
}

func (e *BusinessError) Error() string {
	if e.Cause != nil {
		return fmt.Sprintf("%s: %v", e.Message, e.Cause)
	}
	return e.Message
}

func (e *BusinessError) Unwrap() error {
	return e.Cause
}

// Error wrapping helpers
func WrapValidationError(field, message, code string) *ValidationError {
	return &ValidationError{
		Field:   field,
		Message: message,
		Code:    code,
	}
}

func WrapBusinessError(code, message string, cause error) *BusinessError {
	return &BusinessError{
		Code:    code,
		Message: message,
		Cause:   cause,
	}
}
```

## Docker Infrastructure Configuration

### PostgreSQL and Redis Hosting

The Thing Class implementation requires Docker hosting for PostgreSQL and Redis services. This provides consistent development and production environments with proper multi-tenant schema isolation.

#### docker-compose.yml for Development

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: udm-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: udm_development
      POSTGRES_USER: udm_user
      POSTGRES_PASSWORD: udm_password
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./docker/postgres/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U udm_user -d udm_development"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - udm-network

  redis:
    image: redis:7-alpine
    container_name: udm-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass udm_redis_password
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./docker/redis/redis.conf:/usr/local/etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - udm-network

  udm-backend:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: development
    container_name: udm-backend
    restart: unless-stopped
    environment:
      # Database configuration
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: udm_development
      DB_USER: udm_user
      DB_PASSWORD: udm_password
      DB_SSL_MODE: disable
      
      # Redis configuration
      REDIS_ADDR: redis:6379
      REDIS_PASSWORD: udm_redis_password
      REDIS_DB: 0
      
      # Application configuration
      ENVIRONMENT: development
      LOG_LEVEL: debug
      JWT_SECRET: development_secret_key_change_in_production
      
      # Server configuration
      SERVER_PORT: 8080
      SERVER_READ_TIMEOUT: 30s
      SERVER_WRITE_TIMEOUT: 30s
      
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - .:/app
      - go_modules:/go/pkg/mod
    working_dir: /app
    networks:
      - udm-network

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local  
  go_modules:
    driver: local

networks:
  udm-network:
    driver: bridge
```

#### Multi-Stage Dockerfile for Go Application

```dockerfile
# docker/Dockerfile
FROM golang:1.21-alpine AS base

# Install build dependencies
RUN apk add --no-cache git ca-certificates tzdata postgresql-client redis

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Development stage
FROM base AS development

# Install development tools
RUN go install github.com/99designs/gqlgen@latest
RUN go install github.com/air-verse/air@latest
RUN go install github.com/golang-migrate/migrate/v4/cmd/migrate@latest

# Copy source code
COPY . .

# Generate GraphQL code
RUN go generate ./...

# Use air for hot reloading in development
CMD ["air", "-c", ".air.toml"]

# Build stage
FROM base AS builder

# Copy source code
COPY . .

# Generate GraphQL code
RUN go generate ./...

# Build binary with optimizations
RUN CGO_ENABLED=0 GOOS=linux go build \
    -ldflags='-w -s -extldflags "-static"' \
    -a -installsuffix cgo \
    -o main cmd/server/main.go

# Production stage
FROM alpine:latest AS production

# Install runtime dependencies
RUN apk --no-cache add ca-certificates tzdata postgresql-client redis

WORKDIR /root/

# Create non-root user
RUN addgroup -g 1001 -S udm && \
    adduser -S udm -u 1001

# Copy binary from builder stage
COPY --from=builder /app/main .

# Copy migration files
COPY --from=builder /app/migrations ./migrations

# Set ownership
RUN chown -R udm:udm /root

# Switch to non-root user
USER udm

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health || exit 1

# Run the binary
CMD ["./main"]
```

### Go Database Configuration for Docker

#### Docker-aware Connection Configuration

```go
// internal/config/docker.go
package config

import (
	"fmt"
	"os"
	"strconv"
	"time"
)

type DockerConfig struct {
	Database DatabaseConfig `json:"database"`
	Redis    RedisConfig    `json:"redis"`
	Server   ServerConfig   `json:"server"`
}

type DatabaseConfig struct {
	Host            string        `env:"DB_HOST" default:"localhost"`
	Port            int           `env:"DB_PORT" default:"5432"`
	Name            string        `env:"DB_NAME" default:"udm_development"`
	User            string        `env:"DB_USER" default:"udm_user"`
	Password        string        `env:"DB_PASSWORD" default:""`
	SSLMode         string        `env:"DB_SSL_MODE" default:"require"`
	MaxOpenConns    int           `env:"DB_MAX_OPEN_CONNS" default:"25"`
	MaxIdleConns    int           `env:"DB_MAX_IDLE_CONNS" default:"5"`
	ConnMaxLifetime time.Duration `env:"DB_CONN_MAX_LIFETIME" default:"1h"`
	ConnMaxIdleTime time.Duration `env:"DB_CONN_MAX_IDLE_TIME" default:"10m"`
	
	// Docker-specific settings
	RetryAttempts   int           `env:"DB_RETRY_ATTEMPTS" default:"10"`
	RetryDelay      time.Duration `env:"DB_RETRY_DELAY" default:"5s"`
}

type RedisConfig struct {
	Addr         string `env:"REDIS_ADDR" default:"localhost:6379"`
	Password     string `env:"REDIS_PASSWORD" default:""`
	DB           int    `env:"REDIS_DB" default:"0"`
	PoolSize     int    `env:"REDIS_POOL_SIZE" default:"10"`
	MinIdleConns int    `env:"REDIS_MIN_IDLE_CONNS" default:"2"`
	
	// Docker-specific settings  
	DialTimeout  time.Duration `env:"REDIS_DIAL_TIMEOUT" default:"5s"`
	ReadTimeout  time.Duration `env:"REDIS_READ_TIMEOUT" default:"3s"`
	WriteTimeout time.Duration `env:"REDIS_WRITE_TIMEOUT" default:"3s"`
}

func LoadDockerConfig() (*DockerConfig, error) {
	config := &DockerConfig{}
	
	// Database configuration
	config.Database = DatabaseConfig{
		Host:     getEnvString("DB_HOST", "localhost"),
		Port:     getEnvInt("DB_PORT", 5432),
		Name:     getEnvString("DB_NAME", "udm_development"),
		User:     getEnvString("DB_USER", "udm_user"),
		Password: getEnvString("DB_PASSWORD", ""),
		SSLMode:  getEnvString("DB_SSL_MODE", "require"),
	}
	
	// Redis configuration
	config.Redis = RedisConfig{
		Addr:     getEnvString("REDIS_ADDR", "localhost:6379"),
		Password: getEnvString("REDIS_PASSWORD", ""),
		DB:       getEnvInt("REDIS_DB", 0),
	}
	
	return config, nil
}

func getEnvString(key, defaultValue string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return defaultValue
}

func getEnvInt(key string, defaultValue int) int {
	if value := os.Getenv(key); value != "" {
		if intValue, err := strconv.Atoi(value); err == nil {
			return intValue
		}
	}
	return defaultValue
}

// BuildPostgresDSN constructs PostgreSQL connection string for Docker
func (c *DatabaseConfig) BuildPostgresDSN(tenantSchema string) string {
	dsn := fmt.Sprintf(
		"host=%s port=%d dbname=%s user=%s sslmode=%s",
		c.Host, c.Port, c.Name, c.User, c.SSLMode,
	)
	
	if c.Password != "" {
		dsn += fmt.Sprintf(" password=%s", c.Password)
	}
	
	if tenantSchema != "" {
		dsn += fmt.Sprintf(" search_path=%s,public", tenantSchema)
	}
	
	return dsn
}
```

### Docker Health Check Integration

```go
// internal/health/docker.go
package health

import (
	"context"
	"database/sql"
	"fmt"
	"net/http"
	"time"

	"github.com/redis/go-redis/v9"
	"go.uber.org/zap"
)

type DockerHealthChecker struct {
	db     *sql.DB
	redis  *redis.Client
	logger *zap.Logger
}

func NewDockerHealthChecker(db *sql.DB, redis *redis.Client, logger *zap.Logger) *DockerHealthChecker {
	return &DockerHealthChecker{
		db:     db,
		redis:  redis,
		logger: logger,
	}
}

func (h *DockerHealthChecker) HealthHandler(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
	defer cancel()

	status := h.checkHealth(ctx)
	
	if status.Healthy {
		w.WriteHeader(http.StatusOK)
	} else {
		w.WriteHeader(http.StatusServiceUnavailable)
	}
	
	w.Header().Set("Content-Type", "application/json")
	fmt.Fprintf(w, `{
		"healthy": %t,
		"timestamp": "%s",
		"services": {
			"database": {
				"healthy": %t,
				"error": "%s"
			},
			"redis": {
				"healthy": %t,
				"error": "%s"
			}
		}
	}`, status.Healthy, time.Now().Format(time.RFC3339),
		status.Database.Healthy, status.Database.Error,
		status.Redis.Healthy, status.Redis.Error)
}

type HealthStatus struct {
	Healthy  bool
	Database ServiceStatus
	Redis    ServiceStatus
}

type ServiceStatus struct {
	Healthy bool
	Error   string
}

func (h *DockerHealthChecker) checkHealth(ctx context.Context) HealthStatus {
	var status HealthStatus
	status.Healthy = true

	// Check PostgreSQL
	if err := h.db.PingContext(ctx); err != nil {
		status.Database.Healthy = false
		status.Database.Error = err.Error()
		status.Healthy = false
		h.logger.Error("PostgreSQL health check failed", zap.Error(err))
	} else {
		status.Database.Healthy = true
	}

	// Check Redis
	if err := h.redis.Ping(ctx).Err(); err != nil {
		status.Redis.Healthy = false
		status.Redis.Error = err.Error()
		status.Healthy = false
		h.logger.Error("Redis health check failed", zap.Error(err))
	} else {
		status.Redis.Healthy = true
	}

	return status
}
```

### Docker Development Workflow

#### Air Configuration for Hot Reloading

```toml
# .air.toml
root = "."
testdata_dir = "testdata"
tmp_dir = "tmp"

[build]
  args_bin = []
  bin = "./tmp/main"
  cmd = "go build -o ./tmp/main cmd/server/main.go"
  delay = 1000
  exclude_dir = ["assets", "tmp", "vendor", "testdata", "docker"]
  exclude_file = []
  exclude_regex = ["_test.go"]
  exclude_unchanged = false
  follow_symlink = false
  full_bin = ""
  include_dir = []
  include_ext = ["go", "tpl", "tmpl", "html", "graphql"]
  kill_delay = "0s"
  log = "build-errors.log"
  send_interrupt = false
  stop_on_root = false

[color]
  app = ""
  build = "yellow"
  main = "magenta"
  runner = "green"
  watcher = "cyan"

[log]
  time = false

[misc]
  clean_on_exit = false
```

#### Docker Scripts

```bash
#!/bin/bash
# scripts/docker-dev.sh

set -e

echo "Starting UDM development environment with Docker..."

# Create necessary directories
mkdir -p docker/postgres/init
mkdir -p docker/redis

# Create PostgreSQL init script
cat > docker/postgres/init/01-init.sql << EOF
-- Create additional schemas for testing
CREATE SCHEMA IF NOT EXISTS tenant_test_1;
CREATE SCHEMA IF NOT EXISTS tenant_test_2;

-- Grant permissions
GRANT ALL PRIVILEGES ON SCHEMA tenant_test_1 TO udm_user;
GRANT ALL PRIVILEGES ON SCHEMA tenant_test_2 TO udm_user;
EOF

# Create Redis configuration
cat > docker/redis/redis.conf << EOF
# Redis configuration for UDM development
maxmemory 256mb
maxmemory-policy allkeys-lru
save 900 1
save 300 10
save 60 10000
EOF

# Build and start services
docker-compose up --build -d postgres redis

# Wait for services to be healthy
echo "Waiting for PostgreSQL to be ready..."
docker-compose exec postgres pg_isready -U udm_user -d udm_development

echo "Waiting for Redis to be ready..."
docker-compose exec redis redis-cli ping

# Run migrations
echo "Running database migrations..."
docker-compose exec udm-backend migrate -path migrations -database "postgres://udm_user:udm_password@postgres:5432/udm_development?sslmode=disable" up

# Start the application
echo "Starting UDM backend..."
docker-compose up udm-backend

echo "Development environment ready!"
echo "GraphQL Playground: http://localhost:8080/playground"
echo "Health Check: http://localhost:8080/health"
```

```bash
#!/bin/bash
# scripts/docker-clean.sh

set -e

echo "Cleaning up Docker environment..."

# Stop and remove containers
docker-compose down -v

# Remove images
docker-compose down --rmi all

# Clean up volumes
docker volume prune -f

# Clean up networks  
docker network prune -f

echo "Docker environment cleaned!"
```

## Go Module Dependencies

```go
module github.com/company/udm-backend

go 1.21

require (
	github.com/99designs/gqlgen v0.17.40
	github.com/golang-jwt/jwt/v4 v4.5.0
	github.com/google/uuid v1.4.0
	github.com/jmoiron/sqlx v1.3.5
	github.com/lib/pq v1.10.9
	github.com/redis/go-redis/v9 v9.3.0
	github.com/go-playground/validator/v10 v10.16.0
	github.com/golang-migrate/migrate/v4 v4.16.2
	github.com/gorilla/mux v1.8.1
	github.com/rs/cors v1.10.1
	go.uber.org/zap v1.26.0
	github.com/stretchr/testify v1.8.4
)

require (
	github.com/cespare/xxhash/v2 v2.2.0 // indirect
	github.com/dgryski/go-rendezvous v0.0.0-20200823014737-9f7001d12a5f // indirect
	github.com/hashicorp/errwrap v1.1.0 // indirect
	github.com/hashicorp/go-multierror v1.1.1 // indirect
	go.uber.org/atomic v1.7.0 // indirect
	go.uber.org/multierr v1.6.0 // indirect
)
```

### Key Package Usage

- **gqlgen**: GraphQL server generation and resolvers
- **sqlx**: Enhanced SQL database interactions with struct scanning
- **lib/pq**: PostgreSQL driver with JSON-B support
- **go-redis**: Redis client for caching and session management
- **validator**: Input validation with struct tags
- **migrate**: Database migration management
- **zap**: Structured logging with performance focus
- **uuid**: UUID generation and parsing
- **jwt**: JWT token handling for authentication

## Testing Strategy

### Unit Tests for Resolvers

```go
package graph_test

import (
	"context"
	"testing"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

type MockThingClassRepo struct {
	mock.Mock
}

func (m *MockThingClassRepo) Create(ctx context.Context, tenant *domain.TenantContext, thingClass *domain.ThingClass) (*domain.ThingClass, error) {
	args := m.Called(ctx, tenant, thingClass)
	return args.Get(0).(*domain.ThingClass), args.Error(1)
}

func TestCreateThingClass(t *testing.T) {
	// Setup
	mockRepo := new(MockThingClassRepo)
	resolver := &mutationResolver{
		thingClassRepo: mockRepo,
	}

	tenantCtx := &domain.TenantContext{
		TenantID:   uuid.New(),
		SchemaName: "tenant_test",
		UserID:     uuid.New(),
		Permissions: []string{"CREATE_THING_CLASS"},
	}

	input := CreateThingClassInput{
		Name:        "TestClass",
		Description: stringPtr("Test Description"),
	}

	expectedThingClass := &domain.ThingClass{
		ID:          uuid.New(),
		Name:        input.Name,
		Description: input.Description,
		IsActive:    true,
	}

	// Mock expectations
	mockRepo.On("Create", mock.Anything, tenantCtx, mock.AnythingOfType("*domain.ThingClass")).Return(expectedThingClass, nil)

	// Execute
	ctx := context.WithValue(context.Background(), "tenant", tenantCtx)
	result, err := resolver.CreateThingClass(ctx, input)

	// Assert
	assert.NoError(t, err)
	assert.NotNil(t, result)
	assert.Equal(t, expectedThingClass, result.ThingClass)
	assert.Empty(t, result.Errors)
	mockRepo.AssertExpectations(t)
}

func stringPtr(s string) *string {
	return &s
}
```

### Integration Tests

```go
package integration_test

import (
	"context"
	"testing"
	"github.com/stretchr/testify/suite"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

type ThingClassIntegrationSuite struct {
	suite.Suite
	db        *sqlx.DB
	container testcontainers.Container
	repo      repository.ThingClassRepository
}

func (suite *ThingClassIntegrationSuite) SetupSuite() {
	ctx := context.Background()
	
	// Start PostgreSQL container
	req := testcontainers.ContainerRequest{
		Image:        "postgres:15",
		ExposedPorts: []string{"5432/tcp"},
		Env: map[string]string{
			"POSTGRES_DB":       "test",
			"POSTGRES_PASSWORD": "password",
			"POSTGRES_USER":     "postgres",
		},
		WaitingFor: wait.ForLog("database system is ready to accept connections"),
	}
	
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	suite.Require().NoError(err)
	suite.container = container
	
	// Get connection details and setup database
	host, err := container.Host(ctx)
	suite.Require().NoError(err)
	port, err := container.MappedPort(ctx, "5432")
	suite.Require().NoError(err)
	
	dsn := fmt.Sprintf("postgres://postgres:password@%s:%s/test?sslmode=disable", host, port.Port())
	suite.db, err = sqlx.Connect("postgres", dsn)
	suite.Require().NoError(err)
	
	// Run migrations
	suite.runMigrations()
	
	// Setup repository
	suite.repo = repository.NewPostgresThingClassRepository(suite.db)
}

func (suite *ThingClassIntegrationSuite) TestCreateAndRetrieveThingClass() {
	tenantCtx := &domain.TenantContext{
		TenantID:   uuid.New(),
		SchemaName: "public", // Use public schema for test
		UserID:     uuid.New(),
	}
	
	thingClass := &domain.ThingClass{
		ID:          uuid.New(),
		Name:        "IntegrationTestClass",
		Description: stringPtr("Integration test description"),
		CreatedBy:   tenantCtx.UserID,
		IsActive:    true,
	}
	
	// Create
	created, err := suite.repo.Create(context.Background(), tenantCtx, thingClass)
	suite.Require().NoError(err)
	suite.Equal(thingClass.Name, created.Name)
	
	// Retrieve
	retrieved, err := suite.repo.GetByID(context.Background(), tenantCtx, created.ID)
	suite.Require().NoError(err)
	suite.Equal(created.ID, retrieved.ID)
	suite.Equal(created.Name, retrieved.Name)
}

func TestThingClassIntegrationSuite(t *testing.T) {
	suite.Run(t, new(ThingClassIntegrationSuite))
}
```