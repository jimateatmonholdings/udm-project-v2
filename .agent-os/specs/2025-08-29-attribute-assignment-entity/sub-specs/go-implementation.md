# Attribute Assignment Entity - Go Implementation Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**Language:** Go + gqlgen + PostgreSQL  
**Entity:** Attribute Assignment - Bridge connecting Thing Classes to Attributes with constraints  

## Implementation Architecture Overview

The Attribute Assignment entity implementation follows advanced Go patterns for junction table management with rich metadata and constraint handling. This specification provides comprehensive patterns for implementing the critical bridge between Thing Classes and Attributes, including schema composition, impact analysis, and validation rule processing within the gqlgen GraphQL framework and PostgreSQL multi-tenant architecture.

## Project Structure

### Go Module Organization

```
github.com/company/udm-backend/
├── cmd/
│   └── server/
│       └── main.go                          # Application entry point
├── internal/
│   ├── domain/                              # Domain models and business entities
│   │   ├── assignment.go                    # Assignment entity and types
│   │   ├── schema_composition.go            # Schema composition logic
│   │   ├── validation_rules.go              # Assignment validation structures
│   │   └── impact_analysis.go               # Impact analysis types
│   ├── graph/                               # GraphQL layer (gqlgen)
│   │   ├── generated/                       # gqlgen generated code
│   │   ├── resolvers/                       # Custom resolver implementations
│   │   │   ├── assignment.go                # Assignment resolvers
│   │   │   ├── schema_composition.go        # Schema composition resolvers
│   │   │   └── assignment_mutation.go       # Assignment mutation resolvers
│   │   ├── directives/                      # Custom GraphQL directives
│   │   │   ├── assignment_validation.go     # Assignment validation directive
│   │   │   └── impact_analysis.go           # Impact analysis directive
│   │   ├── schema.graphqls                  # GraphQL schema definition
│   │   └── gqlgen.yml                       # gqlgen configuration
│   ├── repository/                          # Data access layer
│   │   ├── interfaces/                      # Repository interfaces
│   │   │   └── assignment.go                # Assignment repository interface
│   │   └── postgres/                        # PostgreSQL implementations
│   │       ├── assignment.go                # Assignment repository implementation
│   │       ├── schema_composition.go        # Schema composition queries
│   │       └── assignment_queries.go        # SQL query definitions
│   ├── service/                             # Business logic layer
│   │   ├── assignment.go                    # Assignment service implementation
│   │   ├── schema_composition.go            # Schema composition service
│   │   ├── impact_analysis.go               # Impact analysis service
│   │   └── assignment_validation/           # Assignment validation processors
│   │       ├── constraint_validator.go      # Constraint validation
│   │       ├── default_value_validator.go   # Default value validation
│   │       └── rule_merger.go               # Validation rule merging
│   ├── cache/                               # Caching layer
│   │   ├── assignment_cache.go              # Assignment-specific caching
│   │   └── schema_cache.go                  # Schema composition caching
│   └── middleware/                          # HTTP middleware
│       ├── assignment_tracking.go           # Assignment change tracking
│       └── schema_validation.go             # Schema validation middleware
├── migrations/                              # Database migrations
│   ├── 004_create_assignments.up.sql
│   └── 004_create_assignments.down.sql
├── scripts/                                 # Development scripts
│   ├── generate_assignment_schema.sh        # Assignment schema generation
│   └── validate_assignments.sh              # Assignment validation
├── go.mod
├── go.sum
└── README.md
```

## Domain Model Implementation

### Core Assignment Entity

```go
// internal/domain/assignment.go
package domain

import (
    "encoding/json"
    "fmt"
    "time"

    "github.com/google/uuid"
)

// ThingClassAttribute represents the assignment bridge between Thing Classes and Attributes
type ThingClassAttribute struct {
    BaseEntity
    ThingClassID              uuid.UUID       `json:"thingClassId" db:"thing_class_id" validate:"required"`
    AttributeID               uuid.UUID       `json:"attributeId" db:"attribute_id" validate:"required"`
    IsRequired                bool            `json:"isRequired" db:"is_required"`
    SortOrder                 int             `json:"sortOrder" db:"sort_order" validate:"min=0,max=999999"`
    DisplayName               *string         `json:"displayName" db:"display_name" validate:"omitempty,min=1,max=255"`
    AssignmentValidationRules json.RawMessage `json:"assignmentValidationRules" db:"assignment_validation_rules"`
    DefaultValue              json.RawMessage `json:"defaultValue" db:"default_value"`
    UIHints                   json.RawMessage `json:"uiHints" db:"ui_hints"`
    Description               *string         `json:"description" db:"description" validate:"omitempty,max=1000"`
    AssignmentType            AssignmentType  `json:"assignmentType" db:"assignment_type" validate:"required"`
    Visibility                VisibilityType  `json:"visibility" db:"visibility" validate:"required"`

    // Loaded relationships (not persisted, populated by repository/service)
    ThingClass        *ThingClass        `json:"thingClass,omitempty" db:"-"`
    Attribute         *Attribute         `json:"attribute,omitempty" db:"-"`
    UsageCount        *int               `json:"usageCount,omitempty" db:"-"`
    ValidationSummary *ValidationSummary `json:"validationSummary,omitempty" db:"-"`
}

// Assignment type enumeration
type AssignmentType string

const (
    AssignmentTypeStandard  AssignmentType = "standard"
    AssignmentTypeComputed  AssignmentType = "computed"
    AssignmentTypeInherited AssignmentType = "inherited"
)

// String returns string representation
func (at AssignmentType) String() string {
    return string(at)
}

// IsValid validates if the AssignmentType is supported
func (at AssignmentType) IsValid() bool {
    switch at {
    case AssignmentTypeStandard, AssignmentTypeComputed, AssignmentTypeInherited:
        return true
    default:
        return false
    }
}

// Visibility type enumeration
type VisibilityType string

const (
    VisibilityTypeVisible  VisibilityType = "visible"
    VisibilityTypeHidden   VisibilityType = "hidden"
    VisibilityTypeReadonly VisibilityType = "readonly"
)

// String returns string representation
func (vt VisibilityType) String() string {
    return string(vt)
}

// IsValid validates if the VisibilityType is supported
func (vt VisibilityType) IsValid() bool {
    switch vt {
    case VisibilityTypeVisible, VisibilityTypeHidden, VisibilityTypeReadonly:
        return true
    default:
        return false
    }
}

// ValidationSummary provides comprehensive validation information for an assignment
type ValidationSummary struct {
    HasBaseValidation       bool            `json:"hasBaseValidation"`
    HasAssignmentValidation bool            `json:"hasAssignmentValidation"`
    HasDefaultValue         bool            `json:"hasDefaultValue"`
    EffectiveRules          json.RawMessage `json:"effectiveRules"`
    RulesSummary            string          `json:"rulesSummary"`
    ConflictWarnings        []string        `json:"conflictWarnings"`
}

// GetEffectiveDisplayName returns the display name or falls back to the attribute name
func (tca *ThingClassAttribute) GetEffectiveDisplayName() string {
    if tca.DisplayName != nil && *tca.DisplayName != "" {
        return *tca.DisplayName
    }
    if tca.Attribute != nil {
        return tca.Attribute.Name
    }
    return ""
}

// HasCustomValidation checks if the assignment has custom validation rules
func (tca *ThingClassAttribute) HasCustomValidation() bool {
    return len(tca.AssignmentValidationRules) > 0 && string(tca.AssignmentValidationRules) != "{}"
}

// HasDefaultValue checks if the assignment has a default value configured
func (tca *ThingClassAttribute) HasDefaultValue() bool {
    return len(tca.DefaultValue) > 0
}

// IsVisible checks if the assignment is visible in UI
func (tca *ThingClassAttribute) IsVisible() bool {
    return tca.Visibility == VisibilityTypeVisible
}

// IsReadonly checks if the assignment is readonly
func (tca *ThingClassAttribute) IsReadonly() bool {
    return tca.Visibility == VisibilityTypeReadonly
}
```

