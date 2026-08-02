# Notion Databases and the Views That Sit On Top (table, board, calendar)

Date: 2026-08-02
Product: Notion
Feature: Databases and database views (turning one collection of pages into a table, a board, a calendar, filtered and sorted), and the block data model plus Postgres sharding that hold it all up.

---

## 1. The user

Meet Ananya. She runs content for a small SaaS company. It is Monday morning, coffee in one hand, and she opens Notion to the page she lives in: "Content Calendar." She has 240 blog posts and social drafts in there, going back a year. Right now she needs three different things from the exact same pile of posts.

First she wants a simple grid: every post, its status, its author, its publish date, sorted by date. Then her designer pings her: "just show me what is in Draft, as cards I can drag." Then her manager wants "everything shipping this week, on a calendar."

Same 240 posts. Three completely different shapes. Ananya does not want three copies of the data. She wants one source of truth and three lenses onto it. That is a Notion database with three views.

## 2. The real problem

Here is the pain, said plainly. A spreadsheet gives you a grid and nothing else. If you want a Kanban board you copy the data into Trello. If you want a calendar you copy it into Google Calendar. Now you have three copies and they drift. Ananya marks a post "Published" in the sheet, forgets the Trello card, and her designer keeps working on a post that already shipped.

The real job is not "make a table." The real job is: let one set of records be looked at many ways at once, and keep them all in sync, without me maintaining three tools. And do it fast enough that filtering 240 posts, or 24,000, feels instant.

## 3. The feature in one sentence

A Notion database is a single collection of pages that all share a set of properties (columns), and a view is a saved lens over that collection (table, board, calendar, list, gallery) with its own filters, sorts, and grouping, so the same data appears in as many shapes as you need and every shape updates the moment the underlying page changes.

## 4. Jobs to be done

- "Give me one place for all my posts and let me never keep two copies in sync again."
- "Let me look at the same posts as a grid, a board, and a calendar, and switch in one tap."
- "Show me only what matters right now (Status is Draft, Date is this week) without deleting the rest."
- "Let me drag a card from Draft to Published and have the record actually change, not just the card."
- "Let me add a column (a property) once and have every view understand it."

## 5. How it works for the user

Ananya types `/database` and inserts a table. Each row is a post. The columns are properties: Title (text), Status (a select with Draft, In Review, Published), Author (a person), Publish Date (a date), Rating (a number). Every row is not just a row. It is a full Notion page. She clicks a row and it opens as a page she can write a whole article inside.

Now the magic. At the top of the database there is a row of view tabs. She clicks "+ Add view," picks Board, and groups by Status. Instantly the same 240 posts rearrange into columns: a Draft column, an In Review column, a Published column. She drags "Our Series B announcement" from Draft to Published. The card moves, and because moving the card sets the Status property, the table view she made earlier now shows that same post as Published too. She never touched the table.

She adds a third view, Calendar, keyed on Publish Date. The same posts land on the days they ship. She adds a filter to the board: Status is not Published. Now the board hides everything already shipped. The table still shows everything. Each view remembers its own filter and sort. The data is one. The lenses are many.

## 6. The actual flow, step by step

1. Ananya types `/table` (or `/board`, `/calendar`). Notion inserts a database block on the page.
2. She adds properties. Each property she creates (Status, Author, Rating) is written into the database's schema once.
3. She adds rows. Each row becomes a page whose parent is this database. Its cell values are stored as that page's properties.
4. She clicks "+ Add view," picks Board, sets "Group by: Status." Notion saves a new view record: type = board, group_by = Status, plus any filters and sorts.
5. She drags a card from Draft to Published. The client sends one small edit: set this page's Status property to "Published."
6. Every open view that reads this database re-reads the changed page and re-renders. The table shows Published. The calendar keeps it on its date. The board shows it in the Published column.
7. She types a filter on the board: "Status is not Published." The board recomputes which pages pass and hides the rest. The filter is saved on the board view only.

The important thing to notice: the drag did not "move" anything. It changed one property on one page. The views are just different queries over the same pages, so they all reflect the change at once.

## 7. Under the hood, like the engineer

This is the heart of it. Notion has said a lot about this publicly, so much of what follows is confirmed fact from their own engineering writing. Where I reason past what they published, I label it inference.

### Everything is a block (confirmed)

