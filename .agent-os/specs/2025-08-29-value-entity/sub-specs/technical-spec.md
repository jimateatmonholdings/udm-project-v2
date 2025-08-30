# Value Entity - Technical Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**Entity:** Value - Property data storage for UDM system  

## Technical Architecture Overview

The Value entity represents the core data storage layer within the Universal Data Model, responsible for persisting actual property values for Thing instances according to their Attribute Assignment configurations. Each Value record stores typed data that respects both base Attribute validation rules and assignment-level constraints while maintaining comprehensive versioning and audit capabilities across tenant boundaries.

## Core Requirements

### Functional Requirements

**FR-1: Value Data Management**
- Create new Value records with automatic data type validation against Attribute definitions
- Update existing Value records with version management and change tracking
- Retrieve Value records with efficient filtering by Thing, Attribute, data type, and temporal constraints
- Soft delete Value records with historical preservation and audit trail maintenance

**FR-2: Multi-Data Type Support**
- Support all 8 core data types: string, integer, decimal, boolean, date, datetime, json, reference
- Implement data type-specific validation using base Attribute rules and Assignment constraints
- Provide data type conversion capabilities for safe schema evolution scenarios
- Handle reference-type values with referential integrity enforcement and cascade operations

**FR-3: Version and History Management**
- Maintain complete version history for all Value changes with efficient storage optimization
- Support temporal queries for retrieving Values at specific points in time
- Implement change tracking with user attribution, timestamps, and change reason metadata
- Provide rollback capabilities for critical value recovery scenarios

**FR-4: Performance and Scalability**
- Support high-volume value operations with horizontal scaling across tenant schemas
- Implement efficient indexing strategies for common query patterns and data type operations
- Provide caching layers for frequently accessed Values with intelligent invalidation
- Maintain linear performance characteristics as value count grows per tenant

### Non-Functional Requirements

**NFR-1: Performance Targets**
- Single Value retrieval: <200ms P95 latency including validation
- Value bulk operations: <500ms P95 latency for up to 100 values
- Value creation/updates: <300ms P95 latency including audit logging
- Historical value queries: <1000ms P95 latency for timeline operations

**NFR-2: Scalability Requirements**
- Support 10M+ Values per tenant schema with optimized storage patterns
- Handle 5,000+ concurrent value operations across all tenants
- Scale to 100+ tenant schemas without performance degradation
- Maintain sub-linear storage growth through data compression and optimization

**NFR-3: Data Integrity and Consistency**
- Ensure Value data type consistency with Attribute definitions
- Maintain referential integrity for reference-type values with Thing entities
- Support transactional operations for complex multi-value updates
- Provide ACID guarantees for critical value operations and state changes

## Data Model Specification

### Database Schema

