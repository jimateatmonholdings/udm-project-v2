# Attribute Entity - GraphQL API Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**API Type:** GraphQL-First with Multi-Tenant Architecture  
**Entity:** Attribute - Property definitions for UDM system  

## API Overview

The Attribute GraphQL API provides comprehensive CRUD operations for property definition management within the Universal Data Model system. Built on a schema-per-tenant architecture, the API enables data architects and developers to create, manage, and validate attribute definitions that serve as reusable property templates across all Thing Classes.

## GraphQL Schema Definition

### Core Types

```graphql
# Scalar types
scalar DateTime
scalar JSON
scalar UUID

# Attribute data type enumeration
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

# Main Attribute entity
type Attribute implements Node {
  # Core identification
  id: ID!
  
  # Attribute properties
  name: String!
  dataType: AttributeType!
  validationRules: JSON
  description: String
  
  # Computed relationships and metadata
  assignmentCount: Int!
  assignments(
    first: Int = 20
    after: String
    filter: ThingClassAttributeFilter
  ): ThingClassAttributeConnection!
  
  # Standard entity metadata
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  updatedBy: ID
  isActive: Boolean!
  version: Int!
  
  # Schema evolution information
  evolutionHistory(
    first: Int = 10
    after: String
  ): AttributeEvolutionConnection!
}

# Connection types for pagination
type AttributeConnection {
  edges: [AttributeEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type AttributeEdge {
  node: Attribute!
  cursor: String!
}

# Schema evolution tracking
type AttributeEvolution implements Node {
  id: ID!
  attributeId: ID!
  attribute: Attribute!
  changeType: AttributeChangeType!
  oldValues: JSON!
  newValues: JSON!
  impactAnalysis: JSON
  safetyLevel: EvolutionSafetyLevel!
  appliedAt: DateTime!
  appliedBy: ID!
  rollbackAvailable: Boolean!
  rollbackData: JSON
}

enum AttributeChangeType {
  VALIDATION_RULES_RELAXED
  VALIDATION_RULES_TIGHTENED
  DESCRIPTION_CHANGED
  NAME_CHANGED
  STATUS_CHANGED
}

enum EvolutionSafetyLevel {
  SAFE
  WARNING
  BREAKING
}

type AttributeEvolutionConnection {
  edges: [AttributeEvolutionEdge!]!
  pageInfo: PageInfo!
}

type AttributeEvolutionEdge {
  node: AttributeEvolution!
  cursor: String!
}
```

### Input Types

```graphql
# Attribute creation input
input CreateAttributeInput {
  name: String! @validate(constraint: "required,min=1,max=255,alphanum_underscore")
  dataType: AttributeType!
  validationRules: JSON
  description: String @validate(constraint: "max=1000")
}

# Attribute update input
input UpdateAttributeInput {
  name: String @validate(constraint: "min=1,max=255,alphanum_underscore")
  validationRules: JSON
  description: String @validate(constraint: "max=1000")
  # Note: dataType cannot be updated to maintain data integrity
}

# Filtering options for attribute queries
input AttributeFilter {
  name: String
  dataType: AttributeType
  isActive: Boolean
  createdBy: ID
  createdAfter: DateTime
  createdBefore: DateTime
  hasAssignments: Boolean
  validationRulesContains: JSON
  nameContains: String
}

# Sorting options
input AttributeSort {
  field: AttributeSortField!
  direction: SortDirection!
}

enum AttributeSortField {
  NAME
  DATA_TYPE
  CREATED_AT
  UPDATED_AT
  ASSIGNMENT_COUNT
}

enum SortDirection {
  ASC
  DESC
}
```

### Payload Types

