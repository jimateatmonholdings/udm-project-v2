# Database Schema

This is the database schema implementation for the spec detailed in @.agent-os/specs/2025-08-30-relationship-entity/spec.md

> Created: 2025-08-30
> Version: 1.0.0

## Schema Changes

### New Tables

#### relationships
```sql
CREATE TABLE relationships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    relationship_type_id UUID NOT NULL, -- References thing_classes.id
    source_thing_id UUID NOT NULL,      -- References things.id
    target_thing_id UUID NOT NULL,      -- References things.id
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    metadata JSONB DEFAULT '{}',
    is_bidirectional BOOLEAN DEFAULT false,
    effective_from TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    effective_until TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    created_by UUID,
    updated_by UUID,
    version INTEGER DEFAULT 1,
    
    -- Constraints
    CONSTRAINT relationships_tenant_id_fk 
        FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE,
    CONSTRAINT relationships_relationship_type_fk 
        FOREIGN KEY (tenant_id, relationship_type_id) 
        REFERENCES thing_classes(tenant_id, id) ON DELETE RESTRICT,
    CONSTRAINT relationships_source_thing_fk 
        FOREIGN KEY (tenant_id, source_thing_id) 
        REFERENCES things(tenant_id, id) ON DELETE CASCADE,
    CONSTRAINT relationships_target_thing_fk 
        FOREIGN KEY (tenant_id, target_thing_id) 
        REFERENCES things(tenant_id, id) ON DELETE CASCADE,
    CONSTRAINT relationships_status_check 
        CHECK (status IN ('active', 'inactive', 'pending', 'deleted')),
    CONSTRAINT relationships_no_self_reference 
        CHECK (source_thing_id != target_thing_id),
    CONSTRAINT relationships_effective_dates_check 
        CHECK (effective_until IS NULL OR effective_until > effective_from)
);
```

#### relationship_constraints
```sql
CREATE TABLE relationship_constraints (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    relationship_type_id UUID NOT NULL,
    source_thing_class_id UUID,      -- NULL means any class allowed
    target_thing_class_id UUID,      -- NULL means any class allowed
    min_instances INTEGER DEFAULT 0,
    max_instances INTEGER,           -- NULL means unlimited
    is_required BOOLEAN DEFAULT false,
    constraint_type VARCHAR(50) NOT NULL,
    constraint_metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    
    CONSTRAINT relationship_constraints_tenant_id_fk 
        FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE,
    CONSTRAINT relationship_constraints_relationship_type_fk 
        FOREIGN KEY (tenant_id, relationship_type_id) 
        REFERENCES thing_classes(tenant_id, id) ON DELETE CASCADE,
    CONSTRAINT relationship_constraints_source_class_fk 
        FOREIGN KEY (tenant_id, source_thing_class_id) 
        REFERENCES thing_classes(tenant_id, id) ON DELETE CASCADE,
    CONSTRAINT relationship_constraints_target_class_fk 
        FOREIGN KEY (tenant_id, target_thing_class_id) 
        REFERENCES thing_classes(tenant_id, id) ON DELETE CASCADE,
    CONSTRAINT relationship_constraints_min_max_check 
        CHECK (max_instances IS NULL OR max_instances >= min_instances),
    CONSTRAINT relationship_constraints_type_check 
        CHECK (constraint_type IN ('cardinality', 'exclusivity', 'dependency', 'hierarchy'))
);
```

#### relationship_history
```sql
CREATE TABLE relationship_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL,
    relationship_id UUID NOT NULL,
    operation_type VARCHAR(20) NOT NULL,
    old_values JSONB,
    new_values JSONB,
    changed_by UUID,
    changed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    change_reason TEXT,
    
    CONSTRAINT relationship_history_tenant_id_fk 
        FOREIGN KEY (tenant_id) REFERENCES tenants(id) ON DELETE CASCADE,
    CONSTRAINT relationship_history_operation_check 
        CHECK (operation_type IN ('created', 'updated', 'deleted', 'activated', 'deactivated'))
);
```

### Indexes

#### Performance Indexes
```sql
-- Primary relationship lookups
CREATE INDEX idx_relationships_tenant_source_type_status 
ON relationships (tenant_id, source_thing_id, relationship_type_id, status)
WHERE status = 'active';

CREATE INDEX idx_relationships_tenant_target_type_status 
ON relationships (tenant_id, target_thing_id, relationship_type_id, status)
WHERE status = 'active';

-- Bidirectional relationship queries
CREATE INDEX idx_relationships_tenant_source_target_type 
ON relationships (tenant_id, source_thing_id, target_thing_id, relationship_type_id);

-- Relationship type queries
CREATE INDEX idx_relationships_tenant_type_status 
ON relationships (tenant_id, relationship_type_id, status);

-- Effective date queries
CREATE INDEX idx_relationships_tenant_effective_dates 
ON relationships (tenant_id, effective_from, effective_until)
WHERE status = 'active';

-- Metadata searches
CREATE GIN INDEX idx_relationships_metadata 
ON relationships USING gin (metadata);
```

