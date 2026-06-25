# Notion: the slash command and the block model

Date: 2026-06-25
Product: Notion
Feature: The "/" slash command (and the block data model it writes into)

---

## 1. The user

Meet Priya. She is a product manager at a 40-person startup called Acme Inc. It
is Monday morning and she is staring at a blank Notion page titled "Q3 Launch
Plan." She has a meeting in fifteen minutes and she wants the page to have a
heading, a short paragraph, a checklist of tasks, and a callout box for the one
risk she keeps worrying about.

She is not a designer. She does not want to open a formatting toolbar, hunt for
the right button, click it, and come back. She wants to type, and she wants the
document to take shape as fast as she can think. Her hands should never leave the
keyboard.

## 2. The real problem

Here is the honest version of the pain. Every other writing tool makes you choose
the tool first, then write. You move the mouse to a toolbar, pick "Heading 1,"
then go back to the page, then type. To make a checkbox you find another button.
To make a table you open a menu. The act of formatting keeps interrupting the act
of thinking.

For Priya this is death by a thousand small context switches. She has the whole
launch plan in her head right now, and every trip to a toolbar is a chance to lose
the thread. She does not want a word processor. She wants a single keyboard motion
that says "turn this line into a to-do" and then gets out of her way.

## 3. The feature in one sentence

Press "/" anywhere you are typing, and a menu appears that lets you insert any kind
of content block, or convert the current line into one, without ever touching the
mouse.

## 4. Jobs to be done

What is Priya really hiring the slash command to do?

- "When I have an idea, let me give it the right shape without leaving the keyboard."
- "Let me build a structured document (headings, lists, to-dos, tables, callouts)
  as fast as I can build a plain note."
- "Don't make me learn where 60 different buttons live. Let me type the name of
  what I want."
- "Let me change my mind. If this paragraph should have been a bullet, let me flip
  it in one motion."

The deeper job: collapse "format" and "write" into the same gesture.

## 5. How it works for the user

Priya types "/" on an empty line. A small menu pops up right where her cursor is.
It lists block types: Text, Heading 1, To-do list, Table, Callout, Quote, Code,
Image, and dozens more.

She keeps typing: "/to". The menu filters live to show "To-do list." She presses
Enter. The line becomes a checkbox. She types "Finalize pricing," hits Enter, gets
another checkbox automatically, types "Brief the sales team," and so on.

Then she selects a paragraph she already wrote, clicks the six-dot handle on its
left, picks "Turn into," and chooses "Callout." The plain paragraph becomes a
yellow callout box with an icon. Same content, new shape, no retyping.

The whole launch plan takes ninety seconds. Her hands never left the keyboard
except for that one "Turn into."

## 6. The actual flow, step by step

1. Cursor sits on an empty line in the "Q3 Launch Plan" page.
2. Priya presses "/". The editor opens the slash menu anchored at the caret.
3. She types "to". The menu does a live prefix filter over the block-type names
   and shows the matches ("To-do list" first).
4. She presses Enter. The editor does NOT just change some local CSS. It creates a
   new block, or converts the current empty block, to type `to_do`.
5. That change becomes one or more "operations." Operations are grouped into a
   "transaction."
6. The transaction is applied to her screen instantly (optimistic update). The
   checkbox appears before any server has heard about it.
7. The transaction is written to a local queue on her device so it survives a
   refresh or a dropped network.
8. The transaction is posted to the server, which validates it and commits it to
   Postgres.
9. The server tells every other client viewing that page (her teammate Sam, say)
   to pull the new block. Sam sees the checkbox appear a moment later.

The magic the user feels is step 6: it is already done before the network is
involved.

## 7. Under the hood, like the engineer

This is the heart of the report. The slash command looks like a text trick. It is
actually a thin UI over Notion's one big idea: everything is a block.

### 7.1 Everything is a block

In Notion, a page is not a document. A page is a block. A heading is a block. A
paragraph is a block. A single to-do checkbox is a block. A row in a database is a
block. A whole database is a block. As of 2024 there were more than 200 billion of
these block rows in production (source: Notion data lake post; ByteByteGo). At the
start of 2021 there were around 20 billion (source: "Herding elephants," Notion).