```graphql
# Mutation response types with error handling
type AttributePayload {
  attribute: Attribute
  errors: [UserError!]!
  clientMutationId: String
}

type DeleteAttributePayload {
  success: Boolean!
  deletedAttributeId: ID
  errors: [UserError!]!
  clientMutationId: String
}

type BulkAttributePayload {
  attributes: [Attribute!]!
  errors: [UserError!]!
  successCount: Int!
  failureCount: Int!
  clientMutationId: String
}

# Impact analysis response
type AttributeImpactAnalysis {
  attribute: Attribute!
  safetyLevel: EvolutionSafetyLevel!
  affectedAssignments: Int!
  affectedThings: Int!
  impactDetails: JSON!
  recommendedActions: [String!]!
}

# Validation rule templates by data type
type ValidationRuleTemplate {
  dataType: AttributeType!
  schema: JSON!
  examples: [JSON!]!
  description: String!
}

# Error types
type UserError {
  field: String
  message: String!
  code: String
  details: JSON
}
```

## Query Operations

### Root Query Type

```graphql
type Query {
  # Single attribute retrieval
  attribute(id: ID!): Attribute @auth(requires: ["READ_ATTRIBUTE"])
  
  # Attribute collection queries
  attributes(
    filter: AttributeFilter
    sort: [AttributeSort!]
    first: Int = 20
    after: String
  ): AttributeConnection! @auth(requires: ["READ_ATTRIBUTE"])
  
  # Search attributes by name or description
  searchAttributes(
    query: String!
    dataType: AttributeType
    first: Int = 20
    after: String
  ): AttributeConnection! @auth(requires: ["READ_ATTRIBUTE"])
  
  # Validation rule templates and schemas
  attributeValidationTemplate(
    dataType: AttributeType!
  ): ValidationRuleTemplate @auth(requires: ["READ_ATTRIBUTE"])
  
  attributeValidationTemplates: [ValidationRuleTemplate!]! 
    @auth(requires: ["READ_ATTRIBUTE"])
  
  # Impact analysis for proposed changes
  analyzeAttributeImpact(
    attributeId: ID!
    proposedChanges: UpdateAttributeInput!
  ): AttributeImpactAnalysis! @auth(requires: ["UPDATE_ATTRIBUTE"])
  
  # Schema evolution history
  attributeEvolutionHistory(
    attributeId: ID!
    first: Int = 10
    after: String
  ): AttributeEvolutionConnection! @auth(requires: ["READ_ATTRIBUTE"])
  
  # Statistics and reporting
  attributeStatistics(
    filter: AttributeFilter
  ): AttributeStatistics! @auth(requires: ["READ_ATTRIBUTE"])
}

# Statistics and reporting type
type AttributeStatistics {
  totalCount: Int!
  activeCount: Int!
  countByDataType: [DataTypeCount!]!
  averageAssignmentsPerAttribute: Float!
  mostUsedAttributes: [Attribute!]!
  leastUsedAttributes: [Attribute!]!
  attributesWithoutAssignments: Int!
  recentlyCreated: [Attribute!]!
}

type DataTypeCount {
  dataType: AttributeType!
  count: Int!
  percentage: Float!
}
```

## Mutation Operations

### Root Mutation Type

```graphql
type Mutation {
  # Core CRUD operations
  createAttribute(
    input: CreateAttributeInput!
    clientMutationId: String
  ): AttributePayload! @auth(requires: ["CREATE_ATTRIBUTE"])
  
  updateAttribute(
    id: ID!
    input: UpdateAttributeInput!
    clientMutationId: String
  ): AttributePayload! @auth(requires: ["UPDATE_ATTRIBUTE"])
  
  deleteAttribute(
    id: ID!
    clientMutationId: String
  ): DeleteAttributePayload! @auth(requires: ["DELETE_ATTRIBUTE"])
  
  # Bulk operations for efficiency
  createAttributesBulk(
    inputs: [CreateAttributeInput!]!
    clientMutationId: String
  ): BulkAttributePayload! @auth(requires: ["CREATE_ATTRIBUTE"])
  
  # Attribute lifecycle management
  activateAttribute(
    id: ID!
    clientMutationId: String
  ): AttributePayload! @auth(requires: ["UPDATE_ATTRIBUTE"])
  
  deactivateAttribute(
    id: ID!
    clientMutationId: String
  ): AttributePayload! @auth(requires: ["UPDATE_ATTRIBUTE"])
  
  # Advanced operations
  cloneAttribute(
    sourceId: ID!
    newName: String!
    clientMutationId: String
  ): AttributePayload! @auth(requires: ["CREATE_ATTRIBUTE"])
  
  # Validation rule management
  validateAttributeRules(
    dataType: AttributeType!
    validationRules: JSON!
  ): ValidationResult! @auth(requires: ["READ_ATTRIBUTE"])
}

type ValidationResult {
  isValid: Boolean!
  errors: [ValidationError!]!
  warnings: [ValidationWarning!]!
}

type ValidationError {
  field: String
  message: String!
  code: String!
}

type ValidationWarning {
  field: String
  message: String!
  recommendation: String
}
```

