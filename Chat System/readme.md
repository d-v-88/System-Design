# Chat Application

A scalable, real-time chat application architecture designed to support **Web and Mobile clients**, bi-directional real-time communication, media sharing, group conversations, online presence, message persistence, and asynchronous push notifications.

The architecture separates real-time connection management from business logic and supporting services, allowing each component to scale independently.

> **Architecture:** Real-Time • Event-Driven • Horizontally Scalable • Fault Tolerant • Multi-Service

---

## Architecture

![Chat Application Architecture](./architecture.png)

### High-Level Flow

```text
                         ┌──────────────────────┐
                         │       Clients        │
                         │ Web / Mobile /       │
                         │ Desktop              │
                         └──────────┬───────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
             ┌─────────────┐                 ┌─────────────┐
             │ API Gateway │                 │ CDN for     │
             │             │                 │ Media       │
             └──────┬──────┘                 └─────────────┘
                    │
          ┌─────────┴──────────┐
          │                    │
          ▼                    ▼
┌──────────────────┐   ┌──────────────────┐
│ Connection       │   │ User Service     │
│ Manager          │   │                  │
│                  │   │ Auth             │
│ Redis            │   │ Profiles         │
└────────┬─────────┘   └──────────────────┘
         │
         ▼
┌──────────────────┐
│ WebSocket        │
│ Servers          │
│                  │
│ Real-time        │
│ Communication    │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Chat Service     │
│                  │
│ Messages         │
│ Delivery State   │
│ Persistence      │
└───────┬──────────┘
        │
   ┌────┴───────────────┐
   │                    │
   ▼                    ▼
┌─────────────┐   ┌─────────────────┐
│ Group       │   │ Presence        │
│ Service     │   │ Service         │
│             │   │                 │
│ Groups      │   │ Online/Offline  │
│ Membership  │   │ Typing          │
└─────────────┘   └─────────────────┘

        Chat Service
              │
              ▼
     ┌─────────────────┐
     │ Notification    │
     │ Queue           │
     │ (Async Events)  │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ Notification    │
     │ Service         │
     │                 │
     │ Push / In-App   │
     └─────────────────┘
```

---

# Core Components

## 1. Clients

The system supports multiple client types:

- Web
- Mobile
- Desktop

Clients communicate with the backend through:

- HTTP/HTTPS APIs
- WebSockets
- CDN endpoints for media

The client uses WebSockets for real-time events such as:

- New messages
- Message delivery updates
- Typing indicators
- Online/offline status
- Read receipts
- Connection state changes

---

# 2. API Gateway

The API Gateway is the primary entry point for synchronous client requests.

### Responsibilities

- Authentication
- Rate limiting
- Request routing
- Load balancing
- API versioning
- Request validation
- Security policies

Example request flow:

```text
Client
  │
  ▼
API Gateway
  │
  ├── User Service
  ├── Chat Service
  ├── Group Service
  └── Media Service
```

The gateway prevents clients from needing to know the internal structure of the backend.

---

# 3. WebSocket Servers

WebSocket servers maintain persistent connections with clients and enable bi-directional real-time communication.

### Responsibilities

- Establish WebSocket connections
- Authenticate connections
- Receive real-time messages
- Send messages to connected clients
- Handle connection lifecycle
- Broadcast real-time events

Example:

```text
Client A
   │
   │ WebSocket
   ▼
WebSocket Server
   │
   │
   ▼
Chat Service
   │
   ▼
WebSocket Server
   │
   │ WebSocket
   ▼
Client B
```

Because multiple WebSocket servers can exist behind a load balancer, the system requires shared connection state.

That responsibility is handled by the **Connection Manager**.

---

# 4. Connection Manager

The Connection Manager manages the lifecycle of WebSocket connections.

### Responsibilities

- Track active WebSocket connections
- Handle connect/disconnect events
- Maintain connection metadata
- Map users to active connections
- Coordinate connections across WebSocket servers
- Manage session state

