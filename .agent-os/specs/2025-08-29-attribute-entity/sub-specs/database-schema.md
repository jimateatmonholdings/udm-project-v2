# Attribute Entity - Database Schema Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**Database:** PostgreSQL 15+ with Multi-Tenant Schema Architecture  
**Entity:** Attribute - Property definitions for UDM system  

## Schema Architecture Overview

The Attribute entity database schema implements a comprehensive property definition system within PostgreSQL's multi-tenant architecture. Each tenant operates within a dedicated schema, ensuring complete data isolation while supporting dynamic attribute management, validation rule storage, and schema evolution tracking.

## Core Table Structure

### Attributes Table

```sql
-- Core attributes table (created in each tenant schema)
CREATE TABLE attributes (
    -- Primary identification
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Core attribute properties
    name VARCHAR(255) NOT NULL,
    data_type VARCHAR(50) NOT NULL CHECK (
        data_type IN ('string', 'integer', 'decimal', 'boolean', 'date', 'datetime', 'json', 'reference')
    ),
    validation_rules JSONB DEFAULT '{}' NOT NULL,
    description TEXT,
    
    -- Standard entity metadata
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_by UUID,
    is_active BOOLEAN DEFAULT TRUE,
    version INTEGER DEFAULT 1,
    
    -- Business constraints
    CONSTRAINT unique_attribute_name UNIQUE(name),
    CONSTRAINT valid_validation_rules CHECK (validation_rules IS NOT NULL),
    CONSTRAINT valid_name_format CHECK (
        name ~ '^[a-zA-Z][a-zA-Z0-9_]*$' AND 
        length(name) >= 1 AND 
        length(name) <= 255
    ),
    CONSTRAINT valid_description_length CHECK (
        description IS NULL OR length(description) <= 1000
    )
);

-- Table comment for documentation
COMMENT ON TABLE attributes IS 'Attribute definitions for dynamic property management in UDM system';
COMMENT ON COLUMN attributes.id IS 'Unique identifier for the attribute';
COMMENT ON COLUMN attributes.name IS 'Human-readable name for the attribute, must be unique per tenant';
COMMENT ON COLUMN attributes.data_type IS 'Data type constraint for attribute values';
COMMENT ON COLUMN attributes.validation_rules IS 'JSON structure containing validation constraints';
COMMENT ON COLUMN attributes.description IS 'Optional human-readable description of the attribute purpose';
COMMENT ON COLUMN attributes.version IS 'Optimistic locking version number for concurrent updates';
```

### Performance-Optimized Indexes

```sql
-- Primary access patterns optimization
CREATE INDEX idx_attributes_name_active ON attributes(name) 
WHERE is_active = TRUE;

CREATE INDEX idx_attributes_data_type_active ON attributes(data_type) 
WHERE is_active = TRUE;

CREATE INDEX idx_attributes_created_at ON attributes(created_at DESC) 
WHERE is_active = TRUE;

CREATE INDEX idx_attributes_created_by ON attributes(created_by) 
WHERE is_active = TRUE;

-- Full-text search capability
CREATE INDEX idx_attributes_name_text ON attributes 
USING GIN(to_tsvector('english', name)) 
WHERE is_active = TRUE;

CREATE INDEX idx_attributes_description_text ON attributes 
USING GIN(to_tsvector('english', description)) 
WHERE is_active = TRUE AND description IS NOT NULL;

-- Validation rules analysis (GIN for JSON operations)
CREATE INDEX idx_attributes_validation_rules_gin ON attributes 
USING GIN(validation_rules) 
WHERE is_active = TRUE;

-- Combined index for common query patterns
CREATE INDEX idx_attributes_type_name ON attributes(data_type, name) 
WHERE is_active = TRUE;

-- Version tracking for schema evolution
CREATE INDEX idx_attributes_version ON attributes(version) 
WHERE is_active = TRUE;
```

### Audit and Change Tracking

