# Attribute Entity - Go Implementation Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**Language:** Go + gqlgen + PostgreSQL  
**Entity:** Attribute - Property definitions for UDM system  

## Implementation Architecture Overview

The Attribute entity implementation follows a clean architecture pattern with clear separation of concerns across domain, service, repository, and GraphQL layers. This specification provides comprehensive Go implementation patterns optimized for the gqlgen GraphQL framework with PostgreSQL schema-per-tenant architecture.

## Project Structure

### Go Module Organization

```
github.com/company/udm-backend/
├── cmd/
│   └── server/
│       └── main.go                          # Application entry point
├── internal/
│   ├── domain/                              # Domain models and business entities  
│   │   ├── attribute.go                     # Attribute entity and types
│   │   ├── validation.go                    # Validation rule structures
│   │   └── errors.go                        # Domain-specific errors
│   ├── graph/                               # GraphQL layer (gqlgen)
│   │   ├── generated/                       # gqlgen generated code
│   │   ├── resolvers/                       # Custom resolver implementations
│   │   │   ├── attribute.go                 # Attribute resolvers
│   │   │   ├── mutation.go                  # Mutation resolvers
│   │   │   └── subscription.go              # Subscription resolvers
│   │   ├── directives/                      # Custom GraphQL directives
│   │   │   ├── auth.go                      # Authentication directive
│   │   │   └── validate.go                  # Validation directive
│   │   ├── schema.graphqls                  # GraphQL schema definition
│   │   └── gqlgen.yml                       # gqlgen configuration
│   ├── repository/                          # Data access layer
│   │   ├── interfaces/                      # Repository interfaces
│   │   │   └── attribute.go                 # Attribute repository interface
│   │   └── postgres/                        # PostgreSQL implementations
│   │       ├── attribute.go                 # Attribute repository implementation
│   │       └── queries.go                   # SQL query definitions
│   ├── service/                             # Business logic layer
│   │   ├── attribute.go                     # Attribute service implementation
│   │   └── validation/                      # Validation rule processors
│   │       ├── string.go                    # String validation rules
│   │       ├── integer.go                   # Integer validation rules
│   │       └── factory.go                   # Validation factory
│   ├── cache/                               # Caching layer
│   │   ├── redis.go                         # Redis cache implementation
│   │   └── memory.go                        # In-memory cache for hot data
│   ├── middleware/                          # HTTP middleware
│   │   ├── tenant.go                        # Multi-tenant routing
│   │   ├── auth.go                          # Authentication middleware
│   │   └── logging.go                       # Request logging
│   └── config/                              # Configuration management
│       └── config.go                        # Application configuration
├── migrations/                              # Database migrations
│   ├── 003_create_attributes.up.sql
│   └── 003_create_attributes.down.sql
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── scripts/                                 # Development scripts
│   ├── generate.sh                          # Code generation
│   └── test.sh                              # Test runner
├── go.mod
├── go.sum
└── README.md
```

## Domain Model Implementation

### Core Attribute Entity

```go
// internal/domain/attribute.go
package domain

import (
	"encoding/json"
	"fmt"
	"time"

	"github.com/google/uuid"
)

// Attribute represents a reusable property definition in the UDM system
type Attribute struct {
	BaseEntity
	Name            string          `json:"name" db:"name" validate:"required,min=1,max=255"`
	DataType        AttributeType   `json:"dataType" db:"data_type" validate:"required"`
	ValidationRules json.RawMessage `json:"validationRules" db:"validation_rules"`
	Description     *string         `json:"description" db:"description" validate:"omitempty,max=1000"`

	// Computed fields (loaded separately, not persisted)
	AssignmentCount *int                    `json:"assignmentCount,omitempty" db:"-"`
	Assignments     []*ThingClassAttribute `json:"assignments,omitempty" db:"-"`
	Evolution       []*AttributeEvolution  `json:"evolution,omitempty" db:"-"`
}

// AttributeType enum for supported data types
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

// String returns string representation of AttributeType
func (at AttributeType) String() string {
	return string(at)
}

// IsValid validates if the AttributeType is supported
func (at AttributeType) IsValid() bool {
	switch at {
	case AttributeTypeString, AttributeTypeInteger, AttributeTypeDecimal,
		AttributeTypeBoolean, AttributeTypeDate, AttributeTypeDateTime,
		AttributeTypeJSON, AttributeTypeReference:
		return true
	default:
		return false
	}
}

// GetValidAttributeTypes returns all valid attribute types
func GetValidAttributeTypes() []AttributeType {
	return []AttributeType{
		AttributeTypeString, AttributeTypeInteger, AttributeTypeDecimal,
		AttributeTypeBoolean, AttributeTypeDate, AttributeTypeDateTime,
		AttributeTypeJSON, AttributeTypeReference,
	}
}

// AttributeEvolution tracks schema evolution changes
type AttributeEvolution struct {
	BaseEntity
	AttributeID     uuid.UUID            `json:"attributeId" db:"attribute_id"`
	ChangeType      AttributeChangeType  `json:"changeType" db:"change_type"`
	OldValues       json.RawMessage      `json:"oldValues" db:"old_values"`
	NewValues       json.RawMessage      `json:"newValues" db:"new_values"`
	ImpactAnalysis  json.RawMessage      `json:"impactAnalysis" db:"impact_analysis"`
	SafetyLevel     EvolutionSafetyLevel `json:"safetyLevel" db:"safety_level"`
	AppliedAt       time.Time            `json:"appliedAt" db:"applied_at"`
	AppliedBy       uuid.UUID            `json:"appliedBy" db:"applied_by"`
	RollbackData    json.RawMessage      `json:"rollbackData" db:"rollback_data"`

	// Loaded relationships
	Attribute *Attribute `json:"attribute,omitempty" db:"-"`
}

type AttributeChangeType string

const (
	AttributeChangeValidationRulesRelaxed   AttributeChangeType = "VALIDATION_RULES_RELAXED"
	AttributeChangeValidationRulesTightened AttributeChangeType = "VALIDATION_RULES_TIGHTENED"
	AttributeChangeDescriptionChanged       AttributeChangeType = "DESCRIPTION_CHANGED"
	AttributeChangeNameChanged              AttributeChangeType = "NAME_CHANGED"
	AttributeChangeStatusChanged            AttributeChangeType = "STATUS_CHANGED"
)

type EvolutionSafetyLevel string

const (
	EvolutionSafetyLevelSafe     EvolutionSafetyLevel = "SAFE"
	EvolutionSafetyLevelWarning  EvolutionSafetyLevel = "WARNING"
	EvolutionSafetyLevelBreaking EvolutionSafetyLevel = "BREAKING"
)
```

### Input and Filter Types

