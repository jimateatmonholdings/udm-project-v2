# Attribute Entity - Technical Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**Entity:** Attribute - Property definitions for UDM system  

## Technical Architecture Overview

The Attribute entity serves as the foundational property definition system within the Universal Data Model, enabling dynamic data modeling through reusable attribute templates. Each Attribute defines a specific property type that can be assigned to Thing Classes, supporting multiple data types and validation rules while maintaining schema isolation across tenant environments.

## Core Requirements

### Functional Requirements

**FR-1: Attribute Definition Management**
- Create new Attribute definitions with name, data type, and validation rules
- Update existing Attribute definitions with backward compatibility validation
- Retrieve Attribute definitions with filtering and pagination support
- Soft delete Attribute definitions with impact analysis on existing assignments

**FR-2: Data Type System**
- Support eight core data types: string, integer, decimal, boolean, date, datetime, json, reference
- Validate data type constraints during Attribute creation and modification
- Provide data type-specific validation rule templates and examples
- Enable data type conversion guidelines for schema evolution

**FR-3: Validation Rule Engine**
- Store JSON-based validation rules for each Attribute definition
- Validate validation rule syntax and structure during Attribute operations
- Support common validation patterns: required, min/max length, regex patterns, value ranges
- Enable validation rule inheritance and composition for complex constraints

**FR-4: Schema Evolution Support**
- Track Thing Class assignments for each Attribute to assess modification impact
- Validate Attribute changes against existing Thing Class assignments
- Support safe Attribute modifications that don't break existing data structures
- Provide migration paths for breaking Attribute changes

### Non-Functional Requirements

**NFR-1: Performance Targets**
- Single Attribute retrieval: <200ms P95 latency
- Attribute listing with pagination: <500ms P95 latency
- Attribute creation/updates: <300ms P95 latency including validation
- Schema evolution checks: <1000ms P95 latency for impact analysis

**NFR-2: Scalability Requirements**
- Support 10,000+ Attributes per tenant schema
- Handle 1,000+ concurrent Attribute operations across all tenants
- Scale to 100+ tenant schemas without performance degradation
- Maintain linear performance characteristics as Attribute count grows

**NFR-3: Data Consistency**
- Ensure Attribute name uniqueness within each tenant schema
- Maintain referential integrity with Thing Class assignments
- Support transactional operations for complex Attribute modifications
- Provide data validation at both API and database levels

## Data Model Specification

### Database Schema

```sql
-- Attributes table (per tenant schema)
CREATE TABLE attributes (
    -- Primary key and identity
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Core attribute properties
    name VARCHAR(255) NOT NULL,
    data_type VARCHAR(50) NOT NULL CHECK (data_type IN ('string', 'integer', 'decimal', 'boolean', 'date', 'datetime', 'json', 'reference')),
    validation_rules JSONB DEFAULT '{}',
    description TEXT,
    
    -- Metadata and versioning
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_by UUID,
    is_active BOOLEAN DEFAULT TRUE,
    version INTEGER DEFAULT 1,
    
    -- Constraints and indexes
    CONSTRAINT unique_attribute_name UNIQUE(name),
    CONSTRAINT valid_validation_rules CHECK (validation_rules IS NOT NULL)
);

-- Indexes for performance optimization
CREATE INDEX idx_attributes_data_type ON attributes(data_type) WHERE is_active = TRUE;
CREATE INDEX idx_attributes_created_at ON attributes(created_at) WHERE is_active = TRUE;
CREATE INDEX idx_attributes_name_text ON attributes USING GIN(to_tsvector('english', name));
CREATE INDEX idx_attributes_validation_rules_gin ON attributes USING GIN(validation_rules);

-- Audit table for change tracking
CREATE TABLE attributes_audit (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attribute_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    old_values JSONB,
    new_values JSONB,
    changed_by UUID NOT NULL,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    change_reason TEXT
);

-- Trigger for automatic audit logging
CREATE OR REPLACE FUNCTION attributes_audit_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO attributes_audit (attribute_id, operation, new_values, changed_by, change_reason)
        VALUES (NEW.id, 'INSERT', to_jsonb(NEW), NEW.created_by, 'Attribute created');
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO attributes_audit (attribute_id, operation, old_values, new_values, changed_by, change_reason)
        VALUES (NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), NEW.updated_by, 'Attribute updated');
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO attributes_audit (attribute_id, operation, old_values, changed_by, change_reason)
        VALUES (OLD.id, 'DELETE', to_jsonb(OLD), OLD.updated_by, 'Attribute deleted');
        RETURN OLD;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER attributes_audit_trig
    AFTER INSERT OR UPDATE OR DELETE ON attributes
    FOR EACH ROW EXECUTE FUNCTION attributes_audit_trigger();
```

