# Day 73 - Why does typing one character into a shared Google Doc need a transform function, when moving a shape in Figma does not?

**Date:** 2026-09-27
**Difficulty:** Expert
**Topic:** Operational transformation, the mechanism Google Docs uses to merge concurrent keystrokes from many people into one document, and the reason it caps out hard at 100 simultaneous editors on a single file. This lesson sits deliberately next to two the ledger already taught. Day 3 (Figma) solved "many people, one document, in real time" by fanning property diffs out of one stateful process, but Figma's edits are scalar property writes (move this shape to x=140) that commute: whichever one lands last wins, no rewriting required. Day 15 (CRDTs) solved concurrent editing with no server at all, at the cost of tombstones and metadata that grow forever. Google Docs' problem looks the same from the outside, two people typing in the same paragraph, but the edits do not commute: inserting a character at position 12 shifts every later position, so "apply both inserts in whatever order they arrive" produces two different, both-wrong documents unless something rewrites one operation in light of the other. That rewriting step is operational transformation (OT), and the "something" that makes it tractable is a single sequencing server per document, which is exactly the thing that makes OT scale differently, and worse, than either of this ledger's other two answers to the same-sounding problem.
**Stack relevance:** Rare.lab's node-based editor is a graph of nodes and edges, closer to Figma's shape-property model than to a flat text buffer, so most Rare.lab edits probably do commute and do not need OT's transform machinery, the CRDT-style approach from Day 15 is likely the right default. But the moment Rare.lab lets two people edit the *same expression inside the same node* (a GLSL snippet, a formula field, an ordered list of shader passes where position matters), that sub-problem stops commuting and becomes exactly Google Docs' problem in miniature. This lesson is the playbook for that one case: keep it centralized and per-object, and know in advance that "add more collaborators" is not a capacity problem you can throw more servers at.

---

## 1. The company and the breaking number

**Google Docs, and the number 100.** Google Docs enforces a hard cap: up to 100 people can simultaneously edit (and comment on) a single document; more people than that can only view it. That is not a made-up round number for a demo, it is the shipped, documented product limit, and it exists because of what happens inside the server the moment concurrent editors climb past roughly that range.

**The mechanism underneath the cap.** Google Docs' real-time editing, shipped in its current form in September 2010, is built on operational transformation, a technique first described by Nichols, Curtis, Dixon, and Lamping in their 1995 UIST paper on the Jupiter collaboration system. The idea: every edit is not a snapshot of "here is my new document," it is an *operation* ("insert 'x' at position 12"), and every operation that arrives out of order relative to another concurrent operation must be mathematically transformed, its position or intent adjusted, against every operation it did not know about yet, before it is safe to apply.

**Where the wall actually is.** Etherpad, an open-source pad editor built on the same OT lineage, hit this wall in production and documented it precisely. Real numbers from Etherpad's own scaling investigation: a single pad stayed stable with 75 concurrent editors, needed connection-pooling fixes to hold steady at 100, and became unreliable, with connection success rates dropping to 53-99%, once concurrent editors passed roughly 150. Server CPU and memory were nowhere near exhausted when this happened (7.5 of 12 cores, 3.5 of 62 GB of RAM); the constraint was structural, not a hardware shortage. Etherpad's own engineers name the reason directly: OT's transformation cost per incoming operation scales with how many other operations are concurrently in flight, so total server-side work across a burst of concurrent edits grows roughly as the square of the number of concurrent editors, O(n^2), not linearly. Double the simultaneous typists in one document, and you do not double the sequencing server's work, you roughly quadruple it.

**The number that breaks the naive design:** one document, 100 people typing, each sending a handful of keystroke-operations per second, all of which must be sequenced and transformed by exactly one process that owns that document's current revision number. That process cannot be swapped for "add more app servers behind a load balancer," because the thing it is doing, deciding the one true order operations happened in, is inherently a single-writer problem for that one document.

---

## 2. Why the naive (demo) design dies

**The obvious version:** treat this like any other stateless web feature. Put a fleet of identical app servers behind a load balancer, have each client send its edits to whichever server it lands on, have each server write straight to a shared database, and let the database's own consistency handle the rest.