```go
// Input types for GraphQL operations
type CreateAttributeInput struct {
	Name            string          `json:"name" validate:"required,min=1,max=255,alphanum_underscore"`
	DataType        AttributeType   `json:"dataType" validate:"required"`
	ValidationRules json.RawMessage `json:"validationRules,omitempty"`
	Description     *string         `json:"description,omitempty" validate:"omitempty,max=1000"`
}

type UpdateAttributeInput struct {
	Name            *string         `json:"name,omitempty" validate:"omitempty,min=1,max=255,alphanum_underscore"`
	ValidationRules json.RawMessage `json:"validationRules,omitempty"`
	Description     *string         `json:"description,omitempty" validate:"omitempty,max=1000"`
	// Note: DataType intentionally omitted - cannot be changed after creation
}

type AttributeFilter struct {
	Name                     *string        `json:"name,omitempty"`
	DataType                 *AttributeType `json:"dataType,omitempty"`
	IsActive                 *bool          `json:"isActive,omitempty"`
	CreatedBy                *uuid.UUID     `json:"createdBy,omitempty"`
	CreatedAfter             *time.Time     `json:"createdAfter,omitempty"`
	CreatedBefore            *time.Time     `json:"createdBefore,omitempty"`
	HasAssignments           *bool          `json:"hasAssignments,omitempty"`
	ValidationRulesContains  json.RawMessage `json:"validationRulesContains,omitempty"`
	NameContains             *string        `json:"nameContains,omitempty"`
}

type AttributeSort struct {
	Field     AttributeSortField `json:"field"`
	Direction SortDirection      `json:"direction"`
}

type AttributeSortField string

const (
	AttributeSortFieldName            AttributeSortField = "NAME"
	AttributeSortFieldDataType        AttributeSortField = "DATA_TYPE"
	AttributeSortFieldCreatedAt       AttributeSortField = "CREATED_AT"
	AttributeSortFieldUpdatedAt       AttributeSortField = "UPDATED_AT"
	AttributeSortFieldAssignmentCount AttributeSortField = "ASSIGNMENT_COUNT"
)

type SortDirection string

const (
	SortDirectionAsc  SortDirection = "ASC"
	SortDirectionDesc SortDirection = "DESC"
)

// Pagination types
type AttributeConnection struct {
	Edges      []*AttributeEdge `json:"edges"`
	PageInfo   *PageInfo        `json:"pageInfo"`
	TotalCount int              `json:"totalCount"`
}

type AttributeEdge struct {
	Node   *Attribute `json:"node"`
	Cursor string     `json:"cursor"`
}

// Statistics and reporting types
type AttributeStatistics struct {
	TotalCount                       int                `json:"totalCount"`
	ActiveCount                      int                `json:"activeCount"`
	CountByDataType                  []*DataTypeCount   `json:"countByDataType"`
	AverageAssignmentsPerAttribute   float64            `json:"averageAssignmentsPerAttribute"`
	MostUsedAttributes               []*Attribute       `json:"mostUsedAttributes"`
	LeastUsedAttributes              []*Attribute       `json:"leastUsedAttributes"`
	AttributesWithoutAssignments     int                `json:"attributesWithoutAssignments"`
	RecentlyCreated                  []*Attribute       `json:"recentlyCreated"`
}

type DataTypeCount struct {
	DataType   AttributeType `json:"dataType"`
	Count      int           `json:"count"`
	Percentage float64       `json:"percentage"`
}

// Impact analysis types
type AttributeImpactAnalysis struct {
	Attribute            *Attribute           `json:"attribute"`
	SafetyLevel          EvolutionSafetyLevel `json:"safetyLevel"`
	AffectedAssignments  int                  `json:"affectedAssignments"`
	AffectedThings       int                  `json:"affectedThings"`
	ImpactDetails        json.RawMessage      `json:"impactDetails"`
	RecommendedActions   []string             `json:"recommendedActions"`
}
```

### Validation Rule Structures

```go
// internal/domain/validation.go
package domain

import (
	"encoding/json"
	"fmt"
	"regexp"
	"time"
)

// ValidationRuleProcessor interface for type-specific validation
type ValidationRuleProcessor interface {
	ValidateRules(rules json.RawMessage) error
	GetDefaultRules() json.RawMessage
	GetExampleRules() []json.RawMessage
	ApplyRules(value interface{}, rules json.RawMessage) error
}

// String validation rules
type StringValidationRules struct {
	Required  *bool   `json:"required,omitempty"`
	MinLength *int    `json:"minLength,omitempty"`
	MaxLength *int    `json:"maxLength,omitempty"`
	Pattern   *string `json:"pattern,omitempty"`
	Format    *string `json:"format,omitempty"` // email, url, uuid, phone, date, datetime
	Enum      []string `json:"enum,omitempty"`
	Default   *string `json:"default,omitempty"`
}

func (s *StringValidationRules) Validate() error {
	if s.MinLength != nil && *s.MinLength < 0 {
		return fmt.Errorf("minLength cannot be negative")
	}
	if s.MaxLength != nil && *s.MaxLength < 1 {
		return fmt.Errorf("maxLength must be at least 1")
	}
	if s.MinLength != nil && s.MaxLength != nil && *s.MinLength > *s.MaxLength {
		return fmt.Errorf("minLength cannot be greater than maxLength")
	}
	if s.Pattern != nil {
		if _, err := regexp.Compile(*s.Pattern); err != nil {
			return fmt.Errorf("invalid regex pattern: %w", err)
		}
	}
	if s.Format != nil {
		validFormats := []string{"email", "url", "uuid", "phone", "date", "datetime"}
		valid := false
		for _, format := range validFormats {
			if *s.Format == format {
				valid = true
				break
			}
		}
		if !valid {
			return fmt.Errorf("invalid format: %s", *s.Format)
		}
	}
	return nil
}

// Integer validation rules
type IntegerValidationRules struct {
	Required   *bool    `json:"required,omitempty"`
	Min        *int64   `json:"min,omitempty"`
	Max        *int64   `json:"max,omitempty"`
	MultipleOf *int64   `json:"multipleOf,omitempty"`
	Enum       []int64  `json:"enum,omitempty"`
	Default    *int64   `json:"default,omitempty"`
}

func (i *IntegerValidationRules) Validate() error {
	if i.Min != nil && i.Max != nil && *i.Min > *i.Max {
		return fmt.Errorf("min cannot be greater than max")
	}
	if i.MultipleOf != nil && *i.MultipleOf <= 0 {
		return fmt.Errorf("multipleOf must be positive")
	}
	return nil
}

// Decimal validation rules  
type DecimalValidationRules struct {
	Required   *bool     `json:"required,omitempty"`
	Min        *float64  `json:"min,omitempty"`
	Max        *float64  `json:"max,omitempty"`
	Precision  *int      `json:"precision,omitempty"` // Total number of digits
	Scale      *int      `json:"scale,omitempty"`     // Number of digits after decimal point
	MultipleOf *float64  `json:"multipleOf,omitempty"`
	Enum       []float64 `json:"enum,omitempty"`
	Default    *float64  `json:"default,omitempty"`
}

func (d *DecimalValidationRules) Validate() error {
	if d.Min != nil && d.Max != nil && *d.Min > *d.Max {
		return fmt.Errorf("min cannot be greater than max")
	}
	if d.Precision != nil && (*d.Precision < 1 || *d.Precision > 15) {
		return fmt.Errorf("precision must be between 1 and 15")
	}
	if d.Scale != nil && (*d.Scale < 0 || *d.Scale > 10) {
		return fmt.Errorf("scale must be between 0 and 10")
	}
	if d.Precision != nil && d.Scale != nil && *d.Scale > *d.Precision {
		return fmt.Errorf("scale cannot be greater than precision")
	}
	return nil
}

// Boolean validation rules
type BooleanValidationRules struct {
	Required *bool `json:"required,omitempty"`
	Default  *bool `json:"default,omitempty"`
}

func (b *BooleanValidationRules) Validate() error {
	// Boolean rules are always valid - no constraints to check
	return nil
}

// Date validation rules
type DateValidationRules struct {
	Required *bool      `json:"required,omitempty"`
	Min      *time.Time `json:"min,omitempty"`
	Max      *time.Time `json:"max,omitempty"`
	Format   *string    `json:"format,omitempty"` // YYYY-MM-DD by default
	Default  *time.Time `json:"default,omitempty"`
}

func (d *DateValidationRules) Validate() error {
	if d.Min != nil && d.Max != nil && d.Min.After(*d.Max) {
		return fmt.Errorf("min date cannot be after max date")
	}
	return nil
}

// DateTime validation rules
type DateTimeValidationRules struct {
	Required *bool      `json:"required,omitempty"`
	Min      *time.Time `json:"min,omitempty"`
	Max      *time.Time `json:"max,omitempty"`
	Format   *string    `json:"format,omitempty"` // ISO8601 by default
	Timezone *string    `json:"timezone,omitempty"` // UTC by default
	Default  *time.Time `json:"default,omitempty"`
}

func (dt *DateTimeValidationRules) Validate() error {
	if dt.Min != nil && dt.Max != nil && dt.Min.After(*dt.Max) {
		return fmt.Errorf("min datetime cannot be after max datetime")
	}
	if dt.Timezone != nil {
		if _, err := time.LoadLocation(*dt.Timezone); err != nil {
			return fmt.Errorf("invalid timezone: %w", err)
		}
	}
	return nil
}

// JSON validation rules
type JSONValidationRules struct {
	Required     *bool           `json:"required,omitempty"`
	Schema       json.RawMessage `json:"schema,omitempty"`       // JSON Schema for validation
	MaxDepth     *int            `json:"maxDepth,omitempty"`
	MaxSize      *int            `json:"maxSize,omitempty"`      // Max size in bytes
	AllowedKeys  []string        `json:"allowedKeys,omitempty"`
	RequiredKeys []string        `json:"requiredKeys,omitempty"`
	Default      json.RawMessage `json:"default,omitempty"`
}

func (j *JSONValidationRules) Validate() error {
	if j.MaxDepth != nil && (*j.MaxDepth < 1 || *j.MaxDepth > 10) {
		return fmt.Errorf("maxDepth must be between 1 and 10")
	}
	if j.MaxSize != nil && *j.MaxSize < 1 {
		return fmt.Errorf("maxSize must be positive")
	}
	if j.Schema != nil {
		// Basic validation that schema is valid JSON
		var schemaMap map[string]interface{}
		if err := json.Unmarshal(j.Schema, &schemaMap); err != nil {
			return fmt.Errorf("schema must be valid JSON: %w", err)
		}
	}
	return nil
}

// Reference validation rules
type ReferenceValidationRules struct {
	Required        *bool       `json:"required,omitempty"`
	TargetThingClass *uuid.UUID  `json:"targetThingClass,omitempty"`
	AllowedClasses  []uuid.UUID `json:"allowedClasses,omitempty"`
	Cardinality     *string     `json:"cardinality,omitempty"` // "one" or "many"
	CascadeDelete   *bool       `json:"cascadeDelete,omitempty"`
}

func (r *ReferenceValidationRules) Validate() error {
	if r.Cardinality != nil && *r.Cardinality != "one" && *r.Cardinality != "many" {
		return fmt.Errorf("cardinality must be 'one' or 'many'")
	}
	if r.TargetThingClass != nil && len(r.AllowedClasses) > 0 {
		return fmt.Errorf("cannot specify both targetThingClass and allowedClasses")
	}
	return nil
}

// ValidationRuleTemplate provides examples and schemas for each data type
type ValidationRuleTemplate struct {
	DataType    AttributeType     `json:"dataType"`
	Schema      json.RawMessage   `json:"schema"`
	Examples    []json.RawMessage `json:"examples"`
	Description string            `json:"description"`
}
```

