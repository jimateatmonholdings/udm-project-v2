# Go Implementation Details

This document provides comprehensive Go implementation patterns and project structure for the Thing Entity specification.

> Created: 2025-08-29
> Version: 1.0.0
> Language: Go + gqlgen + PostgreSQL

## gqlgen Configuration Updates

### Updated gqlgen.yml Configuration

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
  Thing:
    model:
      - github.com/company/udm-backend/internal/domain.Thing
  ThingRelationship:
    model:
      - github.com/company/udm-backend/internal/domain.ThingRelationship
  ThingFilter:
    model:
      - github.com/company/udm-backend/internal/domain.ThingFilter
  ThingConnection:
    model:
      - github.com/company/udm-backend/internal/domain.ThingConnection
  ThingEdge:
    model:
      - github.com/company/udm-backend/internal/domain.ThingEdge
  CreateThingInput:
    model:
      - github.com/company/udm-backend/internal/domain.CreateThingInput
  UpdateThingInput:
    model:
      - github.com/company/udm-backend/internal/domain.UpdateThingInput
  ThingPayload:
    model:
      - github.com/company/udm-backend/internal/domain.ThingPayload
  DeleteThingPayload:
    model:
      - github.com/company/udm-backend/internal/domain.DeleteThingPayload
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

## Project Structure Extensions

### Additional Go Module Layout

```
github.com/company/udm-backend/
├── internal/
│   ├── domain/                     # Domain models and interfaces
│   │   ├── thing.go                # Thing entity domain model
│   │   ├── thingclass.go
│   │   └── errors.go
│   ├── graph/                      # GraphQL layer (gqlgen generated + custom)
│   │   ├── resolvers/              # Custom resolver implementations
│   │   │   ├── thing.go            # Thing resolvers
│   │   │   └── thingclass.go
│   │   └── schema.graphqls         # Updated GraphQL schema definitions
│   ├── repository/                 # Data access layer
│   │   ├── interfaces/
│   │   │   ├── thing.go            # Thing repository interface
│   │   │   └── thingclass.go
│   │   └── postgres/
│   │       ├── thing.go            # Thing PostgreSQL implementation
│   │       └── thingclass.go
│   ├── service/                    # Business logic layer
│   │   ├── thing.go                # Thing business logic
│   │   └── thingclass.go
│   ├── dataloader/                 # DataLoader implementations
│   │   ├── thing.go                # Thing batch loading
│   │   └── thingclass.go           # ThingClass batch loading
│   └── cache/                      # Caching layer
│       ├── redis.go
│       └── thing.go                # Thing-specific caching
├── migrations/                     # Database migrations
│   ├── 002_create_things.up.sql
│   └── 002_create_things.down.sql
```

## Domain Model Implementation

### Thing Entity Domain Model

```go
// internal/domain/thing.go
package domain

import (
	"encoding/json"
	"github.com/google/uuid"
)

type Thing struct {
	BaseEntity
	ThingClassID uuid.UUID `json:"thingClassId" db:"thing_class_id"`
	TenantID     uuid.UUID `json:"tenantId" db:"tenant_id"`
	Name         string    `json:"name" db:"name"`
	
	// Loaded relationships (not persisted directly)
	ThingClass    *ThingClass        `json:"thingClass,omitempty" db:"-"`
	AttributeValues []*Value         `json:"attributeValues,omitempty" db:"-"`
	Relationships []*ThingRelationship `json:"relationships,omitempty" db:"-"`
}

// Value represents an actual attribute value for a specific thing
type Value struct {
	BaseEntity
	ThingID     uuid.UUID       `json:"thingId" db:"thing_id"`
	AttributeID uuid.UUID       `json:"attributeId" db:"attribute_id"`
	ValueData   json.RawMessage `json:"valueData" db:"value_data"`
	
	// Loaded relationships
	Thing     *Thing     `json:"thing,omitempty" db:"-"`
	Attribute *Attribute `json:"attribute,omitempty" db:"-"`
}

type ThingRelationship struct {
	BaseEntity
	SourceThingID    uuid.UUID       `json:"sourceThingId" db:"source_thing_id"`
	TargetThingID    uuid.UUID       `json:"targetThingId" db:"target_thing_id"`
	RelationshipType string          `json:"relationshipType" db:"relationship_type"`
	Metadata         json.RawMessage `json:"metadata" db:"metadata"`
	
	// Loaded relationships
	SourceThing *Thing `json:"sourceThing,omitempty" db:"-"`
	TargetThing *Thing `json:"targetThing,omitempty" db:"-"`
}

// Input types for GraphQL mutations
type CreateThingInput struct {
	ThingClassID uuid.UUID `json:"thingClassId" validate:"required"`
	Name         string    `json:"name" validate:"required,min=1,max=255"`
}

type UpdateThingInput struct {
	Name *string `json:"name,omitempty" validate:"omitempty,min=1,max=255"`
}

type CreateValueInput struct {
	ThingID     uuid.UUID       `json:"thingId" validate:"required"`
	AttributeID uuid.UUID       `json:"attributeId" validate:"required"`
	ValueData   json.RawMessage `json:"valueData" validate:"required"`
}

type UpdateValueInput struct {
	ValueData json.RawMessage `json:"valueData" validate:"required"`
}

type CreateThingRelationshipInput struct {
	SourceThingID    uuid.UUID       `json:"sourceThingId" validate:"required"`
	TargetThingID    uuid.UUID       `json:"targetThingId" validate:"required"`
	RelationshipType string          `json:"relationshipType" validate:"required,min=1,max=100"`
	Metadata         json.RawMessage `json:"metadata,omitempty"`
}

// Filter types for GraphQL queries
type ThingFilter struct {
	ThingClassID *uuid.UUID      `json:"thingClassId,omitempty"`
	Name         *string         `json:"name,omitempty"`
	Attributes   json.RawMessage `json:"attributes,omitempty"`
	CreatedAfter *time.Time      `json:"createdAfter,omitempty"`
	CreatedBy    *uuid.UUID      `json:"createdBy,omitempty"`
}

type ThingConnection struct {
	Edges    []*ThingEdge `json:"edges"`
	PageInfo *PageInfo    `json:"pageInfo"`
}

type ThingEdge struct {
	Node   *Thing `json:"node"`
	Cursor string `json:"cursor"`
}

// Validation and business logic methods
func (t *Thing) ValidateAttributes(thingClass *ThingClass) error {
	if len(thingClass.ValidationRules) == 0 {
		return nil
	}
	
	var rules map[string]interface{}
	if err := json.Unmarshal(thingClass.ValidationRules, &rules); err != nil {
		return fmt.Errorf("invalid validation rules in thing class: %w", err)
	}
	
	var attributes map[string]interface{}
	if err := json.Unmarshal(t.Attributes, &attributes); err != nil {
		return fmt.Errorf("invalid attributes JSON: %w", err)
	}
	
	// Apply validation rules
	return validateAttributesAgainstRules(attributes, rules)
}

func validateAttributesAgainstRules(attributes, rules map[string]interface{}) error {
	// Implementation would depend on your validation rule schema
	// This is a simplified example
	for ruleKey, ruleValue := range rules {
		if required, ok := ruleValue.(map[string]interface{})["required"].(bool); ok && required {
			if _, exists := attributes[ruleKey]; !exists {
				return fmt.Errorf("required attribute '%s' is missing", ruleKey)
			}
		}
	}
	return nil
}

// Thing cache keys
func (t *Thing) CacheKey() string {
	return fmt.Sprintf("thing:%s", t.ID)
}

func (t *Thing) RelationshipsCacheKey() string {
	return fmt.Sprintf("thing_relationships:%s", t.ID)
}
```