Redis can be used as the shared connection/session store.

Example:

```text
User ID
   │
   ▼
Connection Manager
   │
   ▼
Redis
   │
   ├── WebSocket Server 1
   ├── WebSocket Server 2
   └── WebSocket Server 3
```

This prevents the system from depending on a single WebSocket server.

---

# 5. User Service

The User Service manages user-related functionality.

### Responsibilities

- Authentication
- User profiles
- Device management
- Session-related user information

### User Profile Storage

PostgreSQL can store:

```text
Users
Profiles
Devices
Authentication metadata
```

Example:

```json
{
  "userId": "user_123",
  "name": "Alex",
  "email": "alex@example.com",
  "devices": [
    {
      "deviceId": "device_001",
      "platform": "ios"
    }
  ]
}
```

---

# 6. Chat Service

The Chat Service is the core business service responsible for message processing.

### Responsibilities

- Validate messages
- Persist messages
- Manage message delivery state
- Orchestrate message delivery
- Process conversations
- Coordinate with group and presence services

Example message:

```json
{
  "messageId": "msg_123",
  "conversationId": "conv_456",
  "senderId": "user_001",
  "content": "Hello!",
  "type": "text",
  "createdAt": "2026-09-22T15:30:00Z"
}
```

### Message Lifecycle

```text
Client
  │
  ▼
WebSocket
  │
  ▼
Chat Service
  │
  ├── Validate
  │
  ├── Persist
  │
  ├── Determine Recipients
  │
  └── Update Delivery State
           │
           ▼
      Recipient
```

---

# 7. Message Store

PostgreSQL can be used to persist messages and delivery state.

Example entities:

```text
Users
Conversations
ConversationMembers
Messages
MessageDelivery
Attachments
```

A message record could contain:

```json
{
  "messageId": "msg_001",
  "conversationId": "conversation_001",
  "senderId": "user_001",
  "messageType": "text",
  "content": "Hello",
  "createdAt": "2026-09-22T10:30:00Z"
}
```

Delivery state can be represented as:

```text
SENT
DELIVERED
READ
FAILED
```

---

# 8. Group Service

The Group Service manages group conversations.

### Responsibilities

- Group creation
- Group management
- Membership management
- Recipient resolution
- Group metadata

Example:

```json
{
  "groupId": "group_123",
  "name": "Engineering",
  "createdBy": "user_001",
  "members": ["user_001", "user_002", "user_003"]
}
```

### Group Message Flow

```text
Sender
  │
  ▼
Chat Service
  │
  ▼
Group Service
  │
  ▼
Resolve Members
  │
  ├── User A
  ├── User B
  └── User C
```

The Chat Service can then determine how each recipient should receive the message.

---

# 9. Presence Service

The Presence Service manages real-time user presence information.

### Responsibilities

- Online/offline state
- Last seen
- Typing indicators
- Presence updates

Redis is suitable for high-frequency presence data because these values are frequently updated and queried.

Example:

```json
{
  "userId": "user_123",
  "status": "online",
  "lastSeen": "2026-09-22T15:35:00Z"
}
```

### Typing Indicator

```text
User A
  │
  │ "typing"
  ▼
Presence Service
  │
  ▼
Redis
  │
  ▼
WebSocket Server
  │
  ▼
User B
```

Typing indicators should generally have a short expiration time so stale state does not remain in Redis.

---

# 10. Media Service

The Media Service handles chat attachments.

### Responsibilities

- Upload media
- Download media
- Generate thumbnails
- Store media metadata
- Manage media access

Amazon S3 can be used for object storage.

Example supported media:

- Images
- Videos
- Audio
- Documents
- Other attachments

---

## Media Upload Flow

```text
Client
   │
   ▼
Media Service
   │
   ▼
Object Storage
(S3)
   │
   ▼
Media Metadata
   │
   ▼
Chat Message
```

For large files, clients can use pre-signed URLs to upload directly to object storage instead of sending the complete file through the application servers.

