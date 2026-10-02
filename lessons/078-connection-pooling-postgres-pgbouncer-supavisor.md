# Day 78: Why does a database with 8 cores fall over at 500 connections, and how do 100,000 clients share it? Connection pooling with PgBouncer, Supavisor and HikariCP

**Date:** 2026-10-02
**Difficulty:** Intermediate to Advanced (a queueing-theory result that surprises people, plus the traps of transaction pooling)
**Topic:** Connection pooling. Why connections are the scarcest resource in Postgres, the three pooling modes, how to size a pool, and the feedback loop that turns a slow query into an outage.
**Stack relevance:** Rare.lab runs on Supabase Postgres. Every request from an edge function or serverless worker is a connection. Section 7 maps what you already get for free and where the trap is.

---

## 0. The framework (same six steps as Day 74)

1. Functional: many app instances run SQL against one Postgres. Each request needs a database session for a few milliseconds.
2. Non-functional: stay up when app instances scale out 10x, keep p99 latency flat, never run out of connections.
3. Entities: Client, App instance, Pooler, Server connection (a Postgres backend), Pool.
4. API: `connect()`, `BEGIN ... COMMIT`, `query()`. The pooler speaks the Postgres wire protocol so apps change only a connection string.
5. Naive design: every request opens its own connection, or every app instance holds a big private pool.
6. Deep dives: the cost of one connection, Little's law for sizing, session vs transaction vs statement pooling, serverless fan-in, the retry death spiral.

---

## 1. The company and the breaking number

**Supabase, Postgres, and one number: a typical Postgres server is happiest with a few dozen active connections, and a serverless app can ask for 10,000.**

- Postgres forks one **operating system process per client connection** (the "process per connection" model). An idle backend costs roughly **1.5 to 2 MB** of private memory fresh, and **5 to 15 MB** once it has run queries and cached catalog data (figures from AWS and Postgres community write-ups, see Section 9). At `max_connections = 500` and 10 MB each, that is **5 GB of RAM** before a single query runs.
- Memory is not even the main wall. Andres Freund (Postgres committer, then at Microsoft) showed in 2020 that **snapshot building** (`GetSnapshotData()`, the step where Postgres works out which transactions each query can see) scaled with the number of connections, so thousands of mostly idle connections slowed every active query. His snapshot work shipped in **Postgres 14**.
- Supabase built **Supavisor**, a pooler in Elixir, because its older pooler **PgBouncer is single threaded** and could not fan in the load of a platform hosting many projects. Their August 2023 test held **1,003,200 simultaneous client connections across two 64 core instances**, all funneled into a small number of real Postgres connections.

Analogy: a restaurant kitchen with 8 stoves. Seating 10,000 guests does not need 10,000 chefs. It needs a host who hands each guest to a free chef for the two minutes they actually need cooking. A connection pool is the host.

---

## 2. Why the naive design dies

**Naive version A: one connection per request.**
- Opening a connection is slow. TCP handshake, TLS handshake, authentication, and a fork of a new Postgres process. Rough range (my estimate, varies by network): **20 to 100 ms** before the first query. Your query takes 2 ms. You paid 50x overhead.
- Real example: a serverless function on Cloudflare Workers or Vercel handling 2,000 requests per second would open and close 2,000 connections per second. Postgres spends its CPU forking and tearing down processes.

**Naive version B: every app instance keeps its own pool of 20.**
- 50 instances x 20 = 1,000 connections. Scale to 500 instances during a spike: **10,000 connections**. Postgres hits `max_connections` and answers `FATAL: sorry, too many clients already`. New requests fail while old connections sit idle.
- Autoscaling makes it worse: the more load, the more instances, the more connections, the less capacity per connection.

**Naive version C: "just raise max_connections".**
- More backends fighting for 8 cores means more context switching, more lock contention, more snapshot cost (Freund). Throughput goes **down**, not up, past a point. This is the part people find surprising.

---

## 3. The architecture

```
Clients (browsers, mobile apps, 100,000 users)
        |   HTTPS
        v
Edge / CDN (Cloudflare)              job: absorb static + cached reads, never touch the DB
        |
        v
Load balancer                        job: spread requests across app instances
        |
        v
Stateless app tier (N instances,     job: run business logic. Holds NO state, and only
 serverless or containers)           a SMALL local pool (or none)
        |   thousands of short client connections
        v
Pooler tier (PgBouncer / Supavisor / pgcat / RDS Proxy)
                                     job: multiplex many client sessions onto few server
                                     connections. The host with the seating chart.
        |   ~20 to 100 real server connections
        v
Postgres primary                     job: execute queries. Sized so active backends ~ cores
        |
        +--> Read replicas           job: absorb reads (each replica has its own pooler path)
```

