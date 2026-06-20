# Stripe Idempotency Keys: charging a card exactly once on a flaky network

Date: 2026-06-20
Product: Stripe
Feature: Idempotency keys (the `Idempotency-Key` header that makes a retried payment safe)

---

## 1. The user

Meet Aarav. He runs a small online store that sells running shoes, built on
top of Stripe. It is 9pm on a sale night. A customer named Neha is on a train
between Pune and Mumbai, buying a pair of shoes for 4,499 rupees. Her phone has
two bars of patchy signal.

Aarav is not the person who literally types the idempotency key. His backend
server does, every time it asks Stripe to charge a card. But Aarav is the one
who gets the angry email at 11pm if Neha is charged twice. So the real user
here is the developer who is terrified of double charging a customer, and the
end customer who must never see two debits for one pair of shoes.

The moment this feature matters: the exact instant Neha taps "Pay" and the
network hiccups.

## 2. The real problem

Here is the thing that keeps payment engineers up at night. The internet
forgets to tell you whether your request worked.

Neha's phone sends "charge this card 4,499 rupees" to Aarav's server. Aarav's
server forwards it to Stripe. Stripe charges the card successfully. Then, on
the way back, the train enters a tunnel and the response is lost. Aarav's
server waited 30 seconds, heard nothing, and now has to make a horrible guess:

- Did the charge go through and only the reply got lost? If so, retrying will
  charge Neha twice.
- Did the charge never happen at all? If so, NOT retrying means Aarav loses the
  sale and ships nothing.

A failed network call is not the same as a failed payment. The server cannot
tell the two apart from where it sits. This is the classic "two generals"
problem of distributed systems, and with money on the line it is not academic.
Without a fix, every dropped connection forces a coin flip between "charge
twice" and "charge zero times." Both are bad. One of them is a refund and a
furious customer.

## 3. The feature in one sentence

An idempotency key is a unique ID the client attaches to a request so that no
matter how many times that exact request is retried, Stripe performs the work
once and replays the same saved answer every time after.

## 4. Jobs to be done

What is Aarav's server really hiring this feature to do?

- "Let me retry a payment without fear." Retrying should be the safe default,
  not a gamble.
- "Give me exactly-once, not at-least-once or at-most-once." One tap, one
  charge, always.
- "Make my own code simpler." Aarav should not have to build a charge-dedup
  system himself. He attaches one header.
- "Survive my own crashes." If Aarav's server dies mid-request and reboots, it
  should be able to ask the same question again and get a consistent answer.
- "Tell me clearly when I am doing something dumb." If he reuses a key but
  changes the amount, he wants an error, not silent corruption.

## 5. How it works for the user

From the developer's side it is almost insultingly simple. You generate a
random unique string, usually a UUID like `a1b2c3d4-...`, and put it in one
HTTP header:

```
POST /v1/charges
Idempotency-Key: 5mUP59iveriphkIJ
amount=4499
currency=inr
source=tok_visa
```

That is the whole API surface. Now:

- First time Stripe sees `5mUP59iveriphkIJ`, it charges the card and saves the
  result.
- Every later request carrying the same key gets the saved result back, byte
  for byte, without charging again. The card is touched once.
- If two requests with that key arrive at the same instant, one runs and the
  other gets a `409 Conflict` telling it to back off.
- If Aarav reuses the key but changes `amount` to 9999, Stripe rejects it,
  because the parameters do not match the original. (Source: Stripe API docs.)

Documented behavior worth pinning down (Stripe API reference):

- Keys can be up to 255 characters long.
- Results are stored and replayable for 24 hours, then pruned. Reuse a key
  after that and Stripe treats it as a brand new request.
- All `POST` requests accept idempotency keys. `GET` and `DELETE` are naturally
  idempotent already, so they do not need one.
- The official Stripe client libraries (Ruby, Python, Node, and so on)
  automatically generate an idempotency key and retry for you on connection
  errors. So a lot of developers get this protection without writing a line.

## 6. The actual flow, step by step

Walk Neha's 4,499 rupee shoe purchase end to end, including the tunnel.

1. Neha taps "Pay." Her phone POSTs the order to Aarav's server.
2. Aarav's server generates a fresh idempotency key, say
   `order_88213_attempt`. It is derived from the order, so a retry of THIS
   order reuses the same key.
3. Server calls Stripe: `POST /v1/charges` with
   `Idempotency-Key: order_88213_attempt` and amount 4499.
4. Stripe receives it, records the key as "in progress," charges Neha's HDFC
   Visa, and gets an approval from the card network.
5. Stripe saves the successful response under that key and starts sending it
   back.
6. The train enters a tunnel. The response never reaches Aarav's server. After
   30 seconds the server times out. It does not know step 4 happened.
7. Aarav's server retries. Same call, same key `order_88213_attempt`.
8. Stripe looks up the key, sees it is already "finished," and replays the
   exact saved response. It does NOT call the card network again.
9. Aarav's server finally gets a clean "charge succeeded," marks the order
   paid, and ships one pair of shoes.

Neha is charged once. The retry was free. The tunnel did not cost anyone money.

## 7. Under the hood, like the engineer

