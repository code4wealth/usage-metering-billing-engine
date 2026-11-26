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
- A billable request uses **two short transactions, not one**. The old rule "one request = one DB transaction for counter + usage record + idempotency completion" is retired: an `in_progress` row committed in the same transaction as the usage row is invisible to every other connection until commit, so FR-16 (409) would be unreachable.
- `Idempotency-Key` is required (8–128 chars); missing or malformed → 422 `validation_error` before any claim or write.
- Gate order: authenticate → **read-only billing gate** (402, no writes) → claim transaction → usage transaction, which **re-checks billing status inside the transaction** because Stripe webhooks can change status between steps.
- Claim transaction (commits immediately): `INSERT ... ON CONFLICT DO NOTHING` on `(tenant_id, idem_key)` with `state='in_progress'`, random `owner_token` (UUID4), `expires_at = now() + 60s`, and the SHA-256 request hash. On conflict, read the row: `completed` + same hash → replay with `Idempotency-Replayed: true`; **any state with a different hash → 422 `idempotency_key_reuse`** (never take over, never 409); `in_progress` + same hash + not expired → 409; `in_progress` + same hash + expired → conditional takeover `UPDATE ... WHERE state='in_progress' AND expires_at < now() RETURNING` with a new owner and fresh lease (zero rows → 409). If the insert conflicts but a follow-up read finds no row (a 402/429 release deleted it between statements), retry the whole claim step at most **3 attempts**, then return 409.
- Usage transaction (all-or-nothing): re-check billing gate → lazy counter `INSERT ... ON CONFLICT DO NOTHING` → atomic quota `UPDATE ... RETURNING` → ledger insert → claim completion. A zero-row update reads the counter in the same transaction to pick the 429 code (calls wins if both fail). The completion `UPDATE ... SET state='completed', completed_at=now(), expires_at=completed_at + IDEMPOTENCY_RETENTION_HOURS, response_status, response_body WHERE idempotency_id=:id AND owner_token=:owner AND state='in_progress'` must affect exactly one row; zero rows → roll back the whole transaction and return 409. An `IntegrityError` on the ledger's unique `(tenant_id, idempotency_key)` also maps to 409 `idempotency_in_progress`; a planned FR-17 test must prove no second usage row exists.
- On **402 or 429**: roll back the usage transaction, then release the claim with a committed `DELETE ... WHERE idempotency_id=:id AND owner_token=:owner`. No ledger row or counter increment survives a rejected request. A 401 (bad key) happens before claiming, so no claim exists.
- **Crash recovery:** if the process dies after claiming but before completing, the 60-second lease expires and a later request may take over. Takeover replaces `owner_token` and refreshes `expires_at`; a stale worker's completion affects zero rows and rolls back. The ledger's unique `(tenant_id, idempotency_key)` constraint is the second, database-level guard against duplicate usage.
- `60s` is `IDEMPOTENCY_CLAIM_TTL_SECONDS` in `.env.example` (default 60 in v1). All lease times use PostgreSQL `now()`, never application-local clocks.
- `expires_at` has two meanings: a 60-second lease while `in_progress`, and a retention deadline once `completed` (`IDEMPOTENCY_RETENTION_HOURS`, default 24). The cleanup job deletes **only expired completed rows**; it never deletes an in-progress claim.
- Replays never write to the ledger or the counters. Store only the request hash, response status and response snapshot; never log plaintext keys or raw bodies. A retry after the retention window is a new request.

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