Per layer, one sentence plus analogy:
- **Edge/CDN:** the lobby kiosk. Answers questions without bothering the kitchen.
- **Load balancer:** the doorman splitting guests across dining rooms.
- **App tier:** waiters. Many of them, cheap to add, each holds no kitchen state.
- **Pooler:** the host. Only so many guests are in the kitchen at once, the rest wait politely in a queue.
- **Postgres:** the kitchen. Fast when not crowded, slow when crowded.

---

## 4. The transferable mechanisms

### 4.1 Multiplexing (many logical, few physical)
A pooler accepts, say, 10,000 client connections and owns, say, 50 server connections. A client only holds a server connection while it actually needs one. Most clients, most of the time, are waiting on the network, the user, or other code. That idle time is what you reclaim.

### 4.2 Pool sizing via Little's law (the counterintuitive one)
**Little's law:** `concurrency in the system = arrival rate x time each request spends in it`.
- Worked example: 2,000 queries per second, each takes 5 ms on the database. Busy connections needed = 2,000 x 0.005 = **10**. Not 2,000.
- The HikariCP guide (from the PostgreSQL community wiki, quoted widely) gives a starting rule: `connections = (core_count x 2) + effective_spindle_count`. For an 8 core server on SSD, about **17**. For many workloads with the working set cached in RAM, closer to `cores x 2`.
- The HikariCP wiki also shows a well known Oracle demo (recalled from memory, not re-verified in this session): cutting a pool from thousands of connections to about 96 dropped response times dramatically, because the database stopped thrashing. Treat the exact numbers as unverified.
- Why smaller wins: queries that are all CPU bound cannot run faster by being run at once on 8 cores. Extra connections only add context switches and waiting.

### 4.3 The three pooling modes (what the connection is "leased" for)
- **Session pooling:** a client keeps a server connection until it disconnects. Safe for everything (temp tables, `SET`, advisory locks, `LISTEN`). Saves almost nothing under serverless, because clients hold on.
- **Transaction pooling:** a server connection is leased for one transaction, then returned. The sweet spot. 10,000 clients can share 50 server connections. **Cost:** session state does not follow you. Things that break: session level `SET`, temp tables, `LISTEN/NOTIFY`, session advisory locks, and classically **prepared statements** (a statement prepared on server connection A does not exist on B). **PgBouncer 1.21 (2023)** added protocol level prepared statement tracking when `max_prepared_statements` is non zero, which fixes much of this.
- **Statement pooling:** lease per single statement. Multi statement transactions are forbidden. Rare.

### 4.4 Queueing instead of failing (waiting is a feature)
When all server connections are busy, a good pooler **queues** the client for a short time rather than erroring. A 10 ms wait in a queue beats a failed request. This is the queue as shock absorber from Day 9 applied to connections. Set a **max wait** (for example 2 to 5 seconds) so the queue itself cannot grow without bound.

### 4.5 Fan-in for serverless
Serverless functions cannot share an in-process pool: each cold start is a new process. The pooler becomes the shared pool. This is why Supabase gives every project a pooler connection string and recommends it for serverless and edge code.

### 4.6 Separate pools by workload
One slow analytical query in the same pool as 1,000 fast checkout queries steals connections from them. Give batch jobs, migrations and dashboards their own small pool with their own cap. This is the bulkhead idea from Day 67 (noisy neighbor), applied to connections.

---

## 5. The trade-offs

**CAP per data type.** Pooling is mostly orthogonal to CAP, but it changes failure behavior:
- **Writes and transactions:** consistency first. Transaction pooling still gives full ACID inside one transaction. What you lose is session convenience, not correctness.
- **Reads that can be stale:** route to replicas, accept replication lag (Day 31, session guarantees).
- **During overload:** the pooler chooses **availability for some, rejection for others**: admit up to the cap, queue briefly, then fail fast. Failing 5% of requests quickly beats making 100% slow.

