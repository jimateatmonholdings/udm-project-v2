# Value Entity - Go Implementation Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**Language:** Go with gqlgen GraphQL  
**Entity:** Value - Property data storage implementation patterns  

## Implementation Architecture Overview

The Value entity Go implementation provides a comprehensive, high-performance data storage layer within the Universal Data Model system. Built using domain-driven design principles with Go and gqlgen, the implementation supports all 8 attribute data types, advanced validation, versioning, and multi-tenant isolation while maintaining sub-200ms performance targets through optimized repository patterns and intelligent caching strategies.

## Domain Layer Implementation

### Core Value Entity

```go
// Package domain defines the core Value entity and related types
package domain

import (
    "encoding/json"
    "time"
    "github.com/google/uuid"
    "github.com/shopspring/decimal"
)

// Value represents an actual property value stored for a Thing-Attribute combination
type Value struct {
    BaseEntity
    
    // Core relationships
    ThingID      uuid.UUID `json:"thingId" db:"thing_id" validate:"required"`
    AttributeID  uuid.UUID `json:"attributeId" db:"attribute_id" validate:"required"`
    AssignmentID uuid.UUID `json:"assignmentId" db:"assignment_id" validate:"required"`
    
    // Polymorphic value storage optimized for different data types
    ValueText        *string          `json:"valueText,omitempty" db:"value_text"`
    ValueInteger     *int64           `json:"valueInteger,omitempty" db:"value_integer"`
    ValueDecimal     *decimal.Decimal `json:"valueDecimal,omitempty" db:"value_decimal"`
    ValueBoolean     *bool            `json:"valueBoolean,omitempty" db:"value_boolean"`
    ValueDate        *time.Time       `json:"valueDate,omitempty" db:"value_date"`
    ValueDateTime    *time.Time       `json:"valueDateTime,omitempty" db:"value_datetime"`
    ValueJSON        json.RawMessage  `json:"valueJson,omitempty" db:"value_json"`
    ValueReferenceID *uuid.UUID       `json:"valueReferenceId,omitempty" db:"value_reference_id"`
    
    // Data type and validation metadata
    DataType             AttributeType   `json:"dataType" db:"data_type" validate:"required"`
    IsValid              bool            `json:"isValid" db:"is_valid"`
    ValidationErrors     json.RawMessage `json:"validationErrors" db:"validation_errors"`
    ValidationTimestamp  time.Time       `json:"validationTimestamp" db:"validation_timestamp"`
    
    // Version and history tracking
    Version       int        `json:"version" db:"version" validate:"min=1"`
    IsCurrent     bool       `json:"isCurrent" db:"is_current"`
    EffectiveFrom time.Time  `json:"effectiveFrom" db:"effective_from"`
    EffectiveTo   *time.Time `json:"effectiveTo,omitempty" db:"effective_to"`
    SupersededBy  *uuid.UUID `json:"supersededBy,omitempty" db:"superseded_by"`
    
    // Computed hash for change detection and deduplication
    ValueHash string `json:"valueHash,omitempty" db:"value_hash"`
    
    // Loaded relationships (not persisted)
    Thing          *Thing                 `json:"thing,omitempty" db:"-"`
    Attribute      *Attribute             `json:"attribute,omitempty" db:"-"`
    Assignment     *ThingClassAttribute   `json:"assignment,omitempty" db:"-"`
    ReferenceThing *Thing                 `json:"referenceThing,omitempty" db:"-"`
    Versions       []*Value               `json:"versions,omitempty" db:"-"`
}

// GetTypedValue returns the value as the appropriate Go type based on data type
func (v *Value) GetTypedValue() (interface{}, error) {
    switch v.DataType {
    case AttributeTypeString:
        if v.ValueText == nil {
            return nil, fmt.Errorf("string value is nil")
        }
        return *v.ValueText, nil
    case AttributeTypeInteger:
        if v.ValueInteger == nil {
            return nil, fmt.Errorf("integer value is nil")
        }
        return *v.ValueInteger, nil
    case AttributeTypeDecimal:
        if v.ValueDecimal == nil {
            return nil, fmt.Errorf("decimal value is nil")
        }
        return *v.ValueDecimal, nil
    case AttributeTypeBoolean:
        if v.ValueBoolean == nil {
            return nil, fmt.Errorf("boolean value is nil")
        }
        return *v.ValueBoolean, nil
    case AttributeTypeDate:
        if v.ValueDate == nil {
            return nil, fmt.Errorf("date value is nil")
        }
        return *v.ValueDate, nil
    case AttributeTypeDateTime:
        if v.ValueDateTime == nil {
            return nil, fmt.Errorf("datetime value is nil")
        }
        return *v.ValueDateTime, nil
    case AttributeTypeJSON:
        if v.ValueJSON == nil {
            return nil, fmt.Errorf("json value is nil")
        }
        var result interface{}
        if err := json.Unmarshal(v.ValueJSON, &result); err != nil {
            return nil, fmt.Errorf("failed to unmarshal JSON value: %w", err)
        }
        return result, nil
    case AttributeTypeReference:
        if v.ValueReferenceID == nil {
            return nil, fmt.Errorf("reference value is nil")
        }
        return *v.ValueReferenceID, nil
    default:
        return nil, fmt.Errorf("unsupported data type: %s", v.DataType)
    }
}

// SetTypedValue sets the appropriate value field based on data type
func (v *Value) SetTypedValue(dataType AttributeType, value interface{}) error {
    // Clear all value fields first
    v.ValueText = nil
    v.ValueInteger = nil
    v.ValueDecimal = nil
    v.ValueBoolean = nil
    v.ValueDate = nil
    v.ValueDateTime = nil
    v.ValueJSON = nil
    v.ValueReferenceID = nil
    
    v.DataType = dataType
    
    switch dataType {
    case AttributeTypeString:
        if str, ok := value.(string); ok {
            v.ValueText = &str
        } else {
            return fmt.Errorf("expected string value, got %T", value)
        }
    case AttributeTypeInteger:
        switch val := value.(type) {
        case int64:
            v.ValueInteger = &val
        case int:
            i64 := int64(val)
            v.ValueInteger = &i64
        case float64:
            // Handle JSON number conversion
            i64 := int64(val)
            v.ValueInteger = &i64
        default:
            return fmt.Errorf("expected integer value, got %T", value)
        }
    case AttributeTypeDecimal:
        switch val := value.(type) {
        case decimal.Decimal:
            v.ValueDecimal = &val
        case float64:
            dec := decimal.NewFromFloat(val)
            v.ValueDecimal = &dec
        case string:
            dec, err := decimal.NewFromString(val)
            if err != nil {
                return fmt.Errorf("invalid decimal string: %w", err)
            }
            v.ValueDecimal = &dec
        default:
            return fmt.Errorf("expected decimal value, got %T", value)
        }
    case AttributeTypeBoolean:
        if b, ok := value.(bool); ok {
            v.ValueBoolean = &b
        } else {
            return fmt.Errorf("expected boolean value, got %T", value)
        }
    case AttributeTypeDate:
        switch val := value.(type) {
        case time.Time:
            // Truncate to date only
            date := time.Date(val.Year(), val.Month(), val.Day(), 0, 0, 0, 0, time.UTC)
            v.ValueDate = &date
        case string:
            parsed, err := time.Parse("2006-01-02", val)
            if err != nil {
                return fmt.Errorf("invalid date format, expected YYYY-MM-DD: %w", err)
            }
            v.ValueDate = &parsed
        default:
            return fmt.Errorf("expected date value, got %T", value)
        }
    case AttributeTypeDateTime:
        switch val := value.(type) {
        case time.Time:
            v.ValueDateTime = &val
        case string:
            parsed, err := time.Parse(time.RFC3339, val)
            if err != nil {
                return fmt.Errorf("invalid datetime format, expected RFC3339: %w", err)
            }
            v.ValueDateTime = &parsed
        default:
            return fmt.Errorf("expected datetime value, got %T", value)
        }
    case AttributeTypeJSON:
        jsonBytes, err := json.Marshal(value)
        if err != nil {
            return fmt.Errorf("failed to marshal JSON value: %w", err)
        }
        v.ValueJSON = jsonBytes
    case AttributeTypeReference:
        switch val := value.(type) {
        case uuid.UUID:
            v.ValueReferenceID = &val
        case string:
            parsed, err := uuid.Parse(val)
            if err != nil {
                return fmt.Errorf("invalid UUID format: %w", err)
            }
            v.ValueReferenceID = &parsed
        default:
            return fmt.Errorf("expected UUID reference value, got %T", value)
        }
    default:
        return fmt.Errorf("unsupported data type: %s", dataType)
    }
    
    return nil
}

// IsEmpty returns true if the value has no content
func (v *Value) IsEmpty() bool {
    return v.ValueText == nil && 
           v.ValueInteger == nil && 
           v.ValueDecimal == nil && 
           v.ValueBoolean == nil && 
           v.ValueDate == nil && 
           v.ValueDateTime == nil && 
           v.ValueJSON == nil && 
           v.ValueReferenceID == nil
}

// String returns a string representation of the value
func (v *Value) String() string {
    typedValue, err := v.GetTypedValue()
    if err != nil {
        return fmt.Sprintf("<%s: ERROR(%s)>", v.DataType, err.Error())
    }
    return fmt.Sprintf("<%s: %v>", v.DataType, typedValue)
}

// Input and filter types for value operations
type CreateValueInput struct {
    ThingID      uuid.UUID   `json:"thingId" validate:"required"`
    AttributeID  uuid.UUID   `json:"attributeId" validate:"required"`
    AssignmentID uuid.UUID   `json:"assignmentId" validate:"required"`
    Value        interface{} `json:"value" validate:"required"`
    ValidateOnly bool        `json:"validateOnly,omitempty"`
    ChangeReason *string     `json:"changeReason,omitempty" validate:"omitempty,max=500"`
}

type UpdateValueInput struct {
    Value        interface{} `json:"value" validate:"required"`
    ValidateOnly bool        `json:"validateOnly,omitempty"`
    ChangeReason *string     `json:"changeReason,omitempty" validate:"omitempty,max=500"`
}

type BulkValueInput struct {
    ThingID      uuid.UUID              `json:"thingId" validate:"required"`
    Values       map[uuid.UUID]interface{} `json:"values" validate:"required,min=1,max=100"`
    ChangeReason *string                `json:"changeReason,omitempty" validate:"omitempty,max=500"`
}

// Comprehensive value filtering
type ValueFilter struct {
    // Entity filters
    ThingID        *uuid.UUID     `json:"thingId,omitempty"`
    ThingIDs       []uuid.UUID    `json:"thingIds,omitempty"`
    AttributeID    *uuid.UUID     `json:"attributeId,omitempty"`
    AttributeIDs   []uuid.UUID    `json:"attributeIds,omitempty"`
    AssignmentID   *uuid.UUID     `json:"assignmentId,omitempty"`
    DataType       *AttributeType `json:"dataType,omitempty"`
    IsValid        *bool          `json:"isValid,omitempty"`
    IsCurrentOnly  bool           `json:"isCurrentOnly"`
    
    // Data type specific filters
    StringValue    *StringFilter   `json:"stringValue,omitempty"`
    IntegerValue   *IntegerFilter  `json:"integerValue,omitempty"`
    DecimalValue   *DecimalFilter  `json:"decimalValue,omitempty"`
    BooleanValue   *bool           `json:"booleanValue,omitempty"`
    DateValue      *DateFilter     `json:"dateValue,omitempty"`
    DateTimeValue  *DateTimeFilter `json:"dateTimeValue,omitempty"`
    JSONValue      *JSONFilter     `json:"jsonValue,omitempty"`
    ReferenceValue *uuid.UUID      `json:"referenceValue,omitempty"`
    
    // Temporal filters
    EffectiveAt    *time.Time `json:"effectiveAt,omitempty"`
    EffectiveFrom  *time.Time `json:"effectiveFrom,omitempty"`
    EffectiveTo    *time.Time `json:"effectiveTo,omitempty"`
    CreatedAfter   *time.Time `json:"createdAfter,omitempty"`
    CreatedBefore  *time.Time `json:"createdBefore,omitempty"`
    UpdatedAfter   *time.Time `json:"updatedAfter,omitempty"`
    UpdatedBefore  *time.Time `json:"updatedBefore,omitempty"`
    
    // Version and audit filters
    Version    *int       `json:"version,omitempty"`
    CreatedBy  *uuid.UUID `json:"createdBy,omitempty"`
    UpdatedBy  *uuid.UUID `json:"updatedBy,omitempty"`
}

// Data type specific filters
type StringFilter struct {
    Equals         *string       `json:"equals,omitempty"`
    Contains       *string       `json:"contains,omitempty"`
    StartsWith     *string       `json:"startsWith,omitempty"`
    EndsWith       *string       `json:"endsWith,omitempty"`
    Regex          *string       `json:"regex,omitempty"`
    Length         *IntegerFilter `json:"length,omitempty"`
    FullTextSearch *string       `json:"fullTextSearch,omitempty"`
}

type IntegerFilter struct {
    Equals         *int64        `json:"equals,omitempty"`
    NotEquals      *int64        `json:"notEquals,omitempty"`
    GreaterThan    *int64        `json:"greaterThan,omitempty"`
    GreaterOrEqual *int64        `json:"greaterOrEqual,omitempty"`
    LessThan       *int64        `json:"lessThan,omitempty"`
    LessOrEqual    *int64        `json:"lessOrEqual,omitempty"`
    Between        *IntegerRange `json:"between,omitempty"`
    In             []int64       `json:"in,omitempty"`
}

type DecimalFilter struct {
    Equals         *decimal.Decimal `json:"equals,omitempty"`
    GreaterThan    *decimal.Decimal `json:"greaterThan,omitempty"`
    GreaterOrEqual *decimal.Decimal `json:"greaterOrEqual,omitempty"`
    LessThan       *decimal.Decimal `json:"lessThan,omitempty"`
    LessOrEqual    *decimal.Decimal `json:"lessOrEqual,omitempty"`
    Between        *DecimalRange    `json:"between,omitempty"`
}

type DateFilter struct {
    Equals  *time.Time `json:"equals,omitempty"`
    Before  *time.Time `json:"before,omitempty"`
    After   *time.Time `json:"after,omitempty"`
    Between *DateRange `json:"between,omitempty"`
}

type DateTimeFilter struct {
    Equals  *time.Time    `json:"equals,omitempty"`
    Before  *time.Time    `json:"before,omitempty"`
    After   *time.Time    `json:"after,omitempty"`
    Between *DateTimeRange `json:"between,omitempty"`
}

type JSONFilter struct {
    Path     string      `json:"path" validate:"required"`
    Operator string      `json:"operator" validate:"required,oneof=equals not_equals contains not_contains exists not_exists greater_than less_than"`
    Value    interface{} `json:"value,omitempty"`
    DataType string      `json:"dataType,omitempty" validate:"omitempty,oneof=string number boolean null"`
}

// Range types
type IntegerRange struct {
    Min *int64 `json:"min,omitempty"`
    Max *int64 `json:"max,omitempty"`
}

type DecimalRange struct {
    Min *decimal.Decimal `json:"min,omitempty"`
    Max *decimal.Decimal `json:"max,omitempty"`
}

type DateRange struct {
    Start *time.Time `json:"start,omitempty"`
    End   *time.Time `json:"end,omitempty"`
}

type DateTimeRange struct {
    Start *time.Time `json:"start,omitempty"`
    End   *time.Time `json:"end,omitempty"`
}

// Result types
type BulkValueResult struct {
    CreatedCount   int                  `json:"createdCount"`
    UpdatedCount   int                  `json:"updatedCount"`
    ErrorCount     int                  `json:"errorCount"`
    Values         []*Value             `json:"values"`
    Errors         []BulkOperationError `json:"errors,omitempty"`
}

type BulkOperationError struct {
    AttributeID uuid.UUID `json:"attributeId"`
    Error       string    `json:"error"`
    Index       int       `json:"index"`
}

type ValueValidationResult struct {
    IsValid     bool              `json:"isValid"`
    Errors      []ValidationError `json:"errors,omitempty"`
    Warnings    []ValidationWarning `json:"warnings,omitempty"`
    DataType    AttributeType     `json:"dataType"`
    ParsedValue interface{}       `json:"parsedValue,omitempty"`
}

type ValueHistory struct {
    ValueID       uuid.UUID   `json:"valueId"`
    Version       int         `json:"version"`
    TypedValue    interface{} `json:"typedValue"`
    EffectiveFrom time.Time   `json:"effectiveFrom"`
    EffectiveTo   *time.Time  `json:"effectiveTo"`
    CreatedBy     uuid.UUID   `json:"createdBy"`
    CreatedAt     time.Time   `json:"createdAt"`
    ChangeReason  *string     `json:"changeReason,omitempty"`
}

type ValueStatistics struct {
    ThingID         uuid.UUID              `json:"thingId"`
    AttributeID     uuid.UUID              `json:"attributeId"`
    DataType        AttributeType          `json:"dataType"`
    TotalCount      int                    `json:"totalCount"`
    ValidCount      int                    `json:"validCount"`
    InvalidCount    int                    `json:"invalidCount"`
    VersionCount    int                    `json:"versionCount"`
    FirstCreated    time.Time              `json:"firstCreated"`
    LastUpdated     time.Time              `json:"lastUpdated"`
    TypedStatistics map[string]interface{} `json:"typedStatistics,omitempty"`
}
```

