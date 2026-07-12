# Day 30 — How do you reach agreement when some of the nodes voting might be lying?

**Date:** 2026-07-12
**Difficulty:** Expert
**Topic:** Byzantine fault tolerance: the escalation from Day 11's Raft, which only survives nodes that crash or go silent, to a consensus protocol that survives nodes that actively lie, via Castro and Liskov's 1999 PBFT algorithm and the O(n²) message-broadcast wall it hits at real validator counts, and Yin, Malkhi, Reiter, Gueta and Abraham's 2019 HotStuff protocol that collapses that wall to O(n) and now sits underneath Meta's Diem (formerly Libra) blockchain as DiemBFT
**Stack relevance:** Rare.lab's multiplayer node-graph runtime (Day 15's CRDTs) merges every peer's edit as though it were sent in good faith, because every peer today is an authenticated Rare.lab client on infrastructure Rare.lab controls. The instant that assumption breaks, a public multiplayer room, a marketplace where a third party's compiled node bundle merges into a shared library, an edit accepted from a client Rare.lab does not fully control, CRDT convergence stops being a safety property and starts being an attack surface, because a CRDT merge function faithfully applies a malicious op exactly as faithfully as it applies an honest one

---

## 1. The company and the breaking number

**Every consensus protocol this ledger has covered so far, Raft (Day 11), leaderless quorums (Day 22), TrueTime (Day 27), assumes a node that fails does so by crashing or going silent, never by sending a well-formed message that is a lie.** That assumption, crash-fault tolerance, is correct for Rare.lab's own servers, for Google's Spanner replicas, for a Postgres primary and its followers, because all of those nodes run code one organization controls, on infrastructure one organization operates. Raft's safety proof does not hold for a single second past the moment one of the voting nodes is actively adversarial rather than merely broken, because a crashed node stops talking, but a Byzantine node keeps talking, and it can tell different peers different, mutually contradictory things in the same round, which is a failure mode crash-fault consensus was never built to detect, let alone survive.

Here is the breaking number, and it is a shape, not a rate: **O(n²) messages per consensus decision.** Castro and Liskov's 1999 PBFT (Practical Byzantine Fault Tolerance) algorithm was the first to make Byzantine consensus fast enough to be practical rather than purely theoretical, and it requires n = 3f + 1 total replicas to survive f Byzantine (arbitrarily malicious) nodes, one more full replica set than Raft's crash-only 2f + 1. But the real cost is in PBFT's message pattern: its prepare phase has every one of the n replicas broadcast to every other replica, and its commit phase does the identical all-to-all broadcast again. Both phases are O(n²) by construction. At n = 20 replicas, a small permissioned validator set, that is a few hundred authenticated messages per decision, entirely tolerable. At n = 100 replicas, the number this lesson's real example was built around, all-to-all broadcast means each of the two phases sends on the order of n(n-1) ≈ 9,900 point-to-point authenticated messages, roughly 20,000 messages to finalize a single decision, and doubling the validator count to 200 does not double that cost, it roughly quadruples it, because the cost scales with n², not n.

Facebook (now Meta) hit this wall directly building Diem, the permissioned blockchain formerly called Libra, and said so in their own technical papers. A payments network meant to onboard banks, payment processors and other institutions as validators needed a validator set that could plausibly grow into the hundreds, and classic PBFT's O(n²) broadcast cost meant throughput and latency would degrade sharply as exactly the kind of institutional adoption the project wanted actually happened. The Diem team's answer was not to run PBFT at a smaller validator count and call it done, it was to adopt HotStuff, Yin, Malkhi, Reiter, Gueta and Abraham's 2019 protocol, specifically because HotStuff's communication complexity is linear, O(n), in the number of replicas, the first Byzantine protocol to combine that linearity with responsiveness (running at the actual speed of the network rather than a fixed worst-case timeout). DiemBFT is HotStuff, adapted and shipped in production.

Hold onto the shape of that number: **quadratic broadcast cost is what makes textbook Byzantine consensus impractical past a few dozen nodes, and the entire second half of this lesson is the engineering story of turning that quadratic cost linear without weakening the guarantee it buys.**

---

## 2. Why the naive (demo) design dies

The demo version of "tolerate a lying node" looks like a straightforward extension of majority voting, the same intuition Raft already trained: collect votes, if more than half agree, commit.

```
Client -> broadcasts request to all n replicas
Each replica -> executes the request, broadcasts its own result to
                every other replica
Client -> waits for f+1 matching replies, accepts that as the answer
```

