# Day 22 — How does Amazon's shopping cart keep accepting writes when 2 of its own 3 replicas just went dark, with no leader to ask permission?

**Date:** 2026-07-04
**Difficulty:** Expert
**Topic:** Leaderless replication — consistent-hash preference lists, tunable N/W/R quorums, sloppy quorum + hinted handoff, vector clocks and sibling reconciliation (contrasted with wall-clock "last write wins"), Merkle-tree anti-entropy — via the original Amazon Dynamo paper, its open-source descendants (Cassandra, Riak), and Amazon's own 2022 departure from the model in DynamoDB
**Stack relevance:** Supabase Postgres (single-writer, strict-consistency side of this lesson today), Cloudflare R2 manifest reconciliation across offline/multi-device edits, any future leaderless or offline-first sync path in Rare.lab's runtime

---

## 1. The company and the breaking number

**Amazon, mid-2000s. The shopping cart service had one non-negotiable requirement: a customer must always be able to add an item to their cart, even while parts of the fleet are unreachable, because a rejected "add to cart" is a lost sale, and a customer who sees an error does not retry, they leave.**

By the time Amazon's engineers wrote up Dynamo for SOSP 2007, they had already lived through the alternative. A traditional relational setup — one master, one or more synchronous replicas — gives you a single, simple story for consistency: every reader sees the same answer. But it buys that simplicity by coupling *availability* to the health of a small, specific set of machines. The paper's own framing of the requirement is blunt: services at Amazon's scale need to state their latency guarantees at the **99.9th percentile**, not the mean, because at millions of requests a day the tail is not an edge case, it is a constant, and a hard majority-quorum or single-master design turns any tail failure into a full write rejection.

