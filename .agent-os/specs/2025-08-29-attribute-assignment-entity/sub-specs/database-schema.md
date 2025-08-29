# Attribute Assignment Entity - Database Schema Specification

**Created:** 2025-08-29  
**Version:** 1.0.0  
**Database:** PostgreSQL 15+ with Multi-Tenant Schema Architecture  
**Entity:** Attribute Assignment - Bridge connecting Thing Classes to Attributes with constraints  

## Schema Architecture Overview

The Attribute Assignment database schema implements the critical junction table design pattern that bridges Thing Classes and Attributes while storing rich metadata about their relationships. Operating within PostgreSQL's multi-tenant architecture, each assignment defines not only the relationship between entities but also behavioral characteristics, validation extensions, and presentation metadata that transform static data models into dynamic, configurable business schemas.

## Core Table Structure

### Thing Class Attributes Assignment Table

```sql
-- Core assignment bridge table (created in each tenant schema)
CREATE TABLE thing_class_attributes (
    -- Primary identification
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    
    -- Core relationship identifiers
    thing_class_id UUID NOT NULL,
    attribute_id UUID NOT NULL,
    
    -- Assignment configuration
    is_required BOOLEAN DEFAULT FALSE,
    sort_order INTEGER DEFAULT 0,
    display_name VARCHAR(255),
    assignment_validation_rules JSONB DEFAULT '{}' NOT NULL,
    default_value JSONB,
    ui_hints JSONB DEFAULT '{}' NOT NULL,
    
    -- Assignment metadata
    description TEXT,
    assignment_type VARCHAR(50) DEFAULT 'standard' NOT NULL CHECK (
        assignment_type IN ('standard', 'computed', 'inherited')
    ),
    visibility VARCHAR(20) DEFAULT 'visible' NOT NULL CHECK (
        visibility IN ('visible', 'hidden', 'readonly')
    ),
    
    -- Standard entity metadata
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    created_by UUID NOT NULL,
    updated_by UUID,
    is_active BOOLEAN DEFAULT TRUE,
    version INTEGER DEFAULT 1,
    
    -- Business constraints
    CONSTRAINT unique_thing_class_attribute UNIQUE(thing_class_id, attribute_id),
    CONSTRAINT valid_sort_order CHECK (sort_order >= 0 AND sort_order <= 999999),
    CONSTRAINT valid_assignment_validation_rules CHECK (assignment_validation_rules IS NOT NULL),
    CONSTRAINT valid_ui_hints CHECK (ui_hints IS NOT NULL),
    CONSTRAINT valid_display_name_length CHECK (
        display_name IS NULL OR (length(display_name) >= 1 AND length(display_name) <= 255)
    ),
    CONSTRAINT valid_description_length CHECK (
        description IS NULL OR length(description) <= 1000
    ),
    CONSTRAINT valid_assignment_type_value CHECK (
        assignment_type IN ('standard', 'computed', 'inherited')
    ),
    CONSTRAINT valid_visibility_value CHECK (
        visibility IN ('visible', 'hidden', 'readonly')
    ),
    
    -- Foreign key constraints (within tenant schema)
    CONSTRAINT fk_thing_class_attribute_thing_class 
        FOREIGN KEY (thing_class_id) 
        REFERENCES thing_classes(id) 
        ON DELETE CASCADE 
        ON UPDATE CASCADE,
    CONSTRAINT fk_thing_class_attribute_attribute 
        FOREIGN KEY (attribute_id) 
        REFERENCES attributes(id) 
        ON DELETE CASCADE 
        ON UPDATE CASCADE
);

-- Comprehensive table and column comments
COMMENT ON TABLE thing_class_attributes IS 'Assignment bridge connecting Thing Classes to Attributes with constraints, validation rules, and presentation metadata';
COMMENT ON COLUMN thing_class_attributes.id IS 'Unique identifier for the assignment relationship';
COMMENT ON COLUMN thing_class_attributes.thing_class_id IS 'Reference to the Thing Class that receives the attribute';
COMMENT ON COLUMN thing_class_attributes.attribute_id IS 'Reference to the Attribute being assigned';
COMMENT ON COLUMN thing_class_attributes.is_required IS 'Whether this attribute is required when creating Things of this class';
COMMENT ON COLUMN thing_class_attributes.sort_order IS 'Display order for UI presentation (0-999999)';
COMMENT ON COLUMN thing_class_attributes.display_name IS 'Override name for UI display, falls back to Attribute name if null';
COMMENT ON COLUMN thing_class_attributes.assignment_validation_rules IS 'JSON validation rules that extend base Attribute validation';
COMMENT ON COLUMN thing_class_attributes.default_value IS 'JSON-encoded default value respecting Attribute data type constraints';
COMMENT ON COLUMN thing_class_attributes.ui_hints IS 'JSON metadata for UI presentation, layout, and interaction hints';
COMMENT ON COLUMN thing_class_attributes.description IS 'Assignment-specific description extending base Attribute description';
COMMENT ON COLUMN thing_class_attributes.assignment_type IS 'Type of assignment: standard (direct), computed (calculated), inherited (from parent)';
COMMENT ON COLUMN thing_class_attributes.visibility IS 'UI visibility control: visible, hidden, or readonly';
COMMENT ON COLUMN thing_class_attributes.version IS 'Optimistic locking version number for concurrent update safety';
```

