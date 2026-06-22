# Day 11 — How do 5 servers agree on a single truth when any of them can crash or lie?

**Date:** 2026-06-22
**Difficulty:** Advanced — Consensus and Raft
**Stack relevance:** etcd (Kubernetes), CockroachDB, TiKV, Kafka KRaft, Consul, Supabase Postgres HA

---

## 1. Named Company + The Breaking Number

**CockroachDB. 50,000 writes per second. One network cable disconnects.**

CockroachDB is a distributed SQL database used by companies like Bose, Comcast, and Netflix for multi-region, high-availability workloads. It replicates every write across a 5-node cluster using the Raft consensus algorithm.

Here is the breaking number: if you have 5 nodes and the network between the leader (Node 1) and the other 4 nodes goes down for 10 seconds, Node 1 keeps accepting writes. Nodes 2-5 elect a new leader and also accept writes. The same bank account record is now being updated simultaneously by two different leaders. At 50,000 writes per second, 500,000 writes land in two diverging logs. This is called **split-brain**. The two logs cannot be mechanically reconciled. Someone's money is wrong.

The naive design for failover — "if the leader crashes, any surviving node declares itself leader" — produces split-brain in exactly this scenario.

**The breaking number: one 10-second network partition at 50,000 writes/second creates 500,000 unreconcilable conflicting writes in a naive leaderless-failover system.**

---

## 2. Why the Naive Design Dies

The naive design is a **single-leader with automatic failover**. Simple version:

- Node 1 is the designated leader. All writes go to Node 1. Node 1 replicates to followers.
- If Node 1 crashes, a script detects it and promotes a new leader.

### It collapses in three concrete ways:

**a) A network partition looks like a crash but is not.**

Node 1's network disconnects. Followers 2-5 cannot reach Node 1. From their perspective, Node 1 is dead. From Node 1's perspective, the followers are dead. Both sides are alive. The failover script on Node 2 promotes Node 2 as the new leader.

Now two active leaders exist: Node 1 (which still believes it is the leader) and Node 2. Clients on Node 1's side of the partition keep writing. Clients on Node 2's side keep writing. The logs diverge. This is split-brain. Stripe's engineering blog notes that a 30-second split-brain in a payment system at 1,000 TPS produces 30,000 conflicting transaction states with no automatic reconciliation path.

**b) You cannot tell the difference between "slow" and "dead."**

How long do you wait before declaring a leader dead? Wait 1 second: you trigger spurious elections during normal garbage-collection pauses. Wait 60 seconds: your cluster is down for a full minute on every real failure. No timeout is simultaneously safe and fast. The naive design picks one arbitrarily and lives with the consequences.

**c) Partial writes have no recovery protocol.**

Node 1 receives a write request. It writes to itself and starts replicating to followers. It crashes after replicating to Node 2 but not Nodes 3, 4, 5. Is that write committed? Node 2 says yes. Nodes 3, 4, 5 say no. The naive system has no protocol for this. You either lose the write or you have permanent data inconsistency.

---

## 3. The Architecture: Raft, Drawn Top to Bottom

Raft is a **consensus algorithm**. Consensus means: a group of machines agree on one value, even if some of them crash and messages are lost or delayed. Raft was designed in 2014 specifically to be understandable. Its predecessor, Paxos, was so notoriously hard to implement correctly that every team got it wrong in different ways. Raft's defining choice: be understandable first, and prove correctness from that understandable foundation.

Raft's answer to split-brain is the **majority quorum rule**: a write is committed only when more than half the cluster confirms it. If a partition isolates 2 nodes from 3, the minority of 2 can never commit anything. Only the majority of 3 can form a quorum. Two majorities cannot exist simultaneously in a set of N things. This is not a heuristic. It is a mathematical fact: two groups each requiring more than N/2 members would together require more than N members, which is impossible.

### The Three Roles

Every Raft node is in one of three states at any moment:

- **Follower** - The default state. Receives log entries from the leader. Votes in elections. Stays silent unless spoken to. Like a new employee waiting for instructions.
- **Candidate** - A follower whose election timer fires. It has not heard from the leader recently, so it starts an election by asking others to vote for it.
- **Leader** - Won the election. Receives all writes. Replicates to followers. Sends heartbeats every 50-100ms to prevent followers from timing out and starting new elections.