---

# 11. CDN for Media

A CDN distributes media files geographically to reduce latency.

Example:

```text
Client
   │
   ▼
CDN
   │
   ▼
Object Storage
(S3)
```

Benefits include:

- Lower latency
- Reduced application-server bandwidth
- Global media delivery
- Better scalability
- Caching of frequently accessed media

---

# 12. Notification Queue

The Notification Queue separates real-time chat processing from push notification processing.

The queue receives asynchronous notification events.

Example:

```json
{
  "eventId": "evt_001",
  "eventType": "MESSAGE_RECEIVED",
  "userId": "user_123",
  "messageId": "msg_456",
  "conversationId": "conv_789"
}
```

The Chat Service does not need to wait for push notification providers to respond.

Instead:

```text
Chat Service
     │
     ▼
Notification Queue
     │
     ▼
Notification Service
```

---

# 13. Notification Service

The Notification Service processes asynchronous notification events.

### Responsibilities

- Process notification events
- Determine notification targets
- Send push notifications
- Send in-app notifications
- Handle provider integrations

Potential push providers:

- Firebase Cloud Messaging
- Apple Push Notification Service

Example:

```text
Notification Queue
       │
       ▼
Notification Service
       │
       ├── FCM
       ├── APNs
       └── In-App Notification Store
```

---

# Real-Time Message Flow

A typical one-to-one message flow looks like this:

```text
Client A
   │
   │ WebSocket
   ▼
WebSocket Server
   │
   ▼
Chat Service
   │
   ├── Validate Message
   │
   ├── Persist Message
   │
   ├── Determine Recipient
   │
   └── Update Delivery State
   │
   ├───────────────┐
   │               │
   ▼               ▼
WebSocket       Notification
Server          Queue
   │               │
   ▼               ▼
Client B      Notification
              Service
```

---

# Message Delivery States

A message can move through the following states:

```text
CREATED
   │
   ▼
SENT
   │
   ▼
DELIVERED
   │
   ▼
READ
```

Failures can be represented separately:

```text
SENT
  │
  └──► FAILED
```

The exact delivery semantics depend on the implementation.

---

# Offline User Flow

When a recipient is offline:

```text
Sender
   │
   ▼
Chat Service
   │
   ▼
Persist Message
   │
   ▼
Presence Service
   │
   ├── User Online
   │      │
   │      ▼
   │   WebSocket
   │
   └── User Offline
          │
          ▼
   Notification Queue
          │
          ▼
   Notification Service
          │
          ▼
     Push Provider
```

When the user reconnects, the client can fetch messages that were not yet synchronized.

---

# Connection Lifecycle

A WebSocket connection follows:

```text
CONNECT
   │
   ▼
AUTHENTICATE
   │
   ▼
REGISTER CONNECTION
   │
   ▼
ACTIVE
   │
   ├── Receive Message
   ├── Send Message
   ├── Typing Event
   └── Presence Event
   │
   ▼
DISCONNECT
   │
   ▼
UPDATE PRESENCE
```

The Connection Manager is responsible for maintaining this lifecycle.

---

# Scalability

The architecture is designed so that services can be horizontally scaled.

Example:

```text
                 Load Balancer
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
 WebSocket 1      WebSocket 2    WebSocket 3
       │              │              │
       └──────────────┼──────────────┘
                      │
                    Redis
```

Other services can similarly run multiple instances:

```text
API Gateway
     │
     ├── User Service × N
     ├── Chat Service × N
     ├── Group Service × N
     ├── Presence Service × N
     ├── Media Service × N
     └── Notification Service × N
```

---

# Load Balancing

Load balancers distribute requests across service instances.

For HTTP APIs:

```text
Client
  │
  ▼
API Gateway
  │
  ▼
Service Load Balancer
  │
  ├── Instance 1
  ├── Instance 2
  └── Instance 3
```

For WebSockets, the architecture should account for long-lived connections and connection affinity where required.