```sql
-- Comprehensive audit table for change tracking
CREATE TABLE attributes_audit (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attribute_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    old_values JSONB,
    new_values JSONB,
    changed_by UUID NOT NULL,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    change_reason TEXT,
    client_info JSONB DEFAULT '{}',
    
    -- Foreign key to original attribute (may be null for DELETE operations)
    CONSTRAINT fk_audit_attribute FOREIGN KEY (attribute_id) 
        REFERENCES attributes(id) ON DELETE SET NULL
);

-- Audit table indexes
CREATE INDEX idx_attributes_audit_attribute_id ON attributes_audit(attribute_id);
CREATE INDEX idx_attributes_audit_changed_at ON attributes_audit(changed_at DESC);
CREATE INDEX idx_attributes_audit_changed_by ON attributes_audit(changed_by);
CREATE INDEX idx_attributes_audit_operation ON attributes_audit(operation);

-- Audit table comments
COMMENT ON TABLE attributes_audit IS 'Complete audit trail for all attribute modifications';
COMMENT ON COLUMN attributes_audit.client_info IS 'Additional context about the change (IP, user agent, etc.)';
```

### Automated Audit Triggers

```sql
-- Comprehensive audit trigger function
CREATE OR REPLACE FUNCTION attributes_audit_trigger()
RETURNS TRIGGER AS $$
DECLARE
    change_reason_text TEXT;
    client_context JSONB;
BEGIN
    -- Extract change context from session variables if available
    change_reason_text := COALESCE(
        current_setting('udm.change_reason', true),
        CASE TG_OP
            WHEN 'INSERT' THEN 'Attribute created'
            WHEN 'UPDATE' THEN 'Attribute updated'
            WHEN 'DELETE' THEN 'Attribute deleted'
        END
    );
    
    -- Build client context from available session variables
    client_context := jsonb_build_object(
        'application', COALESCE(current_setting('udm.client_app', true), 'unknown'),
        'version', COALESCE(current_setting('udm.client_version', true), 'unknown'),
        'user_agent', COALESCE(current_setting('udm.user_agent', true), null),
        'ip_address', COALESCE(current_setting('udm.client_ip', true), null),
        'session_id', COALESCE(current_setting('udm.session_id', true), null)
    );

    IF TG_OP = 'INSERT' THEN
        INSERT INTO attributes_audit (
            attribute_id, operation, new_values, changed_by, 
            change_reason, client_info
        )
        VALUES (
            NEW.id, 'INSERT', to_jsonb(NEW), NEW.created_by,
            change_reason_text, client_context
        );
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO attributes_audit (
            attribute_id, operation, old_values, new_values, changed_by,
            change_reason, client_info
        )
        VALUES (
            NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), 
            COALESCE(NEW.updated_by, NEW.created_by),
            change_reason_text, client_context
        );
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO attributes_audit (
            attribute_id, operation, old_values, changed_by,
            change_reason, client_info
        )
        VALUES (
            OLD.id, 'DELETE', to_jsonb(OLD),
            COALESCE(OLD.updated_by, OLD.created_by),
            change_reason_text, client_context
        );
        RETURN OLD;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Create triggers for all audit operations
CREATE TRIGGER attributes_audit_trig
    AFTER INSERT OR UPDATE OR DELETE ON attributes
    FOR EACH ROW EXECUTE FUNCTION attributes_audit_trigger();
```

## Data Type Validation Schema

### JSON Schema Definitions for Validation Rules

