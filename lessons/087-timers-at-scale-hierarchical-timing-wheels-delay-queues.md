# Day 87: How do you keep 10 million timers ticking (request timeouts, retries, "send in 15 minutes") when almost every one of them gets cancelled? Timing wheels and delay queues

**Date:** 2026-10-11
**Difficulty:** Advanced (data structures under a hot path, O(1) insert and cancel, clock-driven vs event-driven, the cancel-heavy workload)
**Topic:** Day 57 covered distributed cron (WHO fires a job). This lesson is the layer below: HOW one machine tracks millions of pending deadlines cheaply.
**Stack relevance:** Rare.lab has timers everywhere without naming them: debounce on autosave, render-job timeouts, shader compile deadlines, presigned URL expiry, retry backoff.

---

## 0. The framework (same recipe as Days 74 and 86)

The mock interview in your prompt (design LeetCode) uses a repeatable recipe. Use it on any product:

1. **Functional requirements** (what the user can do), plus an explicit **out of scope** list.
2. **Non-functional requirements** (availability vs consistency, latency target, scale, isolation).
3. **Core entities**, then **API routes**.
4. **Simplest high-level design**, one requirement at a time.
5. **Break it, then optimize**: run code on the API server (bad), then VMs (heavy), then containers (light), then a **queue** between API and workers to flatten spikes, **retries with exponential backoff**, **cache** for the leaderboard, **replicas** for the submissions DB.
6. **Name the trade-off** out loud each time.

The full LeetCode walk-through is in [Day 74](074-leetcode-online-judge-end-to-end.md). Notice one line in that video: "put a timeout of 2 or 5 seconds to stop infinite loops". That sentence hides today's lesson. Every submission needs a deadline timer. In a contest with 100,000 people each submitting several times, you hold hundreds of thousands of live timers, and 99 percent are cancelled because the code finished early. Who tracks them, and how cheaply?

Today's recipe:
- Functional: "call me back at time T unless I cancel first."
- Non-functional: insert and cancel in O(1), tolerate a few ms of lateness, survive millions of live timers, no thread per timer.
- Naive design: one OS timer or one sleeping thread per deadline.

---

## 1. The company and the breaking number

**Apache Kafka (Confluent).** A broker holds requests that cannot finish yet but have not failed: a produce request with `acks=all` waits for all replicas, a fetch with `min.bytes=1` waits for data. Each has a timeout. The number of these in flight scales with connections, "often tens of thousands" per broker (Confluent engineering blog, see sources). Most complete normally, so most timeouts are cancelled before they fire.