## GraphQL Schema Extensions

### Updated GraphQL Schema

```graphql
# internal/graph/schema.graphqls (additions to existing schema)

type Thing {
  id: ID!
  thingClassId: ID!
  thingClass: ThingClass!
  name: String!
  attributeValues: [Value!]!
  relationships: [ThingRelationship!]!
  # Common entity fields
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  isActive: Boolean!
  version: Int!
  tenantId: ID!
}

type Value {
  id: ID!
  thingId: ID!
  attributeId: ID!
  thing: Thing!
  attribute: Attribute!
  valueData: JSON!
  # Common entity fields
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  isActive: Boolean!
  version: Int!
}

type ThingRelationship {
  id: ID!
  sourceThingId: ID!
  targetThingId: ID!
  sourceThing: Thing!
  targetThing: Thing!
  relationshipType: String!
  metadata: JSON
  createdAt: DateTime!
}

input CreateThingInput {
  thingClassId: ID! @validate(constraint: "required")
  name: String! @validate(constraint: "required,min=1,max=255")
}

input UpdateThingInput {
  name: String @validate(constraint: "min=1,max=255")
}

input CreateValueInput {
  thingId: ID! @validate(constraint: "required")
  attributeId: ID! @validate(constraint: "required")
  valueData: JSON! @validate(constraint: "required")
}

input UpdateValueInput {
  valueData: JSON! @validate(constraint: "required")
}

input CreateThingRelationshipInput {
  sourceThingId: ID! @validate(constraint: "required")
  targetThingId: ID! @validate(constraint: "required")
  relationshipType: String! @validate(constraint: "required,min=1,max=100")
  metadata: JSON
}

input ThingFilter {
  thingClassId: ID
  name: String
  attributes: JSON
  createdAfter: DateTime
  createdBy: ID
}

type ThingConnection {
  edges: [ThingEdge!]!
  pageInfo: PageInfo!
}

type ThingEdge {
  node: Thing!
  cursor: String!
}

type ThingPayload {
  thing: Thing
  errors: [UserError!]!
}

type DeleteThingPayload {
  success: Boolean!
  errors: [UserError!]!
}

extend type Query {
  thing(id: ID!): Thing @auth(requires: ["READ_THING"])
  things(
    first: Int
    after: String
    filter: ThingFilter
  ): ThingConnection! @auth(requires: ["READ_THING"])
}

extend type Mutation {
  createThing(input: CreateThingInput!): ThingPayload! @auth(requires: ["CREATE_THING"])
  updateThing(id: ID!, input: UpdateThingInput!): ThingPayload! @auth(requires: ["UPDATE_THING"])
  deleteThing(id: ID!): DeleteThingPayload! @auth(requires: ["DELETE_THING"])
}
```

## Repository Layer Implementation

### Thing Repository Interface

```go
// internal/repository/interfaces/thing.go
package interfaces

import (
	"context"
	
	"github.com/company/udm-backend/internal/domain"
	"github.com/google/uuid"
)

type ThingRepository interface {
	// CRUD operations
	Create(ctx context.Context, tenant *domain.TenantContext, thing *domain.Thing) (*domain.Thing, error)
	GetByID(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.Thing, error)
	GetByIDs(ctx context.Context, tenant *domain.TenantContext, ids []uuid.UUID) ([]*domain.Thing, error)
	Update(ctx context.Context, tenant *domain.TenantContext, thing *domain.Thing) (*domain.Thing, error)
	Delete(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) error
	
	// Query operations
	List(ctx context.Context, tenant *domain.TenantContext, filter *domain.ThingFilter, pagination *domain.PaginationInput) (*domain.ThingConnection, error)
	ExistsByName(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID, name string) (bool, error)
	CountByThingClass(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID) (int64, error)
	
	// Relationship operations
	GetRelationships(ctx context.Context, tenant *domain.TenantContext, thingID uuid.UUID) ([]*domain.ThingRelationship, error)
	CreateRelationship(ctx context.Context, tenant *domain.TenantContext, relationship *domain.ThingRelationship) (*domain.ThingRelationship, error)
	DeleteRelationship(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) error
	
	// Batch operations for DataLoader
	GetThingsByThingClassIDs(ctx context.Context, tenant *domain.TenantContext, thingClassIDs []uuid.UUID) (map[uuid.UUID][]*domain.Thing, error)
}
```

### PostgreSQL Implementation

