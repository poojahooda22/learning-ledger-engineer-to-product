# Day 15 — How does a collaborative editor merge two hours of offline edits from three people without losing a single keystroke?

**Date:** 2026-06-26
**Difficulty:** Advanced
**Stack relevance:** CRDTs, Y.js, Automerge, Cloudflare Durable Objects, local-first architecture, Supabase real-time, node-based editors

---

## 1. Named Company + The Breaking Number

**Company: Notion**

Notion's editor is a tree of blocks. A page is a block. A paragraph is a block. A heading, a table, a toggle, an image — all blocks. A large team wiki can have 50,000 blocks. Every edit (typing a character, moving a block, nesting something inside something else) is an operation on that tree.

Notion's original real-time layer used **server-serialized Operational Transformation (OT)**: every edit flows to Notion's servers, the server picks an order, transforms conflicting ops, and broadcasts. This works when everyone is online. It collapses the moment a laptop goes offline.

**The breaking number: 9.7 million.**

A team of 5 engineers runs sprint planning for 2 hours. One engineer's laptop loses Wi-Fi 30 minutes in. During those 90 offline minutes:

- The offline engineer makes 1,800 block operations (typing notes, reordering tasks, updating statuses — roughly 1 op every 3 seconds).
- The 4 online teammates make 5,400 combined operations.

When the offline engineer reconnects, their OT client must **transform each of their 1,800 ops against each of the 5,400 server ops** to compute what their local state "should have been" had they been online. That is 1,800 x 5,400 = **9,720,000 transformation pairs**.

Each transform call is a function call: inspect op A, inspect op B, mutate op A's positions to account for op B's effect. On a laptop CPU at 10 microseconds per call: 97 seconds of single-threaded CPU. On a 2021 iPhone: 10x slower, 16 minutes. The app freezes. Users force-quit. Their 90 minutes of offline work is stuck behind a spinner.

This is not a theoretical problem. Confluence, Google Wave (shut down 2010), and early Notion all hit versions of this wall. The fix is CRDTs.

---

## 2. Why the Naive Design Dies

### Operational Transformation: what it is and where it breaks

OT was invented by Ellis and Gibbs in 1989 for the GROVE editor. Google Docs uses a variant called Jupiter (1995). The idea is seductively simple:

Alice types "X" at position 5. Bob types "Y" at position 5 simultaneously. They send their ops to the server. The server picks Alice first. Now it must send Bob's op to Alice, but "insert Y at position 5" is wrong — Alice already put X there, shifting everything right. The server **transforms** Bob's op to "insert Y at position 6" before broadcasting. Alice and Bob both end up with "XY". Correct.

**Collapse 1: The transform matrix explodes.**

OT needs a transform function for every pair of operation types: (insert, insert), (insert, delete), (delete, delete), (move, insert), (move, delete), (move, move), and so on. With just 3 op types you need 9 transform functions. With 10 op types you need 100. Get one wrong and you have **silent document corruption** — text that looks right locally is wrong for the other user. Google Wave's OT engine had 21 documented correctness bugs. The team spent years patching them. Some were never fixed. Wave shut down.

**Collapse 2: OT requires a single total ordering authority.**

The Jupiter protocol only works with one central server that serializes all operations. Every write must pass through that server before any client can apply it. This means:

- No peer-to-peer sync (two laptops on the same LAN still need a server round-trip).
- No true offline editing (you can optimistically apply ops locally, but you cannot merge with others until you reconnect and the server runs transforms).
- Server downtime = collaboration downtime. Notion's server goes down, everyone's editing stops.

**Collapse 3: Tree operations have no clean transform.**

OT was designed for linear text (a sequence of characters). Notion's document is a tree of blocks. "Move block 42 from parent A to parent B" while "Bob deletes parent B" is a concurrent tree operation. What is the correct transform? There is no consensus in the literature. Different OT papers propose different rules and each has counterexamples. This is why collaborative Google Slides (a tree of slides with nested text) lagged years behind Google Docs (linear text).

**Collapse 4: The reconnect complexity is O(n * m).**

