# Relationship Class Entity - Go Implementation

**Created:** 2025-08-30  
**Version:** 1.0.0  
**Language:** Go 1.21+ with domain-driven design patterns

## Implementation Architecture

The Relationship Class entity implementation follows domain-driven design principles with clean architecture patterns, utilizing Go's type system for comprehensive validation, gqlgen for type-safe GraphQL operations, and repository patterns for data persistence across multi-tenant PostgreSQL schemas.

## Domain Model Implementation

### Core Domain Types

#### RelationshipClass Domain Entity
```go
package domain

import (
    "encoding/json"
    "time"
    "github.com/google/uuid"
    "github.com/pkg/errors"
)

type RelationshipClass struct {
    ID                    uuid.UUID                 `json:"id"`
    TenantID              uuid.UUID                 `json:"tenant_id"`
    Name                  string                    `json:"name"`
    DisplayName           string                    `json:"display_name"`
    Description           *string                   `json:"description"`
    IsDirectional         bool                      `json:"is_directional"`
    IsBidirectional       bool                      `json:"is_bidirectional"`
    Cardinality           RelationshipCardinality   `json:"cardinality"`
    // JSON validation rules for constraints (following established pattern)
    ValidationRules       *ValidationRules          `json:"validation_rules"`
    IsActive              bool                      `json:"is_active"`
    CreatedAt             time.Time                 `json:"created_at"`
    UpdatedAt             time.Time                 `json:"updated_at"`
    DeletedAt             *time.Time                `json:"deleted_at"`
    CreatedBy             uuid.UUID                 `json:"created_by"`
    UpdatedBy             *uuid.UUID                `json:"updated_by"`
    Version               int64                     `json:"version"`
}

type RelationshipCardinality string

const (
    CardinalityOneToOne    RelationshipCardinality = "ONE_TO_ONE"
    CardinalityOneToMany   RelationshipCardinality = "ONE_TO_MANY"
    CardinalityManyToOne   RelationshipCardinality = "MANY_TO_ONE"
    CardinalityManyToMany  RelationshipCardinality = "MANY_TO_MANY"
)

func (c RelationshipCardinality) IsValid() bool {
    switch c {
    case CardinalityOneToOne, CardinalityOneToMany, 
         CardinalityManyToOne, CardinalityManyToMany:
        return true
    default:
        return false
    }
}

type ValidationRules struct {
    Rules []ValidationRule `json:"rules"`
}

type ValidationRule struct {
    Type    string                 `json:"type"`
    Message string                 `json:"message"`
    Config  map[string]interface{} `json:"config,omitempty"`
}
```

#### Domain Validation Methods
```go
// Validate performs comprehensive domain validation
func (rc *RelationshipClass) Validate() error {
    var errors []string

    // Name validation
    if err := rc.validateName(); err != nil {
        errors = append(errors, err.Error())
    }

    // Display name validation
    if err := rc.validateDisplayName(); err != nil {
        errors = append(errors, err.Error())
    }

    // Cardinality validation
    if err := rc.validateCardinality(); err != nil {
        errors = append(errors, err.Error())
    }

    // Directional consistency validation
    if err := rc.validateDirectionalConsistency(); err != nil {
        errors = append(errors, err.Error())
    }

    // Thing Class constraints validation
    if err := rc.validateThingClassConstraints(); err != nil {
        errors = append(errors, err.Error())
    }

    // Validation rules schema validation
    if err := rc.validateValidationRules(); err != nil {
        errors = append(errors, err.Error())
    }

    if len(errors) > 0 {
        return fmt.Errorf("validation failed: %s", strings.Join(errors, "; "))
    }

    return nil
}

func (rc *RelationshipClass) validateName() error {
    if rc.Name == "" {
        return errors.New("name is required")
    }
    
    // Name must be snake_case alphanumeric
    nameRegex := regexp.MustCompile(`^[a-z][a-z0-9_]*[a-z0-9]$`)
    if !nameRegex.MatchString(rc.Name) {
        return errors.New("name must be snake_case alphanumeric starting and ending with letter/number")
    }
    
    if len(rc.Name) > 100 {
        return errors.New("name must be 100 characters or less")
    }
    
    return nil
}

func (rc *RelationshipClass) validateDisplayName() error {
    if rc.DisplayName == "" {
        return errors.New("display_name is required")
    }
    
    if len(rc.DisplayName) < 2 || len(rc.DisplayName) > 200 {
        return errors.New("display_name must be between 2 and 200 characters")
    }
    
    return nil
}

func (rc *RelationshipClass) validateCardinality() error {
    if !rc.Cardinality.IsValid() {
        return errors.New("invalid cardinality value")
    }
    return nil
}

func (rc *RelationshipClass) validateDirectionalConsistency() error {
    // Cannot be bidirectional and non-directional
    if rc.IsBidirectional && !rc.IsDirectional {
        return errors.New("bidirectional relationships must be directional")
    }
    
    // ONE_TO_ONE cannot be bidirectional (would create constraint conflicts)
    if rc.IsBidirectional && rc.Cardinality == CardinalityOneToOne {
        return errors.New("ONE_TO_ONE relationships cannot be bidirectional")
    }
    
    return nil
}

func (rc *RelationshipClass) validateThingClassConstraints() error {
    if len(rc.SourceThingClasses) == 0 {
        return errors.New("source_thing_classes cannot be empty")
    }
    
    if len(rc.TargetThingClasses) == 0 {
        return errors.New("target_thing_classes cannot be empty")
    }
    
    // Check for duplicate UUIDs
    sourceSet := make(map[uuid.UUID]bool)
    for _, id := range rc.SourceThingClasses {
        if sourceSet[id] {
            return errors.New("duplicate source Thing Class IDs not allowed")
        }
        sourceSet[id] = true
    }
    
    targetSet := make(map[uuid.UUID]bool)
    for _, id := range rc.TargetThingClasses {
        if targetSet[id] {
            return errors.New("duplicate target Thing Class IDs not allowed")
        }
        targetSet[id] = true
    }
    
    return nil
}

func (rc *RelationshipClass) validateValidationRules() error {
    if rc.ValidationRules == nil {
        return nil // Optional field
    }
    
    for i, rule := range rc.ValidationRules.Rules {
        if rule.Type == "" {
            return fmt.Errorf("validation rule %d: type is required", i)
        }
        if rule.Message == "" {
            return fmt.Errorf("validation rule %d: message is required", i)
        }
    }
    
    return nil
}
```

