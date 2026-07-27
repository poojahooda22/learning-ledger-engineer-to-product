# Netflix: "Continue Watching" and cross-device resume (the viewing-history and bookmark engine)

Date: 2026-07-27
Product: Netflix
Feature: The "Continue Watching" row and the bookmark that remembers where you stopped, across every device

---

## 1. The user

Priya is on the Mumbai metro at 8:40pm, standing, one hand on the pole, the
other on her phone. She is watching "Stranger Things" season 4, episode 7. The
train hits her stop at the 41 minute 12 second mark, mid-scene. She locks the
phone and gets off.

Ninety minutes later she is on her sofa. She picks up the TV remote, opens
Netflix, and the very first thing on the home screen, top-left, is a row called
"Continue Watching for Priya." The Stranger Things tile is right there with a
thin red progress bar under it, about two-thirds full. She presses play. It
starts at 41 minutes 12 seconds. Not the start of the episode. Not the start of
the scene. The exact second she stopped, on a completely different device, on a
different network, in a different room.

She never thinks about any of this. That is the point.

## 2. The real problem

Watching a show is not one sitting. It is forty sittings spread over three
weeks, across a phone, a laptop, a tablet, a TV, and sometimes a friend's
console. A season of Stranger Things is around 9 hours. Nobody watches 9 hours
straight.

So the real pain, described like a friend would: "I was 41 minutes into episode
7. I do not remember the exact spot, I do not want to drag the scrubber around
hunting for it, and I definitely do not want to rewatch 41 minutes I already
saw. Also, do not make me remember which episode I was even on. Just put me back
where I was, on whatever screen I happen to be holding."

Two hard parts hide inside that simple wish:

1. Remember the exact position (down to the second) for every title, for every
   member, forever, and make it survive a device switch that might happen
   seconds later or three weeks later.
2. Surface the right handful of "you were watching these" tiles instantly on a
   home screen that has to render in a fraction of a second, for hundreds of
   millions of members, many of them opening the app at the exact same 8pm.

## 3. The feature in one sentence

Continue Watching is a per-member, per-title bookmark of your exact playback
position that is written constantly as you watch and read back instantly on any
device, surfaced as a home-screen row so you can resume with one tap.

## 4. Jobs to be done

- "Put me back exactly where I stopped, without me hunting for the spot."
- "Remember which episode I am on so I do not have to."
- "Let me start on my phone and finish on my TV with zero friction."
- "Show me what I have not finished, ranked so the thing I actually want to
  resume is near the front."
- "Do not lose my place if the app crashes, the train goes into a tunnel, or my
  battery dies mid-episode."

## 5. How it works for the user

While Priya watches, the player quietly reports her position. She does nothing.
When she stops, the last reported position is her bookmark.

When she opens Netflix anywhere, the home screen shows the Continue Watching
row. Each tile has a red progress bar showing how far in she is. Tapping the
tile resumes from the bookmark. Finish an episode and Netflix advances the
bookmark to the next episode, so the row now points her at episode 8. Finish the
whole season and the title politely drops out of the row so it does not clutter
the screen.

There is also a "remove from row" option, because sometimes you abandon a show
and do not want it staring at you every night.

## 6. The actual flow, step by step

On the phone, on the metro:

1. Priya taps the Stranger Things tile. The app asks the viewing service "where
   is she in this title?" and gets back 0 for a fresh episode, or a saved
   position for a resume.
2. Playback starts. From this moment the player sends periodic progress updates:
   "member 12345, title S4E7, position 00:05, still playing," then "position
   00:35," and so on. (The exact cadence is not public. Inference: a lightweight
   heartbeat every few seconds to tens of seconds, plus an immediate send on
   pause, seek, and stop, because those are the moments the position actually
   matters.)
3. At 41:12 the train arrives, she locks the phone. The player fires a final
   "stopped at 00:41:12" update. That number is now the live truth for this
   member and title.

On the TV, 90 minutes later:

4. The home screen loads. It asks for this member's Continue Watching list. Back
   comes a small ranked set of titles, each with its saved position and progress
   fraction.
5. The Stranger Things tile renders with a two-thirds-full red bar (41 of ~62
   minutes).
6. She presses play. The player reads the bookmark, 00:41:12, and tells the
   streaming pipeline to start delivering video from that offset. Playback
   resumes at the exact second.

The magic is that step 3 (write on the phone) and step 4 (read on the TV) touch
the same member record, and the write was visible to the read even though only
90 minutes and a network switch separated them.

## 7. Under the hood, like the engineer

This is the heart of it. Continue Watching is really two engineering problems
wearing one coat.

- A tiny, blisteringly hot, write-heavy problem: capture the live position of
  every active view (tens of millions of streams at prime time), each one
  updating every few seconds.
