# Yak - System Design Overview

## Architecture

Yak is a **chat-as-a-service platform** built with Spring Boot 4 microservices (Java 25, Spring Framework 7). It provides real-time messaging capabilities that third-party applications can integrate via REST APIs and webhooks.

## Services

### Platform Services (Yak Core)

| Service | Port (local) | Port (docker) | Purpose |
|---------|-------------|---------------|---------|
| **yak-gateway** | 8000 | 8080 | Spring Cloud Gateway — single entry point, routes to backend services |
| **yak-api** | 8081 | 8080 | Core chat API — users, conversations, messages, groups, reports, webhook event publishing |
| **yak-websocket** | 8088 | 8080 | WebSocket/STOMP server — real-time message delivery via RabbitMQ relay |
| **yak-filemanager** | 8082 | 8080 | File storage service — upload/download with AWS S3 or local filesystem |
| **yak-webhook-notifier** | 8083 | 8080 | Async event consumer — reads from Redis, delivers webhooks to clients |

### Client Applications (consumers of Yak platform)

| Service | Port (local) | Port (docker) | Purpose |
|---------|-------------|---------------|---------|
| **yak-messenger-api** | 9090 | 9010 | Mobile messenger backend — Firebase push, receives webhooks from Yak |
| **yak-demo** | 8080 | 9000 | Demo web app — Thymeleaf UI showcasing Yak integration |

> **Note:** `yak-messenger-api` and `yak-demo` are independent client applications. They do NOT communicate with each other. Both integrate with the Yak platform through the gateway.

## Infrastructure

| Component | Purpose | Used By |
|-----------|---------|---------|
| **MongoDB** | Primary data store | All services (shared `reltalk` DB, messenger uses `yak_messenger`) |
| **Redis** | Event queue + session/cache | yak-api, yak-websocket, yak-webhook-notifier |
| **RabbitMQ** | STOMP message broker relay | yak-websocket |
| **Nginx** (certbot) | TLS termination, reverse proxy | External traffic → gateway |

