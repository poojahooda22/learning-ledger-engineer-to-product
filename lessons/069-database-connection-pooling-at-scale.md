# Day 69 — A Postgres box ships with room for about 100 connections. How does an app whose serverless functions can fan out to tens of thousands of concurrent instances not fall over the first time it gets popular?

**Date:** 2026-08-31
**Difficulty:** Expert
**Topic:** Database connection pooling at scale: why a Postgres connection is not a cheap, disposable handle but a full operating-system process with real memory behind it, why "just raise max_connections" is not the fix it sounds like, and the mechanism (transaction-mode pooling, then a multi-tenant cloud-native pooler) that lets thousands of stateless, bursty clients share a database that was only ever built to hold a few hundred connections open at once.
**Stack relevance:** Rare.lab's own stack is Supabase Postgres with row-level security. Every save from the node-based editor, every manifest lookup the embeddable runtime triggers through a backend function, is a database connection somewhere in the chain. This ledger has already covered what happens when many tenants share one resource (Day 67, the noisy-neighbor problem) and what happens when one server owns too much blast radius (Day 34, cell-based isolation). This lesson is the mechanism underneath both of those, applied to the one resource Rare.lab cannot shard away: the Postgres connection slot itself. It also carries a trap specific to Rare.lab's exact setup, RLS policies that read identity out of a per-connection setting, that a generic pooling tutorial will not warn about.

---

## 1. The company and the breaking number

**Supabase, and the number that made them stop shipping someone else's pooler.** Supabase runs Postgres as a managed platform for a very large number of independent customer projects, each project getting its own database and, originally, its own PgBouncer instance running on the same box as that project's Postgres. PgBouncer, the open-source pooler nearly every Postgres shop reaches for first, is single-threaded: a busy instance saturates exactly one CPU core, no matter how many cores the machine has. As Supabase's project count grew, that single-threaded ceiling stopped being a rounding error and started being the thing standing between a customer and their database. In August 2023 Supabase published the fix they had to build themselves, Supavisor, and the benchmark number attached to its launch is the one worth sitting with: they drove it to 1,000,000 concurrent connections, using 20 AWS EC2 instances (16 cores, 32GB RAM each) as load generators, sustained over two-hour soak runs. That is not a number a single-threaded, single-box pooler was ever going to reach, and it is not a number a raw, unpooled Postgres server was ever going to reach either.

**The everyday version of the same wall: a Postgres default of 100.** `max_connections` in a stock Postgres install defaults to 100. That number is not arbitrary stinginess, it maps to a real resource: every Postgres connection is a forked operating-system process, not a thread and not a lightweight coroutine, with its own memory space. Andres Freund's widely cited 2020 measurement of a minimal, freshly started, read-only connection put the overhead at roughly 1.3 to 2 MiB with huge pages on, and around 7.6 MiB with huge pages off; in a real, long-lived connection running a real application's queries, per-connection memory climbs well past that as the connection's catalog caches (cached metadata about every table, index, and type it has touched) fill in, landing most operational guidance in the 5 to 10 MB range per connection, before that connection has run a single query for the request in front of it. At 200 idle connections that is already on the order of 1 to 2 GB of RAM spent before any actual work happens, and every one of those 200 is also a process the OS scheduler has to context-switch between, a cost that starts dominating query time well before you approach the connection ceiling itself, not just at it.

**Where the two numbers collide: a serverless fan-out.** Prisma's GitHub issue #5174, opened by developers hitting `FATAL: too many connections` in production, describes the mechanism plainly: a serverless platform (Lambda, Vercel, Cloudflare Workers) spins up an isolated runtime per concurrent request, and each isolated runtime that instantiates its own database client opens its own connection pool. Fifty concurrent function instances, each holding even a modest default pool of 9 connections, is 450 connections arriving at a database plan that, on a default install, has room for 100. A demo that felt instant at ten users hits `too many clients already` the first time a marketing email, a Hacker News front page, or a single popular customer sends a real burst of concurrent traffic, and it hits that wall in seconds, not minutes, because the wall is a hard connection-slot ceiling, not a gracefully degrading one.

