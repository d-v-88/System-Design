# Notification System

A scalable, event-driven notification system designed to deliver
notifications across multiple channels such as **Email, SMS, Push
Notifications, and In-App Notifications**.

The system uses an asynchronous architecture with an **Event Broker,
Notification Orchestrator, Channel Queues, Independent Channel Workers,
external delivery providers, retry handling, and observability**.

> **Architecture:** Event-driven • Asynchronous • Horizontally Scalable
> • Fault Tolerant • Multi-Channel

---

## Architecture

![Notification System
Architecture](./architecture.png)

### High-Level Flow

```text
Business Services
       │
       ▼
Event Ingestor
(Validation / Auth / Rate Limiting / Idempotency)
       │
       ▼
Event Broker
(Kafka / SQS)
       │
       ▼
Notification Orchestrator
       │
       ├── Preference Service
       ├── Template Service
       ├── Delivery Tracker
       ├── Redis Cache
       ├── Template Cache
       └── Status Store
       │
       ▼
Channel Queues
 ┌─────┼─────┬──────────┐
 ▼     ▼     ▼          ▼
Email  SMS   Push      In-App
Queue  Queue Queue     Queue
 │      │     │          │
 ▼      ▼     ▼          ▼
Workers Workers Workers Workers
 │      │     │          │
 ▼      ▼     ▼          ▼
Email   SMS   Push     In-App
Providers Providers Providers Store
 │      │     │          │
 └──────┴─────┴──────────┘
              │
              ▼
       User Notification API
              │
              ▼
          Client Apps
```

---

## Key Components

### 1. Business Services

Business services generate notification events.

Examples:

- Order Service
- User Service
- Payment Service
- Authentication Service
- Messaging Service
- Other domain services

Example event:

```json
{
  "eventId": "evt_123456",
  "eventType": "ORDER_CREATED",
  "userId": "user_123",
  "timestamp": "2026-09-22T10:30:00Z",
  "data": {
    "orderId": "order_987",
    "amount": 1499
  }
}
```

---

### 2. Event Ingestor

The Event Ingestor acts as the entry point for notification events.

Responsibilities:

- Request validation
- Authentication and authorization
- Rate limiting
- Idempotency
- Event normalization
- Publishing events to the event broker

This layer prevents invalid or duplicate events from entering the
notification pipeline.

---

### 3. Event Broker

The Event Broker decouples business services from the notification
system.

Supported architectural options:

- Apache Kafka
- Amazon SQS

The broker provides asynchronous processing and allows the notification
system to scale independently from upstream services.

---

### 4. Notification Orchestrator

The Notification Orchestrator is the core decision-making layer.

It determines:

- Which users should receive the notification
- Which channels should be used
- Whether the user has opted into a channel
- Which template should be used
- What content should be generated
- Which channel queues should receive the notification

The orchestrator can integrate with:

#### Preference Service

Stores user notification preferences.

Example:

```json
{
  "userId": "user_123",
  "email": true,
  "sms": false,
  "push": true,
  "inApp": true
}
```

#### Template Service

Manages reusable notification templates.

Example:

```text
Order Confirmation

Hi {{name}},

Your order {{orderId}} has been successfully placed.
```

#### Delivery Tracker

Tracks the lifecycle of each notification:

```text
CREATED
   ↓
QUEUED
   ↓
PROCESSING
   ↓
SENT
```

Failure states can include:

```text
FAILED
RETRYING
DEAD_LETTERED
```

---

## 5. Channel Queues

Notifications are separated into independent queues.

```text
Email Queue
SMS Queue
Push Queue
In-App Queue
```

Separating channels provides:

- Independent scaling
- Failure isolation
- Channel-specific retry policies
- Better throughput management
- Reduced impact from provider failures

For example, if an SMS provider is unavailable, email and push
notification processing can continue independently.

---

## 6. Channel Workers

Each channel has its own independently scalable worker pool.

```text
Email Worker
SMS Worker
Push Worker
In-App Worker
```

Workers consume messages from their respective queues and perform the
actual delivery.

Example:

```text
Email Queue
     │
     ▼
Email Worker
     │
     ▼
Email Provider
```