### Performance-Optimized Indexes

```sql
-- Primary access patterns optimization
CREATE INDEX idx_thing_class_attributes_thing_class_active ON thing_class_attributes(thing_class_id) 
WHERE is_active = TRUE;

CREATE INDEX idx_thing_class_attributes_attribute_active ON thing_class_attributes(attribute_id) 
WHERE is_active = TRUE;

-- Schema composition index (most critical for performance)
CREATE INDEX idx_thing_class_attributes_schema_composition ON thing_class_attributes(thing_class_id, is_active, sort_order)
WHERE is_active = TRUE;

-- Required/optional filtering with sort order
CREATE INDEX idx_thing_class_attributes_required_sorted ON thing_class_attributes(thing_class_id, is_required, sort_order)
WHERE is_active = TRUE;

-- Assignment type and visibility filtering
CREATE INDEX idx_thing_class_attributes_type_visibility ON thing_class_attributes(assignment_type, visibility)
WHERE is_active = TRUE;

-- Sort order uniqueness enforcement helper
CREATE INDEX idx_thing_class_attributes_sort_order_unique ON thing_class_attributes(thing_class_id, sort_order)
WHERE is_active = TRUE;

-- JSON indexing for validation rules and UI hints
CREATE INDEX idx_thing_class_attributes_validation_rules_gin ON thing_class_attributes 
USING GIN(assignment_validation_rules) 
WHERE is_active = TRUE;

CREATE INDEX idx_thing_class_attributes_ui_hints_gin ON thing_class_attributes 
USING GIN(ui_hints) 
WHERE is_active = TRUE;

CREATE INDEX idx_thing_class_attributes_default_value_gin ON thing_class_attributes 
USING GIN(default_value) 
WHERE is_active = TRUE AND default_value IS NOT NULL;

-- Full-text search capabilities
CREATE INDEX idx_thing_class_attributes_display_name_text ON thing_class_attributes 
USING GIN(to_tsvector('english', COALESCE(display_name, ''))) 
WHERE is_active = TRUE AND display_name IS NOT NULL;

CREATE INDEX idx_thing_class_attributes_description_text ON thing_class_attributes 
USING GIN(to_tsvector('english', COALESCE(description, ''))) 
WHERE is_active = TRUE AND description IS NOT NULL;

-- Combined text search on display name and description
CREATE INDEX idx_thing_class_attributes_full_text ON thing_class_attributes 
USING GIN(to_tsvector('english', 
    COALESCE(display_name, '') || ' ' || COALESCE(description, '')
)) 
WHERE is_active = TRUE;

-- Temporal indexes for audit and change tracking
CREATE INDEX idx_thing_class_attributes_created_at ON thing_class_attributes(created_at DESC) 
WHERE is_active = TRUE;

CREATE INDEX idx_thing_class_attributes_updated_at ON thing_class_attributes(updated_at DESC) 
WHERE is_active = TRUE;

CREATE INDEX idx_thing_class_attributes_created_by ON thing_class_attributes(created_by) 
WHERE is_active = TRUE;

-- Version tracking for optimistic locking and schema evolution
CREATE INDEX idx_thing_class_attributes_version ON thing_class_attributes(version) 
WHERE is_active = TRUE;

-- Composite index for filtered listings with relationships
CREATE INDEX idx_thing_class_attributes_filtered_list ON thing_class_attributes(
    is_active, assignment_type, visibility, sort_order
) WHERE is_active = TRUE;
```

### Schema Composition Optimization