```go
// internal/repository/postgres/thing.go
package postgres

import (
	"context"
	"database/sql"
	"fmt"
	"strings"
	"time"
	
	"github.com/company/udm-backend/internal/database"
	"github.com/company/udm-backend/internal/domain"
	"github.com/company/udm-backend/internal/repository/interfaces"
	"github.com/google/uuid"
	"github.com/jmoiron/sqlx"
	"github.com/lib/pq"
	"go.uber.org/zap"
)

type thingRepository struct {
	db     *database.Manager
	logger *zap.Logger
}

func NewThingRepository(db *database.Manager, logger *zap.Logger) interfaces.ThingRepository {
	return &thingRepository{
		db:     db,
		logger: logger,
	}
}

func (r *thingRepository) Create(ctx context.Context, tenant *domain.TenantContext, thing *domain.Thing) (*domain.Thing, error) {
	conn, err := r.db.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}
	
	// Set timestamps and tenant ID
	now := time.Now()
	thing.CreatedAt = now
	thing.UpdatedAt = now
	thing.TenantID = tenant.TenantID
	thing.Version = 1
	
	query := `
		INSERT INTO things (
			id, thing_class_id, tenant_id, name, thing_attributes,
			version, created_at, updated_at, created_by
		) VALUES (
			$1, $2, $3, $4, $5, $6, $7, $8, $9
		) RETURNING *`
	
	var created domain.Thing
	err = conn.GetContext(ctx, &created, query,
		thing.ID, thing.ThingClassID, thing.TenantID, thing.Name, thing.Attributes,
		thing.Version, thing.CreatedAt, thing.UpdatedAt, thing.CreatedBy,
	)
	
	if err != nil {
		if pqErr, ok := err.(*pq.Error); ok {
			switch pqErr.Code {
			case "23505": // unique_violation
				if strings.Contains(pqErr.Message, "name") {
					return nil, fmt.Errorf("thing with name '%s' already exists", thing.Name)
				}
			case "23503": // foreign_key_violation
				return nil, fmt.Errorf("invalid thing class ID")
			}
		}
		r.logger.Error("Failed to create thing",
			zap.String("name", thing.Name),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, err
	}
	
	return &created, nil
}

func (r *thingRepository) GetByID(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.Thing, error) {
	conn, err := r.db.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}
	
	query := `
		SELECT * FROM things 
		WHERE id = $1 AND tenant_id = $2 AND deleted_at IS NULL`
	
	var thing domain.Thing
	err = conn.GetContext(ctx, &thing, query, id, tenant.TenantID)
	
	if err != nil {
		if err == sql.ErrNoRows {
			return nil, nil
		}
		r.logger.Error("Failed to get thing by ID",
			zap.String("id", id.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, err
	}
	
	return &thing, nil
}

func (r *thingRepository) GetByIDs(ctx context.Context, tenant *domain.TenantContext, ids []uuid.UUID) ([]*domain.Thing, error) {
	if len(ids) == 0 {
		return []*domain.Thing{}, nil
	}
	
	conn, err := r.db.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}
	
	query := `
		SELECT * FROM things 
		WHERE id = ANY($1) AND tenant_id = $2 AND deleted_at IS NULL`
	
	var things []*domain.Thing
	err = conn.SelectContext(ctx, &things, query, pq.Array(ids), tenant.TenantID)
	
	if err != nil {
		r.logger.Error("Failed to get things by IDs",
			zap.Int("count", len(ids)),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, err
	}
	
	return things, nil
}

func (r *thingRepository) Update(ctx context.Context, tenant *domain.TenantContext, thing *domain.Thing) (*domain.Thing, error) {
	conn, err := r.db.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}
	
	// Implement optimistic locking
	thing.UpdatedAt = time.Now()
	thing.Version++
	
	query := `
		UPDATE things 
		SET name = $1, thing_attributes = $2, version = $3, updated_at = $4
		WHERE id = $5 AND tenant_id = $6 AND version = $7 AND deleted_at IS NULL
		RETURNING *`
	
	var updated domain.Thing
	err = conn.GetContext(ctx, &updated, query,
		thing.Name, thing.Attributes, thing.Version, thing.UpdatedAt,
		thing.ID, tenant.TenantID, thing.Version-1,
	)
	
	if err != nil {
		if err == sql.ErrNoRows {
			return nil, fmt.Errorf("thing not found or version conflict")
		}
		r.logger.Error("Failed to update thing",
			zap.String("id", thing.ID.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, err
	}
	
	return &updated, nil
}

func (r *thingRepository) Delete(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) error {
	conn, err := r.db.GetConnection(tenant.SchemaName)
	if err != nil {
		return fmt.Errorf("failed to get database connection: %w", err)
	}
	
	// Soft delete implementation
	query := `
		UPDATE things 
		SET deleted_at = NOW(), updated_at = NOW()
		WHERE id = $1 AND tenant_id = $2 AND deleted_at IS NULL`
	
	result, err := conn.ExecContext(ctx, query, id, tenant.TenantID)
	if err != nil {
		r.logger.Error("Failed to delete thing",
			zap.String("id", id.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return err
	}
	
	rowsAffected, err := result.RowsAffected()
	if err != nil {
		return err
	}
	
	if rowsAffected == 0 {
		return fmt.Errorf("thing not found")
	}
	
	return nil
}

func (r *thingRepository) List(ctx context.Context, tenant *domain.TenantContext, filter *domain.ThingFilter, pagination *domain.PaginationInput) (*domain.ThingConnection, error) {
	conn, err := r.db.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}
	
	// Build WHERE clause based on filter
	whereClause, args := r.buildWhereClause(tenant.TenantID, filter)
	
	// Build pagination
	limit := 20 // default
	if pagination != nil && pagination.First != nil {
		limit = *pagination.First
	}
	
	query := fmt.Sprintf(`
		SELECT * FROM things 
		%s
		ORDER BY created_at DESC
		LIMIT $%d`, whereClause, len(args)+1)
	
	args = append(args, limit+1) // Get one extra to check if there's a next page
	
	var things []*domain.Thing
	err = conn.SelectContext(ctx, &things, query, args...)
	if err != nil {
		r.logger.Error("Failed to list things",
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, err
	}
	
	// Build connection response
	hasNextPage := len(things) > limit
	if hasNextPage {
		things = things[:limit]
	}
	
	edges := make([]*domain.ThingEdge, len(things))
	for i, thing := range things {
		edges[i] = &domain.ThingEdge{
			Node:   thing,
			Cursor: thing.ID.String(),
		}
	}
	
	pageInfo := &domain.PageInfo{
		HasNextPage: hasNextPage,
	}
	
	if len(edges) > 0 {
		pageInfo.StartCursor = &edges[0].Cursor
		pageInfo.EndCursor = &edges[len(edges)-1].Cursor
	}
	
	return &domain.ThingConnection{
		Edges:    edges,
		PageInfo: pageInfo,
	}, nil
}

func (r *thingRepository) buildWhereClause(tenantID uuid.UUID, filter *domain.ThingFilter) (string, []interface{}) {
	conditions := []string{"tenant_id = $1", "deleted_at IS NULL"}
	args := []interface{}{tenantID}
	argIndex := 2
	
	if filter == nil {
		return fmt.Sprintf("WHERE %s", strings.Join(conditions, " AND ")), args
	}
	
	if filter.ThingClassID != nil {
		conditions = append(conditions, fmt.Sprintf("thing_class_id = $%d", argIndex))
		args = append(args, *filter.ThingClassID)
		argIndex++
	}
	
	if filter.Name != nil {
		conditions = append(conditions, fmt.Sprintf("name ILIKE $%d", argIndex))
		args = append(args, "%"+*filter.Name+"%")
		argIndex++
	}
	
	if filter.CreatedAfter != nil {
		conditions = append(conditions, fmt.Sprintf("created_at > $%d", argIndex))
		args = append(args, *filter.CreatedAfter)
		argIndex++
	}
	
	if filter.CreatedBy != nil {
		conditions = append(conditions, fmt.Sprintf("created_by = $%d", argIndex))
		args = append(args, *filter.CreatedBy)
		argIndex++
	}
	
	if len(filter.Attributes) > 0 {
		conditions = append(conditions, fmt.Sprintf("thing_attributes @> $%d", argIndex))
		args = append(args, filter.Attributes)
		argIndex++
	}
	
	return fmt.Sprintf("WHERE %s", strings.Join(conditions, " AND ")), args
}

func (r *thingRepository) ExistsByName(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID, name string) (bool, error) {
	conn, err := r.db.GetConnection(tenant.SchemaName)
	if err != nil {
		return false, fmt.Errorf("failed to get database connection: %w", err)
	}
	
	query := `
		SELECT EXISTS(
			SELECT 1 FROM things 
			WHERE thing_class_id = $1 AND name = $2 AND tenant_id = $3 AND deleted_at IS NULL
		)`
	
	var exists bool
	err = conn.GetContext(ctx, &exists, query, thingClassID, name, tenant.TenantID)
	if err != nil {
		r.logger.Error("Failed to check thing name existence",
			zap.String("name", name),
			zap.String("thing_class_id", thingClassID.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return false, err
	}
	
	return exists, nil
}

func (r *thingRepository) GetRelationships(ctx context.Context, tenant *domain.TenantContext, thingID uuid.UUID) ([]*domain.ThingRelationship, error) {
	conn, err := r.db.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}
	
	query := `
		SELECT * FROM thing_relationships 
		WHERE (source_thing_id = $1 OR target_thing_id = $1) 
		AND tenant_id = $2
		ORDER BY created_at DESC`
	
	var relationships []*domain.ThingRelationship
	err = conn.SelectContext(ctx, &relationships, query, thingID, tenant.TenantID)
	if err != nil {
		r.logger.Error("Failed to get thing relationships",
			zap.String("thing_id", thingID.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, err
	}
	
	return relationships, nil
}

func (r *thingRepository) GetThingsByThingClassIDs(ctx context.Context, tenant *domain.TenantContext, thingClassIDs []uuid.UUID) (map[uuid.UUID][]*domain.Thing, error) {
	if len(thingClassIDs) == 0 {
		return make(map[uuid.UUID][]*domain.Thing), nil
	}
	
	conn, err := r.db.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}
	
	query := `
		SELECT * FROM things 
		WHERE thing_class_id = ANY($1) AND tenant_id = $2 AND deleted_at IS NULL
		ORDER BY created_at DESC`
	
	var things []*domain.Thing
	err = conn.SelectContext(ctx, &things, query, pq.Array(thingClassIDs), tenant.TenantID)
	if err != nil {
		r.logger.Error("Failed to get things by thing class IDs",
			zap.Int("count", len(thingClassIDs)),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, err
	}
	
	// Group by thing class ID
	result := make(map[uuid.UUID][]*domain.Thing)
	for _, thing := range things {
		result[thing.ThingClassID] = append(result[thing.ThingClassID], thing)
	}
	
	return result, nil
}
```

