# References: Netflix Top 10 row and approximate top-k counting (2026-10-25)

## Netflix methodology and scale (primary / near-primary)
- Netflix, "Top 10 things about Netflix Top 10" (methodology overview): https://about.netflix.com/news/top-10-things-about-netflix-top-10
- Netflix, "What We Watched: A Netflix Engagement Report" (18,000+ titles = ~99% of viewing, ~100B hours, The Night Agent S1 = 812.1M hours, Jan-Jun 2023): https://about.netflix.com/en/news/what-we-watched-a-netflix-engagement-report
- Advanced Television, weekly Top 10 website launch, 17 Nov 2021: https://www.advanced-television.com/2021/11/17/netflix-launches-top-10-wesbite/
- Digital TV Europe, global Top 10 rollout Feb 2020 after UK/Mexico trials (daily, country-level): https://digitaltveurope.com/2020/02/25/netflix-rolls-out-top-10-feature-following-mexico-and-uk-trials

## The counting-unit history
- Tubefilter, 2-minute rule from Q4 2019 letter (~35% higher): https://tubefilter.com/2020/01/22/netflix-how-calculates-view-counts-the-witcher-you-number-of-viewers-q4-earnings/
- BGR, why the 2-minute rule, then Nov 2021 switch to 28-day hours viewed: https://www.bgr.com/2192308/what-netflix-2-minute-rule-is-explained/
- TVLine, June 2023 switch to Views = hours / runtime, window 28 to 91 days: https://tvline.com/ratings/netflix-viewership-changes-wednesday-squid-game-most-popular-1235003018

## Netflix data pipeline (Keystone)
- InfoQ, "Netflix: Migrating Stream Processing" (Keystone on Kafka + Flink, SPaaS): https://www.infoq.com/articles/netflix-migrating-stream-processing
- StreamNative, "Data Mesh and Streaming SQL at Netflix" (20,000+ Flink jobs, thousands of Kafka topics, 2025): https://streamnative.io/videos/dss-virtual-2025-building-the-next-generation-of-real-time-data-pipelines-data-mesh-and-streaming-sql-at-netflix

## Approximate top-k / heavy hitters (the counting trick)
- Redis Top-K (HeavyKeeper: count-min-like table + min-heap, exponential decay, may briefly misrank, converges): https://redis.io/docs/latest/develop/data-types/probabilistic/top-k/
- Cormode & Muthukrishnan, Count-Min Sketch (2005), bounded memory, never undercounts: http://dimacs.rutgers.edu/~graham/pubs/papers/cm-full.pdf
- Metwally, Agrawal, El Abbadi, Space-Saving top-k in streams (2005): https://www.cs.ucsb.edu/sites/default/files/documents/2005-23.pdf

## Key takeaways worth keeping
- The feature LOOKS like sorting the catalog; it is actually counting a firehose. Catalog
  is tiny (~18k titles), the event stream is the scale wall.
- Two halves: COUNT (expensive, streaming aggregation) then SELECT TOP-K (cheap, min-heap).
- Two counting paths matched to two needs: approximate + fresh for the in-app daily row,
  exact + audited (EY) for the weekly public headline.
- Offline-think, online-lookup: the client reads ten precomputed IDs, never sorts.
- Approximate counting puts the error in the tail (ranks 8 to 10) where it does not matter
  and keeps the head (ranks 1 to 2) solid, which is exactly what heavy-hitter sketches do.
