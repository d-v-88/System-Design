The Final Design — TinyURL

1. Users / Clients

Web / Mobile Clients

Users can:

Create short URLs
Redirect using short URLs
View analytics
Manage their URLs 2. API Gateway

The API Gateway acts as the single entry point for client requests.

Responsibilities:

Authentication / JWT validation
Request validation
Rate limiting
Routing requests to appropriate services
Basic security checks

Create flow:

User
↓
API Gateway
↓
URL Generate Service

Analytics flow:

User
↓
API Gateway
↓
Analytics Service 3. Auth Service

Handles user authentication and authorization.

Responsibilities:

Sign up
Login
JWT access tokens
Refresh tokens
Role-based authorization
User management
Client
↓
API Gateway
↓
Auth Service

The generated JWT is then used when accessing protected APIs.

4. URL Generation Service

This is the write path of the system.

Run multiple instances behind a load balancer:

              ┌─ Generate Instance 1

API Gateway ──┼─ Generate Instance 2
└─ Generate Instance 3

Responsibilities:

Validate long URL
Authenticate user
Request an ID range from ZooKeeper when local range is exhausted
Generate numeric ID
Convert ID → Base62
Create short code
Store mapping in database
Warm Redis
Return TinyURL
Example
Long URL
↓
Numeric ID: 12584932
↓
Base62 encoding
↓
Xy12Ab
↓
https://t.example.com/r/Xy12Ab 5. ZooKeeper — ID Coordination

ZooKeeper is responsible for distributed ID allocation.

Instead of every URL-generation request asking ZooKeeper for an ID:

Generate Service
↓
ZooKeeper
↓
1 ID

you allocate a large block:

Generate Instance 1 → IDs 1,000,000–1,999,999
Generate Instance 2 → IDs 2,000,000–2,999,999
Generate Instance 3 → IDs 3,000,000–3,999,999

Each instance then generates IDs locally.

ZooKeeper
Global Atomic Counter
↓
Allocate ID Range
↓
URL Generate Instance
↓
Local ID Generation

This dramatically reduces coordination traffic.

6. Database

Use PostgreSQL as the durable source of truth.

Stores:

Users
URLs
URL metadata
Expiration
Ownership
Tags
Analytics aggregates

Main mapping:

shortCode → longUrl

Example:

Xy12Ab → https://example.com/product/123

Database is primarily used on the write path and cache-miss path, rather than every redirect.

7. Redis

Redis is the hot-path cache.

code:Xy12Ab
↓
https://example.com/product/123
Redirect flow
Short URL
↓
Redis
↓
Cache Hit
↓
Original URL

This avoids hitting PostgreSQL for the majority of redirects.

Redis can also maintain:

clicks:Xy12Ab:2026-09-06
unique_ips:Xy12Ab:2026-09-06

for fast analytics counters.

8. Redirection Service

This is the ultra-low-latency read path.

Run multiple instances behind a load balancer:

                 ┌─ Redirect Instance 1

Load Balancer ───┼─ Redirect Instance 2
└─ Redirect Instance 3
Flow
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
Cache miss
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
HTTP 302 9. Kafka — Click Event Pipeline

The redirect service should not perform analytics synchronously.

Instead:

Redirection Service
│
├──────────────→ User: HTTP 302
│
└──────────────→ Kafka

This keeps analytics completely decoupled from the redirect path.

A click event might contain:

eventId
shortCode
timestamp
IP
userAgent
referrer
country

The redirect request doesn't wait for analytics processing.

10. Analytics Service

Kafka consumers process click events asynchronously.

Architecture:

                  Kafka
                    │
                    ↓
              Click Events
                    │
          ┌─────────┴─────────┐
          ↓                   ↓
      Enricher            Aggregator
          ↓                   ↓

Geo / Device Redis Counters
↓
↓
PostgreSQL
Enricher

Adds:

Country
Region
Device
Browser
OS
Bot classification
Aggregator

Calculates:

Total clicks
Unique visitors
Clicks per minute/hour/day
Top referrers
Countries
Devices

Then periodically persists durable aggregates to PostgreSQL.

11. Complete Architecture

Your diagram should essentially communicate this:

                         ┌──────────────────────┐
                         │        USERS         │
                         │   Web / Mobile       │
                         └──────────┬───────────┘
                                    │
                                    ↓
                         ┌──────────────────────┐
                         │     API GATEWAY      │
                         │ Auth • Validation    │
                         │ Rate Limiting        │
                         └───────┬───────┬──────┘
                                 │       │
                    ┌────────────┘       └─────────────┐
                    ↓                                  ↓
          ┌───────────────────┐              ┌───────────────────┐
          │    AUTH SERVICE   │              │ URL GENERATE SVC  │
          └───────────────────┘              └─────────┬─────────┘
                                                       │
                                                       ↓
                                              ┌─────────────────┐
                                              │   ZOOKEEPER     │
                                              │ Global Counter  │
                                              └─────────────────┘
                                                       │
                                                       ↓
                                              ID → Base62 → Code
                                                       │
                                                       ↓
                                              ┌─────────────────┐
                                              │    DATABASE     │
                                              │   PostgreSQL    │
                                              └────────┬────────┘
                                                       │
                                                       ↓
                                              ┌─────────────────┐
                                              │      REDIS      │
                                              │  Hot URL Cache  │
                                              └─────────────────┘

User clicks short URL
│
↓
┌───────────────────┐
│ LOAD BALANCER │
└─────────┬─────────┘
↓
┌───────────────────┐
│ REDIRECTION SVC │
└─────────┬─────────┘
│
↓
Redis
│
Cache Hit
│
├────────────────────→ HTTP 302 → User
│
↓
Kafka
│
↓
┌───────────────────┐
│ ANALYTICS SERVICE │
└─────────┬─────────┘
│
┌────┴─────┐
↓ ↓
Enricher Aggregator
│
┌─────┴─────┐
↓ ↓
Redis PostgreSQL

## Architecture Overview

![System Design](./architecture.svg)