## Service Layer Implementation

### Attribute Business Logic Service

```go
// internal/service/attribute.go
package service

import (
	"context"
	"encoding/json"
	"fmt"
	"time"

	"github.com/company/udm-backend/internal/domain"
	"github.com/company/udm-backend/internal/repository/interfaces"
	"github.com/company/udm-backend/internal/service/validation"
	"github.com/go-playground/validator/v10"
	"github.com/google/uuid"
	"go.uber.org/zap"
)

type AttributeService struct {
	repo             interfaces.AttributeRepository
	thingClassRepo   interfaces.ThingClassRepository
	validator        *validator.Validate
	validationFactory *validation.Factory
	cache            interfaces.Cache
	logger           *zap.Logger
}

func NewAttributeService(
	repo interfaces.AttributeRepository,
	thingClassRepo interfaces.ThingClassRepository,
	validationFactory *validation.Factory,
	cache interfaces.Cache,
	logger *zap.Logger,
) *AttributeService {
	return &AttributeService{
		repo:              repo,
		thingClassRepo:    thingClassRepo,
		validator:         validator.New(),
		validationFactory: validationFactory,
		cache:             cache,
		logger:            logger,
	}
}

// CreateAttribute creates a new attribute with comprehensive validation
func (s *AttributeService) CreateAttribute(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateAttributeInput) (*domain.Attribute, error) {
	// Input validation
	if err := s.validateCreateInput(input); err != nil {
		return nil, domain.NewValidationError("input validation failed", err)
	}

	// Business rule validation
	if err := s.validateBusinessRules(ctx, tenant, input); err != nil {
		return nil, fmt.Errorf("business rule validation failed: %w", err)
	}

	// Validate validation rules for the specific data type
	if err := s.validateValidationRulesForDataType(input.DataType, input.ValidationRules); err != nil {
		return nil, domain.NewValidationRulesError(input.DataType, err)
	}

	// Check name uniqueness
	exists, err := s.repo.ExistsByName(ctx, tenant, input.Name)
	if err != nil {
		s.logger.Error("Failed to check attribute name uniqueness",
			zap.String("name", input.Name),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to check name uniqueness: %w", err)
	}
	if exists {
		return nil, domain.NewAttributeNameExistsError(input.Name)
	}

	// Create domain object
	now := time.Now()
	attribute := &domain.Attribute{
		BaseEntity: domain.BaseEntity{
			ID:        uuid.New(),
			CreatedAt: now,
			UpdatedAt: now,
			CreatedBy: tenant.UserID,
			IsActive:  true,
			Version:   1,
		},
		Name:            input.Name,
		DataType:        input.DataType,
		ValidationRules: input.ValidationRules,
		Description:     input.Description,
	}

	// Persist to database
	created, err := s.repo.Create(ctx, tenant, attribute)
	if err != nil {
		s.logger.Error("Failed to create attribute",
			zap.String("name", input.Name),
			zap.String("data_type", string(input.DataType)),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to create attribute: %w", err)
	}

	// Invalidate related caches
	s.invalidateAttributeCaches(ctx, tenant)

	// Log successful creation
	s.logger.Info("Successfully created attribute",
		zap.String("id", created.ID.String()),
		zap.String("name", created.Name),
		zap.String("data_type", string(created.DataType)),
		zap.String("tenant_schema", tenant.SchemaName),
	)

	return created, nil
}

// UpdateAttribute updates an existing attribute with schema evolution validation
func (s *AttributeService) UpdateAttribute(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID, input *domain.UpdateAttributeInput) (*domain.Attribute, error) {
	// Retrieve existing attribute
	existing, err := s.repo.GetByID(ctx, tenant, id)
	if err != nil {
		return nil, fmt.Errorf("failed to get attribute: %w", err)
	}
	if existing == nil {
		return nil, domain.NewAttributeNotFoundError(id)
	}

	// Input validation
	if err := s.validateUpdateInput(input); err != nil {
		return nil, domain.NewValidationError("input validation failed", err)
	}

	// Check for schema evolution impact
	impactAnalysis, err := s.analyzeSchemaEvolutionImpact(ctx, tenant, existing, input)
	if err != nil {
		return nil, fmt.Errorf("failed to analyze schema evolution impact: %w", err)
	}

	// Block breaking changes if attribute has assignments
	if impactAnalysis.SafetyLevel == domain.EvolutionSafetyLevelBreaking && impactAnalysis.AffectedAssignments > 0 {
		return nil, domain.NewSchemaEvolutionError("breaking changes not allowed for attributes with existing assignments", impactAnalysis.AffectedAssignments)
	}

	// Apply updates
	updated := *existing
	updated.UpdatedAt = time.Now()
	updated.Version++

	hasChanges := false

	// Update name if provided
	if input.Name != nil && *input.Name != existing.Name {
		// Check name uniqueness
		exists, err := s.repo.ExistsByName(ctx, tenant, *input.Name)
		if err != nil {
			return nil, fmt.Errorf("failed to check name uniqueness: %w", err)
		}
		if exists {
			return nil, domain.NewAttributeNameExistsError(*input.Name)
		}
		updated.Name = *input.Name
		hasChanges = true
	}

	// Update validation rules if provided
	if input.ValidationRules != nil {
		if err := s.validateValidationRulesForDataType(existing.DataType, input.ValidationRules); err != nil {
			return nil, domain.NewValidationRulesError(existing.DataType, err)
		}
		updated.ValidationRules = input.ValidationRules
		hasChanges = true
	}

	// Update description if provided
	if input.Description != nil {
		updated.Description = input.Description
		hasChanges = true
	}

	if !hasChanges {
		return existing, nil // No changes to apply
	}

	// Record schema evolution
	if err := s.recordSchemaEvolution(ctx, tenant, existing, &updated, impactAnalysis); err != nil {
		s.logger.Error("Failed to record schema evolution",
			zap.String("attribute_id", id.String()),
			zap.Error(err),
		)
		// Continue with update - schema evolution recording is non-critical
	}

	// Persist changes
	result, err := s.repo.Update(ctx, tenant, &updated)
	if err != nil {
		s.logger.Error("Failed to update attribute",
			zap.String("id", id.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to update attribute: %w", err)
	}

	// Invalidate caches
	s.invalidateAttributeCaches(ctx, tenant)
	s.cache.Delete(ctx, tenant.SchemaName, fmt.Sprintf("attribute:%s", id.String()))

	s.logger.Info("Successfully updated attribute",
		zap.String("id", id.String()),
		zap.String("name", result.Name),
		zap.Int("version", result.Version),
		zap.String("safety_level", string(impactAnalysis.SafetyLevel)),
		zap.String("tenant_schema", tenant.SchemaName),
	)

	return result, nil
}

// GetAttribute retrieves an attribute by ID with caching
func (s *AttributeService) GetAttribute(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.Attribute, error) {
	// Try cache first
	cacheKey := fmt.Sprintf("attribute:%s", id.String())
	var cached domain.Attribute
	if err := s.cache.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
		return &cached, nil
	}

	// Fetch from database
	attribute, err := s.repo.GetByID(ctx, tenant, id)
	if err != nil {
		s.logger.Error("Failed to get attribute",
			zap.String("id", id.String()),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to get attribute: %w", err)
	}

	if attribute == nil {
		return nil, domain.NewAttributeNotFoundError(id)
	}

	// Cache the result
	s.cache.Set(ctx, tenant.SchemaName, cacheKey, attribute)

	return attribute, nil
}

// ListAttributes retrieves paginated attributes with filtering and sorting
func (s *AttributeService) ListAttributes(ctx context.Context, tenant *domain.TenantContext, filter *domain.AttributeFilter, sorts []*domain.AttributeSort, limit int, offset int) ([]*domain.Attribute, int, error) {
	// Generate cache key based on filter parameters
	cacheKey := s.generateListCacheKey("attributes:list", filter, sorts, limit, offset)
	
	// Try cache first
	type CachedResult struct {
		Attributes []*domain.Attribute `json:"attributes"`
		Total      int                 `json:"total"`
	}
	var cached CachedResult
	if err := s.cache.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
		return cached.Attributes, cached.Total, nil
	}

	// Fetch from database
	attributes, total, err := s.repo.List(ctx, tenant, filter, sorts, limit, offset)
	if err != nil {
		s.logger.Error("Failed to list attributes",
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, 0, fmt.Errorf("failed to list attributes: %w", err)
	}

	// Cache the result
	result := CachedResult{Attributes: attributes, Total: total}
	s.cache.Set(ctx, tenant.SchemaName, cacheKey, &result)

	return attributes, total, nil
}

// SearchAttributes performs full-text search on attribute names and descriptions
func (s *AttributeService) SearchAttributes(ctx context.Context, tenant *domain.TenantContext, query string, dataType *domain.AttributeType, limit int, offset int) ([]*domain.Attribute, int, error) {
	attributes, total, err := s.repo.Search(ctx, tenant, query, dataType, limit, offset)
	if err != nil {
		s.logger.Error("Failed to search attributes",
			zap.String("query", query),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, 0, fmt.Errorf("failed to search attributes: %w", err)
	}

	return attributes, total, nil
}

// DeleteAttribute performs soft delete of an attribute
func (s *AttributeService) DeleteAttribute(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) error {
	// Check if attribute exists
	attribute, err := s.repo.GetByID(ctx, tenant, id)
	if err != nil {
		return fmt.Errorf("failed to get attribute: %w", err)
	}
	if attribute == nil {
		return domain.NewAttributeNotFoundError(id)
	}

	// Check if attribute is used in any Thing Class assignments
	assignmentCount, err := s.repo.GetAssignmentCount(ctx, tenant, id)
	if err != nil {
		return fmt.Errorf("failed to check assignments: %w", err)
	}
	if assignmentCount > 0 {
		return domain.NewAttributeInUseError(attribute.Name, assignmentCount)
	}

	// Perform soft delete
	if err := s.repo.SoftDelete(ctx, tenant, id); err != nil {
		s.logger.Error("Failed to delete attribute",
			zap.String("id", id.String()),
			zap.String("name", attribute.Name),
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return fmt.Errorf("failed to delete attribute: %w", err)
	}

	// Invalidate caches
	s.invalidateAttributeCaches(ctx, tenant)
	s.cache.Delete(ctx, tenant.SchemaName, fmt.Sprintf("attribute:%s", id.String()))

	s.logger.Info("Successfully deleted attribute",
		zap.String("id", id.String()),
		zap.String("name", attribute.Name),
		zap.String("tenant_schema", tenant.SchemaName),
	)

	return nil
}

// GetAttributeStatistics retrieves comprehensive attribute statistics
func (s *AttributeService) GetAttributeStatistics(ctx context.Context, tenant *domain.TenantContext, filter *domain.AttributeFilter) (*domain.AttributeStatistics, error) {
	stats, err := s.repo.GetStatistics(ctx, tenant, filter)
	if err != nil {
		s.logger.Error("Failed to get attribute statistics",
			zap.String("tenant_schema", tenant.SchemaName),
			zap.Error(err),
		)
		return nil, fmt.Errorf("failed to get attribute statistics: %w", err)
	}

	return stats, nil
}

// AnalyzeAttributeImpact analyzes the impact of proposed changes to an attribute
func (s *AttributeService) AnalyzeAttributeImpact(ctx context.Context, tenant *domain.TenantContext, attributeID uuid.UUID, proposedChanges *domain.UpdateAttributeInput) (*domain.AttributeImpactAnalysis, error) {
	// Get existing attribute
	existing, err := s.repo.GetByID(ctx, tenant, attributeID)
	if err != nil {
		return nil, fmt.Errorf("failed to get attribute: %w", err)
	}
	if existing == nil {
		return nil, domain.NewAttributeNotFoundError(attributeID)
	}

	// Perform impact analysis
	return s.analyzeSchemaEvolutionImpact(ctx, tenant, existing, proposedChanges)
}

// Private helper methods

func (s *AttributeService) validateCreateInput(input *domain.CreateAttributeInput) error {
	if err := s.validator.Struct(input); err != nil {
		return err
	}

	// Validate data type
	if !input.DataType.IsValid() {
		return fmt.Errorf("invalid data type: %s", input.DataType)
	}

	return nil
}

func (s *AttributeService) validateUpdateInput(input *domain.UpdateAttributeInput) error {
	if err := s.validator.Struct(input); err != nil {
		return err
	}
	return nil
}

func (s *AttributeService) validateBusinessRules(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateAttributeInput) error {
	// Add any business-specific validation rules here
	// For example, certain attribute names might be reserved
	reservedNames := []string{"id", "created_at", "updated_at", "version"}
	for _, reserved := range reservedNames {
		if input.Name == reserved {
			return fmt.Errorf("attribute name '%s' is reserved", input.Name)
		}
	}
	return nil
}

func (s *AttributeService) validateValidationRulesForDataType(dataType domain.AttributeType, rules json.RawMessage) error {
	if len(rules) == 0 {
		return nil // Empty rules are valid
	}

	processor := s.validationFactory.GetProcessor(dataType)
	if processor == nil {
		return fmt.Errorf("unsupported data type: %s", dataType)
	}

	return processor.ValidateRules(rules)
}

func (s *AttributeService) analyzeSchemaEvolutionImpact(ctx context.Context, tenant *domain.TenantContext, existing *domain.Attribute, input *domain.UpdateAttributeInput) (*domain.AttributeImpactAnalysis, error) {
	// Count affected assignments
	assignmentCount, err := s.repo.GetAssignmentCount(ctx, tenant, existing.ID)
	if err != nil {
		return nil, fmt.Errorf("failed to count assignments: %w", err)
	}

	// Count affected things (things that use this attribute)
	thingCount, err := s.repo.GetThingUsageCount(ctx, tenant, existing.ID)
	if err != nil {
		return nil, fmt.Errorf("failed to count thing usage: %w", err)
	}

	analysis := &domain.AttributeImpactAnalysis{
		Attribute:           existing,
		AffectedAssignments: assignmentCount,
		AffectedThings:      thingCount,
		RecommendedActions:  []string{},
	}

	// Analyze changes
	if input.ValidationRules != nil {
		safetyLevel, details := s.analyzeValidationRuleChanges(existing.ValidationRules, input.ValidationRules)
		analysis.SafetyLevel = safetyLevel
		analysis.ImpactDetails = details
	} else {
		analysis.SafetyLevel = domain.EvolutionSafetyLevelSafe
		analysis.ImpactDetails = json.RawMessage(`{"type": "no_validation_changes"}`)
	}

	// Generate recommendations
	if assignmentCount > 0 {
		analysis.RecommendedActions = append(analysis.RecommendedActions, "Review affected Thing Classes before applying changes")
		if thingCount > 0 {
			analysis.RecommendedActions = append(analysis.RecommendedActions, fmt.Sprintf("Validate existing data in %d Things", thingCount))
		}
	}

	return analysis, nil
}

func (s *AttributeService) analyzeValidationRuleChanges(oldRules, newRules json.RawMessage) (domain.EvolutionSafetyLevel, json.RawMessage) {
	// Simplified analysis - in production this would be more sophisticated
	if len(oldRules) == 0 && len(newRules) > 0 {
		// Adding rules to attribute with no rules - potentially breaking
		return domain.EvolutionSafetyLevelWarning, json.RawMessage(`{"type": "rules_added", "description": "Adding validation rules may cause existing data to become invalid"}`)
	}

	if len(oldRules) > 0 && len(newRules) == 0 {
		// Removing all rules - generally safe
		return domain.EvolutionSafetyLevelSafe, json.RawMessage(`{"type": "rules_removed", "description": "Removing validation rules is safe"}`)
	}

	// For now, consider all other changes as warnings
	return domain.EvolutionSafetyLevelWarning, json.RawMessage(`{"type": "rules_modified", "description": "Validation rules have been modified"}`)
}

func (s *AttributeService) recordSchemaEvolution(ctx context.Context, tenant *domain.TenantContext, old, new *domain.Attribute, analysis *domain.AttributeImpactAnalysis) error {
	evolution := &domain.AttributeEvolution{
		BaseEntity: domain.BaseEntity{
			ID:        uuid.New(),
			CreatedAt: time.Now(),
			UpdatedAt: time.Now(),
			CreatedBy: tenant.UserID,
			IsActive:  true,
			Version:   1,
		},
		AttributeID:    old.ID,
		ChangeType:     s.determineChangeType(old, new),
		OldValues:      s.serializeAttributeForEvolution(old),
		NewValues:      s.serializeAttributeForEvolution(new),
		ImpactAnalysis: analysis.ImpactDetails,
		SafetyLevel:    analysis.SafetyLevel,
		AppliedAt:      time.Now(),
		AppliedBy:      tenant.UserID,
	}

	return s.repo.RecordEvolution(ctx, tenant, evolution)
}

func (s *AttributeService) determineChangeType(old, new *domain.Attribute) domain.AttributeChangeType {
	if old.Name != new.Name {
		return domain.AttributeChangeNameChanged
	}
	if old.Description != new.Description {
		return domain.AttributeChangeDescriptionChanged
	}
	if string(old.ValidationRules) != string(new.ValidationRules) {
		// This is simplified - would need more sophisticated comparison
		return domain.AttributeChangeValidationRulesRelaxed
	}
	return domain.AttributeChangeDescriptionChanged
}

func (s *AttributeService) serializeAttributeForEvolution(attr *domain.Attribute) json.RawMessage {
	data := map[string]interface{}{
		"name":            attr.Name,
		"dataType":        attr.DataType,
		"validationRules": attr.ValidationRules,
		"description":     attr.Description,
		"version":         attr.Version,
	}
	
	bytes, _ := json.Marshal(data)
	return json.RawMessage(bytes)
}

func (s *AttributeService) generateListCacheKey(prefix string, filter *domain.AttributeFilter, sorts []*domain.AttributeSort, limit, offset int) string {
	// Generate a stable cache key from parameters
	key := fmt.Sprintf("%s:l%d:o%d", prefix, limit, offset)
	
	if filter != nil {
		if filter.DataType != nil {
			key += fmt.Sprintf(":dt%s", *filter.DataType)
		}
		if filter.Name != nil {
			key += fmt.Sprintf(":n%s", *filter.Name)
		}
		// Add other filter parameters as needed
	}
	
	for _, sort := range sorts {
		key += fmt.Sprintf(":s%s%s", sort.Field, sort.Direction)
	}
	
	return key
}

func (s *AttributeService) invalidateAttributeCaches(ctx context.Context, tenant *domain.TenantContext) {
	patterns := []string{
		"attributes:list:*",
		"attribute_stats:*",
		"validation_templates:*",
	}
	
	for _, pattern := range patterns {
		s.cache.InvalidatePattern(ctx, tenant.SchemaName, pattern)
	}
}
```

