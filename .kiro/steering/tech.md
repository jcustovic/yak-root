# Yak — Tech

## Stack
- Java 25, Spring Boot 4.0.6, Spring Framework 7
- Spring Cloud 2025.1 (Oakwood)
- Spring WebSocket + STOMP
- Spring Data MongoDB
- Spring Data Redis
- Spring Security 7 + OAuth2 (JWT)
- Jackson 3 (tools.jackson)
- RabbitMQ (STOMP broker relay)
- Docker / Docker Swarm
- Nginx + Let's Encrypt (TLS)
- AWS S3 (optional file storage)

## Conventions
- Lombok for boilerplate reduction (all `@Builder` classes require `@NoArgsConstructor` + `@AllArgsConstructor` for Jackson 3)
- SpringDoc OpenAPI 3.x for API documentation
- Feign clients for inter-service HTTP calls
- JWT authentication (client JWT for server-to-server, user JWT for end-users)
- Environment variables for configuration with local defaults in `application-local.yml`
- MongoDB connection properties under `spring.mongodb.*` (Boot 4 change from `spring.data.mongodb.*`)
- Each service has its own git repository and Docker image
- Domain-driven design: `domain/` (business logic), `infrastructure/` (adapters), `web/` (controllers)

## Infrastructure
| Component | Purpose |
|-----------|---------|
| MongoDB | Primary data store (shared `reltalk` DB) |
| Redis | Event queue, session/presence state |
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