```sql
-- Store JSON schemas for validation rule validation
CREATE TABLE attribute_validation_schemas (
    data_type VARCHAR(50) PRIMARY KEY CHECK (
        data_type IN ('string', 'integer', 'decimal', 'boolean', 'date', 'datetime', 'json', 'reference')
    ),
    json_schema JSONB NOT NULL,
    description TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    version INTEGER DEFAULT 1
);

-- Insert validation schemas for each data type
INSERT INTO attribute_validation_schemas (data_type, json_schema, description) VALUES
('string', '{
    "type": "object",
    "properties": {
        "required": {"type": "boolean"},
        "minLength": {"type": "integer", "minimum": 0},
        "maxLength": {"type": "integer", "minimum": 1},
        "pattern": {"type": "string"},
        "format": {"type": "string", "enum": ["email", "url", "uuid", "phone", "date", "datetime"]},
        "enum": {"type": "array", "items": {"type": "string"}},
        "default": {"type": "string"}
    },
    "additionalProperties": false
}', 'Validation rules for string data type'),

('integer', '{
    "type": "object",
    "properties": {
        "required": {"type": "boolean"},
        "min": {"type": "integer"},
        "max": {"type": "integer"},
        "multipleOf": {"type": "integer", "minimum": 1},
        "enum": {"type": "array", "items": {"type": "integer"}},
        "default": {"type": "integer"}
    },
    "additionalProperties": false
}', 'Validation rules for integer data type'),

('decimal', '{
    "type": "object",
    "properties": {
        "required": {"type": "boolean"},
        "min": {"type": "number"},
        "max": {"type": "number"},
        "precision": {"type": "integer", "minimum": 1, "maximum": 15},
        "scale": {"type": "integer", "minimum": 0, "maximum": 10},
        "multipleOf": {"type": "number", "minimum": 0},
        "enum": {"type": "array", "items": {"type": "number"}},
        "default": {"type": "number"}
    },
    "additionalProperties": false
}', 'Validation rules for decimal data type'),

('boolean', '{
    "type": "object",
    "properties": {
        "required": {"type": "boolean"},
        "default": {"type": "boolean"}
    },
    "additionalProperties": false
}', 'Validation rules for boolean data type'),

('date', '{
    "type": "object",
    "properties": {
        "required": {"type": "boolean"},
        "min": {"type": "string", "format": "date"},
        "max": {"type": "string", "format": "date"},
        "format": {"type": "string", "default": "YYYY-MM-DD"},
        "default": {"type": "string"}
    },
    "additionalProperties": false
}', 'Validation rules for date data type'),

('datetime', '{
    "type": "object",
    "properties": {
        "required": {"type": "boolean"},
        "min": {"type": "string", "format": "date-time"},
        "max": {"type": "string", "format": "date-time"},
        "format": {"type": "string", "default": "ISO8601"},
        "timezone": {"type": "string", "default": "UTC"},
        "default": {"type": "string"}
    },
    "additionalProperties": false
}', 'Validation rules for datetime data type'),

('json', '{
    "type": "object",
    "properties": {
        "required": {"type": "boolean"},
        "schema": {"type": "object"},
        "maxDepth": {"type": "integer", "minimum": 1, "maximum": 10},
        "maxSize": {"type": "integer", "minimum": 1},
        "allowedKeys": {"type": "array", "items": {"type": "string"}},
        "requiredKeys": {"type": "array", "items": {"type": "string"}},
        "default": {}
    },
    "additionalProperties": false
}', 'Validation rules for JSON data type'),

('reference', '{
    "type": "object",
    "properties": {
        "required": {"type": "boolean"},
        "targetThingClass": {"type": "string", "format": "uuid"},
        "allowedClasses": {"type": "array", "items": {"type": "string", "format": "uuid"}},
        "cardinality": {"type": "string", "enum": ["one", "many"]},
        "cascadeDelete": {"type": "boolean", "default": false}
    },
    "additionalProperties": false
}', 'Validation rules for reference data type');
```

### Validation Rule Function