## DataLoader Implementation

### Thing DataLoader

```go
// internal/dataloader/thing.go
package dataloader

import (
	"context"
	"time"
	
	"github.com/company/udm-backend/internal/domain"
	"github.com/company/udm-backend/internal/repository/interfaces"
	"github.com/graph-gophers/dataloader/v7"
	"github.com/google/uuid"
	"go.uber.org/zap"
)

type ThingLoader struct {
	loader         *dataloader.Loader[uuid.UUID, *domain.Thing]
	repo           interfaces.ThingRepository
	logger         *zap.Logger
	tenantContext  *domain.TenantContext
}

func NewThingLoader(repo interfaces.ThingRepository, tenantContext *domain.TenantContext, logger *zap.Logger) *ThingLoader {
	tl := &ThingLoader{
		repo:          repo,
		tenantContext: tenantContext,
		logger:        logger,
	}
	
	tl.loader = dataloader.NewBatchedLoader(tl.batchFunction, dataloader.WithWait[uuid.UUID, *domain.Thing](16*time.Millisecond))
	
	return tl
}

func (tl *ThingLoader) Load(ctx context.Context, id uuid.UUID) (*domain.Thing, error) {
	return tl.loader.Load(ctx, id)()
}

func (tl *ThingLoader) LoadMany(ctx context.Context, ids []uuid.UUID) ([]*domain.Thing, []error) {
	return tl.loader.LoadMany(ctx, ids)()
}

func (tl *ThingLoader) batchFunction(ctx context.Context, ids []uuid.UUID) []*dataloader.Result[*domain.Thing] {
	// Create a map to store the results
	thingMap := make(map[uuid.UUID]*domain.Thing)
	
	// Fetch things from repository
	things, err := tl.repo.GetByIDs(ctx, tl.tenantContext, ids)
	if err != nil {
		tl.logger.Error("Failed to batch load things",
			zap.Int("count", len(ids)),
			zap.Error(err),
		)
		// Return errors for all requested IDs
		results := make([]*dataloader.Result[*domain.Thing], len(ids))
		for i := range ids {
			results[i] = &dataloader.Result[*domain.Thing]{Error: err}
		}
		return results
	}
	
	// Build the map from results
	for _, thing := range things {
		thingMap[thing.ID] = thing
	}
	
	// Build results in the same order as requested IDs
	results := make([]*dataloader.Result[*domain.Thing], len(ids))
	for i, id := range ids {
		if thing, ok := thingMap[id]; ok {
			results[i] = &dataloader.Result[*domain.Thing]{Data: thing}
		} else {
			results[i] = &dataloader.Result[*domain.Thing]{Data: nil}
		}
	}
	
	return results
}

// ThingRelationshipLoader for loading relationships
type ThingRelationshipLoader struct {
	loader        *dataloader.Loader[uuid.UUID, []*domain.ThingRelationship]
	repo          interfaces.ThingRepository
	logger        *zap.Logger
	tenantContext *domain.TenantContext
}

func NewThingRelationshipLoader(repo interfaces.ThingRepository, tenantContext *domain.TenantContext, logger *zap.Logger) *ThingRelationshipLoader {
	trl := &ThingRelationshipLoader{
		repo:          repo,
		tenantContext: tenantContext,
		logger:        logger,
	}
	
	trl.loader = dataloader.NewBatchedLoader(trl.batchRelationships, dataloader.WithWait[uuid.UUID, []*domain.ThingRelationship](16*time.Millisecond))
	
	return trl
}

func (trl *ThingRelationshipLoader) Load(ctx context.Context, thingID uuid.UUID) ([]*domain.ThingRelationship, error) {
	return trl.loader.Load(ctx, thingID)()
}

func (trl *ThingRelationshipLoader) batchRelationships(ctx context.Context, thingIDs []uuid.UUID) []*dataloader.Result[[]*domain.ThingRelationship] {
	results := make([]*dataloader.Result[[]*domain.ThingRelationship], len(thingIDs))
	
	// For relationships, we'll load individually since the query pattern is different
	// In a more optimized implementation, you could batch these as well
	for i, thingID := range thingIDs {
		relationships, err := trl.repo.GetRelationships(ctx, trl.tenantContext, thingID)
		if err != nil {
			trl.logger.Error("Failed to load thing relationships",
				zap.String("thing_id", thingID.String()),
				zap.Error(err),
			)
			results[i] = &dataloader.Result[[]*domain.ThingRelationship]{Error: err}
		} else {
			results[i] = &dataloader.Result[[]*domain.ThingRelationship]{Data: relationships}
		}
	}
	
	return results
}
```

## Service Layer Implementation

### Thing Business Logic Service

