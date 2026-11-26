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
A-10 Quota period and Retry-After always use the UTC calendar month, independent of Stripe's billing period; Stripe supplies payment status only Owner-approved; payment state must not move quota boundaries
A-11 Completed idempotency rows expire after IDEMPOTENCY_RETENTION_HOURS=24; expires_at is a 60s lease while in_progress and a retention deadline once completed. Cleanup deletes only expired completed rows Owner-approved; replay window is bounded and documented
D-1 Hard-stop with 429 for both Free and Pro, calls and tokens Owner-approved; no overage invoicing in v1
D-2 Flat monthly Pro fee from PRO_PRICE_MICRO_USD; usage cost reporting only Owner-approved; no exact price/rates in capstone brief
D-3 Token counts supplied up front by /v1/usage/simulate; exact pre-check Owner-approved; real inference needs reservation/true-up (future limitation)
D-4 Both /tenants/me and GET /tenants/{tenant_id}; path-ID is GET only and returns 404 not_found for another tenant's ID Owner-approved

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
D-5 Who can call POST /tenants? Public self-signup, or protected by an admin token? Public, rate-limited
D-6 When Stripe subscription is canceled, fall back to Free limits? Yes
D-7 Stripe test price: you create a Pro product/price in Stripe test dashboard and give me only the price ID (never secret keys)? Yes, ID goes in .env
D-8 Should rejected (402/429) requests be logged as rejected rows in a separate table? Log only, no ledger rows
D-9 Per-key rate limiting (requests/second) in v1, in-memory? Yes, simple in-memory limiter, documented as single-instance
D-10 Repo name and visibility: usage-metering-billing-engine, public? Yes

Capstone brief: no exact price, rates or invoice rules in the brief; I will choose values. PRO_PRICE_MICRO_USD=20000000 ($20) is a labelled dev placeholder in .env.example; token rates remain unset until chosen.
