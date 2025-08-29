# Go Implementation Details

This document provides comprehensive Go implementation patterns and project structure for the Thing Class entity specification.

> Created: 2025-08-29
> Version: 1.0.0
> Language: Go + gqlgen + PostgreSQL

## Domain Model Implementation

### Base Entity Pattern

```go
// internal/domain/base.go
package domain

import (
	"time"
	"github.com/google/uuid"
)

// BaseEntity contains common fields for all entities
type BaseEntity struct {
	ID        uuid.UUID  `json:"id" db:"id"`
	CreatedAt time.Time  `json:"createdAt" db:"created_at"`
	UpdatedAt time.Time  `json:"updatedAt" db:"updated_at"`
	CreatedBy uuid.UUID  `json:"createdBy" db:"created_by"`
	IsActive  bool       `json:"isActive" db:"is_active"`
	Version   int        `json:"version" db:"version"`
}
```

### Thing Class Entity Domain Model

```go
// internal/domain/thingclass.go
package domain

import (
	"encoding/json"
	"github.com/google/uuid"
)

type ThingClass struct {
	BaseEntity
	Name        string  `json:"name" db:"name"`
	Description *string `json:"description" db:"description"`
	
	// Loaded relationships (not persisted directly)
	Attributes []*ThingClassAttribute `json:"attributes,omitempty" db:"-"`
}

// ThingClassAttribute represents the assignment of an attribute to a thing class
type ThingClassAttribute struct {
	BaseEntity
	ThingClassID    uuid.UUID       `json:"thingClassId" db:"thing_class_id"`
	AttributeID     uuid.UUID       `json:"attributeId" db:"attribute_id"`
	IsRequired      bool            `json:"isRequired" db:"is_required"`
	ValidationRules json.RawMessage `json:"validationRules" db:"validation_rules"`
	SortOrder       int             `json:"sortOrder" db:"sort_order"`
	
	// Loaded relationships
	ThingClass *ThingClass `json:"thingClass,omitempty" db:"-"`
	Attribute  *Attribute  `json:"attribute,omitempty" db:"-"`
}

// Attribute represents a reusable attribute definition
type Attribute struct {
	BaseEntity
	Name            string          `json:"name" db:"name"`
	DataType        AttributeType   `json:"dataType" db:"data_type"`
	ValidationRules json.RawMessage `json:"validationRules" db:"validation_rules"`
	Description     *string         `json:"description" db:"description"`
}

type AttributeType string

const (
	AttributeTypeString    AttributeType = "string"
	AttributeTypeInteger   AttributeType = "integer"
	AttributeTypeDecimal   AttributeType = "decimal"
	AttributeTypeBoolean   AttributeType = "boolean"
	AttributeTypeDate      AttributeType = "date"
	AttributeTypeDateTime  AttributeType = "datetime"
	AttributeTypeJSON      AttributeType = "json"
	AttributeTypeReference AttributeType = "reference"
)

// Input types for GraphQL mutations
type CreateThingClassInput struct {
	Name        string  `json:"name" validate:"required,min=1,max=100"`
	Description *string `json:"description,omitempty" validate:"omitempty,max=500"`
}

type UpdateThingClassInput struct {
	Name        *string `json:"name,omitempty" validate:"omitempty,min=1,max=100"`
	Description *string `json:"description,omitempty" validate:"omitempty,max=500"`
}

type CreateAttributeInput struct {
	Name            string          `json:"name" validate:"required,min=1,max=100"`
	DataType        AttributeType   `json:"dataType" validate:"required"`
	ValidationRules json.RawMessage `json:"validationRules,omitempty"`
	Description     *string         `json:"description,omitempty" validate:"omitempty,max=500"`
}

type AddAttributeToThingClassInput struct {
	ThingClassID    uuid.UUID       `json:"thingClassId" validate:"required"`
	AttributeID     uuid.UUID       `json:"attributeId" validate:"required"`
	IsRequired      bool            `json:"isRequired"`
	ValidationRules json.RawMessage `json:"validationRules,omitempty"`
	SortOrder       int             `json:"sortOrder"`
}

// Connection and pagination types
type ThingClassConnection struct {
	Edges    []*ThingClassEdge `json:"edges"`
	PageInfo *PageInfo         `json:"pageInfo"`
}

type ThingClassEdge struct {
	Node   *ThingClass `json:"node"`
	Cursor string      `json:"cursor"`
}

type AttributeConnection struct {
	Edges    []*AttributeEdge `json:"edges"`
	PageInfo *PageInfo        `json:"pageInfo"`
}

type AttributeEdge struct {
	Node   *Attribute `json:"node"`
	Cursor string     `json:"cursor"`
}

type PageInfo struct {
	HasNextPage     bool    `json:"hasNextPage"`
	HasPreviousPage bool    `json:"hasPreviousPage"`
	StartCursor     *string `json:"startCursor"`
	EndCursor       *string `json:"endCursor"`
}
```

## Project Structure

### Go Module Layout

