# Day 41 — How does a filesystem serve exabytes of data across tens of thousands of machines, when the one process that knows where every file lives can only run on one machine?

*2026-07-27*

---

## 1. The company and the number that breaks a naive design

**Google, going from the Google File System (GFS, described publicly in 2003) to its successor Colossus (built starting around 2010, described publicly by Google Cloud in 2021).** GFS was Google's original answer to "how do you store more data than fits on any one machine": split every file into large chunks, spread the chunks across thousands of cheap commodity disks, and put one machine, the master, in charge of knowing which chunk lives where. It worked, and it is one of the most cited systems papers of the 2000s.

The number that eventually broke it is not a request-rate number, it's a *capacity-of-one-brain* number. Google's own account of why it replaced GFS is that a single Colossus cluster now scales to **exabytes of storage across tens of thousands of machines, more than 100x the largest GFS clusters ever ran.** GFS's entire design puts one property in one place: the mapping from "this file, this byte range" to "this chunk, on these three machines" lives in the memory of a single master process. A single machine's RAM and a single process's ability to serialize incoming requests is a hard ceiling, and Google's own storage footprint grew straight through it. A back-of-envelope version of the same wall: at GFS's 64 MB chunk size, one exabyte of data alone is about 15.6 billion chunks; even at GFS's own claimed **under 64 bytes of metadata per chunk**, that is roughly a terabyte of chunk-location data before you've counted the namespace tree, open-file state, or lease bookkeeping, all of it living in the memory of, and every mutation to it serialized through, one machine.

## 2. Why the naive design dies

The naive version, which is exactly what GFS itself was for its first several years: one master holds the whole namespace and the whole chunk-to-location map in memory, appends every mutation (create a file, allocate a chunk, rename, delete) to one operation log on one disk, and answers every client's "where is this file's data" question itself.

**a. One process's memory is a hard, non-negotiable ceiling.** You can put more RAM in one machine, but there is a largest machine that exists, and Google's data kept growing past whatever that number was. Unlike the actual file *data*, which GFS already spread across thousands of chunkservers from day one, the *metadata* had nowhere to spread to, because the whole point of a single master was having one place with global knowledge.

**b. Every metadata mutation funnels through one append-only log on one machine.** Creating a file, renaming it, allocating a new chunk: each of these has to be durably logged by the master before it's considered committed. That log lives on one machine's disk. As the rate of file creates/deletes/renames across all of Google's services grew (indexing pipelines, ever more products landing on the same storage layer), that single log's throughput, not the aggregate disk bandwidth of the whole cluster, becomes the limit on how fast the *filesystem itself* can change.

**c. The master is a single point of stall for the entire cluster, not just for writes.** Every client that wants to open a file, create a new one, or refresh which chunkservers hold a chunk it's about to read has to ask the master. GFS mitigated outright data loss here with an operation log plus periodic checkpoints (so a restarted master rebuilds state fast) and read-only "shadow masters" for staleness-tolerant reads, but every *authoritative* answer, every file creation, every chunk lease, still had exactly one machine that could give it. Lose that machine, even briefly, and the whole cluster's ability to create or open anything new stalls, no matter how many of the thousands of chunkservers are perfectly healthy.

The analogy: imagine one country with a single air-traffic control tower for every airport. A regional airline with fifty flights a day is no problem, one controller can hold every plane's position in their head and radio each pilot in turn. Scale that to a continent's worth of daily flights and the tower doesn't run out of runway, it runs out of a single human's attention and a single radio channel's throughput. Building more runways (more chunkservers, more disks) doesn't help at all if the one tower is what's actually maxed out.

## 3. The architecture, top to bottom

```
Clients (Search's indexing pipeline, Gmail, YouTube ingestion, BigQuery,
         Spanner, Google Cloud Storage — anything inside Google that
         needs to read or write a file)
   |  asks "where does this path live" — a metadata question,
   |  answered by the control plane, never by touching actual bytes
   v
Curators (many, horizontally sharded — GFS's "master, but split")
   each curator owns a partition of the file namespace and answers
   metadata questions for only its slice: create, open, rename,
   "which D servers hold this chunk." No single curator holds the
   whole filesystem's state in its own memory anymore.
   analogy: instead of one phone-book clerk for the whole country,
   many clerks, each responsible for one alphabetical slice of names
   |
   v
Bigtable (the durable, ordered key-value store curators persist
          metadata into, instead of a bespoke hand-rolled operation
          log per curator)
   analogy: the actual filing cabinet the clerks write into, built
   once, reused by every clerk, instead of every clerk keeping their
   own private, ad hoc notebook
   |  (curator hands back chunk/location info; client then talks
   |   to the data plane directly — the control plane never sees
   |   the file's bytes)
   v
D servers (the data plane — network-attached disks that hold the
           actual chunk bytes, analogous to GFS's chunkservers)
   clients read and write bytes straight to and from D servers once
   a curator has told them which ones to use; the metadata layer is
   never in the path of the actual data transfer
   analogy: the warehouse floor holding the physical boxes; the
   filing-cabinet clerk tells you which shelf, then steps out of
   the way while you carry the box yourself
   |
   v
Custodians (background processes: rebalance disk space across D
            servers, rebuild data after a disk or machine fails
            using replication or erasure coding, migrate cold data
            from flash to larger, cheaper drives as it ages)
   analogy: the overnight rebalancing and repair crew, always
   running, that nobody has to ask for
```