```sql
-- Materialized view for optimized Thing Class schema composition
CREATE MATERIALIZED VIEW thing_class_schemas AS
SELECT 
    tc.id as thing_class_id,
    tc.name as thing_class_name,
    tc.description as thing_class_description,
    tc.created_at as thing_class_created_at,
    tc.updated_at as thing_class_updated_at,
    tc.version as thing_class_version,
    
    -- Assignment statistics
    COUNT(tca.id) as total_assignments,
    COUNT(CASE WHEN tca.is_required = TRUE THEN 1 END) as required_assignments,
    COUNT(CASE WHEN tca.is_required = FALSE THEN 1 END) as optional_assignments,
    COUNT(CASE WHEN tca.default_value IS NOT NULL THEN 1 END) as assignments_with_defaults,
    COUNT(CASE WHEN tca.assignment_validation_rules != '{}' THEN 1 END) as assignments_with_custom_validation,
    COUNT(CASE WHEN tca.visibility = 'hidden' THEN 1 END) as hidden_assignments,
    COUNT(CASE WHEN tca.assignment_type = 'computed' THEN 1 END) as computed_assignments,
    
    -- Assignment arrays for quick access
    ARRAY_AGG(
        tca.id ORDER BY tca.sort_order ASC NULLS LAST, tca.created_at ASC
    ) FILTER (WHERE tca.id IS NOT NULL) as assignment_ids,
    
    ARRAY_AGG(
        tca.attribute_id ORDER BY tca.sort_order ASC NULLS LAST, tca.created_at ASC
    ) FILTER (WHERE tca.id IS NOT NULL) as attribute_ids,
    
    -- Comprehensive assignment summary JSON
    ARRAY_AGG(
        jsonb_build_object(
            'assignmentId', tca.id,
            'attributeId', tca.attribute_id,
            'attributeName', a.name,
            'attributeDataType', a.data_type,
            'displayName', COALESCE(tca.display_name, a.name),
            'isRequired', tca.is_required,
            'sortOrder', tca.sort_order,
            'hasDefaultValue', (tca.default_value IS NOT NULL),
            'hasCustomValidation', (tca.assignment_validation_rules != '{}'),
            'assignmentType', tca.assignment_type,
            'visibility', tca.visibility,
            'description', COALESCE(tca.description, a.description),
            'createdAt', tca.created_at,
            'updatedAt', tca.updated_at
        ) ORDER BY tca.sort_order ASC NULLS LAST, tca.created_at ASC
    ) FILTER (WHERE tca.id IS NOT NULL) as assignment_summary,
    
    -- Schema metadata
    MAX(tca.updated_at) as schema_last_modified,
    COUNT(DISTINCT tca.updated_by) as schema_contributors,
    
    -- Schema hash for change detection (simplified)
    md5(
        COALESCE(
            string_agg(
                tca.id::text || tca.is_required::text || COALESCE(tca.sort_order::text, '') || tca.version::text,
                '|' ORDER BY tca.sort_order ASC NULLS LAST, tca.created_at ASC
            ),
            ''
        )
    ) as schema_hash

FROM thing_classes tc
LEFT JOIN thing_class_attributes tca ON tc.id = tca.thing_class_id AND tca.is_active = TRUE
LEFT JOIN attributes a ON tca.attribute_id = a.id AND a.is_active = TRUE
WHERE tc.is_active = TRUE
GROUP BY tc.id, tc.name, tc.description, tc.created_at, tc.updated_at, tc.version;

-- Indexes on materialized view for optimal query performance
CREATE UNIQUE INDEX idx_thing_class_schemas_thing_class_id ON thing_class_schemas(thing_class_id);
CREATE INDEX idx_thing_class_schemas_total_assignments ON thing_class_schemas(total_assignments);
CREATE INDEX idx_thing_class_schemas_required_assignments ON thing_class_schemas(required_assignments);
CREATE INDEX idx_thing_class_schemas_schema_last_modified ON thing_class_schemas(schema_last_modified DESC);
CREATE INDEX idx_thing_class_schemas_schema_hash ON thing_class_schemas(schema_hash);

-- Function to refresh materialized view efficiently
CREATE OR REPLACE FUNCTION refresh_thing_class_schemas(p_thing_class_id UUID DEFAULT NULL)
RETURNS VOID AS $$
BEGIN
    IF p_thing_class_id IS NOT NULL THEN
        -- Refresh specific Thing Class (not directly supported in PostgreSQL, so full refresh)
        REFRESH MATERIALIZED VIEW CONCURRENTLY thing_class_schemas;
    ELSE
        -- Full refresh
        REFRESH MATERIALIZED VIEW CONCURRENTLY thing_class_schemas;
    END IF;
    
    -- Log refresh operation
    INSERT INTO system_maintenance_log (operation, completed_at, details)
    VALUES ('refresh_thing_class_schemas', NOW(), 
            jsonb_build_object(
                'thing_class_id', p_thing_class_id,
                'operation', 'materialized_view_refresh'
            ));
END;
$$ LANGUAGE plpgsql;
```

