# Stripe: API rate limiting and load shedding (the "429", the token buckets, and the four dials that keep Stripe up on Black Friday)

Date: 2026-09-01
Product: Stripe
Feature: API rate limiting and load shedding (the machinery behind `429 Too Many Requests` and `503 Service Unavailable`)

---

## 1. The user

Meet Priya. She is the backend engineer at a mid-size Indian D2C brand that sells
running shoes online. Her checkout runs on Stripe. Most days her code fires a
calm trickle of API calls: a shopper taps Pay, her server calls
`POST /v1/charges` (in current Stripe terms `POST /v1/payment_intents`), Stripe
replies in 200 milliseconds, done. Maybe 5 charges a second at lunch.

Then Black Friday arrives. Her brand runs a "70% off, first 2 hours only" drop.
At 12:00:00 the homepage gets hammered. Her own servers autoscale fine. But now
her code is calling Stripe 800 times a second, and she has a retry loop that,
on any error, immediately tries again. Somewhere in her stack a bug starts
double-firing. Suddenly she is sending Stripe 3,000 requests a second from one
account.

Priya is not thinking about Stripe's infrastructure. She is watching her orders
dashboard and praying it does not stall. The other people in this story are the
40,000 shoppers on other Stripe merchants (a bakery in Pune, a SaaS tool in
Berlin, a charity taking donations) who have nothing to do with Priya's drop but
share the same Stripe fleet. If Priya's runaway loop is allowed to eat the whole
API, all of them fail at checkout too.

The feature we are tearing down is the thing that stops that: the layer that
decides, thousands of times a second, which requests Stripe answers and which it
politely refuses, so that one merchant's bad afternoon never becomes everyone's
outage.

## 2. The real problem

Here is the pain, told like a friend would tell it.

A shared API is a shared kitchen. Stripe cooks for millions of businesses out of
one set of servers. On a normal Tuesday there is plenty of stove space. But
capacity is finite and demand is spiky, and there are three separate ways the
kitchen catches fire:

1. One customer goes berserk. Priya's retry loop, or an infinite loop in someone
   else's cron job, starts sending a flood from a single account. Left alone it
   starves everyone else.
2. A few slow orders clog the line. Some endpoints are cheap (read a charge).
   Some are heavy (create a charge, which touches a bank). If one account opens
   500 heavy requests at once and holds them, it pins CPU and memory even at a
   modest request-per-second number.
3. The whole building is on fire. A database gets slow, a dependency times out,
   or real global traffic just exceeds what the fleet can serve. Now no single
   customer is at fault. The system as a whole is underwater.

The naive fix ("just cap everyone at N requests per second") solves only the
first problem and makes the other two worse. A flat cap does nothing when the
danger is 500 slow held-open requests at a legal request rate, and it does
nothing when the servers themselves are dying for reasons unrelated to any one
caller. Stripe's insight, published in 2017, is that these are three different
problems that need three different tools, plus a fourth as the floor. One
algorithm is never enough.

## 3. The feature in one sentence

Stripe runs four independent limiters in series in front of the API, two that
protect the system from any single abusive caller and two that protect the whole
fleet from itself, so that overload degrades gracefully into honest `429` and
`503` refusals instead of a total outage.

## 4. Jobs to be done

What is the user really hiring this feature to do?

- Priya is hiring it to keep her own good traffic flowing even when her bad
  traffic (that buggy retry loop) is out of control. She wants a clear signal
  ("you are going too fast, back off") instead of silent timeouts.
- The bakery in Pune is hiring it to guarantee that a stranger's traffic spike
  never takes down its checkout. It is buying isolation.
- Stripe is hiring it to protect availability, which is the entire product.
  A payments API that is down loses money by the second, for everyone. Stripe's
  own framing: keep the core of the business working while the rest is on fire.
- Every shopper, without knowing it, is hiring it to make sure that "Pay" still
  works during the exact minutes when the most people in the world are trying to
  pay at once.

## 5. How it works for the user

From Priya's seat it is almost invisible until she crosses a line.

Normal path: she calls the API, gets `200 OK`, moves on. She never sees the
limiter.

Too fast: when her account exceeds its allowed request rate, Stripe answers
`429 Too Many Requests`. The body says `rate_limit_error` with a message like
"Too many requests hit the API too quickly." Stripe's docs tell her the right
move is to back off and retry with exponential spacing, not to hammer harder.
Her well-written client library (stripe-node, stripe-go) already does this: it
sees the `429`, waits, retries with jitter.