### Schema Composition Types

```go
// internal/domain/schema_composition.go
package domain

import (
    "encoding/json"
    "time"
    "crypto/md5"
    "fmt"
    "sort"

    "github.com/google/uuid"
)

// ThingClassSchema represents the complete schema composition for a Thing Class
type ThingClassSchema struct {
    ThingClass      *ThingClass            `json:"thingClass"`
    Assignments     []*ThingClassAttribute `json:"assignments"`
    TotalCount      int                    `json:"totalCount"`
    RequiredCount   int                    `json:"requiredCount"`
    OptionalCount   int                    `json:"optionalCount"`
    ValidationRules json.RawMessage        `json:"validationRules"`
    SchemaHash      string                 `json:"schemaHash"`
    LastModified    time.Time              `json:"lastModified"`
}

// BuildValidationRules aggregates all validation rules for the schema
func (tcs *ThingClassSchema) BuildValidationRules() json.RawMessage {
    rules := make(map[string]interface{})
    required := make([]string, 0)
    
    for _, assignment := range tcs.Assignments {
        if assignment.IsRequired {
            required = append(required, assignment.GetEffectiveDisplayName())
        }
        
        if assignment.Attribute != nil {
            attrRules := make(map[string]interface{})
            
            // Add base attribute validation rules
            if len(assignment.Attribute.ValidationRules) > 0 {
                var baseRules map[string]interface{}
                json.Unmarshal(assignment.Attribute.ValidationRules, &baseRules)
                for k, v := range baseRules {
                    attrRules[k] = v
                }
            }
            
            // Add assignment-specific validation rules
            if assignment.HasCustomValidation() {
                var assignmentRules map[string]interface{}
                json.Unmarshal(assignment.AssignmentValidationRules, &assignmentRules)
                for k, v := range assignmentRules {
                    attrRules[k] = v
                }
            }
            
            if len(attrRules) > 0 {
                rules[assignment.GetEffectiveDisplayName()] = attrRules
            }
        }
    }
    
    if len(required) > 0 {
        rules["required"] = required
    }
    
    rulesBytes, _ := json.Marshal(rules)
    return json.RawMessage(rulesBytes)
}

// CalculateSchemaHash generates a hash for change detection
func (tcs *ThingClassSchema) CalculateSchemaHash() string {
    // Sort assignments by ID for consistent hashing
    sortedAssignments := make([]*ThingClassAttribute, len(tcs.Assignments))
    copy(sortedAssignments, tcs.Assignments)
    sort.Slice(sortedAssignments, func(i, j int) bool {
        return sortedAssignments[i].ID.String() < sortedAssignments[j].ID.String()
    })
    
    hashData := fmt.Sprintf("tc_%s", tcs.ThingClass.ID.String())
    for _, assignment := range sortedAssignments {
        hashData += fmt.Sprintf("_a_%s_%t_%d_%d", 
            assignment.ID.String(), 
            assignment.IsRequired, 
            assignment.SortOrder,
            assignment.Version)
    }
    
    hash := md5.Sum([]byte(hashData))
    return fmt.Sprintf("%x", hash)
}

// GetAssignmentBySortOrder returns assignment at specific sort order
func (tcs *ThingClassSchema) GetAssignmentBySortOrder(sortOrder int) *ThingClassAttribute {
    for _, assignment := range tcs.Assignments {
        if assignment.SortOrder == sortOrder {
            return assignment
        }
    }
    return nil
}

// GetAssignmentByAttributeID returns assignment for specific attribute
func (tcs *ThingClassSchema) GetAssignmentByAttributeID(attributeID uuid.UUID) *ThingClassAttribute {
    for _, assignment := range tcs.Assignments {
        if assignment.AttributeID == attributeID {
            return assignment
        }
    }
    return nil
}
```

### Input and Filter Types

```go
// Input types for GraphQL operations
type CreateAttributeAssignmentInput struct {
    ThingClassID              uuid.UUID       `json:"thingClassId" validate:"required"`
    AttributeID               uuid.UUID       `json:"attributeId" validate:"required"`
    IsRequired                bool            `json:"isRequired"`
    SortOrder                 int             `json:"sortOrder" validate:"min=0,max=999999"`
    DisplayName               *string         `json:"displayName,omitempty" validate:"omitempty,min=1,max=255"`
    AssignmentValidationRules json.RawMessage `json:"assignmentValidationRules,omitempty"`
    DefaultValue              json.RawMessage `json:"defaultValue,omitempty"`
    UIHints                   json.RawMessage `json:"uiHints,omitempty"`
    Description               *string         `json:"description,omitempty" validate:"omitempty,max=1000"`
    AssignmentType            AssignmentType  `json:"assignmentType" validate:"required"`
    Visibility                VisibilityType  `json:"visibility" validate:"required"`
}

type UpdateAttributeAssignmentInput struct {
    IsRequired                *bool           `json:"isRequired,omitempty"`
    SortOrder                 *int            `json:"sortOrder,omitempty" validate:"omitempty,min=0,max=999999"`
    DisplayName               *string         `json:"displayName,omitempty" validate:"omitempty,min=1,max=255"`
    AssignmentValidationRules json.RawMessage `json:"assignmentValidationRules,omitempty"`
    DefaultValue              json.RawMessage `json:"defaultValue,omitempty"`
    UIHints                   json.RawMessage `json:"uiHints,omitempty"`
    Description               *string         `json:"description,omitempty" validate:"omitempty,max=1000"`
    AssignmentType            *AssignmentType `json:"assignmentType,omitempty"`
    Visibility                *VisibilityType `json:"visibility,omitempty"`
}

// Bulk operations input
type BulkAssignmentInput struct {
    ThingClassID    uuid.UUID                        `json:"thingClassId" validate:"required"`
    Assignments     []*CreateAttributeAssignmentInput `json:"assignments" validate:"required,min=1,max=100"`
    ReplaceExisting bool                             `json:"replaceExisting"`
}

// Reordering input
type ReorderAssignmentsInput struct {
    ThingClassID      uuid.UUID              `json:"thingClassId" validate:"required"`
    AssignmentOrders  []*AssignmentOrderInput `json:"assignmentOrders" validate:"required,min=1"`
}

type AssignmentOrderInput struct {
    AssignmentID uuid.UUID `json:"assignmentId" validate:"required"`
    SortOrder    int       `json:"sortOrder" validate:"min=0,max=999999"`
}

// Filter types
type AttributeAssignmentFilter struct {
    ThingClassID            *uuid.UUID      `json:"thingClassId,omitempty"`
    AttributeID             *uuid.UUID      `json:"attributeId,omitempty"`
    IsRequired              *bool           `json:"isRequired,omitempty"`
    AssignmentType          *AssignmentType `json:"assignmentType,omitempty"`
    Visibility              *VisibilityType `json:"visibility,omitempty"`
    HasDefaultValue         *bool           `json:"hasDefaultValue,omitempty"`
    HasCustomValidation     *bool           `json:"hasCustomValidation,omitempty"`
    CreatedBy               *uuid.UUID      `json:"createdBy,omitempty"`
    CreatedAfter            *time.Time      `json:"createdAfter,omitempty"`
    CreatedBefore           *time.Time      `json:"createdBefore,omitempty"`
    DisplayNameContains     *string         `json:"displayNameContains,omitempty"`
    SortOrderRange          *IntRange       `json:"sortOrderRange,omitempty"`
}

type IntRange struct {
    Min int `json:"min"`
    Max int `json:"max"`
}

// Sort types
type AttributeAssignmentSort struct {
    Field     AttributeAssignmentSortField `json:"field"`
    Direction SortDirection                `json:"direction"`
}

type AttributeAssignmentSortField string

const (
    AttributeAssignmentSortFieldSortOrder     AttributeAssignmentSortField = "SORT_ORDER"
    AttributeAssignmentSortFieldDisplayName   AttributeAssignmentSortField = "DISPLAY_NAME"
    AttributeAssignmentSortFieldCreatedAt     AttributeAssignmentSortField = "CREATED_AT"
    AttributeAssignmentSortFieldUpdatedAt     AttributeAssignmentSortField = "UPDATED_AT"
    AttributeAssignmentSortFieldIsRequired    AttributeAssignmentSortField = "IS_REQUIRED"
    AttributeAssignmentSortFieldUsageCount    AttributeAssignmentSortField = "USAGE_COUNT"
    AttributeAssignmentSortFieldAssignmentType AttributeAssignmentSortField = "ASSIGNMENT_TYPE"
    AttributeAssignmentSortFieldVisibility    AttributeAssignmentSortField = "VISIBILITY"
)

// Pagination types
type AttributeAssignmentConnection struct {
    Edges      []*AttributeAssignmentEdge `json:"edges"`
    PageInfo   *PageInfo                  `json:"pageInfo"`
    TotalCount int                        `json:"totalCount"`
}

type AttributeAssignmentEdge struct {
    Node   *ThingClassAttribute `json:"node"`
    Cursor string               `json:"cursor"`
}

type ThingClassSchemaConnection struct {
    Edges      []*ThingClassSchemaEdge `json:"edges"`
    PageInfo   *PageInfo               `json:"pageInfo"`
    TotalCount int                     `json:"totalCount"`
}

type ThingClassSchemaEdge struct {
    Node   *ThingClassSchema `json:"node"`
    Cursor string            `json:"cursor"`
}
```

