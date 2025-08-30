# Value Entity - GraphQL API Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**API Type:** GraphQL with Multi-Tenant Support  
**Entity:** Value - Property data storage operations  

## API Architecture Overview

The Value GraphQL API provides comprehensive data operations for storing, retrieving, and managing property values within the Universal Data Model system. Supporting all 8 attribute data types with advanced filtering, validation, and versioning capabilities, the API maintains sub-200ms performance targets while ensuring data integrity and multi-tenant isolation.

## GraphQL Schema Definition

### Core Value Types

```graphql
"""
Represents an actual property value stored for a Thing-Attribute combination
"""
type Value {
  """Unique identifier for the value record"""
  id: UUID!
  
  """Thing that owns this value"""
  thingId: UUID!
  
  """Attribute definition for this value"""
  attributeId: UUID!
  
  """Assignment configuration used for this value"""
  assignmentId: UUID!
  
  """Data type of the stored value"""
  dataType: AttributeDataType!
  
  """Typed value content based on data type"""
  typedValue: ValueContent!
  
  """Whether the value passes all validation rules"""
  isValid: Boolean!
  
  """Array of validation errors if any"""
  validationErrors: [ValidationError!]!
  
  """When the value was last validated"""
  validationTimestamp: DateTime!
  
  """Version number for this value"""
  version: Int!
  
  """Whether this is the current active version"""
  isCurrent: Boolean!
  
  """When this value version became effective"""
  effectiveFrom: DateTime!
  
  """When this value version was superseded (null for current)"""
  effectiveTo: DateTime
  
  """Reference to the value version that replaced this one"""
  supersededBy: UUID
  
  """Standard entity metadata"""
  createdAt: DateTime!
  updatedAt: DateTime!
  createdBy: UUID!
  updatedBy: UUID
  isActive: Boolean!
  
  """Resolved relationships (loaded on demand)"""
  thing: Thing
  attribute: Attribute
  assignment: ThingClassAttribute
  referenceThing: Thing
  
  """Value history and versions"""
  versions: [Value!]!
  previousVersion: Value
  nextVersion: Value
}

"""
Union type for typed value content supporting all data types
"""
union ValueContent = 
    StringValue 
  | IntegerValue 
  | DecimalValue 
  | BooleanValue 
  | DateValue 
  | DateTimeValue 
  | JSONValue 
  | ReferenceValue

"""String value content"""
type StringValue {
  value: String!
  length: Int!
}

"""Integer value content"""
type IntegerValue {
  value: Int!
}

"""Decimal value content with precision"""
type DecimalValue {
  value: Decimal!
  precision: Int!
  scale: Int!
}

"""Boolean value content"""
type BooleanValue {
  value: Boolean!
}

"""Date value content (date only)"""
type DateValue {
  value: Date!
  formatted: String!
}

"""DateTime value content with timezone"""
type DateTimeValue {
  value: DateTime!
  timezone: String!
  formatted: String!
}

"""JSON value content with metadata"""
type JSONValue {
  value: JSON!
  size: Int!
  depth: Int!
  keys: [String!]!
}

"""Reference value content linking to another Thing"""
type ReferenceValue {
  thingId: UUID!
  thing: Thing
  isValid: Boolean!
}

"""Validation error information"""
type ValidationError {
  field: String!
  code: String!
  message: String!
  details: JSON
  rule: String
  actualValue: JSON
  expectedValue: JSON
}

"""Value history entry"""
type ValueHistory {
  valueId: UUID!
  version: Int!
  typedValue: ValueContent!
  effectiveFrom: DateTime!
  effectiveTo: DateTime
  createdBy: UUID!
  createdAt: DateTime!
  changeReason: String
}

"""Value validation result"""
type ValueValidationResult {
  isValid: Boolean!
  errors: [ValidationError!]!
  warnings: [ValidationWarning!]!
  dataType: AttributeDataType!
  parsedValue: JSON
}

"""Validation warning information"""
type ValidationWarning {
  field: String!
  code: String!
  message: String!
  details: JSON
}

"""Value statistics and analytics"""
type ValueStatistics {
  thingId: UUID!
  attributeId: UUID!
  dataType: AttributeDataType!
  totalCount: Int!
  validCount: Int!
  invalidCount: Int!
  versionCount: Int!
  firstCreated: DateTime!
  lastUpdated: DateTime!
  typedStatistics: JSON
}

"""Bulk operation result"""
type BulkValueResult {
  createdCount: Int!
  updatedCount: Int!
  errorCount: Int!
  values: [Value!]!
  errors: [BulkOperationError!]!
}

"""Bulk operation error"""
type BulkOperationError {
  attributeId: UUID!
  error: String!
  index: Int!
}
```

