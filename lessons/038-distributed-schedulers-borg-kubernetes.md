# Day 38 — How does Google pack hundreds of thousands of jobs onto tens of thousands of machines without wasting a datacenter's worth of capacity?

*2026-07-22*

---

## 1. The company and the number that breaks a naive design

**Google, Borg.** Borg is the cluster manager that has sat underneath essentially all of Google's production and batch workloads since roughly 2004, and it is the direct ancestor of Kubernetes (several of Kubernetes's original authors were Borg and its successor Omega's engineers). The 2015 EuroSys paper "Large-scale cluster management at Google with Borg" (Verma, Pedrosa, Korupolu, Oppenheimer, Tune, Wilkes) describes a system that runs hundreds of thousands of jobs, from many thousands of different applications, across clusters ("cells") of **up to tens of thousands of machines**, with a single logically-centralized controller, the Borgmaster, managing each cell.

The number that breaks a naive scheduler is not the machine count by itself, it's the collision of machine count with request rate: the paper reports that **several cells see task arrival rates above 10,000 tasks per minute**. Put those two numbers together. A scheduler that, for every incoming task, computes an exact fitness score against every one of tens of thousands of machines is doing tens or hundreds of millions of scoring computations per minute, sustained, forever, while also fielding a constant stream of machines going up and down, jobs finishing, and priority tasks that need to preempt something right now. That is the constraint a naive design cannot survive: **placement decisions have to keep pace with an arrival rate in the thousands per minute, against a machine pool in the tens of thousands, without ever falling behind.**

## 2. Why the naive design dies

The naive version: one scheduler process holds the full list of machines in memory. Every time a task is submitted, it walks the entire machine list, computes an exact best-fit score for each one (how much CPU and memory would be left over if this task landed here), and assigns the task to the single best-scoring machine. This collapses in three concrete ways.

**a. The scoring cost is O(tasks × machines), and both sides of that product are growing.** At 10,000 tasks a minute against tens of thousands of machines, a full exact rescoring per task is tens of millions of comparisons a minute before the scheduler has assigned a single task. The queue of unplaced tasks grows faster than it drains. This is the same shape as a naive search engine that rescans every document for every query, matching engineering across completely different domains: exhaustive scan does not survive catalog growth, ever, in any system.

**b. No isolation between tenants means one job's burst starves or crashes its neighbors.** If every task is treated identically, a burst of low-priority batch work (say, a backfill job that some team kicked off) can consume all available capacity right as a latency-critical production service needs to scale up, and there is no mechanism to say "this one matters more, take the capacity back." Equally, packing tasks purely by their declared resource request, with no headroom, means a legitimate usage spike on one task now steals CPU or memory from whatever else landed on the same machine, a noisy-neighbor problem that a single flat scoring pass cannot see coming.

**c. A single scheduler process is also a single point of failure for placement itself.** If the one process holding "what should run where" dies, nothing new gets scheduled, and depending on design, the system may not even remember what was already running. Losing the scheduler doesn't just pause growth, it can strand the whole fleet.

The analogy: imagine a single maître d' at a 10,000-table restaurant who, for every walk-in party, personally re-measures every single table in the building to find the exact best fit, while also being the only person in the building who remembers who is currently seated where. The line at the door only ever gets longer, and if that one maître d' steps away, the whole restaurant forgets who's sitting where.

## 3. The architecture, top to bottom

```
Clients (teams submitting Jobs: batch pipelines, latency-critical
         services, per-customer periodic tasks)
   |  submit job/task spec (declared resource request + priority)
   v
API / control frontend (Borg: Borgmaster RPC frontend; Kubernetes:
                         kube-apiserver)
   |  validates the request, writes it as DESIRED state
   |  analogy: the reservations desk, takes the request, does not
   |  itself decide which table
   v
Replicated state store (Borg: Paxos-replicated store inside the
                         Borgmaster, a 5-machine Paxos group with
                         periodic checkpoints + a change log;
                         Kubernetes: etcd, Raft-replicated)
   |  the single, strongly-consistent source of truth for
   |  "what SHOULD be running, and what IS currently bound to what"
   |  master election/failover typically ~10s, up to ~1 min in a
   |  very large cell
   v
Scheduler (Borg: the scheduling loop inside Borgmaster;
           Kubernetes: kube-scheduler)
   |  pulls unplaced tasks/pods one at a time, runs FILTER
   |  (feasibility: which machines even could run this) then
   |  SCORE (rank the feasible ones), binds the task to the winner
   |  analogy: the actual matchmaking step, "which open tables
   |  fit this party AND leave the best leftover space for the
   |  next party"
   v
Node agent (Borg: Borglet, one per machine; Kubernetes: kubelet)
   |  starts/stops/monitors the actual containers assigned to
   |  this one machine, reports health and resource use back up
   |  analogy: the floor waiter who actually seats the party and
   |  radios the front desk the moment a table empties or a
   |  problem shows up
   v
Node fleet (tens of thousands of heterogeneous machines; latency-
            critical "production" tasks and lower-priority batch
            tasks deliberately co-located on the same machines)
   |
   v
Reconciliation controllers (Kubernetes: Deployment/ReplicaSet
            controllers; Borg's equivalent logic is built into
            the Borgmaster's job-management loop)
      continuously diff DESIRED state (in the store) against
      OBSERVED state (from node-agent reports) and re-issue work
      to close any gap, forever, not as a one-shot action
```