### Impact Analysis Types

```go
// internal/domain/impact_analysis.go
package domain

import (
    "encoding/json"
    "time"

    "github.com/google/uuid"
)

// AssignmentImpactAnalysis provides comprehensive impact assessment for assignment changes
type AssignmentImpactAnalysis struct {
    Assignment              *ThingClassAttribute `json:"assignment"`
    ImpactType              ImpactType           `json:"impactType"`
    AffectedThingCount      int                  `json:"affectedThingCount"`
    RequiredThingCount      int                  `json:"requiredThingCount"`
    MissingValueCount       int                  `json:"missingValueCount"`
    ImpactDetails           json.RawMessage      `json:"impactDetails"`
    RecommendedActions      []string             `json:"recommendedActions"`
    MigrationRequired       bool                 `json:"migrationRequired"`
    EstimatedMigrationTime  *string              `json:"estimatedMigrationTime,omitempty"`
    RollbackPlan           *RollbackPlan        `json:"rollbackPlan,omitempty"`
}

type ImpactType string

const (
    ImpactTypeSafe              ImpactType = "SAFE"
    ImpactTypeWarning           ImpactType = "WARNING"
    ImpactTypeBreaking          ImpactType = "BREAKING"
    ImpactTypeMigrationRequired ImpactType = "MIGRATION_REQUIRED"
)

// RollbackPlan provides information about reversing assignment changes
type RollbackPlan struct {
    CanRollback       bool            `json:"canRollback"`
    RollbackSteps     []string        `json:"rollbackSteps"`
    RollbackData      json.RawMessage `json:"rollbackData"`
    EstimatedTime     *string         `json:"estimatedTime,omitempty"`
    RequiresDowntime  bool            `json:"requiresDowntime"`
}

// MigrationStrategy defines approaches for handling breaking assignment changes
type MigrationStrategy string

const (
    MigrationStrategyProvideDefaults     MigrationStrategy = "PROVIDE_DEFAULTS"
    MigrationStrategyMarkOptional        MigrationStrategy = "MARK_OPTIONAL"
    MigrationStrategyDeleteMissingValues MigrationStrategy = "DELETE_MISSING_VALUES"
    MigrationStrategyCustomScript        MigrationStrategy = "CUSTOM_SCRIPT"
)

type AssignmentMigrationInput struct {
    Strategy   MigrationStrategy `json:"strategy" validate:"required"`
    Parameters json.RawMessage   `json:"parameters,omitempty"`
    DryRun     bool              `json:"dryRun"`
}

type AssignmentMigrationResult struct {
    Strategy              MigrationStrategy `json:"strategy"`
    AffectedThingCount    int               `json:"affectedThingCount"`
    MigratedValueCount    int               `json:"migratedValueCount"`
    ErrorCount            int               `json:"errorCount"`
    ExecutionTimeMs       int               `json:"executionTimeMs"`
    DryRun                bool              `json:"dryRun"`
    Details               json.RawMessage   `json:"details"`
}

// BuildImpactDetails creates comprehensive impact analysis details
func (aia *AssignmentImpactAnalysis) BuildImpactDetails(changeType string, oldValue, newValue interface{}) {
    details := map[string]interface{}{
        "changeType":           changeType,
        "timestamp":            time.Now(),
        "affectedThings":       aia.AffectedThingCount,
        "missingValues":        aia.MissingValueCount,
        "migrationRequired":    aia.MigrationRequired,
        "impactType":           aia.ImpactType,
    }
    
    if oldValue != nil {
        details["oldValue"] = oldValue
    }
    if newValue != nil {
        details["newValue"] = newValue
    }
    
    detailsBytes, _ := json.Marshal(details)
    aia.ImpactDetails = json.RawMessage(detailsBytes)
}

// EstimateMigrationTime provides time estimates based on affected entity counts
func (aia *AssignmentImpactAnalysis) EstimateMigrationTime() string {
    if !aia.MigrationRequired {
        return "No migration required"
    }
    
    switch {
    case aia.AffectedThingCount < 100:
        return "< 1 minute"
    case aia.AffectedThingCount < 1000:
        return "1-5 minutes"
    case aia.AffectedThingCount < 10000:
        return "5-30 minutes"
    default:
        return "30+ minutes"
    }
}
```

## Service Layer Implementation

### Assignment Business Logic Service