### Domain Model

```go
// Core Attribute entity with all properties and relationships
type Attribute struct {
    BaseEntity
    Name            string          `json:"name" db:"name" validate:"required,min=1,max=255"`
    DataType        AttributeType   `json:"dataType" db:"data_type" validate:"required"`
    ValidationRules json.RawMessage `json:"validationRules" db:"validation_rules"`
    Description     *string         `json:"description" db:"description" validate:"omitempty,max=1000"`
    
    // Computed fields (not persisted, loaded separately)
    AssignmentCount *int                    `json:"assignmentCount,omitempty" db:"-"`
    Assignments     []*ThingClassAttribute `json:"assignments,omitempty" db:"-"`
}

// Supported data types with validation constraints
type AttributeType string

const (
    AttributeTypeString    AttributeType = "string"    // Text data with length constraints
    AttributeTypeInteger   AttributeType = "integer"   // Whole numbers with range constraints
    AttributeTypeDecimal   AttributeType = "decimal"   // Decimal numbers with precision constraints
    AttributeTypeBoolean   AttributeType = "boolean"   // True/false values
    AttributeTypeDate      AttributeType = "date"      // Date only (YYYY-MM-DD)
    AttributeTypeDateTime  AttributeType = "datetime"  // Date and time with timezone
    AttributeTypeJSON      AttributeType = "json"      // Structured JSON data
    AttributeTypeReference AttributeType = "reference" // Reference to other Things
)

// Input types for GraphQL operations
type CreateAttributeInput struct {
    Name            string          `json:"name" validate:"required,min=1,max=255,alphanum_underscore"`
    DataType        AttributeType   `json:"dataType" validate:"required,oneof=string integer decimal boolean date datetime json reference"`
    ValidationRules json.RawMessage `json:"validationRules,omitempty"`
    Description     *string         `json:"description,omitempty" validate:"omitempty,max=1000"`
}

type UpdateAttributeInput struct {
    Name            *string         `json:"name,omitempty" validate:"omitempty,min=1,max=255,alphanum_underscore"`
    ValidationRules json.RawMessage `json:"validationRules,omitempty"`
    Description     *string         `json:"description,omitempty" validate:"omitempty,max=1000"`
    // Note: DataType cannot be updated after creation to maintain data integrity
}

// Filter options for attribute queries
type AttributeFilter struct {
    Name        *string        `json:"name,omitempty"`
    DataType    *AttributeType `json:"dataType,omitempty"`
    IsActive    *bool          `json:"isActive,omitempty"`
    CreatedBy   *uuid.UUID     `json:"createdBy,omitempty"`
    CreatedAfter *time.Time    `json:"createdAfter,omitempty"`
    CreatedBefore *time.Time   `json:"createdBefore,omitempty"`
    HasAssignments *bool       `json:"hasAssignments,omitempty"`
}

// Validation rule structure definitions by data type
type ValidationRuleTemplates struct {
    String ValidationRuleString `json:"string,omitempty"`
    Integer ValidationRuleInteger `json:"integer,omitempty"`
    Decimal ValidationRuleDecimal `json:"decimal,omitempty"`
    Boolean ValidationRuleBoolean `json:"boolean,omitempty"`
    Date ValidationRuleDate `json:"date,omitempty"`
    DateTime ValidationRuleDateTime `json:"datetime,omitempty"`
    JSON ValidationRuleJSON `json:"json,omitempty"`
    Reference ValidationRuleReference `json:"reference,omitempty"`
}

type ValidationRuleString struct {
    Required  *bool   `json:"required,omitempty"`
    MinLength *int    `json:"minLength,omitempty"`
    MaxLength *int    `json:"maxLength,omitempty"`
    Pattern   *string `json:"pattern,omitempty"`
    Format    *string `json:"format,omitempty"` // email, url, uuid, etc.
}

type ValidationRuleInteger struct {
    Required *bool  `json:"required,omitempty"`
    Min      *int64 `json:"min,omitempty"`
    Max      *int64 `json:"max,omitempty"`
    Multiple *int64 `json:"multiple,omitempty"`
}

type ValidationRuleDecimal struct {
    Required  *bool    `json:"required,omitempty"`
    Min       *float64 `json:"min,omitempty"`
    Max       *float64 `json:"max,omitempty"`
    Precision *int     `json:"precision,omitempty"`
    Scale     *int     `json:"scale,omitempty"`
}

type ValidationRuleBoolean struct {
    Required *bool `json:"required,omitempty"`
    Default  *bool `json:"default,omitempty"`
}

type ValidationRuleDate struct {
    Required *bool      `json:"required,omitempty"`
    Min      *time.Time `json:"min,omitempty"`
    Max      *time.Time `json:"max,omitempty"`
    Format   *string    `json:"format,omitempty"`
}

type ValidationRuleDateTime struct {
    Required *bool      `json:"required,omitempty"`
    Min      *time.Time `json:"min,omitempty"`
    Max      *time.Time `json:"max,omitempty"`
    Format   *string    `json:"format,omitempty"`
    Timezone *string    `json:"timezone,omitempty"`
}

type ValidationRuleJSON struct {
    Required     *bool              `json:"required,omitempty"`
    Schema       json.RawMessage    `json:"schema,omitempty"`       // JSON Schema for validation
    MaxDepth     *int               `json:"maxDepth,omitempty"`
    AllowedKeys  []string           `json:"allowedKeys,omitempty"`
    RequiredKeys []string           `json:"requiredKeys,omitempty"`
}

type ValidationRuleReference struct {
    Required       *bool      `json:"required,omitempty"`
    TargetThingClass *uuid.UUID `json:"targetThingClass,omitempty"`
    AllowedClasses []uuid.UUID `json:"allowedClasses,omitempty"`
    Cardinality    *string    `json:"cardinality,omitempty"` // one, many
}
```