### The Four Key Steps, Top to Bottom

```
[Client]
    |
    | "Write: account #7 balance = $900"
    v
[Leader — Node 1]
    |  Step 1: Append entry to its own log (NOT committed yet)
    |          Log entry: { term: 4, index: 1001, op: "set account#7 = $900" }
    |
    +---> [Follower Node 2]  AppendEntries RPC — replicate log entry
    +---> [Follower Node 3]  AppendEntries RPC — replicate log entry
    +---> [Follower Node 4]  AppendEntries RPC — replicate log entry
    +---> [Follower Node 5]  AppendEntries RPC — replicate log entry
    |
    | Step 2: Wait for acknowledgement from majority (3 of 5)
    |         Nodes 2 and 3 reply: "I have index 1001."
    |         That's 3 confirmations total (me + Nodes 2 and 3). Quorum reached.
    |
    | Step 3: Mark entry as COMMITTED. Apply to state machine.
    |         Account #7 is now actually $900 in the database.
    |         Notify Nodes 2 and 3 that index 1001 is committed.
    |         (Nodes 4 and 5 learn of the commit on the next AppendEntries)
    |
    v
[Client gets success response — durably committed]
```

**What each layer does in plain language:**

- **Leader** - The project manager. Only it assigns work (accepts writes). It coordinates all decisions.
- **Followers** - The team. They copy the manager's decisions into their own notebooks (log). They vote on who should be manager next.
- **Quorum** - The vote threshold. Like a company board requiring a majority before a resolution passes. Without a quorum, nothing is decided.
- **Log** - The ordered record of every decision made. Every node keeps its own copy. They must end up identical.
- **State machine** - The actual data (the database, the configuration). The log is the list of instructions. The state machine is the result of applying all instructions in order from the beginning.

### Leader Election

A follower starts an election when it does not receive a heartbeat within the **election timeout** (150-300ms default in etcd). It increments its term number and sends RequestVote to all peers.

```
[Candidate — Node 2, proposing to lead Term 4]
    |
    | Sends RequestVote to all nodes:
    | "I want to be leader of Term 4. My log goes up to index 1000."
    v
Node 3: "Node 2's log is as current as mine. I grant my vote."
Node 4: "Node 2's log is as current as mine. I grant my vote."
Node 5: "I already voted for Node 3 this term. I deny my vote."
Node 1: (partitioned, cannot respond)

Node 2 receives 3 votes (itself + Nodes 3 and 4).
That is a majority in a 5-node cluster.
Node 2 becomes leader of Term 4.
```

**The term number is the safety mechanism that prevents split-brain.** Every election increments the global term number. Nodes always defer to the node with the highest term they have seen. When Node 1 reconnects from the partition, it receives a message containing term 4. It was in term 3. It knows it is a stale leader. It immediately steps down to follower status. It rolls back any uncommitted entries in its log. It adopts Node 2's log. No split-brain. One term can have at most one leader. This is provable by induction.

### Scale Story at Three Tiers

**At 1,000 writes/second (startup scale):**
One 3-node Raft cluster. Leader on one machine, two followers on others. Quorum needs 2 of 3. Election completes in under 500ms. At 1,000 writes/second, a 500ms leader-election pause means 500 queued writes, all flushed when the new leader takes over. Trivial. This is what Supabase HA runs for a typical project.

**At 100,000 writes/second (growth scale):**
The single Raft leader is the write bottleneck. One leader serializes all consensus decisions. CockroachDB's solution: **range-based sharding where each range of the key-space has its own independent Raft group.** Instead of 1 Raft group for all data, you get 1,000 Raft groups, each handling a slice of the keyspace. 1,000 parallel leaders. 1,000 parallel consensus chains. Write throughput scales linearly with the number of ranges. This is how CockroachDB handles multi-hundred-thousand-TPS workloads.

**At 10+ million writes/second (hyperscale):**
Google Spanner runs one Paxos group per tablet (their name for a 100MB key-range). A 10PB database has 100 million tablets, each with its own independent consensus history and its own leader. No single leader is ever a global bottleneck. Spanner additionally uses TrueTime — hardware atomic clocks in every Google datacenter with bounded uncertainty of 7ms — to give cross-region transactions globally consistent timestamps without waiting for distributed clock agreement. The lesson: at hyperscale, consensus is run per-shard in parallel, not globally.