```
github.com/company/udm-backend/
├── cmd/
│   └── server/
│       └── main.go                 # Application entry point
├── internal/
│   ├── domain/                     # Domain models and interfaces
│   │   ├── thingclass.go
│   │   ├── tenant.go
│   │   └── errors.go
│   ├── graph/                      # GraphQL layer (gqlgen generated + custom)
│   │   ├── generated/              # gqlgen generated code
│   │   ├── resolvers/              # Custom resolver implementations
│   │   │   └── thingclass.go
│   │   ├── schema.graphqls         # GraphQL schema definitions
│   │   └── gqlgen.yml              # gqlgen configuration
│   ├── repository/                 # Data access layer
│   │   ├── interfaces/
│   │   │   └── thingclass.go
│   │   └── postgres/
│   │       └── thingclass.go
│   ├── service/                    # Business logic layer
│   │   └── thingclass.go
│   ├── middleware/                 # HTTP middleware
│   │   ├── tenant.go
│   │   ├── auth.go
│   │   └── logging.go
│   ├── database/                   # Database connection management
│   │   ├── connection.go
│   │   └── migration.go
│   └── config/                     # Configuration management
│       └── config.go
├── migrations/                     # Database migrations
│   ├── 001_create_thing_classes.up.sql
│   └── 001_create_thing_classes.down.sql
├── scripts/                        # Build and deployment scripts
│   ├── generate.sh
│   └── migrate.sh
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── go.mod
├── go.sum
├── gqlgen.yml                      # gqlgen configuration
└── README.md
```

## gqlgen Configuration and Code Generation

### gqlgen.yml Configuration

```yaml
schema:
  - internal/graph/schema.graphqls

exec:
  filename: internal/graph/generated/generated.go
  package: generated

resolver:
  filename: internal/graph/resolvers/resolver.go
  package: resolvers
  type: Resolver

models:
  ID:
    model:
      - github.com/google/uuid.UUID
  ThingClass:
    model:
      - github.com/company/udm-backend/internal/domain.ThingClass
  Attribute:
    model:
      - github.com/company/udm-backend/internal/domain.Attribute
  ThingClassAttribute:
    model:
      - github.com/company/udm-backend/internal/domain.ThingClassAttribute
  AttributeType:
    model:
      - github.com/company/udm-backend/internal/domain.AttributeType
  CreateThingClassInput:
    model:
      - github.com/company/udm-backend/internal/domain.CreateThingClassInput
  UpdateThingClassInput:
    model:
      - github.com/company/udm-backend/internal/domain.UpdateThingClassInput
  CreateAttributeInput:
    model:
      - github.com/company/udm-backend/internal/domain.CreateAttributeInput
  AddAttributeToThingClassInput:
    model:
      - github.com/company/udm-backend/internal/domain.AddAttributeToThingClassInput
  ThingClassConnection:
    model:
      - github.com/company/udm-backend/internal/domain.ThingClassConnection
  ThingClassEdge:
    model:
      - github.com/company/udm-backend/internal/domain.ThingClassEdge
  AttributeConnection:
    model:
      - github.com/company/udm-backend/internal/domain.AttributeConnection
  AttributeEdge:
    model:
      - github.com/company/udm-backend/internal/domain.AttributeEdge
  TenantContext:
    model:
      - github.com/company/udm-backend/internal/domain.TenantContext
  DateTime:
    model:
      - time.Time
  JSON:
    model:
      - encoding/json.RawMessage

autobind:
  - github.com/company/udm-backend/internal/domain

directives:
  auth:
    skip_runtime: false
  validate:
    skip_runtime: false
```

### GraphQL Schema with gqlgen Directives

```graphql
# internal/graph/schema.graphqls

directive @auth(requires: [String!]!) on FIELD_DEFINITION
directive @validate(constraint: String!) on INPUT_FIELD_DEFINITION

scalar DateTime
scalar JSON

enum AttributeType {
  STRING
  INTEGER
  DECIMAL
  BOOLEAN
  DATE
  DATETIME
  JSON
  REFERENCE
}

type ThingClass {
  id: ID!
  name: String!
  description: String
  attributes: [ThingClassAttribute!]!
  # Common entity fields
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  isActive: Boolean!
  version: Int!
}

type ThingClassAttribute {
  id: ID!
  thingClassId: ID!
  attributeId: ID!
  thingClass: ThingClass!
  attribute: Attribute!
  isRequired: Boolean!
  validationRules: JSON
  sortOrder: Int!
  # Common entity fields
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  isActive: Boolean!
  version: Int!
}

type Attribute {
  id: ID!
  name: String!
  dataType: AttributeType!
  validationRules: JSON
  description: String
  # Common entity fields
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  isActive: Boolean!
  version: Int!
}

input CreateThingClassInput {
  name: String! @validate(constraint: "required,min=1,max=100")
  description: String @validate(constraint: "max=500")
}

input UpdateThingClassInput {
  name: String @validate(constraint: "min=1,max=100")
  description: String @validate(constraint: "max=500")
}

input CreateAttributeInput {
  name: String! @validate(constraint: "required,min=1,max=100")
  dataType: AttributeType!
  validationRules: JSON
  description: String @validate(constraint: "max=500")
}

input AddAttributeToThingClassInput {
  thingClassId: ID! @validate(constraint: "required")
  attributeId: ID! @validate(constraint: "required")
  isRequired: Boolean!
  validationRules: JSON
  sortOrder: Int!
}

type ThingClassConnection {
  edges: [ThingClassEdge!]!
  pageInfo: PageInfo!
}

type ThingClassEdge {
  node: ThingClass!
  cursor: String!
}

type AttributeConnection {
  edges: [AttributeEdge!]!
  pageInfo: PageInfo!
}

type AttributeEdge {
  node: Attribute!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

type ThingClassPayload {
  thingClass: ThingClass
  errors: [UserError!]!
}

type AttributePayload {
  attribute: Attribute
  errors: [UserError!]!
}

type ThingClassAttributePayload {
  thingClassAttribute: ThingClassAttribute
  errors: [UserError!]!
}

type DeleteThingClassPayload {
  success: Boolean!
  errors: [UserError!]!
}

type UserError {
  field: String
  message: String!
  code: String
}

input ThingClassFilter {
  name: String
  isActive: Boolean
  createdAfter: DateTime
  createdBy: ID
}

input AttributeFilter {
  name: String
  dataType: AttributeType
  isActive: Boolean
}

type Query {
  # Thing Class queries
  thingClass(id: ID!): ThingClass @auth(requires: ["READ_THING_CLASS"])
  thingClasses(
    first: Int = 20
    after: String
    filter: ThingClassFilter
  ): ThingClassConnection! @auth(requires: ["READ_THING_CLASS"])
  
  # Attribute queries
  attribute(id: ID!): Attribute @auth(requires: ["READ_ATTRIBUTE"])
  attributes(
    first: Int = 20
    after: String
    filter: AttributeFilter
  ): AttributeConnection! @auth(requires: ["READ_ATTRIBUTE"])
}

type Mutation {
  # Thing Class mutations
  createThingClass(input: CreateThingClassInput!): ThingClassPayload! @auth(requires: ["CREATE_THING_CLASS"])
  updateThingClass(id: ID!, input: UpdateThingClassInput!): ThingClassPayload! @auth(requires: ["UPDATE_THING_CLASS"])
  deleteThingClass(id: ID!): DeleteThingClassPayload! @auth(requires: ["DELETE_THING_CLASS"])
  
  # Attribute mutations
  createAttribute(input: CreateAttributeInput!): AttributePayload! @auth(requires: ["CREATE_ATTRIBUTE"])
  
  # Thing Class Attribute assignment mutations
  addAttributeToThingClass(input: AddAttributeToThingClassInput!): ThingClassAttributePayload! @auth(requires: ["UPDATE_THING_CLASS"])
  removeAttributeFromThingClass(thingClassId: ID!, attributeId: ID!): Boolean! @auth(requires: ["UPDATE_THING_CLASS"])
}
```