## Service Layer Architecture

### Business Logic Implementation

```go
// AttributeService handles all business logic for Attribute operations
type AttributeService struct {
    repo          interfaces.AttributeRepository
    thingClassRepo interfaces.ThingClassRepository
    validator     *validator.Validate
    cache         cache.Cache
    logger        *zap.Logger
}

// Core business operations with comprehensive validation
func (s *AttributeService) CreateAttribute(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateAttributeInput) (*domain.Attribute, error) {
    // Input validation
    if err := s.validateCreateInput(input); err != nil {
        return nil, fmt.Errorf("validation failed: %w", err)
    }

    // Business rule validation
    if err := s.validateBusinessRules(ctx, tenant, input); err != nil {
        return nil, fmt.Errorf("business rule validation failed: %w", err)
    }

    // Validation rules syntax validation
    if err := s.validateValidationRules(input.DataType, input.ValidationRules); err != nil {
        return nil, fmt.Errorf("validation rules invalid: %w", err)
    }

    // Check name uniqueness
    exists, err := s.repo.ExistsByName(ctx, tenant, input.Name)
    if err != nil {
        return nil, fmt.Errorf("failed to check name uniqueness: %w", err)
    }
    if exists {
        return nil, fmt.Errorf("attribute name '%s' already exists", input.Name)
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
            zap.String("tenant_schema", tenant.SchemaName),
            zap.Error(err),
        )
        return nil, fmt.Errorf("failed to create attribute: %w", err)
    }

    // Invalidate related caches
    s.cache.InvalidatePattern(ctx, tenant.SchemaName, "attributes:*")

    s.logger.Info("Successfully created attribute",
        zap.String("id", created.ID.String()),
        zap.String("name", created.Name),
        zap.String("data_type", string(created.DataType)),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return created, nil
}

// Update attribute with schema evolution validation
func (s *AttributeService) UpdateAttribute(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID, input *domain.UpdateAttributeInput) (*domain.Attribute, error) {
    // Retrieve existing attribute
    existing, err := s.repo.GetByID(ctx, tenant, id)
    if err != nil {
        return nil, fmt.Errorf("failed to get attribute: %w", err)
    }
    if existing == nil {
        return nil, fmt.Errorf("attribute not found")
    }

    // Input validation
    if err := s.validateUpdateInput(input); err != nil {
        return nil, fmt.Errorf("validation failed: %w", err)
    }

    // Check for breaking changes
    if err := s.validateSchemaEvolution(ctx, tenant, existing, input); err != nil {
        return nil, fmt.Errorf("schema evolution validation failed: %w", err)
    }

    // Apply updates
    updated := *existing
    updated.UpdatedAt = time.Now()
    updated.Version++

    if input.Name != nil {
        // Check name uniqueness if changing
        if *input.Name != existing.Name {
            exists, err := s.repo.ExistsByName(ctx, tenant, *input.Name)
            if err != nil {
                return nil, fmt.Errorf("failed to check name uniqueness: %w", err)
            }
            if exists {
                return nil, fmt.Errorf("attribute name '%s' already exists", *input.Name)
            }
        }
        updated.Name = *input.Name
    }

    if input.ValidationRules != nil {
        if err := s.validateValidationRules(existing.DataType, input.ValidationRules); err != nil {
            return nil, fmt.Errorf("validation rules invalid: %w", err)
        }
        updated.ValidationRules = input.ValidationRules
    }

    if input.Description != nil {
        updated.Description = input.Description
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
    s.cache.InvalidatePattern(ctx, tenant.SchemaName, "attributes:*")
    s.cache.Delete(ctx, tenant.SchemaName, fmt.Sprintf("attribute:%s", id.String()))

    s.logger.Info("Successfully updated attribute",
        zap.String("id", id.String()),
        zap.String("name", result.Name),
        zap.Int("version", result.Version),
        zap.String("tenant_schema", tenant.SchemaName),
    )

    return result, nil
}

// Validation methods
func (s *AttributeService) validateValidationRules(dataType domain.AttributeType, rules json.RawMessage) error {
    if len(rules) == 0 {
        return nil // Empty rules are valid
    }

    // Parse as generic JSON first
    var rulesMap map[string]interface{}
    if err := json.Unmarshal(rules, &rulesMap); err != nil {
        return fmt.Errorf("validation rules must be valid JSON: %w", err)
    }

    // Validate based on data type
    switch dataType {
    case domain.AttributeTypeString:
        var stringRules domain.ValidationRuleString
        if err := json.Unmarshal(rules, &stringRules); err != nil {
            return fmt.Errorf("invalid string validation rules: %w", err)
        }
        return s.validateStringRules(&stringRules)
    case domain.AttributeTypeInteger:
        var intRules domain.ValidationRuleInteger
        if err := json.Unmarshal(rules, &intRules); err != nil {
            return fmt.Errorf("invalid integer validation rules: %w", err)
        }
        return s.validateIntegerRules(&intRules)
    // ... other data types
    }

    return nil
}

func (s *AttributeService) validateSchemaEvolution(ctx context.Context, tenant *domain.TenantContext, existing *domain.Attribute, input *domain.UpdateAttributeInput) error {
    // Check if attribute is used in any Thing Class assignments
    assignmentCount, err := s.repo.GetAssignmentCount(ctx, tenant, existing.ID)
    if err != nil {
        return fmt.Errorf("failed to check assignments: %w", err)
    }

    if assignmentCount == 0 {
        return nil // No assignments, safe to modify anything
    }

    // If has assignments, only allow safe changes
    if input.ValidationRules != nil {
        if err := s.validateSafeValidationRuleChanges(existing.ValidationRules, input.ValidationRules); err != nil {
            return fmt.Errorf("validation rule change would break existing assignments: %w", err)
        }
    }

    return nil
}

func (s *AttributeService) validateSafeValidationRuleChanges(oldRules, newRules json.RawMessage) error {
    // This would implement logic to determine if validation rule changes
    // are backward compatible (e.g., relaxing constraints is safe, 
    // tightening constraints may break existing data)
    
    // For now, implement basic checks
    if len(oldRules) == 0 && len(newRules) > 0 {
        return nil // Adding rules to attribute with no rules is safe
    }

    if len(oldRules) > 0 && len(newRules) == 0 {
        return fmt.Errorf("removing validation rules from attribute with existing assignments is not allowed")
    }

    // More sophisticated rule comparison would go here
    return nil
}
```