Notion's core idea, in their own words, is that "the block is the fundamental unit." A paragraph is a block. A heading is a block. A to-do is a block. A whole page is a block. And a database row is a block too.

A block, as Notion describes it, is a record with a handful of attributes:

- An **id** (a UUID, for example `2a4f1e90-1c3b-4a...`).
- A **type** (`text`, `to_do`, `page`, `collection_view_page`, and so on).
- **Properties**: a map holding the block's own data, for example the actual words in a paragraph, or the cell values of a database row.
- **Content**: an ordered array of child block ids. This is the key structural choice. A block does not embed its children. It holds a list of their ids, in order. Ananya's toggle that contains three bullet points stores those three bullets as an array `[id1, id2, id3]`.
- A **parent** id, the block this one lives inside.

So the whole Notion document is a **tree**, expressed as parent pointers plus ordered child-id arrays. Why a tree of id references and not nested objects? Because you must be able to move, reorder, and re-parent any block cheaply, and two users must edit different branches at the same time. If children were embedded, moving one bullet would rewrite its whole parent. With an ordered array of ids, moving a bullet is editing one small list. This is a classic **adjacency list** representation of a tree, chosen so edits stay local and small.

### A database is a "collection," a row is a page, a column is a schema entry (confirmed)

Notion databases are internally called **collections**. The pieces, as documented by Notion and mirrored in the well-known unofficial `notion-py` client, are:

- A **collection** record. It holds the **schema**: the set of properties (columns). The schema is a map. Here is the detail that tells you a real engineer designed this: the keys in that map are **4 randomly generated characters**, not the human column name. So "Status" might be stored under key `J=x2`, "Rating" under `k9Lp`. The human name lives inside the value. Why random short keys instead of the display name? Because a user renames "Status" to "Stage" ten times a day, and if the key were the name, every rename would have to rewrite every row's property map. With a stable random key, renaming a column touches one entry in the schema and zero rows. The special key `title` is reserved for the row's title.

- Each **row is a page** (a block) whose parent is the collection. Its cell values live in that page's **properties** map, keyed by those same 4-character schema keys. So Ananya's "Series B" post is a block with properties like `{ "title": [["Our Series B announcement"]], "J=x2": [["Published"]], "k9Lp": [[5]] }`.

- A **collection_view** record. This is the view. It stores the view **type** (table, board, calendar, list, gallery) and a query object holding the **filters**, the **sorts**, the **group_by**, and any aggregations. Ananya's board is one collection_view with `type: board`, `group_by: Status`, `filters: [Status is not Published]`.

Read that structure again and the feature falls out of it for free. The rows exist once, as pages under the collection. A view does not own any rows. A view is a saved query. That is exactly why one edit to a page shows up in the table, the board, and the calendar at the same time. There is only one copy of the truth, and three saved queries reading it.

### Matching then ranking: how a view is actually computed (fact plus grounded inference)

Filtering and sorting a database is a two-half problem, the same shape as search.

**Half one, matching (which pages belong in this view).** The server has to find the pages whose parent is this collection and whose properties pass the view's filters. Confirmed: the rows are pages stored in Postgres. So this is a SQL query, run **on the server**, not in Ananya's browser. Conceptually:

```
select * from block
where parent_id = <collection id>
  and space_id = <workspace id>
  and alive = true
  and properties ->> 'J=x2' = 'Published'   -- Status filter
order by properties ->> 'date-key'           -- sort
```

The filter and the sort run in the database engine, close to the data, and only the matching, already-ordered pages come back over the network. The phone never downloads 240 posts to sort them locally. It receives the answer. For Ananya at 240 rows this is trivial. The design matters at 24,000 rows and beyond, where sending everything to the client and sorting in JavaScript would be slow and would drain the battery.

**Half two, rendering (shape the matched pages for this view).** Once the matched, sorted pages are in hand, the board groups them into columns by the group_by property, the calendar drops them onto dates, the table lays them in a grid. This shaping is cheap and can happen client-side because it runs over the small already-filtered set, not the whole collection.

Inference, clearly labeled: the exact SQL and index strategy Notion uses for property filters is not fully public. Filtering on a value buried inside a JSON properties map is the known hard part, because a plain b-tree index does not cover arbitrary keys inside a JSON blob. The standard way this class of problem is solved in Postgres is a **GIN index** on the JSON column, or promoting hot filter/sort properties into real indexed columns. Notion has not published which they do, so treat the specific mechanism as the well-grounded general answer, not a confirmed Notion internal.

