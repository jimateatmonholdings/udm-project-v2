# Attribute Assignment Entity - GraphQL API Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**API Type:** GraphQL-First with Multi-Tenant Architecture  
**Entity:** Attribute Assignment - Bridge connecting Thing Classes to Attributes with constraints  

## API Overview

The Attribute Assignment GraphQL API provides comprehensive CRUD operations and advanced schema composition capabilities for managing the relationships between Thing Classes and Attributes. Built on a schema-per-tenant architecture, this API enables data architects and developers to dynamically configure business entity schemas through assignment management, constraint definition, and impact analysis.

## GraphQL Schema Definition

### Core Types

```graphql
# Scalar types
scalar DateTime
scalar JSON
scalar UUID

# Assignment type enumeration
enum AssignmentType {
  STANDARD
  COMPUTED
  INHERITED
}

# Visibility type enumeration
enum VisibilityType {
  VISIBLE
  HIDDEN
  READONLY
}

# Impact type enumeration for change analysis
enum ImpactType {
  SAFE
  WARNING
  BREAKING
  MIGRATION_REQUIRED
}

# Main Attribute Assignment entity
type ThingClassAttribute implements Node {
  # Core identification
  id: ID!
  
  # Relationship identifiers
  thingClassId: ID!
  attributeId: ID!
  
  # Assignment configuration
  isRequired: Boolean!
  sortOrder: Int!
  displayName: String
  assignmentValidationRules: JSON
  defaultValue: JSON
  uiHints: JSON
  description: String
  assignmentType: AssignmentType!
  visibility: VisibilityType!
  
  # Loaded relationships
  thingClass: ThingClass!
  attribute: Attribute!
  
  # Computed metadata
  usageCount: Int!
  validationSummary: ValidationSummary!
  
  # Standard entity metadata
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: ID!
  updatedBy: ID
  isActive: Boolean!
  version: Int!
}

# Schema composition result for Thing Classes
type ThingClassSchema implements Node {
  id: ID!
  thingClass: ThingClass!
  assignments(
    filter: ThingClassAttributeFilter
    sort: [ThingClassAttributeSort!]
    first: Int = 20
    after: String
  ): ThingClassAttributeConnection!
  totalCount: Int!
  requiredCount: Int!
  optionalCount: Int!
  validationRules: JSON
  schemaHash: String!
  lastModified: DateTime!
}

# Validation summary for assignment
type ValidationSummary {
  hasBaseValidation: Boolean!
  hasAssignmentValidation: Boolean!
  hasDefaultValue: Boolean!
  effectiveRules: JSON
  rulesSummary: String!
  conflictWarnings: [String!]!
}

# Connection types for pagination
type ThingClassAttributeConnection {
  edges: [ThingClassAttributeEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ThingClassAttributeEdge {
  node: ThingClassAttribute!
  cursor: String!
}

# Impact analysis for assignment changes
type AssignmentImpactAnalysis {
  assignment: ThingClassAttribute!
  impactType: ImpactType!
  affectedThingCount: Int!
  requiredThingCount: Int!
  missingValueCount: Int!
  impactDetails: JSON!
  recommendedActions: [String!]!
  migrationRequired: Boolean!
  estimatedMigrationTime: String
}

# Assignment statistics and reporting
type AssignmentStatistics {
  totalCount: Int!
  activeCount: Int!
  requiredCount: Int!
  optionalCount: Int!
  countByAssignmentType: [AssignmentTypeCount!]!
  countByVisibility: [VisibilityCount!]!
  averageAssignmentsPerThingClass: Float!
  thingClassesWithMostAssignments: [ThingClassWithCount!]!
  attributesWithMostAssignments: [AttributeWithCount!]!
  recentlyCreated: [ThingClassAttribute!]!
}

type AssignmentTypeCount {
  assignmentType: AssignmentType!
  count: Int!
  percentage: Float!
}

type VisibilityCount {
  visibility: VisibilityType!
  count: Int!
  percentage: Float!
}

type ThingClassWithCount {
  thingClass: ThingClass!
  assignmentCount: Int!
  requiredAssignmentCount: Int!
}

type AttributeWithCount {
  attribute: Attribute!
  assignmentCount: Int!
  usageCount: Int!
}
```