The split that matters: the replicated state store is the one tier that must be strongly consistent, because two schedulers ever believing they each independently own the same slot on the same machine is a correctness bug, not a performance problem. Everything downstream of it, the scheduler's own scoring loop, the node agents, the controllers, works off periodic reads and reports and tolerates being briefly stale, because a few seconds of staleness in "is this node still healthy" is recoverable, while a double-booked machine is not.

## 4. The transferable mechanisms

**a. Filter, then score, never one combined pass.** First cheaply throw out every machine that flatly cannot run the task (wrong architecture, insufficient free resources, an unsatisfied placement constraint), and only run the more expensive ranking logic against whatever small feasible set survives. Kubernetes's own docs describe this exact two-step pipeline, filter plugins (formerly called "predicates") eliminate infeasible nodes, score plugins (formerly "priorities") rank what's left. Cutting the search space before you spend money ranking it is the same principle as an inverted index cutting a document set before a ranker ever touches it.

**b. Equivalence classes and score caching, don't recompute what you already know.** Tasks belonging to the same job usually share identical resource shapes, so Borg computes a machine's score once per equivalence class of task, not once per individual task, and caches a machine's score until something material about that machine or task actually changes. This turns an O(tasks × machines) problem into something much closer to O(distinct shapes × machines), which is the difference between a scheduler that keeps up and one that doesn't.

**c. Sample instead of exhaustively scanning at scale.** Kubernetes's `kube-scheduler` will stop looking for feasible nodes once it has found "enough" of them for the cluster's configured `percentageOfNodesToScore`, by default scaling down from roughly 50% of nodes considered in a 100-node cluster to around 10% in a 5,000-node cluster. That is a direct, explicit trade of perfect placement for bounded, predictable per-decision latency, the scheduler intentionally stops looking rather than guaranteeing it found the single best machine in the whole fleet.

**d. Priority bands plus preemption, not just admission control.** Every task carries a priority tier (Borg documents bands from low-priority best-effort batch work up through monitoring and production/latency-sensitive tiers, with a small reserved band above even production for the most critical infrastructure). A higher-priority task can evict, preempt, a lower-priority one to free capacity it needs right now. This means overload degrades gracefully, starting with the work that matters least, instead of either turning away new latency-critical work outright or letting a batch burst silently starve production.

**e. Oversubscription plus a hybrid best-fit/worst-fit scorer.** Borg schedules against a dynamically reclaimed estimate of what a task is actually likely to use, not just its declared reservation, and blends best-fit scoring (pack tightly, minimize wasted fragments) with worst-fit/E-PVM-style scoring (spread load, leave headroom for bursts). The paper reports this hybrid beats a pure best-fit approach by several percentage points of utilization by reducing resources stranded as unusable fragments, and Google states more broadly that running mixed production and batch workloads on one shared fleet, instead of segregating them onto separate machines, avoids needing a substantially larger fleet to do the same work.

**f. Declarative desired state with a continuous reconciliation loop, not an imperative one-shot command.** You never tell Kubernetes "start this pod now" as a fire-and-forget action, you declare "this is what should exist," write it once to the strongly-consistent store, and a controller keeps comparing that declaration against reality forever, re-issuing work any time the two drift apart, whether from a crash, a node failure, or a manual change. A scheduler or agent restarting just re-syncs from the same durable declaration; no in-flight intent is ever lost because none of it lived only in a process's memory.

## 5. The trade-offs