```sql
-- Values table (per tenant schema) with optimized storage
CREATE TABLE values (
    -- Primary key and identity
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Core value relationships
    thing_id UUID NOT NULL,
    attribute_id UUID NOT NULL,
    assignment_id UUID NOT NULL, -- Direct reference to thing_class_attributes
    
    -- Polymorphic value storage with data type optimization
    value_text TEXT,
    value_integer BIGINT,
    value_decimal NUMERIC(19,6),
    value_boolean BOOLEAN,
    value_date DATE,
    value_datetime TIMESTAMP WITH TIME ZONE,
    value_json JSONB,
    value_reference_id UUID, -- Reference to other Things
    
    -- Data type and validation metadata
    data_type VARCHAR(50) NOT NULL CHECK (data_type IN ('string', 'integer', 'decimal', 'boolean', 'date', 'datetime', 'json', 'reference')),
    is_valid BOOLEAN DEFAULT TRUE,
    validation_errors JSONB DEFAULT '[]',
    validation_timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Version and history tracking
    version INTEGER DEFAULT 1,
    is_current BOOLEAN DEFAULT TRUE,
    effective_from TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    effective_to TIMESTAMP WITH TIME ZONE,
    superseded_by UUID, -- Reference to newer version
    
    -- Standard entity metadata
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_by UUID,
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Business constraints
    CONSTRAINT unique_current_value UNIQUE(thing_id, attribute_id) DEFERRABLE INITIALLY DEFERRED,
    CONSTRAINT valid_data_type_value CHECK (
        (data_type = 'string' AND value_text IS NOT NULL) OR
        (data_type = 'integer' AND value_integer IS NOT NULL) OR
        (data_type = 'decimal' AND value_decimal IS NOT NULL) OR
        (data_type = 'boolean' AND value_boolean IS NOT NULL) OR
        (data_type = 'date' AND value_date IS NOT NULL) OR
        (data_type = 'datetime' AND value_datetime IS NOT NULL) OR
        (data_type = 'json' AND value_json IS NOT NULL) OR
        (data_type = 'reference' AND value_reference_id IS NOT NULL)
    ),
    CONSTRAINT valid_version_sequence CHECK (version > 0),
    CONSTRAINT valid_effective_period CHECK (effective_from <= COALESCE(effective_to, 'infinity'::timestamp)),
    CONSTRAINT valid_validation_errors CHECK (validation_errors IS NOT NULL),
    
    -- Foreign key constraints (within tenant schema)
    CONSTRAINT fk_values_thing 
        FOREIGN KEY (thing_id) 
        REFERENCES things(id) 
        ON DELETE CASCADE 
        ON UPDATE CASCADE,
    CONSTRAINT fk_values_attribute 
        FOREIGN KEY (attribute_id) 
        REFERENCES attributes(id) 
        ON DELETE RESTRICT 
        ON UPDATE CASCADE,
    CONSTRAINT fk_values_assignment 
        FOREIGN KEY (assignment_id) 
        REFERENCES thing_class_attributes(id) 
        ON DELETE RESTRICT 
        ON UPDATE CASCADE,
    CONSTRAINT fk_values_reference 
        FOREIGN KEY (value_reference_id) 
        REFERENCES things(id) 
        ON DELETE SET NULL 
        ON UPDATE CASCADE
);

-- Performance-optimized indexes for common access patterns
CREATE INDEX idx_values_thing_current ON values(thing_id) WHERE is_current = TRUE AND is_active = TRUE;
CREATE INDEX idx_values_attribute_current ON values(attribute_id) WHERE is_current = TRUE AND is_active = TRUE;
CREATE INDEX idx_values_assignment_current ON values(assignment_id) WHERE is_current = TRUE AND is_active = TRUE;

-- Data type specific indexes for efficient filtering
CREATE INDEX idx_values_string_current ON values(value_text text_pattern_ops) WHERE data_type = 'string' AND is_current = TRUE AND is_active = TRUE;
CREATE INDEX idx_values_integer_current ON values(value_integer) WHERE data_type = 'integer' AND is_current = TRUE AND is_active = TRUE;
CREATE INDEX idx_values_decimal_current ON values(value_decimal) WHERE data_type = 'decimal' AND is_current = TRUE AND is_active = TRUE;
CREATE INDEX idx_values_boolean_current ON values(value_boolean) WHERE data_type = 'boolean' AND is_current = TRUE AND is_active = TRUE;
CREATE INDEX idx_values_date_current ON values(value_date) WHERE data_type = 'date' AND is_current = TRUE AND is_active = TRUE;
CREATE INDEX idx_values_datetime_current ON values(value_datetime) WHERE data_type = 'datetime' AND is_current = TRUE AND is_active = TRUE;
CREATE INDEX idx_values_reference_current ON values(value_reference_id) WHERE data_type = 'reference' AND is_current = TRUE AND is_active = TRUE;

-- JSON indexing for structured data queries
CREATE INDEX idx_values_json_gin ON values USING GIN(value_json) WHERE data_type = 'json' AND is_current = TRUE AND is_active = TRUE;

-- Full-text search for string values
CREATE INDEX idx_values_text_search ON values USING GIN(to_tsvector('english', value_text)) 
WHERE data_type = 'string' AND is_current = TRUE AND is_active = TRUE;

-- Historical and versioning indexes
CREATE INDEX idx_values_version_history ON values(thing_id, attribute_id, version DESC) WHERE is_active = TRUE;
CREATE INDEX idx_values_effective_period ON values(effective_from, effective_to) WHERE is_active = TRUE;
CREATE INDEX idx_values_superseded_chain ON values(superseded_by) WHERE superseded_by IS NOT NULL;

-- Validation and data quality indexes
CREATE INDEX idx_values_invalid ON values(thing_id, attribute_id) WHERE is_valid = FALSE AND is_current = TRUE;
CREATE INDEX idx_values_validation_errors ON values USING GIN(validation_errors) WHERE array_length(ARRAY(SELECT jsonb_array_elements(validation_errors)), 1) > 0;

-- Performance monitoring indexes
CREATE INDEX idx_values_created_at ON values(created_at DESC) WHERE is_current = TRUE;
CREATE INDEX idx_values_updated_at ON values(updated_at DESC) WHERE is_current = TRUE;
CREATE INDEX idx_values_created_by ON values(created_by) WHERE is_current = TRUE;
```

