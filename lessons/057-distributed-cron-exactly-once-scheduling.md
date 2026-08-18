# Day 57: How do you fire a scheduled job on exactly one machine, exactly once, when your service runs 50 identical replicas and any of them could be the one awake at 3am?

**Date:** 2026-08-18
**Difficulty:** Expert
**Topic:** Distributed cron and exactly-once scheduled execution. Why a single crontab entry, the thing every engineer reaches for first, is both a single point of failure (the one box dies, the job silently never fires again) and, the moment you "fix" that by running the same crontab on every replica of a horizontally-scaled service, a duplicate-execution machine (N replicas, N identical firings). How real systems (Kubernetes' CronJob controller, Airbnb's Chronos, Quartz's JDBC-clustered scheduler, the ShedLock library) solve this with leader election, distributed locks, and idempotency keys instead of pretending the problem away, and why even Kubernetes' own official docs admit they only guarantee "at least once," not "exactly once."
**Stack relevance:** Rare.lab will eventually need background jobs that are not driven by a user request: garbage-collecting orphaned objects in the content-addressed R2 store, rolling up compute-seconds for billing, regenerating cached thumbnails. The moment any of those workers scale past a single replica, which is the entire point of running them on Cloudflare's or any other elastic compute layer, the naive "just add a cron trigger" approach silently becomes either a missed job or a duplicated one, and which failure mode you can tolerate depends entirely on what the job touches.

---

## 1. The company and the breaking number

**CircleCI**, December 2, 2025, 16:20 UTC. A code change shipped to the service that computes workflow status introduced a race condition in how the system decided a workflow had finished. When concurrent jobs inside a workflow terminated close together, the platform sometimes published the same "workflow completed" event twice instead of once. For most customers that duplicate event was harmless noise. For customers who had CircleCI's auto-rerun feature turned on (automatically re-running a workflow when it fails), each duplicate completion event was treated as a fresh trigger. The scheduler didn't know it had already reacted to this exact completion a few milliseconds earlier, so it reacted again.

The breaking number: over the roughly 22 hours the race condition was live, before it was fully resolved at 22:50 UTC on December 3, this produced **33,135 duplicate workflow runs across 38 customer projects**, with **18 organizations** receiving duplicate email notifications for work their systems had, correctly, only run once. Nobody's individual workflow config was wrong. Nobody's job definition had a bug. The failure was entirely in the layer that is supposed to answer one narrow question correctly, every single time: *has this specific trigger already been acted on, or hasn't it?*

That is the whole problem this lesson is about, stated as plainly as it gets: a scheduled or triggered action fired more than once because the system had no durable, load-bearing answer to "have I already done this." The CircleCI incident is the expensive, real-world version. The cheap, everyday version of the exact same bug is one almost every backend engineer has shipped by accident: a service deployed with 50 replicas for uptime, a completely ordinary and correct choice, with a `* * * * *` crontab entry for "send the daily digest email" baked into the same container image. Midnight arrives. All 50 replicas are alive, because that is the entire point of running 50 replicas. All 50 fire the same cron line. Every user on the platform gets the same digest email 50 times, not once, and the bug report that comes back doesn't say "race condition," it says "why did I get 50 emails."

---

## 2. Why the naive (demo) design dies

**Version one: a single box runs `crontab -e` and that's the whole scheduler.** This is the honest starting point, and for a personal project or a single-tenant internal tool it is completely fine. It has exactly one failure mode, but that failure mode is silent and total: the box reboots for a kernel patch, or the container gets rescheduled onto different hardware, or someone `docker stop`s the wrong container during a deploy, and every job that was supposed to fire on that box simply does not fire. There is no error. There is no stack trace. There is no alert, because nothing threw an exception, the box that would have thrown it isn't running. The invoice that was supposed to go out at midnight doesn't go out, and the first anyone hears about it is a support ticket three days later asking where the invoice is.

**Version two, the "fix" everyone reaches for: run the exact same service, with the exact same crontab, on every replica.** This is what almost every engineer does the first time they hit version one's failure, because it's the same instinct that fixes every other single-point-of-failure problem in a stateless HTTP service: add replicas. It is exactly the wrong fix here, because a cron schedule isn't a request that needs any one server to answer it, it's a clock tick that every identical replica sees at the same wall-clock moment. Fifty replicas don't share the work of firing the job once. They each independently, correctly, faithfully fire their own copy of it. A background-jobs write-up describing exactly this pattern put it plainly: replicas can "wake up at the exact same time, query the database, find the same pending jobs, and process them in parallel," and the result is duplicate emails to users and a database CPU spike from fifty simultaneous identical queries, not resilience. The fix for a stateless request-handling problem made a stateful scheduling problem worse, not better.

**Version three, the grown-up naive design: hand the whole problem to your orchestrator's built-in scheduler (Kubernetes' `CronJob` resource).** This genuinely fixes the "50 identical firings" problem, a `CronJob` object creates one `Job` per scheduled tick, not one per replica of your `Deployment`. But it does not fix the underlying problem, it narrows it, and Kubernetes' own documentation says so directly: CronJobs are documented as providing **"at least once"** execution, not exactly once, and the docs go further to state plainly that "jobs should be idempotent" as a design requirement for anyone using the feature, not a nice-to-have. The controller itself can still create two `Job` objects for one scheduled time under real conditions (a controller restart mid-tick, clock skew, a slow API server), and it caps how far back it will try to catch up: if the controller has been down long enough to miss more than **100** scheduled firings, it gives up on the backlog entirely and logs an error rather than trying to fire all of them at once. That cap isn't a bug, it's a deliberate admission that unbounded catch-up is its own failure mode, covered in Section 6, but it means the "grown-up" naive design still leaves you holding two open problems: duplicates are possible, and a long enough outage means firings are simply, permanently lost.