---

## 4. The Transferable Mechanisms

These patterns appear in every consensus system, not just Raft.

**a) Majority quorum prevents split-brain.**

A write is committed only when floor(N/2) + 1 nodes confirm it. For 5 nodes: 3. For 3 nodes: 2. A network partition can isolate at most N/2 - 1 nodes into a minority (by definition, the partition boundary splits the cluster). The minority can never form a quorum. The minority can never commit. Only the majority commits. Only the majority can elect a leader. Two simultaneous leaders is mathematically impossible. This is the single most important mechanism in all of distributed systems.

**b) Monotonic term numbers prevent stale leaders.**

Every election increments the global term. Nodes always follow the node with the highest term they have seen. A partitioned stale leader, when it reconnects, immediately discovers the newer term and steps down. You cannot impersonate a leader without winning an election in the current term, and you cannot win an election without a quorum. The term number is the "your security badge is revoked" signal.

**c) Log-first, commit-second decouples durability from acknowledgement.**

The leader appends to its log before committing. Followers append to their logs before acknowledging. Commit happens only after the majority has the entry in their local logs. If the leader crashes after logging but before committing, at least the majority has the entry. The next leader will discover these uncommitted entries and either commit them (if the previous leader had quorum) or roll them back (if it did not). Either outcome is deterministic and consistent. This is the same principle as Postgres's write-ahead log: record the intent before changing the data.

**d) Heartbeats suppress unnecessary elections.**

The leader sends heartbeats (empty AppendEntries RPCs) every 50-100ms. Followers reset their election timers on every heartbeat. In a healthy cluster, no elections ever happen. Elections are the failure-recovery mechanism, not normal operation. Keeping the heartbeat interval well below the election timeout (typically 10x smaller) means transient network delays do not trigger spurious elections.

**e) Pre-vote prevents disruptive isolated nodes.**

An isolated node that cannot hear the leader keeps incrementing its term. When it reconnects, it sends a RequestVote with a very high term number. The current leader, seeing a higher term, steps down. The isolated node cannot actually win an election (it does not have a quorum) but it has already disrupted the cluster. Fix: a **pre-vote phase** where a candidate first asks "would you vote for me if I started an election?" only if it gets quorum pre-votes does it actually increment the term and start the real election. An isolated node that cannot reach a quorum stays silent. etcd and CockroachDB both implement pre-vote.

**f) Linearizable reads require a quorum check.**

A naive read from the leader returns whatever the leader currently has. But the leader might be the minority leader from a partition that ended. It might not know the real current state. Safe linearizable reads: before serving the read, the leader confirms it still has a quorum via a lightweight round of heartbeats. The leader is confirmed to be the real leader. Only then does it return data. This is called **ReadIndex** in etcd. It adds one network round-trip to every read. Systems that tolerate slightly stale reads can read from any follower without this check, gaining read throughput at the cost of staleness.

---

## 5. The Trade-offs

### CAP Made Concrete

Raft is explicitly **CP: it chooses Consistency over Availability.**

During leader election (typically 150-500ms), the cluster rejects all writes. If you lose a majority of nodes simultaneously (3 of 5 die at once), the cluster halts. It does not serve stale data. It does not guess. It waits. This is a deliberate design choice: it is safer to be unavailable than to be wrong.

| Data type | What Raft does | What this means in practice |
|---|---|---|
| Bank account balance | Linearizable read (confirms quorum before returning) | You always see the real balance, never stale |
| Kubernetes pod state in etcd | Linearizable write required | No two controllers schedule the same pod |
| Service discovery config in Consul | Linearizable write | All services see the same routing table simultaneously |
| User session data | Often done outside Raft (Redis, memcached) | Availability over consistency is acceptable here |
| Shopping cart | Often done outside Raft (Cassandra, DynamoDB) | Prefer to always accept adds; reconcile later |
| "Likes" count | AP system (Cassandra) | A stale count by 2 seconds is acceptable |

**The choice CockroachDB makes:**

