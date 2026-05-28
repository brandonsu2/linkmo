# LinkMo — Additional Recommendations

> This document covers everything not in the other three docs: testing strategy, observability setup, security hardening, API design best practices, git workflow, learning resources, career framing, and architectural evolution paths. These are things experienced engineers wish someone had told them earlier.

---

## Table of Contents

1. [Testing Strategy](#1-testing-strategy)
2. [Observability: Logging, Metrics, and Tracing](#2-observability-logging-metrics-and-tracing)
3. [API Design Best Practices](#3-api-design-best-practices)
4. [Error Handling Patterns](#4-error-handling-patterns)
5. [Git Workflow & Code Quality](#5-git-workflow--code-quality)
6. [Security Hardening Checklist](#6-security-hardening-checklist)
7. [Cost Optimization](#7-cost-optimization)
8. [Learning Resources by Technology](#8-learning-resources-by-technology)
9. [Career Framing: How to Talk About This Project](#9-career-framing-how-to-talk-about-this-project)
10. [Architectural Evolution: What Comes After This](#10-architectural-evolution-what-comes-after-this)
11. [Common Pitfalls to Avoid](#11-common-pitfalls-to-avoid)

---

## 1. Testing Strategy

Testing is one of the most under-taught skills in software engineering. A project with a good test suite is dramatically easier to refactor, extend, and hand off. Here is how to test each layer of LinkMo.

### Testing Pyramid

```
                    /\
                   /  \
                  / E2E \        (few, slow, test full system)
                 /--------\
                /Integration\    (moderate, test service contracts)
               /------------\
              /  Unit Tests   \  (many, fast, test functions in isolation)
             /________________\
```

Start with unit tests (easiest, fastest). Add integration tests as services stabilize. Add E2E tests last.

### Unit Tests

Test individual functions in isolation. Mock all external dependencies (DB, Redis, Kafka).

**Node.js (Gateway + URL Service):** Use Jest.

```bash
npm install --save-dev jest supertest
```

```javascript
// services/url/src/__tests__/shortcode.test.js
const { generateCode, isValidCode } = require('../shortcode');

describe('generateCode', () => {
  test('returns a string of correct length', () => {
    const code = generateCode(7);
    expect(code).toHaveLength(7);
  });

  test('only contains alphanumeric characters', () => {
    const code = generateCode(7);
    expect(code).toMatch(/^[a-zA-Z0-9]+$/);
  });

  test('generates unique codes over 1000 iterations', () => {
    const codes = new Set(Array.from({ length: 1000 }, () => generateCode(7)));
    expect(codes.size).toBe(1000);
  });
});
```

**Python (Ingest + Analytics):** Use pytest.

```bash
pip install pytest pytest-mock
```

```python
# services/ingest/tests/test_enricher.py
from unittest.mock import MagicMock, patch
from enricher import enrich

def test_enricher_adds_device_type():
    event = {
        'ip': '1.2.3.4',
        'user_agent': 'Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X)',
    }
    with patch('enricher.reader.city') as mock_city:
        mock_city.side_effect = Exception("geo lookup failed")
        result = enrich(event)
    
    assert result['device_type'] == 'mobile'
    assert result['country'] == 'XX'  # fallback on geo failure
```

### Integration Tests

Test that two services communicate correctly. For the Gateway ↔ URL Service gRPC contract:

```javascript
// Use a real gRPC server (URL Service) but mock its dependencies (DB, Redis)
// Test that the Gateway correctly calls gRPC and handles responses

test('GET /:shortCode returns 302 for known code', async () => {
  // Mock the gRPC URL Service to return a known URL
  jest.mock('../clients/url-client', () => ({
    resolveLink: jest.fn().mockResolvedValue({ long_url: 'https://google.com', found: true })
  }));

  const response = await request(app).get('/aB3x');
  expect(response.status).toBe(302);
  expect(response.headers.location).toBe('https://google.com');
});
```

### End-to-End Tests

Spin up the full `docker compose` stack and run tests against the live system. Use `supertest` or `httpie` from a test script:

```bash
# e2e/test.sh
#!/bin/bash
set -e

BASE="http://localhost:3000"

# Register
REGISTER=$(curl -s -X POST $BASE/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"e2e@test.com","password":"password123"}')

# Login
LOGIN=$(curl -s -X POST $BASE/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"e2e@test.com","password":"password123"}')
TOKEN=$(echo $LOGIN | jq -r '.token')

# Create link
LINK=$(curl -s -X POST $BASE/api/links \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://example.com"}')
CODE=$(echo $LINK | jq -r '.shortCode')

# Redirect
STATUS=$(curl -s -o /dev/null -w "%{http_code}" -L $BASE/$CODE)
[ "$STATUS" == "200" ] && echo "E2E PASSED" || echo "E2E FAILED"
```

### Running Tests in CI

Add a test job to your GitHub Actions workflow that runs before the deploy job:

```yaml
# .github/workflows/ci.yml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: linkmo_test
          POSTGRES_USER: linkmo
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm test
        env:
          DATABASE_URL: postgresql://linkmo:test@localhost:5432/linkmo_test
```

---

## 2. Observability: Logging, Metrics, and Tracing

You cannot debug what you cannot observe. This is true even for a learning project — when things break (and they will), good observability is the difference between 10 minutes of debugging and 3 hours.

### Structured Logging

**Never do this:**
```javascript
console.log('User logged in: ' + userId);
```

**Do this:**
```javascript
const logger = require('pino')();
logger.info({ userId, email }, 'User logged in');
```

Structured logs are JSON objects. They're machine-parseable, searchable in CloudWatch, and include automatic fields like `time`, `level`, `pid`.

**Node.js:** `pino` is the fastest JSON logger. Use `pino-pretty` in development for human-readable output.

```bash
npm install pino pino-pretty
```

**Python:** `structlog` adds structure to Python's standard logging.

```bash
pip install structlog
```

```python
import structlog
log = structlog.get_logger()
log.info("click_ingested", short_code="aB3x", country="US", device="mobile")
```

**What to log:**
- Every incoming request (method, path, status, latency)
- Every outgoing gRPC call (method, success/failure, latency)
- Every Kafka publish/consume (topic, partition, offset)
- Every DB query that fails
- Every cache hit/miss (at DEBUG level)
- Startup and shutdown events

**What not to log:**
- Passwords (ever, under any circumstance)
- Full JWT tokens (log only the userId extracted from it)
- Full IP addresses in production if GDPR applies (log last-octet-masked: `1.2.3.x`)

### Metrics with Prometheus

Expose a `/metrics` endpoint in each service in Prometheus format. A Prometheus server scrapes these endpoints and stores time-series data. Grafana visualizes it.

**Node.js:**
```bash
npm install prom-client
```

```javascript
const client = require('prom-client');
client.collectDefaultMetrics();  // CPU, memory, event loop lag

const redirectCounter = new client.Counter({
  name: 'linkmo_redirects_total',
  help: 'Total number of short link redirects',
  labelNames: ['short_code', 'cache_hit'],
});

// In your redirect handler:
redirectCounter.inc({ short_code: shortCode, cache_hit: cacheHit ? '1' : '0' });
```

**The RED Method — 3 metrics every service should have:**
- **R**ate: requests per second (`http_requests_total` counter)
- **E**rrors: error rate (`http_request_errors_total` counter)
- **D**uration: latency distribution (`http_request_duration_seconds` histogram)

**Most valuable LinkMo-specific metrics:**
- `linkmo_redirects_total` — total clicks
- `linkmo_cache_hits_total` vs `linkmo_cache_misses_total` — cache effectiveness
- `linkmo_kafka_publish_latency_seconds` — Ingest Service performance
- `linkmo_kafka_consumer_lag` — Analytics Consumer backlog

Add Prometheus + Grafana to docker-compose for local observability:
```yaml
prometheus:
  image: prom/prometheus:latest
  volumes:
    - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
  ports:
    - "9090:9090"

grafana:
  image: grafana/grafana:latest
  ports:
    - "3001:3000"
```

### Distributed Tracing

A trace follows a single request across all services. This is how you answer "why did this specific redirect take 200ms?" when p99 is 20ms.

Use **OpenTelemetry** — it's the vendor-neutral standard. In development, export to **Jaeger** (free, local). In production, use AWS X-Ray.

```bash
npm install @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node
```

Traces automatically capture:
- Incoming HTTP request at the Gateway
- Outgoing gRPC call to the URL Service
- Redis query
- PostgreSQL query (on cache miss)
- Kafka publish to Ingest Service

You can see the entire request timeline in the Jaeger UI. This is the most impressive observability feature to demo in an interview.

---

## 3. API Design Best Practices

### Versioning

Prefix all API routes with `/api/v1/`:

```
POST /api/v1/auth/register
POST /api/v1/links
GET  /api/v1/links
```

When you change a response shape later, you can introduce `/api/v2/` without breaking existing clients.

### Consistent Response Format

Pick a format and stick to it. A common convention:

```json
// Success:
{
  "data": { "shortCode": "aB3x", "longUrl": "https://..." },
  "meta": { "requestId": "uuid" }
}

// Error:
{
  "error": {
    "code": "LINK_NOT_FOUND",
    "message": "No link found for code 'xyz'",
    "requestId": "uuid"
  }
}
```

**Never** mix these patterns within the same API. Inconsistency is the number-one frustration for API consumers.

### HTTP Status Codes — Use Them Correctly

| Situation | Code |
|-----------|------|
| Success, data returned | 200 OK |
| Resource created | 201 Created |
| Success, no body | 204 No Content |
| Redirect | 302 Found |
| Link expired | 410 Gone |
| Validation error | 400 Bad Request |
| Not authenticated | 401 Unauthorized |
| Authenticated but not authorized | 403 Forbidden |
| Resource not found | 404 Not Found |
| Rate limited | 429 Too Many Requests |
| Internal error | 500 Internal Server Error |

### Idempotency

For `POST /api/links`, if the same URL is submitted twice, should it return the same short code or create a new one? Define this behavior explicitly. Idempotency keys (a client-generated UUID in the request header) let clients safely retry without creating duplicates.

### Pagination

Any endpoint returning a list must be paginated. Use cursor-based pagination rather than offset-based for large datasets:

```json
// Instead of ?page=3&limit=20 (offset-based, breaks on inserts)
// Use ?cursor=LAST_ITEM_ID&limit=20 (cursor-based, stable)
{
  "data": [...],
  "nextCursor": "eyJpZCI6MTAwfQ==",
  "hasMore": true
}
```

---

## 4. Error Handling Patterns

### Never Let Unhandled Errors Crash Your Process

```javascript
// Global uncaught exception handler
process.on('uncaughtException', (err) => {
  logger.fatal({ err }, 'Uncaught exception — shutting down');
  process.exit(1);  // let Kubernetes restart the pod
});

process.on('unhandledRejection', (reason) => {
  logger.fatal({ reason }, 'Unhandled promise rejection');
  process.exit(1);
});
```

In Kubernetes, `process.exit(1)` is fine — the pod restarts automatically. An unkilled process in an inconsistent state is worse than a restart.

### Distinguish Error Types

```javascript
class AppError extends Error {
  constructor(message, statusCode, code) {
    super(message);
    this.statusCode = statusCode;
    this.code = code;
    this.isOperational = true;  // operational errors are expected (not bugs)
  }
}

class LinkNotFoundError extends AppError {
  constructor(shortCode) {
    super(`No link found for code '${shortCode}'`, 404, 'LINK_NOT_FOUND');
  }
}

class RateLimitExceededError extends AppError {
  constructor() {
    super('Rate limit exceeded', 429, 'RATE_LIMIT_EXCEEDED');
  }
}
```

Operational errors (wrong input, not found, rate limit) get user-facing error responses. Programming errors (null dereference, type errors) get logged and the process exits.

### Circuit Breaker Pattern

If the URL Service is down and the Gateway keeps trying to call it, every request will hang until timeout. A circuit breaker detects repeated failures and "trips" — subsequent requests fail immediately (fast failure) rather than waiting.

Use the `opossum` library:

```javascript
const CircuitBreaker = require('opossum');
const breaker = new CircuitBreaker(resolveLinkViaGrpc, {
  timeout: 3000,       // fail if function takes longer than 3s
  errorThresholdPercentage: 50,  // trip if >50% of calls fail
  resetTimeout: 30000  // try again after 30s
});
```

This pattern is worth implementing in Phase 5+ and explaining in interviews — it demonstrates understanding of resilience patterns.

---

## 5. Git Workflow & Code Quality

### Branch Strategy

```
main        — protected, requires PR + CI passing before merge
dev         — integration branch, merge feature branches here first
feature/*   — one branch per feature/ticket
fix/*       — bug fix branches
phase/*     — large phase branches
```

Never commit directly to `main`. Even on a solo project, PRs are good practice because they trigger CI and create a paper trail.

### Conventional Commits

Use the conventional commit format — it makes changelogs automatic and helps future-you understand why things were changed:

```
feat: add Redis cache-aside to URL resolution
fix: handle expired link with 410 response
chore: upgrade pino to v8.15
docs: update README with local dev setup
refactor: extract short code generation to separate module
test: add unit tests for bcrypt password validation
```

Tools like `commitlint` can enforce this format in your CI pipeline.

### Linting & Formatting

Use these from day one:

**Node.js:**
```bash
npm install --save-dev eslint prettier eslint-config-prettier
npx eslint --init
```

**Python:**
```bash
pip install black flake8
black services/ingest/  # auto-format
flake8 services/ingest/ # lint
```

Add pre-commit hooks to run these automatically:
```bash
npm install --save-dev husky lint-staged
```

`.pre-commit-config.yaml` (Python):
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.12.0
    hooks:
      - id: black
  - repo: https://github.com/pycqa/flake8
    rev: 7.0.0
    hooks:
      - id: flake8
```

Code that passes a linter is code that looks like it was written by the same person. Reviewers (and future you) can focus on logic, not formatting debates.

### What to Put in Your README

A good README is your project's landing page. It should answer:

```markdown
# LinkMo

> Distributed URL shortener and real-time click analytics platform

## What it does
One sentence.

## Architecture
Diagram or link to architecture.md

## Local Development
### Prerequisites
### Running
```bash
docker compose up --build
```

## API Reference
Table of endpoints with examples

## Tech Stack
| Technology | Purpose |
| ... | ... |

## Deployment
Brief note on how the system is deployed

## What I Learned
This section is gold for recruiters. Be specific.
```

---

## 6. Security Hardening Checklist

Beyond the requirements in requirements.md, these are specific steps to harden the system.

### Input Validation

- [ ] Validate URL input with Node's `URL` constructor — reject anything that throws
- [ ] Block `javascript:`, `data:`, `vbscript:` URL schemes (XSS via redirect)
- [ ] Block requests to private IP ranges in URLs (`127.0.0.0/8`, `10.0.0.0/8`, `192.168.0.0/16`, `169.254.0.0/16`) — prevent Server-Side Request Forgery (SSRF)
- [ ] Limit URL length to 2048 characters
- [ ] Sanitize short code inputs to `/^[a-zA-Z0-9]{1,10}$/` before any DB lookup

### Authentication Hardening

- [ ] JWT secret must be ≥ 32 cryptographically random bytes: `openssl rand -hex 32`
- [ ] Set JWT algorithm explicitly to `HS256` — reject tokens with `alg: none`
- [ ] Implement refresh tokens (long-lived) + access tokens (short-lived, 15 min) — out of scope for MVP but important for production
- [ ] Add `email` to the JWT blacklist table on logout (if sessions are needed)

### HTTP Headers

`helmet` middleware sets secure defaults, but verify these are all active:

```javascript
app.use(helmet({
  contentSecurityPolicy: true,
  crossOriginEmbedderPolicy: true,
  hsts: { maxAge: 31536000, includeSubDomains: true },
  noSniff: true,
  xssFilter: true,
}));
```

### Dependency Security

```bash
npm audit                    # check for known vulnerabilities
npm audit fix                # auto-fix safe upgrades
pip install safety
safety check                 # Python equivalent
```

Enable **Dependabot** on GitHub — it automatically opens PRs for security patches.

### Secrets Scanning

Enable GitHub's secret scanning: Settings → Security → Secret scanning. It will alert you if you accidentally commit an API key or password.

Add a `.gitleaks.toml` config file and run `gitleaks scan .` locally before pushing to catch secrets before they're in git history.

---

## 7. Cost Optimization

### The Only Rule That Matters

**Run `terraform destroy` at the end of every learning session.**

Set a reminder. Better yet, alias it: `alias done='terraform -chdir=infrastructure destroy -auto-approve && echo "AWS resources destroyed"'`

### Per-Service Cost Breakdown

| Resource | Cost When Running | Optimization |
|----------|------------------|-------------|
| EKS control plane | $0.10/hr | Destroy when not learning |
| EC2 nodes (2× t3.small) | ~$0.04/hr | Use spot instances (70% cheaper) |
| RDS db.t3.micro | $0.017/hr | Use single-AZ; stop instance when idle |
| ElastiCache cache.t3.micro | $0.017/hr | Stop when idle |
| MSK (2 brokers) | $0.42/hr | SKIP — use Confluent Cloud free tier |
| DynamoDB on-demand | ~$0/month at low volume | Fine to leave running |
| ECR | $0.10/GB/month | Fine to leave running |
| ALB | $0.008/hr + data | Fine to leave running |

**Approximate hourly cost (all running):** ~$0.58/hr (without MSK)  
**Approximate daily cost:** ~$14  
**Per session (3 hrs of active work):** ~$2

### Confluent Cloud Instead of MSK

Confluent Cloud's free tier includes:
- 5GB of storage
- No expiry
- Managed Kafka with a web UI

This saves ~$0.42/hr (MSK) + significant Terraform complexity. Use Confluent locally and for Phases 3–5. Only provision MSK if you specifically need to demonstrate AWS MSK experience.

### Spot Instances for EKS Nodes

Spot instances use spare AWS capacity at 60–70% discount. They can be interrupted with 2 minutes' notice, but Kubernetes handles this gracefully (pods are rescheduled). For a learning project, interruptions are acceptable.

```hcl
# infrastructure/modules/eks/main.tf
resource "aws_eks_node_group" "default" {
  capacity_type = "SPOT"  # was "ON_DEMAND"
  instance_types = ["t3.small", "t3.medium"]  # multiple types = more spot capacity
}
```

---

## 8. Learning Resources by Technology

### Docker

- **Start here**: Docker's official "Getting Started" guide (docs.docker.com/get-started) — 1 hour
- **Deep dive**: "Docker in Practice" by Ian Miell — best book for the practical side
- **Key concepts**: images vs containers, layers, build cache, multi-stage builds, volumes, networks

### Kubernetes

- **Start here**: Kelsey Hightower's "Kubernetes the Hard Way" (GitHub) — build a cluster manually to understand what managed EKS hides from you
- **Practical guide**: "Kubernetes in Action" by Lukša — comprehensive, readable
- **Interactive**: Play with Kubernetes at play-with-k8s.com (free, browser-based)
- **Key concepts**: pods, deployments, services, configmaps, secrets, namespaces, HPA, readiness/liveness probes

### Terraform

- **Start here**: HashiCorp Learn (learn.hashicorp.com/terraform) — official, free, excellent
- **Key concepts**: providers, resources, state, modules, plan/apply/destroy, remote state
- **Best practice**: never `terraform apply` without reviewing `terraform plan` first

### gRPC & Protocol Buffers

- **Start here**: grpc.io documentation and the official Node.js tutorial
- **Key concepts**: service definitions, message types, field numbers (breaking change if changed), streaming RPCs, error codes
- **Tool**: grpcurl for testing gRPC endpoints (like curl for HTTP)

### Apache Kafka

- **Start here**: Confluent's "Apache Kafka Fundamentals" course (free at confluent.io/learn)
- **Key concepts**: topics, partitions, consumer groups, offsets, retention, exactly-once vs at-least-once delivery
- **Visualization**: Confluent Cloud's web UI makes the Kafka internals visible — use it to understand partitions and consumer lag

### Redis

- **Start here**: redis.io/docs/latest — the official docs are excellent
- **Key concepts**: data structures (strings, hashes, sorted sets, lists), TTL, eviction policies, pub/sub, persistence options
- **Important**: redis.io/commands has a complete command reference with time complexity for each command

### DynamoDB

- **Start here**: Alex DeBrie's "The DynamoDB Book" — the definitive resource
- **Key concepts**: partition key, sort key, GSIs, single-table design, capacity modes, eventual vs strong consistency
- **Common mistake**: DynamoDB requires you to design your data model around access patterns *before* writing code. Unlike SQL, you can't easily add queries later without restructuring.

### PostgreSQL

- **Start here**: The official PostgreSQL 16 documentation (postgresql.org/docs)
- **Key concepts for this project**: EXPLAIN ANALYZE (query planning), indexes (B-tree, GiST), connection pooling with pgBouncer or pg-pool, ACID transactions
- **Tool**: pgAdmin or TablePlus for visual DB management

### AWS (General)

- **Start here**: "AWS Solutions Architect Associate" exam prep — the certification study materials are an excellent structured overview of all AWS services
- **Video**: Stephane Maarek's AWS courses on Udemy are clear and comprehensive
- **Free tier**: aws.amazon.com/free — know what's in the free tier vs what costs money

---

## 9. Career Framing: How to Talk About This Project

This section teaches you how to *communicate* what you built, not just how to build it. The goal is to be able to describe LinkMo in a 30-second elevator pitch, a 5-minute system design walkthrough, and a deep technical dive.

### The 30-Second Version

> "I built a URL shortener with real-time analytics, implemented as four microservices. Users create short links, and every click flows asynchronously through a Kafka pipeline to a DynamoDB analytics store. I deployed the whole thing to Kubernetes on AWS using Terraform for infrastructure as code."

### The 5-Minute Whiteboard Version

Draw the architecture diagram (see architecture.md). Walk through a single request:
1. User clicks `lnk.io/aB3x`
2. Request hits the API Gateway
3. Gateway calls URL Service via gRPC
4. URL Service checks Redis (cache hit → done, 5ms)
5. Gateway issues 302 redirect
6. Gateway fires async call to Ingest Service
7. Ingest enriches with geo/device data, publishes to Kafka
8. Analytics Consumer reads from Kafka, writes to DynamoDB

Then explain *why* each technology was chosen — this is what separates engineers who understand their stack from those who just followed a tutorial.

### Common Interview Questions and How to Answer Them

**"Why Kafka instead of just writing directly to DynamoDB?"**

> "Decoupling. If I write to DynamoDB synchronously inside the redirect handler, a DynamoDB slowdown or outage adds latency to every redirect — or worse, breaks it. Kafka lets the redirect complete immediately and the analytics write happen independently. Kafka also durably buffers events, so if the Analytics Consumer is briefly down, no clicks are lost — they're just processed when it comes back up."

**"How would this handle 1 million clicks per day?"**

> "The redirect path is stateless and horizontally scalable — add more API Gateway replicas. Redis caching means most redirects never hit the database. Kafka partitions can be increased to parallelize the click pipeline. DynamoDB on-demand billing scales write throughput automatically. The Kubernetes HPA scales consumer replicas based on Kafka lag."

**"What would break first at high scale?"**

> "Probably PostgreSQL for URL resolution. Even with Redis caching, a cold-cache scenario (new viral link) would spike DB connections. I'd address this with connection pooling (pgBouncer), replica reads, or moving to a more scalable store for the hot path."

**"What would you do differently?"**

> "For a real production system, I'd add: mTLS between services, proper secrets rotation, distributed tracing from day one, a proper frontend, and I'd revisit whether DynamoDB is the right analytics store — ClickHouse or TimescaleDB might be better for time-series analytics queries."

### On Your Resume

```
LinkMo — Distributed URL Shortener & Analytics Platform
  • Architected a microservices system across 4 containerized services communicating via gRPC,
    with a Kafka-based click event pipeline decoupling ingestion from analytics
  • Deployed to AWS EKS using Terraform-provisioned infrastructure (RDS, ElastiCache, MSK, DynamoDB)
  • Implemented Redis read-through caching achieving sub-10ms redirect p99 latency
  • Automated CI/CD via GitHub Actions with ECR image pushes and rolling Kubernetes deployments
  Tech: Node.js, Python, PostgreSQL, Redis, Kafka, DynamoDB, Docker, Kubernetes, Terraform, AWS
```

---

## 10. Architectural Evolution: What Comes After This

Once you've completed all 6 phases, here are natural extensions that introduce new concepts:

### Extension 1: Service Mesh (Istio or AWS App Mesh)

A service mesh adds:
- mTLS between all services automatically (no code changes)
- Distributed tracing without manual instrumentation
- Traffic splitting (canary deployments, A/B testing)
- Circuit breaking at the infrastructure level

This is the next step after Kubernetes proficiency.

### Extension 2: Event Sourcing & CQRS

Instead of storing the current state of a link (is it active? what's the long URL?), store a log of *events*: "LinkCreated", "LinkUpdated", "LinkDeleted". The current state is derived by replaying events. This is called event sourcing.

CQRS (Command Query Responsibility Segregation) separates write models from read models. LinkMo already has a partial CQRS pattern: writes go to the Kafka → DynamoDB pipeline, reads come from the DynamoDB query API.

### Extension 3: Rate Limiting at the Edge

Move rate limiting from the API Gateway application layer to an edge layer (AWS WAF, Cloudflare). This blocks DDoS traffic before it reaches your infrastructure. Faster, cheaper, and more resilient than application-layer rate limiting.

### Extension 4: Real-Time Dashboard with WebSockets

Add a WebSocket connection from the API Gateway to a frontend dashboard. When a new click event is processed by the Analytics Consumer, it publishes to a Redis Pub/Sub channel. The Gateway subscribes and pushes the event to connected clients in real time. Users see click counts increment live.

### Extension 5: Multi-Region Active-Active

Deploy to two AWS regions (e.g., `us-east-1` and `eu-west-1`). Use Amazon Global Accelerator to route users to the nearest region. DynamoDB Global Tables replicate data across regions automatically. Now you have truly global, sub-20ms redirect latency from anywhere in the world.

This is what Bitly, TinyURL, and similar services actually run in production.

---

## 11. Common Pitfalls to Avoid

These are mistakes almost everyone makes building this kind of project for the first time.

**Pitfall 1: Building all microservices from scratch simultaneously.**
Build the monolith first. Always. You need to understand the domain before splitting it.

**Pitfall 2: Hardcoding configuration.**
Every URL, secret, and connection string belongs in an environment variable. From day one. No exceptions. The refactor from hardcoded to env vars is painful and error-prone.

**Pitfall 3: Not handling the "link deleted while cached" case.**
When a user deletes a link, you MUST invalidate the Redis cache entry immediately. If you don't, the link keeps working for up to an hour (the cache TTL). Always think about cache invalidation when any data changes.

**Pitfall 4: Committing `.env` files.**
Add `.env` and `.env.*` to `.gitignore` immediately. GitHub secret scanning will catch leaked credentials, but it's better to never commit them at all. Use `.env.example` with placeholder values as documentation.

**Pitfall 5: Running Terraform without reviewing the plan.**
`terraform apply` without reading `terraform plan` is how you accidentally delete a production database. Always review the plan. Look for any `destroy` actions and understand why they're there before proceeding.

**Pitfall 6: Ignoring Kafka consumer lag.**
If your consumer is processing events slower than they're being produced, lag will build up. Watch the `consumer_lag` metric. If lag grows unboundedly, add more consumer replicas or optimize the consumer.

**Pitfall 7: Not understanding connection pooling.**
Node.js `pg` without connection pooling will open a new database connection on every request. PostgreSQL has a connection limit (~85 for a t3.micro RDS). You'll hit "too many clients" errors under load. Always use `pg.Pool` with a reasonable `max` value.

**Pitfall 8: Storing analytics in the same PostgreSQL as URLs.**
At scale, analytics write traffic is orders of magnitude higher than URL write traffic. If you use the same database, analytics writes will degrade URL resolution latency. The Kafka → DynamoDB separation exists specifically to prevent this. Don't shortcut it.

**Pitfall 9: Skipping health checks.**
Without liveness/readiness probes, Kubernetes will route traffic to pods that aren't ready yet (during startup) or are in a broken state (hung process). Always implement both probes. They're 5 lines of code and prevent a class of frustrating production issues.

**Pitfall 10: Not writing tests for the short code generator.**
Short codes must be unique. If your generation logic has a subtle bug (off-by-one in the character set, incorrect length), you may not notice until you get a collision in production. Test this function exhaustively: length, character set, uniqueness over N iterations.
