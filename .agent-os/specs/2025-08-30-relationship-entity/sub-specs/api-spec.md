# API Specification

This is the API specification for the spec detailed in @.agent-os/specs/2025-08-30-relationship-entity/spec.md

> Created: 2025-08-30
> Version: 1.0.0

## GraphQL Schema

### Types

#### Relationship
```graphql
type Relationship {
  id: ID!
  tenantId: ID!
  relationshipType: ThingClass!
  sourceThingId: ID!
  sourceThing: Thing!
  targetThingId: ID!
  targetThing: Thing!
  status: RelationshipStatus!
  metadata: JSON
  isBidirectional: Boolean!
  effectiveFrom: DateTime!
  effectiveUntil: DateTime
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID
  updatedBy: ID
  version: Int!
  
  # Computed fields
  values: [Value!]!
  attributes: [Attribute!]!
  isActive: Boolean!
  reverseRelationship: Relationship
}
```

#### RelationshipConstraint
```graphql
type RelationshipConstraint {
  id: ID!
  tenantId: ID!
  relationshipType: ThingClass!
  sourceThingClass: ThingClass
  targetThingClass: ThingClass
  minInstances: Int!
  maxInstances: Int
  isRequired: Boolean!
  constraintType: RelationshipConstraintType!
  constraintMetadata: JSON
  createdAt: DateTime!
  updatedAt: DateTime!
}
```

#### RelationshipHistory
```graphql
type RelationshipHistory {
  id: ID!
  tenantId: ID!
  relationshipId: ID!
  operationType: RelationshipOperationType!
  oldValues: JSON
  newValues: JSON
  changedBy: ID
  changedAt: DateTime!
  changeReason: String
}
```

### Enums

#### RelationshipStatus
```graphql
enum RelationshipStatus {
  ACTIVE
  INACTIVE
  PENDING
  DELETED
}
```

#### RelationshipConstraintType
```graphql
enum RelationshipConstraintType {
  CARDINALITY
  EXCLUSIVITY
  DEPENDENCY
  HIERARCHY
}
```

#### RelationshipOperationType
```graphql
enum RelationshipOperationType {
  CREATED
  UPDATED
  DELETED
  ACTIVATED
  DEACTIVATED
}
```

### Input Types

#### CreateRelationshipInput
```graphql
input CreateRelationshipInput {
  relationshipTypeId: ID!
  sourceThingId: ID!
  targetThingId: ID!
  status: RelationshipStatus = ACTIVE
  metadata: JSON
  isBidirectional: Boolean = false
  effectiveFrom: DateTime
  effectiveUntil: DateTime
  values: [CreateValueInput!]
}
```

#### UpdateRelationshipInput
```graphql
input UpdateRelationshipInput {
  id: ID!
  status: RelationshipStatus
  metadata: JSON
  isBidirectional: Boolean
  effectiveFrom: DateTime
  effectiveUntil: DateTime
  values: [UpdateValueInput!]
}
```

#### RelationshipFilter
```graphql
input RelationshipFilter {
  relationshipTypeIds: [ID!]
  sourceThingIds: [ID!]
  targetThingIds: [ID!]
  status: [RelationshipStatus!]
  isBidirectional: Boolean
  effectiveAtDate: DateTime
  createdDateRange: DateTimeRange
  updatedDateRange: DateTimeRange
  metadataContains: JSON
  search: String
}
```

#### RelationshipTraversalInput
```graphql
input RelationshipTraversalInput {
  startThingId: ID!
  relationshipTypeIds: [ID!]
  direction: TraversalDirection = BOTH
  maxDepth: Int = 1
  includeInactive: Boolean = false
  filter: RelationshipFilter
}
```

#### TraversalDirection
```graphql
enum TraversalDirection {
  OUTBOUND    # Follow relationships where thing is source
  INBOUND     # Follow relationships where thing is target
  BOTH        # Follow relationships in both directions
}
```

## Queries

### Basic Relationship Queries

#### Get Relationship by ID
```graphql
query GetRelationship($id: ID!) {
  relationship(id: $id) {
    id
    relationshipType {
      id
      name
      description
    }
    sourceThing {
      id
      name
      thingClass {
        name
      }
    }
    targetThing {
      id
      name
      thingClass {
        name
      }
    }
    status
    metadata
    isBidirectional
    effectiveFrom
    effectiveUntil
    values {
      id
      attribute {
        name
        dataType
      }
      stringValue
      numberValue
      booleanValue
      dateValue
      jsonValue
    }
    createdAt
    updatedAt
    version
  }
}
```

#### List Relationships
```graphql
query ListRelationships(
  $filter: RelationshipFilter,
  $pagination: PaginationInput,
  $sorting: [SortInput!]
) {
  relationships(
    filter: $filter,
    pagination: $pagination,
    sorting: $sorting
  ) {
    items {
      id
      relationshipType {
        name
      }
      sourceThing {
        name
      }
      targetThing {
        name
      }
      status
      isBidirectional
      createdAt
    }
    pageInfo {
      hasNextPage
      hasPreviousPage
      totalCount
    }
  }
}
```

### Relationship Traversal Queries

#### Traverse Relationships
```graphql
query TraverseRelationships($input: RelationshipTraversalInput!) {
  traverseRelationships(input: $input) {
    paths {
      depth
      relationships {
        id
        relationshipType {
          name
        }
        sourceThing {
          id
          name
        }
        targetThing {
          id
          name
        }
        direction # OUTBOUND, INBOUND based on traversal
      }
      targetThing {
        id
        name
        thingClass {
          name
        }
      }
    }
    statistics {
      totalPaths
      maxDepthReached
      uniqueThingsFound
      relationshipTypesUsed
    }
  }
}
```