This fails in three concrete, distinct ways the instant one participant is actually malicious rather than merely absent.

**a. A simple majority (n/2 + 1, Raft's threshold) is not enough once a node can actively lie, because a Byzantine node can vote in the majority for two contradictory outcomes at once, and majority alone cannot prove which one is real.** Raft's crash-fault math works because a crashed node contributes zero votes, so a majority of the *remaining* honest nodes is unambiguous. A Byzantine node can send replica A a message saying "commit value X" and replica B a message saying "commit value Y" in the same round, both signed, both well-formed. If ordinary majority voting is used, it becomes possible to assemble two different "majorities" for two different, conflicting outcomes, which is exactly the split-brain a consensus protocol exists to prevent. This is why PBFT and every serious Byzantine protocol after it requires a **quorum of 2f + 1 out of n = 3f + 1**, not a bare majority: the arithmetic is deliberately set so that any two quorums must overlap in at least one *honest* node, which is the only thing that can catch a leader lying to different parts of the network.

**b. Waiting for unanimous or near-unanimous agreement before committing means a single unresponsive node, honest or not, stalls the whole system, which fails the same liveness test Raft was built to pass.** If the naive design requires all n replicas to reply before the client accepts an answer, then a single crashed replica, with zero malicious intent at all, halts every future decision forever. This is the same problem Raft solves with "any majority of alive nodes can proceed," and any working Byzantine protocol needs the identical property: liveness under up to f simultaneous failures, whether those failures are crashes or lies, not just safety against lies while paying for full unanimity.

**c. Broadcasting every vote to every other node, the intuitive way to let everyone independently verify everyone else's claim, is exactly the O(n²) cost from section 1, and it gets worse, not better, as trust decreases.** The instinct to fix "a node might lie" by having every node cross-check every other node directly is correct in spirit, an honest node needs to see enough independent votes to know a claimed quorum is real, but implemented as literal all-to-all broadcast it is the mechanism that makes classic PBFT's cost quadratic. The naive design's mistake is treating "more redundant cross-checking" as free, when each additional replica added specifically to tolerate more Byzantine faults *also* adds a full round of n messages to every other replica, so the fix for the trust problem is what directly causes the scale problem.

---

## 3. The architecture

Top to bottom, the path a single client request takes through a modern linear-communication Byzantine protocol (HotStuff / DiemBFT), contrasted at the phase level against where classic PBFT paid its O(n²) tax:

```
[Client: submits a signed request/transaction]
   the request itself carries the client's signature, so no replica
   can forge what the client actually asked for, only argue about
   the ORDER requests are applied in
   analogy: a sealed, signed letter, nobody along the relay chain can
   rewrite its contents, they can only argue about which letter
   arrived first
   |
   v
[Leader (rotates every view): collects requests, proposes a block]
   PBFT's PRE-PREPARE phase and HotStuff's PROPOSE phase both start
   here, one node, the current leader, packages pending requests
   into a proposal and sends it out; this step is O(n) in both
   protocols, a single broadcast from one node, never the bottleneck
   analogy: the meeting chair who writes up the agenda and circulates
   it, one document going out to everyone, not everyone individually
   comparing notes with everyone else
   |
   v
[PBFT: PREPARE phase, all-to-all broadcast, O(n²)]
   every replica that accepts the proposal broadcasts its own signed
   vote directly to every OTHER replica, so each replica can locally
   assemble 2f+1 matching votes without trusting the leader's word
   for it; this is the first quadratic phase
[HotStuff: replicas vote by sending their signature to the LEADER
 only, O(n); the leader combines 2f+1 signatures into ONE threshold
 signature, a Quorum Certificate (QC), and broadcasts the QC back]
   this is the single structural change that kills the O(n²) cost:
   votes flow replica-to-leader (n messages), not replica-to-everyone
   (n² messages), and a threshold signature scheme lets those n
   individual signatures collapse into one constant-size proof
   analogy: PBFT is every juror phoning every other juror to compare
   verdicts; HotStuff is every juror phoning ONE court clerk, who
   then posts a single certified verdict on the courthouse door
   |
   v
[PBFT: COMMIT phase, all-to-all broadcast again, O(n²) a second time]
   PBFT repeats the identical all-to-all pattern a second time before
   any replica actually applies the request, doubling the quadratic
   cost per decision
[HotStuff: a second QC round, still leader-relayed and O(n), CHAINED
 onto the next proposal rather than run as a separate round-trip]
   HotStuff's real efficiency trick, beyond linearity: it PIPELINES
   the phases, each new proposal simultaneously carries the previous
   proposal's second-round vote, so steady-state throughput pays for
   roughly one phase's worth of latency per block, not three
   analogy: an assembly line where inspecting the last part and
   starting the next part happen in the same motion, instead of
   stopping the whole line to finish one inspection before starting
   the next part
   |
   v
[Commit: 2f+1 matching QCs on the same value = safe to apply]
   once a replica has seen the requisite chain of quorum certificates
   agree, it applies the request to its state machine; the 3f+1
   replica count guarantees any two quorums of 2f+1 share at least
   one honest node, which is the actual source of the safety proof
   |
   v
[View change / pacemaker: leader timeout -> next leader rotates in]
   if the current leader stalls or is caught behaving dishonestly,
   a timeout fires and the protocol elects the next leader in a
   fixed rotation, carrying forward the highest QC any honest replica
   has already seen, so a new leader can never silently erase
   progress a quorum already certified
   analogy: if the meeting chair goes silent or starts reading from
   the wrong agenda, the room moves to the next person on the
   pre-agreed chair rotation, who has to pick up from the last
   officially minuted decision, not start over from scratch
```

The one box with no equivalent in Raft is the **quorum size and threshold-signature mechanics** (2f+1 of 3f+1, collapsed into one QC). Everything else, a rotating leader, a timeout-driven view change, chained/pipelined phases, has a direct Raft analog. It is specifically the extra replica, the extra quorum overlap guarantee, and the move from all-to-all to leader-relayed voting that turns "tolerates a node going silent" into "tolerates a node actively lying," without paying Raft's crash-only protocol's assumption that every vote it receives is honestly reported.

---

## 4. The transferable mechanisms

**a. Quorum sizing tuned to the failure model, not just "a majority."** Raft's 2f+1-out-of-2f+1-alive is correct for crash faults because a crashed node casts no vote at all. Byzantine consensus needs 3f+1 total nodes and a 2f+1 quorum specifically so that any two quorums are guaranteed to overlap in at least one *honest* node, the extra f replicas exist purely to make that overlap-guarantee-despite-lying possible. The transferable idea: quorum math is not a fixed constant, it is derived from exactly what kind of failure you need to survive, and getting the failure model wrong (using Raft's math against Byzantine nodes) breaks safety silently.