### Domain Model

```go
// Core Value entity with polymorphic data storage
type Value struct {
    BaseEntity
    
    // Core relationships
    ThingID      uuid.UUID `json:"thingId" db:"thing_id" validate:"required"`
    AttributeID  uuid.UUID `json:"attributeId" db:"attribute_id" validate:"required"`
    AssignmentID uuid.UUID `json:"assignmentId" db:"assignment_id" validate:"required"`
    
    // Polymorphic value storage
    ValueText        *string                  `json:"valueText,omitempty" db:"value_text"`
    ValueInteger     *int64                   `json:"valueInteger,omitempty" db:"value_integer"`
    ValueDecimal     *decimal.Decimal         `json:"valueDecimal,omitempty" db:"value_decimal"`
    ValueBoolean     *bool                    `json:"valueBoolean,omitempty" db:"value_boolean"`
    ValueDate        *time.Time               `json:"valueDate,omitempty" db:"value_date"`
    ValueDateTime    *time.Time               `json:"valueDateTime,omitempty" db:"value_datetime"`
    ValueJSON        json.RawMessage          `json:"valueJson,omitempty" db:"value_json"`
    ValueReferenceID *uuid.UUID               `json:"valueReferenceId,omitempty" db:"value_reference_id"`
    
    // Data type and validation metadata
    DataType             AttributeType         `json:"dataType" db:"data_type" validate:"required"`
    IsValid              bool                  `json:"isValid" db:"is_valid"`
    ValidationErrors     json.RawMessage       `json:"validationErrors" db:"validation_errors"`
    ValidationTimestamp  time.Time             `json:"validationTimestamp" db:"validation_timestamp"`
    
    // Version and history tracking
    Version      int        `json:"version" db:"version"`
    IsCurrent    bool       `json:"isCurrent" db:"is_current"`
    EffectiveFrom time.Time `json:"effectiveFrom" db:"effective_from"`
    EffectiveTo   *time.Time `json:"effectiveTo,omitempty" db:"effective_to"`
    SupersededBy  *uuid.UUID `json:"supersededBy,omitempty" db:"superseded_by"`
    
    // Computed fields (not persisted, loaded separately)
    TypedValue     interface{} `json:"typedValue,omitempty" db:"-"`
    ReferenceThing *Thing      `json:"referenceThing,omitempty" db:"-"`
    Assignment     *ThingClassAttribute `json:"assignment,omitempty" db:"-"`
    Attribute      *Attribute  `json:"attribute,omitempty" db:"-"`
}

// Input types for GraphQL operations
type CreateValueInput struct {
    ThingID      uuid.UUID       `json:"thingId" validate:"required"`
    AttributeID  uuid.UUID       `json:"attributeId" validate:"required"`
    AssignmentID uuid.UUID       `json:"assignmentId" validate:"required"`
    Value        interface{}     `json:"value" validate:"required"`
    ValidateOnly bool           `json:"validateOnly,omitempty"`
}

type UpdateValueInput struct {
    Value        interface{} `json:"value" validate:"required"`
    ValidateOnly bool       `json:"validateOnly,omitempty"`
    ChangeReason *string    `json:"changeReason,omitempty" validate:"omitempty,max=500"`
}

type BulkValueInput struct {
    ThingID uuid.UUID              `json:"thingId" validate:"required"`
    Values  map[uuid.UUID]interface{} `json:"values" validate:"required,min=1,max=100"` // attribute_id -> value
}

// Filter options for value queries
type ValueFilter struct {
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
    Version        *int           `json:"version,omitempty"`
    CreatedBy      *uuid.UUID     `json:"createdBy,omitempty"`
    UpdatedBy      *uuid.UUID     `json:"updatedBy,omitempty"`
}

// Data type specific filter structures
type StringFilter struct {
    Equals      *string `json:"equals,omitempty"`
    Contains    *string `json:"contains,omitempty"`
    StartsWith  *string `json:"startsWith,omitempty"`
    EndsWith    *string `json:"endsWith,omitempty"`
    Regex       *string `json:"regex,omitempty"`
    Length      *IntegerFilter `json:"length,omitempty"`
    FullTextSearch *string `json:"fullTextSearch,omitempty"`
}

type IntegerFilter struct {
    Equals          *int64 `json:"equals,omitempty"`
    NotEquals       *int64 `json:"notEquals,omitempty"`
    GreaterThan     *int64 `json:"greaterThan,omitempty"`
    GreaterOrEqual  *int64 `json:"greaterOrEqual,omitempty"`
    LessThan        *int64 `json:"lessThan,omitempty"`
    LessOrEqual     *int64 `json:"lessOrEqual,omitempty"`
    Between         *IntegerRange `json:"between,omitempty"`
    In              []int64 `json:"in,omitempty"`
}

type DecimalFilter struct {
    Equals          *decimal.Decimal `json:"equals,omitempty"`
    GreaterThan     *decimal.Decimal `json:"greaterThan,omitempty"`
    GreaterOrEqual  *decimal.Decimal `json:"greaterOrEqual,omitempty"`
    LessThan        *decimal.Decimal `json:"lessThan,omitempty"`
    LessOrEqual     *decimal.Decimal `json:"lessOrEqual,omitempty"`
    Between         *DecimalRange    `json:"between,omitempty"`
}

type DateFilter struct {
    Equals     *time.Time `json:"equals,omitempty"`
    Before     *time.Time `json:"before,omitempty"`
    After      *time.Time `json:"after,omitempty"`
    Between    *DateRange `json:"between,omitempty"`
}

type DateTimeFilter struct {
    Equals     *time.Time    `json:"equals,omitempty"`
    Before     *time.Time    `json:"before,omitempty"`
    After      *time.Time    `json:"after,omitempty"`
    Between    *DateTimeRange `json:"between,omitempty"`
}

type JSONFilter struct {
    Path      string      `json:"path" validate:"required"`      // JSON path expression
    Operator  string      `json:"operator" validate:"required"`  // equals, contains, exists, etc.
    Value     interface{} `json:"value,omitempty"`
    DataType  string      `json:"dataType,omitempty"`           // string, number, boolean, null
}

// Range filter structures
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

// Value history and audit structures
type ValueHistory struct {
    ValueID       uuid.UUID   `json:"valueId" db:"id"`
    Version       int         `json:"version" db:"version"`
    TypedValue    interface{} `json:"typedValue"`
    EffectiveFrom time.Time   `json:"effectiveFrom" db:"effective_from"`
    EffectiveTo   *time.Time  `json:"effectiveTo" db:"effective_to"`
    CreatedBy     uuid.UUID   `json:"createdBy" db:"created_by"`
    CreatedAt     time.Time   `json:"createdAt" db:"created_at"`
    ChangeReason  *string     `json:"changeReason,omitempty"`
}

type ValueValidationResult struct {
    IsValid    bool              `json:"isValid"`
    Errors     []ValidationError `json:"errors,omitempty"`
    Warnings   []ValidationWarning `json:"warnings,omitempty"`
    DataType   AttributeType     `json:"dataType"`
    ParsedValue interface{}      `json:"parsedValue,omitempty"`
}

type ValidationError struct {
    Field       string                 `json:"field"`
    Code        string                 `json:"code"`
    Message     string                 `json:"message"`
    Details     map[string]interface{} `json:"details,omitempty"`
    Rule        string                 `json:"rule,omitempty"`
    ActualValue interface{}            `json:"actualValue,omitempty"`
    ExpectedValue interface{}          `json:"expectedValue,omitempty"`
}

type ValidationWarning struct {
    Field   string      `json:"field"`
    Code    string      `json:"code"`
    Message string      `json:"message"`
    Details interface{} `json:"details,omitempty"`
}

// Aggregation and statistics structures
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

## Service Layer Architecture

### Business Logic Implementation

```go
// ValueService handles all business logic for Value operations
type ValueService struct {
    repo               interfaces.ValueRepository
    attributeRepo      interfaces.AttributeRepository
    assignmentRepo     interfaces.AttributeAssignmentRepository
    thingRepo          interfaces.ThingRepository
    validator          *validator.Validate
    cache              cache.Cache
    logger             *zap.Logger
    validationService  *ValidationService
    auditService      *AuditService
}

