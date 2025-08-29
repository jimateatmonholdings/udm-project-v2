# API Specification

This is the API specification for the spec detailed in @.agent-os/specs/2025-08-28-thing-entity/spec.md

> Created: 2025-08-28
> Version: 1.0.0

## GraphQL Schema

### Thing Type Definition

```graphql
type Thing {
  id: ID!
  name: String!
  thingClass: ThingClass!
  attributeValues: [Value!]!
  relationships: [ThingRelationship!]!
  # Common entity fields
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  isActive: Boolean!
  version: Int!
  tenantId: ID!
}

type Value {
  id: ID!
  thingId: ID!
  attributeId: ID!
  thing: Thing!
  attribute: Attribute!
  valueData: JSON!
  # Common entity fields
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  isActive: Boolean!
  version: Int!
}

type ThingClass {
  id: ID!
  name: String!
  description: String
  attributes: [ThingClassAttribute!]!
  # Common entity fields
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  isActive: Boolean!
  version: Int!
}

type Relationship {
  id: ID!
  fromThing: Thing!
  toThing: Thing!
  relationshipType: RelationshipType!
  attributes: JSON
  createdAt: DateTime!
  updatedAt: DateTime!
}

type RelationshipType {
  id: ID!
  name: String!
  description: String
  fromThingClass: ThingClass!
  toThingClass: ThingClass!
  attributeSchema: JSON
}
```

### Connection Types for Pagination

```graphql
type ThingConnection {
  edges: [ThingEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ThingEdge {
  node: Thing!
  cursor: String!
}

type RelationshipConnection {
  edges: [RelationshipEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type RelationshipEdge {
  node: Relationship!
  cursor: String!
}

type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}
```

### Input Types

```graphql
input CreateThingInput {
  thingClassId: ID! @validate(constraint: "required")
  name: String! @validate(constraint: "required,min=1,max=255")
}

input UpdateThingInput {
  id: ID!
  name: String @validate(constraint: "min=1,max=255")
}

input CreateValueInput {
  thingId: ID! @validate(constraint: "required")
  attributeId: ID! @validate(constraint: "required")
  valueData: JSON! @validate(constraint: "required")
}

input UpdateValueInput {
  valueData: JSON! @validate(constraint: "required")
}

input CreateThingRelationshipInput {
  sourceThingId: ID! @validate(constraint: "required")
  targetThingId: ID! @validate(constraint: "required")
  relationshipType: String! @validate(constraint: "required,min=1,max=100")
  metadata: JSON
}

input ThingFiltersInput {
  thingClassIds: [ID!]
  attributeFilters: [AttributeFilterInput!]
  createdAfter: DateTime
  createdBefore: DateTime
  updatedAfter: DateTime
  updatedBefore: DateTime
  hasRelationships: Boolean
  relationshipTypeIds: [ID!]
  relatedThingIds: [ID!]
}

input AttributeFilterInput {
  key: String!
  operator: FilterOperator!
  value: JSON!
}

enum FilterOperator {
  EQUALS
  NOT_EQUALS
  CONTAINS
  NOT_CONTAINS
  GREATER_THAN
  LESS_THAN
  GREATER_THAN_OR_EQUAL
  LESS_THAN_OR_EQUAL
  IN
  NOT_IN
  EXISTS
  NOT_EXISTS
}
```

### Queries

```graphql
type Query {
  # Single Thing retrieval
  thing(id: ID!): Thing
  
  # Multiple Things with filtering and pagination
  things(
    filters: ThingFiltersInput
    first: Int
    after: String
    last: Int
    before: String
    orderBy: ThingOrderBy
  ): ThingConnection!
  
  # Search Things by text
  searchThings(
    query: String!
    thingClassIds: [ID!]
    first: Int
    after: String
  ): ThingConnection!
}

enum ThingOrderBy {
  CREATED_AT_ASC
  CREATED_AT_DESC
  UPDATED_AT_ASC
  UPDATED_AT_DESC
  THING_CLASS_NAME_ASC
  THING_CLASS_NAME_DESC
}
```

### Mutations