### Code Generation Script

```bash
#!/bin/bash
# scripts/generate.sh

set -e

echo "Generating GraphQL resolvers and types..."
go run github.com/99designs/gqlgen generate

echo "Generating database models..."
# Add any additional code generation here

echo "Formatting generated code..."
go fmt ./...

echo "Code generation complete!"
```

## Database Connection Management

### Multi-Tenant Connection Pool Manager

```go
// internal/database/connection.go
package database

import (
	"context"
	"fmt"
	"sync"
	"time"
	
	"github.com/jmoiron/sqlx"
	"github.com/lib/pq"
	"go.uber.org/zap"
)

type Manager struct {
	pools     map[string]*sqlx.DB
	mu        sync.RWMutex
	config    *Config
	logger    *zap.Logger
}

type Config struct {
	Host            string        `env:"DB_HOST" default:"localhost"`
	Port            int           `env:"DB_PORT" default:"5432"`
	Database        string        `env:"DB_NAME" default:"udm"`
	User            string        `env:"DB_USER" default:"postgres"`
	Password        string        `env:"DB_PASSWORD" default:""`
	SSLMode         string        `env:"DB_SSL_MODE" default:"disable"`
	MaxOpenConns    int           `env:"DB_MAX_OPEN_CONNS" default:"25"`
	MaxIdleConns    int           `env:"DB_MAX_IDLE_CONNS" default:"5"`
	ConnMaxLifetime time.Duration `env:"DB_CONN_MAX_LIFETIME" default:"1h"`
	ConnMaxIdleTime time.Duration `env:"DB_CONN_MAX_IDLE_TIME" default:"10m"`
}

func NewManager(config *Config, logger *zap.Logger) *Manager {
	return &Manager{
		pools:  make(map[string]*sqlx.DB),
		config: config,
		logger: logger,
	}
}

func (m *Manager) GetConnection(tenantSchema string) (*sqlx.DB, error) {
	m.mu.RLock()
	if pool, exists := m.pools[tenantSchema]; exists {
		m.mu.RUnlock()
		return pool, nil
	}
	m.mu.RUnlock()

	return m.createConnection(tenantSchema)
}

func (m *Manager) createConnection(tenantSchema string) (*sqlx.DB, error) {
	m.mu.Lock()
	defer m.mu.Unlock()

	// Double-check pattern
	if pool, exists := m.pools[tenantSchema]; exists {
		return pool, nil
	}

	dsn := fmt.Sprintf(
		"host=%s port=%d dbname=%s user=%s password=%s sslmode=%s search_path=%s",
		m.config.Host,
		m.config.Port,
		m.config.Database,
		m.config.User,
		m.config.Password,
		m.config.SSLMode,
		pq.QuoteIdentifier(tenantSchema),
	)

	db, err := sqlx.Connect("postgres", dsn)
	if err != nil {
		m.logger.Error("Failed to create database connection",
			zap.String("tenant_schema", tenantSchema),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to connect to database for tenant %s: %w", tenantSchema, err)
	}

	// Configure connection pool
	db.SetMaxOpenConns(m.config.MaxOpenConns)
	db.SetMaxIdleConns(m.config.MaxIdleConns)
	db.SetConnMaxLifetime(m.config.ConnMaxLifetime)
	db.SetConnMaxIdleTime(m.config.ConnMaxIdleTime)

	// Test connection
	if err := db.Ping(); err != nil {
		db.Close()
		return nil, fmt.Errorf("failed to ping database for tenant %s: %w", tenantSchema, err)
	}

	m.pools[tenantSchema] = db
	m.logger.Info("Created new database connection pool",
		zap.String("tenant_schema", tenantSchema),
		zap.Int("max_open_conns", m.config.MaxOpenConns),
	)

	return db, nil
}

func (m *Manager) CloseAll() error {
	m.mu.Lock()
	defer m.mu.Unlock()

	var errs []error
	for schema, pool := range m.pools {
		if err := pool.Close(); err != nil {
			errs = append(errs, fmt.Errorf("failed to close pool for %s: %w", schema, err))
		}
		m.logger.Info("Closed database connection pool", zap.String("tenant_schema", schema))
	}

	if len(errs) > 0 {
		return fmt.Errorf("errors closing connection pools: %v", errs)
	}
	return nil
}

// HealthCheck returns the health status of all connection pools
func (m *Manager) HealthCheck(ctx context.Context) map[string]error {
	m.mu.RLock()
	defer m.mu.RUnlock()

	results := make(map[string]error)
	for schema, pool := range m.pools {
		if err := pool.PingContext(ctx); err != nil {
			results[schema] = err
		} else {
			results[schema] = nil
		}
	}
	
	return results
}
```

