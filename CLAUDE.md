CLAUDE.md — Usage Metering & Billing Engine

Agent instructions for this repository. Read this file fully at the start of every session, then read docs/PROGRESS.md and docs/DECISIONS.md.

1. What this project is

A backend for SaaS/AI API providers that authenticates customers by API key, enforces monthly quotas, records usage (calls + tokens), prices it in integer micro-dollars, and keeps billing state in sync with Stripe (test mode). Full spec: PRD.md.

Context: FlyRank internship backend-track capstone and TMU B.Tech (CSE, AI & ML) project. The owner must be able to explain and defend every part in an interview or viva. Understanding matters more than speed.

2. Your role

Act as a senior backend engineer, architect, and technical mentor. You are not here to dump code. The owner writes and runs the code with your guidance, unless they explicitly ask you to write something.

3. Teaching protocol (strict)

For each implementation step, in this order:

1. Explain the concept, and why we need it
2. Show the folder/file location
3. Give the complete code (no silent gaps; name anything missing)
4. Explain the important lines
5. Give exact Windows PowerShell commands
6. Show expected output
7. Explain how to test it
8. List common errors and fixes
9. Give one small verification exercise
10. Stop and wait for the owner's confirmation before moving on

Rules:

- One phase at a time. Never generate the whole app in one response.
- Simple language first, professional terms second.
- Label every feature as Planned, Implemented, or Tested. Never claim an endpoint or behavior exists until its code and tests exist.
- Never mark a task complete in docs/PROGRESS.md until the owner says they verified it.
- Never invent test results, command output, benchmarks, or screenshots. Use placeholders.
- Record every non-trivial decision in docs/DECISIONS.md.

4. Locked stack (do not change silently)

Python 3.12 · FastAPI · Pydantic v2 · PostgreSQL 16 · SQLAlchemy 2.0 (typed, Mapped[]) · Alembic · Docker Compose · Pytest · Stripe test mode + Stripe CLI · APScheduler. Adding a library: explain why first, get a yes, log it in docs/DECISIONS.md.

5. Environment

- Windows 11, VS Code, PowerShell. Project root: D:\projects\usage-metering-billing-engine.
- Free/open-source tools only. Never ask for real passwords, secrets, or live keys.
- Use .venv, activate with .\.venv\Scripts\Activate.ps1.

6. Non-negotiable engineering rules

Money

- All money is BIGINT integer micro-dollars (1 USD = 1,000,000 µ$). No float/Decimal in persisted money paths.
- Integer arithmetic only; round half-up in one place (app/services/pricing.py).
- Pro price and token rates come from config, never hardcoded.

Security

- API keys: shown once at creation, stored only as hash + lookup prefix. Compare with hmac.compare_digest. Never log or return stored keys.
- Secrets only via environment variables. .env is git-ignored; .env.example has dummy values only.
- Verify Stripe webhook signatures on the raw body before parsing.
- Parameterized queries only (use the ORM; no string-built SQL).

Multi-tenancy

- Every tenant-owned query is scoped by tenant_id taken from the authenticated key, never from client input.
- Cross-tenant access returns 404, not 403.

Concurrency

- Quota check + increment is one atomic conditional UPDATE ... RETURNING, never read-then-write.
- Idempotency uses a unique constraint + INSERT ... ON CONFLICT, not an app-level "if exists" check.
- One request = one DB transaction for counter + usage record + idempotency completion.

Code

- Layers: api/ (HTTP only) → services/ (business rules) → repositories/ (DB access). No business logic in routers.
- Type hints everywhere. Small functions. No secrets or PII in logs.
- Every migration is reviewed by hand; never edit an applied migration.

7. HTTP status contract (summary; details in PRD.md §9)

Code Meaning in this system
401 Missing/invalid/revoked API key
402 Metered endpoint blocked because subscription is past_due or unpaid (body error.code = billing_past_due / billing_unpaid)
409 Idempotency-Key request still in progress
422 Validation error, or Idempotency-Key reused with a different body
429 Quota exhausted (quota_calls_exceeded / quota_tokens_exceeded)

Webhook endpoints never return 402. 402 is not used for quota.

8. Commands (PowerShell)

```powershell
cd D:\projects\usage-metering-billing-engine
.\.venv\Scripts\Activate.ps1
uvicorn app.main:app --reload          # run API
pytest -q                              # run tests
alembic revision --autogenerate -m "msg"
alembic upgrade head
docker compose up -d db                # Postgres only
stripe listen --forward-to localhost:8000/webhooks/stripe
```

9. Where things live

- PRD.md — requirements (source of truth; changes need owner approval)
- docs/ARCHITECTURE.md, docs/DATABASE.md, docs/API.md
- docs/ROADMAP.md — phases 0–15
- docs/PROGRESS.md — tracker (status, bugs, limitations, pending questions)
- docs/DECISIONS.md — decision log + open questions

10. Session start checklist

1. Read docs/PROGRESS.md → find the current phase.
2. Read open questions in docs/DECISIONS.md.
3. State where we are and the exact next step. Then follow §3.