---

## 3. The architecture

```
[Wall clock / schedule definition]
   "run this job at 00:00 UTC daily" -- a cron expression or an
   ISO8601 repeating interval, stored durably, not just in memory
   on whichever box happens to be running
   analogy: the printed timetable pinned to the station wall, not
   a memory one train conductor happens to be carrying around

   |
   v

[Leader election / distributed lock layer]
   at the scheduled moment, every eligible node ASKS to be the one
   that fires this job, but only ONE of them is granted the right
   to actually do it -- Quartz's JDBC clustering does this with a
   SELECT ... FOR UPDATE row lock on a shared QRTZ_LOCKS table:
   whichever node's transaction gets the row lock first is the
   node that fires, every other node's attempt blocks, sees the
   trigger already claimed, and backs off
   job: turn "N nodes all noticed it's time" into "exactly one
   node is allowed to act on it"
   analogy: fifty people hear the same alarm bell, but only the
   one who physically grabs the fire extinguisher first gets to
   use it, everyone else stands down the instant they see someone
   already has it

   |
   v

[Idempotency check: (job_id, scheduled_fire_time) as a unique key]
   before doing the actual work, the winning node checks a durable
   record: "has (this job, this specific scheduled time) already
   been marked done?" -- if yes, stop, even though this node won
   the lock, someone already finished this exact firing
   job: make a SECOND line of defense that doesn't depend on the
   lock being perfect, a lock can still theoretically be granted
   twice under partition or clock-skew edge cases, the idempotency
   key is what makes a rare double-grant harmless instead of a
   duplicate charge or a duplicate email
   analogy: a wedding guest list checked at the door even though
   there's already a bouncer, belt AND suspenders

   |
   v

[Durable job queue, decoupled from the firing decision]
   the winning, deduplicated firing doesn't run the actual work
   inline, it enqueues one message: "execute job X, scheduled for
   time T" -- onto a queue that survives a crash between "we
   decided to fire" and "the work actually ran"
   job: separate the (fast, security-critical) decision of WHEN
   and WHETHER to fire from the (slow, failure-prone) act of doing
   the work, so a job that takes 40 minutes to run doesn't hold
   the scheduling lock for 40 minutes
   analogy: the dispatcher who decides which ambulance gets sent
   doesn't ALSO drive it, dispatching and driving fail differently
   and at different timescales

   |
   v

[Stateless worker pool, pulls from the queue]
   any available worker, regardless of which node won the lock
   above, picks up the queued message and executes the job, then
   writes "(job_id, scheduled_fire_time) = done" back to the
   durable record from the idempotency-check step
   job: let the actual work scale independently of the scheduling
   decision, and let a crashed worker's job be safely re-picked-up
   by another worker, because the durable record, not the worker's
   own memory, is the source of truth for what's done

   |
   v

[Dead man's switch: alert on ABSENCE, not just on error]
   a separate, simple watcher checks: "did (job_id, its expected
   time) get marked done within some grace window after it was
   supposed to run?" -- if not, page someone, even though nothing
   ever threw an exception anywhere in the pipeline above
   job: catch the silent-skip failure mode from Section 2, version
   one, which by definition produces no error for a normal
   monitoring system to catch
   analogy: a security guard who is paid to notice the 2am check-in
   call that DIDN'T come, not just the ones that reported trouble
```

