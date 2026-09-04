# References: Instagram home-feed fan-out and delivery (2026-09-04)

Keepers for the feed-delivery / timeline fan-out problem.

## Primary and authoritative
- Instagram Engineering, "Open-sourcing a 10x reduction in Apache Cassandra tail latency" (Rocksandra; RocksDB storage engine for Cassandra; P99 read 60 ms -> 20 ms, GC stalls 2.5% -> 0.3%). https://instagram-engineering.com/open-sourcing-a-10x-reduction-in-apache-cassandra-tail-latency-d64f86b43589
- Meta Engineering, "TAO: The power of the graph" (graph-aware cache over sharded MySQL; objects + associations; leader/follower cache tiers; ~1B reads/sec at 96.4% follower-cache hit rate in 2013). https://engineering.fb.com/2013/06/25/core-infra/tao-the-power-of-the-graph/  (note: egress-blocked in this environment; verified via summaries below)
- Micah Lerner, paper summary of "TAO: Facebook's Distributed Data Store for the Social Graph" (USENIX ATC '13), including 2021 figure of 10B+ reads/sec. https://www.micahlerner.com/2021/10/13/tao-facebooks-distributed-data-store-for-the-social-graph.html
- Lisa Guo, "Scaling Instagram Infrastructure," QCon London 2017 (Django/Celery/RabbitMQ, Memcache, memcache leases against thundering herd, Cassandra). https://qconlondon.com/london-2017/london-2017/presentation/scaling-instagram-infrastructure.html
- Nishtala et al., "Scaling Memcache at Facebook," USENIX NSDI 2013 (leases to stop stale sets and thundering herd). https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final170_update.pdf

## Fan-out / timeline delivery (class-of-problem, used as labeled inference)
- Twitter/X news feed: fan-out-on-write vs fan-out-on-read, the celebrity problem, hybrid split by follower count (Katy Perry ~80M followers example). https://www.techinterview.org/post/3233474168/system-design-twitter-news-feed-timeline-fanout-on-write-fanout-on-read-celebrity-problem-ranking-caching/
- "Fan-out on Write vs Fan-out on Read: The Core Trade-off." https://wittycoder.in/courses/news-feed/fan-out-strategies
- ByteByteGo, "How Instagram Scaled Its Infrastructure To Support a Billion Users." https://blog.bytebytego.com/p/how-instagram-scaled-its-infrastructure

## Key takeaways worth remembering
- The two data shapes: author timeline (a poster's own posts) vs the per-user home-feed inbox (a precomputed list of post IDs pushed to you). Reads hit the inbox: O(N) list read, not O(followees) scan.
- Hybrid by follower count: push (fan-out on write) for normal accounts, pull (fan-out on read) + merge for celebrities. Threshold = crossover of write-amplification vs read-frequency cost.
- Sorting is server-side. Matching (gather candidates) and ranking (order them) are two halves, both server-side.
- Scale breakpoints: 1k = pure pull is fine; 100k = switch to push + materialized inboxes + Memcache; 10M+/2B = hybrid + queue-fed async fan-out + shard inboxes by user id + graph-cache the follower-list reads.