### The scale story: 1,000 blocks, then billions

Now the part Notion has written about in remarkable detail. This is confirmed engineering, with numbers.

**Tier one, thousands of blocks.** Early Notion ran a single Postgres database. Ananya's 240 posts, and everyone else's data, sat in one table called `block`. At this size one Postgres box is not just enough, it is the right answer. Sharding here would be pure over-engineering. A single `select` with a `where parent_id = ...` returns in milliseconds.

**Tier two, the single database starts to swell.** By the start of 2021 Notion had crossed **20 billion block rows** in that one Postgres instance, and response times were suffering. Two specific things broke, and they are worth naming because they are not the obvious "we ran out of disk."

- **VACUUM stalled.** Postgres keeps dead row versions around until a background process called VACUUM cleans them up. As the block table ballooned, VACUUM could not keep up, tables bloated, and performance sagged.
- **Transaction ID wraparound loomed.** This is the scary one. Postgres tracks transactions with a counter that tops out near **4 billion**. VACUUM is also what stops that counter from wrapping around. If VACUUM cannot keep up on a huge table, you march toward a **TXID wraparound**, which forces the database into a protective read-only shutdown. Adding disk does not save you. You have to reduce how much data any one Postgres instance is responsible for.

So the fix was not a bigger machine. It was **horizontal sharding**: split the one database into many, each holding a slice of the data.

**The sharding design (confirmed).** Notion sharded by **workspace**. Every piece of content, every block, comment, and collection, belongs to a workspace (they call it a space), and Ananya's whole company lives in one workspace. So the partition key is the workspace id. All of a workspace's data lands together on one shard. That is deliberate: almost every query Notion runs is scoped to a single workspace ("give me this collection's rows in this space"), so keeping a workspace whole on one shard means those queries never have to fan out across machines.

The numbers Notion published:

- **480 logical shards**, spread across **32 physical Postgres databases**, so 15 logical shards per machine at the start.
- Why the odd number **480**? Because it factors cleanly. 480 is divisible by 2, 3, 4, 5, 6, 8, 10, 12, 15, 16, and more. That means they can redistribute those 480 logical shards across many possible counts of physical machines (32, 48, 96) while keeping every machine holding an equal number of shards. They picked the logical shard count once, up front, to buy themselves years of physical rebalancing without ever re-hashing the data. This is the single smartest line in the whole story: separate the logical partition (fixed at 480 forever) from the physical machine count (grows over time).

**The migration without downtime (confirmed).** You cannot take Notion offline to copy 20 billion rows. So they **double-wrote**: every new write went to both the old single database and the new shards at the same time, coordinated through an **audit log** so the two could not silently drift, while a backfill script copied the historical data onto the shards. Then they verified the shards matched, and flipped reads over. They ran the backfill on a beefy **m5.24xlarge** box, 96 CPUs.

**Tier three, the re-shard (confirmed).** By 2023 those original 32 machines were themselves at their limit. Here is the payoff of choosing 480. Because there were 480 logical shards, moving from **32 to 96 physical databases** was a redistribution, not a re-hash: 96 divides 480, so each new machine simply took 5 logical shards. They used Postgres **logical replication** (publications on the old databases, subscriptions on the new) to stream the data across, and put **PgBouncer** in front as the connection layer so the app could be pointed at the new machines gradually and rolled back instantly if something looked wrong. Concrete details they shared: four new PgBouncer clusters, each fronting 24 databases, and a cap around 200 connections per Postgres instance so the pooler does not exhaust the database. The logical-replication approach cut the data-sync time from about **3 days to roughly 12 hours**.

The data has since grown past **200 billion blocks**, hundreds of terabytes even compressed. Same block model Ananya's 240 posts use. Just sharded 480 ways.

**What this means for Ananya's board filter.** When she filters "Status is not Published," the query is routed by her workspace id to exactly one shard, which holds all of her company's data together. It never touches the other 479 logical shards or the other 95 machines. Her filter over 240 posts, and a huge enterprise's filter over 240,000, both land as a single indexed, single-shard query. The catalog is billions of blocks. Her query reads one workspace's slice. That routing, workspace id to shard, is what keeps a per-user action cheap while the whole system holds hundreds of billions of rows.

## 8. The retention and habit mechanic