Covered above. The offline client's queue of n ops must be transformed against the server's m ops. Complexity is quadratic. At moderate n and m (the sprint planning scenario), it freezes the client.

---

## 3. The Architecture — Top to Bottom

CRDTs (Conflict-free Replicated Data Types) solve all four collapses by changing the fundamental model: **instead of transforming operations, give every operation a permanent unique identity and embed the merge logic into the data structure itself**.

```
[Alice — online]    [Bob — online]    [Charlie — 90min offline]
     |                    |                      |
 [Local Y.js doc]    [Local Y.js doc]      [Local Y.js doc]
  Full replica of     Full replica of       Full replica of
  document CRDT.      document CRDT.        document CRDT.
  Every edit applied  Every edit applied    Changes buffered
  locally first.      locally first.        locally — no network
  "Optimistic UI."    "Optimistic UI."      needed to keep working.
     |                    |
     | WebSocket (wss://) | WebSocket
     v                    v
+----------------------------------------------------------+
|              SYNC SERVER                                 |
|  (Cloudflare Durable Object, or Hocuspocus, or custom)  |
|                                                          |
|  Job: Hold one canonical CRDT state vector per doc.     |
|  Route updates between connected clients.               |
|  Persist durable ops to storage for offline replay.     |
|                                                          |
|  NOT a transform engine. It does not decide order.      |
|  It is a mailbox, not a judge.                          |
|                                                          |
|  Analogy: A postal sorting office. It receives          |
|  packages (ops) with addresses (doc IDs), stamps        |
|  them with arrival time, and delivers them to all       |
|  subscribers. It does not rewrite the packages.         |
+----------------------------------------------------------+
     |                    |
     | Persist op log     | Serve ops to new clients
     v                    v
+-------------------+  +-------------------------+
|  DURABLE OP LOG   |  |  DOCUMENT SNAPSHOT      |
|  (append-only)    |  |  (checkpoint every N ops)|
|                   |  |                          |
|  Each row:        |  |  Full CRDT state as of   |
|  {docID, clientID,|  |  op #N. New clients load |
|   clock, opType,  |  |  snapshot then replay    |
|   content,        |  |  ops > N. Bounded replay.|
|   leftNeighborID, |  |                          |
|   rightNeighborID}|  |  Analogy: A save file.   |
|                   |  |  You do not replay the   |
|  Analogy: A ship's|  |  whole game from the     |
|  log. Every event |  |  beginning; you load      |
|  recorded, never  |  |  the save and replay      |
|  overwritten.     |  |  the last 10 minutes.    |
+-------------------+  +-------------------------+

--- CHARLIE RECONNECTS AFTER 90 MINUTES ---

1. Charlie's Y.js sends its STATE VECTOR to the sync server.
   State vector = {aliceClientID: 1800, charlieClientID: 900, bobClientID: 0}
   (Charlie has seen ops 1-1800 from Alice, 1-900 of his own, nothing from Bob)

2. Sync server compares Charlie's state vector against its own.
   Server state vector = {alice: 5400, charlie: 900, bob: 3600}
   Missing from Charlie: alice ops 1801-5400, bob ops 1-3600.
   
3. Server sends Charlie ONLY the ops he is missing. Not everything. Just the diff.
   Total ops to send: 3600 + 3600 = 7200 ops.

4. Charlie's Y.js applies each incoming op in O(1):
   Each op says "insert X between neighbor A and neighbor B."
   Charlie's local CRDT resolves the position from the neighbor IDs, not raw positions.
   No transform functions called. 7200 ops applied. Done in < 2 seconds.

5. Charlie's 1800 local ops are sent to the server.
   Alice and Bob's Y.js applies them in O(1) each.
   All three replicas converge to identical state.
```

**The key difference from OT:** In step 4, Charlie does not transform his 1800 ops against the server's 7200 ops. He simply applies the incoming ops directly to his CRDT state. The CRDT's structure makes the merge commutative: "apply A then B" and "apply B then A" reach the same final state. No central authority needed to define order.

---

## 4. The Transferable Mechanisms

### Mechanism 1: The Semilattice — why any merge order gives the same answer