Workers can be scaled independently based on traffic.

For example:

```text
Email Workers: 10
SMS Workers: 4
Push Workers: 8
In-App Workers: 3
```

The exact number of workers can be adjusted based on workload and queue
depth.

---

## 7. Delivery Providers

The system can integrate with multiple external delivery providers.

### Email

Examples:

- SendGrid
- Amazon SES
- SMTP

### SMS

Examples:

- Twilio
- Amazon SNS

### Push Notifications

Examples:

- Firebase Cloud Messaging
- Apple Push Notification Service (APNs)

### In-App Notifications

Notifications can be stored internally and exposed through the
application's notification API.

---

## 8. Retry & Failure Handling

The system uses asynchronous retry processing to handle temporary
delivery failures.

### Retry Flow

```text
Worker Failure
      │
      ▼
Retry Policy
(Exponential Backoff)
      │
      ▼
Retry Queue
      │
      ▼
Worker
      │
      ├── Success ─────► Delivery Tracker
      │
      └── Failure
              │
              ▼
        Dead Letter Queue
```

### Exponential Backoff

A typical retry schedule could be:

```text
Attempt 1 → Immediate
Attempt 2 → 1 second
Attempt 3 → 5 seconds
Attempt 4 → 30 seconds
Attempt 5 → 5 minutes
```

The exact policy should be configurable.

### Dead Letter Queue

Messages that repeatedly fail are moved to a Dead Letter Queue (DLQ).

This prevents permanently failing events from blocking normal
processing.

DLQ messages can later be:

- Investigated
- Replayed
- Manually corrected
- Permanently discarded

---

## 9. Observability

The system is designed with observability as a first-class concern.

### Metrics

Example metrics:

- Notifications processed
- Notifications delivered
- Notifications failed
- Queue depth
- Processing latency
- Provider latency
- Retry count
- Dead-lettered messages
- Channel-specific throughput

Prometheus can be used for metrics collection.

### Logs

Centralized logs can be collected using systems such as:

- ELK
- Loki
- CloudWatch

### Distributed Tracing

OpenTelemetry can be used to trace a notification across services:

```text
Business Service
      │
      ▼
Event Ingestor
      │
      ▼
Kafka / SQS
      │
      ▼
Orchestrator
      │
      ▼
Channel Worker
      │
      ▼
External Provider
```

### Dashboards & Alerts

Grafana can be used for dashboards.

Alerting can be configured for conditions such as:

- High queue depth
- Increased delivery failures
- Provider outages
- High processing latency
- Excessive retries
- Worker failures

---

# Core Design Principles

## Asynchronous Processing

Business services should not wait for external notification providers.

Instead:

```text
Business Request
      │
      ▼
Create Event
      │
      ▼
Publish Event
      │
      ▼
Return Response
```

Notification delivery happens asynchronously.

This reduces latency for upstream services and prevents provider
failures from directly affecting business requests.

---

## Idempotency

Every event should have a unique identifier.

Example:

```json
{
  "eventId": "evt_123456"
}
```

The system should ensure that processing the same event multiple times
does not result in unintended duplicate notifications.

An idempotency store or status store can be used to track processed
events.

---

## Horizontal Scalability

Channel workers are independently scalable.

```text
             ┌── Email Worker
Email Queue ─┼── Email Worker
             └── Email Worker


             ┌── Push Worker
Push Queue ──┼── Push Worker
             └── Push Worker
```

This allows individual channels to scale based on demand.

---

## Fault Isolation

External provider failures should not bring down the complete
notification system.

For example:

```text
SMS Provider ❌
      │
      ▼
SMS Worker → Retry Queue → DLQ

Email Provider ✓
      │
      ▼
Email Worker → Successful Delivery
```

Other channels can continue operating normally.

---

# Notification Lifecycle

A notification follows this general lifecycle:

```text
1. Business Event Created
          ↓
2. Event Validated
          ↓
3. Event Authenticated
          ↓
4. Idempotency Check
          ↓
5. Event Published
          ↓
6. Notification Orchestrated
          ↓
7. User Preferences Checked
          ↓
8. Template Selected
          ↓
9. Notification Added to Channel Queue
          ↓
10. Channel Worker Processes Message
          ↓
11. External Provider / In-App Store
          ↓
12. Delivery Status Recorded
```