#### Constraint Indexes
```sql
CREATE INDEX idx_relationship_constraints_tenant_type 
ON relationship_constraints (tenant_id, relationship_type_id);

CREATE INDEX idx_relationship_constraints_source_target 
ON relationship_constraints (tenant_id, source_thing_class_id, target_thing_class_id);
```

#### History Indexes
```sql
CREATE INDEX idx_relationship_history_tenant_relationship 
ON relationship_history (tenant_id, relationship_id, changed_at DESC);

CREATE INDEX idx_relationship_history_tenant_operation 
ON relationship_history (tenant_id, operation_type, changed_at DESC);
```

### Views

#### active_relationships
```sql
CREATE VIEW active_relationships AS
SELECT 
    r.*,
    tc.name as relationship_type_name,
    st.name as source_thing_name,
    tt.name as target_thing_name
FROM relationships r
JOIN thing_classes tc ON r.tenant_id = tc.tenant_id AND r.relationship_type_id = tc.id
JOIN things st ON r.tenant_id = st.tenant_id AND r.source_thing_id = st.id
JOIN things tt ON r.tenant_id = tt.tenant_id AND r.target_thing_id = tt.id
WHERE r.status = 'active'
AND (r.effective_until IS NULL OR r.effective_until > CURRENT_TIMESTAMP);
```

#### relationship_counts
```sql
CREATE VIEW relationship_counts AS
SELECT 
    tenant_id,
    relationship_type_id,
    source_thing_id,
    target_thing_id,
    COUNT(*) as relationship_count
FROM relationships
WHERE status = 'active'
GROUP BY tenant_id, relationship_type_id, source_thing_id, target_thing_id;
```

## Migrations

### Migration 001: Create Core Relationships Table
```sql
-- Up
CREATE TABLE relationships (
    -- Core fields as defined above
);

-- Down
DROP TABLE IF EXISTS relationships CASCADE;
```

### Migration 002: Create Relationship Constraints Table
```sql
-- Up
CREATE TABLE relationship_constraints (
    -- Constraint fields as defined above
);

-- Down
DROP TABLE IF EXISTS relationship_constraints CASCADE;
```

### Migration 003: Create Relationship History Table
```sql
-- Up
CREATE TABLE relationship_history (
    -- History fields as defined above
);

-- Down
DROP TABLE IF EXISTS relationship_history CASCADE;
```

### Migration 004: Create Indexes
```sql
-- Up
-- All indexes as defined above

-- Down
-- Drop all relationship-related indexes
```

### Migration 005: Create Views
```sql
-- Up
-- All views as defined above

-- Down
DROP VIEW IF EXISTS relationship_counts;
DROP VIEW IF EXISTS active_relationships;
```

### Migration 006: Add Relationship Triggers
```sql
-- Up
-- Trigger for updating updated_at
CREATE OR REPLACE FUNCTION update_relationships_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = CURRENT_TIMESTAMP;
    NEW.version = OLD.version + 1;
    RETURN NEW;
END;
$$ language 'plpgsql';

CREATE TRIGGER relationships_update_trigger
    BEFORE UPDATE ON relationships
    FOR EACH ROW
    EXECUTE FUNCTION update_relationships_updated_at();

-- Trigger for relationship history
CREATE OR REPLACE FUNCTION log_relationship_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        INSERT INTO relationship_history (
            tenant_id, relationship_id, operation_type, 
            old_values, changed_by, change_reason
        ) VALUES (
            OLD.tenant_id, OLD.id, 'deleted',
            to_jsonb(OLD), OLD.updated_by, 'Record deleted'
        );
        RETURN OLD;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO relationship_history (
            tenant_id, relationship_id, operation_type,
            old_values, new_values, changed_by, change_reason
        ) VALUES (
            NEW.tenant_id, NEW.id, 'updated',
            to_jsonb(OLD), to_jsonb(NEW), NEW.updated_by, 'Record updated'
        );
        RETURN NEW;
    ELSIF TG_OP = 'INSERT' THEN
        INSERT INTO relationship_history (
            tenant_id, relationship_id, operation_type,
            new_values, changed_by, change_reason
        ) VALUES (
            NEW.tenant_id, NEW.id, 'created',
            to_jsonb(NEW), NEW.created_by, 'Record created'
        );
        RETURN NEW;
    END IF;
    RETURN NULL;
END;
$$ language 'plpgsql';

CREATE TRIGGER relationships_history_trigger
    AFTER INSERT OR UPDATE OR DELETE ON relationships
    FOR EACH ROW
    EXECUTE FUNCTION log_relationship_changes();

-- Down
DROP TRIGGER IF EXISTS relationships_history_trigger ON relationships;
DROP TRIGGER IF EXISTS relationships_update_trigger ON relationships;
DROP FUNCTION IF EXISTS log_relationship_changes();
DROP FUNCTION IF EXISTS update_relationships_updated_at();
```