This is the heart of it. Stripe has never published its exact internal schema,
so I am leaning on the canonical public design written by Brandur Leach, who
was a Stripe engineer, in "Implementing Stripe-like Idempotency Keys in
Postgres" and his "Rocket Rides" reference code. I will mark what is confirmed
Stripe behavior versus what is the well-grounded public pattern.

### The naive idea, and why it is not enough

The obvious first attempt: keep a table of seen keys. When a request arrives,
check if the key exists. If yes, return the saved answer. If no, do the work
and save it.

This works right up until the work has TWO parts that cannot both live in one
database transaction. A card charge is exactly that. Charging the card is a
call to Visa or Mastercard out on the public network. You cannot roll back
Visa. So the request is really:

```
write to my database  ->  call the card network  ->  write to my database again
```

The middle step is what Brandur calls a "foreign state mutation": a change to a
system outside your own database that you cannot undo with a `ROLLBACK`. The
two database writes around it are "atomic phases": groups of local writes that
either all happen or none happen, guarded by a Postgres transaction.

If your server crashes between "call the card network" and "write the result,"
a dumb seen-keys table is stuck. It does not know the charge happened. This is
why the real design is a small state machine, not a lookup table.

### The data structures in play

1. A hash map style lookup, by key. The first thing every request does is find
   its key. Logically this is a dictionary: key string maps to a row. In
   Postgres it is a B-tree unique index on `(user_id, idempotency_key)`, which
   gives O(log n) lookups and, just as important, enforces "this key exists at
   most once per account." That uniqueness constraint is the quiet hero: it is
   what makes two simultaneous first-attempts collide instead of both
   inserting.

2. A row per key holding the whole story. The public design's
   `idempotency_keys` table carries, among others:
   - `idempotency_key` (the client string)
   - `request_method`, `request_path`, `request_params` (so a mismatch can be
     detected and rejected)
   - `response_code`, `response_body` (the saved answer to replay)
   - `recovery_point` (where in the state machine we are)
   - `locked_at` (a timestamp meaning "someone is working on this right now")
   - `last_run_at`, `user_id`

3. A state machine, shaped as a tiny directed acyclic graph. `recovery_point`
   only ever moves forward through named checkpoints. In the Rocket Rides ride
   example the points are exactly:
   `"started"` -> `"ride_created"` -> `"charge_created"` -> `"finished"`.
   Each arrow is one atomic phase. A retry reads `recovery_point` and jumps
   straight to the next unfinished phase. If it already says `"finished"`, the
   server skips everything and replays `response_body`. This is the part that
   survives a crash in the middle: the checkpoint was committed to Postgres in
   the same transaction as the work it describes, so it can never claim a phase
   finished that did not.

4. A lock, expressed as a column not a separate lock service. `locked_at` is
   set when a request starts working. A second request for the same key that
   finds a fresh `locked_at` returns `409 Conflict` and tells the caller to
   retry later. This is confirmed Stripe behavior: concurrent requests with the
   same key get a conflict, and the result is not saved twice.

5. A queue for the after-work. Things like "send the receipt email" must not
   fire until the database transaction actually commits, otherwise a rolled
   back charge could still email a receipt. The public design stages these in a
   `staged_jobs` table inside the same transaction, and a separate enqueuer
   drains them to the real job queue only after commit. A queue, used for
   correctness, not just throughput.

### Walking one real charge through the machine

Take Neha's 4,499 rupee charge. Confirmed-pattern flow:

- Phase 1, reach `"started"`: one transaction either inserts a new key row or,
  if the key already exists, reads it and checks the params match. If params
  differ (someone changed the amount), reject now. Set `locked_at`.
- Foreign call: ask the card network to charge 4,499 rupees. This is outside
  any transaction. It can succeed even if the next write fails.
- Phase 2, reach `"charge_created"`: one transaction saves the card network's
  charge ID and advances `recovery_point`. Both commit together.
- Phase 3, reach `"finished"`: one transaction writes `response_code` and
  `response_body`, stages the receipt email job, clears `locked_at`. Commit.

Now the tunnel hits and the client retries. The retry finds
`recovery_point = "finished"` and simply returns the stored 200 response. Visa
is never called again. Notice the charge to the card carries ITS OWN
idempotency key too (Rocket Rides uses one like
`rocket-rides-atomic-{id}`), so even the foreign call is safe to retry. The
safety is recursive: every hop down the chain repeats the same trick.

### Where the sorting and "matching" really happen

Idempotency is not a search-and-rank feature, but it has the same split the
ranking features do: matching then acting. The "match" is the indexed lookup of
the key, done by Postgres server side using the B-tree, never by scanning. The
"act" is the state machine deciding what to do. Both are cheap and both run on
the server, never on Neha's phone. The phone's only job is to send the same key
again.

### The scale story, three tiers

Tier 1, about 1,000 keys (a hobby store, a few orders an hour). A single
Postgres table on one box. The unique index lookup is microseconds. Nothing is
hard here. You could almost get away with the naive seen-keys table, but you
still want the state machine for the crash case.