### Input Types

```graphql
"""Input for creating a new value"""
input CreateValueInput {
  """Thing that will own this value"""
  thingId: UUID!
  
  """Attribute definition for this value"""
  attributeId: UUID!
  
  """Assignment configuration to use"""
  assignmentId: UUID!
  
  """Value content (type determined by attribute data type)"""
  value: JSON!
  
  """Only validate without persisting"""
  validateOnly: Boolean = false
  
  """Reason for creating this value"""
  changeReason: String
}

"""Input for updating an existing value"""
input UpdateValueInput {
  """New value content"""
  value: JSON!
  
  """Only validate without persisting"""
  validateOnly: Boolean = false
  
  """Reason for this change"""
  changeReason: String
}

"""Input for bulk value operations"""
input BulkValueInput {
  """Thing that will own all values"""
  thingId: UUID!
  
  """Map of attribute ID to value content"""
  values: JSON! # Map<UUID, JSON>
  
  """Reason for bulk operation"""
  changeReason: String
}

"""Filter options for value queries"""
input ValueFilter {
  """Filter by specific thing"""
  thingId: UUID
  
  """Filter by multiple things"""
  thingIds: [UUID!]
  
  """Filter by specific attribute"""
  attributeId: UUID
  
  """Filter by multiple attributes"""
  attributeIds: [UUID!]
  
  """Filter by assignment"""
  assignmentId: UUID
  
  """Filter by data type"""
  dataType: AttributeDataType
  
  """Filter by validation status"""
  isValid: Boolean
  
  """Only return current versions"""
  isCurrentOnly: Boolean = true
  
  """Data type specific filters"""
  stringValue: StringFilter
  integerValue: IntegerFilter
  decimalValue: DecimalFilter
  booleanValue: Boolean
  dateValue: DateFilter
  dateTimeValue: DateTimeFilter
  jsonValue: JSONFilter
  referenceValue: UUID
  
  """Temporal filters"""
  effectiveAt: DateTime
  effectiveFrom: DateTime
  effectiveTo: DateTime
  createdAfter: DateTime
  createdBefore: DateTime
  updatedAfter: DateTime
  updatedBefore: DateTime
  
  """Version and audit filters"""
  version: Int
  createdBy: UUID
  updatedBy: UUID
}

"""String value filtering options"""
input StringFilter {
  equals: String
  contains: String
  startsWith: String
  endsWith: String
  regex: String
  length: IntegerFilter
  fullTextSearch: String
}

"""Integer value filtering options"""
input IntegerFilter {
  equals: Int
  notEquals: Int
  greaterThan: Int
  greaterOrEqual: Int
  lessThan: Int
  lessOrEqual: Int
  between: IntegerRange
  in: [Int!]
}

"""Decimal value filtering options"""
input DecimalFilter {
  equals: Decimal
  greaterThan: Decimal
  greaterOrEqual: Decimal
  lessThan: Decimal
  lessOrEqual: Decimal
  between: DecimalRange
}

"""Date value filtering options"""
input DateFilter {
  equals: Date
  before: Date
  after: Date
  between: DateRange
}

"""DateTime value filtering options"""
input DateTimeFilter {
  equals: DateTime
  before: DateTime
  after: DateTime
  between: DateTimeRange
}

"""JSON value filtering options"""
input JSONFilter {
  """JSON path expression"""
  path: String!
  
  """Filter operator (equals, contains, exists, etc.)"""
  operator: JSONFilterOperator!
  
  """Value to compare against"""
  value: JSON
  
  """Expected data type at path"""
  dataType: String
}

"""Range filter for integers"""
input IntegerRange {
  min: Int
  max: Int
}

"""Range filter for decimals"""
input DecimalRange {
  min: Decimal
  max: Decimal
}

"""Range filter for dates"""
input DateRange {
  start: Date
  end: Date
}

"""Range filter for datetimes"""
input DateTimeRange {
  start: DateTime
  end: DateTime
}

"""JSON filter operators"""
enum JSONFilterOperator {
  EQUALS
  NOT_EQUALS
  CONTAINS
  NOT_CONTAINS
  EXISTS
  NOT_EXISTS
  GREATER_THAN
  LESS_THAN
  STARTS_WITH
  ENDS_WITH
  REGEX
}
```