// Core business operations with comprehensive validation
func (s *ValueService) CreateValue(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateValueInput) (*domain.Value, error) {
    // Input validation
    if err := s.validateCreateInput(input); err != nil {
        return nil, fmt.Errorf("input validation failed: %w", err)
    }

    // Load and validate assignment configuration
    assignment, err := s.assignmentRepo.GetByID(ctx, tenant, input.AssignmentID)
    if err != nil {
        return nil, fmt.Errorf("failed to load assignment: %w", err)
    }
    if assignment == nil {
        return nil, fmt.Errorf("assignment not found: %s", input.AssignmentID)
    }

    // Verify Thing exists and matches assignment's Thing Class
    thing, err := s.thingRepo.GetByID(ctx, tenant, input.ThingID)
    if err != nil {
        return nil, fmt.Errorf("failed to load thing: %w", err)
    }
    if thing == nil {
        return nil, fmt.Errorf("thing not found: %s", input.ThingID)
    }
    if thing.ThingClassID != assignment.ThingClassID {
        return nil, fmt.Errorf("thing class mismatch: thing belongs to %s, assignment expects %s", 
            thing.ThingClassID, assignment.ThingClassID)
    }

    // Load attribute for data type validation
    attribute, err := s.attributeRepo.GetByID(ctx, tenant, input.AttributeID)
    if err != nil {
        return nil, fmt.Errorf("failed to load attribute: %w", err)
    }
    if attribute == nil {
        return nil, fmt.Errorf("attribute not found: %s", input.AttributeID)
    }
    if assignment.AttributeID != attribute.ID {
        return nil, fmt.Errorf("attribute mismatch: assignment expects %s, got %s",
            assignment.AttributeID, attribute.ID)
    }

    // Validate value against attribute and assignment rules
    validationResult, err := s.validationService.ValidateValue(ctx, tenant, attribute, assignment, input.Value)
    if err != nil {
        return nil, fmt.Errorf("validation service error: %w", err)
    }
    
    if input.ValidateOnly {
        // Return validation result without persisting
        return &domain.Value{
            ValidationErrors: must(json.Marshal(validationResult.Errors)),
            IsValid:         validationResult.IsValid,
        }, nil
    }

    if !validationResult.IsValid {
        return nil, fmt.Errorf("value validation failed: %s", formatValidationErrors(validationResult.Errors))
    }

    // Check for existing current value
    existing, err := s.repo.GetCurrentValue(ctx, tenant, input.ThingID, input.AttributeID)
    if err != nil {
        return nil, fmt.Errorf("failed to check existing value: %w", err)
    }

    now := time.Now()
    var value *domain.Value

    if existing != nil {
        // Update existing value by creating new version
        value = &domain.Value{
            BaseEntity: domain.BaseEntity{
                ID:        uuid.New(),
                CreatedAt: now,
                UpdatedAt: now,
                CreatedBy: tenant.UserID,
                IsActive:  true,
            },
            ThingID:      input.ThingID,
            AttributeID:  input.AttributeID,
            AssignmentID: input.AssignmentID,
            DataType:     attribute.DataType,
            Version:      existing.Version + 1,
            IsCurrent:    true,
            EffectiveFrom: now,
            IsValid:      validationResult.IsValid,
            ValidationErrors: must(json.Marshal(validationResult.Errors)),
            ValidationTimestamp: now,
        }

        // Set typed value based on data type
        if err := s.setTypedValue(value, attribute.DataType, validationResult.ParsedValue); err != nil {
            return nil, fmt.Errorf("failed to set typed value: %w", err)
        }

        // Supersede existing value in transaction
        created, err := s.repo.CreateWithSupersede(ctx, tenant, value, existing.ID)
        if err != nil {
            s.logger.Error("Failed to create versioned value",
                zap.String("thing_id", input.ThingID.String()),
                zap.String("attribute_id", input.AttributeID.String()),
                zap.String("tenant_schema", tenant.SchemaName),
                zap.Error(err),
            )
            return nil, fmt.Errorf("failed to create versioned value: %w", err)
        }
        value = created

    } else {
        // Create initial value
        value = &domain.Value{
            BaseEntity: domain.BaseEntity{
                ID:        uuid.New(),
                CreatedAt: now,
                UpdatedAt: now,
                CreatedBy: tenant.UserID,
                IsActive:  true,
            },
            ThingID:      input.ThingID,
            AttributeID:  input.AttributeID,
            AssignmentID: input.AssignmentID,
            DataType:     attribute.DataType,
            Version:      1,
            IsCurrent:    true,
            EffectiveFrom: now,
            IsValid:      validationResult.IsValid,
            ValidationErrors: must(json.Marshal(validationResult.Errors)),
            ValidationTimestamp: now,
        }

        // Set typed value
        if err := s.setTypedValue(value, attribute.DataType, validationResult.ParsedValue); err != nil {
            return nil, fmt.Errorf("failed to set typed value: %w", err)
        }

        // Persist new value
        created, err := s.repo.Create(ctx, tenant, value)
        if err != nil {
            s.logger.Error("Failed to create value",
                zap.String("thing_id", input.ThingID.String()),
                zap.String("attribute_id", input.AttributeID.String()),
                zap.String("tenant_schema", tenant.SchemaName),
                zap.Error(err),
            )
            return nil, fmt.Errorf("failed to create value: %w", err)
        }
        value = created
    }

    // Invalidate related caches
    s.cache.InvalidatePattern(ctx, tenant.SchemaName, fmt.Sprintf("values:thing:%s:*", input.ThingID))
    s.cache.InvalidatePattern(ctx, tenant.SchemaName, fmt.Sprintf("values:attribute:%s:*", input.AttributeID))

    s.logger.Info("Successfully created value",
        zap.String("id", value.ID.String()),
        zap.String("thing_id", input.ThingID.String()),
        zap.String("attribute_id", input.AttributeID.String()),
        zap.String("data_type", string(attribute.DataType)),
        zap.Int("version", value.Version),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return value, nil
}

// Bulk value creation with optimized transaction handling
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
        return nil, fmt.Errorf("thing not found: %s", input.ThingID)
    }

    // Pre-load all assignments and attributes
    attributeIDs := make([]uuid.UUID, 0, len(input.Values))
    for attributeID := range input.Values {
        attributeIDs = append(attributeIDs, attributeID)
    }

    assignments, err := s.assignmentRepo.GetByThingClassAndAttributes(ctx, tenant, thing.ThingClassID, attributeIDs)
    if err != nil {
        return nil, fmt.Errorf("failed to load assignments: %w", err)
    }

    attributes, err := s.attributeRepo.GetByIDs(ctx, tenant, attributeIDs)
    if err != nil {
        return nil, fmt.Errorf("failed to load attributes: %w", err)
    }

    // Build lookup maps
    assignmentMap := make(map[uuid.UUID]*domain.ThingClassAttribute)
    attributeMap := make(map[uuid.UUID]*domain.Attribute)
    
    for _, assignment := range assignments {
        assignmentMap[assignment.AttributeID] = assignment
    }
    for _, attribute := range attributes {
        attributeMap[attribute.ID] = attribute
    }

    // Validate and prepare all values
    valueOperations := make([]*ValueOperation, 0, len(input.Values))
    validationResults := make(map[uuid.UUID]*domain.ValueValidationResult)

    for attributeID, value := range input.Values {
        assignment := assignmentMap[attributeID]
        if assignment == nil {
            return nil, fmt.Errorf("no assignment found for attribute %s on thing class %s", 
                attributeID, thing.ThingClassID)
        }

        attribute := attributeMap[attributeID]
        if attribute == nil {
            return nil, fmt.Errorf("attribute not found: %s", attributeID)
        }

        // Validate value
        validationResult, err := s.validationService.ValidateValue(ctx, tenant, attribute, assignment, value)
        if err != nil {
            return nil, fmt.Errorf("validation error for attribute %s: %w", attributeID, err)
        }
        
        validationResults[attributeID] = validationResult
        
        if !validationResult.IsValid {
            return nil, fmt.Errorf("validation failed for attribute %s: %s", 
                attributeID, formatValidationErrors(validationResult.Errors))
        }

        valueOperations = append(valueOperations, &ValueOperation{
            AttributeID:      attributeID,
            AssignmentID:     assignment.ID,
            Value:           value,
            ParsedValue:     validationResult.ParsedValue,
            ValidationResult: validationResult,
            Attribute:       attribute,
        })
    }

    // Execute bulk operation in transaction
    result, err := s.repo.BulkCreateOrUpdate(ctx, tenant, input.ThingID, valueOperations)
    if err != nil {
        s.logger.Error("Failed to execute bulk value operation",
            zap.String("thing_id", input.ThingID.String()),
            zap.Int("operation_count", len(valueOperations)),
            zap.String("tenant_schema", tenant.SchemaName),
            zap.Error(err),
        )
        return nil, fmt.Errorf("failed to execute bulk operation: %w", err)
    }

    // Invalidate caches
    s.cache.InvalidatePattern(ctx, tenant.SchemaName, fmt.Sprintf("values:thing:%s:*", input.ThingID))
    for attributeID := range input.Values {
        s.cache.InvalidatePattern(ctx, tenant.SchemaName, fmt.Sprintf("values:attribute:%s:*", attributeID))
    }

    s.logger.Info("Successfully completed bulk value operation",
        zap.String("thing_id", input.ThingID.String()),
        zap.Int("created_count", result.CreatedCount),
        zap.Int("updated_count", result.UpdatedCount),
        zap.Int("total_operations", len(valueOperations)),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return result, nil
}