```go
// internal/service/assignment.go
package service

import (
    "context"
    "encoding/json"
    "fmt"
    "time"

    "github.com/company/udm-backend/internal/domain"
    "github.com/company/udm-backend/internal/repository/interfaces"
    "github.com/company/udm-backend/internal/service/assignment_validation"
    "github.com/go-playground/validator/v10"
    "github.com/google/uuid"
    "go.uber.org/zap"
)

type AssignmentService struct {
    repo                 interfaces.AssignmentRepository
    thingClassRepo       interfaces.ThingClassRepository
    attributeRepo        interfaces.AttributeRepository
    thingRepo            interfaces.ThingRepository
    validator            *validator.Validate
    constraintValidator  *assignment_validation.ConstraintValidator
    impactAnalyzer       *ImpactAnalysisService
    cache                interfaces.Cache
    logger               *zap.Logger
}

func NewAssignmentService(
    repo interfaces.AssignmentRepository,
    thingClassRepo interfaces.ThingClassRepository,
    attributeRepo interfaces.AttributeRepository,
    thingRepo interfaces.ThingRepository,
    constraintValidator *assignment_validation.ConstraintValidator,
    impactAnalyzer *ImpactAnalysisService,
    cache interfaces.Cache,
    logger *zap.Logger,
) *AssignmentService {
    return &AssignmentService{
        repo:                repo,
        thingClassRepo:      thingClassRepo,
        attributeRepo:       attributeRepo,
        thingRepo:           thingRepo,
        validator:           validator.New(),
        constraintValidator: constraintValidator,
        impactAnalyzer:      impactAnalyzer,
        cache:               cache,
        logger:              logger,
    }
}

// CreateAttributeAssignment creates a new assignment with comprehensive validation
func (s *AssignmentService) CreateAttributeAssignment(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateAttributeAssignmentInput) (*domain.ThingClassAttribute, error) {
    // Input validation
    if err := s.validateCreateInput(input); err != nil {
        return nil, domain.NewValidationError("input validation failed", err)
    }

    // Business rule validation
    if err := s.validateBusinessRules(ctx, tenant, input); err != nil {
        return nil, fmt.Errorf("business rule validation failed: %w", err)
    }

    // Check for existing assignment
    exists, err := s.repo.ExistsAssignment(ctx, tenant, input.ThingClassID, input.AttributeID)
    if err != nil {
        return nil, fmt.Errorf("failed to check existing assignment: %w", err)
    }
    if exists {
        return nil, domain.NewAssignmentAlreadyExistsError(input.ThingClassID, input.AttributeID)
    }

    // Validate sort order uniqueness
    if err := s.validateSortOrderUniqueness(ctx, tenant, input.ThingClassID, input.SortOrder, uuid.Nil); err != nil {
        return nil, fmt.Errorf("sort order validation failed: %w", err)
    }

    // Validate assignment configuration
    if err := s.constraintValidator.ValidateAssignmentConfiguration(ctx, tenant, input); err != nil {
        return nil, fmt.Errorf("assignment configuration validation failed: %w", err)
    }

    // Create domain object
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
        ThingClassID:              input.ThingClassID,
        AttributeID:               input.AttributeID,
        IsRequired:                input.IsRequired,
        SortOrder:                 input.SortOrder,
        DisplayName:               input.DisplayName,
        AssignmentValidationRules: input.AssignmentValidationRules,
        DefaultValue:              input.DefaultValue,
        UIHints:                   input.UIHints,
        Description:               input.Description,
        AssignmentType:            input.AssignmentType,
        Visibility:                input.Visibility,
    }

    // Persist to database
    created, err := s.repo.Create(ctx, tenant, assignment)
    if err != nil {
        s.logger.Error("Failed to create assignment",
            zap.String("thing_class_id", input.ThingClassID.String()),
            zap.String("attribute_id", input.AttributeID.String()),
            zap.String("tenant_schema", tenant.SchemaName),
            zap.Error(err),
        )
        return nil, fmt.Errorf("failed to create assignment: %w", err)
    }

    // Load relationships
    if err := s.loadAssignmentRelationships(ctx, tenant, created); err != nil {
        s.logger.Warn("Failed to load assignment relationships",
            zap.String("assignment_id", created.ID.String()),
            zap.Error(err),
        )
        // Don't fail the operation, just log the warning
    }

    // Invalidate related caches
    s.invalidateAssignmentCaches(ctx, tenant, input.ThingClassID)

    s.logger.Info("Successfully created assignment",
        zap.String("id", created.ID.String()),
        zap.String("thing_class_id", input.ThingClassID.String()),
        zap.String("attribute_id", input.AttributeID.String()),
        zap.Bool("is_required", created.IsRequired),
        zap.Int("sort_order", created.SortOrder),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return created, nil
}

// UpdateAttributeAssignment updates an existing assignment with impact analysis
func (s *AssignmentService) UpdateAttributeAssignment(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID, input *domain.UpdateAttributeAssignmentInput) (*domain.ThingClassAttribute, error) {
    // Retrieve existing assignment
    existing, err := s.repo.GetByID(ctx, tenant, id)
    if err != nil {
        return nil, fmt.Errorf("failed to get assignment: %w", err)
    }
    if existing == nil {
        return nil, domain.NewAssignmentNotFoundError(id)
    }

    // Input validation
    if err := s.validateUpdateInput(input); err != nil {
        return nil, domain.NewValidationError("input validation failed", err)
    }

    // Analyze impact of changes
    impactAnalysis, err := s.impactAnalyzer.AnalyzeAssignmentUpdate(ctx, tenant, existing, input)
    if err != nil {
        return nil, fmt.Errorf("failed to analyze impact: %w", err)
    }

    // Block breaking changes if no migration plan
    if impactAnalysis.ImpactType == domain.ImpactTypeBreaking && !impactAnalysis.MigrationRequired {
        return nil, fmt.Errorf("breaking change detected: %s", impactAnalysis.RecommendedActions)
    }

    // Apply updates
    updated := *existing
    updated.UpdatedAt = time.Now()
    updated.Version++

    hasChanges := false

    if input.IsRequired != nil && *input.IsRequired != existing.IsRequired {
        updated.IsRequired = *input.IsRequired
        hasChanges = true
    }

    if input.SortOrder != nil && *input.SortOrder != existing.SortOrder {
        // Validate sort order uniqueness
        if err := s.validateSortOrderUniqueness(ctx, tenant, existing.ThingClassID, *input.SortOrder, existing.ID); err != nil {
            return nil, fmt.Errorf("sort order validation failed: %w", err)
        }
        updated.SortOrder = *input.SortOrder
        hasChanges = true
    }

    if input.DisplayName != nil {
        updated.DisplayName = input.DisplayName
        hasChanges = true
    }

    if len(input.AssignmentValidationRules) > 0 {
        // Validate new validation rules
        if err := s.constraintValidator.ValidateAssignmentValidationRules(ctx, tenant, existing.AttributeID, input.AssignmentValidationRules); err != nil {
            return nil, fmt.Errorf("validation rules validation failed: %w", err)
        }
        updated.AssignmentValidationRules = input.AssignmentValidationRules
        hasChanges = true
    }

    if len(input.DefaultValue) > 0 {
        // Validate new default value
        if err := s.constraintValidator.ValidateDefaultValue(ctx, tenant, existing.AttributeID, input.DefaultValue); err != nil {
            return nil, fmt.Errorf("default value validation failed: %w", err)
        }
        updated.DefaultValue = input.DefaultValue
        hasChanges = true
    }

    if len(input.UIHints) > 0 {
        updated.UIHints = input.UIHints
        hasChanges = true
    }

    if input.Description != nil {
        updated.Description = input.Description
        hasChanges = true
    }

    if input.AssignmentType != nil && *input.AssignmentType != existing.AssignmentType {
        updated.AssignmentType = *input.AssignmentType
        hasChanges = true
    }

    if input.Visibility != nil && *input.Visibility != existing.Visibility {
        updated.Visibility = *input.Visibility
        hasChanges = true
    }

    if !hasChanges {
        return existing, nil // No changes to apply
    }

    // Persist changes
    result, err := s.repo.Update(ctx, tenant, &updated)
    if err != nil {
        s.logger.Error("Failed to update assignment",
            zap.String("id", id.String()),
            zap.String("tenant_schema", tenant.SchemaName),
            zap.Error(err),
        )
        return nil, fmt.Errorf("failed to update assignment: %w", err)
    }

    // Load relationships
    if err := s.loadAssignmentRelationships(ctx, tenant, result); err != nil {
        s.logger.Warn("Failed to load assignment relationships after update",
            zap.String("assignment_id", result.ID.String()),
            zap.Error(err),
        )
    }

    // Invalidate caches
    s.invalidateAssignmentCaches(ctx, tenant, result.ThingClassID)

    s.logger.Info("Successfully updated assignment",
        zap.String("id", id.String()),
        zap.Int("version", result.Version),
        zap.String("impact_type", string(impactAnalysis.ImpactType)),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return result, nil
}

// GetAttributeAssignment retrieves an assignment by ID with full relationships
func (s *AssignmentService) GetAttributeAssignment(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.ThingClassAttribute, error) {
    // Try cache first
    cacheKey := fmt.Sprintf("assignment:%s", id.String())
    var cached domain.ThingClassAttribute
    if err := s.cache.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
        return &cached, nil
    }

    // Fetch from database
    assignment, err := s.repo.GetByID(ctx, tenant, id)
    if err != nil {
        s.logger.Error("Failed to get assignment",
            zap.String("id", id.String()),
            zap.String("tenant_schema", tenant.SchemaName),
            zap.Error(err),
        )
        return nil, fmt.Errorf("failed to get assignment: %w", err)
    }

    if assignment == nil {
        return nil, domain.NewAssignmentNotFoundError(id)
    }

    // Load relationships
    if err := s.loadAssignmentRelationships(ctx, tenant, assignment); err != nil {
        s.logger.Warn("Failed to load assignment relationships",
            zap.String("assignment_id", assignment.ID.String()),
            zap.Error(err),
        )
    }

    // Cache the result
    s.cache.Set(ctx, tenant.SchemaName, cacheKey, assignment)

    return assignment, nil
}

// ListAssignmentsByThingClass retrieves assignments for a specific Thing Class
func (s *AssignmentService) ListAssignmentsByThingClass(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID, filter *domain.AttributeAssignmentFilter, sorts []*domain.AttributeAssignmentSort, limit int, offset int) ([]*domain.ThingClassAttribute, int, error) {
    // Create filter for Thing Class
    if filter == nil {
        filter = &domain.AttributeAssignmentFilter{}
    }
    filter.ThingClassID = &thingClassID

    return s.ListAssignments(ctx, tenant, filter, sorts, limit, offset)
}

// ListAssignments retrieves paginated assignments with filtering and sorting
func (s *AssignmentService) ListAssignments(ctx context.Context, tenant *domain.TenantContext, filter *domain.AttributeAssignmentFilter, sorts []*domain.AttributeAssignmentSort, limit int, offset int) ([]*domain.ThingClassAttribute, int, error) {
    assignments, total, err := s.repo.List(ctx, tenant, filter, sorts, limit, offset)
    if err != nil {
        s.logger.Error("Failed to list assignments",
            zap.String("tenant_schema", tenant.SchemaName),
            zap.Error(err),
        )
        return nil, 0, fmt.Errorf("failed to list assignments: %w", err)
    }

    // Load relationships for all assignments
    for _, assignment := range assignments {
        if err := s.loadAssignmentRelationships(ctx, tenant, assignment); err != nil {
            s.logger.Warn("Failed to load relationships for assignment in list",
                zap.String("assignment_id", assignment.ID.String()),
                zap.Error(err),
            )
        }
    }

    return assignments, total, nil
}

// DeleteAssignment performs soft delete with impact analysis
func (s *AssignmentService) DeleteAssignment(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID, force bool) (*domain.AssignmentImpactAnalysis, error) {
    // Check if assignment exists
    assignment, err := s.repo.GetByID(ctx, tenant, id)
    if err != nil {
        return nil, fmt.Errorf("failed to get assignment: %w", err)
    }
    if assignment == nil {
        return nil, domain.NewAssignmentNotFoundError(id)
    }

    // Analyze impact of deletion
    impactAnalysis, err := s.impactAnalyzer.AnalyzeAssignmentDeletion(ctx, tenant, assignment)
    if err != nil {
        return nil, fmt.Errorf("failed to analyze deletion impact: %w", err)
    }

    // Block breaking deletions unless forced
    if impactAnalysis.ImpactType == domain.ImpactTypeBreaking && !force {
        return impactAnalysis, fmt.Errorf("breaking deletion blocked - use force flag to proceed: %s", impactAnalysis.RecommendedActions)
    }

    // Perform soft delete
    if err := s.repo.SoftDelete(ctx, tenant, id); err != nil {
        s.logger.Error("Failed to delete assignment",
            zap.String("id", id.String()),
            zap.String("tenant_schema", tenant.SchemaName),
            zap.Error(err),
        )
        return nil, fmt.Errorf("failed to delete assignment: %w", err)
    }

    // Invalidate caches
    s.invalidateAssignmentCaches(ctx, tenant, assignment.ThingClassID)

    s.logger.Info("Successfully deleted assignment",
        zap.String("id", id.String()),
        zap.String("thing_class_id", assignment.ThingClassID.String()),
        zap.String("attribute_id", assignment.AttributeID.String()),
        zap.String("impact_type", string(impactAnalysis.ImpactType)),
        zap.Bool("forced", force),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return impactAnalysis, nil
}

// Private helper methods

func (s *AssignmentService) validateCreateInput(input *domain.CreateAttributeAssignmentInput) error {
    if err := s.validator.Struct(input); err != nil {
        return err
    }

    // Validate assignment type
    if !input.AssignmentType.IsValid() {
        return fmt.Errorf("invalid assignment type: %s", input.AssignmentType)
    }

    // Validate visibility type
    if !input.Visibility.IsValid() {
        return fmt.Errorf("invalid visibility type: %s", input.Visibility)
    }

    return nil
}

func (s *AssignmentService) validateUpdateInput(input *domain.UpdateAttributeAssignmentInput) error {
    if err := s.validator.Struct(input); err != nil {
        return err
    }

    if input.AssignmentType != nil && !input.AssignmentType.IsValid() {
        return fmt.Errorf("invalid assignment type: %s", *input.AssignmentType)
    }

    if input.Visibility != nil && !input.Visibility.IsValid() {
        return fmt.Errorf("invalid visibility type: %s", *input.Visibility)
    }

    return nil
}

func (s *AssignmentService) validateBusinessRules(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateAttributeAssignmentInput) error {
    // Validate Thing Class exists
    thingClass, err := s.thingClassRepo.GetByID(ctx, tenant, input.ThingClassID)
    if err != nil {
        return fmt.Errorf("failed to validate Thing Class: %w", err)
    }
    if thingClass == nil {
        return fmt.Errorf("Thing Class not found: %s", input.ThingClassID)
    }

    // Validate Attribute exists
    attribute, err := s.attributeRepo.GetByID(ctx, tenant, input.AttributeID)
    if err != nil {
        return fmt.Errorf("failed to validate Attribute: %w", err)
    }
    if attribute == nil {
        return fmt.Errorf("Attribute not found: %s", input.AttributeID)
    }

    return nil
}

func (s *AssignmentService) validateSortOrderUniqueness(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID, sortOrder int, excludeID uuid.UUID) error {
    exists, err := s.repo.ExistsSortOrder(ctx, tenant, thingClassID, sortOrder, excludeID)
    if err != nil {
        return fmt.Errorf("failed to check sort order uniqueness: %w", err)
    }
    if exists {
        return fmt.Errorf("sort order %d already exists for Thing Class %s", sortOrder, thingClassID)
    }
    return nil
}

func (s *AssignmentService) loadAssignmentRelationships(ctx context.Context, tenant *domain.TenantContext, assignment *domain.ThingClassAttribute) error {
    // Load Thing Class
    if assignment.ThingClass == nil {
        thingClass, err := s.thingClassRepo.GetByID(ctx, tenant, assignment.ThingClassID)
        if err != nil {
            return fmt.Errorf("failed to load Thing Class: %w", err)
        }
        assignment.ThingClass = thingClass
    }

    // Load Attribute
    if assignment.Attribute == nil {
        attribute, err := s.attributeRepo.GetByID(ctx, tenant, assignment.AttributeID)
        if err != nil {
            return fmt.Errorf("failed to load Attribute: %w", err)
        }
        assignment.Attribute = attribute
    }

    // Load usage count
    if assignment.UsageCount == nil {
        usageCount, err := s.repo.GetUsageCount(ctx, tenant, assignment.ID)
        if err != nil {
            s.logger.Warn("Failed to load usage count", zap.Error(err))
        } else {
            assignment.UsageCount = &usageCount
        }
    }

    // Build validation summary
    if assignment.ValidationSummary == nil {
        validationSummary := s.buildValidationSummary(assignment)
        assignment.ValidationSummary = validationSummary
    }

    return nil
}

func (s *AssignmentService) buildValidationSummary(assignment *domain.ThingClassAttribute) *domain.ValidationSummary {
    summary := &domain.ValidationSummary{
        HasBaseValidation:       false,
        HasAssignmentValidation: assignment.HasCustomValidation(),
        HasDefaultValue:         assignment.HasDefaultValue(),
        ConflictWarnings:        []string{},
    }

    // Check for base validation rules
    if assignment.Attribute != nil && len(assignment.Attribute.ValidationRules) > 0 {
        summary.HasBaseValidation = true
    }

    // Build effective rules by merging base and assignment rules
    effectiveRules := make(map[string]interface{})

    if summary.HasBaseValidation {
        var baseRules map[string]interface{}
        json.Unmarshal(assignment.Attribute.ValidationRules, &baseRules)
        for k, v := range baseRules {
            effectiveRules[k] = v
        }
    }

    if summary.HasAssignmentValidation {
        var assignmentRules map[string]interface{}
        json.Unmarshal(assignment.AssignmentValidationRules, &assignmentRules)
        for k, v := range assignmentRules {
            if _, exists := effectiveRules[k]; exists {
                summary.ConflictWarnings = append(summary.ConflictWarnings, 
                    fmt.Sprintf("Assignment rule '%s' overrides base attribute rule", k))
            }
            effectiveRules[k] = v
        }
    }

    effectiveRulesBytes, _ := json.Marshal(effectiveRules)
    summary.EffectiveRules = json.RawMessage(effectiveRulesBytes)

    // Build rules summary
    summary.RulesSummary = s.buildRulesSummary(effectiveRules)

    return summary
}

func (s *AssignmentService) buildRulesSummary(rules map[string]interface{}) string {
    if len(rules) == 0 {
        return "No validation rules"
    }

    summary := fmt.Sprintf("%d validation rule(s)", len(rules))
    
    if required, exists := rules["required"]; exists && required == true {
        summary += ", required"
    }
    
    if minLength, exists := rules["minLength"]; exists {
        summary += fmt.Sprintf(", min length: %v", minLength)
    }
    
    if maxLength, exists := rules["maxLength"]; exists {
        summary += fmt.Sprintf(", max length: %v", maxLength)
    }

    return summary
}

func (s *AssignmentService) invalidateAssignmentCaches(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID) {
    patterns := []string{
        "assignment:*",
        "assignments:list:*",
        fmt.Sprintf("thing_class_schema:%s", thingClassID.String()),
        "thing_class_schemas:*",
        "assignment_stats:*",
    }
    
    for _, pattern := range patterns {
        s.cache.InvalidatePattern(ctx, tenant.SchemaName, pattern)
    }
}
```

