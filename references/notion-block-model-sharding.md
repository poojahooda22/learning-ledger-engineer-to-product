# References: Notion block model + slash command + sharding

Keeper links for the 2026-06-25 teardown.

## Primary (Notion engineering)
- Exploring Notion's data model: a block-based architecture
  https://www.notion.com/blog/data-model-behind-notion
  Block = id (UUID v4), type, properties (hash map), content (ordered array of
  child block IDs / pointers), parent. The assembled "render tree."
- Herding elephants: lessons learned from sharding Postgres at Notion (Oct 2021)
  https://www.notion.com/blog/sharding-postgres-at-notion
  ~20B block rows by 2020; CPU spikes, VACUUM/wraparound risk, PgBouncer near
  limits. Shard by workspace_id. 32 physical instances x 15 logical shards = 480
  logical shards. Schema-per-shard (schema001.block ...), routing in the app.
  Audit-log migration beat logical replication (which lagged on the bulk snapshot).
  Cutover cost ~seconds.
- The Great Re-shard: adding Postgres capacity (again) with zero downtime (2023)
  https://www.notion.com/blog/the-great-re-shard
  32 -> 96 physical instances, 15 -> 5 logical shards per box, SAME 480 logical
  shards so no re-partition (whole shards just move). PgBouncer as traffic
  controller. Peak CPU/IOPS ~90% -> ~20%. Worst case ~1s "Saving" spinner.
- Building and scaling Notion's data lake
  https://www.notion.com/blog/building-and-scaling-notions-data-lake
  CDC (Debezium) -> Kafka -> Spark -> S3 (Hudi) to offload whole-corpus scans,
  search, and AI from the transactional shards.
- How we made Notion available offline
  https://www.notion.com/blog/how-we-made-notion-available-offline
  Operations grouped into transactions, optimistic local apply, TransactionQueue
  (IndexedDB/SQLite), RecordCache LRU, /saveTransactions, MessageStore WebSocket
  notify, syncRecordValues.

## Secondary deep-dives
- ByteByteGo, Storing 200 billion entities: Notion's data lake project
  https://blog.bytebytego.com/p/storing-200-billion-entities-notions
  200B+ blocks by 2024 (doubling roughly every 6-12 months).
- Quastor, How Notion sharded their Postgres database
  https://blog.quastor.org/p/notion-sharded-postgres-database-8af4
- Hacker News discussion, The Great Re-shard
  https://news.ycombinator.com/item?id=43157296

## Why 480 logical shards
Divisible by many factors (2,3,4,5,6,8,10,12,15,16,...), so physical hosts can be
added or removed while keeping shards evenly distributed. The 2023 re-shard proved
it: tripling hosts moved whole logical shards without re-partitioning data.

## The one-line takeaway
Uniform node (UUID + type + properties + ordered child-ID array + parent) is what
makes BOTH a cheap insert menu and clean sharding by owner-entity possible. Keep
the single-entity path on the shards; push whole-corpus scans to a data lake.