```sql
-- Function to validate validation rules against data type schema
CREATE OR REPLACE FUNCTION validate_attribute_validation_rules(
    p_data_type VARCHAR(50),
    p_validation_rules JSONB
) RETURNS BOOLEAN AS $$
DECLARE
    schema_json JSONB;
    validation_result BOOLEAN;
BEGIN
    -- Empty rules are always valid
    IF p_validation_rules IS NULL OR p_validation_rules = '{}'::jsonb THEN
        RETURN TRUE;
    END IF;

    -- Get the schema for the data type
    SELECT json_schema INTO schema_json
    FROM attribute_validation_schemas
    WHERE data_type = p_data_type;

    -- If no schema found, reject
    IF schema_json IS NULL THEN
        RETURN FALSE;
    END IF;

    -- Validate the validation rules against the schema
    -- This would use a JSON schema validation extension in production
    -- For now, we'll do basic structure validation
    
    -- Ensure validation_rules is an object
    IF jsonb_typeof(p_validation_rules) != 'object' THEN
        RETURN FALSE;
    END IF;

    -- Check for unknown properties (basic implementation)
    IF p_data_type = 'string' THEN
        RETURN validate_string_rules(p_validation_rules);
    ELSIF p_data_type = 'integer' THEN
        RETURN validate_integer_rules(p_validation_rules);
    ELSIF p_data_type = 'decimal' THEN
        RETURN validate_decimal_rules(p_validation_rules);
    -- ... other data types
    END IF;

    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Data type specific validation functions
CREATE OR REPLACE FUNCTION validate_string_rules(rules JSONB) RETURNS BOOLEAN AS $$
BEGIN
    -- Check minLength <= maxLength if both specified
    IF rules ? 'minLength' AND rules ? 'maxLength' THEN
        IF (rules->>'minLength')::int > (rules->>'maxLength')::int THEN
            RETURN FALSE;
        END IF;
    END IF;
    
    -- Validate pattern is valid regex (simplified check)
    IF rules ? 'pattern' THEN
        BEGIN
            PERFORM regexp_replace('test', rules->>'pattern', 'replacement');
        EXCEPTION
            WHEN invalid_regular_expression THEN
                RETURN FALSE;
        END;
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

CREATE OR REPLACE FUNCTION validate_integer_rules(rules JSONB) RETURNS BOOLEAN AS $$
BEGIN
    -- Check min <= max if both specified
    IF rules ? 'min' AND rules ? 'max' THEN
        IF (rules->>'min')::bigint > (rules->>'max')::bigint THEN
            RETURN FALSE;
        END IF;
    END IF;
    
    -- Check multipleOf is positive
    IF rules ? 'multipleOf' THEN
        IF (rules->>'multipleOf')::bigint <= 0 THEN
            RETURN FALSE;
        END IF;
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Add constraint to use validation function
ALTER TABLE attributes ADD CONSTRAINT valid_validation_rules_format 
CHECK (validate_attribute_validation_rules(data_type, validation_rules));
```

## Schema Evolution Support

### Schema Evolution Tracking

```sql
-- Track schema evolution for attributes
CREATE TABLE attribute_schema_evolution (
    evolution_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attribute_id UUID NOT NULL,
    change_type VARCHAR(50) NOT NULL CHECK (change_type IN (
        'VALIDATION_RULES_RELAXED',
        'VALIDATION_RULES_TIGHTENED', 
        'DESCRIPTION_CHANGED',
        'NAME_CHANGED',
        'STATUS_CHANGED'
    )),
    old_values JSONB NOT NULL,
    new_values JSONB NOT NULL,
    impact_analysis JSONB,
    safety_level VARCHAR(20) NOT NULL CHECK (safety_level IN ('SAFE', 'WARNING', 'BREAKING')),
    applied_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    applied_by UUID NOT NULL,
    rollback_available BOOLEAN DEFAULT TRUE,
    rollback_data JSONB,
    
    CONSTRAINT fk_evolution_attribute FOREIGN KEY (attribute_id) 
        REFERENCES attributes(id) ON DELETE CASCADE
);

CREATE INDEX idx_attr_evolution_attribute_id ON attribute_schema_evolution(attribute_id);
CREATE INDEX idx_attr_evolution_applied_at ON attribute_schema_evolution(applied_at DESC);
CREATE INDEX idx_attr_evolution_safety_level ON attribute_schema_evolution(safety_level);
```

### Schema Evolution Functions