---

## 4. The transferable mechanisms

- **Leader election / distributed lock for "exactly one actor acts."** Whether it's Quartz's `SELECT ... FOR UPDATE` on a shared database row, ShedLock's lightweight lock table (backed by your choice of JDBC, Redis, MongoDB, or ZooKeeper, explicitly built to do only this one job rather than be a full scheduler), or Airbnb's Chronos running its scheduling decisions through Mesos's own leader-elected master, the mechanism is the same shape every time: many identical nodes see the same trigger at the same moment, and a lock, not a coin flip and not a fixed "node 1 always wins" rule, decides which one gets to act. The general form: whenever multiple independent, identically-configured processes could all correctly decide to do the same thing at the same time, that's the signal you need a lock, not more replicas.

- **Idempotency key scoped to (what, when), not just (what).** A billing job isn't idempotent because "charge the customer" is safe to call twice, it's idempotent because you key the operation on `(job_id, scheduled_fire_time)` and refuse to execute a key that's already recorded done, the same pattern Day 12's lesson on Stripe's idempotency keys covers for payment retries. The addition specific to scheduling is the second half of the key: the *time slot* matters as much as the job, because the same `job_id` legitimately needs to run again tomorrow, you're not deduplicating the job forever, only this one firing of it.

- **Decouple the scheduling decision from the work itself.** The lock only needs to be held for the microseconds it takes to write "I'm firing this" to a queue, not for the 40 minutes it might take a heavy batch job to actually finish. This is the same "keep the hot, contended path short" discipline Day 24's fencing-tokens lesson applies to distributed locks generally: hold the lock for the minimum possible time, hand the actual work to something that can fail and retry independently of the lock.

- **Bounded catch-up, not unbounded replay.** Kubernetes' CronJob controller capping its catch-up at 100 missed schedules, and requiring an explicit `startingDeadlineSeconds` if you want anything less conservative, is a deliberate refusal to let "the scheduler was down for a while" turn into "fire three days of missed jobs all at once the second it comes back." An outage in the scheduler should not become a thundering herd the moment the scheduler recovers.

- **Dead man's switch monitoring: alert on missing events, not just failing ones.** A job that silently never fires produces zero signal for a monitoring system built only to watch for errors and non-2xx responses. The fix is a separate check whose entire job is confirming presence: "this heartbeat, this completion record, this row, should exist by now, and it doesn't." This is the same shape as a system watching for a driver who's gone quiet mid-ride, or a service that's stopped emitting metrics entirely rather than emitting bad ones, the absence itself is the signal.

---

## 5. The trade-offs

**At-least-once vs. at-most-once vs. effectively-exactly-once, and the right answer depends entirely on what the job touches, not on some universal best practice.** A cache-warming job that runs twice by accident wastes a little compute and nothing else, at-least-once with no lock at all is a completely reasonable choice there, the naive "every replica fires it" design from Section 2 is actually fine if the only cost of duplication is a few wasted CPU cycles. A billing rollup or a "charge the card" job cannot tolerate at-least-once without an idempotency key underneath it, because "ran twice" there means "charged twice," a real, refundable, trust-damaging error. There is no version of distributed scheduling that gives you true exactly-once for free, what these systems actually deliver is at-least-once firing plus an idempotency key that makes the *observable effect* exactly-once even when the underlying mechanism occasionally fires twice. Chasing literal exactly-once at the firing layer, rather than accepting at-least-once and making the work idempotent, is generally the more expensive and more fragile choice.

