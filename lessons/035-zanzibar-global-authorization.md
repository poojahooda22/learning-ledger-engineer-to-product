# Day 35: How does Google answer "can Alice see this file" in under 10 milliseconds, 10 million times a second, without ever giving a wrong answer?

*2026-07-18*

---

## 1. The company and the number that breaks a naive design

**Google, Zanzibar, in production since roughly 2014, published as a paper at USENIX ATC in 2019.** Zanzibar is the single authorization system behind Calendar, Cloud, Drive, Maps, Photos, and YouTube, permissions services managing billions of objects for more than a billion users. The breaking number is not one figure, it is three that all have to hold at once: **more than 10 million authorization checks per second, over more than 2 trillion stored permission edges occupying close to 100 terabytes, with 95th-percentile latency held under 10 milliseconds for three straight years of production use.**

Layer one more requirement on top and the naive design does not just get slow, it gets *wrong*. Every one of those 10 million checks a second must respect the exact order in which permissions were last edited. If Alice removes Bob from a folder's ACL and then asks Charlie to drop new documents into that folder, Bob must never see the new documents, even if his check request lands on a database replica that has not yet heard about the removal. The paper calls this the "new enemy" problem. So the real breaking number is: **10M checks/sec, 2T+ edges, sub-10ms p95, replicated to more than 30 locations around the world, and never allowed to return a stale "yes."**

## 2. Why the naive design dies

The naive version: one relational table per app, something like `shares(document_id, user_id, role)`, with a live SQL join whenever a group is involved (`shares(document_id, group_id, role)` joined against `group_members(group_id, user_id)`). It collapses in three concrete ways.

**a. Group nesting turns a lookup into an unbounded graph walk.** A real permission is rarely "user 10 owns doc:readme." It is closer to "anyone in group:eng can view doc:readme, and group:eng contains group:eng-backend, which contains group:eng-backend-oncall, which contains a Google Group synced from an LDAP export." A single naive join only covers one level. Handling arbitrary nesting means a recursive query, and recursive queries have no upper bound on fan-out. At Google's scale, some namespaces hold tuple counts "from tens to a trillion, with the median near 15,000" per namespace, and groups can and do nest deeply. A recursive join over that shape, run live, on every one of 10 million checks a second, saturates any single database's CPU in minutes.

**b. One search result page needs tens to hundreds of these checks, not one.** A client like Drive filters a page of search results down to what the user is actually allowed to see, which means firing off a full authorization check per candidate object before it can even show the list. That multiplies the naive join's cost by the length of every results page, for every search, all day.

**c. A single shared table becomes the hottest row in the company the moment one document or group goes viral.** If a widely-shared folder or a large corporate group is referenced indirectly by thousands of other objects' ACLs (paper's own example: a search that touches many documents that all inherit from the same shared group), every one of those checks converges on the same underlying rows at the same instant. A plain relational table has no defense against that; it is one lock, one buffer pool, one point of contention, for a request rate that was supposed to be spread across the internet.

The analogy: imagine a bouncer at a nightclub who, for every single person in a 10,000-person line, has to phone HR, ask "is this person still employed, and are they still on the guest-list-approval committee, and is that committee still authorized by the building owner," and wait for the full chain to resolve, one call at a time, before waving anyone through. That bouncer is the whole club's throughput ceiling.

## 3. The architecture, top to bottom