**b. Threshold signatures to collapse O(n) individual proofs into O(1) verification cost.** HotStuff's jump from PBFT's O(n²) to O(n) rests on a specific cryptographic primitive: instead of a leader forwarding n separate signatures to prove a quorum voted, a threshold signature scheme lets 2f+1 partial signatures combine into a single, constant-size signature that any node can verify in one operation. This is the same class of move as Day 28's probabilistic data structures, trading a small, well-understood cost (the cryptographic combination step) for a large, structural win (proof size stops growing with replica count).

**c. Leader-relay instead of all-to-all broadcast: route votes through one aggregation point, not through every peer.** This is the single change that matters most in section 3's diagram. It generalizes far beyond consensus: any time a design has every node individually verifying every other node's claim, ask whether a trusted (or provably-checkable) aggregation point can turn an O(n²) fan-out into an O(n) funnel, the same shape as a load balancer replacing direct client-to-client negotiation, or a message queue replacing direct producer-to-consumer coupling (Day 9).

**d. Chained pipelining: overlap consecutive rounds instead of completing one fully before starting the next.** HotStuff's phases are designed so each new proposal simultaneously carries the previous round's second vote, the same latency-hiding idea as TCP's sliding window or a CPU pipeline overlapping instruction fetch and execute. The transferable lesson: once a protocol has more than one sequential round-trip per decision, look for whether the next round's request can be dispatched before the current round's response is fully processed, rather than serializing rounds that do not actually have a data dependency on each other.

**e. View change with carried-forward state, not "start the election from zero."** A new leader in both Raft and HotStuff must prove it is picking up from the highest already-certified state any honest replica has seen, never allowed to silently roll back a decision a quorum already agreed on. This is the same guarantee Day 24's fencing tokens provide against a stale lock-holder: a leadership change is only safe if it is provably monotonic, later leaders strictly dominate earlier ones in what they know was already committed.

