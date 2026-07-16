# References: Notion offline mode + sync/conflict engine (CRDT)

Keeper links for the 2026-07-16 teardown.

## Primary (Notion engineering and product)
- How we made Notion available offline
  https://www.notion.com/blog/how-we-made-notion-available-offline
  SQLite local cache turned into a durable tracked store. Each offline page keeps a
  lastDownloadedTimestamp; on reconnect compare vs server lastUpdatedTime, fetch only
  newer pages. offline_action is the source of truth for "why is this page offline,"
  reconciled incrementally so the offline forest stays eventually consistent without
  full rebuilds. Pages marked "Available offline" are dynamically migrated to a NEW
  CRDT data model for conflict resolution. Text edits auto-merge (edit para A offline,
  colleague edits para B online, both survive). Non-text properties (select, date,
  relation) do NOT merge; one version survives (last-writer-wins).
- Exploring Notion's data model: a block-based architecture
  https://www.notion.com/blog/data-model-behind-notion
  Block = UUID + type + properties (hash map) + content (ordered child ID array) +
  parent. Operations grouped into transactions, optimistic local apply to RecordCache
  (LRU on SQLite/IndexedDB), TransactionQueue, POST /saveTransactions, server
  validates+commits, MessageStore WebSocket notifies subscribers -> syncRecordValues.
- Herding elephants: sharding Postgres at Notion (Oct 2021)
  https://www.notion.com/blog/sharding-postgres-at-notion
  Shard by workspace_id. 480 logical shards.
- The Great Re-shard (2023)
  https://www.notion.com/blog/the-great-re-shard
  32 -> 96 physical boxes, same 480 logical shards, peak CPU/IOPS ~90% -> ~20%.
- Notion release notes: v2.53 Offline mode, 19 Aug 2025
  https://www.notion.com/releases/2025-08-19

## Primary (the replicated-tree move problem: the hard half)
- Kleppmann, Mulligan, Gomes, Beresford, "A highly-available move operation for
  replicated trees," IEEE TPDS 2021.
  https://martin.kleppmann.com/2021/10/07/crdt-tree-move-operation.html
  CRDT for arbitrary concurrent tree moves. Guarantees: no cycles introduced, all
  replicas converge, no synchronous coordination. Formally proved in Isabelle/HOL.
  Documents real concurrent-move bugs in Google Drive and Dropbox (duplicated/lost
  subtrees). This is the exact invariant Notion's block tree must satisfy under
  offline merge.
- Reference implementation: https://github.com/trvedata/move-op
- Kleppmann, "CRDTs: The Hard Parts" (talk)
  https://martin.kleppmann.com/2020/07/06/crdt-hard-parts-hydra.html

## Secondary and coverage
- TechCrunch, "Finally, Notion now works without an internet connection" (20 Aug 2025)
  https://techcrunch.com/2025/08/20/finally-notion-now-works-without-an-internet-connection/
  Framed as closing a long-standing gap vs Apple Notes and Obsidian.
- XDA Developers, "Notion's offline mode is finally here, but it comes with a strange
  limitation." https://www.xda-developers.com/notion-offline-mode-launched/
- Educative, "Notion System Design Explained." https://www.educative.io/blog/notion-system-design
- ByteByteGo, "Storing 200 billion entities: Notion's data lake project."
  https://blog.bytebytego.com/p/storing-200-billion-entities-notions
  200B+ blocks by 2024.

## Fact vs inference
- CONFIRMED: block model, transaction pipeline, SQLite cache, 480 shards, "Available
  offline" toggle, lastDownloadedTimestamp delta cursor, offline_action source of
  truth, migration of offline pages to a CRDT model, text auto-merge, non-text-property
  LWW, v2.53 / 19 Aug 2025 launch.
- INFERENCE (labeled in report): the specific CRDT algorithms (sequence CRDT such as
  RGA/Yjs for text; Kleppmann-style move op for the block tree; fractional indexing
  for sibling order; tombstone garbage collection). Notion confirmed "a new CRDT data
  model" and the merge behavior, not the internal algorithm choices.

## The one-liner
Online Notion never needed real conflict resolution because the always-reachable
server was the single referee (optimistic apply + LWW is fine when divergence is
milliseconds). Offline removes the referee, so block-level LWW would silently eat
hours of work; the fix is to migrate only offline-flagged pages to a CRDT model,
split each page into three merge-semantics halves (text = sequence CRDT auto-merge,
tree = cycle-safe move-op CRDT, scalars = LWW), and resync by delta cursor. Fence the
expensive machinery to the set that needs it; offline-think/online-lookup again.