## Repository Layer Implementation

### PostgreSQL Repository

```go
// internal/repository/postgres/attribute.go
package postgres

import (
	"context"
	"database/sql"
	"encoding/json"
	"fmt"
	"strings"
	"time"

	"github.com/company/udm-backend/internal/domain"
	"github.com/company/udm-backend/internal/database"
	"github.com/company/udm-backend/internal/repository/postgres/queries"
	"github.com/google/uuid"
	"github.com/jmoiron/sqlx"
	"go.uber.org/zap"
)

type PostgresAttributeRepository struct {
	dbManager *database.Manager
	logger    *zap.Logger
	queries   *queries.AttributeQueries
}

func NewAttributeRepository(dbManager *database.Manager, logger *zap.Logger) *PostgresAttributeRepository {
	return &PostgresAttributeRepository{
		dbManager: dbManager,
		logger:    logger,
		queries:   queries.NewAttributeQueries(),
	}
}

func (r *PostgresAttributeRepository) Create(ctx context.Context, tenant *domain.TenantContext, attribute *domain.Attribute) (*domain.Attribute, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}

	var result domain.Attribute
	err = db.QueryRowxContext(ctx, r.queries.CreateAttribute,
		attribute.ID, attribute.Name, attribute.DataType, attribute.ValidationRules,
		attribute.Description, attribute.CreatedAt, attribute.UpdatedAt,
		attribute.CreatedBy, attribute.IsActive, attribute.Version,
	).StructScan(&result)

	if err != nil {
		if r.isUniqueViolation(err) {
			return nil, domain.NewAttributeNameExistsError(attribute.Name)
		}
		return nil, fmt.Errorf("failed to create attribute: %w", err)
	}

	return &result, nil
}

func (r *PostgresAttributeRepository) GetByID(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.Attribute, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}

	var result domain.Attribute
	err = db.QueryRowxContext(ctx, r.queries.GetAttributeByID, id).StructScan(&result)
	if err != nil {
		if err == sql.ErrNoRows {
			return nil, nil
		}
		return nil, fmt.Errorf("failed to get attribute: %w", err)
	}

	return &result, nil
}

func (r *PostgresAttributeRepository) GetByName(ctx context.Context, tenant *domain.TenantContext, name string) (*domain.Attribute, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}

	var result domain.Attribute
	err = db.QueryRowxContext(ctx, r.queries.GetAttributeByName, name).StructScan(&result)
	if err != nil {
		if err == sql.ErrNoRows {
			return nil, nil
		}
		return nil, fmt.Errorf("failed to get attribute by name: %w", err)
	}

	return &result, nil
}

func (r *PostgresAttributeRepository) ExistsByName(ctx context.Context, tenant *domain.TenantContext, name string) (bool, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return false, fmt.Errorf("failed to get database connection: %w", err)
	}

	var exists bool
	err = db.QueryRowxContext(ctx, r.queries.CheckAttributeNameExists, name).Scan(&exists)
	if err != nil {
		return false, fmt.Errorf("failed to check attribute name existence: %w", err)
	}

	return exists, nil
}

func (r *PostgresAttributeRepository) List(ctx context.Context, tenant *domain.TenantContext, filter *domain.AttributeFilter, sorts []*domain.AttributeSort, limit, offset int) ([]*domain.Attribute, int, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, 0, fmt.Errorf("failed to get database connection: %w", err)
	}

	// Build dynamic query
	whereClause, whereArgs := r.buildWhereClause(filter)
	orderClause := r.buildOrderClause(sorts)

	// Count query
	countQuery := fmt.Sprintf(r.queries.CountAttributesTemplate, whereClause)
	var totalCount int
	if err := db.QueryRowxContext(ctx, countQuery, whereArgs...).Scan(&totalCount); err != nil {
		return nil, 0, fmt.Errorf("failed to count attributes: %w", err)
	}

	// Main query with pagination
	listQuery := fmt.Sprintf(r.queries.ListAttributesTemplate, whereClause, orderClause)
	args := append(whereArgs, limit, offset)

	rows, err := db.QueryxContext(ctx, listQuery, args...)
	if err != nil {
		return nil, 0, fmt.Errorf("failed to query attributes: %w", err)
	}
	defer rows.Close()

	var attributes []*domain.Attribute
	for rows.Next() {
		var attr domain.Attribute
		if err := rows.StructScan(&attr); err != nil {
			return nil, 0, fmt.Errorf("failed to scan attribute: %w", err)
		}
		attributes = append(attributes, &attr)
	}

	if err := rows.Err(); err != nil {
		return nil, 0, fmt.Errorf("error iterating attribute rows: %w", err)
	}

	return attributes, totalCount, nil
}

func (r *PostgresAttributeRepository) Search(ctx context.Context, tenant *domain.TenantContext, query string, dataType *domain.AttributeType, limit, offset int) ([]*domain.Attribute, int, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, 0, fmt.Errorf("failed to get database connection: %w", err)
	}

	args := []interface{}{query}
	searchQuery := r.queries.SearchAttributes

	if dataType != nil {
		searchQuery = r.queries.SearchAttributesByDataType
		args = append(args, *dataType)
	}

	args = append(args, limit, offset)

	// Count total results
	countQuery := strings.Replace(searchQuery, "SELECT id, name,", "SELECT COUNT(*)", 1)
	countQuery = strings.Replace(countQuery, "ORDER BY rank DESC, name ASC LIMIT $2 OFFSET $3", "", 1)
	if dataType != nil {
		countQuery = strings.Replace(countQuery, "LIMIT $3 OFFSET $4", "", 1)
	}

	var totalCount int
	countArgs := args[:len(args)-2] // Remove limit and offset
	if err := db.QueryRowxContext(ctx, countQuery, countArgs...).Scan(&totalCount); err != nil {
		return nil, 0, fmt.Errorf("failed to count search results: %w", err)
	}

	// Execute search
	rows, err := db.QueryxContext(ctx, searchQuery, args...)
	if err != nil {
		return nil, 0, fmt.Errorf("failed to search attributes: %w", err)
	}
	defer rows.Close()

	var attributes []*domain.Attribute
	for rows.Next() {
		var attr domain.Attribute
		var rank float64 // For search ranking
		
		if err := rows.Scan(
			&attr.ID, &attr.Name, &attr.DataType, &attr.ValidationRules,
			&attr.Description, &attr.CreatedAt, &attr.UpdatedAt,
			&attr.CreatedBy, &attr.UpdatedBy, &attr.IsActive, &attr.Version,
			&rank,
		); err != nil {
			return nil, 0, fmt.Errorf("failed to scan search result: %w", err)
		}
		attributes = append(attributes, &attr)
	}

	return attributes, totalCount, nil
}

func (r *PostgresAttributeRepository) Update(ctx context.Context, tenant *domain.TenantContext, attribute *domain.Attribute) (*domain.Attribute, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}

	var result domain.Attribute
	err = db.QueryRowxContext(ctx, r.queries.UpdateAttribute,
		attribute.Name, attribute.ValidationRules, attribute.Description,
		attribute.UpdatedAt, attribute.UpdatedBy, attribute.Version,
		attribute.ID, attribute.Version-1, // Use optimistic locking
	).StructScan(&result)

	if err != nil {
		if err == sql.ErrNoRows {
			return nil, fmt.Errorf("attribute not found or version conflict")
		}
		if r.isUniqueViolation(err) {
			return nil, domain.NewAttributeNameExistsError(attribute.Name)
		}
		return nil, fmt.Errorf("failed to update attribute: %w", err)
	}

	return &result, nil
}

func (r *PostgresAttributeRepository) SoftDelete(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) error {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return fmt.Errorf("failed to get database connection: %w", err)
	}

	result, err := db.ExecContext(ctx, r.queries.SoftDeleteAttribute, time.Now(), id)
	if err != nil {
		return fmt.Errorf("failed to soft delete attribute: %w", err)
	}

	rowsAffected, err := result.RowsAffected()
	if err != nil {
		return fmt.Errorf("failed to get rows affected: %w", err)
	}

	if rowsAffected == 0 {
		return fmt.Errorf("attribute not found")
	}

	return nil
}

func (r *PostgresAttributeRepository) GetAssignmentCount(ctx context.Context, tenant *domain.TenantContext, attributeID uuid.UUID) (int, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return 0, fmt.Errorf("failed to get database connection: %w", err)
	}

	var count int
	err = db.QueryRowxContext(ctx, r.queries.CountAttributeAssignments, attributeID).Scan(&count)
	if err != nil {
		return 0, fmt.Errorf("failed to count attribute assignments: %w", err)
	}

	return count, nil
}

func (r *PostgresAttributeRepository) GetThingUsageCount(ctx context.Context, tenant *domain.TenantContext, attributeID uuid.UUID) (int, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return 0, fmt.Errorf("failed to get database connection: %w", err)
	}

	var count int
	err = db.QueryRowxContext(ctx, r.queries.CountThingUsage, attributeID).Scan(&count)
	if err != nil {
		return 0, fmt.Errorf("failed to count thing usage: %w", err)
	}

	return count, nil
}

func (r *PostgresAttributeRepository) GetStatistics(ctx context.Context, tenant *domain.TenantContext, filter *domain.AttributeFilter) (*domain.AttributeStatistics, error) {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return nil, fmt.Errorf("failed to get database connection: %w", err)
	}

	// Build filter for statistics query
	whereClause, whereArgs := r.buildWhereClause(filter)
	statsQuery := fmt.Sprintf(r.queries.GetAttributeStatisticsTemplate, whereClause)

	rows, err := db.QueryxContext(ctx, statsQuery, whereArgs...)
	if err != nil {
		return nil, fmt.Errorf("failed to get attribute statistics: %w", err)
	}
	defer rows.Close()

	stats := &domain.AttributeStatistics{
		CountByDataType: []*domain.DataTypeCount{},
	}

	for rows.Next() {
		var dataType string
		var count int
		var percentage float64
		
		if err := rows.Scan(&dataType, &count, &percentage); err != nil {
			return nil, fmt.Errorf("failed to scan statistics: %w", err)
		}

		stats.CountByDataType = append(stats.CountByDataType, &domain.DataTypeCount{
			DataType:   domain.AttributeType(dataType),
			Count:      count,
			Percentage: percentage,
		})
		
		stats.TotalCount += count
		if dataType == "string" { // Only count active for total
			stats.ActiveCount = count
		}
	}

	return stats, nil
}

func (r *PostgresAttributeRepository) RecordEvolution(ctx context.Context, tenant *domain.TenantContext, evolution *domain.AttributeEvolution) error {
	db, err := r.dbManager.GetConnection(tenant.SchemaName)
	if err != nil {
		return fmt.Errorf("failed to get database connection: %w", err)
	}

	_, err = db.ExecContext(ctx, r.queries.RecordAttributeEvolution,
		evolution.ID, evolution.AttributeID, evolution.ChangeType,
		evolution.OldValues, evolution.NewValues, evolution.ImpactAnalysis,
		evolution.SafetyLevel, evolution.AppliedAt, evolution.AppliedBy,
		evolution.RollbackData,
	)

	if err != nil {
		return fmt.Errorf("failed to record attribute evolution: %w", err)
	}

	return nil
}

// Private helper methods

func (r *PostgresAttributeRepository) buildWhereClause(filter *domain.AttributeFilter) (string, []interface{}) {
	conditions := []string{"is_active = TRUE"}
	args := []interface{}{}
	argCount := 0

	if filter == nil {
		return strings.Join(conditions, " AND "), args
	}

	if filter.Name != nil {
		argCount++
		conditions = append(conditions, fmt.Sprintf("name ILIKE $%d", argCount))
		args = append(args, "%"+*filter.Name+"%")
	}

	if filter.NameContains != nil {
		argCount++
		conditions = append(conditions, fmt.Sprintf("name ILIKE $%d", argCount))
		args = append(args, "%"+*filter.NameContains+"%")
	}

	if filter.DataType != nil {
		argCount++
		conditions = append(conditions, fmt.Sprintf("data_type = $%d", argCount))
		args = append(args, *filter.DataType)
	}

	if filter.IsActive != nil {
		conditions = conditions[:len(conditions)-1] // Remove default is_active = TRUE
		argCount++
		conditions = append(conditions, fmt.Sprintf("is_active = $%d", argCount))
		args = append(args, *filter.IsActive)
	}

	if filter.CreatedBy != nil {
		argCount++
		conditions = append(conditions, fmt.Sprintf("created_by = $%d", argCount))
		args = append(args, *filter.CreatedBy)
	}

	if filter.CreatedAfter != nil {
		argCount++
		conditions = append(conditions, fmt.Sprintf("created_at >= $%d", argCount))
		args = append(args, *filter.CreatedAfter)
	}

	if filter.CreatedBefore != nil {
		argCount++
		conditions = append(conditions, fmt.Sprintf("created_at <= $%d", argCount))
		args = append(args, *filter.CreatedBefore)
	}

	if filter.HasAssignments != nil && *filter.HasAssignments {
		conditions = append(conditions, `EXISTS (
			SELECT 1 FROM thing_class_attributes tca 
			WHERE tca.attribute_id = attributes.id AND tca.is_active = TRUE
		)`)
	} else if filter.HasAssignments != nil && !*filter.HasAssignments {
		conditions = append(conditions, `NOT EXISTS (
			SELECT 1 FROM thing_class_attributes tca 
			WHERE tca.attribute_id = attributes.id AND tca.is_active = TRUE
		)`)
	}

	if filter.ValidationRulesContains != nil && len(filter.ValidationRulesContains) > 0 {
		argCount++
		conditions = append(conditions, fmt.Sprintf("validation_rules @> $%d", argCount))
		args = append(args, filter.ValidationRulesContains)
	}

	return strings.Join(conditions, " AND "), args
}

func (r *PostgresAttributeRepository) buildOrderClause(sorts []*domain.AttributeSort) string {
	if len(sorts) == 0 {
		return "ORDER BY name ASC"
	}

	var clauses []string
	for _, sort := range sorts {
		var field string
		switch sort.Field {
		case domain.AttributeSortFieldName:
			field = "name"
		case domain.AttributeSortFieldDataType:
			field = "data_type"
		case domain.AttributeSortFieldCreatedAt:
			field = "created_at"
		case domain.AttributeSortFieldUpdatedAt:
			field = "updated_at"
		case domain.AttributeSortFieldAssignmentCount:
			// This would require a subquery for assignment count
			field = "(SELECT COUNT(*) FROM thing_class_attributes tca WHERE tca.attribute_id = attributes.id AND tca.is_active = TRUE)"
		default:
			field = "name" // fallback
		}

		direction := "ASC"
		if sort.Direction == domain.SortDirectionDesc {
			direction = "DESC"
		}

		clauses = append(clauses, fmt.Sprintf("%s %s", field, direction))
	}

	return "ORDER BY " + strings.Join(clauses, ", ")
}

func (r *PostgresAttributeRepository) isUniqueViolation(err error) bool {
	// Check if error is a unique constraint violation
	// This would be database-specific error checking
	return strings.Contains(err.Error(), "unique_attribute_name") ||
		strings.Contains(err.Error(), "duplicate key value")
}
```