- A large, cold, storage problem: keep the full viewing history and last
  positions for every member and every title they ever touched, for years, and
  read it back fast.

Netflix's published answer splits the system by exactly that seam.

### The two-tier viewing service (the classic design)

Fact, from Netflix's QCon 2014 talk and the TechBlog post "Netflix's Viewing
Data: How We Know Where You Are in House of Cards." The viewing service is split
into a stateful tier and a stateless tier.

The stateful tier holds the latest data for all active views in memory. Members
are partitioned across N stateful nodes by a simple `account_id mod N`. So Priya
watching right now lives entirely in the RAM of one specific node. Why memory
and not a database on the hot path? Because this is the highest-volume read and
write use case in the whole system (every active stream pinging its position),
and at the time Cassandra was pre-1.0 and not yet running on SSDs in AWS, so
memory speed was the only way to eat that write rate. The active bookmark for
Priya's Stranger Things stream is, at that instant, just an entry in a hash map
on one machine: key is her account id, value is the latest position per active
title.

That memory-first choice has a cost the design accepts on purpose. `mod N`
partitioning is prone to hot spots (load is not evenly spread), and the active
state is not replicated, so if a stateful node dies its in-flight positions
degrade gracefully to the last durable value rather than being perfectly
preserved. Losing a few seconds of "exact position" is an acceptable failure. It
is a bookmark, not a bank balance.

The stateless tier is the durable memory. Cassandra is the primary store for all
persistent viewing data, chosen because it handles very high volume, low latency
writes and models time-series data naturally (each member row can hold a dynamic
number of columns, one per viewing event). In front of Cassandra sits a
memory cache (Memcached in the original design, EVCache in the later one) that
serves very high volume, low latency reads of materialized, possibly slightly
stale, views. After a write lands in Cassandra it is pushed back into the cache;
staleness is bounded by short TTLs and periodic refresh.

So the write path for "stopped at 41:12" is: player update lands on Priya's
stateful node (instant, in memory), and is also durably persisted down into
Cassandra with the cache updated on top. The read path for the TV is: ask the
cache first, and on a miss fall through to Cassandra. This is the ledger's
recurring shape again, offline-think or in this case hot-tier-think, online cheap
lookup: the expensive high-frequency churn happens in the fast tier, and the
thing the home screen reads is a cheap keyed fetch.

### The data model, and why these structures

The natural shape of viewing history is a time series per member: an
append-mostly list of "at time T, member M was at position P in title X." That
maps to a wide-row model.

Fact, from Netflix's "Scaling Time Series Data Storage" TechBlog series. The
row key is the CustomerId. Each viewing event is a column in that row, keyed and
ordered by time. Reads are point lookups by customer id or time-range scans over
recent columns. A hash-map-like keyed store (Cassandra) is exactly right here
because the only questions are "give me this member's row" and "give me the
recent slice," never "scan everyone." An inverted index or a B-tree over the
whole catalog would be the wrong tool; there is no cross-member search on the hot
path.

But one member row grows without bound. A heavy viewer builds up thousands of
viewing records over years. A single unbounded partition is poison for
Cassandra: reads get slower as the row gets fatter, and the row can outgrow
comfortable limits. So Netflix split the model into two by temperature.

- LiveVH (Live Viewing History): a small number of recent records, frequently
  updated, stored uncompressed for fast read and write. This is where Priya's
  fresh Stranger Things progress lives.
- CompressedVH (Compressed or Archival Viewing History): the large tail of old
  records that rarely change, rolled up, compressed, and stored as basically one
  compact blob per chunk to shrink the storage footprint.

A background rollup task periodically takes older LiveVH records, compresses
them, and moves them into CompressedVH. This is the same log-structured instinct
as an LSM-tree: keep the hot writable head small and cheap, sweep the cold tail
into compacted archival form offline.

And because even the compressed archive can be huge for a decade-long member,
CompressedVH is chunked. The row key becomes `CustomerId$Version$ChunkNumber`,
with a small metadata row holding the chunk count and version. A read is bounded
to two steps: read the metadata (how many chunks, which version), then read the
chunks in parallel. A write is similarly bounded: write the new chunk(s), then
write the metadata. So even a member with a massive history has a read cost that
is a couple of round trips plus a parallel fan-out, not a linear crawl through
one enormous partition. Versioning also makes the rollup safe: build the new
compressed version alongside the old, flip the metadata pointer, done.

Fact: the EVCache layer in front of this, keyed by customer id with the
compressed history as the value, runs at close to a 99% hit rate. Nearly every
Continue Watching read is served from memory. Cassandra only sees the misses.
The later design added a summary cache that stores precomputed per-member
summaries, so the home screen fetches a ready-made answer instead of recomputing
"what should be in this row" on every open.

