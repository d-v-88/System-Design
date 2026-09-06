# TinyURL — System Design

A production-grade, horizontally scalable URL-shortening system designed to generate short URLs, perform ultra-low-latency redirects, and process click analytics asynchronously.

---

## Architecture Overview

![System Design](./architecture.svg)

## 1. Overview

TinyURL allows users to:

- Create short URLs
- Redirect users using short URLs
- View URL analytics
- Manage their URLs
- Set URL expiration
- Track clicks, devices, countries, browsers, and referrers

The architecture separates the system into two primary paths:

- **Write Path** — URL creation and persistence
- **Read Path** — High-volume URL redirection

Analytics are processed asynchronously using Kafka so that analytics processing does not increase redirect latency.

---

## 2. Functional Requirements

The system should allow users to:

1. Sign up and log in
2. Create short URLs
3. Redirect users using short URLs
4. View and manage their URLs
5. Set URL expiration
6. View click analytics
7. View geographic analytics
8. View device and browser analytics

---

## 3. Non-Functional Requirements

The system should provide:

- Low-latency redirects
- High availability
- Horizontal scalability
- Durable URL storage
- Globally unique short codes
- Fault isolation
- Asynchronous analytics processing
- Rate limiting
- Security and authentication
- Observability

---

# 4. High-Level Architecture

```text
                              ┌──────────────────────┐
                              │        USERS         │
                              │    Web / Mobile      │
                              └──────────┬───────────┘
                                         │
                                         ▼
                              ┌──────────────────────┐
                              │     API GATEWAY      │
                              │                      │
                              │ • Authentication     │
                              │ • Validation         │
                              │ • Rate Limiting      │
                              │ • Routing            │
                              └───────┬───────┬──────┘
                                      │       │
                       ┌──────────────┘       └──────────────┐
                       ▼                                     ▼
             ┌───────────────────┐               ┌───────────────────┐
             │    AUTH SERVICE   │               │ URL GENERATE SVC  │
             └───────────────────┘               └─────────┬─────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │    ZOOKEEPER    │
                                                  │  ID Allocation  │
                                                  └────────┬────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │  ID → Base62    │
                                                  │  Short Code     │
                                                  └────────┬────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │   POSTGRESQL    │
                                                  │ Durable Storage │
                                                  └────────┬────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │      REDIS      │
                                                  │   URL Cache     │
                                                  └─────────────────┘


                         SHORT URL REDIRECTION
                                  │
                                  ▼
                         ┌───────────────────┐
                         │   LOAD BALANCER   │
                         └─────────┬─────────┘
                                   │
                                   ▼
                         ┌───────────────────┐
                         │ REDIRECTION SVC   │
                         └─────────┬─────────┘
                                   │
                                   ▼
                               ┌───────┐
                               │ Redis │
                               └───┬───┘
                                   │
                         ┌─────────┴─────────┐
                         │                   │
                    Cache Hit            Cache Miss
                         │                   │
                         ▼                   ▼
                    HTTP 302             PostgreSQL
                         │                   │
                         │                   ▼
                         │                 Redis
                         │
                         ▼
                        USER

                         │
                         │ Click Event
                         ▼
                      ┌───────┐
                      │ Kafka │
                      └───┬───┘
                          │
                          ▼
               ┌─────────────────────┐
               │  ANALYTICS SERVICE  │
               └──────────┬──────────┘
                          │
               ┌──────────┴──────────┐
               ▼                     ▼
           Enricher              Aggregator
                                     │
                              ┌──────┴──────┐
                              ▼             ▼
                            Redis       PostgreSQL
```

---

# 5. Users / Clients

Users interact with the system through:

- Web applications
- Mobile applications

Users can:

- Create short URLs
- Redirect using short URLs
- View analytics
- Manage URLs
- Configure expiration
- Delete or disable URLs

---

# 6. API Gateway

The API Gateway acts as the **single entry point** for application API requests.

## Responsibilities

