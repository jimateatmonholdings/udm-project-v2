# Value Entity - Lite Summary

The Value entity is the data storage layer in UDM that holds actual property values for Thing-Attribute combinations, supporting all 8 data types with validation, versioning, and historical tracking.

## Key Points
- Stores actual property values for Thing instances with full data type validation (string, integer, decimal, boolean, date, datetime, json, reference)
- Implements efficient versioning and historical value tracking with <200ms query performance targets
- Provides comprehensive GraphQL API for CRUD operations with advanced filtering and reference resolution capabilities