---
name: ms-spec
description: Map the high-level microservice architecture and generate per-service spec .md files that give AI agents full context — owned APIs, DB tables, events, inter-service dependencies, and data flows. Use when user wants to document microservice interactions, generate context files for AI agents, understand service boundaries, or says "ms-spec" or "map my services".
tools: Read, Glob, Grep, Bash
---

# Microservice Spec Generator

You are helping a backend engineer document a microservice architecture. The output is a set of `.md` spec files — one per service — that any AI agent can read in future sessions to understand the full system without needing to ask again.

## Goal

1. Build a **system map**: what services exist, what they own, and how they interact
2. **Deep dive** into each service: APIs, DB schema, events, dependencies
3. Write a spec `.md` for each service + one top-level `ARCHITECTURE.md`
4. All files go into `docs/architecture/` (or a path the user specifies)

---

## Phase 1: Discovery

### 1A. Ask the user for their starting point

```
To map your microservices, I need some starting context. Please share what you have:

1. [PREFERRED] A list of your service repo names or a monorepo folder listing
2. [OPTIONAL] An API gateway config, service mesh config, or docker-compose.yml
3. [OPTIONAL] Any existing architecture diagrams or docs
4. [OPTIONAL] CI/CD config that references service names (e.g. GitHub Actions workflows)

If we're inside a specific service repo right now, I'll start there and ask you to repeat this for each other service.
```

### 1B. Auto-discover from the current repo

Search the current directory for signals:

```bash
# Service identity
cat package.json 2>/dev/null | grep '"name"'
cat pyproject.toml 2>/dev/null | grep "^name"
cat pom.xml 2>/dev/null | grep -m1 "<artifactId>"
cat go.mod 2>/dev/null | grep "^module"

# Routes / API surface
find . -name "*.route.*" -o -name "*.controller.*" -o -name "*.handler.*" | head -20
find . -name "router.*" -o -name "routes.*" | head -10

# DB schema
find . -name "schema.prisma" -o -name "schema.sql" -o -name "*.migration.*" | head -10

# Events (Kafka, RabbitMQ, SQS, etc.)
grep -r "publish\|subscribe\|emit\|consumer\|producer\|topic\|queue\|exchange" --include="*.ts" --include="*.js" --include="*.go" --include="*.py" --include="*.java" -l | head -10

# Inter-service HTTP calls
grep -r "http\|axios\|fetch\|grpc\|feign\|RestTemplate" --include="*.ts" --include="*.js" --include="*.go" --include="*.py" --include="*.java" -l | head -10

# Docker / deployment
cat docker-compose.yml 2>/dev/null
cat docker-compose.yaml 2>/dev/null
find . -name "Dockerfile" | head -5
```

### 1C. If docker-compose.yml or gateway config is available

Extract the full service list from it — this is your definitive service inventory. List:
- Service name
- Port
- Dependencies (depends_on)
- Environment variables that reference other services (e.g. `ORDERS_SERVICE_URL`)

---

## Phase 2: System Map

Build the high-level interaction map. For each service, determine:

| Dimension | How to Find It |
|-----------|---------------|
| **Owns** | What domain/data it is responsible for |
| **Exposes** | REST/gRPC/GraphQL endpoints |
| **Publishes** | Events/messages it emits |
| **Subscribes** | Events/messages it consumes |
| **Calls** | Other services it makes HTTP/gRPC calls to |
| **DB** | Which database/tables it owns |
| **Auth** | How it authenticates requests |

Search for inter-service calls:
```bash
# Find service URLs referenced in env/config
grep -r "SERVICE_URL\|_HOST\|_ENDPOINT\|grpc\|service\." --include="*.env*" --include="*.yml" --include="*.yaml" --include="*.ts" --include="*.go" -h | sort -u | head -30

# Find event topic/queue names
grep -r "topic\|queue\|exchange\|subject" --include="*.ts" --include="*.go" --include="*.py" --include="*.java" -h | grep -v "\/\/" | sort -u | head -30
```

Output a **System Map** in this format before writing any files:

```
## System Map

Services:
  - api-gateway      → routes to: orders-service, user-service, inventory-service
  - orders-service   → calls: inventory-service, payment-service
                     → publishes: order.created, order.shipped
                     → DB: orders, order_items
  - inventory-service→ subscribes: order.created
                     → DB: products, stock
  - user-service     → DB: users, sessions
  - payment-service  → calls: (external: Stripe)
                     → subscribes: order.created
                     → publishes: payment.success, payment.failed

Data Flows:
  Place Order:  FE → api-gateway → orders-service → inventory-service (stock check)
                                                   → payment-service (charge)
                                 ← order.created event → inventory-service (reserve stock)
```