A transaction writes to the Raft log and waits for majority acknowledgement before returning success. Typical commit latency: 1-10ms for a 3-node cluster in the same datacenter. Cross-region (US East, US West, EU): 100-200ms for the quorum round trip. CockroachDB's payment-processor customers accept this latency because the alternative (split-brain producing double-charges) is catastrophic. A social app does not accept 100ms commit latency for "likes" and uses Cassandra instead.

### Cost vs Latency

Raft requires an odd number of nodes to avoid tie votes:

- **3 nodes**: tolerates 1 failure. Quorum = 2. Minimum viable production setup. What Supabase HA uses.
- **5 nodes**: tolerates 2 simultaneous failures. Quorum = 3. Standard for critical production databases.
- **7 nodes**: tolerates 3 failures. Used when nodes span multiple geographic regions and you need to survive a full region going dark.

More nodes means more round-trips for each write. The leader waits for the (quorum-size)th slowest response. With 7 nodes and quorum of 4, the 4th-slowest response drives commit latency. Adding nodes past 7 almost always hurts more than it helps. The latency tail of the quorum grows faster than the failure tolerance improves.

---

## 6. The Systems-Thinking Lens

### The Failure Loop: Election Storm

1. The cluster is under heavy write load. The leader's CPU is saturated. Heartbeats still go out on time, but the leader's AppendEntries RPCs are delayed because the network buffer is full.
2. Follower Node 3's election timer fires. It has not received a heartbeat within 150ms. It starts an election by incrementing to Term 5 and sending RequestVote.
3. The current leader sees a Term 5 message. It must step down, because it was in Term 4. The cluster stops accepting writes for 300ms while electing the new leader.
4. The new leader (Node 3) is immediately under the same heavy write load. Its heartbeats are again delayed. Another follower times out. Another election.
5. The cluster is in an election storm. No leader survives long enough to serve real traffic. Users see continuous write errors.

This is a metastable failure: the recovery mechanism (elect a new leader) causes the problem (disruption) that triggers the need for the recovery. Adding more nodes makes it worse by adding more followers that can time out and trigger elections.

### The Senior Fix: Break the Loop

**Fix 1 — Dedicate the heartbeat path.**

Heartbeats are tiny, empty messages. They should fire reliably even when write throughput is high. Fix: run heartbeats on a dedicated goroutine/thread with a separate queue, completely isolated from the data replication path. In etcd, the heartbeat goroutine has the highest scheduler priority. It fires on a clock tick, not after waiting for the data pipeline to drain. Never mix heartbeat processing with log replication processing.

**Fix 2 — Widen the election timeout range.**

Raft specifies that each follower picks a random timeout in a range (e.g., 150-300ms). This prevents all followers from timing out simultaneously and splitting their votes. Many teams accidentally tune this to a too-narrow range (e.g., 150-155ms), which effectively makes it deterministic. All followers fire at the same moment. Everyone starts an election. Nobody wins on the first round. Split votes repeat. Fix: use a range where the max is at least 2x the min. The default of 150-300ms in etcd is deliberately wide.

**Fix 3 — Backpressure before the leader.**

If the leader's log-replication pipeline is backing up, it should reject new incoming writes (HTTP 503 with Retry-After) rather than becoming slower. A slow leader generates delayed heartbeats, which generates elections. A leader that rejects writes early stays fast, sends heartbeats on time, and stays elected. The client gets a 503 and retries in 100ms. That is better than triggering a 300ms election that stops all writes for everyone. Kafka's KRaft implementation has explicit write-rejection thresholds for exactly this reason.

**Fix 4 — ReadIndex for reads.**

If every read is routed through the Raft log (as a no-op entry to establish linearizability), every read adds a consensus round-trip. Under read-heavy load, this adds to the leader's processing burden and can delay heartbeats. ReadIndex is cheaper: the leader just confirms it still has a quorum via the next heartbeat round. It does not write anything to the log. Reads complete in one heartbeat RTT instead of a full Raft round. etcd, TiKV, and CockroachDB all use ReadIndex as the default path for linearizable reads.

**The principle:** Raft's election mechanism is a feedback loop designed to detect and recover from leader failure. If you trigger elections too easily — short timeouts, overloaded heartbeat paths, narrow timeout ranges — the recovery mechanism becomes the failure mode itself. The senior fix is not "add more nodes." It is "protect the heartbeat path so elections only fire during real failures, not under load."