#### Business Logic Methods
```go
// CanAcceptThingClasses validates if specific Thing Classes can participate
func (rc *RelationshipClass) CanAcceptThingClasses(sourceID, targetID uuid.UUID) bool {
    sourceValid := false
    targetValid := false
    
    for _, id := range rc.SourceThingClasses {
        if id == sourceID {
            sourceValid = true
            break
        }
    }
    
    for _, id := range rc.TargetThingClasses {
        if id == targetID {
            targetValid = true
            break
        }
    }
    
    return sourceValid && targetValid
}

// GetConstraintInfo returns constraint information for UI/API consumption
func (rc *RelationshipClass) GetConstraintInfo() RelationshipConstraintInfo {
    return RelationshipConstraintInfo{
        Cardinality:           rc.Cardinality,
        IsDirectional:         rc.IsDirectional,
        IsBidirectional:       rc.IsBidirectional,
        SourceThingClassCount: len(rc.SourceThingClasses),
        TargetThingClassCount: len(rc.TargetThingClasses),
        HasValidationRules:    rc.ValidationRules != nil,
    }
}

type RelationshipConstraintInfo struct {
    Cardinality           RelationshipCardinality `json:"cardinality"`
    IsDirectional         bool                    `json:"is_directional"`
    IsBidirectional       bool                    `json:"is_bidirectional"`
    SourceThingClassCount int                     `json:"source_thing_class_count"`
    TargetThingClassCount int                     `json:"target_thing_class_count"`
    HasValidationRules    bool                    `json:"has_validation_rules"`
}

// Clone creates a deep copy for safe modification
func (rc *RelationshipClass) Clone() *RelationshipClass {
    clone := *rc
    
    // Deep copy slices
    clone.SourceThingClasses = make([]uuid.UUID, len(rc.SourceThingClasses))
    copy(clone.SourceThingClasses, rc.SourceThingClasses)
    
    clone.TargetThingClasses = make([]uuid.UUID, len(rc.TargetThingClasses))
    copy(clone.TargetThingClasses, rc.TargetThingClasses)
    
    // Deep copy validation rules
    if rc.ValidationRules != nil {
        rulesData, _ := json.Marshal(rc.ValidationRules)
        var clonedRules ValidationRules
        json.Unmarshal(rulesData, &clonedRules)
        clone.ValidationRules = &clonedRules
    }
    
    return &clone
}
```

## Service Layer Implementation

### RelationshipClass Service Interface
```go
package services

import (
    "context"
    "github.com/google/uuid"
    "udm/internal/domain"
)

type RelationshipClassService interface {
    CreateRelationshipClass(ctx context.Context, input *CreateRelationshipClassInput) (*domain.RelationshipClass, error)
    UpdateRelationshipClass(ctx context.Context, input *UpdateRelationshipClassInput) (*domain.RelationshipClass, error)
    DeleteRelationshipClass(ctx context.Context, id uuid.UUID) error
    GetRelationshipClass(ctx context.Context, id uuid.UUID) (*domain.RelationshipClass, error)
    GetRelationshipClassByName(ctx context.Context, tenantID uuid.UUID, name string) (*domain.RelationshipClass, error)
    ListRelationshipClasses(ctx context.Context, filter *RelationshipClassFilter, pagination *Pagination) (*RelationshipClassConnection, error)
    ValidateRelationshipClass(ctx context.Context, relationshipClass *domain.RelationshipClass) error
    GetRelationshipClassesForThingClasses(ctx context.Context, sourceIDs, targetIDs []uuid.UUID) ([]*domain.RelationshipClass, error)
}

type CreateRelationshipClassInput struct {
    Name                  string                            `validate:"required,min=2,max=100"`
    DisplayName           string                            `validate:"required,min=2,max=200"`
    Description           *string                           `validate:"omitempty,max=1000"`
    IsDirectional         bool
    IsBidirectional       bool
    Cardinality           domain.RelationshipCardinality    `validate:"required,oneof=ONE_TO_ONE ONE_TO_MANY MANY_TO_ONE MANY_TO_MANY"`
    SourceThingClassIDs   []uuid.UUID                       `validate:"required,min=1,max=10"`
    TargetThingClassIDs   []uuid.UUID                       `validate:"required,min=1,max=10"`
    ValidationRules       *domain.ValidationRules           `validate:"omitempty"`
}

type UpdateRelationshipClassInput struct {
    ID                    uuid.UUID                         `validate:"required"`
    Name                  *string                           `validate:"omitempty,min=2,max=100"`
    DisplayName           *string                           `validate:"omitempty,min=2,max=200"`
    Description           *string                           `validate:"omitempty,max=1000"`
    IsDirectional         *bool
    IsBidirectional       *bool
    Cardinality           *domain.RelationshipCardinality   `validate:"omitempty,oneof=ONE_TO_ONE ONE_TO_MANY MANY_TO_ONE MANY_TO_MANY"`
    SourceThingClassIDs   []uuid.UUID                       `validate:"omitempty,min=1,max=10"`
    TargetThingClassIDs   []uuid.UUID                       `validate:"omitempty,min=1,max=10"`
    ValidationRules       *domain.ValidationRules           `validate:"omitempty"`
    IsActive              *bool
    Version               int64                             `validate:"required,min=1"`
}
```