## GraphQL Resolver Implementation

### Attribute Resolvers

```go
// internal/graph/resolvers/attribute.go
package resolvers

import (
	"context"
	"encoding/json"
	"fmt"
	"strconv"

	"github.com/company/udm-backend/internal/domain"
	"github.com/company/udm-backend/internal/graph/generated"
	"github.com/google/uuid"
)

// Attribute resolver
func (r *queryResolver) Attribute(ctx context.Context, id string) (*domain.Attribute, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return nil, fmt.Errorf("tenant context required")
	}

	attributeID, err := uuid.Parse(id)
	if err != nil {
		return nil, fmt.Errorf("invalid attribute ID: %w", err)
	}

	return r.AttributeService.GetAttribute(ctx, tenantCtx, attributeID)
}

func (r *queryResolver) Attributes(ctx context.Context, filter *domain.AttributeFilter, sort []*domain.AttributeSort, first *int, after *string) (*domain.AttributeConnection, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return nil, fmt.Errorf("tenant context required")
	}

	// Handle pagination
	limit := 20 // default
	if first != nil {
		if *first > 100 {
			return nil, fmt.Errorf("maximum first value is 100")
		}
		limit = *first
	}

	offset := 0
	if after != nil {
		var err error
		offset, err = decodeCursor(*after)
		if err != nil {
			return nil, fmt.Errorf("invalid cursor: %w", err)
		}
	}

	attributes, total, err := r.AttributeService.ListAttributes(ctx, tenantCtx, filter, sort, limit, offset)
	if err != nil {
		return nil, err
	}

	// Build connection
	edges := make([]*domain.AttributeEdge, len(attributes))
	for i, attr := range attributes {
		edges[i] = &domain.AttributeEdge{
			Node:   attr,
			Cursor: encodeCursor(offset + i),
		}
	}

	pageInfo := &domain.PageInfo{
		HasNextPage:     offset+limit < total,
		HasPreviousPage: offset > 0,
	}

	if len(edges) > 0 {
		pageInfo.StartCursor = &edges[0].Cursor
		pageInfo.EndCursor = &edges[len(edges)-1].Cursor
	}

	return &domain.AttributeConnection{
		Edges:      edges,
		PageInfo:   pageInfo,
		TotalCount: total,
	}, nil
}

func (r *queryResolver) SearchAttributes(ctx context.Context, query string, dataType *domain.AttributeType, first *int, after *string) (*domain.AttributeConnection, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return nil, fmt.Errorf("tenant context required")
	}

	limit := 20
	if first != nil {
		if *first > 100 {
			return nil, fmt.Errorf("maximum first value is 100")
		}
		limit = *first
	}

	offset := 0
	if after != nil {
		var err error
		offset, err = decodeCursor(*after)
		if err != nil {
			return nil, fmt.Errorf("invalid cursor: %w", err)
		}
	}

	attributes, total, err := r.AttributeService.SearchAttributes(ctx, tenantCtx, query, dataType, limit, offset)
	if err != nil {
		return nil, err
	}

	// Build connection (same logic as Attributes resolver)
	edges := make([]*domain.AttributeEdge, len(attributes))
	for i, attr := range attributes {
		edges[i] = &domain.AttributeEdge{
			Node:   attr,
			Cursor: encodeCursor(offset + i),
		}
	}

	pageInfo := &domain.PageInfo{
		HasNextPage:     offset+limit < total,
		HasPreviousPage: offset > 0,
	}

	if len(edges) > 0 {
		pageInfo.StartCursor = &edges[0].Cursor
		pageInfo.EndCursor = &edges[len(edges)-1].Cursor
	}

	return &domain.AttributeConnection{
		Edges:      edges,
		PageInfo:   pageInfo,
		TotalCount: total,
	}, nil
}

func (r *queryResolver) AttributeStatistics(ctx context.Context, filter *domain.AttributeFilter) (*domain.AttributeStatistics, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return nil, fmt.Errorf("tenant context required")
	}

	return r.AttributeService.GetAttributeStatistics(ctx, tenantCtx, filter)
}

func (r *queryResolver) AnalyzeAttributeImpact(ctx context.Context, attributeID string, proposedChanges domain.UpdateAttributeInput) (*domain.AttributeImpactAnalysis, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return nil, fmt.Errorf("tenant context required")
	}

	id, err := uuid.Parse(attributeID)
	if err != nil {
		return nil, fmt.Errorf("invalid attribute ID: %w", err)
	}

	return r.AttributeService.AnalyzeAttributeImpact(ctx, tenantCtx, id, &proposedChanges)
}

// Attribute field resolvers
func (r *attributeResolver) AssignmentCount(ctx context.Context, obj *domain.Attribute) (int, error) {
	if obj.AssignmentCount != nil {
		return *obj.AssignmentCount, nil
	}

	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return 0, fmt.Errorf("tenant context required")
	}

	// Lazy load assignment count
	count, err := r.AttributeService.GetAssignmentCount(ctx, tenantCtx, obj.ID)
	if err != nil {
		return 0, err
	}

	obj.AssignmentCount = &count
	return count, nil
}

func (r *attributeResolver) Assignments(ctx context.Context, obj *domain.Attribute, first *int, after *string, filter *domain.ThingClassAttributeFilter) (*domain.ThingClassAttributeConnection, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return nil, fmt.Errorf("tenant context required")
	}

	// Handle pagination
	limit := 20
	if first != nil {
		if *first > 100 {
			return nil, fmt.Errorf("maximum first value is 100")
		}
		limit = *first
	}

	offset := 0
	if after != nil {
		var err error
		offset, err = decodeCursor(*after)
		if err != nil {
			return nil, fmt.Errorf("invalid cursor: %w", err)
		}
	}

	// Add attribute ID filter
	if filter == nil {
		filter = &domain.ThingClassAttributeFilter{}
	}
	filter.AttributeID = &obj.ID

	assignments, total, err := r.ThingClassAttributeService.List(ctx, tenantCtx, filter, nil, limit, offset)
	if err != nil {
		return nil, err
	}

	// Build connection
	edges := make([]*domain.ThingClassAttributeEdge, len(assignments))
	for i, assignment := range assignments {
		edges[i] = &domain.ThingClassAttributeEdge{
			Node:   assignment,
			Cursor: encodeCursor(offset + i),
		}
	}

	pageInfo := &domain.PageInfo{
		HasNextPage:     offset+limit < total,
		HasPreviousPage: offset > 0,
	}

	if len(edges) > 0 {
		pageInfo.StartCursor = &edges[0].Cursor
		pageInfo.EndCursor = &edges[len(edges)-1].Cursor
	}

	return &domain.ThingClassAttributeConnection{
		Edges:    edges,
		PageInfo: pageInfo,
	}, nil
}

func (r *attributeResolver) EvolutionHistory(ctx context.Context, obj *domain.Attribute, first *int, after *string) (*domain.AttributeEvolutionConnection, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return nil, fmt.Errorf("tenant context required")
	}

	limit := 10
	if first != nil {
		if *first > 50 {
			return nil, fmt.Errorf("maximum first value is 50")
		}
		limit = *first
	}

	offset := 0
	if after != nil {
		var err error
		offset, err = decodeCursor(*after)
		if err != nil {
			return nil, fmt.Errorf("invalid cursor: %w", err)
		}
	}

	evolution, total, err := r.AttributeService.GetEvolutionHistory(ctx, tenantCtx, obj.ID, limit, offset)
	if err != nil {
		return nil, err
	}

	// Build connection
	edges := make([]*domain.AttributeEvolutionEdge, len(evolution))
	for i, evo := range evolution {
		edges[i] = &domain.AttributeEvolutionEdge{
			Node:   evo,
			Cursor: encodeCursor(offset + i),
		}
	}

	pageInfo := &domain.PageInfo{
		HasNextPage:     offset+limit < total,
		HasPreviousPage: offset > 0,
	}

	if len(edges) > 0 {
		pageInfo.StartCursor = &edges[0].Cursor
		pageInfo.EndCursor = &edges[len(edges)-1].Cursor
	}

	return &domain.AttributeEvolutionConnection{
		Edges:    edges,
		PageInfo: pageInfo,
	}, nil
}

// Helper functions
func encodeCursor(offset int) string {
	return fmt.Sprintf("cursor:%d", offset)
}

func decodeCursor(cursor string) (int, error) {
	var offset int
	_, err := fmt.Sscanf(cursor, "cursor:%d", &offset)
	return offset, err
}

func getTenantContext(ctx context.Context) *domain.TenantContext {
	if tenant, ok := ctx.Value("tenant").(*domain.TenantContext); ok {
		return tenant
	}
	return nil
}

// Mutation resolvers
func (r *mutationResolver) CreateAttribute(ctx context.Context, input domain.CreateAttributeInput, clientMutationID *string) (*domain.AttributePayload, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return &domain.AttributePayload{
			Errors: []*domain.UserError{{
				Message: "Authentication required",
				Code:    "UNAUTHORIZED",
			}},
			ClientMutationID: clientMutationID,
		}, nil
	}

	attribute, err := r.AttributeService.CreateAttribute(ctx, tenantCtx, &input)
	if err != nil {
		return &domain.AttributePayload{
			Errors: []*domain.UserError{convertError(err)},
			ClientMutationID: clientMutationID,
		}, nil
	}

	return &domain.AttributePayload{
		Attribute:        attribute,
		Errors:           []*domain.UserError{},
		ClientMutationID: clientMutationID,
	}, nil
}

func (r *mutationResolver) UpdateAttribute(ctx context.Context, id string, input domain.UpdateAttributeInput, clientMutationID *string) (*domain.AttributePayload, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return &domain.AttributePayload{
			Errors: []*domain.UserError{{
				Message: "Authentication required",
				Code:    "UNAUTHORIZED",
			}},
			ClientMutationID: clientMutationID,
		}, nil
	}

	attributeID, err := uuid.Parse(id)
	if err != nil {
		return &domain.AttributePayload{
			Errors: []*domain.UserError{{
				Field:   "id",
				Message: "Invalid attribute ID",
				Code:    "INVALID_ID",
			}},
			ClientMutationID: clientMutationID,
		}, nil
	}

	attribute, err := r.AttributeService.UpdateAttribute(ctx, tenantCtx, attributeID, &input)
	if err != nil {
		return &domain.AttributePayload{
			Errors: []*domain.UserError{convertError(err)},
			ClientMutationID: clientMutationID,
		}, nil
	}

	return &domain.AttributePayload{
		Attribute:        attribute,
		Errors:           []*domain.UserError{},
		ClientMutationID: clientMutationID,
	}, nil
}

func (r *mutationResolver) DeleteAttribute(ctx context.Context, id string, clientMutationID *string) (*domain.DeleteAttributePayload, error) {
	tenantCtx := getTenantContext(ctx)
	if tenantCtx == nil {
		return &domain.DeleteAttributePayload{
			Success: false,
			Errors: []*domain.UserError{{
				Message: "Authentication required",
				Code:    "UNAUTHORIZED",
			}},
			ClientMutationID: clientMutationID,
		}, nil
	}

	attributeID, err := uuid.Parse(id)
	if err != nil {
		return &domain.DeleteAttributePayload{
			Success: false,
			Errors: []*domain.UserError{{
				Field:   "id",
				Message: "Invalid attribute ID",
				Code:    "INVALID_ID",
			}},
			ClientMutationID: clientMutationID,
		}, nil
	}

	if err := r.AttributeService.DeleteAttribute(ctx, tenantCtx, attributeID); err != nil {
		return &domain.DeleteAttributePayload{
			Success: false,
			Errors:  []*domain.UserError{convertError(err)},
			ClientMutationID: clientMutationID,
		}, nil
	}

	return &domain.DeleteAttributePayload{
		Success:          true,
		DeletedAttributeID: &id,
		Errors:           []*domain.UserError{},
		ClientMutationID: clientMutationID,
	}, nil
}

func convertError(err error) *domain.UserError {
	// Convert domain errors to GraphQL user errors
	switch e := err.(type) {
	case *domain.AttributeError:
		return &domain.UserError{
			Field:   e.Field,
			Message: e.Message,
			Code:    e.Code,
			Details: e.Details,
		}
	default:
		return &domain.UserError{
			Message: err.Error(),
			Code:    "INTERNAL_ERROR",
		}
	}
}
```

This comprehensive Go implementation specification provides all the necessary patterns and structures to implement the Attribute entity using Go, gqlgen, and PostgreSQL with proper multi-tenant architecture, comprehensive error handling, performance optimization, and maintainable code organization.