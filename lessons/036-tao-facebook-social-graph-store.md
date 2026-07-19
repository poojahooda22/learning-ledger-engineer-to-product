# Day 36: How do you serve a billion reads a second against a graph that gains one new edge at a time, without every single edge forcing a full-list refetch?

*2026-07-19*

---

## 1. The company and the number that breaks a naive design

**Facebook, TAO ("The Associations and Objects" store), in production since 2012, published as a paper at USENIX ATC in 2013.** TAO is the data store sitting behind the entire social graph: friend lists, likes, comments, checkins, tags, and every "who is connected to what" query that News Feed, profiles, and notifications all fire constantly. The paper's own headline numbers: TAO handles **over 1 billion reads a second and millions of writes a second**, at a **96.4% in-memory cache hit rate**, spread across **hundreds of thousands of shards** holding many petabytes of graph data, replicated to multiple geographic regions with typical replication lag **under one second**, and a measured failure rate over a 90-day production window of about **4.9 x 10^-6 queries** (roughly 99.9995% of queries succeed).

The number that actually breaks a naive cache is not the raw QPS, it is the *shape* of the query underneath it. The paper's own running example: Alice checks in at the Golden Gate Bridge and tags Bob there, Cathy comments on the checkin, David likes it. Answering "show me Bob's friends," "how many people liked this checkin," or "give me the last 20 comments, paginated" is not "fetch one value by one key." It is a query over a *list that keeps growing one edge at a time, forever, read far more often than it's written*. That mismatch, a single-value cache asked to serve a live, orderable, incrementally-growing collection, is what a plain key-value cache cannot do without either being wrong or being wasteful.

## 2. Why the naive design dies

The naive version was Facebook's actual pre-TAO architecture: PHP/Hack web servers reading and writing MySQL directly, with memcache sitting in front as a look-aside cache, and every application call site responsible for its own read-fill-cache and write-invalidate-cache logic. It collapses in three concrete ways.

**a. A key-value cache can only cache a list as one blob, so one new edge invalidates the whole thing.** Memcache stores `key -> value`. Bob's friend list has to be cached as one serialized value under one key. The moment a single friend request is accepted, that one new edge means the *entire* cached blob is stale, so the entire list has to be dropped and refetched from MySQL on the next read, even though 4,999 of Bob's 5,000 friends did not change. On a popular object, that is not one refetch, it is a burst of near-simultaneous full-list re-reads landing on the same MySQL rows the instant a busy profile's list changes.

**b. Look-aside logic duplicated across every call site is a correctness liability, not just an inefficiency.** With no dedicated data-store layer, every one of thousands of PHP call sites across every product team has to hand-write its own "check cache, miss, read DB, write cache, and on write, remember to invalidate" logic. There is no single place enforcing that invalidation actually happens correctly on every write path, which is exactly the kind of duplicated, easy-to-get-wrong logic that produces real production bugs, like counts and comment counts drifting stale in some code paths but not others.

**c. Cross-region reads either round-trip an ocean or need bespoke regional caching per team.** A MySQL master for a given shard of the graph lives in exactly one region. A memcache-only design run from a distant region either does a live round trip back to the home region's database on every cache miss, adding hundreds of milliseconds to a request that should be instant, or it requires every feature team to bolt on its own ad hoc regional cache and invalidation scheme, which does not scale to "every team building their own copy of graph caching."

The analogy: imagine every store clerk in a national chain keeping a private notebook of "who's on the VIP list," instead of everyone reading the same shared, always-current ledger. The moment head office adds one VIP, every clerk's notebook is now wrong in its entirety, not just on that one name, and each clerk has to throw the whole notebook away and copy it out again from scratch, at the same moment every other clerk in that store does the same thing.

## 3. The architecture, top to bottom