### Comprehensive Audit and Change Tracking

```sql
-- Enhanced audit table for comprehensive assignment change tracking
CREATE TABLE thing_class_attributes_audit (
    -- Audit record identification
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assignment_id UUID NOT NULL,
    operation VARCHAR(10) NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
    
    -- Change data
    old_values JSONB,
    new_values JSONB,
    field_changes JSONB DEFAULT '{}', -- Specific fields that changed
    
    -- Change metadata
    changed_by UUID NOT NULL,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    change_reason TEXT,
    client_info JSONB DEFAULT '{}',
    
    -- Impact analysis
    impact_analysis JSONB DEFAULT '{}',
    affected_thing_count INTEGER DEFAULT 0,
    schema_version_before INTEGER,
    schema_version_after INTEGER,
    
    -- Change categorization
    change_type VARCHAR(50) DEFAULT 'configuration' CHECK (
        change_type IN ('configuration', 'constraint', 'metadata', 'structural')
    ),
    impact_level VARCHAR(20) DEFAULT 'low' CHECK (
        impact_level IN ('low', 'medium', 'high', 'breaking')
    ),
    
    -- Rollback information
    rollback_available BOOLEAN DEFAULT TRUE,
    rollback_data JSONB,
    rollback_instructions TEXT,
    
    -- Foreign key to original assignment (nullable for DELETE operations)
    CONSTRAINT fk_audit_assignment FOREIGN KEY (assignment_id) 
        REFERENCES thing_class_attributes(id) ON DELETE SET NULL
);

-- Comprehensive audit indexes for performance and analysis
CREATE INDEX idx_thing_class_attributes_audit_assignment_id ON thing_class_attributes_audit(assignment_id);
CREATE INDEX idx_thing_class_attributes_audit_changed_at ON thing_class_attributes_audit(changed_at DESC);
CREATE INDEX idx_thing_class_attributes_audit_changed_by ON thing_class_attributes_audit(changed_by);
CREATE INDEX idx_thing_class_attributes_audit_operation ON thing_class_attributes_audit(operation);
CREATE INDEX idx_thing_class_attributes_audit_change_type ON thing_class_attributes_audit(change_type);
CREATE INDEX idx_thing_class_attributes_audit_impact_level ON thing_class_attributes_audit(impact_level);

-- GIN index for field changes analysis
CREATE INDEX idx_thing_class_attributes_audit_field_changes ON thing_class_attributes_audit 
USING GIN(field_changes);

-- GIN index for impact analysis data
CREATE INDEX idx_thing_class_attributes_audit_impact_analysis ON thing_class_attributes_audit 
USING GIN(impact_analysis);

-- Advanced audit trigger with comprehensive impact analysis
CREATE OR REPLACE FUNCTION thing_class_attributes_audit_trigger()
RETURNS TRIGGER AS $$
DECLARE
    change_reason_text TEXT;
    client_context JSONB;
    impact_data JSONB;
    field_changes_data JSONB;
    thing_count INTEGER;
    change_type_val VARCHAR(50);
    impact_level_val VARCHAR(20);
BEGIN
    -- Extract change context from session variables
    change_reason_text := COALESCE(
        current_setting('udm.change_reason', true),
        CASE TG_OP
            WHEN 'INSERT' THEN 'Assignment created'
            WHEN 'UPDATE' THEN 'Assignment updated'  
            WHEN 'DELETE' THEN 'Assignment deleted'
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
        -- Count affected Things for impact analysis
        SELECT COUNT(*) INTO thing_count
        FROM things 
        WHERE thing_class_id = NEW.thing_class_id AND is_active = TRUE;
        
        impact_data := jsonb_build_object(
            'thing_class_id', NEW.thing_class_id,
            'thing_class_name', (SELECT name FROM thing_classes WHERE id = NEW.thing_class_id),
            'attribute_id', NEW.attribute_id,
            'attribute_name', (SELECT name FROM attributes WHERE id = NEW.attribute_id),
            'attribute_data_type', (SELECT data_type FROM attributes WHERE id = NEW.attribute_id),
            'is_required', NEW.is_required,
            'has_default_value', (NEW.default_value IS NOT NULL),
            'has_custom_validation', (NEW.assignment_validation_rules != '{}'),
            'affected_things', thing_count,
            'potential_impact', CASE 
                WHEN NEW.is_required = TRUE AND NEW.default_value IS NULL THEN 'requires_data_entry'
                WHEN NEW.is_required = TRUE AND NEW.default_value IS NOT NULL THEN 'automatic_defaults'
                ELSE 'optional_extension'
            END
        );
        
        change_type_val := CASE 
            WHEN NEW.assignment_type = 'computed' THEN 'structural'
            WHEN NEW.is_required = TRUE THEN 'constraint'
            ELSE 'configuration'
        END;
        
        impact_level_val := CASE
            WHEN NEW.is_required = TRUE AND thing_count > 0 THEN 'high'
            WHEN thing_count > 100 THEN 'medium'
            ELSE 'low'
        END;
        
        INSERT INTO thing_class_attributes_audit (
            assignment_id, operation, new_values, changed_by, 
            change_reason, client_info, impact_analysis, affected_thing_count,
            schema_version_after, change_type, impact_level, rollback_data
        )
        VALUES (
            NEW.id, 'INSERT', to_jsonb(NEW), NEW.created_by,
            change_reason_text, client_context, impact_data, thing_count,
            NEW.version, change_type_val, impact_level_val, to_jsonb(NEW)
        );
        RETURN NEW;
        
    ELSIF TG_OP = 'UPDATE' THEN
        -- Analyze specific field changes
        field_changes_data := jsonb_build_object();
        
        IF OLD.is_required != NEW.is_required THEN
            field_changes_data := field_changes_data || jsonb_build_object(
                'is_required', jsonb_build_object(
                    'old', OLD.is_required,
                    'new', NEW.is_required,
                    'impact', CASE 
                        WHEN OLD.is_required = FALSE AND NEW.is_required = TRUE THEN 'breaking_change'
                        WHEN OLD.is_required = TRUE AND NEW.is_required = FALSE THEN 'relaxing_change'
                    END
                )
            );
        END IF;
        
        IF OLD.sort_order != NEW.sort_order THEN
            field_changes_data := field_changes_data || jsonb_build_object(
                'sort_order', jsonb_build_object(
                    'old', OLD.sort_order,
                    'new', NEW.sort_order,
                    'impact', 'presentation_change'
                )
            );
        END IF;
        
        IF OLD.assignment_validation_rules != NEW.assignment_validation_rules THEN
            field_changes_data := field_changes_data || jsonb_build_object(
                'assignment_validation_rules', jsonb_build_object(
                    'old', OLD.assignment_validation_rules,
                    'new', NEW.assignment_validation_rules,
                    'impact', 'validation_change'
                )
            );
        END IF;
        
        IF COALESCE(OLD.default_value, 'null'::jsonb) != COALESCE(NEW.default_value, 'null'::jsonb) THEN
            field_changes_data := field_changes_data || jsonb_build_object(
                'default_value', jsonb_build_object(
                    'old', OLD.default_value,
                    'new', NEW.default_value,
                    'impact', 'default_value_change'
                )
            );
        END IF;
        
        -- Count affected Things
        SELECT COUNT(*) INTO thing_count
        FROM things 
        WHERE thing_class_id = NEW.thing_class_id AND is_active = TRUE;
        
        impact_data := jsonb_build_object(
            'thing_class_id', NEW.thing_class_id,
            'thing_class_name', (SELECT name FROM thing_classes WHERE id = NEW.thing_class_id),
            'attribute_id', NEW.attribute_id,
            'attribute_name', (SELECT name FROM attributes WHERE id = NEW.attribute_id),
            'fields_changed', jsonb_object_keys(field_changes_data),
            'affected_things', thing_count,
            'change_summary', field_changes_data
        );
        
        -- Determine change type and impact level
        change_type_val := CASE 
            WHEN field_changes_data ? 'is_required' THEN 'constraint'
            WHEN field_changes_data ? 'assignment_validation_rules' THEN 'constraint'
            WHEN field_changes_data ? 'default_value' THEN 'configuration'
            ELSE 'metadata'
        END;
        
        impact_level_val := CASE
            WHEN (field_changes_data -> 'is_required' ->> 'impact') = 'breaking_change' AND thing_count > 0 THEN 'breaking'
            WHEN thing_count > 1000 AND change_type_val = 'constraint' THEN 'high'
            WHEN thing_count > 100 THEN 'medium'
            ELSE 'low'
        END;
        
        INSERT INTO thing_class_attributes_audit (
            assignment_id, operation, old_values, new_values, field_changes, changed_by,
            change_reason, client_info, impact_analysis, affected_thing_count,
            schema_version_before, schema_version_after, change_type, impact_level, rollback_data
        )
        VALUES (
            NEW.id, 'UPDATE', to_jsonb(OLD), to_jsonb(NEW), field_changes_data,
            COALESCE(NEW.updated_by, NEW.created_by),
            change_reason_text, client_context, impact_data, thing_count,
            OLD.version, NEW.version, change_type_val, impact_level_val, to_jsonb(OLD)
        );
        RETURN NEW;
        
    ELSIF TG_OP = 'DELETE' THEN
        -- Count affected Things
        SELECT COUNT(*) INTO thing_count
        FROM things 
        WHERE thing_class_id = OLD.thing_class_id AND is_active = TRUE;
        
        impact_data := jsonb_build_object(
            'thing_class_id', OLD.thing_class_id,
            'thing_class_name', (SELECT name FROM thing_classes WHERE id = OLD.thing_class_id),
            'attribute_id', OLD.attribute_id,
            'attribute_name', (SELECT name FROM attributes WHERE id = OLD.attribute_id),
            'was_required', OLD.is_required,
            'had_default_value', (OLD.default_value IS NOT NULL),
            'affected_things', thing_count,
            'potential_impact', CASE 
                WHEN OLD.is_required = TRUE THEN 'breaking_change'
                ELSE 'schema_reduction'
            END
        );
        
        change_type_val := CASE 
            WHEN OLD.is_required = TRUE THEN 'structural'
            ELSE 'configuration'
        END;
        
        impact_level_val := CASE
            WHEN OLD.is_required = TRUE AND thing_count > 0 THEN 'breaking'
            WHEN thing_count > 100 THEN 'high'
            ELSE 'medium'
        END;
        
        INSERT INTO thing_class_attributes_audit (
            assignment_id, operation, old_values, changed_by,
            change_reason, client_info, impact_analysis, affected_thing_count,
            schema_version_before, change_type, impact_level, rollback_data,
            rollback_instructions
        )
        VALUES (
            OLD.id, 'DELETE', to_jsonb(OLD),
            COALESCE(OLD.updated_by, OLD.created_by),
            change_reason_text, client_context, impact_data, thing_count,
            OLD.version, change_type_val, impact_level_val, to_jsonb(OLD),
            'To restore this assignment, create new assignment with the provided rollback_data'
        );
        RETURN OLD;
    END IF;
    
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Create comprehensive audit trigger
CREATE TRIGGER thing_class_attributes_audit_trig
    AFTER INSERT OR UPDATE OR DELETE ON thing_class_attributes
    FOR EACH ROW EXECUTE FUNCTION thing_class_attributes_audit_trigger();
```

