# References: Day 14 — Multi-Region Active-Active

## Primary Engineering Sources

- Cloudflare: "Sequential consistency without borders: how D1 implements global read replication" (March 2026) — https://blog.cloudflare.com/d1-read-replication-beta/
- Cloudflare: "Building D1: a Global Database" (April 2024) — https://blog.cloudflare.com/building-d1-a-global-database/
- CockroachDB: "Multi-Active Availability" — https://www.cockroachlabs.com/docs/stable/multi-active-availability
- CockroachDB: "How to build a highly available database for a multi-region architecture" — https://www.cockroachlabs.com/blog/build-a-highly-available-multi-region-database/
- Redis: "Active-Active vs Active-Passive" — https://redis.io/blog/active-active-vs-active-passive/
- AWS: "Latency-based routing with Amazon CloudFront for a multi-region active-active architecture" — https://aws.amazon.com/blogs/networking-and-content-delivery/latency-based-routing-leveraging-amazon-cloudfront-for-a-multi-region-active-active-architecture/
- Microsoft Learn: "Retry Storm Antipattern" — https://learn.microsoft.com/en-us/azure/architecture/antipatterns/retry-storm/

## Clocks and Consistency

- Kevin Sookocheff: "Hybrid Logical Clocks" — https://sookocheff.com/post/time/hybrid-logical-clocks/
- AWS Database Blog: "How Aurora DSQL uses clocks" — https://aws.amazon.com/blogs/database/everything-you-dont-need-to-know-about-amazon-aurora-dsql-part-5-how-the-service-uses-clocks/

## Research Papers

- "Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases" — Kulkarni et al., 2014 (original HLC paper)
- "Spanner: Google's Globally-Distributed Database" — Corbett et al., 2012 — https://research.google/pubs/pub39966/

## Books

- "Designing Data-Intensive Applications" — Martin Kleppmann, O'Reilly 2017 (Chapters 5, 6, 9)