## Repository Layer Implementation

### Data Access Patterns

```go
// PostgreSQL-specific repository implementation with multi-tenant support
type PostgresAttributeRepository struct {
    dbManager *database.Manager
    cache     cache.Cache
    logger    *zap.Logger
}

func (r *PostgresAttributeRepository) Create(ctx context.Context, tenant *domain.TenantContext, attribute *domain.Attribute) (*domain.Attribute, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return nil, fmt.Errorf("failed to get database connection: %w", err)
    }

    query := `
        INSERT INTO attributes (id, name, data_type, validation_rules, description, 
                              created_at, updated_at, created_by, is_active, version)
        VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10)
        RETURNING *`

    var result domain.Attribute
    err = db.QueryRowxContext(ctx, query,
        attribute.ID, attribute.Name, attribute.DataType, attribute.ValidationRules,
        attribute.Description, attribute.CreatedAt, attribute.UpdatedAt,
        attribute.CreatedBy, attribute.IsActive, attribute.Version,
    ).StructScan(&result)

    if err != nil {
        if isUniqueViolation(err) {
            return nil, fmt.Errorf("attribute name already exists: %w", err)
        }
        return nil, fmt.Errorf("failed to insert attribute: %w", err)
    }

    return &result, nil
}

func (r *PostgresAttributeRepository) GetByID(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.Attribute, error) {
    // Try cache first
    cacheKey := fmt.Sprintf("attribute:%s", id.String())
    var cached domain.Attribute
    if err := r.cache.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
        return &cached, nil
    }

    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return nil, fmt.Errorf("failed to get database connection: %w", err)
    }

    query := `
        SELECT id, name, data_type, validation_rules, description,
               created_at, updated_at, created_by, updated_by,
               is_active, version
        FROM attributes 
        WHERE id = $1 AND is_active = TRUE`

    var result domain.Attribute
    err = db.QueryRowxContext(ctx, query, id).StructScan(&result)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, nil
        }
        return nil, fmt.Errorf("failed to query attribute: %w", err)
    }

    // Cache the result
    r.cache.Set(ctx, tenant.SchemaName, cacheKey, &result)

    return &result, nil
}

func (r *PostgresAttributeRepository) List(ctx context.Context, tenant *domain.TenantContext, filter *domain.AttributeFilter, limit int, offset int) ([]*domain.Attribute, int, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return nil, 0, fmt.Errorf("failed to get database connection: %w", err)
    }

    // Build dynamic query with filters
    whereClause, args := r.buildWhereClause(filter)
    
    // Count query
    countQuery := fmt.Sprintf("SELECT COUNT(*) FROM attributes WHERE %s", whereClause)
    var totalCount int
    if err := db.QueryRowxContext(ctx, countQuery, args...).Scan(&totalCount); err != nil {
        return nil, 0, fmt.Errorf("failed to count attributes: %w", err)
    }

    // Main query with pagination
    query := fmt.Sprintf(`
        SELECT id, name, data_type, validation_rules, description,
               created_at, updated_at, created_by, updated_by,
               is_active, version
        FROM attributes 
        WHERE %s
        ORDER BY name ASC
        LIMIT $%d OFFSET $%d`, whereClause, len(args)+1, len(args)+2)

    args = append(args, limit, offset)

    rows, err := db.QueryxContext(ctx, query, args...)
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

    return attributes, totalCount, nil
}

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

    if filter.DataType != nil {
        argCount++
        conditions = append(conditions, fmt.Sprintf("data_type = $%d", argCount))
        args = append(args, *filter.DataType)
    }

    if filter.IsActive != nil {
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

    return strings.Join(conditions, " AND "), args
}

// Helper method to get assignment count for schema evolution validation
func (r *PostgresAttributeRepository) GetAssignmentCount(ctx context.Context, tenant *domain.TenantContext, attributeID uuid.UUID) (int, error) {
    db, err := r.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return 0, fmt.Errorf("failed to get database connection: %w", err)
    }

    query := `
        SELECT COUNT(*) 
        FROM thing_class_attributes 
        WHERE attribute_id = $1 AND is_active = TRUE`

    var count int
    err = db.QueryRowxContext(ctx, query, attributeID).Scan(&count)
    if err != nil {
        return 0, fmt.Errorf("failed to count assignments: %w", err)
    }

    return count, nil
}
```