### The 2026 twist: dynamic partition splitting for the wide row

The wide-partition problem never fully goes away, and Netflix published a fresh
fix in June 2026 (covered by InfoQ and MarkTechPost in July 2026). Their
TimeSeries Abstraction, the internal platform that ingests and queries petabytes
of temporal event data on Apache Cassandra 4.x, hit the same wall: some
partitions grow past 500MB and their reads blow out to seconds.

Fact, the fix: split wide partitions per TimeSeries ID, asynchronously and
transparently, with no application changes and no downtime.

- Detection lives on the read path. As reads stream a partition, the system
  counts bytes, and when a partition crosses the threshold it emits a Kafka
  event to trigger a split. Detection is a side effect of work already being
  done, so it costs almost nothing.
- Splitting targets immutable partitions first (the cold, done-being-written
  ones), because moving data that is not changing underneath you is safe and
  simple. Same instinct as compressing CompressedVH and not LiveVH.
- After a split, reads are routed to the smaller child partitions using Bloom
  filters (single-digit microsecond checks) plus a cached `wide_row` metadata
  lookup that says which children exist. A Bloom filter here is the perfect tool:
  it answers "could this id's data be in this partition?" in microseconds and
  never gives a false negative, so it prunes the search to the right child
  cheaply.

Result: reads on oversized partitions dropped from seconds to low double-digit
milliseconds, tail latency fell to around 200ms, and 500MB-plus partitions
stayed available the whole time. This is the chunking idea from
"Scaling Time Series" grown up into something automatic and self-healing:
instead of a fixed chunk scheme, the system watches its own read path and splits
when reality demands it.

### The scale story, three tiers

Walk the same feature at three sizes, with Priya's bookmark as the unit.

At 1,000 members: trivial. One database table, a row per member, a column per
viewing event, and a `SELECT` for the Continue Watching row. No cache needed. A
single Postgres box would not notice the load. At this size the two-tier
architecture would be pure over-engineering.

At 100,000 members: the write rate starts to bite. At prime time thousands of
streams are each pinging their position every few seconds, so the hot path is
now write-dominated. This is where you split hot from cold: keep active-view
positions in an in-memory tier so the constant heartbeats never touch disk, and
persist to a time-series store behind a cache for the durable read. You also
discover that recomputing the Continue Watching row on every home-screen open is
wasteful, so you start caching a materialized per-member summary. Individual
member rows are still small enough that one partition per member is fine.

At 10 million plus members (Netflix is well past 300 million now, and the older
public figure was already around 140 million hours of viewing recorded per day):
three things break and each has a survival move.
- The active-view firehose. Answer: the stateful in-memory tier, sharded
  `account_id mod N`, absorbs the write storm at memory speed; durable writes go
  down to Cassandra asynchronously. Accept that a node death loses a few seconds
  of live position, because a bookmark can be slightly stale.
- The read fan-out at 8pm when everyone opens the app at once. Answer: EVCache in
  front of Cassandra at a ~99% hit rate, plus a summary cache so the home screen
  reads a precomputed row. Cassandra only serves the 1% of misses.
- The unbounded per-member partition. Answer: LiveVH/CompressedVH temperature
  tiering, chunked archival rows keyed `CustomerId$Version$ChunkNumber`, and, as
  of 2026, automatic dynamic partition splitting that detects a fat partition on
  the read path and carves it into Bloom-filter-routed children so reads stay in
  milliseconds even at 500MB.

The through-line: the hot ephemeral state (where you are right now) and the cold
durable state (everywhere you have ever been) have opposite access patterns, so
they get opposite machinery. Trying to serve both from one store is what breaks
at the next tier every single time.

## 8. The retention and habit mechanic

Continue Watching is arguably the single most important retention surface
Netflix has, and it is not flashy. It is the on-ramp.

The loop: episodic content is built on cliffhangers, so every stop leaves an open
thread. Continue Watching turns that open thread into a one-tap resume that sits
top-left, the first thing your eye lands on and the first thing the remote
highlights. The cost of picking the show back up is driven to nearly zero. Low
resume friction means more sessions started, more episodes finished, more of a
season completed, and finishing a season is strongly tied to sticking around for
the next one.

The metric it moves is retention and engagement, not activation or direct
revenue. A member who can frictionlessly resume across phone, commute, and couch
watches more, and a member who watches more churns less. The row is also honest
housekeeping: finished titles drop out, so the row always reads as "unfinished
business you care about," which keeps it a high-signal, high-tap surface instead
of clutter.

