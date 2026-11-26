Product Requirements Document (PRD)

Usage Metering & Billing Engine for SaaS and AI Applications

Version 0.1 (Draft, pending owner confirmation of open questions in docs/DECISIONS.md)
Status Phase 0: Planning. Nothing below is implemented yet.
Payment provider Stripe (test mode) with Stripe CLI, taken from the project's locked capstone requirements

Status labels used everywhere: Planned (designed, no code) → Implemented (code exists) → Tested (automated tests pass and owner verified). Every requirement below is currently Planned.

1. Purpose

SaaS and AI API providers need to know who used what, stop customers who exceed their plan, and bill accurately. This system is a backend that:

Authenticates API customers (tenants) by API key
Checks subscription/billing state and monthly quota
Records each billable request exactly once (idempotent)
Prices usage in integer micro-dollars
Syncs subscription/payment state from Stripe webhooks
Exposes usage and billing reports

Analogy: a prepaid mobile recharge system. Plan = recharge pack, quota = remaining data/calls, billing status = whether the account is active.

2. Goals and non-goals

Goals

Correct, auditable usage and cost records
Quota that cannot be bypassed by concurrent requests
No duplicate usage/billing from retries or duplicate webhooks
Strict tenant isolation
Learning outcome: owner can explain and defend every design choice

Non-goals (v1)

Real AI inference (usage is simulated via POST /v1/usage/simulate)
Admin dashboard/UI, multi-user logins per tenant, tax/VAT, refunds, coupons
Microservices (modular monolith only)
Live-mode payments

3. Users and personas

Persona Needs
API customer (tenant) Get a key, call API, see usage/limits/invoices, upgrade to Pro
Platform operator (owner/dev) Trust the numbers, observe failures, run tests
Evaluator (FlyRank reviewer / TMU guide) Clear docs, working Compose setup, test evidence

4. Fixed requirements (from the starting spec; not to be changed silently)

4.1 Stack

Python 3.12, FastAPI, Pydantic v2, PostgreSQL 16, SQLAlchemy 2.0, Alembic, Docker Compose, Pytest, REST, Stripe test mode + Stripe CLI, APScheduler.

4.2 Plans

Plan Calls / month Tokens / month Price
Free 1,000 100,000 None
Pro 50,000 5,000,000 Configurable (PRO_PRICE_MICRO_USD, plus Stripe Price ID in env). No default invented.

Billing period: UTC calendar month. Quota period and Retry-After use the UTC calendar month, independent of Stripe's billing period; Stripe supplies payment status only (A-10).

4.3 Authentication

API key via X-API-Key
Only hashed keys stored; plaintext shown once at creation
Keys never in logs or responses (after creation response)
Secure generation + constant-time verification; revoked keys rejected

4.4 Idempotency

Idempotency-Key header on billable requests. Same key + same body → same stored response, no second usage record. Enforced by DB unique constraint inside the transaction.

4.5 Usage record fields

Tenant ID, request ID, endpoint, timestamp, call count, input tokens, output tokens, total tokens, cost (µ$), idempotency key, status.

4.6 Money

Integer micro-dollars (BIGINT). No floating-point money.

4.7 Quota

Track monthly usage; check calls and tokens; return 429 when Free-plan quota is exhausted.

4.8 Billing status

active → full access. past_due / unpaid → restricted; 402 on metered endpoints (see §9 for exact semantics).

5. Functional requirements

IDs are stable; reference them in commits and tests.

Tenants and keys

FR-1 Register a tenant (name, email). Auto-create Free subscription. Return first API key once.
FR-2 Tenant can fetch/update own profile only.
FR-3 Create additional API keys; list keys (prefix only); revoke a key.
FR-4 Revoked/unknown/missing key → 401.

Plans and subscriptions

FR-5 GET /plans lists Free and Pro with limits (and configured Pro price).
FR-6 Tenant has exactly one current subscription with a status and billing period.
FR-7 Upgrade to Pro via Stripe Checkout (test mode); status updates arrive via webhooks.
FR-8 Subscription statuses: active, past_due, unpaid, canceled.