## Subscription Operations

### Real-time Updates

```graphql
type Subscription {
  # Attribute lifecycle events
  attributeUpdated(
    attributeId: ID
    dataType: AttributeType
  ): AttributeUpdateEvent! @auth(requires: ["READ_ATTRIBUTE"])
  
  attributeCreated(
    dataType: AttributeType
  ): AttributeCreateEvent! @auth(requires: ["READ_ATTRIBUTE"])
  
  attributeDeleted: AttributeDeleteEvent! @auth(requires: ["READ_ATTRIBUTE"])
  
  # Schema evolution events
  attributeSchemaEvolution(
    attributeId: ID
    safetyLevel: EvolutionSafetyLevel
  ): AttributeEvolutionEvent! @auth(requires: ["READ_ATTRIBUTE"])
}

# Event types for subscriptions
type AttributeUpdateEvent {
  attribute: Attribute!
  previousVersion: Attribute
  changeType: AttributeChangeType!
  timestamp: DateTime!
  updatedBy: ID!
}

type AttributeCreateEvent {
  attribute: Attribute!
  timestamp: DateTime!
  createdBy: ID!
}

type AttributeDeleteEvent {
  attributeId: ID!
  attributeName: String!
  timestamp: DateTime!
  deletedBy: ID!
}

type AttributeEvolutionEvent {
  evolution: AttributeEvolution!
  attribute: Attribute!
  timestamp: DateTime!
}
```

## Directive Implementations

### Authentication and Authorization

```graphql
# Authentication directive for role-based access control
directive @auth(requires: [String!]!) on FIELD_DEFINITION

# Field-level validation directive
directive @validate(constraint: String!) on INPUT_FIELD_DEFINITION

# Rate limiting directive
directive @rateLimit(max: Int!, window: String!) on FIELD_DEFINITION

# Caching directive for performance optimization
directive @cached(ttl: Int, scope: CacheScope) on FIELD_DEFINITION

enum CacheScope {
  PUBLIC
  PRIVATE
  TENANT
}
```

## Example Queries and Mutations

### Query Examples

```graphql
# 1. Get a single attribute with assignments
query GetAttributeWithAssignments($id: ID!) {
  attribute(id: $id) {
    id
    name
    dataType
    validationRules
    description
    assignmentCount
    assignments(first: 10) {
      edges {
        node {
          id
          thingClass {
            id
            name
          }
          isRequired
          sortOrder
        }
      }
      pageInfo {
        hasNextPage
        endCursor
      }
    }
    createdAt
    updatedAt
    version
  }
}

# 2. List attributes with filtering and pagination
query ListStringAttributes($first: Int!, $after: String) {
  attributes(
    filter: {
      dataType: STRING
      isActive: true
    }
    sort: [{field: NAME, direction: ASC}]
    first: $first
    after: $after
  ) {
    edges {
      node {
        id
        name
        validationRules
        description
        assignmentCount
      }
      cursor
    }
    pageInfo {
      hasNextPage
      hasPreviousPage
      startCursor
      endCursor
    }
    totalCount
  }
}

# 3. Search attributes by name
query SearchAttributes($query: String!, $first: Int!) {
  searchAttributes(query: $query, first: $first) {
    edges {
      node {
        id
        name
        dataType
        description
        assignmentCount
      }
    }
    totalCount
  }
}

# 4. Get validation rule template
query GetValidationTemplate($dataType: AttributeType!) {
  attributeValidationTemplate(dataType: $dataType) {
    dataType
    schema
    examples
    description
  }
}

# 5. Analyze impact of proposed changes
query AnalyzeAttributeImpact($attributeId: ID!, $changes: UpdateAttributeInput!) {
  analyzeAttributeImpact(attributeId: $attributeId, proposedChanges: $changes) {
    attribute {
      id
      name
      dataType
    }
    safetyLevel
    affectedAssignments
    affectedThings
    impactDetails
    recommendedActions
  }
}

# 6. Get attribute statistics
query GetAttributeStatistics {
  attributeStatistics {
    totalCount
    activeCount
    countByDataType {
      dataType
      count
      percentage
    }
    averageAssignmentsPerAttribute
    mostUsedAttributes {
      id
      name
      assignmentCount
    }
    attributesWithoutAssignments
  }
}
```