## Inter-Service Communication

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            EXTERNAL CLIENTS                                  │
│           (Web browsers, Mobile apps, Third-party integrations)              │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │ HTTPS (443)
                                   ▼
                      ┌────────────────────────┐
                      │   Nginx (TLS + Proxy)  │
                      └──┬──────────┬──────┬───┘
                         │          │      │
  chat-api.reltalk.com   │          │      │  yak-messenger.reltalk.com
                         │          │      │
                         ▼          │      ▼
         ┌──────────────────┐       │    ┌────────────────────┐
         │  yak-gateway     │       │    │  yak-messenger-api │──────────▶ MongoDB
         │  (:8000)         │◀──────┼────┤  (:9090)           │           (yak_messenger)
         └────────┬─────────┘       │    └────────────────────┘
                  │          ▲      │       Feign (via gateway)
                  │          │      │
                  │          │      │  demo.reltalk.com
                  │          │      │
                  │          │      ▼
                  │          │    ┌─────────────────┐
                  │          └────┤    yak-demo     │
                  │               │  (:8080)        │
                  │               └─────────────────┘
                  │                 Feign + WebSocket (via gateway)
                  │
                  │  Gateway Routes:
                  │    /chat-api/**    → yak-api
                  │    /filemanager/** → yak-filemanager
                  │    /yak-ws/**      → yak-websocket
                  │
         ┌────────┼────────────────────┐
         │        │                    │
         ▼        ▼                    ▼
  ┌────────────┐ ┌──────────────┐ ┌─────────────┐
  │  yak-api   │ │yak-filemanager│ │yak-websocket│
  │  (:8081)   │ │  (:8082)     │ │  (:8088)    │
  └──┬───┬──┬──┘ └──┬───────┬───┘ └──┬───┬──┬──┘
     │   │  │       ▲       │        ▲   │  │
     │   │  │       │       │        │   │  │
     │   │  │ Feign │       │  Feign │   │  │
     │   │  └───────┘       │        │   │  │
     │   │                   │        │   │  │
     │   └───────────────────┼────────┘   │  │
     │                       │            │  │
     ▼                       ▼            ▼  ▼
  MongoDB                  S3/FS       RabbitMQ
     ▲                                    (STOMP relay)
     │
     ├─── yak-filemanager
     ├─── yak-websocket
     │
     ▼
   Redis
     ▲
     ├─── yak-api
     ├─── yak-websocket
     │
     │  RPUSH (webhook:events:new)
     ▼
  ┌─────────────────────┐
  │yak-webhook-notifier │──── HTTP POST (X-Hub-Signature) ──▶ Client Webhook URLs
  │  (:8083)            │                                     (e.g. yak-messenger-api)
  └─────────────────────┘
     │
     ▼
  MongoDB, Redis

  Infrastructure usage summary:
  ─────────────────────────────────────────────────────────
  yak-api              → MongoDB, Redis
  yak-filemanager      → MongoDB, AWS S3 / Local FS
  yak-websocket        → MongoDB, Redis, RabbitMQ
  yak-webhook-notifier → MongoDB, Redis
```

### Simplified Communication Map

```
yak-demo ──────────────── Feign + WebSocket ──────▶ yak-gateway ──▶ yak-api / yak-websocket
yak-messenger-api ─────── Feign ──────────────────▶ yak-gateway ──▶ yak-api

yak-gateway ───────────── routes /chat-api/** ────▶ yak-api
yak-gateway ───────────── routes /filemanager/** ─▶ yak-filemanager
yak-gateway ───────────── routes /yak-ws/** ──────▶ yak-websocket

yak-api ───────────────── Feign (direct) ─────────▶ yak-websocket
yak-api ───────────────── Feign (direct) ─────────▶ yak-filemanager
yak-api ───────────────── Redis RPUSH ────────────▶ yak-webhook-notifier

yak-webhook-notifier ──── HTTP POST (signed) ─────▶ yak-messenger-api (/webhook)

yak-websocket ─────────── STOMP relay ────────────▶ RabbitMQ
yak-websocket ─────────── Feign ──────────────────▶ RabbitMQ Management API
```

## Communication Flows

### 1. Gateway Routing (HTTP)

The gateway exposes all backend services under a single domain (`chat-api.reltalk.com`):

| Route Pattern | Target Service | Rewrite |
|---------------|---------------|---------|
| `/chat-api/**` | yak-api | Strip `/chat-api` prefix |
| `/filemanager/api/**` | yak-filemanager | Rewrite to `/api/**` |
| `/yak-ws/info/**` | yak-websocket (HTTP) | SockJS info endpoint |
| `/yak-ws/**` | yak-websocket (WS) | WebSocket upgrade |

### 2. yak-api → yak-websocket (Feign HTTP)

When a message is sent, `yak-api` calls `yak-websocket` to:
- Set up user messaging exchanges (`/messaging-backend/v1/users/setup-exchanges`)
- Subscribe to presence updates (`/messaging-backend/v1/users/subscribe-presence`)
- Deliver messages in real-time (`/messaging-backend/v1/users/{id}/send-message`)

For group chat, `yak-api` also manages group exchange topology:
- Create group exchange (`/messaging-backend/v1/groups/setup-exchange`)
- Bind/unbind user exchanges to group (`/messaging-backend/v1/groups/{id}/subscribe`, `/unsubscribe`)
- Publish group messages (`/messaging-backend/v1/groups/{id}/send-message`)
- Delete group exchange on soft-delete (`DELETE /messaging-backend/v1/groups/{id}/exchange`)

### 3. yak-api → yak-filemanager (Feign HTTP)

`yak-api` calls `yak-filemanager` to manage file access permissions when files are shared in conversations.

### 4. yak-api → Redis → yak-webhook-notifier (Async Queue)

Event flow for webhook delivery:
1. `yak-api` publishes `WebhookEvent` to Redis list `webhook:events:new`
2. `yak-webhook-notifier` polls this list in a background thread
3. Events are distributed to per-client queues (`webhook:events:client:{clientId}`)
4. Worker threads deliver events via HTTP POST to each client's configured webhook URL
5. Requests are signed with `X-Hub-Signature` (HMAC-SHA1) for verification

### 5. yak-webhook-notifier → yak-messenger-api (HTTP Webhook)

The webhook-notifier delivers message events to `yak-messenger-api`'s `/webhook` endpoint. The messenger-api:
- Verifies the `X-Hub-Signature` header
- Sends Firebase Cloud Messaging (FCM) push notifications to mobile devices

### 6. yak-messenger-api → yak-gateway → yak-api (Feign HTTP)

The messenger app backend calls `yak-api` through the gateway to:
- Create/manage users (`/chat-api/client-api/v1/users`)
- Create/manage conversations (`/chat-api/client-api/v1/conversations`)

### 7. yak-demo → yak-gateway → yak-api (Feign HTTP + WebSocket)

The demo app calls `yak-api` through the gateway to:
- Manage users and conversations (client API)
- Send messages and list conversations (user API)
- Connect to `yak-websocket` via STOMP/WebSocket (`/yak-ws/websocket` through gateway)

### 8. yak-websocket ↔ RabbitMQ (STOMP Relay)

WebSocket uses RabbitMQ as a STOMP broker relay:
- Clients subscribe to `/exchange/...` destinations
- Messages are routed through RabbitMQ exchanges per user
- `yak-websocket` manages exchange/binding setup via RabbitMQ Management API (Feign)

Exchange topology:
- `user-{userId}-incoming` (fanout) — all events destined for a user
- `user-{userId}-presence` (fanout) — broadcasts user's presence changes
- `group-{groupId}` (fanout) — group messages fan out to all bound user incoming exchanges

Group message flow: yak-api → publish to `group-{id}` exchange → RabbitMQ fans out to all bound `user-{x}-incoming` exchanges → delivered to each user's STOMP session

## Security

- **JWT Authentication**: All services validate JWT tokens for user/client identity
- **Two API layers in yak-api**:
  - `client-api` — for server-to-server (client credentials with shared secret)
  - `user-api` — for end-user operations (user JWT)
- **Webhook signing**: HMAC-SHA1 signature in `X-Hub-Signature` header
- **OAuth2**: Services support OAuth2 client authentication
- **TLS**: Nginx handles SSL termination with Let's Encrypt certificates

## Deployment

- **Docker Compose** for local development and production (single-host or Swarm)
- Each service has its own `docker/Dockerfile`
- Infrastructure services (MongoDB, Redis, RabbitMQ) run in `docker-compose-infra.yml` for local dev
- Full stack (infra + services + nginx) in `docker-compose.yml`
- Domain: `*.reltalk.com` with subdomains for each public service