## Schema Composition Service

```go
// internal/service/schema_composition.go
package service

import (
    "context"
    "fmt"
    "sort"
    "time"

    "github.com/company/udm-backend/internal/domain"
    "github.com/company/udm-backend/internal/repository/interfaces"
    "github.com/google/uuid"
    "go.uber.org/zap"
)

type SchemaCompositionService struct {
    assignmentRepo interfaces.AssignmentRepository
    thingClassRepo interfaces.ThingClassRepository
    cache          interfaces.Cache
    logger         *zap.Logger
}

func NewSchemaCompositionService(
    assignmentRepo interfaces.AssignmentRepository,
    thingClassRepo interfaces.ThingClassRepository,
    cache interfaces.Cache,
    logger *zap.Logger,
) *SchemaCompositionService {
    return &SchemaCompositionService{
        assignmentRepo: assignmentRepo,
        thingClassRepo: thingClassRepo,
        cache:          cache,
        logger:         logger,
    }
}

// GetThingClassSchema builds complete schema composition for a Thing Class
func (s *SchemaCompositionService) GetThingClassSchema(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID) (*domain.ThingClassSchema, error) {
    // Try cache first
    cacheKey := fmt.Sprintf("thing_class_schema:%s", thingClassID.String())
    var cached domain.ThingClassSchema
    if err := s.cache.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
        return &cached, nil
    }

    // Get Thing Class
    thingClass, err := s.thingClassRepo.GetByID(ctx, tenant, thingClassID)
    if err != nil {
        return nil, fmt.Errorf("failed to get Thing Class: %w", err)
    }
    if thingClass == nil {
        return nil, fmt.Errorf("Thing Class not found: %s", thingClassID)
    }

    // Get all assignments for this Thing Class
    filter := &domain.AttributeAssignmentFilter{
        ThingClassID: &thingClassID,
    }
    sorts := []*domain.AttributeAssignmentSort{
        {
            Field:     domain.AttributeAssignmentSortFieldSortOrder,
            Direction: domain.SortDirectionAsc,
        },
        {
            Field:     domain.AttributeAssignmentSortFieldCreatedAt,
            Direction: domain.SortDirectionAsc,
        },
    }

    assignments, _, err := s.assignmentRepo.List(ctx, tenant, filter, sorts, 1000, 0) // Large limit to get all
    if err != nil {
        return nil, fmt.Errorf("failed to get assignments: %w", err)
    }

    // Build schema
    schema := &domain.ThingClassSchema{
        ThingClass:   thingClass,
        Assignments:  assignments,
        TotalCount:   len(assignments),
        LastModified: s.getLatestModificationTime(assignments),
    }

    // Calculate counts
    for _, assignment := range assignments {
        if assignment.IsRequired {
            schema.RequiredCount++
        } else {
            schema.OptionalCount++
        }
    }

    // Build validation rules
    schema.ValidationRules = schema.BuildValidationRules()

    // Calculate schema hash
    schema.SchemaHash = schema.CalculateSchemaHash()

    // Cache the result
    s.cache.SetWithTTL(ctx, tenant.SchemaName, cacheKey, schema, 10*time.Minute)

    s.logger.Debug("Built Thing Class schema",
        zap.String("thing_class_id", thingClassID.String()),
        zap.Int("assignment_count", schema.TotalCount),
        zap.Int("required_count", schema.RequiredCount),
        zap.String("schema_hash", schema.SchemaHash),
    )

    return schema, nil
}

// ValidateThingClassSchema validates a complete schema configuration
func (s *SchemaCompositionService) ValidateThingClassSchema(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID, assignments []*domain.CreateAttributeAssignmentInput) (*domain.ThingClassSchemaValidationResult, error) {
    // Get Thing Class
    thingClass, err := s.thingClassRepo.GetByID(ctx, tenant, thingClassID)
    if err != nil {
        return nil, fmt.Errorf("failed to get Thing Class: %w", err)
    }
    if thingClass == nil {
        return nil, fmt.Errorf("Thing Class not found: %s", thingClassID)
    }

    result := &domain.ThingClassSchemaValidationResult{
        ThingClass:         thingClass,
        IsValid:            true,
        ValidAssignments:   []*domain.CreateAttributeAssignmentInput{},
        InvalidAssignments: []*domain.InvalidAssignmentResult{},
        Errors:             []*domain.UserError{},
        Warnings:           []*domain.ValidationWarning{},
        Suggestions:        []*domain.ValidationSuggestion{},
    }

    // Validate each assignment
    sortOrdersUsed := make(map[int]bool)
    attributeIDsUsed := make(map[uuid.UUID]bool)

    for _, assignment := range assignments {
        invalid := &domain.InvalidAssignmentResult{
            Input:       assignment,
            Errors:      []*domain.ValidationError{},
            CanAutoFix:  false,
        }

        // Check for duplicate Attribute ID
        if attributeIDsUsed[assignment.AttributeID] {
            invalid.Errors = append(invalid.Errors, &domain.ValidationError{
                Field:   "attributeId",
                Message: "Attribute is already assigned to this Thing Class",
                Code:    "DUPLICATE_ATTRIBUTE",
            })
        } else {
            attributeIDsUsed[assignment.AttributeID] = true
        }

        // Check for duplicate sort order
        if sortOrdersUsed[assignment.SortOrder] {
            invalid.Errors = append(invalid.Errors, &domain.ValidationError{
                Field:   "sortOrder",
                Message: "Sort order is already used by another assignment",
                Code:    "DUPLICATE_SORT_ORDER",
            })
            // This can be auto-fixed by finding the next available sort order
            invalid.CanAutoFix = true
            if invalid.SuggestedFix == nil {
                suggestedFix := *assignment
                suggestedFix.SortOrder = s.findNextAvailableSortOrder(sortOrdersUsed)
                invalid.SuggestedFix = &suggestedFix
            }
        } else {
            sortOrdersUsed[assignment.SortOrder] = true
        }

        // Add other validations as needed...

        if len(invalid.Errors) > 0 {
            result.InvalidAssignments = append(result.InvalidAssignments, invalid)
            result.IsValid = false
        } else {
            result.ValidAssignments = append(result.ValidAssignments, assignment)
        }
    }

    return result, nil
}

// Private helper methods

func (s *SchemaCompositionService) getLatestModificationTime(assignments []*domain.ThingClassAttribute) time.Time {
    var latest time.Time
    for _, assignment := range assignments {
        if assignment.UpdatedAt.After(latest) {
            latest = assignment.UpdatedAt
        }
    }
    return latest
}

func (s *SchemaCompositionService) findNextAvailableSortOrder(usedOrders map[int]bool) int {
    for i := 0; i <= 999999; i++ {
        if !usedOrders[i] {
            return i
        }
    }
    return 999999 // Fallback
}
```