```go
// internal/service/thing.go
package service

import (
	"context"
	"fmt"
	
	"github.com/company/udm-backend/internal/domain"
	"github.com/company/udm-backend/internal/repository/interfaces"
	"github.com/go-playground/validator/v10"
	"github.com/google/uuid"
	"go.uber.org/zap"
)

type ThingService struct {
	repo              interfaces.ThingRepository
	thingClassRepo    interfaces.ThingClassRepository
	validator         *validator.Validate
	logger            *zap.Logger
	cache             ThingCache
}

type ThingCache interface {
	Set(ctx context.Context, key string, value interface{}) error
	Get(ctx context.Context, key string, dest interface{}) error
	Delete(ctx context.Context, key string) error
	InvalidatePattern(ctx context.Context, pattern string) error
}

func NewThingService(repo interfaces.ThingRepository, thingClassRepo interfaces.ThingClassRepository, cache ThingCache, logger *zap.Logger) *ThingService {
	return &ThingService{
		repo:           repo,
		thingClassRepo: thingClassRepo,
		validator:      validator.New(),
		logger:         logger,
		cache:          cache,
	}
}

func (s *ThingService) CreateThing(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateThingInput) (*domain.Thing, error) {
	// Validate input
	if err := s.validateCreateInput(input); err != nil {
		return nil, fmt.Errorf("validation failed: %w", err)
	}
	
	// Get and validate thing class
	thingClass, err := s.thingClassRepo.GetByID(ctx, tenant, input.ThingClassID)
	if err != nil {
		return nil, fmt.Errorf("failed to get thing class: %w", err)
	}
	
	if thingClass == nil {
		return nil, fmt.Errorf("thing class not found")
	}
	
	// Check if name already exists within this thing class
	exists, err := s.repo.ExistsByName(ctx, tenant, input.ThingClassID, input.Name)
	if err != nil {
		return nil, fmt.Errorf("failed to check thing name existence: %w", err)
	}
	
	if exists {
		return nil, fmt.Errorf("thing with name '%s' already exists in this thing class", input.Name)
	}
	
	// Create domain object
	thing := &domain.Thing{
		ID:           uuid.New(),
		ThingClassID: input.ThingClassID,
		Name:         input.Name,
		Attributes:   input.Attributes,
		CreatedBy:    tenant.UserID,
	}
	
	// Validate attributes against thing class rules
	if err := thing.ValidateAttributes(thingClass); err != nil {
		return nil, fmt.Errorf("attribute validation failed: %w", err)
	}
	
	// Create in repository
	created, err := s.repo.Create(ctx, tenant, thing)
	if err != nil {
		s.logger.Error("Failed to create thing",
			zap.String("name", input.Name),
			zap.String("thing_class_id", input.ThingClassID.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to create thing: %w", err)
	}
	
	// Cache the created thing
	if err := s.cache.Set(ctx, created.CacheKey(), created); err != nil {
		s.logger.Warn("Failed to cache created thing", zap.Error(err))
	}
	
	s.logger.Info("Successfully created thing",
		zap.String("id", created.ID.String()),
		zap.String("name", created.Name),
		zap.String("thing_class_id", created.ThingClassID.String()),
		zap.String("tenant_schema", tenant.SchemaName),
	)
	
	return created, nil
}

func (s *ThingService) GetThing(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.Thing, error) {
	// Try to get from cache first
	cacheKey := (&domain.Thing{ID: id}).CacheKey()
	var cachedThing domain.Thing
	if err := s.cache.Get(ctx, cacheKey, &cachedThing); err == nil {
		return &cachedThing, nil
	}
	
	// Get from repository
	thing, err := s.repo.GetByID(ctx, tenant, id)
	if err != nil {
		s.logger.Error("Failed to get thing",
			zap.String("id", id.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to get thing: %w", err)
	}
	
	if thing == nil {
		return nil, fmt.Errorf("thing not found")
	}
	
	// Cache the result
	if err := s.cache.Set(ctx, cacheKey, thing); err != nil {
		s.logger.Warn("Failed to cache thing", zap.Error(err))
	}
	
	return thing, nil
}

func (s *ThingService) UpdateThing(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID, input *domain.UpdateThingInput) (*domain.Thing, error) {
	// Get current thing
	current, err := s.GetThing(ctx, tenant, id)
	if err != nil {
		return nil, err
	}
	
	// Validate input
	if err := s.validateUpdateInput(input); err != nil {
		return nil, fmt.Errorf("validation failed: %w", err)
	}
	
	// Apply updates
	updated := *current
	if input.Name != nil {
		updated.Name = *input.Name
	}
	if len(input.Attributes) > 0 {
		updated.Attributes = input.Attributes
	}
	
	// Get thing class for attribute validation
	thingClass, err := s.thingClassRepo.GetByID(ctx, tenant, updated.ThingClassID)
	if err != nil {
		return nil, fmt.Errorf("failed to get thing class: %w", err)
	}
	
	// Validate attributes against thing class rules
	if err := updated.ValidateAttributes(thingClass); err != nil {
		return nil, fmt.Errorf("attribute validation failed: %w", err)
	}
	
	// Update in repository
	result, err := s.repo.Update(ctx, tenant, &updated)
	if err != nil {
		s.logger.Error("Failed to update thing",
			zap.String("id", id.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to update thing: %w", err)
	}
	
	// Invalidate cache
	cacheKey := result.CacheKey()
	if err := s.cache.Delete(ctx, cacheKey); err != nil {
		s.logger.Warn("Failed to invalidate thing cache", zap.Error(err))
	}
	
	s.logger.Info("Successfully updated thing",
		zap.String("id", result.ID.String()),
		zap.String("name", result.Name),
		zap.String("tenant_schema", tenant.SchemaName),
	)
	
	return result, nil
}

func (s *ThingService) DeleteThing(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) error {
	// Check if thing exists
	thing, err := s.GetThing(ctx, tenant, id)
	if err != nil {
		return err
	}
	
	// Delete from repository (soft delete)
	if err := s.repo.Delete(ctx, tenant, id); err != nil {
		s.logger.Error("Failed to delete thing",
			zap.String("id", id.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return fmt.Errorf("failed to delete thing: %w", err)
	}
	
	// Invalidate cache
	cacheKey := thing.CacheKey()
	if err := s.cache.Delete(ctx, cacheKey); err != nil {
		s.logger.Warn("Failed to invalidate thing cache", zap.Error(err))
	}
	
	// Invalidate related caches
	relationshipsCacheKey := thing.RelationshipsCacheKey()
	if err := s.cache.Delete(ctx, relationshipsCacheKey); err != nil {
		s.logger.Warn("Failed to invalidate relationships cache", zap.Error(err))
	}
	
	s.logger.Info("Successfully deleted thing",
		zap.String("id", id.String()),
		zap.String("name", thing.Name),
		zap.String("tenant_schema", tenant.SchemaName),
	)
	
	return nil
}

func (s *ThingService) ListThings(ctx context.Context, tenant *domain.TenantContext, filter *domain.ThingFilter, pagination *domain.PaginationInput) (*domain.ThingConnection, error) {
	connection, err := s.repo.List(ctx, tenant, filter, pagination)
	if err != nil {
		s.logger.Error("Failed to list things",
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to list things: %w", err)
	}
	
	// Cache individual things
	for _, edge := range connection.Edges {
		cacheKey := edge.Node.CacheKey()
		if err := s.cache.Set(ctx, cacheKey, edge.Node); err != nil {
			s.logger.Warn("Failed to cache thing in list", zap.Error(err))
		}
	}
	
	return connection, nil
}

func (s *ThingService) validateCreateInput(input *domain.CreateThingInput) error {
	if input.Name == "" {
		return fmt.Errorf("name is required")
	}
	
	if len(input.Name) > 255 {
		return fmt.Errorf("name must be 255 characters or less")
	}
	
	if input.ThingClassID == uuid.Nil {
		return fmt.Errorf("thing class ID is required")
	}
	
	return nil
}

func (s *ThingService) validateUpdateInput(input *domain.UpdateThingInput) error {
	if input.Name != nil {
		if *input.Name == "" {
			return fmt.Errorf("name cannot be empty")
		}
		if len(*input.Name) > 255 {
			return fmt.Errorf("name must be 255 characters or less")
		}
	}
	
	return nil
}
```

## GraphQL Resolver Implementation

### Thing Resolvers

