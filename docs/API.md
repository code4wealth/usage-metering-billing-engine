API Contract (Planned; no endpoint is implemented until code + tests exist)

Base URL (local): http://localhost:8000. JSON only. Auth header: X-API-Key. Error shape:

{"error": {"code": "quota_calls_exceeded", "message": "Monthly API call limit reached", "request_id": "..."}}

Method + URL Auth Purpose Success Errors
1 GET /health None Liveness 200 n/a
2 GET /health/ready None DB connectivity 200 503 readiness_db_unavailable
3 POST /tenants None (rate-limited) Register tenant, create Free subscription, return first key once 201 409 tenant_email_exists, 422 validation_error
4 GET /tenants/me Key Own profile 200 401
5 GET /tenants/{tenant_id} Key Own profile by ID 200 401, 404 if not caller's ID
6 PATCH /tenants/me Key Update name/email 200 401, 409 tenant_email_exists, 422 validation_error
7 POST /api-keys Key Create extra key (plaintext shown once) 201 401, 422
8 GET /api-keys Key List keys (prefix, label, status only) 200 401
9 DELETE /api-keys/{id} Key Revoke key 204 401, 404, 409 last_key_required
10 GET /plans None List plans and limits 200 n/a
11 POST /subscriptions Key Start/reuse Stripe Checkout for Pro; returns URL 201 new / 200 existing open session 401, 409 subscription_already_pro, 422 validation_error, 503 checkout_unavailable
12 GET /subscriptions/current Key Plan, status, period 200 401
13 POST /v1/usage/simulate Key + Idempotency-Key One billable call 200 401, 402, 409, 422 (validation_error / idempotency_key_reuse), 429
14 GET /usage Key Current-period used vs limit 200 401
15 GET /usage/history Key Paginated usage records 200 401, 422 validation_error
16 GET /billing/invoices Key Invoice list 200 401
17 GET /billing/summary Key Current-period usage cost so far (reporting only, not charged) + status 200 401
18 POST /webhooks/stripe Stripe-Signature Provider events 200 400 invalid_signature, 503 webhook_unavailable, 500 handler failure (Stripe retries)

Key endpoints in detail

POST /tenants

Request: {"name": "Acme Labs", "email": "dev@acme.example"} Validation: name 1–100 chars; valid email (lowercased before storage/comparison); unique email. 409 `tenant_email_exists`: a unique-constraint `IntegrityError` on lowercased email maps to 409, on PATCH `/tenants/me` as well as registration (D-5: this reveals registration). Response 201: {"tenant_id": "...", "api_key": "ume_xxxxxxxx_...", "plan": "free", "note": "Store this key now; it will not be shown again."} DB: insert tenant, subscription (Free, active, current UTC month), api_key (hash) in one transaction.

DELETE /api-keys/{id}

Revokes a key by id for the caller's tenant only (404 for another tenant's id). 409 `last_key_required` when it is the tenant's last active key; create another key first.

POST /subscriptions

Auth: X-API-Key; no body required. First call creates the Stripe customer, saves `tenants.stripe_customer_id`, creates a Checkout session with `client_reference_id` and `metadata.tenant_id`, sets status `checkout_pending`, stores `stripe_checkout_session_id`, and returns 201 with the URL. While pending, the session is retrieved from Stripe and its URL returned with 200 only when `status='open'`; if it is not open, a new session replaces it and the old open session is expired. 401 `invalid_api_key`. 409 `subscription_already_pro` when already active Pro. 422 `validation_error` for invalid input only. 503 `checkout_unavailable` for Stripe failures or missing Stripe config (error body never contains secrets). `checkout.session.completed` confirms only with `payment_status='paid'`; `checkout.session.expired` returns the tenant to active Free. While pending, limits come from the Free plan (A-11, A-16).

POST /v1/usage/simulate

Headers: X-API-Key, Idempotency-Key (required, 8–128 chars; missing/malformed → 422 validation_error). Request: {"input_tokens": 120, "output_tokens": 380} (each 0–1,000,000; at least one > 0) Response 200:

{"request_id": "...", "total_tokens": 500, "cost_micro_usd": 1510,
 "usage": {"api_calls_used": 1, "api_calls_limit": 1000, "tokens_used": 500, "tokens_limit": 100000}}

(cost_micro_usd is 1510 µ$ only under the dev placeholders INPUT_RATE_MICRO_PER_MILLION=1500000 and OUTPUT_RATE_MICRO_PER_MILLION=3500000; in general it depends on configured rates.) Status: 401 invalid_api_key · 402 billing_past_due/billing_unpaid (read-only pre-check and in-transaction re-check) · 409 idempotency_in_progress (fresh in-progress claim, failed stale takeover, or ledger unique (tenant_id, idempotency_key) conflict) · 422 validation_error (including missing Idempotency-Key) / idempotency_key_reuse (different hash in any state, even in_progress or expired) · 429 quota_calls_exceeded/quota_tokens_exceeded with Retry-After. D-1: hard-stop 429 applies to both Free and Pro. Replay: same key + same body returns the stored response with Idempotency-Replayed: true; no new usage. Completed responses are retained IDEMPOTENCY_RETENTION_HOURS (default 24); after cleanup the same key is a new request. DB ops: see request lifecycle in ARCHITECTURE.md §6.

GET /usage/history

Query: limit (1–100, default 20), cursor (keyset on requested_at, id), optional from, to (ISO UTC). A bad cursor or `from > to` returns 422 `validation_error`. Response: {"items": [...], "next_cursor": "..."}. Always filtered by caller's tenant_id.

POST /webhooks/stripe

Reads raw body; verifies signature with STRIPE_WEBHOOK_SECRET; a bad signature returns 400 `invalid_signature`, a missing secret returns 503 `webhook_unavailable`; inserts into webhook_events (unique event_id); dispatches handler; 200 on success or duplicate, 500 on handler failure so Stripe retries. Never returns 402.
