# Day 37 — How does Uber keep a single business process alive for days, across dozens of crashes and deploys, without losing a single completed step?

*2026-07-20*

---

## 1. The company and the number that breaks a naive design

**Uber, Cadence (open sourced in 2017, now stewarded as part of the CNCF, with Temporal as the commercial fork of the same design started by the original Cadence authors).** Cadence is the workflow orchestration engine sitting underneath a huge share of Uber's backend: trip lifecycle, driver onboarding, payments reconciliation, and any process that is really a sequence of steps spread out over time, not a single request-response call. Uber's own numbers, from their engineering blog announcing Cadence 1.0 and the Cadence multi-tenant task-processing post: Cadence processes **over 12 billion workflow executions and 270 billion actions a month**, powering **more than 1,000 services** at Uber across hundreds of different teams, from T0 (most critical) down to T5. One specific use case Uber calls out directly: **100+ million parallel per-customer periodic (cron-style) workflows**, running without a separate batch-processing framework.

The number that actually breaks a naive design is not the raw volume, it's the *duration and the failure surface underneath it*. A workflow orchestration engine has to keep a business process's exact position, "I finished step 22 of 40, I am now waiting on an external vendor's webhook," durable and correct across a duration where the process running it is guaranteed to die at least once. Cadence and Temporal workflows can run for days, weeks, or months, there is documented **no upper bound on workflow timeout values**. Over a multi-day window, a fleet running continuous deploys, routine host restarts, and hardware failures will kill the specific process holding any given workflow's state, not once, but repeatedly, before that workflow finishes. That is the constraint a naive design cannot survive: **the process is going to die mid-workflow, guaranteed, and the system has to keep going anyway.**

## 2. Why the naive design dies

The naive version is the one every engineer reaches for first: write the multi-step process as ordinary code, a function with loops, sleeps, and conditionals, running in one process, holding its progress in local variables and memory. For scheduling, pair it with a `jobs` or `cron_tasks` table in a relational database that a poller scans on a timer for rows that are due. This collapses in three concrete ways.

**a. In-memory state dies the instant the host does.** If the process running step 22 of a 40-step driver-onboarding workflow is killed by a deploy, an OOM, or a hardware fault, every local variable holding "which step am I on, what did the last vendor response say" is gone. There is no way to resume from step 22. The only options are: restart from step 1, which is wrong if step 5 was "charge a card" or "send an SMS" and it is not safe to redo, or lose the workflow entirely. Neither is acceptable for a process that is supposed to represent a real business outcome.

**b. A single poller scanning a jobs table does not survive either the row count or the clustering of due times.** At 100 million simultaneous per-customer periodic workflows, a poll loop doing "`SELECT * FROM jobs WHERE next_run_at <= now()`" against one table is a full-scan bottleneck on every tick, and it is a single point of failure, if that poller process dies, nothing fires until it is restarted. Worse, due times cluster: any team that picks a round number like midnight or the top of the hour for its cron schedule creates a synchronized spike of "due right now" rows that all land on the same database at the same instant, the classic thundering herd.

**c. A long wait ties up a resource for the whole wait, which does not scale to millions of concurrent waits.** A step like "wait up to 7 days for a background-check vendor to call us back" either blocks a thread or process for up to 7 days (which does not scale past a tiny number of concurrent workflows before the fleet runs out of threads), or forces every team to hand-roll their own polling and resumption logic around that wait, which is exactly the kind of duplicated, easy-to-get-subtly-wrong logic that produces real production bugs.

The analogy: imagine a single clerk personally holding, in their own head, every detail of every multi-day paperwork process they've started, for hundreds of customers at once, with no notebook. The moment that clerk goes home sick, every single one of those in-progress processes is gone, no way to tell where any of them had gotten to, and the next clerk who shows up has to start every single one of them over from scratch, including re-mailing letters that already went out.

## 3. The architecture, top to bottom

