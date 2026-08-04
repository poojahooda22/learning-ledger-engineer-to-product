# Day 47 — How does Cloudflare check a firewall rule 3 billion times a second, at every edge node, with no database call in the request path?

**Date:** 2026-08-04
**Difficulty:** Expert
**Topic:** Quicksilver, Cloudflare's internally built configuration-distribution key-value store. How a DNS record, a WAF rule, or a Workers script edit made in a dashboard reaches every one of Cloudflare's 300-plus edge data centers in seconds, and why the store the reads hit at request time is not really a database at all, it is a local, memory-mapped file every edge server already has a copy of.
**Stack relevance:** Cloudflare Workers/R2 runtime for Rare.lab's embeddable player, plus the content-addressed R2 manifest from Day 23. The day thousands of embedded Rare.lab runtimes need to learn about a shader hot-fix or a kill-switch without each one hammering Supabase.

---

## 1. The company and the breaking number

**Cloudflare, running its edge configuration store on Kyoto Tycoon, an open-source key-value server, before 2020.** Every single thing Cloudflare's edge needs to do its job, which DNS records exist, which firewall rules are active, which Workers script maps to which route, which customer is on which plan, lived in this one store, replicated out to every data center. It is the purest possible "config database" workload: read constantly, on every single HTTP request that touches Cloudflare's network, written to only occasionally, whenever a customer changes a setting.

The breaking number is not a request-per-second figure, it is an operational one, and it is more damning than a raw QPS ceiling: **keeping Kyoto Tycoon alive was costing Cloudflare's site reliability team roughly 48 hours of engineer time a week.** That is more than one full-time engineer, every week, just to keep one piece of infrastructure from falling over. The specific mechanical cause was a single exclusive write lock shared by reads and writes inside one KT instance: the lock was sensitive to I/O, so a burst of writes (a customer pushing a bulk DNS import, or Cloudflare's own internal systems propagating a change) stalled every read behind it. Read latency, which should be the cheap, constant-time operation in a system that is read 3 billion times a second at its later scale, would blow past 1 millisecond and keep climbing for as long as the write burst lasted. A system whose entire job is answering "is this rule active" fast enough to sit in the hot path of every request was, on a bad day, slower than the origin server it was supposed to be protecting.

That is the real "one-server" failure here. It is not that KT could not hold enough data. It is that combining the read path and the write path inside one lock meant the read path's latency was hostage to whatever the write path happened to be doing at that exact moment, and at Cloudflare's actual production write volume, that happened often enough to need a full-time babysitter.

---

## 2. Why the naive design dies

The naive design, the one Cloudflare was actually running, looks completely reasonable on a whiteboard: one key-value store, master-slave replication, every data center gets a replica. Kyoto Tycoon even had the right consistency model already, timestamp-based replication for eventual consistency, the same target Quicksilver would later ship with. The consistency model was never the problem. The implementation was.

**a. One shared lock across two workloads with opposite latency requirements.** Writes are bursty and can tolerate being slow, nobody notices if a bulk DNS import takes an extra second. Reads are constant, extremely high volume, and sit directly in the critical path of every HTTP request Cloudflare's network serves. Putting both behind the same exclusive lock means the read path inherits the write path's worst-case latency, every time. This is the general failure mode of colocating a rare, bursty write workload with a constant, latency-sensitive read workload on the same resource: the read SLA is only ever as good as the write pattern happens to allow, and the write pattern is not something the read path controls.

**b. No structural separation between "the place that decides what's true" and "the place that answers questions about it."** KT ran the same kind of instance everywhere, root and edge were not architecturally distinct roles, they were the same software doing double duty. That meant there was nowhere to put the fix except tuning the one shared instance harder, which is exactly the kind of dead end a single-server design always reaches: you can tune a lock, you cannot architect your way out of a lock you have already committed to sharing.

**c. It was unreliable in a way that consumed human time, not just machine time.** Cloudflare's own account of this period describes replication and database functionality breaking "quite often." A system that needs 48 hours of SRE attention a week is not scaling with the business, it is a fixed, growing tax on every future feature the company ships, because every new product (a new firewall feature, a new DNS record type, a new Workers capability) adds more writes and more reads to the same fragile store.

---

## 3. The architecture, top to bottom

The system Cloudflare built to replace it, Quicksilver, launched in 2020 covering roughly 200 cities in 90 countries, and its core move is to stop treating "write path" and "read path" as the same problem:

