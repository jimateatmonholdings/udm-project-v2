# Relationship Class - GraphQL API Specification

**Created:** 2025-08-30
**Version:** 1.0.0
**API Type:** GraphQL with Multi-Tenant Support
**Entity:** Relationship Class - Relationship type templates

## API Architecture Overview

The Relationship Class GraphQL API provides comprehensive operations for managing relationship type templates in the Universal Data Model system. Following established patterns from Thing Class and other entities, it uses JSON-based validation rules for flexible constraint management while maintaining sub-200ms performance targets and multi-tenant isolation.

## GraphQL Schema Definition

### Core Types

```graphql
"""
Relationship Class represents a template for relationship types between Things
"""
type RelationshipClass {
  id: ID!
  tenantId: ID!
  name: String!
  description: String
  displayName: String
  
  # Relationship behavior configuration  
  isDirectional: Boolean!
  isBidirectional: Boolean!
  cardinality: RelationshipCardinality!
  
  # Labels for UI and traversal
  sourceLabel: String
  targetLabel: String  
  inverseLabel: String
  
  # JSON-based validation rules (following established pattern)
  validationRules: JSON!
  
  # Display configuration
  color: String
  icon: String
  sortOrder: Int!
  
  # Standard metadata
  isActive: Boolean!
  createdAt: DateTime!
  updatedAt: DateTime!
  deletedAt: DateTime
  createdBy: ID!
  updatedBy: ID
  version: Int!
  
  # Computed fields
  activeRelationshipCount: Int!
  
  # Related data (loaded via DataLoader)
  attributeAssignments: [AttributeAssignment!]!
  relationships: [Relationship!]!
}

"""
Cardinality options for relationships
"""
enum RelationshipCardinality {
  ONE_TO_ONE
  ONE_TO_MANY
  MANY_TO_ONE
  MANY_TO_MANY
}
```

### Input Types

```graphql
"""
Input for creating a new Relationship Class
"""
input CreateRelationshipClassInput {
  name: String!
  description: String
  displayName: String
  isDirectional: Boolean! = true
  isBidirectional: Boolean! = false
  cardinality: RelationshipCardinality!
  sourceLabel: String
  targetLabel: String
  inverseLabel: String
  validationRules: JSON = "{}"
  color: String
  icon: String
  sortOrder: Int = 0
}

"""
Input for updating a Relationship Class
"""
input UpdateRelationshipClassInput {
  name: String
  description: String
  displayName: String
  isDirectional: Boolean
  isBidirectional: Boolean
  cardinality: RelationshipCardinality
  sourceLabel: String
  targetLabel: String
  inverseLabel: String
  validationRules: JSON
  color: String
  icon: String
  sortOrder: Int
}

"""
Filter options for Relationship Class queries
"""
input RelationshipClassFilter {
  name: String
  nameContains: String
  isDirectional: Boolean
  isBidirectional: Boolean
  cardinality: RelationshipCardinality
  isActive: Boolean
  createdAfter: DateTime
  createdBefore: DateTime
  createdBy: ID
}

"""
Ordering options for Relationship Class queries
"""
input RelationshipClassOrderBy {
  field: RelationshipClassOrderField!
  direction: OrderDirection!
}

enum RelationshipClassOrderField {
  NAME
  CREATED_AT
  UPDATED_AT
  SORT_ORDER
}

enum OrderDirection {
  ASC
  DESC
}
```

### Query Operations

```graphql
type Query {
  """Get a Relationship Class by ID"""
  relationshipClass(id: ID!): RelationshipClass
  
  """List Relationship Classes with filtering and pagination"""
  relationshipClasses(
    filter: RelationshipClassFilter
    orderBy: [RelationshipClassOrderBy!] = [{ field: SORT_ORDER, direction: ASC }]
    limit: Int = 50
    offset: Int = 0
  ): RelationshipClassConnection!
  
  """Search Relationship Classes by name"""
  searchRelationshipClasses(
    query: String!
    limit: Int = 20
  ): [RelationshipClass!]!
}

"""
Connection type for paginated results
"""
type RelationshipClassConnection {
  nodes: [RelationshipClass!]!
  totalCount: Int!
  pageInfo: PageInfo!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### Mutation Operations

```graphql
type Mutation {
  """Create a new Relationship Class"""
  createRelationshipClass(
    input: CreateRelationshipClassInput!
  ): CreateRelationshipClassPayload!
  
  """Update a Relationship Class"""
  updateRelationshipClass(
    id: ID!
    input: UpdateRelationshipClassInput!
  ): UpdateRelationshipClassPayload!
  
  """Soft delete a Relationship Class"""
  deleteRelationshipClass(
    id: ID!
    reason: String
  ): DeleteRelationshipClassPayload!
  
  """Restore a soft deleted Relationship Class"""
  restoreRelationshipClass(
    id: ID!
  ): RestoreRelationshipClassPayload!
}