```
Clients (services starting/signaling/querying workflows: trip lifecycle,
         driver onboarding, payments reconciliation, per-customer cron)
   |  StartWorkflow / SignalWorkflow / QueryWorkflow calls
   v
Frontend service (stateless gateway tier)
   |  auth, rate limiting, request routing; holds no workflow state itself
   |  scales by adding more instances, like any stateless API tier
   v
Matching service (stateless task-queue tier)
   |  holds task queues; matches a pending Workflow Task or Activity Task
   |  to an available Worker long-polling that queue
   |  analogy: a job board, workers pull work, nobody pushes onto a worker
   v
History service (the stateful "brain," partitioned into History Shards)
   |  1 to 128K shards per cluster, workflow ID hashed to a shard, shard
   |  count fixed at cluster creation; shard ownership assigned to History
   |  hosts and coordinated via a Ringpop gossip membership protocol
   |  (same primitive as Day 25's gossip-based failure detection)
   |  ALL state transitions for a workflow are serialized through the one
   |  shard that owns it, no two decisions for the same workflow ever race
   v
Persistence (Cassandra / MySQL / PostgreSQL, one DB partition per shard)
   |  this is where the Event History actually lives durably: an
   |  append-only log of every decision and every external result for
   |  that workflow, since the moment it started
   v
Workers (stateless, horizontally scaled, run YOUR code)
   |  pull Workflow Tasks: replay/advance the deterministic workflow
   |  function against the Event History to decide "what happens next"
   |  pull Activity Tasks: run the actual non-deterministic side effects
   |  (call the vendor API, charge the card, send the SMS), at least once,
   |  retried on failure with a platform-default exponential backoff
   |  (1s initial interval, 2.0x coefficient, capped at 100s)
   v
Visibility store (Elasticsearch)
   |  indexes workflow metadata for search/list/debug UIs; not on the
   |  durability critical path
```

The split that matters: the History service is the only stateful, strongly-consistent tier, and it is strongly consistent *per shard*, not globally. Everything else, frontend, matching, workers, is stateless and scales by adding machines with no coordination required.

## 4. The transferable mechanisms

**a. Event sourcing as the durability primitive, not checkpointing raw memory.** Instead of trying to snapshot a process's live memory (fragile, language-specific, huge), the platform durably appends every decision and every external result to an ordered Event History. Recovery is not "restore a memory dump," it is "run the same deterministic code again and let it reach the same state by replaying the same inputs." This is the same idea as a database's write-ahead log from Day 17, applied to arbitrary business logic instead of row mutations.

**b. Determinism is the price of durability, and it has to be enforced by convention, not by the type system.** Workflow code must be a pure function of its inputs and its Event History: same inputs, same history, same decisions, every time it is replayed. That means no direct calls to wall-clock time, no random numbers, no unordered map iteration, and no direct network or database calls inside workflow code, those go into separate, explicitly non-deterministic **Activities**, which are the only place side effects are allowed. Split the parts of your system that must be replayable from the parts that are allowed to be messy and non-repeatable, and never let the messy part leak into the replayable part.

**c. Split the durable "brain" from the stateless "muscle."** One shard, one owner, fully serialized, holds the small amount of state that must never race with itself, exactly the workflow's current position. Everything that actually does work, the Workers running Activities, is stateless and horizontally scalable, because none of it needs to agree with anything else. This is the same leader/follower split as Day 36's TAO cache tiers and Day 22's leaderless replication, applied to compute instead of storage.

**d. Amortize the cost of durability with a warm cache, not by skipping the durability.** Sticky Execution keeps a workflow's live state cached in the specific Worker process that last handled it, so the next decision does not require a full replay from persisted history, only the new events since the cache was warmed. If that worker dies, the cache is simply gone and the next worker rebuilds it from the real source of truth, the persisted Event History. The cache is a performance optimization on top of a durability guarantee that does not depend on it.

**e. At-least-once execution plus idempotency keys make retries safe, the same primitive as Day 12.** Activities are retried on failure with platform-default exponential backoff (1s, doubling, capped at 100s) until they succeed or exhaust their policy. That only works because the actual side effect, charging a card, sending a message, is expected to be idempotent or de-duplicated on the receiving end, the orchestration layer's retry guarantee and the idempotency-key discipline from Day 12 are two halves of the same mechanism.

**f. Cap the unbounded, don't just monitor it.** A workflow that runs forever (the 100-million-instance per-customer cron case) would otherwise let its Event History grow without bound. The platform enforces a hard ceiling, documented at roughly 51,200 events or 50 MB per workflow execution, past which the workflow is terminated outright rather than allowed to slowly degrade everyone sharing its shard.

## 5. The trade-offs