```graphql
type Mutation {
  # Create a new Thing
  createThing(input: CreateThingInput!): CreateThingPayload!
  
  # Update an existing Thing
  updateThing(input: UpdateThingInput!): UpdateThingPayload!
  
  # Delete a Thing
  deleteThing(id: ID!): DeleteThingPayload!
}

type CreateThingPayload {
  thing: Thing
  errors: [UserError!]!
}

type UpdateThingPayload {
  thing: Thing
  errors: [UserError!]!
}

type DeleteThingPayload {
  deletedThingId: ID
  errors: [UserError!]!
}

type UserError {
  field: String
  message: String!
  code: String!
}
```

## Field Selection Optimization

### Selective Loading Strategies

1. **Lazy Loading for Relationships**
   - Relationships are loaded only when requested
   - Use DataLoader pattern to batch relationship queries
   - Implement N+1 query prevention

2. **Attribute Schema Validation**
   - Validate attributes against ThingClass schema on write
   - Cache ThingClass schemas for read operations
   - Use JSON path queries for efficient attribute filtering

3. **Connection Optimization**
   - Implement cursor-based pagination with database indexes
   - Use connection pooling for database queries
   - Cache frequently accessed Things with Redis

### Query Optimization Examples

```graphql
# Minimal query - only essential fields
query GetThingBasic($id: ID!) {
  thing(id: $id) {
    id
    thingClass {
      name
    }
  }
}

# Full query with relationships
query GetThingWithRelationships($id: ID!) {
  thing(id: $id) {
    id
    thingClass {
      name
      attributeSchema
    }
    attributes
    relationships(first: 10) {
      edges {
        node {
          relationshipType {
            name
          }
          toThing {
            id
            thingClass {
              name
            }
          }
        }
      }
      pageInfo {
        hasNextPage
      }
    }
  }
}
```

## Error Handling

### Thing-Specific Error Codes

```graphql
enum ThingErrorCode {
  THING_NOT_FOUND
  THING_CLASS_NOT_FOUND
  INVALID_ATTRIBUTES
  ATTRIBUTE_VALIDATION_FAILED
  THING_CLASS_MISMATCH
  RELATIONSHIP_CONSTRAINT_VIOLATION
  DUPLICATE_THING
  INSUFFICIENT_PERMISSIONS
  THING_IN_USE
  ATTRIBUTE_SCHEMA_VIOLATION
}
```

### Error Response Examples

```json
{
  "errors": [
    {
      "message": "Thing not found",
      "code": "THING_NOT_FOUND",
      "path": ["thing"],
      "extensions": {
        "thingId": "123",
        "timestamp": "2025-08-28T15:30:00Z"
      }
    }
  ]
}

{
  "data": {
    "createThing": {
      "thing": null,
      "errors": [
        {
          "field": "attributes.name",
          "message": "Name is required for this ThingClass",
          "code": "ATTRIBUTE_VALIDATION_FAILED"
        }
      ]
    }
  }
}
```

## Performance Considerations

### Response Time Targets (<200ms)

1. **Database Optimization**
   - Index on `thingClassId`, `createdAt`, `updatedAt`
   - Composite indexes for common filter combinations
   - JSONB indexes on frequently queried attribute paths
   - Connection pooling with 10-50 connections

2. **Caching Strategy**
   - Redis cache for frequently accessed Things (5-minute TTL)
   - ThingClass schema caching (1-hour TTL)
   - Query result caching for complex filtered queries
   - Cache invalidation on Thing updates

3. **Query Optimization**
   - Use prepared statements for all queries
   - Implement query complexity analysis
   - Limit nested relationship depth to 3 levels
   - Batch attribute validation queries

4. **DataLoader Implementation**
   ```javascript
   const thingLoader = new DataLoader(async (ids) => {
     const things = await db.things.findMany({
       where: { id: { in: ids } },
       include: { thingClass: true }
     });
     return ids.map(id => things.find(thing => thing.id === id));
   });
   ```

### Pagination Performance

- Use cursor-based pagination for large datasets
- Implement stable sorting with `id` as secondary sort key
- Cache page counts for filtered queries
- Use database `LIMIT` and `OFFSET` efficiently

### Monitoring and Metrics

- Track query execution times per operation type
- Monitor cache hit ratios
- Alert on queries exceeding 150ms
- Log slow queries for optimization analysis