```
Clients (Drive, Calendar, Photos, YouTube, Cloud, hundreds of services)
   |  send Check("does user U have relation R on object O?"),
   |  Read, Expand, or Write requests over RPC
   v
aclserver cluster (the stateless "app tier")
   |  requests fan out to whichever server owns the shard for this
   |  object; each server can recursively call OTHER aclservers to
   |  resolve indirect ACLs, forming a fan-out tree per check
   |  routing uses consistent hashing on a forwarding key (usually
   |  the object ID), so all checks/reads for one object tend to
   |  land on the same server, which is what makes caching work
   |  analogy: a call center where every agent can transfer part of
   |  a question to a specialist, and transfers for the same
   |  customer always land on the same specialist
   v
Distributed cache + lock table (in front of every aclserver)
   |  caches leaf tuple reads AND intermediate check results, keyed
   |  by a coarsened snapshot timestamp so unrelated requests can
   |  share a cache entry; the lock table gives only ONE in-flight
   |  request per cache key permission to actually hit storage,
   |  every other concurrent request for the same key just waits
   |  analogy: one barista makes the drink, everyone else who
   |  ordered the exact same thing at the exact same moment waits
   |  for that one cup instead of each brewing their own pot
   v
Leopard indexing system (a specialized shortcut for deep/wide groups)
   |  precomputes group-to-group and member-to-group reachability
   |  as ordered integer sets on a skip list, so "is U in group G"
   |  becomes a set intersection, O(min(|A|,|B|)), not pointer-chasing
   |  a live incremental layer folds in updates since the last
   |  offline snapshot, so it stays fresh without a full rebuild
   |  analogy: instead of tracing an org chart every time someone
   |  asks "does this person report up to that VP," you keep a
   |  pre-sorted roster per VP and just check for a name in it
   v
Spanner (the DB primary + globally replicated read layer)
   |  ACLs are sharded per namespace, further sharded by object ID
   |  (client-configured), each shard replicated to dozens of
   |  locations worldwide, 5 voting Paxos replicas kept within 25ms
   |  of each other so writes commit fast; TrueTime timestamps every
   |  write so causal order survives across the whole planet
   |  analogy: a single global ledger, but with local branch offices
   |  everywhere that can answer "as of a moment ago" without
   |  phoning headquarters every time
   v
Changelog + watchserver (the async/streaming pipeline)
   |  every write also appends to a changelog; watchservers tail it
   |  and push near-real-time tuple-change events to clients that
   |  maintain their own secondary indexes (e.g. a search index that
   |  needs to know when an ACL changed)
   v
Periodic offline pipeline (garbage collection + Leopard snapshot builder)
   |  runs across all of Spanner's ACL data, drops tuple versions
   |  past the GC window, and rebuilds Leopard's base index from a
   |  fresh dump on a schedule, off the hot path entirely
```

## 4. The transferable mechanisms

**a. The consistency token ("zookie"): a portable proof of "at least this fresh."** Instead of always reading the absolute latest data (slow, needs cross-region round trips) or always reading a fixed stale snapshot (fast, but can violate causality), Zanzibar hands clients an opaque token that encodes a global timestamp. A client that just changed a document's contents gets a zookie for that change and includes it in the next permission check, which forces the check to be evaluated no earlier than that timestamp. Everything else defaults to "whatever snapshot is already safely replicated nearby," which is what lets 10 million checks a second mostly avoid a network round trip at all. This is the general pattern of a **bounded-staleness read token**: cheap by default, exact when the caller proves it needs to be.

**b. Request hedging for tail latency.** For calls to the storage layer or the Leopard index, Zanzibar sends the same request to two replicas and takes whichever answers first, cancelling the loser. It only fires the second request after a dynamically-estimated delay threshold, so this costs almost nothing in the common case but caps the damage from one slow straggler. This is a reusable trick anywhere p99 matters more than average-case efficiency: pay for redundant work only on the rare request that is already running slow.

**c. Single-flight de-duplication via a lock table, to stop cache stampedes.** When a cache entry is empty and thousands of identical concurrent requests arrive for it (a suddenly-popular document, a huge group), only one request is allowed to actually go compute the answer; the rest block on a lock table entry and get the same result once it lands. Without this, a hot key turns into a self-inflicted denial-of-service the moment it goes viral.

**d. Precompute the expensive graph problem, serve the cheap set-intersection.** Group membership is a reachability problem on a graph that can be arbitrarily deep. Zanzibar refuses to solve that graph problem live for every check. Leopard periodically flattens the graph into two lookup tables (group-to-group, member-to-group) as sorted integer sets, so answering "is U in G" becomes a bounded-cost set intersection. The live incremental layer only has to fold in *changes since the last snapshot*, not rebuild the whole structure. This is the same offline-think, online-lookup split this ledger keeps finding everywhere, applied here to a graph reachability query instead of a ranking model.

**e. Performance isolation as a first-class serving concern, not an afterthought.** Every client gets a hard CPU-seconds-per-second budget; every server caps outstanding RPCs and per-object, per-client concurrent reads against storage; different clients get different lock-table keys so one client's storm cannot throttle another client's unrelated requests on the same backend. A shared multi-tenant service without this degrades exactly like the shared-fleet failure from Day 34: one noisy client's load leaks into everyone else's latency.

**f. Route and shard by the same key you cache on.** Forwarding keys for internal delegated RPCs are usually derived from the object ID, so checks and reads for the same object consistently land on the same server. That single choice is what makes both the distributed cache and the lock table actually effective, because cache hit rate depends entirely on requests for the same key colliding on the same machine instead of scattering randomly across the cluster.

## 5. The trade-offs

