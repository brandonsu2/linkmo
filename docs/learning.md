# LinkMo — Technology Learning Guide

> This document explains every technology used in LinkMo from the ground up, assuming you know what's on your resume and nothing else. Each section starts by anchoring to something you already know, then builds from there. Read this *before* you start building.

---

## Table of Contents

1. [What You Already Know (Your Starting Point)](#1-what-you-already-know-your-starting-point)
2. [Microservices Architecture](#2-microservices-architecture)
3. [Docker — Packaging Code Into Containers](#3-docker--packaging-code-into-containers)
4. [PostgreSQL — MySQL But Different](#4-postgresql--mysql-but-different)
5. [Redis — A Database That Lives in RAM](#5-redis--a-database-that-lives-in-ram)
6. [JWT — Stateless Authentication Tokens](#6-jwt--stateless-authentication-tokens)
7. [gRPC & Protocol Buffers — REST But Faster and Stricter](#7-grpc--protocol-buffers--rest-but-faster-and-stricter)
8. [Kafka — A Permanent, Distributed Log of Events](#8-kafka--a-permanent-distributed-log-of-events)
9. [DynamoDB — A NoSQL Database Built for Scale](#9-dynamodb--a-nosql-database-built-for-scale)
10. [Kubernetes — A System That Runs and Manages Your Containers](#10-kubernetes--a-system-that-runs-and-manages-your-containers)
11. [Terraform — Code That Builds Cloud Infrastructure](#11-terraform--code-that-builds-cloud-infrastructure)
12. [AWS Managed Services — Renting Instead of Building](#12-aws-managed-services--renting-instead-of-building)
13. [How Everything Connects — The Full Picture](#13-how-everything-connects--the-full-picture)

---

## 1. What You Already Know (Your Starting Point)

Before diving into new things, let's be explicit about what you already have that directly applies here. You are not starting from zero.

**Node.js + Express** — You built a full-stack analytics app with Express REST endpoints. You understand how `app.get()`, `app.post()`, middleware, and `req`/`res` work. The API Gateway in LinkMo is just another Express app. 90% of it will feel familiar.

**REST APIs** — You've built and consumed them. You know that a client sends an HTTP request with a method (GET, POST, etc.) and a path, and the server sends back JSON. You know status codes (200, 404, 500). This is exactly what gRPC replaces for one part of this project — and understanding REST makes gRPC easy to grasp by contrast.

**MySQL (SQL databases)** — You've used MySQL with Node.js. You know about tables, rows, columns, SELECT/INSERT/JOIN, and indexes. PostgreSQL is the same mental model with minor syntax differences. If you can write a MySQL query, you can write a PostgreSQL query.

**CI/CD with GitHub Actions** — You've already built pipelines. You know that pushing to main can trigger a workflow that runs tests, builds, and deploys. In LinkMo, the CI/CD pipeline builds Docker images and pushes them to Kubernetes instead of SSH-ing to a VPS — but the concept is identical.

**Python + Cloud Storage (S3/Azure Blob)** — At NociQuant you used `boto3` to interact with S3-compatible storage. AWS DynamoDB uses the same `boto3` SDK. The client setup is almost identical — just a different service name.

**Flask** — You built TrackTonic with Flask. The Python microservices in LinkMo (Click Ingest Service, Analytics Service) are Flask apps. You already know how to build them.

**ThreadPoolExecutor / Concurrency** — At NociQuant you ran 15 concurrent Gemini API workers. This shows you understand the idea of doing multiple things at once rather than sequentially. Kafka's consumer groups (covered below) are the distributed version of this same idea.

With all of that solid, you're actually only learning: Docker, Redis, JWT, gRPC, Kafka, DynamoDB, Kubernetes, and Terraform. That's eight new things, and they all connect logically. Let's go through them one by one.

---

## 2. Microservices Architecture

**What you know:** You built your Database Analytics App as a single Express app. Everything — authentication, URL handling, database queries, analytics — lived in one codebase running as one process.

**What a monolith is:**

A monolith is exactly what you built: one codebase, one process, everything together. When you run `node server.js`, that one process handles every type of request.

```
Monolith:
┌─────────────────────────────────────┐
│           node server.js             │
│                                      │
│  [auth logic]  [DB logic]  [charts]  │
│                                      │
│  → MySQL                             │
└─────────────────────────────────────┘
```

This is completely fine for smaller projects. The problem shows up when the app grows:
- If one part of the app crashes, the entire app crashes
- If you want to scale just the part that's slow, you can't — you have to scale the whole thing
- If you want to update one feature, you redeploy everything
- If two developers are working on different features, they're editing the same files

**What microservices are:**

Instead of one big app, you split the system into multiple small apps, each responsible for one thing. Each runs as its own independent process. They communicate with each other over the network.

```
Microservices:
┌──────────────┐       ┌──────────────┐
│ API Gateway  │──────▶│  URL Service │──→ PostgreSQL
│ (auth, route)│       └──────────────┘   Redis
└──────┬───────┘
       │
       ▼
┌──────────────┐       ┌──────────────┐
│ Ingest Svc   │──────▶│  Analytics   │──→ DynamoDB
│ (Kafka prod) │ Kafka │  Service     │
└──────────────┘       └──────────────┘
```

**The real-world analogy:** A restaurant kitchen. In a tiny restaurant, one chef does everything — takes orders, cooks, plates, cleans. In a large restaurant, there's a different station for each job: appetizers, grill, pastry, expediting. Each station can be scaled independently (add more grill chefs during a rush), and if the pastry station is having a problem, it doesn't stop the grill from working.

**Why LinkMo is built this way:** Not because four microservices is the most practical way to build a URL shortener (it isn't — a monolith would be fine for this scale). It's built this way because learning distributed systems requires a real distributed system to work on. Every technology in this project exists because microservices require it.

**The honest trade-off:** Microservices are harder to build, harder to debug, and harder to deploy than a monolith. You will feel this pain. That is the learning.

---

## 3. Docker — Packaging Code Into Containers

**What you know:** When you deployed your Analytics App to a DigitalOcean VPS, you SSH'd in, installed Node.js, installed your dependencies with `npm install`, and ran your app. If you wanted to run it on a different machine, you'd do all of that again. If the new machine had a different Node.js version, things might break.

**The problem Docker solves:**

"It works on my machine" is one of the oldest complaints in software. The reason things break between machines is *environment differences* — different OS, different language version, different libraries installed, different system paths.

Docker solves this by packaging not just your code, but your entire environment — the OS layer, the language runtime, the dependencies — into a single portable unit called a **container**.

**What a container is:**

A container is a self-contained, isolated process. It has its own filesystem, its own network interface, its own process space. From inside a container, it looks like a complete Linux machine. From outside, it's just a process running on your computer.

Think of it like a shipping container. Before shipping containers, loading cargo onto a ship was chaotic — every item was a different size, needed different handling, required different equipment. The shipping container standardized the *unit of transport*. Every port, every ship, every truck is built to handle the same standard container. It doesn't matter what's inside.

Docker does the same thing for software. It doesn't matter what language your app is written in, what OS it targets, or what dependencies it has. It's in a container. Any machine that runs Docker can run your container.

**A Dockerfile is a recipe:**

A `Dockerfile` describes exactly how to build your container image. It's a text file with instructions:

```dockerfile
# Start from an official Node.js image (someone else already set up Node for us)
FROM node:20-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy our dependencies file
COPY package*.json ./

# Install dependencies (inside the container, not on our machine)
RUN npm ci --only=production

# Copy our app code
COPY . .

# Tell Docker which port our app uses
EXPOSE 3000

# The command to run when the container starts
CMD ["node", "src/server.js"]
```

When you run `docker build`, Docker executes each line and produces an **image** — a snapshot of that environment. When you run `docker run`, Docker starts a container from that image.

**Image vs Container:**
- An **image** is the blueprint — like a class definition
- A **container** is a running instance — like an object created from that class

You have one image, you can run ten containers from it simultaneously.

**Why this matters for LinkMo:**

In Phase 2, you'll write a Dockerfile for each of the 4 services. On any machine — your laptop, a CI server, an AWS server in us-east-1 — anyone can run your service the same way:

```bash
docker build -t linkmo-gateway .
docker run -p 3000:3000 linkmo-gateway
```

No "install Node.js first," no "make sure Python is 3.10+," no "did you run npm install?" It just works. This is why Docker is foundational to everything that comes after.

**Docker Compose — running multiple containers together:**

`docker-compose.yml` is a configuration file that defines multiple containers and how they connect. Instead of running 4 separate `docker run` commands with 15 flags each, you write one YAML file and run `docker compose up`. Docker Compose is your local development environment for the entire system.

```yaml
# This is conceptually what docker-compose.yml does:
# "Run these 4 containers together, let them talk to each other,
#  and here are the environment variables each one needs"
services:
  api-gateway:
    build: ./services/gateway
    ports: ["3000:3000"]
  url-service:
    build: ./services/url
  postgres:
    image: postgres:16
```

**Mental model to remember:** Docker is a shipping container for software. A Dockerfile is the packing list. An image is the packed container. A running container is the container on the ship. Docker Compose is the manifest for an entire ship's worth of containers.

---

## 4. PostgreSQL — MySQL But Different

**What you know:** MySQL. Tables, rows, columns, `SELECT * FROM links WHERE short_code = 'aB3x'`, indexes, foreign keys, joins. You've used `mysql2` with Node.js.

**What PostgreSQL is:**

PostgreSQL (often called "Postgres") is another relational SQL database. It stores data in tables, uses SQL, has transactions, indexes, foreign keys — all the same concepts. The reason to learn it is that PostgreSQL is the industry standard in professional environments. Most companies you'll work at use Postgres over MySQL.

**The practical differences you'll notice:**

| MySQL | PostgreSQL | Notes |
|-------|-----------|-------|
| `AUTO_INCREMENT` | `SERIAL` or `GENERATED ALWAYS AS IDENTITY` | Auto-incrementing IDs |
| `VARCHAR(255)` | `VARCHAR(255)` | Same |
| `TINYINT(1)` for booleans | `BOOLEAN` | Postgres has a real boolean type |
| Double-quotes for identifiers are optional | Double-quotes required for case-sensitive identifiers | Use lowercase names to avoid this |
| `SHOW TABLES` | `\dt` (psql) | Different CLI commands |
| `LIMIT 10 OFFSET 20` | Same | Pagination is identical |
| `mysql2` npm package | `pg` npm package | Different client library |

**`EXPLAIN ANALYZE` — the most important thing to learn:**

This is a Postgres command that shows you *how* Postgres executes a query and how long each step takes. It's how you find slow queries:

```sql
EXPLAIN ANALYZE SELECT * FROM links WHERE short_code = 'aB3x';
```

Without an index on `short_code`, Postgres does a **Sequential Scan** (reads every row). With an index, it does an **Index Scan** (jumps directly to the row). For a table with 10 million links, this is the difference between 50ms and 0.1ms. You've used indexes before — this is just a way to see whether they're actually being used.

**Connection pooling — why you can't just open a connection per request:**

In your Analytics App, you probably opened a database connection and used it. In a production app handling hundreds of requests per second, you can't open a new connection for every request — PostgreSQL can only handle ~100 connections at once, and opening a connection takes ~100ms.

The solution is a **connection pool**: a cache of pre-opened connections that requests borrow and return. The `pg` library includes `pg.Pool`. You create it once at startup:

```javascript
const { Pool } = require('pg');
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,          // maximum 10 connections in the pool
  idleTimeoutMillis: 30000,
});

// In a request handler:
const client = await pool.connect();  // borrow a connection
try {
  const result = await client.query('SELECT * FROM links WHERE short_code = $1', [shortCode]);
  return result.rows[0];
} finally {
  client.release();  // return it to the pool
}
```

This is the same concept as your ThreadPoolExecutor at NociQuant — reuse workers instead of spawning new ones for each task.

**Mental model to remember:** PostgreSQL is MySQL with a different name and slightly better defaults. The SQL you already know works. The important new skill is using `EXPLAIN ANALYZE` to understand query performance.

---

## 5. Redis — A Database That Lives in RAM

**What you know:** Databases (MySQL, PostgreSQL) store data on disk. Reading from disk is slow compared to reading from RAM — your CPU accesses RAM in ~100 nanoseconds and disk in ~1 millisecond. That's 10,000x slower.

**What Redis is:**

Redis is a database where *all data lives in RAM*. Because of this, reads and writes take microseconds — orders of magnitude faster than disk-based databases. The trade-off: RAM is expensive and limited, and data is lost if the machine restarts (by default — Redis has optional persistence, but it's usually not the primary store).

Redis is not a replacement for PostgreSQL. You don't put your entire `links` table in Redis. Redis stores a small, hot subset of your data that gets accessed frequently — the cache.

**The cache-aside pattern — the most common way to use Redis:**

This is the exact pattern LinkMo uses for URL resolution:

```
Question: what URL does short code 'aB3x' map to?

Step 1: Check Redis
  → If found (cache HIT): return it immediately (microseconds, no DB needed)
  → If not found (cache MISS): continue to Step 2

Step 2: Query PostgreSQL
  → Get the URL from the DB

Step 3: Write it to Redis with an expiry time
  → So the next time someone asks, Redis has it

Step 4: Return the URL
```

This is called **cache-aside** or **lazy loading**. The cache is populated on demand, not upfront. Links that nobody clicks are never cached. Links that get thousands of clicks per minute are almost always served from cache.

**How Redis stores data:**

Redis is a key-value store. Every piece of data has a key (a string) and a value (one of several types). For LinkMo's URL cache:

```
Key:   "url:aB3x"
Value: "https://www.google.com"
TTL:   3600 seconds (expires automatically after 1 hour)
```

The command to write this: `SET url:aB3x "https://www.google.com" EX 3600`  
The command to read: `GET url:aB3x`  
The command to delete: `DEL url:aB3x`

That's essentially all you need to know for the cache use case.

**Redis for rate limiting:**

Redis is also used to implement rate limiting. The idea: for each user, store a counter in Redis that increments with each request. The counter expires after 60 seconds. If the counter exceeds the limit (say, 100), reject the request.

```
Key:   "rl:userId:42"       (rate limit for user 42)
Value: 7                     (they've made 7 requests this minute)
TTL:   53 seconds remaining

On next request:
  INCR "rl:userId:42"       → increments atomically to 8
  GET TTL                   → still 53 seconds
  If value > 100: reject request
```

The `INCR` command in Redis is **atomic** — even with 1000 concurrent requests, it's impossible for two requests to both read "99" and both increment to "100" and both sneak through. Redis processes commands one at a time. This is why Redis, not PostgreSQL, is used for rate limiting — it's fast enough to check on every single request.

**Redis in the Node.js code:**

Use the `ioredis` package (more modern than `redis`):

```javascript
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

// Cache write:
await redis.set(`url:${shortCode}`, longUrl, 'EX', 3600);

// Cache read:
const cached = await redis.get(`url:${shortCode}`);
if (cached) return cached;  // cache hit

// Cache delete (on link deletion):
await redis.del(`url:${shortCode}`);
```

**Mental model to remember:** Redis is RAM with a key-value API. Use it for data that is accessed *very frequently* and where losing it temporarily (on restart) is acceptable. The cache makes your redirects fast; without it, every click would need a PostgreSQL query.

---

## 6. JWT — Stateless Authentication Tokens

**What you know:** In your Database Analytics App, you implemented role-based access (Viewer, Analyst, Super Admin). You know that after logging in, a user should stay logged in across requests. The server needs to know who is making each request.

**The traditional way (sessions):**

In a classic session-based system:
1. User logs in with email/password
2. Server creates a **session** (stored in the database or in memory): "session_abc123 belongs to userId 42"
3. Server sends the client a cookie: `session_id=session_abc123`
4. On every future request, the client sends that cookie
5. The server looks up the session in its database to find out who the user is

The problem with sessions in a microservices architecture: which service stores the sessions? If the API Gateway stores sessions and you add a second Gateway replica, the second replica doesn't have the sessions from the first one. You'd need a shared session store (like Redis), which adds complexity.

**JWTs — the stateless alternative:**

A **JSON Web Token (JWT)** is a token that contains user information *inside the token itself*, signed with a secret key. The server doesn't store anything. It just verifies the signature.

A JWT looks like this: `eyJhbGciOiJIUzI1NiJ9.eyJ1c2VySWQiOjQyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`

That's three base64-encoded parts separated by dots:

```
Header:    eyJhbGciOiJIUzI1NiJ9     → {"alg": "HS256"}
Payload:   eyJ1c2VySWQiOjQyfQ       → {"userId": 42, "email": "user@example.com", "exp": 1705436400}
Signature: SflKxwRJSMeKKF2QT4fw...  → HMAC_SHA256(header + "." + payload, SECRET_KEY)
```

**How JWT authentication works in LinkMo:**

```
Login:
  1. POST /api/auth/login { email, password }
  2. Server looks up user in PostgreSQL
  3. bcrypt.compare(submittedPassword, storedHashedPassword)
  4. If matches: sign a JWT with the user's ID and email, set it to expire in 24h
  5. Return the JWT to the client

Every subsequent request:
  1. Client sends: Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
  2. Server extracts the token
  3. Server calls jwt.verify(token, SECRET_KEY)
  4. If valid and not expired: the payload is decoded, req.user = { userId: 42, email: "..." }
  5. If invalid or expired: return 401 Unauthorized
```

**Why this is "stateless":**

The server never stores the token anywhere. To verify a token, it just recalculates the signature using the secret key and checks if it matches. If it does, the token is valid. No database lookup needed.

This is perfect for microservices — *any* replica of the API Gateway can verify *any* JWT, because they all have the same secret key. There's no shared session store to coordinate.

**bcrypt — why you don't store passwords as plaintext:**

Passwords are hashed with bcrypt before storing in the database. bcrypt is a one-way function — you can turn a password into a hash, but you cannot turn a hash back into a password. When a user logs in, you hash their submitted password and compare it to the stored hash.

```javascript
const bcrypt = require('bcryptjs');

// On registration:
const hash = await bcrypt.hash(password, 12);  // 12 = cost factor (how slow/expensive to crack)
// Store hash in DB, never the original password

// On login:
const isValid = await bcrypt.compare(submittedPassword, storedHash);
```

The cost factor (12) makes the hashing slow — about 300ms. That's intentional. If someone steals your database, they'd need 300ms per guess to crack each password. At scale, that makes brute-force attacks impractical.

**Mental model to remember:** A JWT is like a signed passport — issued by the server, carried by the client, verifiable by any server that knows the secret. Stateless: no lookup needed.

---

## 7. gRPC & Protocol Buffers — REST But Faster and Stricter

**What you know:** REST APIs. A client sends an HTTP request with a method and path (`POST /api/links`), a JSON body, and gets a JSON response back. You've built and consumed these.

**The problems with REST for service-to-service communication:**

REST works great when a human (or browser) is the client. But when two backend services are talking to each other, REST has some inefficiencies:

1. **JSON is verbose.** The string `"shortCode": "aB3x"` takes 22 bytes. Binary encoding of the same data takes ~5 bytes.
2. **No contract enforcement.** Nothing stops the URL Service from adding a field `"short_code"` in one version and renaming it to `"shortCode"` in the next, silently breaking the Gateway. There's no enforced schema.
3. **HTTP/1.1 connection overhead.** Each REST request typically opens its own connection. HTTP/2 (which gRPC uses) lets multiple requests share one connection simultaneously.

**What gRPC is:**

gRPC is a framework for making remote procedure calls (RPC). The idea of RPC is simple: make calling a function on another server feel like calling a local function.

Instead of:
```javascript
// REST: manually construct HTTP request, parse JSON response
const response = await fetch(`http://url-service:8080/resolve/${shortCode}`);
const { longUrl } = await response.json();
```

With gRPC:
```javascript
// gRPC: call a function directly (it happens to be on another server)
const { long_url } = await urlClient.resolveLink({ short_code: shortCode });
```

From the calling code's perspective, it's just a function call. The gRPC framework handles everything underneath: serialization, networking, error handling.

**Protocol Buffers — the schema language:**

gRPC uses **Protocol Buffers** (protobuf) to define the structure of messages. You write a `.proto` file that defines what functions exist and what data they take/return:

```protobuf
// protos/url.proto
syntax = "proto3";
package url;

// Define the service: what functions does the URL Service expose?
service UrlService {
  rpc CreateLink (CreateLinkRequest) returns (CreateLinkResponse);
  rpc ResolveLink (ResolveLinkRequest) returns (ResolveLinkResponse);
}

// Define the message types: what data do the functions take/return?
message ResolveLinkRequest {
  string short_code = 1;   // the "= 1" is a field number, not a default value
}

message ResolveLinkResponse {
  string long_url = 1;
  bool found = 2;
}
```

This `.proto` file is the **contract** between the API Gateway (caller) and the URL Service (callee). Both sides must agree on this schema. If the URL Service changes `short_code` to `shortCode`, the `.proto` changes, both sides regenerate their code, and the mismatch is caught at compile time — not in a 3am production incident.

**How it works under the hood:**

```
API Gateway                          URL Service
(gRPC client)                        (gRPC server)

resolveLinkRequest         →         receives the request
  { short_code: "aB3x" }            runs the handler function
                                     returns:
  { long_url: "...",      ←         resolveLinkResponse
    found: true }
```

The `.proto` compiler (`protoc`) generates code in your language that handles all the networking. You just call functions.

**What "binary serialization" means and why it's faster:**

JSON is text: `{"short_code":"aB3x","found":true}` — 33 bytes, human-readable.

Protobuf is binary: `0x0a 0x04 0x61 0x42 0x33 0x78 0x10 0x01` — ~8 bytes, not human-readable but much smaller and faster to parse.

For a URL shortener doing thousands of redirects per second, this difference compounds.

**Mental model to remember:** gRPC is like REST, but instead of "send HTTP to a URL and parse JSON," you "call a function defined in a .proto schema." It's faster (binary, HTTP/2), stricter (schema enforced at compile time), and better suited to service-to-service calls.

---

## 8. Kafka — A Permanent, Distributed Log of Events

This is the most conceptually new technology in the project. Take your time with this section.

**What you know:** At NociQuant, you ran 15 workers in parallel with `ThreadPoolExecutor`. You know the idea of doing work asynchronously — submitting a task to a pool and letting a worker pick it up later, rather than waiting for it yourself.

**The problem Kafka solves — a scenario:**

Imagine the redirect handler in your URL Service does three things synchronously:
1. Look up the long URL (fast — Redis cache)
2. Issue the 302 redirect (instant)
3. Write a click event to the database (slow — 50ms)

Step 3 makes every user wait 50ms longer for their redirect, for something they don't care about (analytics). Worse: if the analytics database is down, the redirect *fails*. A database you need to write to for stats is now in the critical path of a redirect.

**The solution: decouple with a message queue / event log.**

Instead of writing to the analytics database directly, the redirect handler *publishes an event* to Kafka: "a click happened." It then immediately returns the redirect. Separately, the Analytics Service *reads* that event from Kafka and writes it to DynamoDB — asynchronously, after the redirect has already completed.

The user gets their redirect in ~5ms. The analytics write happens ~1 second later. The user never waits.

```
Without Kafka:
Redirect request → [Redis lookup] → [redirect] → [DB write 50ms] → response (total: ~55ms)

With Kafka:
Redirect request → [Redis lookup] → [redirect] → [publish to Kafka <1ms] → response (total: ~6ms)
                                              ↓ (separately, async)
                                    [Analytics Consumer reads event]
                                    [writes to DynamoDB]
                                    (user is already on their destination page)
```

**What Kafka actually is:**

Kafka is a **distributed, durable, ordered log of events**.

The word "log" is important. Kafka doesn't delete messages after they're consumed (unlike a traditional queue). It keeps every message for a configured retention period (7 days in LinkMo). This means:

- If your Analytics Consumer crashes, all the unprocessed messages are still in Kafka, waiting
- When it restarts, it picks up exactly where it left off
- You can "replay" events from any point in history (useful if your analytics schema changes and you need to reprocess data)

**Core Kafka concepts — explained with analogies:**

**Topic:** A category of messages. Like a named folder for events. LinkMo has one topic: `click.events`. All click events from all users go into this one topic.

**Producer:** The service that writes messages into a topic. In LinkMo, the Click Ingest Service is the producer — it calls `producer.produce('click.events', message)`.

**Consumer:** The service that reads messages from a topic. The Analytics Service is the consumer — it continuously reads from `click.events` and writes to DynamoDB.

**Partition:** Each topic is divided into N partitions. A partition is an ordered, append-only log. Think of it like splitting a single Google Doc into multiple separate documents, each on a different server. Partitions enable parallelism.

```
Topic: click.events (3 partitions)

Partition 0: [click on aB3x] [click on zK9m] [click on aB3x] ...
Partition 1: [click on qR5t] [click on aB3x] [click on pL2w] ...
Partition 2: [click on mN7x] [click on zK9m] [click on qR5t] ...
```

**Offset:** The position of a message within a partition. Message at offset 0 is first, offset 1 is second, etc. The consumer tracks its offset to know where it left off.

**Consumer Group:** If you run multiple Analytics Consumer instances, they form a consumer group. Kafka automatically assigns each partition to exactly one consumer in the group. If you have 3 partitions and 3 consumer instances, each instance reads one partition. If one instance crashes, Kafka reassigns its partition to another instance. This is your distributed version of `ThreadPoolExecutor` — multiple workers sharing the load.

```
Consumer Group: analytics-consumers (3 instances)

Partition 0 → Consumer Instance A
Partition 1 → Consumer Instance B
Partition 2 → Consumer Instance C

If Consumer A crashes:
Partition 0 → Consumer Instance B (now reading 2 partitions)
Partition 1 → Consumer Instance B
Partition 2 → Consumer Instance C
```

**The producer key — why messages are keyed by short_code:**

When producing a message, you provide a key. Kafka uses the key to determine which partition the message goes to (via hashing). All messages with the same key always go to the same partition.

In LinkMo, messages are keyed by `short_code`. This means all clicks for link `aB3x` land on the same partition, in order. This matters if you later want to do stream processing that aggregates events by link (e.g., counting clicks in real time) — you need all events for a link to be in one place.

**Exactly-once vs at-least-once delivery:**

By default, Kafka gives **at-least-once** delivery: a message is guaranteed to be delivered, but might be delivered more than once if a consumer crashes after processing but before committing its offset.

In code, this looks like:

```python
# BAD: auto-commit — commits offset automatically, even if processing failed
consumer = Consumer({'enable.auto.commit': True})

# GOOD: manual commit — commit ONLY after successful DynamoDB write
consumer = Consumer({'enable.auto.commit': False})

msg = consumer.poll()
write_to_dynamodb(event)  # if this fails, we don't commit
consumer.commit()         # commit only after successful write
```

If the consumer crashes between `write_to_dynamodb` and `consumer.commit()`, the message will be reprocessed after restart. The DynamoDB write happens twice. For analytics, this is acceptable — a click count slightly inflated by a double-write is not critical. For payments, it would not be acceptable.

**Mental model to remember:** Kafka is like a durable, distributed newspaper. The producer (Ingest Service) publishes articles (click events). The newspaper holds them for 7 days. Consumers (Analytics Service) subscribe and read articles at their own pace. If a consumer misses today's paper, it can catch up — the articles are still there. The newspaper doesn't disappear once someone reads it.

---

## 9. DynamoDB — A NoSQL Database Built for Scale

**What you know:** SQL databases (MySQL, PostgreSQL). Rows, columns, tables, JOIN, WHERE. You know how to query data with SQL.

**What NoSQL means:**

"NoSQL" doesn't mean "no SQL" — it means "not only SQL." NoSQL databases abandon some of the constraints of relational databases to gain performance and scalability at extreme scale.

The key thing relational databases guarantee that NoSQL databases relax: **joins**. In PostgreSQL you can do `SELECT * FROM links JOIN users ON links.user_id = users.id`. DynamoDB cannot do this. There are no joins. Ever.

This sounds like a limitation (it is), but it's also what makes DynamoDB incredibly fast and infinitely scalable. When there are no joins, a query always touches exactly one partition of one table. There's no query that accidentally scans 10 million rows. Performance is predictable.

**What DynamoDB is:**

DynamoDB is AWS's fully managed NoSQL database. "Fully managed" means AWS handles everything: servers, storage, replication, backups, scaling. You never SSH into a DynamoDB server. You just use it.

Data is stored as **items** (like rows) in a **table**. Each item is a collection of **attributes** (like columns), but unlike SQL, different items in the same table can have different attributes. One click event might have a `city` attribute; another might not.

**The most important DynamoDB concept: design around access patterns first.**

In SQL, you design your schema first, then query it however you want. In DynamoDB, you design your schema *around your queries*. If you design the wrong schema, you cannot fix it with a different SELECT statement — you have to restructure your data.

**Partition Key and Sort Key:**

Every DynamoDB table has a **primary key**. There are two options:

1. **Simple primary key:** Just a Partition Key. Each item is uniquely identified by this one attribute.
2. **Composite primary key:** Partition Key + Sort Key. Items with the same Partition Key are grouped together and sorted by Sort Key.

LinkMo uses a composite key for the `click_events` table:

```
Partition Key (PK): short_code     → "aB3x"
Sort Key (SK):      timestamp      → "2024-01-15T14:32:00.000Z"
```

This enables the primary access pattern: **"give me all clicks for link 'aB3x' between time X and time Y."**

```python
# Query: all clicks for link 'aB3x' in January 2024
response = table.query(
    KeyConditionExpression=Key('short_code').eq('aB3x') &
                           Key('timestamp').between('2024-01-01', '2024-02-01')
)
```

This is fast because DynamoDB stores all items with the same Partition Key together, sorted by Sort Key. The query touches one slice of one partition — no table scan.

**Why ISO-8601 strings for the timestamp sort key?**

`"2024-01-15T14:32:00.000Z"` sorts alphabetically in chronological order — January 1 sorts before January 15, which sorts before January 31. This is intentional. DynamoDB sorts sort keys as strings, so using an ISO format ensures time-order equals alphabetical order.

**Global Secondary Indexes (GSIs):**

Your primary key defines one access pattern. But what if you also need to query "all clicks from the US"? The primary key is `short_code` — you can't query by country without scanning the entire table.

A **Global Secondary Index (GSI)** is essentially a copy of your table with a *different* primary key, maintained automatically by DynamoDB. You define a GSI with `country` as the partition key and `timestamp` as the sort key. Now both of these are fast:

```
Primary table:      "give me all clicks for link 'aB3x' in January"  → query by PK+SK
GSI-1 (country):    "give me all clicks from the US in January"       → query by GSI
GSI-2 (device):     "give me all clicks from mobile devices in January" → query by GSI-2
```

Think of a GSI as an index in a textbook — the main content is organized one way (chronologically), but the index at the back lets you look up content by a different attribute (subject).

**Why DynamoDB instead of PostgreSQL for click events?**

At scale, click events are extremely write-heavy. Imagine a viral link getting 100,000 clicks per hour. PostgreSQL can handle this, but you'd need to carefully tune it: partitioned tables, write-ahead log settings, connection pooling, read replicas. It's manageable but complex.

DynamoDB at on-demand billing just handles it. No configuration, no tuning. You write items; DynamoDB scales. For high-write, schema-flexible, key-value-access-pattern data, DynamoDB is the right tool.

**Mental model to remember:** DynamoDB is a hash map in the cloud that scales infinitely. Know your access patterns before you design your table. The partition key determines where your data lives; the sort key determines the order within a partition. No joins — ever.

---

## 10. Kubernetes — A System That Runs and Manages Your Containers

**What you know:** You know Docker makes containers. You know GitHub Actions can run scripts automatically. In your Analytics App, deploying meant SSH-ing to a VPS and running `git pull && pm2 restart`.

**The problem with running containers manually:**

Docker is great for running one container. But LinkMo has 4 services. In production, you want at least 2 copies of each service (so one crashing doesn't take the site down). That's 8+ containers. Plus PostgreSQL, Redis, Kafka — more containers. On multiple VPS machines.

Questions arise:
- Which machine should each container run on?
- What happens if a container crashes?
- What happens if a machine dies?
- How do containers discover each other's IP addresses?
- How do you roll out an update to the URL Service without any downtime?
- How do you scale the Analytics Service to 5 copies when Kafka lag grows?

Doing this manually with SSH and shell scripts is possible but painful and error-prone. **Kubernetes automates all of this.**

**What Kubernetes is:**

Kubernetes (K8s) is a container orchestration system. It manages a cluster of machines (called **nodes**) and schedules containers across them. You tell Kubernetes *what you want* (run 2 copies of the URL Service, always), and Kubernetes makes it happen and keeps it that way.

The key mental shift: **you tell Kubernetes what the desired state is. It figures out how to achieve it and maintains it.**

```
You say:                       Kubernetes does:
"Run 2 copies of URL Service"  → starts 2 containers across available nodes
                               → if one crashes, starts a new one automatically
                               → if a node dies, reschedules the container elsewhere
                               → keeps this up 24/7
```

**Core Kubernetes objects — the vocabulary you need:**

**Pod:** The smallest unit in Kubernetes. A Pod is one or more containers that run together on the same machine. Usually one container per Pod. Pods are ephemeral — they can be killed and replaced at any time.

**Deployment:** A declaration of *how many* Pods of a given type you want running, and which container image to use. A Deployment is how you tell Kubernetes "I want 2 copies of the URL Service." If a Pod crashes, the Deployment controller creates a new one to maintain the count.

```yaml
# "I want 2 copies of the URL Service, using this Docker image"
apiVersion: apps/v1
kind: Deployment
metadata:
  name: url-service
spec:
  replicas: 2          # always maintain 2 Pods
  selector:
    matchLabels:
      app: url-service
  template:
    spec:
      containers:
        - name: url-service
          image: your-ecr-repo/url-service:latest
```

**Service (Kubernetes Service, not your microservice):** Pods have IP addresses, but those addresses change when Pods restart. A Kubernetes Service is a stable network address (like a load balancer) that routes traffic to healthy Pods matching a label. Other Pods use the Service name as a hostname.

```
api-gateway container calls:  url-service:50051 (stable, never changes)
    ↓
Kubernetes Service "url-service" routes to:
    → url-service Pod 1 (IP: 10.0.0.4)   or
    → url-service Pod 2 (IP: 10.0.0.7)
  (whichever is healthy, load balanced automatically)
```

**ConfigMap:** A place to store non-secret configuration (e.g., the Redis hostname, Kafka broker address). Injected into containers as environment variables.

**Secret:** Like a ConfigMap but encrypted. Used for database passwords, JWT secrets, API keys. Kubernetes Secrets are base64-encoded and stored in etcd (Kubernetes' internal database), with access controls.

**HorizontalPodAutoscaler (HPA):** Automatically scales the number of Pod replicas based on a metric. For the Analytics Service: "if Kafka consumer lag > 1000 messages per replica, add more replicas."

**Rolling deployments — zero-downtime updates:**

When you push a new Docker image for the URL Service, Kubernetes performs a **rolling update**:
1. Start a new Pod with the new image
2. Wait for it to pass health checks
3. Route some traffic to it
4. Terminate an old Pod
5. Repeat until all Pods are on the new version

At no point are there zero healthy Pods. Users don't experience downtime.

**kubectl — the CLI for Kubernetes:**

Just like `docker` is the CLI for Docker, `kubectl` is the CLI for Kubernetes:

```bash
kubectl get pods -n linkmo          # list all pods
kubectl logs url-service-xyz -n linkmo  # see logs from a pod
kubectl describe pod url-service-xyz       # debug a pod
kubectl apply -f k8s/url-service/          # apply manifest files
kubectl rollout status deployment/url-service  # watch a deployment roll out
```

**Mental model to remember:** Kubernetes is a self-healing robot that runs your containers. You write YAML files declaring what you want. Kubernetes makes it real and keeps it that way. If something breaks, it fixes it. You interact with it via `kubectl`.

---

## 11. Terraform — Code That Builds Cloud Infrastructure

**What you know:** You've used GitHub Actions for CI/CD — code that automates what used to be a manual process (run tests, deploy). Terraform applies the same idea to infrastructure setup.

**The problem Terraform solves:**

To run LinkMo on AWS, you need to create a lot of infrastructure:
- An EKS cluster (Kubernetes)
- An RDS PostgreSQL database
- An ElastiCache Redis cluster
- A DynamoDB table
- ECR repositories for Docker images
- A VPC with subnets
- IAM roles and permissions
- A load balancer

You *could* click through the AWS console to create all of this. But then:
- How do you do it again exactly when you need to recreate it?
- How does a teammate recreate your exact setup?
- How do you know what's deployed?
- How do you delete everything cleanly at the end?

Clicking through a UI is not reproducible, not reviewable, and not automatable.

**Infrastructure as Code (IaC):**

Terraform lets you describe your entire infrastructure in code — `.tf` files. The code declares what AWS resources you want. Running `terraform apply` creates them. Running `terraform destroy` deletes them. Every change is in git history. Your entire cloud environment is reproducible from a text file.

**What Terraform code looks like:**

```hcl
# Create a DynamoDB table
resource "aws_dynamodb_table" "click_events" {
  name         = "click_events"
  billing_mode = "PAY_PER_REQUEST"   # on-demand

  hash_key  = "short_code"          # partition key
  range_key = "timestamp"           # sort key

  attribute {
    name = "short_code"
    type = "S"   # String
  }

  attribute {
    name = "timestamp"
    type = "S"
  }
}
```

This is not code that *runs* on AWS — it's a declaration of what you want to exist. Terraform reads this, figures out what currently exists in AWS, computes the difference (the **plan**), and shows you what it will create, change, or destroy before doing anything.

**The Terraform workflow:**

```
terraform init      → download the AWS provider plugin (like npm install for Terraform)
terraform plan      → "here's what I would do" (always review this)
terraform apply     → actually do it
terraform destroy   → undo everything
```

**State — how Terraform knows what's deployed:**

Terraform maintains a **state file** (`terraform.tfstate`) that maps your code to real AWS resources. This is how it knows that the `aws_dynamodb_table.click_events` in your code corresponds to the actual DynamoDB table named `click_events` in AWS account `123456789`.

The state file must be stored in a shared location (S3 bucket) so that any machine running Terraform sees the current state. Never commit the state file to git — it contains sensitive resource IDs and can contain secrets.

**Modules — reusable infrastructure components:**

Just like functions in code, Terraform **modules** are reusable infrastructure components. The `eks` module encapsulates all the resources needed to create an EKS cluster. You call it like a function:

```hcl
module "eks" {
  source       = "./modules/eks"
  cluster_name = "linkmo-cluster"
  region       = "us-east-1"
}
```

**Mental model to remember:** Terraform is a git-committed receipt of your cloud infrastructure. `terraform apply` spins everything up. `terraform destroy` tears it all down. Changes are versioned and reviewable. Never click around the AWS console for anything that Terraform can manage.

---

## 12. AWS Managed Services — Renting Instead of Building

You've used S3 at NociQuant. AWS has managed versions of all the databases and infrastructure LinkMo needs. "Managed" means AWS handles the server, OS, patching, backups, and scaling — you just use the service.

**EKS — Elastic Kubernetes Service**

AWS-managed Kubernetes. Instead of setting up Kubernetes yourself (which takes days and requires deep knowledge), AWS runs the Kubernetes **control plane** (the scheduler, the API server, the etcd database). You just provision worker nodes (EC2 instances) and tell EKS what containers to run.

EKS costs $0.10/hr for the control plane, always. This is the reason to `terraform destroy` when not learning.

**RDS — Relational Database Service**

AWS-managed PostgreSQL (and MySQL, and others). Instead of running `postgres` in a Docker container, AWS runs a dedicated PostgreSQL server, handles backups, applies security patches, provides automated failover. You just connect with a connection string.

The connection string format: `postgresql://username:password@your-rds-endpoint.amazonaws.com:5432/dbname`

You've already used `DATABASE_URL` environment variables — this is the same, just pointing at RDS.

**ElastiCache — Managed Redis**

AWS-managed Redis. Same Redis commands, same `ioredis` client. You just connect to the ElastiCache endpoint instead of `localhost:6379`.

**MSK — Managed Streaming for Kafka**

AWS-managed Kafka. Same `confluent-kafka` client, same producer/consumer code. You just point `bootstrap.servers` at the MSK broker addresses instead of `kafka:9092`.

Note: MSK requires minimum 2 brokers and is expensive (~$0.42/hr total). For this project, use Confluent Cloud's free tier until the final deployment.

**DynamoDB**

DynamoDB is entirely AWS — there's no "install DynamoDB" equivalent. You interact with it exclusively through `boto3`:

```python
import boto3
dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('click_events')
table.put_item(Item={'short_code': 'aB3x', 'timestamp': '...'})
```

This is identical to how you used `boto3` for S3 at NociQuant — just replace `s3.upload_file()` with `table.put_item()`.

**ECR — Elastic Container Registry**

AWS-managed Docker image storage. Like Docker Hub, but private and integrated with AWS. Your CI/CD pipeline builds Docker images and pushes them to ECR. EKS pulls images from ECR to run containers.

Think of ECR like GitHub — but for Docker images instead of code.

---

## 13. How Everything Connects — The Full Picture

You've now learned each technology individually. Here's how they all fit together into one coherent system.

**The dependency chain — why each technology requires the next:**

```
You want to build a URL shortener
  → Multiple features (auth, URLs, analytics) → Microservices
    → Microservices need to be packaged consistently → Docker
      → Docker containers need to talk to a database → PostgreSQL
        → PostgreSQL queries are slow for hot data → Redis cache
          → Services need to authenticate users → JWT
            → Services need to call each other efficiently → gRPC
              → Click recording must not slow down redirects → Kafka
                → Kafka events need a write-optimized store → DynamoDB
                  → Multiple Docker containers need orchestration → Kubernetes
                    → Kubernetes cluster needs cloud infra → Terraform + AWS
```

**The request lifecycle — everything in action at once:**

Let's trace what happens when `aB3x` is clicked, from start to finish:

```
1. User's browser requests GET /aB3x
   → DNS resolves to AWS Application Load Balancer
   → Load Balancer routes to API Gateway Pod (Kubernetes)

2. API Gateway (Node.js/Express, in Docker container, running on EKS)
   → JWT middleware checks Authorization header (stateless — no DB lookup)
   → Rate limiter checks Redis (ElastiCache) — fast, ~0.1ms
   → Calls URL Service via gRPC: resolveLink({ short_code: "aB3x" })

3. URL Service (Node.js, different Docker container on EKS)
   → Checks Redis (ElastiCache): GET url:aB3x
   → CACHE HIT: returns "https://google.com" immediately (~0.2ms)
   → Returns gRPC response: { long_url: "https://google.com", found: true }

4. API Gateway sends HTTP 302 redirect to browser
   → User's browser navigates to Google
   → User is already on Google before steps 5–8 complete

5. API Gateway fires-and-forgets to Click Ingest Service:
   POST /ingest { short_code, ip, user_agent, referrer, timestamp }

6. Click Ingest Service (Python/Flask, another Docker container)
   → Enriches event with geo (MaxMind GeoLite2) and device type (user-agent parser)
   → Publishes to Kafka topic "click.events" (AWS MSK)
   → Message key: "aB3x" → always goes to same Kafka partition

7. Kafka (AWS MSK) durably stores the event
   → Partition 2, offset 14,872
   → Will hold it for 7 days

8. Analytics Service (Python, another Docker container)
   → Kafka Consumer continuously polls for new messages
   → Reads the click event
   → Writes to DynamoDB (AWS): { PK: "aB3x", SK: "2024-01-15T...", country: "US", device: "mobile" }
   → Commits Kafka offset (acknowledges successful processing)

Total time user waited: ~5ms
Total time for analytics to be queryable: ~3 seconds
```

**The technologies by role:**

| Role | Technology | Why This One |
|------|-----------|-------------|
| HTTP server / routing | Express.js | You already know it |
| Container packaging | Docker | Reproducible, portable environments |
| Local multi-service dev | Docker Compose | Runs all 4 services with one command |
| URL + user data storage | PostgreSQL (RDS) | Relational, ACID, you know SQL |
| Hot path caching | Redis (ElastiCache) | RAM-speed reads, cache-aside pattern |
| Auth tokens | JWT + bcrypt | Stateless, no session store needed |
| Service-to-service calls | gRPC + protobuf | Fast, strict contract, HTTP/2 |
| Async click pipeline | Kafka (MSK) | Durable, decoupled, replayable events |
| Analytics storage | DynamoDB | Write-optimized, key-value access pattern |
| Container orchestration | Kubernetes (EKS) | Self-healing, scalable, rolling deploys |
| Infrastructure provisioning | Terraform | Reproducible, version-controlled infra |
| Automated deployment | GitHub Actions | You already know this pattern |

**Where your existing skills appear:**

- Your Express/Node.js skills → API Gateway and URL Service (same as your Analytics App)
- Your Flask skills → Click Ingest Service and Analytics Service (same as TrackTonic)
- Your SQL skills → PostgreSQL schema design and queries
- Your `boto3`/S3 skills → DynamoDB (same SDK, different service)
- Your GitHub Actions skills → CI/CD pipeline that builds Docker images and deploys to EKS
- Your Python skills → enrichment logic, Kafka consumer, DynamoDB writes
- Your CI/CD knowledge → you understand *why* the deployment pipeline works

**The one insight that ties it all together:**

Every technology in this system exists to solve one specific problem that emerges from scale. You could build LinkMo's features in a single Express app with one MySQL database — and at low traffic, it would work fine. The technologies exist because at 10,000 clicks per second:

- A single process becomes a bottleneck → Microservices
- Environments diverge → Docker
- Disk reads are too slow for the hot path → Redis
- Synchronous analytics writes slow down redirects → Kafka
- SQL schemas are too rigid for high-write event data → DynamoDB
- Manual container management breaks down → Kubernetes
- Manual infrastructure setup is not reproducible → Terraform

You're not adding complexity for its own sake. Each layer solves a concrete problem that the layer below it creates.

---

## Quick Reference: "What do I look up when X happens?"

| Situation | Command / Tool |
|-----------|---------------|
| My container won't start | `docker compose logs SERVICE_NAME` |
| I need to check what's in Redis | `docker compose exec redis redis-cli` then `GET url:aB3x` |
| I need to check if Kafka got my message | `docker compose exec kafka kafka-console-consumer --topic click.events --bootstrap-server kafka:9092 --from-beginning` |
| My PostgreSQL query is slow | `EXPLAIN ANALYZE SELECT ...` in psql |
| A Kubernetes pod is crashing | `kubectl describe pod POD_NAME -n linkmo` and `kubectl logs POD_NAME -n linkmo` |
| I want to see what Terraform will create | `terraform plan` |
| I want to delete all AWS resources | `terraform destroy` |
| I need to test a gRPC endpoint | `grpcurl -plaintext localhost:50051 url.UrlService/ResolveLink` |
| I need to query DynamoDB locally | `aws dynamodb scan --table-name click_events --endpoint-url http://localhost:8000` |