Every block, no matter its type, has the same shape. Confirmed from Notion's own
"data model behind Notion" post:

- `id`: a randomly generated UUID (version 4). Priya's new to-do might be
  `a1b2c3d4-...-9f0`.
- `type`: a string like `to_do`, `header`, `text`, `callout`, `bulleted_list`.
- `properties`: a hash map of the block's own content and attributes. For the
  to-do this holds the text "Finalize pricing" and a checked/unchecked flag.
- `content`: an ordered array of child block IDs. This is a list of pointers, not
  the children themselves. The page block's `content` array holds the IDs of the
  heading, the paragraph, the three to-dos, and the callout, in order.
- `parent`: the ID of the block this one lives inside.

Why this shape, in plain data-structure terms:

- The document is a tree. The page is the root. Its `content` array names its
  children in order; each child names its children. To render the page, the client
  walks this tree from the root, top to bottom. This is exactly a parent-pointer
  tree plus an explicit ordered child list. Notion calls the assembled result the
  "render tree."
- A block is a node. Its `properties` is the node's payload (a hash map, so adding
  a new attribute is cheap and needs no schema change). Its `content` is the node's
  edge list to its children.
- Ordering lives in the parent's `content` array, an ordered list of IDs. So
  moving Priya's "Brief the sales team" to-do above "Finalize pricing" is not a
  rewrite of either block. It is one reordering of two IDs inside the page block's
  `content` array. Cheap.

The slash command, seen this way, is tiny. "/to-do then Enter" is just: set this
block's `type` to `to_do`, or create a new block of `type` `to_do` and splice its
ID into the parent's `content` array at the cursor position. "Turn into Callout" is
even smaller: change one block's `type` from `text` to `callout` and leave its
`properties` text alone. Because rendering logic is decoupled from storage, the same
stored text now renders as a yellow box. That decoupling is the whole reason a
40-line slash menu can offer 60 block types without 60 different storage paths.

### 7.2 The write path: operations, transactions, optimistic apply

When Priya hits Enter on "/to-do," the client does not send "draw a checkbox." It
creates structured edits. Reconstructed from Notion's offline post and engineering
write-ups:

- An "operation" is the smallest unit of change. Example: set `block X`
  `properties.title` to "Finalize pricing." Another: set `block X` `type` to
  `to_do`. Another: insert `block X`'s id into `parent.content` at index 3.
- Operations are grouped into a "transaction" so a single user action commits all
  or nothing.
- The transaction is applied to the local in-memory store immediately. This is the
  optimistic update. Priya sees the checkbox with zero network round trip.
- The transaction is written to a local `TransactionQueue` (backed by IndexedDB in
  the browser, SQLite in the desktop and mobile apps). If her laptop sleeps or the
  wifi drops, the edit is safe and will be sent later.
- The client posts the transaction to the server endpoint that saves transactions.
  The server validates permissions and conflicts, then commits to Postgres.
- The server notifies every other client subscribed to that page over a WebSocket.
  Those clients call back to fetch the changed records and update their own local
  cache. That local cache is an LRU cache also called `RecordCache`, again backed
  by SQLite or IndexedDB.

So there are two stores doing opposite jobs, much like the Figma split covered on
2026-06-18. The local `RecordCache` is fast and disposable (rebuildable from the
server). The Postgres block table is the slow, permanent source of truth. The
slash command writes to the fast one first and lets the slow one catch up.

### 7.3 The read path: assembling a page from blocks

To open "Q3 Launch Plan," the client needs the page block, then its children, then
their children. Naively this is one query per block, which would be a latency
disaster for a page with hundreds of blocks. In practice the client checks
`RecordCache` first and only asks the server for the block IDs it does not already
have, fetching them in batches by id. The render tree is then assembled on the
client by following `content` arrays from the root down. Sorting and tree assembly
happen against records already pulled into memory, not as a giant join on the
phone.

### 7.4 The scale story at three tiers

The block model is elegant. The problem is that "elegant" and "200 billion rows"
do not automatically get along. Here is what broke and what Notion did, with real
numbers.