A **semilattice** is a mathematical structure with one operation: merge. The merge operation has three properties:

- **Commutative**: merge(A, B) = merge(B, A). Order does not matter.
- **Associative**: merge(merge(A, B), C) = merge(A, merge(B, C)). Grouping does not matter.
- **Idempotent**: merge(A, A) = A. Applying the same op twice is safe.

Any data structure with these three properties can be replicated across nodes with zero coordination. Every CRDT is a semilattice. The magic is: you can receive ops in any order, you can receive duplicates, and you always converge.

**The most common CRDTs (and their semilattice rule):**

| CRDT | State | Merge rule | Use case |
|---|---|---|---|
| G-Counter (Grow-only) | Map of {nodeID: count} | Take max per nodeID | Like counts, view counters |
| PN-Counter | Two G-Counters (P and N) | merge both, subtract | Upvotes minus downvotes |
| LWW-Register | {value, timestamp} | Take the value with the higher timestamp | Single property (color, name) |
| OR-Set (Observed-Remove Set) | Set of {value, uniqueTag} pairs | Union of sets, remove only items seen locally | User presence, tags, members |
| RGA / YATA | Linked list of {id, content, leftID, rightID} | Insert by neighbor, tiebreak by clientID | Text editing, block ordering |

### Mechanism 2: Lamport Timestamps and Hybrid Logical Clocks

OT needs wall-clock time to order ops. Wall clocks drift. Two servers can report the same millisecond. CRDTs use **Lamport timestamps** instead:

- Every client maintains a local counter (its "logical clock").
- On every operation, the client increments its counter. The op is tagged `{clientID: "alice", clock: 42}`.
- On receiving an op with clock N, the client sets its own clock to max(local, N) + 1.
- Result: every operation gets a globally unique, causally consistent ID. No wall-clock dependency.

The op ID `(clientID, clock)` is permanent. It is how CRDTs refer to "where this character lives" instead of "at position 47." Positions are fragile (insertion shifts them). IDs are forever.

**Hybrid Logical Clocks (HLC)** blend physical time with logical time: `{wallClock: 1719360000, logical: 3}`. The wallClock gives human-readable ordering for debugging. The logical component breaks ties. Y.js uses a variant of this.

### Mechanism 3: YATA — how Y.js makes text editing O(1) per merge

Text editing is the hardest CRDT problem. Naive approaches (inserting at "position 5") break when two people insert at the same position simultaneously.

Y.js uses **YATA (Yet Another Transformation Algorithm)**, formalized by Nicolaas Matthijs in 2015. The insight: every character carries the IDs of its left and right neighbors **at the time of insertion**, not a position number.

Example:

```
Initial text: "Hello World"
Each character has an ID: (alice, 1) = H, (alice, 2) = e, ..., (alice, 11) = d

Alice inserts "!" after (alice, 11):
Op: {id: (alice, 12), content: "!", left: (alice, 11), right: null}

Bob simultaneously inserts "?" after (alice, 11):
Op: {id: (bob, 1), content: "?", left: (alice, 11), right: null}
```

Both ops say "insert after (alice, 11)." Classic conflict. YATA's merge rule: when two ops claim the same left neighbor and neither is the other's right neighbor, sort them by clientID lexicographically. "alice" < "bob", so Alice's "!" comes before Bob's "?".

Both Alice and Bob apply this rule locally. Both end up with "Hello World!?". No server needed. No transform functions. Deterministic.

**The performance win:** Each incoming op is applied in O(1) average case (look up the neighbor by ID in a hashtable, splice into the linked list). Reconnect with 7,200 ops: apply 7,200 times in O(1). Total: O(7,200) = milliseconds. OT reconnect: O(1,800 * 5,400) = O(9.7 million). CRDTs are asymptotically superior for offline-heavy workflows.

### Mechanism 4: State Vector for Efficient Sync

When Charlie reconnects, he does not want to download the entire document history. He wants only what he missed. The **state vector** makes this efficient:

```
Charlie's state vector: {alice: 1800, bob: 0, charlie: 900}
Server state vector:    {alice: 5400, bob: 3600, charlie: 900}

Diff: alice ops 1801-5400, bob ops 1-3600
```

The sync server sends Charlie exactly 7,200 ops. Not the 8,700 total ops in the log. The state vector is O(number of clients), typically a few hundred bytes. It turns reconnect sync from "download everything" to "download exactly what you missed."

### Mechanism 5: OR-Set for Safe Delete-then-Undo

Deletes are the hardest operation in CRDTs. If Alice deletes a block and Bob simultaneously edits that block's text, what should happen? Naive "delete wins" loses Bob's work silently.

The **OR-Set (Observed-Remove Set)** solves this by tagging every insertion with a unique token:

```
Bob adds block 42 with tag T1.
Server broadcasts: "block 42, tag T1 exists"

Alice: sees block 42. Deletes it. Sends: "remove all tags from block 42 that I have seen" (specifically T1).
Bob: simultaneously edits block 42, creating a new edit with tag T2.

Merge result:
- Alice's delete removes T1.
- Bob's edit is a new tag T2, which Alice never saw, so her delete did not cover it.
- Block 42 survives with Bob's edited content.
```

The rule: a delete removes only the tags it observed. New concurrent additions survive. This models real intent: Alice deleted what she saw; she could not have intended to delete Bob's simultaneous edit.

### Mechanism 6: Tree Move Operations (the hardest case)

Moving a block in Notion's tree (drag block 42 from under parent A to under parent B) while Bob simultaneously deletes parent B is the classic **tree anomaly** in CRDTs.

The naive CRDT: block 42 ends up under a deleted parent. The document is in a broken state (a block with no living parent).

The correct fix (from the 2021 Kleppmann et al. paper "A Highly-Available Move Operation for Replicated Trees"):

1. The move operation is stored as `{moveOp, undoMetadata}` — a log of what the tree looked like before the move.
2. When a concurrent delete reaches a node, the CRDT checks the move log.
3. If the move happened to a node whose parent was concurrently deleted, the move is **undone** and the block is placed back in its original parent.
4. Net result: no block is ever orphaned. All blocks have a parent. The document tree is always well-formed.

This is implemented in Automerge (2023 paper: "Extending JSON CRDTs with Move Operations"). Y.js does not yet support tree move CRDTs natively; applications that need it (like Notion) implement custom logic on top.

---

## 5. The Trade-offs

### CAP: CRDTs choose Availability and Partition Tolerance over Consistency

CRDTs are AP systems. Every replica accepts reads and writes even when disconnected. They converge to the same state eventually, but at any moment two replicas may disagree. This is the definition of eventual consistency.

**What this means per data type:**

| Data | CRDT choice | What "inconsistent" looks like |
|---|---|---|
| Text content | Eventual consistency (YATA merge) | Charlie sees "Hello!" while Alice sees "Hello!?" for 200ms until sync |
| Presence (who is online) | Eventual consistency (OR-Set) | Bob sees 3 active users, Alice sees 4, for up to 1 second |
| Block order | Eventual consistency (RGA) | During sync, blocks may appear in different orders briefly |
| Block existence | Eventual consistency (OR-Set) | Deleted block may appear alive briefly for offline users |

**None of these are catastrophic for a document editor.** A 200ms flicker where text looks different is acceptable. Losing a user's typed paragraph is not. CRDTs make the right trade.

### Storage grows without bound (tombstones)

When a character is deleted in YATA, its node is not removed from the linked list. It is marked as a **tombstone** (deleted but still present). Future inserts that reference it as a neighbor still work correctly. But tombstones accumulate.

After 10,000 insertions and 9,500 deletions, the CRDT still holds all 10,000 nodes — 9,500 of which are tombstones. Memory usage does not shrink with the document.

**The fix: garbage collection with causal consensus.**
When all known replicas have seen a tombstone (confirmed via state vector), the tombstone is safe to remove. The sync server can compact the log. This requires knowing the set of all replicas — which is trivial for closed documents (you know who has a copy) and hard for open-sharing (anyone could have a copy). Y.js documents that are never shared publicly can be compacted. Public shared documents need more careful protocols.