```
[Dashboard / API: customer edits a DNS record, a WAF rule, a Workers route]
   |
   v
[Control-plane API layer: validates the change, compiles it]
   analogy: an editor who reads your submission before it goes to print,
   not the printing press itself.
   (Cloudflare's own "Zone Builder" is one example: it compiles a zone's
   full DNS record set before Quicksilver ever sees it. This step alone
   used to take up to 35 seconds to build ~1 million DNS records, before
   Cloudflare's own optimization work cut that by over 4,000x. This is
   upstream of Quicksilver, worth knowing exists, not the same latency
   budget as replication itself.)
   |
   v
[Quicksilver ROOT NODES: a small number of servers, terabytes of storage,
 living in Cloudflare's core/hyperscale data centers]
   single job: be the one place writes land, elected write-aggregating
   tier, not spread across every data center.
   analogy: a newspaper's one printing plant. Every edition starts here,
   nowhere else.
   |
   v  (push, not poll: the tier below does not ask, it is told)
[INTERMEDIATE NODES: regional hubs, replicate from root nodes or from
 other intermediates]
   single job: fan the update out one more level without every one of
   Cloudflare's edge servers connecting directly back to the root tier.
   analogy: regional newspaper distribution depots. The plant does not
   truck every single edition to every single newsstand itself.
   |
   v
[LEAF NODES: one set per edge data center, serve real end-user HTTP
 traffic directly]
   single job: hold a local, complete, queryable copy close to where
   requests actually land.
   analogy: the newsstand. The customer buys the paper here, they never
   call the printing plant.
   |
   v
[The actual request: "is rule X active for this zone?"]
   answered by a LOCAL read against a memory-mapped file on the same
   machine handling the HTTP request. Zero network hop. No database
   round trip. This is the whole point: by the time a real request
   arrives, the "database call" has already happened, days or seconds
   earlier, as replication, not as part of this request.
```

Two things happened to this picture after 2020 that are worth knowing, because they show the same lesson learned twice.

First, the original storage engine underneath every node was **LMDB**, a memory-mapped, copy-on-write key-value library. Copy-on-write makes reads extremely cheap (you are reading straight out of mapped memory) but makes writes expensive in a specific way: updating even a tiny key-value pair can force copying the entire memory page it lives on. Cloudflare's own measurement of this: **writing 1 megabyte of logical data to Quicksilver could flush 30 megabytes to disk**, a 30x write amplification. As the dataset kept growing (it was compounding at roughly 50% a year), that amplification, multiplied across a full replica sitting on every single edge server, started eating disk and I/O budget everywhere at once. It is the same shape of problem as the original KT lock, a design decision that was fine at one write volume becoming a tax that scales worse than the business, just one layer further down the stack.

Second, the fix mirrored the earlier one: **stop putting a full copy on every server.** Quicksilver v2 moved to a tiered structure inside each data center: a small **local per-server cache**, a **data-center-wide shared cache shard** (populated from that data center's own aggregate cache misses, so a value fetched once for one server becomes available to its neighbors), and the **full dataset kept only on a small number of dedicated storage nodes**, reached through a handful of servers per data center running in **relay mode** so the rest of the fleet never fans out directly against the storage tier. The storage engine itself moved from LMDB to **RocksDB**, an LSM-tree design (the same family covered in Day 21): in Cloudflare's own tests, RocksDB stored the same data in about 40% of LMDB's disk space and cut write amplification sharply, at the cost of roughly double the CPU (about 150% versus LMDB's 70% in their comparison). After the move, average read latency settled around 500 microseconds, down from a system that used to blow past 1 millisecond under write load, the exact symptom the original KT replacement was built to fix, showing up again inside Quicksilver's own evolution and getting the same structural answer: separate the expensive, growing thing from the cheap, constant thing, do not just tune the shared resource harder.

---

## 4. The transferable mechanisms

**a. Push replication down a tree, do not let every leaf poll or connect back to one root.** Root pushes to intermediates, intermediates push to leaves. Fan-out at any single level stays bounded and small (dozens of root servers, a manageable number of intermediates), instead of every one of hundreds of thousands of edge machines opening a connection straight to the handful of root servers. Total propagation time grows with tree depth, not with leaf count, which is why a change reaches every data center in seconds even as the number of data centers grows into the hundreds.

