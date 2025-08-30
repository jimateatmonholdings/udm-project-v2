# Relationship Class Entity - Database Schema

**Created:** 2025-08-30  
**Version:** 1.0.0  
**Database:** PostgreSQL 15+ with schema-per-tenant architecture

## Schema Overview

The Relationship Class entity implements a comprehensive database schema designed for high-performance relationship type template storage within the UDM's multi-tenant PostgreSQL architecture. The schema supports complex relationship modeling with cardinality constraints, directional specifications, and flexible constraint management through JSON validation rules while maintaining optimal query performance and referential integrity.

## Multi-Tenant Architecture

### Schema-Per-Tenant Design
```sql
-- Each tenant gets isolated schema: tenant_{tenant_uuid}
-- Example: tenant_550e8400_e29b_41d4_a716_446655440000

-- Tenant schema creation template
CREATE SCHEMA IF NOT EXISTS tenant_{tenant_id};
SET search_path = tenant_{tenant_id};
```

### Cross-Tenant Isolation Strategy
- Complete data isolation through PostgreSQL schema boundaries
- Tenant-specific connection pools with schema context switching
- Row-level security policies as additional protection layers
- Foreign key constraints enforced within tenant schema boundaries only

## Core Table Definition

### relationship_classes Table
```sql
CREATE TABLE relationship_classes (
    -- Primary identification
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id              UUID NOT NULL,
    
    -- Basic properties
    name                   VARCHAR(100) NOT NULL,
    display_name           VARCHAR(200) NOT NULL,
    description            TEXT,
    
    -- Relationship configuration
    is_directional         BOOLEAN NOT NULL DEFAULT true,
    is_bidirectional       BOOLEAN NOT NULL DEFAULT false,
    cardinality           relationship_cardinality NOT NULL,
    
    -- Validation and business rules (replaces Thing Class arrays)
    validation_rules       JSONB DEFAULT '{}',
    
    -- Status and lifecycle
    is_active              BOOLEAN NOT NULL DEFAULT true,
    
    -- Audit fields
    created_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at             TIMESTAMPTZ,
    created_by             UUID NOT NULL,
    updated_by             UUID,
    
    -- Optimistic locking
    version                BIGINT NOT NULL DEFAULT 1,
    
    -- Constraints
    CONSTRAINT relationship_classes_tenant_name_unique 
        UNIQUE (tenant_id, name) 
        WHERE deleted_at IS NULL,
        
    CONSTRAINT relationship_classes_name_format 
        CHECK (name ~ '^[a-z][a-z0-9_]*[a-z0-9]$'),
        
    CONSTRAINT relationship_classes_display_name_length 
        CHECK (char_length(display_name) >= 2),
        
    CONSTRAINT relationship_classes_directional_consistency 
        CHECK (
            NOT (is_bidirectional = true AND is_directional = false)
        ),
        
    CONSTRAINT relationship_classes_cardinality_bidirectional_consistency 
        CHECK (
            NOT (is_bidirectional = true AND cardinality = 'ONE_TO_ONE')
        )
);
```

### Cardinality Enumeration
```sql
CREATE TYPE relationship_cardinality AS ENUM (
    'ONE_TO_ONE',
    'ONE_TO_MANY', 
    'MANY_TO_ONE',
    'MANY_TO_MANY'
);
```

## Indexing Strategy

### Primary Access Patterns
```sql
-- Primary key index (automatic)
-- relationship_classes_pkey on (id)

-- Tenant + name lookup (most common query pattern)
CREATE INDEX idx_relationship_classes_tenant_name 
ON relationship_classes(tenant_id, name) 
WHERE deleted_at IS NULL;

-- Active relationship classes filtering
CREATE INDEX idx_relationship_classes_tenant_active 
ON relationship_classes(tenant_id, is_active) 
WHERE is_active = true AND deleted_at IS NULL;

-- Cardinality-based queries
CREATE INDEX idx_relationship_classes_tenant_cardinality 
ON relationship_classes(tenant_id, cardinality) 
WHERE deleted_at IS NULL;

-- Directional property queries
CREATE INDEX idx_relationship_classes_tenant_directional 
ON relationship_classes(tenant_id, is_directional, is_bidirectional) 
WHERE deleted_at IS NULL;
```