---

## Rare.lab Stack Mapping

**What you already have:**

Supabase Postgres HA is built on Patroni plus etcd. When you provision a Supabase HA project, a 3-node etcd cluster runs Raft behind the scenes, deciding which Postgres node is the primary at any moment. You already benefit from Raft without knowing it. Your writes to Supabase are protected by consensus-based leader election.

**What this means for your resilience:**

Your Supabase HA setup survives 1 node failure without data loss. The etcd quorum (2 of 3) elects a new primary in under 500ms. Users see a brief write rejection during that window. The Supabase client SDK retries automatically. This is already configured correctly for you.

**Where your next ceiling is:**

Supabase runs a single Postgres primary. At your current scale — a node-based shader editor with per-user projects — this is not a constraint. The ceiling appears when:
- Hundreds of concurrent shader compile jobs each write intermediate results simultaneously.
- Real-time collaboration has multiple users writing to the same scene node at the same millisecond.

**For real-time collaboration, Raft is not the right tool:**

If two users simultaneously move the same shader node in Rare.lab, you have a concurrent write conflict. The Raft approach serializes both writes through the leader — one write wins, one loses. The losing user sees their change disappear. This feels broken and destroys the collaborative experience.

The correct tool for collaborative editing is **CRDTs (Conflict-free Replicated Data Types)** — the next lesson in this curriculum. CRDTs make concurrent edits commutative: any node can accept any write, and the results merge correctly without coordination. Figma, Linear, and Notion all use CRDTs for collaborative document editing. Raft tells you exactly when NOT to use Raft. Understanding its limitations is as important as understanding how it works.

**One immediate action:**

Test what happens to your Cloudflare Worker and Supabase client during the 500ms Supabase leader election. Does the client SDK retry? Does it surface an error to the user? Add retry with exponential backoff (3 attempts, 100ms then 200ms then 400ms delay) on any Supabase write that returns a 503 or a connection error. This single change makes your app resilient to the one real-world failure Raft guarantees will happen periodically: leader death and re-election.

---

## References and Summaries

Every resource below is worth your time. Summary explains what is inside it so you know what to expect before opening.

---

### Research Papers

**1. "In Search of an Understandable Consensus Algorithm" — Diego Ongaro and John Ousterhout, Stanford, USENIX ATC 2014**

This is the Raft paper. The title explains the entire motivation: Paxos, the previous consensus standard, was so hard to understand that every team implementing it made different mistakes. Ongaro's full PhD thesis runs 187 pages; this conference paper is the accessible 18-page version. Section 5 covers leader election step by step with state diagrams. Section 6 covers log replication. The paper includes a user study: Stanford students who learned Raft scored significantly higher on comprehension tests than students who learned Paxos from the same instructor. Read Section 5 and 6 first, then go back to Section 3 (what Raft guarantees) once you understand the mechanics.
URL: https://raft.github.io/raft.pdf

**2. "Paxos Made Simple" — Leslie Lamport, Microsoft Research, 2001**

The original Paxos paper (1989) was rejected by peer reviewers who said it was "too basic." This 2001 follow-up explains Paxos in plain English, 13 pages. Read it after learning Raft, not before. It will show you exactly why Ongaro found Paxos hard to implement: Paxos describes the algorithm in protocol phases (Phase 1a, 1b, 2a, 2b) without explaining the intuition behind each phase. The phases are correct but the reasoning is implicit. Raft was designed to make that reasoning explicit. Reading Paxos after Raft lets you see what Raft improved and why.
URL: https://lamport.azurewebsites.net/pubs/paxos-simple.pdf

**3. "Spanner: Google's Globally Distributed Database" — Corbett et al., Google, OSDI 2012**

Shows what happens when you run Paxos at global scale across multiple continents. Spanner runs one Paxos group per shard (called a tablet). It uses TrueTime — hardware atomic clocks in every Google datacenter — to assign globally ordered timestamps to transactions. Without atomic clocks, cross-region serialized transactions require waiting for clock uncertainty to resolve (potentially hundreds of milliseconds). With TrueTime bounded to 7ms uncertainty, Spanner commits cross-region transactions with at most 7ms of extra wait. This paper is why clock synchronization is a first-class infrastructure concern at hyperscale, not an afterthought. Read Sections 3 and 4.
URL: https://www.usenix.org/system/files/conference/osdi12/osdi12-final-16.pdf