Whole-fleet trouble: in a genuine overload she may instead see
`503 Service Unavailable` on her non-critical calls (a dashboard `GET`, a
test-mode request) while her real money-moving calls (`POST /v1/payment_intents`)
keep going through. She might not even notice, because the traffic Stripe sheds
first is the traffic that matters least.

The key user-visible promise: the failure is fast, labeled, and selective.
Priya gets a specific error code she can code against, not a hung connection.
And the calls Stripe chooses to keep alive are the ones that move money.

## 6. The actual flow, step by step

Walk one real request. A shopper on Priya's site taps Pay at 12:00:03 on Black
Friday. Her server sends `POST /v1/payment_intents` with the amount 4,999 rupees.

1. The request lands on Stripe's edge. Before any business logic runs, it passes
   through the limiter gauntlet, in order.
2. Request rate limiter. Stripe checks: has Priya's account exceeded its
   per-second request budget? This is a token bucket (details below). If her
   bucket is empty, she gets `429` right here and the request never touches the
   payments code. If she has tokens, one is spent and she moves on.
3. Concurrent requests limiter. Stripe checks: how many of Priya's requests are
   in flight right now, this instant? If she already has, say, 100 open requests
   being processed, this one is refused with `429`, because a pile of slow
   concurrent requests is its own kind of overload even at a legal rate.
4. Fleet usage load shedder. Stripe checks the whole fleet's health, not Priya.
   Is the fleet's reserved headroom for critical calls being eaten? Creating a
   payment is critical, so this request is waved through even if the fleet is
   stressed. A `GET /v1/charges?limit=100` at the same moment might get `503`
   here so that Priya's `POST` has room.
5. Worker utilization load shedder. The last gate. The individual worker box
   about to run this request looks at its own CPU. If boxes across the fleet are
   saturated, it starts probabilistically shedding the least important work.
   Creating a payment survives; a test-mode call does not.
6. Only after clearing all four does the request reach the actual payment logic,
   talk to the bank, and return `200` with the created PaymentIntent.

Four checks, all in memory, all before the expensive work. A refusal at gate 1
costs almost nothing, which is the whole point: saying no has to be far cheaper
than saying yes, or the act of refusing floods you too.

## 7. Under the hood, like the engineer

This is the heart. Confirmed facts are from Stripe's 2017 engineering post
"Scaling your API with rate limiters" (Paul Tarjan) and from Brandur Leach's
companion write-up on the GCRA algorithm behind Stripe's open-source `throttled`
library. Where I extend past what Stripe published, I label it inference.

### The core data structure: a counter in Redis, mutated atomically

Every one of these limiters is, underneath, a small number kept in Redis and
changed by a Lua script. Redis is chosen for one reason: it is in-memory, so a
read-modify-write is microseconds, and it runs Lua scripts atomically, so two
concurrent requests can never both read "1 token left" and both spend it. That
atomicity is the entire trick. Confirmed, quoting the post: the request rate
limiter "uses a basic token bucket algorithm and relies on the fact that Redis
scripts execute atomically."

Why Redis and not the main Postgres database? Because the limiter runs on the
hottest path in the company, in front of every single API call. It cannot add a
database round trip. It also cannot be a hard dependency: if Redis is down, you
do not want all of Stripe to go down with it. So the design fails open. Stripe
observed Redis unavailability roughly 0.01% of the time and simply allows the
traffic through in that tiny window rather than blocking money movement on a
cache.

### Limiter 1: the request rate limiter (token bucket)

The mental model is a bucket that holds tokens. Each request takes one token.
The bucket refills at a steady rate. If the bucket is empty, you are refused.

Stripe's example numbers from the post: refill 100 tokens per second, bucket
capacity 500. So a caller can sustain 100 requests a second forever, and can
also burst up to 500 requests instantly if the bucket was full (they had been
quiet and let it fill). This burst allowance is deliberate. Real clients are
bursty. A merchant that sends nothing for 4 seconds and then 400 requests in one
second is normal, not abusive, so the bucket lets that through while still
capping the long-run average at 100 per second.

Concrete walk: Priya's account has capacity 500, refill 100 per second. At
12:00:00 her bucket is full at 500. Her buggy loop dumps 500 requests in the
first 200 milliseconds. All 500 pass (burst absorbed). Now the bucket is empty.
Over the next second only 100 more get through (the refill). Request 601 in that
window gets `429`. Her retry loop backs off, the flood is capped at the true
sustainable rate, and the bakery in Pune never felt a thing.