```
Clients (web/mobile product surfaces: News Feed, profiles, comments,
         checkins, likes, notifications, hundreds of call sites)
   |  call a typed graph API, not raw SQL:
   |  assoc_add(id1, atype, id2, time, data), assoc_get, assoc_range,
   |  assoc_count, obj_get/obj_add for the objects themselves
   v
Follower cache tier (many machines per region, per shard)
   |  serves the overwhelming majority of that 1B+ reads/sec straight
   |  from RAM; a miss is forwarded to this region's single leader tier
   |  read throughput scales by adding more follower machines
   |  analogy: many teller windows, all reading the same central ledger
   v
Leader cache tier (one per region, per shard's home)
   |  the only tier allowed to talk to persistent storage directly;
   |  applies writes to its cache write-through, so the very next read
   |  in this region sees the write immediately
   |  if this shard's master lives in a DIFFERENT region, the leader
   |  forwards the write onward to that region's leader instead of
   |  applying it locally
   |  analogy: the one teller in the branch actually allowed to touch
   |  the vault; every other window just reads what the vault says
   v
MySQL (sharded across hundreds of thousands of shards)
   |  exactly one authoritative master per shard, living in exactly
   |  one "master region"; shards are assigned to cache tiers by
   |  consistent hashing, so growth doesn't require a global remap
   v
Async cross-region replication (the shock absorber across oceans)
   |  every committed write on the master ships via async MySQL
   |  replication to a full slave copy in every other region, lag
   |  normally under one second
   |  a write issued in a slave region round-trips once to the master
   |  region's leader, then becomes visible everywhere else once
   |  replication catches up, no synchronous cross-ocean commit
```

Every region runs a complete copy of this stack, follower tier, leader tier, and a full MySQL replica set. What differs by region is only which region is the master for a given shard's writes; reads are always served locally, everywhere.

## 4. The transferable mechanisms

**a. Expose the query you actually need, not a generic get/set.** `assoc_range` and `assoc_count` let the cache maintain an ordered, paginated list and its count as first-class, independently-updatable things. A new edge is spliced into the cached list and the count is incremented in place, instead of forcing a full-blob invalidate-and-refetch. The reusable idea: model the shape of the query in the cache's API, not just the storage's, so a small write produces a small cache update.

**b. Split "absorb the reads" from "own the writes" into two tiers.** Followers exist purely to soak up read fan-out and never touch the database; a single leader tier per shard is the only thing allowed to mutate state and keep writes serialized. Read scale (add more followers) and write correctness (one leader per shard) stop competing for the same machines.

**c. Write-through cache buys read-your-writes cheaply, without buying strict consistency for everyone.** A write updates the leader's cache synchronously, so the writer's own next read sees the effect immediately. Every other reader, in every other region, can still see a slightly stale value until replication catches up. This is picking the cheapest consistency guarantee that is actually needed (the author sees their own action) instead of the most expensive one (every reader everywhere sees every write instantly).

**d. Single master per shard, async replication everywhere else, instead of global consensus on every write.** Rather than allow writes from anywhere (which needs conflict resolution) or pay a synchronous cross-region commit on every write, TAO picks one home region per shard and accepts a bounded replication lag, normally under a second, as the price for letting every other region read locally, at memory speed, with no network hop.

**e. Consistent hashing to place shards on physical machines.** Hundreds of thousands of logical shards map onto a much smaller, changeable set of follower and leader machines by consistent hashing, so adding capacity or rebalancing does not require moving every shard at once. Same primitive as Day 10, applied here to cache-tier placement instead of DB partitioning.

**f. Favor a fast, possibly-stale answer over no answer at all.** If a leader is briefly unreachable, followers keep serving whatever they last cached rather than failing the read. The measured 90-day failure rate of about 4.9 x 10^-6 queries is the receipt for that bet: near-total availability, paid for with an explicit, bounded amount of staleness.

## 5. The trade-offs