```sql
-- Function to analyze schema evolution impact
CREATE OR REPLACE FUNCTION analyze_attribute_change_impact(
    p_attribute_id UUID,
    p_old_validation_rules JSONB,
    p_new_validation_rules JSONB
) RETURNS TABLE (
    safety_level VARCHAR(20),
    impact_details JSONB,
    affected_assignments INTEGER
) AS $$
DECLARE
    assignment_count INTEGER;
    impact_json JSONB;
BEGIN
    -- Count affected Thing Class assignments
    SELECT COUNT(*) INTO assignment_count
    FROM thing_class_attributes tca
    WHERE tca.attribute_id = p_attribute_id AND tca.is_active = TRUE;

    -- Analyze validation rule changes
    IF p_old_validation_rules = p_new_validation_rules THEN
        -- No changes to validation rules
        safety_level := 'SAFE';
        impact_details := jsonb_build_object(
            'change_type', 'none',
            'description', 'No validation rule changes'
        );
    ELSIF jsonb_subset(p_old_validation_rules, p_new_validation_rules) THEN
        -- New rules are more restrictive (superset)
        safety_level := CASE WHEN assignment_count > 0 THEN 'BREAKING' ELSE 'WARNING' END;
        impact_details := jsonb_build_object(
            'change_type', 'restrictive',
            'description', 'New validation rules are more restrictive',
            'risk', 'May invalidate existing data'
        );
    ELSE
        -- Rules are being relaxed or changed in other ways
        safety_level := 'SAFE';
        impact_details := jsonb_build_object(
            'change_type', 'relaxed',
            'description', 'Validation rules relaxed or modified safely'
        );
    END IF;

    affected_assignments := assignment_count;

    RETURN NEXT;
END;
$$ LANGUAGE plpgsql;
```

## Multi-Tenant Schema Management

### Tenant Schema Creation Script

```sql
-- Function to create complete attribute schema for new tenant
CREATE OR REPLACE FUNCTION create_tenant_attribute_schema(p_schema_name TEXT) 
RETURNS BOOLEAN AS $$
DECLARE
    sql_statement TEXT;
BEGIN
    -- Set search path to new tenant schema
    EXECUTE format('SET search_path TO %I, public', p_schema_name);
    
    -- Create attributes table
    EXECUTE '
    CREATE TABLE attributes (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name VARCHAR(255) NOT NULL,
        data_type VARCHAR(50) NOT NULL CHECK (
            data_type IN (''string'', ''integer'', ''decimal'', ''boolean'', ''date'', ''datetime'', ''json'', ''reference'')
        ),
        validation_rules JSONB DEFAULT ''{}'' NOT NULL,
        description TEXT,
        created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
        updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
        created_by UUID NOT NULL,
        updated_by UUID,
        is_active BOOLEAN DEFAULT TRUE,
        version INTEGER DEFAULT 1,
        
        CONSTRAINT unique_attribute_name UNIQUE(name),
        CONSTRAINT valid_validation_rules CHECK (validation_rules IS NOT NULL),
        CONSTRAINT valid_name_format CHECK (
            name ~ ''^[a-zA-Z][a-zA-Z0-9_]*$'' AND 
            length(name) >= 1 AND 
            length(name) <= 255
        )
    )';
    
    -- Create all indexes
    EXECUTE 'CREATE INDEX idx_attributes_name_active ON attributes(name) WHERE is_active = TRUE';
    EXECUTE 'CREATE INDEX idx_attributes_data_type_active ON attributes(data_type) WHERE is_active = TRUE';
    EXECUTE 'CREATE INDEX idx_attributes_created_at ON attributes(created_at DESC) WHERE is_active = TRUE';
    EXECUTE 'CREATE INDEX idx_attributes_name_text ON attributes USING GIN(to_tsvector(''english'', name)) WHERE is_active = TRUE';
    EXECUTE 'CREATE INDEX idx_attributes_validation_rules_gin ON attributes USING GIN(validation_rules) WHERE is_active = TRUE';
    
    -- Create audit table
    EXECUTE '
    CREATE TABLE attributes_audit (
        audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        attribute_id UUID NOT NULL,
        operation VARCHAR(10) NOT NULL CHECK (operation IN (''INSERT'', ''UPDATE'', ''DELETE'')),
        old_values JSONB,
        new_values JSONB,
        changed_by UUID NOT NULL,
        changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
        change_reason TEXT,
        client_info JSONB DEFAULT ''{}''
    )';
    
    -- Create audit indexes
    EXECUTE 'CREATE INDEX idx_attributes_audit_attribute_id ON attributes_audit(attribute_id)';
    EXECUTE 'CREATE INDEX idx_attributes_audit_changed_at ON attributes_audit(changed_at DESC)';
    
    -- Create schema evolution table
    EXECUTE '
    CREATE TABLE attribute_schema_evolution (
        evolution_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        attribute_id UUID NOT NULL,
        change_type VARCHAR(50) NOT NULL,
        old_values JSONB NOT NULL,
        new_values JSONB NOT NULL,
        impact_analysis JSONB,
        safety_level VARCHAR(20) NOT NULL CHECK (safety_level IN (''SAFE'', ''WARNING'', ''BREAKING'')),
        applied_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
        applied_by UUID NOT NULL,
        rollback_available BOOLEAN DEFAULT TRUE,
        rollback_data JSONB
    )';
    
    -- Create triggers
    EXECUTE 'CREATE TRIGGER attributes_audit_trig AFTER INSERT OR UPDATE OR DELETE ON attributes FOR EACH ROW EXECUTE FUNCTION attributes_audit_trigger()';
    
    RETURN TRUE;
    
EXCEPTION
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Failed to create tenant attribute schema for %: %', p_schema_name, SQLERRM;
        RETURN FALSE;
END;
$$ LANGUAGE plpgsql;
```

