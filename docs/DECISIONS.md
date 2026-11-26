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
D-1 Does Pro hard-stop at 429 when its 50k/5M limits are hit, or may it overage-bill? Hard-stop with 429 (same as Free)
D-2 Is the Pro invoice a flat monthly fee only, or flat fee + metered usage cost? Flat fee; usage cost recorded for reporting
D-3 How do you want to handle token limit when tokens are only known after processing? simulate takes token counts up front, so the check is exact
D-4 GET /tenants/{tenant_id}: keep path-ID form or only /tenants/me? Both; path-ID returns 404 unless it is the caller's ID
D-5 Who can call POST /tenants? Public self-signup, or protected by an admin token? Public, rate-limited
D-6 When Stripe subscription is canceled, fall back to Free limits? Yes
D-7 Stripe test price: you create a Pro product/price in Stripe test dashboard and give me only the price ID (never secret keys)? Yes, ID goes in .env
D-8 Should rejected (402/429) requests be logged as rejected rows in a separate table? Log only, no ledger rows
D-9 Per-key rate limiting (requests/second) in v1, in-memory? Yes, simple in-memory limiter, documented as single-instance
D-10 Repo name and visibility: usage-metering-billing-engine, public? Yes

Also still needed from you: does the existing capstone brief give exact Pro price, token rates, or invoice rules? If so, paste it and I will reconcile this PRD against it.
