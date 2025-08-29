# Attribute Assignment Entity - Technical Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**Entity:** Attribute Assignment - Bridge connecting Thing Classes to Attributes with constraints  

## Technical Architecture Overview

The Attribute Assignment entity implements the critical bridge layer between Thing Classes and Attributes, enabling dynamic schema composition through configurable assignments. Each assignment defines not only which Attributes belong to which Thing Classes, but also their behavioral characteristics: required vs optional, validation rule extensions, default values, display metadata, and UI presentation hints. This entity serves as the schema configuration layer that transforms the UDM system from a static schema model into a truly dynamic, runtime-configurable data architecture.

## Core Requirements

### Functional Requirements

**FR-1: Assignment Lifecycle Management**
- Create new Attribute Assignments linking Thing Classes to Attributes with constraints and metadata
- Update existing Assignment configurations with impact analysis and validation
- Retrieve Assignment configurations with full relationship hydration and schema composition
- Soft delete Assignment configurations with dependency validation and impact assessment

**FR-2: Schema Composition Engine**
- Generate complete Thing Class schemas by aggregating all active Attribute Assignments
- Validate schema coherence and detect conflicts between assignment constraints
- Support assignment-level validation rule extensions that complement base Attribute rules
- Enable dynamic schema evolution through assignment configuration changes

**FR-3: Constraint and Metadata Management**
- Define required/optional flags for each Attribute Assignment with inheritance rules
- Store assignment-specific validation rules that extend base Attribute validation
- Manage default values with type safety and validation against Attribute data types
- Support UI presentation metadata including display names, ordering, and layout hints

**FR-4: Impact Analysis and Validation**
- Analyze the impact of Assignment changes on existing Thing instances and their data
- Validate Assignment configurations against existing Thing Class and Attribute definitions
- Provide migration recommendations for breaking Assignment changes
- Ensure referential integrity across tenant boundaries and entity relationships

### Non-Functional Requirements

**NFR-1: Performance Targets**
- Single Assignment retrieval with relationships: <200ms P95 latency
- Assignment listing with filtering and pagination: <500ms P95 latency
- Thing Class schema composition (all assignments): <300ms P95 latency
- Assignment creation/updates with validation: <400ms P95 latency
- Impact analysis for assignment changes: <1000ms P95 latency

**NFR-2: Scalability Requirements**
- Support 10,000+ Assignments per tenant schema without performance degradation
- Handle 1,000+ concurrent Assignment operations across all tenants
- Scale to 100+ tenant schemas with linear performance characteristics
- Maintain sub-linear query performance as Assignment count grows through strategic indexing

**NFR-3: Data Consistency and Integrity**
- Ensure Assignment uniqueness per Thing Class + Attribute pair within tenant scope
- Maintain referential integrity with Thing Class and Attribute entities
- Support transactional operations for complex Assignment modifications
- Provide data validation at API, service, and database levels with comprehensive error handling

## Data Model Specification

### Database Schema

```sql
-- Thing Class Attributes assignment table (per tenant schema)
CREATE TABLE thing_class_attributes (
    -- Primary key and identity
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Core relationship identifiers
    thing_class_id UUID NOT NULL,
    attribute_id UUID NOT NULL,
    
    -- Assignment configuration
    is_required BOOLEAN DEFAULT FALSE,
    sort_order INTEGER DEFAULT 0,
    display_name VARCHAR(255),
    assignment_validation_rules JSONB DEFAULT '{}',
    default_value JSONB,
    ui_hints JSONB DEFAULT '{}',
    
    -- Assignment metadata
    description TEXT,
    assignment_type VARCHAR(50) DEFAULT 'standard' CHECK (assignment_type IN ('standard', 'computed', 'inherited')),
    visibility VARCHAR(20) DEFAULT 'visible' CHECK (visibility IN ('visible', 'hidden', 'readonly')),
    
    -- Standard entity metadata
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_by UUID,
    is_active BOOLEAN DEFAULT TRUE,
    version INTEGER DEFAULT 1,
    
    -- Business constraints
    CONSTRAINT unique_thing_class_attribute UNIQUE(thing_class_id, attribute_id),
    CONSTRAINT valid_sort_order CHECK (sort_order >= 0),
    CONSTRAINT valid_assignment_validation_rules CHECK (assignment_validation_rules IS NOT NULL),
    CONSTRAINT valid_display_name_length CHECK (
        display_name IS NULL OR (length(display_name) >= 1 AND length(display_name) <= 255)
    ),
    CONSTRAINT valid_description_length CHECK (
        description IS NULL OR length(description) <= 1000
    ),
    
    -- Foreign key constraints (within tenant schema)
    CONSTRAINT fk_thing_class_attribute_thing_class 
        FOREIGN KEY (thing_class_id) REFERENCES thing_classes(id) ON DELETE CASCADE,
    CONSTRAINT fk_thing_class_attribute_attribute 
        FOREIGN KEY (attribute_id) REFERENCES attributes(id) ON DELETE CASCADE
);

-- Table comments for documentation
COMMENT ON TABLE thing_class_attributes IS 'Assignment bridge connecting Thing Classes to Attributes with constraints and metadata';
COMMENT ON COLUMN thing_class_attributes.assignment_validation_rules IS 'JSON validation rules that extend base Attribute validation';
COMMENT ON COLUMN thing_class_attributes.default_value IS 'JSON-encoded default value respecting Attribute data type constraints';
COMMENT ON COLUMN thing_class_attributes.ui_hints IS 'JSON metadata for UI presentation and layout configuration';
COMMENT ON COLUMN thing_class_attributes.assignment_type IS 'Type of assignment: standard, computed, or inherited';
COMMENT ON COLUMN thing_class_attributes.visibility IS 'UI visibility: visible, hidden, or readonly';
```