### Query Operations

```graphql
type Query {
  """Get a specific value by ID"""
  value(
    """Value ID"""
    id: UUID!
    
    """Include version history"""
    includeVersions: Boolean = false
    
    """Include resolved relationships"""
    includeRelationships: Boolean = false
  ): Value

  """List values with filtering and pagination"""
  values(
    """Filtering options"""
    filter: ValueFilter
    
    """Sorting options"""
    orderBy: [ValueOrderBy!] = [{ field: CREATED_AT, direction: DESC }]
    
    """Pagination limit"""
    limit: Int = 50
    
    """Pagination offset"""
    offset: Int = 0
    
    """Include resolved relationships"""
    includeRelationships: Boolean = false
  ): ValueConnection!

  """Get current values for a specific Thing"""
  thingValues(
    """Thing ID"""
    thingId: UUID!
    
    """Optional attribute filtering"""
    attributeIds: [UUID!]
    
    """Include invalid values"""
    includeInvalid: Boolean = false
    
    """Include resolved relationships"""
    includeRelationships: Boolean = true
  ): [Value!]!

  """Get values for a specific Attribute across multiple Things"""
  attributeValues(
    """Attribute ID"""
    attributeId: UUID!
    
    """Optional thing filtering"""
    thingIds: [UUID!]
    
    """Value filtering"""
    valueFilter: JSON
    
    """Pagination limit"""
    limit: Int = 100
    
    """Pagination offset"""
    offset: Int = 0
  ): ValueConnection!

  """Get value history for a specific Thing-Attribute combination"""
  valueHistory(
    """Thing ID"""
    thingId: UUID!
    
    """Attribute ID"""
    attributeId: UUID!
    
    """Include only major versions"""
    majorVersionsOnly: Boolean = false
    
    """Pagination limit"""
    limit: Int = 20
  ): [ValueHistory!]!

  """Search values using full-text search"""
  searchValues(
    """Search query"""
    query: String!
    
    """Data types to search"""
    dataTypes: [AttributeDataType!] = [STRING, JSON]
    
    """Additional filtering"""
    filter: ValueFilter
    
    """Pagination limit"""
    limit: Int = 50
    
    """Pagination offset"""
    offset: Int = 0
  ): ValueConnection!

  """Get value statistics and analytics"""
  valueStatistics(
    """Thing ID for specific statistics"""
    thingId: UUID
    
    """Attribute ID for specific statistics"""
    attributeId: UUID
    
    """Time range for statistics"""
    dateRange: DateRange
  ): [ValueStatistics!]!

  """Validate a value without persisting"""
  validateValue(
    """Thing ID"""
    thingId: UUID!
    
    """Attribute ID"""
    attributeId: UUID!
    
    """Assignment ID"""
    assignmentId: UUID!
    
    """Value to validate"""
    value: JSON!
  ): ValueValidationResult!
}

"""Value connection for paginated results"""
type ValueConnection {
  nodes: [Value!]!
  totalCount: Int!
  pageInfo: PageInfo!
}

"""Page information for pagination"""
type PageInfo {
  hasNextPage: Boolean!
  hasPreviousPage: Boolean!
  startCursor: String
  endCursor: String
}

"""Value ordering options"""
input ValueOrderBy {
  field: ValueOrderField!
  direction: OrderDirection!
}

"""Fields available for value ordering"""
enum ValueOrderField {
  CREATED_AT
  UPDATED_AT
  EFFECTIVE_FROM
  VERSION
  DATA_TYPE
  IS_VALID
}

"""Ordering directions"""
enum OrderDirection {
  ASC
  DESC
}
```

