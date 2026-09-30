# Day 76: How does a service made of healthy servers still feel slow to users? Tail latency, hedged requests, and the power of two choices

**Date:** 2026-09-30
**Difficulty:** Advanced (probability, queueing, and control loops in one lesson)
**Topic:** Tail latency at fan-out scale. Why p99 is the normal experience, how hedged requests cut it, how power-of-two-choices load balancing avoids building the tail, and the retry and hedge feedback loop that can turn a slow day into an outage.
**Stack relevance:** Rare.lab's runtime has a tail problem too, in milliseconds per frame instead of per request. Section 7 maps it.

---

## 0. The framework (same six steps as Day 74)

1. Functional requirements: a user request fans out to many backends and returns one merged answer (search results, a feed page, a scene load).
2. Non-functional: p99 and p99.9 latency targets, not averages. High availability. Bounded extra load.
3. Entities: Request, Backend replica, Latency distribution, Hedge budget.
4. API: `call(service, request, deadline)`. The deadline travels with the request.
5. Naive design: pick one replica (round robin or random), wait for the answer, retry on timeout.
6. Deep dives: fan-out math, hedging, load balancing choice, retry budgets, the metastable trap.

---

## 1. The company and the breaking number

**Google Search and Bigtable, from the paper "The Tail at Scale" (Jeff Dean and Luiz Andre Barroso, Communications of the ACM, 2013). One number: a 1% slow rate per server.**

Worked arithmetic (this is plain probability, not a company figure):
- One server answers slower than 1 second for 1 in 100 requests. Looks fine.
- A user request waits on **100** such servers in parallel. It is fast only if ALL 100 are fast.
- P(all fast) = 0.99^100 = 0.366. So **about 63% of user requests take over 1 second.** This is the figure the paper gives.
- Make each server 100 times better: 1 in 10,000 slow. With **2,000** servers, P(at least one slow) = 1 - 0.9999^2000 = about 18%. Nearly one user request in five, and the paper says the same.

The breaking number is not load or bandwidth. It is **fan-out multiplied by a small per-server slow rate.** Every box can be "99% healthy" and the user still sees slowness most of the time. Analogy: a relay race with 100 runners. Each runner trips one time in a hundred. The team almost always has someone who tripped.

---

## 2. Why the naive design dies

**Naive version:** the frontend picks a replica at random, waits, and on timeout retries.

- **It waits on the slowest.** The response time of a fan-out is the MAX of its parts, not the average. Averages hide this. A dashboard showing a 10 ms mean can sit on top of users waiting 1 s. Analogy: a group dinner cannot start until the last guest arrives.
- **Random choice piles up queues.** Pure random assignment sometimes sends 3 requests to one server and 0 to its neighbor. The unlucky server has a queue. Queue wait is the tail. Analogy: choosing a supermarket checkout lane by closing your eyes.
- **Timeout-then-retry is too late.** A 1 s timeout means the user already lost 1 s before the retry even starts. And the retry adds load to a system that may already be struggling.
- **The causes are everywhere and unfixable one by one.** The paper lists shared resources, background daemons, garbage collection, maintenance, queueing, power limits, and in flash storage, periodic garbage collection. You cannot remove all of them. You have to build a system that stays fast while some part is always slow. This is called being **tail tolerant**.

---

## 3. The architecture

```
Client
   |
   v
Frontend / root aggregator           <- owns the DEADLINE and the hedge budget
   |  fans out to N leaf shards in parallel
   |-----------------------|-----------------------|
   v                       v                       v
Shard 1                 Shard 2       ...       Shard N
 (3 replicas)            (3 replicas)            (3 replicas)
   ^
   |
Per-shard replica picker:  power of two choices
   (sample 2 replicas, send to the one with fewer in-flight requests)
   |
   +--> if no answer after p95 latency: send ONE hedge to another replica,
        take the first reply, cancel the loser
   |
   +--> hedges and retries draw from a TOKEN BUCKET (about 5% of traffic)
   |
   v
Merge: answer with what you have by the deadline (partial results allowed)
```

Job of each layer:
- **Root aggregator:** splits the request, sets one absolute deadline for all children, merges. Analogy: a head waiter who tells every kitchen station "plates out by 8:15, no matter what."
- **Replica picker (P2C):** avoids building queues in the first place. Analogy: glance at two checkout lanes, join the shorter one.
- **Hedger:** insures against the one replica that is stuck. Analogy: if your first taxi has not arrived in the time 95% of taxis take, call a second and cancel whichever comes late.
- **Token bucket:** caps the extra load. Analogy: a spending limit on the insurance.
- **Partial-result merge:** fast answer from 99% of shards beats a perfect answer in 3 s.

---

## 4. The transferable mechanisms

