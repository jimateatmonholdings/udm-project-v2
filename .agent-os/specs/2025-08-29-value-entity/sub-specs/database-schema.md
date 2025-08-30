# Value Entity - Database Schema Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**Database:** PostgreSQL 15+ with Multi-Tenant Schema Architecture  
**Entity:** Value - Property data storage for Thing-Attribute combinations  

## Schema Architecture Overview

The Value database schema implements a high-performance, versioned data storage system that maintains actual property values for Thing instances according to their Attribute Assignment configurations. Operating within PostgreSQL's multi-tenant architecture, the schema utilizes polymorphic storage patterns, advanced indexing strategies, and comprehensive audit capabilities to support millions of values per tenant while maintaining sub-200ms query performance and full data integrity.

## Core Table Structure

### Values Storage Table

```sql
-- Core values table (created in each tenant schema)
CREATE TABLE values (
    -- Primary identification
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Core relationship identifiers
    thing_id UUID NOT NULL,
    attribute_id UUID NOT NULL,
    assignment_id UUID NOT NULL,
    
    -- Polymorphic value storage optimized for data types
    value_text TEXT,
    value_integer BIGINT,
    value_decimal NUMERIC(19,6),
    value_boolean BOOLEAN,
    value_date DATE,
    value_datetime TIMESTAMP WITH TIME ZONE,
    value_json JSONB,
    value_reference_id UUID,
    
    -- Data type and validation metadata
    data_type VARCHAR(50) NOT NULL CHECK (data_type IN ('string', 'integer', 'decimal', 'boolean', 'date', 'datetime', 'json', 'reference')),
    is_valid BOOLEAN DEFAULT TRUE,
    validation_errors JSONB DEFAULT '[]' NOT NULL,
    validation_timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    -- Version and history tracking
    version INTEGER DEFAULT 1 NOT NULL,
    is_current BOOLEAN DEFAULT TRUE,
    effective_from TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    effective_to TIMESTAMP WITH TIME ZONE,
    superseded_by UUID,
    
    -- Standard entity metadata
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    created_by UUID NOT NULL,
    updated_by UUID,
    is_active BOOLEAN DEFAULT TRUE,
    
    -- Computed fields for performance
    value_hash VARCHAR(64) GENERATED ALWAYS AS (
        md5(COALESCE(value_text, '') || 
            COALESCE(value_integer::text, '') || 
            COALESCE(value_decimal::text, '') || 
            COALESCE(value_boolean::text, '') || 
            COALESCE(value_date::text, '') || 
            COALESCE(value_datetime::text, '') || 
            COALESCE(value_json::text, '') || 
            COALESCE(value_reference_id::text, ''))
    ) STORED,
    
    -- Business constraints
    CONSTRAINT unique_current_value UNIQUE(thing_id, attribute_id) DEFERRABLE INITIALLY DEFERRED,
    CONSTRAINT valid_data_type_value CHECK (
        (data_type = 'string' AND value_text IS NOT NULL AND 
         value_integer IS NULL AND value_decimal IS NULL AND value_boolean IS NULL AND 
         value_date IS NULL AND value_datetime IS NULL AND value_json IS NULL AND value_reference_id IS NULL) OR
        (data_type = 'integer' AND value_integer IS NOT NULL AND 
         value_text IS NULL AND value_decimal IS NULL AND value_boolean IS NULL AND 
         value_date IS NULL AND value_datetime IS NULL AND value_json IS NULL AND value_reference_id IS NULL) OR
        (data_type = 'decimal' AND value_decimal IS NOT NULL AND 
         value_text IS NULL AND value_integer IS NULL AND value_boolean IS NULL AND 
         value_date IS NULL AND value_datetime IS NULL AND value_json IS NULL AND value_reference_id IS NULL) OR
        (data_type = 'boolean' AND value_boolean IS NOT NULL AND 
         value_text IS NULL AND value_integer IS NULL AND value_decimal IS NULL AND 
         value_date IS NULL AND value_datetime IS NULL AND value_json IS NULL AND value_reference_id IS NULL) OR
        (data_type = 'date' AND value_date IS NOT NULL AND 
         value_text IS NULL AND value_integer IS NULL AND value_decimal IS NULL AND value_boolean IS NULL AND 
         value_datetime IS NULL AND value_json IS NULL AND value_reference_id IS NULL) OR
        (data_type = 'datetime' AND value_datetime IS NOT NULL AND 
         value_text IS NULL AND value_integer IS NULL AND value_decimal IS NULL AND value_boolean IS NULL AND 
         value_date IS NULL AND value_json IS NULL AND value_reference_id IS NULL) OR
        (data_type = 'json' AND value_json IS NOT NULL AND 
         value_text IS NULL AND value_integer IS NULL AND value_decimal IS NULL AND value_boolean IS NULL AND 
         value_date IS NULL AND value_datetime IS NULL AND value_reference_id IS NULL) OR
        (data_type = 'reference' AND value_reference_id IS NOT NULL AND 
         value_text IS NULL AND value_integer IS NULL AND value_decimal IS NULL AND value_boolean IS NULL AND 
         value_date IS NULL AND value_datetime IS NULL AND value_json IS NULL)
    ),
    CONSTRAINT valid_version_sequence CHECK (version > 0),
    CONSTRAINT valid_effective_period CHECK (effective_from <= COALESCE(effective_to, 'infinity'::timestamp)),
    CONSTRAINT valid_validation_errors CHECK (validation_errors IS NOT NULL),
    CONSTRAINT valid_current_versioning CHECK (
        (is_current = TRUE AND effective_to IS NULL) OR
        (is_current = FALSE AND effective_to IS NOT NULL)
    ),
    CONSTRAINT valid_superseded_relationship CHECK (
        (superseded_by IS NULL) OR 
        (superseded_by IS NOT NULL AND is_current = FALSE)
    ),
    
    -- Foreign key constraints (within tenant schema)
    CONSTRAINT fk_values_thing 
        FOREIGN KEY (thing_id) 
        REFERENCES things(id) 
        ON DELETE CASCADE 
        ON UPDATE CASCADE,
    CONSTRAINT fk_values_attribute 
        FOREIGN KEY (attribute_id) 
        REFERENCES attributes(id) 
        ON DELETE RESTRICT 
        ON UPDATE CASCADE,
    CONSTRAINT fk_values_assignment 
        FOREIGN KEY (assignment_id) 
        REFERENCES thing_class_attributes(id) 
        ON DELETE RESTRICT 
        ON UPDATE CASCADE,
    CONSTRAINT fk_values_reference 
        FOREIGN KEY (value_reference_id) 
        REFERENCES things(id) 
        ON DELETE SET NULL 
        ON UPDATE CASCADE,
    CONSTRAINT fk_values_superseded_by 
        FOREIGN KEY (superseded_by) 
        REFERENCES values(id) 
        ON DELETE SET NULL 
        ON UPDATE CASCADE
);

-- Comprehensive table and column comments
COMMENT ON TABLE values IS 'Stores actual property values for Thing instances with versioning, validation, and multi-data-type support';
COMMENT ON COLUMN values.id IS 'Unique identifier for the value record';
COMMENT ON COLUMN values.thing_id IS 'Reference to the Thing that owns this value';
COMMENT ON COLUMN values.attribute_id IS 'Reference to the Attribute definition for this value';
COMMENT ON COLUMN values.assignment_id IS 'Reference to the specific Assignment configuration used';
COMMENT ON COLUMN values.data_type IS 'Data type of the stored value, must match Attribute definition';
COMMENT ON COLUMN values.value_text IS 'Storage for string data type values';
COMMENT ON COLUMN values.value_integer IS 'Storage for integer data type values';
COMMENT ON COLUMN values.value_decimal IS 'Storage for decimal data type values with precision';
COMMENT ON COLUMN values.value_boolean IS 'Storage for boolean data type values';
COMMENT ON COLUMN values.value_date IS 'Storage for date data type values (date only)';
COMMENT ON COLUMN values.value_datetime IS 'Storage for datetime data type values with timezone';
COMMENT ON COLUMN values.value_json IS 'Storage for JSON data type values using JSONB';
COMMENT ON COLUMN values.value_reference_id IS 'Storage for reference data type values (Thing IDs)';
COMMENT ON COLUMN values.is_valid IS 'Whether the value passes all validation rules';
COMMENT ON COLUMN values.validation_errors IS 'JSON array of validation errors if any';
COMMENT ON COLUMN values.validation_timestamp IS 'When the value was last validated';
COMMENT ON COLUMN values.version IS 'Version number for this value, increments with each update';
COMMENT ON COLUMN values.is_current IS 'Whether this is the current active version';
COMMENT ON COLUMN values.effective_from IS 'When this value version became effective';
COMMENT ON COLUMN values.effective_to IS 'When this value version was superseded (null for current)';
COMMENT ON COLUMN values.superseded_by IS 'Reference to the value version that replaced this one';
COMMENT ON COLUMN values.value_hash IS 'Computed hash of the value content for deduplication';
```