### Performance-Optimized Indexes

```sql
-- Primary access patterns optimization
CREATE INDEX idx_thing_class_attributes_thing_class_id ON thing_class_attributes(thing_class_id) 
WHERE is_active = TRUE;

CREATE INDEX idx_thing_class_attributes_attribute_id ON thing_class_attributes(attribute_id) 
WHERE is_active = TRUE;

CREATE INDEX idx_thing_class_attributes_sort_order ON thing_class_attributes(thing_class_id, sort_order) 
WHERE is_active = TRUE;

-- Composite index for schema composition queries
CREATE INDEX idx_thing_class_attributes_composition ON thing_class_attributes(thing_class_id, is_active, sort_order)
WHERE is_active = TRUE;

-- Index for required/optional filtering
CREATE INDEX idx_thing_class_attributes_required ON thing_class_attributes(thing_class_id, is_required, sort_order)
WHERE is_active = TRUE;

-- JSON index for validation rules analysis
CREATE INDEX idx_thing_class_attributes_validation_rules_gin ON thing_class_attributes 
USING GIN(assignment_validation_rules) 
WHERE is_active = TRUE;

-- Index for assignment metadata searches
CREATE INDEX idx_thing_class_attributes_assignment_type ON thing_class_attributes(assignment_type) 
WHERE is_active = TRUE;

CREATE INDEX idx_thing_class_attributes_visibility ON thing_class_attributes(visibility) 
WHERE is_active = TRUE;

-- Full-text search on display names and descriptions
CREATE INDEX idx_thing_class_attributes_display_name_text ON thing_class_attributes 
USING GIN(to_tsvector('english', COALESCE(display_name, ''))) 
WHERE is_active = TRUE AND display_name IS NOT NULL;

CREATE INDEX idx_thing_class_attributes_description_text ON thing_class_attributes 
USING GIN(to_tsvector('english', COALESCE(description, ''))) 
WHERE is_active = TRUE AND description IS NOT NULL;

-- Index for temporal queries and audit trails
CREATE INDEX idx_thing_class_attributes_created_at ON thing_class_attributes(created_at DESC) 
WHERE is_active = TRUE;

CREATE INDEX idx_thing_class_attributes_updated_at ON thing_class_attributes(updated_at DESC) 
WHERE is_active = TRUE;

-- Version tracking for optimistic locking
CREATE INDEX idx_thing_class_attributes_version ON thing_class_attributes(version) 
WHERE is_active = TRUE;
```

### Schema Composition Views