### Input Types

```graphql
# Assignment creation input
input CreateAttributeAssignmentInput {
  thingClassId: ID! @validate(constraint: "required")
  attributeId: ID! @validate(constraint: "required")
  isRequired: Boolean = false
  sortOrder: Int = 0 @validate(constraint: "min=0,max=999999")
  displayName: String @validate(constraint: "min=1,max=255")
  assignmentValidationRules: JSON
  defaultValue: JSON
  uiHints: JSON
  description: String @validate(constraint: "max=1000")
  assignmentType: AssignmentType = STANDARD
  visibility: VisibilityType = VISIBLE
}

# Assignment update input
input UpdateAttributeAssignmentInput {
  isRequired: Boolean
  sortOrder: Int @validate(constraint: "min=0,max=999999")
  displayName: String @validate(constraint: "min=1,max=255")
  assignmentValidationRules: JSON
  defaultValue: JSON
  uiHints: JSON
  description: String @validate(constraint: "max=1000")
  assignmentType: AssignmentType
  visibility: VisibilityType
}

# Bulk assignment operations
input BulkAssignmentInput {
  thingClassId: ID!
  assignments: [CreateAttributeAssignmentInput!]!
  replaceExisting: Boolean = false
}

# Assignment reordering input
input ReorderAssignmentsInput {
  thingClassId: ID!
  assignmentOrders: [AssignmentOrderInput!]!
}

input AssignmentOrderInput {
  assignmentId: ID!
  sortOrder: Int!
}

# Filtering options
input ThingClassAttributeFilter {
  thingClassId: ID
  attributeId: ID
  isRequired: Boolean
  assignmentType: AssignmentType
  visibility: VisibilityType
  hasDefaultValue: Boolean
  hasCustomValidation: Boolean
  createdBy: ID
  createdAfter: DateTime
  createdBefore: DateTime
  displayNameContains: String
  sortOrderRange: IntRange
}

input IntRange {
  min: Int
  max: Int
}

# Sorting options
input ThingClassAttributeSort {
  field: ThingClassAttributeSortField!
  direction: SortDirection!
}

enum ThingClassAttributeSortField {
  SORT_ORDER
  DISPLAY_NAME
  CREATED_AT
  UPDATED_AT
  IS_REQUIRED
  USAGE_COUNT
  ASSIGNMENT_TYPE
  VISIBILITY
}

# Schema composition filtering
input ThingClassSchemaFilter {
  thingClassId: ID
  hasRequiredAssignments: Boolean
  minAssignmentCount: Int
  maxAssignmentCount: Int
  schemaHash: String
  lastModifiedAfter: DateTime
}
```

### Payload Types

```graphql
# Mutation response types with comprehensive error handling
type AttributeAssignmentPayload {
  assignment: ThingClassAttribute
  errors: [UserError!]!
  clientMutationId: String
}

type BulkAssignmentPayload {
  assignments: [ThingClassAttribute!]!
  errors: [UserError!]!
  successCount: Int!
  failureCount: Int!
  clientMutationId: String
}

type DeleteAssignmentPayload {
  success: Boolean!
  deletedAssignmentId: ID
  impactAnalysis: AssignmentImpactAnalysis
  errors: [UserError!]!
  clientMutationId: String
}

type ReorderAssignmentsPayload {
  assignments: [ThingClassAttribute!]!
  thingClassSchema: ThingClassSchema
  errors: [UserError!]!
  clientMutationId: String
}

# Validation and analysis responses
type AssignmentValidationResult {
  isValid: Boolean!
  errors: [ValidationError!]!
  warnings: [ValidationWarning!]!
  suggestions: [ValidationSuggestion!]!
}

type ValidationError {
  field: String
  message: String!
  code: String!
  details: JSON
}

type ValidationWarning {
  field: String
  message: String!
  recommendation: String
  severity: ValidationSeverity!
}

type ValidationSuggestion {
  field: String
  suggestion: String!
  reason: String!
  autoFixable: Boolean!
}

enum ValidationSeverity {
  LOW
  MEDIUM
  HIGH
  CRITICAL
}

# Error types
type UserError {
  field: String
  message: String!
  code: String!
  details: JSON
  suggestion: String
}
```