**f. Signed, non-repudiable messages as the trust anchor everything else is built on.** Every message in PBFT and HotStuff, the client's request, each replica's vote, the leader's proposal, is cryptographically signed by its sender. This is what makes a Byzantine node's lie *provable after the fact*, a malicious replica that sends replica A one signed claim and replica B a contradictory signed claim cannot later deny having sent either, because both carry an unforgeable signature. Crash-fault protocols like Raft do not need this, because a crash-fault node is assumed truthful when it does speak; Byzantine protocols need it because the entire safety argument rests on being able to prove, not just infer, what a dishonest node actually said.

---

## 5. The trade-offs

**This is CAP made concrete at the cost of the replica set itself, not just at the replication topology.** Tolerating f Byzantine faults costs 3f+1 total replicas and a full round of cryptographic signing and verification on every message, against Raft's 2f+1 replicas and no cryptography at all beyond transport-level TLS. For f = 1 (survive one faulty node), that is 4 replicas versus 3, a real but modest difference; the gap widens fast as f grows, and the cryptographic signing cost, negligible per message, becomes real aggregate CPU and latency at the message volumes section 1's numbers describe. A team pays this cost only when the actual failure model includes participants that might lie, not merely participants that might crash, and paying it when crash-fault tolerance would have sufficed is pure waste.

**HotStuff's linearity trades PBFT's simpler, un-pipelined design for materially more implementation complexity, in exchange for the throughput that makes Byzantine consensus viable past small validator counts.** PBFT's all-to-all broadcast is easier to reason about and easier to implement correctly, every replica sees every vote directly, no aggregation step to get wrong. HotStuff's threshold-signature aggregation and chained pipelining are the reason DiemBFT could target a validator set two orders of magnitude larger than classic PBFT deployments typically ran, but that throughput is bought with meaningfully harder correctness engineering, exactly the kind of complexity-for-throughput trade Day 21's LSM-tree compaction and Day 27's TrueTime uncertainty-interval math both make elsewhere in this ledger.

**Consistency-per-trust-boundary is the practical resolution: Byzantine tolerance belongs at the edge where trust actually ends, not uniformly across an entire system.** A company's own internal replica set, Rare.lab's Postgres primary and its followers, a Raft-managed configuration store, is a crash-fault domain: every node runs code the company controls, on infrastructure the company operates, and paying Byzantine consensus's extra replica and cryptographic cost there buys protection against a threat, an internally-controlled node lying, that is not the actual risk. Byzantine tolerance earns its cost specifically at the boundary where independent, mutually distrusting parties, separate banks and payment processors validating Diem, separate organizations federating a system, must agree on shared state without any one of them being able to unilaterally corrupt it.

---

## 6. The systems-thinking lens

**The feedback loop: a stalling or dishonest leader triggers a view change, and if the view-change timeout is set too aggressively, the network can enter a cascading view-change storm where no leader ever survives long enough to make real progress, the Byzantine-protocol cousin of the retry-storm pattern Day 9 and Day 13 name for queues and backpressure.** Trace it through: a leader is slow, perhaps only network-congested, not actually malicious → replicas' view-change timers fire before that leader's proposal arrives → the network elects a new leader and discards the in-flight round → the new leader, inheriting a network that is still just as congested, is now also likely to miss its own timeout window → another view change fires. Each view change resets in-flight progress and adds its own round of leader-election messages on top of an already-congested network, so the mechanism built to route around a bad leader can itself become the reason no leader ever gets enough clear time to finish a round, a system that appears to be actively working (electing leader after leader) while making zero real progress, the same signature as a metastable failure elsewhere in this ledger.

**The senior fix is not "elect leaders faster," it is to make the timeout adapt to observed network conditions rather than firing on a fixed, worst-case-guessed clock, exactly the same move Day 13's backpressure lesson makes against a fixed retry interval.** DiemBFT's pacemaker mechanism does this directly: rather than a single fixed view-change timeout, it exponentially backs off the timeout on repeated failures (so a genuinely struggling network gets progressively more slack before the next election fires) and incorporates a **leader-reputation** signal, deprioritizing leaders that have recently timed out from being re-elected soon, so the rotation does not keep handing the gavel back to the same struggling node. Both mechanisms break the loop the same way Day 24's lock-service backoff and Day 9's queue backoff do: stop treating every failure to complete a round on schedule as a fresh, independent event, and instead let the system's recent history of failures change how aggressively it retries, which is what actually stops the storm from feeding itself.

---

## Map to Rare.lab's stack