"""Payload types for mutations"""
type CreateRelationshipClassPayload {
  relationshipClass: RelationshipClass
  errors: [UserError!]!
}

type UpdateRelationshipClassPayload {
  relationshipClass: RelationshipClass
  errors: [UserError!]!
}

type DeleteRelationshipClassPayload {
  deletedRelationshipClassId: ID
  errors: [UserError!]!
}

type RestoreRelationshipClassPayload {
  relationshipClass: RelationshipClass
  errors: [UserError!]!
}

type UserError {
  field: String
  message: String!
  code: String!
}
```

## Validation Rules Examples

The `validationRules` JSON field supports flexible constraint definitions:

```json
{
  "sourceConstraints": {
    "allowedThingClasses": ["Person", "Department"],
    "requiredAttributes": ["active_status"],
    "maxConnections": null
  },
  "targetConstraints": {
    "allowedThingClasses": ["Task", "Project"],
    "requiredAttributes": ["assignable"],
    "maxConnections": 10
  },
  "businessRules": {
    "preventCycles": true,
    "requireApproval": false,
    "allowSelfReference": false
  }
}
```

## Performance Optimizations

### DataLoader Implementation

```graphql
"""
DataLoader configuration for efficient related data loading
"""
type RelationshipClassDataLoaders {
  attributeAssignments: [AttributeAssignment!]!  # Batched by relationshipClassId
  relationships: [Relationship!]!                # Batched by relationshipClassId
  statistics: RelationshipClassStats!            # Cached per class
}
```

### Caching Strategy

- **Query Results**: Cache frequently accessed Relationship Class lists
- **Validation Rules**: Cache parsed validation rule objects
- **Statistics**: Cache computed fields with smart invalidation
- **Search Results**: Cache search results for common queries

## Error Handling

```graphql
"""
Error codes for Relationship Class operations
"""
enum RelationshipClassErrorCode {
  NOT_FOUND
  DUPLICATE_NAME
  INVALID_CARDINALITY
  INVALID_DIRECTIONAL_CONFIG
  VALIDATION_FAILED
  CONSTRAINT_VIOLATION
  INSUFFICIENT_PERMISSIONS
}
```

## Example Operations

### Create Relationship Class

```graphql
mutation CreateAssignmentRelation {
  createRelationshipClass(input: {
    name: "assigned_to"
    description: "Tasks assigned to users or teams"
    displayName: "Assigned To"
    isDirectional: true
    isBidirectional: false
    cardinality: MANY_TO_MANY
    sourceLabel: "Assignee"
    targetLabel: "Task"
    inverseLabel: "Assigned By"
    validationRules: {
      sourceConstraints: {
        allowedThingClasses: ["User", "Team"]
      },
      targetConstraints: {
        allowedThingClasses: ["Task", "Issue"]
      }
    }
  }) {
    relationshipClass {
      id
      name
      cardinality
      isDirectional
      validationRules
    }
    errors {
      field
      message
      code
    }
  }
}
```

### Query with Filtering

```graphql
query GetDirectionalRelationships {
  relationshipClasses(
    filter: { 
      isDirectional: true 
      isActive: true 
    }
    orderBy: [{ field: NAME, direction: ASC }]
    limit: 20
  ) {
    nodes {
      id
      name
      displayName
      cardinality
      sourceLabel
      targetLabel
      activeRelationshipCount
    }
    totalCount
    pageInfo {
      hasNextPage
    }
  }
}
```

This simplified API specification follows the established patterns from other entities while providing comprehensive Relationship Class management capabilities through JSON-based validation rules rather than over-engineered array constraints.