**b. Physically separate the write path from the read path, not just logically.** Root nodes exist to absorb writes and are a different tier, different hardware role, different failure domain, from leaf nodes that exist to answer reads. This is the direct structural fix for the KT lock problem: there is no shared lock to contend on, because there is no shared instance doing both jobs anymore.

**c. Make the hot-path read local, not networked.** The entire reason Quicksilver can be read 3 billion times a second without falling over is that a "read" at request time is a memory-mapped local file access on the same machine already handling the HTTP request, not an RPC, not a database connection, not even a loopback network call. Replication is the expensive step, and it has already happened, off the request's critical path, before the request arrives.

**d. Tier your caching by blast radius, cheapest and most local first.** Per-server cache, then data-center-shared cache, then a small dedicated full-replica tier, mirrors ordinary CDN caching (edge cache, then regional cache, then origin) applied to a key-value store instead of static files. Almost all reads get served by the cheapest tier; the expensive, complete copy lives in as few places as the system can get away with.

**e. Choose your storage engine for your actual write shape, not by default, and be willing to re-choose it as that shape changes.** LMDB's copy-on-write is close to free for read-mostly workloads and brutal for anything with real write volume. RocksDB's LSM structure accepts more CPU to buy far lower write amplification. Neither is universally right; Quicksilver picked LMDB first because reads dominated, then re-picked RocksDB once the dataset's growth rate made write amplification the binding constraint.

**f. Treat consistency as a decision made per data type, not once for the whole company.** Quicksilver is deliberately eventually consistent: no global writes, one elected write-aggregating tier, values flow outward and the target is "within seconds," not "immediately everywhere." Cloudflare did not try to make this stronger, because config reads vastly outnumber config writes and a few seconds of staleness on a firewall rule is a cost worth paying for availability and speed everywhere else. Where that trade genuinely does not work, Cloudflare built something else entirely: Meerkat, a newer, separate system using QuePaxa (a 2023 leaderless consensus algorithm) for the specific control-plane state that must be linearizable, like electing a single leader for a replicated database or placing an AI model instance uniquely. Same company, two different consistency guarantees, chosen deliberately per workload.

---

## 5. The trade-offs

**Consistency versus availability, decided per data type, not as a blanket policy.** Config data (DNS records, firewall rules, Workers routes) is eventually consistent by design. A dashboard edit is visible everywhere within seconds, not instantly, and every edge node keeps answering requests with whatever it currently has rather than blocking or erroring while it waits to hear about a pending change. That is the correct trade for this data: almost all reads, very few writes, and staleness measured in single-digit seconds is invisible to a normal user. It would be the wrong trade for something like leader election, where two nodes both believing they are the leader for even a few seconds is a correctness bug, not a UX nit, which is exactly why Cloudflare built Meerkat as a separate, strongly consistent system instead of trying to stretch Quicksilver's model to cover it.