Metering and quota

FR-9 POST /v1/usage/simulate accepts input/output token counts and records one billable call.
FR-10 Total tokens = input + output; stored explicitly.
FR-11 Before recording, check billing status, then atomically check-and-increment calls and tokens for the current period.
FR-12 Quota exhausted → 429 with Retry-After set to the next UTC calendar-month start (A-10); no usage record created for rejected requests (rejections are logged as rejected events only if decision D-8 is accepted). `period_start` is computed once per request in UTC. In transaction 2 the order is: re-check billing status → lazy `INSERT ... ON CONFLICT DO NOTHING` of the counter row → conditional `UPDATE ... RETURNING` (A-9). After a zero-row UPDATE, read the counter row **in the same transaction** to choose the code: calls exhausted → `quota_calls_exceeded`; otherwise tokens exhausted → `quota_tokens_exceeded`; if both are exhausted, `quota_calls_exceeded` wins (A-12).
FR-13 Concurrent requests must never let usage exceed limits.

Idempotency

FR-14 (Claim and ownership) `Idempotency-Key` is required (8–128 chars); missing or malformed → 422 `validation_error` before any claim. After a read-only billing gate (402), every billable request creates a claim in a short, separately committed transaction: `INSERT ... ON CONFLICT DO NOTHING` on `(tenant_id, Idempotency-Key)` with `state='in_progress'`, a random `owner_token`, a SHA-256 hash of canonical method+path+body, and `expires_at = now() + 60s`. If the insert conflicts but the row is gone when read (released by a 402/429 in between), retry the claim at most 3 times, then return 409. No billing, quota, or ledger write happens in the claim transaction.

FR-15 (Replay and reuse) A completed row with the same hash returns its stored status/body with `Idempotency-Replayed: true` and creates no usage record. A different hash on **any** state, including `in_progress` or expired, returns 422 `idempotency_key_reuse` and never takes over and never returns 409. Raw bodies and keys are not logged; only the hash and response snapshot are stored.

FR-16 (Concurrency) An `in_progress`, unexpired claim with the same hash returns 409 `idempotency_in_progress` without touching quota or the ledger. An expired claim with the same hash may be taken over only by a conditional `UPDATE ... WHERE state='in_progress' AND expires_at < now() RETURNING` that replaces the owner token and refreshes the 60-second lease; zero rows means another worker won → 409. An `IntegrityError` on the ledger's unique `(tenant_id, idempotency_key)` also maps to 409 `idempotency_in_progress`, with a test proving no second usage row is created.

FR-17 (Completion, release, crash safety) Billing status is re-checked inside the usage transaction, which then performs the atomic counter increment, ledger insert, and claim completion as one all-or-nothing unit. Completion sets `state='completed'`, `completed_at=now()` and `expires_at = completed_at + IDEMPOTENCY_RETENTION_HOURS` (default 24) in that same transaction; it is conditional on the current `owner_token` and must affect exactly one row, otherwise the transaction rolls back. On 402 or 429 the claim is released with a committed `DELETE ... WHERE owner_token=:owner`, with no counter change or ledger row. A crash leaves a claim whose 60-second lease can be taken over; the owner-token condition plus the ledger unique constraint guarantee at most one usage record per key. `expires_at` is the lease while `in_progress` and the retention deadline once `completed`; cleanup deletes only expired completed rows, so a retry after the retention window is a new request. Idempotency keys are scoped per tenant.

Pricing and billing

FR-18 Cost computed with integer math from configurable per-million-token rates (input and output separately).
FR-19 Rounding: half-up to whole micro-dollar, done once per usage record.
FR-20 Period-end job (APScheduler) writes an idempotent UTC calendar-month `usage_snapshot` per tenant (calls, tokens, µ$ cost) keyed by `(tenant_id, period_start)`, via upsert. It sums `usage_records` grouped by `usage_records.period_start` (not `requested_at`), never creates invoices, and never writes `usage_counters`; reruns are safe and the job is independent of metering (A-9, A-14).
FR-21 Invoices/payments mirror Stripe state via webhooks only, keyed by `stripe_invoice_id`; Stripe is authoritative for status and amounts. Free tenants have no invoices (A-14).