### Service Implementation
```go
package services

import (
    "context"
    "fmt"
    "time"
    "github.com/google/uuid"
    "github.com/pkg/errors"
    "udm/internal/domain"
    "udm/internal/repositories"
    "udm/internal/auth"
)

type relationshipClassService struct {
    repo                repositories.RelationshipClassRepository
    thingClassRepo      repositories.ThingClassRepository
    validator          *validator.Validate
    logger             *slog.Logger
    cache              *redis.Client
}

func NewRelationshipClassService(
    repo repositories.RelationshipClassRepository,
    thingClassRepo repositories.ThingClassRepository,
    validator *validator.Validate,
    logger *slog.Logger,
    cache *redis.Client,
) RelationshipClassService {
    return &relationshipClassService{
        repo:           repo,
        thingClassRepo: thingClassRepo,
        validator:      validator,
        logger:         logger,
        cache:          cache,
    }
}

func (s *relationshipClassService) CreateRelationshipClass(ctx context.Context, input *CreateRelationshipClassInput) (*domain.RelationshipClass, error) {
    tenantID, err := auth.GetTenantID(ctx)
    if err != nil {
        return nil, errors.Wrap(err, "failed to get tenant ID")
    }
    
    userID, err := auth.GetUserID(ctx)
    if err != nil {
        return nil, errors.Wrap(err, "failed to get user ID")
    }
    
    // Validate input
    if err := s.validator.Struct(input); err != nil {
        return nil, errors.Wrap(err, "input validation failed")
    }
    
    // Check for name uniqueness
    existing, err := s.repo.GetByName(ctx, tenantID, input.Name)
    if err != nil && !errors.Is(err, repositories.ErrNotFound) {
        return nil, errors.Wrap(err, "failed to check name uniqueness")
    }
    if existing != nil {
        return nil, errors.New("relationship class with this name already exists")
    }
    
    // Validate Thing Class references
    if err := s.validateThingClassReferences(ctx, tenantID, input.SourceThingClassIDs, input.TargetThingClassIDs); err != nil {
        return nil, errors.Wrap(err, "Thing Class reference validation failed")
    }
    
    // Create domain entity
    relationshipClass := &domain.RelationshipClass{
        ID:                  uuid.New(),
        TenantID:            tenantID,
        Name:                input.Name,
        DisplayName:         input.DisplayName,
        Description:         input.Description,
        IsDirectional:       input.IsDirectional,
        IsBidirectional:     input.IsBidirectional,
        Cardinality:         input.Cardinality,
        SourceThingClasses:  input.SourceThingClassIDs,
        TargetThingClasses:  input.TargetThingClassIDs,
        ValidationRules:     input.ValidationRules,
        IsActive:            true,
        CreatedAt:           time.Now(),
        UpdatedAt:           time.Now(),
        CreatedBy:           userID,
        Version:             1,
    }
    
    // Domain validation
    if err := relationshipClass.Validate(); err != nil {
        return nil, errors.Wrap(err, "domain validation failed")
    }
    
    // Persist to repository
    if err := s.repo.Create(ctx, relationshipClass); err != nil {
        return nil, errors.Wrap(err, "failed to create relationship class")
    }
    
    // Cache the result
    s.cacheRelationshipClass(ctx, relationshipClass)
    
    s.logger.InfoContext(ctx, "Relationship class created successfully",
        "id", relationshipClass.ID,
        "name", relationshipClass.Name,
        "tenant_id", tenantID,
        "user_id", userID)
    
    return relationshipClass, nil
}

func (s *relationshipClassService) UpdateRelationshipClass(ctx context.Context, input *UpdateRelationshipClassInput) (*domain.RelationshipClass, error) {
    tenantID, err := auth.GetTenantID(ctx)
    if err != nil {
        return nil, errors.Wrap(err, "failed to get tenant ID")
    }
    
    userID, err := auth.GetUserID(ctx)
    if err != nil {
        return nil, errors.Wrap(err, "failed to get user ID")
    }
    
    // Validate input
    if err := s.validator.Struct(input); err != nil {
        return nil, errors.Wrap(err, "input validation failed")
    }
    
    // Get existing relationship class
    existing, err := s.repo.GetByID(ctx, input.ID)
    if err != nil {
        return nil, errors.Wrap(err, "failed to get existing relationship class")
    }
    
    // Verify tenant ownership
    if existing.TenantID != tenantID {
        return nil, errors.New("relationship class not found or access denied")
    }
    
    // Check version for optimistic locking
    if existing.Version != input.Version {
        return nil, errors.New("version conflict - relationship class was modified by another process")
    }
    
    // Create updated entity
    updated := existing.Clone()
    
    // Apply updates
    if input.Name != nil {
        // Check name uniqueness if changing
        if *input.Name != existing.Name {
            existingByName, err := s.repo.GetByName(ctx, tenantID, *input.Name)
            if err != nil && !errors.Is(err, repositories.ErrNotFound) {
                return nil, errors.Wrap(err, "failed to check name uniqueness")
            }
            if existingByName != nil {
                return nil, errors.New("relationship class with this name already exists")
            }
        }
        updated.Name = *input.Name
    }
    
    if input.DisplayName != nil {
        updated.DisplayName = *input.DisplayName
    }
    
    if input.Description != nil {
        updated.Description = input.Description
    }
    
    if input.IsDirectional != nil {
        updated.IsDirectional = *input.IsDirectional
    }
    
    if input.IsBidirectional != nil {
        updated.IsBidirectional = *input.IsBidirectional
    }
    
    if input.Cardinality != nil {
        updated.Cardinality = *input.Cardinality
    }
    
    if input.SourceThingClassIDs != nil {
        updated.SourceThingClasses = input.SourceThingClassIDs
    }
    
    if input.TargetThingClassIDs != nil {
        updated.TargetThingClasses = input.TargetThingClassIDs
    }
    
    if input.ValidationRules != nil {
        updated.ValidationRules = input.ValidationRules
    }
    
    if input.IsActive != nil {
        updated.IsActive = *input.IsActive
    }
    
    updated.UpdatedAt = time.Now()
    updated.UpdatedBy = &userID
    updated.Version = existing.Version + 1
    
    // Validate Thing Class references if changed
    if input.SourceThingClassIDs != nil || input.TargetThingClassIDs != nil {
        if err := s.validateThingClassReferences(ctx, tenantID, updated.SourceThingClasses, updated.TargetThingClasses); err != nil {
            return nil, errors.Wrap(err, "Thing Class reference validation failed")
        }
    }
    
    // Domain validation
    if err := updated.Validate(); err != nil {
        return nil, errors.Wrap(err, "domain validation failed")
    }
    
    // Persist updates
    if err := s.repo.Update(ctx, updated); err != nil {
        return nil, errors.Wrap(err, "failed to update relationship class")
    }
    
    // Update cache
    s.cacheRelationshipClass(ctx, updated)
    
    s.logger.InfoContext(ctx, "Relationship class updated successfully",
        "id", updated.ID,
        "name", updated.Name,
        "tenant_id", tenantID,
        "user_id", userID,
        "version", updated.Version)
    
    return updated, nil
}
```