### Database Migration Management

```go
// internal/database/migration.go
package database

import (
	"database/sql"
	"fmt"
	"path/filepath"
	
	"github.com/golang-migrate/migrate/v4"
	"github.com/golang-migrate/migrate/v4/database/postgres"
	_ "github.com/golang-migrate/migrate/v4/source/file"
	"go.uber.org/zap"
)

type MigrationManager struct {
	config *Config
	logger *zap.Logger
}

func NewMigrationManager(config *Config, logger *zap.Logger) *MigrationManager {
	return &MigrationManager{
		config: config,
		logger: logger,
	}
}

func (m *MigrationManager) MigrateTenant(tenantSchema string) error {
	// Create schema if it doesn't exist
	if err := m.createSchemaIfNotExists(tenantSchema); err != nil {
		return fmt.Errorf("failed to create schema %s: %w", tenantSchema, err)
	}

	// Run migrations for the specific tenant schema
	dsn := fmt.Sprintf(
		"host=%s port=%d dbname=%s user=%s password=%s sslmode=%s search_path=%s",
		m.config.Host,
		m.config.Port,
		m.config.Database,
		m.config.User,
		m.config.Password,
		m.config.SSLMode,
		tenantSchema,
	)

	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return fmt.Errorf("failed to connect to database: %w", err)
	}
	defer db.Close()

	driver, err := postgres.WithInstance(db, &postgres.Config{})
	if err != nil {
		return fmt.Errorf("failed to create postgres driver: %w", err)
	}

	migrationPath := filepath.Join("migrations")
	migrator, err := migrate.NewWithDatabaseInstance(
		fmt.Sprintf("file://%s", migrationPath),
		tenantSchema,
		driver,
	)
	if err != nil {
		return fmt.Errorf("failed to create migrator: %w", err)
	}

	if err := migrator.Up(); err != nil && err != migrate.ErrNoChange {
		return fmt.Errorf("failed to run migrations: %w", err)
	}

	m.logger.Info("Successfully migrated tenant schema",
		zap.String("tenant_schema", tenantSchema),
	)

	return nil
}

func (m *MigrationManager) createSchemaIfNotExists(schemaName string) error {
	dsn := fmt.Sprintf(
		"host=%s port=%d dbname=%s user=%s password=%s sslmode=%s",
		m.config.Host,
		m.config.Port,
		m.config.Database,
		m.config.User,
		m.config.Password,
		m.config.SSLMode,
	)

	db, err := sql.Open("postgres", dsn)
	if err != nil {
		return err
	}
	defer db.Close()

	query := fmt.Sprintf("CREATE SCHEMA IF NOT EXISTS %s", pq.QuoteIdentifier(schemaName))
	_, err = db.Exec(query)
	return err
}
```

## Advanced Resolver Patterns

### Custom Scalar Resolvers

```go
// internal/graph/resolvers/scalars.go
package resolvers

import (
	"encoding/json"
	"errors"
	"fmt"
	"io"
	"strconv"
	"time"
	
	"github.com/google/uuid"
)

// DateTime scalar resolver
func MarshalDateTime(t time.Time) string {
	return t.Format(time.RFC3339)
}

func UnmarshalDateTime(v interface{}) (time.Time, error) {
	switch v := v.(type) {
	case string:
		return time.Parse(time.RFC3339, v)
	case int64:
		return time.Unix(v, 0), nil
	default:
		return time.Time{}, fmt.Errorf("cannot unmarshal %T as DateTime", v)
	}
}

// JSON scalar resolver
func MarshalJSON(data interface{}) string {
	switch data := data.(type) {
	case json.RawMessage:
		return string(data)
	case []byte:
		return string(data)
	default:
		bytes, _ := json.Marshal(data)
		return string(bytes)
	}
}

func UnmarshalJSON(v interface{}) (interface{}, error) {
	switch v := v.(type) {
	case string:
		var result interface{}
		if err := json.Unmarshal([]byte(v), &result); err != nil {
			return nil, err
		}
		return result, nil
	case json.RawMessage:
		return v, nil
	default:
		return nil, fmt.Errorf("cannot unmarshal %T as JSON", v)
	}
}

// ID (UUID) scalar resolver
func MarshalID(id uuid.UUID) string {
	return id.String()
}

func UnmarshalID(v interface{}) (uuid.UUID, error) {
	switch v := v.(type) {
	case string:
		return uuid.Parse(v)
	default:
		return uuid.UUID{}, fmt.Errorf("cannot unmarshal %T as UUID", v)
	}
}
```

