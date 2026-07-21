# References: Spotify Wrapped data pipeline

Saved 2026-07-21 for the teardown of Spotify Wrapped (the annual recap + its batch data pipeline).

## Primary (Spotify engineering)

- Spotify Engineering, "How Spotify Optimized the Largest Dataflow Job Ever for Wrapped 2020" (2021):
  https://engineering.atspotify.com/2021/02/how-spotify-optimized-the-largest-dataflow-job-ever-for-wrapped-2020
  Key facts: Sort Merge Bucket to join ~1PB with no conventional shuffle and no Bigtable intermediate; ~50% lower Dataflow cost vs previous Bigtable approach; ~50% storage reduction from collocation/compression; avoided scaling Bigtable 2-3x.

- Spotify Engineering, "Spotify Unwrapped: How we brought you a decade of data" (Wrapped 2019, 2020 post):
  https://engineering.atspotify.com/2020/02/spotify-unwrapped-how-we-brought-you-a-decade-of-data
  Key facts: decade edition ~5x larger than 2018 at ~3/4 the cost; decomposed into data collection, aggregation, transformation; per-user Bigtable row, per-data-story column families; Bigtable as remediation layer between Dataflow jobs to avoid regrouping/shuffle.

- Spotify Engineering, "Big Data Processing at Spotify: The Road to Scio (Part 1)" (2017):
  https://engineering.atspotify.com/2017/10/big-data-processing-at-spotify-the-road-to-scio-part-1
  Scio = Scala API for Apache Beam, runs on Google Cloud Dataflow (managed, autoscaling, dynamic work rebalancing).

- Scio docs, "Sort Merge Bucket":
  https://spotify.github.io/scio/extras/Sort-Merge-Bucket.html
  SMB writes to deterministic file locations bucketed by hash of key and sorted by key, so later joins are merge-sorts of matching buckets with no shuffle.

- Scio project on GitHub:
  https://github.com/spotify/scio

## Secondary / coverage

- TechCrunch, "How Spotify ran the largest Google Dataflow job ever for Wrapped 2019" (2020):
  https://techcrunch.com/2020/02/18/how-spotify-ran-the-largest-google-dataflow-job-ever-for-wrapped-2019/

- ByteByteGo, "How Spotify Built Its Data Platform To Understand 1.4 Trillion Data Points":
  https://blog.bytebytego.com/p/how-spotify-built-its-data-platform

- KTH master's thesis, Andrea Nardelli, "Sort Merge Buckets: Optimizing Repeated Skewed Joins in Data Flow" (2018), the origin of Scio's SMB:
  https://kth.diva-portal.org/smash/get/diva2:1334587/FULLTEXT01.pdf

## Retention / virality numbers

- Forbes, "Spotify Wrapped 2023 ... How It Became A Viral And Widely Copied Marketing Tactic":
  https://www.forbes.com/sites/conormurray/2023/11/28/spotify-wrapped-2023-comes-soon-heres-how-it-became-a-viral-and-widely-copied-marketing-tactic/
  227M users shared in 2023; ~2.3B social impressions; 170 markets, 35+ languages.

- Sprout Social, "Spotify Wrapped: What marketers can learn from the viral campaign":
  https://sproutsocial.com/insights/spotify-wrapped/
  ~400M posts on X in 3 days after 2022 launch; ~21% jump in app downloads first week of Dec 2020.