**Availability of the schedule vs. strict correctness of every single tick.** Chronos and Kubernetes' CronJob controller both choose to keep trying rather than halt: a node that can't currently reach the lock, or a controller that just came back from an outage, keeps attempting to schedule rather than refusing to run anything until some stronger consistency guarantee is satisfied. That's a deliberate CP-vs-AP-style choice made per system, not per request: the scheduler favors staying available and occasionally over-firing (bounded, and made harmless by idempotency) over becoming unavailable and firing nothing until some quorum condition is perfectly met.

**Cost vs. latency, paid specifically in coordination infrastructure.** A single crontab entry costs nothing extra. A distributed lock, whether it's a shared database with row-level locking, a Redis instance for ShedLock, or a ZooKeeper ensemble backing Chronos's Mesos cluster, is infrastructure you run and pay for continuously, for jobs that might fire once a day. That cost is completely justified for a job whose duplicate execution is expensive (a billing run touching real money) and hard to justify for a job whose duplicate execution costs almost nothing (an internal cache refresh). Applying the heavy mechanism everywhere, out of an abundance of caution, is itself a cost with no matching benefit.

---

## 6. The systems-thinking lens

The feedback loop that actually causes damage here isn't a thundering herd in the usual sense, it's a **silent-decay loop**: a scheduled job stops firing (a box died, a lock provider became unreachable, a deploy quietly dropped the cron entry) and, because the failure mode produces no error, no stack trace, and no alert, nothing in a normal error-rate dashboard notices. The gap between "stopped firing" and "someone notices" isn't measured in seconds, it's measured in however long it takes a human to ask "wait, why didn't X happen," which can be days. By the time someone does notice, the fix that seems obvious, "just run everything that was missed," is exactly the move that turns a quiet outage into a loud one: replaying days of accumulated missed jobs all at once creates the very thundering herd that a live, healthy scheduler was designed to avoid in the first place. This is precisely why Kubernetes' CronJob controller hard-caps its own catch-up at 100 missed schedules rather than trying to be "helpful" and firing an unbounded backlog the moment it comes back online.

The senior fix isn't "add more redundancy to the scheduler," that only makes the silent-decay window statistically rarer, it doesn't change its shape. The fix is to **treat absence as its own monitored condition**, a dead man's switch that alerts specifically on "this should have happened by now and didn't," running as a check that is structurally independent from the scheduler it's watching, so a bug that takes down the scheduler doesn't also take down the thing watching for the scheduler going quiet. And pair that with a deliberately bounded catch-up policy, decided in advance, not improvised during an incident, so that recovering from an outage is a controlled, rate-limited replay instead of an unbounded flood the moment the lock comes back available. Breaking the loop means shortening the silent window and capping the recovery, not just making the underlying failure less frequent.

---

## Map to Rare.lab's stack

Rare.lab doesn't run a scheduler today in any load-bearing way, but the shape of this problem is waiting the moment any background job scales past one instance. Supabase Postgres supports `pg_cron`, which runs inside the single primary Postgres instance itself, and that's actually a quietly correct design for this exact problem: because there's only one primary, there's structurally only one place the schedule can fire from, the N-identical-replicas failure mode from Section 2 simply can't occur, the database itself is the lock. That's a reasonable, low-effort place to run something like a periodic reconciliation query today.