Shared connection state in Redis helps prevent application-level dependence on a single server.

---

# Caching Strategy

Redis can be used for high-frequency, low-latency data.

Potential cached data:

- Active connections
- User presence
- Typing indicators
- Sessions
- Frequently accessed user metadata
- Temporary delivery state

Example:

```text
Redis
├── user:{id}:presence
├── user:{id}:connections
├── conversation:{id}:members
└── session:{id}
```

Cache expiration should be configured appropriately for temporary data.

---

# Data Storage

Different types of data can be stored according to access patterns.

| Data                   | Suggested Storage            |
| ---------------------- | ---------------------------- |
| User profiles          | PostgreSQL                   |
| Groups                 | PostgreSQL                   |
| Group memberships      | PostgreSQL                   |
| Messages               | PostgreSQL                   |
| Message delivery state | PostgreSQL                   |
| Presence               | Redis                        |
| Active connections     | Redis                        |
| Typing indicators      | Redis                        |
| Sessions               | Redis                        |
| Media files            | Amazon S3                    |
| Media metadata         | PostgreSQL                   |
| Notifications          | PostgreSQL / dedicated store |

---

# Security

The system should implement:

- Authentication
- Authorization
- TLS/HTTPS
- WebSocket authentication
- API rate limiting
- Input validation
- Message authorization
- Group membership validation
- Secure media access
- Signed media URLs
- Secret management
- Database access controls
- Audit logging

### WebSocket Authentication

A WebSocket connection should be authenticated before being registered as active.

```text
Client
  │
  │ Authentication Token
  ▼
WebSocket Server
  │
  ▼
User Service / Auth
  │
  ├── Valid → Register Connection
  │
  └── Invalid → Reject Connection
```

---

# Reliability

The system should be designed to tolerate failures in individual components.

For example:

```text
Notification Service ❌
        │
        ▼
Notification Queue
        │
        ▼
Events remain available
        │
        ▼
Notification Service recovers
        │
        ▼
Events processed
```

Similarly, a WebSocket server failure should not result in permanent message loss because messages are persisted independently of the connection.

---

# Idempotency

Message processing should be idempotent where possible.

Each message should have a unique ID:

```json
{
  "messageId": "msg_123456"
}
```

If a request is retried:

```text
Request 1 → msg_123
Request 2 → msg_123
```

The system should recognize that the message has already been processed and avoid creating unintended duplicates.

---

# Message Ordering

Real-time chat may require ordering guarantees.

A common strategy is to maintain ordering within a conversation.

For example:

```text
Conversation A

Message 101
Message 102
Message 103
Message 104
```

The implementation can use:

- Conversation-level sequence numbers
- Server timestamps
- Ordered queue partitions
- Database constraints

Global ordering across the entire system is generally unnecessary and more expensive than conversation-level ordering.

---

# Backpressure

High traffic can cause a sudden increase in message volume.

The system can use:

- Queue buffering
- Rate limiting
- Worker autoscaling
- Connection limits
- Database connection pooling
- Provider throttling

Example:

```text
High Traffic
     │
     ▼
Chat Service
     │
     ▼
Queue / Buffer
     │
     ▼
Workers
     │
     ▼
Downstream Services
```

This prevents sudden traffic spikes from overwhelming downstream services.

---

# Observability

A production implementation should include centralized observability.

### Metrics

Useful metrics include:

- Active WebSocket connections
- Messages per second
- Message processing latency
- Message delivery latency
- WebSocket connection failures
- Online users
- Queue depth
- Notification delivery rate
- API latency
- Error rate
- Database latency
- Redis latency

### Logs

Centralized logging can use:

- ELK
- Loki
- CloudWatch

### Distributed Tracing

OpenTelemetry can trace requests across services:

```text
Client
  │
  ▼
API Gateway
  │
  ▼
Chat Service
  │
  ├── User Service
  ├── Group Service
  ├── Presence Service
  └── Notification Queue
```