- JWT validation
- Authentication enforcement
- Request validation
- Rate limiting
- Request routing
- Basic security checks
- Request logging
- Correlation ID generation

## Create URL Flow

```text
User
  ↓
API Gateway
  ↓
URL Generation Service
```

## Analytics Flow

```text
User
  ↓
API Gateway
  ↓
Analytics Service
```

The API Gateway prevents clients from directly communicating with internal services.

---

# 7. Auth Service

The Auth Service handles authentication and authorization.

## Responsibilities

- User registration
- Login
- Password management
- JWT access tokens
- Refresh tokens
- Role-based authorization
- User management

## Authentication Flow

```text
Client
   ↓
API Gateway
   ↓
Auth Service
   ↓
JWT Access Token
   ↓
Client
```

The JWT access token is subsequently used when accessing protected APIs.

---

# 8. URL Generation Service

The URL Generation Service handles the **write path**.

Multiple instances run behind a load balancer or service-discovery mechanism.

```text
                 ┌─ Generate Instance 1
API Gateway ─────┼─ Generate Instance 2
                 └─ Generate Instance 3
```

## Responsibilities

1. Validate the long URL
2. Authenticate the user
3. Request an ID range from ZooKeeper when required
4. Generate numeric IDs locally
5. Convert the ID to Base62
6. Generate the short code
7. Store the URL mapping in PostgreSQL
8. Warm Redis
9. Return the shortened URL

---

# 9. Short Code Generation

The system first generates a globally unique numeric ID.

That numeric ID is then converted into a Base62 string.

## Example

```text
Long URL
   ↓
Numeric ID
12,584,932
   ↓
Base62 Encoding
   ↓
Xy12Ab
   ↓
https://t.example.com/r/Xy12Ab
```

## Base62 Character Set

```text
0-9
a-z
A-Z
```

Base62 allows the system to represent large numeric IDs using relatively short strings.

---

# 10. ZooKeeper — Distributed ID Coordination

ZooKeeper is responsible for coordinating distributed ID allocation.

Instead of asking ZooKeeper for one ID per request:

```text
Generate Service
       ↓
   ZooKeeper
       ↓
      1 ID
```

the service requests a large range of IDs.

## Example

```text
Generate Instance 1 → IDs 1,000,000 – 1,999,999
Generate Instance 2 → IDs 2,000,000 – 2,999,999
Generate Instance 3 → IDs 3,000,000 – 3,999,999
```

Each instance generates IDs locally after receiving its range.

## ID Allocation Flow

```text
Global Counter
      ↓
ZooKeeper
      ↓
Allocate ID Range
      ↓
URL Generate Instance
      ↓
Local ID Generation
```

## Benefits

- Reduces ZooKeeper coordination
- Reduces network calls
- Lowers URL creation latency
- Enables horizontal scaling
- Prevents ID collisions

A failed instance may leave some IDs unused, which is acceptable because uniqueness is more important than perfect ID utilization.

---

# 11. PostgreSQL Database

PostgreSQL acts as the **durable source of truth**.

## Stores

- Users
- URL mappings
- URL metadata
- Ownership
- Expiration information
- Tags
- Persistent analytics aggregates

## Main URL Mapping

```text
short_code → long_url
```

Example:

```text
Xy12Ab → https://example.com/product/123
```

PostgreSQL is primarily used for:

- Durable writes
- Cache misses
- URL management
- Persistent analytics

It is intentionally avoided on the majority of redirect requests.

---

# 12. Database Schema

## Users Table

```text
users
-------------------------
id
email
password_hash
role
created_at
updated_at
```

## URLs Table

```text
urls
-------------------------
id
short_code
long_url
user_id
created_at
expires_at
status
```

## Recommended Indexes

```text
UNIQUE(short_code)
INDEX(user_id)
INDEX(expires_at)
```

## Analytics Table

```text
url_analytics
-------------------------
id
short_code
date
total_clicks
unique_visitors
mobile_clicks
desktop_clicks
top_country
top_referrer
updated_at
```

---