The habit here is not a dopamine ping. It is **lock-in through accumulated structure**, which is the stickiest retention there is.

The loop: Ananya builds one database. Then she adds a board view for her designer, a calendar for her manager, a filtered "my drafts" view for herself. Each view is five minutes of setup that saves her an hour a week forever. Every view she adds makes the database more load-bearing for her whole team, and makes leaving Notion mean rebuilding all of it somewhere else. The multi-view database is not a feature she uses once. It becomes the operating system for her team's work.

Which metric does it move? Primarily **retention and expansion**, not a daily-active vanity spike. A single note is easy to abandon. A database with six views, three teams reading it, and a year of history is not. Notion's own growth from a handful of blocks per user to **200 billion blocks** is the aggregate of millions of Ananyas turning scratch pages into structured databases they cannot walk away from. Real observed example of the mechanic in the wild: the enormous ecosystem of Notion "templates" (content calendars, CRMs, habit trackers), each one a pre-built database with pre-built views, exists precisely because the multi-view database is the thing people build their lives inside and then want to clone.

## 9. The lesson for Rare.lab

Separate the **logical partition from the physical machine**, and fix the logical count once, on day one, before you have any scale problem.

Rare.lab will accumulate a huge, growing pile of durable objects: shader graphs, compiled artifacts, per-user asset libraries, render caches. The instinct is to start with one database and shard later "when it hurts." Notion shows the trap. When it hurt, they were at 20 billion rows staring down a TXID-wraparound read-only shutdown that no bigger machine could fix. What saved them was a decision made earlier: pick a large, highly divisible **fixed** logical shard count (they chose 480) and route by a natural tenant key (workspace) so every real query lives on one shard. Then physical machines can grow 32 to 96 to more as a pure redistribution, no re-hashing, no rebuild, streamed over with logical replication and swapped behind a pooler with instant rollback.

Concretely for Rare.lab: choose the partition key now, and make it the thing every hot query already filters by. A shader graph belongs to a project (or a workspace); a compiled variant belongs to a graph. Partition durable storage by **project id** into a fixed set of logical shards (pick a number like 480 that divides many ways), so "load this project's graphs and cached compiles" is always a single-shard, single-indexed lookup no matter how many projects exist globally. Keep the logical shard count constant forever and let the physical database count float. And store your objects the Notion way: a stable machine id as the key, the human-facing name as mutable data inside, and children as ordered arrays of ids, so renaming a node or reordering a graph is one tiny write, not a rewrite of everything downstream. Do the boring partition math before the first big customer, because retrofitting it at 20 billion rows is the hardest migration there is, and Notion had to invent an audit-logged double-write and a 12-hour logical-replication cutover to survive doing it live.

---

## Sources

- Notion Engineering, "The data model behind Notion's flexibility" (blocks, content array, parent, collection schema, 4-character property keys): https://www.notion.com/blog/data-model-behind-notion
- Notion Engineering, "Herding elephants: lessons learned from sharding Postgres at Notion" (480 logical shards, 32 physical databases, workspace partition key, VACUUM and TXID wraparound, double-write with audit log, m5.24xlarge backfill): https://www.notion.com/blog/sharding-postgres-at-notion
- Notion Engineering, "The Great Re-shard: adding Postgres capacity (again) with zero downtime" (32 to 96 databases, logical replication, PgBouncer clusters, 3 days to 12 hours): https://www.notion.com/blog/the-great-re-shard
- Notion Engineering, "Building and scaling Notion's data lake" (20 billion blocks in 2021 growing past 200 billion, hundreds of terabytes): https://www.notion.com/blog/building-and-scaling-notions-data-lake
- notion-py (unofficial client) collection/collection_view API, documenting Collection, CollectionView, schema and RecordStore concepts: https://notion-py.readthedocs.io/en/latest/api/notion.block.collection.html
- Quastor deep dive summarizing the sharding post (480 shards, VACUUM, double-write): https://blog.quastor.org/p/notion-sharded-postgres-database-8af4
- ByteByteGo, "Storing 200 Billion Entities: Notion's Data Lake Project" (scale numbers): https://blog.bytebytego.com/p/storing-200-billion-entities-notions
- pganalyze, "How Figma and Notion scaled Postgres" (horizontal sharding between servers): https://pganalyze.com/blog/5mins-postgres-partitioning-tables-between-servers-horizontal-sharding
