# References: Stripe API rate limiting and load shedding

Saved 2026-09-01 for the Stripe rate limiting teardown.

## Primary source (Stripe engineering)

- **Scaling your API with rate limiters** (Paul Tarjan, Stripe Engineering, 2017).
  The canonical write-up. Describes the four production limiters running in series:
  1. Request rate limiter (token bucket in Redis, example 100 tokens/sec refill,
     500 capacity, atomic Lua script, returns 429, key TTL = fill_time * 2).
  2. Concurrent requests limiter (Redis sorted set per account, add on start /
     remove on finish, ZCARD vs cap of 100, trim entries older than 60s, returns 429).
  3. Fleet usage load shedder (system-state, tags critical vs non-critical methods,
     reserves fleet capacity for critical money-movement, sheds non-critical with 503).
  4. Worker utilization load shedder (last resort, per-box CPU, ramps rejection
     gradually, example ~28s before shedding, ~120s to full).
  Also: Redis executes Lua atomically; the limiter fails open (Redis observed
  unavailable ~0.01% of the time, traffic allowed through rather than blocked);
  the distinction between rate limiters (protect against one caller) and load
  shedders (protect the whole fleet from itself).
  https://stripe.com/blog/rate-limiters
  Annotated gist / mirror with code: https://gist.github.com/ptarjan/e38f45f2dfe601419ca3af937fff574d

## The GCRA upgrade

- **Rate limiting, cells, and GCRA** (Brandur Leach, Stripe engineer). Explains the
  Generic Cell Rate Algorithm behind Stripe's open-source `throttled` library, which
  was rewritten from a naive counter to GCRA. GCRA stores a single timestamp per key
  (the theoretical arrival time / TAT), needs no background drip process, and is O(1)
  per request, which is why it scales to millions of keys.
  https://brandur.org/rate-limiting

- **`throttled` (Go)**, the open-source GCRA rate limiter maintained with Stripe
  origins, already taking production traffic at Stripe.
  https://github.com/throttled/throttled

## Current product limits (for the user-facing numbers)

- **Stripe Documentation, Rate limits.** Current defaults: live mode allows on the
  order of 100 read and 100 write operations per second, sandbox lower; some
  individual resources are stricter (around 25/sec). Error shape: `rate_limit_error`,
  HTTP 429, guidance to back off and retry with exponential spacing; client libraries
  auto-retry on 429.
  https://docs.stripe.com/rate-limits

## Secondary walkthroughs (useful cross-checks, not primary)

- System Design newsletter, "This is how Stripe does rate limiting to build scalable
  APIs": https://newsletter.systemdesign.one/p/rate-limiter

## Notes on fact vs inference in the teardown

- Confirmed from Stripe/Brandur: the four limiter types and their roles, token bucket
  + Redis + atomic Lua, the 100/500 and 100-concurrent/60s example numbers, the
  fleet-vs-worker shedder split, 429 vs 503, fail-open at ~0.01% Redis downtime,
  TTL = fill_time * 2, and the GCRA rewrite of `throttled`.
- Clearly labeled inference in the report: sharding the limiter keyspace by account id
  and the sub-bucket split for a single hot account are the standard solution for this
  class (mirrored from Amazon Lightning Deals' stock-counter sharding in this ledger),
  not published Stripe internals.
