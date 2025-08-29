# Attribute Entity Spec - Condensed Summary

**Created:** 2025-08-29  
**Status:** Ready for Implementation  
**Entity:** Attribute - Property definitions for UDM system  

## Core Purpose
Reusable property templates that define data types, validation rules, and constraints for Thing Class assignments. Enables dynamic attribute management across all business objects in the UDM system.

## Key Features
- **Data Type Support**: string, integer, decimal, boolean, date, datetime, json, reference
- **Validation Rules**: JSON-based constraint definitions for each attribute
- **GraphQL CRUD**: Full create, read, update, delete operations via GraphQL API
- **Assignment Tracking**: Monitor which Thing Classes use specific attributes
- **Schema Evolution**: Safe modification with backward compatibility checking

## GraphQL Operations
```graphql
# Core mutations
createAttribute(input: CreateAttributeInput!): AttributePayload!
updateAttribute(id: ID!, input: UpdateAttributeInput!): AttributePayload!

# Core queries
attribute(id: ID!): Attribute
attributes(filter: AttributeFilter, first: Int, after: String): AttributeConnection!
```

## Database Schema
```sql
CREATE TABLE attributes (
  id UUID PRIMARY KEY,
  name VARCHAR(255) UNIQUE NOT NULL,
  data_type VARCHAR(50) NOT NULL,
  validation_rules JSONB DEFAULT '{}',
  description TEXT,
  -- Standard entity fields
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1
);
```

## Domain Model
```go
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
```

## Multi-Tenant Architecture
- **Schema Isolation**: Each tenant operates within dedicated PostgreSQL schema
- **Connection Routing**: JWT token determines tenant schema for all operations
- **Data Validation**: Tenant-scoped attribute name uniqueness and validation
- **Security**: Complete tenant separation with schema-level access control

## Performance Targets
- **Single Attribute Query**: <200ms P95 latency via GraphQL field selection
- **Attribute Listing**: <500ms for paginated results with filtering
- **Creation/Updates**: <300ms for validation and persistence
- **Schema Evolution**: <100ms for attribute definition changes

## Implementation Dependencies
- **Prerequisites**: Thing Class entity spec completed for attribute assignments
- **Database**: PostgreSQL 15+ with JSON-B support for validation rules
- **Framework**: Go + gqlgen for GraphQL server implementation
- **Caching**: Redis for attribute definition and validation rule caching

## Success Criteria
1. **Functional**: Complete CRUD operations through GraphQL with data type validation
2. **Performance**: All operations meet sub-500ms response time targets
3. **Integration**: Seamless attribute assignment to Thing Classes
4. **Security**: Full tenant isolation with proper access controls
5. **Evolution**: Safe attribute modification without breaking existing assignments

---
*This condensed spec serves as implementation reference. Full details in main spec document and sub-specifications.*