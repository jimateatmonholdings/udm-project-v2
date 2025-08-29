# Database Schema

This is the database schema implementation for the spec detailed in @.agent-os/specs/2025-08-28-thing-class-entity/spec.md

> Created: 2025-08-28
> Version: 1.0.0

## Schema Changes

### Core Tables Structure

Based on the UDM PRD specification, the normalized attribute system requires the following table structures within each tenant schema:

#### Thing Classes Table
```sql
CREATE TABLE thing_classes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  -- Common entity fields
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1,
  
  CONSTRAINT unique_thing_class_name UNIQUE(name),
  CONSTRAINT valid_thing_class_name CHECK (name IS NOT NULL AND trim(name) != ''),
  INDEX idx_thing_classes_active (is_active),
  INDEX idx_thing_classes_created_at (created_at),
  INDEX idx_thing_classes_name (name)
);
```

#### Attributes Table
```sql
CREATE TABLE attributes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  data_type VARCHAR(50) NOT NULL CHECK (data_type IN ('string', 'integer', 'decimal', 'boolean', 'date', 'datetime', 'json', 'reference')),
  validation_rules JSONB DEFAULT '{}',
  description TEXT,
  -- Common entity fields
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1,
  
  CONSTRAINT unique_attribute_name UNIQUE(name),
  CONSTRAINT valid_attribute_name CHECK (name IS NOT NULL AND trim(name) != ''),
  CONSTRAINT valid_validation_rules_json CHECK (validation_rules IS NULL OR jsonb_typeof(validation_rules) = 'object'),
  INDEX idx_attributes_data_type (data_type),
  INDEX idx_attributes_active (is_active),
  INDEX idx_attributes_name (name)
);
```

#### Thing Class Attributes Assignment Table
```sql
CREATE TABLE thing_class_attributes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  thing_class_id UUID NOT NULL,
  attribute_id UUID NOT NULL,
  is_required BOOLEAN DEFAULT FALSE,
  validation_rules JSONB DEFAULT '{}',
  sort_order INTEGER DEFAULT 0,
  -- Common entity fields
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1,
  
  CONSTRAINT fk_thing_class_attributes_thing_class 
    FOREIGN KEY (thing_class_id) REFERENCES thing_classes(id) ON DELETE CASCADE,
  CONSTRAINT fk_thing_class_attributes_attribute 
    FOREIGN KEY (attribute_id) REFERENCES attributes(id) ON DELETE RESTRICT,
  CONSTRAINT unique_thing_class_attribute 
    UNIQUE(thing_class_id, attribute_id),
  CONSTRAINT valid_assignment_rules_json CHECK (validation_rules IS NULL OR jsonb_typeof(validation_rules) = 'object'),
  INDEX idx_thing_class_attributes_thing_class (thing_class_id),
  INDEX idx_thing_class_attributes_attribute (attribute_id),
  INDEX idx_thing_class_attributes_active (is_active),
  INDEX idx_thing_class_attributes_required (is_required),
  INDEX idx_thing_class_attributes_sort_order (sort_order)
);
```

### Additional Supporting Tables

#### Values Table
Stores actual attribute values for Thing instances:

```sql
CREATE TABLE values (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  thing_id UUID NOT NULL,
  attribute_id UUID NOT NULL,
  value_data JSONB NOT NULL,
  -- Common entity fields
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1,
  
  CONSTRAINT fk_values_thing 
    FOREIGN KEY (thing_id) REFERENCES things(id) ON DELETE CASCADE,
  CONSTRAINT fk_values_attribute 
    FOREIGN KEY (attribute_id) REFERENCES attributes(id) ON DELETE RESTRICT,
  CONSTRAINT unique_thing_attribute_value 
    UNIQUE(thing_id, attribute_id),
  INDEX idx_values_thing (thing_id),
  INDEX idx_values_attribute (attribute_id),
  INDEX idx_values_data_gin (value_data) USING GIN,
  INDEX idx_values_active (is_active)
);
```

#### Thing Class Relationships Table
Defines allowed relationship types between Thing Classes:

