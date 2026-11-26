Roadmap (Phases 0–15)

Rule: finish a phase, owner verifies, then next phase. Each phase delivers: objective, concepts, files, commands, code, explanation, expected output, tests, common errors, checklist, and a Git commit.

Phase Objective Key concepts Main outputs Suggested commit
0 Understand and plan metering, quota, idempotency, tenancy PRD, architecture, schema, decisions docs: add PRD, architecture, database design
1 Dev environment venv, Docker, Git, project layout repo skeleton, Postgres container chore: project skeleton and tooling
2 FastAPI fundamentals routers, Pydantic, DI, exceptions /health, config, error handler feat: app factory, health endpoints
3 Database integration engine, sessions, models, Alembic, transactions models + first migration feat: db models and migrations
4 Tenant management isolation, repositories tenant endpoints feat: tenant registration and profile
5 API key auth hashing, prefix lookup, constant-time compare auth dependency, key endpoints feat: api key authentication
6 Subscriptions plans seeding, status, periods, billing gate plans + subscription endpoints, 402 gate feat: plans and subscription status
7 Metering engine ledger, token accounting simulate endpoint, usage_records feat: usage metering
8 Quota enforcement atomic UPDATE, row locks, 429 counters, concurrency test feat: atomic quota enforcement
9 Idempotency unique constraint, request hash, in-progress idempotency service feat: idempotency keys
10 Pricing + billing µ$ math, rounding, invoices, scheduler pricing service, period close job feat: pricing and invoicing
11 Stripe webhooks signature verify, event dedupe, state sync webhook endpoint, Checkout feat: stripe webhooks
12 Reporting pagination, aggregation usage + billing read APIs feat: usage and billing reports
13 Automated testing unit/integration/concurrency full pytest suite + report test: full suite
14 Dockerization Dockerfile, Compose, healthchecks docker compose up works build: docker compose
15 Documentation README, API docs, limitations final repo polish docs: final documentation

Post-project (on request): synopsis, SRS, use case diagram, data dictionary, internship report chapter, PPT, speaking script, 30+ viva questions grounded in the real implementation.

Phase 0 exit checklist

☐ Owner has read PRD and architecture
☐ Open questions D-1…D-10 answered
☐ Owner confirms scope
☐ Repo created on D: drive and PRD committed

Phase 1 preview (first implementation task)

Create D:\projects\usage-metering-billing-engine, git init, Python 3.12 venv, install base dependencies, start Postgres 16 in Docker, and confirm psql/connection works. No application code yet.
