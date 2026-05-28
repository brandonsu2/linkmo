# LinkMo — Project Planning & Implementation Guide

> This document is your step-by-step execution guide. It covers prerequisites, the 6-phase build plan with timeframes, exactly what to implement in each phase, how to verify you're done, and FAQs for the questions you will inevitably have.

---

## Table of Contents

1. [Prerequisites & Setup](#1-prerequisites--setup)
2. [Repository Initialization](#2-repository-initialization)
3. [Phase Overview](#3-phase-overview)
4. [Phase 1 — Monolith (Weeks 1–2)](#4-phase-1--monolith-weeks-12)
5. [Phase 2 — Dockerize & Split Services (Weeks 2–3)](#5-phase-2--dockerize--split-services-weeks-23)
6. [Phase 3 — Add Kafka (Weeks 3–4)](#6-phase-3--add-kafka-weeks-34)
7. [Phase 4 — Add gRPC (Weeks 4–5)](#7-phase-4--add-grpc-weeks-45)
8. [Phase 5 — Terraform + AWS (Weeks 5–7)](#8-phase-5--terraform--aws-weeks-57)
9. [Phase 6 — Kubernetes (Weeks 7–9)](#9-phase-6--kubernetes-weeks-79)
10. [Definition of Done](#10-definition-of-done)
11. [Frequently Asked Questions](#11-frequently-asked-questions)

---

## 1. Prerequisites & Setup

Before writing a single line of code, make sure you have everything installed and understand the basics of each tool.

### Required Installations

Run these commands to verify your environment is ready:

```bash
node --version          # Need v18+
python3 --version       # Need 3.10+
docker --version        # Need 24+
docker compose version  # Need 2.20+
git --version
```

Install these before starting Phase 5:
```bash
terraform --version     # Need 1.6+
kubectl version --client
aws --version           # AWS CLI v2
```

### Required Accounts

- **GitHub** — for the repository and GitHub Actions CI/CD
- **AWS** — for Phase 5+. Create a free-tier account if you don't have one. You will need to add a credit card; estimated spend is $15–50 total if you `terraform destroy` regularly.
- **Confluent Cloud** (optional) — Free tier Kafka (5GB/month). Use this instead of AWS MSK to save money during Phases 3–5.

### Prerequisite Knowledge Checklist

You do not need to be an expert in any of these, but you should be comfortable with:

- [ ] Writing Node.js/Express REST APIs (you have this from CSE 135)
- [ ] Basic SQL (SELECT, INSERT, JOIN, indexes)
- [ ] Basic Python (functions, classes, pip, venv)
- [ ] Git branching, PRs, merging
- [ ] Reading JSON API responses with curl or Postman/Insomnia
- [ ] What a Docker container is conceptually (even if you haven't built one)

You do **not** need prior knowledge of: Kubernetes, Terraform, gRPC, Kafka, DynamoDB, or Redis. Each phase introduces these from scratch.

### Recommended Learning (15–30 min each, before starting the relevant phase)

| Topic | Resource | Before Phase |
|-------|----------|-------------|
| Docker fundamentals | Docker's "Getting Started" tutorial (docs.docker.com) | Phase 2 |
| Kafka in 10 minutes | Confluent's "Kafka Fundamentals" course (free) | Phase 3 |
| gRPC basics | grpc.io "Introduction to gRPC" | Phase 4 |
| Terraform basics | HashiCorp's "Get Started: AWS" tutorial (learn.hashicorp.com) | Phase 5 |
| Kubernetes basics | "Kubernetes: Up and Running" Ch. 1–5, or Kelsey Hightower's "Kubernetes the Hard Way" | Phase 6 |

---

## 2. Repository Initialization

Do this once, before Phase 1.

### Step 1: Create the repo

```bash
mkdir linkmo && cd linkmo
git init
git remote add origin https://github.com/YOUR_USERNAME/linkmo.git
```

### Step 2: Create the directory structure

```bash
mkdir -p services/{gateway,url,ingest,analytics}
mkdir -p protos
mkdir -p infrastructure/modules/{vpc,eks,rds,elasticache,msk,dynamodb,ecr,iam}
mkdir -p k8s/{api-gateway,url-service,click-ingest,analytics}
mkdir -p .github/workflows
```

### Step 3: Create root files

```bash
touch README.md
touch .gitignore
touch docker-compose.yml
```

**.gitignore** — add these at minimum:
```
# Node
node_modules/
*.env
.env.*

# Python
__pycache__/
*.pyc
venv/
.venv/

# Terraform
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars          # If contains secrets

# Docker
.dockerignore

# OS
.DS_Store
```

### Step 4: Create a branch strategy

```
main          — always deployable; protected branch
dev           — integration branch
feature/*     — individual features (e.g., feature/url-shortening)
phase/*       — large phase branches (e.g., phase/1-monolith)
```

Commit small and often. Aim for commits that represent a single logical change with a descriptive message:
```
feat: add short code generation with collision check
fix: handle expired links with 410 response
chore: add Dockerfile for url-service
docs: update architecture diagram
```

---

## 3. Phase Overview

```
Phase 1 (Weeks 1–2):  Monolith — single Express app, Postgres, all features working locally
Phase 2 (Weeks 2–3):  Docker — split into 4 services, docker-compose, add Redis
Phase 3 (Weeks 3–4):  Kafka — async click events, Python consumer
Phase 4 (Weeks 4–5):  gRPC — replace REST with gRPC for Gateway ↔ URL Service
Phase 5 (Weeks 5–7):  Terraform + AWS — provision cloud infra, connect managed services
Phase 6 (Weeks 7–9):  Kubernetes — deploy to EKS, write manifests, add CI/CD
```

Each phase builds on the last and is independently shippable. You should be able to demo something working at the end of every phase. Never break main.

---

## 4. Phase 1 — Monolith (Weeks 1–2)

**Goal**: Build the full feature set as a single Express app. No microservices, no Docker, no Kafka. Just make it work.

**Why start with a monolith?** You need to understand the domain — URLs, users, short codes, click events — before layering infrastructure on top. Fighting Kafka and gRPC while also trying to figure out how to generate short codes is too much at once. Build the logic first, then extract it into services.

### What to Build

A single Express app that:
- Connects directly to a local PostgreSQL instance (`pg-pool`)
- Handles user registration and login with JWT
- Creates short codes and stores them in Postgres
- Redirects on `GET /:shortCode`
- Writes click events *synchronously* to a `click_events` table in Postgres (will be replaced with Kafka in Phase 3)
- Exposes analytics queries against that table

### Step-by-Step: Phase 1

**Step 1.1 — Set up PostgreSQL locally**

```bash
# Install PostgreSQL if not already installed (macOS)
brew install postgresql@16
brew services start postgresql@16

# Create DB and user
createdb linkmo
createuser linkmo -P   # set password: linkmo
psql -d linkmo -c "GRANT ALL ON DATABASE linkmo TO linkmo;"
```

**Step 1.2 — Initialize the Node project**

```bash
cd services/gateway    # we'll start here; this IS the monolith in Phase 1
npm init -y
npm install express pg dotenv jsonwebtoken bcryptjs morgan helmet cors
npm install --save-dev nodemon
```

Create `.env`:
```
DATABASE_URL=postgresql://linkmo:linkmo@localhost:5432/linkmo
JWT_SECRET=replace_this_with_32_random_bytes_at_least
PORT=3000
```

**Step 1.3 — Run database migrations**

Create a `db/migrate.js` script that runs your SQL schema (see architecture.md for the schema). Run it once to create tables.

```bash
node db/migrate.js
```

**Step 1.4 — Build in this order**

Build features in this sequence — each depends on the last:

1. **Database connection** — test that `pg.Pool` connects and queries work
2. **User registration** (`POST /api/auth/register`) — insert user, hash password
3. **User login** (`POST /api/auth/login`) — query user, compare bcrypt, sign JWT
4. **Auth middleware** — verify JWT, attach `req.user`
5. **Create short link** (`POST /api/links`) — generate code, insert into DB
6. **Redirect** (`GET /:shortCode`) — query DB, redirect
7. **List links** (`GET /api/links`) — query user's links
8. **Click recording** — on each redirect, insert a `click_events` row synchronously
9. **Analytics query** (`GET /api/analytics/:shortCode`) — aggregate click_events by country, device, time

**Step 1.5 — Test everything with curl/Postman**

```bash
# Register
curl -X POST localhost:3000/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Login
curl -X POST localhost:3000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}'

# Create a short link (use token from login response)
curl -X POST localhost:3000/api/links \
  -H "Authorization: Bearer YOUR_JWT" \
  -H "Content-Type: application/json" \
  -d '{"url":"https://www.google.com"}'

# Test the redirect (use shortCode from above)
curl -v localhost:3000/aB3x
# Should return: 302 Location: https://www.google.com
```

### Phase 1 Completion Criteria

- [ ] User registration and login work; JWT returned
- [ ] Short links are created and stored in Postgres
- [ ] `GET /:shortCode` redirects correctly
- [ ] `GET /:shortCode` for unknown code returns 404
- [ ] Click events are written to the DB on each redirect
- [ ] `GET /api/analytics/:shortCode` returns click count
- [ ] All tests pass (or all manual curl tests succeed)

### Phase 1 Common Mistakes

- **Not hashing passwords**: Never store plaintext. Always `bcrypt.hash(password, 12)`.
- **Forgetting `WHERE user_id = req.user.userId`**: Without this, any user can delete any link.
- **Not handling the Redis miss properly yet**: You don't have Redis in Phase 1 — that's fine. Skip caching entirely.
- **Using `Math.random()` for short codes and not checking collisions**: Always query the DB to confirm a generated code doesn't already exist before inserting.

---

## 5. Phase 2 — Dockerize & Split Services (Weeks 2–3)

**Goal**: Split the monolith into 4 services. Containerize everything. Get all 4 services running with `docker compose up`. Add Redis caching to the URL Service.

### Why Split Now?

You have working code. Splitting is much easier when you know exactly what each piece does. Do not split first and build second.

### Step-by-Step: Phase 2

**Step 2.1 — Copy the monolith into service directories**

```bash
cp -r services/gateway services/url      # Start from the monolith
cp -r services/gateway services/ingest   # (These will be heavily modified)
# Or start fresh for Python services
```

**Step 2.2 — Define service boundaries**

| What it does | Belongs in |
|-------------|------------|
| Auth (register, login, JWT) | API Gateway |
| Short code generation | URL Service |
| DB queries (links, users) | URL Service |
| Redirect logic | API Gateway (calls URL Service) |
| Redis cache logic | URL Service |
| Click event capture | API Gateway (calls Ingest) |
| IP/UA enrichment | Ingest Service |
| Click event storage | Analytics Service |
| Analytics queries | Analytics Service |

**Step 2.3 — Write Dockerfiles for each service**

Write a Dockerfile in each `services/` directory. Use multi-stage builds (see architecture.md for the pattern). Test each one individually:

```bash
cd services/gateway
docker build -t linkmo-gateway .
docker run --env-file .env -p 3000:3000 linkmo-gateway
```

**Step 2.4 — Write `docker-compose.yml`**

See architecture.md for the full `docker-compose.yml`. Key things to get right:
- `depends_on` ensures services start in the right order
- Services communicate using container names as hostnames (e.g., `redis` resolves to the Redis container)
- Use `env_file` to pass environment variables

**Step 2.5 — Add Redis to the URL Service**

Install `ioredis`. Implement the cache-aside pattern in the resolve function (see architecture.md). Test that:
1. First redirect to a code: cache miss, Postgres query, cache write
2. Second redirect to the same code: cache hit, no Postgres query

Verify cache hits by checking Redis directly:
```bash
docker compose exec redis redis-cli
127.0.0.1:6379> GET url:aB3x
"https://www.google.com"
```

**Step 2.6 — Wire the services together (for now, use HTTP)**

In Phase 4 you'll replace this with gRPC, but for now make services call each other over HTTP. The Gateway calls the URL Service at `http://url-service:50051` (or whatever port you use). This is okay temporarily.

**Step 2.7 — Test with docker compose**

```bash
docker compose up --build
# Wait for all services to report healthy
# Run the same curl tests as Phase 1
```

### Phase 2 Completion Criteria

- [ ] All 4 services are containerized with Dockerfiles
- [ ] `docker compose up` starts everything successfully
- [ ] All Phase 1 functionality still works
- [ ] Redis cache is active: cache hit logs appear on second redirect
- [ ] Services communicate with each other (not just with the host machine)

---

## 6. Phase 3 — Add Kafka (Weeks 3–4)

**Goal**: Replace the synchronous click event write (Postgres) with an async Kafka publish. Build the Python Analytics Consumer that reads from Kafka and writes to DynamoDB (local mock for now using `localstack` or just log to console).

### Conceptual Shift: Why Async?

In Phase 1, click recording happened *inside* the redirect request handler. If the DB write was slow, the user waited longer for their redirect. With Kafka, the redirect returns *immediately* and the click event is processed by a separate service asynchronously. The user never waits for analytics.

### Step-by-Step: Phase 3

**Step 3.1 — Add Kafka to docker-compose**

Add Zookeeper + Kafka containers (see architecture.md). Start them:

```bash
docker compose up zookeeper kafka -d
```

Create the topic:
```bash
docker compose exec kafka kafka-topics --create \
  --topic click.events \
  --bootstrap-server kafka:9092 \
  --partitions 3 \
  --replication-factor 1
```

**Step 3.2 — Build the Python Click Ingest Service**

```bash
cd services/ingest
python3 -m venv venv && source venv/bin/activate
pip install flask confluent-kafka geoip2 user-agents gunicorn
```

Build `app.py` (Flask endpoint) and `producer.py` (Kafka producer). For now, skip geo enrichment — just publish the raw click data.

Test the producer works:
```bash
# Start a Kafka console consumer to watch for messages
docker compose exec kafka kafka-console-consumer \
  --topic click.events \
  --bootstrap-server kafka:9092 \
  --from-beginning

# In another terminal, send a test event
curl -X POST localhost:5001/ingest \
  -H "Content-Type: application/json" \
  -d '{"short_code":"aB3x","ip":"1.2.3.4","user_agent":"Mozilla/5.0"}'
```

You should see the JSON message appear in the console consumer.

**Step 3.3 — Modify the API Gateway to call the Ingest Service**

After a successful redirect, fire an async HTTP POST to the Ingest Service. Use `fetch` without `await` — this is intentionally fire-and-forget.

```javascript
// After sending the 302 redirect:
fetch(`${INGEST_SERVICE_URL}/ingest`, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    short_code: shortCode,
    ip: req.ip,
    user_agent: req.headers['user-agent'],
    referrer: req.headers['referer'] || '',
    timestamp: new Date().toISOString(),
  }),
}).catch(err => logger.error('Ingest service error:', err));
```

The redirect is not slowed down by analytics processing.

**Step 3.4 — Build the Python Analytics Consumer**

```bash
cd services/analytics
python3 -m venv venv && source venv/bin/activate
pip install confluent-kafka boto3 flask gunicorn
```

Build `consumer.py` that reads from `click.events` and, for now, just logs each message to stdout. You'll add DynamoDB writes in Phase 5.

```bash
# Test: start the consumer, click some links, watch the logs
docker compose up analytics
docker compose logs -f analytics
```

**Step 3.5 — Add DynamoDB Local (for testing without AWS)**

AWS provides a local DynamoDB emulator:

```yaml
# add to docker-compose.yml
dynamodb-local:
  image: amazon/dynamodb-local:latest
  ports:
    - "8000:8000"
  command: ["-jar", "DynamoDBLocal.jar", "-sharedDb"]
```

Configure `boto3` to point at `http://dynamodb-local:8000`:
```python
dynamodb = boto3.resource(
  'dynamodb',
  endpoint_url='http://dynamodb-local:8000',
  region_name='us-east-1',
  aws_access_key_id='fake',
  aws_secret_access_key='fake'
)
```

Create the table and test writes locally before touching real AWS.

### Phase 3 Completion Criteria

- [ ] Kafka is running locally via docker compose
- [ ] Clicking a short link publishes a message to `click.events`
- [ ] The redirect is not delayed by click processing
- [ ] The Analytics Consumer reads messages and writes to DynamoDB Local
- [ ] Querying DynamoDB Local returns click records

---

## 7. Phase 4 — Add gRPC (Weeks 4–5)

**Goal**: Replace the HTTP calls between API Gateway and URL Service with gRPC. Write your first `.proto` file. Generate client and server stubs. Understand why gRPC is used here.

### Why gRPC Over REST Here?

HTTP/JSON works fine. gRPC is used here because:
1. **You need to learn it** — it's on every SDE interview rubric now
2. **Binary serialization** (protobuf) is faster than JSON parsing under load
3. **Strict schema** — both sides agree on the exact structure of messages; no "what does this field mean?" ambiguity
4. **HTTP/2 multiplexing** — multiple requests over a single connection

### Step-by-Step: Phase 4

**Step 4.1 — Write the proto definition**

Create `protos/url.proto` (see architecture.md for the full definition). This file is the *contract* between Gateway and URL Service.

**Step 4.2 — Generate Node.js stubs**

```bash
npm install -g grpc-tools
grpc_tools_node_protoc \
  --js_out=import_style=commonjs,binary:./generated \
  --grpc_out=grpc_tools_node_protoc_plugin,mode=grpc-js:./generated \
  --proto_path=../protos \
  ../protos/url.proto
```

Or use `@grpc/proto-loader` (simpler — loads the proto at runtime, no codegen step):
```javascript
const packageDefinition = protoLoader.loadSync('../protos/url.proto', {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});
```

**Step 4.3 — Build the gRPC server in the URL Service**

```javascript
// services/url/src/grpc-server.js
const server = new grpc.Server();
server.addService(UrlService, {
  createLink: async (call, callback) => {
    const { long_url, user_id } = call.request;
    try {
      const shortCode = await createLink(long_url, user_id);
      callback(null, { short_code: shortCode });
    } catch (err) {
      callback({ code: grpc.status.INTERNAL, message: err.message });
    }
  },
  resolveLink: async (call, callback) => {
    const { short_code } = call.request;
    const longUrl = await resolveLink(short_code);  // checks Redis, then Postgres
    if (!longUrl) {
      callback({ code: grpc.status.NOT_FOUND, message: 'Link not found' });
    } else {
      callback(null, { long_url: longUrl, found: true });
    }
  },
});

server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
  server.start();
  console.log('gRPC server listening on :50051');
});
```

**Step 4.4 — Build the gRPC client in the API Gateway**

```javascript
// services/gateway/src/clients/url-client.js
const urlClient = new proto.url.UrlService(
  process.env.URL_SERVICE_ADDR,  // 'url-service:50051'
  grpc.credentials.createInsecure()
);

// Promisify for async/await usage
const { promisify } = require('util');
const createLink = promisify(urlClient.createLink.bind(urlClient));
const resolveLink = promisify(urlClient.resolveLink.bind(urlClient));
```

**Step 4.5 — Replace HTTP calls with gRPC calls in the Gateway**

```javascript
// Before (HTTP):
const response = await fetch(`${URL_SERVICE}/resolve/${shortCode}`);
const { longUrl } = await response.json();

// After (gRPC):
const { long_url, found } = await resolveLink({ short_code: shortCode });
if (!found) return res.status(404).send('Not found');
```

**Step 4.6 — Test gRPC directly with grpcurl**

```bash
brew install grpcurl
grpcurl -plaintext -d '{"short_code":"aB3x"}' \
  localhost:50051 url.UrlService/ResolveLink
```

### Phase 4 Completion Criteria

- [ ] Gateway → URL Service communication uses gRPC (no more HTTP between them)
- [ ] All Phase 1 features still work through the gRPC path
- [ ] Error handling: gRPC `NOT_FOUND` maps to HTTP 404, `INTERNAL` maps to HTTP 500
- [ ] `grpcurl` can call the URL Service directly for debugging

---

## 8. Phase 5 — Terraform + AWS (Weeks 5–7)

**Goal**: Provision real AWS infrastructure using Terraform. Connect your services to managed AWS resources (RDS, ElastiCache, MSK/Confluent, DynamoDB). Deploy a working version on AWS (not yet on Kubernetes).

**Warning**: This phase costs money. Run `terraform destroy` at the end of every session.

### Step-by-Step: Phase 5

**Step 5.1 — Configure AWS credentials**

```bash
aws configure
# Enter: Access Key ID, Secret Access Key, region (us-east-1), output format (json)
```

Create a dedicated IAM user for Terraform with minimal permissions. Do not use your root account.

**Step 5.2 — Set up Terraform remote state**

Before running any Terraform, create an S3 bucket for state storage:

```bash
aws s3 mb s3://linkmo-tfstate-YOUR_ACCOUNT_ID --region us-east-1
```

Configure the backend in `infrastructure/main.tf`:
```hcl
terraform {
  backend "s3" {
    bucket = "linkmo-tfstate-YOUR_ACCOUNT_ID"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}
```

This ensures your state is not lost if your local machine explodes.

**Step 5.3 — Write and apply infrastructure modules in order**

Apply one module at a time, verify it works, then move to the next. Do not `terraform apply` everything at once on the first try.

```
Order:
1. VPC → network foundation
2. ECR → you'll push images here before deploying
3. RDS → PostgreSQL
4. ElastiCache → Redis
5. DynamoDB → table + GSIs
6. MSK (or skip and use Confluent Cloud free tier)
7. EKS → last, most expensive
```

**Step 5.4 — Update service configuration to use AWS resources**

Replace local connection strings with the RDS endpoint, ElastiCache endpoint, etc. These come from Terraform outputs:

```bash
terraform output rds_endpoint
terraform output elasticache_endpoint
```

**Step 5.5 — Push Docker images to ECR**

```bash
# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Build and push each service
docker build -t ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/linkmo-gateway:latest ./services/gateway
docker push ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/linkmo-gateway:latest
```

**Step 5.6 — Run services with `docker compose` pointing at AWS resources**

Before deploying to Kubernetes, verify that your services work locally with the real AWS backends:

```bash
# Update .env files with real AWS endpoints
# Then:
docker compose up api-gateway url-service click-ingest analytics
```

Everything should work exactly as in Phase 2, just with AWS resources instead of local containers.

### Phase 5 Completion Criteria

- [ ] `terraform apply` provisions all resources without error
- [ ] RDS is reachable from services (run migrations against it)
- [ ] Redis ElastiCache is reachable; cache-aside works
- [ ] DynamoDB table exists; Analytics Consumer writes to it
- [ ] All services work end-to-end using AWS backends
- [ ] `terraform destroy` cleanly removes everything

---

## 9. Phase 6 — Kubernetes (Weeks 7–9)

**Goal**: Deploy all 4 services to EKS. Write Kubernetes manifests. Set up CI/CD with GitHub Actions. Add autoscaling.

### Step-by-Step: Phase 6

**Step 6.1 — Configure kubectl for EKS**

```bash
aws eks update-kubeconfig --region us-east-1 --name linkmo-cluster
kubectl get nodes    # should show your worker nodes
```

**Step 6.2 — Create namespace and secrets**

```bash
kubectl create namespace linkmo

# Create secrets from your .env values
kubectl create secret generic linkmo-secrets \
  --from-literal=database-url="$DATABASE_URL" \
  --from-literal=jwt-secret="$JWT_SECRET" \
  --namespace linkmo
```

**Step 6.3 — Write and apply manifests in order**

Apply services in dependency order:
1. `kubectl apply -f k8s/` (namespaces, configmaps first)
2. URL Service (no external dependencies)
3. Click Ingest Service
4. Analytics Service
5. API Gateway (depends on URL Service)

```bash
kubectl apply -f k8s/namespaces.yaml
kubectl apply -f k8s/configmaps.yaml
kubectl apply -f k8s/url-service/
kubectl apply -f k8s/click-ingest/
kubectl apply -f k8s/analytics/
kubectl apply -f k8s/api-gateway/

# Watch pods come up
kubectl get pods -n linkmo -w
```

**Step 6.4 — Get the external IP**

```bash
kubectl get service api-gateway -n linkmo
# Note the EXTERNAL-IP (this is your Load Balancer address)
```

Test with curl against this external IP:
```bash
curl http://EXTERNAL_IP/api/auth/register -X POST ...
```

**Step 6.5 — Set up CI/CD with GitHub Actions**

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to EKS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push images
        run: |
          for service in gateway url ingest analytics; do
            IMAGE_URI=${{ secrets.ECR_REGISTRY }}/linkmo-$service:${{ github.sha }}
            docker build -t $IMAGE_URI ./services/$service
            docker push $IMAGE_URI
          done

      - name: Update kubeconfig
        run: aws eks update-kubeconfig --name linkmo-cluster --region us-east-1

      - name: Deploy to EKS
        run: |
          # Update image tags in manifests, then apply
          kubectl set image deployment/api-gateway \
            api-gateway=${{ secrets.ECR_REGISTRY }}/linkmo-gateway:${{ github.sha }} \
            -n linkmo
          # (repeat for each service)
          kubectl rollout status deployment/api-gateway -n linkmo
```

**Step 6.6 — Add Horizontal Pod Autoscaler**

```bash
kubectl autoscale deployment api-gateway \
  --cpu-percent=70 \
  --min=2 \
  --max=10 \
  -n linkmo
```

### Phase 6 Completion Criteria

- [ ] All 4 services are running on EKS
- [ ] External URL (Load Balancer) is accessible and functional
- [ ] CI/CD: push to main triggers an automatic deploy to EKS
- [ ] HPA is configured on at least the API Gateway
- [ ] `kubectl logs` and `kubectl describe` work for debugging

---

## 10. Definition of Done

The project is complete when:

- [ ] All 6 phases are complete
- [ ] End-to-end test: create account → create short link → click it → see analytics
- [ ] All services are deployed to EKS and accessible via a public URL
- [ ] CI/CD pipeline runs automatically on every push to main
- [ ] `terraform destroy` tears down all AWS resources cleanly
- [ ] README explains what the project does, how to run it locally, and how it's deployed
- [ ] You can explain every technology choice in an interview ("why Kafka instead of direct DB write?", "why Redis?", "what is a Kubernetes HPA?")

---

## 11. Frequently Asked Questions

**Q: Do I actually need to run this on AWS? Can I do it all locally?**

Yes, you can do Phases 1–4 entirely locally. DynamoDB Local + local Kafka mean you never need AWS until Phase 5. The AWS phase is where you learn Terraform and real cloud infra — worth doing, but not required to understand the architecture.

---

**Q: Can I use a different database instead of PostgreSQL?**

Technically yes, but Postgres is the right choice here. It's the industry standard relational database, you're already familiar with MySQL (similar), and RDS makes it easy to run managed Postgres on AWS. Switching to MongoDB or SQLite misses the point.

---

**Q: Why does the API Gateway handle auth instead of each service handling its own auth?**

This is a common microservices pattern called the "API Gateway pattern." Centralizing auth at the gateway means each downstream service can trust that any request it receives has already been authenticated. Services don't need their own JWT libraries. The tradeoff: the gateway is a single point of failure for auth — if it's misconfigured, nothing is secure.

---

**Q: What happens if Kafka goes down? Are click events lost?**

If Kafka is unavailable, the Ingest Service's `producer.produce()` call will fail. Click events published during the outage are lost. To prevent this, you can implement a local buffer (the Kafka producer has an internal buffer, and `producer.flush()` will retry). For even stronger guarantees, write to a fallback store (Postgres or Redis queue) when Kafka is unavailable. This is out of scope for this project but worth understanding.

---

**Q: What happens if the Analytics Consumer crashes mid-processing?**

Because you set `enable.auto.commit: False` and commit manually *after* a successful DynamoDB write, the consumer will restart from the last committed offset. The message will be reprocessed. This means DynamoDB writes are "at-least-once" — the same click might be written twice. For analytics, this is acceptable (counts off by a small margin). For payment processing, you'd need idempotency keys.

---

**Q: Why use HTTP 302 for redirects instead of 301?**

HTTP 301 is a permanent redirect — browsers cache it. If a user visits `lnk.io/aB3x` and gets a 301, their browser will redirect to the cached destination on every subsequent visit *without hitting your server at all*. This means you can't track the click. HTTP 302 tells the browser "check with me every time," ensuring every click flows through your system.

---

**Q: Should I build a frontend for this?**

Not as part of this project. A REST API is sufficient to demonstrate and learn all the backend concepts. If you want a frontend later, a simple React app calling your API is straightforward to add. For the resume bullet, the backend architecture is what matters.

---

**Q: How do I manage costs? I'm worried about accidentally leaving AWS resources running.**

1. Set a **billing alarm** in AWS: go to CloudWatch → Alarms → Billing Alarm → alert at $20. You'll get an email before spending too much.
2. Alias `td='terraform destroy'` in your shell. Make it muscle memory.
3. The most expensive resources are EKS ($0.10/hr) and MSK ($0.42+/hr). Skip MSK, use Confluent Cloud's free tier instead.
4. Tag all resources with `Project=linkmo` so you can see your costs in the AWS Cost Explorer.

---

**Q: What order should I tackle bugs when things aren't working?**

Follow the request path from top to bottom:
1. Does the API Gateway receive the request? (check Gateway logs)
2. Does the gRPC call to the URL Service succeed? (check URL Service logs)
3. Is Redis returning the right value? (`redis-cli GET url:{shortCode}`)
4. Is the DB query correct? (test the query directly in `psql`)
5. Is the Kafka message being published? (use `kafka-console-consumer`)
6. Is the Kafka consumer reading it? (check Analytics logs)
7. Is the DynamoDB write succeeding? (check DynamoDB console or aws CLI)

Most bugs are at service boundaries (network config, wrong hostname, wrong port). Always check `docker compose logs SERVICE_NAME` first.

---

**Q: How do I handle schema migrations in production?**

In Phase 1, just drop and recreate the schema. From Phase 5 onward, use a migration tool like `node-pg-migrate` or `flyway`. Migrations are versioned SQL files that are applied in order and tracked in a `schema_migrations` table. Never manually `ALTER TABLE` in production.

---

**Q: Is this project enough to put on my resume?**

Yes, absolutely. The combination of Docker, Kubernetes, Terraform, gRPC, Kafka, Redis, PostgreSQL, and DynamoDB — all motivated by real architectural needs — is more impressive than most internship/entry-level projects. Be prepared to explain every technology choice and walk through the system architecture on a whiteboard. That's where this project earns its value.