### Dashboards

Grafana can provide dashboards for:

- System health
- WebSocket connections
- Message throughput
- Queue health
- Service latency
- Error rates

---

# API Examples

## Send Message

```http
POST /api/v1/conversations/{conversationId}/messages
Authorization: Bearer <token>
Content-Type: application/json
```

Request:

```json
{
  "type": "text",
  "content": "Hello!"
}
```

Response:

```json
{
  "messageId": "msg_123",
  "status": "SENT"
}
```

---

## Get Conversation Messages

```http
GET /api/v1/conversations/{conversationId}/messages
Authorization: Bearer <token>
```

Example response:

```json
{
  "messages": [
    {
      "messageId": "msg_001",
      "senderId": "user_001",
      "content": "Hello!",
      "type": "text",
      "createdAt": "2026-09-22T10:30:00Z",
      "status": "READ"
    }
  ]
}
```

---

## Create Group

```http
POST /api/v1/groups
Authorization: Bearer <token>
Content-Type: application/json
```

Request:

```json
{
  "name": "Engineering",
  "members": ["user_001", "user_002", "user_003"]
}
```

---

## Upload Media

```http
POST /api/v1/media/upload
Authorization: Bearer <token>
```

For large files, the API can return a pre-signed object-storage URL:

```json
{
  "uploadUrl": "<signed-url>",
  "mediaId": "media_123"
}
```

The client can then upload the file directly to object storage.

---

# Suggested Technology Stack

| Layer              | Technology                     |
| ------------------ | ------------------------------ |
| Frontend           | React / Next.js / React Native |
| API                | Node.js / Java / Go / Python   |
| Real-Time          | WebSockets                     |
| API Gateway        | NGINX / Kong / AWS API Gateway |
| Cache              | Redis                          |
| Primary Database   | PostgreSQL                     |
| Object Storage     | Amazon S3                      |
| CDN                | CloudFront / Cloudflare        |
| Message Queue      | Kafka / RabbitMQ / Amazon SQS  |
| Push Notifications | FCM / APNs                     |
| Metrics            | Prometheus                     |
| Dashboards         | Grafana                        |
| Logging            | ELK / Loki / CloudWatch        |
| Tracing            | OpenTelemetry                  |
| Containers         | Docker                         |
| Orchestration      | Kubernetes / ECS               |

The exact technologies can be changed depending on deployment requirements.

---

# Suggested Project Structure

```text
chat-system/
│
├── services/
│   ├── api-gateway/
│   ├── user-service/
│   ├── chat-service/
│   ├── group-service/
│   ├── presence-service/
│   ├── media-service/
│   ├── connection-manager/
│   └── notification-service/
│
├── realtime/
│   └── websocket-server/
│
├── workers/
│   └── notification-worker/
│
├── shared/
│   ├── types/
│   ├── events/
│   ├── config/
│   ├── auth/
│   └── utilities/
│
├── infrastructure/
│   ├── postgres/
│   ├── redis/
│   ├── kafka/
│   ├── docker/
│   └── kubernetes/
│
├── docs/
│   └── chat-system-architecture.png
│
├── .env.example
├── docker-compose.yml
└── README.md
```

---

# Environment Variables

Example:

```env
# Application
NODE_ENV=development
PORT=3000

# PostgreSQL
DATABASE_URL=postgresql://user:password@localhost:5432/chat

# Redis
REDIS_URL=redis://localhost:6379

# Object Storage
AWS_REGION=
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET=

# Notifications
FCM_PROJECT_ID=
FCM_PRIVATE_KEY=
FCM_CLIENT_EMAIL=

# Observability
OTEL_EXPORTER_OTLP_ENDPOINT=
```

Never commit production credentials or secrets to source control.

---

# Running Locally

## Prerequisites

Install:

- Node.js
- Docker
- Docker Compose
- PostgreSQL
- Redis

Optional infrastructure:

- Kafka
- LocalStack / AWS services