Here is the **breaking number**, and it is not a throughput number, it is a *locality* number: with a replication factor of **N = 3** (Dynamo's own canonical example throughout the paper), a strict-quorum design ties the availability of any single key to exactly 3 specific machines out of what might be a thousand-node fleet. The moment **2 of those 3** — not 2 of 1,000, just 2 of the 3 nodes this one key happens to hash to — are unreachable (a rack switch flaps, one disk is doing compaction, one AZ has a network blip), a strict quorum rejects **every write to that key**, full stop, while 997 other nodes sit perfectly healthy and idle. Multiply that across millions of keys spread evenly over the ring, and on any given day *some* key's specific 3 nodes will have 2 down, somewhere, purely from ordinary hardware churn. A design that fails a request because of purely local bad luck, in a fleet built specifically to survive bad luck at scale, is the exact failure Dynamo was built to eliminate.

Hold onto that number: **2 out of a key's own 3 replicas being briefly unreachable is enough to take "always writable" down to "sometimes rejects," inside a fleet that is otherwise almost entirely healthy.** Everything in this lesson is the set of mechanisms Dynamo used to make that specific failure mode disappear, without adding a single extra machine.

---

## 2. Why the naive (demo) design dies

The demo version of "store the cart" looks like this:

```sql
CREATE TABLE cart_items (
  cart_id     BIGINT,
  item_id     BIGINT,
  quantity    INT,
  updated_at  TIMESTAMP,
  PRIMARY KEY (cart_id, item_id)
);
```

One Postgres primary, maybe a synchronous standby for safety. This works fine for a demo, and it fails in three concrete ways once "always writable, worldwide, all the time" is the actual requirement:

**a. A single master is a hard ceiling on availability, not a soft one.** Every write funnels through one primary. If that primary is down, cart writes are down, full stop, until failover completes — and failover itself takes time and carries its own risk (a failover that picks the wrong replica, or races with a still-alive old primary, is how you get split brain). There is no way to make "one machine is temporarily unreachable" not matter, because the design has exactly one machine that is allowed to accept writes.

**b. Even a synchronously-replicated, no-single-master design still ties availability to a small, fixed set of nodes per key.** Suppose instead of one master you require a strict quorum: a write only succeeds if a fixed majority of that key's specific replica set acknowledges it (2 of 3, say). This removes the single point of failure, but it does not remove the coupling between "this one key" and "these three specific machines." As shown above, 2-of-3 unreachable for a key's own replica set is still a full write rejection for that key, and at scale, some key's replica set is unlucky on any given day.

**c. "Take the value with the newest timestamp" is not well-defined across machines, and Dynamo could not assume Google-grade clock hardware to fix it.** A tempting shortcut for resolving disagreement between replicas is "compare timestamps, keep the newer one." But clocks drift across machines and data centers by milliseconds to seconds without correction, so "newer" by wall clock is not the same as "happened after" in reality — a genuinely later write from a machine with a slightly-behind clock can silently lose to a genuinely earlier write from a machine with a fast clock, and the data is gone with no error raised. Google's answer to this exact problem, a decade later, was TrueTime: atomic clocks and GPS receivers in every data center, bounding clock uncertainty to single-digit milliseconds, at real infrastructure cost (Day 20 covered this). Amazon in 2007 needed an answer that required **no specialized clock hardware at all**, because ordinary commodity servers were the entire point.

The naive design's real mistake is assuming that "consistent" and "available" are both free if you just pick good hardware. Dynamo's premise is the opposite: pick availability as the non-negotiable property for this specific kind of data (a cart), and build the machinery to make eventual, reconciled consistency safe and cheap instead.

---

## 3. The architecture

Top to bottom, the shape Dynamo (and its open-source descendants, Cassandra and Riak) actually run:

```
[Client: browser / mobile app]
   "add running shoes to cart"
   |
   v
[Coordinator — ANY node in the cluster can play this role for THIS request]
   there is no fixed leader anywhere in this diagram; whichever node the
   client's request lands on (often via a load balancer) acts as coordinator
   analogy: whoever picks up the phone at the office handles your call end
   to end, there is no single "the manager" everyone must go through
   |
   v
[Consistent-hash ring → preference list]
   hash(cart_id) picks a point on the ring; walk clockwise and take the next
   N=3 DISTINCT physical nodes — that ordered list of 3 is this key's
   "preference list" (Day 10's ring, doing a second job: not just "which
   node owns this key" but "which N nodes, in priority order")
   analogy: a restaurant's default table, second-choice table, and
   third-choice table for a given party size, decided in advance
   |
   v
[Write fans out to the preference list, in parallel]
   |
   +--> [Replica 1 — healthy] ACK
   +--> [Replica 2 — healthy] ACK
   +--> [Replica 3 — UNREACHABLE]
   |     |
   |     v
   |    [Sloppy quorum: hand this replica's copy to the NEXT healthy node
   |     on the ring instead, tagged with a hint: "this belongs to
   |     Replica 3, forward it back once you see them again"]
   |     analogy: the post office does not refuse your package because
   |     your neighbor is on vacation — a different neighbor holds it and
   |     hands it over the fence the moment the vacationer is back
   |
   v
[Coordinator waits for W acknowledgments (W=2), then returns success
 to the client — it never waited on Replica 3 at all]
   |
   v (background, off the request path)
[Hinted handoff delivery] — once Replica 3 rejoins, the stand-in node
   pushes the held write back to its rightful home
[Read repair] — the next read that touches a stale replica compares
   versions and patches it up in-line, opportunistically
[Merkle-tree anti-entropy] — a periodic background job, independent of
   any single read or write, that finds and fixes divergence between
   replicas that neither hinted handoff nor read repair happened to touch
[Gossip protocol] — every node tells 1-3 random peers "here is who I
   think is alive" every second or two; membership and failure detection
   converge cluster-wide in a handful of rounds, with no central registry
   analogy: a rumor spreading through a large office by word of mouth
   reaches everyone in minutes, with no memo sent from a central desk
```

The request-path work (write fan-out, sloppy quorum, W acknowledgments) is what makes the write to Section 1's example finish successfully in milliseconds despite a dead replica. The background work (hinted handoff, read repair, Merkle-tree anti-entropy, gossip) is what quietly heals the resulting divergence afterward, on nobody's critical path.

---

## 4. The transferable mechanisms

**a. The preference list: consistent hashing answers "which N nodes," not just "which one."** Day 10 used the ring to answer "which single node owns this key." Leaderless replication reuses the exact same ring for a second, richer question: walk clockwise from the key's hash position and collect the next N *distinct physical* nodes (skipping past multiple virtual nodes that map to the same physical box) — that ordered list is the preference list, and it is computed by every node independently from the same ring state, with no coordinator needed to hand out the answer.

**b. Tunable quorums: N, W, R are a dial per operation, not a fixed property of the cluster.** Pick N (how many copies exist), W (how many must acknowledge a write before it succeeds), and R (how many must respond to a read before you trust it). The classic configuration is N=3, W=2, R=2: any write only needs 2 of 3 acks, any read only needs 2 of 3 responses, and because W+R > N, any read quorum and any write quorum are mathematically guaranteed to overlap by at least one node — so a read can never see a value that no acknowledged write ever touched. Raise W for stronger write durability at the cost of write latency; raise R for stronger read freshness at the cost of read latency; this is a per-table, sometimes per-request, choice, not a cluster-wide constant.

**c. Sloppy quorum + hinted handoff: trade "the same fixed 3 nodes, always" for "3 healthy nodes, whoever they are today."** This is the direct fix for Section 1's breaking number. Instead of insisting the write land on this key's exact 3 preference-list nodes even when one is down, Dynamo lets the write "slop over" onto the next healthy node on the ring, carrying a hint metadata field naming the node it's really meant for. Apache Cassandra's own architecture documentation describes this precisely: a coordinator stores a hint for an unreachable replica and, once that replica is confirmed alive again, replays the held writes back to it — with a bounded hint-retention window (Cassandra's default is **3 hours**, `max_hint_window_in_ms`) past which a held hint is discarded and the replica must instead be caught up by anti-entropy repair.

**d. Vector clocks: replace "which timestamp is bigger" with "which version is a strict ancestor of the other."** Instead of tagging a value with a wall-clock time, tag it with a small map of `{node_id: counter}` pairs, incremented by whichever node handled the write. Comparing two vector clocks tells you one of two things with certainty and no clock hardware involved: either one is a strict descendant of the other (safe to discard the ancestor, no information lost), or neither dominates the other, meaning they were written **concurrently**, without either write knowing about the other. That second case is not an error, it is expected, and it produces a **sibling**: two (or more) values Dynamo keeps *both* of, and hands back to the application to reconcile, rather than silently picking one and destroying the other's information. For a shopping cart, the reconciliation rule can be as simple as "union the two item lists" — an add-to-cart should never be the one that gets thrown away.

**e. Merkle trees: find divergence between two large replicas without comparing every row.** Each replica builds a binary hash tree: every leaf is the hash of one row's data, every parent is the hash of its two children, all the way up to a single root hash. Two replicas with identical data have identical root hashes in one comparison. If they differ, walk down: compare the next level's hashes, and only recurse into the branches that actually differ, ignoring every subtree whose hash matches. Apache Cassandra's own anti-entropy repair documentation gives the concrete shape of this: a Merkle tree built for repair is typically capped at **depth 15 (2^15 = roughly 32,000 leaves)**, regardless of how many billions of rows the underlying data actually has, because the tree's job is to localize *where* two replicas diverge, cheaply, not to enumerate every row.

**f. Gossip-based membership: no central node tracks who is alive.** Every node periodically picks 1-3 random peers and exchanges its current view of cluster membership and health. Disagreement resolves itself within a handful of rounds (`O(log N)` in cluster size) purely from repeated random pairwise exchange, the same mechanism Day 10 used for ring-membership convergence, doing double duty here for general failure detection with no single point of failure in the detection mechanism itself.

---

## 5. The trade-offs

**Consistency vs availability, concretely different per data type, inside the same company.** For the shopping cart, Amazon chose availability without hesitation: every replica keeps accepting adds during a partition, accepting the cost that two replicas might temporarily disagree and need reconciliation later, because "your add didn't go through" is a worse customer outcome than "your cart briefly showed slightly different contents on two devices before merging." That is not a universal answer even inside Amazon — a payment ledger or an inventory count that must never double-spend or oversell needs the opposite choice (Day 6 and Day 12's Stripe idempotency world), and Dynamo's own paper is explicit that it was never meant for that class of data.

**The real cost is moved, not eliminated: someone still has to write the merge logic.** Vector clocks and siblings do not resolve conflicts, they *surface* them safely instead of silently losing data. Every sibling that comes back from a read has to be reconciled by application code that understands the data's semantics (union two cart item-lists; for a plain counter or set, a CRDT — Day 15 — can automate this instead). Riak's own conflict-resolution documentation names the failure mode when this reconciliation does not keep up: **sibling explosion**, where an object accumulates dozens of unreconciled concurrent versions because clients keep writing without passing back the causal context they read, and Riak's docs recommend dotted version vectors specifically because they bound sibling growth far better than plain vector clocks do.

**Amazon itself revisited this exact trade twenty years later, for the AWS product that shares Dynamo's name.** This is worth being precise about, because the naming causes real confusion: **Dynamo (the 2007 paper) is leaderless**, per the architecture above. **DynamoDB (the AWS product, launched 2012)** is a different system that only borrowed the name and the partitioning idea. Amazon's own 2022 USENIX ATC paper on DynamoDB's internals describes each partition as a **replication group with an elected leader**, replicated via **Multi-Paxos**, where the leader holds a renewable lease and a peer can only trigger a new election once it detects the current leader as unhealthy. That is a leader-based, Raft/Paxos-family design (Day 11), not a leaderless quorum design — Amazon chose the opposite trade for DynamoDB-the-product's default consistency model, because most application developers, it turns out, would rather have a database resolve "who won" once per partition than receive siblings and write merge logic themselves. Leaderless replication is not obsolete (Cassandra and Riak both still ship it, at real scale, today), but it is a trade you should make deliberately, not by default.

---

## 6. The systems-thinking lens

**The feedback loop: sibling explosion is a metastable divergence loop, not a one-time conflict.** Under normal load, a rare concurrent write produces a sibling, a client reads both versions, merges them, and writes back a single reconciled value with the correct causal context — divergence resolves itself in one round trip. The loop breaks down under sustained concurrent write pressure on the same key: writes keep arriving faster than any single reader reconciles the growing sibling set → each new writer, unaware of the others' concurrent writes, adds yet another sibling instead of resolving one → the object's siblings grow → reads carrying that entire ballooning sibling set get larger and slower → slow reads get retried by impatient clients → retries are themselves new concurrent writers → more siblings pile on. Riak's own documentation names this exact failure mode "sibling explosion," and, like Day 21's compaction-falling-behind loop, once an object is deep in it, the system does not return to the healthy state on its own just because the traffic spike that triggered it subsides — the divergence is now the new steady state for that key.

**The senior fix breaks the loop's structure; it does not add nodes to it.** Adding more replicas or more capacity does nothing here, because the loop's bottleneck is causal bookkeeping on one specific key, not aggregate cluster throughput. The structural fixes operate on the loop directly: (1) **dotted version vectors instead of plain vector clocks**, which bound how many siblings a given set of concurrent writes can even produce, shrinking the loop's ceiling before it forms; (2) **application-level "last writer" hints or CRDTs for the specific data type**, so reconciliation happens automatically on read instead of waiting for a client to notice and merge; (3) the more radical structural fix Amazon itself took for DynamoDB-the-product — **give the key's partition a leader** so concurrent writes to the same key are serialized once, per-partition, before they ever get the chance to diverge into siblings in the first place, trading a small amount of write availability during leader re-election for the guarantee that this particular failure mode cannot happen at all. All three are the same move in different clothes: stop the divergence from compounding, rather than trying to out-merge it once it already has.

---

## Map to Rare.lab's stack

Rare.lab today sits entirely on the strict-consistency side of this lesson, and that is the right default for most of what it stores.

**Where the naive design in section 2 is exactly Rare.lab's current shape, and correctly so:** Supabase Postgres, single writer, is the right choice for scene metadata, project ownership, and billing — low write concurrency per row, relational by nature, and a lost or delayed write here is not remotely the same class of problem as a lost shopping-cart add. There is no reason to trade this away for leaderless replication's complexity; Section 5's lesson that Amazon itself only wants this trade for specific data types applies just as much to Rare.lab.

**Where it stops applying: multi-device and offline editing of the same project.** The moment a designer edits a shader graph on a laptop while offline, and the same project gets touched from a second device before the first reconnects, Rare.lab has exactly Section 1's structural problem in miniature: two writers, no leader reachable to either of them at the time of writing, and a requirement that neither edit simply vanish. Lesson 15's CRDTs already cover the case where the edited data has a well-defined automatic merge (structural node-graph edits, ordered lists). This lesson covers the case CRDTs do not: **the R2 manifest pointer itself** — "which scene-JSON object is current" — is a single value, not a mergeable structure, so two devices reconnecting with two different opinions about "current" is a genuine sibling, not something a CRDT can merge away.

**The concrete, actionable move:** stamp every manifest write with a small version vector — one counter per device or edit session, incremented on that device's writes — instead of, or in addition to, a wall-clock timestamp. When two devices reconnect and their manifest versions are compared, the version vector tells the runtime, with certainty and no reliance on either device's clock being correct, whether one manifest pointer is a strict descendant of the other (fast-forward, no conflict) or whether they are genuinely concurrent (a sibling: surface both to the user, or apply a documented policy such as "keep both as named branches" rather than silently discarding one device's work because its system clock happened to read a few seconds behind).

---

## References and summaries

**DeCandia, Hastorun, Jampani, Kakulapati, Lakshman, Pilchin, Sivasubramanian, Vosshall, Vogels: "Dynamo: Amazon's Highly Available Key-value Store"** (Amazon, SOSP 2007)
https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
The primary source for this entire lesson: the shopping-cart "always writable" requirement, the 99.9th-percentile SLA framing, the N/W/R quorum model with N=3 as the paper's own running example, sloppy quorum and hinted handoff, vector clocks and sibling reconciliation, and Merkle-tree anti-entropy. Also mirrored officially by Amazon at https://assets.amazon.science/ac/1d/eb50c4064c538c8ac440ce6a1d91/dynamo-amazons-highly-available-key-value-store.pdf.

**Vogels: "Amazon's Dynamo"** (All Things Distributed, 2007)
https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html
Amazon CTO Werner Vogels' own plain-language summary of the paper's motivation: the shopping cart's business requirement to always accept writes, and the conscious choice of eventual consistency for that specific data type. A faster on-ramp than the full paper.

**Apache Cassandra Documentation: "Dynamo"** (architecture overview)
https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html
The current, vendor-neutral (Apache Foundation) reference describing how Cassandra implements the Dynamo model in production: hinted handoff with the concrete default hint-retention window, read repair, and Merkle-tree anti-entropy repair including the depth-15 (~32,000 leaf) tree used to localize divergence between large replicas cheaply, the source for section 4's concrete Cassandra numbers.

**Lakshman, Malik: "Cassandra — A Decentralized Structured Storage System"** (Facebook, LADIS 2009)
https://www.cs.cornell.edu/projects/ladis2009/papers/lakshman-ladis2009.pdf
The original Cassandra paper, describing it as a deliberate combination of Dynamo's leaderless replication and partitioning model with a Bigtable-style (Day 21's SSTable lineage) data model — the direct link between this lesson and Day 21's LSM-tree lesson, since Cassandra is the concrete system running both mechanisms at once in production.

**Basho / Riak Documentation: "Conflict Resolution"** and **"Causal Context"**
https://docs.riak.com/riak/kv/latest/developing/usage/conflict-resolution/index.html
https://docs.riak.com/riak/kv/2.2.3/learn/concepts/causal-context/index.html
Riak is the open-source system that most faithfully implements the original Dynamo paper's sibling-based conflict model end to end. These pages are the primary source for section 4d and section 5's "sibling explosion" failure mode, including Riak's own recommendation to use dotted version vectors instead of plain vector clocks specifically to bound sibling growth.

**Elhemali, Gallagher, Gordon, Idziorek, et al.: "Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service"** (Amazon, USENIX ATC 2022)
https://www.usenix.org/system/files/atc22-elhemali.pdf
https://www.usenix.org/conference/atc22/presentation/elhemali
The source for section 5 and section 6's closing point: DynamoDB the AWS product uses per-partition replication groups with an elected leader and Multi-Paxos replication, a leader-lease renewal mechanism, and peer-triggered re-election on leader failure — a deliberate, documented departure from the original 2007 paper's leaderless model, made explicit by Amazon's own engineers fifteen years later.

**Brooker: "The DynamoDB paper"** (personal engineering blog, 2022)
https://brooker.co.za/blog/2022/07/12/dynamodb.html
An AWS engineer's own accessible walkthrough of the 2022 USENIX paper, useful as a faster second read after the primary PDF, and a clear, independent explanation of why DynamoDB-the-product diverged from Dynamo-the-paper's leaderless design.
