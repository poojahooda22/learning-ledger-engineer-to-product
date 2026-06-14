# Day 3 — How does Figma let 100 designers edit one file at the same time?

**Date:** 2026-06-14
**Topic:** Real-time multiplayer at scale
**Difficulty:** Intermediate-Advanced
**Prereqs:** WebSockets basics, what a document tree is, rough idea of what "conflict" means when two people edit simultaneously

---

## 1. The company and the number that breaks everything

**Figma.** Real-time collaborative design. 8 million users as of 2021, growing fast.

The specific number: **100 concurrent editors sending updates every 33ms through one Rust process.**

Do the math. 100 people. Each sends a property change (move a shape, resize, recolor) 30 times a second. That is 100 x 30 = **3,000 messages per second** arriving at a single server process that holds the document in memory. Now imagine a viral onboarding doc with 200 editors in it. **6,000 messages per second.** One crash and you lose everyone's last 60 seconds of work.

That is the design challenge.

---

## 2. Why the naive version collapses

**The naive design:** One Node.js server. All clients connect to it via WebSocket. When Alice moves a rectangle, the server receives the update, saves it to Postgres, and broadcasts the change to Bob and Carol. Simple.

Here is where it falls over, specifically:

**Problem 1: Garbage collection pauses at 3,000 messages/sec.**
Node.js uses V8's garbage collector. Under sustained high throughput, GC can pause the process for 50-200ms to reclaim memory. During that pause, no messages are processed and no acks are sent. Every connected client sees a stutter. At 3,000 msg/sec, GC fires constantly. Figma actually measured this. The pauses were unpredictable and correlated with document complexity (more objects = more heap = more GC pressure). This was not fixable by tuning.