// Helper structures for bulk operations
type ValueOperation struct {
    AttributeID      uuid.UUID
    AssignmentID     uuid.UUID
    Value           interface{}
    ParsedValue     interface{}
    ValidationResult *domain.ValueValidationResult
    Attribute       *domain.Attribute
}

type BulkValueResult struct {
    CreatedCount   int                    `json:"createdCount"`
    UpdatedCount   int                    `json:"updatedCount"`
    ErrorCount     int                    `json:"errorCount"`
    Values         []*domain.Value        `json:"values"`
    Errors         []BulkOperationError   `json:"errors,omitempty"`
}

type BulkOperationError struct {
    AttributeID uuid.UUID `json:"attributeId"`
    Error       string    `json:"error"`
    Index       int       `json:"index"`
}

// Helper methods
func (s *ValueService) setTypedValue(value *domain.Value, dataType domain.AttributeType, parsedValue interface{}) error {
    switch dataType {
    case domain.AttributeTypeString:
        if str, ok := parsedValue.(string); ok {
            value.ValueText = &str
        } else {
            return fmt.Errorf("invalid string value type")
        }
    case domain.AttributeTypeInteger:
        if i, ok := parsedValue.(int64); ok {
            value.ValueInteger = &i
        } else {
            return fmt.Errorf("invalid integer value type")
        }
    case domain.AttributeTypeDecimal:
        if d, ok := parsedValue.(decimal.Decimal); ok {
            value.ValueDecimal = &d
        } else {
            return fmt.Errorf("invalid decimal value type")
        }
    case domain.AttributeTypeBoolean:
        if b, ok := parsedValue.(bool); ok {
            value.ValueBoolean = &b
        } else {
            return fmt.Errorf("invalid boolean value type")
        }
    case domain.AttributeTypeDate:
        if t, ok := parsedValue.(time.Time); ok {
            value.ValueDate = &t
        } else {
            return fmt.Errorf("invalid date value type")
        }
    case domain.AttributeTypeDateTime:
        if t, ok := parsedValue.(time.Time); ok {
            value.ValueDateTime = &t
        } else {
            return fmt.Errorf("invalid datetime value type")
        }
    case domain.AttributeTypeJSON:
        if jsonBytes, err := json.Marshal(parsedValue); err != nil {
            return fmt.Errorf("failed to marshal JSON value: %w", err)
        } else {
            value.ValueJSON = jsonBytes
        }
    case domain.AttributeTypeReference:
        if ref, ok := parsedValue.(uuid.UUID); ok {
            value.ValueReferenceID = &ref
        } else {
            return fmt.Errorf("invalid reference value type")
        }
    default:
        return fmt.Errorf("unsupported data type: %s", dataType)
    }
    return nil
}

func formatValidationErrors(errors []domain.ValidationError) string {
    if len(errors) == 0 {
        return "no errors"
    }
    var messages []string
    for _, err := range errors {
        messages = append(messages, err.Message)
    }
    return strings.Join(messages, "; ")
}

func must[T any](value T, err error) T {
    if err != nil {
        panic(err)
    }
    return value
}
```

This technical specification provides comprehensive implementation guidance for the Value entity, covering all aspects from data modeling to service layer implementation with proper validation, versioning, and performance optimization.