### Mutation Examples

```graphql
# 1. Create a string attribute with validation rules
mutation CreateEmailAttribute {
  createAttribute(
    input: {
      name: "email_address"
      dataType: STRING
      validationRules: {
        required: true
        format: "email"
        maxLength: 255
      }
      description: "User email address with validation"
    }
    clientMutationId: "create-email-attr-001"
  ) {
    attribute {
      id
      name
      dataType
      validationRules
      description
      version
    }
    errors {
      field
      message
      code
    }
    clientMutationId
  }
}

# 2. Update attribute validation rules
mutation UpdateAttributeRules($id: ID!, $rules: JSON!) {
  updateAttribute(
    id: $id
    input: {
      validationRules: $rules
    }
    clientMutationId: "update-rules-001"
  ) {
    attribute {
      id
      name
      validationRules
      version
      updatedAt
    }
    errors {
      field
      message
      code
      details
    }
  }
}

# 3. Bulk create attributes
mutation CreateMultipleAttributes($inputs: [CreateAttributeInput!]!) {
  createAttributesBulk(
    inputs: $inputs
    clientMutationId: "bulk-create-001"
  ) {
    attributes {
      id
      name
      dataType
    }
    successCount
    failureCount
    errors {
      field
      message
      code
    }
  }
}

# 4. Clone an existing attribute
mutation CloneAttribute($sourceId: ID!, $newName: String!) {
  cloneAttribute(
    sourceId: $sourceId
    newName: $newName
    clientMutationId: "clone-attr-001"
  ) {
    attribute {
      id
      name
      dataType
      validationRules
      description
    }
    errors {
      field
      message
      code
    }
  }
}

# 5. Soft delete an attribute
mutation DeleteAttribute($id: ID!) {
  deleteAttribute(
    id: $id
    clientMutationId: "delete-attr-001"
  ) {
    success
    deletedAttributeId
    errors {
      message
      code
      details
    }
  }
}

# 6. Validate attribute rules before creation
mutation ValidateRules($dataType: AttributeType!, $rules: JSON!) {
  validateAttributeRules(
    dataType: $dataType
    validationRules: $rules
  ) {
    isValid
    errors {
      field
      message
      code
    }
    warnings {
      field
      message
      recommendation
    }
  }
}
```

### Subscription Examples

```graphql
# 1. Listen for attribute updates
subscription AttributeUpdates($attributeId: ID) {
  attributeUpdated(attributeId: $attributeId) {
    attribute {
      id
      name
      version
      updatedAt
    }
    previousVersion {
      version
      validationRules
    }
    changeType
    timestamp
    updatedBy
  }
}

# 2. Listen for new string attributes
subscription NewStringAttributes {
  attributeCreated(dataType: STRING) {
    attribute {
      id
      name
      validationRules
      description
    }
    timestamp
    createdBy
  }
}

# 3. Monitor schema evolution events
subscription SchemaEvolutionMonitor($safetyLevel: EvolutionSafetyLevel) {
  attributeSchemaEvolution(safetyLevel: $safetyLevel) {
    evolution {
      id
      changeType
      safetyLevel
      impactAnalysis
    }
    attribute {
      id
      name
      dataType
    }
    timestamp
  }
}
```