Tier 1, about 1,000 blocks (one workspace, early days). Everything lives in a
single Postgres instance. The block table is one table. Reads and writes are a
simple indexed lookup by `id` or by `parent`. Nothing is hard yet. This is Notion
in its first years: one Postgres monolith on Amazon RDS.

Tier 2, about 100,000 to billions of blocks (Notion through 2020). The single
Postgres box starts to hurt. By mid-2020 the block table had passed tens of
billions of rows. Confirmed pain from "Herding elephants": on-call engineers were
woken by database CPU spikes, the working set no longer fit comfortably in memory,
and the `VACUUM` process (Postgres reclaiming dead rows) started to stall, which
threatens transaction-ID wraparound. Even a routine schema migration on a table
that large became risky. The connection pooler, PgBouncer, was approaching its
limits. A single box cannot be made bigger forever.

What they did: shard the block table by `workspace_id`. The key choice, confirmed:
because every block belongs to exactly one workspace, and Priya almost always
queries inside one workspace at a time (the Acme Inc workspace), partitioning by
workspace ID means almost no query has to cross shards. No expensive cross-shard
joins on the hot path. They split into 32 physical Postgres instances, each holding
15 logical shards, for 480 logical shards in total. The number 480 was picked
because it divides cleanly by many numbers, which makes future rebalancing tidy.
Each logical shard is its own schema (`schema001.block`, `schema002.block`, and so
on), and the application itself computes which database and which schema a given
`workspace_id` routes to. Routing lives in the app, not in a fancy proxy.

The migration itself is the part most teams get wrong. Notion could not take Notion
offline to copy 20 billion rows. Confirmed approach: they first tried Postgres
logical replication to stream the monolith into the shards, but it could not keep
up with the block table's write volume during the initial bulk snapshot. So they
built an audit log table that recorded every write to the tables under migration,
then ran a catch-up process that replayed the audit log into the new shards until
the new fleet had caught up to live traffic. Final cutover cost users at worst a
few seconds.

Tier 3, 200 billion plus blocks (2023 to 2024). Three years later the 32-instance
fleet was hot again, running near 90% CPU and IOPS at peak. They did "The Great
Re-shard": tripled the fleet from 32 to 96 physical instances, moving from 15
logical shards per box down to 5 per box, while keeping the same 480 logical
shards. Because the logical shard count never changed, no data had to be
re-partitioned; whole logical shards just moved to new, emptier machines.
PgBouncer acted as the traffic controller so shards could be moved while traffic
kept flowing. Confirmed result: peak CPU and IOPS dropped from about 90% to about
20%, and users saw at worst about one second of a "Saving" spinner during
failover. Zero perceived downtime.

Separately, analytics and search were drowning the transactional shards, because
asking "scan all blocks of this type across the company" is the opposite of the
single-workspace lookup the shards are tuned for. Confirmed fix: Notion built a
data lake. Change-data-capture (Debezium) streams every block change off Postgres
into Kafka, Spark processes it, and it lands in S3 in Hudi table format for offline
analytics and to feed search and AI features. The expensive whole-corpus scans run
there, off the hot path, exactly like the offline-versus-online split seen across
these teardowns.

What breaks at the next tier and the general lesson: the thing that scales is the
partition key. Workspace ID worked because it matches how people actually read and
write (inside one workspace). The thing that does not scale on the same database is
the cross-cutting question (every block of this type, everywhere), and the survival
move is to copy that workload off to a system built for scans. Keep the
single-entity path on the shards; push the whole-corpus path to the lake.

Where this is fact versus inference: the block fields (`id`, `type`, `properties`,
`content`, `parent`), the UUID v4 ids, and the 200 billion / 20 billion counts are
from Notion's own posts. The 32-to-96 instances, 480 shards, workspace-ID
partition key, audit-log migration, PgBouncer, and the 90%-to-20% utilization are
all confirmed in "Herding elephants" and "The Great Re-shard." The exact operation
and transaction wire format (operation lists, `TransactionQueue`, `RecordCache`,
`syncRecordValues`) is reconstructed from Notion's offline post and reverse
engineering write-ups; the names are real but the precise schema is not fully
public, so treat the field-level detail there as well-grounded inference.