```sql
-- Materialized view for optimized Thing Class schema composition
CREATE MATERIALIZED VIEW thing_class_schemas AS
SELECT 
    tc.id as thing_class_id,
    tc.name as thing_class_name,
    tc.description as thing_class_description,
    COUNT(tca.id) as total_assignments,
    COUNT(CASE WHEN tca.is_required = TRUE THEN 1 END) as required_assignments,
    COUNT(CASE WHEN tca.is_required = FALSE THEN 1 END) as optional_assignments,
    ARRAY_AGG(
        tca.id ORDER BY tca.sort_order ASC, tca.created_at ASC
    ) as assignment_ids,
    ARRAY_AGG(
        jsonb_build_object(
            'id', tca.id,
            'attributeId', tca.attribute_id,
            'attributeName', a.name,
            'attributeDataType', a.data_type,
            'isRequired', tca.is_required,
            'sortOrder', tca.sort_order,
            'displayName', COALESCE(tca.display_name, a.name),
            'hasDefaultValue', (tca.default_value IS NOT NULL),
            'hasCustomValidation', (tca.assignment_validation_rules != '{}'),
            'visibility', tca.visibility,
            'assignmentType', tca.assignment_type
        ) ORDER BY tca.sort_order ASC, tca.created_at ASC
    ) as assignment_summary
FROM thing_classes tc
LEFT JOIN thing_class_attributes tca ON tc.id = tca.thing_class_id AND tca.is_active = TRUE
LEFT JOIN attributes a ON tca.attribute_id = a.id AND a.is_active = TRUE
WHERE tc.is_active = TRUE
GROUP BY tc.id, tc.name, tc.description;

-- Unique index on materialized view
CREATE UNIQUE INDEX idx_thing_class_schemas_thing_class_id ON thing_class_schemas(thing_class_id);

-- Index for efficient schema queries
CREATE INDEX idx_thing_class_schemas_total_assignments ON thing_class_schemas(total_assignments);
CREATE INDEX idx_thing_class_schemas_required_assignments ON thing_class_schemas(required_assignments);

-- Function to refresh materialized view
CREATE OR REPLACE FUNCTION refresh_thing_class_schemas()
RETURNS VOID AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY thing_class_schemas;
END;
$$ LANGUAGE plpgsql;
```

### Audit and Change Tracking