Real observed example: the whole reason the House of Cards TechBlog post exists
is that binge viewing of Netflix originals made accurate, instant cross-device
resume a first-class product requirement. When your product is "watch a 13-hour
season over two weeks on five devices," the bookmark is not a nicety, it is the
thing that makes the binge physically possible.

## 9. The lesson for Rare.lab

Split state by temperature and access pattern, and keep the hot tier a cheap
keyed lookup.

Rare.lab is a node-based shader editor plus an embeddable runtime, and it has the
exact same two-halves shape as Netflix viewing data.

- The hot, ephemeral half: while an artist is editing, or while the embedded
  runtime is playing an effect, there is a constant stream of tiny state updates,
  the current parameter values, the playhead position in a timeline, the live
  values feeding each node this frame. That is Priya's active bookmark. Do not
  write it to durable storage on every tick. Hold it in a fast in-memory session
  tier keyed by session id, and accept that a crash loses the last few
  milliseconds, because a live scrub position is a bookmark, not a bank balance.
- The cold, durable half: the saved project, the version history, the per-user
  library of shaders they have ever built. That is CompressedVH. Roll the hot
  session state down into durable, compressed, versioned storage on a background
  cadence (a periodic checkpoint), not on every keystroke.

Then take the two scaling lessons directly:

1. Materialize and cache the thing the UI reads. Netflix's home screen reads a
   precomputed Continue Watching summary at a 99% cache hit rate, not a live
   recomputation. Rare.lab's gallery, the "recent projects" row, the
   shader-variant list for a given device, should each be a precomputed row keyed
   by user or device and served from cache, so opening the editor is one keyed
   fetch, never a scan.

2. Never let a single entity's log grow into one unbounded partition. A
   power user's project with a decade of edit history, or a popular shader's full
   telemetry stream, is Netflix's fat member row. Chunk it by version like
   `ProjectId$Version$ChunkNumber` so any read is bounded to a metadata lookup
   plus a parallel chunk fetch, and borrow the 2026 trick: detect the oversized
   partition on the read path by counting bytes, split the immutable older chunks
   asynchronously, and route reads to the children with a cheap Bloom-filter
   check. Keep read latency flat as the data grows instead of letting your
   heaviest, most valuable users get the slowest experience.

The concrete win: the artist scrubbing a timeline and the artist reopening a
year-old project hit two completely different subsystems tuned for opposite
things, and neither one ever waits on the other.

---

## Sources

- Netflix TechBlog, "Netflix's Viewing Data: How We Know Where You Are in House
  of Cards" (viewing service, stateful/stateless tiers, in-memory active views,
  `account_id mod N`, Cassandra + Memcached):
  https://netflixtechblog.com/netflixs-viewing-data-how-we-know-where-you-are-in-house-of-cards-608dd61077da
- Netflix TechBlog, "Scaling Time Series Data Storage, Part I and Part II"
  (LiveVH/CompressedVH, chunking with `CustomerId$Version$ChunkNumber`, rollup
  compression, EVCache ~99% hit rate):
  https://netflixtechblog.com/scaling-time-series-data-storage-part-i-ec2b6d44ba39
  and https://medium.com/netflix-techblog/scaling-time-series-data-storage-part-ii-d67939655586
- Netflix TechBlog, "Dynamically Splitting Wide Partitions in Cassandra for Time
  Series Workloads" (TimeSeries Abstraction, Cassandra 4.x, read-path byte
  counting + Kafka, split immutable partitions first, Bloom filters, `wide_row`
  metadata, seconds to low double-digit ms, ~200ms tail, 500MB+ partitions),
  June 2026.
- InfoQ, "Netflix Cuts Cassandra Read Latency from Seconds to Milliseconds with
  Dynamic Partition Splitting" (July 2026):
  https://www.infoq.com/news/2026/07/netflix-cassandra-partition/
- MarkTechPost, "Netflix AI Team Cuts Wide-Partition Read Latency from Seconds to
  Milliseconds by Splitting Cassandra Partitions Per ID" (July 2026):
  https://www.marktechpost.com/2026/07/08/netflix-ai-team-cuts-wide-partition-read-latency-from-seconds-to-milliseconds-by-splitting-cassandra-partitions-per-id/
- ByteByteGo, "How Netflix Stores 140 Million Hours of Viewing Data Per Day"
  (LiveVH/CompressedVH, EVCache, scale figure):
  https://blog.bytebytego.com/p/how-netflix-stores-140-million-hours
- puncsky / system-design-and-architecture, "How to design Netflix view state
  service" (two-tier, mod N partitioning, hot spots, CAP tradeoff, Memcached
  cache-update strategy):
  https://github.com/puncsky/system-design-and-architecture/blob/master/en/45-how-to-design-netflix-view-state-service.md