# 13. Redis

Redis is used as the **hot-path cache**.

## URL Cache

```text
Key:
code:Xy12Ab

Value:
https://example.com/product/123
```

Redis prevents the Redirection Service from querying PostgreSQL for every request.

---

# 14. Cache Strategy

The system follows a **cache-aside** strategy.

## Cache Hit

```text
Short URL
    ↓
Redirection Service
    ↓
Redis
    ↓
Cache Hit
    ↓
Original URL
    ↓
HTTP 302
```

## Cache Miss

```text
Short URL
    ↓
Redirection Service
    ↓
Redis
    ↓
Cache Miss
    ↓
PostgreSQL
    ↓
Original URL
    ↓
Redis
    ↓
HTTP 302
```

After a cache miss, the URL is placed into Redis so future requests can be served directly from the cache.

---

# 15. Redirection Service

The Redirection Service handles the **high-volume read path**.

Multiple stateless instances run behind a load balancer.

```text
                 ┌─ Redirect Instance 1
Load Balancer ───┼─ Redirect Instance 2
                 └─ Redirect Instance 3
```

## Redirect Flow

```text
User
 ↓
Load Balancer
 ↓
Redirection Service
 ↓
Redis
 ↓
Original URL
 ↓
HTTP 302
```

The service is intentionally lightweight because redirect traffic can be significantly higher than URL creation traffic.

---

# 16. Cache-Miss Redirect Flow

```text
User
 │
 ▼
Load Balancer
 │
 ▼
Redirection Service
 │
 ▼
Redis
 │
 └── Cache Miss
       │
       ▼
   PostgreSQL
       │
       ▼
     Redis
       │
       ▼
    HTTP 302
       │
       ▼
      User
```

---

# 17. Kafka — Click Event Pipeline

The Redirection Service should not process analytics synchronously.

Instead, it publishes a click event to Kafka.

```text
Redirection Service
        │
        ├──────────────► User
        │                HTTP 302
        │
        └──────────────► Kafka
```

The redirect request does not wait for analytics processing.

---

# 18. Click Event

A click event can contain:

```json
{
  "eventId": "evt_12345",
  "shortCode": "Xy12Ab",
  "timestamp": "2026-09-06T12:30:45Z",
  "ip": "203.0.113.10",
  "userAgent": "Mozilla/5.0",
  "referrer": "https://google.com"
}
```

---

# 19. Analytics Service

The Analytics Service consumes click events from Kafka.

```text
                         Kafka
                           │
                           ▼
                     Click Events
                           │
                  ┌────────┴────────┐
                  ▼                 ▼
              Enricher          Aggregator
                  │                 │
                  ▼                 ▼
           Geo / Device        Redis Counters
                                    │
                                    ▼
                               PostgreSQL
```

---

# 20. Analytics Enricher

The Enricher adds additional information to raw click events.

## Enrichment

- Country
- Region
- Device
- Browser
- Operating system
- Bot classification

## Example

```text
Raw Click Event
      ↓
Analytics Enricher
      ↓
Country: India
Region: Maharashtra
Device: Mobile
Browser: Chrome
OS: Android
Bot: false
```

---

# 21. Analytics Aggregator

The Aggregator processes enriched events and calculates analytics.

## Metrics

- Total clicks
- Unique visitors
- Clicks per minute
- Clicks per hour
- Clicks per day
- Top referrers
- Countries
- Devices
- Browsers
- Operating systems

## Redis Counters

```text
clicks:Xy12Ab:2026-09-06

unique_ips:Xy12Ab:2026-09-06
```

Redis provides fast access to frequently requested analytics counters.

Periodic aggregation can persist durable analytics data into PostgreSQL.

---

# 22. Create Short URL Flow

```text
User
 │
 ▼
API Gateway
 │
 ▼
URL Generation Service
 │
 ├── Validate Long URL
 │
 ├── Authenticate User
 │
 ├── Get Local ID
 │       │
 │       └── ZooKeeper if range exhausted
 │
 ├── ID → Base62
 │
 ├── Store Mapping
 │       │
 │       ▼
 │   PostgreSQL
 │
 ├── Warm Redis
 │
 └──────────────► Return Short URL
```