```sql
CREATE TABLE thing_class_relationships (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  from_thing_class_id UUID NOT NULL,
  to_thing_class_id UUID NOT NULL,
  relationship_type_id UUID NOT NULL,
  cardinality_rules JSONB DEFAULT '{}',
  is_bidirectional BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  
  CONSTRAINT fk_class_rel_from 
    FOREIGN KEY (from_thing_class_id) REFERENCES thing_classes(id) ON DELETE CASCADE,
  CONSTRAINT fk_class_rel_to 
    FOREIGN KEY (to_thing_class_id) REFERENCES thing_classes(id) ON DELETE CASCADE,
  CONSTRAINT fk_class_rel_type 
    FOREIGN KEY (relationship_type_id) REFERENCES relationship_types(id),
  CONSTRAINT unique_class_relationship 
    UNIQUE(from_thing_class_id, to_thing_class_id, relationship_type_id),
  INDEX idx_class_rel_from (from_thing_class_id),
  INDEX idx_class_rel_to (to_thing_class_id),
  INDEX idx_class_rel_type (relationship_type_id),
  INDEX idx_class_rel_active (is_active)
);
```

### Schema-Based Multi-Tenancy Implementation

Each tenant operates within its own PostgreSQL schema, requiring the following setup for Thing Class functionality:

```sql
-- Create tenant schema
CREATE SCHEMA tenant_[company_name];

-- Set default privileges
ALTER DEFAULT PRIVILEGES IN SCHEMA tenant_[company_name] 
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO tenant_[company_name]_role;

-- Apply search path for tenant isolation
SET search_path TO tenant_[company_name], public;

-- Create all UDM tables within tenant schema (thing_classes, attributes, etc.)
```

## Migrations

### Core Schema Creation Migration

```sql
-- Migration: 001_create_core_udm_tables.sql
-- Description: Create core UDM tables with normalized attribute system

BEGIN;

-- Create thing_classes table
CREATE TABLE IF NOT EXISTS thing_classes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  description TEXT,
  -- Common entity fields
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1,
  
  CONSTRAINT unique_thing_class_name UNIQUE(name),
  CONSTRAINT valid_thing_class_name CHECK (name IS NOT NULL AND trim(name) != '')
);

-- Create attributes table
CREATE TABLE IF NOT EXISTS attributes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name VARCHAR(255) NOT NULL,
  data_type VARCHAR(50) NOT NULL CHECK (data_type IN ('string', 'integer', 'decimal', 'boolean', 'date', 'datetime', 'json', 'reference')),
  validation_rules JSONB DEFAULT '{}',
  description TEXT,
  -- Common entity fields
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1,
  
  CONSTRAINT unique_attribute_name UNIQUE(name),
  CONSTRAINT valid_attribute_name CHECK (name IS NOT NULL AND trim(name) != ''),
  CONSTRAINT valid_validation_rules_json CHECK (validation_rules IS NULL OR jsonb_typeof(validation_rules) = 'object')
);

-- Create thing_class_attributes assignment table
CREATE TABLE IF NOT EXISTS thing_class_attributes (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  thing_class_id UUID NOT NULL,
  attribute_id UUID NOT NULL,
  is_required BOOLEAN DEFAULT FALSE,
  validation_rules JSONB DEFAULT '{}',
  sort_order INTEGER DEFAULT 0,
  -- Common entity fields
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE,
  version INTEGER DEFAULT 1,
  
  CONSTRAINT fk_thing_class_attributes_thing_class 
    FOREIGN KEY (thing_class_id) REFERENCES thing_classes(id) ON DELETE CASCADE,
  CONSTRAINT fk_thing_class_attributes_attribute 
    FOREIGN KEY (attribute_id) REFERENCES attributes(id) ON DELETE RESTRICT,
  CONSTRAINT unique_thing_class_attribute 
    UNIQUE(thing_class_id, attribute_id),
  CONSTRAINT valid_assignment_rules_json CHECK (validation_rules IS NULL OR jsonb_typeof(validation_rules) = 'object')
);

-- Add indexes for performance
CREATE INDEX IF NOT EXISTS idx_thing_classes_active ON thing_classes(is_active);
CREATE INDEX IF NOT EXISTS idx_thing_classes_created_at ON thing_classes(created_at);
CREATE INDEX IF NOT EXISTS idx_thing_classes_name ON thing_classes(name);

CREATE INDEX IF NOT EXISTS idx_attributes_data_type ON attributes(data_type);
CREATE INDEX IF NOT EXISTS idx_attributes_active ON attributes(is_active);
CREATE INDEX IF NOT EXISTS idx_attributes_name ON attributes(name);

CREATE INDEX IF NOT EXISTS idx_thing_class_attributes_thing_class ON thing_class_attributes(thing_class_id);
CREATE INDEX IF NOT EXISTS idx_thing_class_attributes_attribute ON thing_class_attributes(attribute_id);
CREATE INDEX IF NOT EXISTS idx_thing_class_attributes_active ON thing_class_attributes(is_active);
CREATE INDEX IF NOT EXISTS idx_thing_class_attributes_required ON thing_class_attributes(is_required);
CREATE INDEX IF NOT EXISTS idx_thing_class_attributes_sort_order ON thing_class_attributes(sort_order);

-- Add updated_at trigger for all tables
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_thing_classes_updated_at
  BEFORE UPDATE ON thing_classes
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_attributes_updated_at
  BEFORE UPDATE ON attributes
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_thing_class_attributes_updated_at
  BEFORE UPDATE ON thing_class_attributes
  FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

COMMIT;
```

