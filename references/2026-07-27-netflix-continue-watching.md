# References: Netflix Continue Watching / viewing-history + bookmark engine (2026-07-27)

Keeper links for the viewing-data teardown. Primary Netflix engineering first.

## Netflix primary (engineering blog / talks)

- Netflix TechBlog, "Netflix's Viewing Data: How We Know Where You Are in House of Cards"
  https://netflixtechblog.com/netflixs-viewing-data-how-we-know-where-you-are-in-house-of-cards-608dd61077da
  The canonical viewing-service post. Stateful tier (active views in memory,
  `account_id mod N`) vs stateless tier (Cassandra primary store + Memcached for
  low-latency reads). Explains why memory-first for the hot write path (Cassandra
  was pre-1.0, not on SSD in AWS at the time).

- Netflix TechBlog, "Scaling Time Series Data Storage, Part I"
  https://netflixtechblog.com/scaling-time-series-data-storage-part-i-ec2b6d44ba39
  LiveVH (recent, uncompressed, frequently updated) vs CompressedVH (archival,
  compressed, single blob per chunk). Rollup background task. Row key CustomerId;
  each viewing event a time-ordered column. EVCache in front at ~99% hit rate.

- Netflix TechBlog, "Scaling Time Series Data Storage, Part II"
  https://medium.com/netflix-techblog/scaling-time-series-data-storage-part-ii-d67939655586
  Chunking of CompressedVH with row key `CustomerId$Version$ChunkNumber` + a
  metadata row (chunk count, version). Reads bounded to metadata read + parallel
  chunk reads; writes bounded similarly. Summary cache cluster for precomputed
  per-member summaries.

- Netflix TechBlog, "Dynamically Splitting Wide Partitions in Cassandra for Time
  Series Workloads" (June 2026)
  TimeSeries Abstraction on Apache Cassandra 4.x (petabytes of temporal event
  data). Wide-partition problem (>500MB partitions read in seconds). Fix: split
  per TimeSeries ID, async, transparent, no app changes. Read-path byte-counting
  detection + Kafka event; split immutable partitions first; Bloom filters
  (single-digit microseconds) + cached `wide_row` metadata route reads to child
  partitions. Result: seconds -> low double-digit ms, ~200ms tail latency.

## Secondary / coverage

- InfoQ, "Netflix Cuts Cassandra Read Latency from Seconds to Milliseconds with
  Dynamic Partition Splitting" (July 2026)
  https://www.infoq.com/news/2026/07/netflix-cassandra-partition/

- MarkTechPost, "Netflix AI Team Cuts Wide-Partition Read Latency from Seconds to
  Milliseconds by Splitting Cassandra Partitions Per ID" (2026-07-08)
  https://www.marktechpost.com/2026/07/08/netflix-ai-team-cuts-wide-partition-read-latency-from-seconds-to-milliseconds-by-splitting-cassandra-partitions-per-id/

- ByteByteGo, "How Netflix Stores 140 Million Hours of Viewing Data Per Day"
  https://blog.bytebytego.com/p/how-netflix-stores-140-million-hours
  Clean summary of LiveVH/CompressedVH + EVCache; source of the ~140M
  hours/day historical scale figure.

- puncsky / system-design-and-architecture, "How to design Netflix view state service"
  https://github.com/puncsky/system-design-and-architecture/blob/master/en/45-how-to-design-netflix-view-state-service.md
  Two-tier breakdown, mod N hot spots, CAP tradeoff (consistency over availability
  in the stateful tier, no replication of active state), Memcached cache-update
  strategy (short TTL + periodic refresh, eventual consistency).

## Notes / open questions

- Bookmark write cadence (heartbeat interval) is NOT public. Teardown labels the
  "every few seconds to tens of seconds + immediate on pause/seek/stop" as
  inference.
- Continue Watching row *ranking* (which unfinished titles show and in what order)
  is a separate personalization problem, not covered in depth here; this teardown
  is about the bookmark capture + storage engine, not the row's ML ordering.