### Comprehensive Performance Indexing Strategy

```sql
-- Core relationship indexes for primary access patterns
CREATE INDEX idx_values_thing_current ON values(thing_id) 
WHERE is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_attribute_current ON values(attribute_id) 
WHERE is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_assignment_current ON values(assignment_id) 
WHERE is_current = TRUE AND is_active = TRUE;

-- Multi-column composite indexes for common query patterns
CREATE INDEX idx_values_thing_attribute_current ON values(thing_id, attribute_id) 
WHERE is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_thing_data_type_current ON values(thing_id, data_type) 
WHERE is_current = TRUE AND is_active = TRUE;

-- Data type specific indexes for value filtering and searching
CREATE INDEX idx_values_string_current ON values(value_text text_pattern_ops) 
WHERE data_type = 'string' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_string_lower_current ON values(lower(value_text)) 
WHERE data_type = 'string' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_integer_current ON values(value_integer) 
WHERE data_type = 'integer' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_decimal_current ON values(value_decimal) 
WHERE data_type = 'decimal' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_boolean_current ON values(value_boolean) 
WHERE data_type = 'boolean' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_date_current ON values(value_date) 
WHERE data_type = 'date' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_datetime_current ON values(value_datetime) 
WHERE data_type = 'datetime' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_reference_current ON values(value_reference_id) 
WHERE data_type = 'reference' AND is_current = TRUE AND is_active = TRUE;

-- Advanced JSON indexing for structured data queries
CREATE INDEX idx_values_json_gin ON values USING GIN(value_json) 
WHERE data_type = 'json' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_json_path_ops ON values USING GIN(value_json jsonb_path_ops) 
WHERE data_type = 'json' AND is_current = TRUE AND is_active = TRUE;

-- Full-text search capabilities for string values
CREATE INDEX idx_values_text_search ON values USING GIN(to_tsvector('english', value_text)) 
WHERE data_type = 'string' AND is_current = TRUE AND is_active = TRUE;

-- Trigram indexing for similarity and fuzzy matching
CREATE INDEX idx_values_text_trigram ON values USING GIN(value_text gin_trgm_ops) 
WHERE data_type = 'string' AND is_current = TRUE AND is_active = TRUE;

-- Version and history tracking indexes
CREATE INDEX idx_values_version_history ON values(thing_id, attribute_id, version DESC) 
WHERE is_active = TRUE;

CREATE INDEX idx_values_version_chain ON values(thing_id, attribute_id, effective_from DESC) 
WHERE is_active = TRUE;

CREATE INDEX idx_values_effective_period ON values(effective_from, effective_to) 
WHERE is_active = TRUE;

CREATE INDEX idx_values_superseded_chain ON values(superseded_by) 
WHERE superseded_by IS NOT NULL;

-- Validation and data quality indexes
CREATE INDEX idx_values_invalid_current ON values(thing_id, attribute_id) 
WHERE is_valid = FALSE AND is_current = TRUE;

CREATE INDEX idx_values_validation_errors_gin ON values USING GIN(validation_errors) 
WHERE jsonb_array_length(validation_errors) > 0;

CREATE INDEX idx_values_validation_timestamp ON values(validation_timestamp DESC) 
WHERE is_current = TRUE;

-- Performance and monitoring indexes
CREATE INDEX idx_values_created_at ON values(created_at DESC) 
WHERE is_current = TRUE;

CREATE INDEX idx_values_updated_at ON values(updated_at DESC) 
WHERE is_current = TRUE;

CREATE INDEX idx_values_created_by ON values(created_by) 
WHERE is_current = TRUE;

-- Value hash index for deduplication and change detection
CREATE INDEX idx_values_hash_current ON values(value_hash) 
WHERE is_current = TRUE AND is_active = TRUE;

-- Data type distribution analysis
CREATE INDEX idx_values_data_type_stats ON values(data_type, is_current) 
WHERE is_active = TRUE;

-- Range queries optimization indexes
CREATE INDEX idx_values_integer_range ON values(value_integer, thing_id) 
WHERE data_type = 'integer' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_decimal_range ON values(value_decimal, thing_id) 
WHERE data_type = 'decimal' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_date_range ON values(value_date, thing_id) 
WHERE data_type = 'date' AND is_current = TRUE AND is_active = TRUE;

CREATE INDEX idx_values_datetime_range ON values(value_datetime, thing_id) 
WHERE data_type = 'datetime' AND is_current = TRUE AND is_active = TRUE;

-- Partial indexes for reference integrity
CREATE INDEX idx_values_dangling_references ON values(value_reference_id) 
WHERE data_type = 'reference' AND is_current = TRUE AND is_active = TRUE 
  AND NOT EXISTS (SELECT 1 FROM things WHERE id = value_reference_id);
```

