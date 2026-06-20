# References: Stripe idempotency keys

Saved for the 2026-06-20 teardown. Primary sources first.

## Primary / canonical

- Brandur Leach (ex-Stripe), "Implementing Stripe-like Idempotency Keys in
  Postgres" — the canonical public design: atomic phases, foreign state
  mutations, recovery_point state machine, locked_at, reaper/completer.
  https://brandur.org/idempotency-keys
- Article source on GitHub (raw markdown):
  https://github.com/brandur/sorg/blob/master/content/articles/idempotency-keys.md
- "Rocket Rides Atomic" reference implementation (Ruby/Sequel/Postgres),
  shows the started -> ride_created -> charge_created -> finished phases and
  the staged_jobs enqueuer:
  https://github.com/brandur/rocket-rides-atomic

## Official Stripe

- Stripe API reference, idempotent requests (header, 255-char keys, 24h
  storage, 409 on concurrent, param-mismatch error, POST only):
  https://docs.stripe.com/api/idempotent_requests
- Stripe docs, idempotency overview:
  https://stripe.com/docs/idempotency
- stripe-ruby issue on 409 retry behavior during same-key concurrency:
  https://github.com/stripe/stripe-ruby/issues/431

## Key facts pinned (confirmed)

- Header: `Idempotency-Key`. Keys up to 255 chars. Results stored/replayable
  for 24 hours, then pruned (reuse after = new request).
- Concurrent same-key requests: one runs, the other gets `409 Conflict`;
  result is not double-saved.
- Reusing a key with different parameters is rejected.
- Official Stripe SDKs auto-generate a key and auto-retry on connection errors.

## Design facts (Brandur's public pattern, Stripe-adjacent; exact internal
## Stripe schema is not public)

- Table `idempotency_keys`: idempotency_key, request_method/path/params,
  response_code/body, recovery_point, locked_at, last_run_at, user_id.
- Unique index on `(user_id, idempotency_key)` enforces one-key-per-account
  and turns concurrent first-attempts into a collision instead of a double row.
- `recovery_point` is a forward-only DAG; each transition commits in the same
  transaction as the work it describes (crash-safe checkpoint).
- "Atomic phase" = local writes in one Postgres transaction (serializable).
  "Foreign state mutation" = the card-network call that cannot be rolled back.
- After-work (emails) staged in `staged_jobs` inside the txn, drained by an
  enqueuer only after commit. Reaper deletes expired keys (~72h in the demo,
  24h in Stripe's docs). Completer retries abandoned in-flight requests.