### Validation and Integrity Functions

```sql
-- Function to validate assignment configuration
CREATE OR REPLACE FUNCTION validate_assignment_configuration(
    p_thing_class_id UUID,
    p_attribute_id UUID,
    p_assignment_validation_rules JSONB,
    p_default_value JSONB
) RETURNS BOOLEAN AS $$
DECLARE
    attribute_record RECORD;
    validation_processor VARCHAR(50);
BEGIN
    -- Get attribute information
    SELECT id, name, data_type, validation_rules 
    INTO attribute_record
    FROM attributes 
    WHERE id = p_attribute_id AND is_active = TRUE;
    
    -- Ensure attribute exists
    IF attribute_record.id IS NULL THEN
        RAISE EXCEPTION 'Attribute % not found or inactive', p_attribute_id;
    END IF;
    
    -- Validate assignment validation rules format
    IF p_assignment_validation_rules IS NOT NULL AND p_assignment_validation_rules != '{}' THEN
        -- Basic JSON structure validation
        IF jsonb_typeof(p_assignment_validation_rules) != 'object' THEN
            RAISE EXCEPTION 'Assignment validation rules must be a JSON object';
        END IF;
    END IF;
    
    -- Validate default value against attribute data type
    IF p_default_value IS NOT NULL THEN
        -- This is a simplified validation - in production, would use data-type-specific validation
        CASE attribute_record.data_type
            WHEN 'string' THEN
                IF jsonb_typeof(p_default_value) NOT IN ('string', 'null') THEN
                    RAISE EXCEPTION 'Default value for string attribute must be a string or null';
                END IF;
            WHEN 'integer' THEN
                IF jsonb_typeof(p_default_value) NOT IN ('number', 'null') THEN
                    RAISE EXCEPTION 'Default value for integer attribute must be a number or null';
                END IF;
            WHEN 'boolean' THEN
                IF jsonb_typeof(p_default_value) NOT IN ('boolean', 'null') THEN
                    RAISE EXCEPTION 'Default value for boolean attribute must be a boolean or null';
                END IF;
            -- Add other data type validations as needed
        END CASE;
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Function to validate sort order uniqueness within Thing Class
CREATE OR REPLACE FUNCTION validate_sort_order_uniqueness(
    p_thing_class_id UUID,
    p_sort_order INTEGER,
    p_assignment_id UUID DEFAULT NULL
) RETURNS BOOLEAN AS $$
DECLARE
    conflict_count INTEGER;
BEGIN
    SELECT COUNT(*)
    INTO conflict_count
    FROM thing_class_attributes
    WHERE thing_class_id = p_thing_class_id
      AND sort_order = p_sort_order
      AND is_active = TRUE
      AND (p_assignment_id IS NULL OR id != p_assignment_id);
    
    IF conflict_count > 0 THEN
        RAISE EXCEPTION 'Sort order % already exists for Thing Class %. Sort orders must be unique within each Thing Class.', 
                       p_sort_order, p_thing_class_id;
    END IF;
    
    RETURN TRUE;
END;
$$ LANGUAGE plpgsql;

-- Constraint trigger for sort order uniqueness
CREATE OR REPLACE FUNCTION enforce_sort_order_uniqueness()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM validate_sort_order_uniqueness(NEW.thing_class_id, NEW.sort_order, NEW.id);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE CONSTRAINT TRIGGER sort_order_uniqueness_trigger
    AFTER INSERT OR UPDATE ON thing_class_attributes
    DEFERRABLE INITIALLY DEFERRED
    FOR EACH ROW
    EXECUTE FUNCTION enforce_sort_order_uniqueness();
```