**Cost versus latency.**
- A pooler adds one network hop, about **0.2 to 1 ms** in the same region (my estimate). In exchange it removes 20 to 100 ms connection setup.
- A pooler is a new thing to run and a new single point of failure. Run two behind a load balancer, or use the managed one.
- Smaller pools cost nothing but can feel wrong. Bigger pools feel safe but are slower under load.

**Choice made by Supabase:** transaction mode pooling on the shared pooler, direct connections for long lived servers and migrations, and a pool size capped by the instance's compute size.

---

## 6. The systems-thinking lens

**The loop: slow queries hold connections longer, the pool runs dry, requests queue, clients time out and retry, retries add load, queries slow further. This is a retry death spiral, a form of metastable failure.**

```
One slow query (lock wait, bad plan, cache miss)
    -> connection held 10x longer (Little's law: busy = rate x time)
    -> pool exhausted, new requests wait
    -> app timeouts fire, clients RETRY
    -> arrival rate rises, more queries, more waiting
    -> database slower, connections held even longer
    -> stays broken even after the original slow query is gone
```

Worked numbers: pool of 20, normal query 5 ms, capacity = 20 / 0.005 = **4,000 queries per second**. One bad deploy makes queries take 100 ms. Capacity falls to 20 / 0.1 = **200 per second**, a 20x drop, with no change in hardware. Traffic of 1,000 per second now exceeds capacity by 5x, and the queue grows forever.

The senior fix breaks the loop, it does not add connections (adding connections makes the database slower, Section 2C):
- **Statement timeout** (`statement_timeout`, `idle_in_transaction_session_timeout`) so no query holds a connection forever.
- **Bounded queue with a max wait**, then **fail fast** (load shedding, Day 13).
- **Retries with exponential backoff plus jitter**, and a **retry budget** (for example no more than 10% extra load from retries). Idempotency keys (Day 12) make retries safe.
- **Circuit breaker** in the app: after N failures, stop calling the database for a few seconds and serve cached or degraded responses.
- **Separate pools** so one workload cannot starve another.
- **Cache reads** (Day 19) so fewer requests need a connection at all.

---

## 7. Map to Rare.lab's stack

Rare.lab: Supabase Postgres with RLS, Cloudflare R2 (content addressed immutable scene JSON plus manifest), an embeddable runtime with one shared WebGL context.

| Pattern | Rare.lab touchpoint | Status | Action |
|---|---|---|---|
| Edge/CDN absorbs reads | Scenes served from R2 by content hash, immutable | Already doing this, and it is the biggest win | The runtime never needs Postgres to play a scene. Keep it that way: no per-view DB call. |
| Multiplexing pooler | Supabase gives a Supavisor string (transaction mode) | Available, check you use it | Any Cloudflare Worker or edge function must use the **pooled** string, never the direct one. |
| Transaction mode caveats | Prepared statements, `SET` | Likely a trap | RLS often relies on `SET LOCAL request.jwt.claims` style settings. `SET LOCAL` inside a transaction is safe in transaction mode, a plain session level `SET` is not. Supabase's own client sets claims per request, so check any custom pg driver you add. |
| Statelessness | Editor API tier | Yes | Keep it, so adding instances only adds pooler clients, not DB connections. |
| Read caching | Scene metadata, manifest | Partly | The manifest is one small object. Cache it at the edge with a short TTL. |
| Pool sizing | Free to Pro compute tiers | Check | On a small instance (2 cores), a healthy pool is single digits to a couple dozen, not hundreds. |
| Timeouts and backoff | Editor autosave | Unknown | Autosave retries are the classic spiral starter. Add backoff with jitter and idempotent writes keyed by content hash (the key is already the checksum, Day 23). |

**Where the next ceiling is (inference, not measured):** not data size. The first wall is **RLS cost per query plus connection concurrency on a small instance**. Policies that call functions or subselects run on every row. If saves from many editors arrive together (a launch, a shared template going viral), the pool fills with slightly slow queries and the Section 6 loop starts. Order of fixes: (1) pooled string everywhere, (2) statement timeouts, (3) index the columns your RLS policies filter on, (4) edge cache the manifest, (5) read replica for dashboards, (6) bigger instance, (7) only then shard (Day 77).

**One-line lesson for Rare.lab:** size the pool to the database's cores, not to your users, and put a timeout, a bounded queue and jittered retries in front of it, because the outage is never the first slow query, it is the retries that follow.

