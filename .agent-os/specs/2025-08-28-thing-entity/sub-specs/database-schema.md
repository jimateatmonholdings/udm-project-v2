# Database Schema

This is the database schema implementation for the spec detailed in @.agent-os/specs/2025-08-28-thing-entity/spec.md

> Created: 2025-08-28
> Version: 1.0.0

## Schema Changes

### Things Table

The core `things` table structure based on UDM PRD specifications:

```sql
CREATE TABLE things (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thing_class_id UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    created_by UUID NOT NULL,
    updated_by UUID NOT NULL,
    version INTEGER DEFAULT 1 NOT NULL,
    is_deleted BOOLEAN DEFAULT FALSE NOT NULL,
    
    -- Constraints
    CONSTRAINT fk_things_thing_class FOREIGN KEY (thing_class_id) REFERENCES thing_classes(id),
    CONSTRAINT fk_things_created_by FOREIGN KEY (created_by) REFERENCES users(id),
    CONSTRAINT fk_things_updated_by FOREIGN KEY (updated_by) REFERENCES users(id),
    CONSTRAINT check_version_positive CHECK (version > 0)
);
```

### Values Table

The `values` table for storing Thing attribute values with JSONB support:

```sql
CREATE TABLE values (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thing_id UUID NOT NULL,
    attribute_id UUID NOT NULL,
    value_data JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    created_by UUID NOT NULL,
    updated_by UUID NOT NULL,
    is_deleted BOOLEAN DEFAULT FALSE NOT NULL,
    
    -- Constraints
    CONSTRAINT fk_values_thing FOREIGN KEY (thing_id) REFERENCES things(id) ON DELETE CASCADE,
    CONSTRAINT fk_values_attribute FOREIGN KEY (attribute_id) REFERENCES attributes(id),
    CONSTRAINT fk_values_created_by FOREIGN KEY (created_by) REFERENCES users(id),
    CONSTRAINT fk_values_updated_by FOREIGN KEY (updated_by) REFERENCES users(id),
    
    -- Unique constraint to prevent duplicate attribute values per thing
    CONSTRAINT uk_values_thing_attribute UNIQUE (thing_id, attribute_id)
);
```

### Indexes for Performance Optimization

```sql
-- Primary indexes for things table
CREATE INDEX idx_things_thing_class_id ON things(thing_class_id) WHERE is_deleted = FALSE;
CREATE INDEX idx_things_created_by ON things(created_by);
CREATE INDEX idx_things_updated_by ON things(updated_by);
CREATE INDEX idx_things_created_at ON things(created_at);
CREATE INDEX idx_things_updated_at ON things(updated_at);
CREATE INDEX idx_things_is_deleted ON things(is_deleted);

-- Composite indexes for common query patterns
CREATE INDEX idx_things_class_created ON things(thing_class_id, created_at DESC) WHERE is_deleted = FALSE;
CREATE INDEX idx_things_class_updated ON things(thing_class_id, updated_at DESC) WHERE is_deleted = FALSE;

-- Indexes for values table
CREATE INDEX idx_values_thing_id ON values(thing_id) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_attribute_id ON values(attribute_id) WHERE is_deleted = FALSE;

-- GIN indexes for JSONB performance
CREATE INDEX idx_values_data_gin ON values USING GIN (value_data) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_data_jsonb_path_ops ON values USING GIN (value_data jsonb_path_ops) WHERE is_deleted = FALSE;

-- Composite indexes for values query patterns
CREATE INDEX idx_values_thing_attribute ON values(thing_id, attribute_id) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_created_at ON values(created_at);
CREATE INDEX idx_values_updated_at ON values(updated_at);
```

### Multi-Tenant Schema Considerations

The schema supports multi-tenancy through:

1. **User-based isolation**: All records are linked to users via `created_by` and `updated_by` fields
2. **Soft deletion**: Using `is_deleted` flag to maintain data integrity while allowing logical deletion
3. **Versioning**: Built-in version tracking for audit trails and conflict resolution

```sql
-- Additional indexes for multi-tenant queries
CREATE INDEX idx_things_tenant_isolation ON things(created_by, thing_class_id) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_tenant_isolation ON values(created_by, thing_id) WHERE is_deleted = FALSE;

-- Row-level security policies (example implementation)
ALTER TABLE things ENABLE ROW LEVEL SECURITY;
ALTER TABLE values ENABLE ROW LEVEL SECURITY;

-- Policies will be defined based on the application's tenant isolation requirements
```

### Relationship to Thing Classes Table

