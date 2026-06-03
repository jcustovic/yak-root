# Yak — Structure

## Multi-Root Workspace

This workspace contains 8 independent modules, each with its own git repository.

```
Yak/
├── .kiro/steering/          # Workspace-level steering docs
├── SYSTEM_DESIGN.md         # Architecture and communication diagrams
│
├── yak-gateway/             # API Gateway (Spring Cloud Gateway)
├── yak-api/                 # Core Chat API
├── yak-websocket/           # WebSocket/STOMP Server
├── yak-filemanager/         # File Storage Service
├── yak-webhook-notifier/    # Async Webhook Delivery
│
├── yak-messenger-api/       # Client App: Mobile Messenger Backend
├── yak-demo/                # Client App: Demo Web UI
│
└── yak-docker/              # Docker Infrastructure & Deployment
```

## Module Categories

### Platform Services (Yak Core)
Services that make up the chat platform. They communicate internally.

| Module | Port (local) | Responsibility |
|--------|-------------|----------------|
| yak-gateway | 8000 | Routes external traffic to backend services |
| yak-api | 8081 | Users, conversations, messages, groups, reports, webhook publishing |
| yak-websocket | 8088 | Real-time delivery via STOMP/RabbitMQ |
| yak-filemanager | 8082 | File upload/download, access control |
| yak-webhook-notifier | 8083 | Consumes Redis stream, delivers webhooks |

### Client Applications
Independent apps that consume the Yak platform through the gateway. They do NOT communicate with each other.

| Module | Port (local) | Responsibility |
|--------|-------------|----------------|
| yak-messenger-api | 9090 | Mobile app backend (Firebase, push notifications) |
| yak-demo | 8080 | Browser-based demo chat UI (Thymeleaf) |

### Infrastructure
| Module | Responsibility |
|--------|----------------|
| yak-docker | Docker Compose, Nginx, deployment configs |

## Communication Rules
1. **Client apps → always through gateway** (yak-demo and yak-messenger-api never call backend services directly)
2. **Platform services → direct Feign calls** (yak-api calls yak-websocket and yak-filemanager directly)
3. **Async events → Redis stream** (yak-api publishes, yak-webhook-notifier consumes via consumer group)
4. **Real-time → RabbitMQ STOMP relay** (yak-websocket uses RabbitMQ for message routing)
5. **Resource access isolation** — all queries and mutations must be scoped to the authenticated `clientId` (client-api) or `userId` (user-api). A client must never access another client's resources; a user must never access another user's resources.

## Key Spring Boot 4 Conventions
- MongoDB connection: `spring.mongodb.*` (not `spring.data.mongodb.*`)
- Jackson 3: all `@Builder` DTOs must have `@NoArgsConstructor` + `@AllArgsConstructor`
- Test annotations: `@MockitoBean` (not `@MockBean`), `@WebMvcTest` from `org.springframework.boot.webmvc.test.autoconfigure`
- SpringDoc OpenAPI 3.x for Swagger UI
- Spring Security 7: `@EnableWebSocketSecurity` replaces `AbstractSecurityWebSocketMessageBrokerConfigurer`
- QueryDSL: use OpenFeign fork (`io.github.openfeign.querydsl` 7.x), not the original `com.querydsl` (unmaintained). Requires `mysema-commons-lang` bridge dependency.

## Module Internal Structure (typical)
```
<module>/
├── .kiro/steering/
│   ├── product.md           # What it does, users, features
│   └── tech.md              # Stack, architecture, config, build
├── src/main/java/com/reltalk/
│   ├── config/              # Spring configuration classes
│   ├── web/                 # Controllers (REST / WebSocket)
│   ├── domain/              # Business logic, models, services, repository interfaces
│   └── infrastructure/      # Repository implementations, Feign clients, security, MongoDB documents
├── src/main/resources/
│   ├── application.yml      # Default config
│   └── application-local.yml # Local dev overrides
├── docker/Dockerfile
├── pom.xml
└── Readme.md
```

Layered call flow (all features):
```
Controller (web) → Service (domain) → Repository interface (domain) → RepositoryImpl (infrastructure) → MongoRepository (Spring Data)
```

**Boundary rule:** MongoDB document classes (`*Document`) are only used within the `infrastructure/` layer. Repository implementations map between documents and domain models. Services and controllers only work with domain models — never with persistence-layer objects.