---

# 23. Redirect Flow

```text
User
 │
 ▼
Load Balancer
 │
 ▼
Redirection Service
 │
 ▼
Redis
 │
 ├── Cache Hit ──────────────► HTTP 302
 │
 └── Cache Miss
          │
          ▼
      PostgreSQL
          │
          ▼
        Redis
          │
          ▼
       HTTP 302
```

At the same time:

```text
Redirection Service
       │
       └──────────► Kafka
                       │
                       ▼
                Analytics Service
```

---

# 24. Analytics Flow

```text
Redirection Service
        │
        ▼
      Kafka
        │
        ▼
   Click Events
        │
   ┌────┴─────┐
   ▼          ▼
Enricher   Aggregator
              │
              ├──► Redis
              │
              └──► PostgreSQL
```

---

# 25. URL Expiration

URLs can optionally have an expiration timestamp.

Example:

```text
short_code: Xy12Ab
expires_at: 2026-10-01T00:00:00Z
```

During redirection:

```text
Redis
  ↓
URL Metadata
  │
  ├── Valid ──────► HTTP 302
  │
  └── Expired ────► HTTP 404 / 410
```

Expired URLs can be removed asynchronously using a background cleanup process.

---

# 26. Horizontal Scalability

The system is designed to scale horizontally.

## URL Generation

```text
              ┌─ Instance 1
API Gateway ──┼─ Instance 2
              ├─ Instance 3
              └─ Instance N
```

Each instance generates IDs locally from its assigned ZooKeeper range.

## Redirection

```text
              ┌─ Instance 1
Load Balancer ┼─ Instance 2
              ├─ Instance 3
              └─ Instance N
```

Redirect services are stateless and can therefore be scaled independently.

## Analytics

Kafka consumers can scale independently.

```text
Kafka
 │
 ├── Consumer 1
 ├── Consumer 2
 ├── Consumer 3
 └── Consumer N
```

Kafka partitions allow multiple consumers to process events concurrently.

---

# 27. Why Separate Generate and Redirect Services?

URL creation and URL redirection have significantly different traffic characteristics.

## URL Generation

```text
- Lower traffic
- Write-heavy
- Database writes
- ID allocation
- Authentication
```

## Redirection

```text
- Extremely high traffic
- Read-heavy
- Ultra-low latency
- Redis-heavy
- Stateless
```

Separating these services allows each path to scale independently.

For example:

```text
URL Generation Service
        ↓
    3 instances

Redirection Service
        ↓
   50 instances
```

The actual number of instances depends on system traffic and infrastructure capacity.

---

# 28. Reliability

## Redis Failure

If Redis becomes unavailable, the Redirection Service can fall back to PostgreSQL.

```text
Redirection Service
        │
        ▼
      Redis
        │
        X
        │
        ▼
   PostgreSQL
```

This increases latency but allows URL resolution to continue.

## Kafka Failure

Redirect responses should not depend on analytics processing.

The Kafka producer should use retries and appropriate durability settings.

Analytics processing can resume once Kafka becomes available.

## Generate Instance Failure

If a URL-generation instance fails, its unused ID range may be lost.

This is acceptable because ID uniqueness is more important than perfect ID utilization.

---

# 29. Security

The system should implement:

- JWT authentication
- Secure password hashing
- API rate limiting
- HTTPS
- URL validation
- Malicious URL detection
- Input sanitization
- Request size limits
- Bot detection
- Authorization checks
- Audit logging

Users should only be allowed to manage URLs they own or are authorized to access.

---

# 30. Rate Limiting

Rate limiting is implemented at the API Gateway.

```text
User
 │
 ▼
API Gateway
 │
 ▼
Rate Limiter
 │
 ├── Allowed ───► Service
 │
 └── Rejected ──► HTTP 429
```