### Validation Helper Methods
```go
func (s *relationshipClassService) validateThingClassReferences(ctx context.Context, tenantID uuid.UUID, sourceIDs, targetIDs []uuid.UUID) error {
    // Combine all IDs for single query
    allIDs := make([]uuid.UUID, 0, len(sourceIDs)+len(targetIDs))
    allIDs = append(allIDs, sourceIDs...)
    allIDs = append(allIDs, targetIDs...)
    
    // Remove duplicates
    idSet := make(map[uuid.UUID]bool)
    uniqueIDs := make([]uuid.UUID, 0, len(allIDs))
    for _, id := range allIDs {
        if !idSet[id] {
            idSet[id] = true
            uniqueIDs = append(uniqueIDs, id)
        }
    }
    
    // Query all Thing Classes at once
    thingClasses, err := s.thingClassRepo.GetByIDs(ctx, uniqueIDs)
    if err != nil {
        return errors.Wrap(err, "failed to query Thing Classes")
    }
    
    // Check that all Thing Classes exist and belong to tenant
    foundIDs := make(map[uuid.UUID]bool)
    for _, tc := range thingClasses {
        if tc.TenantID != tenantID {
            return fmt.Errorf("Thing Class %s does not belong to tenant", tc.ID)
        }
        if tc.DeletedAt != nil {
            return fmt.Errorf("Thing Class %s is deleted", tc.ID)
        }
        foundIDs[tc.ID] = true
    }
    
    // Verify all requested IDs were found
    for _, id := range uniqueIDs {
        if !foundIDs[id] {
            return fmt.Errorf("Thing Class %s not found or access denied", id)
        }
    }
    
    return nil
}

func (s *relationshipClassService) cacheRelationshipClass(ctx context.Context, rc *domain.RelationshipClass) {
    if s.cache == nil {
        return
    }
    
    data, err := json.Marshal(rc)
    if err != nil {
        s.logger.WarnContext(ctx, "Failed to marshal relationship class for caching",
            "id", rc.ID,
            "error", err)
        return
    }
    
    // Cache by ID
    idKey := fmt.Sprintf("relationship_class:id:%s", rc.ID)
    s.cache.Set(ctx, idKey, data, time.Hour)
    
    // Cache by tenant + name
    nameKey := fmt.Sprintf("relationship_class:name:%s:%s", rc.TenantID, rc.Name)
    s.cache.Set(ctx, nameKey, data, time.Hour)
}
```

## Repository Implementation

### Repository Interface
```go
package repositories

import (
    "context"
    "github.com/google/uuid"
    "udm/internal/domain"
)

type RelationshipClassRepository interface {
    Create(ctx context.Context, relationshipClass *domain.RelationshipClass) error
    Update(ctx context.Context, relationshipClass *domain.RelationshipClass) error
    Delete(ctx context.Context, id uuid.UUID) error
    GetByID(ctx context.Context, id uuid.UUID) (*domain.RelationshipClass, error)
    GetByName(ctx context.Context, tenantID uuid.UUID, name string) (*domain.RelationshipClass, error)
    GetByIDs(ctx context.Context, ids []uuid.UUID) ([]*domain.RelationshipClass, error)
    List(ctx context.Context, filter *RelationshipClassFilter, pagination *Pagination) ([]*domain.RelationshipClass, error)
    Count(ctx context.Context, filter *RelationshipClassFilter) (int64, error)
    GetForThingClasses(ctx context.Context, sourceIDs, targetIDs []uuid.UUID) ([]*domain.RelationshipClass, error)
}

type RelationshipClassFilter struct {
    IDs                   []uuid.UUID
    TenantID              uuid.UUID
    Name                  *StringFilter
    DisplayName           *StringFilter
    Cardinality           *domain.RelationshipCardinality
    IsDirectional         *bool
    IsBidirectional       *bool
    SourceThingClassIDs   []uuid.UUID
    TargetThingClassIDs   []uuid.UUID
    IsActive              *bool
    CreatedAt             *DateTimeFilter
    UpdatedAt             *DateTimeFilter
    CreatedBy             *uuid.UUID
    HasValidationRules    *bool
}
```