**Where this matters, concretely, is the trust boundary of Rare.lab's multiplayer runtime, not its database.** Supabase Postgres, with its primary and read replicas, is a crash-fault domain end to end, every node runs Rare.lab's own code on Rare.lab-controlled infrastructure, so Raft-style (or Postgres's own WAL-replication) crash tolerance is exactly the right, and exactly sufficient, tool there; reaching for anything Byzantine on that path would be pure overhead against a threat that does not exist. The place this lesson's mechanisms actually become relevant is Day 15's CRDT-based multiplayer node graph, the moment a peer contributing edits to a shared scene is not a Rare.lab server the company operates, but an external client: a public or semi-public multiplayer room, a marketplace flow where a third party's compiled node bundle gets merged into a shared library, an AI agent proposing graph edits on behalf of an untrusted account. A CRDT's merge function has no concept of a lying peer, it will merge a maliciously crafted op, one engineered to corrupt a shared invariant (a node that silently rewrites another collaborator's uniform buffer binding, say) exactly as faithfully as an honest one, because convergence, not honesty, is all a CRDT was ever built to guarantee.

**The concrete, proportionate fix is not "implement HotStuff," it is to borrow the one mechanism that generalizes far below blockchain scale: require quorum agreement from multiple independent validators before an untrusted op is admitted into the shared, trusted state, rather than admitting it on one submitter's say-so.** For a community node-marketplace flow specifically, that looks like requiring N independent, automated checks, a schema/type validator, a sandboxed compile pass, a resource-cost budget check, to each sign off before a submitted node bundle is merged into the shared library, structurally the same 2f+1-style "no single party's claim is trusted alone" pattern this lesson's quorum certificates encode, just applied at the scale of a handful of independent validators rather than a hundred-node blockchain. That is a proportionate answer to Rare.lab's actual near-term ceiling: not full Byzantine state-machine replication, which would be solving a problem the architecture does not yet have, but recognizing early which trust boundary, multiplayer rooms and marketplace submissions, is the one that will eventually need it, and building the "more than one independent party must agree before this is trusted" habit into that boundary now, before it is load-bearing.

---

## References and summaries

**Castro, Liskov: "Practical Byzantine Fault Tolerance"** (OSDI 1999)
http://pmg.csail.mit.edu/papers/osdi99.pdf
The original paper making Byzantine consensus fast enough for real systems: the 3f+1 replica requirement, the pre-prepare/prepare/commit three-phase protocol this lesson's section 3 contrasts against HotStuff, and the view-change mechanism for replacing a faulty primary. The source of this lesson's O(n²) all-to-all broadcast cost, both the prepare and commit phases are described here as full replica-to-replica broadcasts.

**Yin, Malkhi, Reiter, Golan Gueta, Abraham: "HotStuff: BFT Consensus with Linearity and Responsiveness"** (ACM PODC 2019)
https://arxiv.org/abs/1803.05069
The protocol behind this lesson's sections 3 and 4: leader-relayed voting, threshold-signature quorum certificates, and chained pipelining, the combination that turns PBFT's O(n²) message cost into O(n) while keeping responsiveness, running at actual network speed rather than a fixed worst-case timeout. The paper explicitly frames PBFT's quadratic broadcast as the motivating bottleneck.

**The Diem Team: "DiemBFT v4: State Machine Replication in the Diem Blockchain"**
https://developers.diem.com/papers/diem-consensus-state-machine-replication-in-the-diem-blockchain/2021-08-17.pdf
The production adaptation of HotStuff running Meta's Diem (formerly Libra) blockchain, source for this lesson's section 6 pacemaker and leader-reputation mechanisms, the exponential-backoff view-change timeout and the deprioritization of recently-failed leaders that break the view-change storm feedback loop.

**Diem Association: "The Libra consensus protocol: Today and next steps"**
https://www.diem.com/en-us/blog/libra-consensus-protocol/
The plain-language account of why the Libra/Diem team moved from evaluating classic PBFT-style approaches to adopting and productionizing HotStuff, framed around the validator-count growth the project needed to support, the real-world business motivation behind section 1's breaking number.

**Trail of Bits: "On LibraBFT's use of broadcasts"**
https://blog.trailofbits.com/2019/07/12/librabft/
An independent security firm's technical review of LibraBFT's (DiemBFT's predecessor name) actual broadcast and network-layer behavior in production code, useful as a second, non-Meta-authored source cross-checking how the leader-relay and quorum-certificate mechanisms this lesson describes actually behave outside the whitepaper's idealized description.