### Multi-Tenant Schema Management

```sql
-- Function to create complete assignment schema for new tenant
CREATE OR REPLACE FUNCTION create_tenant_assignment_schema(p_schema_name TEXT) 
RETURNS BOOLEAN AS $$
DECLARE
    sql_statement TEXT;
BEGIN
    -- Set search path to new tenant schema
    EXECUTE format('SET search_path TO %I, public', p_schema_name);
    
    -- Create main assignment table with all constraints and indexes
    -- (Full table creation SQL would go here - abbreviated for space)
    
    -- Create materialized view
    EXECUTE format('CREATE MATERIALIZED VIEW %I.thing_class_schemas AS (SELECT ...)', p_schema_name);
    
    -- Create audit table
    EXECUTE format('CREATE TABLE %I.thing_class_attributes_audit (...)', p_schema_name);
    
    -- Create all indexes
    -- (All index creation SQL would go here)
    
    -- Create functions and triggers
    -- (Function and trigger creation SQL would go here)
    
    -- Grant appropriate permissions
    EXECUTE format('GRANT SELECT, INSERT, UPDATE, DELETE ON %I.thing_class_attributes TO udm_application', p_schema_name);
    EXECUTE format('GRANT SELECT ON %I.thing_class_attributes_audit TO udm_application', p_schema_name);
    EXECUTE format('GRANT SELECT ON %I.thing_class_schemas TO udm_application', p_schema_name);
    
    RETURN TRUE;
    
EXCEPTION
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Failed to create tenant assignment schema for %: %', p_schema_name, SQLERRM;
        RETURN FALSE;
END;
$$ LANGUAGE plpgsql;
```