### Attribute Assignment Migration

```sql
-- Migration: 002_create_attribute_assignments_table.sql
-- Description: Create attribute_assignments table for Thing Class to Attribute relationships

BEGIN;

CREATE TABLE IF NOT EXISTS attribute_assignments (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  thing_class_id UUID NOT NULL,
  attribute_id UUID NOT NULL,
  is_required BOOLEAN DEFAULT FALSE,
  assignment_rules JSONB DEFAULT '{}',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  created_by UUID NOT NULL,
  is_active BOOLEAN DEFAULT TRUE
);

-- Add foreign key constraints
ALTER TABLE attribute_assignments 
  ADD CONSTRAINT fk_assignment_thing_class 
  FOREIGN KEY (thing_class_id) REFERENCES thing_classes(id) ON DELETE CASCADE;

ALTER TABLE attribute_assignments 
  ADD CONSTRAINT fk_assignment_attribute 
  FOREIGN KEY (attribute_id) REFERENCES attributes(id);

-- Add unique constraint and indexes
ALTER TABLE attribute_assignments 
  ADD CONSTRAINT unique_thing_class_attribute 
  UNIQUE(thing_class_id, attribute_id);

CREATE INDEX IF NOT EXISTS idx_assignments_thing_class ON attribute_assignments(thing_class_id);
CREATE INDEX IF NOT EXISTS idx_assignments_attribute ON attribute_assignments(attribute_id);
CREATE INDEX IF NOT EXISTS idx_assignments_active ON attribute_assignments(is_active);
CREATE INDEX IF NOT EXISTS idx_assignments_required ON attribute_assignments(is_required);

-- Add JSONB index for assignment_rules
CREATE INDEX IF NOT EXISTS idx_assignments_rules_gin 
  ON attribute_assignments USING GIN(assignment_rules);

COMMIT;
```

## Indexes and Constraints

### Performance Indexes

```sql
-- Primary performance indexes for Thing Class queries
CREATE INDEX CONCURRENTLY idx_thing_classes_name_active ON thing_classes(name, is_active);
CREATE INDEX CONCURRENTLY idx_thing_classes_created_by_date ON thing_classes(created_by, created_at);

-- Composite indexes for common query patterns
CREATE INDEX CONCURRENTLY idx_thing_classes_active_created ON thing_classes(is_active, created_at DESC);

-- GIN indexes for JSONB fields to support GraphQL field selection
CREATE INDEX CONCURRENTLY idx_thing_classes_validation_gin ON thing_classes USING GIN(validation_rules);
CREATE INDEX CONCURRENTLY idx_thing_classes_permissions_gin ON thing_classes USING GIN(permissions);

-- Attribute assignment performance indexes
CREATE INDEX CONCURRENTLY idx_assignments_class_active ON attribute_assignments(thing_class_id, is_active);
CREATE INDEX CONCURRENTLY idx_assignments_attr_required ON attribute_assignments(attribute_id, is_required);
```