```go
// internal/graph/resolvers/thing.go
package resolvers

import (
	"context"
	"fmt"
	
	"github.com/company/udm-backend/internal/dataloader"
	"github.com/company/udm-backend/internal/domain"
	"github.com/company/udm-backend/internal/graph/generated"
	"github.com/google/uuid"
)

// Thing resolver
func (r *Resolver) Thing() generated.ThingResolver {
	return &thingResolver{r}
}

type thingResolver struct{ *Resolver }

func (r *thingResolver) ThingClass(ctx context.Context, obj *domain.Thing) (*domain.ThingClass, error) {
	// Get tenant context
	tenantCtx, err := getTenantFromContext(ctx)
	if err != nil {
		return nil, err
	}
	
	// Use DataLoader to batch load thing classes
	loader := dataloader.GetThingClassLoader(ctx)
	if loader == nil {
		// Fallback to direct service call
		return r.ThingClassService.GetThingClass(ctx, tenantCtx, obj.ThingClassID)
	}
	
	return loader.Load(ctx, obj.ThingClassID)
}

func (r *thingResolver) Relationships(ctx context.Context, obj *domain.Thing) ([]*domain.ThingRelationship, error) {
	// Get tenant context
	tenantCtx, err := getTenantFromContext(ctx)
	if err != nil {
		return nil, err
	}
	
	// Use DataLoader to batch load relationships
	loader := dataloader.GetThingRelationshipLoader(ctx)
	if loader == nil {
		// Fallback to direct repository call
		return r.ThingService.GetThingRelationships(ctx, tenantCtx, obj.ID)
	}
	
	return loader.Load(ctx, obj.ID)
}

// Query resolvers
func (r *queryResolver) Thing(ctx context.Context, id string) (*domain.Thing, error) {
	// Get tenant context
	tenantCtx, err := getTenantFromContext(ctx)
	if err != nil {
		return nil, err
	}
	
	// Parse UUID
	thingID, err := uuid.Parse(id)
	if err != nil {
		return nil, fmt.Errorf("invalid thing ID: %w", err)
	}
	
	return r.ThingService.GetThing(ctx, tenantCtx, thingID)
}

func (r *queryResolver) Things(ctx context.Context, first *int, after *string, filter *domain.ThingFilter) (*domain.ThingConnection, error) {
	// Get tenant context
	tenantCtx, err := getTenantFromContext(ctx)
	if err != nil {
		return nil, err
	}
	
	// Build pagination input
	pagination := &domain.PaginationInput{
		First: first,
		After: after,
	}
	
	return r.ThingService.ListThings(ctx, tenantCtx, filter, pagination)
}

// Mutation resolvers
func (r *mutationResolver) CreateThing(ctx context.Context, input domain.CreateThingInput) (*domain.ThingPayload, error) {
	// Get tenant context
	tenantCtx, err := getTenantFromContext(ctx)
	if err != nil {
		return &domain.ThingPayload{
			Errors: []*domain.UserError{{
				Message: "Authentication required",
				Code:    "UNAUTHENTICATED",
			}},
		}, nil
	}
	
	// Create thing
	thing, err := r.ThingService.CreateThing(ctx, tenantCtx, &input)
	if err != nil {
		return &domain.ThingPayload{
			Errors: []*domain.UserError{{
				Message: err.Error(),
				Code:    "CREATE_FAILED",
			}},
		}, nil
	}
	
	return &domain.ThingPayload{
		Thing:  thing,
		Errors: []*domain.UserError{},
	}, nil
}

func (r *mutationResolver) UpdateThing(ctx context.Context, id string, input domain.UpdateThingInput) (*domain.ThingPayload, error) {
	// Get tenant context
	tenantCtx, err := getTenantFromContext(ctx)
	if err != nil {
		return &domain.ThingPayload{
			Errors: []*domain.UserError{{
				Message: "Authentication required",
				Code:    "UNAUTHENTICATED",
			}},
		}, nil
	}
	
	// Parse UUID
	thingID, err := uuid.Parse(id)
	if err != nil {
		return &domain.ThingPayload{
			Errors: []*domain.UserError{{
				Message: "Invalid thing ID",
				Code:    "INVALID_ID",
			}},
		}, nil
	}
	
	// Update thing
	thing, err := r.ThingService.UpdateThing(ctx, tenantCtx, thingID, &input)
	if err != nil {
		return &domain.ThingPayload{
			Errors: []*domain.UserError{{
				Message: err.Error(),
				Code:    "UPDATE_FAILED",
			}},
		}, nil
	}
	
	return &domain.ThingPayload{
		Thing:  thing,
		Errors: []*domain.UserError{},
	}, nil
}

func (r *mutationResolver) DeleteThing(ctx context.Context, id string) (*domain.DeleteThingPayload, error) {
	// Get tenant context
	tenantCtx, err := getTenantFromContext(ctx)
	if err != nil {
		return &domain.DeleteThingPayload{
			Success: false,
			Errors: []*domain.UserError{{
				Message: "Authentication required",
				Code:    "UNAUTHENTICATED",
			}},
		}, nil
	}
	
	// Parse UUID
	thingID, err := uuid.Parse(id)
	if err != nil {
		return &domain.DeleteThingPayload{
			Success: false,
			Errors: []*domain.UserError{{
				Message: "Invalid thing ID",
				Code:    "INVALID_ID",
			}},
		}, nil
	}
	
	// Delete thing
	err = r.ThingService.DeleteThing(ctx, tenantCtx, thingID)
	if err != nil {
		return &domain.DeleteThingPayload{
			Success: false,
			Errors: []*domain.UserError{{
				Message: err.Error(),
				Code:    "DELETE_FAILED",
			}},
		}, nil
	}
	
	return &domain.DeleteThingPayload{
		Success: true,
		Errors:  []*domain.UserError{},
	}, nil
}
```

## Database Migration

### Thing Entity Migration