## Query Operations

### Root Query Type

```graphql
type Query {
  # Single assignment retrieval
  attributeAssignment(id: ID!): ThingClassAttribute @auth(requires: ["READ_ASSIGNMENT"])
  
  # Assignment collection queries
  attributeAssignments(
    filter: ThingClassAttributeFilter
    sort: [ThingClassAttributeSort!]
    first: Int = 20
    after: String
  ): ThingClassAttributeConnection! @auth(requires: ["READ_ASSIGNMENT"])
  
  # Schema composition queries
  thingClassSchema(
    thingClassId: ID!
    includeInactive: Boolean = false
  ): ThingClassSchema @auth(requires: ["READ_ASSIGNMENT"])
  
  thingClassSchemas(
    filter: ThingClassSchemaFilter
    first: Int = 20
    after: String
  ): ThingClassSchemaConnection! @auth(requires: ["READ_ASSIGNMENT"])
  
  # Assignment by Thing Class
  assignmentsByThingClass(
    thingClassId: ID!
    filter: ThingClassAttributeFilter
    sort: [ThingClassAttributeSort!]
    first: Int = 50
    after: String
  ): ThingClassAttributeConnection! @auth(requires: ["READ_ASSIGNMENT"])
  
  # Assignment by Attribute (where is this attribute used)
  assignmentsByAttribute(
    attributeId: ID!
    filter: ThingClassAttributeFilter
    sort: [ThingClassAttributeSort!]
    first: Int = 50
    after: String
  ): ThingClassAttributeConnection! @auth(requires: ["READ_ASSIGNMENT"])
  
  # Impact analysis for proposed changes
  analyzeAssignmentImpact(
    assignmentId: ID!
    proposedChanges: UpdateAttributeAssignmentInput!
  ): AssignmentImpactAnalysis! @auth(requires: ["UPDATE_ASSIGNMENT"])
  
  # Validation of assignment configuration
  validateAssignmentConfiguration(
    thingClassId: ID!
    attributeId: ID!
    configuration: CreateAttributeAssignmentInput!
  ): AssignmentValidationResult! @auth(requires: ["CREATE_ASSIGNMENT"])
  
  # Bulk validation for schema composition
  validateThingClassSchema(
    thingClassId: ID!
    assignments: [CreateAttributeAssignmentInput!]
  ): ThingClassSchemaValidationResult! @auth(requires: ["READ_ASSIGNMENT"])
  
  # Statistics and reporting
  assignmentStatistics(
    filter: ThingClassAttributeFilter
  ): AssignmentStatistics! @auth(requires: ["READ_ASSIGNMENT"])
  
  # Search assignments by display name or description
  searchAssignments(
    query: String!
    filter: ThingClassAttributeFilter
    first: Int = 20
    after: String
  ): ThingClassAttributeConnection! @auth(requires: ["READ_ASSIGNMENT"])
}

# Additional response types for schema validation
type ThingClassSchemaValidationResult {
  isValid: Boolean!
  thingClass: ThingClass!
  validAssignments: [CreateAttributeAssignmentInput!]!
  invalidAssignments: [InvalidAssignmentResult!]!
  errors: [UserError!]!
  warnings: [ValidationWarning!]!
  suggestions: [ValidationSuggestion!]!
}

type InvalidAssignmentResult {
  input: CreateAttributeAssignmentInput!
  errors: [ValidationError!]!
  canAutoFix: Boolean!
  suggestedFix: CreateAttributeAssignmentInput
}

type ThingClassSchemaConnection {
  edges: [ThingClassSchemaEdge!]!
  pageInfo: PageInfo!
  totalCount: Int!
}

type ThingClassSchemaEdge {
  node: ThingClassSchema!
  cursor: String!
}
```