## Error Handling and Validation

### Comprehensive Error Management

```go
// Custom error types for Attribute operations
type AttributeError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Field   string `json:"field,omitempty"`
    Details map[string]interface{} `json:"details,omitempty"`
}

func (e *AttributeError) Error() string {
    return e.Message
}

// Error constants for consistent error handling
const (
    ErrCodeAttributeNotFound        = "ATTRIBUTE_NOT_FOUND"
    ErrCodeAttributeNameExists      = "ATTRIBUTE_NAME_EXISTS"
    ErrCodeInvalidDataType          = "INVALID_DATA_TYPE"
    ErrCodeInvalidValidationRules   = "INVALID_VALIDATION_RULES"
    ErrCodeSchemaEvolutionBlocked   = "SCHEMA_EVOLUTION_BLOCKED"
    ErrCodeAttributeInUse           = "ATTRIBUTE_IN_USE"
    ErrCodeValidationFailed         = "VALIDATION_FAILED"
)

// Error factory functions for consistent error creation
func NewAttributeNotFoundError(id uuid.UUID) *AttributeError {
    return &AttributeError{
        Code:    ErrCodeAttributeNotFound,
        Message: fmt.Sprintf("Attribute with ID %s not found", id.String()),
        Details: map[string]interface{}{"attribute_id": id.String()},
    }
}

func NewAttributeNameExistsError(name string) *AttributeError {
    return &AttributeError{
        Code:    ErrCodeAttributeNameExists,
        Message: fmt.Sprintf("Attribute name '%s' already exists", name),
        Field:   "name",
        Details: map[string]interface{}{"name": name},
    }
}

func NewValidationRulesError(dataType domain.AttributeType, err error) *AttributeError {
    return &AttributeError{
        Code:    ErrCodeInvalidValidationRules,
        Message: fmt.Sprintf("Invalid validation rules for data type %s: %s", dataType, err.Error()),
        Field:   "validationRules",
        Details: map[string]interface{}{
            "data_type": dataType,
            "validation_error": err.Error(),
        },
    }
}

func NewSchemaEvolutionError(reason string, assignmentCount int) *AttributeError {
    return &AttributeError{
        Code:    ErrCodeSchemaEvolutionBlocked,
        Message: fmt.Sprintf("Schema evolution blocked: %s (affects %d assignments)", reason, assignmentCount),
        Details: map[string]interface{}{
            "reason": reason,
            "assignment_count": assignmentCount,
        },
    }
}
```