**The breaking number: 1,000,000 live timers where 95 percent are cancelled before firing.** (My workload, a realistic shape for RPC timeouts; Kafka's post states the same cancel-heavy pattern qualitatively.)

- A binary heap (priority queue) costs O(log n) per insert: about 20 comparisons at 1M entries. Cancel is worse: finding an arbitrary entry is O(n), so systems use "lazy delete" and leave dead entries in the heap. In my toy run the heap held all **1,000,000 entries** even if 950,000 were already dead.
- A wheel costs O(1) insert and O(1) cancel and frees the entry immediately.

The second breaking number: the Linux kernel. Thomas Gleixner's 2016 rework argued that the old cascading timer wheel did "largely wasted effort because most timers are cancelled or rearmed before they expire" (LWN and LKML, see sources). Same pattern, different layer.

Analogy: a hotel wake-up call desk. Naive: one clerk with a stopwatch per guest. Wheel: a wall of 60 pigeonholes labelled by minute; drop the note in the right hole, and each minute the clerk empties one hole.

---

## 2. Why the naive design dies

- **One thread (or OS timer) per timer.** 100,000 threads cost gigabytes of stack and drown the scheduler. Fine for 10 timers, dead at 100,000.
- **One sorted structure (heap, red-black tree).** Insert O(log n), cancel needs a handle or lazy delete. Dead entries pile up and memory grows with arrival rate, not live count. Also a single lock around the structure becomes the hot spot.
- **Scan everything every tick.** A loop "every 10 ms, walk all timers, fire expired" is O(n) per tick: 1M timers x 100 ticks/s = 100 million checks per second doing nothing. Kafka's old purgatory did a version of this: a timeout reaper plus hash-map watcher lists that had to be scanned, and scanning too often "carries a significant performance penalty" (Confluent blog).
- **Sleeping until the next deadline with a heap.** Good for CPU when timers are sparse, but every insert that becomes the new earliest deadline must wake the sleeper (a signal per insert).

Real example: LeetCode contest, minute 1. 100,000 users hit Submit within 60 seconds. Each submission needs a 5 s kill timer plus a queue-wait timeout plus a retry timer. That is 300,000 timers created in a minute, about 5,000 per second, nearly all cancelled in under 2 s.

---

## 3. The architecture, drawn top to bottom

```
Callers (RPC layer, job runner, retry scheduler)
        |  schedule(task, delay_ms)  /  cancel(handle)
        v
Timer facade  (one per shard or per core; no global lock)
        |
        v
Hierarchical timing wheel
   Level 0: 64 buckets x 1 ms   = covers 64 ms
   Level 1: 64 buckets x 64 ms  = covers 4.1 s     <- overflow of level 0
   Level 2: 64 buckets x 4.1 s  = covers 262 s     <- overflow of level 1
   Level 3: ...                                    <- and so on
        |
        v
Clock driver:  DelayQueue of NON-EMPTY buckets, ordered by bucket expiry
        |   wakes only when a real bucket is due (no 1 ms busy tick)
        v
Expiry worker pool  -> runs the callbacks (never inside the wheel thread)
```

| Layer | Single job | Analogy |
|---|---|---|
| **Timer facade** | Accept schedule and cancel; return a handle | The front desk taking wake-up requests |
| **Bucket** | Hold every task due inside one time slice, as a linked list or set | A pigeonhole for one minute |
| **Level 0 wheel** | Fine resolution for near deadlines | The minute hand |
| **Higher levels** | Coarse resolution for far deadlines | The hour hand |
| **Reinsertion** | When a coarse bucket comes due, push its tasks down into finer wheels | Sorting the "tomorrow" tray into hourly slots when tomorrow arrives |
| **DelayQueue clock driver** | Sleep until the next non-empty bucket is due | An alarm set for the next real appointment, not a tick every millisecond |
| **Expiry pool** | Run callbacks off the wheel thread | A courier so the clerk never leaves the desk |

Vocabulary:
- **Tick**: the smallest time unit of a wheel (1 ms here).
- **Wheel**: a circular array of buckets; a pointer (current time) moves around it.
- **Bucket**: a list of tasks sharing a time slice.
- **Hierarchical**: several wheels, each covering the overflow of the one below, like seconds, minutes, hours.
- **Cascading**: moving tasks from a coarse wheel into finer wheels as time advances.
- **DelayQueue**: a priority queue that releases an element only when its delay has elapsed.

---

## 4. The transferable mechanisms

### 4.1 Index by time, not by sort order: the wheel itself

The key move from Varghese and Lauck's 1987 paper ("Hashed and Hierarchical Timing Wheels", SOSP 1987): instead of keeping timers sorted, **use the expiry time as an array index.** A timer due in 37 ms goes in bucket `(now + 37) mod 64`. Insert is one array write. No comparisons. Cancel is one list unlink if you hold the node. The paper's own motivation: per-tick work and per-timer cost both matter when timers are numerous (summary via Adrian Colyer's "morning paper" and slides, see sources).

Walked end to end, with Kafka-style numbers (tick 1 ms, 64 buckets per level, my arithmetic):
- Level 0 covers 64 ms. Level 1 buckets are 64 ms wide and cover 4,096 ms. Level 2 buckets are 4,096 ms wide and cover 262,144 ms (about 4.4 minutes).
- A 30 s deadline fits in level 2. Three levels is enough for 30 seconds; 4 levels reach about 4.7 hours; 5 levels about 12 days.
- In my toy simulation (1M random deadlines up to 30 s, 64 slots, 1 ms tick) an insert touched on average **2.86 levels**, i.e. about 3 cheap array steps. The heap needed about 20 comparisons per insert at the same size. (I measured work counts; I do not claim the Python wall-clock times transfer to the JVM.)

### 4.2 Cascade only what is actually due: reinsertion

A task in a level 2 bucket is only precise to 4 seconds. When the clock reaches that bucket's start, you re-add each task: its remaining delay is now under 4 s, so it falls to level 1, and later level 0. Each task is touched at most once per level. Cost per timer over its life: O(levels), a small constant. But if the task is cancelled while still sitting in level 2, **it is never touched again.** This is why wheels win on cancel-heavy loads.

The kernel took the opposite stance in 2016: it removed cascading entirely (commit 500462a9de65, "timers: Switch to a non-cascading wheel") because cascading is "an unbound operation" and mostly wasted when timers are cancelled. Far timers now live in coarse buckets and fire a little late (up to the bucket width, 12.5 percent of the delay by design as I read the summaries). Trade: accuracy for bounded work. Timeouts are by nature fuzzy; a 30 s timeout firing at 31 s is fine.

### 4.3 Drive the clock by events, not by ticking: DelayQueue

A thread that wakes every 1 ms to advance a wheel burns CPU even with zero timers. Kafka's design avoids this: put each **non-empty bucket** into a `DelayQueue` keyed by its expiry. One thread blocks on `poll()` until the earliest bucket is due, advances the wheel to that time, and reinserts or fires its tasks (Confluent blog and the Apache wiki design page summaries). Cost is proportional to buckets with work, not to elapsed time. The kernel does similar with a per-level bitmap of non-empty buckets, so it finds the next expiring timer fast after long idle (LWN summary of the 2016 series).

### 4.4 Never run user callbacks on the wheel thread

If a callback takes 200 ms, every timer behind it fires 200 ms late, and the wheel thread is blocked from advancing. Hand expired tasks to a worker pool. Netty's `HashedWheelTimer` documents the same trap: long tasks delay all others; use it for short, cheap callbacks (from memory of its Javadoc, which cites the Varghese paper).

### 4.5 Make timers stateless to survive restarts: the durable delay queue

An in-memory wheel dies with the process. For "send this email in 15 minutes", pair it with a durable store:
- Store the task in a database or log with a `due_at` column (Postgres: `where due_at <= now() order by due_at limit 100 for update skip locked`, Day 57).
- Keep a small in-memory wheel only for the next few seconds of work, loaded in batches ("windowed prefetch").
- Tasks are idempotent (Day 12), because after a crash the same task may load twice.

Real example: Amazon SQS exposes this as per-message `DelaySeconds`, up to 15 minutes (from memory of the public docs; I did not re-verify this session). Beyond that, systems store due times in a database and poll.

### 4.6 Shard timers by key, not by lock

One wheel, one thread, one core worth of throughput. To scale, run N independent wheels (per core, or per partition like Kafka's per-partition delayed operations) and hash the key to pick one (Day 10). No shared lock, linear scaling. Netty runs one wheel per event loop for this reason.

---

## 5. The trade-offs

**Consistency vs availability, per data type:**

| Data | Choice | Why |
|---|---|---|
| In-memory timer state | Volatile, local, **lose on crash** | A request timeout is meaningless after the process that made the request died |
| Durable scheduled work ("email in 15 min") | **Persist first, fire at-least-once** | Losing it is a bug; firing twice is fixed by idempotency |
| Exact fire time | **Approximate (bounded lateness)** | Exactness costs O(log n) and wake-ups; nobody notices 1 ms |

**Cost vs latency:**
- Finer tick = more precision, more buckets, more wake-ups. 1 ms is the usual floor on JVMs; the kernel uses jiffies (1 to 10 ms, per CONFIG_HZ).
- More levels = bigger range, slightly more reinsertion work per long timer.
- Coarse far buckets save memory and CPU at the price of late firing (the kernel's 2016 choice).

**The choice this system makes:** cheap and approximate in memory, exact and durable only where losing a deadline matters.

---

## 6. The systems-thinking lens

**The feedback loop: the timeout-retry storm (a metastable failure).**

1. A dependency gets slightly slow (say judge workers at 90 percent CPU).
2. Requests hit their timeouts and callers retry.
3. Retries add load, so the dependency gets slower.
4. More timeouts, more retries. Load now stays high even after the original slowdown ends.
5. And the timer facility itself pays: every retry creates new timers, so wheel size, memory and expiry callbacks all spike exactly when the system is weakest.

Why this is a timer lesson, not just a retry lesson: **timeouts are the trigger of the loop.** A timeout set too short fires on healthy slow requests and manufactures retries. A timeout set too long ties up memory and threads.

How the senior fix **breaks** the loop (it does not add machines):
- **Deadline propagation.** Pass the remaining budget downstream. If the caller gave up 3 s ago, the callee drops the work instead of finishing it (Day 13).
- **Retry budget plus exponential backoff with jitter.** The video says "exponential retry"; add jitter so retries do not synchronize into a thundering herd (Day 76).
- **Circuit breaker.** After N timeouts, stop calling for a cool-down. Fewer timers created, loop starved.
- **Cancel eagerly.** On success, cancel the timer immediately (O(1) in a wheel). In a lazy-delete heap, dead timers still count against memory.
- **Hedge instead of blind retry.** Day 76.

---

## 7. Mapping to Rare.lab

Rare.lab: Supabase Postgres with RLS, Cloudflare R2 (immutable content-addressed scene JSON plus manifest), embeddable runtime with one shared WebGL context.

**Already in use, whether you named them or not:**
- Presigned R2 URL expiry (time-bounded access with no server-side timer; the clock is in the signature).
- Debounced autosave in the editor (a one-timer-per-editor pattern).

**Where timers apply next:**

| Idea today | Rare.lab application | Action |
|---|---|---|
| Cancel-heavy workload | Autosave debounce and shader-compile timeouts in the editor: each keystroke or slider drag resets a timer | Reuse ONE timer and push its deadline instead of clear/set per event. At 60 slider events per second per user, `clearTimeout`/`setTimeout` churn is avoidable allocation |
| Durable delay queue | Scheduled publishes, trial expiry, "retry failed export in 5 min" | A `jobs` table with `due_at`, `attempts`; claim with `for update skip locked`; idempotency key per job (Day 12, Day 57). Do not hold these in a Worker's memory; Workers die |
| Deadline propagation | Embeddable runtime asking the Worker, which asks Supabase | Pass an absolute `deadline` header; Workers and DB calls abort when it passes. Stops the retry storm at the edge |
| Retry budget + jitter | Client fetching scene JSON from R2 | Max 2 retries, jittered 100 to 400 ms, then show the cached/older scene |
| Per-core wheels | The runtime's frame scheduler | One shared WebGL context means one render loop. Use a single `requestAnimationFrame` driver and a small bucketed scheduler for animations, not one timer per effect instance |

**Next ceiling:** the embeddable runtime on a page with 50 effects. If each effect owns its own `setTimeout`/`setInterval`, the browser's timer queue and your frame budget (16.6 ms at 60 fps) both degrade. Centralize into one driver that wakes on the next due item, exactly the DelayQueue idea (4.3).

**One-line lesson for Rare.lab:** do not create a timer per thing; keep one clock driver and index work by due time, cancel eagerly, and put any deadline you cannot afford to lose in a database row, not in memory.

---

## 8. Sources and what is actually in them

**Honest note on access:** the network proxy blocked direct page fetches this session (confluent.io, lwn.net and blog.acolyer.org did not resolve). Everything attributed below comes from web search result summaries, not the original pages. Numbers in sections 1 and 4.1 (heap 1,000,000 retained entries, 2.86 levels touched, level ranges) come from my own toy run and arithmetic and are labelled as mine. The retry-storm framing is my own synthesis. SQS delay and Netty Javadoc details are from memory and flagged as such.

- [Varghese and Lauck, "Hashed and Hierarchical Timing Wheels: Data Structures for the Efficient Implementation of a Timer Facility", SOSP 1987](https://cseweb.ucsd.edu/users/varghese/PAPERS/twheel.ps.Z). Plain language: the original paper. It compares ways to keep many timers (unsorted list, sorted list, tree, wheel) and shows that indexing by time gives constant-time start and stop. Hashed wheel for one range, hierarchical wheels for a huge range.
- [Adrian Colyer, the morning paper: Hashed and hierarchical timing wheels](https://blog.acolyer.org/2015/11/23/hashed-and-hierarchical-timing-wheels/). Plain language: a readable walkthrough of the paper's schemes and why the hierarchy works, and a note on Kafka's use.
- [Confluent: Apache Kafka, Purgatory, and Hierarchical Timing Wheels](https://www.confluent.io/blog/apache-kafka-purgatory-hierarchical-timing-wheels/). Plain language: Kafka brokers hold requests that cannot finish yet. The old way scanned lists of waiting requests, which was slow with tens of thousands of connections. The new way uses timing wheels for O(1) insert and delete, and a DelayQueue so one thread sleeps until the next bucket is due.
- [Apache Kafka wiki design page (KIP discussion draft)](https://cwiki-test.apache.org/confluence/x/go5JAw). Plain language: the earlier design notes, including why a thread that wakes every tick wastes CPU when requests are sparse.
- [LWN: Reimplementing the timer wheel (2016)](https://lwn.net/Articles/691064/) and the LKML patch series ([V3](https://lkml.iu.edu/hypermail/linux/kernel/1606.3/00404.html)). Plain language: Thomas Gleixner explains why Linux dropped cascading: it is unbounded work and mostly wasted because most timers are cancelled. The replacement uses coarser buckets for far timers (they fire slightly late) plus a bitmap to find the next due bucket. V3 kept the 1 ms first level after Eric Dumazet pointed out datacenter workloads need it.
- [Kernel commit 500462a9de65, "timers: Switch to a non-cascading wheel"](https://upd.dev/kernel/linux/commit/500462a9de65). Plain language: the actual change, merged July 2016.
- [Netty HashedWheelTimer source](https://code.yawk.at/io.netty/netty-all/4.1.25.Final/io/netty/util/HashedWheelTimer.java). Plain language: a production wheel in Java. Its header cites the Varghese paper and warns that slow tasks delay the others.
- [Akka HashedWheelTimer Javadoc](https://doc.akka.io/japi/akka/2.2.5/akka/util/internal/HashedWheelTimer.html). Plain language: another implementation citing the same paper.
- [US patent 6360329, interval-timing with overbuilt hierarchical wheels](https://patents.google.com/patent/US6360329). Plain language: shows the classic "hours, minutes, seconds wheels cover 24 hours at one-second resolution" picture.
- Video context: the LeetCode mock interview from your prompt. The speaker recommends Hello Interview (blog and videos) and a YouTube channel she names as "Shrians Jen" (spelling unverified).

**Confirmed (from search summaries):** Varghese and Lauck SOSP 1987; hierarchical wheels with hours/minutes/seconds example; Kafka purgatory design (tens of thousands of connections, O(1) insert and delete, DelayQueue driving the clock); Netty and Akka implementations cite the paper; Linux 2016 non-cascading wheel, commit 500462a9de65, motivations (cascading unbounded, most timers cancelled, bitmap for next expiry), V2 vs V3 granularity story. **Inference or mine:** all numbers in sections 1 and 4.1; the retry-storm lens; the Rare.lab mapping; SQS 15 minute delay (memory).

**Related lessons:** Day 12 (idempotency), Day 13 (backpressure, deadlines), Day 57 (distributed cron), Day 74 (LeetCode judge), Day 76 (hedged requests), Day 84 (control loops).
