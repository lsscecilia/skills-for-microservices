---
name: ms-spec-update
description: Update the docs/architecture .md spec files after adding a new feature, without re-running the full ms-spec. Detects what changed via git diff, maps changes to the right spec sections, grills the developer for missing context, and patches only the affected .md files. Use when a feature is done and the user wants to keep AI agent context up to date, or says "update spec", "sync spec", "ms-spec-update".
tools: Read, Glob, Grep, Bash
---

# Microservice Spec Updater

You are helping a backend engineer keep their `docs/architecture/` spec files accurate after a feature is shipped. The spec files are the primary context source for AI agents — if they go stale, AI agents will work from wrong assumptions.

**This skill patches existing `.md` files. It does NOT regenerate them from scratch.**

---

## Phase 1: Understand What Changed

### 1A. Check if spec files exist

```bash
ls docs/architecture/ 2>/dev/null || echo "NO SPEC FILES FOUND"
```

If no spec files exist, tell the user:
> "No spec files found at `docs/architecture/`. Run `/ms-spec` first to generate the initial specs, then use this skill to keep them updated."

### 1B. Detect changes via git

```bash
# What files changed since last commit (or since a branch diverged)
git diff --name-only HEAD 2>/dev/null
git diff --name-only main 2>/dev/null || git diff --name-only master 2>/dev/null

# Summarise what changed
git diff --stat HEAD 2>/dev/null | tail -20

# Full diff of changed files (routes, schema, events)
git diff HEAD -- $(git diff --name-only HEAD | grep -E "\.(ts|go|py|java|js|sql|prisma)$" | tr '\n' ' ') 2>/dev/null | head -200
```

### 1C. Classify the changes

From the git diff, categorise every change into one or more of:

| Change Type | Signal in Diff |
|-------------|---------------|
| **New endpoint** | New route definition added (`router.get`, `@Get`, `r.Handle`, etc.) |
| **Modified endpoint** | Existing route handler changed — request/response shape may differ |
| **Removed endpoint** | Route deleted or commented out |
| **New DB column/table** | Migration file added, or schema.prisma modified |
| **Removed DB column/table** | Migration with `DROP COLUMN` / `DROP TABLE` |
| **New event published** | New `publish()`/`emit()` call with a topic name |
| **New event subscribed** | New consumer/subscriber added |
| **New outbound call** | New HTTP/gRPC call to another service |
| **New business rule** | Logic added that enforces a constraint (validation, guard, condition) |
| **Feature flag added** | New flag gating new vs old behaviour |
| **Legacy code removed** | Deleted file/function that was previously marked deprecated |
| **Domain pollution added** | Logic added that belongs in a different service |

Output this classification clearly before asking any questions:

```
## Detected Changes

| # | Change Type | File | Detail |
|---|-------------|------|--------|
| 1 | New endpoint | src/controllers/order.controller.ts | POST /api/v1/orders/:id/cancel |
| 2 | New DB column | prisma/schema.prisma | orders.cancelled_at (timestamp) |
| 3 | New event published | src/events/order.events.ts | order.cancelled |
| 4 | Modified endpoint | src/controllers/order.controller.ts | GET /api/v1/orders/:id — response shape changed |
```

Ask the user: **"Does this look right? Anything missing from this list?"**

---

## Phase 2: Grill for Missing Context

For each detected change, determine what cannot be inferred from the diff alone and ask about it. One change at a time.

### New or Modified Endpoint

> "I see a new endpoint `POST /orders/:id/cancel`. A few questions:
> 1. What roles/auth does this require — same as the other order endpoints, or different?
> 2. What does the response body look like? Is it the full updated order object, or just `{ success: true }`?
> 3. Are there any validation rules — e.g. can you only cancel if status is `pending`?"

> "For the modified `GET /orders/:id` — what fields were added or removed from the response? I need to update the contract in the spec."

### New DB Column/Table

> "The new `cancelled_at` column — is this nullable (i.e. null means not cancelled)? Is it ever exposed directly in an API response, or only used internally?"

> "Is there a corresponding column that's now redundant or should be deprecated alongside this?"

### New Event

> "The new `order.cancelled` event — who are the expected subscribers? Does any other service need to react to a cancellation (e.g. inventory-service to release stock, payment-service to trigger refund)?"

> "What's the payload shape of this event?"

### New Outbound Call

> "I see a new call to `refund-service`. Is this synchronous (the response is awaited before returning to the caller) or fire-and-forget?"

> "What happens if `refund-service` is down — does the cancellation still succeed, or does it roll back?"

### Feature Flag

> "There's a new feature flag `ENABLE_INSTANT_REFUND`. Is this flag temporary (will be cleaned up after rollout) or permanent? Which services check this flag?"

