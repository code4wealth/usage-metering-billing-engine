Architecture (Planned, nothing implemented yet)

Style: modular monolith. One deployable FastAPI app, one PostgreSQL database. No extra microservices, because there is no scaling or team-boundary reason for them yet.

1. System context

DIAGRAM SOURCE (Mermaid)

```mermaid
flowchart LR
    C[API Customer App] -->|X-API-Key, Idempotency-Key| S[Metering & Billing Engine]
    S <-->|SQL| DB[(PostgreSQL 16)]
    S -->|Checkout / Customer API| ST[Stripe test mode]
    ST -->|Webhooks + Stripe-Signature| S
    O[Operator] -->|logs, health, CLI| S
```

Plain words: customers call our API; we store everything in Postgres; Stripe tells us when payments succeed or fail.

2. Components and responsibilities

Component Responsibility Module
1 FastAPI app Routing, validation, error mapping, OpenAPI docs app/main.py, app/api/
2 Authentication Create/verify/revoke API keys, resolve tenant services/auth.py, api/deps.py
3 Tenant management Register, read, update tenant services/tenants.py
4 Subscription management Plans, subscription status, billing period services/subscriptions.py
5 Usage metering Write usage ledger rows services/metering.py
6 Quota enforcement Atomic check-and-increment services/quota.py
7 Pricing engine Tokens → µ$ (pure functions) services/pricing.py
8 Billing Period close, invoices, payment mirror services/billing.py
9 Idempotency Claim/complete/replay keys services/idempotency.py
10 Webhook handler Verify, dedupe, dispatch Stripe events services/webhooks.py
11 PostgreSQL System of record, constraints db/, migrations/
12 Reporting Read-only usage/billing queries services/reporting.py

Layering: api → services → repositories → db. Routers never touch SQL.

3. High-level architecture

DIAGRAM SOURCE (Mermaid)

```mermaid
flowchart TB
    subgraph App[FastAPI modular monolith]
        R[api routers] --> M[middleware: request-id, logging, rate limit]
        M --> D[deps: auth + billing gate]
        D --> SV[services]
        SV --> RP[repositories]
        SCH[APScheduler jobs] --> SV
    end
    RP --> PG[(PostgreSQL)]
    SV --> STR[Stripe SDK]
```

4. Level 0 DFD

DIAGRAM SOURCE (Mermaid)

```mermaid
flowchart LR
    Customer([Customer]) -->|API request + key| P0((0. Metering & Billing System))
    P0 -->|response / 401 / 402 / 429| Customer
    P0 <-->|usage, tenants, invoices| Store[(Database)]
    P0 -->|checkout request| Stripe([Stripe])
    Stripe -->|payment events| P0
```

5. Level 1 DFD

DIAGRAM SOURCE (Mermaid)

```mermaid
flowchart TB
    Cust([Customer]) -->|key| P1((1 Authenticate))
    P1 -->|tenant| P2((2 Check billing + quota))
    P2 -->|allowed| P3((3 Idempotency claim))
    P3 -->|new| P4((4 Record usage + price))
    P4 --> P5((5 Respond))
    P5 --> Cust
    P1 <--> D1[(api_keys)]
    P2 <--> D2[(subscriptions, usage_counters)]
    P3 <--> D3[(idempotency_records)]
    P4 --> D4[(usage_records)]
    Stripe([Stripe]) -->|event| P6((6 Webhook handler))
    P6 <--> D5[(webhook_events)]
    P6 --> D2
    P6 --> D6[(invoices, payments)]
```

6. Request lifecycle (POST /v1/usage/simulate)