## Mutation Operations

### Root Mutation Type

```graphql
type Mutation {
  # Core CRUD operations
  createAttributeAssignment(
    input: CreateAttributeAssignmentInput!
    clientMutationId: String
  ): AttributeAssignmentPayload! @auth(requires: ["CREATE_ASSIGNMENT"])
  
  updateAttributeAssignment(
    id: ID!
    input: UpdateAttributeAssignmentInput!
    clientMutationId: String
  ): AttributeAssignmentPayload! @auth(requires: ["UPDATE_ASSIGNMENT"])
  
  deleteAttributeAssignment(
    id: ID!
    force: Boolean = false
    clientMutationId: String
  ): DeleteAssignmentPayload! @auth(requires: ["DELETE_ASSIGNMENT"])
  
  # Bulk operations for efficiency
  createAssignmentsBulk(
    input: BulkAssignmentInput!
    clientMutationId: String
  ): BulkAssignmentPayload! @auth(requires: ["CREATE_ASSIGNMENT"])
  
  deleteAssignmentsBulk(
    assignmentIds: [ID!]!
    force: Boolean = false
    clientMutationId: String
  ): BulkDeleteAssignmentPayload! @auth(requires: ["DELETE_ASSIGNMENT"])
  
  # Advanced assignment management
  reorderAssignments(
    input: ReorderAssignmentsInput!
    clientMutationId: String
  ): ReorderAssignmentsPayload! @auth(requires: ["UPDATE_ASSIGNMENT"])
  
  cloneAssignmentsToThingClass(
    sourceThingClassId: ID!
    targetThingClassId: ID!
    assignmentIds: [ID!]
    clientMutationId: String
  ): BulkAssignmentPayload! @auth(requires: ["CREATE_ASSIGNMENT"])
  
  # Assignment lifecycle management
  activateAssignment(
    id: ID!
    clientMutationId: String
  ): AttributeAssignmentPayload! @auth(requires: ["UPDATE_ASSIGNMENT"])
  
  deactivateAssignment(
    id: ID!
    clientMutationId: String
  ): AttributeAssignmentPayload! @auth(requires: ["UPDATE_ASSIGNMENT"])
  
  # Schema composition operations
  replaceThingClassSchema(
    thingClassId: ID!
    assignments: [CreateAttributeAssignmentInput!]!
    clientMutationId: String
  ): ThingClassSchemaPayload! @auth(requires: ["UPDATE_ASSIGNMENT"])
  
  # Advanced validation and analysis
  applyAssignmentMigration(
    assignmentId: ID!
    migrationPlan: AssignmentMigrationInput!
    clientMutationId: String
  ): AssignmentMigrationPayload! @auth(requires: ["MIGRATE_ASSIGNMENT"])
}

# Additional payload types for advanced operations
type BulkDeleteAssignmentPayload {
  deletedAssignmentIds: [ID!]!
  errors: [UserError!]!
  successCount: Int!
  failureCount: Int!
  impactAnalysis: [AssignmentImpactAnalysis!]!
  clientMutationId: String
}

type ThingClassSchemaPayload {
  thingClassSchema: ThingClassSchema
  assignments: [ThingClassAttribute!]!
  errors: [UserError!]!
  clientMutationId: String
}

type AssignmentMigrationPayload {
  assignment: ThingClassAttribute
  migrationResults: MigrationResults!
  errors: [UserError!]!
  clientMutationId: String
}

input AssignmentMigrationInput {
  strategy: MigrationStrategy!
  parameters: JSON
  dryRun: Boolean = true
}

enum MigrationStrategy {
  PROVIDE_DEFAULTS
  MARK_OPTIONAL
  DELETE_MISSING_VALUES
  CUSTOM_SCRIPT
}

type MigrationResults {
  strategy: MigrationStrategy!
  affectedThingCount: Int!
  migratedValueCount: Int!
  errorCount: Int!
  executionTimeMs: Int!
  dryRun: Boolean!
  details: JSON
}
```