---

### Engineering Blogs

**4. "CockroachDB's Raft Implementation" — CockroachDB Engineering Blog**

CockroachDB runs one Raft group per range (16MB of keyspace by default). This post covers the operational reality: what happens when a node crashes, when a network partition occurs, when a disk fills up, and when a range grows and needs to split. Range splits are important: when a range exceeds 64MB, it splits into two ranges, each with its own new Raft group. This is how CockroachDB scales from 1 Raft group to thousands without operator action. The blog also explains how CockroachDB handles cross-range transactions (which cannot be decided by a single Raft group) via a two-phase commit protocol layered on top of Raft.
URL: https://www.cockroachlabs.com/blog/cockroachdb-on-rocksd/

**5. "etcd: How We Operate the Kubernetes Brain" — etcd Documentation**

etcd is the key-value store that holds all Kubernetes cluster state: pods, services, secrets, deployments, config maps. Every kubectl apply writes to etcd via Raft. This documentation explains the three tuning parameters that matter in production: `--heartbeat-interval` (default 100ms, how often the leader pings followers), `--election-timeout` (default 1000ms, how long a follower waits before starting an election), and `--snapshot-count` (default 10,000 log entries, when to compact the log into a snapshot). Also explains `etcdctl defrag` — the command you run to reclaim disk space from compacted logs, something every Kubernetes operator eventually needs.
URL: https://etcd.io/docs/v3.5/op-guide/configuration/

**6. "Why Kafka is Replacing ZooKeeper with KRaft" — Confluent Engineering Blog**

Until 2022, Kafka used Apache ZooKeeper (which implements ZAB, a Paxos variant) for leader election and metadata consensus. ZooKeeper was a separate system that operators had to run, monitor, and tune alongside Kafka. KRaft is Kafka's own Raft implementation that eliminates the ZooKeeper dependency entirely. This post explains the motivation (ZooKeeper was a scalability bottleneck at clusters with millions of partitions, where ZooKeeper's election traffic became overwhelming), the implementation choices, and the specific Raft optimizations Kafka added for its write-heavy patterns. Read this to understand why removing an external consensus coordinator simplifies operations dramatically.
URL: https://www.confluent.io/blog/why-replace-zookeeper-with-kafka-raft-the-log-of-all-logs/

**7. "Paxos vs Raft: Have We Reached Consensus?" — Heidi Howard and Richard Mortier, Cambridge**

Academic paper arguing that Paxos and Raft are more similar than they appear and the practical differences are in the specifics of leader rotation, log compaction, and membership changes rather than the core algorithm. Useful for understanding why teams that implement Raft end up with something that behaves like Multi-Paxos in practice. The key takeaway: the correctness argument for both is the same (majority quorum), but Raft's explicit state machine makes correctness easier to verify and audit. Important if you are evaluating which implementation library to use in a new system.
URL: https://arxiv.org/abs/1803.05069

---

### Video Resources

**8. "Raft: Understandable Distributed Consensus" — The Secret Lives of Data (Interactive Animation)**

Not a video but an animated, interactive web visualization. You watch 5 nodes go through leader election in real time, animated step by step. You can click on any node to kill it and watch the election happen live. You can pause the network between nodes and watch the partition play out. Every state transition is labeled with the message being sent. Spend 20 minutes clicking on things here before reading the paper. This single resource builds the visual intuition of Raft better than any textbook page.
URL: http://thesecretlivesofdata.com/raft/

**9. "Designing Data-Intensive Applications: Consensus and Replication" — Martin Kleppmann, GOTO Conferences**

Martin Kleppmann wrote "Designing Data-Intensive Applications" (DDIA), the most widely recommended book for distributed systems. This 45-minute conference talk covers the full arc without requiring you to buy the book first: why replication is hard, why single-leader replication breaks, what consensus means formally, how Paxos and Raft solve it, and what the Jepsen framework (which deliberately crashes distributed databases and checks for data loss) found when it tested real implementations. The section on why "multi-leader replication" systems like MySQL multi-master are not actually safe for financial data is particularly clear. Best video resource on this topic.
URL: https://www.youtube.com/watch?v=noUNH3jDLC0