**Consistency vs. availability, split by scope.** Within a single workflow, on its one shard, the system is strongly consistent: state transitions are strictly serialized, there is exactly one current position, and it can never be raced or double-applied. Across the fleet, the system is fully available and horizontally scalable, because different workflows never need to agree with each other, adding shards and machines adds capacity without anyone waiting on anyone else. This is the right split: the thing that must never be wrong (this one workflow's own history) is narrow and cheap to keep strict; the thing that must scale to billions (the whole fleet) is exactly the part that does not need cross-workflow agreement.

**Cost vs. latency, twice over.** More History Shards means less lock contention and higher aggregate throughput, at the cost of more CPU and memory overhead per History host and more pressure on the persistence layer, that trade-off is fixed once at cluster creation and cannot be changed later without standing up a new cluster. Separately, Sticky Execution trades Worker memory (holding a workflow's state warm) for avoiding a full history replay on every single decision, a real and continuous cost paid to keep decision latency low.

**Power vs. a sharp, easy-to-violate constraint.** Writing an ordinary-looking function with loops and sleeps that is secretly fully durable is a much easier programming model than hand-writing an explicit state machine. The price is determinism: one accidental call to a wall-clock or a random number generator inside workflow code silently breaks replay, and the failure mode is not a compile error, it is a workflow that cannot recover after a crash. Same shape of trade as CRDTs from Day 15 or TrueTime commit-wait from Day 27, a powerful abstraction that leaks under one specific, easy-to-forget constraint.

## 6. The systems-thinking lens

**The feedback loop here is a replay storm, and it is self-reinforcing.** Trace it: a long-running or effectively-infinite workflow keeps appending events to its own Event History → every decision this workflow makes requires replaying that entire history from the start to reconstruct current state → as the history grows, each replay takes longer → if a replay takes too long, the Workflow Task times out before it can finish → a timed-out task is retried, and the retry itself becomes new events appended to the same already-oversized history, with zero net forward progress made → the history is now even bigger than before the retry, so the next replay is slower still. Left alone, this does not recover on its own, it gets strictly worse with every cycle, and it is the same shape as the retry-adds-more-load spiral in Day 13's backpressure lesson and Day 36's cache-invalidation stampede, except here the trigger is a workflow's own success at staying alive, not an external traffic spike.

**The senior fix is not a bigger Worker or more memory, it is resetting the loop's own input.** Continue-As-New closes out the current run and atomically starts a fresh one under the same Workflow ID with a brand-new, empty Event History, carrying forward only the small amount of state that actually matters as new input. Replay cost per decision is now bounded by design, not managed after the fact, because there is never an unboundedly large history to replay in the first place. The hard 51,200-event ceiling is a backstop that kills a workflow before it can hurt its neighbors on the same shard; Continue-As-New is the actual fix, applied proactively at a defined checkpoint, that means the backstop is never needed at all.

---

## References and summaries

**Uber Engineering Blog. "Announcing Cadence 1.0: The Powerful Workflow Platform Built for Scale and Reliability."**
https://www.uber.com/en-US/blog/announcing-cadence/
Direct fetch of this page returned HTTP 403 in this session (the fetch tool was blocked broadly this run, the same failure mode noted in Day 36's references, not specific to this host). The figures used here, over 12 billion executions and 270 billion actions a month, 1,000+ services, hundreds of teams, and the 100+ million parallel per-customer cron workflow use case, were cross-corroborated across multiple independent search results quoting the post directly and are consistent with each other.

**Uber Engineering Blog. "Conducting Better Business with Uber's Open Source Orchestration Tool, Cadence."**
https://eng.uber.com/open-source-orchestration-tool-cadence-overview/
Uber's own overview describing Cadence as a fault-tolerant, horizontally-scalable, multi-tenant orchestration framework that manages distributed state, retries, and failure recovery so application teams only write business logic. Source for the general framing of what problem Cadence exists to solve.

**Uber Engineering Blog. "Cadence Multi-Tenant Task Processing."**
https://www.uber.com/en-CA/blog/cadence-multi-tenant-task-processing/
Describes how workflow states move between tasks (timer, signal, activity, child-workflow) and are atomically committed with state transitions, and how the system scales horizontally to serve many customers from shared infrastructure.

**Temporal documentation. "Temporal Server" and "History Service" architecture.**
https://docs.temporal.io/temporal-service/temporal-server and https://github.com/temporalio/temporal/blob/main/docs/architecture/history-service.md
Source for the History Shard model: 1 to 128K shards per cluster, workflow ID hashed to a shard for routing, shard count fixed at cluster creation, ShardController coordinating ownership via the Ringpop gossip membership library, and each shard mapping to one persistence partition.

**Mikhail Shilkov. "Choosing the Number of Shards in Temporal History Service."**
https://mikhail.io/2021/05/choose-the-number-of-shards-in-temporal-history-service/
Independent deep-dive corroborating that more shards reduce lock contention but increase per-host CPU/memory overhead and persistence-layer pressure, and that a single shard's throughput is bounded by its own database operation latency since updates within a shard are serialized.

**PlanetScale Engineering Blog. "Temporal workflows at scale, Part 2: Sharding in production."**
https://planetscale.com/blog/temporal-workflows-at-scale-sharding-in-production
Independent production account of the same shard-ownership and scaling model, used as secondary corroboration.

**Temporal documentation. "Workflow Execution overview," "Event History walkthrough," and community forum discussion on determinism.**
https://docs.temporal.io/workflow-execution, https://docs.temporal.io/encyclopedia/event-history/event-history-go, https://community.temporal.io/t/what-is-missing-in-my-understanding-of-determinism-and-event-history-of-a-workflow-receiving-signals/16565
Source for the replay mechanism: a Worker re-runs workflow code to produce a sequence of Commands, compared against the recorded Event History; a mismatch means non-determinism was introduced and replay cannot continue. Also source for the workflow/activity split, deterministic workflow code versus arbitrary-code activities.

**Temporal documentation and blog. "Continue-As-New," "Events and Event History," and "Managing very long-running Workflows with Temporal."**
https://docs.temporal.io/workflow-execution/continue-as-new, https://docs.temporal.io/workflow-execution/event, https://temporal.io/blog/very-long-running-workflows
Source for the unbounded-history problem and its fix: replaying a very large history costs network IO, memory, and CPU; a documented hard limit of roughly 51,200 events or 50 MB per workflow execution triggers termination; Continue-As-New atomically closes the current run and starts a fresh one under the same Workflow ID with an empty history, described in Temporal's own docs as the correct long-running-workflow pattern, not a workaround.

**Temporal documentation. "What is a Temporal Retry Policy?" and "Sticky Execution."**
https://docs.temporal.io/encyclopedia/retry-policies, https://docs.temporal.io/sticky-execution
Source for the default Activity retry policy (1 second initial interval, 2.0x backoff coefficient, capped at 100 seconds) and for Sticky Execution, directing tasks back to the Worker that already has a workflow's state warm in memory to avoid replaying from persistence on every decision.

**Google. Site Reliability Engineering book, Chapter 24: "Distributed Periodic Scheduling with Cron."**
https://sre.google/sre-book/distributed-periodic-scheduling/
An independent, non-Cadence account of the same underlying problem, keeping a scheduling decision durable and correct despite host failure, at Google, using Paxos-replicated state and Chubby-based leader election instead of an event-sourced workflow engine. Used here as a cross-company confirmation that "a naive single poller against one table" is a recognized, previously-solved failure mode, not one specific to Uber's stack, and that leader election with fast failover (Google's cron aims for sub-one-minute failover) is the older, narrower-scope cousin of the sharded History-service model described above.

---

## Map to Rare.lab's stack

Rare.lab's "publish a shader" path is, structurally, exactly this kind of multi-step business process: compile the node graph to shippable code, render a preview thumbnail, upload the compiled artifact and any textures to Cloudflare R2 under their content-addressed keys, then commit the new manifest pointing at those keys. At current scale, running that whole sequence inside one Supabase Edge Function invocation is the right call, the same way a single in-memory process was the right call for Uber before workflows started spanning days and the fleet started deploying underneath them.

The ceiling arrives the same way it did in section 2. A compile-and-publish pipeline that starts doing real work, waiting on a slower render step, retrying a flaky R2 upload, or eventually waiting on an external review or moderation step before a shader goes public, stops fitting inside one function invocation's timeout. Worse, if a naive version of that pipeline crashes partway, say the R2 upload of the compiled artifact succeeds but the Postgres manifest commit fails, the system is left in a state that is neither the old published version nor the new one, with nothing recording exactly which steps actually finished.

The concrete, actionable move: the moment publish becomes multi-step and can span a retry or a wait, stop modeling it as one function that must complete atomically, and model it as a durable workflow with named, independently-retryable steps, compile, render, upload-to-R2, commit-manifest, each idempotent (R2's content-addressing already makes re-uploading the same bytes to the same key a safe no-op, which is most of the idempotency work already done for that one step). Cheaply, that can mean a Postgres-backed state machine keyed by publish-attempt ID under the existing RLS model, recording which step last completed so a retried publish resumes instead of restarting; if the publish pipeline grows more steps or longer waits than that can comfortably track, that is the specific, concrete signal to reach for a real durable-execution engine like Temporal rather than growing the hand-rolled state machine further. Either way, the design principle from this lesson is the one to build against from day one: a publish that is 60% done should be resumable, never silently lost or silently half-applied.