Different rate limits can be applied to:

- URL creation
- Login attempts
- Analytics APIs
- URL management APIs

Redirect traffic can have a separate high-throughput protection strategy.

---

# 31. Observability

The system should provide centralized observability.

## Metrics

Track:

- Request rate
- Redirect latency
- Cache hit ratio
- Database latency
- Kafka consumer lag
- URL creation rate
- Error rate
- Analytics processing latency

## Logging

Each request should have a unique correlation ID.

```text
Request
  │
  ├── API Gateway
  ├── Service
  ├── Redis
  ├── PostgreSQL
  └── Kafka
```

## Distributed Tracing

Distributed tracing can be used to trace requests across multiple services.

---

# 32. Key Design Decisions

| Component       | Technology         | Reason                                 |
| --------------- | ------------------ | -------------------------------------- |
| API Gateway     | API Gateway        | Authentication, routing, rate limiting |
| Authentication  | Auth Service       | Centralized authentication             |
| URL Generation  | Multiple instances | Horizontal scaling                     |
| ID Coordination | ZooKeeper          | Distributed ID allocation              |
| Short Code      | Base62             | Compact representation                 |
| Database        | PostgreSQL         | Durable source of truth                |
| Cache           | Redis              | Low-latency URL lookups                |
| Message Broker  | Kafka              | Asynchronous event processing          |
| Analytics       | Analytics Service  | Decoupled analytics processing         |
| Load Balancer   | Load Balancer      | Traffic distribution                   |

---

# 33. Important Trade-offs

## ZooKeeper vs Database Auto-Increment

### Database Auto-Increment

**Advantages**

- Simple
- Easy to implement
- Less infrastructure

**Disadvantages**

- Centralized ID generation
- Can become a bottleneck at very high scale

### ZooKeeper ID Blocks

**Advantages**

- Distributed ID allocation
- Local ID generation
- Reduced coordination traffic
- Better scalability

**Disadvantages**

- Additional infrastructure
- More operational complexity
- Unused IDs when instances fail

For a distributed-system learning project, ZooKeeper demonstrates an important coordination pattern.

---

# 34. Redis vs PostgreSQL for Redirects

Reading PostgreSQL for every redirect would create unnecessary database load.

Instead, the system uses:

```text
Redis
  ↓
PostgreSQL
```

Redis serves the majority of redirect requests.

PostgreSQL remains the durable source of truth.

---

# 35. Synchronous vs Asynchronous Analytics

## Synchronous Analytics

```text
Redirect
   │
   ▼
Analytics
   │
   ▼
HTTP Response
```

### Problems

- Increases redirect latency
- Couples redirect availability to analytics
- Analytics processing can become a bottleneck

## Asynchronous Analytics

```text
Redirect ─────────► User
   │
   ▼
 Kafka
   │
   ▼
Analytics Service
```

### Advantages

- Low redirect latency
- Independent scaling
- Better fault isolation
- Analytics can tolerate temporary delays

Therefore, analytics is intentionally asynchronous.

---

# 36. Performance Characteristics

The ideal redirect path is:

```text
User
 │
 ▼
Load Balancer
 │
 ▼
Redirection Service
 │
 ▼
Redis
 │
 ▼
HTTP 302
```

The ideal case requires:

- No PostgreSQL query
- No synchronous analytics processing
- Minimal application logic

This makes Redis a critical component for achieving low redirect latency.

---

# 37. Failure Scenarios

## Scenario 1 — Redis Unavailable

```text
Redirect Service
       │
       ▼
     Redis
       │
       X
       │
       ▼
 PostgreSQL
```

Result:

- Redirects continue
- Latency increases
- Cache can be rebuilt

---

## Scenario 2 — Analytics Service Unavailable

```text
Redirect
   │
   ├────────► User
   │
   └────────► Kafka
```

Result:

- Redirect remains available
- Analytics processing can catch up later

---

## Scenario 3 — URL Generation Instance Crashes

```text
Instance 1
IDs: 1M – 2M
   X
```