### Temporal and Audit Indexes
```sql
-- Creation date sorting and filtering
CREATE INDEX idx_relationship_classes_tenant_created_at 
ON relationship_classes(tenant_id, created_at DESC) 
WHERE deleted_at IS NULL;

-- Update tracking
CREATE INDEX idx_relationship_classes_tenant_updated_at 
ON relationship_classes(tenant_id, updated_at DESC) 
WHERE deleted_at IS NULL;

-- Soft delete management
CREATE INDEX idx_relationship_classes_tenant_deleted_at 
ON relationship_classes(tenant_id, deleted_at) 
WHERE deleted_at IS NOT NULL;

-- User audit tracking
CREATE INDEX idx_relationship_classes_created_by 
ON relationship_classes(tenant_id, created_by) 
WHERE deleted_at IS NULL;
```

### Validation Rules Indexes
```sql
-- JSONB validation rules queries
CREATE INDEX idx_relationship_classes_validation_rules 
ON relationship_classes USING gin(validation_rules) 
WHERE validation_rules IS NOT NULL AND deleted_at IS NULL;

-- Specific validation rule path queries for source constraints
CREATE INDEX idx_relationship_classes_source_constraints 
ON relationship_classes USING gin((validation_rules -> 'sourceConstraints')) 
WHERE validation_rules ? 'sourceConstraints' AND deleted_at IS NULL;

-- Specific validation rule path queries for target constraints
CREATE INDEX idx_relationship_classes_target_constraints 
ON relationship_classes USING gin((validation_rules -> 'targetConstraints')) 
WHERE validation_rules ? 'targetConstraints' AND deleted_at IS NULL;
```

## Foreign Key Relationships

### User References
```sql
-- Created by user reference
ALTER TABLE relationship_classes
ADD CONSTRAINT fk_relationship_classes_created_by
FOREIGN KEY (created_by) REFERENCES users(id) ON DELETE SET NULL;

-- Updated by user reference
ALTER TABLE relationship_classes
ADD CONSTRAINT fk_relationship_classes_updated_by
FOREIGN KEY (updated_by) REFERENCES users(id) ON DELETE SET NULL;
```

## Triggers and Automation

### Updated Timestamp Trigger
```sql
CREATE OR REPLACE FUNCTION update_relationship_classes_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    NEW.version = OLD.version + 1;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_relationship_classes_updated_at
    BEFORE UPDATE ON relationship_classes
    FOR EACH ROW
    EXECUTE FUNCTION update_relationship_classes_updated_at();
```

### Validation Rules JSON Schema Trigger
```sql
CREATE OR REPLACE FUNCTION validate_relationship_class_validation_rules()
RETURNS TRIGGER AS $$
BEGIN
    -- Validate validation_rules JSON structure if present
    IF NEW.validation_rules IS NOT NULL AND NEW.validation_rules != '{}'::jsonb THEN
        -- Ensure basic structure exists
        IF NOT (NEW.validation_rules ? 'sourceConstraints' OR NEW.validation_rules ? 'targetConstraints') THEN
            RAISE EXCEPTION 'validation_rules must contain either sourceConstraints or targetConstraints';
        END IF;
        
        -- Additional JSON schema validation can be added here
        -- Example: validate allowedThingClasses arrays contain valid UUIDs
    END IF;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_relationship_classes_validate_rules
    BEFORE INSERT OR UPDATE ON relationship_classes
    FOR EACH ROW
    EXECUTE FUNCTION validate_relationship_class_validation_rules();
```

## Performance Optimization

### Statistics and Analysis
```sql
-- Ensure accurate query planning statistics
ALTER TABLE relationship_classes ALTER COLUMN tenant_id SET STATISTICS 1000;
ALTER TABLE relationship_classes ALTER COLUMN name SET STATISTICS 1000;
ALTER TABLE relationship_classes ALTER COLUMN cardinality SET STATISTICS 1000;
ALTER TABLE relationship_classes ALTER COLUMN validation_rules SET STATISTICS 1000;

-- Enable auto-vacuum tuning for optimal performance
ALTER TABLE relationship_classes SET (
    autovacuum_vacuum_scale_factor = 0.05,
    autovacuum_analyze_scale_factor = 0.02
);
```

## Connection to Attribute Assignments

