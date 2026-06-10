# Yak — Product

## What It Is
A chat-as-a-service platform that enables any application to add real-time messaging through simple API integration.

## Problem
Building real-time chat is complex — WebSocket management, message persistence, presence tracking, file sharing, push notifications, and webhook delivery all require significant engineering effort. Yak handles this so application developers don't have to.

## Users
- **Client developers** — integrate chat into their products via REST API and webhooks
- **End users** — send/receive messages in real-time through client applications

## Core Capabilities
- Multi-tenant chat backend (isolated per client)
- 1-on-1 conversations with text and file messages
- Group chat (up to 100 participants, admin management, soft delete)
- Real-time message delivery via WebSocket/STOMP
- Typing indicators (real-time via WebSocket)
- Read receipts and delivery status (SENT → DELIVERED → READ)
- Message reactions (emoji reactions delivered as messages, compatible with deleteAfterDelivery)
- Message deletion (delete for me / delete for everyone with soft-delete and reconnect sync)
- User blocking (block/unblock per conversation; stops messages, calls, presence, group invites)
- Conversation reporting (user submits description + last 10 messages; stored in MongoDB for admin review)
- Webhook-based event delivery for async integrations
- Message acknowledgement & server deletion (opt-in per client; messages deleted from server once all recipients acknowledge; foundation for E2EE)
- File deletion on acknowledgement (when deleteAfterDelivery is enabled, associated files and thumbnails are deleted from yak-filemanager when the message is fully acknowledged)
- File upload/download with access control
- Push notification support (via webhook → FCM)
- JWT-based authentication (client and user level)
- Child accounts (QR-based login, QR-based contact adding, single-device, contacts-only messaging)

## Client Integration Model
1. Client registers and gets credentials (username + secret)
2. Client creates users and conversations via server-to-server API
3. End users connect via WebSocket for real-time messaging
4. Client receives events at webhook URL for offline processing
5. Users create groups, add members, and send group messages

## Planned Capabilities
- Temporary/anonymous users (help desk, chat rooms)
- Per-client statistics and monitoring
- SDK / client libraries (JavaScript, iOS, Android)
- End-to-end encryption (key exchange API complete)