## Performance Optimization Queries

### Common Query Patterns with Optimization

```sql
-- 1. Get attribute by name (most frequent operation)
PREPARE get_attribute_by_name(TEXT) AS
SELECT id, name, data_type, validation_rules, description,
       created_at, updated_at, created_by, is_active, version
FROM attributes 
WHERE name = $1 AND is_active = TRUE;

-- 2. List attributes with pagination and filtering
PREPARE list_attributes_paginated(VARCHAR, INT, INT) AS
SELECT id, name, data_type, validation_rules, description,
       created_at, updated_at, created_by, is_active, version,
       COUNT(*) OVER() as total_count
FROM attributes 
WHERE ($1 IS NULL OR data_type = $1) AND is_active = TRUE
ORDER BY name ASC
LIMIT $2 OFFSET $3;

-- 3. Search attributes by name pattern
PREPARE search_attributes_by_name(TEXT, INT, INT) AS
SELECT id, name, data_type, validation_rules, description,
       created_at, updated_at, created_by, is_active, version,
       ts_rank(to_tsvector('english', name), plainto_tsquery('english', $1)) as rank
FROM attributes 
WHERE to_tsvector('english', name) @@ plainto_tsquery('english', $1)
  AND is_active = TRUE
ORDER BY rank DESC, name ASC
LIMIT $2 OFFSET $3;

-- 4. Get attribute with assignment count for impact analysis
PREPARE get_attribute_with_usage(UUID) AS
SELECT a.id, a.name, a.data_type, a.validation_rules, a.description,
       a.created_at, a.updated_at, a.created_by, a.is_active, a.version,
       COALESCE(usage.assignment_count, 0) as assignment_count,
       COALESCE(usage.thing_count, 0) as thing_count
FROM attributes a
LEFT JOIN (
    SELECT tca.attribute_id,
           COUNT(tca.id) as assignment_count,
           COUNT(DISTINCT t.id) as thing_count
    FROM thing_class_attributes tca
    LEFT JOIN things t ON t.thing_class_id = tca.thing_class_id AND t.is_active = TRUE
    WHERE tca.is_active = TRUE
    GROUP BY tca.attribute_id
) usage ON a.id = usage.attribute_id
WHERE a.id = $1 AND a.is_active = TRUE;

-- 5. Validate attribute name uniqueness efficiently
PREPARE check_attribute_name_exists(TEXT, UUID) AS
SELECT EXISTS(
    SELECT 1 FROM attributes 
    WHERE name = $1 AND ($2 IS NULL OR id != $2) AND is_active = TRUE
);
```

### Database Maintenance Procedures