## Subscription Operations

### Real-time Updates

```graphql
type Subscription {
  # Assignment lifecycle events
  assignmentUpdated(
    thingClassId: ID
    attributeId: ID
    assignmentType: AssignmentType
  ): AssignmentUpdateEvent! @auth(requires: ["READ_ASSIGNMENT"])
  
  assignmentCreated(
    thingClassId: ID
    assignmentType: AssignmentType
  ): AssignmentCreateEvent! @auth(requires: ["READ_ASSIGNMENT"])
  
  assignmentDeleted(
    thingClassId: ID
  ): AssignmentDeleteEvent! @auth(requires: ["READ_ASSIGNMENT"])
  
  # Schema composition events
  thingClassSchemaChanged(
    thingClassId: ID
  ): ThingClassSchemaChangeEvent! @auth(requires: ["READ_ASSIGNMENT"])
  
  # Impact analysis events
  assignmentImpactAnalyzed(
    assignmentId: ID
    impactType: ImpactType
  ): AssignmentImpactEvent! @auth(requires: ["READ_ASSIGNMENT"])
}

# Event types for subscriptions
type AssignmentUpdateEvent {
  assignment: ThingClassAttribute!
  previousVersion: ThingClassAttribute
  changeType: AssignmentChangeType!
  impactAnalysis: AssignmentImpactAnalysis
  timestamp: DateTime!
  updatedBy: ID!
}

type AssignmentCreateEvent {
  assignment: ThingClassAttribute!
  thingClassSchema: ThingClassSchema!
  timestamp: DateTime!
  createdBy: ID!
}

type AssignmentDeleteEvent {
  assignmentId: ID!
  thingClassId: ID!
  attributeId: ID!
  impactAnalysis: AssignmentImpactAnalysis!
  timestamp: DateTime!
  deletedBy: ID!
}

type ThingClassSchemaChangeEvent {
  thingClassSchema: ThingClassSchema!
  changeType: SchemaChangeType!
  affectedAssignments: [ThingClassAttribute!]!
  timestamp: DateTime!
  changedBy: ID!
}

type AssignmentImpactEvent {
  assignment: ThingClassAttribute!
  impactAnalysis: AssignmentImpactAnalysis!
  timestamp: DateTime!
  analyzedBy: ID!
}

enum AssignmentChangeType {
  CONSTRAINT_CHANGED
  METADATA_CHANGED
  VALIDATION_CHANGED
  ORDERING_CHANGED
  STATUS_CHANGED
}

enum SchemaChangeType {
  ASSIGNMENT_ADDED
  ASSIGNMENT_REMOVED
  ASSIGNMENT_REORDERED
  CONSTRAINTS_CHANGED
  BULK_UPDATE
}
```

## Example Queries and Mutations

### Query Examples