### Directive Implementations

```go
// internal/graph/directives/auth.go
package directives

import (
	"context"
	"fmt"
	
	"github.com/99designs/gqlgen/graphql"
	"github.com/company/udm-backend/internal/domain"
)

func AuthDirective(ctx context.Context, obj interface{}, next graphql.Resolver, requires []string) (interface{}, error) {
	// Extract tenant context from GraphQL context
	tenantCtx, ok := ctx.Value("tenant").(*domain.TenantContext)
	if !ok || tenantCtx == nil {
		return nil, fmt.Errorf("authentication required")
	}

	// Check if user has any of the required permissions
	hasPermission := false
	for _, required := range requires {
		for _, userPerm := range tenantCtx.Permissions {
			if userPerm == required {
				hasPermission = true
				break
			}
		}
		if hasPermission {
			break
		}
	}

	if !hasPermission {
		return nil, fmt.Errorf("insufficient permissions: requires one of %v", requires)
	}

	return next(ctx)
}

// internal/graph/directives/validate.go
func ValidateDirective(ctx context.Context, obj interface{}, next graphql.Resolver, constraint string) (interface{}, error) {
	// Get the field value from the input
	fieldCtx := graphql.GetFieldContext(ctx)
	if fieldCtx == nil {
		return next(ctx)
	}

	// Apply validation based on constraint
	// This would integrate with a validation library like go-playground/validator
	// For now, we'll do basic validation
	
	return next(ctx)
}
```

## Service Layer Patterns

### Business Logic Service

```go
// internal/service/thingclass.go
package service

import (
	"context"
	"encoding/json"
	"fmt"
	
	"github.com/company/udm-backend/internal/domain"
	"github.com/company/udm-backend/internal/repository/interfaces"
	"github.com/go-playground/validator/v10"
	"github.com/google/uuid"
	"go.uber.org/zap"
)

type ThingClassService struct {
	repo      interfaces.ThingClassRepository
	validator *validator.Validate
	logger    *zap.Logger
}

func NewThingClassService(repo interfaces.ThingClassRepository, logger *zap.Logger) *ThingClassService {
	return &ThingClassService{
		repo:      repo,
		validator: validator.New(),
		logger:    logger,
	}
}

func (s *ThingClassService) CreateThingClass(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateThingClassInput) (*domain.ThingClass, error) {
	// Validate input
	if err := s.validateCreateInput(input); err != nil {
		return nil, fmt.Errorf("validation failed: %w", err)
	}

	// Check if name already exists
	exists, err := s.repo.ExistsByName(ctx, tenant, input.Name)
	if err != nil {
		s.logger.Error("Failed to check thing class existence",
			zap.String("name", input.Name),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to check thing class existence: %w", err)
	}
	
	if exists {
		return nil, fmt.Errorf("thing class with name '%s' already exists", input.Name)
	}

	// Create domain object with base entity fields
	now := time.Now()
	thingClass := &domain.ThingClass{
		BaseEntity: domain.BaseEntity{
			ID:        uuid.New(),
			CreatedAt: now,
			UpdatedAt: now,
			CreatedBy: tenant.UserID,
			IsActive:  true,
			Version:   1,
		},
		Name:        input.Name,
		Description: input.Description,
	}

	// Create in repository
	created, err := s.repo.Create(ctx, tenant, thingClass)
	if err != nil {
		s.logger.Error("Failed to create thing class",
			zap.String("name", input.Name),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to create thing class: %w", err)
	}

	s.logger.Info("Successfully created thing class",
		zap.String("id", created.ID.String()),
		zap.String("name", created.Name),
		zap.String("tenant_schema", tenant.SchemaName),
	)

	return created, nil
}

func (s *ThingClassService) AddAttributeToThingClass(ctx context.Context, tenant *domain.TenantContext, input *domain.AddAttributeToThingClassInput) (*domain.ThingClassAttribute, error) {
	// Validate input
	if err := s.validateAddAttributeInput(input); err != nil {
		return nil, fmt.Errorf("validation failed: %w", err)
	}

	// Verify thing class exists
	thingClass, err := s.repo.GetByID(ctx, tenant, input.ThingClassID)
	if err != nil {
		return nil, fmt.Errorf("failed to get thing class: %w", err)
	}
	if thingClass == nil {
		return nil, fmt.Errorf("thing class not found")
	}

	// Verify attribute exists
	attribute, err := s.attributeRepo.GetByID(ctx, tenant, input.AttributeID)
	if err != nil {
		return nil, fmt.Errorf("failed to get attribute: %w", err)
	}
	if attribute == nil {
		return nil, fmt.Errorf("attribute not found")
	}

	// Check if assignment already exists
	exists, err := s.repo.AttributeAssignmentExists(ctx, tenant, input.ThingClassID, input.AttributeID)
	if err != nil {
		return nil, fmt.Errorf("failed to check attribute assignment existence: %w", err)
	}
	if exists {
		return nil, fmt.Errorf("attribute already assigned to this thing class")
	}

	// Create assignment
	now := time.Now()
	assignment := &domain.ThingClassAttribute{
		BaseEntity: domain.BaseEntity{
			ID:        uuid.New(),
			CreatedAt: now,
			UpdatedAt: now,
			CreatedBy: tenant.UserID,
			IsActive:  true,
			Version:   1,
		},
		ThingClassID:    input.ThingClassID,
		AttributeID:     input.AttributeID,
		IsRequired:      input.IsRequired,
		ValidationRules: input.ValidationRules,
		SortOrder:       input.SortOrder,
	}

	created, err := s.repo.CreateAttributeAssignment(ctx, tenant, assignment)
	if err != nil {
		s.logger.Error("Failed to create attribute assignment",
			zap.String("thing_class_id", input.ThingClassID.String()),
			zap.String("attribute_id", input.AttributeID.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to create attribute assignment: %w", err)
	}

	s.logger.Info("Successfully created attribute assignment",
		zap.String("id", created.ID.String()),
		zap.String("thing_class_id", input.ThingClassID.String()),
		zap.String("attribute_id", input.AttributeID.String()),
		zap.String("tenant_schema", tenant.SchemaName),
	)

	return created, nil
}

func (s *ThingClassService) GetThingClass(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.ThingClass, error) {
	thingClass, err := s.repo.GetByID(ctx, tenant, id)
	if err != nil {
		s.logger.Error("Failed to get thing class",
			zap.String("id", id.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to get thing class: %w", err)
	}

	if thingClass == nil {
		return nil, fmt.Errorf("thing class not found")
	}

	return thingClass, nil
}

func (s *ThingClassService) validateCreateInput(input *domain.CreateThingClassInput) error {
	if input.Name == "" {
		return fmt.Errorf("name is required")
	}
	
	if len(input.Name) > 100 {
		return fmt.Errorf("name must be 100 characters or less")
	}
	
	if input.Description != nil && len(*input.Description) > 500 {
		return fmt.Errorf("description must be 500 characters or less")
	}
	
	return nil
}

func (s *ThingClassService) validateValidationRules(rules json.RawMessage) error {
	// Parse and validate the JSON structure
	var rulesMap map[string]interface{}
	if err := json.Unmarshal(rules, &rulesMap); err != nil {
		return fmt.Errorf("validation rules must be valid JSON: %w", err)
	}
	
	// Add specific validation logic for rule structure
	// This would depend on your validation rule schema
	
	return nil
}
```