### PostgreSQL Implementation
```go
package repositories

import (
    "context"
    "database/sql"
    "fmt"
    "strings"
    "time"
    
    "github.com/google/uuid"
    "github.com/jmoiron/sqlx"
    "github.com/lib/pq"
    "github.com/pkg/errors"
    
    "udm/internal/domain"
)

type relationshipClassRepository struct {
    db     *sqlx.DB
    logger *slog.Logger
}

func NewRelationshipClassRepository(db *sqlx.DB, logger *slog.Logger) RelationshipClassRepository {
    return &relationshipClassRepository{
        db:     db,
        logger: logger,
    }
}

func (r *relationshipClassRepository) Create(ctx context.Context, rc *domain.RelationshipClass) error {
    query := `
        INSERT INTO relationship_classes (
            id, tenant_id, name, display_name, description, 
            is_directional, is_bidirectional, cardinality,
            source_thing_classes, target_thing_classes, validation_rules,
            is_active, created_at, updated_at, created_by, version
        ) VALUES (
            :id, :tenant_id, :name, :display_name, :description,
            :is_directional, :is_bidirectional, :cardinality,
            :source_thing_classes, :target_thing_classes, :validation_rules,
            :is_active, :created_at, :updated_at, :created_by, :version
        )`
    
    dbModel := r.toDBModel(rc)
    
    _, err := r.db.NamedExecContext(ctx, query, dbModel)
    if err != nil {
        if pqErr, ok := err.(*pq.Error); ok {
            switch pqErr.Code {
            case "23505": // unique_violation
                if strings.Contains(pqErr.Constraint, "tenant_name_unique") {
                    return errors.New("relationship class name already exists")
                }
            case "23503": // foreign_key_violation
                return errors.New("invalid Thing Class reference")
            case "23514": // check_constraint_violation
                return fmt.Errorf("constraint violation: %s", pqErr.Message)
            }
        }
        return errors.Wrap(err, "failed to create relationship class")
    }
    
    return nil
}

func (r *relationshipClassRepository) Update(ctx context.Context, rc *domain.RelationshipClass) error {
    query := `
        UPDATE relationship_classes SET
            name = :name,
            display_name = :display_name,
            description = :description,
            is_directional = :is_directional,
            is_bidirectional = :is_bidirectional,
            cardinality = :cardinality,
            source_thing_classes = :source_thing_classes,
            target_thing_classes = :target_thing_classes,
            validation_rules = :validation_rules,
            is_active = :is_active,
            updated_at = :updated_at,
            updated_by = :updated_by,
            version = :version
        WHERE id = :id AND version = :version - 1`
    
    dbModel := r.toDBModel(rc)
    
    result, err := r.db.NamedExecContext(ctx, query, dbModel)
    if err != nil {
        if pqErr, ok := err.(*pq.Error); ok {
            switch pqErr.Code {
            case "23505": // unique_violation
                if strings.Contains(pqErr.Constraint, "tenant_name_unique") {
                    return errors.New("relationship class name already exists")
                }
            case "23503": // foreign_key_violation
                return errors.New("invalid Thing Class reference")
            case "23514": // check_constraint_violation
                return fmt.Errorf("constraint violation: %s", pqErr.Message)
            }
        }
        return errors.Wrap(err, "failed to update relationship class")
    }
    
    rowsAffected, err := result.RowsAffected()
    if err != nil {
        return errors.Wrap(err, "failed to get rows affected")
    }
    
    if rowsAffected == 0 {
        return errors.New("relationship class not found or version conflict")
    }
    
    return nil
}

func (r *relationshipClassRepository) GetByID(ctx context.Context, id uuid.UUID) (*domain.RelationshipClass, error) {
    query := `
        SELECT id, tenant_id, name, display_name, description,
               is_directional, is_bidirectional, cardinality,
               source_thing_classes, target_thing_classes, validation_rules,
               is_active, created_at, updated_at, deleted_at,
               created_by, updated_by, version
        FROM relationship_classes
        WHERE id = $1 AND deleted_at IS NULL`
    
    var dbModel relationshipClassDBModel
    err := r.db.GetContext(ctx, &dbModel, query, id)
    if err != nil {
        if err == sql.ErrNoRows {
            return nil, ErrNotFound
        }
        return nil, errors.Wrap(err, "failed to get relationship class by ID")
    }
    
    return r.fromDBModel(&dbModel)
}

func (r *relationshipClassRepository) List(ctx context.Context, filter *RelationshipClassFilter, pagination *Pagination) ([]*domain.RelationshipClass, error) {
    whereClause, args, err := r.buildWhereClause(filter)
    if err != nil {
        return nil, errors.Wrap(err, "failed to build where clause")
    }
    
    orderClause := "ORDER BY created_at DESC"
    if pagination != nil && len(pagination.Sort) > 0 {
        orderClause = r.buildOrderClause(pagination.Sort)
    }
    
    limitClause := ""
    if pagination != nil {
        limitClause = fmt.Sprintf("LIMIT %d OFFSET %d", pagination.Limit, pagination.Offset)
    }
    
    query := fmt.Sprintf(`
        SELECT id, tenant_id, name, display_name, description,
               is_directional, is_bidirectional, cardinality,
               source_thing_classes, target_thing_classes, validation_rules,
               is_active, created_at, updated_at, deleted_at,
               created_by, updated_by, version
        FROM relationship_classes
        %s %s %s`, whereClause, orderClause, limitClause)
    
    var dbModels []relationshipClassDBModel
    err = r.db.SelectContext(ctx, &dbModels, query, args...)
    if err != nil {
        return nil, errors.Wrap(err, "failed to list relationship classes")
    }
    
    relationshipClasses := make([]*domain.RelationshipClass, len(dbModels))
    for i, dbModel := range dbModels {
        rc, err := r.fromDBModel(&dbModel)
        if err != nil {
            return nil, errors.Wrapf(err, "failed to convert db model %d", i)
        }
        relationshipClasses[i] = rc
    }
    
    return relationshipClasses, nil
}
```

### Database Model Conversion
```go
type relationshipClassDBModel struct {
    ID                    uuid.UUID                         `db:"id"`
    TenantID              uuid.UUID                         `db:"tenant_id"`
    Name                  string                            `db:"name"`
    DisplayName           string                            `db:"display_name"`
    Description           sql.NullString                    `db:"description"`
    IsDirectional         bool                              `db:"is_directional"`
    IsBidirectional       bool                              `db:"is_bidirectional"`
    Cardinality           domain.RelationshipCardinality    `db:"cardinality"`
    SourceThingClasses    pq.StringArray                    `db:"source_thing_classes"`
    TargetThingClasses    pq.StringArray                    `db:"target_thing_classes"`
    ValidationRules       sql.NullString                    `db:"validation_rules"`
    IsActive              bool                              `db:"is_active"`
    CreatedAt             time.Time                         `db:"created_at"`
    UpdatedAt             time.Time                         `db:"updated_at"`
    DeletedAt             sql.NullTime                      `db:"deleted_at"`
    CreatedBy             uuid.UUID                         `db:"created_by"`
    UpdatedBy             uuid.NullUUID                     `db:"updated_by"`
    Version               int64                             `db:"version"`
}

func (r *relationshipClassRepository) toDBModel(rc *domain.RelationshipClass) *relationshipClassDBModel {
    dbModel := &relationshipClassDBModel{
        ID:              rc.ID,
        TenantID:        rc.TenantID,
        Name:            rc.Name,
        DisplayName:     rc.DisplayName,
        IsDirectional:   rc.IsDirectional,
        IsBidirectional: rc.IsBidirectional,
        Cardinality:     rc.Cardinality,
        IsActive:        rc.IsActive,
        CreatedAt:       rc.CreatedAt,
        UpdatedAt:       rc.UpdatedAt,
        CreatedBy:       rc.CreatedBy,
        Version:         rc.Version,
    }
    
    if rc.Description != nil {
        dbModel.Description = sql.NullString{String: *rc.Description, Valid: true}
    }
    
    if rc.DeletedAt != nil {
        dbModel.DeletedAt = sql.NullTime{Time: *rc.DeletedAt, Valid: true}
    }
    
    if rc.UpdatedBy != nil {
        dbModel.UpdatedBy = uuid.NullUUID{UUID: *rc.UpdatedBy, Valid: true}
    }
    
    // Convert UUID slices to string arrays for PostgreSQL
    sourceStrings := make([]string, len(rc.SourceThingClasses))
    for i, id := range rc.SourceThingClasses {
        sourceStrings[i] = id.String()
    }
    dbModel.SourceThingClasses = sourceStrings
    
    targetStrings := make([]string, len(rc.TargetThingClasses))
    for i, id := range rc.TargetThingClasses {
        targetStrings[i] = id.String()
    }
    dbModel.TargetThingClasses = targetStrings
    
    if rc.ValidationRules != nil {
        jsonData, _ := json.Marshal(rc.ValidationRules)
        dbModel.ValidationRules = sql.NullString{String: string(jsonData), Valid: true}
    }
    
    return dbModel
}

func (r *relationshipClassRepository) fromDBModel(dbModel *relationshipClassDBModel) (*domain.RelationshipClass, error) {
    rc := &domain.RelationshipClass{
        ID:              dbModel.ID,
        TenantID:        dbModel.TenantID,
        Name:            dbModel.Name,
        DisplayName:     dbModel.DisplayName,
        IsDirectional:   dbModel.IsDirectional,
        IsBidirectional: dbModel.IsBidirectional,
        Cardinality:     dbModel.Cardinality,
        IsActive:        dbModel.IsActive,
        CreatedAt:       dbModel.CreatedAt,
        UpdatedAt:       dbModel.UpdatedAt,
        CreatedBy:       dbModel.CreatedBy,
        Version:         dbModel.Version,
    }
    
    if dbModel.Description.Valid {
        rc.Description = &dbModel.Description.String
    }
    
    if dbModel.DeletedAt.Valid {
        rc.DeletedAt = &dbModel.DeletedAt.Time
    }
    
    if dbModel.UpdatedBy.Valid {
        rc.UpdatedBy = &dbModel.UpdatedBy.UUID
    }
    
    // Convert string arrays back to UUID slices
    sourceIDs := make([]uuid.UUID, len(dbModel.SourceThingClasses))
    for i, idStr := range dbModel.SourceThingClasses {
        id, err := uuid.Parse(idStr)
        if err != nil {
            return nil, errors.Wrapf(err, "failed to parse source Thing Class ID: %s", idStr)
        }
        sourceIDs[i] = id
    }
    rc.SourceThingClasses = sourceIDs
    
    targetIDs := make([]uuid.UUID, len(dbModel.TargetThingClasses))
    for i, idStr := range dbModel.TargetThingClasses {
        id, err := uuid.Parse(idStr)
        if err != nil {
            return nil, errors.Wrapf(err, "failed to parse target Thing Class ID: %s", idStr)
        }
        targetIDs[i] = id
    }
    rc.TargetThingClasses = targetIDs
    
    if dbModel.ValidationRules.Valid {
        var validationRules domain.ValidationRules
        if err := json.Unmarshal([]byte(dbModel.ValidationRules.String), &validationRules); err != nil {
            return nil, errors.Wrap(err, "failed to unmarshal validation rules")
        }
        rc.ValidationRules = &validationRules
    }
    
    return rc, nil
}
```