## Error Handling

### GraphQL Error Extensions

```json
{
  "errors": [
    {
      "message": "Attribute name already exists",
      "extensions": {
        "code": "ATTRIBUTE_NAME_EXISTS",
        "field": "name",
        "details": {
          "name": "email_address",
          "existingAttributeId": "123e4567-e89b-12d3-a456-426614174000"
        },
        "tenantSchema": "tenant_company_a",
        "requestId": "req_8c2d4f6e9a1b"
      },
      "path": ["createAttribute"],
      "locations": [{"line": 2, "column": 3}]
    }
  ]
}
```

### Error Code Catalog

```graphql
enum AttributeErrorCode {
  # Validation errors
  ATTRIBUTE_NOT_FOUND
  ATTRIBUTE_NAME_EXISTS
  INVALID_DATA_TYPE
  INVALID_VALIDATION_RULES
  INVALID_NAME_FORMAT
  DESCRIPTION_TOO_LONG
  
  # Business logic errors
  SCHEMA_EVOLUTION_BLOCKED
  ATTRIBUTE_IN_USE
  CANNOT_CHANGE_DATA_TYPE
  VALIDATION_RULES_TOO_RESTRICTIVE
  
  # System errors
  TENANT_ISOLATION_ERROR
  PERMISSION_DENIED
  RATE_LIMITED
  COMPLEXITY_TOO_HIGH
  
  # Data integrity errors
  CONCURRENT_MODIFICATION
  INVALID_VERSION
  CONSTRAINT_VIOLATION
}
```

## Performance Characteristics

### Query Complexity Analysis

```graphql
# Query complexity calculation rules
# - Basic field: 1 point
# - Connection field: multiplier based on first/last arguments
# - Nested object: additive complexity
# - Maximum allowed complexity: 100 points

query ComplexAttributeQuery {
  # Complexity: 1 (base) + 5 (attributes connection) + 3*5 (nested assignments) = 21 points
  attributes(first: 5) {
    edges {
      node {
        id            # +1
        name          # +1
        assignments(first: 3) {  # +3*3 = 9
          edges {
            node {
              id      # +1 per edge
              thingClass {
                name  # +1 per edge
              }
            }
          }
        }
      }
    }
  }
}
```

### Caching Strategy

```graphql
type Query {
  # Cached for 5 minutes, tenant-scoped
  attribute(id: ID!): Attribute 
    @cached(ttl: 300, scope: TENANT)
  
  # Cached for 2 minutes due to dynamic nature
  attributes(filter: AttributeFilter, first: Int, after: String): AttributeConnection!
    @cached(ttl: 120, scope: TENANT)
  
  # Cached for 1 hour, rarely changes
  attributeValidationTemplates: [ValidationRuleTemplate!]!
    @cached(ttl: 3600, scope: PUBLIC)
}
```

### Performance Targets

- **Single attribute query**: <200ms P95 latency
- **Attribute list query (20 items)**: <500ms P95 latency  
- **Search query**: <800ms P95 latency
- **Mutation operations**: <300ms P95 latency
- **Subscription events**: <100ms delivery latency
- **Maximum query complexity**: 100 points
- **Rate limits**: 2000 queries/minute, 500 mutations/minute per tenant

## Integration Patterns

### Client SDK Generation

The GraphQL schema supports automatic client SDK generation for:

- **TypeScript/JavaScript**: Full type safety with generated hooks
- **Go**: Strongly typed client with automatic serialization
- **Python**: Type-hinted client with validation support
- **Java**: Generated classes with builder patterns

### Federation Support

```graphql
# Attribute entity can be extended by other services
extend type Attribute @key(fields: "id") {
  id: ID! @external
  
  # Extended by Thing Class service
  thingClassUsage: [ThingClassUsage!]!
  
  # Extended by Analytics service  
  usageStatistics: AttributeUsageStats!
}
```

This comprehensive GraphQL API specification provides a robust, performant, and secure interface for managing Attribute entities within the UDM system, supporting all CRUD operations, advanced querying, real-time updates, and comprehensive error handling.