## 4. The transferable mechanisms

**a. Separate the control plane from the data plane, then shard the control plane too.** GFS already got the first half right: the master answers "where," but clients move actual bytes directly with chunkservers, so bulk data traffic never passes through the coordinator. Colossus's real innovation is realizing that separation alone is not enough forever, the coordinator itself needs the same sharding treatment as the data: many curators, each owning a slice of the namespace, instead of one master assumed to be big enough permanently. This is Day 10's consistent-hashing lesson (shard the thing that's getting too big) applied one layer up, to the metadata layer instead of the data layer.

**b. Don't hand-roll durability for your control-plane state, build it on a store that already solved that problem.** Curators persist metadata into Bigtable rather than each curator maintaining its own bespoke operation log and checkpoint scheme. Reuse a battle-tested, already-durable, already-scalable storage layer for your coordinator's own state, rather than re-solving "how do I make this durable and recoverable" independently at every layer of the stack.

**c. Lease-based ownership for who's allowed to order writes to a piece of data.** GFS's master grants a time-bounded lease (originally 60 seconds, renewable) to one chunk replica, the primary, and only that replica decides the serial order of concurrent mutations to that chunk. This is the same fencing-token-shaped primitive Day 24 covered for distributed locks: a coordinator hands out a time-bounded, unambiguous "you're the one in charge of ordering writes right now" token, so writers don't need to negotiate directly with each other, only briefly with whoever holds the lease.

**d. Large, coarse units of data to keep metadata proportional to structure, not to bytes.** GFS's 64 MB chunk size (versus a typical filesystem's few-KB block) means the amount of metadata to track grows with the number of *chunks*, not the number of *bytes*, buying real headroom, under 64 bytes of tracked metadata per 64 MB of actual data. That headroom is real but not infinite, which is exactly why Google still hit the ceiling eventually; it postpones the wall, it doesn't remove it.

**e. Tiered placement driven continuously by access pattern, not a one-time decision.** Colossus's custodians move hot data onto flash and let cold data age onto larger, cheaper drives automatically, in the background, as a continuously re-evaluated job rather than a placement choice made once at write time. The same idea generalizes to any system where "where should this live" has a right answer that changes over time as data ages.

**f. Background self-healing decoupled from the request path.** Custodians rebuild lost replicas (or reconstruct data from erasure-coded fragments, the mechanism Day 32 covers in depth) as an ongoing background process, not something a client request ever waits on. Durability repair and live traffic are two separate concerns running on two separate timelines.

## 5. The trade-offs

**Consistency vs. availability, and it's different for the data plane than the metadata plane.** File *data* gets replication or erasure coding, a durability-vs-storage-cost trade Day 32 already covers: 3x replication costs roughly 200% storage overhead for simple, fast reconstruction; Reed-Solomon erasure coding costs far less overhead (commonly cited in the 1.4-1.5x range for comparable durability) at the price of a costlier, slower reconstruction when a fragment is actually lost. File *metadata* gets strong consistency, but scoped per shard: each curator (backed by its Bigtable tablet) is the single authoritative source for its slice of the namespace, so a rename or create inside that slice is immediately, consistently visible everywhere. The availability win over GFS is structural: lose one curator or one Bigtable tablet, and only that slice of the namespace stalls, not the entire filesystem, the way losing GFS's one master stalled everything. That win costs real complexity: something now has to correctly route "which curator owns this path" for every single request.

**Cost vs. latency, made an explicit, continuously-tuned dial rather than a fixed choice.** Colossus's own public description of its storage strategy is to buy just enough flash to push I/O density per gigabyte up to what spinning disks can't provide, and just enough disk capacity for the exabytes, keeping disks full and busy rather than sitting idle and expensive. That's cost-vs-latency treated as a knob custodians turn continuously as access patterns shift, not a hardware decision made once per cluster.

## 6. The systems-thinking lens

**The feedback loop here is a single-coordinator bottleneck compounding on itself, the same shape as Day 16's hot-key problem, except the "hot key" is the entire filesystem namespace, concentrated onto one process by design.** Trace it: total file count and mutation rate grow across Google's services → the single master has more create/rename/delete/lease operations to serialize through one operation log on one machine → each operation takes longer to commit → clients waiting on metadata responses hold connections and retry longer → more concurrent outstanding requests pile up against the same one process → the master falls further behind, and average operation latency keeps climbing even though every one of the thousands of chunkservers holding actual data is completely healthy and underutilized. The bottleneck a client actually experiences has nothing to do with disk throughput; it's entirely queueing on one coordinator.