Result:

- Some IDs may remain unused
- Other instances continue generating URLs
- No duplicate IDs are generated

---

## Scenario 4 — PostgreSQL Temporarily Unavailable

Existing URLs in Redis may continue redirecting successfully.

However:

- New URL creation can fail
- Cache-miss redirects can fail
- URL management operations can be affected

---

# 38. End-to-End Architecture

```text
                         ┌──────────────────────┐
                         │        USERS         │
                         │   Web / Mobile       │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │     API GATEWAY      │
                         │ Auth • Validation    │
                         │ Rate Limiting        │
                         │ Routing              │
                         └───────┬───────┬──────┘
                                 │       │
                    ┌────────────┘       └─────────────┐
                    ▼                                  ▼
          ┌───────────────────┐              ┌───────────────────┐
          │    AUTH SERVICE   │              │ URL GENERATE SVC  │
          └───────────────────┘              └─────────┬─────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │   ZOOKEEPER     │
                                              │ Global Counter  │
                                              └────────┬────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │  ID → Base62    │
                                              │  Short Code     │
                                              └────────┬────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │   POSTGRESQL    │
                                              │ Durable Storage │
                                              └────────┬────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │      REDIS      │
                                              │  Hot URL Cache  │
                                              └─────────────────┘


Short URL Request
        │
        ▼
┌───────────────────┐
│   LOAD BALANCER   │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐
│ REDIRECTION SVC   │
└─────────┬─────────┘
          │
          ▼
       Redis
          │
     ┌────┴────┐
     │         │
    Hit       Miss
     │         │
     │         ▼
     │    PostgreSQL
     │         │
     │         ▼
     │       Redis
     │
     └──────────────► HTTP 302
                           │
                           ▼
                          USER

Redirection Service
        │
        ▼
      Kafka
        │
        ▼
Analytics Service
        │
   ┌────┴─────┐
   ▼          ▼
Enricher   Aggregator
              │
        ┌─────┴─────┐
        ▼           ▼
      Redis      PostgreSQL
```

---

# 39. System Design Principles Demonstrated

This project demonstrates the following system-design concepts:

1. **Horizontal Scaling**
2. **Load Balancing**
3. **Caching**
4. **Distributed ID Generation**
5. **Base62 Encoding**
6. **Asynchronous Processing**
7. **Event-Driven Architecture**
8. **Database Durability**
9. **Stateless Services**
10. **Service Separation**
11. **Fault Isolation**
12. **Rate Limiting**
13. **Authentication and Authorization**
14. **Observability**
15. **High-Availability Design**

---

# 40. Interview Explanation

A concise way to explain the architecture in a system-design interview:

> I would separate the URL creation path from the high-volume redirect path. URL generation uses ZooKeeper to allocate large ID ranges to individual service instances, allowing IDs to be generated locally and converted into Base62 short codes. PostgreSQL acts as the durable source of truth, while Redis caches frequently accessed URL mappings for low-latency redirects. The Redirection Service is stateless and horizontally scalable. Click events are published asynchronously to Kafka, allowing analytics to be processed independently without blocking the redirect response.

---

# 41. Final Architecture Summary

The system consists of:

```text
Clients
   │
   ▼
API Gateway
   │
   ├──────────────► Auth Service
   │
   └──────────────► URL Generation Service
                         │
                         ▼
                    ZooKeeper
                         │
                         ▼
                     Base62 ID
                         │
                         ▼
                    PostgreSQL
                         │
                         ▼
                       Redis


Short URL
   │
   ▼
Load Balancer
   │
   ▼
Redirection Service
   │
   ├──────────────► Redis ─────────► HTTP 302
   │
   └──────────────► Kafka
                         │
                         ▼
                  Analytics Service
                         │
                  ┌──────┴──────┐
                  ▼             ▼
               Redis        PostgreSQL
```

The core design principle is:

> **Keep the redirect path extremely fast and move non-critical work, such as analytics, to an asynchronous pipeline.**

---