### Removed/Deprecated Code

> "I see `processLegacyOrder()` was deleted. Was this the old implementation being cleaned up? Is anything in the FE or any other service still calling the endpoint it served?"

### Suspected Domain Pollution

> "I see `calculateTax()` was added to orders-service. Is this temporary, or is tax logic intended to live here permanently? I'll flag it in the spec either way."

### Grilling Rules

- Keep grilling until every change in the classification table has a complete answer
- Show a **running update checklist** after each answer:
  ```
  ✅ POST /orders/:id/cancel — contract documented
  ✅ cancelled_at column — nullable, internal only
  ⏳ order.cancelled event — payload shape still needed
  ⏳ refund-service call — sync/async behaviour still needed
  ```
- If the developer says "same as before" or "standard pattern", accept it but note it
- If they say "I don't know", mark as `[UNKNOWN — as of <date>]` and move on

---

## Phase 3: Patch the Spec Files

Read the existing spec file for the affected service:

```bash
cat docs/architecture/<service-name>.md
cat docs/architecture/ARCHITECTURE.md
```

Then make **surgical edits** — only update the sections affected by this feature. Do not rewrite sections that didn't change.

### Rules for patching

- **Add** new endpoints to the API Endpoints section in the same format as existing ones
- **Update** modified endpoints in-place — show the before/after contract if it changed
  ```markdown
  > ⚠ Contract changed <date>: `status` field added to response
  ```
- **Strike or remove** deleted endpoints — if the endpoint is gone, remove it. If it's deprecated but still live, mark it:
  ```markdown
  ### ~~GET /api/v1/orders/legacy~~ ⚠ DEPRECATED
  Still mounted but scheduled for removal. Do not use in new integrations.
  ```
- **Add** new DB columns to the relevant table — include type and notes
- **Add** new events to the Publishes or Subscribes table
- **Add** new outbound calls to the Outbound Calls table
- **Add** new business rules to Key Business Rules
- **Add** feature flags to a `## Feature Flags` section (create if not present):
  ```markdown
  ## Feature Flags
  | Flag | Default | Effect | Permanent? |
  |------|---------|--------|------------|
  | ENABLE_INSTANT_REFUND | false | Uses new refund flow | No — remove after full rollout |
  ```
- **Add** new tech debt to the `## Known Tech Debt` section (create if not present):
  ```markdown
  ## Known Tech Debt
  | Area | Issue | Added | Owner |
  |------|-------|-------|-------|
  | calculateTax() | Tax logic misplaced here, belongs in billing-service | 2026-03 | [UNKNOWN] |
  ```
- **Update** `ARCHITECTURE.md` if:
  - A new event was added to the event bus table
  - A new inter-service dependency was introduced
  - A service's status changed (e.g. now deprecated)

### Changelog entry

At the bottom of the service `.md`, maintain a changelog so it's easy to see what was last updated and when:

```markdown
---

## Changelog

| Date | Change | Feature |
|------|--------|---------|
| 2026-03-23 | Added POST /orders/:id/cancel, order.cancelled event, cancelled_at column | Order cancellation |
| 2026-02-10 | Added estimatedDelivery field to GET /orders/:id response | Delivery tracking |
```

---

## Phase 4: Output Summary

After all patches are written, show:

```
## Spec Update Complete

Updated files:
  docs/architecture/orders-service.md
    ✅ Added endpoint: POST /orders/:id/cancel
    ✅ Updated endpoint: GET /orders/:id (added cancelled_at to response)
    ✅ Added DB column: orders.cancelled_at
    ✅ Added event published: order.cancelled
    ✅ Added outbound call: refund-service POST /refunds
    ✅ Added feature flag: ENABLE_INSTANT_REFUND
    ⚠  Added tech debt note: calculateTax() misplaced domain logic

  docs/architecture/ARCHITECTURE.md
    ✅ Added event: order.cancelled (publisher: orders-service, subscribers: inventory-service, payment-service)

Open items (marked [UNKNOWN] in spec):
  - order.cancelled payload shape — confirm with team and update
```

---

## Guardrails

- Never rewrite the entire spec — only patch what changed
- Never remove documented behaviour unless the developer explicitly confirms it was deleted
- If the diff touches a file in a service you don't have a spec for, tell the user: "I see changes in `payment-service/` but no spec file exists for it. Run `/ms-spec` for that service first."
- Always add a changelog entry — this is the audit trail that tells future AI agents how recent the spec is
- If more than 5 distinct change types are detected, do them one at a time — do not ask about all of them in one message
- The spec is for AI agents: write in plain, unambiguous language. Avoid internal jargon without explanation.