**Consistency vs. availability, split sharply by data type.** The record of "what should be running, and what is currently bound to which machine" lives in a Paxos- or Raft-replicated store and must be strongly consistent, two schedulers disagreeing about who owns a given slot on a given machine is a silent double-placement bug, not a tolerable inconsistency. Everything downstream, node health reports, resource-usage numbers used for oversubscription, controller reconciliation state, is allowed to be eventually consistent: a health check that's a few seconds stale just means a slightly slower reaction to a real failure, corrected on the next report, never a corrupted placement decision. This is a genuinely CP choice for the ownership ledger specifically: if the replicated store can't reach quorum, Borg stops issuing new placement decisions rather than risk two replicas independently double-booking a machine, while already-running tasks keep running untouched, since node agents don't need quorum to keep serving what they were already told to run.

**Cost vs. latency, paid twice over.** Sampling nodes instead of scoring all of them (`percentageOfNodesToScore`) trades a small, bounded chance of missing the single best machine for scheduling latency that stays flat as the cluster grows, without it, a 5,000-node cluster would score 10x more candidates per pod than a 500-node one for every single placement decision. Separately, oversubscription trades a managed, monitored risk of resource contention for real utilization dollars, deliberately not leaving the same safety margin on every machine that a fully conservative, reservation-only scheduler would.

## 6. The systems-thinking lens

**The feedback loop here is retry-driven overload at the scheduling layer, a metastable failure.** Trace it: the scheduler falls behind its arrival rate, even briefly, during a burst → submitted tasks sit visibly unscheduled for longer than usual → anxious client-side automation or an on-call engineer resubmits or duplicates the request, assuming it was lost → those resubmissions land as new entries in the same already-backed-up queue → the queue is now deeper than before the retries, so the scheduler falls further behind → more requests now look "stuck," triggering more retries. Once triggered, this does not recover on its own even after the original burst passes, the retry traffic alone is enough to keep the queue saturated, the same self-sustaining shape as a cache-stampede or a thundering herd.

**The senior fix bounds the work per cycle and makes retries harmless, it does not just add more scheduler capacity.** Equivalence-class scoring and percentage-based node sampling cap the work done per scheduling decision so throughput stays roughly constant regardless of how deep the backlog gets, instead of degrading further the more overloaded the system already is. Priority bands mean a flood of low-priority resubmissions cannot compete evenly with fresh high-priority work for the scheduler's attention. And idempotent, declarative submission, the same job spec resubmitted twice is recognized as the same desired-state entry, not two, means an anxious retry storm cannot multiply queue depth in the first place. Backpressure applied at the point where load enters the system, not just more machines thrown at the queue after the fact, is what actually breaks the loop.

---

## References and summaries