### Polymorphic Relationship Support
```sql
-- The relationship_classes table integrates with attribute_assignments
-- through polymorphic association where:
-- attribute_assignments.class_type = 'relationship_class'
-- attribute_assignments.class_id = relationship_classes.id

-- Example query to get Relationship Class with its attributes:
SELECT 
    rc.*,
    aa.attribute_id,
    a.name AS attribute_name,
    aa.is_required,
    aa.default_value
FROM relationship_classes rc
LEFT JOIN attribute_assignments aa ON (
    aa.class_id = rc.id 
    AND aa.class_type = 'relationship_class'
    AND aa.deleted_at IS NULL
)
LEFT JOIN attributes a ON (a.id = aa.attribute_id)
WHERE rc.tenant_id = $1 
  AND rc.deleted_at IS NULL;
```

## Validation Rules JSON Schema

### Schema Structure
```json
{
  "sourceConstraints": {
    "allowedThingClasses": ["Person", "Department", "Team"],
    "requiredAttributes": ["active_status", "permissions"],
    "maxConnections": 100,
    "validationRules": [
      {
        "type": "attribute_value",
        "attribute": "status",
        "operator": "equals",
        "value": "active"
      }
    ]
  },
  "targetConstraints": {
    "allowedThingClasses": ["Task", "Project", "Goal"],
    "requiredAttributes": ["priority", "due_date"],
    "maxConnections": 50,
    "validationRules": [
      {
        "type": "attribute_exists",
        "attribute": "assignable"
      }
    ]
  },
  "relationshipRules": {
    "requireApproval": true,
    "cascadeDelete": false,
    "inheritPermissions": true
  }
}
```

### Example Validation Rules
```sql
-- Example: Assignment relationship with flexible constraints
INSERT INTO relationship_classes (
    name, display_name, cardinality, validation_rules
) VALUES (
    'assigned_to',
    'Assigned To',
    'MANY_TO_ONE',
    '{
        "sourceConstraints": {
            "allowedThingClasses": ["Task", "Issue", "Bug"],
            "requiredAttributes": ["status"]
        },
        "targetConstraints": {
            "allowedThingClasses": ["Person", "Team"],
            "maxConnections": 1,
            "validationRules": [
                {
                    "type": "attribute_value",
                    "attribute": "active",
                    "operator": "equals", 
                    "value": true
                }
            ]
        }
    }'::jsonb
);
```

## Migration Scripts

### Initial Schema Creation
```sql
-- V001__create_relationship_classes_table.sql
BEGIN;

-- Create cardinality enum
CREATE TYPE relationship_cardinality AS ENUM (
    'ONE_TO_ONE',
    'ONE_TO_MANY', 
    'MANY_TO_ONE',
    'MANY_TO_MANY'
);

-- Create main table
CREATE TABLE relationship_classes (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id              UUID NOT NULL,
    name                   VARCHAR(100) NOT NULL,
    display_name           VARCHAR(200) NOT NULL,
    description            TEXT,
    is_directional         BOOLEAN NOT NULL DEFAULT true,
    is_bidirectional       BOOLEAN NOT NULL DEFAULT false,
    cardinality           relationship_cardinality NOT NULL,
    validation_rules       JSONB DEFAULT '{}',
    is_active              BOOLEAN NOT NULL DEFAULT true,
    created_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at             TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    deleted_at             TIMESTAMPTZ,
    created_by             UUID NOT NULL,
    updated_by             UUID,
    version                BIGINT NOT NULL DEFAULT 1,
    
    -- Add all constraints
    CONSTRAINT relationship_classes_tenant_name_unique 
        UNIQUE (tenant_id, name) WHERE deleted_at IS NULL,
    CONSTRAINT relationship_classes_name_format 
        CHECK (name ~ '^[a-z][a-z0-9_]*[a-z0-9]$'),
    CONSTRAINT relationship_classes_display_name_length 
        CHECK (char_length(display_name) >= 2),
    CONSTRAINT relationship_classes_directional_consistency 
        CHECK (NOT (is_bidirectional = true AND is_directional = false)),
    CONSTRAINT relationship_classes_cardinality_bidirectional_consistency 
        CHECK (NOT (is_bidirectional = true AND cardinality = 'ONE_TO_ONE'))
);

-- Create all indexes
CREATE INDEX idx_relationship_classes_tenant_name 
ON relationship_classes(tenant_id, name) WHERE deleted_at IS NULL;

CREATE INDEX idx_relationship_classes_tenant_active 
ON relationship_classes(tenant_id, is_active) WHERE is_active = true AND deleted_at IS NULL;

CREATE INDEX idx_relationship_classes_tenant_cardinality 
ON relationship_classes(tenant_id, cardinality) WHERE deleted_at IS NULL;

CREATE INDEX idx_relationship_classes_validation_rules 
ON relationship_classes USING gin(validation_rules) WHERE validation_rules IS NOT NULL AND deleted_at IS NULL;

CREATE INDEX idx_relationship_classes_tenant_created_at 
ON relationship_classes(tenant_id, created_at DESC) WHERE deleted_at IS NULL;

-- Create triggers
CREATE OR REPLACE FUNCTION update_relationship_classes_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    NEW.version = OLD.version + 1;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_relationship_classes_updated_at
    BEFORE UPDATE ON relationship_classes
    FOR EACH ROW
    EXECUTE FUNCTION update_relationship_classes_updated_at();

COMMIT;
```

