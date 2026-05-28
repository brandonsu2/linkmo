# LinkMo — Features, Requirements, Limitations & Constraints

> This document defines what LinkMo does (features), what it must do (requirements), what it intentionally does not do (limitations), and what bounds the system (constraints). Read this before writing any code.

---

## Table of Contents

1. [Functional Requirements](#1-functional-requirements)
2. [Feature Breakdown](#2-feature-breakdown)
3. [Non-Functional Requirements](#3-non-functional-requirements)
4. [Out of Scope (Intentional Limitations)](#4-out-of-scope-intentional-limitations)
5. [Technical Constraints](#5-technical-constraints)
6. [Business / Product Constraints](#6-business--product-constraints)
7. [Security Requirements](#7-security-requirements)
8. [Data Requirements](#8-data-requirements)
9. [Observability Requirements](#9-observability-requirements)

---

## 1. Functional Requirements

Functional requirements define the *behaviors* the system must exhibit. Each is labeled with a priority: **P0** (must have at launch), **P1** (important, build in Phase 2–4), **P2** (nice to have, Phase 5+).

### User Management

| ID | Requirement | Priority |
|----|-------------|----------|
| U-1 | Users can register with an email address and password | P0 |
| U-2 | Passwords must be stored as bcrypt hashes (never plaintext) | P0 |
| U-3 | Users can log in and receive a JWT valid for 24 hours | P0 |
| U-4 | JWTs must be signed with a secret stored in environment variables, not hardcoded | P0 |
| U-5 | Users can only view and manage their own links | P0 |
| U-6 | Users can log out (client-side token deletion; no server-side session) | P1 |

### URL Shortening

| ID | Requirement | Priority |
|----|-------------|----------|
| L-1 | Authenticated users can submit a long URL and receive a unique short code | P0 |
| L-2 | Short codes must be globally unique across all users | P0 |
| L-3 | Short codes must be URL-safe (alphanumeric only, no special characters) | P0 |
| L-4 | Short codes must be 6–8 characters long | P0 |
| L-5 | The system must validate that submitted URLs are well-formed (valid scheme, host) | P0 |
| L-6 | Users can view a list of all their short links | P0 |
| L-7 | Users can delete their own short links | P1 |
| L-8 | Users can set an expiry date/time on a link (link stops redirecting after expiry) | P1 |
| L-9 | Custom short codes: users can request a specific short code instead of a generated one | P2 |
| L-10 | Users can create links with a title/description for organization | P2 |

### Redirect Behavior

| ID | Requirement | Priority |
|----|-------------|----------|
| R-1 | `GET /{shortCode}` must redirect to the long URL using HTTP 302 | P0 |
| R-2 | Redirect must respond in under 50ms at p99 under normal load | P0 |
| R-3 | If a short code does not exist, return a 404 page | P0 |
| R-4 | If a link is expired, return a 410 Gone response | P1 |
| R-5 | If a link is deleted/inactive, return a 404 | P0 |
| R-6 | Bots and crawlers (identified by User-Agent) should not be counted as clicks | P2 |

### Click Analytics

| ID | Requirement | Priority |
|----|-------------|----------|
| A-1 | Every non-bot click on a short link is recorded as a click event | P0 |
| A-2 | Each click event captures: short_code, timestamp, IP, User-Agent, Referer header | P0 |
| A-3 | Click events are enriched with geo data (country, city, lat/lon) | P1 |
| A-4 | Click events are enriched with device metadata (mobile/tablet/desktop, browser, OS) | P1 |
| A-5 | Analytics must be available to query within 30 seconds of a click occurring | P1 |
| A-6 | Users can query total clicks for a link over a time range | P0 |
| A-7 | Users can query click breakdown by country | P1 |
| A-8 | Users can query click breakdown by device type | P1 |
| A-9 | Users can query click breakdown by referrer | P1 |
| A-10 | Users can query click time series (clicks per hour/day) | P1 |
| A-11 | Raw click records are paginated (max 100 per page) | P1 |

---

## 2. Feature Breakdown

### Core Feature: URL Shortening

The user submits a URL via `POST /api/links`. The system:
1. Validates the URL is well-formed using Node's `URL` constructor
2. Generates a short code using Base62 encoding or random generation with collision check
3. Inserts a row into the `links` table in PostgreSQL
4. Returns the short code to the user

The user then shares the short link (`https://yourdomain.com/{shortCode}`).

**Edge cases to handle:**
- Duplicate URL submission: return the same short code if the user already shortened this URL (optional deduplication)
- Extremely long URLs (> 2048 chars): reject with 400
- Non-HTTP/HTTPS URLs: reject with 400
- Localhost/private IP URLs: optionally block to prevent SSRF

### Core Feature: Redirect with Caching

`GET /{shortCode}` is the highest-volume endpoint. The URL Service:
1. Checks Redis: `GET url:{shortCode}`
2. If cache hit: return the URL immediately
3. If cache miss: query PostgreSQL, write to Redis with 1-hour TTL, return the URL
4. Issues HTTP 302 redirect

On cache hit, this is essentially just a Redis lookup — single-digit millisecond latency.

### Core Feature: Real-Time Analytics Pipeline

After a redirect, a click event is published to Kafka asynchronously. This is **fire-and-forget** from the redirect path's perspective — the user's browser is already being redirected before the event is even processed.

The Analytics Consumer reads from Kafka and writes to DynamoDB. Because DynamoDB is purpose-built for high-write workloads, it handles any volume without schema migrations or tuning.

The query API aggregates DynamoDB records on read and returns summaries. For high-traffic links, consider pre-aggregating with DynamoDB Streams + Lambda (out of scope for MVP).

### Feature: Rate Limiting

Without rate limiting, a malicious actor can:
- Flood your redirect endpoint (DDoS)
- Spam link creation to fill your database
- Enumerate all short codes to discover others' links

Rate limits:
- `POST /api/links`: 20 requests/minute per authenticated user
- `GET /{shortCode}`: 1000 requests/minute per IP
- `POST /api/auth/*`: 5 requests/minute per IP (brute-force protection)

Counters are stored in Redis with TTL so they expire automatically.

### Feature: Authentication

JWT-based stateless authentication. No server-side sessions. This means the API Gateway can validate tokens without querying a database on every request — it just verifies the JWT signature using the shared secret.

```
Login flow:
  1. POST /api/auth/login { email, password }
  2. Gateway queries URL Service for user record
  3. bcrypt.compare(plaintext, hash)
  4. If match: sign JWT { userId, email, iat, exp }
  5. Return JWT to client
  6. Client stores token (localStorage or memory) and sends in Authorization header
```

**JWT payload:**
```json
{
  "userId": 42,
  "email": "user@example.com",
  "iat": 1705350000,
  "exp": 1705436400
}
```

---

## 3. Non-Functional Requirements

Non-functional requirements define the *quality attributes* of the system — how well it does things rather than what it does.

### Performance

| Metric | Target | Rationale |
|--------|--------|-----------|
| Redirect p99 latency | < 50ms | Users feel delays > 100ms. Redirects should be imperceptible. |
| Link creation p99 latency | < 200ms | Not in the hot path; users tolerate slightly more wait |
| Analytics query p99 | < 500ms | Dashboard load; acceptable with spinner |
| Click event propagation delay | < 30s | Near-real-time analytics; Kafka + consumer should process within seconds |

### Availability

| Metric | Target |
|--------|--------|
| Redirect endpoint uptime | 99.9% (< 8.7 hrs downtime/year) |
| Link creation uptime | 99.5% |
| Analytics uptime | 99.0% |

The redirect path is most availability-critical. Multi-replica deployments on Kubernetes + Redis caching mean even a PostgreSQL outage won't break redirects for cached links.

### Scalability

The system is designed for horizontal scaling:
- **API Gateway**: Add replicas; Redis-backed rate limiting keeps limits consistent
- **URL Service**: Add replicas; stateless gRPC server
- **Ingest Service**: Add replicas; Kafka partitions allow parallel producers
- **Analytics Consumer**: Add replicas (Kafka consumer group distributes partitions)

### Reliability

- Kafka ensures click events are not lost even if the Analytics Consumer is temporarily down
- Redis cache means a PostgreSQL blip does not break redirects for cached links
- Kubernetes restarts crashed pods automatically
- DynamoDB is inherently highly available (multi-AZ by default)

### Maintainability

- Each service is independently deployable — you can update the Analytics Service without touching the Gateway
- Terraform state is stored in S3 (remote state) so multiple developers/machines can run Terraform
- All configuration in environment variables or Kubernetes ConfigMaps — no hardcoded config

---

## 4. Out of Scope (Intentional Limitations)

These features are *deliberately excluded* from this project. Many are real features you'd build in production, but they would distract from the learning goals at this stage.

### Excluded Features

| Feature | Why Excluded |
|---------|-------------|
| **Frontend UI / Dashboard** | Building a React SPA is a separate skill set; a REST API is sufficient for demonstrating the backend architecture |
| **Email verification** | Adds SMTP/SendGrid complexity without teaching new distributed systems concepts |
| **Password reset flow** | Same as above |
| **Custom domains** (e.g., `yourcompany.com/abc123`) | Requires DNS management, TLS certificate provisioning per domain — significant infrastructure complexity |
| **QR code generation** | Trivial feature; adds no learning value |
| **Link analytics aggregation at write time** | Pre-aggregating (e.g., increment counters on each click) is a useful optimization but complicates the Kafka consumer significantly |
| **Multi-region deployment** | Global anycast routing, cross-region replication — too advanced for a learning project |
| **Webhooks / Link event callbacks** | Useful in production but adds complexity without introducing new concepts |
| **Social preview metadata (og:image, etc.)** | Requires fetching the destination URL at creation time — SSRF risk, added latency |
| **A/B testing with links** | One short code routing to different destinations — interesting but not core |
| **Team / organization accounts** | Multi-tenancy adds authorization complexity |

### Known Simplifications

- **Bot detection** is heuristic-only (User-Agent check). Production systems use more sophisticated signals.
- **Geo accuracy** using GeoLite2 is ~80% accurate at city level. Production uses commercial databases (MaxMind GeoIP2, ipstack).
- **IP privacy**: This system stores raw IP addresses. GDPR-compliant systems should hash or truncate IPs.
- **Analytics consistency**: DynamoDB reads after writes may not immediately reflect the latest data (eventual consistency). This is acceptable for analytics.

---

## 5. Technical Constraints

These are constraints imposed by the technology choices, not business decisions.

### PostgreSQL (RDS)

- **Max connections**: RDS `db.t3.micro` allows ~85 simultaneous connections. The URL Service must use connection pooling (`pg-pool`) to avoid exhausting connections under load. Pool size should be `max: 10` per service replica.
- **Storage**: RDS volumes scale automatically but have a cost. For this project, start with 20GB.
- **Single-AZ for dev**: Multi-AZ doubles the cost. Use single-AZ while learning, switch to Multi-AZ only if simulating production.

### Redis (ElastiCache)

- **Memory limit**: `cache.t3.micro` has 0.5GB RAM. If your Redis eviction policy is `allkeys-lru`, this is fine — it evicts the least recently used keys when full.
- **No persistence by default**: ElastiCache Redis (in cluster mode) does not persist to disk by default. If it restarts, your rate limit counters and URL cache are cleared. This is acceptable — the cache rebuilds itself and rate limit counters reset.
- **Connection limit**: ~65,000 connections. Not a concern at this scale.

### Kafka (MSK / local)

- **Minimum 2 brokers in MSK**: AWS MSK requires at least 2 brokers (~$0.21/hr each). This is expensive for a learning project. Use Confluent Cloud's free tier (5GB/month) for the Kafka layer and skip MSK until the final deployment test.
- **Topic creation**: Topics should be pre-created with correct partition counts. Changing partition counts later requires data migration.
- **Message size**: Default max message size is 1MB. Click event JSON is ~500 bytes. Not a concern.

### DynamoDB

- **On-demand billing**: Pay per read/write request. Very cheap at low volume (~$0.25 per million read units, $1.25 per million write units). No idle cost.
- **Item size limit**: 400KB per item. Click events are ~1KB. Not a concern.
- **GSI eventual consistency**: GSIs are updated asynchronously. Queries on a GSI may not immediately reflect the latest writes (typically < 1 second behind).
- **No JOIN queries**: DynamoDB is not relational. Cross-table queries (e.g., "give me all clicks for links created by user X") require multiple requests or denormalization. Design your access patterns upfront.

### Kubernetes (EKS)

- **EKS control plane cost**: $0.10/hr (~$72/month). Always running even when pods are scaled to 0. Use `terraform destroy` when not actively using.
- **Node pool**: Start with 2× `t3.small` spot instances to minimize cost. Spot instances can be interrupted — acceptable for a learning project.
- **Namespace isolation**: Use a `linkmo` namespace to keep resources organized and prevent accidental cross-service interference.

### gRPC

- **HTTP/2 only**: gRPC requires HTTP/2. Nginx/ALB must be configured to pass through HTTP/2 (not terminate it) for gRPC to work with external load balancers. For internal service-to-service calls (ClusterIP), this is not an issue.
- **Proto schema is a contract**: Changing field numbers in a `.proto` file is a breaking change. Treat proto changes like API version changes.
- **Code generation**: Both services must regenerate stubs whenever the `.proto` changes. Automate this in your CI pipeline.

---

## 6. Business / Product Constraints

### Cost Budget

- **Local development**: ~$0/month (Docker Compose, no cloud resources)
- **AWS dev environment (active)**: ~$15–30/day
- **AWS dev environment (torn down)**: ~$1/month (S3 for Terraform state, Route53 for domain if any)
- **Recommendation**: Run `terraform destroy` at the end of each learning session. Budget $50–100 total for this project.

### Learning Goal Priority

This is a learning project, not a production system. When faced with a trade-off between "correct for production" and "teaches the most", choose the latter. Examples:

- Run a monolith first (Phase 1) even though you'll throw it away — it clarifies the domain model
- Use a single Kafka broker locally even though production uses 3
- Store analytics in DynamoDB even though at this scale PostgreSQL would work fine — the point is to learn DynamoDB

### Time Budget

The project is designed to be completable in 9 weeks of part-time work (~10 hrs/week). Individual phases can be paused and resumed.

---

## 7. Security Requirements

### Authentication & Authorization

- Passwords must use bcrypt with a cost factor of at least 12
- JWTs must expire (24-hour max)
- JWT secret must be ≥ 32 random bytes, stored in Kubernetes Secrets / AWS Secrets Manager
- All endpoints except `GET /{shortCode}` and `POST /auth/*` require a valid JWT
- Users must not be able to access or modify other users' links (enforce `WHERE user_id = ?` on all queries)

### Input Validation

- All user inputs must be validated server-side (never trust client validation alone)
- URL inputs must be validated with Node's `URL` constructor; reject URLs with `javascript:` scheme or private IP destinations (SSRF prevention)
- Short code inputs must match `/^[a-zA-Z0-9]{1,10}$/` before any database lookup

### Network Security

- Services communicate within a private VPC; only the API Gateway is exposed to the internet
- All internal gRPC calls are unencrypted (mTLS is out of scope, but note this in documentation)
- Use Kubernetes NetworkPolicy to restrict which pods can communicate with which
- Redis and PostgreSQL are in private subnets with no internet access

### Secrets Management

- No secrets in source code or Docker images
- All secrets injected via environment variables from Kubernetes Secrets
- Kubernetes Secrets backed by AWS Secrets Manager (via External Secrets Operator) in production
- Rotate JWT secrets periodically (at least once before considering this "production-ready")

---

## 8. Data Requirements

### Data Retention

| Data | Retention Policy | Rationale |
|------|-----------------|-----------|
| User accounts | Until account deleted | Standard |
| Short link records | Until explicitly deleted by user or admin | Links must remain resolvable |
| Click events (DynamoDB) | 90 days | Balance between analytics value and storage cost |
| Kafka messages | 7 days | Allows consumer lag recovery up to 1 week |
| Redis cache | TTL-based (1hr for URLs, 60s for rate limits) | Auto-expires; no manual cleanup needed |

### Data Privacy

- IP addresses are personal data under GDPR. For a learning project, storing raw IPs is acceptable. In production, you would hash or truncate IPs after geo enrichment.
- Do not log passwords or JWTs anywhere in application logs
- User email addresses should not appear in Kafka messages or DynamoDB records (use `user_id` foreign key instead)

### Data Consistency Model

| Store | Consistency Model | Implication |
|-------|------------------|-------------|
| PostgreSQL | Strong (ACID) | Link creation is immediately consistent |
| Redis | Eventual (cache) | Cache may be stale for up to the TTL after a link is deleted |
| Kafka → DynamoDB | Eventually consistent | Clicks appear in analytics within ~10–30 seconds |

**Important**: When a user deletes a link, you must also `DEL url:{shortCode}` from Redis immediately. Otherwise the cache will serve the old redirect for up to 1 hour after deletion.

---

## 9. Observability Requirements

A system you cannot observe is a system you cannot debug. Even for a learning project, implement basic observability from day one.

### Logging

- Each service must emit structured JSON logs (not plain text strings)
- Every log entry must include: `service`, `level`, `message`, `timestamp`, `traceId` (if available)
- Node.js: use `pino` (fast, JSON by default)
- Python: use `structlog`
- Log levels: `DEBUG` in development, `INFO` in production

### Metrics

- Each service should expose a `/metrics` endpoint in Prometheus format
- Node.js: use `prom-client`
- Python: use `prometheus-client`
- Key metrics to track:
  - Request rate, error rate, latency (the RED method: Rate, Errors, Duration)
  - Redis cache hit/miss ratio
  - Kafka consumer lag (for the Analytics Service)
  - DynamoDB read/write latency

### Tracing

Distributed tracing lets you follow a single request across all services. This is how you debug "the redirect is slow sometimes."

- Use OpenTelemetry (vendor-neutral) with AWS X-Ray as the backend
- Each request gets a `traceId` at the Gateway; this ID is propagated in gRPC metadata and Kafka message headers
- Low priority for MVP but extremely valuable when debugging Phase 3+ issues

### Health Checks

Every service must implement:
- `GET /health/live` — returns 200 if the process is running (liveness probe)
- `GET /health/ready` — returns 200 only if the service can handle requests (readiness probe): checks DB connection, Redis connection, Kafka producer connectivity

Kubernetes uses these to decide whether to route traffic to a pod.