**The senior fix is structural, not "buy a bigger master."** Vertical scaling (a bigger machine, more RAM, a faster disk for the operation log) buys a fixed multiple and then hits the same wall one order of magnitude later; it doesn't change the shape of the problem, which is that global metadata is being made to fit through one gate. Colossus's fix is Day 13's backpressure lesson applied to a coordinator instead of a queue: shard the bottleneck itself, so growth in namespace size is absorbed by adding more curators and more Bigtable tablets, the same way adding more read replicas absorbs read growth or adding more chunkservers absorbs data growth. The load-bearing insight generalizes past filesystems: whenever one coordinator process is the thing that "knows everything" about a system, that coordinator is a hot key waiting to happen, and the fix is always to shard the knowledge, not to make the one place that holds it faster.

---

## References and summaries

**Google Cloud Blog. "A peek behind Colossus, Google's file system."**
https://cloud.google.com/blog/products/storage-data-transfer/a-peek-behind-colossus-googles-file-system
Primary source, Google's own official description of Colossus's architecture, used throughout sections 3 through 6: curators as the horizontally-scaled metadata layer built on Bigtable, D servers as the network-attached data plane, custodians as the background disk-balancing and reconstruction process, the claim that a single Colossus cluster scales to exabytes across tens of thousands of machines and more than 100x the largest GFS clusters, and the flash-versus-disk tiering strategy described in section 5.

**Ghemawat, S., Gobioff, H., and Leung, S-T. "The Google File System." SOSP 2003.**
https://research.google.com/archive/gfs-sosp2003.pdf
The original, primary paper for GFS's design: the single-master architecture, the 64 MB chunk size and its metadata-density rationale (under 64 bytes of metadata tracked per chunk), and the chunk-lease mechanism (a roughly 60-second, renewable lease granted to one replica, the primary, which alone decides the serial order of concurrent mutations to that chunk). Flagging per this lesson series' fact-vs-inference discipline: the direct PDF returned an access error (HTTP 403) during research for this lesson, so these details are drawn from this paper's well-documented, widely and independently summarized public content (consistently described the same way across multiple independent technical write-ups, including course material and engineering summaries, rather than from a direct read of the PDF in this session).

**SysTutorials. "Colossus: Google's Next-Generation Distributed File System."** and **Pierre Zemb. "What can be gleaned about GFS successor codenamed Colossus?"**
https://www.systutorials.com/colossus-successor-to-google-file-system-gfs/ and https://pierrezemb.fr/posts/colossus-google/
Secondary sources, corroborating and providing additional color on Colossus's motivation and design used in sections 1, 2, and 4: that GFS's single-master metadata design became the specific bottleneck Colossus was built to remove, and that Colossus adopted Reed-Solomon erasure coding for part of its durability strategy in addition to straight replication. Flagging per this lesson series' discipline: both pages returned access errors (HTTP 403) when fetched directly during research; the summarized claims above reflect consistent framing repeated across independent secondary sources indexed during research, not a direct read of either page's full text in this session.

**Chang, F. et al. (Google). "Bigtable: A Distributed Storage System for Structured Data." OSDI 2006.**
https://research.google/pubs/bigtable-a-distributed-storage-system-for-structured-data/
Background primary source for Bigtable itself, the durable, ordered key-value store that Colossus's curators persist metadata into (section 3 and section 4b). Well-established, widely cited paper; referenced here for context on what Bigtable is rather than for any Colossus-specific claim.

---

## Map to Rare.lab's stack

Rare.lab isn't running an exabyte-scale filesystem, but the shape of the problem, one coordinator process assumed to hold all the authoritative state forever, is already a design decision waiting to be made the moment Rare.lab's node-graph storage or its compiled-shader cache grows past what a single Supabase Postgres instance (even a well-tuned one) can comfortably serve as the one source of truth for "where does this scene's data live" and "what's the latest compiled artifact for this graph."

The concrete, actionable piece to borrow now: keep the separation Rare.lab already has, mostly by accident of using R2, between the control plane (Postgres holding the manifest: which scene, which version, which content hash) and the data plane (R2 holding the actual immutable, content-addressed scene JSON), and treat that separation as a deliberate architectural principle, not an implementation detail. As Rare.lab's catalog of scenes, shaders, and compiled runtime artifacts grows, the instinct from this lesson to actually apply is: if a single Postgres table or a single RLS-scoped query pattern ever becomes the thing every request has to pass through to find "where's the actual content," that's the same single-master shape GFS had, and the fix is the same one Colossus applied, shard that lookup layer (by project ID, by content-hash prefix, whatever the natural partition is) before it becomes the whole system's one hot gate, rather than reaching for a bigger Postgres instance and hoping the ceiling moves far enough away.