Webhooks (Stripe)

FR-22 POST /webhooks/stripe verifies Stripe-Signature on the raw body.
FR-23 Each Stripe event.id processed at most once (unique `(provider, event_id)` in `webhook_events`). The event's `created` timestamp is stored as `stripe_event_created`; a handler applies an update only when the event is strictly newer than the target's `last_event_created`, otherwise it stores the event as `ignored` and returns 200 (A-15).
FR-24 Handled events (initial set): checkout.session.completed, customer.subscription.updated, customer.subscription.deleted, invoice.paid, invoice.payment_failed. Handlers are state-based, not delta-based, and compare `stripe_event_created` with `last_event_created` on `invoices`/`subscriptions` before writing (A-15). `customer.subscription.deleted` takes effect when the event arrives (at the end of the paid period if `cancel_at_period_end`).
FR-25 Unknown event types are stored and acknowledged (200), not errored.

Reporting

FR-26 GET /usage current-period totals vs limits.
FR-27 GET /usage/history paginated usage records (own tenant only).
FR-28 GET /subscriptions/current, GET /billing/invoices, GET /billing/summary.

6. Non-functional requirements

ID Requirement
NFR-1 Security Hashed keys, env-based secrets, signature-verified webhooks, parameterized SQL, safe logging
NFR-2 Correctness DB constraints (unique, FK, CHECK ≥ 0) back every business invariant
NFR-3 Concurrency Atomic conditional updates; proven by a concurrent test
NFR-4 Reliability Idempotent writes; transactional usage recording; migrations reproducible
NFR-5 Observability Structured logs with request ID; /health and /health/ready
NFR-6 Maintainability Layered modules; ≥ 80% coverage on services; type hints
NFR-7 Portability docker compose up runs API + Postgres on Windows 11
NFR-8 Cost Free/open-source tooling only
NFR-9 Rate limiting Basic per-key request rate limit (decision D-9)

7. Architecture summary

Modular monolith: FastAPI app, service layer, repository layer, PostgreSQL. Details and diagrams: docs/ARCHITECTURE.md. Schema: docs/DATABASE.md.

8. Key design decisions (summary; full log in docs/DECISIONS.md)

Quota counter table (usage_counters) updated atomically; usage_records is the append-only ledger. Counters are derivable from the ledger for audit.
Quota check is one SQL statement: UPDATE ... SET calls = calls + 1, tokens = tokens + :t WHERE calls + 1 <= :call_limit AND tokens + :t <= :token_limit RETURNING .... Zero rows → 429.
Idempotency via unique (tenant_id, key) plus request hash and response snapshot.
Money in µ$ BIGINT; token rates stored as µ$ per 1,000,000 tokens so no fractions are needed.
API keys: random 256-bit secret, SHA-256 hash stored (keys are high-entropy, so a slow password hash is unnecessary), unique lookup prefix, constant-time compare.
No users table in v1: tenant is the account.

9. API status-code contract

Status When error.code
200/201 Success n/a
401 Missing/invalid/revoked key invalid_api_key
402 Metered endpoint, subscription past_due billing_past_due
402 Metered endpoint, subscription unpaid billing_unpaid
404 Not found or other tenant's resource not_found
409 Same Idempotency-Key still processing idempotency_in_progress
422 Invalid body, or key reused with different body validation_error / idempotency_key_reuse
429 Calls exhausted / tokens exhausted quota_calls_exceeded / quota_tokens_exceeded
429 Rate-limit exceeded (per key or per IP) rate_limit_exceeded

Rules:

402 means "billing problem, fix payment". 429 quota means "quota used up, wait or upgrade"; 429 rate_limit_exceeded means "too many requests, slow down" with a short Retry-After and no quota/usage side effects. They are never interchangeable.
Limits are resolved from the tenant's current subscription and plan on every request; canceled falls back to Free limits and there is no counter reset on downgrade (A-11).
If both quota limits fail, `quota_calls_exceeded` is returned (A-12).
402 applies only to metered endpoints (/v1/*). Read-only account endpoints (/usage, /billing/*, /subscriptions/current) stay accessible so a customer can see why they are blocked and pay.
Webhook endpoints never return 402.
canceled subscriptions fall back to Free limits when `customer.subscription.deleted` arrives (at the end of the paid period if `cancel_at_period_end`); because there is no counter reset, a tenant already over Free limits gets 429 until the next UTC month (D-6, A-11).
Error body shape: {"error": {"code": "...", "message": "...", "request_id": "..."}}.

10. Endpoints (summary; full contract in docs/API.md)

Method Path Auth Purpose
GET /health, /health/ready None Liveness/readiness
POST /tenants None (rate-limited) Register tenant + first key
GET/PATCH /tenants/me Key Own profile
GET /tenants/{tenant_id} Key Own profile by ID (GET only) 200 401, 404 not_found
POST/GET/DELETE /api-keys Key Manage keys
GET /plans None List plans
POST /subscriptions Key Start Pro checkout
GET /subscriptions/current Key Current subscription
POST /v1/usage/simulate Key + Idempotency-Key Billable test call
GET /usage, /usage/history Key Usage reports
GET /billing/invoices, /billing/summary Key Billing reports
POST /webhooks/stripe Stripe signature Provider events

Note: the path-ID variant is GET only and returns 404 `not_found` for any ID other than the caller's; PATCH remains /tenants/me only (D-4).

11. Acceptance criteria

All unchecked until the owner verifies with real test runs.

☐ 1. Tenant can be registered
☐ 2. API key generated securely, shown once, stored hashed
☐ 3. X-API-Key authentication works; revoked key rejected
☐ 4. Tenant receives Free plan on signup
☐ 5. Requests create usage records
☐ 6. API calls counted correctly
☐ 7. Tokens counted correctly
☐ 8. Cost calculated exactly in µ$ (known-answer tests)
☐ 9. Free limits enforced with 429
☐ 10. Duplicate requests create no duplicate usage/billing
☐ 11. N concurrent requests cannot exceed quota
☐ 12. past_due/unpaid restricted with correct 402
☐ 13. Stripe webhooks verified, deduplicated, processed
☐ 14. Usage retrievable via API
☐ 15. Tenant A cannot read tenant B's data
☐ 16. Automated tests for all workflows above
☐ 17. docker compose up runs the full app

12. Risks

Risk Mitigation
Quota race conditions Atomic conditional UPDATE + concurrency test
Double webhook delivery Unique stripe_event_id + idempotent handlers
Webhook arrives out of order Compare Stripe event timestamps; handlers are state-based, not delta-based
Float money bugs µ$ integers, lint/test guard against float in pricing
Scope creep Phase gates; owner confirmation required per phase
Idempotency crash mid-request in_progress record is committed with the usage transaction; stale locks expire

13. Milestones

Phases 0–15 in docs/ROADMAP.md. Live status in docs/PROGRESS.md.

14. Deliverables

Public GitHub repo, README, API docs, ER/architecture diagrams, test report, limitations and future scope; TMU synopsis/SRS/viva material later (Phase 15+ support).

15. Out of scope / future

Usage-based overage invoicing, multiple currencies, admin UI, per-endpoint pricing, Redis rate limiting, outbox pattern for emails, partitioned usage tables.

v1 limitation (D-3): token counts are supplied up front by `/v1/usage/simulate`, so the quota check is exact. Real AI inference does not know output tokens before processing; a production version would need a reservation/true-up design, which is future work.
v1 signup limitation (D-5): `POST /tenants` is public and rate-limited per IP. There is no email verification; a duplicate-email 409 reveals that an email is already registered; client IP handling behind a proxy must be made explicit before relying on the per-IP limit.