Tier 2, about 100,000 keys a day (a growing business). Now two things start to
bite. First, the table grows forever if you never clean up, and old keys are
useless after a day. So a "reaper" job deletes keys older than the retention
window. This is exactly why Stripe documents a 24 hour life: bounded storage,
predictable lookups. Second, you get real concurrency. Two app servers might
both get a retry for the same key in the same 50 milliseconds. The unique
constraint plus `locked_at` plus serializable transactions are what stop them
from both charging. Without the lock, the index alone would let a crash-and-
retry race slip a second charge through the gap between the foreign call and
the write.

Tier 3, 10 million plus keys and many regions. Now you cannot keep all keys in
one Postgres box, and you do not need to. The key is scoped per account, so you
shard by account: account A's keys live on shard 1, account B's on shard 5. A
request always carries its account, so it always knows which shard to ask. No
cross-shard query is ever needed for a lookup, because a key only has to be
unique within one account. Read replicas absorb the "is this key finished yet"
reads. The reaper runs per shard. The thing that breaks at this tier if you are
careless is a single hot account hammering one shard on a sale day; the fix is
the same family of tricks the inventory teardown used, splitting load and
keeping the hot path to a single indexed write. The deeper point: because each
key is independent and self-routing, idempotency scales almost linearly with
shards. There is no global coordinator to become the bottleneck.

What breaks at each jump, summarized: at 1k nothing; at 100k it is unbounded
growth (fixed by the reaper) and concurrency races (fixed by the lock plus
serializable phases); at 10M it is single-box limits (fixed by sharding per
account and read replicas).

### A confirmed real-world detail

Stripe's own client libraries auto-retry failed connections using a generated
idempotency key, and there are public GitHub issues (for example in
`stripe-ruby`) discussing exactly how a `409` during a same-key concurrent
retry should behave. That is the machine above, visible from the outside: a
retry that hits an in-flight key gets a conflict, and a retry that hits a
finished key gets the replay.

## 8. The retention and habit mechanic

This one is not a consumer dopamine loop. The "user" who comes back is the
developer, and the loop is trust.

The mechanic is reliability becoming a habit. Once Aarav has shipped retries
with idempotency keys and watched a sale night survive flaky networks with zero
double charges, he stops writing his own dedup logic forever. He reaches for
the key by reflex on every new POST. The official SDKs make it the default, so
the habit is baked in before he even decides to form it.

Which metric does it move? Revenue and retention, indirectly but strongly.
Every dropped connection that would have been a lost sale (because the dev was
too scared to retry) now safely completes. Every double charge that would have
been a refund, a chargeback, and a customer who never comes back is prevented.
Stripe processed well over a trillion dollars of total payment volume in a
recent year; at that scale even a tiny fraction of network blips turning into
double charges would be a mountain of refunds and lost trust. The feature's
job is to make "retry" the safe, boring default, and boring is exactly what you
want around money. The developer stays on Stripe because the scary part is
already solved.

## 9. The lesson for Rare.lab

Rare.lab compiles node graphs to shippable shader code and runs an embeddable
runtime. Two parts of that pipeline are foreign state mutations in disguise:
submitting a compile job to a build farm, and submitting a render or
GPU-bake job. Both can be slow, both run on someone else's machine, and both
can have their result lost on the way back. That is the exact shape of the
charge-the-card problem.

Concrete lesson: give every compile and bake request a content-derived
idempotency key, and build the pipeline as resumable atomic phases around it.

- The key should be a hash of the graph plus the compile settings. If a user
  hits "compile" twice on an unchanged graph, or a flaky editor reconnect
  re-sends the request, you do the expensive shader compile ONCE and replay the
  cached artifact. This is free caching that falls straight out of idempotency,
  and at scale it is the difference between recompiling 10 million identical
  graphs and serving 10 million cache hits.
- Model the pipeline as `started -> compiled -> baked -> published`, with each
  arrow committed atomically alongside its work, exactly like Brandur's
  `recovery_point`. If a GPU bake crashes halfway, a retry resumes at `compiled`
  instead of redoing the compile. Long render jobs are precisely where you do
  not want to start from zero.
- Scope and shard the key table by project, the way Stripe scopes by account.
  A key only has to be unique within a project, so the system shards linearly
  and never needs a global lock. The hot-project case on a launch day is your
  sale night; plan the split now.

The one-line takeaway: treat every expensive offline job (compile, bake,
publish) as a payment. Attach an idempotency key, checkpoint it as a forward
only state machine, and make "retry" the safe default. You get crash recovery
and a free artifact cache from the same design.

---

## Sources

- Brandur Leach, "Implementing Stripe-like Idempotency Keys in Postgres":
  https://brandur.org/idempotency-keys
- Article source on GitHub:
  https://github.com/brandur/sorg/blob/master/content/articles/idempotency-keys.md
- Brandur, "Rocket Rides Atomic" reference implementation:
  https://github.com/brandur/rocket-rides-atomic
- Stripe API reference, idempotent requests:
  https://docs.stripe.com/api/idempotent_requests
- Stripe docs, idempotency overview: https://stripe.com/docs/idempotency
- stripe-ruby issue on 409 retry behavior:
  https://github.com/stripe/stripe-ruby/issues/431
