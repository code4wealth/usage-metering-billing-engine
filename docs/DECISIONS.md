Decision Log

Statuses: Accepted (locked), Proposed (my recommendation, needs owner OK), Open (needs owner input).

Accepted

ID Decision Reason
A-1 Modular monolith No need for microservices; simpler to build, test, defend
A-2 Stack: Python 3.12, FastAPI, PostgreSQL 16, SQLAlchemy 2.0, Alembic, Docker Compose, Pytest Fixed requirement
A-3 Payment provider: Stripe test mode + Stripe CLI Locked in capstone requirements
A-4 Scheduler: APScheduler Locked in capstone requirements
A-5 Billing period: UTC calendar month Locked in capstone requirements
A-6 Money: integer micro-dollars (BIGINT) Fixed requirement
A-7 Plans: Free 1,000 calls / 100k tokens; Pro 50,000 calls / 5M tokens; Pro price configurable Fixed requirement
A-8 Two-phase idempotency claim with 60s lease A single transaction hides in_progress from other connections, making FR-16 (409) unreachable
A-9 Lazy counter row: in transaction 2 re-check billing status, then INSERT usage_counters ON CONFLICT DO NOTHING, then conditional UPDATE; period_start computed once per request in UTC; close job independent, idempotent, never writes usage_counters Owner-approved; prevents false 429 on first request of a new month
A-10 Quota period and Retry-After always use the UTC calendar month, independent of Stripe's billing period; Stripe supplies payment status only Owner-approved; payment state must not move quota boundaries
A-11 Limits resolve from the tenant's current subscription and plan on every request; canceled falls back to Free; no counter reset on downgrade Owner-approved; one deterministic rule, no retroactive credits
A-12 After a zero-row counter UPDATE, read the counter row in the same transaction to choose the 429 code; if both limits fail, quota_calls_exceeded Owner-approved; deterministic error code without a second quota check
A-13 Completed idempotency rows expire after IDEMPOTENCY_RETENTION_HOURS=24; expires_at is a 60s lease while in_progress and a retention deadline once completed. Cleanup deletes only expired completed rows Owner-approved; replay window is bounded and documented
A-14 Invoices/payments mirror Stripe only (webhook-created, keyed by stripe_invoice_id); close job writes idempotent calendar-month usage snapshots, not invoices; Free tenants get no invoices Owner-approved; Stripe billing period differs from UTC calendar month, so (tenant_id, period_start) cannot key a mirrored invoice
A-15 Webhook handlers apply state-based updates only when Stripe event.created is strictly newer than target.last_event_created; webhook_events stores stripe_event_created; invoices and subscriptions store last_event_created Owner-approved; prevents out-of-order webhooks from overwriting newer state
D-1 Hard-stop with 429 for both Free and Pro, calls and tokens Owner-approved; no overage invoicing in v1
D-2 Flat monthly Pro fee from PRO_PRICE_MICRO_USD; usage cost reporting only Owner-approved; no exact price/rates in capstone brief
D-3 Token counts supplied up front by /v1/usage/simulate; exact pre-check Owner-approved; real inference needs reservation/true-up (future limitation)
D-4 Both /tenants/me and GET /tenants/{tenant_id}; path-ID is GET only and returns 404 not_found for another tenant's ID Owner-approved
D-5 Public self-signup for POST /tenants with a per-IP rate limit Owner-approved; limitations recorded: no email verification, duplicate-email 409 reveals registered emails, proxy client-IP handling must be explicit
D-6 Canceled subscriptions fall back to Free limits when customer.subscription.deleted arrives (end of paid period if cancel_at_period_end); no counter reset, so a tenant already over Free limits gets 429 until the next UTC month Owner-approved; named test case TC-D6-01 in PROGRESS.md
D-9 Per-key rate limit: RATE_LIMIT_PER_MINUTE, in-memory fixed window per key, single-instance only, plus a per-IP limit on POST /tenants; 429 rate_limit_exceeded with short Retry-After Owner-approved; separate from quota codes

Proposed (change if you disagree)

ID Decision Reason
P-1 SHA-256 for API key hashes, 256-bit random secrets Keys are high-entropy; bcrypt/argon2 add latency per request with no real benefit here
P-2 Quota via usage_counters + atomic conditional UPDATE Race-free, fast; ledger stays append-only
P-3 Token rates stored as µ$ per 1M tokens Integer math with no fractions
P-4 Half-up rounding once per usage record Single, testable rounding point
P-5 No users table in v1 No human login requirement
P-6 Account/read endpoints stay open during 402 states Customer must be able to see and fix billing

Open questions (need your answer)

ID Question My default if you don't decide
D-7 Stripe test price: you create a Pro product/price in Stripe test dashboard and give me only the price ID (never secret keys)? Yes, ID goes in .env
D-8 Should rejected (402/429) requests be logged as rejected rows in a separate table? Log only, no ledger rows
D-10 Repo name and visibility: usage-metering-billing-engine, public? Yes

Capstone brief: no exact price, rates or invoice rules in the brief; I will choose values. PRO_PRICE_MICRO_USD=20000000 ($20) is a labelled dev placeholder in .env.example; token rates remain unset until chosen.
