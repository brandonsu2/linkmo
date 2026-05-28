# LinkMo — Architecture & Implementation Guide

> A distributed URL shortener and real-time analytics platform. This document covers the system architecture, service breakdown, data models, inter-service communication patterns, infrastructure, and every library/component you'll use across the codebase.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [High-Level Architecture Diagram](#2-high-level-architecture-diagram)
3. [Service 1: API Gateway](#3-service-1-api-gateway)
4. [Service 2: URL Service](#4-service-2-url-service)
5. [Service 3: Click Ingestion Service](#5-service-3-click-ingestion-service)
6. [Service 4: Analytics Service](#6-service-4-analytics-service)
7. [Inter-Service Communication](#7-inter-service-communication)
8. [Data Models](#8-data-models)
9. [Infrastructure (Terraform + AWS)](#9-infrastructure-terraform--aws)
10. [Kubernetes Deployment](#10-kubernetes-deployment)
11. [Local Development Stack](#11-local-development-stack)
12. [Component & Library Reference](#12-component--library-reference)

---

## 1. System Overview

LinkMo is a microservices-based URL shortener with real-time click analytics. A user submits a long URL and receives a short code (e.g., `lnk.io/aB3x`). Every time that short link is visited, a click event flows asynchronously through the system and lands in a queryable analytics store.

The system is intentionally over-engineered relative to its simple problem domain. That is the point. Every technology introduced has a *natural, justified reason to exist* in the architecture, making it excellent for learning distributed systems concepts without feeling contrived.

**What happens when a short link is clicked:**

```
User Browser
  → DNS resolves lnk.io to API Gateway Load Balancer
    → API Gateway (Node.js) validates request, routes to URL Service via gRPC
      → URL Service checks Redis cache
          Cache hit  → return long URL immediately (sub-5ms)
          Cache miss → query PostgreSQL → write back to Redis → return long URL
      → API Gateway issues HTTP 301/302 redirect to browser
      → API Gateway (or Ingest Service) publishes click event to Kafka topic
        → Analytics Consumer reads from Kafka
          → Writes click record to DynamoDB
```

This flow achieves two critical properties: **the redirect is fast** (Redis cache avoids a DB query on hot links), and **the analytics write is decoupled** (Kafka ensures clicks are never lost even if the Analytics Service is down).

---

## 2. High-Level Architecture Diagram

```
                        ┌─────────────────────────────────────────────┐
                        │              Client (Browser / API)           │
                        └────────────────────┬────────────────────────┘
                                             │ HTTPS
                                             ▼
                        ┌─────────────────────────────────────────────┐
                        │           API Gateway (Node.js/Express)       │
                        │  • JWT Authentication                         │
                        │  • Rate Limiting (Redis)                      │
                        │  • Request Routing                            │
                        │  • gRPC Client                                │
                        └──────┬──────────────────────┬───────────────┘
                               │ gRPC                 │ HTTP POST (click event)
                               ▼                      ▼
          ┌──────────────────────────┐   ┌──────────────────────────────┐
          │  URL Service             │   │  Click Ingest Service         │
          │  (Node.js + PostgreSQL)  │   │  (Python/Flask)               │
          │  • Create short URLs     │   │  • Kafka Producer             │
          │  • Resolve short → long  │   │  • Enriches click metadata    │
          │  • Redis cache-aside     │   │    (geo, device, referrer)    │
          └──────┬───────────────────┘   └──────────────┬───────────────┘
                 │                                       │
        ┌────────┴────────┐                   ┌─────────┴──────────┐
        │   PostgreSQL    │  ◄──── miss ────   │   Kafka Topic:      │
        │   (RDS)         │  cache-aside        │   click.events      │
        └─────────────────┘                    └─────────┬──────────┘
                 │                                       │
        ┌────────┴────────┐                   ┌─────────▼──────────┐
        │   Redis          │                  │  Analytics Service   │
        │   (ElastiCache)  │                  │  (Python)            │
        └─────────────────┘                   │  • Kafka Consumer    │
                                              │  • DynamoDB Writer   │
                                              │  • REST query API    │
                                              └─────────┬──────────┘
                                                        │
                                              ┌─────────▼──────────┐
                                              │    DynamoDB          │
                                              │    (click events)    │
                                              └────────────────────┘
```

---

## 3. Service 1: API Gateway

**Language:** Node.js  
**Framework:** Express.js  
**Role:** Single entry point for all external traffic. Handles authentication, rate limiting, and routes requests to downstream services.

### Responsibilities

- Authenticate all incoming requests using JWT (JSON Web Tokens)
- Enforce per-user and per-IP rate limits using Redis
- Proxy URL creation requests to the URL Service via gRPC
- Proxy redirect requests (e.g., `GET /:shortCode`) to the URL Service via gRPC, then issue HTTP redirect
- Forward click metadata to the Click Ingest Service asynchronously after each redirect

### API Endpoints Exposed

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/auth/register` | Create a user account |
| `POST` | `/api/auth/login` | Get a JWT |
| `POST` | `/api/links` | Create a short link (auth required) |
| `GET` | `/api/links` | List user's links with stats (auth required) |
| `DELETE` | `/api/links/:shortCode` | Delete a link (auth required) |
| `GET` | `/:shortCode` | Redirect (public) |
| `GET` | `/api/analytics/:shortCode` | Get click analytics (auth required) |

### Key Implementation Details

**JWT Authentication Middleware**

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');

function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token' });
  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch (err) {
    res.status(401).json({ error: 'Invalid token' });
  }
}
```

**Rate Limiting with Redis**

Use the `rate-limiter-flexible` package, which stores rate limit counters in Redis. This means rate limits are shared across all Gateway replicas — a user hitting replica A cannot bypass the limit by their next request landing on replica B.

```javascript
const { RateLimiterRedis } = require('rate-limiter-flexible');
const limiter = new RateLimiterRedis({
  storeClient: redisClient,
  keyPrefix: 'rl',
  points: 100,       // 100 requests
  duration: 60,      // per 60 seconds
});
```

**gRPC Client Setup**

The gateway calls the URL Service over gRPC using a `.proto` definition. The gateway is the *client*; the URL Service is the *server*.

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const def = protoLoader.loadSync('protos/url.proto');
const proto = grpc.loadPackageDefinition(def);
const urlClient = new proto.url.UrlService(
  process.env.URL_SERVICE_ADDR,
  grpc.credentials.createInsecure()
);
```

### Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `express` | ^4.18 | HTTP server and routing |
| `jsonwebtoken` | ^9.0 | JWT sign and verify |
| `bcryptjs` | ^2.4 | Password hashing for user accounts |
| `@grpc/grpc-js` | ^1.9 | gRPC client (pure JS, no native bindings) |
| `@grpc/proto-loader` | ^0.7 | Load `.proto` files at runtime |
| `rate-limiter-flexible` | ^3.0 | Redis-backed rate limiting |
| `ioredis` | ^5.3 | Redis client (supports clusters, promises) |
| `dotenv` | ^16.0 | Load env vars from `.env` file |
| `helmet` | ^7.0 | HTTP security headers |
| `morgan` | ^1.10 | Request logging |
| `cors` | ^2.8 | CORS headers |

---

## 4. Service 2: URL Service

**Language:** Node.js  
**Framework:** Express.js (internal HTTP) + gRPC server  
**Database:** PostgreSQL (via AWS RDS in production)  
**Cache:** Redis (via AWS ElastiCache in production)  
**Role:** Owns all URL data. Creates short codes, stores mappings, resolves redirects.

### Responsibilities

- Accept gRPC calls from the API Gateway
- Generate unique short codes for submitted URLs
- Store `short_code → long_url` mappings in PostgreSQL
- Serve redirects via a **cache-aside** (lazy-loading) pattern using Redis
- Return link metadata (owner, creation date, click count summary) for listing

### Short Code Generation

Short codes are 6–8 character alphanumeric strings. Two strategies:

**Option A — Random + collision check (simpler to implement first):**
```javascript
const BASE62 = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
function generateCode(length = 7) {
  return Array.from({ length }, () =>
    BASE62[Math.floor(Math.random() * 62)]
  ).join('');
}
// Try up to 5 times, check DB for uniqueness each time
```

**Option B — Base62 encode a DB sequence (no collisions, scales better):**
PostgreSQL `SEQUENCE` generates monotonically increasing integers. Encode the integer in base-62. No uniqueness checks needed. Better for production.

### Redis Cache-Aside Pattern

```
resolve(shortCode):
  1. Check Redis: GET shortCode
  2. If hit → return value (done, no DB query)
  3. If miss → query PostgreSQL
  4. If found → SET shortCode longUrl EX 3600  (write back with 1hr TTL)
  5. Return longUrl
```

This is called **cache-aside** or **lazy loading** — the cache is populated on demand, not upfront. Hot links (frequently clicked) will almost always be in cache.

### PostgreSQL Schema

```sql
CREATE TABLE users (
  id          SERIAL PRIMARY KEY,
  email       VARCHAR(255) UNIQUE NOT NULL,
  password    VARCHAR(255) NOT NULL,         -- bcrypt hash
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE links (
  id          SERIAL PRIMARY KEY,
  short_code  VARCHAR(10) UNIQUE NOT NULL,
  long_url    TEXT NOT NULL,
  user_id     INTEGER REFERENCES users(id) ON DELETE CASCADE,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  expires_at  TIMESTAMPTZ,                   -- nullable, for link expiry feature
  is_active   BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_links_short_code ON links(short_code);
CREATE INDEX idx_links_user_id ON links(user_id);
```

The index on `short_code` is critical — every redirect hits this column. Without it, PostgreSQL would do a full table scan on every click.

### gRPC Server

The URL Service exposes a gRPC server. The `.proto` definition (shared between gateway and service):

```protobuf
// protos/url.proto
syntax = "proto3";
package url;

service UrlService {
  rpc CreateLink (CreateLinkRequest) returns (CreateLinkResponse);
  rpc ResolveLink (ResolveLinkRequest) returns (ResolveLinkResponse);
  rpc DeleteLink (DeleteLinkRequest) returns (DeleteLinkResponse);
  rpc GetUserLinks (GetUserLinksRequest) returns (GetUserLinksResponse);
}

message CreateLinkRequest {
  string long_url = 1;
  int32 user_id = 2;
  int64 expires_at = 3;  // unix timestamp, 0 = no expiry
}

message CreateLinkResponse {
  string short_code = 1;
}

message ResolveLinkRequest {
  string short_code = 1;
}

message ResolveLinkResponse {
  string long_url = 1;
  bool found = 2;
}
```

### Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `@grpc/grpc-js` | ^1.9 | gRPC server |
| `@grpc/proto-loader` | ^0.7 | Load proto definitions |
| `pg` | ^8.11 | PostgreSQL client |
| `pg-pool` | bundled with `pg` | Connection pooling (reuse DB connections) |
| `ioredis` | ^5.3 | Redis client |
| `dotenv` | ^16.0 | Environment variables |

---

## 5. Service 3: Click Ingestion Service

**Language:** Python  
**Framework:** Flask  
**Role:** Receives click events, enriches them with metadata (geo, device, referrer), and publishes them to a Kafka topic.

### Why Python Here?

The API Gateway *could* publish to Kafka directly, but separating ingestion into its own service teaches you:
- How to run a heterogeneous tech stack (Node + Python together)
- How to reason about service boundaries
- How Kafka decouples producers from consumers

In practice, you could also make the Gateway a Kafka producer directly if simplicity matters more.

### What "Enriching" a Click Event Means

When a user clicks `lnk.io/aB3x`, the API Gateway knows:
- The `short_code` (`aB3x`)
- The client IP address
- The `User-Agent` header
- The `Referer` header
- The timestamp

The Ingest Service augments this with:
- **Geolocation** — look up the IP to get country/city using the MaxMind GeoLite2 database
- **Device type** — parse the User-Agent to classify as mobile/tablet/desktop
- **Browser/OS** — parse User-Agent further for browser name and OS

```python
# services/ingest/enricher.py
import geoip2.database
from user_agents import parse as ua_parse

reader = geoip2.database.Reader('/data/GeoLite2-City.mmdb')

def enrich(event: dict) -> dict:
    # Geo lookup
    try:
        geo = reader.city(event['ip'])
        event['country'] = geo.country.iso_code
        event['city'] = geo.city.name
        event['lat'] = geo.location.latitude
        event['lon'] = geo.location.longitude
    except Exception:
        event['country'] = 'XX'

    # User agent parsing
    ua = ua_parse(event.get('user_agent', ''))
    event['device_type'] = 'mobile' if ua.is_mobile else ('tablet' if ua.is_tablet else 'desktop')
    event['browser'] = ua.browser.family
    event['os'] = ua.os.family

    return event
```

### Kafka Producer

```python
# services/ingest/producer.py
from confluent_kafka import Producer
import json

producer = Producer({'bootstrap.servers': os.getenv('KAFKA_BROKERS')})

def publish_click(event: dict):
    producer.produce(
        topic='click.events',
        key=event['short_code'].encode(),  # partition by short_code for ordering
        value=json.dumps(event).encode(),
        callback=delivery_report
    )
    producer.flush()

def delivery_report(err, msg):
    if err:
        logging.error(f'Kafka delivery failed: {err}')
```

**Why key by `short_code`?** Kafka partitions messages by key. Keying by `short_code` ensures all clicks for the same link land on the same partition in the same order. This matters if you later want to do stream processing that aggregates by link.

### Flask Endpoint

```python
# services/ingest/app.py
@app.route('/ingest', methods=['POST'])
def ingest():
    data = request.get_json()
    enriched = enrich(data)
    publish_click(enriched)
    return '', 204  # Fire and forget from the caller's perspective
```

### Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `flask` | ^3.0 | HTTP server |
| `confluent-kafka` | ^2.3 | Kafka producer (C-backed, high performance) |
| `geoip2` | ^4.7 | MaxMind GeoLite2 IP geolocation |
| `user-agents` | ^2.2 | User-Agent string parser |
| `gunicorn` | ^21.2 | Production WSGI server |

---

## 6. Service 4: Analytics Service

**Language:** Python  
**Framework:** Flask (for the query API)  
**Database:** DynamoDB (via AWS)  
**Role:** Consumes click events from Kafka and writes them to DynamoDB. Also exposes a REST API for querying click analytics.

### Why DynamoDB for Click Events?

Click events are:
- **Write-heavy** — potentially millions of writes/day at scale
- **Schema-flexible** — enriched fields (geo, device) may change over time
- **Query pattern is predictable** — "get clicks for shortCode X in time range Y–Z"

DynamoDB's single-table design excels at exactly this access pattern. You define a partition key (`short_code`) and sort key (`timestamp`) upfront, and range queries become extremely fast and cheap.

### DynamoDB Table Design

```
Table: click_events

Partition Key (PK): short_code  (String)
Sort Key (SK):      timestamp   (String, ISO-8601 for sortability)

Attributes:
  country       String
  city          String
  lat           Number
  lon           Number
  device_type   String  (mobile | tablet | desktop)
  browser       String
  os            String
  referrer      String
  ip            String (optionally hashed for privacy)

GSI (Global Secondary Index):
  GSI-1:
    PK: country
    SK: timestamp
    → "Give me all clicks from France in the last week"
```

**Why a GSI?** The base table answers "clicks for link X in time range." A GSI lets you ask different questions without redesigning the table. GSIs are one of DynamoDB's most important concepts.

### Kafka Consumer

```python
# services/analytics/consumer.py
from confluent_kafka import Consumer
import json

consumer = Consumer({
    'bootstrap.servers': os.getenv('KAFKA_BROKERS'),
    'group.id': 'analytics-consumer-group',
    'auto.offset.reset': 'earliest',   # start from beginning if no committed offset
    'enable.auto.commit': False,       # manual commit for exactly-once semantics
})

consumer.subscribe(['click.events'])

while True:
    msg = consumer.poll(timeout=1.0)
    if msg is None:
        continue
    if msg.error():
        handle_error(msg.error())
        continue

    event = json.loads(msg.value().decode('utf-8'))
    write_to_dynamodb(event)
    consumer.commit()  # commit only after successful write
```

**`enable.auto.commit: False` is important.** If the consumer auto-commits the offset *before* the DynamoDB write succeeds, and then the write fails, that click is lost forever. Manual commit ensures you only acknowledge processing *after* a successful write.

### DynamoDB Write

```python
import boto3

dynamodb = boto3.resource('dynamodb', region_name=os.getenv('AWS_REGION'))
table = dynamodb.Table('click_events')

def write_to_dynamodb(event: dict):
    table.put_item(Item={
        'short_code': event['short_code'],
        'timestamp': event['timestamp'],
        'country': event.get('country', 'XX'),
        'device_type': event.get('device_type', 'unknown'),
        'browser': event.get('browser', 'unknown'),
        'referrer': event.get('referrer', ''),
    })
```

### Query REST API

The Analytics Service also exposes a REST API for the dashboard:

```
GET /analytics/{shortCode}?from=2024-01-01&to=2024-01-31
→ Returns aggregated click counts, top countries, device breakdown

GET /analytics/{shortCode}/raw?from=...&to=...&limit=100
→ Returns raw click records (paginated)
```

### Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| `flask` | ^3.0 | REST query API |
| `confluent-kafka` | ^2.3 | Kafka consumer |
| `boto3` | ^1.34 | AWS SDK — DynamoDB, IAM |
| `gunicorn` | ^21.2 | Production WSGI server |

---

## 7. Inter-Service Communication

### gRPC: Gateway ↔ URL Service

gRPC is used for synchronous request/response calls between services when low latency matters. The redirect path is latency-sensitive — a user is waiting for a page to load — so gRPC's binary serialization (Protocol Buffers) and HTTP/2 multiplexing give it an edge over REST JSON.

**Protocol Buffers (protobuf)** define the schema for messages. Both the client (Gateway) and server (URL Service) generate code from the same `.proto` file, ensuring type safety across service boundaries.

Key concepts to understand:
- **Stub generation** — `grpc_tools.protoc` compiles `.proto` → language-specific client/server code
- **Channels** — a connection pool between client and server
- **Unary RPC** — one request, one response (what you'll use here)
- **Streaming RPC** — not used here, but worth knowing exists

### Kafka: Ingest Service → Analytics Service

Kafka is used for asynchronous, decoupled event streaming. The Ingest Service (producer) does not know or care whether the Analytics Service (consumer) is running. Events are durably stored in Kafka until the consumer reads them.

**Key Kafka concepts:**

| Concept | What it means |
|---------|---------------|
| **Topic** | A named stream of records (`click.events`) |
| **Partition** | A topic is split into N partitions for parallelism. Each partition is an ordered log. |
| **Offset** | The position of a message within a partition. Consumers track their offset. |
| **Consumer Group** | Multiple consumer instances sharing work. Each partition is read by exactly one consumer in the group. |
| **Producer Key** | Determines which partition a message goes to. Same key → same partition → ordered delivery. |
| **Replication Factor** | How many brokers hold a copy of each partition. Set to 3 in production for fault tolerance. |

**Why not just use a message queue (RabbitMQ)?** Kafka is a *log*, not just a queue. Messages are retained after consumption. You can replay events from any offset, which is invaluable for reprocessing historical data (e.g., if your analytics schema changes).

---

## 8. Data Models

### PostgreSQL — URL Service

```
users
  id            SERIAL PK
  email         VARCHAR(255) UNIQUE NOT NULL
  password      VARCHAR(255) NOT NULL        -- bcrypt($password, 12)
  created_at    TIMESTAMPTZ

links
  id            SERIAL PK
  short_code    VARCHAR(10) UNIQUE NOT NULL  -- indexed
  long_url      TEXT NOT NULL
  user_id       INTEGER FK → users.id
  created_at    TIMESTAMPTZ
  expires_at    TIMESTAMPTZ (nullable)
  is_active     BOOLEAN DEFAULT TRUE
```

### Redis — API Gateway + URL Service

```
Key pattern:       url:{shortCode}
Value:             longUrl (string)
TTL:               3600 seconds (1 hour)
Eviction policy:   allkeys-lru (evict least recently used when memory full)

Key pattern:       rl:{userId}  (rate limiter)
Value:             request count (managed by rate-limiter-flexible)
TTL:               60 seconds (rolling window)
```

### Kafka — Click Events Topic

```
Topic:             click.events
Partitions:        6  (adjust based on throughput needs)
Replication:       3 (production) / 1 (local dev)
Retention:         7 days

Message schema (JSON):
{
  "short_code":   "aB3x",
  "timestamp":    "2024-01-15T14:32:00.000Z",
  "ip":           "1.2.3.4",
  "user_agent":   "Mozilla/5.0 ...",
  "referrer":     "https://twitter.com",
  "country":      "US",
  "city":         "San Francisco",
  "lat":          37.7749,
  "lon":          -122.4194,
  "device_type":  "mobile",
  "browser":      "Chrome",
  "os":           "iOS"
}
```

### DynamoDB — Analytics Service

```
Table:             click_events
Billing:           On-demand (pay per request, no capacity planning)

Primary Key:
  PK: short_code   (String)
  SK: timestamp    (String, ISO-8601 → lexicographic sort = chronological sort)

GSI-1 (country-time-index):
  PK: country
  SK: timestamp
  Projection: ALL

GSI-2 (device-time-index):
  PK: device_type
  SK: timestamp
  Projection: ALL
```

---

## 9. Infrastructure (Terraform + AWS)

Terraform manages all AWS resources as code. You define *what* you want; Terraform figures out *how* to create, update, or destroy it.

### Directory Structure

```
infrastructure/
  main.tf           -- root module: calls child modules
  variables.tf      -- input variables (region, cluster name, etc.)
  outputs.tf        -- outputs (cluster endpoint, DB URL, etc.)
  terraform.tfvars  -- variable values (gitignored for secrets)

  modules/
    vpc/            -- VPC, subnets, route tables, NAT gateway
    eks/            -- EKS cluster and node groups
    rds/            -- PostgreSQL RDS instance
    elasticache/    -- Redis ElastiCache cluster
    msk/            -- Managed Kafka (MSK)
    dynamodb/       -- DynamoDB table and GSIs
    ecr/            -- ECR repositories (one per service)
    iam/            -- IAM roles and policies for services
```

### AWS Resources Summary

| Resource | AWS Service | Purpose |
|----------|------------|---------|
| Container orchestration | EKS | Kubernetes cluster |
| Container registry | ECR | Docker image storage (one repo per service) |
| PostgreSQL | RDS | URL mappings (Multi-AZ for HA) |
| Redis | ElastiCache | Caching + rate limits |
| Kafka | MSK | Click event streaming |
| NoSQL DB | DynamoDB | Click event storage |
| Networking | VPC | Isolated private network |
| Secret management | Secrets Manager | DB passwords, JWT secret |
| Load balancing | ALB | Route traffic to EKS |

### Terraform Workflow

```bash
cd infrastructure/

terraform init          # Download providers (AWS, Kubernetes)
terraform plan          # Preview what will be created/changed/destroyed
terraform apply         # Actually provision resources (~15-20 mins first time)
terraform destroy       # Tear everything down (stop paying)
```

**Cost control tip:** Always run `terraform destroy` when you're done learning for the day. EKS costs ~$0.10/hr even when idle. MSK costs ~$0.21/hr per broker (minimum 2 brokers). Running all day = ~$7/day. Spin up only when actively working.

---

## 10. Kubernetes Deployment

Each service gets its own Kubernetes `Deployment` and `Service`. All manifests live in `k8s/`.

### Directory Structure

```
k8s/
  namespaces.yaml
  configmaps.yaml         -- non-secret config (Kafka broker, Redis host)
  secrets.yaml            -- DB passwords, JWT secret (use Kubernetes Secrets or AWS Secrets Manager)

  api-gateway/
    deployment.yaml
    service.yaml          -- type: LoadBalancer (external traffic)
    hpa.yaml              -- HorizontalPodAutoscaler

  url-service/
    deployment.yaml
    service.yaml          -- type: ClusterIP (internal gRPC only)

  click-ingest/
    deployment.yaml
    service.yaml          -- type: ClusterIP (called by gateway)

  analytics/
    deployment.yaml
    service.yaml          -- type: ClusterIP (query API internal)
    hpa.yaml              -- scale based on Kafka consumer lag
```

### Example Deployment Manifest

```yaml
# k8s/url-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: url-service
  namespace: linkmo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: url-service
  template:
    metadata:
      labels:
        app: url-service
    spec:
      containers:
        - name: url-service
          image: <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/linkmo-url-service:latest
          ports:
            - containerPort: 50051   # gRPC port
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: linkmo-secrets
                  key: database-url
            - name: REDIS_URL
              valueFrom:
                configMapKeyRef:
                  name: linkmo-config
                  key: redis-url
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "200m"
          readinessProbe:
            grpc:
              port: 50051
            initialDelaySeconds: 5
            periodSeconds: 10
```

### Horizontal Pod Autoscaler (Analytics Service)

The Analytics Service should scale based on Kafka consumer lag — how many unread messages are piling up. KEDA (Kubernetes Event Driven Autoscaler) enables this:

```yaml
# k8s/analytics/keda-scaledobject.yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: analytics-scaler
spec:
  scaleTargetRef:
    name: analytics-service
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: kafka
      metadata:
        bootstrapServers: <MSK_BROKER>
        consumerGroup: analytics-consumer-group
        topic: click.events
        lagThreshold: "1000"   # scale up when >1000 unread messages per replica
```

---

## 11. Local Development Stack

Everything runs locally using Docker Compose. No AWS account needed until Phase 5.

```yaml
# docker-compose.yml
version: '3.9'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: linkmo
      POSTGRES_USER: linkmo
      POSTGRES_PASSWORD: localpassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"

  api-gateway:
    build: ./services/gateway
    ports:
      - "3000:3000"
    depends_on: [postgres, redis, url-service]
    env_file: ./services/gateway/.env

  url-service:
    build: ./services/url
    ports:
      - "50051:50051"
    depends_on: [postgres, redis]
    env_file: ./services/url/.env

  click-ingest:
    build: ./services/ingest
    ports:
      - "5001:5001"
    depends_on: [kafka]
    env_file: ./services/ingest/.env

  analytics:
    build: ./services/analytics
    ports:
      - "5002:5002"
    depends_on: [kafka]
    env_file: ./services/analytics/.env

volumes:
  postgres_data:
```

Start everything: `docker compose up --build`  
Stop and remove: `docker compose down -v`

---

## 12. Component & Library Reference

### Full Dependency Map

| Service | Language | Key Dependencies |
|---------|----------|-----------------|
| API Gateway | Node.js | express, jsonwebtoken, bcryptjs, @grpc/grpc-js, rate-limiter-flexible, ioredis |
| URL Service | Node.js | @grpc/grpc-js, pg, ioredis |
| Click Ingest | Python | flask, confluent-kafka, geoip2, user-agents, gunicorn |
| Analytics | Python | flask, confluent-kafka, boto3, gunicorn |

### Infrastructure Tools

| Tool | Version | Purpose |
|------|---------|---------|
| Docker | ^24 | Containerize each service |
| Docker Compose | ^2.20 | Local multi-service orchestration |
| Kubernetes | ^1.28 | Production container orchestration (EKS) |
| Terraform | ^1.6 | AWS infrastructure as code |
| GitHub Actions | N/A | CI/CD pipeline |
| KEDA | ^2.12 | Kafka-based pod autoscaling |

### Dockerfile Pattern (Multi-Stage Build)

```dockerfile
# services/gateway/Dockerfile

# Stage 1: Install dependencies
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Runtime image (no dev dependencies)
FROM node:20-alpine AS runtime
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["node", "src/server.js"]
```

Multi-stage builds keep your final image small by not including build tools or dev dependencies.

### Repository Structure

```
linkmo/
  services/
    gateway/          -- Node.js API Gateway
    url/              -- Node.js URL Service
    ingest/           -- Python Click Ingest Service
    analytics/        -- Python Analytics Service
  protos/             -- Shared .proto definitions
  infrastructure/     -- Terraform modules
  k8s/                -- Kubernetes manifests
  docker-compose.yml  -- Local dev orchestration
  .github/
    workflows/
      ci.yml          -- Build + test on PR
      deploy.yml      -- Build, push ECR, deploy to EKS on merge to main
  README.md
```