## 8. The retention and habit mechanic

The slash command's habit is muscle memory. Once "/" lives in your fingers, you
stop reaching for any other writing tool. Priya does not "use Notion for docs." She
just hits "/" the way she hits Enter. That reflex is switching-cost: a competitor
now has to beat not a feature but a habit her hands already have.

The loop that brings her back: every page she builds becomes structured data she
can later view as a table, a board, or a calendar, because each line was a typed
block, not loose text. So the page she made in ninety seconds today becomes the
launch tracker she opens every morning this quarter. The slash command is the cheap
front door; the block model is what keeps her returning, because her content is now
queryable, not just readable.

Which metric it moves: primarily activation, then retention. A brand-new user who
discovers "/" in their first session crosses the line from "this is a blank box I
do not understand" to "I just built something structured." That first structured
page is the classic activation moment. The block model then drives retention,
because the more blocks you create, the more your own data locks you into coming
back. Real observed pattern: Notion grew to over 100 million users and 200 billion
blocks largely on word of mouth, and the slash command is the single gesture every
one of those users learns first.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code. Notion's deepest lesson is not the slash menu. It is that one
uniform node type, addressed by stable ID, with payload and children kept separate,
is what let a tiny insert-menu scale to 200 billion nodes without 200 billion
special cases.

Make every element in the Rare.lab graph a single uniform node record: a stable
UUID, a `type` string, a `properties` hash map for that node's parameters, and a
`content` or `inputs` array of child or upstream node IDs (pointers, never embedded
copies). If you do this, three hard things get easy. First, your own slash command
(a quick-insert palette: type "/blur" and drop a blur node at the cursor) becomes
trivial, because inserting a node is just minting a UUID and splicing one ID into a
parent's array, not a bespoke code path per node type. Second, "convert this node
into that node," undo, and reordering all become small edits to `type` or to an
ordered ID array, not graph rewrites. Third, and most important for your bias
toward scale: when one project graph outgrows what fits in memory, you can shard by
project ID exactly the way Notion shards by workspace ID, because every node belongs
to exactly one project and editors almost always work inside one project at a time.
That single key choice is what lets you grow the back end from one box to ninety-six
without re-partitioning a thing.

And copy the two-store split for the runtime. Keep a fast, disposable local cache
of node records for the live editor (optimistic edits applied before the server
hears about them, like Notion's `RecordCache`), and treat the persisted store as the
slow source of truth that catches up. The compile-to-code step is your equivalent of
the "render tree": walk the node graph from the output node down, following the ID
arrays, and emit shader code, the same way Notion walks `content` arrays to paint a
page. Decouple how a node renders or compiles from how it is stored, and you can add
new node types forever without touching the storage or the sync layer.

---

## Sources

- Notion engineering, "Exploring Notion's data model: a block-based architecture":
  https://www.notion.com/blog/data-model-behind-notion
- Notion engineering, "Herding elephants: lessons learned from sharding Postgres at
  Notion" (Oct 2021): https://www.notion.com/blog/sharding-postgres-at-notion
- Notion engineering, "The Great Re-shard: adding Postgres capacity (again) with
  zero downtime" (2023): https://www.notion.com/blog/the-great-re-shard
- Notion engineering, "Building and scaling Notion's data lake":
  https://www.notion.com/blog/building-and-scaling-notions-data-lake
- Notion engineering, "How we made Notion available offline":
  https://www.notion.com/blog/how-we-made-notion-available-offline
- ByteByteGo, "Storing 200 billion entities: Notion's data lake project":
  https://blog.bytebytego.com/p/storing-200-billion-entities-notions
- Quastor, "How Notion sharded their Postgres database":
  https://blog.quastor.org/p/notion-sharded-postgres-database-8af4
- Notion help, "Using slash commands":
  https://www.notion.com/help/guides/using-slash-commands
- Hacker News discussion of "The Great Re-shard":
  https://news.ycombinator.com/item?id=43157296
