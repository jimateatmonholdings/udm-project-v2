# Relationship Class Entity - Lite Summary

The Relationship Class entity is the template/schema definition layer for relationship types between Things in UDM, defining structural blueprints for typed connections with directionality, cardinality constraints, and Thing Class specifications.

## Key Points
- Defines relationship type templates (contains, assigned_to, reports_to) with comprehensive constraint management and validation rules
- Implements cardinality enforcement (one-to-one, one-to-many, many-to-many) with Thing Class source/target specifications and directional support
- Provides comprehensive GraphQL API for relationship schema CRUD operations with <200ms query performance targets and multi-tenant isolation