**Consistency vs. availability, split by who is asking.** TAO is explicit that it favors efficiency and availability over strict consistency. The writer gets read-your-writes in their own region via the write-through leader cache; every other reader, in every other region, can see a value that is stale by up to the replication lag, typically under a second. That is the right call for a like count or a comment list, a second of staleness is invisible to a person scrolling a feed, and it would be the wrong call for a payment ledger (contrast Stripe's idempotency keys from Day 6) or for data that must never appear to move backward in time (contrast Spanner's TrueTime commit-wait from Day 27).

**Cost vs. latency.** Full graph replicas, many petabytes of it, are kept live in every region purely so reads never have to leave the region. That is a continuous, real storage and cross-region bandwidth cost, paid every day, specifically so that the overwhelming majority of that one-billion-reads-a-second workload never crosses an ocean to get an answer.

**Simplicity vs. performance, specifically for lists.** A plain key-value cache is simpler to build than a system that understands "this is an ordered, paginated, incrementally-updatable edge list." The simple version is exactly what forces a full-list refetch on every single edge change, which is the specific bottleneck TAO exists to remove.

## 6. The systems-thinking lens

**The feedback loop here is a cache-invalidation stampede, and it is triggered by writes, not just by popularity.** Trace it under the naive look-aside design: a popular object's edge list changes once (one new comment, one new like) → the whole cached blob for that object is dropped, because a key-value cache cannot invalidate "just the new part" → every subsequent reader for that object is now a cache miss at once, because popular objects are read constantly → a burst of near-simultaneous, identical, expensive queries lands on the same MySQL rows at the same moment → the database slows under that spike → some of those requests time out → app-server retries fire, adding more identical load onto the exact same rows that were already struggling. Left alone this does not recover on its own, and it does not even need a traffic spike to start it, an ordinary single write to a popular object is the trigger, and it happens again on the very next write to that same object. It is the same shape as the cache stampede named in Day 35's systems-thinking lens and Day 16's hot-key problem, except here the recurring trigger is "any edit to a popular list," which means it would fire over and over, indefinitely, under the naive design.

**The senior fix is not a bigger cache or more memcache boxes, it is changing what gets invalidated.** By modeling the edge list as a structure the cache understands, `assoc_range` and `assoc_count` over an `(id1, atype)`-keyed range, a single new edge is applied as an incremental patch to the cached list and its count. There is no full-list eviction, so there is no burst of full-list re-reads for anyone to stampede on in the first place. The loop is broken at its first step, not absorbed downstream with more hardware.

---

## References and summaries