## Start Infrastructure

```bash
docker compose up -d
```

## Install Dependencies

```bash
npm install
```

## Start Development Environment

```bash
npm run dev
```

For production:

```bash
npm run build
npm start
```

> Update these commands according to the actual implementation of the repository.

---

# Design Principles

## Separation of Concerns

Each service owns a specific responsibility.

```text
User Service        → Users
Chat Service        → Messages
Group Service       → Groups
Presence Service    → Presence
Media Service       → Media
Notification        → Push Notifications
Connection Manager  → Connections
WebSocket Servers   → Real-Time Transport
```

This makes individual components easier to develop, deploy, and scale.

---

## Synchronous vs Asynchronous Communication

The architecture distinguishes between synchronous requests and asynchronous events.

### Synchronous

Used when an immediate response is required:

```text
Client
  │
  ▼
API Gateway
  │
  ▼
Service
  │
  ▼
Response
```

### Asynchronous

Used for operations that do not need to block the main request:

```text
Chat Service
     │
     ▼
Notification Queue
     │
     ▼
Notification Service
```

---

# Failure Scenarios

### WebSocket Server Failure

```text
WebSocket Server ❌
        │
        ▼
Client reconnects
        │
        ▼
Another WebSocket Server
```

Persisted messages remain available because message storage is independent of the WebSocket connection.

### Redis Failure

The system should have appropriate fallback behavior depending on the type of Redis data.

For example:

- Presence may temporarily become unavailable.
- Connection metadata may need reconstruction after reconnect.
- Persistent message data remains in PostgreSQL.

### Notification Service Failure

Messages remain in the notification queue until the service recovers.

```text
Chat Service
     │
     ▼
Notification Queue
     │
     X
Notification Service
     │
     │ Recovery
     ▼
Notification Service
```

---

# Future Improvements

Possible extensions include:

- End-to-end encryption
- Message reactions
- Message editing and deletion
- Read receipts
- Voice and video calling
- Message search
- Full-text search with Elasticsearch/OpenSearch
- Message pagination
- Conversation archiving
- Scheduled messages
- Multi-device synchronization
- Message fan-out optimization
- Multi-region deployment
- Automated WebSocket autoscaling
- Advanced abuse detection
- Spam prevention
- Content moderation
- Message retention policies
- Data export and deletion workflows

---

# End-to-End Architecture

```text
                         CLIENTS
                  ┌─────────┼─────────┐
                  │         │         │
                 Web      Mobile    Desktop
                  │         │         │
                  └─────────┼─────────┘
                            │
                 ┌──────────┴──────────┐
                 │                     │
                 ▼                     ▼
           API Gateway             Media CDN
                 │                     │
       ┌─────────┼──────────┐         │
       │         │          │         ▼
       ▼         ▼          ▼      S3 Storage
     User      Chat       Group
    Service   Service     Service
                 │
                 ├───────────────┐
                 │               │
                 ▼               ▼
             Presence       Connection
              Service         Manager
                 │               │
                 ▼               ▼
               Redis        WebSocket
                              Servers
                                 │
                                 ▼
                              Clients

                 Chat Service
                      │
                      ▼
              Notification Queue
                      │
                      ▼
             Notification Service
                      │
               ┌──────┴──────┐
               ▼             ▼
              FCM           APNs
```

---

# Summary

This project demonstrates a production-oriented architecture for a **distributed real-time chat platform**.

The system separates:

**Client Communication + API Routing + WebSocket Connections + Chat Processing + User Management + Group Management + Presence + Media + Notifications**

The architecture combines:

- Real-time WebSocket communication
- Microservice-oriented separation
- Redis-based connection and presence management
- PostgreSQL-based persistent data
- S3-based media storage
- CDN-based media delivery
- Asynchronous notification processing
- Horizontal scalability
- Fault isolation
- Idempotent message processing
- Observability

The overall design is intended to provide a foundation for a chat application that can scale from a small deployment to a large, distributed system.