**Verma, A., Pedrosa, L., Korupolu, M., Oppenheimer, D., Tune, E., Wilkes, J. "Large-scale cluster management at Google with Borg." EuroSys 2015.**
https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/ (paper listing) and https://research.google.com/pubs/archive/43438.pdf (PDF)
The primary source for this lesson. Direct PDF fetch returned HTTP 403 in this session (the fetch tool was blocked broadly this run, the same limitation noted in Day 36 and Day 37's references). The figures used here, cells of up to tens of thousands of machines, several cells with arrival rates above 10,000 tasks per minute, the Borgmaster as a 5-machine Paxos group with periodic checkpoints and a change log, master failover typically ~10 seconds and up to about a minute in a large cell, and the hybrid best-fit/worst-fit (E-PVM) scoring approach yielding several percentage points of utilization gain over pure best-fit, were cross-corroborated across multiple independent secondary sources quoting the paper directly (below), consistent with each other and with the paper's own abstract as indexed by Google Research and ACM.

**ACM Digital Library. "Large-scale cluster management at Google with Borg."**
https://dl.acm.org/doi/10.1145/2741948.2741964
Publisher record confirming venue (EuroSys 2015) and author list, used to verify the primary citation.

**Adrian Colyer, "the morning paper." "Large-scale cluster management at Google with Borg."**
https://blog.acolyer.org/2015/05/07/large-scale-cluster-management-at-google-with-borg/
A well-known, widely-cited academic paper-summary blog; direct fetch also returned HTTP 403 this session, content reached only via cross-corroborated search-result snippets, used as secondary confirmation of the equivalence-class scoring, score-caching, and hybrid-scoring mechanisms described above.

**Google Research. "Omega: flexible, scalable schedulers for large compute clusters."**
https://research.google.com/pubs/archive/41684.pdf
Google's own 2013 follow-up paper, describing why Borg's original single monolithic scheduling loop needed to evolve toward a shared-state architecture with optimistic concurrency control as scale and the diversity of scheduling policies grew, the direct intellectual bridge between Borg and Kubernetes's own pluggable scheduler design.

**Kubernetes documentation. "Scheduler Performance Tuning" and "Scheduler Configuration."**
https://kubernetes.io/docs/concepts/scheduling-eviction/scheduler-perf-tuning/ and https://kubernetes.io/docs/reference/scheduling/config/
Primary source for `percentageOfNodesToScore`: the scheduler can stop searching once it has found enough feasible nodes for the configured percentage, with a compiled-in default that scales down from roughly 50% of nodes at small cluster sizes to around 10% at 5,000 nodes, trading placement optimality for bounded per-decision latency.

**Kubernetes documentation / KEP. "Scheduling Framework" (kubernetes/enhancements, sig-scheduling, KEP-624).**
https://github.com/kubernetes/enhancements/blob/master/keps/sig-scheduling/624-scheduling-framework/README.md
Primary source for the Filter -> Score -> Bind pipeline and the full extension-point list (PreFilter, Filter, PostFilter, PreScore, Score, NormalizeScore, Reserve, Permit, PreBind, Bind, PostBind, Unreserve), and for the mapping from the older "predicates"/"priorities" terminology to today's Filter/Score plugins.

**Kubernetes Blog. "1000 nodes and beyond: updates to Kubernetes performance and scalability in 1.2" and "Scalability updates in Kubernetes 1.6."**
https://kubernetes.io/blog/2016/03/1000-nodes-and-beyond-updates-to-kubernetes-performance-and-scalability-in-12/ and https://kubernetes.io/blog/2017/03/scalability-updates-in-kubernetes-1-6/
Kubernetes project's own scalability benchmarking posts, source for the 5,000-node cluster SLO target and the 5-10x scheduler throughput improvement delivered in the 1.6 release, used as an independent, non-Borg confirmation that exhaustive per-pod node scanning does not survive growth into the thousands-of-nodes range without deliberate optimization.

**OpenAI. "Scaling Kubernetes to 7,500 Nodes."**
https://openai.com/index/scaling-kubernetes-to-7500-nodes/
A real-world production account of running Kubernetes clusters at Borg-comparable node counts for large machine-learning training workloads, including OpenAI's own team-resource-manager for capacity reservation and their use of coscheduling to place all-or-nothing multi-pod training jobs, independent, modern corroboration that priority- and reservation-aware scheduling becomes necessary once cluster size and contention both grow.

**PayPal Technology Blog. "Scaling Kubernetes to Over 4k Nodes and 200k Pods."**
https://medium.com/paypal-tech/scaling-kubernetes-to-over-4k-nodes-and-200k-pods-29988fad6ed
An independent production account (outside Google and OpenAI) of the same scheduler-and-etcd throughput tuning problem at large node and pod counts, used as further cross-company corroboration.

---

## Map to Rare.lab's stack

Rare.lab doesn't run tens of thousands of machines, but the exact same placement problem already exists in miniature, in two places, and it will only get sharper as usage grows.

The first is the compile-and-render pipeline itself. Every time a user hits "compile" on a node graph or requests a preview render, that's a task that needs to land on some worker with spare capacity, today, at small scale, running each request as an independent, standalone job is the right call, the same way Borg's own naive single-process version would have been fine at a handful of machines. The ceiling arrives the moment concurrent demand creates a real arrival-rate spike, a workshop, a feature launch, a batch of users all hitting compile within the same few minutes, which is precisely the "10,000 tasks a minute against a fixed pool" shape this lesson describes, just at Rare.lab's own scale rather than Google's.

The second is the embeddable runtime's single shared WebGL context. That context is itself a scarce, oversubscribed resource shared across every active node-graph preview and shader running in one page, structurally the same problem as many tasks competing for a fixed pool of machines, just compressed down to one process's frame budget instead of a datacenter.

The concrete, actionable move, before either ceiling is hit rather than after: give both the compile/render pipeline and the shared WebGL context an explicit priority tier now, interactive preview and publish-path compiles as a "production" tier that always gets capacity first, batch work like precomputing device-capability variants or nightly cache-warming as a "best-effort" tier that yields the moment interactive demand shows up, exactly Borg's priority-preemption pattern, applied at Rare.lab's scale. And once the worker pool or the per-frame candidate set grows past what an exhaustive best-of-everything scan can score inside its latency budget, reach for the same fix Kubernetes reaches for: score a bounded sample instead of everything, and cap it explicitly rather than letting an exhaustive scan quietly become the bottleneck no one budgeted for.
