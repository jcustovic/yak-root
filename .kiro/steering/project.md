# Yak Platform

## Overview
Yak is a chat-as-a-service platform built with Spring Boot microservices (Java 25). It provides real-time messaging capabilities that third-party applications integrate via REST APIs and webhooks.

## Architecture
- Microservices communicating via HTTP (Feign), Redis queues, and WebSocket/STOMP
- All external traffic enters through yak-gateway
- Client applications (yak-demo, yak-messenger-api) always communicate through the gateway
- Internal services (yak-api → yak-websocket, yak-api → yak-filemanager) communicate directly

## Modules

### Platform Services
- **yak-gateway** (:8000) — Spring Cloud Gateway, routes to backend services
- **yak-api** (:8081) — Core chat API: users, conversations, messages, groups, webhook publishing
- **yak-websocket** (:8088) — WebSocket/STOMP server with RabbitMQ relay
- **yak-filemanager** (:8082) — File upload/download/deletion with S3 or local storage
- **yak-webhook-notifier** (:8083) — Async webhook delivery from Redis queue

### Client Applications
- **yak-messenger-api** (:9090) — Mobile messenger backend with Firebase push
- **yak-demo** (:8080) — Demo web app showcasing Yak integration

### Infrastructure
- MongoDB (shared `reltalk` DB), Redis, RabbitMQ, Nginx (TLS), AWS S3

## Conventions
- Java 25, Spring Boot 4.0.6, Spring Framework 7
- Jackson 3 (tools.jackson) — all `@Builder` DTOs need `@NoArgsConstructor` + `@AllArgsConstructor`
- Lombok for boilerplate reduction
- JWT authentication (client JWT for server-to-server, user JWT for end-users)
- Feign clients for inter-service HTTP calls
- Each service has its own git repository and Docker image
- Config via environment variables with sensible local defaults in application-local.yml
- MongoDB connection under `spring.mongodb.*` (not `spring.data.mongodb.*`)
- SpringDoc OpenAPI 3.x for Swagger UI