### Mutation Operations

```graphql
type Mutation {
  """Create a new value"""
  createValue(
    """Value creation input"""
    input: CreateValueInput!
  ): CreateValuePayload!

  """Update an existing value"""
  updateValue(
    """Value ID to update"""
    id: UUID!
    
    """Update input"""
    input: UpdateValueInput!
  ): UpdateValuePayload!

  """Create or update multiple values in a single transaction"""
  bulkUpsertValues(
    """Bulk operation input"""
    input: BulkValueInput!
  ): BulkValuePayload!

  """Soft delete a value (marks as inactive)"""
  deleteValue(
    """Value ID to delete"""
    id: UUID!
    
    """Reason for deletion"""
    changeReason: String
  ): DeleteValuePayload!

  """Restore a specific version of a value as current"""
  restoreValueVersion(
    """Value ID to restore"""
    valueId: UUID!
    
    """Version number to restore"""
    version: Int!
    
    """Reason for restoration"""
    changeReason: String
  ): RestoreValuePayload!

  """Revalidate values based on updated rules"""
  revalidateValues(
    """Thing ID for specific revalidation"""
    thingId: UUID
    
    """Attribute ID for specific revalidation"""
    attributeId: UUID
    
    """Assignment ID for specific revalidation"""
    assignmentId: UUID
    
    """Force revalidation even for valid values"""
    forceRevalidation: Boolean = false
  ): RevalidationPayload!
}

"""Create value operation result"""
type CreateValuePayload {
  value: Value
  errors: [UserError!]!
}

"""Update value operation result"""
type UpdateValuePayload {
  value: Value
  errors: [UserError!]!
}

"""Bulk value operation result"""
type BulkValuePayload {
  result: BulkValueResult
  errors: [UserError!]!
}

"""Delete value operation result"""
type DeleteValuePayload {
  deletedValueId: UUID
  errors: [UserError!]!
}

"""Restore value operation result"""
type RestoreValuePayload {
  value: Value
  errors: [UserError!]!
}

"""Revalidation operation result"""
type RevalidationPayload {
  processedCount: Int!
  validatedCount: Int!
  errorCount: Int!
  errors: [UserError!]!
}

"""User-facing error information"""
type UserError {
  field: String
  message: String!
  code: String!
  details: JSON
}
```

### Subscription Operations

```graphql
type Subscription {
  """Subscribe to value changes for a specific Thing"""
  thingValueChanged(
    """Thing ID to monitor"""
    thingId: UUID!
    
    """Optional attribute filtering"""
    attributeIds: [UUID!]
  ): ValueChangeEvent!

  """Subscribe to value changes for a specific Attribute"""
  attributeValueChanged(
    """Attribute ID to monitor"""
    attributeId: UUID!
    
    """Optional thing filtering"""
    thingIds: [UUID!]
  ): ValueChangeEvent!

  """Subscribe to validation status changes"""
  valueValidationChanged(
    """Thing ID to monitor"""
    thingId: UUID
    
    """Attribute ID to monitor"""
    attributeId: UUID
  ): ValidationChangeEvent!

  """Subscribe to bulk operation progress"""
  bulkOperationProgress(
    """Operation ID to monitor"""
    operationId: UUID!
  ): BulkOperationProgress!
}

"""Value change event information"""
type ValueChangeEvent {
  operation: ValueOperation!
  value: Value!
  previousValue: Value
  changeReason: String
  timestamp: DateTime!
}

"""Value operation types"""
enum ValueOperation {
  CREATED
  UPDATED
  DELETED
  VERSIONED
  RESTORED
}

"""Validation change event information"""
type ValidationChangeEvent {
  thingId: UUID!
  attributeId: UUID!
  isValid: Boolean!
  previousValidationStatus: Boolean
  validationErrors: [ValidationError!]!
  timestamp: DateTime!
}

"""Bulk operation progress information"""
type BulkOperationProgress {
  operationId: UUID!
  totalOperations: Int!
  completedOperations: Int!
  successfulOperations: Int!
  failedOperations: Int!
  currentOperation: String
  estimatedTimeRemaining: Int
}
```