**Consistency vs. availability, split cleanly by data type.** Writes to a relation tuple require a synchronous Spanner commit with a Paxos quorum, expensive on purpose, and correspondingly rare: over a sampled 7-day window, Writes peaked at only about 25,000 QPS versus roughly 4.2 million QPS for Check and 8.2 million QPS for Read. Reads, by contrast, default to "whatever locally-replicated snapshot is safely fresh enough," explicitly trading a small amount of staleness for the ability to answer almost every request without leaving the region. The paper's own numbers show why this split matters: Check-Recent requests (checks with a zookie younger than 10 seconds, needing fresher data) run roughly 20 to 25 times slower at the tail than Check-Safe requests, 60.0ms versus 9.46ms at p95, purely because freshness sometimes forces a cross-region round trip.

**Cost vs. latency.** ACL data is fully replicated to more than 30 locations worldwide, purely to buy read locality; that is a large storage and write-fan-out cost paid continuously so that the vast majority of the 10 million checks a second can be served from something physically close. Hedging pays a similar tax: duplicate work sent to a second replica, discarded the moment the first answer arrives, all to shave the tail off latency that would otherwise be dominated by the occasional slow straggler.

**Flexibility vs. simplicity.** The userset-rewrite language (union, intersection, exclusion, `computed_userset`, `tuple_to_userset`) lets one relation's meaning depend on another relation on a completely different object, which is what makes "viewers of a document include viewers of its parent folder" a config change instead of a data migration. That expressive power is also why check evaluation is a boolean-expression tree walk instead of one table lookup, and why a whole specialized subsystem (Leopard) had to be built just to keep that tree walk fast when relations nest deeply.

## 6. The systems-thinking lens

**The feedback loop that actually threatens a system like this is a cache stampede on a suddenly-hot object, and it is a close cousin of the hot-key/celebrity problem this ledger has already named.** Trace it: an object is cold, ordinary traffic, cache handles it fine → the object (or, just as often, a group that many other objects indirectly reference) suddenly gets referenced by a burst of unrelated requests, a document goes viral, a search touches many results that all inherit from the same shared group → the cache entry for that key is empty or has just expired → many concurrent requests all miss at once and all try to read the same underlying Spanner rows simultaneously → the storage server hosting those rows sees a spike of identical, redundant reads → latency on that hot shard rises → clients that time out or get slow responses retry → the retries pile more identical load onto the exact same hot rows that were already struggling. Left alone, this does not recover on its own; it is the same self-sustaining shape as a metastable failure, just triggered by popularity instead of a capacity change.

**The senior fix does not add more cache servers or a bigger database.** It breaks the loop at the point where many concurrent, identical requests are allowed to independently hit the same resource: the lock table collapses N concurrent misses for the same key into exactly one storage read, with everyone else waiting on that single answer instead of generating their own. For objects detected to be especially hot, Zanzibar goes further and preemptively caches the object's *entire* set of relation tuples in one shot rather than serving them one lookup at a time, and Slicer auto-shards a single hot key across multiple physical servers so the "one row, one machine" assumption stops being the bottleneck. None of these add raw capacity; they all remove the step in the loop where popularity turns into redundant, self-amplifying work.

---

## References and summaries

**Pang, Ruoming et al. "Zanzibar: Google's Consistent, Global Authorization System." Proc. of the 2019 USENIX Annual Technical Conference (USENIX ATC '19).**
https://www.usenix.org/system/files/atc19-pang.pdf (direct fetch returned HTTP 403; full text read successfully via Google's own hosted copy at https://storage.googleapis.com/gweb-research2023-media/pubtools/5068.pdf, rendered and read page by page)
This is the primary source for essentially every number and mechanism in this lesson. Confirmed directly from the paper's text: relation tuple data model and the `object#relation@user` notation (Table 1's `doc:readme#owner@10`, `group:eng#member@11`, `doc:readme#viewer@group:eng#member` examples used in section 3); the "new enemy" problem and the zookie/external-consistency protocol built on Spanner's TrueTime (section 2.2); namespace configs and userset rewrite rules including `_this`, `computed_userset`, and `tuple_to_userset` (section 2.3.1); the architecture of aclservers, watchservers, Spanner storage, and the Leopard indexing system (Figure 2, section 3); Leopard's GROUP2GROUP/MEMBER2GROUP set-intersection design (section 3.2.4); hot-spot handling via distributed cache, lock table, cache-prefetching for hot objects, and Slicer for hot forwarding keys (section 3.2.5); performance isolation via per-client CPU-seconds quotas (section 3.2.6); request hedging (section 3.2.7); and every headline number, more than 2 trillion relation tuples occupying close to 100TB, more than 1,500 namespaces, more than 10 million QPS, sample-week breakdown of Check at 4.2M QPS / Read at 8.2M / Expand at 760K / Write at 25K, more than 10,000 servers in several dozen clusters across more than 30 locations, 5 Spanner voting replicas within 25ms of each other, latency percentiles (Check Safe: 3.0/9.46/15.0ms at 50/95/99%, Check Recent: 2.86/60.0/76.3ms), availability above 99.999% over 3 years, Leopard serving at 1.56M QPS median and 2.22M QPS at p99 with sub-150-microsecond median response, and 22 million internal delegated RPCs/sec at peak (sections 1, 4, 4.1-4.5).