**Show this map to the user and ask for corrections before proceeding.**

---

## Phase 3: Deep Dive Per Service

For each service, extract the full spec. If in a monorepo, switch directories. If separate repos, ask the user to provide access or paste the relevant files.

### API Surface

```bash
# List all route definitions
grep -rn "router\.\(get\|post\|put\|patch\|delete\)\|@Get\|@Post\|@Patch\|@Delete\|@Put\|r\.Handle\|mux\.Handle" \
  --include="*.ts" --include="*.go" --include="*.py" --include="*.java" | head -50

# Find OpenAPI/Swagger spec if present
find . -name "openapi.*" -o -name "swagger.*" | head -5
```

For each endpoint, capture:
- Method + path
- Path/query params
- Request body shape (from DTOs, validators, or type definitions)
- Response shape (from serializers, response types, or example responses)
- Auth required (yes/no/role)

### DB Schema

```bash
# Prisma
cat prisma/schema.prisma 2>/dev/null

# SQL migrations
find . -name "*.sql" -path "*/migrations/*" | sort | tail -5 | xargs cat 2>/dev/null

# Django models
find . -name "models.py" | xargs grep "class.*Model" 2>/dev/null

# TypeORM / Hibernate entities
find . -name "*.entity.*" | head -10 | xargs cat 2>/dev/null
```

### Events

```bash
# Event names published
grep -rn "publish\|emit\|send\|produce" --include="*.ts" --include="*.go" --include="*.py" | grep -i "topic\|event\|message" | head -20

# Event names subscribed
grep -rn "subscribe\|consume\|listen\|on(" --include="*.ts" --include="*.go" --include="*.py" | grep -i "topic\|event\|message" | head -20
```

### Outbound Calls

```bash
grep -rn "http\.\|axios\.\|fetch(\|grpc\." --include="*.ts" --include="*.go" | grep -v "test\|spec\|mock" | head -20
```

### Domain Pollution Scan

Look for code that does not belong to this service's stated purpose:

```bash
# Find all business domain keywords used across the codebase
# e.g. if this is orders-service, flag references to "payment", "inventory", "user profile" logic
grep -rn "class\|function\|def \|func " --include="*.ts" --include="*.go" --include="*.py" --include="*.java" | grep -iv "test\|spec\|mock" | head -60

# Find TODO/FIXME/HACK comments — often signal misplaced logic or known debt
grep -rn "TODO\|FIXME\|HACK\|XXX\|TEMP\|WORKAROUND\|should not be here\|belongs to\|move this" \
  --include="*.ts" --include="*.js" --include="*.go" --include="*.py" --include="*.java" | head -30
```

Flag any functions/classes whose names suggest they belong to a different business domain. These are candidates for the "Domain Pollution" section in the spec.

### Legacy & Dead Code Scan

```bash
# Deprecated markers
grep -rn "@deprecated\|@Deprecated\|# deprecated\|// deprecated\|DEPRECATED\|obsolete\|legacy\|old_\|_old\|_v1\|_v2\|_new\|_backup" \
  --include="*.ts" --include="*.js" --include="*.go" --include="*.py" --include="*.java" | head -30

# Feature flags / toggles — often gate old vs new implementations
grep -rn "featureFlag\|feature_flag\|isEnabled\|FEATURE_\|toggle\|rollout\|experiment\|if.*flag\|if.*enabled" \
  --include="*.ts" --include="*.js" --include="*.go" --include="*.py" --include="*.java" | head -20

# Dual route versions (v1 vs v2 of same endpoint)
grep -rn "v1\|v2\|v3" --include="*.route.*" --include="*.controller.*" --include="*.handler.*" | head -20

# Commented-out code blocks (sign of abandoned implementation)
grep -rn "^[[:space:]]*//" --include="*.ts" --include="*.go" | grep -v "import\|eslint\|prettier\|license" | wc -l

# Old migration files vs current schema — large gaps suggest abandoned columns
find . -name "*.sql" -path "*/migrations/*" | sort | head -5
find . -name "*.sql" -path "*/migrations/*" | sort | tail -5
```

Identify and classify:
| Type | Signal | Action in Spec |
|------|--------|---------------|
| **Legacy endpoint** | `/v1/` route still mounted alongside `/v2/` | Document as `⚠ LEGACY` with note on whether it's still called |
| **Dead feature flag** | Flag that's always `true` or `false` in all envs | Flag as likely removable |
| **Dual implementation** | Two classes/functions doing the same thing, one newer | Document which is active, which is legacy |
| **Misplaced domain logic** | Orders service with `calculateTax()` or `updateUserProfile()` | Flag as domain pollution |
| **Orphaned DB columns** | Column exists in schema, never queried in code | Flag as potentially safe to drop |
| **Commented-out routes** | Endpoint defined but commented out | Flag as dead code |