### Data Integrity Constraints

```sql
-- Validation rules constraint to ensure proper JSON structure
ALTER TABLE thing_classes ADD CONSTRAINT valid_validation_rules_json
  CHECK (validation_rules IS NULL OR jsonb_typeof(validation_rules) = 'object');

ALTER TABLE thing_classes ADD CONSTRAINT valid_permissions_json
  CHECK (permissions IS NULL OR jsonb_typeof(permissions) = 'object');

-- Name constraint to prevent empty or whitespace-only names
ALTER TABLE thing_classes ADD CONSTRAINT valid_thing_class_name
  CHECK (name IS NOT NULL AND trim(name) != '');

-- Assignment rules validation
ALTER TABLE attribute_assignments ADD CONSTRAINT valid_assignment_rules_json
  CHECK (assignment_rules IS NULL OR jsonb_typeof(assignment_rules) = 'object');
```

### Tenant Isolation Constraints

```sql
-- Row Level Security policies for additional tenant protection
ALTER TABLE thing_classes ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_thing_classes ON thing_classes
  USING (current_schema() = current_setting('search_path')::text);

ALTER TABLE attribute_assignments ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_assignments ON attribute_assignments
  USING (current_schema() = current_setting('search_path')::text);
```

## Performance Considerations for Multi-Tenant Architecture

### Connection Pool Optimization

- **Schema-aware connection pooling**: Use PgBouncer with tenant-specific pools
- **Connection pool sizing**: 10-20 connections per tenant schema for typical workloads
- **Search path caching**: Cache search_path settings to avoid repeated SET commands

### Query Performance Optimization

```sql
-- Materialized view for frequently accessed Thing Class metadata
CREATE MATERIALIZED VIEW thing_class_summary AS
SELECT 
  tc.id,
  tc.name,
  tc.description,
  tc.is_active,
  COUNT(aa.id) as attribute_count,
  COUNT(DISTINCT t.id) as thing_count
FROM thing_classes tc
LEFT JOIN attribute_assignments aa ON tc.id = aa.thing_class_id AND aa.is_active = true
LEFT JOIN things t ON tc.id = t.thing_class_id AND t.is_deleted = false
WHERE tc.is_active = true
GROUP BY tc.id, tc.name, tc.description, tc.is_active;

-- Refresh strategy for materialized view
CREATE INDEX ON thing_class_summary(name);
CREATE INDEX ON thing_class_summary(attribute_count);
```

### GraphQL Query Optimization

- **Field selection indexes**: GIN indexes on JSONB fields support GraphQL field selection optimization
- **Relationship traversal**: Foreign key indexes ensure efficient GraphQL relationship queries
- **Complexity analysis**: Query depth limited to 5 levels to prevent expensive operations

### Scaling Characteristics

- **Per-schema storage**: Each tenant schema scales independently up to ~1M Thing Classes
- **Cross-tenant queries**: Prohibited by design - prevents performance interference
- **Backup/restore**: Schema-level operations enable tenant-specific data management
- **Index maintenance**: Per-schema index management prevents cross-tenant impact

### Monitoring and Maintenance

```sql
-- Thing Class usage monitoring queries
CREATE VIEW thing_class_metrics AS
SELECT 
  tc.name,
  tc.created_at,
  COUNT(t.id) as instance_count,
  COUNT(aa.id) as attribute_count,
  MAX(t.updated_at) as last_instance_update
FROM thing_classes tc
LEFT JOIN things t ON tc.id = t.thing_class_id AND t.is_deleted = false
LEFT JOIN attribute_assignments aa ON tc.id = aa.thing_class_id AND aa.is_active = true
WHERE tc.is_active = true
GROUP BY tc.id, tc.name, tc.created_at;

-- Schema size monitoring
CREATE VIEW schema_storage_metrics AS
SELECT 
  schemaname,
  tablename,
  pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as size,
  pg_stat_get_tuples_returned(c.oid) as rows_read,
  pg_stat_get_tuples_inserted(c.oid) as rows_inserted
FROM pg_tables pt
JOIN pg_class c ON c.relname = pt.tablename
WHERE schemaname LIKE 'tenant_%'
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC;
```