**Storage cost versus read latency.** A full copy of the dataset on every server (Quicksilver v1's approach) gives the fastest possible read, zero network hop, no cache-miss path, at the cost of storing the entire dataset (1.6 terabytes and growing about 50% a year) on every single machine in the fleet. Quicksilver v2's tiered caching accepts a small, bounded chance of a local miss (one extra hop to a data-center-local relay node) in exchange for keeping the full replica on a small, dedicated set of storage nodes instead of the whole fleet. The hot path stays sub-millisecond either way; the difference is how much redundant storage the company pays for to get there.

**CPU versus disk versus write amplification, a genuine three-way trade, not a free upgrade.** RocksDB beat LMDB on disk footprint (about 40% of the space) and write amplification, but cost more CPU (about 150% versus 70% in Cloudflare's own comparison). That is not "RocksDB is just better," it is a trade Cloudflare made because their bottleneck had shifted from CPU headroom (which they had) to disk growth and write amplification (which they didn't). A team with the opposite constraint could reasonably make the opposite choice.

---

## 6. The systems-thinking lens

**The feedback loop: a shared lock turns a rare event (a write burst) into a widespread one (every read stalls behind it).** Trace it through the original Kyoto Tycoon failure: a burst of writes arrives (a bulk config change, or Cloudflare's own systems propagating updates) → the exclusive lock, shared by reads and writes on one instance, is I/O sensitive, so it holds longer under write pressure → every read waiting on that lock queues behind it, and read latency, which should be flat and cheap, spikes past 1 millisecond → because reads sit in the hot path of every HTTP request Cloudflare serves, that latency spike is not contained to one subsystem, it is now visible on the edge of the network → nothing about the system's design limits how bad this gets, more writes simply means more reads queue longer, there is no backpressure mechanism separating the two, just one lock both sides share. This is the same class of failure Day 13's backpressure lesson names directly: a resource shared between a bursty producer and a latency-sensitive consumer, with no structural wall between them, turns the producer's bad day into the consumer's bad day, automatically, every time.

**The senior fix is architectural separation, not a bigger lock or a faster disk.** Cloudflare did not solve this by tuning Kyoto Tycoon's lock, adding more RAM, or throwing faster SSDs at the existing single-instance design, all of which would have bought some headroom and left the coupling intact. The actual fix removes the coupling: root nodes absorb writes, an entirely separate tree of intermediate and leaf nodes serves reads, and the two share no lock because they are not the same process anymore. The quieter echo of this same lesson shows up again inside Quicksilver itself years later: LMDB's copy-on-write write amplification (1MB in, 30MB to disk) was invisible at 2020's dataset size and became a real constraint once the dataset compounded at 50% a year, and the fix was, again, structural, a new storage engine plus a caching tier, not a tuning knob on the existing one. Both times, the instinct to "add capacity" would have worked for a while and then failed again at the next order of magnitude. Breaking the coupling was the fix that actually held.

---

## Map to Rare.lab's stack

**Where the same shape of problem is currently invisible, because Rare.lab hasn't hit the scale where it shows up yet.** Rare.lab's embeddable runtime reads its scene/shader manifest from R2, and presumably checks back with Supabase or a control API for anything dynamic, a kill-switch flag, a hot-fixed shader, a manifest version bump. At a few dozen embeds, that is a trivial read load, indistinguishable from noise. It is the exact same comfort Kyoto Tycoon had at Cloudflare's early scale: the design is not wrong, it just has not been asked the question that breaks it yet.

**The question that breaks it: thousands of embedded runtimes, on thousands of third-party sites, each independently polling or fetching to find out "is my manifest still the current one, is there a kill-switch."** That is Quicksilver's original problem with the roles renamed: Supabase (or a control API in front of it) is the root, and every embedded runtime in the wild is a leaf that, in the naive design, talks directly back to the root on every check. At 100 embeds this is invisible. At 100,000 embeds, each one polling even once a minute, that is a sustained, origin-hammering load on infrastructure that was sized for editor traffic, not for every deployed instance of the runtime in the world.

**The concrete move, worth making before that day arrives, not after:** stop having embedded runtimes read dynamic config (kill-switches, manifest-version pointers, feature flags for the runtime itself) directly from Supabase. Push it instead, the same push-not-poll, tree-shaped move Quicksilver makes, onto infrastructure Rare.lab already has access to on Cloudflare's edge: Workers KV (Cloudflare's own customer-facing product built on Quicksilver-adjacent principles) or a small set of Durable Objects keyed by manifest ID, written to once when a scene publishes or a kill-switch flips, read by every embedded runtime as a local edge read, not a database round trip to Supabase. This costs one extra write on publish, in exchange for the read side of that traffic, which is the side that actually scales with embed count, never touching the origin database at all. It is the same trade Cloudflare made when it stopped answering "is this rule active" with a database call and started answering it with a local file read that had already been kept up to date by replication, done ahead of time, off the request's critical path.

---

## References and summaries

**Cloudflare Blog: "Introducing Quicksilver: Configuration Distribution at Internet Scale"** (March 30, 2020)
https://blog.cloudflare.com/introducing-quicksilver-configuration-distribution-at-internet-scale/
Primary source for Quicksilver's original architecture: replacing Kyoto Tycoon, the root/intermediate/leaf replication tree, LMDB as the storage engine, and the 2020-era footprint of roughly 200 cities in 90 countries with changes propagating within seconds. Direct confirmation could not be re-fetched in this session (the URL returned HTTP 403 to automated fetch tools); facts here are corroborated across multiple independent search-engine summaries of the post's content plus corroborating secondary sources below, not a single unverified source.

**Cloudflare Blog: "Moving Quicksilver into production"**
https://blog.cloudflare.com/moving-quicksilver-into-production/
Source for the rollout story: the full migration off Kyoto Tycoon took more than a year, used LMDB's ability to copy a full environment over a socket to bootstrap new nodes, and was tested first on Cloudflare's own "dog-fooding" data center before wider release.

**Cloudflare Blog: "Quicksilver v2: evolution of a globally distributed key-value store" (Part 1 and Part 2 of 2)** (July 2025)
https://blog.cloudflare.com/quicksilver-v2-evolution-of-a-globally-distributed-key-value-store-part-1/
https://blog.cloudflare.com/quicksilver-v2-evolution-of-a-globally-distributed-key-value-store-part-2-of-2/
Primary source for this lesson's v2 numbers: the dataset grew to over 5 billion key-value pairs at 1.6 TB combined, growing roughly 50% a year, serving over 3 billion key reads per second worldwide; the LMDB copy-on-write write-amplification problem (1MB of logical writes flushing 30MB to disk); the move to RocksDB (about 40% of LMDB's disk footprint, roughly 150% CPU versus LMDB's 70% in Cloudflare's own comparison, much lower write amplification); the v1.5 "proxy mode" stopgap that cut per-server disk roughly in half before the full tiered redesign; and the v2 architecture of per-server cache, data-center-wide shared cache shards, dedicated storage-replica nodes, and relay-mode servers brokering the rest of the fleet's access to the storage tier. This session's automated fetch tools returned HTTP 403 on direct retrieval of both posts; figures here are cross-corroborated across independent search-engine summaries and the InfoQ writeup below rather than a single source, and should get a final primary-text check before being quoted as verbatim blog text.