The `things` table maintains a foreign key relationship to the `thing_classes` table:

```sql
-- Ensure thing_classes table exists with proper structure
-- (This would be defined in the thing-classes schema spec)
CREATE TABLE IF NOT EXISTS thing_classes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    schema_definition JSONB,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    created_by UUID NOT NULL,
    updated_by UUID NOT NULL,
    is_deleted BOOLEAN DEFAULT FALSE NOT NULL
);

-- Index for thing_classes lookups
CREATE INDEX idx_thing_classes_name ON thing_classes(name) WHERE is_deleted = FALSE;
```

## Migrations

### Migration 001: Create Things and Values Tables

```sql
-- Migration: 001_create_things_and_values_tables.sql
-- Description: Initial creation of things and values tables with proper constraints and indexes

BEGIN;

-- Create things table
CREATE TABLE things (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thing_class_id UUID NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    created_by UUID NOT NULL,
    updated_by UUID NOT NULL,
    version INTEGER DEFAULT 1 NOT NULL,
    is_deleted BOOLEAN DEFAULT FALSE NOT NULL,
    
    CONSTRAINT fk_things_thing_class FOREIGN KEY (thing_class_id) REFERENCES thing_classes(id),
    CONSTRAINT fk_things_created_by FOREIGN KEY (created_by) REFERENCES users(id),
    CONSTRAINT fk_things_updated_by FOREIGN KEY (updated_by) REFERENCES users(id),
    CONSTRAINT check_version_positive CHECK (version > 0)
);

-- Create values table
CREATE TABLE values (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    thing_id UUID NOT NULL,
    attribute_id UUID NOT NULL,
    value_data JSONB NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    created_by UUID NOT NULL,
    updated_by UUID NOT NULL,
    is_deleted BOOLEAN DEFAULT FALSE NOT NULL,
    
    CONSTRAINT fk_values_thing FOREIGN KEY (thing_id) REFERENCES things(id) ON DELETE CASCADE,
    CONSTRAINT fk_values_attribute FOREIGN KEY (attribute_id) REFERENCES attributes(id),
    CONSTRAINT fk_values_created_by FOREIGN KEY (created_by) REFERENCES users(id),
    CONSTRAINT fk_values_updated_by FOREIGN KEY (updated_by) REFERENCES users(id),
    CONSTRAINT uk_values_thing_attribute UNIQUE (thing_id, attribute_id)
);

COMMIT;
```

### Migration 002: Create Performance Indexes

```sql
-- Migration: 002_create_performance_indexes.sql
-- Description: Add indexes for optimal query performance

BEGIN;

-- Things table indexes
CREATE INDEX idx_things_thing_class_id ON things(thing_class_id) WHERE is_deleted = FALSE;
CREATE INDEX idx_things_created_by ON things(created_by);
CREATE INDEX idx_things_updated_by ON things(updated_by);
CREATE INDEX idx_things_created_at ON things(created_at);
CREATE INDEX idx_things_updated_at ON things(updated_at);
CREATE INDEX idx_things_is_deleted ON things(is_deleted);
CREATE INDEX idx_things_class_created ON things(thing_class_id, created_at DESC) WHERE is_deleted = FALSE;
CREATE INDEX idx_things_class_updated ON things(thing_class_id, updated_at DESC) WHERE is_deleted = FALSE;

-- Values table indexes
CREATE INDEX idx_values_thing_id ON values(thing_id) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_attribute_id ON values(attribute_id) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_data_gin ON values USING GIN (value_data) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_data_jsonb_path_ops ON values USING GIN (value_data jsonb_path_ops) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_thing_attribute ON values(thing_id, attribute_id) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_created_at ON values(created_at);
CREATE INDEX idx_values_updated_at ON values(updated_at);

-- Multi-tenant indexes
CREATE INDEX idx_things_tenant_isolation ON things(created_by, thing_class_id) WHERE is_deleted = FALSE;
CREATE INDEX idx_values_tenant_isolation ON values(created_by, thing_id) WHERE is_deleted = FALSE;

COMMIT;
```

### Migration 003: Enable Row Level Security

```sql
-- Migration: 003_enable_row_level_security.sql
-- Description: Enable RLS for multi-tenant data isolation

BEGIN;

-- Enable RLS on tables
ALTER TABLE things ENABLE ROW LEVEL SECURITY;
ALTER TABLE values ENABLE ROW LEVEL SECURITY;

-- Note: Specific RLS policies will be defined based on application requirements
-- and tenant isolation strategy

COMMIT;
```