```graphql
# 1. Get complete Thing Class schema with all assignments
query GetThingClassSchema($thingClassId: ID!) {
  thingClassSchema(thingClassId: $thingClassId) {
    id
    thingClass {
      id
      name
      description
    }
    totalCount
    requiredCount
    optionalCount
    schemaHash
    lastModified
    assignments {
      edges {
        node {
          id
          isRequired
          sortOrder
          displayName
          assignmentType
          visibility
          defaultValue
          attribute {
            id
            name
            dataType
            description
          }
          validationSummary {
            hasBaseValidation
            hasAssignmentValidation
            hasDefaultValue
            effectiveRules
            conflictWarnings
          }
        }
      }
      totalCount
    }
  }
}

# 2. List assignments with filtering and sorting
query ListAssignmentsByThingClass($thingClassId: ID!, $first: Int, $after: String) {
  assignmentsByThingClass(
    thingClassId: $thingClassId
    sort: [{field: SORT_ORDER, direction: ASC}]
    first: $first
    after: $after
  ) {
    edges {
      node {
        id
        isRequired
        sortOrder
        displayName
        assignmentType
        visibility
        attribute {
          id
          name
          dataType
        }
        usageCount
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

# 3. Search assignments across all Thing Classes
query SearchAssignments($query: String!, $first: Int!) {
  searchAssignments(
    query: $query
    first: $first
    sort: [{field: DISPLAY_NAME, direction: ASC}]
  ) {
    edges {
      node {
        id
        displayName
        description
        thingClass {
          id
          name
        }
        attribute {
          id
          name
          dataType
        }
        isRequired
        assignmentType
        visibility
      }
    }
    totalCount
  }
}

# 4. Analyze impact of proposed changes
query AnalyzeAssignmentImpact($assignmentId: ID!, $changes: UpdateAttributeAssignmentInput!) {
  analyzeAssignmentImpact(
    assignmentId: $assignmentId
    proposedChanges: $changes
  ) {
    assignment {
      id
      displayName
      thingClass {
        name
      }
      attribute {
        name
        dataType
      }
    }
    impactType
    affectedThingCount
    missingValueCount
    migrationRequired
    recommendedActions
    estimatedMigrationTime
    impactDetails
  }
}

# 5. Get assignment statistics
query GetAssignmentStatistics {
  assignmentStatistics {
    totalCount
    activeCount
    requiredCount
    optionalCount
    countByAssignmentType {
      assignmentType
      count
      percentage
    }
    countByVisibility {
      visibility
      count
      percentage
    }
    averageAssignmentsPerThingClass
    thingClassesWithMostAssignments {
      thingClass {
        id
        name
      }
      assignmentCount
      requiredAssignmentCount
    }
    attributesWithMostAssignments {
      attribute {
        id
        name
        dataType
      }
      assignmentCount
      usageCount
    }
  }
}

# 6. Validate assignment configuration before creation
query ValidateAssignmentConfig($thingClassId: ID!, $attributeId: ID!, $config: CreateAttributeAssignmentInput!) {
  validateAssignmentConfiguration(
    thingClassId: $thingClassId
    attributeId: $attributeId
    configuration: $config
  ) {
    isValid
    errors {
      field
      message
      code
      details
    }
    warnings {
      field
      message
      recommendation
      severity
    }
    suggestions {
      field
      suggestion
      reason
      autoFixable
    }
  }
}
```

### Mutation Examples