```sql
-- Assignment audit table for comprehensive change tracking
CREATE TABLE thing_class_attributes_audit (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assignment_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    old_values JSONB,
    new_values JSONB,
    changed_by UUID NOT NULL,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    change_reason TEXT,
    client_info JSONB DEFAULT '{}',
    impact_analysis JSONB DEFAULT '{}',
    
    -- Foreign key to original assignment (may be null for DELETE operations)
    CONSTRAINT fk_audit_assignment FOREIGN KEY (assignment_id) 
        REFERENCES thing_class_attributes(id) ON DELETE SET NULL
);

-- Audit indexes for performance
CREATE INDEX idx_thing_class_attributes_audit_assignment_id ON thing_class_attributes_audit(assignment_id);
CREATE INDEX idx_thing_class_attributes_audit_changed_at ON thing_class_attributes_audit(changed_at DESC);
CREATE INDEX idx_thing_class_attributes_audit_changed_by ON thing_class_attributes_audit(changed_by);
CREATE INDEX idx_thing_class_attributes_audit_operation ON thing_class_attributes_audit(operation);

-- Trigger for automatic audit logging with impact analysis
CREATE OR REPLACE FUNCTION thing_class_attributes_audit_trigger()
RETURNS TRIGGER AS $$
DECLARE
    change_reason_text TEXT;
    client_context JSONB;
    impact_data JSONB;
BEGIN
    -- Extract change context from session variables if available
    change_reason_text := COALESCE(
        current_setting('udm.change_reason', true),
        CASE TG_OP
            WHEN 'INSERT' THEN 'Assignment created'
            WHEN 'UPDATE' THEN 'Assignment updated'
            WHEN 'DELETE' THEN 'Assignment deleted'
        END
    );
    
    -- Build client context
    client_context := jsonb_build_object(
        'application', COALESCE(current_setting('udm.client_app', true), 'unknown'),
        'version', COALESCE(current_setting('udm.client_version', true), 'unknown'),
        'user_agent', COALESCE(current_setting('udm.user_agent', true), null),
        'ip_address', COALESCE(current_setting('udm.client_ip', true), null),
        'session_id', COALESCE(current_setting('udm.session_id', true), null)
    );

    -- Build impact analysis data
    IF TG_OP = 'INSERT' THEN
        impact_data := jsonb_build_object(
            'thing_class_name', (SELECT name FROM thing_classes WHERE id = NEW.thing_class_id),
            'attribute_name', (SELECT name FROM attributes WHERE id = NEW.attribute_id),
            'is_required', NEW.is_required,
            'has_default', (NEW.default_value IS NOT NULL),
            'potential_impact', 'new_assignment'
        );
        
        INSERT INTO thing_class_attributes_audit (
            assignment_id, operation, new_values, changed_by, 
            change_reason, client_info, impact_analysis
        )
        VALUES (
            NEW.id, 'INSERT', to_jsonb(NEW), NEW.created_by,
            change_reason_text, client_context, impact_data
        );
        RETURN NEW;
        
    ELSIF TG_OP = 'UPDATE' THEN
        impact_data := jsonb_build_object(
            'thing_class_name', (SELECT name FROM thing_classes WHERE id = NEW.thing_class_id),
            'attribute_name', (SELECT name FROM attributes WHERE id = NEW.attribute_id),
            'required_changed', (OLD.is_required != NEW.is_required),
            'sort_order_changed', (OLD.sort_order != NEW.sort_order),
            'validation_rules_changed', (OLD.assignment_validation_rules != NEW.assignment_validation_rules),
            'default_value_changed', (COALESCE(OLD.default_value, 'null'::jsonb) != COALESCE(NEW.default_value, 'null'::jsonb)),
            'potential_impact', CASE 
                WHEN OLD.is_required = FALSE AND NEW.is_required = TRUE THEN 'breaking_change'
                WHEN OLD.is_required = TRUE AND NEW.is_required = FALSE THEN 'relaxing_change'
                ELSE 'metadata_change'
            END
        );
        
        INSERT INTO thing_class_attributes_audit (
            assignment_id, operation, old_values, new_values, changed_by,
            change_reason, client_info, impact_analysis
        )
        VALUES (
            NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), 
            COALESCE(NEW.updated_by, NEW.created_by),
            change_reason_text, client_context, impact_data
        );
        RETURN NEW;
        
    ELSIF TG_OP = 'DELETE' THEN
        impact_data := jsonb_build_object(
            'thing_class_name', (SELECT name FROM thing_classes WHERE id = OLD.thing_class_id),
            'attribute_name', (SELECT name FROM attributes WHERE id = OLD.attribute_id),
            'was_required', OLD.is_required,
            'had_default', (OLD.default_value IS NOT NULL),
            'potential_impact', CASE 
                WHEN OLD.is_required = TRUE THEN 'breaking_change'
                ELSE 'schema_reduction'
            END
        );
        
        INSERT INTO thing_class_attributes_audit (
            assignment_id, operation, old_values, changed_by,
            change_reason, client_info, impact_analysis
        )
        VALUES (
            OLD.id, 'DELETE', to_jsonb(OLD),
            COALESCE(OLD.updated_by, OLD.created_by),
            change_reason_text, client_context, impact_data
        );
        RETURN OLD;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Create trigger for audit logging
CREATE TRIGGER thing_class_attributes_audit_trig
    AFTER INSERT OR UPDATE OR DELETE ON thing_class_attributes
    FOR EACH ROW EXECUTE FUNCTION thing_class_attributes_audit_trigger();
```

### Domain Model