## Query Examples

### Common Query Patterns
```sql
-- Find relationship classes by name
SELECT * FROM relationship_classes 
WHERE tenant_id = $1 AND name = $2 AND deleted_at IS NULL;

-- Get relationship classes that can connect specific Thing Class types
SELECT * FROM relationship_classes 
WHERE tenant_id = $1 
  AND (
    validation_rules->'sourceConstraints'->'allowedThingClasses' ? $2
    OR validation_rules->'targetConstraints'->'allowedThingClasses' ? $2
  )
  AND is_active = true 
  AND deleted_at IS NULL;

-- Find bidirectional relationship classes
SELECT * FROM relationship_classes 
WHERE tenant_id = $1 
  AND is_bidirectional = true 
  AND deleted_at IS NULL
ORDER BY created_at DESC;

-- Get relationship classes with specific cardinality
SELECT * FROM relationship_classes 
WHERE tenant_id = $1 
  AND cardinality = 'MANY_TO_MANY'::relationship_cardinality
  AND deleted_at IS NULL;

-- Complex filtering with validation rules
SELECT * FROM relationship_classes 
WHERE tenant_id = $1 
  AND validation_rules ? 'sourceConstraints'
  AND validation_rules->'sourceConstraints' ? 'maxConnections'
  AND is_active = true 
  AND deleted_at IS NULL;
```

### Performance Analysis Queries
```sql
-- Analyze relationship class distribution by cardinality
SELECT 
    cardinality,
    COUNT(*) as count,
    COUNT(CASE WHEN validation_rules ? 'sourceConstraints' THEN 1 END) as with_source_constraints,
    COUNT(CASE WHEN validation_rules ? 'targetConstraints' THEN 1 END) as with_target_constraints
FROM relationship_classes 
WHERE tenant_id = $1 AND deleted_at IS NULL
GROUP BY cardinality;

-- Monitor relationship class complexity
SELECT 
    name,
    cardinality,
    CASE WHEN validation_rules = '{}'::jsonb THEN 0 
         ELSE jsonb_array_length(
           COALESCE(validation_rules->'sourceConstraints'->'allowedThingClasses', '[]'::jsonb)
         ) + jsonb_array_length(
           COALESCE(validation_rules->'targetConstraints'->'allowedThingClasses', '[]'::jsonb)
         )
    END as total_thing_class_constraints
FROM relationship_classes 
WHERE tenant_id = $1 AND deleted_at IS NULL
ORDER BY total_thing_class_constraints DESC;
```

## Backup and Recovery Considerations

### Point-in-Time Recovery
- Full database backups with WAL archiving for complete recovery scenarios
- Tenant-level backup capabilities using schema-specific dump/restore operations
- Relationship class definition export/import utilities for schema migration

### Data Archival Strategy
- Archive deleted relationship classes after configurable retention periods
- Maintain relationship class definition history for audit and compliance requirements
- Implement automated cleanup procedures for soft-deleted relationship classes

## Security and Compliance

### Row-Level Security
```sql
-- Enable RLS for additional tenant isolation
ALTER TABLE relationship_classes ENABLE ROW LEVEL SECURITY;

-- Create tenant isolation policy
CREATE POLICY relationship_classes_tenant_isolation ON relationship_classes
    FOR ALL TO application_role
    USING (tenant_id = current_setting('app.tenant_id')::UUID);
```

### Audit Trail Integration
- All relationship class modifications logged through trigger-based audit system
- Change tracking includes old/new values for critical fields and user attribution
- Integration with external audit systems for compliance reporting and analysis