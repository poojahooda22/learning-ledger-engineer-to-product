# References: Notion databases and views (data model + Postgres sharding)

Saved 2026-08-02 for the Notion databases/views teardown.

## Primary (Notion Engineering blog)

- The data model behind Notion's flexibility: https://www.notion.com/blog/data-model-behind-notion
  - The block is the fundamental unit. Attributes: id, type, properties (map), content (ordered array of child block ids), parent id.
  - Databases are "collections." A collection holds the schema (properties). Property keys are 4 randomly generated characters; `title` is reserved. Renaming a column edits the schema, not every row.
  - A database row is a page (a block) whose parent is the collection; cell values live in that page's properties map under the schema keys.

- Herding elephants: lessons learned from sharding Postgres at Notion: https://www.notion.com/blog/sharding-postgres-at-notion
  - ~20 billion block rows by early 2021 in a single Postgres instance.
  - Failure modes that forced sharding: VACUUM stalling on a bloated block table, and looming TXID wraparound (Postgres transaction counter tops out near 4 billion; wraparound forces read-only).
  - Partition key = workspace (space) id, so a workspace's data stays whole on one shard and queries do not fan out.
  - 480 logical shards across 32 physical databases (15 logical shards each). 480 chosen for divisibility.
  - Zero-downtime migration via double-write coordinated through an audit log, plus a backfill on an m5.24xlarge (96 CPUs).

- The Great Re-shard: adding Postgres capacity (again) with zero downtime: https://www.notion.com/blog/the-great-re-shard
  - 2023: 32 to 96 physical databases; 96 divides 480 so each new machine takes 5 logical shards (redistribution, not re-hash).
  - Postgres logical replication (publications on old, subscriptions on new). PgBouncer as the connection/routing layer, gradual cutover with instant rollback.
  - 4 new PgBouncer clusters, 24 databases each; ~200 connections cap per Postgres instance. Sync time cut from ~3 days to ~12 hours.

- Building and scaling Notion's data lake: https://www.notion.com/blog/building-and-scaling-notions-data-lake
  - Growth from ~20 billion blocks (2021) to 200+ billion (2024), hundreds of terabytes compressed.

## Secondary / corroborating

- notion-py collection API (Collection, CollectionView, schema, RecordStore): https://notion-py.readthedocs.io/en/latest/api/notion.block.collection.html
- Quastor, "How Notion Sharded Their Postgres Database": https://blog.quastor.org/p/notion-sharded-postgres-database-8af4
- ByteByteGo, "Storing 200 Billion Entities: Notion's Data Lake Project": https://blog.bytebytego.com/p/storing-200-billion-entities-notions
- pganalyze, "How Figma and Notion scaled Postgres": https://pganalyze.com/blog/5mins-postgres-partitioning-tables-between-servers-horizontal-sharding

## Notes / open questions (inference, not confirmed by Notion)

- Exact index strategy for filtering/sorting on values inside the JSON properties map is not public. Standard solutions for this class: GIN index on the JSON column, or promoting hot filter/sort properties into real indexed columns. Treat as grounded inference.
- Filters/sorts/group_by are computed server-side in the database query (rows are Postgres-backed), not shipped to the client to sort. View rendering (grouping into board columns, dropping onto calendar days) runs over the already-filtered small set.