### Cost: CRDTs are more expensive than OT for small teams online

If everyone is always online and latency is low, OT is cheaper. The transform is O(1) per operation, the state is minimal (no tombstones), and the server is the single source of truth. CRDTs carry overhead:
- State vector per client (small but non-zero)
- Tombstones (bounded by total historical insertions, not current document size)
- YATA linked list (uses more memory than a plain text buffer for small documents)

For small documents (<1,000 ops), OT and CRDTs have similar real-world performance. For long-lived documents with heavy offline usage, CRDTs win decisively.

---

## 6. The Systems-Thinking Lens

**The failure mode: Transform Debt Spiral (a specific metastable failure)**

OT systems have a feedback loop that amplifies failures under exactly the conditions when you most need them to work.

Picture Notion's server under load during a product launch. Latency spikes from 50ms to 500ms. Clients accumulate more un-acked ops before the server confirms them. The offline op queue grows from 10 ops to 200 ops. When the server recovers, each client sends 200 ops instead of 10. The server runs 200 * 200 = 40,000 transform pairs per reconnecting client, times 1,000 clients = 40 million transforms. CPU maxes out. Latency rises higher. Clients accumulate even more ops. More transforms on reconnect. The cycle accelerates. The server never recovers. This is the **transform debt spiral** — a direct analogy to the retry death spiral for HTTP requests.

**The senior fix: break the loop, not the server.**

Adding CPU to an OT server under transform debt buys minutes, not hours. The loop just moves to the next capacity boundary.

CRDTs break the loop structurally:
1. **Each op is O(1) to apply.** Reconnect with 7,200 ops = 7,200 O(1) operations, not 9.7 million transform pairs. The work does not grow with contention.
2. **The sync server is stateless regarding merge logic.** It is a log store and message router. It cannot be overwhelmed by complex transform computation because it does no transform computation.
3. **Backpressure is simple.** If Charlie's 7,200 ops overwhelm the server's bandwidth, the server queues them. Each op is independent. Applying them in batches converges. There is no "I must apply all ops in the right order before I can serve anyone" bottleneck.

The principle: when your system has an O(n*m) hot path, adding capacity is never the fix. You must find the O(n+m) formulation. For collaborative editing, CRDTs are that formulation.

---

## Rare.lab Application

Rare.lab's core product is a **node-based shader editor** that compiles a visual graph to shippable GLSL/WGSL. The scene is a JSON tree: nodes, edges, parameter bindings, shader snippets. This is structurally identical to Notion's block tree.

**What Rare.lab already has (by analogy to this lesson):**
- Cloudflare R2 for immutable content-addressed scene snapshots — this is the checkpoint store.
- Supabase Postgres for manifest storage — this works as the op log if you add an append-only ops table.

**The current ceiling:**
Real-time multiplayer collaboration on shader graphs. Today, two users editing the same shader node graph would overwrite each other's work (last HTTP save wins — a naive LWW-Register on the whole document, not per node). This works for solo use; it breaks at 2 concurrent editors.

**The fix path:**
1. Add Y.js (or Automerge) to the client. Each node's properties become LWW-Registers. The node list is a Y.Map. Edge connections are an OR-Set.
2. Use **Cloudflare Durable Objects** as the CRDT sync server. Durable Objects are single-threaded, globally routed, and stateful — they are the perfect "one process holds the canonical CRDT state" design for the document server. No Redis, no WebSocket management infrastructure.
3. Persist the op log to the existing Supabase Postgres (one row per CRDT op, ordered by Lamport clock). Snapshots go to R2, same as today.
4. Result: two engineers can edit the same node graph simultaneously. Alice moves a noise node; Bob changes its frequency parameter. They see each other's cursors. Offline edits merge on reconnect. Nothing is lost.

