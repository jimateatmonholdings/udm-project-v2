# API Specification

This is the API specification for the spec detailed in @.agent-os/specs/2025-08-28-thing-class-entity/spec.md

> Created: 2025-08-28
> Version: 1.0.0

## GraphQL Types

### ThingClass Type

```graphql
type ThingClass {
  id: ID!
  name: String!
  description: String
  validationRules: JSON
  createdAt: DateTime!
  updatedAt: DateTime!
  things(
    first: Int
    after: String
    last: Int
    before: String
    filter: ThingFilter
  ): ThingConnection!
}
```

### Input Types

```graphql
input CreateThingClassInput {
  name: String!
  description: String
  validationRules: JSON
}

input UpdateThingClassInput {
  name: String
  description: String
  validationRules: JSON
}

input ThingClassFilter {
  name: String
  description: String
  createdAfter: DateTime
  createdBefore: DateTime
}
```

### Connection Types

```graphql
type ThingClassConnection {
  edges: [ThingClassEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ThingClassEdge {
  node: ThingClass!
  cursor: String!
}
```

## GraphQL Queries

### Single ThingClass Retrieval

```graphql
extend type Query {
  thingClass(id: ID!): ThingClass
}
```

**Description**: Retrieves a single Thing Class by its unique identifier.

**Parameters**:
- `id`: The unique identifier of the Thing Class

**Returns**: ThingClass object or null if not found

**Authorization**: Requires authenticated user with READ_THING_CLASS permission

### Multiple ThingClasses Retrieval

```graphql
extend type Query {
  thingClasses(
    first: Int
    after: String
    last: Int
    before: String
    filter: ThingClassFilter
  ): ThingClassConnection!
}
```

**Description**: Retrieves a paginated list of Thing Classes with optional filtering.

**Parameters**:
- `first`: Number of items to return from the beginning
- `after`: Cursor for forward pagination
- `last`: Number of items to return from the end
- `before`: Cursor for backward pagination
- `filter`: Optional filter criteria

**Returns**: ThingClassConnection with paginated results

**Authorization**: Requires authenticated user with READ_THING_CLASS permission

## GraphQL Mutations

### Create ThingClass

```graphql
extend type Mutation {
  createThingClass(input: CreateThingClassInput!): ThingClassPayload!
}

type ThingClassPayload {
  thingClass: ThingClass
  errors: [UserError!]!
}
```

**Description**: Creates a new Thing Class in the system.

**Parameters**:
- `input`: CreateThingClassInput containing required and optional fields

**Returns**: ThingClassPayload with created ThingClass or errors

**Authorization**: Requires authenticated user with CREATE_THING_CLASS permission

**Validation Rules**:
- Name must be unique within the system
- Name must be between 1-100 characters
- Description is optional, max 500 characters
- ValidationRules must be valid JSON schema if provided

### Update ThingClass

```graphql
extend type Mutation {
  updateThingClass(id: ID!, input: UpdateThingClassInput!): ThingClassPayload!
}
```

**Description**: Updates an existing Thing Class.

**Parameters**:
- `id`: The unique identifier of the Thing Class to update
- `input`: UpdateThingClassInput containing fields to update

**Returns**: ThingClassPayload with updated ThingClass or errors

**Authorization**: Requires authenticated user with UPDATE_THING_CLASS permission

**Validation Rules**:
- ThingClass must exist
- Name must be unique if being changed
- Name must be between 1-100 characters if provided
- Description max 500 characters if provided
- ValidationRules must be valid JSON schema if provided

### Delete ThingClass

```graphql
extend type Mutation {
  deleteThingClass(id: ID!): DeleteThingClassPayload!
}

type DeleteThingClassPayload {
  deletedThingClassId: ID
  errors: [UserError!]!
}
```

**Description**: Deletes a Thing Class from the system.

**Parameters**:
- `id`: The unique identifier of the Thing Class to delete

**Returns**: DeleteThingClassPayload with deleted ID or errors

**Authorization**: Requires authenticated user with DELETE_THING_CLASS permission

**Business Rules**:
- Cannot delete Thing Class if it has associated Things
- Soft delete implementation - marks as deleted rather than physical removal

## Error Handling

### Thing Class Specific Errors

```graphql
enum ThingClassErrorCode {
  THING_CLASS_NOT_FOUND
  THING_CLASS_NAME_ALREADY_EXISTS
  THING_CLASS_HAS_ASSOCIATED_THINGS
  INVALID_VALIDATION_RULES
  THING_CLASS_NAME_REQUIRED
  THING_CLASS_NAME_TOO_LONG
  THING_CLASS_DESCRIPTION_TOO_LONG
}
```

### Error Response Format

```graphql
type UserError {
  field: String
  message: String!
  code: String
}
```

**Common Error Scenarios**:
- `THING_CLASS_NOT_FOUND`: When querying/updating/deleting non-existent Thing Class
- `THING_CLASS_NAME_ALREADY_EXISTS`: When creating/updating with duplicate name
- `THING_CLASS_HAS_ASSOCIATED_THINGS`: When attempting to delete Thing Class with existing Things
- `INVALID_VALIDATION_RULES`: When provided JSON schema is malformed
- `THING_CLASS_NAME_REQUIRED`: When name is missing in create operation
- `THING_CLASS_NAME_TOO_LONG`: When name exceeds 100 characters
- `THING_CLASS_DESCRIPTION_TOO_LONG`: When description exceeds 500 characters

## Authentication and Authorization

### Required Permissions

- `READ_THING_CLASS`: Required for all query operations
- `CREATE_THING_CLASS`: Required for createThingClass mutation
- `UPDATE_THING_CLASS`: Required for updateThingClass mutation
- `DELETE_THING_CLASS`: Required for deleteThingClass mutation

### Authentication Requirements

- All operations require valid JWT token
- Token must contain valid user ID and permissions
- Token must not be expired or revoked

### Authorization Flow

1. Extract JWT token from Authorization header
2. Validate token signature and expiration
3. Extract user permissions from token claims
4. Verify user has required permission for requested operation
5. Return 401 Unauthorized if authentication fails
6. Return 403 Forbidden if authorization fails

## Rate Limiting

- Query operations: 1000 requests per hour per user
- Mutation operations: 100 requests per hour per user
- Bulk operations: 10 requests per hour per user

## Caching Strategy

- ThingClass queries cached for 5 minutes
- Cache invalidated on any mutation operation
- ETags supported for conditional requests
- Cache key includes user permissions for security