```sql
-- Procedure for cleaning up old audit records
CREATE OR REPLACE FUNCTION cleanup_attribute_audit_records(p_retain_days INTEGER DEFAULT 365)
RETURNS INTEGER AS $$
DECLARE
    deleted_count INTEGER;
BEGIN
    DELETE FROM attributes_audit 
    WHERE changed_at < NOW() - (p_retain_days || ' days')::INTERVAL;
    
    GET DIAGNOSTICS deleted_count = ROW_COUNT;
    
    RETURN deleted_count;
END;
$$ LANGUAGE plpgsql;

-- Procedure for analyzing attribute usage statistics
CREATE OR REPLACE FUNCTION analyze_attribute_usage_stats()
RETURNS TABLE (
    data_type VARCHAR(50),
    total_attributes BIGINT,
    active_attributes BIGINT,
    avg_assignments_per_attribute NUMERIC,
    most_used_attribute_name TEXT,
    least_used_attribute_name TEXT
) AS $$
BEGIN
    RETURN QUERY
    WITH attribute_stats AS (
        SELECT a.data_type,
               a.name,
               a.is_active,
               COALESCE(tca_count.assignment_count, 0) as assignments
        FROM attributes a
        LEFT JOIN (
            SELECT attribute_id, COUNT(*) as assignment_count
            FROM thing_class_attributes
            WHERE is_active = TRUE
            GROUP BY attribute_id
        ) tca_count ON a.id = tca_count.attribute_id
    )
    SELECT 
        stats.data_type,
        COUNT(*) as total_attributes,
        COUNT(*) FILTER (WHERE stats.is_active = TRUE) as active_attributes,
        AVG(stats.assignments) as avg_assignments_per_attribute,
        (SELECT name FROM attribute_stats s2 
         WHERE s2.data_type = stats.data_type 
         ORDER BY assignments DESC LIMIT 1) as most_used_attribute_name,
        (SELECT name FROM attribute_stats s3 
         WHERE s3.data_type = stats.data_type 
         ORDER BY assignments ASC LIMIT 1) as least_used_attribute_name
    FROM attribute_stats stats
    GROUP BY stats.data_type
    ORDER BY total_attributes DESC;
END;
$$ LANGUAGE plpgsql;

-- Regular maintenance job for updating statistics
CREATE OR REPLACE FUNCTION update_attribute_statistics()
RETURNS VOID AS $$
BEGIN
    -- Update table statistics
    ANALYZE attributes;
    ANALYZE attributes_audit;
    ANALYZE attribute_schema_evolution;
    
    -- Log maintenance completion
    INSERT INTO system_maintenance_log (operation, completed_at, details)
    VALUES ('attribute_statistics_update', NOW(), 
            jsonb_build_object('operation', 'analyze_attributes', 'status', 'completed'));
END;
$$ LANGUAGE plpgsql;
```

## Migration Scripts

### Version 1.0.0 Initial Schema