The specific ceiling to watch: **parameter hot spots.** If a popular shader template has 10 engineers all tweaking the same `roughness` float simultaneously, you get 10 LWW-Register conflicts per second on that one property. The CRDT resolves them, but the winner-takes-all resolution means 9 of 10 edits per second are discarded. For creative tools where intent matters, show the conflict explicitly (like Figma's "property updated by Bob" indicator) rather than silently discarding 9 out of 10 writes.

---

## Sources and Summaries

**1. Y.js documentation + YATA internals**
https://docs.yjs.dev/api/internals

Y.js is the most widely deployed CRDT library in production (VS Code Live Share, Liveblocks, Hocuspocus, Remirror). The internals page covers YATA — the linked-list CRDT for sequences — including the corrected version of the original algorithm (the original YATA paper has pseudocode bugs; Y.js fixed them). Key fact: Y.js is the only known CRDT library that handles the "same-site concurrent insert" case correctly in all documented edge cases.

Summary in plain language: Y.js is the battle-tested open-source library that turns the theory in this lesson into code you can install in one line. The internals docs show the actual data structure: a linked list where each node has a unique ID and knows who its neighbors were at insertion time. Reading this makes the YATA mechanism concrete.

**2. Ink & Switch — "Local-First Software: You Own Your Data, in Spite of the Cloud" (2019)**
https://www.inkandswitch.com/local-first-software/

Co-authored by Martin Kleppmann, Adam Wiggins, Peter van Hardenberg, and Mark McGranaghan. Published at the 2019 ACM SIGPLAN Onward! conference. The paper that coined the term "local-first." It makes the philosophical case that cloud apps (Google Docs, Notion, Figma) give you convenience in exchange for data ownership — if the service shuts down, your documents may be gone. CRDTs are the technical foundation for apps that work fully offline and still sync when online, without a central server holding your data hostage.

Summary in plain language: A 10-year practitioner's manifesto on why software should work offline first and sync second, rather than the reverse. Section 3 lays out 7 ideals: fast (no round-trip for local edits), multi-device, offline, collaborative, long-term data preservation, privacy + security, user in control. CRDTs satisfy all 7. Worth reading in full; the Ink & Switch team built Automerge, the CRDT library Figma and others evaluated before building their own.

**3. Martin Kleppmann — "CRDTs: The Hard Parts" (Hydra 2020)**
https://martin.kleppmann.com/2020/07/06/crdt-hard-parts-hydra.html
Slides: https://speakerdeck.com/ept/crdts-the-hard-parts

Kleppmann is the author of "Designing Data-Intensive Applications" and a researcher at Cambridge who has spent years finding bugs in published CRDT algorithms. This 45-minute talk covers the edge cases that introductory CRDT content misses: the interleaving anomaly in text CRDTs (naive linked-list CRDTs can produce "aaabbb" when Alice types "aaa" and Bob types "bbb" at the same position — they should interleave, not block), the move operation problem for trees, and why garbage collection for tombstones requires global coordination.

Summary in plain language: "CRDTs sound easy; they are not." Kleppmann shows specific inputs that break naive CRDT implementations. He then shows how to fix them (the YATA tiebreaker, the undo-log for tree moves). This is the talk to watch before shipping any CRDT-based feature to production. Essential viewing for anyone implementing multiplayer on a node graph editor like Rare.lab.

**4. Kleppmann et al. — "A Highly-Available Move Operation for Replicated Trees" (2021)**
https://arxiv.org/abs/2105.14007

The academic paper that formally proves how to implement a correct tree-move CRDT. Previous CRDT literature had no good answer for "move node A to parent B while Bob deletes B concurrently." This paper introduces the undo-log approach: store the pre-move state alongside every move op, detect conflicts via causal ordering, and undo conflicting moves. The result is a tree CRDT that never orphans nodes.

Summary in plain language: The paper that makes tree-based collaborative editors (Notion blocks, Figma layers, shader node graphs) theoretically sound. Section 3 walks through the algorithm with concrete examples. If Rare.lab adds collaborative editing of a node tree, implement this algorithm or use Automerge's move support, which is based on this paper.

**5. Automerge — "Extending JSON CRDTs with Move Operations" (2023)**
https://arxiv.org/abs/2311.14007

Automerge is an open-source CRDT library (MIT license) that provides a full JSON datatype: maps, arrays, text, and as of 2023, move operations for list reordering. This paper documents how move operations interact with concurrent non-move operations (inserting into a list while someone reorders it). Automerge and Y.js are the only two production CRDT libraries with full JSON support.

Summary in plain language: The engineering paper behind Automerge's 2023 move-operation feature. Shows concrete test cases for list reordering conflicts and the algorithm used to resolve them. If you want to let users drag-reorder shader nodes while a collaborator edits one of those same nodes, this is the paper that proves the algorithm is correct.

**6. Notion Engineering — "The Data Model Behind Notion's Flexibility" (2021)**
https://www.notion.com/blog/data-model-behind-notion

Notion's official explanation of their block-based architecture. Every piece of content is a block (a typed node in a tree). Properties are decoupled from block type. The two-pointer system (upward parent pointer for permissions, downward content pointer for nesting) is how Notion handles arbitrary nesting. The real-time layer uses RecordCache + TransactionQueue on the client for local-first optimistic updates, then /saveTransactions for server validation, then MessageStore WebSocket subscriptions for push delivery.

Summary in plain language: The best public source on how Notion's document model actually works. Key insight: because properties are a flat key-value store per block (not typed fields per block), concurrent edits to different properties of the same block never conflict. This is the same insight Figma uses (lesson 003). The blog post is non-technical enough for PMs but specific enough for engineers to understand the data flow.

**7. Garbage-Collected Graph CRDT (2018) — decomposition.al**
https://decomposition.al/CMPS290S-2018-09/2018/11/12/implementing-a-garbage-collected-graph-crdt-part-1-of-2.html

A two-part implementation journal from a distributed systems research course. Part 1 covers building a 2P2P-Graph CRDT (a graph where nodes and edges can be added and removed, with CRDT semantics). Part 2 covers garbage collection: removing tombstones safely using a distributed commitment protocol (similar to two-phase commit but for "everyone has seen this tombstone").

Summary in plain language: The most concrete explanation of tombstone garbage collection available online. Part 1 shows the code; Part 2 shows why naive "delete tombstones after 30 days" breaks consistency, and the correct approach: you can only remove a tombstone once you have confirmed every replica has seen it. For Rare.lab, this means: if a user deletes a node in the shader graph while offline and later reconnects, the tombstone must be kept until all collaborators have received the deletion op. Only then can the node's storage be freed.

**8. Cloudflare Durable Objects documentation**
https://developers.cloudflare.com/durable-objects/

Durable Objects are Cloudflare Workers with guaranteed single-instance execution and durable storage. One Durable Object instance per document ID. Every write to that instance is linearized (no concurrent writes). The WebSocket connection from all collaborative clients goes to the same Durable Object. This makes Durable Objects the perfect sync server for CRDTs: the CRDT state lives in one place (the DO), all ops are routed there, and the DO fans them out to all connected clients.

Summary in plain language: Cloudflare's documentation on the building block that makes building a Figma/Notion-style collaborative server straightforward on their edge network. The key property: one DO instance per document, guaranteed serialized writes, global routing to the nearest edge node. For Rare.lab (already on Cloudflare), adding collaborative editing means adding a Durable Object per shader project. The DO holds the Y.js document state, accepts ops from clients, and broadcasts them. Zero new infrastructure.

**9. "Designing Data-Intensive Applications" by Martin Kleppmann (O'Reilly, 2017) — Chapter 9**
https://dataintensive.net/

The definitive reference book for distributed systems engineers. Chapter 9 covers consistency and consensus: linearizability, causality, total order, distributed transactions, and consensus (Paxos, Raft). Chapter 5 covers replication (leaders and followers, multi-leader, leaderless) and introduces CRDTs in the context of conflict resolution for multi-leader replication.

Summary in plain language: The book every infrastructure engineer has on their shelf. Chapter 5 pages 171-174 give the clearest one-page explanation of CRDT semantics (set union as a merge operation, why it is always safe). Chapter 9 pages 324-332 explain total-order broadcast and how it relates to consensus — important for understanding why OT needs a server (total-order broadcast) while CRDTs do not. If you read one book on distributed systems, this is it.