**AuthZed: "Zanzibar-Inspired Permissions Systems: A Landscape Review" and related company pages (Carta, Netflix).**
https://authzed.com/blog/zanzibar-implementations, https://medium.com/building-carta/authz-cartas-highly-scalable-permissions-system-782a7f2c840f, https://authzed.com/customers/netflix
Direct fetches of the Carta and Netflix pages both returned HTTP 403; the following is drawn from search-indexed excerpts, not full-text reads, and is clearly labeled as such. Carta's Identity and Access Management team rebuilt its permissions system on Zanzibar-style principles starting in mid-2019, while decomposing a monolith and needing a consistent way to authorize requests across newly-split services, a real-world instance of the exact "hundreds of client services, one shared authorization system" motivation the original paper opens with. Netflix separately adopted SpiceDB (an open-source, Zanzibar-inspired engine) and sponsored work to add attribute-based access control on top of it, to handle its own "complex identity types," household accounts and multiple profiles sharing one subscription being the specific shape of ReBAC-plus-ABAC problem that a pure relation-tuple model does not cover on its own. Both are offered as clearly-labeled, real-world corroboration that the Zanzibar model generalizes past Google, not as primary sources for the numbers in this lesson.

**AuthZed / SpiceDB and OpenFGA project pages (open-source Zanzibar implementations).**
https://github.com/authzed/spicedb, https://github.com/openfga/openfga
Used only to confirm, via search-indexed summaries, that production Zanzibar-inspired deployments exist outside Google at meaningful scale (SpiceDB reporting 5ms p95 at millions of queries per second and billions of relationships in independent deployments), reinforcing that the mechanisms in section 4 are portable, not Google-specific engineering folklore.

---

## Map to Rare.lab's stack

Rare.lab today authorizes access the naive way this lesson opens with, and correctly so at current scale: a shared Supabase Postgres instance with row-level security (RLS) policies deciding who can see or edit which shader project, evaluated live, per row, on every request. That is fine while sharing is still shaped like "owner plus a short list of named collaborators," because that shape needs at most one join, not a recursive graph walk.

The ceiling is exactly section 2's ceiling, and it arrives the moment sharing stops being a flat list. The day Rare.lab ships team folders, "anyone in this org can view any project under this folder," or a public embed link that should resolve to "viewer" without a logged-in user at all, RLS policies start needing recursive joins across projects, folders, and team-membership tables to answer one permission check, evaluated fresh on every request because Postgres RLS has no separate cache layer of its own. That is section 2a's failure mode arriving on a smaller scale: not 10 million checks a second, but every embedded scene render across every website that has embedded a Rare.lab creation, each one needing a live "is this token still allowed to read this scene" check against the same shared Postgres instance that also serves the editor's interactive traffic. A viral embed is this lesson's hot-object stampede, just with fewer zeroes.

The concrete, actionable move, worth making before the recursive-join version of RLS ever gets written: borrow the zookie pattern in miniature. Give each project a version counter that increments only when its permission-relevant rows change (a collaborator added or removed, a folder re-parented, a share link revoked), and cache the answer to "can this token/user view this project" keyed by `(project_id, requester_id_or_token, version)` with a short TTL, say a few seconds, in front of Postgres. A permission edit bumps the version, which invalidates exactly that project's cached answers and nothing else, no cross-project cache flush, no coordination. Read traffic, which for an embeddable runtime is nearly all of it, then hits the cache instead of a live RLS-evaluated recursive query on almost every request, the same 10M-cheap-reads-vs-25K-expensive-writes split Zanzibar makes, scaled down to Rare.lab's actual numbers. This is the point to reach for it: as soon as sharing grows past flat owner-plus-collaborators, not after the first slow embed page is diagnosed in production.