```go
// Core Assignment entity with all properties and relationships
type ThingClassAttribute struct {
    BaseEntity
    ThingClassID              uuid.UUID       `json:"thingClassId" db:"thing_class_id" validate:"required"`
    AttributeID               uuid.UUID       `json:"attributeId" db:"attribute_id" validate:"required"`
    IsRequired                bool            `json:"isRequired" db:"is_required"`
    SortOrder                 int             `json:"sortOrder" db:"sort_order" validate:"min=0"`
    DisplayName               *string         `json:"displayName" db:"display_name" validate:"omitempty,min=1,max=255"`
    AssignmentValidationRules json.RawMessage `json:"assignmentValidationRules" db:"assignment_validation_rules"`
    DefaultValue              json.RawMessage `json:"defaultValue" db:"default_value"`
    UIHints                   json.RawMessage `json:"uiHints" db:"ui_hints"`
    Description               *string         `json:"description" db:"description" validate:"omitempty,max=1000"`
    AssignmentType            AssignmentType  `json:"assignmentType" db:"assignment_type" validate:"required"`
    Visibility                VisibilityType  `json:"visibility" db:"visibility" validate:"required"`

    // Computed fields (not persisted, loaded separately)
    ThingClass         *ThingClass `json:"thingClass,omitempty" db:"-"`
    Attribute          *Attribute  `json:"attribute,omitempty" db:"-"`
    UsageCount         *int        `json:"usageCount,omitempty" db:"-"`
    ValidationSummary  *ValidationSummary `json:"validationSummary,omitempty" db:"-"`
}

// Assignment type enumeration
type AssignmentType string

const (
    AssignmentTypeStandard  AssignmentType = "standard"
    AssignmentTypeComputed  AssignmentType = "computed"
    AssignmentTypeInherited AssignmentType = "inherited"
)

// Visibility type enumeration
type VisibilityType string

const (
    VisibilityTypeVisible  VisibilityType = "visible"
    VisibilityTypeHidden   VisibilityType = "hidden"
    VisibilityTypeReadonly VisibilityType = "readonly"
)

// Schema composition result
type ThingClassSchema struct {
    ThingClass      *ThingClass            `json:"thingClass"`
    Assignments     []*ThingClassAttribute `json:"assignments"`
    TotalCount      int                    `json:"totalCount"`
    RequiredCount   int                    `json:"requiredCount"`
    OptionalCount   int                    `json:"optionalCount"`
    ValidationRules json.RawMessage        `json:"validationRules"`
    SchemaHash      string                 `json:"schemaHash"` // For change detection
}

// Validation summary for assignment
type ValidationSummary struct {
    HasBaseValidation       bool            `json:"hasBaseValidation"`
    HasAssignmentValidation bool            `json:"hasAssignmentValidation"`
    HasDefaultValue         bool            `json:"hasDefaultValue"`
    EffectiveRules          json.RawMessage `json:"effectiveRules"`
    RulesSummary           string          `json:"rulesSummary"`
}

// Input types for GraphQL operations
type CreateAttributeAssignmentInput struct {
    ThingClassID              uuid.UUID       `json:"thingClassId" validate:"required"`
    AttributeID               uuid.UUID       `json:"attributeId" validate:"required"`
    IsRequired                bool            `json:"isRequired"`
    SortOrder                 int             `json:"sortOrder" validate:"min=0"`
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
    SortOrder                 *int            `json:"sortOrder,omitempty" validate:"omitempty,min=0"`
    DisplayName               *string         `json:"displayName,omitempty" validate:"omitempty,min=1,max=255"`
    AssignmentValidationRules json.RawMessage `json:"assignmentValidationRules,omitempty"`
    DefaultValue              json.RawMessage `json:"defaultValue,omitempty"`
    UIHints                   json.RawMessage `json:"uiHints,omitempty"`
    Description               *string         `json:"description,omitempty" validate:"omitempty,max=1000"`
    AssignmentType            *AssignmentType `json:"assignmentType,omitempty"`
    Visibility                *VisibilityType `json:"visibility,omitempty"`
}

// Filter types for queries
type AttributeAssignmentFilter struct {
    ThingClassID    *uuid.UUID      `json:"thingClassId,omitempty"`
    AttributeID     *uuid.UUID      `json:"attributeId,omitempty"`
    IsRequired      *bool           `json:"isRequired,omitempty"`
    AssignmentType  *AssignmentType `json:"assignmentType,omitempty"`
    Visibility      *VisibilityType `json:"visibility,omitempty"`
    HasDefaultValue *bool           `json:"hasDefaultValue,omitempty"`
    CreatedBy       *uuid.UUID      `json:"createdBy,omitempty"`
    CreatedAfter    *time.Time      `json:"createdAfter,omitempty"`
    CreatedBefore   *time.Time      `json:"createdBefore,omitempty"`
    DisplayNameContains *string     `json:"displayNameContains,omitempty"`
}

// Sort options
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
)

// Connection types for pagination
type AttributeAssignmentConnection struct {
    Edges      []*AttributeAssignmentEdge `json:"edges"`
    PageInfo   *PageInfo                  `json:"pageInfo"`
    TotalCount int                        `json:"totalCount"`
}

type AttributeAssignmentEdge struct {
    Node   *ThingClassAttribute `json:"node"`
    Cursor string               `json:"cursor"`
}

// Impact analysis types
type AssignmentImpactAnalysis struct {
    Assignment           *ThingClassAttribute `json:"assignment"`
    ImpactType           ImpactType           `json:"impactType"`
    AffectedThingCount   int                  `json:"affectedThingCount"`
    RequiredThingCount   int                  `json:"requiredThingCount"`
    MissingValueCount    int                  `json:"missingValueCount"`
    ImpactDetails        json.RawMessage      `json:"impactDetails"`
    RecommendedActions   []string             `json:"recommendedActions"`
    MigrationRequired    bool                 `json:"migrationRequired"`
}

type ImpactType string

const (
    ImpactTypeSafe           ImpactType = "SAFE"
    ImpactTypeWarning        ImpactType = "WARNING"
    ImpactTypeBreaking       ImpactType = "BREAKING"
    ImpactTypeMigrationRequired ImpactType = "MIGRATION_REQUIRED"
)

// UI Hints structure
type UIHints struct {
    InputType    *string `json:"inputType,omitempty"`    // text, textarea, select, etc.
    Placeholder  *string `json:"placeholder,omitempty"`
    HelpText     *string `json:"helpText,omitempty"`
    Group        *string `json:"group,omitempty"`        // Grouping for UI layout
    Width        *string `json:"width,omitempty"`        // full, half, third, etc.
    Validation   *UIValidationHints `json:"validation,omitempty"`
}

type UIValidationHints struct {
    ShowValidationMessages bool     `json:"showValidationMessages"`
    ValidateOnBlur         bool     `json:"validateOnBlur"`
    ValidateOnChange       bool     `json:"validateOnChange"`
    CustomErrorMessages    map[string]string `json:"customErrorMessages,omitempty"`
}
```