DIAGRAM SOURCE (Mermaid)

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API
    participant DB as PostgreSQL
    C->>A: POST + X-API-Key + Idempotency-Key
    A->>DB: lookup key by prefix, compare hash
    alt invalid / revoked
        A-->>C: 401
    end
    A->>DB: load subscription
    alt past_due / unpaid
        A-->>C: 402
    end
    A->>DB: BEGIN
    A->>DB: INSERT idempotency (tenant,key) ON CONFLICT DO NOTHING
    alt conflict, completed, same hash
        A-->>C: replay stored response
    else conflict, different hash
        A-->>C: 422
    else conflict, in progress
        A-->>C: 409
    end
    A->>DB: UPDATE usage_counters ... WHERE within limits RETURNING
    alt 0 rows
        A->>DB: ROLLBACK
        A-->>C: 429 + Retry-After
    end
    A->>DB: INSERT usage_records (cost µ$)
    A->>DB: UPDATE idempotency (completed, response snapshot)
    A->>DB: COMMIT
    A-->>C: 200 + usage summary
```

Why one transaction: if anything fails, counter, record, and idempotency row all roll back together, so retries are safe.

7. Authentication flow

DIAGRAM SOURCE (Mermaid)

```mermaid
flowchart TD
    A[Receive X-API-Key] --> B{Header present and well-formed?}
    B -- no --> X[401]
    B -- yes --> C[Split prefix and secret]
    C --> D[Find key row by prefix]
    D --> E{Found and not revoked?}
    E -- no --> X
    E -- yes --> F[SHA-256 of presented key]
    F --> G{hmac.compare_digest with stored hash}
    G -- no --> X
    G -- yes --> H[Set tenant_id in request context, update last_used_at]
```

8. Subscription lifecycle

DIAGRAM SOURCE (Mermaid)

```mermaid
stateDiagram-v2
    [*] --> active_free: tenant registered
    active_free --> checkout_pending: POST /subscriptions
    checkout_pending --> active_pro: checkout.session.completed / invoice.paid
    active_pro --> past_due: invoice.payment_failed
    past_due --> active_pro: invoice.paid
    past_due --> unpaid: retries exhausted (subscription.updated)
    unpaid --> active_pro: invoice.paid
    active_pro --> canceled: subscription.deleted
    past_due --> canceled: subscription.deleted
    unpaid --> canceled: subscription.deleted
    canceled --> active_free: fall back to Free limits
```

9. Billing workflow

DIAGRAM SOURCE (Mermaid)

```mermaid
flowchart TD
    U[usage_records during period] --> J[APScheduler: period close, UTC month end]
    J --> S[Sum calls, tokens, cost µ$ per tenant]
    S --> I[Create invoice row: period, totals, base fee µ$ from config]
    I --> P[Stripe webhook invoice.paid / payment_failed]
    P --> ST[Update invoice + payment + subscription status]
```

10. Webhook flow

DIAGRAM SOURCE (Mermaid)

```mermaid
flowchart TD
    W[POST /webhooks/stripe] --> V{Signature valid on raw body?}
    V -- no --> E400[400]
    V -- yes --> DUP{event.id already in webhook_events?}
    DUP -- yes --> OK200[200 no-op]
    DUP -- no --> INS[INSERT webhook_events]
    INS --> H[Dispatch handler by type, same transaction]
    H --> OK[200]
    H -- handler error --> F[Mark failed, return 500 so Stripe retries]
```

11. Folder structure

usage-metering-billing-engine/
├── app/
│   ├── main.py            # app factory, router mounting
│   ├── core/              # config (pydantic-settings), security, logging, errors
│   ├── api/               # routers + dependencies (HTTP only)
│   ├── models/            # SQLAlchemy 2.0 models
│   ├── schemas/           # Pydantic request/response models
│   ├── services/          # business rules
│   ├── repositories/      # DB access
│   ├── middleware/        # request-id, rate limit
│   ├── db/                # engine, session, base
│   └── utils/             # time/period helpers, money helpers
├── migrations/            # Alembic
├── tests/                 # unit/, integration/, concurrency/
├── docs/
├── scripts/               # seed plans, dev helpers
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── .env.example
└── README.md