**Death one: last-write-wins silently deletes half of everyone's typing.** If two people each insert a character into the same paragraph and the system just applies "whoever's write reaches the database last, wins," one of the two edits is not merged, it is discarded. Unlike Figma's shape-move, where discarding a slightly-stale position update is barely noticeable, discarding someone's actual keystroke is data loss they will see and complain about immediately. Text edits do not commute the way property writes do; position 12 in "hello world" is a different character depending on whether an earlier insert has already happened at position 3.

**Death two: stateless horizontal scaling does not apply to a single hot document.** The instinctive fix for load, add more identical servers, works when requests are independent of each other. It does not work here, because every operation touching one document must be ordered relative to every other operation touching that same document. Routing different editors of the same doc to different, uncoordinated app servers just recreates death one at a different layer: now two servers each think they know the current revision, and neither is right once the other's edit lands. The unit that must scale here is not "requests per second," it is "documents," and one document's edit stream fundamentally cannot be split across independent servers without a rendezvous point.

**Death three: a federation of independent servers, each doing its own OT, is a harder problem than one server doing OT.** Google's own history contains the control experiment. Google Wave, launched in 2009 using the very same Jupiter-derived OT, was designed to federate: any organization could run its own Wave server, and servers would exchange and transform operations with each other the way email servers exchange mail. That meant every pair of federated servers had to agree, live, on a single relative order for operations neither of them originated alone, a much harder version of the TP2 convergence property (see below) that a lone central sequencer never has to solve, because it is the only party deciding order. Wave shut down in 2010, the same year Google Docs shipped its non-federated, single-owner-per-document version of the same underlying algorithm. One version of OT survived past its first year; the harder, decentralized version of the identical algorithm did not.

---

## 3. The architecture

```
Clients (browser tabs, one per editor, in a shared Google Doc)
  - job: apply the user's own keystrokes to its local copy
    immediately (so typing never waits on the network), tag each
    outgoing operation with the document revision it was based on,
    and buffer any operations sent but not yet acknowledged
  - analogy: writing on your own carbon-copy page while a shared
    ledger is being kept somewhere else; you know your page might
    need a correction once the ledger's official version comes back

        |
        v
Realtime gateway / WebSocket tier (stateless, many instances)
  - job: hold the live connection to each client and route its
    operations to whichever backend process currently owns that
    document; itself holds no document state, so it scales the
    ordinary way, more instances behind a load balancer
  - analogy: a bank's teller windows: any window can serve you,
    but every window sends your deposit slip to the one vault
    ledger that actually owns your account balance

        |
        v
Per-document sequencer (one logical, stateful owner per doc)
  - job: hold the document's current revision number and the
    short window of recent operations in memory; for every
    incoming operation, transform it against every operation
    already applied since the revision the client last saw, then
    assign it the next revision number and broadcast it to every
    other connected client
  - analogy: a single scorekeeper at a live scrabble-by-mail game
    who is the only person allowed to say "this move happened
    3rd, so here is how your 4th move actually lands on the board"

        |
        v
Operation log (append-only, per document)
  - job: durable, ordered record of every transformed operation,
    so a client that disconnects for 30 seconds can reconnect and
    replay exactly what it missed instead of resyncing the whole
    document
  - analogy: the deposit-slip ledger's paper trail; you do not
    need to recount the whole vault, just read the entries since
    your last visit

        |
        v
Periodic snapshot + durable storage (per document)
  - job: collapse the operation log into a full document state
    every so often, so old operations can be archived and a fresh
    client join does not require replaying the document's entire
    edit history from character one
  - analogy: a bank periodically printing a fresh statement so it
    does not need last decade's carbon slips to tell you your
    current balance

        |
        v
Sharding by document ID across many sequencer processes
  - job: give each document exactly one owning sequencer, and
    spread millions of documents' sequencers across a fleet, so
    the fleet scales with the number of active documents, not
    with how many editors any single document has
  - analogy: one bank has many vaults, each with its own dedicated
    ledger-keeper; a busy vault does not get to borrow capacity
    from a quiet one, it just means that vault stays busy
```

---

## 4. The transferable mechanisms