## Resolver Implementation Examples

### Query Resolvers

```go
// ValueQueryResolver implements GraphQL query operations for values
type ValueQueryResolver struct {
    service *service.ValueService
    logger  *zap.Logger
}

func (r *ValueQueryResolver) Value(ctx context.Context, args struct {
    ID                   string
    IncludeVersions      *bool
    IncludeRelationships *bool
}) (*ValueResolver, error) {
    tenant := auth.GetTenantFromContext(ctx)
    valueID, err := uuid.Parse(args.ID)
    if err != nil {
        return nil, fmt.Errorf("invalid value ID: %w", err)
    }

    value, err := r.service.GetByID(ctx, tenant, valueID)
    if err != nil {
        return nil, fmt.Errorf("failed to get value: %w", err)
    }
    if value == nil {
        return nil, nil
    }

    resolver := &ValueResolver{value: value, service: r.service}
    
    if args.IncludeVersions != nil && *args.IncludeVersions {
        versions, err := r.service.GetValueVersions(ctx, tenant, value.ThingID, value.AttributeID)
        if err != nil {
            r.logger.Warn("Failed to load value versions", 
                zap.String("value_id", valueID.String()), zap.Error(err))
        } else {
            resolver.versions = versions
        }
    }

    if args.IncludeRelationships != nil && *args.IncludeRelationships {
        if err := resolver.loadRelationships(ctx, tenant); err != nil {
            r.logger.Warn("Failed to load value relationships",
                zap.String("value_id", valueID.String()), zap.Error(err))
        }
    }

    return resolver, nil
}

func (r *ValueQueryResolver) Values(ctx context.Context, args struct {
    Filter               *domain.ValueFilter
    OrderBy              *[]*ValueOrderBy
    Limit                *int32
    Offset               *int32
    IncludeRelationships *bool
}) (*ValueConnectionResolver, error) {
    tenant := auth.GetTenantFromContext(ctx)
    
    limit := 50
    if args.Limit != nil && *args.Limit > 0 && *args.Limit <= 1000 {
        limit = int(*args.Limit)
    }
    
    offset := 0
    if args.Offset != nil && *args.Offset >= 0 {
        offset = int(*args.Offset)
    }

    // Convert GraphQL filter to domain filter
    filter := convertValueFilter(args.Filter)
    
    // Convert GraphQL ordering to domain ordering
    ordering := convertValueOrdering(args.OrderBy)

    values, totalCount, err := r.service.List(ctx, tenant, filter, ordering, limit, offset)
    if err != nil {
        return nil, fmt.Errorf("failed to list values: %w", err)
    }

    // Create resolvers for each value
    valueResolvers := make([]*ValueResolver, len(values))
    for i, value := range values {
        valueResolvers[i] = &ValueResolver{value: value, service: r.service}
        
        if args.IncludeRelationships != nil && *args.IncludeRelationships {
            if err := valueResolvers[i].loadRelationships(ctx, tenant); err != nil {
                r.logger.Warn("Failed to load relationships for value",
                    zap.String("value_id", value.ID.String()), zap.Error(err))
            }
        }
    }

    return &ValueConnectionResolver{
        nodes:      valueResolvers,
        totalCount: totalCount,
        limit:      limit,
        offset:     offset,
    }, nil
}

func (r *ValueQueryResolver) ThingValues(ctx context.Context, args struct {
    ThingID              string
    AttributeIDs         *[]string
    IncludeInvalid       *bool
    IncludeRelationships *bool
}) ([]*ValueResolver, error) {
    tenant := auth.GetTenantFromContext(ctx)
    thingID, err := uuid.Parse(args.ThingID)
    if err != nil {
        return nil, fmt.Errorf("invalid thing ID: %w", err)
    }

    // Build filter for thing values
    filter := &domain.ValueFilter{
        ThingID:       &thingID,
        IsCurrentOnly: true,
    }

    if args.AttributeIDs != nil {
        attributeIDs := make([]uuid.UUID, len(*args.AttributeIDs))
        for i, idStr := range *args.AttributeIDs {
            id, err := uuid.Parse(idStr)
            if err != nil {
                return nil, fmt.Errorf("invalid attribute ID at index %d: %w", i, err)
            }
            attributeIDs[i] = id
        }
        filter.AttributeIDs = attributeIDs
    }

    if args.IncludeInvalid == nil || !*args.IncludeInvalid {
        validFlag := true
        filter.IsValid = &validFlag
    }

    values, _, err := r.service.List(ctx, tenant, filter, nil, 1000, 0)
    if err != nil {
        return nil, fmt.Errorf("failed to get thing values: %w", err)
    }

    // Create resolvers
    resolvers := make([]*ValueResolver, len(values))
    for i, value := range values {
        resolvers[i] = &ValueResolver{value: value, service: r.service}
        
        if args.IncludeRelationships != nil && *args.IncludeRelationships {
            if err := resolvers[i].loadRelationships(ctx, tenant); err != nil {
                r.logger.Warn("Failed to load relationships for value",
                    zap.String("value_id", value.ID.String()), zap.Error(err))
            }
        }
    }

    return resolvers, nil
}

func (r *ValueQueryResolver) ValidateValue(ctx context.Context, args struct {
    ThingID      string
    AttributeID  string
    AssignmentID string
    Value        interface{}
}) (*ValueValidationResultResolver, error) {
    tenant := auth.GetTenantFromContext(ctx)
    
    thingID, err := uuid.Parse(args.ThingID)
    if err != nil {
        return nil, fmt.Errorf("invalid thing ID: %w", err)
    }
    
    attributeID, err := uuid.Parse(args.AttributeID)
    if err != nil {
        return nil, fmt.Errorf("invalid attribute ID: %w", err)
    }
    
    assignmentID, err := uuid.Parse(args.AssignmentID)
    if err != nil {
        return nil, fmt.Errorf("invalid assignment ID: %w", err)
    }

    input := &domain.CreateValueInput{
        ThingID:      thingID,
        AttributeID:  attributeID,
        AssignmentID: assignmentID,
        Value:        args.Value,
        ValidateOnly: true,
    }

    validationValue, err := r.service.CreateValue(ctx, tenant, input)
    if err != nil {
        return nil, fmt.Errorf("validation failed: %w", err)
    }

    // Extract validation result from the returned value
    var validationErrors []domain.ValidationError
    if len(validationValue.ValidationErrors) > 0 {
        if err := json.Unmarshal(validationValue.ValidationErrors, &validationErrors); err != nil {
            return nil, fmt.Errorf("failed to parse validation errors: %w", err)
        }
    }

    return &ValueValidationResultResolver{
        isValid:     validationValue.IsValid,
        errors:      validationErrors,
        warnings:    []domain.ValidationWarning{}, // Populate if warnings are available
        parsedValue: args.Value, // Return the parsed/normalized value if available
    }, nil
}
```