### Database Maintenance and Optimization

```sql
-- Function for assignment statistics and health monitoring
CREATE OR REPLACE FUNCTION get_assignment_statistics()
RETURNS TABLE (
    metric VARCHAR(50),
    value BIGINT,
    percentage NUMERIC(5,2),
    description TEXT
) AS $$
BEGIN
    RETURN QUERY
    WITH stats AS (
        SELECT 
            COUNT(*) as total_assignments,
            COUNT(*) FILTER (WHERE is_required = TRUE) as required_assignments,
            COUNT(*) FILTER (WHERE default_value IS NOT NULL) as assignments_with_defaults,
            COUNT(*) FILTER (WHERE assignment_validation_rules != '{}') as assignments_with_custom_validation,
            COUNT(*) FILTER (WHERE assignment_type = 'computed') as computed_assignments,
            COUNT(*) FILTER (WHERE visibility = 'hidden') as hidden_assignments,
            COUNT(DISTINCT thing_class_id) as thing_classes_with_assignments,
            COUNT(DISTINCT attribute_id) as attributes_in_use
        FROM thing_class_attributes
        WHERE is_active = TRUE
    )
    SELECT 'total_assignments'::VARCHAR(50), total_assignments, 100.0, 'Total active assignments'::TEXT FROM stats
    UNION ALL
    SELECT 'required_assignments', required_assignments, 
           ROUND((required_assignments::NUMERIC / NULLIF(total_assignments, 0)) * 100, 2),
           'Assignments marked as required' FROM stats
    UNION ALL
    SELECT 'assignments_with_defaults', assignments_with_defaults,
           ROUND((assignments_with_defaults::NUMERIC / NULLIF(total_assignments, 0)) * 100, 2),
           'Assignments with default values' FROM stats
    UNION ALL
    SELECT 'assignments_with_custom_validation', assignments_with_custom_validation,
           ROUND((assignments_with_custom_validation::NUMERIC / NULLIF(total_assignments, 0)) * 100, 2),
           'Assignments with custom validation rules' FROM stats
    UNION ALL
    SELECT 'computed_assignments', computed_assignments,
           ROUND((computed_assignments::NUMERIC / NULLIF(total_assignments, 0)) * 100, 2),
           'Computed or derived assignments' FROM stats
    UNION ALL
    SELECT 'hidden_assignments', hidden_assignments,
           ROUND((hidden_assignments::NUMERIC / NULLIF(total_assignments, 0)) * 100, 2),
           'Assignments hidden from UI' FROM stats
    UNION ALL
    SELECT 'thing_classes_with_assignments', thing_classes_with_assignments, NULL,
           'Thing Classes that have at least one assignment' FROM stats
    UNION ALL
    SELECT 'attributes_in_use', attributes_in_use, NULL,
           'Attributes currently assigned to Thing Classes' FROM stats;
END;
$$ LANGUAGE plpgsql;

-- Cleanup function for old audit records
CREATE OR REPLACE FUNCTION cleanup_assignment_audit_records(p_retain_days INTEGER DEFAULT 730)
RETURNS INTEGER AS $$
DECLARE
    deleted_count INTEGER;
BEGIN
    DELETE FROM thing_class_attributes_audit 
    WHERE changed_at < NOW() - (p_retain_days || ' days')::INTERVAL
      AND impact_level NOT IN ('breaking', 'high'); -- Keep high-impact changes longer
    
    GET DIAGNOSTICS deleted_count = ROW_COUNT;
    
    -- Log cleanup operation
    INSERT INTO system_maintenance_log (operation, completed_at, details)
    VALUES ('cleanup_assignment_audit', NOW(), 
            jsonb_build_object('deleted_records', deleted_count, 'retention_days', p_retain_days));
    
    RETURN deleted_count;
END;
$$ LANGUAGE plpgsql;

-- Function to recompute and update materialized view
CREATE OR REPLACE FUNCTION update_assignment_statistics()
RETURNS VOID AS $$
BEGIN
    -- Refresh materialized view
    REFRESH MATERIALIZED VIEW CONCURRENTLY thing_class_schemas;
    
    -- Update table statistics
    ANALYZE thing_class_attributes;
    ANALYZE thing_class_attributes_audit;
    
    -- Log maintenance completion
    INSERT INTO system_maintenance_log (operation, completed_at, details)
    VALUES ('update_assignment_statistics', NOW(), 
            jsonb_build_object('operation', 'analyze_assignments', 'status', 'completed'));
END;
$$ LANGUAGE plpgsql;
```

### Migration Scripts

```sql
-- Migration: 004_create_attribute_assignments.up.sql
-- Description: Create Attribute Assignment entity schema with comprehensive features

BEGIN;

-- Ensure required extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";

-- Create main assignment table with comprehensive constraints
-- (Full table creation from above would go here)

-- Create all performance indexes
-- (All index creation from above would go here)

-- Create materialized view for schema composition
-- (Materialized view creation from above would go here)

-- Create audit table with comprehensive tracking
-- (Audit table creation from above would go here)

-- Create all validation and utility functions
-- (All function definitions from above would go here)

-- Create triggers for audit and validation
-- (All trigger creation from above would go here)

-- Insert initial system data if needed
INSERT INTO system_configuration (key, value, description) VALUES
('assignment_max_sort_order', '999999', 'Maximum allowed sort order for assignments'),
('assignment_audit_retention_days', '730', 'Days to retain assignment audit records'),
('assignment_cache_ttl_minutes', '30', 'Cache TTL for assignment data in minutes');

COMMIT;
```

This comprehensive database schema specification provides a robust foundation for the Attribute Assignment entity with full multi-tenant support, comprehensive auditing, performance optimization, and data integrity enforcement.