```sql
-- migrations/002_create_things.up.sql

-- Create things table (no attributes column)
CREATE TABLE things (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thing_class_id UUID NOT NULL REFERENCES thing_classes(id) ON DELETE RESTRICT,
    tenant_id UUID NOT NULL,
    name VARCHAR(255) NOT NULL,
    -- Common entity fields
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    version INTEGER DEFAULT 1,
    
    -- Constraints
    CONSTRAINT things_name_not_empty CHECK (length(name) > 0),
    CONSTRAINT things_version_positive CHECK (version > 0)
);

-- Create values table for attribute data
CREATE TABLE values (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thing_id UUID NOT NULL REFERENCES things(id) ON DELETE CASCADE,
    attribute_id UUID NOT NULL REFERENCES attributes(id) ON DELETE RESTRICT,
    value_data JSONB NOT NULL,
    -- Common entity fields
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    version INTEGER DEFAULT 1,
    
    CONSTRAINT unique_thing_attribute_value UNIQUE(thing_id, attribute_id)
);

-- Create thing_relationships table
CREATE TABLE thing_relationships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_thing_id UUID NOT NULL REFERENCES things(id) ON DELETE CASCADE,
    target_thing_id UUID NOT NULL REFERENCES things(id) ON DELETE CASCADE,
    tenant_id UUID NOT NULL,
    relationship_type VARCHAR(100) NOT NULL,
    metadata JSONB DEFAULT '{}',
    -- Common entity fields
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID NOT NULL,
    is_active BOOLEAN DEFAULT TRUE,
    version INTEGER DEFAULT 1,
    
    -- Constraints
    CONSTRAINT things_relationships_no_self_reference CHECK (source_thing_id != target_thing_id),
    CONSTRAINT things_relationships_type_not_empty CHECK (length(relationship_type) > 0)
);

-- Performance indexes for things
CREATE INDEX idx_things_thing_class_id ON things(thing_class_id) WHERE is_active = TRUE;
CREATE INDEX idx_things_tenant_id ON things(tenant_id) WHERE is_active = TRUE;
CREATE INDEX idx_things_name ON things(name) WHERE is_active = TRUE;
CREATE INDEX idx_things_created_at ON things(created_at) WHERE is_active = TRUE;
CREATE INDEX idx_things_created_by ON things(created_by) WHERE is_active = TRUE;
CREATE INDEX idx_things_active ON things(is_active);

-- Indexes for values table
CREATE INDEX idx_values_thing_id ON values(thing_id) WHERE is_active = TRUE;
CREATE INDEX idx_values_attribute_id ON values(attribute_id) WHERE is_active = TRUE;
CREATE INDEX idx_values_data_gin ON values USING GIN(value_data) WHERE is_active = TRUE;
CREATE INDEX idx_values_active ON values(is_active);

-- Composite indexes for common query patterns
CREATE INDEX idx_things_class_tenant ON things(thing_class_id, tenant_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_things_tenant_name ON things(tenant_id, name) WHERE deleted_at IS NULL;

-- Indexes for thing_relationships
CREATE INDEX idx_thing_relationships_source ON thing_relationships(source_thing_id);
CREATE INDEX idx_thing_relationships_target ON thing_relationships(target_thing_id);
CREATE INDEX idx_thing_relationships_tenant ON thing_relationships(tenant_id);
CREATE INDEX idx_thing_relationships_type ON thing_relationships(relationship_type);

-- Composite indexes for relationships
CREATE INDEX idx_thing_relationships_source_tenant ON thing_relationships(source_thing_id, tenant_id);
CREATE INDEX idx_thing_relationships_target_tenant ON thing_relationships(target_thing_id, tenant_id);

-- Unique constraint: thing name must be unique within thing class and tenant (for active things)
CREATE UNIQUE INDEX idx_things_unique_name_per_class_tenant 
ON things(thing_class_id, tenant_id, name) 
WHERE deleted_at IS NULL;

-- Row Level Security (RLS) policies
ALTER TABLE things ENABLE ROW LEVEL SECURITY;
ALTER TABLE thing_relationships ENABLE ROW LEVEL SECURITY;

-- RLS Policy for things - users can only access things in their tenant
CREATE POLICY things_tenant_isolation ON things
    FOR ALL
    TO app_role
    USING (tenant_id = current_setting('app.current_tenant_id', true)::uuid);

-- RLS Policy for thing_relationships - users can only access relationships in their tenant  
CREATE POLICY thing_relationships_tenant_isolation ON thing_relationships
    FOR ALL
    TO app_role
    USING (tenant_id = current_setting('app.current_tenant_id', true)::uuid);

-- Trigger to update updated_at timestamp
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER update_things_updated_at 
    BEFORE UPDATE ON things 
    FOR EACH ROW 
    EXECUTE FUNCTION update_updated_at_column();

-- Function to validate JSON attributes against thing class rules
CREATE OR REPLACE FUNCTION validate_thing_attributes()
RETURNS TRIGGER AS $$
DECLARE
    validation_rules JSONB;
    attr_key TEXT;
    attr_value JSONB;
    rule_config JSONB;
BEGIN
    -- Get validation rules from thing class
    SELECT tc.validation_rules INTO validation_rules
    FROM thing_classes tc
    WHERE tc.id = NEW.thing_class_id AND tc.tenant_id = NEW.tenant_id;
    
    -- If no validation rules, allow anything
    IF validation_rules IS NULL OR validation_rules = '{}'::jsonb THEN
        RETURN NEW;
    END IF;
    
    -- Validate each attribute against rules
    FOR attr_key, rule_config IN SELECT * FROM jsonb_each(validation_rules) LOOP
        -- Check if required attribute is present
        IF (rule_config->>'required')::boolean = true THEN
            IF NOT (NEW.thing_attributes ? attr_key) THEN
                RAISE EXCEPTION 'Required attribute "%" is missing', attr_key;
            END IF;
        END IF;
        
        -- Additional validation logic would go here
        -- (type checking, value constraints, etc.)
    END LOOP;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER validate_thing_attributes_trigger
    BEFORE INSERT OR UPDATE ON things
    FOR EACH ROW
    EXECUTE FUNCTION validate_thing_attributes();

-- Add comments for documentation
COMMENT ON TABLE things IS 'Stores thing entities with dynamic attributes in JSONB format';
COMMENT ON TABLE thing_relationships IS 'Stores relationships between thing entities';
COMMENT ON COLUMN things.thing_attributes IS 'Dynamic attributes stored as JSONB, validated against thing class rules';
COMMENT ON COLUMN things.version IS 'Version number for optimistic locking';
COMMENT ON COLUMN thing_relationships.metadata IS 'Additional metadata for the relationship';
```

```sql
-- migrations/002_create_things.down.sql

-- Drop triggers
DROP TRIGGER IF EXISTS update_things_updated_at ON things;
DROP TRIGGER IF EXISTS validate_thing_attributes_trigger ON things;

-- Drop functions
DROP FUNCTION IF EXISTS update_updated_at_column();
DROP FUNCTION IF EXISTS validate_thing_attributes();

-- Drop RLS policies
DROP POLICY IF EXISTS things_tenant_isolation ON things;
DROP POLICY IF EXISTS thing_relationships_tenant_isolation ON thing_relationships;

-- Disable RLS
ALTER TABLE things DISABLE ROW LEVEL SECURITY;
ALTER TABLE thing_relationships DISABLE ROW LEVEL SECURITY;

-- Drop indexes
DROP INDEX IF EXISTS idx_things_thing_class_id;
DROP INDEX IF EXISTS idx_things_tenant_id;
DROP INDEX IF EXISTS idx_things_name;
DROP INDEX IF EXISTS idx_things_created_at;
DROP INDEX IF EXISTS idx_things_created_by;
DROP INDEX IF EXISTS idx_things_attributes;
DROP INDEX IF EXISTS idx_things_class_tenant;
DROP INDEX IF EXISTS idx_things_tenant_name;
DROP INDEX IF EXISTS idx_things_unique_name_per_class_tenant;

DROP INDEX IF EXISTS idx_thing_relationships_source;
DROP INDEX IF EXISTS idx_thing_relationships_target;
DROP INDEX IF EXISTS idx_thing_relationships_tenant;
DROP INDEX IF EXISTS idx_thing_relationships_type;
DROP INDEX IF EXISTS idx_thing_relationships_source_tenant;
DROP INDEX IF EXISTS idx_thing_relationships_target_tenant;

-- Drop tables
DROP TABLE IF EXISTS thing_relationships;
DROP TABLE IF EXISTS things;
```

## Thing-Specific Caching Implementation

### Redis Cache for Thing Entities

```go
// internal/cache/thing.go
package cache

import (
	"context"
	"fmt"
	"time"
	
	"github.com/company/udm-backend/internal/domain"
	"github.com/google/uuid"
	"go.uber.org/zap"
)

type ThingCache struct {
	redis  *RedisCache
	logger *zap.Logger
}

func NewThingCache(redis *RedisCache, logger *zap.Logger) *ThingCache {
	return &ThingCache{
		redis:  redis,
		logger: logger,
	}
}

func (c *ThingCache) Set(ctx context.Context, key string, value interface{}) error {
	return c.redis.Set(ctx, "things", key, value)
}

func (c *ThingCache) Get(ctx context.Context, key string, dest interface{}) error {
	return c.redis.Get(ctx, "things", key, dest)
}

func (c *ThingCache) Delete(ctx context.Context, key string) error {
	return c.redis.Delete(ctx, "things", key)
}

func (c *ThingCache) InvalidatePattern(ctx context.Context, pattern string) error {
	return c.redis.InvalidatePattern(ctx, "things", pattern)
}

// Thing-specific cache methods
func (c *ThingCache) CacheThing(ctx context.Context, tenantSchema string, thing *domain.Thing) error {
	key := fmt.Sprintf("%s:thing:%s", tenantSchema, thing.ID)
	return c.Set(ctx, key, thing)
}

func (c *ThingCache) GetThing(ctx context.Context, tenantSchema string, id uuid.UUID) (*domain.Thing, error) {
	key := fmt.Sprintf("%s:thing:%s", tenantSchema, id)
	var thing domain.Thing
	err := c.Get(ctx, key, &thing)
	if err != nil {
		return nil, err
	}
	return &thing, nil
}

func (c *ThingCache) InvalidateThingCache(ctx context.Context, tenantSchema string, id uuid.UUID) error {
	key := fmt.Sprintf("%s:thing:%s", tenantSchema, id)
	return c.Delete(ctx, key)
}

func (c *ThingCache) CacheThingRelationships(ctx context.Context, tenantSchema string, thingID uuid.UUID, relationships []*domain.ThingRelationship) error {
	key := fmt.Sprintf("%s:thing_relationships:%s", tenantSchema, thingID)
	return c.Set(ctx, key, relationships)
}

func (c *ThingCache) GetThingRelationships(ctx context.Context, tenantSchema string, thingID uuid.UUID) ([]*domain.ThingRelationship, error) {
	key := fmt.Sprintf("%s:thing_relationships:%s", tenantSchema, thingID)
	var relationships []*domain.ThingRelationship
	err := c.Get(ctx, key, &relationships)
	if err != nil {
		return nil, err
	}
	return relationships, nil
}

func (c *ThingCache) InvalidateThingClassThings(ctx context.Context, tenantSchema string, thingClassID uuid.UUID) error {
	pattern := fmt.Sprintf("%s:things_by_class:%s:*", tenantSchema, thingClassID)
	return c.InvalidatePattern(ctx, pattern)
}
```