## Performance Optimization Techniques

### Connection Pool Optimization

```go
// internal/database/optimization.go
package database

import (
	"context"
	"database/sql"
	"fmt"
	"time"
	
	"github.com/jmoiron/sqlx"
	"go.uber.org/zap"
)

type PerformanceMonitor struct {
	logger *zap.Logger
}

func NewPerformanceMonitor(logger *zap.Logger) *PerformanceMonitor {
	return &PerformanceMonitor{logger: logger}
}

// MonitorConnectionPools monitors and logs connection pool statistics
func (pm *PerformanceMonitor) MonitorConnectionPools(ctx context.Context, manager *Manager) {
	ticker := time.NewTicker(30 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			pm.logPoolStats(manager)
		}
	}
}

func (pm *PerformanceMonitor) logPoolStats(manager *Manager) {
	manager.mu.RLock()
	defer manager.mu.RUnlock()

	for schema, db := range manager.pools {
		stats := db.Stats()
		pm.logger.Info("Database connection pool stats",
			zap.String("tenant_schema", schema),
			zap.Int("open_connections", stats.OpenConnections),
			zap.Int("in_use", stats.InUse),
			zap.Int("idle", stats.Idle),
			zap.Int64("wait_count", stats.WaitCount),
			zap.Duration("wait_duration", stats.WaitDuration),
			zap.Int64("max_idle_closed", stats.MaxIdleClosed),
			zap.Int64("max_idle_time_closed", stats.MaxIdleTimeClosed),
			zap.Int64("max_lifetime_closed", stats.MaxLifetimeClosed),
		)

		// Alert if pool is under pressure
		if float64(stats.InUse)/float64(stats.MaxOpenConnections) > 0.8 {
			pm.logger.Warn("Database connection pool under pressure",
				zap.String("tenant_schema", schema),
				zap.Float64("utilization", float64(stats.InUse)/float64(stats.MaxOpenConnections)),
			)
		}
	}
}

// QueryContext wraps sqlx.DB.QueryxContext with performance logging
func (pm *PerformanceMonitor) QueryContext(ctx context.Context, db *sqlx.DB, query string, args ...interface{}) (*sqlx.Rows, error) {
	start := time.Now()
	rows, err := db.QueryxContext(ctx, query, args...)
	duration := time.Since(start)

	pm.logger.Debug("Database query executed",
		zap.String("query", query),
		zap.Duration("duration", duration),
		zap.Error(err),
	)

	// Alert on slow queries
	if duration > 1*time.Second {
		pm.logger.Warn("Slow database query detected",
			zap.String("query", query),
			zap.Duration("duration", duration),
		)
	}

	return rows, err
}
```

### Caching Layer Integration

