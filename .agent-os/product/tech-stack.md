# Technical Stack

> Last Updated: 2025-08-28
> Version: 1.0.0

## Application Framework

- **Framework:** Go + GraphQL with gqlgen library
- **Version:** Go 1.21+ with gqlgen v0.17+
- **Architecture:** GraphQL-first architecture with Apollo Server 4+

## Database

- **Primary Database:** PostgreSQL 15+ with schema-based multi-tenancy
- **Hosting:** PostgreSQL with streaming replication
- **Isolation:** Database-per-tenant isolation
- **Caching:** Redis Cluster for distributed caching

## JavaScript

- **Framework:** React with TypeScript for type safety
- **Runtime:** Node.js 18+
- **Import Strategy:** Node (ES modules)

## CSS Framework

- **Framework:** Tailwind CSS (utility-first)
- **UI Components:** shadcn/ui (accessible, customizable React components)

## Authentication & Security

- **Authentication:** JWT-based authentication
- **Authorization:** Role-based access control (RBAC)

## Message Queue & Communication

- **Message Queue:** RabbitMQ for asynchronous processing
- **API Layer:** GraphQL with real-time subscriptions

## Hosting & Deployment

- **Application Hosting:** Cloud-native architecture with Kubernetes
- **Database Hosting:** PostgreSQL with streaming replication
- **Asset Hosting:** CDN for static asset delivery
- **Deployment:** Docker containerization with CI/CD pipeline

## Development & Operations

- **Code Repository:** https://github.com/jimateatmonholdings/udm-project-v2.git
- **Containerization:** Docker with multi-stage builds
- **Orchestration:** Kubernetes for container orchestration
- **Monitoring:** Application and infrastructure monitoring stack

## Additional Components

- **Caching Layer:** Redis Cluster for distributed caching
- **File Storage:** Cloud object storage for document management
- **Search:** Full-text search capabilities
- **Backup & Recovery:** Automated backup strategies with point-in-time recovery