**Problem 2: You cannot horizontally scale a stateful process.**
The server holds the entire document tree in memory. That is necessary: to apply and validate incoming changes, the server needs to know the current state (what is the object's current x position? what are its children?). If you add a second server, it does not have the state. You cannot load-balance across multiple servers unless they share state through a database, which reintroduces latency and contention.

**Problem 3: One crash loses work.**
The naive server periodically writes a full document snapshot ("checkpoint") to storage every 30-60 seconds. If the server crashes 55 seconds after the last checkpoint, 55 seconds of edits are gone. Users see their work vanish. For a design tool where a single layout decision can take minutes, this is catastrophic.

**Problem 4: Conflicts with no resolution rule.**
What happens when Alice sets rectangle.x = 100 and Bob sets rectangle.x = 200 in the same 33ms window? Without a rule, you get random results depending on which message arrived at the server first. Users learn not to trust the tool.

---

## 3. The real architecture, layer by layer

Read this top to bottom. Each layer has one job.

```
[Alice's Browser]          [Bob's Browser]         [Carol's Browser]
     |                          |                         |
     | WebSocket (wss://)       | WebSocket               | WebSocket
     |                          |                         |
     v                          v                         v
+------------------------------------------------------------------+
|               CONNECTION / ORCHESTRATION LAYER                    |
|          (Node.js, stateless, routes by document ID)             |
|  Job: Accept WebSocket connections. Route each file ID to the    |
|  correct document process. Like a hotel switchboard operator     |
|  connecting your call to the right room.                         |
+------------------------------------------------------------------+
                               |
                    stdin/stdout message bus
                               |
                               v
+------------------------------------------------------------------+
|              DOCUMENT SERVER (Rust process, ONE per active file) |
|  Job: Hold the full document tree in memory. Receive property    |
|  updates. Apply last-writer-wins. Assign sequence numbers.       |
|  Fan out to all connected clients. Like the one referee at a     |
|  sports match who decides what counts.                           |
|                                                                  |
|  State held:                                                     |
|    Map<ObjectID, Map<PropertyName, Value>>                       |
|    (the full document as a flat property store)                  |
|                                                                  |
|  Two event types handled differently:                            |
|    DURABLE: property change  --> write to journal + broadcast    |
|    EPHEMERAL: cursor move    --> broadcast only, never persisted |
+------------------------------------------------------------------+
          |                                    |
          | write every ~0.5s                  | write every 30-60s
          v                                    v
+--------------------+              +----------------------+
|  JOURNAL           |              |  CHECKPOINT STORE    |
|  (AWS DynamoDB)    |              |  (AWS S3)            |
|  Job: Ordered log  |              |  Job: Full snapshot  |
|  of change deltas. |              |  of document state.  |
|  Each delta has a  |              |  Binary-encoded,     |
|  sequence number.  |              |  compressed.         |
|  Like a ship's log |              |  Like a save file in |
|  where every event |              |  a video game.       |
|  is written down   |              |                      |
|  in order.         |              |                      |
+--------------------+              +----------------------+

Crash recovery path:
New process loads latest S3 checkpoint --> reads sequence number N from it
--> fetches all DynamoDB journal entries with sequence > N
--> replays them in order --> document state is restored to within 0.5s of crash
```

**The non-canvas data path (LiveGraph):**
Team member lists, file browser contents, comments, notifications -- this is NOT canvas data. It flows through a separate system called LiveGraph. LiveGraph reads Postgres's replication stream (change data capture), detects which subscriptions are affected by each change, and pushes updates to subscribed clients. The canvas and the UI around the canvas are intentionally decoupled.

---

## 4. The conflict resolution model: why Figma does NOT use OT or CRDTs (and what it uses instead)

This is the heart of the lesson. Understanding why Figma's approach works requires understanding what it is NOT.

### What is Operational Transformation (OT)?

OT is the algorithm used by Google Docs for text editing. When two clients make concurrent edits, each operation is "transformed" to account for the other. Example: Alice inserts "X" at position 5, Bob inserts "Y" at position 5. The server transforms Alice's operation to "insert X at position 6" because Bob's insert shifted everything right. OT requires transform functions for every pair of operation types. These functions are extremely difficult to get right. Google's Wave team (who built OT for Wave) described it as one of the hardest correctness problems they encountered.

### What is a CRDT?

A Conflict-Free Replicated Data Type embeds the merge logic into the data structure itself so that any replica receiving the same set of updates in any order reaches the same state -- no server needed to define ordering. A Last-Writer-Wins Register (LWW register) is the simplest CRDT: track a timestamp with each write, and "merge" by keeping the highest timestamp. Figma uses something structurally similar.

### What Figma actually uses: server-authoritative LWW on (object, property) pairs

Figma's key insight: a design document is a tree of objects (like an HTML DOM). Each object has named properties. Position is a property. Color is a property. Width is a property.

**Most concurrent edits in a design tool touch DIFFERENT properties of DIFFERENT objects.**

Alice resizing the header rectangle does not conflict with Bob changing the button color. These are independent property writes. The only true conflict is: what if Alice sets `rect_1.x = 100` and Bob sets `rect_1.x = 200` at the same moment?

Figma's answer: **the server processes messages in arrival order. Last write wins. No transform functions needed.**

This works because:
- Property writes on different (object, property) pairs are always independent.
- Concurrent writes to the SAME (object, property) pair are genuinely ambiguous (neither Alice nor Bob is "more correct"). LWW is a reasonable resolution.
- Unlike text, you cannot produce a semantically broken result by overwriting one property. Setting x=200 after x=100 is fine; inserting a character at a wrong position in text is not fine.

The server tracks: for each (objectID, propertyName), the latest value received. Any client sending an update simply sets the new value. No transform functions. No vector clocks. The server's arrival order defines the truth.

### Child ordering: fractional indexing

One property is special: child order (which sibling comes before which). You cannot use integers because inserting between sibling 3 and sibling 4 would require renumbering everything.

Figma uses **fractional indexing.** Each child's position is a fraction between 0 and 1. Inserting between 0.5 and 0.75 sets the new child's position to 0.625. No conflicts. No renumbering.

The problem: repeated bisection creates fractions with exponentially growing precision. Inserting 100 times in the same gap produces strings like "0.500000000000...0625" that are hundreds of characters long. Figma's 2017 engineering blog post covers how they solved this with base-95 string encoding and a rebalancing algorithm that triggers when strings grow past a threshold.

---

## 5. Presence vs document state: the bifurcation that matters

Figma sends TWO kinds of messages over the same WebSocket connection, but treats them completely differently.

**Document state (durable):**
- What it is: property changes to canvas objects (move, resize, recolor, rearrange).
- Server action: apply LWW, assign sequence number, write to DynamoDB journal within 0.5s, broadcast to all clients.
- On client reconnect: replay from the latest checkpoint + unread journal entries.
- Guarantee: no change is lost after the 0.5s journal write window.

**Presence/cursor state (ephemeral):**
- What it is: where Alice's cursor is on the canvas right now, what she has selected, whether she is in the viewport.
- Server action: broadcast immediately to other clients. Write NOTHING to storage.
- On reconnect: presence is simply re-established from the current client state.
- Guarantee: none needed. If you disconnect, your cursor disappears from others' screens. That is the correct behavior.

Why this bifurcation matters: cursor updates arrive at 30 FPS (33ms). With 100 users, that is 3,000 cursor updates per second. Writing each one to DynamoDB would cost ~$3,000/day in DynamoDB write units (estimated; not a Figma-published number) and serve no durability purpose. Separating ephemeral traffic from durable traffic reduces storage cost by roughly 99% and eliminates a massive write amplification bottleneck.

---

## 6. The Rust rewrite: why language choice was an architectural decision

The original multiplayer server was Node.js (TypeScript). This was fine for early scale. At ~3,000 msg/sec with complex documents, it broke down:

- V8 garbage collection introduced latency spikes of 50-200ms every few seconds.
- Node's per-process memory overhead made it expensive to run one process per active document. They had to manually route "heavy" documents to dedicated worker pools.

In 2018, Figma rewrote the multiplayer server in Rust. Results:
- **10x improvement in serialization speed** (measured, not estimated).
- GC pauses eliminated entirely (Rust has no garbage collector; memory is managed via ownership).
- Per-process memory overhead dropped enough to make one-process-per-document practical at their scale.
- The Node.js orchestration layer remains; it routes connections and manages process lifecycle. Only the inner hot loop (message processing, state mutation, broadcast) moved to Rust.

The lesson: in a real-time system where tail latency matters (a 200ms GC pause is visible to users as a cursor freeze), language runtime choice is an architecture decision, not just a performance detail.

---

## 7. Scale story: what breaks at each tier

**1,000 concurrent users (early Figma):**
- One Node.js process per document, all users on one machine.
- GC pauses start to matter. Documents with 5,000+ objects cause measurable latency.
- One crash loses 60 seconds of work. Happens occasionally; users complain.
- Database: one Postgres instance. All good.

**10,000 concurrent users (growth stage):**
- Multiple documents active. Multiple servers needed. Routing by document ID.
- Node.js GC is now a real user-visible problem on large documents.
- Crash recovery gap (60s of lost work) is unacceptable for paying enterprise customers.
- The Rust rewrite happens here. Per-document process isolation becomes viable.
- DynamoDB journal added: crash recovery window drops from 60s to 0.5s.

**Millions of concurrent users (2021-present):**
- Hundreds of documents active simultaneously. Rust processes scale horizontally.
- Database becomes the new bottleneck. Single Postgres instance hits its limits.
- Figma adds vertical partitioning (2022): separate databases for different table groups (files, organizations, users). Reduces per-database load.
- LiveGraph (read path for UI data) needs to handle thundering herds when millions of subscriptions reconnect after a deploy. Added subscriber bucketing and lazy re-subscription.
- Horizontal sharding of first table shipped in September 2023. Custom DBProxy middleware handles shard routing transparently to the application layer.
- Current ceiling: the multiplayer process-per-document model scales well up to ~200 concurrent editors per document. At higher concurrency, the broadcast fan-out becomes expensive (one message must be sent to 200 clients = 200 serialization operations per update). This is the next known ceiling.

---

## 8. The 4 transferable mechanisms this topic teaches

### Mechanism 1: Server-authoritative sequencing as the simplest conflict resolution

Instead of implementing full OT or a CRDT, assign every change a monotonically increasing sequence number at a single authoritative server. All clients apply changes in sequence order. Conflicts resolve by arrival order. This works whenever the semantic cost of "last write wins" is acceptable for your data type. For property updates on independent objects, it is acceptable. For character-level text editing, it is not. Know your data model before choosing your conflict resolution strategy.

### Mechanism 2: Write-ahead journal + periodic snapshot (checkpoint + replay)

Do not rely on in-memory state alone for durability. Append every change to an ordered log (journal) at low latency. Periodically, write a full snapshot (checkpoint) to cheap object storage. On crash recovery: load the snapshot, replay the journal from the snapshot's sequence number. This pattern appears everywhere: PostgreSQL WAL, Kafka, Redis AOF, etcd. The journal gives you durability at millisecond granularity without the cost of writing to the database on every change.

### Mechanism 3: Bifurcate ephemeral from durable traffic

Not all real-time data needs durability. Presence signals, cursor positions, typing indicators, "is online" status -- these are self-healing on reconnect. Route them through a broadcast fan-out path (pub-sub) and never write them to storage. Route document mutations through a durable write path. Conflating the two wastes money and introduces unnecessary write amplification.

### Mechanism 4: Stateless routing layer + stateful worker layer

The connection/orchestration layer (Node.js) is stateless. It routes connections by document ID. The document server (Rust process) is stateful. It holds document state in memory. This separation means: the routing layer can be horizontally scaled and load-balanced freely. The stateful layer is constrained (one process per document) but that is acceptable because each document is independent. The job of the routing layer is to maintain the invariant: every client editing document D connects to the same Rust process.

### Mechanism 5: Process-per-tenant isolation

One Rust process per active document means document failures are isolated. A crash in document A does not affect document B. Memory leaks are naturally bounded per document. Garbage collection does not exist. When a document goes idle (no active editors), the process can be shut down and its state is recoverable from S3 + DynamoDB. This is a form of bulkhead isolation applied at the tenant (document) level.

### Mechanism 6: Fractional indexing for conflict-free ordered sequences

When you need to maintain ordered sequences with concurrent inserts, and renumbering is unacceptable, fractional indexing lets you insert between any two elements without touching neighbors. The trade-off is precision growth on repeated bisection, which requires a rebalancing mechanism. Alternative: use a tree-based ordered CRDT (like Logoot or LSEQ) when rebalancing is too expensive. Figma chose fractional indexing because design documents have far fewer insertions than text documents, so string growth is manageable.

---

## 9. The trade-offs: CAP made concrete, per data type

Figma's system makes different CAP trade-offs for different data types.

**Canvas property changes (durable path):**
- Consistency: strong within a document session. All clients see changes in the same server-assigned order. This is not eventual consistency; it is linearizability within the document server.
- Availability: the document server is a single point of failure. If it crashes, clients lose their WebSocket connection and reconnect (recovery takes a few seconds). There is a brief availability gap. Figma accepts this trade-off.
- What they gave up: horizontal scaling of the write path per document. You cannot write to two Rust processes for the same document simultaneously.

**Presence/cursor (ephemeral path):**
- Consistency: none guaranteed. Cursor positions are best-effort. High network jitter means Alice might see Bob's cursor at position (100, 200) even though Bob moved 200ms ago.
- Availability: high. Presence broadcasts are fire-and-forget. A delayed or dropped cursor update causes a cursor flicker at most.
- What they gave up: accuracy of presence data. Acceptable because stale cursor positions carry no semantic weight.

**Cost vs latency:**
- DynamoDB journal: costs more per write than S3, but delivers sub-second write latency. Used for the hot path (recent changes that need crash recovery).
- S3 checkpoint: costs almost nothing per storage byte, but write latency is seconds. Used for periodic snapshots only.
- This is a deliberate cost/latency layering. Hot data (journal) on fast expensive storage. Cold data (checkpoints) on cheap slow storage.

---

## 10. The systems-thinking lens: what feedback loop causes scale failure here

**The failure mode: correlated reconnect thundering herd.**

Scenario: Figma deploys a new version of their connection server. All WebSocket connections drop simultaneously. Thousands of clients reconnect at once. Each reconnection re-subscribes to LiveGraph queries and re-loads the document state. The database sees a spike of millions of reads arriving in seconds.

This is a **thundering herd**: many waiters simultaneously woken and competing for the same shared resource.

The dangerous feedback loop:
1. Server deploys, connections drop.
2. All clients retry simultaneously.
3. Database load spikes, query latency increases.
4. Client reconnect timeouts fire, clients retry AGAIN.
5. Retry storm amplifies the original spike. Database latency increases further.
6. New connections that succeeded start timing out because the database is overloaded.
7. Those connections drop and also retry. The loop is now self-sustaining.

This is a **metastable failure**: the system could have handled the original load if reconnects were staggered, but correlated retries push it past a tipping point from which it cannot recover without intervention.

**The senior fix breaks the loop, it does not add capacity:**

Three mechanisms, each attacking a different part of the loop:

**1. Jittered exponential backoff.** Clients do not retry immediately. They wait a random duration (say, between 1s and 10s) before reconnecting. This staggers the reconnect storm across a 10-second window instead of a 1-second window. The database sees 1/10th the peak load. Simple, free, highly effective.

**2. Lazy re-subscription.** Instead of re-subscribing to all LiveGraph queries immediately on reconnect, wait until the UI component actually needs data before re-subscribing. Most reconnections are brief (network blip). If the client recovers within 5 seconds, the subscription may never be needed.

**3. Load shedding with HTTP 503 + Retry-After.** The connection layer tracks current load. When load exceeds a threshold, it returns HTTP 503 with a "Retry-After: 15" header. Clients that obey this header do not retry for 15 seconds. This prevents the retry storm from forming. The key: 503 must be returned fast (before the database is consulted), otherwise the load shedding itself becomes a bottleneck.

Adding capacity (more database read replicas) helps but does not break the loop. If 10x replicas are added but clients all reconnect simultaneously, the loop simply takes 10x longer to kill itself. The fix is always to break the coordination.

---

## 11. What this means for Rare.lab

Rare.lab is an AI shader and visual-effects node editor. It compiles node graphs to shippable WebGL code and provides an embeddable runtime. Current stack: Supabase Postgres with RLS, Cloudflare R2 for immutable scene JSON and a manifest, one shared WebGL context in the runtime.

**Pattern 1: Journal + checkpoint -- you will need this.**
Right now, scene graphs are stored as content-addressed immutable JSON on R2. If you add collaborative editing (two designers editing the same node graph simultaneously), you need a write-ahead journal for change deltas. The R2 content-addressed approach is already the "checkpoint" layer. Adding a fast journal (could be Supabase's own Postgres table with a sequence column, or DynamoDB, or even a Cloudflare Durable Object) gives you crash recovery and collaborative edit history without changing the R2 architecture.

**Pattern 2: Your data model is Figma-shaped, not Google Docs-shaped.**
A shader node graph is a tree of nodes with named property maps (input ports, parameter values, connections). Concurrent edits to different nodes are independent. LWW on (nodeID, propertyName) pairs is a safe conflict resolution strategy for most cases. You do not need OT or CRDTs for the common case. Reserve CRDT complexity for the specific case of concurrent connection routing changes, where LWW can produce disconnected graphs.

**Pattern 3: Bifurcate presence from scene state.**
If you add multiplayer, cursor position, "which node is Alice hovering over," and live parameter scrubbing previews should be ephemeral. Scene graph mutations should be durable. Route them through different code paths. Do not write cursor positions to Postgres.

**Pattern 4: The WebGL context is your stateful bottleneck.**
One shared WebGL context per runtime instance is your equivalent of "one Rust process per document." It cannot be horizontally scaled within a single browser tab. If you want multiple users to share a single live preview, you have to multiplex updates through a single authoritative render loop. This is exactly Figma's problem. The same process-per-document isolation model applies: one runtime instance per shared canvas.

**Your current ceiling:** Supabase Postgres on a single instance. The moment you have enough users that Supabase's read replica lags or the primary hits its connection limit, you will hit the same wall Figma hit in 2020. The answer is Supabase's read replica feature first (R2 serves reads; Postgres serves only writes and subscription events), then table-level partitioning when that is not enough.

---

## 12. References with summaries

### Primary sources: Figma engineering blog

**"How Figma's multiplayer technology works" (2019, Evan Wallace)**
https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
The original deep-dive. Wallace explains Figma's choice to use server-authoritative LWW instead of OT or CRDTs, why a design tool's data model makes this work where text editing cannot, fractional indexing for ordered sequences, and the WebSocket server architecture. Required reading before any collaborative tool design. The key insight that most engineers miss: Figma is not eventually consistent in the CRDT sense; it uses a central server to define a total order, which is simpler to implement and reason about.

**"Realtime Editing of Ordered Sequences" (2017, Evan Wallace)**
https://www.figma.com/blog/realtime-editing-of-ordered-sequences/
The companion post specifically on fractional indexing. Explains in precise detail why naive fractions break under repeated bisection, what "string length growth" means in practice (after N insertions in the same gap, the position string has O(N) length), and how Figma's base-95 encoding with rebalancing solves it. Directly applicable to any system that needs conflict-free ordered list operations.

**"Rust in Production at Figma" (2018)**
https://www.figma.com/blog/rust-in-production-at-figma/
One of the best real-world Rust adoption case studies available. Shows measured latency data before and after, explains exactly what Node.js GC was doing to real-time message processing, and argues that per-process memory efficiency (not just raw speed) was the key Rust advantage. The 10x serialization improvement is a real measured number. Essential reading for anyone building a real-time system in a GC language.

**"Making Multiplayer More Reliable" (2021)**
https://www.figma.com/blog/making-multiplayer-more-reliable/
Explains the write-ahead journal (DynamoDB) architecture in detail. Covers the exact failure mode (server crash = 60 seconds lost work), why checkpoints alone are insufficient, and how sequence numbers enable precise replay. Also covers the client-side reconnect protocol: how clients merge local offline edits with the server's recovered state.

**"LiveGraph: Real-time Data Fetching at Figma" (2021)**
https://www.figma.com/blog/livegraph-real-time-data-fetching-at-figma/
Covers the non-canvas real-time data system. Explains how Postgres logical replication (change data capture) becomes a real-time push stream for UI data. Demonstrates why the canvas and the surrounding UI are architecturally separate real-time systems.

**"Keeping It 100(x) With Real-time Data at Scale" (2024)**
https://www.figma.com/blog/livegraph-real-time-data-at-scale/
The follow-up to the LiveGraph post. Covers what breaks at 100x scale: thundering herds on reconnect, subscription fan-out under heavy load, and how Figma added subscriber bucketing and lazy re-subscription. The most direct description of the metastable failure mode and how to fix it without just adding hardware.

**"How Figma's Databases Team Lived to Tell the Scale" (2023)**
https://www.figma.com/blog/how-figmas-databases-team-lived-to-tell-the-scale/
100x database growth story. Covers vertical partitioning (splitting one Postgres into many by table group), then horizontal sharding (splitting one table across multiple Postgres instances), and DBProxy (their custom sharding middleware). The central lesson: Postgres can scale far further than most teams think with careful partitioning before you need to shard.

---

### Academic papers

**"Conflict-Free Replicated Data Types" -- Shapiro et al., 2011**
https://dl.acm.org/doi/10.5555/2050613.2050642
Free PDF: https://www.lip6.fr/Marc.Shapiro/papers/2011/CRDTs_SSS-2011.pdf
The paper that formalized CRDTs. Two families: state-based (CvRDT, merge via a join operation on a semilattice) and operation-based (CmRDT, requires causal delivery of messages). Proves convergence: any two replicas that have received the same set of updates will have the same state, regardless of order. This is the mathematical foundation for offline-first collaborative software. If you design any system where replicas may diverge temporarily, this paper defines the vocabulary for talking about correctness.

**"Real Differences between OT and CRDT for Co-Editors" -- Sun et al., 2018**
https://arxiv.org/abs/1810.02137
Extended version: https://arxiv.org/abs/1905.01302
A systematic technical comparison of OT and CRDTs, written by engineers who built production OT systems. Key finding: OT and CRDT are not fundamentally different algorithms; they are two presentations of the same underlying merge problem. OT expresses merge via transform functions applied at message-send time; CRDTs express merge via the data structure's own operation semantics. The practical differences are in where complexity lives: OT complexity is in the transform functions (TP2 correctness is extremely hard); CRDT complexity is in memory overhead (tombstones accumulate forever) and the need for causal delivery.

**"Local-first software: You own your data, in spite of the cloud" -- Kleppmann et al., 2019**
https://martin.kleppmann.com/papers/local-first.pdf
Essay version: https://www.inkandswitch.com/essay/local-first/
The manifesto that reframed how engineers think about collaborative software. Proposes seven ideals: fast (no round trip for local edits), multi-device, offline, collaborative, longevity (data outlives the vendor), privacy, user control. Argues that cloud-centric apps (Figma, Google Docs) satisfy some ideals but fail longevity and offline. CRDTs are proposed as the technical foundation that can satisfy all seven. Read this before designing any collaborative tool. Even if you choose a centralized server architecture (as Figma did), this paper clarifies exactly what you are giving up and why.

**"Peritext: A CRDT for Collaborative Rich Text Editing" -- Litt et al., 2022**
https://dl.acm.org/doi/10.1145/3555644
Free PDF: https://groups.csail.mit.edu/sdg/pubs/2022/Peritext_PACM_HCI_2022.pdf
GitHub: https://github.com/inkandswitch/peritext
Plain-text CRDTs (Automerge, YATA) break when formatting spans are involved. If Alice bolds characters 3-7 and Bob inserts a character at position 5, what happens to the bold? Peritext's answer: link formatting to character identity (not integer offsets), so concurrent formatting operations are commutative. Key insight for node-based editors: any time your data model has "spans" or "connections" across elements, you face this same problem. Peritext's solution (identity-linked anchors) generalizes beyond rich text.

---

### Blog posts and essays

**"I was wrong. CRDTs are the future" -- Joseph Gentle, 2020**
https://josephg.com/blog/crdts-are-the-future/
Joseph Gentle built ShareJS and worked on Google Wave OT. This post is his public reversal after encountering Kleppmann's CRDT work. The most important thing it documents: OT in practice runs at 10,000-100,000 operations/sec in JavaScript and 1-20 million ops/sec in C, which means raw performance is not the argument against OT. The argument against OT is correctness (TP2 is genuinely hard to implement correctly for anything beyond simple text). CRDTs are necessary when you want decentralized or offline-first convergence without a server to define total order. Read this alongside Figma's blog to understand why Figma's server-centric LWW is the simplest solution for their specific constraints.

**"Towards a unified theory of Operational Transformation and CRDT" -- Raph Levien**
https://medium.com/@raphlinus/towards-a-unified-theory-of-operational-transformation-and-crdt-70485876f72f
Levien argues OT and CRDTs are two presentations of the same mathematical structure: a deterministic merge function over concurrent operations. OT expresses the merge function through transform functions applied at operation-send time. CRDTs express it through the data structure's own merge semantics at operation-receive time. This framing is useful for system designers: instead of asking "OT or CRDT?" ask "where do I want the merge complexity to live, and do I have a server to define total order?" If you have a central server, server-side sequencing (Figma's approach) is the simplest answer. If you do not, CRDTs give you convergence without coordination.

---

### Video and talks

**"New algorithms for collaborative text editing" -- Martin Kleppmann, Strange Loop 2023**
https://www.youtube.com/watch?v=Mr0a5KyD6BU
Talk page: https://martin.kleppmann.com/2023/09/22/strange-loop.html
Duration: ~40 minutes. The clearest visual explanation of why naive OT algorithms fail under concurrent edits, how Automerge (Kleppmann's CRDT library) works, and what Eg-walker (a new algorithm combining OT simplicity with CRDT guarantees) achieves. Includes live demos and diagrams. After watching this, the abstract correctness arguments in the papers become concrete. If you build any form of collaborative editing into Rare.lab, watch this first.

---

### HackerNews discussions (signal-to-noise ratio: high)

**"How Figma's multiplayer technology works" discussion (2019)**
https://news.ycombinator.com/item?id=21378858
Engineers debate why Figma's LWW works for a design tool but would fail for text. The crucial insight in the comments: text editing has positional semantics (inserting at position 5 is NOT independent from inserting at position 6) while design object properties are semantically independent (changing x is independent from changing color). This distinction between "positionally dependent" and "semantically independent" operations is the clearest practical test for whether LWW conflict resolution is safe.

**"OT and CRDT trade-offs" discussion**
https://news.ycombinator.com/item?id=22039950
A broader discussion with engineers from Google, Dropbox, and various collaborative tool companies. Covers implementation experience with both OT and CRDTs in production, where each has broken, and why the academic literature underrepresents the tombstone accumulation problem in long-lived CRDTs (every deleted element must be kept forever to prevent re-insertion; this is a memory leak that grows with document edit history).

---

## Summary: the one-paragraph version

Figma handles 100+ concurrent editors by routing all editors of a document to a single Rust process that holds the document in memory, applies last-writer-wins on independent (object, property) pairs, assigns sequence numbers, and broadcasts to all clients. Durability comes from a write-ahead journal in DynamoDB (written every 0.5 seconds) layered under periodic full snapshots to S3. Cursor positions are never persisted -- they are broadcast-only. The Rust rewrite replaced Node.js specifically to eliminate GC pauses, not for raw throughput. The conflict model works because design properties are semantically independent in a way that text characters are not. The database scaled from one Postgres to 12+ vertical partitions to horizontally sharded tables over three years, with custom sharding middleware. The main failure mode to guard against is the correlated reconnect thundering herd, which breaks the database under apparent normal load after a deployment.