## Service Layer Implementation

### Value Service with Comprehensive Business Logic

```go
// Package service implements business logic for Value operations
package service

import (
    "context"
    "encoding/json"
    "fmt"
    "strings"
    "time"
    
    "github.com/google/uuid"
    "go.uber.org/zap"
    "github.com/go-playground/validator/v10"
    
    "udm/internal/domain"
    "udm/internal/interfaces"
    "udm/internal/cache"
)

// ValueService handles all business logic for Value operations
type ValueService struct {
    repo              interfaces.ValueRepository
    attributeRepo     interfaces.AttributeRepository
    assignmentRepo    interfaces.AttributeAssignmentRepository
    thingRepo         interfaces.ThingRepository
    validator         *validator.Validate
    cache             cache.Cache
    logger            *zap.Logger
    validationService *ValidationService
    auditService      *AuditService
}

// NewValueService creates a new value service with dependencies
func NewValueService(
    repo interfaces.ValueRepository,
    attributeRepo interfaces.AttributeRepository,
    assignmentRepo interfaces.AttributeAssignmentRepository,
    thingRepo interfaces.ThingRepository,
    validator *validator.Validate,
    cache cache.Cache,
    logger *zap.Logger,
    validationService *ValidationService,
    auditService *AuditService,
) *ValueService {
    return &ValueService{
        repo:              repo,
        attributeRepo:     attributeRepo,
        assignmentRepo:    assignmentRepo,
        thingRepo:         thingRepo,
        validator:         validator,
        cache:             cache,
        logger:            logger,
        validationService: validationService,
        auditService:      auditService,
    }
}

// CreateValue creates a new value with comprehensive validation
func (s *ValueService) CreateValue(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateValueInput) (*domain.Value, error) {
    // Input validation
    if err := s.validator.Struct(input); err != nil {
        return nil, fmt.Errorf("input validation failed: %w", err)
    }

    // Load and validate assignment configuration
    assignment, err := s.loadAndValidateAssignment(ctx, tenant, input.ThingID, input.AttributeID, input.AssignmentID)
    if err != nil {
        return nil, fmt.Errorf("assignment validation failed: %w", err)
    }

    // Load attribute for data type validation
    attribute, err := s.attributeRepo.GetByID(ctx, tenant, input.AttributeID)
    if err != nil {
        return nil, fmt.Errorf("failed to load attribute: %w", err)
    }
    if attribute == nil {
        return nil, fmt.Errorf("attribute not found: %s", input.AttributeID)
    }

    // Validate value against attribute and assignment rules
    validationResult, err := s.validationService.ValidateValue(ctx, tenant, attribute, assignment, input.Value)
    if err != nil {
        return nil, fmt.Errorf("validation service error: %w", err)
    }

    // If validate-only mode, return validation result without persisting
    if input.ValidateOnly {
        return s.createValidationOnlyResult(validationResult, attribute.DataType), nil
    }

    if !validationResult.IsValid {
        return nil, domain.NewValidationError("value validation failed", validationResult.Errors)
    }

    // Check for existing current value to determine operation type
    existing, err := s.repo.GetCurrentValue(ctx, tenant, input.ThingID, input.AttributeID)
    if err != nil {
        return nil, fmt.Errorf("failed to check existing value: %w", err)
    }

    now := time.Now()
    var value *domain.Value

    if existing != nil {
        // Create new version by superseding existing value
        value, err = s.createVersionedValue(ctx, tenant, existing, input, validationResult, now)
    } else {
        // Create initial value
        value, err = s.createInitialValue(ctx, tenant, input, attribute, validationResult, now)
    }

    if err != nil {
        return nil, fmt.Errorf("failed to create value: %w", err)
    }

    // Invalidate related caches
    s.invalidateValueCaches(ctx, tenant, value)

    // Log successful creation
    s.logger.Info("Successfully created value",
        zap.String("id", value.ID.String()),
        zap.String("thing_id", input.ThingID.String()),
        zap.String("attribute_id", input.AttributeID.String()),
        zap.String("data_type", string(attribute.DataType)),
        zap.Int("version", value.Version),
        zap.Bool("is_initial", existing == nil),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return value, nil
}

// UpdateValue updates an existing value with version management
func (s *ValueService) UpdateValue(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID, input *domain.UpdateValueInput) (*domain.Value, error) {
    // Input validation
    if err := s.validator.Struct(input); err != nil {
        return nil, fmt.Errorf("input validation failed: %w", err)
    }

    // Load existing value
    existing, err := s.repo.GetByID(ctx, tenant, id)
    if err != nil {
        return nil, fmt.Errorf("failed to load existing value: %w", err)
    }
    if existing == nil {
        return nil, domain.NewNotFoundError("value", id)
    }
    if !existing.IsCurrent {
        return nil, fmt.Errorf("cannot update non-current value version")
    }

    // Load related entities for validation
    attribute, assignment, err := s.loadValueContext(ctx, tenant, existing)
    if err != nil {
        return nil, fmt.Errorf("failed to load value context: %w", err)
    }

    // Validate new value
    validationResult, err := s.validationService.ValidateValue(ctx, tenant, attribute, assignment, input.Value)
    if err != nil {
        return nil, fmt.Errorf("validation service error: %w", err)
    }

    // If validate-only mode, return validation result
    if input.ValidateOnly {
        return s.createValidationOnlyResult(validationResult, attribute.DataType), nil
    }

    if !validationResult.IsValid {
        return nil, domain.NewValidationError("value validation failed", validationResult.Errors)
    }

    // Check if value actually changed to avoid unnecessary versioning
    if s.valuesAreEqual(existing, attribute.DataType, validationResult.ParsedValue) {
        s.logger.Debug("Value unchanged, skipping update",
            zap.String("value_id", id.String()),
            zap.String("tenant_schema", tenant.SchemaName))
        return existing, nil
    }

    // Create new version
    now := time.Now()
    newValue, err := s.createVersionedValue(ctx, tenant, existing, &domain.CreateValueInput{
        ThingID:      existing.ThingID,
        AttributeID:  existing.AttributeID,
        AssignmentID: existing.AssignmentID,
        Value:        validationResult.ParsedValue,
        ChangeReason: input.ChangeReason,
    }, validationResult, now)

    if err != nil {
        return nil, fmt.Errorf("failed to create new value version: %w", err)
    }

    // Invalidate caches
    s.invalidateValueCaches(ctx, tenant, newValue)

    s.logger.Info("Successfully updated value",
        zap.String("id", newValue.ID.String()),
        zap.String("original_id", id.String()),
        zap.Int("version", newValue.Version),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return newValue, nil
}

// CreateBulkValues handles bulk value creation/update operations
func (s *ValueService) CreateBulkValues(ctx context.Context, tenant *domain.TenantContext, input *domain.BulkValueInput) (*domain.BulkValueResult, error) {
    if len(input.Values) == 0 {
        return nil, fmt.Errorf("no values provided")
    }
    if len(input.Values) > 100 {
        return nil, fmt.Errorf("bulk operation limited to 100 values per request")
    }

    // Load Thing and validate Thing Class
    thing, err := s.thingRepo.GetByID(ctx, tenant, input.ThingID)
    if err != nil {
        return nil, fmt.Errorf("failed to load thing: %w", err)
    }
    if thing == nil {
        return nil, domain.NewNotFoundError("thing", input.ThingID)
    }

    // Pre-load all required entities
    context, err := s.loadBulkOperationContext(ctx, tenant, thing.ThingClassID, input.Values)
    if err != nil {
        return nil, fmt.Errorf("failed to load operation context: %w", err)
    }

    // Prepare and validate all operations
    operations, err := s.prepareBulkOperations(ctx, tenant, input, context)
    if err != nil {
        return nil, fmt.Errorf("failed to prepare bulk operations: %w", err)
    }

    // Execute bulk operation in transaction
    result, err := s.repo.BulkCreateOrUpdate(ctx, tenant, input.ThingID, operations)
    if err != nil {
        s.logger.Error("Failed to execute bulk value operation",
            zap.String("thing_id", input.ThingID.String()),
            zap.Int("operation_count", len(operations)),
            zap.String("tenant_schema", tenant.SchemaName),
            zap.Error(err),
        )
        return nil, fmt.Errorf("failed to execute bulk operation: %w", err)
    }

    // Invalidate caches for all affected attributes
    for attributeID := range input.Values {
        s.cache.InvalidatePattern(ctx, tenant.SchemaName, fmt.Sprintf("values:attribute:%s:*", attributeID))
    }
    s.cache.InvalidatePattern(ctx, tenant.SchemaName, fmt.Sprintf("values:thing:%s:*", input.ThingID))

    s.logger.Info("Successfully completed bulk value operation",
        zap.String("thing_id", input.ThingID.String()),
        zap.Int("created_count", result.CreatedCount),
        zap.Int("updated_count", result.UpdatedCount),
        zap.Int("total_operations", len(operations)),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return result, nil
}

// GetByID retrieves a value by ID with caching
func (s *ValueService) GetByID(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.Value, error) {
    // Try cache first
    cacheKey := fmt.Sprintf("value:%s", id.String())
    var cached domain.Value
    if err := s.cache.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
        return &cached, nil
    }

    // Load from repository
    value, err := s.repo.GetByID(ctx, tenant, id)
    if err != nil {
        return nil, fmt.Errorf("failed to get value from repository: %w", err)
    }
    if value == nil {
        return nil, nil
    }

    // Cache the result
    s.cache.Set(ctx, tenant.SchemaName, cacheKey, value)

    return value, nil
}

// List retrieves values with filtering and pagination
func (s *ValueService) List(ctx context.Context, tenant *domain.TenantContext, filter *domain.ValueFilter, ordering []domain.OrderBy, limit int, offset int) ([]*domain.Value, int, error) {
    // Validate pagination parameters
    if limit <= 0 || limit > 1000 {
        limit = 50
    }
    if offset < 0 {
        offset = 0
    }

    // Try cache for common queries
    cacheKey := s.buildListCacheKey(filter, ordering, limit, offset)
    if cacheKey != "" {
        var cached struct {
            Values     []*domain.Value `json:"values"`
            TotalCount int             `json:"totalCount"`
        }
        if err := s.cache.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
            return cached.Values, cached.TotalCount, nil
        }
    }

    // Load from repository
    values, totalCount, err := s.repo.List(ctx, tenant, filter, ordering, limit, offset)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to list values: %w", err)
    }

    // Cache result if appropriate
    if cacheKey != "" && len(values) <= 100 {
        cacheValue := struct {
            Values     []*domain.Value `json:"values"`
            TotalCount int             `json:"totalCount"`
        }{
            Values:     values,
            TotalCount: totalCount,
        }
        s.cache.Set(ctx, tenant.SchemaName, cacheKey, cacheValue)
    }

    return values, totalCount, nil
}

// GetCurrentValue retrieves the current value for a Thing-Attribute combination
func (s *ValueService) GetCurrentValue(ctx context.Context, tenant *domain.TenantContext, thingID, attributeID uuid.UUID) (*domain.Value, error) {
    // Try cache first
    cacheKey := fmt.Sprintf("current_value:%s:%s", thingID.String(), attributeID.String())
    var cached domain.Value
    if err := s.cache.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
        return &cached, nil
    }

    // Load from repository
    value, err := s.repo.GetCurrentValue(ctx, tenant, thingID, attributeID)
    if err != nil {
        return nil, fmt.Errorf("failed to get current value: %w", err)
    }
    if value == nil {
        return nil, nil
    }

    // Cache the result
    s.cache.Set(ctx, tenant.SchemaName, cacheKey, value)

    return value, nil
}

// GetThingValues retrieves all current values for a Thing
func (s *ValueService) GetThingValues(ctx context.Context, tenant *domain.TenantContext, thingID uuid.UUID, attributeIDs []uuid.UUID) ([]*domain.Value, error) {
    filter := &domain.ValueFilter{
        ThingID:       &thingID,
        IsCurrentOnly: true,
    }

    if len(attributeIDs) > 0 {
        filter.AttributeIDs = attributeIDs
    }

    // Only include valid values
    validFlag := true
    filter.IsValid = &validFlag

    values, _, err := s.repo.List(ctx, tenant, filter, nil, 1000, 0)
    if err != nil {
        return nil, fmt.Errorf("failed to get thing values: %w", err)
    }

    return values, nil
}

// GetValueHistory retrieves version history for a Thing-Attribute combination
func (s *ValueService) GetValueVersions(ctx context.Context, tenant *domain.TenantContext, thingID, attributeID uuid.UUID) ([]*domain.Value, error) {
    filter := &domain.ValueFilter{
        ThingID:       &thingID,
        AttributeID:   &attributeID,
        IsCurrentOnly: false,
    }

    ordering := []domain.OrderBy{
        {Field: "version", Direction: domain.OrderDirectionDESC},
    }

    values, _, err := s.repo.List(ctx, tenant, filter, ordering, 100, 0)
    if err != nil {
        return nil, fmt.Errorf("failed to get value versions: %w", err)
    }

    return values, nil
}

// SoftDelete marks a value as inactive
func (s *ValueService) SoftDelete(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID, changeReason *string) error {
    value, err := s.repo.GetByID(ctx, tenant, id)
    if err != nil {
        return fmt.Errorf("failed to load value: %w", err)
    }
    if value == nil {
        return domain.NewNotFoundError("value", id)
    }

    if err := s.repo.SoftDelete(ctx, tenant, id); err != nil {
        return fmt.Errorf("failed to soft delete value: %w", err)
    }

    // Invalidate caches
    s.invalidateValueCaches(ctx, tenant, value)

    s.logger.Info("Successfully soft deleted value",
        zap.String("id", id.String()),
        zap.String("thing_id", value.ThingID.String()),
        zap.String("attribute_id", value.AttributeID.String()),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return nil
}

// Helper methods
func (s *ValueService) loadAndValidateAssignment(ctx context.Context, tenant *domain.TenantContext, thingID, attributeID, assignmentID uuid.UUID) (*domain.ThingClassAttribute, error) {
    // Load assignment
    assignment, err := s.assignmentRepo.GetByID(ctx, tenant, assignmentID)
    if err != nil {
        return nil, fmt.Errorf("failed to load assignment: %w", err)
    }
    if assignment == nil {
        return nil, fmt.Errorf("assignment not found: %s", assignmentID)
    }

    // Verify assignment matches the provided attribute
    if assignment.AttributeID != attributeID {
        return nil, fmt.Errorf("assignment attribute mismatch: expected %s, got %s", 
            assignment.AttributeID, attributeID)
    }

    // Verify Thing belongs to the correct Thing Class
    thing, err := s.thingRepo.GetByID(ctx, tenant, thingID)
    if err != nil {
        return nil, fmt.Errorf("failed to load thing: %w", err)
    }
    if thing == nil {
        return nil, fmt.Errorf("thing not found: %s", thingID)
    }
    if thing.ThingClassID != assignment.ThingClassID {
        return nil, fmt.Errorf("thing class mismatch: thing belongs to %s, assignment expects %s",
            thing.ThingClassID, assignment.ThingClassID)
    }

    return assignment, nil
}

func (s *ValueService) createValidationOnlyResult(validationResult *domain.ValueValidationResult, dataType domain.AttributeType) *domain.Value {
    validationErrorsJSON, _ := json.Marshal(validationResult.Errors)
    
    return &domain.Value{
        DataType:            dataType,
        IsValid:             validationResult.IsValid,
        ValidationErrors:    validationErrorsJSON,
        ValidationTimestamp: time.Now(),
    }
}

func (s *ValueService) createInitialValue(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateValueInput, attribute *domain.Attribute, validationResult *domain.ValueValidationResult, now time.Time) (*domain.Value, error) {
    value := &domain.Value{
        BaseEntity: domain.BaseEntity{
            ID:        uuid.New(),
            CreatedAt: now,
            UpdatedAt: now,
            CreatedBy: tenant.UserID,
            IsActive:  true,
        },
        ThingID:             input.ThingID,
        AttributeID:         input.AttributeID,
        AssignmentID:        input.AssignmentID,
        Version:             1,
        IsCurrent:           true,
        EffectiveFrom:       now,
        IsValid:             validationResult.IsValid,
        ValidationTimestamp: now,
    }

    // Set typed value
    if err := value.SetTypedValue(attribute.DataType, validationResult.ParsedValue); err != nil {
        return nil, fmt.Errorf("failed to set typed value: %w", err)
    }

    // Set validation errors
    if validationErrorsJSON, err := json.Marshal(validationResult.Errors); err != nil {
        return nil, fmt.Errorf("failed to marshal validation errors: %w", err)
    } else {
        value.ValidationErrors = validationErrorsJSON
    }

    // Create in repository
    created, err := s.repo.Create(ctx, tenant, value)
    if err != nil {
        return nil, fmt.Errorf("failed to persist initial value: %w", err)
    }

    return created, nil
}

func (s *ValueService) createVersionedValue(ctx context.Context, tenant *domain.TenantContext, existing *domain.Value, input *domain.CreateValueInput, validationResult *domain.ValueValidationResult, now time.Time) (*domain.Value, error) {
    newValue := &domain.Value{
        BaseEntity: domain.BaseEntity{
            ID:        uuid.New(),
            CreatedAt: now,
            UpdatedAt: now,
            CreatedBy: tenant.UserID,
            IsActive:  true,
        },
        ThingID:             input.ThingID,
        AttributeID:         input.AttributeID,
        AssignmentID:        input.AssignmentID,
        Version:             existing.Version + 1,
        IsCurrent:           true,
        EffectiveFrom:       now,
        IsValid:             validationResult.IsValid,
        ValidationTimestamp: now,
    }

    // Set typed value
    if err := newValue.SetTypedValue(existing.DataType, validationResult.ParsedValue); err != nil {
        return nil, fmt.Errorf("failed to set typed value: %w", err)
    }

    // Set validation errors
    if validationErrorsJSON, err := json.Marshal(validationResult.Errors); err != nil {
        return nil, fmt.Errorf("failed to marshal validation errors: %w", err)
    } else {
        newValue.ValidationErrors = validationErrorsJSON
    }

    // Create with supersede operation
    created, err := s.repo.CreateWithSupersede(ctx, tenant, newValue, existing.ID)
    if err != nil {
        return nil, fmt.Errorf("failed to persist versioned value: %w", err)
    }

    return created, nil
}

func (s *ValueService) invalidateValueCaches(ctx context.Context, tenant *domain.TenantContext, value *domain.Value) {
    patterns := []string{
        fmt.Sprintf("value:%s", value.ID.String()),
        fmt.Sprintf("current_value:%s:%s", value.ThingID.String(), value.AttributeID.String()),
        fmt.Sprintf("values:thing:%s:*", value.ThingID.String()),
        fmt.Sprintf("values:attribute:%s:*", value.AttributeID.String()),
        "values:list:*",
    }

    for _, pattern := range patterns {
        s.cache.InvalidatePattern(ctx, tenant.SchemaName, pattern)
    }
}

func (s *ValueService) valuesAreEqual(existing *domain.Value, dataType domain.AttributeType, newValue interface{}) bool {
    existingValue, err := existing.GetTypedValue()
    if err != nil {
        return false
    }

    switch dataType {
    case domain.AttributeTypeString:
        existingStr, ok1 := existingValue.(string)
        newStr, ok2 := newValue.(string)
        return ok1 && ok2 && existingStr == newStr
    case domain.AttributeTypeInteger:
        existingInt, ok1 := existingValue.(int64)
        newInt, ok2 := newValue.(int64)
        return ok1 && ok2 && existingInt == newInt
    case domain.AttributeTypeBoolean:
        existingBool, ok1 := existingValue.(bool)
        newBool, ok2 := newValue.(bool)
        return ok1 && ok2 && existingBool == newBool
    case domain.AttributeTypeJSON:
        // Compare JSON by marshaling both and comparing strings
        existingJSON, err1 := json.Marshal(existingValue)
        newJSON, err2 := json.Marshal(newValue)
        return err1 == nil && err2 == nil && string(existingJSON) == string(newJSON)
    // Add other type comparisons as needed
    default:
        return false
    }
}

func (s *ValueService) buildListCacheKey(filter *domain.ValueFilter, ordering []domain.OrderBy, limit int, offset int) string {
    if filter == nil {
        return ""
    }

    // Only cache simple, common queries
    if filter.ThingID != nil && filter.IsCurrentOnly && len(filter.AttributeIDs) == 0 {
        return fmt.Sprintf("values:list:thing:%s:current:limit_%d:offset_%d", 
            filter.ThingID.String(), limit, offset)
    }

    return ""
}
```

This comprehensive Go implementation provides a full-featured Value entity service with proper domain modeling, business logic validation, caching strategies, and performance optimization for the UDM system.