The ceiling shows up the moment a scheduled job needs to run somewhere that *does* scale horizontally, most plausibly a background worker fleet doing garbage collection on the content-addressed R2 store: sweeping for scene-JSON blobs no manifest references anymore and deleting them. That job is a genuinely good candidate for the cheap end of Section 5's trade-off: because objects are content-addressed, deleting an object that's already gone (because another replica's sweep already caught it) is a no-op, not a bug, so at-least-once firing with no distributed lock at all is fine there, the content-addressing itself provides the idempotency for free. The job that does NOT get to make that trade is anything that touches a usage or billing ledger, "roll up this session's compute-seconds and record it against the user's account for invoicing." The moment that job exists and runs on more than one replica, it needs exactly the machinery in Section 3: a lock so only one replica claims a given billing period's rollup, an idempotency key scoped to `(account_id, billing_period)` so a rare double-claim is still harmless, and a dead man's switch that pages someone the day the rollup silently doesn't run, rather than waiting for a customer to notice their invoice is wrong.

---

## Sources

- [Post Incident Report: December 2, 2025, CircleCI Discuss](https://discuss.circleci.com/t/post-incident-report-december-2-2025-jobs-not-starting-jobs-stuck-in-running-state-pipelines-page-not-loading/54291) and the corresponding [CircleCI status page incident](https://status.circleci.com/incidents/kvzp8x242n34): source for the root cause (a race condition introduced at 16:20 UTC on December 2, 2025 in workflow status/termination computation, causing duplicate workflow-completion events) and the incident timeline (fully resolved 22:50 UTC December 3, 2025). Direct fetch of both discuss.circleci.com and status.circleci.com was blocked by this session's network egress policy; the incident narrative and the specific figures (33,135 duplicate workflows, 38 projects, 18 organizations receiving duplicate email notifications) were relayed through a search-indexed summary of the post-incident report rather than a first-hand read of the primary document, and are worth verifying directly before citing elsewhere.
- [CronJob, Kubernetes documentation](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/): primary source for the "at least once" execution guarantee, the explicit guidance that "jobs should be idempotent," the 100-missed-schedule cap on catch-up behavior, the role of `startingDeadlineSeconds`, and the `concurrencyPolicy` (`Allow`/`Forbid`/`Replace`) options for handling overlapping runs.
- [Chronos: A Replacement for Cron, AirbnbEng, The Airbnb Tech Blog](http://nerds.airbnb.com/introducing-chronos/) and the [Chronos project repository](https://github.com/mesos/chronos): source for Chronos's original 2013 framing as a distributed, fault-tolerant scheduler built on top of Apache Mesos, its ISO8601-based scheduling, and its support for dependency graphs between jobs, built specifically to remove the single-point-of-failure and no-scalability-story problems of a single-box cron.
- [Configure Clustering with JDBC-JobStore, Quartz Scheduler documentation](https://www.quartz-scheduler.org/documentation/quartz-2.3.0/configuration/ConfigJDBCJobStoreClustering.html): primary source for the row-lock mechanism (a `SELECT ... FOR UPDATE`-style lock on a shared lock table) that ensures only one node in a Quartz cluster fires any given trigger, and that load balancing across nodes happens implicitly, whichever node acquires the lock first, rather than through any fixed assignment.
- [ShedLock, GitHub (lukas-krecan/ShedLock)](https://github.com/lukas-krecan/ShedLock/blob/master/README.md): source for ShedLock's explicit scope and design philosophy, a minimal distributed lock specifically for making sure a scheduled task runs at most once at the same time across nodes (backed by JDBC, Redis, MongoDB, Hazelcast, or ZooKeeper), and its own stated caveat that it is deliberately not a full-fledged distributed scheduler.
- [My Kubernetes Background Jobs Were Running 3x, DEV Community](https://dev.to/devvillmonkiventures/my-kubernetes-background-jobs-were-running-3x-heres-the-simple-fix-pl4): practitioner write-up of the exact "N replicas all wake up, query the database, and process the same pending jobs in parallel" failure mode described in Section 2, including the resulting duplicate user-facing emails and database CPU spike.

---

*Inference vs. fact, stated plainly: the CircleCI incident's root cause, timeline, and general shape are documented in CircleCI's own post-incident report and status page, but this session's network egress policy blocked a direct fetch of both, so the specific figures (33,135 duplicate workflows, 38 projects, 18 organizations) are relayed through a search-indexed summary rather than a first-hand read, and should be re-verified against the primary post-incident report before being cited elsewhere with confidence. The Kubernetes CronJob "at least once" guarantee, the 100-missed-schedule cap, and `startingDeadlineSeconds`/`concurrencyPolicy` behavior are documented, stable facts from Kubernetes' own official documentation. Chronos's original design goals and Quartz's JDBC-clustering lock mechanism are documented in their own respective project sources. The architecture diagram (the specific five-layer flow of clock, lock, idempotency check, queue, worker pool, dead man's switch), the "silent-decay loop" framing in Section 6, the fire-extinguisher and dispatcher analogies, and the Rare.lab mapping are this lesson's own synthesis built on top of the documented mechanics above, not a claim that any single company (CircleCI, Airbnb, or the Kubernetes project) implements exactly this five-layer architecture verbatim.*