#### Get Thing Relationships
```graphql
query GetThingRelationships(
  $thingId: ID!,
  $direction: TraversalDirection = BOTH,
  $filter: RelationshipFilter
) {
  thing(id: $thingId) {
    id
    name
    outboundRelationships(filter: $filter) {
      id
      relationshipType {
        name
      }
      targetThing {
        id
        name
      }
      status
    }
    inboundRelationships(filter: $filter) {
      id
      relationshipType {
        name
      }
      sourceThing {
        id
        name
      }
      status
    }
    relationshipCount(direction: $direction, filter: $filter)
  }
}
```

### Constraint and Validation Queries

#### Get Relationship Constraints
```graphql
query GetRelationshipConstraints($relationshipTypeId: ID!) {
  relationshipConstraints(relationshipTypeId: $relationshipTypeId) {
    id
    relationshipType {
      name
    }
    sourceThingClass {
      name
    }
    targetThingClass {
      name
    }
    minInstances
    maxInstances
    isRequired
    constraintType
    constraintMetadata
  }
}
```

#### Validate Relationship
```graphql
query ValidateRelationship($input: CreateRelationshipInput!) {
  validateRelationship(input: $input) {
    isValid
    violations {
      constraintType
      message
      severity
      fieldPath
    }
    suggestions {
      message
      action
    }
  }
}
```

## Mutations

### Relationship Management

#### Create Relationship
```graphql
mutation CreateRelationship($input: CreateRelationshipInput!) {
  createRelationship(input: $input) {
    success
    relationship {
      id
      relationshipType {
        name
      }
      sourceThing {
        name
      }
      targetThing {
        name
      }
      status
      isBidirectional
      createdAt
    }
    errors {
      field
      message
      code
    }
  }
}
```

#### Update Relationship
```graphql
mutation UpdateRelationship($input: UpdateRelationshipInput!) {
  updateRelationship(input: $input) {
    success
    relationship {
      id
      status
      metadata
      updatedAt
      version
    }
    errors {
      field
      message
      code
    }
  }
}
```

#### Delete Relationship
```graphql
mutation DeleteRelationship($id: ID!, $soft: Boolean = true) {
  deleteRelationship(id: $id, soft: $soft) {
    success
    message
    errors {
      field
      message
      code
    }
  }
}
```

### Batch Operations

#### Create Multiple Relationships
```graphql
mutation CreateMultipleRelationships($inputs: [CreateRelationshipInput!]!) {
  createMultipleRelationships(inputs: $inputs) {
    success
    results {
      index
      success
      relationship {
        id
      }
      errors {
        field
        message
        code
      }
    }
    summary {
      total
      successful
      failed
    }
  }
}
```

#### Update Multiple Relationships
```graphql
mutation UpdateMultipleRelationships($inputs: [UpdateRelationshipInput!]!) {
  updateMultipleRelationships(inputs: $inputs) {
    success
    results {
      index
      success
      relationship {
        id
        version
      }
      errors {
        field
        message
        code
      }
    }
    summary {
      total
      successful
      failed
    }
  }
}
```

### Relationship Status Management

#### Activate Relationship
```graphql
mutation ActivateRelationship($id: ID!) {
  activateRelationship(id: $id) {
    success
    relationship {
      id
      status
      updatedAt
    }
    errors {
      field
      message
      code
    }
  }
}
```

#### Deactivate Relationship
```graphql
mutation DeactivateRelationship($id: ID!, $reason: String) {
  deactivateRelationship(id: $id, reason: $reason) {
    success
    relationship {
      id
      status
      updatedAt
    }
    errors {
      field
      message
      code
    }
  }
}
```

## Subscriptions

#### Relationship Changes
```graphql
subscription RelationshipChanged($filter: RelationshipFilter) {
  relationshipChanged(filter: $filter) {
    operation
    relationship {
      id
      relationshipType {
        name
      }
      sourceThing {
        name
      }
      targetThing {
        name
      }
      status
    }
    changedFields
    timestamp
  }
}
```

#### Thing Relationship Changes
```graphql
subscription ThingRelationshipsChanged($thingId: ID!) {
  thingRelationshipsChanged(thingId: $thingId) {
    thingId
    operation
    relationship {
      id
      relationshipType {
        name
      }
      status
    }
    direction # Whether thing was source or target
    timestamp
  }
}
```

## Performance Expectations

- **Relationship Queries**: <200ms for up to 10,000 relationships
- **Relationship Traversal**: <500ms for up to 5 levels deep, 1,000 relationships per level
- **Batch Operations**: <2s for up to 100 relationships
- **Real-time Updates**: <100ms latency for subscription notifications
- **Constraint Validation**: <50ms for relationship validation checks

## Error Handling

### Error Codes
- `RELATIONSHIP_NOT_FOUND`: Relationship with specified ID not found
- `INVALID_RELATIONSHIP_TYPE`: Relationship type not valid for source/target things
- `CONSTRAINT_VIOLATION`: Relationship violates defined constraints
- `CIRCULAR_RELATIONSHIP`: Relationship would create circular dependency
- `DUPLICATE_RELATIONSHIP`: Relationship already exists between things
- `INACTIVE_THING`: Source or target thing is inactive
- `PERMISSION_DENIED`: User lacks permission for operation
- `VALIDATION_FAILED`: Input validation failed

### Error Response Format
```graphql
type RelationshipError {
  field: String
  message: String!
  code: String!
  details: JSON
}
```