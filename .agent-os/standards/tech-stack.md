# Tech Stack

## Context

### Core Backend Stack

**Runtime & Language:**
- **Go 1.21+** - Primary backend language for performance and operational simplicity
- **Single binary deployment** with zero external runtime dependencies
- **Goroutine-based concurrency** for efficient multi-tenant request handling

**GraphQL Framework:**
- **gqlgen** - Go-native GraphQL implementation with code generation
- **Type-safe resolvers** generated from GraphQL schema definitions
- **Built-in query complexity analysis** and performance optimization
- **Schema-first development** with strong typing throughout

**Database Architecture:**
- **PostgreSQL 15+** as primary data store
- **Database-per-tenant isolation** for compliance and security
- **JSONB support** for flexible UDM attribute storage
- **Connection pooling** with tenant-aware routing

**Supporting Infrastructure:**
- **Redis Cluster** - GraphQL query caching and schema metadata
- **RabbitMQ** - Event-driven updates and async processing
- **JWT Authentication** - Stateless authentication with tenant routing
- **Docker containers** - Containerized deployment and scaling

### Core Frontend Stack

- **React**
- **tailwind**
- **shadcn**
- **Apollo GraphQL Client**