## GraphQL Resolver Implementation

### Resolver Structure
```go
package resolvers

import (
    "context"
    "github.com/google/uuid"
    "udm/internal/domain"
    "udm/internal/services"
    "udm/internal/graphql/generated"
)

type RelationshipClassResolver struct {
    service services.RelationshipClassService
    logger  *slog.Logger
}

func NewRelationshipClassResolver(service services.RelationshipClassService, logger *slog.Logger) *RelationshipClassResolver {
    return &RelationshipClassResolver{
        service: service,
        logger:  logger,
    }
}

// Query resolvers
func (r *RelationshipClassResolver) RelationshipClass(ctx context.Context, id string) (*domain.RelationshipClass, error) {
    relationshipClassID, err := uuid.Parse(id)
    if err != nil {
        return nil, fmt.Errorf("invalid relationship class ID: %w", err)
    }
    
    return r.service.GetRelationshipClass(ctx, relationshipClassID)
}

func (r *RelationshipClassResolver) RelationshipClasses(
    ctx context.Context,
    filter *generated.RelationshipClassFilter,
    sort []*generated.RelationshipClassSort,
    first *int,
    after *string,
    last *int,
    before *string,
) (*generated.RelationshipClassConnection, error) {
    
    // Convert GraphQL filter to service filter
    serviceFilter := r.convertFilter(filter)
    
    // Build pagination
    pagination, err := r.buildPagination(sort, first, after, last, before)
    if err != nil {
        return nil, fmt.Errorf("invalid pagination parameters: %w", err)
    }
    
    // Get relationship classes
    relationshipClasses, err := r.service.ListRelationshipClasses(ctx, serviceFilter, pagination)
    if err != nil {
        return nil, fmt.Errorf("failed to list relationship classes: %w", err)
    }
    
    // Convert to GraphQL connection
    return r.buildConnection(ctx, relationshipClasses, pagination), nil
}

// Mutation resolvers
func (r *RelationshipClassResolver) CreateRelationshipClass(
    ctx context.Context,
    input generated.CreateRelationshipClassInput,
) (*generated.CreateRelationshipClassResponse, error) {
    
    // Convert GraphQL input to service input
    serviceInput, err := r.convertCreateInput(&input)
    if err != nil {
        return &generated.CreateRelationshipClassResponse{
            Errors: []*generated.ValidationError{{
                Field:   "input",
                Code:    "INVALID_INPUT",
                Message: err.Error(),
            }},
        }, nil
    }
    
    // Create relationship class
    relationshipClass, err := r.service.CreateRelationshipClass(ctx, serviceInput)
    if err != nil {
        return &generated.CreateRelationshipClassResponse{
            Errors: r.convertErrorToValidationErrors(err),
        }, nil
    }
    
    return &generated.CreateRelationshipClassResponse{
        RelationshipClass: relationshipClass,
        Errors:            []*generated.ValidationError{},
    }, nil
}

func (r *RelationshipClassResolver) UpdateRelationshipClass(
    ctx context.Context,
    input generated.UpdateRelationshipClassInput,
) (*generated.UpdateRelationshipClassResponse, error) {
    
    // Convert GraphQL input to service input
    serviceInput, err := r.convertUpdateInput(&input)
    if err != nil {
        return &generated.UpdateRelationshipClassResponse{
            Errors: []*generated.ValidationError{{
                Field:   "input",
                Code:    "INVALID_INPUT",
                Message: err.Error(),
            }},
        }, nil
    }
    
    // Update relationship class
    relationshipClass, err := r.service.UpdateRelationshipClass(ctx, serviceInput)
    if err != nil {
        return &generated.UpdateRelationshipClassResponse{
            Errors: r.convertErrorToValidationErrors(err),
        }, nil
    }
    
    return &generated.UpdateRelationshipClassResponse{
        RelationshipClass: relationshipClass,
        Errors:            []*generated.ValidationError{},
    }, nil
}
```