**1. Hedged requests.**
- Send the request to one replica. If no reply after the p95 latency of that request class, send a copy to a second replica. Use the first reply, cancel the other.
- Why delay? Sending both always doubles load. Waiting until p95 means only about 5% of requests get a copy.
- Paper result, for Bigtable reading 1,000 keys spread across a 1,000-node cluster: hedging after a 10 ms delay cut the 99.9th percentile from **1,800 ms to 74 ms** while sending only **2% more** requests. Small extra work, huge tail win.
- Works when the slowness is about the server (random stalls), not about the request (a genuinely heavy query). A hedge for a heavy query just makes two replicas do heavy work.

**2. Tied requests.**
- The paper's refinement. Send to two replicas at once, each told about the other. When one STARTS the work, it tells the other to drop it. This removes most of the duplicate work without waiting for p95.
- Needs a way to cancel queued work cheaply. Analogy: two cooks both get the order ticket, the first to pick up the pan shouts "mine" and the other tears up the ticket.

**3. Power of two choices (P2C).**
- Randomly sample TWO servers, send to the one with fewer outstanding requests.
- Mitzenmacher's result (the "supermarket model"): with n servers, purely random placement gives a longest queue that grows like log n / log log n. Two choices gives log log n. That is an exponential improvement. Three choices is only a constant factor better than two.
- Why not just check ALL servers and pick the least loaded? Because load info is stale, and everyone picks the same "least loaded" server at once (a herd). Two random samples spread that out. It is cheap and needs no global state.
- Real-world use: Envoy, NGINX and HAProxy offer this style of balancing (from memory, check their docs), and many RPC libraries use it.

**4. Retry budgets and jittered backoff.**
- Amazon Builders' Library ("Timeouts, retries, and backoff with jitter"): set a timeout on every remote call, use capped exponential backoff, and add **jitter** (random delay) so a thousand clients do not retry at the same instant.
- Budget idea: allow retries and hedges only up to a percentage of normal traffic (for example 10%). When everything is slow, the budget runs out and extra load stops. This is exactly what stops hedging from going wrong.

**5. Deadline propagation and partial results.**
- Pass one absolute deadline down the whole call tree. A child that cannot finish in time returns nothing instead of working on a result nobody will read.
- Google's "good enough" responses: return results from the shards that answered and mark the rest as missing.

**6. Micro-partitioning and selective replication (from the paper).**
- Split data into many more partitions than machines (for example 20 per machine) so load moves in small pieces and recovery is fast.
- Replicate extra copies of hot partitions only. Same tail-tolerance idea, applied to data placement.

---

## 5. The trade-offs

- **Latency vs cost.** Hedging buys a lower tail for about 2 to 5% more backend work. Cheap when spare capacity exists, dangerous near saturation.
- **Consistency vs availability, per data type.**
  - Reads of search results or a feed: serve partial or slightly stale results. Availability and speed win.
  - Writes (payments, inventory): do NOT hedge blindly. A duplicated write executes twice unless it carries an **idempotency key** (Day 12). Hedge reads freely. Hedge writes only when idempotent.
- **Simple vs smart.** P2C is a few lines and needs no coordination. Least-loaded-of-all needs global state and herds. The simple one wins.
- **Completeness vs speed.** Partial results are a product decision. Search can drop 1 shard of 1,000. A bank balance cannot.

---

## 6. The systems-thinking lens

**The feedback loop: the retry and hedge amplification spiral (a metastable failure).**

1. A backend slows slightly (a deploy, a hot shard, a GC pause).
2. Clients hit their timeout and retry. Hedges fire more often because more requests exceed p95.
3. Offered load rises, say from 1.0x to 1.3x. The backend is now slower still.
4. Even more timeouts and hedges. The system is now busy with duplicate work and almost no user-visible work completes.
5. Even after the original trigger is gone, the load from retries keeps it down. That is what **metastable** means: the bad state sustains itself.

Adding capacity is the weak fix. The senior fix **breaks the loop**:
- **Retry and hedge budgets (token bucket):** cap extra load at about 5 to 10% of traffic. When all requests are slow the bucket empties and hedging stops by itself.
- **Adaptive hedge delay:** compute p95 continuously. If everything is slow, p95 rises, so hedges fire less.
- **Load shedding and deadlines (Day 13):** drop work whose deadline has passed.
- **Circuit breakers:** stop calling a replica that is clearly failing.
- **Jitter:** break synchronization so retries do not arrive as one wave.

Name to remember: hedging is safe only when it is **self-limiting**.

---

## 7. Map to Rare.lab's own stack

Rare.lab's tail is per frame, not per request. A 60 fps page has a 16.7 ms budget. The fan-out is the number of effects on the page sharing ONE WebGL context. One slow effect is the slowest shard.

