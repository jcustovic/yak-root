# Yak — Tech

## Stack
- Java 25, Spring Boot 4.0.6, Spring Framework 7
- Spring Cloud 2025.1.1 (Oakwood)
- Spring WebSocket + STOMP
- Spring Data MongoDB
- Spring Data Redis
- Spring Security 7 + OAuth2 (JWT)
- Jackson 3 (tools.jackson)
- QueryDSL (OpenFeign fork `io.github.openfeign.querydsl` 7.x — requires `mysema-commons-lang` bridge for Spring Data MongoDB compatibility)
- RabbitMQ (STOMP broker relay)
- Docker / Docker Swarm
- Nginx + Let's Encrypt (TLS)
- AWS S3 (optional file storage)

## Conventions
- Lombok 1.18.46 (all `@Builder` classes require `@NoArgsConstructor` + `@AllArgsConstructor` for Jackson 3)
- SpringDoc OpenAPI 3.x for API documentation
- Feign clients for inter-service HTTP calls
- JWT authentication (client JWT for server-to-server, user JWT for end-users)
- Environment variables for configuration with local defaults in `application-local.yml`
- MongoDB connection properties under `spring.mongodb.*` (Boot 4 change from `spring.data.mongodb.*`)
- Each service has its own git repository and Docker image
- Domain-driven design: `domain/` (business logic), `infrastructure/` (adapters), `web/` (controllers)
- Layered architecture: Controller → Service (domain) → Repository interface (domain) → RepositoryImpl (infrastructure) → MongoRepository (Spring Data)
- MongoDB documents (`*Document`) never leak above the infrastructure layer; domain models are used in services and controllers
- Resource access isolation: all queries and mutations must be scoped to the authenticated `clientId` (for client-api) or `userId` (for user-api). Never allow access to a resource without verifying ownership.
- Client configuration (deleteAfterDelivery, webhookEnabled) is cached in-memory for 5 minutes via `ClientRepositoryImpl`
- Batch operations preferred over N+1 patterns (batch reply-to fetch, batch user lookup, Redis MGET/pipeline for unread counts and presence)
- Bean Validation (`@Valid`, `@Size`) used on controller DTOs for input limits (e.g., max 100 IDs per batch request)

## Infrastructure
| Component | Purpose |
|-----------|---------|
| MongoDB | Primary data store (shared `reltalk` DB) |
| Redis | Event stream, session/presence state, unread counts |
| RabbitMQ | STOMP message broker relay |
| Nginx | TLS termination, reverse proxy |
| AWS S3 / Local FS | File storage |

## Deployment
- Docker Compose for local dev and production
- Single Docker host (current), Docker Swarm ready
- TLS via Let's Encrypt (auto-renewal with nginx-certbot)
- Domain: `*.reltalk.com`

## Build
All services use Maven:
```bash
mvn spring-boot:run -Dspring-boot.run.profiles=local
DOCKER_BUILDKIT=1 docker build -f docker/Dockerfile -t yak/<service> --no-cache .
```