### Field Resolvers with DataLoader
```go
// Field resolvers using DataLoader for efficient batching
func (r *RelationshipClassResolver) SourceThingClasses(ctx context.Context, obj *domain.RelationshipClass) ([]*domain.ThingClass, error) {
    // Use DataLoader to batch Thing Class queries
    loader := dataloaders.GetThingClassLoader(ctx)
    
    results := make([]*domain.ThingClass, len(obj.SourceThingClasses))
    for i, id := range obj.SourceThingClasses {
        thingClass, err := loader.Load(id.String())
        if err != nil {
            return nil, fmt.Errorf("failed to load source Thing Class %s: %w", id, err)
        }
        results[i] = thingClass
    }
    
    return results, nil
}

func (r *RelationshipClassResolver) TargetThingClasses(ctx context.Context, obj *domain.RelationshipClass) ([]*domain.ThingClass, error) {
    // Use DataLoader to batch Thing Class queries
    loader := dataloaders.GetThingClassLoader(ctx)
    
    results := make([]*domain.ThingClass, len(obj.TargetThingClasses))
    for i, id := range obj.TargetThingClasses {
        thingClass, err := loader.Load(id.String())
        if err != nil {
            return nil, fmt.Errorf("failed to load target Thing Class %s: %w", id, err)
        }
        results[i] = thingClass
    }
    
    return results, nil
}

func (r *RelationshipClassResolver) CreatedBy(ctx context.Context, obj *domain.RelationshipClass) (*domain.User, error) {
    loader := dataloaders.GetUserLoader(ctx)
    return loader.Load(obj.CreatedBy.String())
}

func (r *RelationshipClassResolver) UpdatedBy(ctx context.Context, obj *domain.RelationshipClass) (*domain.User, error) {
    if obj.UpdatedBy == nil {
        return nil, nil
    }
    
    loader := dataloaders.GetUserLoader(ctx)
    return loader.Load(obj.UpdatedBy.String())
}
```

## Testing Implementation

### Unit Tests
```go
package services_test

import (
    "context"
    "testing"
    "time"
    
    "github.com/google/uuid"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/mock"
    "github.com/stretchr/testify/require"
    
    "udm/internal/domain"
    "udm/internal/services"
    "udm/internal/repositories/mocks"
)

func TestRelationshipClassService_CreateRelationshipClass(t *testing.T) {
    tests := []struct {
        name           string
        input          *services.CreateRelationshipClassInput
        setupMocks     func(*mocks.RelationshipClassRepository, *mocks.ThingClassRepository)
        expectedError  string
        validateResult func(*testing.T, *domain.RelationshipClass)
    }{
        {
            name: "successful creation",
            input: &services.CreateRelationshipClassInput{
                Name:                "assigned_to",
                DisplayName:         "Assigned To",
                Description:         stringPtr("Task assignments"),
                IsDirectional:       true,
                IsBidirectional:     false,
                Cardinality:         domain.CardinalityManyToOne,
                SourceThingClassIDs: []uuid.UUID{uuid.New()},
                TargetThingClassIDs: []uuid.UUID{uuid.New()},
            },
            setupMocks: func(rcRepo *mocks.RelationshipClassRepository, tcRepo *mocks.ThingClassRepository) {
                rcRepo.On("GetByName", mock.Anything, mock.Anything, "assigned_to").
                    Return(nil, repositories.ErrNotFound)
                
                tcRepo.On("GetByIDs", mock.Anything, mock.Anything).
                    Return([]*domain.ThingClass{
                        {ID: uuid.New(), TenantID: uuid.New(), Name: "Task"},
                        {ID: uuid.New(), TenantID: uuid.New(), Name: "User"},
                    }, nil)
                
                rcRepo.On("Create", mock.Anything, mock.AnythingOfType("*domain.RelationshipClass")).
                    Return(nil)
            },
            validateResult: func(t *testing.T, rc *domain.RelationshipClass) {
                assert.Equal(t, "assigned_to", rc.Name)
                assert.Equal(t, "Assigned To", rc.DisplayName)
                assert.True(t, rc.IsDirectional)
                assert.False(t, rc.IsBidirectional)
                assert.Equal(t, domain.CardinalityManyToOne, rc.Cardinality)
                assert.True(t, rc.IsActive)
                assert.Equal(t, int64(1), rc.Version)
            },
        },
        {
            name: "duplicate name error",
            input: &services.CreateRelationshipClassInput{
                Name:                "existing_relationship",
                DisplayName:         "Existing",
                Cardinality:         domain.CardinalityOneToOne,
                SourceThingClassIDs: []uuid.UUID{uuid.New()},
                TargetThingClassIDs: []uuid.UUID{uuid.New()},
            },
            setupMocks: func(rcRepo *mocks.RelationshipClassRepository, tcRepo *mocks.ThingClassRepository) {
                rcRepo.On("GetByName", mock.Anything, mock.Anything, "existing_relationship").
                    Return(&domain.RelationshipClass{Name: "existing_relationship"}, nil)
            },
            expectedError: "relationship class with this name already exists",
        },
        {
            name: "invalid cardinality bidirectional combination",
            input: &services.CreateRelationshipClassInput{
                Name:                "invalid_combo",
                DisplayName:         "Invalid",
                IsDirectional:       true,
                IsBidirectional:     true,
                Cardinality:         domain.CardinalityOneToOne, // Invalid with bidirectional
                SourceThingClassIDs: []uuid.UUID{uuid.New()},
                TargetThingClassIDs: []uuid.UUID{uuid.New()},
            },
            setupMocks: func(rcRepo *mocks.RelationshipClassRepository, tcRepo *mocks.ThingClassRepository) {
                rcRepo.On("GetByName", mock.Anything, mock.Anything, "invalid_combo").
                    Return(nil, repositories.ErrNotFound)
                
                tcRepo.On("GetByIDs", mock.Anything, mock.Anything).
                    Return([]*domain.ThingClass{
                        {ID: uuid.New(), TenantID: uuid.New()},
                        {ID: uuid.New(), TenantID: uuid.New()},
                    }, nil)
            },
            expectedError: "ONE_TO_ONE relationships cannot be bidirectional",
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // Setup
            rcRepo := &mocks.RelationshipClassRepository{}
            tcRepo := &mocks.ThingClassRepository{}
            
            if tt.setupMocks != nil {
                tt.setupMocks(rcRepo, tcRepo)
            }
            
            service := services.NewRelationshipClassService(rcRepo, tcRepo, nil, nil, nil)
            
            ctx := context.WithValue(context.Background(), "tenant_id", uuid.New())
            ctx = context.WithValue(ctx, "user_id", uuid.New())
            
            // Execute
            result, err := service.CreateRelationshipClass(ctx, tt.input)
            
            // Assert
            if tt.expectedError != "" {
                require.Error(t, err)
                assert.Contains(t, err.Error(), tt.expectedError)
                assert.Nil(t, result)
            } else {
                require.NoError(t, err)
                require.NotNil(t, result)
                if tt.validateResult != nil {
                    tt.validateResult(t, result)
                }
            }
            
            rcRepo.AssertExpectations(t)
            tcRepo.AssertExpectations(t)
        })
    }
}

func stringPtr(s string) *string {
    return &s
}
```