## Performance Monitoring and Optimization

### Query Performance Monitor

```go
// internal/monitoring/performance.go
package monitoring

import (
	"context"
	"time"
	
	"go.uber.org/zap"
)

type PerformanceMonitor struct {
	logger *zap.Logger
}

func NewPerformanceMonitor(logger *zap.Logger) *PerformanceMonitor {
	return &PerformanceMonitor{logger: logger}
}

func (pm *PerformanceMonitor) TrackQuery(ctx context.Context, operation string, fn func() error) error {
	start := time.Now()
	err := fn()
	duration := time.Since(start)
	
	pm.logger.Info("GraphQL operation completed",
		zap.String("operation", operation),
		zap.Duration("duration", duration),
		zap.Error(err),
	)
	
	// Alert on slow operations (>200ms for single-level hydration)
	if duration > 200*time.Millisecond {
		pm.logger.Warn("Slow GraphQL operation detected",
			zap.String("operation", operation),
			zap.Duration("duration", duration),
		)
	}
	
	return err
}

func (pm *PerformanceMonitor) TrackDataLoaderBatch(operation string, batchSize int, duration time.Duration) {
	pm.logger.Debug("DataLoader batch completed",
		zap.String("operation", operation),
		zap.Int("batch_size", batchSize),
		zap.Duration("duration", duration),
	)
}
```

## Application Bootstrap Updates

### Updated Main Application Entry Point

```go
// cmd/server/main.go (additions to existing main.go)

// Additional imports for Thing Entity
import (
	thingRepo "github.com/company/udm-backend/internal/repository/postgres"
	thingService "github.com/company/udm-backend/internal/service"
	"github.com/company/udm-backend/internal/cache"
	"github.com/company/udm-backend/internal/dataloader"
	"github.com/company/udm-backend/internal/monitoring"
)

func main() {
	// ... existing logger and config setup ...

	// Initialize database manager
	dbManager := database.NewManager(&cfg.Database, logger)
	defer dbManager.CloseAll()

	// Initialize Redis cache
	redisCache := cache.NewRedisCache(
		cfg.Redis.Addr,
		cfg.Redis.Password,
		cfg.Redis.DB,
		logger,
	)

	// Initialize Thing-specific cache
	thingCache := cache.NewThingCache(redisCache, logger)

	// Initialize performance monitor
	perfMonitor := monitoring.NewPerformanceMonitor(logger)

	// Initialize repositories
	thingClassRepo := postgres.NewThingClassRepository(dbManager, logger)
	thingRepo := thingRepo.NewThingRepository(dbManager, logger)

	// Initialize services
	thingClassService := service.NewThingClassService(thingClassRepo, logger)
	thingService := thingService.NewThingService(
		thingRepo, 
		thingClassRepo, 
		thingCache, 
		logger,
	)

	// Initialize GraphQL resolver with all services
	resolver := &resolvers.Resolver{
		ThingClassService:    thingClassService,
		ThingService:         thingService,
		PerformanceMonitor:   perfMonitor,
		Logger:              logger,
	}

	// Create GraphQL server with middleware
	srv := handler.NewDefaultServer(generated.NewExecutableSchema(generated.Config{
		Resolvers: resolver,
		Directives: generated.DirectiveRoot{
			Auth:     directives.AuthDirective,
			Validate: directives.ValidateDirective,
		},
	}))

	// Add DataLoader middleware
	srv.Use(&handler.RequestMiddleware{
		Func: func(ctx context.Context, next func(context.Context) []byte) []byte {
			// Get tenant context
			tenantCtx, _ := getTenantFromContext(ctx)
			if tenantCtx != nil {
				// Add DataLoaders to context
				ctx = dataloader.AddDataLoadersToContext(ctx, dataloader.Config{
					ThingRepo:           thingRepo,
					ThingClassRepo:      thingClassRepo,
					TenantContext:       tenantCtx,
					Logger:             logger,
				})
			}
			return next(ctx)
		},
	})

	// ... rest of existing setup ...
}
```

## DataLoader Context Management

### DataLoader Context Integration

```go
// internal/dataloader/context.go
package dataloader

import (
	"context"
	
	"github.com/company/udm-backend/internal/domain"
	"github.com/company/udm-backend/internal/repository/interfaces"
	"go.uber.org/zap"
)

type contextKey string

const (
	thingLoaderKey         contextKey = "thingLoader"
	thingClassLoaderKey    contextKey = "thingClassLoader"
	relationshipLoaderKey  contextKey = "relationshipLoader"
)

type Config struct {
	ThingRepo      interfaces.ThingRepository
	ThingClassRepo interfaces.ThingClassRepository
	TenantContext  *domain.TenantContext
	Logger         *zap.Logger
}

func AddDataLoadersToContext(ctx context.Context, config Config) context.Context {
	// Create DataLoaders
	thingLoader := NewThingLoader(config.ThingRepo, config.TenantContext, config.Logger)
	thingClassLoader := NewThingClassLoader(config.ThingClassRepo, config.TenantContext, config.Logger)
	relationshipLoader := NewThingRelationshipLoader(config.ThingRepo, config.TenantContext, config.Logger)
	
	// Add to context
	ctx = context.WithValue(ctx, thingLoaderKey, thingLoader)
	ctx = context.WithValue(ctx, thingClassLoaderKey, thingClassLoader)
	ctx = context.WithValue(ctx, relationshipLoaderKey, relationshipLoader)
	
	return ctx
}

func GetThingLoader(ctx context.Context) *ThingLoader {
	if loader, ok := ctx.Value(thingLoaderKey).(*ThingLoader); ok {
		return loader
	}
	return nil
}

func GetThingClassLoader(ctx context.Context) *ThingClassLoader {
	if loader, ok := ctx.Value(thingClassLoaderKey).(*ThingClassLoader); ok {
		return loader
	}
	return nil
}

func GetThingRelationshipLoader(ctx context.Context) *ThingRelationshipLoader {
	if loader, ok := ctx.Value(relationshipLoaderKey).(*ThingRelationshipLoader); ok {
		return loader
	}
	return nil
}
```

This comprehensive Go implementation provides all the patterns and structures needed to implement the Thing Entity using the same architectural patterns as the Thing Class, with additional features for:

- Dynamic JSON-B attributes with validation
- Multi-level relationship traversal  
- DataLoader-based N+1 query prevention
- Redis caching with TTL and invalidation
- Performance monitoring and alerting
- Row-level security for multi-tenant isolation
- Optimistic locking for concurrent updates
- Comprehensive database indexing for performance