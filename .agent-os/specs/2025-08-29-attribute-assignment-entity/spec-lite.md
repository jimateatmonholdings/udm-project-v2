# Attribute Assignment Entity Spec - Condensed Summary

**Created:** 2025-08-29  
**Status:** Ready for Implementation  
**Entity:** Attribute Assignment - Bridge connecting Thing Classes to Attributes with constraints  

## Core Purpose
Configuration layer that defines which Attributes can be assigned to which Thing Classes, along with their requirements, validation rules, ordering, and metadata. Enables dynamic schema composition and flexible business entity definition in the UDM system.

## Key Features
- **Assignment Management**: Link Thing Classes to Attributes with required/optional flags and constraints
- **Constraint Definition**: Default values, validation rule overrides, and assignment-specific metadata
- **Schema Composition**: Build coherent Thing Class schemas through managed attribute assignments
- **Impact Analysis**: Assess changes against existing Thing instances with migration recommendations
- **Ordering & Display**: Sort order, display names, and UI hints for attribute presentation

## GraphQL Operations
```graphql
# Core mutations
createAttributeAssignment(input: CreateAttributeAssignmentInput!): AttributeAssignmentPayload!
updateAttributeAssignment(id: ID!, input: UpdateAttributeAssignmentInput!): AttributeAssignmentPayload!

# Core queries
attributeAssignment(id: ID!): AttributeAssignment
attributeAssignments(filter: AttributeAssignmentFilter, first: Int, after: String): AttributeAssignmentConnection!
thingClassSchema(thingClassId: ID!): ThingClassSchema!
```

## Database Schema
```sql
CREATE TABLE thing_class_attributes (
  id UUID PRIMARY KEY,
  thing_class_id UUID NOT NULL,
  attribute_id UUID NOT NULL,
  is_required BOOLEAN DEFAULT FALSE,
  sort_order INTEGER DEFAULT 0,
  display_name VARCHAR(255),
  assignment_validation_rules JSONB DEFAULT '{}',
  default_value JSONB,
  ui_hints JSONB DEFAULT '{}',
  -- Standard entity fields
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1,
  -- Unique constraint for assignment
  UNIQUE(thing_class_id, attribute_id)
);
```

## Domain Model
```go
type ThingClassAttribute struct {
    BaseEntity
    ThingClassID              uuid.UUID       `json:"thingClassId" db:"thing_class_id"`
    AttributeID               uuid.UUID       `json:"attributeId" db:"attribute_id"`
    IsRequired                bool            `json:"isRequired" db:"is_required"`
    SortOrder                 int             `json:"sortOrder" db:"sort_order"`
    DisplayName               *string         `json:"displayName" db:"display_name"`
    AssignmentValidationRules json.RawMessage `json:"assignmentValidationRules" db:"assignment_validation_rules"`
    DefaultValue              json.RawMessage `json:"defaultValue" db:"default_value"`
    UIHints                   json.RawMessage `json:"uiHints" db:"ui_hints"`
    
    // Loaded relationships
    ThingClass *ThingClass `json:"thingClass,omitempty" db:"-"`
    Attribute  *Attribute  `json:"attribute,omitempty" db:"-"`
}

type ThingClassSchema struct {
    ThingClass   *ThingClass            `json:"thingClass"`
    Assignments  []*ThingClassAttribute `json:"assignments"`
    TotalCount   int                    `json:"totalCount"`
    RequiredCount int                   `json:"requiredCount"`
    OptionalCount int                   `json:"optionalCount"`
}
```

## Multi-Tenant Architecture
- **Schema Isolation**: Each tenant operates within dedicated PostgreSQL schema with isolated assignments
- **Referential Integrity**: Foreign key constraints ensure assignment validity within tenant boundaries
- **Cross-Reference Validation**: Prevent assignments between Thing Classes/Attributes across tenants
- **Security**: Complete tenant separation with schema-level access control and audit trails

## Performance Targets
- **Single Assignment Query**: <200ms P95 latency via GraphQL field selection
- **Assignment Listing**: <500ms for paginated results with Thing Class/Attribute hydration
- **Schema Composition**: <300ms to build complete Thing Class schema with all assignments
- **Impact Analysis**: <1000ms for change impact assessment across existing Thing instances

## Implementation Dependencies
- **Prerequisites**: Thing Class and Attribute entity specs completed
- **Database**: PostgreSQL 15+ with JSON-B support for assignment metadata and validation rules
- **Framework**: Go + gqlgen for GraphQL server implementation with relationship loading
- **Caching**: Redis for Thing Class schema composition and assignment metadata caching

## Business Rules
1. **Uniqueness**: One assignment per Thing Class + Attribute pair
2. **Referential Integrity**: Both Thing Class and Attribute must exist and be active
3. **Tenant Isolation**: Assignments cannot cross tenant boundaries
4. **Constraint Inheritance**: Assignment validation rules extend (not override) base Attribute validation
5. **Ordering**: Sort order must be unique within each Thing Class for consistent presentation

## Success Criteria
1. **Functional**: Complete assignment CRUD operations through GraphQL with constraint validation
2. **Performance**: All operations meet sub-500ms response time targets with relationship hydration
3. **Integration**: Seamless schema composition for Thing Classes with dynamic attribute loading
4. **Security**: Full tenant isolation with proper access controls and audit logging
5. **Flexibility**: Support for assignment-level constraints, defaults, and UI configuration metadata

---
*This condensed spec serves as implementation reference. Full details in main spec document and sub-specifications.*