---

## Phase 4: Write Spec Files

Create one `.md` file per service and one top-level architecture file.

### File: `docs/architecture/ARCHITECTURE.md`

```markdown
# System Architecture

## Services

| Service | Responsibility | Port | DB |
|---------|---------------|------|----|
| api-gateway | Request routing, auth | 8080 | - |
| orders-service | Order lifecycle | 3001 | PostgreSQL (orders) |
| ... | | | |

## Interaction Map

[paste the system map from Phase 2]

## Event Bus

| Event | Publisher | Subscribers |
|-------|-----------|-------------|
| order.created | orders-service | inventory-service, payment-service |
| payment.success | payment-service | orders-service |

## Shared Conventions

- Auth: [e.g. JWT Bearer token passed in Authorization header]
- Service discovery: [e.g. Kubernetes DNS / env vars / Consul]
- API versioning: [e.g. /api/v1/...]
- Error format: [e.g. { error: string, code: string }]

## Data Ownership

Each service owns its own DB. Cross-service data access is via API only, not direct DB queries.

| Data | Owner Service |
|------|--------------|
| users, sessions | user-service |
| orders, order_items | orders-service |
| products, stock | inventory-service |
```

### File: `docs/architecture/<service-name>.md`

```markdown
# <Service Name>

## Purpose

One paragraph: what this service is responsible for and what it is NOT responsible for.

## API Endpoints

### GET /api/v1/orders/:id

**Auth**: Bearer token required
**Path params**: `id` — order UUID

**Response**:
\`\`\`json
{
  "id": "uuid",
  "status": "pending | shipped | delivered",
  "items": [{ "productId": "uuid", "quantity": 2, "price": 9.99 }],
  "createdAt": "ISO8601"
}
\`\`\`

[repeat for each endpoint]

---

## Database

**Engine**: PostgreSQL
**Owns**: `orders`, `order_items`

### orders
| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| user_id | UUID | FK → user-service (by reference, no join) |
| status | ENUM | pending, shipped, delivered |
| created_at | TIMESTAMPTZ | |

### order_items
| Column | Type | Notes |
|--------|------|-------|
| id | UUID | PK |
| order_id | UUID | FK → orders.id |
| product_id | UUID | Reference to inventory-service |
| quantity | INT | |
| price_at_purchase | DECIMAL | Snapshot, not live price |

---

## Events

### Publishes

| Event | Trigger | Payload |
|-------|---------|---------|
| `order.created` | New order placed | `{ orderId, userId, items[], totalAmount }` |
| `order.shipped` | Status updated to shipped | `{ orderId, trackingNumber }` |

### Subscribes

| Event | From | Action |
|-------|------|--------|
| `payment.success` | payment-service | Updates order status to confirmed |
| `payment.failed` | payment-service | Updates order status to payment_failed |

---

## Outbound Calls

| Service | Endpoint | When |
|---------|----------|------|
| inventory-service | GET /products/:id/stock | Before confirming order |
| payment-service | POST /charges | On order placement |

---

## Key Business Rules

- An order cannot be placed if stock < quantity (checked synchronously)
- Price is snapshotted at purchase time — not recalculated
- Orders are soft-deleted (deleted_at) not hard-deleted

---

## Context for AI Agents

When working in this service:
- Route handlers are in `src/controllers/`
- Business logic is in `src/services/`
- DB queries are in `src/repositories/`
- Events are published via `src/events/publisher.ts`
- All responses go through `src/mappers/` before being returned
```

---

## Phase 5: Grill the Developer

Before writing any files, interrogate the developer until every gap is filled. Do NOT proceed to Phase 6 until you have confident answers to all questions in the relevant categories.

### Rules for grilling
- Ask **one category at a time** — do not dump all questions at once
- If an answer is vague ("it depends", "sometimes", "I think"), probe further
- If the developer says "I don't know", mark it as `[UNKNOWN]` in the spec and move on — do not get stuck
- Keep a running checklist of resolved vs unresolved gaps and show it after each round
- Stop grilling only when: every service has a clear owner, every inter-service call is confirmed, and every `[UNKNOWN]` has been either filled or explicitly accepted as unknown

### Category 1: Service Boundaries

For each service where ownership is unclear:

> "You have [service A] and [service B] — both seem to touch orders. Which one is the source of truth for order status? Who writes to it and who only reads?"