- **Single-writer sequencing per shard, not per fleet.** The hard part of OT is not the math, it is that someone must hold a canonical, linear order of events for one object. The fix that makes this tractable is scoping that single-writer requirement down to the smallest unit that actually needs it (one document) and sharding at that boundary, so the constraint never has to apply fleet-wide. This is the same idea as a database primary owning writes for its shard, applied to an in-memory document instead of a database row.
- **Transform, don't overwrite.** Where a CRDT (Day 15) resolves conflicts by defining a merge function on final states, OT resolves them by rewriting one operation's intent ("insert at position 12") in light of another operation that already changed what position 12 means. This is the right tool specifically when operations do not commute, positions, ordered lists, anything where "where" depends on "what already happened."
- **Optimistic local apply with a reconciliation buffer.** Every client applies its own edits to its local copy instantly, never waiting on a round trip, and keeps a buffer of operations it has sent but not yet had acknowledged. When the server's authoritative version of an earlier operation comes back, the client transforms its own buffered, still-unacknowledged operations against it. This is the same optimistic-UI pattern behind Day 52's client-side prediction in multiplayer games, applied to text instead of position.
- **A revision number as the causality token.** Every operation is tagged with the document revision the client had last seen. This is a minimal, single-object stand-in for the vector clocks Day 22 used for leaderless replication: instead of tracking causality across a whole cluster, one monotonically increasing integer per document is enough, because there is only one sequencer deciding order for that object.
- **Sharding by owning key as the actual scaling lever.** The fix for "one hot document has too many editors" is never "give that document's sequencer a bigger machine" beyond a point, it is capping participants (Google's 100-editor limit) and, at the fleet level, spreading documents' sequencers across many shards so the system scales with document count. This is Day 10's consistent hashing, reused with "document ID" as the shard key instead of "user ID" or "cache key."
- **Coalescing before you send.** Google Docs batches rapid, adjacent keystrokes into fewer, larger operations before they ever leave the client, the same debounce principle that keeps Day 65's metrics pipelines from shipping one packet per data point. Fewer, larger operations directly reduce the O(n^2) transform cost, because n is the count of concurrent operations, not the count of keystrokes.

---

## 5. The trade-offs

- **Consistency vs. availability, and it is per-document, not global.** The document's own edit order must be strongly consistent, every client must eventually see the same sequence of transformed operations, because a linear history is the entire point. But that strong-consistency requirement is scoped to one document's sequencer; if that one process is briefly unavailable, only that document's live collaboration stalls, every other document in the fleet is unaffected. Google Docs chooses strict order over availability for the one object that needs it, and buys back overall system availability by making the blast radius exactly one document.
- **Cost vs. latency in how history is kept.** Keeping every operation forever would make reconnection trivial (replay everything) but storage and replay cost would grow without bound. Google Docs pays a small, bounded cost, periodic snapshots plus a short recent-operations log, to keep both storage and reconnection latency flat regardless of how old or how heavily edited a document is.
- **Simplicity vs. decentralization.** A single sequencer per document only has to satisfy the weaker TP1 convergence property (any two concurrent operations, applied in either order, converge to the same result), because there is one authority deciding the canonical order. A federated or peer-to-peer version of the same algorithm has to additionally satisfy TP2 (transforming a third operation along two different, equivalent sequences of prior transforms must still agree), which is dramatically harder to implement correctly. Google Docs deliberately gave up Wave's federation to keep this problem in the easier, TP1-only regime, and that is very likely part of why Docs is still running today and Wave is not.

---

## 6. The systems-thinking lens

**The feedback loop: an OT transform-cost death spiral.** Picture a class of 120 students all opening the same shared doc at the start of a lecture and typing at once. More concurrent, unacknowledged operations arriving at the sequencer means each new operation must be transformed against a longer backlog, which is O(n^2) work as shown by Etherpad's own numbers. That work takes longer, so acknowledgments to clients arrive more slowly. Slower acknowledgments mean clients accumulate more locally-buffered, still-unsent-or-unacked operations before they can safely send the next one, which increases n again. This is a genuine metastable-failure loop, not a plain capacity shortage: past a threshold, more concurrent typists do not just slow the system down proportionally, they push the sequencer further behind at an accelerating rate, and the system does not recover on its own even if some editors stop typing, because the backlog it built up has to be drained first.

**The senior fix breaks the loop instead of adding capacity.** You cannot fix an O(n^2) work function by giving the one sequencer process a bigger machine, that only moves the threshold slightly, it does not change the shape of the curve. The actual fixes are admission control and demand shaping applied before the spiral starts: Google Docs' hard 100-editor cap is load shedding applied preemptively, refusing the 101st concurrent editor rather than degrading everyone's experience once they are already in; client-side coalescing shrinks n by reducing how many discrete operations a burst of typing generates in the first place; and sharding by document ID guarantees that even a maximally overloaded document's spiral stays contained to that one document's sequencer and never becomes a fleet-wide incident. None of these add capacity to the hot spot; all of them shrink or contain the loop that makes the hot spot fatal.

---

## Sources

- [What's different about the new Google Docs: Making collaboration fast, Google Drive Blog, September 2010](https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs.html): Google's own primary account of shipping operational-transformation-based real-time collaboration in the new Google Docs. Direct fetch was blocked by this session's network egress policy; described here from search-indexed summaries.
- [What's different about the new Google Docs: Working together, even apart, Google Drive Blog, September 2010](https://drive.googleblog.com/2010/09/whats-different-about-new-google-docs_21.html): the companion post on the collaboration protocol itself. Direct fetch blocked this session; described from search-indexed summaries.
- [How many simultaneous collaborators can edit a Google Doc file?, Google Docs Editors Community](https://support.google.com/docs/thread/4715007/how-many-simultaneous-collaborators-can-edit-a-google-doc-file?hl=en): primary-adjacent source (Google's own support community) confirming the 100-simultaneous-editor product limit referenced throughout this lesson.
- [Guidance on scaling Etherpad to hundreds/thousands of concurrent editors on a single pad, Etherpad GitHub Issue #7403](https://github.com/ether/etherpad/issues/7403): primary source, read directly, for the concrete concurrency numbers (stable at 75, stabilized at 100 after connection-pooling fixes, unreliable at 150+, unusable at 300), the observation that CPU and memory were not exhausted when failures occurred, and the PostgreSQL connection-exhaustion root cause found in that investigation.
- [Performance: Look deeper into scaling N+ contributors per pad, Etherpad GitHub Issue #7756](https://github.com/ether/etherpad/issues/7756): primary source for the O(n^2) characterization of OT transform cost scaling with concurrent operations per pad.
- [Operational Transformation, Google Wave whitepaper, Apache Wave (incubator) project](https://svn.apache.org/repos/asf/incubator/wave/whitepapers/operational-transform/operational-transform.html): primary technical description of the Jupiter-derived OT model used in Google Wave and, subsequently, Google Docs. Direct fetch blocked this session; described here from search-indexed summaries.
- [On Consistency of Operational Transformation Approach, arXiv:1302.3292](https://arxiv.org/pdf/1302.3292): academic source for the formal TP1 and TP2 convergence properties and why a single-server (central-sequencer) OT system only needs to satisfy the weaker TP1, while a peer-to-peer or federated system must satisfy the strictly harder TP2.
- Nichols, D. A., Curtis, P., Dixon, M., and Lamping, J., "High-latency, low-bandwidth windowing in the Jupiter collaboration system," UIST '95: the original 1995 paper describing the Jupiter algorithm that Google Wave's and Google Docs' OT implementations are derived from, referenced here via secondary academic citations rather than a direct read of the original proceedings.
- Day 3 (Figma real-time multiplayer), Day 10 (consistent hashing and sharding), Day 15 (CRDTs for collaborative editing), Day 22 (leaderless replication, quorums, and vector clocks), Day 52 (real-time multiplayer netcode, client-side prediction), Day 55 (anycast global server load balancing), Day 65 (timeseries metrics, cardinality, and coalescing): the ledger's own prior lessons this one directly reuses, contrasts against, or recombines.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of Google's own 2010 Drive blog posts and the Apache Wave OT whitepaper, so those two sources are summarized here from search-indexed excerpts rather than a full direct read, consistent with how this ledger has flagged network-blocked sources on prior days (see Day 69 through Day 72). The two Etherpad GitHub issues, by contrast, were fetched and read directly, and the concrete concurrency numbers in sections 1 and 2 (75/100/150/300 concurrent editors, the CPU and RAM figures, the O(n^2) characterization) come from that direct read, not a search summary. The 100-editor Google Docs limit is corroborated independently by Google's own support community thread. The historical claims about Jupiter, Google Wave's federation model, and its 2010 shutdown are corroborated across multiple independent search results (academic citations, Wikipedia-indexed summaries, and contemporaneous technology press) even where the primary UIST '95 paper itself was not directly fetched this session.