The TTL trick, confirmed: the Redis key for a bucket is given a time-to-live of
roughly `fill_time * 2`. If a caller goes quiet, the key expires and is cleaned
up, so idle accounts do not cost memory, and a returning caller starts fresh
rather than with some stale phantom debt. This is the same "let it expire, do
not sweep it" discipline seen across this ledger (Stories expiry, seat locks).

The upgrade, confirmed: Stripe's open-source `throttled` (Go) originally used a
naive counter and was rewritten to use GCRA, the Generic Cell Rate Algorithm.
GCRA is a token bucket turned inside out. Instead of storing a token count and a
last-refill time and doing arithmetic, it stores exactly one value per key: a
single timestamp called the "theoretical arrival time" (TAT), the earliest
moment the next request is allowed. To check a request you compare now against
that stored timestamp and, if allowed, push the timestamp forward by one
request's worth of time. One number, one comparison, one write. No background
drip process refilling buckets, no sorted structure. This is why GCRA scales: the
per-key state is a single timestamp and the per-request work is O(1), so a
limiter serving millions of keys is just millions of tiny independent timestamps
in a hash map, which is exactly what Redis is.

### Limiter 2: the concurrent requests limiter (a sorted set)

Rate is not the only danger. Imagine an account that sends only 20 requests a
second, well under any rate cap, but each request is a heavy report that takes 10
seconds to compute. After a few seconds that account has 200 requests running at
once, each holding a worker, a chunk of memory, an open connection. It is
starving the fleet at a request rate that no rate limiter would flag.

So Stripe counts in-flight requests, not arriving requests. Confirmed data
structure: a Redis sorted set per account. When a request starts, its ID is
added to the set scored by the current timestamp. When it finishes, its ID is
removed. To decide whether a new request is allowed, Stripe reads the size of the
set (in Redis, `ZCARD`) and compares against the concurrency cap (the post uses
100). Stale entries older than a TTL (the post uses 60 seconds) are trimmed off
by score so a crashed client that never sent its "I finished" signal does not
leak a slot forever.

Concrete walk: Priya kicks off 100 slow list-and-export calls at once to build a
Black Friday reconciliation file. The 101st is refused with `429` even though her
per-second rate is tiny, because 100 of her requests are already occupying
workers. Her export still completes, just serialized instead of all-at-once, and
the fleet keeps its capacity for everyone's checkouts.

Why a sorted set and not a plain integer counter? Because a counter cannot
self-heal. If you just did "increment on start, decrement on end" and a client
died mid-request, the counter would drift upward forever and slowly lock the
account out. The sorted set stores each live request as an entry scored by time,
so you can atomically drop everything older than 60 seconds in one operation and
the count corrects itself. The structure encodes both "how many" and "since
when," which a scalar cannot.

### Limiters 1 and 2 are about the user. Limiters 3 and 4 are about the system.

This is the conceptual split Stripe draws, and it matters. A rate limiter asks
"is this caller behaving?" A load shedder ignores the caller entirely and asks
"is the whole system healthy right now?" Quoting the idea from the post: load
shedders make decisions based on the state of the whole system rather than the
user making the request, and they exist to keep the core of the business working
while the rest is on fire. Different question, different tool.

### Limiter 3: the fleet usage load shedder (criticality)

