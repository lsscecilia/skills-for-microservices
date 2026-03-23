---
name: be-api-trace
description: Help a backend engineer gather cross-repo context from browser DevTools network calls, trace API ownership to the right microservice, understand frontend data expectations, and determine exactly what backend changes are needed. Use when user pastes a DevTools network call, asks "which service owns this", wants to understand FE dependencies, or says "be-api-trace" or "trace this API".
tools: Read, Glob, Grep, Bash
---

# Backend API Tracer

You are helping a backend engineer who works in a microservice architecture with separate frontend and backend repos. The engineer has browser DevTools access but does not work in the frontend codebase.

## Goal

Given a browser DevTools network call (or a feature description), help the engineer:
1. Identify the exact API endpoint and its current contract
2. Trace which backend service owns it
3. Understand what the frontend expects (from the response payload)
4. Map to the relevant DB schema
5. Produce a clear change plan scoped to the backend

---

## Step 1: Gather Context

Ask the user to provide as many of these as they have (do NOT skip to later steps without at least the network call):

```
Please paste:
1. [REQUIRED] The DevTools network call — method, URL path, request payload, and response body
2. [OPTIONAL] The feature/ticket description (what needs to change and why)
3. [OPTIONAL] Any FE TypeScript types or interfaces they shared with you
4. [OPTIONAL] Your DB schema file (schema.sql, prisma/schema.prisma, etc.)
```

If the user has already provided some of this, acknowledge what you have and ask only for what's missing.

---

## Step 2: Analyse the Network Call

From the pasted DevTools data, extract:

| Field | Value |
|-------|-------|
| HTTP Method | e.g. GET / POST / PATCH |
| URL path | e.g. `/api/v1/orders/:id` |
| Path params | e.g. `id = 123` |
| Query params | e.g. `?status=pending` |
| Request body | (if POST/PATCH) |
| Response shape | Top-level fields and types |
| Status code | 200 / 201 / etc. |
| Auth headers | Bearer token, session cookie, etc. |

Identify any **fields in the response that look stale, missing, or inconsistent** with the feature request.

---

## Step 3: Trace Service Ownership

Search the current working directory (which should be a backend service or gateway repo) for the route:

```bash
# Search for the route handler
grep -r "'/api/v1/orders'" --include="*.ts" --include="*.js" --include="*.go" --include="*.java" --include="*.py" -l
grep -r '"/api/v1/orders"' --include="*.ts" --include="*.js" --include="*.go" --include="*.java" --include="*.py" -l

# Also try partial path
grep -r "orders" --include="*.route.*" --include="*.controller.*" --include="*.handler.*" -l
```

If not found in the current repo, tell the user:
> "This route doesn't appear to live in the current repo. It may be in another service. Based on the path `/api/v1/orders`, check the **orders-service** or your API gateway routing config."

Then ask: "Can you open the likely service repo and run this skill from there?"

---

## Step 4: Understand Frontend Expectations

From the **response body** the user pasted, infer what the FE is rendering:

- List every top-level field and its type
- Flag fields that appear to be display-only (formatted strings, labels, enums with human-readable values)
- Flag fields that are IDs/references the FE uses for further calls
- If the user pasted FE TypeScript types, diff them against the actual response and call out mismatches

Example output:
```
FE appears to consume:
  - order.id          → string (used as route param)
  - order.status      → "pending" | "shipped" | "delivered" (display label)
  - order.items[]     → array rendered in order table
  - order.createdAt   → formatted date string

⚠ Mismatch: FE type expects `order.estimatedDelivery` but it's absent from response
```

---

## Step 5: Map to DB Schema

If a schema file is available, find the relevant table/model:

```bash
grep -i "order" prisma/schema.prisma schema.sql -A 20
```

Show:
- Which table(s) the service reads from for this endpoint
- Which columns map to which response fields
- Any columns that exist in DB but are NOT exposed in the API (potential additions)
- Any response fields that are computed/joined from other tables

---

## Step 6: Produce Change Plan

Output a clear, backend-scoped change plan:

```markdown
## Change Plan

**Endpoint**: PATCH /api/v1/orders/:id
**Service**: orders-service
**File to change**: src/controllers/order.controller.ts (line ~45)

### DB Changes
- [ ] Add column `estimated_delivery` (timestamp, nullable) to `orders` table
- [ ] Migration: `ALTER TABLE orders ADD COLUMN estimated_delivery TIMESTAMPTZ`

### Service Changes
- [ ] Query: include `estimated_delivery` in SELECT
- [ ] Response mapper: expose as `estimatedDelivery` (camelCase ISO string)
- [ ] Validation: accept `estimatedDelivery` in PATCH request body

### Contract Change
Before:
  { id, status, items, createdAt }

After:
  { id, status, items, createdAt, estimatedDelivery }

### What to share with FE
> "Added `estimatedDelivery: string | null` to the order response. ISO 8601 format. Available in orders/:id GET and returned in PATCH response."
```

---

## Guardrails

- Stay scoped to the **backend**. Do not suggest FE code changes.
- If the route is in a different service, say so clearly and stop — do not guess at that service's code.
- If DB schema is not provided, note it as an assumption in the change plan.
- If the user hasn't provided the network call yet, do not proceed — ask for it first.
- Always end by drafting the **one-liner message to send to the FE engineer** describing what changed in the API contract.