## Business Logic Layer

### Assignment Validation Engine

```go
// Assignment validation service for comprehensive constraint checking
type AssignmentValidationService struct {
    attributeRepo      interfaces.AttributeRepository
    thingClassRepo     interfaces.ThingClassRepository
    thingRepo          interfaces.ThingRepository
    validationFactory  *validation.Factory
    logger             *zap.Logger
}

func (s *AssignmentValidationService) ValidateAssignmentCreate(ctx context.Context, tenant *domain.TenantContext, input *domain.CreateAttributeAssignmentInput) error {
    // Validate Thing Class exists and is active
    thingClass, err := s.thingClassRepo.GetByID(ctx, tenant, input.ThingClassID)
    if err != nil {
        return fmt.Errorf("failed to validate Thing Class: %w", err)
    }
    if thingClass == nil {
        return domain.NewValidationError("Thing Class not found", nil)
    }

    // Validate Attribute exists and is active
    attribute, err := s.attributeRepo.GetByID(ctx, tenant, input.AttributeID)
    if err != nil {
        return fmt.Errorf("failed to validate Attribute: %w", err)
    }
    if attribute == nil {
        return domain.NewValidationError("Attribute not found", nil)
    }

    // Validate assignment-specific validation rules
    if len(input.AssignmentValidationRules) > 0 {
        if err := s.validateAssignmentRules(attribute.DataType, attribute.ValidationRules, input.AssignmentValidationRules); err != nil {
            return fmt.Errorf("invalid assignment validation rules: %w", err)
        }
    }

    // Validate default value against attribute data type and rules
    if len(input.DefaultValue) > 0 {
        if err := s.validateDefaultValue(attribute, input.AssignmentValidationRules, input.DefaultValue); err != nil {
            return fmt.Errorf("invalid default value: %w", err)
        }
    }

    return nil
}

func (s *AssignmentValidationService) ValidateAssignmentUpdate(ctx context.Context, tenant *domain.TenantContext, existing *domain.ThingClassAttribute, input *domain.UpdateAttributeAssignmentInput) (*domain.AssignmentImpactAnalysis, error) {
    // Analyze impact of changes
    impactAnalysis, err := s.analyzeAssignmentImpact(ctx, tenant, existing, input)
    if err != nil {
        return nil, fmt.Errorf("failed to analyze assignment impact: %w", err)
    }

    // Block breaking changes if migration is required but not planned
    if impactAnalysis.ImpactType == domain.ImpactTypeBreaking && !impactAnalysis.MigrationRequired {
        return impactAnalysis, fmt.Errorf("breaking changes detected with existing data: %s", impactAnalysis.ImpactDetails)
    }

    return impactAnalysis, nil
}

func (s *AssignmentValidationService) analyzeAssignmentImpact(ctx context.Context, tenant *domain.TenantContext, existing *domain.ThingClassAttribute, input *domain.UpdateAttributeAssignmentInput) (*domain.AssignmentImpactAnalysis, error) {
    analysis := &domain.AssignmentImpactAnalysis{
        Assignment: existing,
        ImpactType: domain.ImpactTypeSafe,
        RecommendedActions: []string{},
    }

    // Count affected Things
    thingCount, err := s.thingRepo.GetCountByThingClass(ctx, tenant, existing.ThingClassID)
    if err != nil {
        return nil, fmt.Errorf("failed to count affected Things: %w", err)
    }
    analysis.AffectedThingCount = thingCount

    // Analyze specific changes
    if input.IsRequired != nil && *input.IsRequired != existing.IsRequired {
        if *input.IsRequired && !existing.IsRequired {
            // Making optional attribute required
            missingCount, err := s.countMissingValues(ctx, tenant, existing.ThingClassID, existing.AttributeID)
            if err != nil {
                return nil, fmt.Errorf("failed to count missing values: %w", err)
            }
            
            analysis.MissingValueCount = missingCount
            if missingCount > 0 {
                analysis.ImpactType = domain.ImpactTypeBreaking
                analysis.MigrationRequired = true
                analysis.RecommendedActions = append(analysis.RecommendedActions, 
                    fmt.Sprintf("Provide default values for %d Things missing this attribute", missingCount))
            } else {
                analysis.ImpactType = domain.ImpactTypeWarning
                analysis.RecommendedActions = append(analysis.RecommendedActions, 
                    "Monitor future Thing creation to ensure required attribute is provided")
            }
        } else {
            // Making required attribute optional - always safe
            analysis.ImpactType = domain.ImpactTypeSafe
            analysis.RecommendedActions = append(analysis.RecommendedActions, 
                "Relaxing requirement is safe - no migration needed")
        }
    }

    // Analyze validation rule changes
    if len(input.AssignmentValidationRules) > 0 {
        ruleImpact := s.analyzeValidationRuleChanges(existing.AssignmentValidationRules, input.AssignmentValidationRules)
        if ruleImpact.ImpactType > analysis.ImpactType {
            analysis.ImpactType = ruleImpact.ImpactType
        }
        analysis.RecommendedActions = append(analysis.RecommendedActions, ruleImpact.RecommendedActions...)
    }

    // Build impact details
    impactDetails := map[string]interface{}{
        "affectedThings": thingCount,
        "missingValues": analysis.MissingValueCount,
        "changeType": s.determineChangeType(existing, input),
        "timestamp": time.Now(),
    }
    
    detailsBytes, _ := json.Marshal(impactDetails)
    analysis.ImpactDetails = json.RawMessage(detailsBytes)

    return analysis, nil
}

func (s *AssignmentValidationService) countMissingValues(ctx context.Context, tenant *domain.TenantContext, thingClassID, attributeID uuid.UUID) (int, error) {
    // Count Things of this class that don't have a value for this attribute
    query := `
        SELECT COUNT(*)
        FROM things t
        WHERE t.thing_class_id = $1 
          AND t.is_active = TRUE
          AND NOT EXISTS (
              SELECT 1 FROM values v 
              WHERE v.thing_id = t.id 
                AND v.attribute_id = $2 
                AND v.is_active = TRUE
          )`
    
    var count int
    db, err := s.dbManager.GetConnection(tenant.SchemaName)
    if err != nil {
        return 0, err
    }
    
    err = db.QueryRowxContext(ctx, query, thingClassID, attributeID).Scan(&count)
    return count, err
}
```