```go
// internal/cache/redis.go
package cache

import (
	"context"
	"encoding/json"
	"fmt"
	"time"
	
	"github.com/redis/go-redis/v9"
	"go.uber.org/zap"
)

type RedisCache struct {
	client *redis.Client
	logger *zap.Logger
	prefix string
	ttl    time.Duration
}

func NewRedisCache(addr, password string, db int, logger *zap.Logger) *RedisCache {
	client := redis.NewClient(&redis.Options{
		Addr:     addr,
		Password: password,
		DB:       db,
	})

	return &RedisCache{
		client: client,
		logger: logger,
		prefix: "udm",
		ttl:    5 * time.Minute,
	}
}

func (c *RedisCache) Set(ctx context.Context, tenantSchema, key string, value interface{}) error {
	cacheKey := c.buildKey(tenantSchema, key)
	
	data, err := json.Marshal(value)
	if err != nil {
		return fmt.Errorf("failed to marshal cache value: %w", err)
	}
	
	if err := c.client.Set(ctx, cacheKey, data, c.ttl).Err(); err != nil {
		c.logger.Error("Failed to set cache value",
			zap.String("key", cacheKey),
			zap.Error(err),
		)
		return err
	}
	
	return nil
}

func (c *RedisCache) Get(ctx context.Context, tenantSchema, key string, dest interface{}) error {
	cacheKey := c.buildKey(tenantSchema, key)
	
	data, err := c.client.Get(ctx, cacheKey).Result()
	if err != nil {
		if err == redis.Nil {
			return ErrCacheMiss
		}
		return err
	}
	
	return json.Unmarshal([]byte(data), dest)
}

func (c *RedisCache) Delete(ctx context.Context, tenantSchema, key string) error {
	cacheKey := c.buildKey(tenantSchema, key)
	return c.client.Del(ctx, cacheKey).Err()
}

func (c *RedisCache) InvalidatePattern(ctx context.Context, tenantSchema, pattern string) error {
	searchKey := c.buildKey(tenantSchema, pattern)
	keys, err := c.client.Keys(ctx, searchKey).Result()
	if err != nil {
		return err
	}
	
	if len(keys) == 0 {
		return nil
	}
	
	return c.client.Del(ctx, keys...).Err()
}

func (c *RedisCache) buildKey(tenantSchema, key string) string {
	return fmt.Sprintf("%s:%s:%s", c.prefix, tenantSchema, key)
}

var ErrCacheMiss = fmt.Errorf("cache miss")
```

## Application Bootstrap

### Main Application Entry Point

```go
// cmd/server/main.go
package main

import (
	"context"
	"fmt"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/99designs/gqlgen/graphql/handler"
	"github.com/99designs/gqlgen/graphql/playground"
	"github.com/gorilla/mux"
	"github.com/rs/cors"
	"go.uber.org/zap"

	"github.com/company/udm-backend/internal/config"
	"github.com/company/udm-backend/internal/database"
	"github.com/company/udm-backend/internal/graph/generated"
	"github.com/company/udm-backend/internal/graph/resolvers"
	"github.com/company/udm-backend/internal/middleware"
	"github.com/company/udm-backend/internal/repository/postgres"
	"github.com/company/udm-backend/internal/service"
)

func main() {
	// Initialize logger
	logger, err := zap.NewProduction()
	if err != nil {
		panic(fmt.Sprintf("Failed to initialize logger: %v", err))
	}
	defer logger.Sync()

	// Load configuration
	cfg, err := config.Load()
	if err != nil {
		logger.Fatal("Failed to load configuration", zap.Error(err))
	}

	// Initialize database manager
	dbManager := database.NewManager(&cfg.Database, logger)
	defer dbManager.CloseAll()

	// Initialize repositories
	thingClassRepo := postgres.NewThingClassRepository(dbManager, logger)

	// Initialize services
	thingClassService := service.NewThingClassService(thingClassRepo, logger)

	// Initialize GraphQL resolver
	resolver := &resolvers.Resolver{
		ThingClassService: thingClassService,
		Logger:           logger,
	}

	// Create GraphQL server
	srv := handler.NewDefaultServer(generated.NewExecutableSchema(generated.Config{
		Resolvers: resolver,
	}))

	// Setup HTTP router
	router := mux.NewRouter()
	
	// Add middleware
	tenantMiddleware := middleware.NewTenantMiddleware(cfg.JWT.Secret)
	router.Use(tenantMiddleware.Handler)
	
	loggingMiddleware := middleware.NewLoggingMiddleware(logger)
	router.Use(loggingMiddleware.Handler)

	// GraphQL endpoints
	router.Handle("/graphql", srv)
	if cfg.Environment == "development" {
		router.Handle("/playground", playground.Handler("GraphQL Playground", "/graphql"))
	}

	// Health check endpoint
	router.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("OK"))
	})

	// CORS configuration
	c := cors.New(cors.Options{
		AllowedOrigins:   cfg.CORS.AllowedOrigins,
		AllowedMethods:   []string{"GET", "POST", "PUT", "DELETE", "OPTIONS"},
		AllowedHeaders:   []string{"*"},
		AllowCredentials: true,
	})

	// Create HTTP server
	server := &http.Server{
		Addr:         fmt.Sprintf(":%d", cfg.Server.Port),
		Handler:      c.Handler(router),
		ReadTimeout:  cfg.Server.ReadTimeout,
		WriteTimeout: cfg.Server.WriteTimeout,
		IdleTimeout:  cfg.Server.IdleTimeout,
	}

	// Start server in goroutine
	go func() {
		logger.Info("Starting HTTP server",
			zap.Int("port", cfg.Server.Port),
			zap.String("environment", cfg.Environment),
		)
		
		if err := server.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			logger.Fatal("Failed to start HTTP server", zap.Error(err))
		}
	}()

	// Wait for interrupt signal to gracefully shutdown
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	logger.Info("Shutting down server...")

	// Create shutdown context with timeout
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	// Shutdown server gracefully
	if err := server.Shutdown(ctx); err != nil {
		logger.Error("Server forced to shutdown", zap.Error(err))
	}

	logger.Info("Server shutdown complete")
}
```