## Repository Layer Implementation

### PostgreSQL Assignment Repository

```go
// internal/repository/postgres/assignment.go
package postgres

import (
    "context"
    "database/sql"
    "fmt"
    "strings"
    "time"

    "github.com/company/udm-backend/internal/domain"
    "github.com/company/udm-backend/internal/database"
    "github.com/google/uuid"
    "github.com/jmoiron/sqlx"
    "go.uber.org/zap"
)

type PostgresAssignmentRepository struct {
    dbManager *database.Manager
    logger    *zap.Logger
}

func NewAssignmentRepository(dbManager *database.Manager, logger *zap.Logger) *PostgresAssignmentRepository {
    return &PostgresAssignmentRepository{
        dbManager: dbManager,
        logger:    logger,
    }
}

func (r *PostgresAssignmentRepository) Create(ctx context.Context, tenant *domain.TenantContext, assignment *domain.ThingClassAttribute) (*domain.ThingClassAttribute, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return nil, fmt.Errorf("failed to get database connection: %w", err)
    }

    query := `
        INSERT INTO thing_class_attributes (
            id, thing_class_id, attribute_id, is_required, sort_order, display_name,
            assignment_validation_rules, default_value, ui_hints, description,
            assignment_type, visibility, created_at, updated_at, created_by,
            is_active, version
        )
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16, $17)
        RETURNING *`

    var result domain.ThingClassAttribute
    err = db.QueryRowxContext(ctx, query,
        assignment.ID, assignment.ThingClassID, assignment.AttributeID,
        assignment.IsRequired, assignment.SortOrder, assignment.DisplayName,
        assignment.AssignmentValidationRules, assignment.DefaultValue, assignment.UIHints,
        assignment.Description, assignment.AssignmentType, assignment.Visibility,
        assignment.CreatedAt, assignment.UpdatedAt, assignment.CreatedBy,
        assignment.IsActive, assignment.Version,
    ).StructScan(&result)

    if err != nil {
        if r.isUniqueViolation(err) {
            if strings.Contains(err.Error(), "unique_thing_class_attribute") {
                return nil, domain.NewAssignmentAlreadyExistsError(assignment.ThingClassID, assignment.AttributeID)
            }
            if strings.Contains(err.Error(), "sort_order") {
                return nil, fmt.Errorf("sort order %d already exists for Thing Class", assignment.SortOrder)
            }
        }
        return nil, fmt.Errorf("failed to create assignment: %w", err)
    }

    return &result, nil
}

func (r *PostgresAssignmentRepository) GetByID(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.ThingClassAttribute, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return nil, fmt.Errorf("failed to get database connection: %w", err)
    }

    query := `
        SELECT id, thing_class_id, attribute_id, is_required, sort_order, display_name,
               assignment_validation_rules, default_value, ui_hints, description,
               assignment_type, visibility, created_at, updated_at, created_by,
               updated_by, is_active, version
        FROM thing_class_attributes 
        WHERE id = $1 AND is_active = TRUE`

    var result domain.ThingClassAttribute
    err = db.QueryRowxContext(ctx, query, id).StructScan(&result)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, nil
        }
        return nil, fmt.Errorf("failed to get assignment: %w", err)
    }

    return &result, nil
}

func (r *PostgresAssignmentRepository) List(ctx context.Context, tenant *domain.TenantContext, filter *domain.AttributeAssignmentFilter, sorts []*domain.AttributeAssignmentSort, limit, offset int) ([]*domain.ThingClassAttribute, int, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to get database connection: %w", err)
    }

    // Build dynamic query
    whereClause, whereArgs := r.buildWhereClause(filter)
    orderClause := r.buildOrderClause(sorts)

    // Count query
    countQuery := fmt.Sprintf(`
        SELECT COUNT(*) 
        FROM thing_class_attributes 
        WHERE %s`, whereClause)
    
    var totalCount int
    if err := db.QueryRowxContext(ctx, countQuery, whereArgs...).Scan(&totalCount); err != nil {
        return nil, 0, fmt.Errorf("failed to count assignments: %w", err)
    }

    // Main query with pagination
    listQuery := fmt.Sprintf(`
        SELECT id, thing_class_id, attribute_id, is_required, sort_order, display_name,
               assignment_validation_rules, default_value, ui_hints, description,
               assignment_type, visibility, created_at, updated_at, created_by,
               updated_by, is_active, version
        FROM thing_class_attributes 
        WHERE %s
        %s
        LIMIT $%d OFFSET $%d`, whereClause, orderClause, len(whereArgs)+1, len(whereArgs)+2)

    args := append(whereArgs, limit, offset)

    rows, err := db.QueryxContext(ctx, listQuery, args...)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to query assignments: %w", err)
    }
    defer rows.Close()

    var assignments []*domain.ThingClassAttribute
    for rows.Next() {
        var assignment domain.ThingClassAttribute
        if err := rows.StructScan(&assignment); err != nil {
            return nil, 0, fmt.Errorf("failed to scan assignment: %w", err)
        }
        assignments = append(assignments, &assignment)
    }

    if err := rows.Err(); err != nil {
        return nil, 0, fmt.Errorf("error iterating assignment rows: %w", err)
    }

    return assignments, totalCount, nil
}

func (r *PostgresAssignmentRepository) Update(ctx context.Context, tenant *domain.TenantContext, assignment *domain.ThingClassAttribute) (*domain.ThingClassAttribute, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return nil, fmt.Errorf("failed to get database connection: %w", err)
    }

    query := `
        UPDATE thing_class_attributes SET
            is_required = $1, sort_order = $2, display_name = $3,
            assignment_validation_rules = $4, default_value = $5, ui_hints = $6,
            description = $7, assignment_type = $8, visibility = $9,
            updated_at = $10, updated_by = $11, version = $12
        WHERE id = $13 AND version = $14 AND is_active = TRUE
        RETURNING *`

    var result domain.ThingClassAttribute
    err = db.QueryRowxContext(ctx, query,
        assignment.IsRequired, assignment.SortOrder, assignment.DisplayName,
        assignment.AssignmentValidationRules, assignment.DefaultValue, assignment.UIHints,
        assignment.Description, assignment.AssignmentType, assignment.Visibility,
        assignment.UpdatedAt, assignment.UpdatedBy, assignment.Version,
        assignment.ID, assignment.Version-1, // Use optimistic locking
    ).StructScan(&result)

    if err != nil {
        if err == sql.ErrNoRows {
            return nil, fmt.Errorf("assignment not found or version conflict")
        }
        if r.isUniqueViolation(err) && strings.Contains(err.Error(), "sort_order") {
            return nil, fmt.Errorf("sort order %d already exists for Thing Class", assignment.SortOrder)
        }
        return nil, fmt.Errorf("failed to update assignment: %w", err)
    }

    return &result, nil
}

func (r *PostgresAssignmentRepository) SoftDelete(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) error {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return fmt.Errorf("failed to get database connection: %w", err)
    }

    query := `
        UPDATE thing_class_attributes 
        SET is_active = FALSE, updated_at = $1
        WHERE id = $2 AND is_active = TRUE`

    result, err := db.ExecContext(ctx, query, time.Now(), id)
    if err != nil {
        return fmt.Errorf("failed to soft delete assignment: %w", err)
    }

    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return fmt.Errorf("failed to get rows affected: %w", err)
    }

    if rowsAffected == 0 {
        return fmt.Errorf("assignment not found")
    }

    return nil
}

func (r *PostgresAssignmentRepository) ExistsAssignment(ctx context.Context, tenant *domain.TenantContext, thingClassID, attributeID uuid.UUID) (bool, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return false, fmt.Errorf("failed to get database connection: %w", err)
    }

    query := `
        SELECT EXISTS(
            SELECT 1 FROM thing_class_attributes 
            WHERE thing_class_id = $1 AND attribute_id = $2 AND is_active = TRUE
        )`

    var exists bool
    err = db.QueryRowxContext(ctx, query, thingClassID, attributeID).Scan(&exists)
    if err != nil {
        return false, fmt.Errorf("failed to check assignment existence: %w", err)
    }

    return exists, nil
}

func (r *PostgresAssignmentRepository) ExistsSortOrder(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID, sortOrder int, excludeID uuid.UUID) (bool, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return false, fmt.Errorf("failed to get database connection: %w", err)
    }

    query := `
        SELECT EXISTS(
            SELECT 1 FROM thing_class_attributes 
            WHERE thing_class_id = $1 AND sort_order = $2 AND is_active = TRUE
            AND ($3 = '00000000-0000-0000-0000-000000000000'::uuid OR id != $3)
        )`

    var exists bool
    err = db.QueryRowxContext(ctx, query, thingClassID, sortOrder, excludeID).Scan(&exists)
    if err != nil {
        return false, fmt.Errorf("failed to check sort order existence: %w", err)
    }

    return exists, nil
}

func (r *PostgresAssignmentRepository) GetUsageCount(ctx context.Context, tenant *domain.TenantContext, assignmentID uuid.UUID) (int, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return 0, fmt.Errorf("failed to get database connection: %w", err)
    }

    // This would count actual usage in Things/Values - simplified implementation
    query := `
        SELECT COUNT(DISTINCT v.thing_id) 
        FROM thing_class_attributes tca
        JOIN values v ON tca.attribute_id = v.attribute_id
        JOIN things t ON v.thing_id = t.id AND t.thing_class_id = tca.thing_class_id
        WHERE tca.id = $1 AND tca.is_active = TRUE AND v.is_active = TRUE AND t.is_active = TRUE`

    var count int
    err = db.QueryRowxContext(ctx, query, assignmentID).Scan(&count)
    if err != nil {
        return 0, fmt.Errorf("failed to count assignment usage: %w", err)
    }

    return count, nil
}

// Private helper methods

func (r *PostgresAssignmentRepository) buildWhereClause(filter *domain.AttributeAssignmentFilter) (string, []interface{}) {
    conditions := []string{"is_active = TRUE"}
    args := []interface{}{}
    argCount := 0

    if filter == nil {
        return strings.Join(conditions, " AND "), args
    }

    if filter.ThingClassID != nil {
        argCount++
        conditions = append(conditions, fmt.Sprintf("thing_class_id = $%d", argCount))
        args = append(args, *filter.ThingClassID)
    }

    if filter.AttributeID != nil {
        argCount++
        conditions = append(conditions, fmt.Sprintf("attribute_id = $%d", argCount))
        args = append(args, *filter.AttributeID)
    }

    if filter.IsRequired != nil {
        argCount++
        conditions = append(conditions, fmt.Sprintf("is_required = $%d", argCount))
        args = append(args, *filter.IsRequired)
    }

    if filter.AssignmentType != nil {
        argCount++
        conditions = append(conditions, fmt.Sprintf("assignment_type = $%d", argCount))
        args = append(args, *filter.AssignmentType)
    }

    if filter.Visibility != nil {
        argCount++
        conditions = append(conditions, fmt.Sprintf("visibility = $%d", argCount))
        args = append(args, *filter.Visibility)
    }

    if filter.HasDefaultValue != nil {
        if *filter.HasDefaultValue {
            conditions = append(conditions, "default_value IS NOT NULL")
        } else {
            conditions = append(conditions, "default_value IS NULL")
        }
    }

    if filter.HasCustomValidation != nil {
        if *filter.HasCustomValidation {
            conditions = append(conditions, "assignment_validation_rules != '{}'")
        } else {
            conditions = append(conditions, "assignment_validation_rules = '{}'")
        }
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

    if filter.DisplayNameContains != nil {
        argCount++
        conditions = append(conditions, fmt.Sprintf("display_name ILIKE $%d", argCount))
        args = append(args, "%"+*filter.DisplayNameContains+"%")
    }

    if filter.SortOrderRange != nil {
        argCount++
        conditions = append(conditions, fmt.Sprintf("sort_order >= $%d", argCount))
        args = append(args, filter.SortOrderRange.Min)
        
        argCount++
        conditions = append(conditions, fmt.Sprintf("sort_order <= $%d", argCount))
        args = append(args, filter.SortOrderRange.Max)
    }

    return strings.Join(conditions, " AND "), args
}

func (r *PostgresAssignmentRepository) buildOrderClause(sorts []*domain.AttributeAssignmentSort) string {
    if len(sorts) == 0 {
        return "ORDER BY sort_order ASC, created_at ASC"
    }

    var clauses []string
    for _, sort := range sorts {
        var field string
        switch sort.Field {
        case domain.AttributeAssignmentSortFieldSortOrder:
            field = "sort_order"
        case domain.AttributeAssignmentSortFieldDisplayName:
            field = "display_name"
        case domain.AttributeAssignmentSortFieldCreatedAt:
            field = "created_at"
        case domain.AttributeAssignmentSortFieldUpdatedAt:
            field = "updated_at"
        case domain.AttributeAssignmentSortFieldIsRequired:
            field = "is_required"
        case domain.AttributeAssignmentSortFieldAssignmentType:
            field = "assignment_type"
        case domain.AttributeAssignmentSortFieldVisibility:
            field = "visibility"
        case domain.AttributeAssignmentSortFieldUsageCount:
            // This would require a subquery - simplified for now
            field = "sort_order"
        default:
            field = "sort_order"
        }

        direction := "ASC"
        if sort.Direction == domain.SortDirectionDesc {
            direction = "DESC"
        }

        clauses = append(clauses, fmt.Sprintf("%s %s", field, direction))
    }

    return "ORDER BY " + strings.Join(clauses, ", ")
}

func (r *PostgresAssignmentRepository) isUniqueViolation(err error) bool {
    return strings.Contains(err.Error(), "unique constraint") ||
           strings.Contains(err.Error(), "duplicate key value")
}
```

This comprehensive Go implementation specification provides all the necessary patterns and structures to implement the Attribute Assignment entity using Go, gqlgen, and PostgreSQL with proper multi-tenant architecture, comprehensive validation, impact analysis, and maintainable code organization.