### Value Statistics and Analytics Views

```sql
-- Materialized view for value statistics and performance monitoring
CREATE MATERIALIZED VIEW value_statistics AS
SELECT 
    -- Basic counts and distribution
    COUNT(*) as total_values,
    COUNT(*) FILTER (WHERE is_current = TRUE) as current_values,
    COUNT(*) FILTER (WHERE is_current = FALSE) as historical_values,
    COUNT(*) FILTER (WHERE is_valid = FALSE AND is_current = TRUE) as invalid_values,
    COUNT(DISTINCT thing_id) as things_with_values,
    COUNT(DISTINCT attribute_id) as attributes_with_values,
    COUNT(DISTINCT assignment_id) as assignments_with_values,
    
    -- Data type distribution
    COUNT(*) FILTER (WHERE data_type = 'string' AND is_current = TRUE) as string_values,
    COUNT(*) FILTER (WHERE data_type = 'integer' AND is_current = TRUE) as integer_values,
    COUNT(*) FILTER (WHERE data_type = 'decimal' AND is_current = TRUE) as decimal_values,
    COUNT(*) FILTER (WHERE data_type = 'boolean' AND is_current = TRUE) as boolean_values,
    COUNT(*) FILTER (WHERE data_type = 'date' AND is_current = TRUE) as date_values,
    COUNT(*) FILTER (WHERE data_type = 'datetime' AND is_current = TRUE) as datetime_values,
    COUNT(*) FILTER (WHERE data_type = 'json' AND is_current = TRUE) as json_values,
    COUNT(*) FILTER (WHERE data_type = 'reference' AND is_current = TRUE) as reference_values,
    
    -- Version statistics
    AVG(version) FILTER (WHERE is_current = TRUE) as avg_version_count,
    MAX(version) as max_version_count,
    COUNT(*) FILTER (WHERE version > 1 AND is_current = TRUE) as versioned_values,
    
    -- Temporal statistics
    MIN(created_at) as first_value_created,
    MAX(created_at) as last_value_created,
    MAX(updated_at) as last_value_updated,
    
    -- Data quality metrics
    ROUND(
        (COUNT(*) FILTER (WHERE is_valid = TRUE AND is_current = TRUE)::DECIMAL / 
         NULLIF(COUNT(*) FILTER (WHERE is_current = TRUE), 0)) * 100, 2
    ) as data_quality_percentage,
    
    -- Storage utilization
    COUNT(*) FILTER (WHERE value_text IS NOT NULL) as text_storage_used,
    COUNT(*) FILTER (WHERE value_json IS NOT NULL) as json_storage_used,
    AVG(length(value_text)) FILTER (WHERE value_text IS NOT NULL) as avg_text_length,
    AVG(pg_column_size(value_json)) FILTER (WHERE value_json IS NOT NULL) as avg_json_size,
    
    -- Update timestamp for cache invalidation
    NOW() as statistics_updated_at
    
FROM values
WHERE is_active = TRUE;

-- Indexes on materialized view
CREATE UNIQUE INDEX idx_value_statistics_singleton ON value_statistics((statistics_updated_at IS NOT NULL));

-- Detailed per-attribute value statistics
CREATE MATERIALIZED VIEW value_statistics_by_attribute AS
SELECT 
    a.id as attribute_id,
    a.name as attribute_name,
    a.data_type as attribute_data_type,
    
    -- Value counts
    COUNT(v.id) as total_values,
    COUNT(v.id) FILTER (WHERE v.is_current = TRUE) as current_values,
    COUNT(v.id) FILTER (WHERE v.version > 1 AND v.is_current = TRUE) as versioned_values,
    COUNT(v.id) FILTER (WHERE v.is_valid = FALSE AND v.is_current = TRUE) as invalid_values,
    
    -- Usage statistics
    COUNT(DISTINCT v.thing_id) as things_using_attribute,
    COUNT(DISTINCT v.assignment_id) as assignments_using_attribute,
    
    -- Value distribution (data type specific)
    CASE a.data_type
        WHEN 'string' THEN jsonb_build_object(
            'unique_count', COUNT(DISTINCT v.value_text) FILTER (WHERE v.is_current = TRUE),
            'avg_length', AVG(length(v.value_text)) FILTER (WHERE v.is_current = TRUE),
            'max_length', MAX(length(v.value_text)) FILTER (WHERE v.is_current = TRUE),
            'empty_count', COUNT(*) FILTER (WHERE v.value_text = '' AND v.is_current = TRUE)
        )
        WHEN 'integer' THEN jsonb_build_object(
            'unique_count', COUNT(DISTINCT v.value_integer) FILTER (WHERE v.is_current = TRUE),
            'min_value', MIN(v.value_integer) FILTER (WHERE v.is_current = TRUE),
            'max_value', MAX(v.value_integer) FILTER (WHERE v.is_current = TRUE),
            'avg_value', AVG(v.value_integer) FILTER (WHERE v.is_current = TRUE)
        )
        WHEN 'decimal' THEN jsonb_build_object(
            'unique_count', COUNT(DISTINCT v.value_decimal) FILTER (WHERE v.is_current = TRUE),
            'min_value', MIN(v.value_decimal) FILTER (WHERE v.is_current = TRUE),
            'max_value', MAX(v.value_decimal) FILTER (WHERE v.is_current = TRUE),
            'avg_value', AVG(v.value_decimal) FILTER (WHERE v.is_current = TRUE)
        )
        WHEN 'boolean' THEN jsonb_build_object(
            'true_count', COUNT(*) FILTER (WHERE v.value_boolean = TRUE AND v.is_current = TRUE),
            'false_count', COUNT(*) FILTER (WHERE v.value_boolean = FALSE AND v.is_current = TRUE),
            'true_percentage', ROUND((COUNT(*) FILTER (WHERE v.value_boolean = TRUE AND v.is_current = TRUE)::DECIMAL / 
                                   NULLIF(COUNT(*) FILTER (WHERE v.is_current = TRUE), 0)) * 100, 2)
        )
        WHEN 'date' THEN jsonb_build_object(
            'unique_count', COUNT(DISTINCT v.value_date) FILTER (WHERE v.is_current = TRUE),
            'min_date', MIN(v.value_date) FILTER (WHERE v.is_current = TRUE),
            'max_date', MAX(v.value_date) FILTER (WHERE v.is_current = TRUE)
        )
        WHEN 'datetime' THEN jsonb_build_object(
            'unique_count', COUNT(DISTINCT v.value_datetime) FILTER (WHERE v.is_current = TRUE),
            'min_datetime', MIN(v.value_datetime) FILTER (WHERE v.is_current = TRUE),
            'max_datetime', MAX(v.value_datetime) FILTER (WHERE v.is_current = TRUE)
        )
        WHEN 'json' THEN jsonb_build_object(
            'unique_count', COUNT(DISTINCT v.value_json) FILTER (WHERE v.is_current = TRUE),
            'avg_size', AVG(pg_column_size(v.value_json)) FILTER (WHERE v.is_current = TRUE),
            'max_size', MAX(pg_column_size(v.value_json)) FILTER (WHERE v.is_current = TRUE)
        )
        WHEN 'reference' THEN jsonb_build_object(
            'unique_references', COUNT(DISTINCT v.value_reference_id) FILTER (WHERE v.is_current = TRUE),
            'null_references', COUNT(*) FILTER (WHERE v.value_reference_id IS NULL AND v.is_current = TRUE)
        )
        ELSE '{}'::jsonb
    END as data_type_statistics,
    
    -- Temporal information
    MIN(v.created_at) as first_value_created,
    MAX(v.created_at) as last_value_created,
    MAX(v.updated_at) as last_value_updated,
    
    -- Data quality
    ROUND(
        (COUNT(*) FILTER (WHERE v.is_valid = TRUE AND v.is_current = TRUE)::DECIMAL / 
         NULLIF(COUNT(*) FILTER (WHERE v.is_current = TRUE), 0)) * 100, 2
    ) as data_quality_percentage,
    
    -- Update timestamp
    NOW() as statistics_updated_at
    
FROM attributes a
LEFT JOIN values v ON a.id = v.attribute_id AND v.is_active = TRUE
WHERE a.is_active = TRUE
GROUP BY a.id, a.name, a.data_type;

-- Indexes on per-attribute statistics
CREATE UNIQUE INDEX idx_value_statistics_by_attribute_id ON value_statistics_by_attribute(attribute_id);
CREATE INDEX idx_value_statistics_by_attribute_data_type ON value_statistics_by_attribute(attribute_data_type);
CREATE INDEX idx_value_statistics_by_attribute_current_values ON value_statistics_by_attribute(current_values DESC);
```