| Pattern | Rare.lab equivalent | Already have? | Next ceiling |
|---|---|---|---|
| Fan-out max latency | A page with 10 embedded effects: frame time is the SUM on one GPU, and any one overrun drops the frame | Partly (one shared context avoids duplicate GPU setup) | Measure per-effect GPU time with timer queries and track p95 and p99 of FRAME time, not the average. The dashboard average will lie, exactly as in Section 2. |
| Deadline plus partial results | Per-frame budget with graceful degrade | No | If an effect overruns its slice, render it at half resolution or skip it this frame rather than miss the frame. This is partial results. |
| Retry and hedge budget | Loading scene JSON and the manifest from Cloudflare R2 | Immutable, content-addressed objects are ideal for hedging (safe, idempotent, cacheable) | Hedge a slow scene fetch after p95 with one extra request, capped by a small budget. Immutable GETs make this risk-free for correctness. |
| P2C replica choice | Any future compile or render worker pool | Not yet needed | Use "sample two workers, pick the less busy" instead of round robin. Cloudflare Durable Objects or Queues can hold the in-flight counts. |
| Pooling to avoid queue tail | Supabase Postgres through its pooler | Yes | Connection pool exhaustion is a hidden queue. Watch pool wait time as a p99 metric, and set a statement timeout and a deadline on every query so one slow query cannot hold a connection. |

**One-line lesson for Rare.lab:** instrument p99 frame time and p99 scene-load time now, then give every effect a per-frame budget with a degrade path. Averages will say "fast" while the tail says "janky".

---

## 8. What is inside the video you shared in this task (recap, already covered)

The mock interview "Design LeetCode" is fully taught in [Day 74](074-leetcode-online-judge-end-to-end.md): the six-step framework, the sandboxed workers, the queue, and the Redis leaderboard. Today applies the same six-step spine (Section 0) to a new system and does not repeat that topic. The one link to today's lesson: the judge's "exponential retry" comment in the video is exactly the mechanism Section 6 warns about. Retries need jitter and a budget, or they become the outage.

---

## 9. References and what is actually in them

**Honest note on access:** I used web search result excerpts only. I could not open full papers in this environment, so quoted numbers come from the search excerpts and from my memory of the paper. Mark anything you plan to cite and check it against the PDF.

- [The Tail at Scale (Dean and Barroso, CACM 2013), summary by The Morning Paper](https://blog.acolyer.org/2015/01/15/the-tail-at-scale/). Plain summary of the paper. Key content: the 63% and 18% fan-out arithmetic, hedged requests, tied requests, micro-partitions, selective replication, and "good enough" partial results. Read the paper itself after this.
- [The Tail at Scale, other summaries: Inside Out](https://codito.in/paper-the-tail-at-scale/) and [Speaker Deck slides](https://speakerdeck.com/pigol1/the-tail-at-scale). Shorter retellings, good for a first pass.
- [Request Hedging: The Tail-at-Scale Technique Most Teams Skip (DEV Community)](https://dev.to/gabrielanhaia/request-hedging-the-tail-at-scale-technique-most-teams-skip-2j37) and [Stragglers, Not Failures: Adaptive Hedged Requests (InfoQ)](https://www.infoq.com/articles/adaptive-hedged-requests-p99-latency/). Practitioner articles. Key ideas: delay hedges until p95, cap them with a token bucket of a few percent, and hedging stops automatically when everything is slow. Treat exact percentages as their claims.
- [The Power of Two Choices in Randomized Load Balancing (Mitzenmacher, IEEE TPDS 2001)](https://www.eecs.harvard.edu/~michaelm/postscripts/tpds2001.pdf), from Harvard. The proof behind "two choices is exponentially better than one, three is only a constant better than two." Also the survey [The Power of Two Random Choices](https://www.researchgate.net/publication/2463849_The_Power_of_Two_Random_Choices_A_Survey_of_Techniques_and_Results).
- [Amazon Builders' Library: Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter). Amazon's own rules: timeouts on every call, capped exponential backoff, jitter so retries do not synchronize. Plain summary at [Lumigo](https://lumigo.io/blog/amazon-builders-library-in-focus-1-timeouts-retries-and-backoff-with-jitter/).
- [Netflix on stateful systems (InfoQ talk)](https://www.infoq.com/presentations/netflix-stateful-cache/). From the excerpt: Netflix uses hedged requests, exponential backoff, concurrency limits and per-namespace objectives in its cache tier. Full talk not read.

**Inference, labeled:** the Rare.lab frame-time mapping, the 5 to 10% budget values, and the use of P2C inside Rare.lab are my reasoning, not published figures. The Envoy, NGINX and HAProxy P2C support is from memory.

**Related ledger lessons:** Day 8 (rate limiting), Day 9 (queue as shock absorber), Day 12 (idempotency), Day 13 (backpressure and load shedding), Day 16 (hot keys), Day 34 (cells and bulkheads), Day 71 (distributed rate limiting), Day 72 (chaos engineering).