## Performance Optimization

### Schema Composition Caching

```go
// Schema composition cache for optimized Thing Class schema building
type SchemaCompositionCache struct {
    redis  cache.Cache
    memory cache.Cache
    logger *zap.Logger
}

func (c *SchemaCompositionCache) GetThingClassSchema(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID) (*domain.ThingClassSchema, error) {
    cacheKey := fmt.Sprintf("thing_class_schema:%s", thingClassID.String())
    
    // Try L1 memory cache first
    var cached domain.ThingClassSchema
    if err := c.memory.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
        c.logger.Debug("Schema cache hit (memory)", zap.String("thing_class_id", thingClassID.String()))
        return &cached, nil
    }
    
    // Try L2 Redis cache
    if err := c.redis.Get(ctx, tenant.SchemaName, cacheKey, &cached); err == nil {
        c.logger.Debug("Schema cache hit (redis)", zap.String("thing_class_id", thingClassID.String()))
        // Store in memory cache for faster access
        c.memory.Set(ctx, tenant.SchemaName, cacheKey, &cached)
        return &cached, nil
    }
    
    return nil, cache.ErrCacheMiss
}

func (c *SchemaCompositionCache) SetThingClassSchema(ctx context.Context, tenant *domain.TenantContext, schema *domain.ThingClassSchema) {
    cacheKey := fmt.Sprintf("thing_class_schema:%s", schema.ThingClass.ID.String())
    
    // Store in both cache layers with different TTLs
    c.memory.SetWithTTL(ctx, tenant.SchemaName, cacheKey, schema, 5*time.Minute)
    c.redis.SetWithTTL(ctx, tenant.SchemaName, cacheKey, schema, 30*time.Minute)
    
    c.logger.Debug("Cached Thing Class schema", 
        zap.String("thing_class_id", schema.ThingClass.ID.String()),
        zap.Int("assignment_count", len(schema.Assignments)),
    )
}

func (c *SchemaCompositionCache) InvalidateThingClassSchema(ctx context.Context, tenant *domain.TenantContext, thingClassID uuid.UUID) {
    cacheKey := fmt.Sprintf("thing_class_schema:%s", thingClassID.String())
    
    c.memory.Delete(ctx, tenant.SchemaName, cacheKey)
    c.redis.Delete(ctx, tenant.SchemaName, cacheKey)
    
    // Also invalidate related patterns
    c.redis.InvalidatePattern(ctx, tenant.SchemaName, "assignment_list:*")
    c.memory.InvalidatePattern(ctx, tenant.SchemaName, "assignment_list:*")
    
    c.logger.Debug("Invalidated Thing Class schema cache", 
        zap.String("thing_class_id", thingClassID.String()))
}
```