**Bronson, Nathan, et al. "TAO: Facebook's Distributed Data Store for the Social Graph." Proc. of the 2013 USENIX Annual Technical Conference (USENIX ATC '13).**
https://www.usenix.org/conference/atc13/technical-sessions/presentation/bronson (paper PDF also mirrored at https://www.usenix.org/system/files/conference/atc13/atc13-bronson.pdf and https://cs.uwaterloo.ca/~brecht/courses/854-Emerging-2014/readings/data-store/tao-facebook-distributed-datastore-atc-2013.pdf)
This is the primary source this lesson is built on. Direct full-text fetches of every mirror above, plus a plain Wikipedia URL used as a control test, all returned HTTP 403 in this session, indicating the fetch tool itself was blocked broadly rather than any one host blocking this request specifically; no full-text page-by-page read was possible this run. The facts and numbers used here were instead cross-corroborated across multiple independent secondary sources that quote or summarize the paper directly (listed below), and are consistent with each other on every figure used: the objects/associations data model (`id1, atype, id2, time, data`), the `assoc_add`/`assoc_get`/`assoc_range`/`assoc_count` API, the Alice/Bob/Cathy/David Golden Gate Bridge checkin example, the leader/follower cache-tier architecture with consistent-hashed shard placement, single-master-per-shard with async cross-region MySQL replication, write-through caching for read-your-writes, over 1 billion reads/sec and millions of writes/sec, 96.4% cache hit rate, hundreds of thousands of shards holding many petabytes of data, replication lag normally under one second, and a measured 90-day failure rate of about 4.9 x 10^-6 queries.

**Facebook Engineering. "TAO: The power of the graph."**
https://engineering.fb.com/2013/06/25/core-infra/tao-the-power-of-the-graph/
Facebook's own announcement of the system, confirming the headline "over a billion reads and millions of writes per second" figure and the framing of TAO as the replacement for ad hoc look-aside memcache use for graph data specifically.

**Hemant Gupta. "Insights from Paper: TAO: Facebook's Distributed Data Store for the Social Graph."**
https://hemantkgupta.medium.com/insights-from-paper-tao-facebooks-distributed-data-store-for-the-social-graph-48446205ba28
A close paper summary corroborating the cache hit rate, the leader/follower read-scaling relationship, the eventual-consistency-via-async-invalidation-messages mechanism, and the 90-day failure rate figure. Used as secondary corroboration, not a primary source.

**Massive Technical Interview Tips / Gaurav's Blog / kophy's notes / Julia Chen's blog (independent system-design writeups on TAO).**
https://massivetechinterview.blogspot.com/2018/10/facebook-tao.html, https://blog.gaurav.ai/2016/12/29/system-design-facebook-tao/, https://kophy.github.io/notes/tao/tao/, http://juliachencoding.blogspot.com/2020/08/system-design-facebook-tao-data-store.html
Used to cross-check the multi-region leader/slave-region write-forwarding description and the "read-what-you-wrote for the writer, eventual consistency otherwise" consistency statement, since all four independently describe the same mechanism in matching terms.

**Nishtala, Rajesh, et al. "Scaling Memcache at Facebook." Proc. of the 10th USENIX Symposium on Networked Systems Design and Implementation (NSDI '13).**
Referenced here, without a full-text read this session, as the companion paper describing the look-aside memcache-in-front-of-MySQL architecture that predated TAO, the naive design this lesson's section 2 is built on. Named for completeness; treat any specific figures from it as unverified until read directly in a future lesson.

---

## Map to Rare.lab's stack

Rare.lab's node-based shader editor is, structurally, exactly the kind of graph TAO was built for: nodes are objects, and "this node feeds into that node," "this shader references that texture," "this user starred that project" are all associations, directed edges with a type and a timestamp. Today, at current scale, answering "what does node X feed into" or "who has starred this project" by walking rows in Supabase Postgres under RLS is the right call, the same way TAO's own naive predecessor was fine before the graph got large and hot.

The ceiling arrives in the same shape as section 2. The moment Rare.lab needs to answer graph questions that are read far more often than written, "which published shaders reference this texture asset, across the whole public library," "what is this popular node's full downstream dependency list, so the compiler knows what to recompile," a naive cache-the-whole-list-and-invalidate-it-all-on-one-edit approach will behave exactly like the pre-TAO design: one asset gets referenced by one new shader, and if the cached "who references texture Y" list is one blob, that whole blob gets dropped and refetched from Postgres, right as everyone else who was already reading that popular asset's reference list piles onto the same query at once.

The concrete, actionable move: when that list-shaped, read-heavy graph query shows up, do not reach for a generic single-key cache in front of Postgres. Model it the TAO way, a keyed, incrementally-updatable association list, `(source_id, edge_type, target_id, updated_at)`, cached with its own count, where adding one edge patches the cached list and bumps the count in place instead of invalidating the whole cached collection. Pair that with the version-counter/zookie idea from Day 35's Rare.lab mapping (a project's version bumps only when its graph-relevant edges change) so a single new reference or star does not manufacture a stampede of full-list re-reads against the shared Postgres instance the moment any one popular asset changes. This is the point to build it, before the node graph or the public shader library gets large enough for one popular node's edge list to become a hot, frequently-rewritten cache key.