## Performance Optimization

### Caching Strategy

```go
// Multi-layer caching for optimal performance
type AttributeCacheManager struct {
    redis    cache.Cache
    memory   cache.Cache  // L1 cache for frequently accessed data
    logger   *zap.Logger
}

func (c *AttributeCacheManager) GetAttribute(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) (*domain.Attribute, error) {
    cacheKey := fmt.Sprintf("attribute:%s", id.String())
    
    // Try L1 memory cache first
    var cached domain.Attribute
    if err := c.memory.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
        c.logger.Debug("Attribute cache hit (memory)", zap.String("id", id.String()))
        return &cached, nil
    }
    
    // Try L2 Redis cache
    if err := c.redis.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
        c.logger.Debug("Attribute cache hit (redis)", zap.String("id", id.String()))
        // Store in memory cache for faster access
        c.memory.Set(ctx, tenant.SchemaName, cacheKey, &cached)
        return &cached, nil
    }
    
    return nil, cache.ErrCacheMiss
}

func (c *AttributeCacheManager) SetAttribute(ctx context.Context, tenant *domain.TenantContext, attribute *domain.Attribute) {
    cacheKey := fmt.Sprintf("attribute:%s", attribute.ID.String())
    
    // Store in both cache layers
    c.memory.Set(ctx, tenant.SchemaName, cacheKey, attribute)
    c.redis.Set(ctx, tenant.SchemaName, cacheKey, attribute)
    
    c.logger.Debug("Cached attribute", 
        zap.String("id", attribute.ID.String()),
        zap.String("name", attribute.Name),
    )
}

func (c *AttributeCacheManager) InvalidateAttribute(ctx context.Context, tenant *domain.TenantContext, id uuid.UUID) {
    cacheKey := fmt.Sprintf("attribute:%s", id.String())
    
    c.memory.Delete(ctx, tenant.SchemaName, cacheKey)
    c.redis.Delete(ctx, tenant.SchemaName, cacheKey)
    
    // Also invalidate related patterns
    c.redis.InvalidatePattern(ctx, tenant.SchemaName, "attributes:list:*")
    c.memory.InvalidatePattern(ctx, tenant.SchemaName, "attributes:list:*")
}
```

### Database Query Optimization

```sql
-- Optimized queries for common access patterns

-- 1. Most frequently used: Get attributes by data type with pagination
EXPLAIN (ANALYZE, BUFFERS) 
SELECT id, name, data_type, validation_rules, description,
       created_at, updated_at, created_by, is_active, version
FROM attributes 
WHERE data_type = 'string' AND is_active = TRUE
ORDER BY name ASC
LIMIT 20 OFFSET 0;

-- 2. Search attributes by name pattern
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, data_type, description,
       (SELECT COUNT(*) FROM thing_class_attributes tca 
        WHERE tca.attribute_id = attributes.id AND tca.is_active = TRUE) as assignment_count
FROM attributes 
WHERE to_tsvector('english', name) @@ plainto_tsquery('english', 'email')
  AND is_active = TRUE
ORDER BY ts_rank(to_tsvector('english', name), plainto_tsquery('english', 'email')) DESC;

-- 3. Get attributes with assignment information for impact analysis
EXPLAIN (ANALYZE, BUFFERS)
SELECT a.id, a.name, a.data_type, a.validation_rules,
       COUNT(tca.id) as assignment_count,
       ARRAY_AGG(tc.name) FILTER (WHERE tca.id IS NOT NULL) as assigned_thing_classes
FROM attributes a
LEFT JOIN thing_class_attributes tca ON a.id = tca.attribute_id AND tca.is_active = TRUE
LEFT JOIN thing_classes tc ON tca.thing_class_id = tc.id AND tc.is_active = TRUE
WHERE a.is_active = TRUE
GROUP BY a.id, a.name, a.data_type, a.validation_rules
ORDER BY assignment_count DESC, a.name ASC;

-- 4. Validation rules analysis query
EXPLAIN (ANALYZE, BUFFERS)
SELECT data_type, 
       COUNT(*) as total_attributes,
       COUNT(CASE WHEN validation_rules != '{}' THEN 1 END) as with_validation_rules,
       AVG(jsonb_array_length(jsonb_object_keys(validation_rules))) as avg_rule_count
FROM attributes 
WHERE is_active = TRUE
GROUP BY data_type
ORDER BY total_attributes DESC;
```