Every Stripe API method is tagged as critical or not. Creating a charge, moving
money: critical. Reading an old charge, listing objects, anything in test mode:
non-critical. The fleet usage load shedder reserves a slice of total fleet
capacity (think of it as "always keep at least X% of workers free for critical
work"). When incoming load threatens to eat that reserve, Stripe starts refusing
non-critical requests with `503` so the reserved headroom stays available for the
critical ones.

Concrete walk: at 12:00 on Black Friday, real global charge volume alone is near
the fleet's ceiling. A flood of dashboard `GET`s and analytics reads pile on top.
The fleet usage shedder starts returning `503` to those reads and to test-mode
traffic. Meanwhile `POST /v1/payment_intents` from Priya, from the bakery, from
everyone, sails through. The dashboards look laggy for a few minutes. Not one
shopper is blocked from paying. That is the trade the shedder exists to make:
sacrifice the least important work to protect the most important work.

Note it returns `503`, not `429`. That difference is a real signal to the client.
`429` means "you, specifically, are going too fast, slow down." `503` means
"the system is overloaded, this is not about you, try again shortly." A good
client treats them differently.

### Limiter 4: the worker utilization load shedder (the floor)

The last line. Each worker box watches its own utilization: how many of its
workers are busy. Under normal conditions it sheds nothing. As boxes across the
fleet approach saturation, each one independently begins rejecting traffic,
starting with the least critical, and it ramps the rejection probability up
gradually rather than slamming to a hard stop. Confirmed shape from the post's
example: it holds off until utilization has been high for a window (the example
uses roughly 28 seconds before shedding begins) and ramps to full shedding over
about 120 seconds. Gradual, so a brief spike does not trigger a cliff, but a
sustained overload gets firm backpressure.

This one does not care about accounts or criticality tags in the fine-grained
way limiter 3 does. It is the dumb, reliable floor: if a box is drowning, it
throws the least valuable work overboard so it does not sink entirely and take
its in-flight critical requests down with it.

### Why all four, in this order

Each catches a failure the others miss. Rate limiter: one caller sending too
fast. Concurrent limiter: one caller holding too many slow requests at a legal
rate. Fleet shedder: global load with no single guilty caller, protect money
over reads. Worker shedder: infrastructure itself failing, protect the box from
death. They run cheapest-first and account-specific-first, so the common case
(one noisy merchant) is stopped at gate 1 or 2 for almost free, and the rare case
(the whole fleet underwater) is caught by gates 3 and 4. Remove any one and a
whole class of outage comes back.

### The scale story at three tiers

The thing that grows here is not a catalog of items. It is requests per second
and the number of distinct accounts (keys) being tracked. Watch what breaks.

Tier 1, about 1,000 requests per second, one Redis node. Everything is easy.
A single Redis instance holds every account's token bucket and sorted set. Each
check is one atomic Lua script, sub-millisecond. At this size you could almost
skip the load shedders. Do not over-engineer. The naive design is correct and
fast. One Redis node handles this without sweating.

Tier 2, about 100,000 requests per second, hundreds of thousands of active keys.
Now two things strain. First, a single Redis node's CPU and network become the
bottleneck, because every one of those 100,000 checks is a round trip to it.
Second, memory: hundreds of thousands of live buckets and sorted sets. The fix is
that the limiter state is embarrassingly shardable. Each account's bucket is
keyed by account ID and depends on nothing else, so you split the keyspace across
a Redis cluster by hashing the account ID. Priya's bucket lives on node 7, the
bakery's on node 12, and the two never contend. This is the same "the key routes
itself, so sharding is free" property that made Stripe's idempotency keys and
Notion's blocks scale. The TTL cleanup (`fill_time * 2`) keeps memory bounded by
active callers, not total callers ever seen.

Tier 3, 10 million plus requests per second on a global sale day, and the hot-key
problem. Two hard things appear. First, the shared read path can develop a hot
key: if one giant merchant is a single account ID, all its limiter traffic lands
on one Redis shard, and that shard becomes the ceiling. This is the same hot-key
wall this ledger hit with Amazon Lightning Deals' single stock counter. The
inference-level fix, standard for this class: split one account's limiter into N
sub-buckets (`acct:123:bucket:0..9`) across shards and spread its requests over
them, trading exact accounting for a tenfold drop in per-shard contention.
Second, and this is the real reason the load shedders exist, at this tier the
danger is no longer any one caller. It is aggregate global load and correlated
failure (a slow dependency during the exact minute of peak traffic). Rate limits
per account do nothing against that, because everyone is individually behaving.
The fleet usage shedder and worker utilization shedder are what survive tier 3:
they shed the least important slice of a fundamentally oversubscribed fleet so
the money-moving core stays up. The lesson Stripe draws and this ledger keeps
seeing: at the top tier you stop trying to serve everything and start choosing,
cheaply and automatically, what to drop.

One more scale note, confirmed: the whole limiter path must itself be cheaper
than the work it guards, or it becomes the bottleneck it was meant to prevent.
That is why saying no is an in-memory timestamp comparison, why the state is one
number per key, and why GCRA (one stored timestamp, one comparison) replaced the
chattier original. The guard is designed to cost almost nothing so that refusing
a million requests a second is affordable.

## 8. The retention and habit mechanic

Rate limiting is not a feature users open and enjoy. Its habit loop is
counterintuitive: the mechanic is trust through the absence of disaster, and the
metric it moves is retention, specifically the retention of every merchant who
would have churned after an outage.

Here is the real observed loop. A payments company lives or dies on its status
page. Every minute of downtime during a sale is money lost by thousands of
businesses at once, and it is the kind of pain that makes a CTO start evaluating
competitors the next morning. Stripe's reputation for staying up during Black
Friday and Cyber Monday, the two highest-traffic commerce days on Earth, is a
direct product of this layer working. Merchants do not consciously think "thank
you rate limiter." They think "Stripe just always works, even on the busy days,"
and that reflex is why they do not migrate. The feature retains customers by
being invisible on the one day it matters most.

There is a second, quieter loop for developers like Priya. Because the failure is
an honest, documented `429` with a clear "back off and retry" instruction, and
because Stripe's own client libraries handle it automatically, Priya's bad day
resolves itself without a support ticket. A limiter that failed with silent
timeouts would generate frustration, blame, and churn. A limiter that fails with
a clean, machine-readable, self-correcting signal builds the sense that the
platform has her back. That sense is switching-cost retention in its purest form,
the same invisible-craft loyalty this ledger found in Figma's vector networks:
the tool earns trust by handling your worst moment gracefully.

The metric, plainly: retention and, underneath it, availability, which is the
number the entire business is measured on. Not activation, not a dopamine loop.
The reward is that nothing bad happened.

## 9. The lesson for Rare.lab

Rare.lab compiles shader graphs and runs an embeddable runtime, and it will have
exactly Stripe's problem the day one embedded title goes viral: a shared backend
(compile workers, cloud-render, asset CDN, telemetry ingest) hit by wildly spiky,
untrusted load from thousands of embeds you do not control. Copy the four-dial
structure, do not invent one clever dial.

1. Put a token bucket (better, GCRA) in front of every expensive backend
   operation, keyed by API key or embed ID, held as one timestamp per key in
   Redis and checked with an atomic Lua script. Give it burst capacity (a scene
   that compiles 40 shader variants at once on load is bursty and legitimate,
   like Priya's 400-in-one-second) while capping the sustained rate. One number
   per key, O(1) per check, so refusing a flood costs almost nothing. Make it
   fail open: if the limiter's Redis is down, let compiles through rather than
   taking the whole product down with the cache.

2. Add a concurrent-work limiter as a separate dial, because your heavy path
   (compiling a 500-node graph, rendering a 4K frame) is slow and a caller can
   pin your workers at a perfectly legal request rate. Count in-flight jobs per
   key with a Redis sorted set scored by start time, trim entries older than a
   TTL so a crashed embed does not leak a worker slot forever, and refuse the
   next job with a clear "too many in flight" once the cap is hit. This is the
   dial most teams forget and the one that actually saves the render farm.

3. Tag operations critical vs non-critical and build a fleet load shedder that
   protects the critical slice. For Rare.lab the critical path is serving already
   compiled, shipped runtime assets to live end-users of a published title. The
   sheddable path is speculative editor previews, thumbnail regeneration, and
   analytics. When the fleet is saturated, return the equivalent of `503` to the
   previews and keep the live runtime serving, exactly as Stripe keeps
   `payment_intents` alive while shedding dashboard reads. A creator's editor
   getting momentarily slow is survivable. A million players of a shipped title
   seeing broken visuals is not.

4. Keep a worker-utilization floor that ramps gradually. Each render/compile box
   watches its own CPU and GPU and, past a sustained threshold, sheds the least
   critical work first with a rising probability rather than a hard cliff, so a
   two-second spike does not trigger a stampede but a real overload gets firm
   backpressure.

The one-line lesson: one flat rate limit is a trap, because overload has at least
three unrelated shapes (one abusive caller, one caller holding too many slow
jobs, and the whole fleet underwater with nobody at fault), so guard the backend
with a cheap per-key token bucket and a per-key concurrency limiter for the
caller-shaped problems and two whole-system load shedders that protect your
critical path for the fleet-shaped ones, make every check an O(1) in-memory
timestamp so saying no is nearly free, and always fail open so the guard can
never become the outage.

---

## Sources

- Stripe Engineering, "Scaling your API with rate limiters" (Paul Tarjan, 2017): https://stripe.com/blog/rate-limiters
- Stripe rate limiters, annotated gist / mirror of the post with code: https://gist.github.com/ptarjan/e38f45f2dfe601419ca3af937fff574d
- Brandur Leach, "Rate limiting, cells, and GCRA" (the GCRA algorithm behind Stripe's `throttled`): https://brandur.org/rate-limiting
- Stripe open-source `throttled` (Go, GCRA implementation): https://github.com/throttled/throttled
- Stripe Documentation, "Rate limits" (current live-mode limits and error shapes): https://docs.stripe.com/rate-limits
- System Design newsletter, "This is how Stripe does rate limiting to build scalable APIs" (secondary walkthrough): https://newsletter.systemdesign.one/p/rate-limiter