**InfoQ: "How Cloudflare Migrated Quicksilver to Multi-Level Caching While Serving Billions of Requests"** (August 2025)
https://www.infoq.com/news/2025/08/cloudflare-key-value-store/
Secondary corroboration of the Quicksilver v2 numbers and architecture above, independently written summary of the same blog series.

**Cloudflare Community forum thread on the original Quicksilver KT-replacement post**
Search-surfaced detail (via Cloudflare TV session descriptions and community discussion) corroborating that Kyoto Tycoon's exclusive write lock was I/O sensitive, that write bursts degraded read latency past 1ms, that keeping KT operational consumed roughly 48 hours of SRE time a week, and that post-RocksDB average read latency settled around 500 microseconds. Treated as corroborated because this detail appeared consistently across multiple independent search results rather than a single page; recommend a direct primary-source read of blog.cloudflare.com/introducing-quicksilver-configuration-distribution-at-internet-scale/ to confirm exact wording before quoting it verbatim elsewhere.

**Hacker News: "Quicksilver: Configuration Distribution at Internet Scale"**
https://news.ycombinator.com/item?id=22726930
Discussion thread on the original 2020 announcement; a Cloudflare employee comment in a related thread (https://news.ycombinator.com/item?id=18892094) confirms Kyoto Tycoon was fully removed and replaced by Quicksilver.

**Cloudflare Blog: "Introducing Meerkat: an experiment in global consensus"**
https://blog.cloudflare.com/meerkat-introduction/
Primary source for the contrast used in this lesson's trade-offs section: Meerkat is a separate Cloudflare system, built specifically because Quicksilver's eventual consistency is not sufficient for control-plane state like leader election or AI model instance placement, using QuePaxa (a 2023 EPFL leaderless consensus algorithm) to achieve linearizable reads and writes across 330-plus data centers.

**Cloudflare Developer Docs: "How KV works"**
https://developers.cloudflare.com/kv/concepts/how-kv-works/
Source for Workers KV's documented eventual-consistency SLA (writes visible immediately at the writing location, up to 60 seconds or more to propagate globally). Cited here only as an illustrative, concretely-documented analogy for what "eventually consistent, propagating outward" means in real elapsed time; this is a distinct customer-facing product built on similar principles, not Quicksilver itself, and its numbers should not be read as Quicksilver's own propagation figures.

**Cloudflare Blog: "How we improved DNS record build speed by more than 4,000x"**
https://blog.cloudflare.com/dns-build-improvement/
Source for the upstream "Zone Builder" pipeline stage: compiling a zone's full DNS record set historically took up to 35 seconds for around 1 million records before Cloudflare's optimization work. This is the config-authoring step that happens before Quicksilver replication begins, not the replication step itself.

**Cloudflare network page**
https://www.cloudflare.com/network/
Source for Cloudflare's general network footprint claims (300-plus cities, a large majority of the internet-connected population within low-latency reach of a data center). Cloudflare's own marketing claim, not independently audited; used here only for general scale context, not as a Quicksilver-specific figure.