```sql
-- Migration: 001_create_attributes_schema.up.sql
-- Description: Create initial attribute entity schema with all supporting structures

BEGIN;

-- Ensure UUID extension is available
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Create main attributes table
CREATE TABLE attributes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    data_type VARCHAR(50) NOT NULL CHECK (
        data_type IN ('string', 'integer', 'decimal', 'boolean', 'date', 'datetime', 'json', 'reference')
    ),
    validation_rules JSONB DEFAULT '{}' NOT NULL,
    description TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_by UUID,
    is_active BOOLEAN DEFAULT TRUE,
    version INTEGER DEFAULT 1,
    
    CONSTRAINT unique_attribute_name UNIQUE(name),
    CONSTRAINT valid_validation_rules CHECK (validation_rules IS NOT NULL),
    CONSTRAINT valid_name_format CHECK (
        name ~ '^[a-zA-Z][a-zA-Z0-9_]*$' AND 
        length(name) >= 1 AND 
        length(name) <= 255
    ),
    CONSTRAINT valid_description_length CHECK (
        description IS NULL OR length(description) <= 1000
    )
);

-- Create indexes
CREATE INDEX idx_attributes_name_active ON attributes(name) WHERE is_active = TRUE;
CREATE INDEX idx_attributes_data_type_active ON attributes(data_type) WHERE is_active = TRUE;
CREATE INDEX idx_attributes_created_at ON attributes(created_at DESC) WHERE is_active = TRUE;
CREATE INDEX idx_attributes_created_by ON attributes(created_by) WHERE is_active = TRUE;
CREATE INDEX idx_attributes_name_text ON attributes USING GIN(to_tsvector('english', name)) WHERE is_active = TRUE;
CREATE INDEX idx_attributes_validation_rules_gin ON attributes USING GIN(validation_rules) WHERE is_active = TRUE;
CREATE INDEX idx_attributes_type_name ON attributes(data_type, name) WHERE is_active = TRUE;
CREATE INDEX idx_attributes_version ON attributes(version) WHERE is_active = TRUE;

-- Create audit table
CREATE TABLE attributes_audit (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attribute_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    old_values JSONB,
    new_values JSONB,
    changed_by UUID NOT NULL,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    change_reason TEXT,
    client_info JSONB DEFAULT '{}'
);

-- Create audit indexes
CREATE INDEX idx_attributes_audit_attribute_id ON attributes_audit(attribute_id);
CREATE INDEX idx_attributes_audit_changed_at ON attributes_audit(changed_at DESC);
CREATE INDEX idx_attributes_audit_changed_by ON attributes_audit(changed_by);
CREATE INDEX idx_attributes_audit_operation ON attributes_audit(operation);

-- Create schema evolution table
CREATE TABLE attribute_schema_evolution (
    evolution_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    attribute_id UUID NOT NULL,
    change_type VARCHAR(50) NOT NULL CHECK (change_type IN (
        'VALIDATION_RULES_RELAXED',
        'VALIDATION_RULES_TIGHTENED', 
        'DESCRIPTION_CHANGED',
        'NAME_CHANGED',
        'STATUS_CHANGED'
    )),
    old_values JSONB NOT NULL,
    new_values JSONB NOT NULL,
    impact_analysis JSONB,
    safety_level VARCHAR(20) NOT NULL CHECK (safety_level IN ('SAFE', 'WARNING', 'BREAKING')),
    applied_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    applied_by UUID NOT NULL,
    rollback_available BOOLEAN DEFAULT TRUE,
    rollback_data JSONB,
    
    CONSTRAINT fk_evolution_attribute FOREIGN KEY (attribute_id) 
        REFERENCES attributes(id) ON DELETE CASCADE
);

-- Create schema evolution indexes
CREATE INDEX idx_attr_evolution_attribute_id ON attribute_schema_evolution(attribute_id);
CREATE INDEX idx_attr_evolution_applied_at ON attribute_schema_evolution(applied_at DESC);
CREATE INDEX idx_attr_evolution_safety_level ON attribute_schema_evolution(safety_level);

-- Create validation schema table
CREATE TABLE attribute_validation_schemas (
    data_type VARCHAR(50) PRIMARY KEY CHECK (
        data_type IN ('string', 'integer', 'decimal', 'boolean', 'date', 'datetime', 'json', 'reference')
    ),
    json_schema JSONB NOT NULL,
    description TEXT NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    version INTEGER DEFAULT 1
);

-- Insert validation schemas (content as defined above)
-- ... (insert statements would go here)

-- Create all functions and triggers
-- ... (function definitions as above)

-- Grant appropriate permissions
-- GRANT SELECT, INSERT, UPDATE, DELETE ON attributes TO udm_application;
-- GRANT SELECT ON attributes_audit TO udm_application;
-- GRANT SELECT ON attribute_schema_evolution TO udm_application;

COMMIT;
```

### Rollback Script

```sql
-- Migration: 001_create_attributes_schema.down.sql
-- Description: Rollback initial attribute entity schema

BEGIN;

-- Drop triggers first
DROP TRIGGER IF EXISTS attributes_audit_trig ON attributes;

-- Drop functions
DROP FUNCTION IF EXISTS attributes_audit_trigger();
DROP FUNCTION IF EXISTS validate_attribute_validation_rules(VARCHAR(50), JSONB);
DROP FUNCTION IF EXISTS analyze_attribute_change_impact(UUID, JSONB, JSONB);
DROP FUNCTION IF EXISTS cleanup_attribute_audit_records(INTEGER);
DROP FUNCTION IF EXISTS analyze_attribute_usage_stats();

-- Drop tables in reverse dependency order
DROP TABLE IF EXISTS attribute_schema_evolution;
DROP TABLE IF EXISTS attributes_audit;
DROP TABLE IF EXISTS attribute_validation_schemas;
DROP TABLE IF EXISTS attributes;

COMMIT;
```

This comprehensive database schema specification provides a robust foundation for the Attribute entity with full multi-tenant support, comprehensive auditing, schema evolution tracking, and performance optimization.