## Testing Strategy

### Comprehensive Test Coverage

```go
// Unit tests for Attribute service layer
func TestAttributeService_CreateAttribute(t *testing.T) {
    tests := []struct {
        name          string
        input         *domain.CreateAttributeInput
        setupMocks    func(*mocks.AttributeRepository)
        expectedError string
        validate      func(*testing.T, *domain.Attribute)
    }{
        {
            name: "successful string attribute creation",
            input: &domain.CreateAttributeInput{
                Name:     "email_address",
                DataType: domain.AttributeTypeString,
                ValidationRules: json.RawMessage(`{
                    "required": true,
                    "format": "email",
                    "maxLength": 255
                }`),
                Description: stringPtr("User email address"),
            },
            setupMocks: func(repo *mocks.AttributeRepository) {
                repo.On("ExistsByName", mock.Anything, mock.Anything, "email_address").Return(false, nil)
                repo.On("Create", mock.Anything, mock.Anything, mock.AnythingOfType("*domain.Attribute")).
                    Return(mockAttributeWithID(), nil)
            },
            validate: func(t *testing.T, result *domain.Attribute) {
                assert.Equal(t, "email_address", result.Name)
                assert.Equal(t, domain.AttributeTypeString, result.DataType)
                assert.NotNil(t, result.Description)
                assert.Equal(t, "User email address", *result.Description)
            },
        },
        {
            name: "duplicate name error",
            input: &domain.CreateAttributeInput{
                Name:     "existing_name",
                DataType: domain.AttributeTypeString,
            },
            setupMocks: func(repo *mocks.AttributeRepository) {
                repo.On("ExistsByName", mock.Anything, mock.Anything, "existing_name").Return(true, nil)
            },
            expectedError: "attribute name 'existing_name' already exists",
        },
        {
            name: "invalid validation rules",
            input: &domain.CreateAttributeInput{
                Name:     "test_attribute",
                DataType: domain.AttributeTypeInteger,
                ValidationRules: json.RawMessage(`{
                    "min": "not_a_number",
                    "max": 100
                }`),
            },
            expectedError: "validation rules invalid",
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            mockRepo := new(mocks.AttributeRepository)
            mockCache := new(mocks.Cache)
            logger := zaptest.NewLogger(t)
            
            if tt.setupMocks != nil {
                tt.setupMocks(mockRepo)
            }

            service := service.NewAttributeService(mockRepo, nil, logger)
            tenant := &domain.TenantContext{
                SchemaName: "test_tenant",
                UserID:     uuid.New(),
            }

            // Execute
            result, err := service.CreateAttribute(context.Background(), tenant, tt.input)

            // Assert
            if tt.expectedError != "" {
                assert.Error(t, err)
                assert.Contains(t, err.Error(), tt.expectedError)
                assert.Nil(t, result)
            } else {
                assert.NoError(t, err)
                assert.NotNil(t, result)
                if tt.validate != nil {
                    tt.validate(t, result)
                }
            }

            mockRepo.AssertExpectations(t)
        })
    }
}

// Integration tests for repository layer
func TestPostgresAttributeRepository_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping integration test")
    }

    // Setup test database
    testDB := setupTestDatabase(t)
    defer teardownTestDatabase(t, testDB)

    repo := postgres.NewAttributeRepository(testDB, nil, zaptest.NewLogger(t))
    tenant := &domain.TenantContext{SchemaName: "test_tenant"}

    t.Run("complete attribute lifecycle", func(t *testing.T) {
        // Create
        attr := &domain.Attribute{
            BaseEntity: domain.BaseEntity{
                ID:        uuid.New(),
                CreatedAt: time.Now(),
                UpdatedAt: time.Now(),
                CreatedBy: uuid.New(),
                IsActive:  true,
                Version:   1,
            },
            Name:     "test_attribute",
            DataType: domain.AttributeTypeString,
            ValidationRules: json.RawMessage(`{"required": true, "maxLength": 100}`),
            Description: stringPtr("Test attribute description"),
        }

        created, err := repo.Create(context.Background(), tenant, attr)
        assert.NoError(t, err)
        assert.NotNil(t, created)
        assert.Equal(t, attr.Name, created.Name)

        // Read
        retrieved, err := repo.GetByID(context.Background(), tenant, created.ID)
        assert.NoError(t, err)
        assert.NotNil(t, retrieved)
        assert.Equal(t, created.ID, retrieved.ID)
        assert.Equal(t, created.Name, retrieved.Name)

        // Update
        updateInput := &domain.UpdateAttributeInput{
            Description: stringPtr("Updated description"),
            ValidationRules: json.RawMessage(`{"required": true, "maxLength": 200}`),
        }

        // Apply updates to retrieved object
        retrieved.Description = updateInput.Description
        retrieved.ValidationRules = updateInput.ValidationRules
        retrieved.UpdatedAt = time.Now()
        retrieved.Version++

        updated, err := repo.Update(context.Background(), tenant, retrieved)
        assert.NoError(t, err)
        assert.NotNil(t, updated)
        assert.Equal(t, "Updated description", *updated.Description)
        assert.Equal(t, 2, updated.Version)

        // List
        attributes, total, err := repo.List(context.Background(), tenant, nil, 10, 0)
        assert.NoError(t, err)
        assert.GreaterOrEqual(t, total, 1)
        assert.GreaterOrEqual(t, len(attributes), 1)

        // Soft delete
        err = repo.SoftDelete(context.Background(), tenant, updated.ID)
        assert.NoError(t, err)

        // Verify soft delete
        deleted, err := repo.GetByID(context.Background(), tenant, updated.ID)
        assert.NoError(t, err)
        assert.Nil(t, deleted) // Should not return soft-deleted records
    })
}

// GraphQL integration tests
func TestAttributeGraphQLAPI_Integration(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping GraphQL integration test")
    }

    // Setup test GraphQL server
    testServer := setupTestGraphQLServer(t)
    defer testServer.Close()

    client := graphql.NewClient(testServer.URL + "/graphql")

    t.Run("create attribute mutation", func(t *testing.T) {
        mutation := `
            mutation CreateAttribute($input: CreateAttributeInput!) {
                createAttribute(input: $input) {
                    attribute {
                        id
                        name
                        dataType
                        validationRules
                        description
                        createdAt
                        version
                    }
                    errors {
                        field
                        message
                        code
                    }
                }
            }
        `

        variables := map[string]interface{}{
            "input": map[string]interface{}{
                "name":     "integration_test_attr",
                "dataType": "STRING",
                "validationRules": map[string]interface{}{
                    "required":  true,
                    "maxLength": 255,
                },
                "description": "Integration test attribute",
            },
        }

        var response struct {
            CreateAttribute struct {
                Attribute *struct {
                    ID              string                 `json:"id"`
                    Name            string                 `json:"name"`
                    DataType        string                 `json:"dataType"`
                    ValidationRules map[string]interface{} `json:"validationRules"`
                    Description     *string                `json:"description"`
                    CreatedAt       string                 `json:"createdAt"`
                    Version         int                    `json:"version"`
                } `json:"attribute"`
                Errors []struct {
                    Field   *string `json:"field"`
                    Message string  `json:"message"`
                    Code    *string `json:"code"`
                } `json:"errors"`
            } `json:"createAttribute"`
        }

        err := client.Query(context.Background(), mutation, variables, &response, addAuthHeaders())
        assert.NoError(t, err)
        assert.Empty(t, response.CreateAttribute.Errors)
        assert.NotNil(t, response.CreateAttribute.Attribute)
        assert.Equal(t, "integration_test_attr", response.CreateAttribute.Attribute.Name)
        assert.Equal(t, "STRING", response.CreateAttribute.Attribute.DataType)
        assert.NotNil(t, response.CreateAttribute.Attribute.Description)
        assert.Equal(t, "Integration test attribute", *response.CreateAttribute.Attribute.Description)
    })
}

// Performance tests
func TestAttributeService_Performance(t *testing.T) {
    if testing.Short() {
        t.Skip("Skipping performance test")
    }

    // Setup service with real dependencies
    service, tenant := setupPerformanceTest(t)

    t.Run("bulk attribute creation performance", func(t *testing.T) {
        attributeCount := 1000
        start := time.Now()

        for i := 0; i < attributeCount; i++ {
            input := &domain.CreateAttributeInput{
                Name:     fmt.Sprintf("perf_test_attr_%d", i),
                DataType: domain.AttributeTypeString,
                ValidationRules: json.RawMessage(`{"required": true}`),
                Description: stringPtr(fmt.Sprintf("Performance test attribute %d", i)),
            }

            _, err := service.CreateAttribute(context.Background(), tenant, input)
            assert.NoError(t, err)
        }

        duration := time.Since(start)
        avgTime := duration / time.Duration(attributeCount)

        t.Logf("Created %d attributes in %v (avg: %v per attribute)", attributeCount, duration, avgTime)
        
        // Assert performance targets
        assert.Less(t, avgTime, 100*time.Millisecond, "Average creation time should be under 100ms")
    })
}
```

This technical specification provides comprehensive implementation guidance for the Attribute entity, covering all aspects from data modeling to performance optimization and testing strategies.