---

# Example Notification Request

A business service could publish an event like:

```json
{
  "eventId": "evt_001",
  "eventType": "PAYMENT_SUCCESS",
  "userId": "user_123",
  "channels": ["email", "push", "inApp"],
  "data": {
    "amount": 2500,
    "currency": "INR",
    "transactionId": "txn_123"
  }
}
```

The orchestrator evaluates the event and user preferences:

```text
PAYMENT_SUCCESS
      │
      ▼
Preference Service
      │
      ├── Email ✓
      ├── SMS ✗
      ├── Push ✓
      └── In-App ✓
      │
      ▼
 ┌────┼────────┐
 ▼    ▼        ▼
Email Push   In-App
Queue Queue   Queue
```

---

# Suggested Technology Stack

The architecture can be implemented using the following technologies:

Layer Technology

---

API / Services Node.js / Java / Go / Python
Event Broker Apache Kafka / Amazon SQS
Cache Redis
Primary Database PostgreSQL
Email SendGrid / Amazon SES / SMTP
SMS Twilio / Amazon SNS
Push Firebase Cloud Messaging / APNs
In-App PostgreSQL / Redis
Metrics Prometheus
Dashboards Grafana
Logs ELK / Loki / CloudWatch
Tracing OpenTelemetry
Containerization Docker
Orchestration Kubernetes / ECS

The exact implementation can vary depending on infrastructure and scale
requirements.

---

# Suggested Project Structure

```text
notification-system/
│
├── services/
│   ├── event-ingestor/
│   ├── notification-orchestrator/
│   ├── preference-service/
│   ├── template-service/
│   ├── delivery-tracker/
│   └── notification-api/
│
├── workers/
│   ├── email-worker/
│   ├── sms-worker/
│   ├── push-worker/
│   └── in-app-worker/
│
├── shared/
│   ├── types/
│   ├── events/
│   ├── config/
│   ├── logging/
│   └── utilities/
│
├── infrastructure/
│   ├── kafka/
│   ├── redis/
│   ├── postgres/
│   └── docker/
│
├── docs/
│   └── notification-system-architecture.png
│
├── .env.example
├── docker-compose.yml
└── README.md
```

---

# Environment Variables

Create a `.env` file based on `.env.example`.

Example:

```env
# Application
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/notifications

# Redis
REDIS_URL=redis://localhost:6379

# Kafka
KAFKA_BROKERS=localhost:9092
KAFKA_CLIENT_ID=notification-system

# Email
SENDGRID_API_KEY=
AWS_SES_REGION=

# SMS
TWILIO_ACCOUNT_SID=
TWILIO_AUTH_TOKEN=
TWILIO_PHONE_NUMBER=

# Push
FCM_PROJECT_ID=
FCM_PRIVATE_KEY=
FCM_CLIENT_EMAIL=

# Observability
OTEL_EXPORTER_OTLP_ENDPOINT=
```

Do not commit production credentials or secrets to the repository.

---

# Running Locally

## Prerequisites

Install:

- Node.js
- Docker
- Docker Compose
- PostgreSQL
- Redis

Kafka can either be run locally using Docker or replaced with Amazon SQS
for AWS deployments.

## Start Infrastructure

```bash
docker compose up -d
```

## Install Dependencies

```bash
npm install
```

## Start the Application

```bash
npm run dev
```

For production:

```bash
npm run build
npm start
```

> Update these commands according to the actual implementation of the
> repository.

---

# API Examples

## Publish Notification Event

```http
POST /api/v1/events
Content-Type: application/json
Authorization: Bearer <token>
```

Request:

```json
{
  "eventType": "ORDER_CREATED",
  "userId": "user_123",
  "data": {
    "orderId": "order_456"
  }
}
```

Response:

```json
{
  "eventId": "evt_123456",
  "status": "accepted"
}
```

---

## Get Notification Status

```http
GET /api/v1/notifications/{notificationId}
Authorization: Bearer <token>
```

Example response:

```json
{
  "notificationId": "notification_123",
  "status": "DELIVERED",
  "channel": "email",
  "deliveredAt": "2026-09-22T10:35:00Z"
}
```

---

## Get User Notifications

```http
GET /api/v1/users/{userId}/notifications
Authorization: Bearer <token>
```

Example:

```json
{
  "notifications": [
    {
      "id": "notification_001",
      "title": "Payment Successful",
      "message": "Your payment was successfully processed.",
      "read": false,
      "createdAt": "2026-09-22T10:30:00Z"
    }
  ]
}
```

---

# Security Considerations

The notification system should include:

- Authentication and authorization
- API rate limiting
- Input validation
- Secret management
- Encryption in transit
- Encryption at rest
- Provider credential isolation
- Audit logging
- Idempotency protection
- Access control for notification data
- Protection against notification abuse and spam

Sensitive provider credentials should be stored using a secret manager
rather than committed to source control.

---

# Scalability Considerations

The architecture is designed to support high notification volumes.

Potential scaling strategies include:

### Queue Partitioning

Kafka topics can be partitioned by:

```text
userId
notificationType
channel
tenantId
```

### Worker Autoscaling

Workers can scale based on:

- Queue depth
- CPU utilization
- Processing latency
- Message age

### Provider Rate Limits

External providers may impose rate limits.

The worker layer should therefore support:

- Rate limiting
- Backpressure
- Connection pooling
- Provider-specific retry policies

### Multi-Provider Failover

A channel can support multiple providers.

Example:

```text
Email Worker
     │
     ▼
Primary Provider
     │
     ├── Success → Done
     │
     └── Failure
            │
            ▼
      Secondary Provider
```

This can improve resilience when a provider experiences an outage.

---

# Reliability Features

The architecture supports:

- Asynchronous processing
- Idempotent event handling
- Retry queues
- Exponential backoff
- Dead Letter Queues
- Delivery tracking
- Independent channel workers
- Provider failover
- Rate limiting
- Monitoring and alerting
- Distributed tracing

---

# Future Improvements

Possible extensions include:

- Multi-tenant notification management
- Scheduled notifications
- Notification batching
- Digest notifications
- User timezone support
- Quiet hours
- Priority-based delivery
- Notification campaigns
- A/B testing of notification templates
- Multi-region deployment
- Provider health checks
- Automatic provider failover
- WebSocket-based real-time notifications
- Notification analytics
- ML-based send-time optimization
- Intelligent channel selection

---

# Design Goals

The main goals of this system are:

```text
High Throughput
      +
Low Latency
      +
Reliability
      +
Fault Isolation
      +
Horizontal Scalability
      +
Observability
      +
Multi-Channel Delivery
```

The architecture separates **event ingestion, notification
decision-making, channel processing, and delivery**, allowing each part
of the system to evolve and scale independently.

---

# License

This project is available under the license specified in the repository.

---

## Architecture Summary

```text
┌──────────────────┐
│ Business Services│
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  Event Ingestor  │
│ Validation       │
│ Auth             │
│ Rate Limiting    │
│ Idempotency      │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│ Event Broker     │
│ Kafka / SQS      │
└────────┬─────────┘
         │
         ▼
┌──────────────────────────┐
│ Notification Orchestrator│
│                          │
│ Preferences              │
│ Templates                │
│ Delivery Tracking        │
└──────────┬───────────────┘
           │
     ┌─────┼─────┬─────────┐
     ▼     ▼     ▼         ▼
   Email   SMS   Push     In-App
   Queue  Queue  Queue     Queue
     │     │      │         │
     ▼     ▼      ▼         ▼
  Worker Worker Worker    Worker
     │     │      │         │
     ▼     ▼      ▼         ▼
 Providers / In-App Store
           │
           ▼
   User Notification API
           │
           ▼
       Client Apps
```

---

## Summary

This project demonstrates a production-oriented architecture for
building a **distributed notification platform** capable of handling
multiple notification channels while maintaining scalability,
reliability, fault isolation, and observability.

The core architectural pattern is:

**Event-Driven Architecture + Message Queues + Independent Workers +
Retry/DLQ + Provider Abstraction + Centralized Observability.**