```graphql
# 1. Create a new assignment with full configuration
mutation CreateAssignment($input: CreateAttributeAssignmentInput!) {
  createAttributeAssignment(
    input: $input
    clientMutationId: "create-assignment-001"
  ) {
    assignment {
      id
      isRequired
      sortOrder
      displayName
      assignmentValidationRules
      defaultValue
      uiHints
      assignmentType
      visibility
      thingClass {
        id
        name
      }
      attribute {
        id
        name
        dataType
      }
      validationSummary {
        hasBaseValidation
        hasAssignmentValidation
        hasDefaultValue
        effectiveRules
      }
    }
    errors {
      field
      message
      code
      suggestion
    }
  }
}

# 2. Update assignment with impact analysis
mutation UpdateAssignment($id: ID!, $input: UpdateAttributeAssignmentInput!) {
  updateAttributeAssignment(
    id: $id
    input: $input
    clientMutationId: "update-assignment-001"
  ) {
    assignment {
      id
      isRequired
      sortOrder
      displayName
      version
      updatedAt
      validationSummary {
        hasBaseValidation
        hasAssignmentValidation
        effectiveRules
        conflictWarnings
      }
    }
    errors {
      field
      message
      code
      details
    }
  }
}

# 3. Bulk create assignments for Thing Class
mutation CreateAssignmentsBulk($input: BulkAssignmentInput!) {
  createAssignmentsBulk(
    input: $input
    clientMutationId: "bulk-create-001"
  ) {
    assignments {
      id
      sortOrder
      displayName
      attribute {
        id
        name
        dataType
      }
    }
    successCount
    failureCount
    errors {
      field
      message
      code
      details
    }
  }
}

# 4. Reorder assignments within Thing Class
mutation ReorderAssignments($input: ReorderAssignmentsInput!) {
  reorderAssignments(
    input: $input
    clientMutationId: "reorder-001"
  ) {
    assignments {
      id
      sortOrder
      displayName
    }
    thingClassSchema {
      schemaHash
      lastModified
    }
    errors {
      field
      message
      code
    }
  }
}

# 5. Delete assignment with impact analysis
mutation DeleteAssignment($id: ID!, $force: Boolean!) {
  deleteAttributeAssignment(
    id: $id
    force: $force
    clientMutationId: "delete-assignment-001"
  ) {
    success
    deletedAssignmentId
    impactAnalysis {
      impactType
      affectedThingCount
      missingValueCount
      recommendedActions
      migrationRequired
    }
    errors {
      message
      code
      details
    }
  }
}

# 6. Clone assignments between Thing Classes
mutation CloneAssignments($sourceThingClassId: ID!, $targetThingClassId: ID!, $assignmentIds: [ID!]) {
  cloneAssignmentsToThingClass(
    sourceThingClassId: $sourceThingClassId
    targetThingClassId: $targetThingClassId
    assignmentIds: $assignmentIds
    clientMutationId: "clone-assignments-001"
  ) {
    assignments {
      id
      sortOrder
      displayName
      thingClass {
        id
        name
      }
      attribute {
        id
        name
        dataType
      }
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

# 7. Replace entire Thing Class schema
mutation ReplaceThingClassSchema($thingClassId: ID!, $assignments: [CreateAttributeAssignmentInput!]!) {
  replaceThingClassSchema(
    thingClassId: $thingClassId
    assignments: $assignments
    clientMutationId: "replace-schema-001"
  ) {
    thingClassSchema {
      id
      totalCount
      requiredCount
      optionalCount
      schemaHash
      lastModified
    }
    assignments {
      id
      sortOrder
      displayName
      isRequired
      attribute {
        id
        name
        dataType
      }
    }
    errors {
      field
      message
      code
      details
    }
  }
}
```

### Subscription Examples

```graphql
# 1. Listen for assignment updates on specific Thing Class
subscription AssignmentUpdates($thingClassId: ID) {
  assignmentUpdated(thingClassId: $thingClassId) {
    assignment {
      id
      displayName
      isRequired
      sortOrder
      version
      thingClass {
        id
        name
      }
      attribute {
        id
        name
        dataType
      }
    }
    previousVersion {
      isRequired
      sortOrder
      version
    }
    changeType
    impactAnalysis {
      impactType
      affectedThingCount
      recommendedActions
    }
    timestamp
    updatedBy
  }
}

# 2. Monitor Thing Class schema changes
subscription ThingClassSchemaChanges($thingClassId: ID) {
  thingClassSchemaChanged(thingClassId: $thingClassId) {
    thingClassSchema {
      id
      totalCount
      requiredCount
      schemaHash
      lastModified
    }
    changeType
    affectedAssignments {
      id
      displayName
      sortOrder
    }
    timestamp
    changedBy
  }
}

# 3. Listen for high-impact assignment changes
subscription HighImpactChanges {
  assignmentImpactAnalyzed(impactType: BREAKING) {
    assignment {
      id
      displayName
      thingClass {
        id
        name
      }
      attribute {
        id
        name
      }
    }
    impactAnalysis {
      impactType
      affectedThingCount
      migrationRequired
      recommendedActions
    }
    timestamp
    analyzedBy
  }
}
```

## Error Handling

### GraphQL Error Extensions

```json
{
  "errors": [
    {
      "message": "Assignment already exists for this Thing Class and Attribute combination",
      "extensions": {
        "code": "ASSIGNMENT_ALREADY_EXISTS",
        "field": "attributeId",
        "details": {
          "thingClassId": "123e4567-e89b-12d3-a456-426614174000",
          "attributeId": "987fcdeb-51d2-43a1-b123-426614174000",
          "existingAssignmentId": "456e7890-e89b-12d3-a456-426614174000"
        },
        "tenantSchema": "tenant_company_a",
        "requestId": "req_8c2d4f6e9a1b",
        "suggestion": "Update the existing assignment instead of creating a new one"
      },
      "path": ["createAttributeAssignment"],
      "locations": [{"line": 2, "column": 3}]
    }
  ]
}
```