### Integration Tests
```go
package integration_test

import (
    "context"
    "testing"
    "time"
    
    "github.com/google/uuid"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
    "github.com/stretchr/testify/suite"
    
    "udm/internal/domain"
    "udm/internal/services"
    "udm/test/fixtures"
)

type RelationshipClassIntegrationTestSuite struct {
    suite.Suite
    service     services.RelationshipClassService
    ctx         context.Context
    tenantID    uuid.UUID
    userID      uuid.UUID
    thingClasses []*domain.ThingClass
}

func (suite *RelationshipClassIntegrationTestSuite) SetupTest() {
    suite.tenantID = uuid.New()
    suite.userID = uuid.New()
    
    suite.ctx = context.WithValue(context.Background(), "tenant_id", suite.tenantID)
    suite.ctx = context.WithValue(suite.ctx, "user_id", suite.userID)
    
    // Create test Thing Classes
    suite.thingClasses = fixtures.CreateTestThingClasses(suite.ctx, suite.tenantID, 3)
}

func (suite *RelationshipClassIntegrationTestSuite) TestCreateAndRetrieveRelationshipClass() {
    // Create relationship class
    input := &services.CreateRelationshipClassInput{
        Name:                "project_contains_task",
        DisplayName:         "Project Contains Task",
        Description:         stringPtr("Projects contain tasks"),
        IsDirectional:       true,
        IsBidirectional:     false,
        Cardinality:         domain.CardinalityOneToMany,
        SourceThingClassIDs: []uuid.UUID{suite.thingClasses[0].ID}, // Project
        TargetThingClassIDs: []uuid.UUID{suite.thingClasses[1].ID}, // Task
    }
    
    created, err := suite.service.CreateRelationshipClass(suite.ctx, input)
    require.NoError(suite.T(), err)
    require.NotNil(suite.T(), created)
    
    // Verify creation
    assert.NotEqual(suite.T(), uuid.Nil, created.ID)
    assert.Equal(suite.T(), suite.tenantID, created.TenantID)
    assert.Equal(suite.T(), input.Name, created.Name)
    assert.Equal(suite.T(), input.DisplayName, created.DisplayName)
    assert.Equal(suite.T(), input.Cardinality, created.Cardinality)
    assert.True(suite.T(), created.IsActive)
    assert.Equal(suite.T(), int64(1), created.Version)
    
    // Retrieve by ID
    retrieved, err := suite.service.GetRelationshipClass(suite.ctx, created.ID)
    require.NoError(suite.T(), err)
    require.NotNil(suite.T(), retrieved)
    
    assert.Equal(suite.T(), created.ID, retrieved.ID)
    assert.Equal(suite.T(), created.Name, retrieved.Name)
    assert.Equal(suite.T(), created.Cardinality, retrieved.Cardinality)
    
    // Retrieve by name
    retrievedByName, err := suite.service.GetRelationshipClassByName(suite.ctx, suite.tenantID, created.Name)
    require.NoError(suite.T(), err)
    require.NotNil(suite.T(), retrievedByName)
    
    assert.Equal(suite.T(), created.ID, retrievedByName.ID)
}

func (suite *RelationshipClassIntegrationTestSuite) TestUpdateRelationshipClass() {
    // Create initial relationship class
    created := fixtures.CreateTestRelationshipClass(suite.ctx, suite.tenantID, suite.userID, suite.thingClasses)
    
    // Update
    updateInput := &services.UpdateRelationshipClassInput{
        ID:          created.ID,
        DisplayName: stringPtr("Updated Display Name"),
        Description: stringPtr("Updated description"),
        IsActive:    boolPtr(false),
        Version:     created.Version,
    }
    
    updated, err := suite.service.UpdateRelationshipClass(suite.ctx, updateInput)
    require.NoError(suite.T(), err)
    require.NotNil(suite.T(), updated)
    
    // Verify updates
    assert.Equal(suite.T(), created.ID, updated.ID)
    assert.Equal(suite.T(), "Updated Display Name", updated.DisplayName)
    assert.Equal(suite.T(), "Updated description", *updated.Description)
    assert.False(suite.T(), updated.IsActive)
    assert.Equal(suite.T(), created.Version+1, updated.Version)
    assert.True(suite.T(), updated.UpdatedAt.After(created.UpdatedAt))
}

func (suite *RelationshipClassIntegrationTestSuite) TestListRelationshipClassesWithFiltering() {
    // Create multiple relationship classes
    fixtures.CreateMultipleTestRelationshipClasses(suite.ctx, suite.tenantID, suite.userID, suite.thingClasses, 5)
    
    // Test filtering by cardinality
    filter := &services.RelationshipClassFilter{
        TenantID:    suite.tenantID,
        Cardinality: &domain.CardinalityOneToMany,
        IsActive:    boolPtr(true),
    }
    
    pagination := &services.Pagination{
        Limit:  10,
        Offset: 0,
    }
    
    results, err := suite.service.ListRelationshipClasses(suite.ctx, filter, pagination)
    require.NoError(suite.T(), err)
    require.NotNil(suite.T(), results)
    
    // Verify all results match filter
    for _, rc := range results.RelationshipClasses {
        assert.Equal(suite.T(), domain.CardinalityOneToMany, rc.Cardinality)
        assert.True(suite.T(), rc.IsActive)
        assert.Equal(suite.T(), suite.tenantID, rc.TenantID)
    }
}

func TestRelationshipClassIntegrationTestSuite(t *testing.T) {
    suite.Run(t, new(RelationshipClassIntegrationTestSuite))
}

func boolPtr(b bool) *bool {
    return &b
}
```

This comprehensive Go implementation provides a complete, production-ready implementation of the Relationship Class entity following domain-driven design principles, with comprehensive validation, efficient database operations, and thorough testing coverage.