**10. "Raft Consensus Algorithm Explained" — ByteByteGo, YouTube**

Alex Xu's animated short under 8 minutes. Covers leader election, log replication, and commitment in animated form. Good for a quick visual refresher after you have already read the Raft paper or worked through the interactive visualization above. The animation of what happens when a leader dies mid-replication and a new leader discovers the partially-replicated entries is particularly clear. Watch this before your next system design interview on distributed databases.
URL: https://www.youtube.com/watch?v=vRsbHoowGcI

**11. "Implementing Raft: Part 0 to Part 3" — Eli Bendersky (Google Engineer), Blog Series**

A practicing engineer who has worked at Google on the Go runtime walks through implementing Raft from scratch in Go, using the MIT 6.824 Distributed Systems lab assignments as the structure. Even if you never run the code, Part 0 and Part 1 explain every subtlety that the Raft paper glosses over: why the election timer must be per-follower with independent randomization, why you must check the term number before processing every incoming RPC (not just the initial ones), and why the commit index and the log index are two different numbers that move independently. These are the bugs every first-time Raft implementer makes. Read the series to avoid them.
URL: https://eli.thegreenplace.net/2020/implementing-raft-part-0-introduction/

**12. "Jepsen: Distributed Systems Safety Research" — Kyle Kingsbury (aphyr), Strange Loop Conference**

Kyle Kingsbury builds a testing framework that deliberately crashes, partitions, and corrupts distributed databases under real client load, then checks whether any confirmed writes were lost or whether any reads returned stale data that should not exist. He has tested PostgreSQL, Cassandra, MongoDB, Redis Cluster, etcd, CockroachDB, VoltDB, RethinkDB, and dozens more. This talk presents the findings: most databases claiming strong consistency failed his tests under realistic network conditions. CockroachDB and systems built on correct Raft implementations consistently passed. The Jepsen tests are the gold standard for distributed system correctness verification. The talk is engaging and the failure modes he finds are alarming.
URL: https://www.youtube.com/watch?v=tRc0O9VgzB0

---

### Articles

**13. "What is Raft? A Deep Dive" — Hello Interview**

Interview-prep-focused article covering leader election, log replication, cluster membership changes, and log compaction with clear diagrams. The section on cluster membership changes is particularly valuable: this is where most Raft implementations have bugs, because you are changing the quorum definition while the quorum is running. Example: if you add a new node to a 3-node cluster (making it 4), the quorum changes from 2 to 3 mid-operation. During the transition, two different quorums could theoretically exist simultaneously. Raft's solution is a two-phase joint-consensus approach; this article explains it without mathematical notation. Also covers what log compaction (snapshotting) is and why it is necessary — without snapshots, the Raft log grows forever.
URL: https://www.hellointerview.com/learn/system-design/deep-dives/raft

**14. "MIT 6.824 Distributed Systems Course Notes — Raft FAQ" — MIT PDOS**

The FAQ written by the course staff for MIT's most famous distributed systems course. Answers the specific questions students ask when they first implement Raft: "What does 'committed' actually mean?" "Can a leader commit an entry from a previous term?" "Why does the paper say you must not commit entries from previous terms by counting replicas?" Each question and answer is short (2-4 sentences) but precise. The answer to "Can a leader commit an entry from a previous term?" is particularly important: no, and understanding why reveals the subtle safety guarantee at the heart of the algorithm.
URL: https://pdos.csail.mit.edu/6.824/papers/raft-faq.txt

**15. "Notes on Distributed Systems for Young Bloods" — Jeff Hodges (Stripe)**

Not Raft-specific but written by a Stripe engineer who has operated distributed payment systems at scale. Covers the concepts Raft is built on: linearizability, serializability, the difference between consistency and durability, and what "eventual consistency" means in practice (it is not a single thing but a family of different, weaker guarantees). Essential background before the next lesson on CRDTs. CRDTs give up linearizability in exchange for high availability and zero coordination overhead. You need to understand exactly what you are giving up to know when that trade is correct.
URL: https://www.somethingsimilar.com/2013/01/14/notes-on-distributed-systems-for-young-bloods/
