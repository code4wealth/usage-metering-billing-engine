API Contract (Planned; no endpoint is implemented until code + tests exist)

Base URL (local): http://localhost:8000. JSON only. Auth header: X-API-Key. Error shape:

{"error": {"code": "quota_calls_exceeded", "message": "Monthly API call limit reached", "request_id": "..."}}

Method + URL Auth Purpose Success Errors
1 GET /health None Liveness 200 n/a
2 GET /health/ready None DB connectivity 200 503
3 POST /tenants None (rate-limited) Register tenant, create Free subscription, return first key once 201 409 email exists, 422
4 GET /tenants/me Key Own profile 200 401
5 GET /tenants/{tenant_id} Key Own profile by ID 200 401, 404 if not caller's ID
6 PATCH /tenants/me Key Update name/email 200 401, 409, 422
7 POST /api-keys Key Create extra key (plaintext shown once) 201 401, 422
8 GET /api-keys Key List keys (prefix, label, status only) 200 401
9 DELETE /api-keys/{id} Key Revoke key 204 401, 404
10 GET /plans None List plans and limits 200 n/a
11 POST /subscriptions Key Start Stripe Checkout for Pro; returns checkout URL 201 401, 409 already Pro, 422
12 GET /subscriptions/current Key Plan, status, period 200 401
13 POST /v1/usage/simulate Key + Idempotency-Key One billable call 200 401, 402, 409, 422 (validation_error / idempotency_key_reuse), 429
14 GET /usage Key Current-period used vs limit 200 401
15 GET /usage/history Key Paginated usage records 200 401, 422
16 GET /billing/invoices Key Invoice list 200 401
17 GET /billing/summary Key Current-period usage cost so far (reporting only, not charged) + status 200 401
18 POST /webhooks/stripe Stripe-Signature Provider events 200 400 bad signature, 500 handler failure (Stripe retries)

Key endpoints in detail

POST /tenants

Request: {"name": "Acme Labs", "email": "dev@acme.example"} Validation: name 1–100 chars; valid email; unique email. Response 201: {"tenant_id": "...", "api_key": "ume_xxxxxxxx_...", "plan": "free", "note": "Store this key now; it will not be shown again."} DB: insert tenant, subscription (Free, active, current UTC month), api_key (hash) in one transaction.

POST /v1/usage/simulate

Headers: X-API-Key, Idempotency-Key (required, 8–128 chars; missing/malformed → 422 validation_error). Request: {"input_tokens": 120, "output_tokens": 380} (each 0–1,000,000; at least one > 0) Response 200:

{"request_id": "...", "total_tokens": 500, "cost_micro_usd": 0,
 "usage": {"api_calls_used": 1, "api_calls_limit": 1000, "tokens_used": 500, "tokens_limit": 100000}}

(cost_micro_usd depends on configured rates.) Status: 401 invalid_api_key · 402 billing_past_due/billing_unpaid (read-only pre-check and in-transaction re-check) · 409 idempotency_in_progress (fresh in-progress claim, failed stale takeover, or ledger unique (tenant_id, idempotency_key) conflict) · 422 validation_error (including missing Idempotency-Key) / idempotency_key_reuse (different hash in any state, even in_progress or expired) · 429 quota_calls_exceeded/quota_tokens_exceeded with Retry-After. D-1: hard-stop 429 applies to both Free and Pro. Replay: same key + same body returns the stored response with Idempotency-Replayed: true; no new usage. Completed responses are retained IDEMPOTENCY_RETENTION_HOURS (default 24); after cleanup the same key is a new request. DB ops: see request lifecycle in ARCHITECTURE.md §6.

GET /usage/history

Query: limit (1–100, default 20), cursor (keyset on requested_at, id), optional from, to (ISO UTC). Response: {"items": [...], "next_cursor": "..."}. Always filtered by caller's tenant_id.

POST /webhooks/stripe

Reads raw body; verifies signature with STRIPE_WEBHOOK_SECRET; inserts into webhook_events (unique event_id); dispatches handler; 200 on success or duplicate. Never returns 402.
