Progress Tracker

Status values: Not Started · In Progress · Completed · Blocked. A task is marked complete only after the owner verifies it.

Phase Name Status
0 Understanding and planning In Progress (docs drafted, awaiting owner review)
1 Development environment Not Started
2 FastAPI fundamentals Not Started
3 Database integration Not Started
4 Tenant management Not Started
5 API key authentication Not Started
6 Subscription management Not Started
7 Usage metering Not Started
8 Quota enforcement Not Started
9 Idempotency Not Started
10 Pricing and billing Not Started
11 Webhook integration Not Started
12 Usage reporting Not Started
13 Automated testing Not Started
14 Dockerization Not Started
15 Documentation Not Started

Completed tasks

- (none verified yet)

Remaining tasks

- Owner review of PRD and open questions
- Phases 1–15

Bugs

- (none)

Planned test cases (Planned, not written; no results claimed)

- `TC-D6-01 canceled_subscription_falls_back_to_free_limits_no_counter_reset`: when `customer.subscription.deleted` arrives (at paid-period end if `cancel_at_period_end`), a tenant whose current-month usage already exceeds Free limits receives 429 `quota_calls_exceeded` on `/v1/usage/simulate` until the next UTC month; no counter reset and no 402.
- `TC-D8-01 rejected_requests_log_fields_and_no_plaintext_key`: 402 and 429 emit structured log lines containing `request_id`, `tenant_id`, endpoint, error code and UTC timestamp; a failed 401 logs `request_id` and reason only; no log line contains a plaintext API key or its prefix.
- `TC-A15-01 tied_event_created_uses_state_ranking_or_refetch`: two webhook events with identical Stripe `created` arrive out of order; the handler applies the event whose state is further along (`checkout_pending` < `active` < `past_due` < `unpaid` < `canceled`) or re-fetches from Stripe; final subscription state is never less advanced.
- `TC-CFG-01 config_errors_never_print_values`: missing/invalid variables raise errors naming only the variable; no error message or log line contains a configured value (including `DATABASE_URL` and Stripe secrets).

Pricing known-answer cases (Planned, not written; explicit rates passed directly into `app.services.pricing`, never read from `.env`):

- `TC-PRICE-01 input_half_up`: 1 input token at 1,500,000 µ$/1M → 2 µ$ (half-up).
- `TC-PRICE-02 output_half_up`: 1 output token at 3,500,000 µ$/1M → 4 µ$.
- `TC-PRICE-03 dev_rate_sample`: 120 input + 380 output at 1,500,000 / 3,500,000 µ$/1M → 1,510 µ$.
- `TC-PRICE-04 zero_tokens`: 0 input + 0 output → 0 µ$.
- `TC-PRICE-05 rounds_down`: 1 input token at 1,200,000 µ$/1M → 1 µ$ (remainder 200,000 < 500,000).
- `TC-PRICE-06 max_tokens_integer_only`: 1,000,000 input + 1,000,000 output at dev rates → 5,000,000 µ$; result is an integer with no float use and no overflow.

Technical decisions

See DECISIONS.md.

Known limitations

- (to be filled as the build progresses)

Pending questions

See open questions in DECISIONS.md.