### Database Query Optimization

```sql
-- Optimized query for Thing Class schema composition
PREPARE get_thing_class_schema(UUID) AS
SELECT 
    tc.id as thing_class_id,
    tc.name as thing_class_name,
    tc.description as thing_class_description,
    tc.created_at as thing_class_created_at,
    tc.updated_at as thing_class_updated_at,
    tc.version as thing_class_version,
    
    tca.id as assignment_id,
    tca.attribute_id,
    tca.is_required,
    tca.sort_order,
    tca.display_name,
    tca.assignment_validation_rules,
    tca.default_value,
    tca.ui_hints,
    tca.description as assignment_description,
    tca.assignment_type,
    tca.visibility,
    tca.created_at as assignment_created_at,
    tca.updated_at as assignment_updated_at,
    tca.version as assignment_version,
    
    a.id as attribute_id,
    a.name as attribute_name,
    a.data_type as attribute_data_type,
    a.validation_rules as attribute_validation_rules,
    a.description as attribute_description
    
FROM thing_classes tc
LEFT JOIN thing_class_attributes tca ON tc.id = tca.thing_class_id AND tca.is_active = TRUE
LEFT JOIN attributes a ON tca.attribute_id = a.id AND a.is_active = TRUE
WHERE tc.id = $1 AND tc.is_active = TRUE
ORDER BY tca.sort_order ASC NULLS LAST, tca.created_at ASC;

-- Optimized query for assignment listing with filters
PREPARE list_assignments_filtered(UUID, UUID, BOOLEAN, VARCHAR, INT, INT) AS
SELECT 
    tca.id,
    tca.thing_class_id,
    tca.attribute_id,
    tca.is_required,
    tca.sort_order,
    tca.display_name,
    tca.assignment_validation_rules,
    tca.default_value,
    tca.ui_hints,
    tca.description,
    tca.assignment_type,
    tca.visibility,
    tca.created_at,
    tca.updated_at,
    tca.created_by,
    tca.is_active,
    tca.version,
    COUNT(*) OVER() as total_count
FROM thing_class_attributes tca
WHERE ($1 IS NULL OR tca.thing_class_id = $1)
  AND ($2 IS NULL OR tca.attribute_id = $2)
  AND ($3 IS NULL OR tca.is_required = $3)
  AND ($4 IS NULL OR tca.assignment_type = $4)
  AND tca.is_active = TRUE
ORDER BY tca.sort_order ASC, tca.created_at ASC
LIMIT $5 OFFSET $6;

-- Query for assignment impact analysis
PREPARE analyze_assignment_impact(UUID, UUID) AS
SELECT 
    COUNT(DISTINCT t.id) as affected_thing_count,
    COUNT(DISTINCT CASE WHEN v.id IS NULL THEN t.id END) as missing_value_count,
    COUNT(DISTINCT v.id) as existing_value_count
FROM things t
LEFT JOIN values v ON t.id = v.thing_id AND v.attribute_id = $2 AND v.is_active = TRUE
WHERE t.thing_class_id = $1 AND t.is_active = TRUE;
```

This comprehensive technical specification provides detailed implementation guidance for the Attribute Assignment entity, covering all aspects from data modeling to performance optimization and business logic implementation.