---

## 8. What is inside the video you shared (recap, already covered)

The "Design LeetCode" mock interview is fully taught in [Day 74](074-leetcode-online-judge-end-to-end.md), so I did not repeat it. Its spine, for your notes: (1) functional requirements, then explicit out of scope (auth, payments, analytics), (2) non functional (availability over consistency, low latency, isolation for untrusted code, 100k concurrent in contests, no single point of failure), (3) core entities, (4) REST API, (5) a basic design per requirement, (6) fixes in layers: containers instead of the API server or VMs, a queue between API and runners with exponential retry, a cache for the leaderboard instead of polling, replicas with failover for the submissions database. Link to today: in that design, many API servers and runner workers all hit the submissions database. Without a pooler and a queue, a contest start would exhaust connections exactly as in Section 2B. The queue she added is the same shock absorber as the pooler's wait queue.

---

## 9. References and what is actually in them

**Honest note on access:** the network proxy blocked supabase.com in this session. The items below were identified through web search excerpts, not read in full. Numbers are as reported there; verify before citing.

- [Supavisor: Scaling Postgres to 1 Million Connections](https://supabase.com/blog/supavisor-1-million) (Supabase, Aug 2023). Why Supabase replaced PgBouncer (single threaded), written in Elixir with José Valim and Dashbit, test of 1,003,200 connections across two 64 core instances. Also a [dev.to copy](https://dev.to/supabase/supavisor-scaling-postgres-to-1-million-connections-n70).
- [Supavisor 1.0](https://supabase.com/blog/supavisor-postgres-connection-pooler) and the [GitHub repo](https://github.com/supabase/supavisor). Multi-tenant cloud native pooler, now behind every Supabase project's pooled string, with plans for query caching and read replica balancing.
- [Analyzing the Limits of Connection Scalability in Postgres](https://www.citusdata.com/blog/2020/10/08/analyzing-connection-scalability/) and [Improving Postgres Connection Scalability: Snapshots](https://www.citusdata.com/blog/2020/10/25/improving-postgres-connection-scalability-snapshots/) (Andres Freund). The deep read: memory per connection, why idle connections still cost, and the snapshot fix in Postgres 14. Also on [Microsoft TechCommunity](https://techcommunity.microsoft.com/t5/azure-database-for-postgresql/analyzing-the-limits-of-connection-scalability-in-postgres/ba-p/1757266).
- [Resources consumed by idle PostgreSQL connections](https://aws.amazon.com/blogs/database/resources-consumed-by-idle-postgresql-connections/) (AWS). Source of the per-connection memory ranges.
- [Scaling Connections in Postgres](https://www.citusdata.com/blog/2017/05/10/scaling-connections-in-postgres/) (Citus). The earlier primer on process per connection and pooling.
- [PgBouncer FAQ](http://www.pgbouncer.org/faq.html) and [PgBouncer on PlanetScale docs](https://planetscale.com/docs/postgres/connecting/pgbouncer). Pooling modes, what breaks in transaction mode, `max_prepared_statements` since 1.21.
- [HikariCP: About Pool Sizing](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing) and a [summary of the formula](https://goldlapel.com/grounds/connection-pooling/hikaricp-pool-sizing-postgres). The `cores x 2 + spindles` rule and the "smaller pool is faster" argument. The Oracle demo numbers in Section 4.2 come from memory of this page.
- Practitioner pieces, lower authority: [Postgres Connection Pool Sizing: How PgBouncer Saved My Launch](https://www.rabinarayanpatra.com/blogs/postgres-connection-pool-pgbouncer-survival-guide), [PgBouncer Pooling Modes: Session vs Transaction](https://www.matthewswong.com/en/blog/pgbouncer-connection-pooling-modes/).

**Inference, labeled:** connection setup latency range (20 to 100 ms), pooler hop cost (0.2 to 1 ms), the worked Little's law examples, the death-spiral diagram (components are real, the framing is mine), and everything in Section 7 about Rare.lab ceilings and RLS behavior.

**Related ledger lessons:** Day 9 (queue as shock absorber), Day 12 (idempotency), Day 13 (backpressure and load shedding), Day 19 (caching), Day 23 (content addressed storage), Day 31 (session guarantees), Day 67 (noisy neighbor), Day 74 (LeetCode end to end), Day 77 (live resharding).