### Mutation Resolvers

```go
// ValueMutationResolver implements GraphQL mutation operations for values
type ValueMutationResolver struct {
    service *service.ValueService
    logger  *zap.Logger
}

func (r *ValueMutationResolver) CreateValue(ctx context.Context, args struct {
    Input *CreateValueInput
}) (*CreateValuePayloadResolver, error) {
    tenant := auth.GetTenantFromContext(ctx)
    
    // Convert GraphQL input to domain input
    domainInput := &domain.CreateValueInput{
        ThingID:      uuid.MustParse(args.Input.ThingID),
        AttributeID:  uuid.MustParse(args.Input.AttributeID),
        AssignmentID: uuid.MustParse(args.Input.AssignmentID),
        Value:        args.Input.Value,
        ValidateOnly: args.Input.ValidateOnly,
    }

    value, err := r.service.CreateValue(ctx, tenant, domainInput)
    if err != nil {
        return &CreateValuePayloadResolver{
            errors: convertToUserErrors(err),
        }, nil
    }

    return &CreateValuePayloadResolver{
        value: &ValueResolver{value: value, service: r.service},
    }, nil
}

func (r *ValueMutationResolver) UpdateValue(ctx context.Context, args struct {
    ID    string
    Input *UpdateValueInput
}) (*UpdateValuePayloadResolver, error) {
    tenant := auth.GetTenantFromContext(ctx)
    valueID, err := uuid.Parse(args.ID)
    if err != nil {
        return &UpdateValuePayloadResolver{
            errors: []*UserErrorResolver{{
                Code:    "INVALID_INPUT",
                Message: "Invalid value ID format",
                Field:   &[]string{"id"}[0],
            }},
        }, nil
    }

    domainInput := &domain.UpdateValueInput{
        Value:        args.Input.Value,
        ValidateOnly: args.Input.ValidateOnly,
        ChangeReason: args.Input.ChangeReason,
    }

    value, err := r.service.UpdateValue(ctx, tenant, valueID, domainInput)
    if err != nil {
        return &UpdateValuePayloadResolver{
            errors: convertToUserErrors(err),
        }, nil
    }

    return &UpdateValuePayloadResolver{
        value: &ValueResolver{value: value, service: r.service},
    }, nil
}

func (r *ValueMutationResolver) BulkUpsertValues(ctx context.Context, args struct {
    Input *BulkValueInput
}) (*BulkValuePayloadResolver, error) {
    tenant := auth.GetTenantFromContext(ctx)
    thingID, err := uuid.Parse(args.Input.ThingID)
    if err != nil {
        return &BulkValuePayloadResolver{
            errors: []*UserErrorResolver{{
                Code:    "INVALID_INPUT",
                Message: "Invalid thing ID format",
                Field:   &[]string{"thingId"}[0],
            }},
        }, nil
    }

    // Parse the values map from JSON
    valuesMap := make(map[uuid.UUID]interface{})
    if valuesJSON, ok := args.Input.Values.(map[string]interface{}); ok {
        for attrIDStr, value := range valuesJSON {
            attrID, err := uuid.Parse(attrIDStr)
            if err != nil {
                return &BulkValuePayloadResolver{
                    errors: []*UserErrorResolver{{
                        Code:    "INVALID_INPUT",
                        Message: fmt.Sprintf("Invalid attribute ID format: %s", attrIDStr),
                        Field:   &[]string{"values"}[0],
                    }},
                }, nil
            }
            valuesMap[attrID] = value
        }
    }

    domainInput := &domain.BulkValueInput{
        ThingID: thingID,
        Values:  valuesMap,
    }

    result, err := r.service.CreateBulkValues(ctx, tenant, domainInput)
    if err != nil {
        return &BulkValuePayloadResolver{
            errors: convertToUserErrors(err),
        }, nil
    }

    // Convert result to resolver
    valueResolvers := make([]*ValueResolver, len(result.Values))
    for i, value := range result.Values {
        valueResolvers[i] = &ValueResolver{value: value, service: r.service}
    }

    bulkResultResolver := &BulkValueResultResolver{
        createdCount: result.CreatedCount,
        updatedCount: result.UpdatedCount,
        errorCount:   result.ErrorCount,
        values:       valueResolvers,
    }

    return &BulkValuePayloadResolver{
        result: bulkResultResolver,
    }, nil
}
```

This comprehensive GraphQL API specification provides full CRUD operations, advanced filtering, bulk operations, validation, and real-time subscriptions for the Value entity while maintaining performance and data integrity standards.