### Error Code Catalog

```graphql
enum AssignmentErrorCode {
  # Validation errors
  ASSIGNMENT_NOT_FOUND
  ASSIGNMENT_ALREADY_EXISTS
  THING_CLASS_NOT_FOUND
  ATTRIBUTE_NOT_FOUND
  INVALID_SORT_ORDER
  SORT_ORDER_CONFLICT
  INVALID_DEFAULT_VALUE
  INVALID_VALIDATION_RULES
  INVALID_UI_HINTS
  
  # Business logic errors
  ASSIGNMENT_IN_USE
  BREAKING_CHANGE_DETECTED
  MIGRATION_REQUIRED
  CONSTRAINT_VIOLATION
  CIRCULAR_DEPENDENCY
  SCHEMA_VALIDATION_FAILED
  
  # System errors
  TENANT_ISOLATION_ERROR
  SCHEMA_COMPOSITION_FAILED
  IMPACT_ANALYSIS_FAILED
  BULK_OPERATION_PARTIAL_FAILURE
  REORDER_OPERATION_FAILED
  
  # Permission errors
  INSUFFICIENT_PERMISSIONS
  READ_ONLY_ASSIGNMENT
  PROTECTED_SCHEMA_MODIFICATION
}
```

## Performance Characteristics

### Query Complexity Analysis

```graphql
# Query complexity calculation rules
# - Basic field: 1 point
# - Connection field: multiplier based on first/last arguments  
# - Nested relationships: additive complexity
# - Schema composition: 5x multiplier for deep hydration
# - Maximum allowed complexity: 150 points

query ComplexSchemaQuery {
  # Complexity: 1 + 10*5 (schema with assignments) + 2*10*5 (nested attributes) = 151 points (over limit)
  thingClassSchemas(first: 10) {
    edges {
      node {
        id                    # +1
        thingClass { name }   # +1
        assignments(first: 5) {  # +5*5 = 25
          edges {
            node {
              id              # +1 per edge
              attribute {     # +1 per edge  
                name
                dataType
              }
              validationSummary {  # +2 per edge
                effectiveRules
                conflictWarnings
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
  # Cached for 10 minutes, tenant-scoped
  attributeAssignment(id: ID!): ThingClassAttribute 
    @cached(ttl: 600, scope: TENANT)
  
  # Cached for 5 minutes due to dynamic nature with relationships
  thingClassSchema(thingClassId: ID!): ThingClassSchema
    @cached(ttl: 300, scope: TENANT)
  
  # Cached for 2 minutes, frequently changing
  attributeAssignments(filter: ThingClassAttributeFilter, first: Int, after: String): ThingClassAttributeConnection!
    @cached(ttl: 120, scope: TENANT)
  
  # Cached for 30 minutes, statistical data changes slowly
  assignmentStatistics(filter: ThingClassAttributeFilter): AssignmentStatistics!
    @cached(ttl: 1800, scope: TENANT)
}
```

### Performance Targets

- **Single assignment query**: <200ms P95 latency with relationship hydration
- **Assignment list query (20 items)**: <500ms P95 latency with filtering
- **Thing Class schema composition**: <300ms P95 latency for complete schema
- **Search queries**: <800ms P95 latency with full-text search
- **Mutation operations**: <400ms P95 latency including validation and impact analysis
- **Subscription events**: <100ms delivery latency for real-time updates
- **Maximum query complexity**: 150 points
- **Rate limits**: 1500 queries/minute, 300 mutations/minute per tenant

This comprehensive GraphQL API specification provides a robust, performant, and secure interface for managing Attribute Assignment entities within the UDM system, supporting all CRUD operations, advanced schema composition, impact analysis, and real-time updates.