> "Does [service X] ever write directly to another service's database, or is everything through API calls?"

> "Are there any services I haven't discovered yet — batch jobs, cron workers, internal admin tools, data pipelines?"

### Category 2: API Contracts

For any endpoint where the request/response shape is incomplete:

> "For `POST /orders` — what fields are required vs optional in the request body? What does a validation error response look like?"

> "Does this endpoint behave differently based on the caller's role or permissions? E.g. an admin vs a regular user gets different fields back?"

> "Are there any deprecated fields still in the response that the FE still reads? I want to flag those in the spec."

### Category 3: Data Flows & Edge Cases

> "Walk me through what happens end-to-end when a user places an order. Which services are called synchronously (blocking) vs asynchronously (events)?"

> "What happens if [downstream service] is down? Does [upstream service] retry, queue, fail fast, or return partial data?"

> "Are there any flows where the same event is published more than once (e.g. retries)? How do consumers handle duplicates?"

### Category 4: Events

For any event where publisher or subscriber is unclear:

> "Who publishes `[event name]`? Is there only one publisher or can multiple services emit it?"

> "When `[event name]` fails to process — does the consumer retry? Dead-letter queue? Alert?"

> "Are there any events that are published but have NO active subscribers right now? I want to flag those."

### Category 5: Data Ownership & Shared Data

> "Is there any data that two services both store copies of? E.g. does orders-service store a copy of the product name at purchase time?"

> "Are there any shared databases — i.e. two services pointing at the same DB or same schema? I want to document that as a known coupling."

> "For cross-service references like `user_id` in the orders table — is that validated at write time, or trusted blindly?"

### Category 6: Auth & Security

> "How does service-to-service auth work — shared secret, mTLS, service account JWT, or nothing?"

> "Are there any endpoints that are public (no auth)? Which ones?"

> "Which endpoints require specific roles or permissions beyond just being authenticated?"

### Category 7: Known Gaps & Tech Debt

> "Are there any interactions between services that are messy or that you'd call out as 'this shouldn't work this way but it does'? I want to document that honestly so future engineers know."

> "Any services that are being deprecated or replaced? I'll add a deprecation notice to their spec."

> "Anything about this system that surprised you when you first joined — something non-obvious that an AI agent would likely get wrong if it only read the code?"

### Grilling Completion Checklist

Before moving to Phase 6, confirm:

- [ ] Every service has a clear stated purpose and data ownership
- [ ] Every inter-service call is confirmed (not just inferred from code)
- [ ] Every event has a confirmed publisher and at least one confirmed subscriber
- [ ] Every API endpoint has a known request/response shape (or `[UNKNOWN]`)
- [ ] Auth mechanism is documented for both external and internal calls
- [ ] All `[UNKNOWN]` placeholders are explicitly accepted by the developer

If any item is unchecked, keep grilling on that specific gap.

---

## Phase 6: Validate & Finalize

After writing all files, output a summary:

```
## Generated Files

docs/architecture/
├── ARCHITECTURE.md          ← system map, service table, event bus, data ownership
├── orders-service.md        ← API endpoints, DB schema, events, outbound calls
├── user-service.md
├── inventory-service.md
└── payment-service.md

## How to use these with AI

In any service repo, add to your CLAUDE.md:
  "See docs/architecture/<service-name>.md for this service's spec."
  "See docs/architecture/ARCHITECTURE.md for the full system map."

Or paste the relevant .md files at the start of your Claude session.
```

Ask the user:
> "Are there any services missing from the map, or any interactions I got wrong? I can update the files before you commit them."

---

## Grilling Philosophy

The goal is not to annoy the developer — it is to make the `.md` files trustworthy enough that a future AI agent (or a new engineer) can rely on them without second-guessing. Every `[UNKNOWN]` left in a spec is a liability. Every assumption left undocumented is a future bug.

Be persistent but focused: one gap at a time, one follow-up at a time. When the developer gives a partial answer, acknowledge what was answered and drill into what's still missing. When they're done, tell them exactly what got resolved and what remains open.

---

## Guardrails

- Do NOT invent service names, endpoints, or events — only document what you find in code or what the user confirms
- If a service's repo is not available, create a stub `.md` with `[UNKNOWN — repo not provided]` placeholders
- If the user's system uses a different event bus (Kafka vs SQS vs RabbitMQ), adapt the Events section accordingly but keep the same structure
- Keep each spec file self-contained — an AI agent should be able to read ONE file and understand that service without reading the others
- Flag assumptions clearly with `> ⚠ Assumption: ...` in the generated files