## Docker Configuration

### Dockerfile

```dockerfile
# Build stage
FROM golang:1.21-alpine AS builder

# Install build dependencies
RUN apk add --no-cache git ca-certificates tzdata

WORKDIR /app

# Copy go mod files
COPY go.mod go.sum ./

# Download dependencies
RUN go mod download

# Copy source code
COPY . .

# Generate GraphQL code
RUN go run github.com/99designs/gqlgen generate

# Build binary
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main cmd/server/main.go

# Final stage
FROM alpine:latest

# Install ca-certificates for HTTPS
RUN apk --no-cache add ca-certificates tzdata

WORKDIR /root/

# Copy binary from builder stage
COPY --from=builder /app/main .

# Copy migration files
COPY --from=builder /app/migrations ./migrations

# Expose port
EXPOSE 8080

# Run the binary
CMD ["./main"]
```

### Production Docker Configuration

```yaml
# docker/docker-compose.prod.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: udm-postgres-prod
    restart: always
    environment:
      POSTGRES_DB: udm_production
      POSTGRES_USER: udm_user
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/postgresql.conf:/etc/postgresql/postgresql.conf
      - ./postgres/init:/docker-entrypoint-initdb.d
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U udm_user -d udm_production"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s
    networks:
      - udm-network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  redis:
    image: redis:7-alpine
    container_name: udm-redis-prod
    restart: always
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD} --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3
    networks:
      - udm-network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  udm-backend:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: production
    container_name: udm-backend-prod
    restart: always
    environment:
      # Database configuration
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: udm_production
      DB_USER: udm_user
      DB_PASSWORD: ${POSTGRES_PASSWORD}
      DB_SSL_MODE: require
      DB_MAX_OPEN_CONNS: 50
      DB_MAX_IDLE_CONNS: 10
      DB_CONN_MAX_LIFETIME: 1h
      
      # Redis configuration
      REDIS_ADDR: redis:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      REDIS_DB: 0
      REDIS_POOL_SIZE: 20
      
      # Application configuration
      ENVIRONMENT: production
      LOG_LEVEL: info
      JWT_SECRET: ${JWT_SECRET}
      
      # Server configuration
      SERVER_PORT: 8080
      SERVER_READ_TIMEOUT: 30s
      SERVER_WRITE_TIMEOUT: 30s
      
      # Security
      CORS_ALLOWED_ORIGINS: ${CORS_ALLOWED_ORIGINS}
      
    ports:
      - "8080:8080"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - udm-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    logging:
      driver: "json-file"
      options:
        max-size: "50m"
        max-file: "5"

volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local

networks:
  udm-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

# Environment file template (.env.prod)
# POSTGRES_PASSWORD=your_secure_postgres_password
# REDIS_PASSWORD=your_secure_redis_password  
# JWT_SECRET=your_jwt_secret_key_minimum_32_characters
# CORS_ALLOWED_ORIGINS=https://yourdomain.com,https://app.yourdomain.com
```

### Development Docker Configuration

```yaml
# docker/docker-compose.dev.yml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: udm-postgres-dev
    restart: unless-stopped
    environment:
      POSTGRES_DB: udm_development
      POSTGRES_USER: udm_user
      POSTGRES_PASSWORD: udm_dev_password
      PGDATA: /var/lib/postgresql/data/pgdata
    ports:
      - "5432:5432"
    volumes:
      - postgres_dev_data:/var/lib/postgresql/data
      - ./postgres/init:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U udm_user -d udm_development"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - udm-dev-network

  redis:
    image: redis:7-alpine
    container_name: udm-redis-dev
    restart: unless-stopped
    command: redis-server --appendonly yes --requirepass udm_dev_redis_password
    ports:
      - "6379:6379"
    volumes:
      - redis_dev_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
    networks:
      - udm-dev-network

  udm-backend:
    build:
      context: .
      dockerfile: docker/Dockerfile
      target: development
    container_name: udm-backend-dev
    restart: unless-stopped
    environment:
      # Database configuration
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: udm_development
      DB_USER: udm_user
      DB_PASSWORD: udm_dev_password
      DB_SSL_MODE: disable
      
      # Redis configuration
      REDIS_ADDR: redis:6379
      REDIS_PASSWORD: udm_dev_redis_password
      REDIS_DB: 0
      
      # Application configuration
      ENVIRONMENT: development
      LOG_LEVEL: debug
      JWT_SECRET: development_jwt_secret_key_change_in_production
      
      # Development features
      ENABLE_PLAYGROUND: true
      ENABLE_INTROSPECTION: true
      
    ports:
      - "8080:8080"
      - "2345:2345" # Delve debugger port
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
      - udm-dev-network

volumes:
  postgres_dev_data:
    driver: local
  redis_dev_data:
    driver: local  
  go_modules:
    driver: local

networks:
  udm-dev-network:
    driver: bridge
```

This comprehensive Go implementation provides all the patterns and structures needed to implement the Thing Class entity using Go, gqlgen, and PostgreSQL with proper multi-tenant architecture.