### Comprehensive Audit and Change Tracking

```sql
-- Enhanced audit table for complete value change tracking
CREATE TABLE values_audit (
    -- Audit record identification
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    value_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE', 'VERSION')),
    
    -- Comprehensive change data
    old_values JSONB,
    new_values JSONB,
    field_changes JSONB DEFAULT '{}' NOT NULL,
    value_changes JSONB DEFAULT '{}' NOT NULL, -- Specific value field changes
    
    -- Change metadata
    changed_by UUID NOT NULL,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    change_reason TEXT,
    client_info JSONB DEFAULT '{}' NOT NULL,
    
    -- Impact and context analysis
    impact_analysis JSONB DEFAULT '{}' NOT NULL,
    thing_context JSONB DEFAULT '{}' NOT NULL, -- Thing information at time of change
    attribute_context JSONB DEFAULT '{}' NOT NULL, -- Attribute information at time of change
    
    -- Data quality impact
    validation_before JSONB DEFAULT '{}' NOT NULL,
    validation_after JSONB DEFAULT '{}' NOT NULL,
    data_quality_impact VARCHAR(20) DEFAULT 'none' CHECK (
        data_quality_impact IN ('none', 'improved', 'degraded', 'critical')
    ),
    
    -- Change categorization
    change_type VARCHAR(50) DEFAULT 'data' CHECK (
        change_type IN ('data', 'validation', 'version', 'structure', 'reference')
    ),
    impact_level VARCHAR(20) DEFAULT 'low' CHECK (
        impact_level IN ('low', 'medium', 'high', 'critical')
    ),
    
    -- Performance and storage metadata
    old_value_size INTEGER,
    new_value_size INTEGER,
    size_change INTEGER,
    
    -- Version relationship tracking
    previous_version INTEGER,
    new_version INTEGER,
    version_operation VARCHAR(20) DEFAULT 'none' CHECK (
        version_operation IN ('none', 'create', 'update', 'supersede', 'restore')
    ),
    
    -- Foreign key to original value (nullable for DELETE operations)
    CONSTRAINT fk_values_audit_value FOREIGN KEY (value_id) 
        REFERENCES values(id) ON DELETE SET NULL
);

-- Comprehensive audit indexes for analysis and performance
CREATE INDEX idx_values_audit_value_id ON values_audit(value_id);
CREATE INDEX idx_values_audit_changed_at ON values_audit(changed_at DESC);
CREATE INDEX idx_values_audit_changed_by ON values_audit(changed_by);
CREATE INDEX idx_values_audit_operation ON values_audit(operation);
CREATE INDEX idx_values_audit_change_type ON values_audit(change_type);
CREATE INDEX idx_values_audit_impact_level ON values_audit(impact_level);
CREATE INDEX idx_values_audit_data_quality_impact ON values_audit(data_quality_impact);
CREATE INDEX idx_values_audit_version_operation ON values_audit(version_operation);

-- GIN indexes for complex JSON analysis
CREATE INDEX idx_values_audit_field_changes ON values_audit USING GIN(field_changes);
CREATE INDEX idx_values_audit_value_changes ON values_audit USING GIN(value_changes);
CREATE INDEX idx_values_audit_impact_analysis ON values_audit USING GIN(impact_analysis);
CREATE INDEX idx_values_audit_thing_context ON values_audit USING GIN(thing_context);
CREATE INDEX idx_values_audit_attribute_context ON values_audit USING GIN(attribute_context);

-- Advanced audit trigger with comprehensive impact analysis
CREATE OR REPLACE FUNCTION values_audit_trigger()
RETURNS TRIGGER AS $$
DECLARE
    change_reason_text TEXT;
    client_context JSONB;
    impact_data JSONB;
    field_changes_data JSONB;
    value_changes_data JSONB;
    thing_context_data JSONB;
    attribute_context_data JSONB;
    validation_before_data JSONB;
    validation_after_data JSONB;
    change_type_val VARCHAR(50);
    impact_level_val VARCHAR(20);
    data_quality_impact_val VARCHAR(20);
    old_size INTEGER;
    new_size INTEGER;
    version_op VARCHAR(20);
BEGIN
    -- Extract change context from session variables
    change_reason_text := COALESCE(
        current_setting('udm.change_reason', true),
        CASE TG_OP
            WHEN 'INSERT' THEN 'Value created'
            WHEN 'UPDATE' THEN 'Value updated'  
            WHEN 'DELETE' THEN 'Value deleted'
        END
    );
    
    -- Build comprehensive client context
    client_context := jsonb_build_object(
        'application', COALESCE(current_setting('udm.client_app', true), 'unknown'),
        'version', COALESCE(current_setting('udm.client_version', true), 'unknown'),
        'user_agent', COALESCE(current_setting('udm.user_agent', true), null),
        'ip_address', COALESCE(current_setting('udm.client_ip', true), null),
        'session_id', COALESCE(current_setting('udm.session_id', true), null),
        'request_id', COALESCE(current_setting('udm.request_id', true), null),
        'operation_context', COALESCE(current_setting('udm.operation_context', true), null)
    );

    IF TG_OP = 'INSERT' THEN
        -- Build thing context
        SELECT jsonb_build_object(
            'thing_id', t.id,
            'thing_name', t.name,
            'thing_class_id', t.thing_class_id,
            'thing_class_name', tc.name
        ) INTO thing_context_data
        FROM things t
        JOIN thing_classes tc ON t.thing_class_id = tc.id
        WHERE t.id = NEW.thing_id;
        
        -- Build attribute context
        SELECT jsonb_build_object(
            'attribute_id', a.id,
            'attribute_name', a.name,
            'attribute_data_type', a.data_type,
            'attribute_validation_rules', a.validation_rules
        ) INTO attribute_context_data
        FROM attributes a
        WHERE a.id = NEW.attribute_id;
        
        -- Calculate value size
        new_size := COALESCE(
            length(NEW.value_text),
            8, -- approximate size for numeric types
            pg_column_size(NEW.value_json),
            16 -- approximate size for UUID/reference
        );
        
        impact_data := jsonb_build_object(
            'operation', 'create',
            'data_type', NEW.data_type,
            'is_valid', NEW.is_valid,
            'version', NEW.version,
            'value_size', new_size
        );
        
        validation_after_data := jsonb_build_object(
            'is_valid', NEW.is_valid,
            'validation_errors', NEW.validation_errors,
            'validation_timestamp', NEW.validation_timestamp
        );
        
        change_type_val := CASE 
            WHEN NEW.data_type = 'reference' THEN 'reference'
            ELSE 'data'
        END;
        
        impact_level_val := CASE
            WHEN NEW.is_valid = FALSE THEN 'high'
            ELSE 'low'
        END;
        
        data_quality_impact_val := CASE
            WHEN NEW.is_valid = FALSE THEN 'degraded'
            ELSE 'none'
        END;
        
        version_op := CASE
            WHEN NEW.version = 1 THEN 'create'
            ELSE 'update'
        END;
        
        INSERT INTO values_audit (
            value_id, operation, new_values, changed_by, 
            change_reason, client_info, impact_analysis, thing_context, attribute_context,
            validation_after, change_type, impact_level, data_quality_impact,
            new_value_size, size_change, new_version, version_operation
        )
        VALUES (
            NEW.id, 'INSERT', to_jsonb(NEW), NEW.created_by,
            change_reason_text, client_context, impact_data, thing_context_data, attribute_context_data,
            validation_after_data, change_type_val, impact_level_val, data_quality_impact_val,
            new_size, new_size, NEW.version, version_op
        );
        RETURN NEW;
        
    ELSIF TG_OP = 'UPDATE' THEN
        -- Analyze specific field changes
        field_changes_data := jsonb_build_object();
        value_changes_data := jsonb_build_object();
        
        -- Track data value changes by type
        IF OLD.data_type = 'string' AND COALESCE(OLD.value_text, '') != COALESCE(NEW.value_text, '') THEN
            value_changes_data := value_changes_data || jsonb_build_object(
                'value_text', jsonb_build_object('old', OLD.value_text, 'new', NEW.value_text)
            );
        END IF;
        
        IF OLD.data_type = 'integer' AND COALESCE(OLD.value_integer, 0) != COALESCE(NEW.value_integer, 0) THEN
            value_changes_data := value_changes_data || jsonb_build_object(
                'value_integer', jsonb_build_object('old', OLD.value_integer, 'new', NEW.value_integer)
            );
        END IF;
        
        IF OLD.data_type = 'decimal' AND COALESCE(OLD.value_decimal, 0) != COALESCE(NEW.value_decimal, 0) THEN
            value_changes_data := value_changes_data || jsonb_build_object(
                'value_decimal', jsonb_build_object('old', OLD.value_decimal, 'new', NEW.value_decimal)
            );
        END IF;
        
        IF OLD.data_type = 'boolean' AND COALESCE(OLD.value_boolean, false) != COALESCE(NEW.value_boolean, false) THEN
            value_changes_data := value_changes_data || jsonb_build_object(
                'value_boolean', jsonb_build_object('old', OLD.value_boolean, 'new', NEW.value_boolean)
            );
        END IF;
        
        IF OLD.data_type = 'date' AND OLD.value_date != NEW.value_date THEN
            value_changes_data := value_changes_data || jsonb_build_object(
                'value_date', jsonb_build_object('old', OLD.value_date, 'new', NEW.value_date)
            );
        END IF;
        
        IF OLD.data_type = 'datetime' AND OLD.value_datetime != NEW.value_datetime THEN
            value_changes_data := value_changes_data || jsonb_build_object(
                'value_datetime', jsonb_build_object('old', OLD.value_datetime, 'new', NEW.value_datetime)
            );
        END IF;
        
        IF OLD.data_type = 'json' AND COALESCE(OLD.value_json, '{}') != COALESCE(NEW.value_json, '{}') THEN
            value_changes_data := value_changes_data || jsonb_build_object(
                'value_json', jsonb_build_object('old', OLD.value_json, 'new', NEW.value_json)
            );
        END IF;
        
        IF OLD.data_type = 'reference' AND OLD.value_reference_id != NEW.value_reference_id THEN
            value_changes_data := value_changes_data || jsonb_build_object(
                'value_reference_id', jsonb_build_object('old', OLD.value_reference_id, 'new', NEW.value_reference_id)
            );
        END IF;
        
        -- Track metadata changes
        IF OLD.is_valid != NEW.is_valid THEN
            field_changes_data := field_changes_data || jsonb_build_object(
                'is_valid', jsonb_build_object('old', OLD.is_valid, 'new', NEW.is_valid)
            );
        END IF;
        
        IF OLD.validation_errors != NEW.validation_errors THEN
            field_changes_data := field_changes_data || jsonb_build_object(
                'validation_errors', jsonb_build_object('old', OLD.validation_errors, 'new', NEW.validation_errors)
            );
        END IF;
        
        IF OLD.is_current != NEW.is_current THEN
            field_changes_data := field_changes_data || jsonb_build_object(
                'is_current', jsonb_build_object('old', OLD.is_current, 'new', NEW.is_current)
            );
        END IF;
        
        -- Build contexts
        SELECT jsonb_build_object(
            'thing_id', t.id,
            'thing_name', t.name,
            'thing_class_id', t.thing_class_id,
            'thing_class_name', tc.name
        ) INTO thing_context_data
        FROM things t
        JOIN thing_classes tc ON t.thing_class_id = tc.id
        WHERE t.id = NEW.thing_id;
        
        SELECT jsonb_build_object(
            'attribute_id', a.id,
            'attribute_name', a.name,
            'attribute_data_type', a.data_type,
            'attribute_validation_rules', a.validation_rules
        ) INTO attribute_context_data
        FROM attributes a
        WHERE a.id = NEW.attribute_id;
        
        -- Calculate size changes
        old_size := COALESCE(
            length(OLD.value_text),
            8, -- approximate size for numeric types
            pg_column_size(OLD.value_json),
            16 -- approximate size for UUID/reference
        );
        
        new_size := COALESCE(
            length(NEW.value_text),
            8, -- approximate size for numeric types
            pg_column_size(NEW.value_json),
            16 -- approximate size for UUID/reference
        );
        
        validation_before_data := jsonb_build_object(
            'is_valid', OLD.is_valid,
            'validation_errors', OLD.validation_errors,
            'validation_timestamp', OLD.validation_timestamp
        );
        
        validation_after_data := jsonb_build_object(
            'is_valid', NEW.is_valid,
            'validation_errors', NEW.validation_errors,
            'validation_timestamp', NEW.validation_timestamp
        );
        
        impact_data := jsonb_build_object(
            'operation', 'update',
            'version_change', jsonb_build_object('old', OLD.version, 'new', NEW.version),
            'current_status_change', jsonb_build_object('old', OLD.is_current, 'new', NEW.is_current),
            'validation_change', jsonb_build_object('old', OLD.is_valid, 'new', NEW.is_valid),
            'size_change', new_size - old_size,
            'has_value_changes', (jsonb_object_keys(value_changes_data) IS NOT NULL),
            'change_summary', value_changes_data
        );
        
        -- Determine change characteristics
        change_type_val := CASE 
            WHEN jsonb_object_keys(value_changes_data) IS NOT NULL THEN 'data'
            WHEN OLD.is_valid != NEW.is_valid THEN 'validation'
            WHEN OLD.version != NEW.version THEN 'version'
            ELSE 'structure'
        END;
        
        impact_level_val := CASE
            WHEN OLD.is_valid = TRUE AND NEW.is_valid = FALSE THEN 'critical'
            WHEN OLD.is_valid = FALSE AND NEW.is_valid = TRUE THEN 'high'
            WHEN jsonb_object_keys(value_changes_data) IS NOT NULL THEN 'medium'
            ELSE 'low'
        END;
        
        data_quality_impact_val := CASE
            WHEN OLD.is_valid = TRUE AND NEW.is_valid = FALSE THEN 'critical'
            WHEN OLD.is_valid = FALSE AND NEW.is_valid = TRUE THEN 'improved'
            WHEN OLD.is_valid != NEW.is_valid THEN 'degraded'
            ELSE 'none'
        END;
        
        version_op := CASE
            WHEN OLD.version != NEW.version THEN 'supersede'
            WHEN OLD.is_current != NEW.is_current THEN 'restore'
            ELSE 'none'
        END;
        
        INSERT INTO values_audit (
            value_id, operation, old_values, new_values, field_changes, value_changes,
            changed_by, change_reason, client_info, impact_analysis, thing_context, attribute_context,
            validation_before, validation_after, change_type, impact_level, data_quality_impact,
            old_value_size, new_value_size, size_change, previous_version, new_version, version_operation
        )
        VALUES (
            NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), field_changes_data, value_changes_data,
            COALESCE(NEW.updated_by, NEW.created_by), change_reason_text, client_context, 
            impact_data, thing_context_data, attribute_context_data,
            validation_before_data, validation_after_data, change_type_val, impact_level_val, data_quality_impact_val,
            old_size, new_size, new_size - old_size, OLD.version, NEW.version, version_op
        );
        RETURN NEW;
        
    ELSIF TG_OP = 'DELETE' THEN
        -- Build contexts
        SELECT jsonb_build_object(
            'thing_id', t.id,
            'thing_name', t.name,
            'thing_class_id', t.thing_class_id,
            'thing_class_name', tc.name
        ) INTO thing_context_data
        FROM things t
        JOIN thing_classes tc ON t.thing_class_id = tc.id
        WHERE t.id = OLD.thing_id;
        
        SELECT jsonb_build_object(
            'attribute_id', a.id,
            'attribute_name', a.name,
            'attribute_data_type', a.data_type,
            'attribute_validation_rules', a.validation_rules
        ) INTO attribute_context_data
        FROM attributes a
        WHERE a.id = OLD.attribute_id;
        
        old_size := COALESCE(
            length(OLD.value_text),
            8,
            pg_column_size(OLD.value_json),
            16
        );
        
        impact_data := jsonb_build_object(
            'operation', 'delete',
            'was_current', OLD.is_current,
            'was_valid', OLD.is_valid,
            'version', OLD.version,
            'value_size_lost', old_size
        );
        
        validation_before_data := jsonb_build_object(
            'is_valid', OLD.is_valid,
            'validation_errors', OLD.validation_errors,
            'validation_timestamp', OLD.validation_timestamp
        );
        
        INSERT INTO values_audit (
            value_id, operation, old_values, changed_by,
            change_reason, client_info, impact_analysis, thing_context, attribute_context,
            validation_before, change_type, impact_level, data_quality_impact,
            old_value_size, size_change, previous_version
        )
        VALUES (
            OLD.id, 'DELETE', to_jsonb(OLD),
            COALESCE(OLD.updated_by, OLD.created_by),
            change_reason_text, client_context, impact_data, thing_context_data, attribute_context_data,
            validation_before_data, 'data', 'high', 'degraded',
            old_size, -old_size, OLD.version
        );
        RETURN OLD;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Create comprehensive audit trigger
CREATE TRIGGER values_audit_trig
    AFTER INSERT OR UPDATE OR DELETE ON values
    FOR EACH ROW EXECUTE FUNCTION values_audit_trigger();
```

This database schema specification provides a comprehensive foundation for the Value entity with full multi-tenant support, advanced indexing, comprehensive auditing, and performance optimization capabilities.