---

## 2. Why the naive (demo) design dies

**The obvious version:** every request, whether it is a Vercel serverless function, a Lambda, or a plain backend process, opens its own direct connection to Postgres (or lets its ORM's connection pool do it once per process), runs its query, and either closes the connection or holds it in a small in-process pool for reuse. This is exactly what a tutorial teaches, because for a single long-running server process handling modest traffic, it works: one process, one pool, a handful of connections, comfortably under 100.

**Death one: serverless breaks the assumption that "the pool" is one shared, long-lived thing.** A connection pool inside a single process is a pool shared across requests *within that process*. Serverless architecture gives you no such guarantee: each concurrent invocation can be its own fresh process with its own fresh pool, and there is no shared memory between them for a pool to live in. The fix that works for a monolith, "just pool connections in the app," silently stops working the moment the app's own architecture is the thing fanning out to dozens or hundreds of concurrent copies of itself. The Prisma issue thread's other attempted fix, a singleton client meant to be reused across invocations, fails for the same underlying reason: serverless runtimes do not reliably keep that singleton warm across cold starts, so the "one pool per process" assumption quietly becomes "one pool per invocation" again under real traffic.

**Death two: raising `max_connections` does not scale the way it looks like it should.** The tempting next move is to just raise the limit, 100 to 500, 500 to 2,000. But every one of those connections is a forked OS process sitting in memory whether or not it is doing anything, at roughly 5 to 10 MB apiece for a connection that has actually done real work, plus its own entry in Postgres's shared lock tables and process array. Push past a few hundred simultaneously active backends and the cost stops being "a bit more RAM" and becomes CPU spent context-switching between hundreds of OS processes, cache and branch-predictor state thrown away on every switch, work that is pure overhead and competes directly with the CPU time your actual queries need. A database that "can't handle load" at 1,000 raw connections is very often a database whose CPU is busy scheduling processes, not busy running SQL.

**Death three: the pooler you bolt on can become the exact bottleneck you built it to remove.** This is Supabase's own story with PgBouncer: it is single-threaded, so a sufficiently busy instance saturates one core no matter how large the underlying machine is, and Supabase's original deployment ran that single-threaded pooler on the very same box as the Postgres server it was protecting, meaning the pooler was competing for compute with the database itself. A pooler that cannot itself scale horizontally, and that shares a fate with the database it fronts, just relocates the single point of contention one hop closer to the client instead of removing it.

---

## 3. The architecture

```
Clients (Rare.lab's node-based editor, backend functions triggered
by the embeddable runtime, any serverless request handler)
  - job: hold a database client and issue queries without needing to
    know or care how many other clients exist right now
  - analogy: a customer walking up to a bank teller window, with no
    idea how many other tellers or vaults are behind the counter

        |  (thousands to millions of these, opened and closed
        |   constantly, many living for one request's duration)
        v
Connection pooler, transaction mode (PgBouncer, or Supabase's own
Supavisor)
  - job: accept a huge number of short-lived client connections and
    hand each one a real backend Postgres connection ONLY for the
    duration of a single transaction, taking it back the instant that
    transaction commits or rolls back, so the same small set of real
    connections gets reused thousands of times a second
  - analogy: a hotel that has 2,000 registered guests but only 20
    room keys in circulation; a guest holds a key only while they are
    actually inside using the room, and hands it straight back at
    checkout so the next guest in line can use that same room

        |  (a small, fixed number of long-lived connections,
        |   comfortably under Postgres's own connection ceiling)
        v
Postgres primary
  - job: hold the actual data and do the actual query work, sized so
    its connection count stays in the hundreds, not the thousands,
    where per-connection process and memory overhead is a rounding
    error, not the dominant cost
  - analogy: the hotel's actual rooms; there are only so many, and
    the room key system exists specifically so the rooms themselves
    never have to scale to guest count

        |  (read traffic can additionally fan out here)
        v
Read replicas (streamed from the primary, queried for reads that
can tolerate a small replication lag)
  - job: absorb read-heavy load the primary alone could not, without
    adding write contention
  - analogy: photocopies of the hotel's guest ledger kept at a second
    desk, a moment behind the master copy, so a second line of
    guests can be served without touching the original

        |  (when a single pooler's own single-threaded ceiling,
        |   or a single project's fate, becomes the new bottleneck)
        v
Multi-tenant, cloud-native pooler layer (Supavisor: multi-threaded,
horizontally scaled across nodes, one deployment serving many
customer projects with per-tenant pool isolation)
  - job: scale the pooler itself the way the database's own
    connections were just scaled for the clients, so one customer's
    traffic spike cannot starve another customer's pool, and so the
    pooler is never back to being the single core, single box you
    started with
  - analogy: replacing one overworked concierge desk with a whole
    dispatch floor of concierges, each assigned their own set of
    hotels to manage, so one hotel's Friday-night rush never leaves
    every other hotel's guests standing in line
```

---

## 4. The transferable mechanisms

- **A connection is a stateful resource with a real cost, not a free handle.** Every mechanism in this lesson exists because a Postgres connection is a forked OS process with real memory and scheduling weight behind it, not a lightweight abstraction the way a plain function call or an HTTP request is. Any resource with this shape (OS threads, file descriptors, TCP sockets held open indefinitely) needs the same treatment: pool it, bound it, and never let client-side concurrency map one-to-one onto it.

- **Multiplex many cheap client-side connections onto few expensive backend connections.** Transaction-mode pooling is the general shape: the pooler holds a small, fixed set of real backend connections and lends one out only for the lifetime of a single transaction, not the lifetime of the client's session. This is the same idea as a load balancer terminating thousands of client TCP connections and reusing a small pool of upstream connections to the backend fleet, applied here to a stateful database connection instead of a stateless HTTP request.

- **Bound the expensive resource, and let the cheap resource queue in front of it.** The pooler's own `max_client_conn` can be set far higher than the database's real connection ceiling, because a queued client waiting a few milliseconds for a pool slot is cheap; an unbounded flood of raw connections hitting Postgres directly is not. This is Day 13's backpressure lesson in miniature: absorb a burst in a queue in front of the scarce resource, rather than letting the burst hit the scarce resource directly and take it down.

- **Scope shared, mutable state to the smallest unit that is actually safe to share.** Transaction-mode pooling works only because Postgres already gives you a primitive scoped exactly to the transaction: `SET LOCAL` (and `set_config(..., true)`), whose value is guaranteed to revert automatically at commit or rollback, before the pooler could ever hand that same backend connection to a different, unrelated client. Anything set with a plain session-level `SET` instead can silently leak from one client's transaction into the next client's, on the same backend connection, because that value has no natural expiry the pooler's connection-lending model respects.

- **Isolate tenants inside a shared pooling layer, not just inside a shared database.** Supavisor's per-tenant pool isolation is the same principle Day 67's multi-tenancy lesson names for shared compute and Day 34's cell-based architecture names for blast radius: a resource shared for efficiency (one pooler fleet serving every customer project) still needs hard boundaries between tenants, or one tenant's connection storm becomes every tenant's outage.

- **Fix the thing that cannot scale before it becomes the bottleneck everything else depends on.** Supabase did not tune PgBouncer harder, they replaced it, because a single-threaded process pinned to one core is a ceiling no amount of tuning removes. The senior move, recognizable from earlier ledger entries, is noticing when the mitigation itself has become the new single point of failure, and re-architecting that layer before it is forced by an outage.

---

## 5. The trade-offs

**Session features vs. connection reuse.** Transaction-mode pooling buys massive connection reuse by giving up anything that depends on state surviving across transactions on the same backend connection: SQL-level `PREPARE` statements, `LISTEN`/`NOTIFY`, session-scoped advisory locks (`pg_advisory_lock`, as opposed to the transaction-scoped `pg_advisory_xact_lock`), cursors held open with `WITH HOLD`, and plain `SET`. PgBouncer 1.21 (late 2023) added support for protocol-level prepared statements in transaction mode, closing part of this gap, but it only covers prepares issued through the extended query protocol, not text-level `PREPARE foo AS ...` SQL, which the pooler cannot see inside. Every one of these features still exists in Postgres; you are choosing not to lean on the ones that assume a client owns one backend connection for its whole session.

**Cost of a dedicated pool vs. blast radius of a shared one.** A pooler dedicated to one project, or one tenant, isolates that tenant's connection storm from everyone else's, at the cost of provisioning and running that pooler even when the tenant is quiet. A shared, multi-tenant pooler (Supavisor's model) amortizes that cost across many tenants, at the cost of needing real per-tenant isolation, exactly the guarantee Supavisor's per-tenant pool separation exists to provide, so that amortization does not become one tenant's Friday-night traffic starving everyone else's pool.

**Latency vs. safety, paid at the RLS boundary.** Supabase's own stack authenticates a request by writing the caller's JWT claims into a per-connection Postgres setting (`request.jwt.claims`, via `set_config(..., true)`) that row-level security policies then read with `current_setting(...)`. Under transaction-mode pooling this is safe only because that write is transaction-scoped: it is guaranteed gone by the time the pooler could hand the same backend connection to a different client. Writing that same identity with a plain session-level `SET` instead would be faster to reason about, and actively unsafe under pooling, since the setting could persist onto whichever unrelated client's transaction runs next on that same backend connection. This is not a hypothetical trade-off, it is the specific, documented hazard transaction-mode pooling carries for exactly the RLS pattern Rare.lab's own stack uses.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is a **retry-amplified connection storm**, a close cousin of the retry death spiral this ledger has named before, but aimed at a hard resource ceiling instead of a soft one. A traffic spike pushes concurrent serverless invocations up; each invocation opens its own fresh connection pool instead of reusing a shared one; the database's connection slots fill; new connection attempts start failing with `too many clients already`; and the client code's natural response to a failed connection attempt is to retry, immediately spinning up another invocation that tries to open yet another fresh connection. Each retry does not reduce the pressure on the scarce resource, it adds to it, which is exactly the shape of a metastable failure: the system cannot recover on its own once the loop starts, because the very behavior meant to recover (retry) is the behavior sustaining the failure, and the load does not need to keep growing externally for the outage to continue, the retries alone are enough to keep every connection slot full.

The naive fix, catching the error and retrying with a slightly longer backoff, does not break this loop, it only slows the rate at which it refills the same finite pool of connection slots; the loop is still shaped the same way, just paced differently. The structural fix is the one this lesson spends most of its time on: stop letting client-side concurrency map one-to-one onto backend connection count at all. A transaction-mode pooler with a bounded number of real backend connections turns "more concurrent clients" into "more clients queued for a pool slot" instead of "more connections opened against the database," which is the same move Day 13's backpressure lesson makes for request volume in general, applied here to the one resource, the connection slot, that a naive serverless architecture is most likely to flood without anyone writing a single line of retry logic on purpose.

---

## Sources

- [Supavisor: Scaling Postgres to 1 Million Connections, supabase.com](https://supabase.com/blog/supavisor-1-million): primary source for the 1,000,000-concurrent-connection benchmark, the 20-instance (16 core, 32GB RAM) load-generation setup, the two-hour soak-test methodology, and Supavisor's multi-threaded, multi-tenant, per-tenant-pool-isolated design as the successor to PgBouncer; direct fetch was blocked by this session's network egress policy, so details are drawn from search-indexed excerpts and corroborating secondary coverage (Elixir Merge, dev.to mirror) rather than a full read of the original post.
- [Measuring the Memory Overhead of a Postgres Connection, blog.anarazel.de](https://blog.anarazel.de/2020/10/07/measuring-the-memory-overhead-of-a-postgres-connection/): primary technical source (Andres Freund, a longtime Postgres core contributor) for the precise per-connection memory measurement of a minimal, freshly started, read-only connection: roughly 1.3 to 2 MiB with huge pages on, roughly 7.6 MiB with huge pages off; direct fetch was blocked by this session's egress policy, figures drawn from search-indexed excerpts and Hacker News discussion of the same post.
- [Resources consumed by idle PostgreSQL connections, AWS Database Blog](https://aws.amazon.com/blogs/database/resources-consumed-by-idle-postgresql-connections/): source for the general claim that idle Postgres connections consume nontrivial, measurable memory and CPU scheduling resources even while doing no query work, and for the framing that connection count itself, not just query load, is a first-class capacity variable; direct fetch blocked by egress policy, drawn from search-indexed summary.
- ["FATAL: too many connections" postgresql error in serverless production, GitHub Issue #5174, prisma/prisma](https://github.com/prisma/prisma/issues/5174): primary source for the serverless connection-storm mechanism (each concurrent function instance opening its own connection pool), the failed workarounds attempted in the thread (`connection_limit=1`, a client singleton that does not survive cold starts), and the general shape of the fix (an external pooler in transaction mode, or a proxy layer such as Prisma Accelerate). Fetched directly.
- [PgBouncer is now available in Supabase, supabase.com](https://supabase.com/blog/supabase-pgbouncer) and search-indexed coverage of Supabase's original PgBouncer deployment: source for the claim that Supabase originally ran a single-threaded PgBouncer instance co-located on the same box as each project's Postgres server, and that this became a scaling bottleneck as project count grew, the problem Supavisor was built to solve.
- [Transaction-mode pooler: are SET LOCAL GUCs inside an explicit transaction guaranteed isolated between clients (RLS claims)?, GitHub Discussion #47946, orgs/supabase](https://github.com/orgs/supabase/discussions/47946): source for the specific, documented mechanism by which Supabase's own stack sets RLS-relevant identity (`request.jwt.claims`) via a transaction-scoped `set_config(..., true)` / `SET LOCAL`, why that scoping is what makes it safe under transaction-mode pooling, and the explicit warning against using plain session-level `SET` for the same purpose. Accessed via search-indexed excerpt.
- Search-indexed secondary coverage of PgBouncer's transaction-mode limitations (prepared statements, advisory locks, `LISTEN`/`NOTIFY`, session-level `SET`), including PgBouncer's own GitHub issue tracker discussion of protocol-level prepared statement support added in version 1.21 (late 2023): corroborated across multiple independent write-ups (jpcamara.com, podostack.com, jailbreak of PgBouncer's own mailing list thread on prepared statements) and used here for the trade-offs section's list of what transaction pooling does and does not preserve.
- Day 13 (this ledger, backpressure and load shedding), Day 34 (cell-based architecture and shuffle sharding), Day 67 (multi-tenancy and the noisy neighbor problem): the ledger's own prior lessons this one builds directly on, for the queue-in-front-of-a-scarce-resource mechanism, the per-tenant blast-radius framing applied here to a shared pooling layer, and the retry-storm feedback loop reused here for connection exhaustion specifically.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of every supabase.com and aws.amazon.com page consulted, and of the anarazel.de primary post, so those figures are drawn from search-indexed excerpts and independent secondary corroboration rather than a full read of the original text. The figures this lesson leans on hardest, the 1,000,000-connection Supavisor benchmark and its load-generation setup, the default `max_connections` of 100, the per-connection memory range, and the Prisma serverless connection-storm mechanism, are each corroborated across multiple independent sources and treated as solid; the exact wording of Supabase's original co-located PgBouncer deployment and the precise CPU-saturation framing rest on fewer, search-indexed sources and are treated as the reported industry account rather than a fact verified against a single, fully read primary write-up.
