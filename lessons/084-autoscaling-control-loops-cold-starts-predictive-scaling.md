# Day 84: How does a service survive a 10x traffic jump when new servers take 90 seconds to arrive? Autoscaling as a control loop, cold starts, and predictive scaling

**Date:** 2026-10-08
**Difficulty:** Advanced (feedback control, delay in the loop, metastable failure, forecasting)
**Topic:** Autoscaling is not "add servers when busy". It is a feedback controller with sensor lag, actuator lag and a cost function. Most scale failures are the controller losing to the delay.
**Stack relevance:** Rare.lab already rides two autoscalers it does not run (Cloudflare Workers, R2). Section 7 shows where autoscaling stops helping: the database and the AI generation path.

---

## 0. The framework (same recipe as Day 74)

1. Functional: a service gets more traffic, the platform adds capacity. Traffic falls, it removes capacity. Nobody pages a human.
2. Non-functional: keep p99 latency under a target, never drop requests during a spike, do not pay for idle machines, do not flap (add, remove, add, remove).
3. Entities: Metric (the sensor), Target (desired value), Replica (one unit of capacity), Policy (the rules that turn metric into replica count).
4. API: `desired_replicas = f(metric, target, current)`, plus min and max bounds.
5. Naive design: "if CPU > 80%, add one server. If CPU < 20%, remove one."
6. Deep dives: the control-loop delay chain, the proportional formula, stabilization windows, cold start, warm pools, panic mode, predictive scaling, and what to do in the gap while capacity is still arriving.

---

## 1. The company and the breaking number

**Netflix (and every company on AWS or Kubernetes).** Netflix described its own breaking case in its 2013 Scryer posts: some traffic spikes arrive when the fleet is only about **20% of its maximum size**, while other rises are modest with the fleet near **80%**. The same reactive rule cannot be right for both.

The number I use to teach it (illustrative arithmetic, not a Netflix figure):

- Normal load: **10,000 requests per second**. Each request holds a server for **50 ms**.
- Little's Law: requests in flight = arrival rate x time each takes = 10,000 x 0.05 = **500 at once**.
- Each pod handles **10 at once**. So you run **50 pods**.
- Traffic jumps **10x** to 100,000 requests per second. You now need **500 pods**.
- A new pod takes **90 seconds** to be ready (pull image, boot, load config, warm caches, pass the readiness check) plus the time for the sensor to notice.
- For those 90 seconds you have 50 pods of capacity facing 100,000 requests per second. That is **90,000 requests per second more than you can serve, for 90 seconds = 8.1 million requests with nowhere to go.**

The breaking number is not 100,000 requests per second. It is **the 90 second gap**. Capacity arrives later than demand. Autoscaling is a race the controller starts behind in.

Analogy: a restaurant whose kitchen takes 90 minutes to hire and train a new cook. A tour bus pulls up. By the time the cooks arrive, the guests have left angry and told everyone online.

---

## 2. Why the naive design dies

**Naive: "if CPU > 80%, add one server."**

- **It is too slow.** Adding one server per check is a linear fix for an exponential problem. At 10x load you need 450 more pods. One at a time, with a 90 second delay each, you arrive next week.
- **It reads a lagging sensor.** CPU is averaged over a window. Metrics are scraped on an interval. The decision runs on another interval. By the time the number says "busy", the queue has been growing for 30 or more seconds.
- **It flaps.** Scale out, load per pod drops, scale in, load per pod jumps, scale out. Every scale-in throws away warm caches and every scale-out pays the cold start again. Analogy: a heating thermostat with no tolerance that clicks on and off every ten seconds.
- **It counts the newcomers wrongly.** A pod that just started is at 100% CPU (loading code) or 0% (not serving yet). Both mislead the average. The controller then adds more pods because the new ones "look overloaded". That is the AWS reason for **instance warmup** (section 4).
- **It makes things worse during an incident.** If the real problem is a slow database, requests pile up, CPU on the app tier stays low (threads are waiting, not computing), and the controller does nothing. Or it adds more app servers that open more database connections and finish off the database. Adding capacity to a stage that is not the bottleneck is how outages get bigger.
- **It cannot go to zero.** If the metric is CPU and there is no traffic, the signal is flat. Something else must wake the service when the first request arrives.

---

## 3. The architecture, top to bottom

```
Clients (a spike arrives)
        |
        v
Edge / CDN ............. absorbs repeat reads. A hit never reaches the autoscaled tier.
        |                Analogy: photocopies of the menu at the door.
        v
Load balancer .......... spreads requests, counts in-flight, can SHED when full (HTTP 503 fast).
        |                Analogy: the host who says "30 minute wait" instead of seating everyone.
        v
Admission / queue ...... bounded buffer. Buys seconds while new replicas boot.
        |                Analogy: a rope line with a limited length.
        v
Stateless app tier ..... N replicas. The thing being scaled. Must be safe to kill any time.
        |                      ^
        |                      | desired = ceil(current x metric / target)
        |             Autoscaler (controller loop)
        |                      ^
        |                      | scrape every ~15 s, smooth over a window
        |                   Metrics store
        v
Warm pool / pre-booted . spare machines already past boot, parked, ready to join in seconds.
        |
        v
Cache (L1 in process, L2 shared) ... absorbs read load so the DB is not scaled by traffic.
        |
        v
DB primary + read replicas ... NOT autoscaled by the same loop. Fixed ceiling. Protect it with pooling.
```

Key idea: **the autoscaler sits in a loop, and every arrow in that loop has a delay.** Read the delay chain once and most autoscaling bugs become obvious.

### The delay chain (typical Kubernetes shape, numbers labeled)

| Step | What happens | Typical delay |
|---|---|---|
| 1. Traffic rises | Requests per pod increase | 0 s |
| 2. Metric scrape | Metrics server or Prometheus reads usage | up to about 15 s (my recollection of common defaults, not re-verified today) |
| 3. Controller sync | HPA loop evaluates and computes desired replicas | about 15 s period (same caveat) |
| 4. Scheduling | Scheduler finds a node. If no node has room, the cluster autoscaler adds a VM | seconds if room, **minutes** if a new VM is needed |
| 5. Image pull and start | Download container image, start process | 5 to 60 s depending on image size |
| 6. App warmup | Open connections, load config, JIT compile, fill caches | 5 to 60 s |
| 7. Readiness | Load balancer sees the pod is ready and sends traffic | 5 to 10 s |

Add them and 60 to 150 seconds is normal. That is the 90 second gap from section 1. It is the sum of seven small, reasonable-looking delays.

---

## 4. The load-bearing patterns (the 5 reusable mechanisms)

### 4.1 Proportional scaling, not one-at-a-time

The Kubernetes Horizontal Pod Autoscaler (HPA) uses a ratio, per the official docs: `desiredReplicas = ceil[ currentReplicas x ( currentMetricValue / desiredMetricValue ) ]`.

- Docs example: current metric 200m, target 100m, so replicas double. Ratio 0.5 halves them.
- A worked example from a secondary guide: 3 replicas at 75% average CPU with a 50% target gives ceil(3 x 75 / 50) = ceil(4.5) = **5 replicas** in one step.
- If the ratio is within a **tolerance of 0.1** of 1.0 (so 0.9 to 1.1), it does nothing. That is the dead band that stops tiny wiggles from causing churn.
- With several metrics, the controller computes a replica count for each and takes the **largest**. The scarcest resource wins.

Why it works: the response is proportional to the error. A 10x jump asks for 10x the pods in one decision, not +1.

### 4.2 Asymmetric stabilization: scale up fast, scale down slow

The same docs describe a stabilization window: for scale-down the controller looks at all recommendations in the window and takes the **highest**. The API reference gives the defaults: **scale up 0 seconds (react now), scale down 300 seconds (5 minutes)**.

Why: being wrong on the way up drops requests (expensive). Being wrong on the way down just costs a few minutes of idle server (cheap). So the loop is deliberately jumpy upward and sticky downward. This is what stops flapping.

### 4.3 Warmup: do not trust a pod's numbers until it has settled

AWS EC2 Auto Scaling has an **instance warmup**: the delay between an instance reaching InService and its metrics being counted in the group's aggregate. While instances are warming, the policy scales out only if the metric from instances that are **not** warming is above the threshold, and it counts warming instances as part of capacity when deciding how many to add. AWS notes it is **not enabled by default** and strongly recommends turning it on. If unset, the default cooldown is used as the warmup time.

Why: without it, a fresh instance spiking at 100% CPU while it boots tricks the controller into adding yet more instances. Counting the newcomers as "capacity on the way" prevents double ordering.

### 4.4 Warm pools and cold-start hiding: shrink the delay, not just react to it

You cannot make the loop instant, but you can make the actuator faster.

- **AWS warm pool:** a set of pre-initialized instances kept beside the group, in Stopped, Running or Hibernated state. On scale-out the group draws from the pool, which "ensures instances are ready to quickly start serving". The default pool size is the group's max capacity minus desired capacity, or you cap it with `MaxGroupPreparedCapacity`. Cost trade: a stopped instance costs storage, not compute.
- **Knative panic mode and scale-to-zero:** stable window **60 s**, panic window **6 s** (10% of stable). If the panic count reaches **2x** the current ready pods, it uses the panic count. Pod math: `pods = concurrent requests / (pod max concurrency x target utilization)`, defaults of **70%** utilization. Example: container concurrency 10 at 70% means 100 concurrent requests need ~15 pods. Scale-to-zero grace period default is **30 s**, and an "activator" buffers the first request while a pod starts.
- **Cloudflare Workers:** stage one hid a roughly **5 ms** cold start inside the TLS handshake (the server name sits in the first message, so start loading the Worker immediately). When Workers grew and TLS 1.3 shrank the handshake to one round trip, that was no longer enough. Stage two, "shard and conquer" (published 2025-09-26), uses a consistent hash ring so all requests for one Worker land on the same server, cutting cold starts about **10x** and giving a **99.99% warm request rate**. Mechanism: stop spreading one app thinly across every machine, so the app's warm copy gets reused. This ties straight back to Day 10 (consistent hashing).

### 4.5 Predictive scaling: act before the sensor can

Netflix Scryer (2013, with a Part 2 around 2015) forecasts demand and provisions ahead of it, because reactive scaling only sees a problem after it starts.

- Pipeline: data collector, predictor, action plan generator.
- Predictor used two algorithms: an augmented linear regression and one based on a Fast Fourier Transform (FFT, which finds repeating daily and weekly cycles).
- It scales on **requests per second**, not load average, because load average depends on the scaling itself (more servers lowers load, which hides the demand).
- The plan generator minimizes the number of scale-up events while keeping each batch effective. Humans can pad for holidays.
- Netflix pairs it with the reactive scaler. Prediction handles the daily rhythm; reaction handles surprises. Neither alone is enough.

---

## 5. The trade-offs accepted

Per data type and per layer, not one global setting:

| Layer | Choice | Why |
|---|---|---|
| Stateless request tier | **Availability and latency over cost.** Run headroom (target 50 to 70%, not 95%). | An idle server costs cents. A dropped checkout costs customers. |
| Scale-in speed | **Cost loses to stability.** 5 minute scale-down window. | Flapping costs more in cold starts than idle minutes cost in money. |
| Burst absorption | **Consistency of experience over completeness.** Shed some requests fast rather than serve all slowly. | A fast 503 with Retry-After lets the client back off; a slow 30 s timeout triggers retries. |
| Database | **Consistency over elasticity.** One primary, capped connections. | You cannot add a writer by turning a knob (Day 45, 62, 78). |
| Cache | **Staleness over load.** Serve slightly old data during the spike. | Day 19. |

Cost versus latency, concretely:

- **Reactive only:** cheapest, but you eat the 90 second gap on every spike.
- **Headroom (over-provision):** no gap, but you pay for idle capacity 24 hours a day.
- **Warm pool:** a middle path. Pay a small standby cost, shrink the gap from 90 s toward 10 s.
- **Predictive:** cheapest way to cover the *predictable* spikes (evening peaks, a scheduled launch). Useless for the unpredictable ones. Netflix still needs both.

---

## 6. The systems-thinking lens

### The feedback loop that actually kills you: the metastable overload

Bronson, Aghayev, Charapko and Zhu, "Metastable Failures in Distributed Systems" (HotOS 2021): a system in a **vulnerable state** (running efficiently near capacity) meets a **trigger** (a short spike, a deploy, a cache flush). A **sustaining effect** (usually retries) then keeps it overloaded **after the trigger is gone**. Goodput (useful work per second) collapses even though raw demand has returned to normal.

How autoscaling joins the loop:

1. Spike arrives. Latency rises. Clients time out and retry, so real arrival rate becomes 2x to 3x the original.
2. The autoscaler sees the high metric and orders pods. They take 90 s.
3. New pods boot cold (empty caches). Cold pods are slow. Latency rises further. More timeouts, more retries.
4. New pods open new database connections all at once. The database slows. App latency rises again.
5. The controller sees the metric still high and orders even more pods.

This is a **retry death spiral plus a thundering herd of new replicas**, and more capacity feeds it instead of ending it.

### Breaking the loop (the senior fix is not "bigger max replicas")

1. **Load shedding in front of the scaler.** AWS's Builders' Library article "Using load shedding to avoid overload" (David Yanacek) says overload makes a vicious cycle: clients time out, work is wasted, retries add more. Shed early and cheaply to keep goodput up for the requests you do accept. An AWS ALB post makes the link explicit: elasticity reduces the need for shedding, but it is still needed when cold start times blunt elasticity. So shedding covers the gap that autoscaling cannot.
2. **Bounded queue, not unbounded.** A queue buys seconds. An infinite queue buys a 10 minute backlog of requests nobody is waiting for any more.
3. **Retry budget and jittered backoff** on clients (Days 13, 83). Retries become a fixed fraction of traffic, not a multiplier.
4. **Warm replicas before they take traffic.** Readiness gates, pre-warmed caches, connection pools with ramp-up, so new pods join gently.
5. **Pool database connections** (Day 78) so 450 new pods do not mean 450 times more Postgres connections.
6. **Scale on the right signal.** Concurrency or queue depth beats CPU for I/O-heavy services, because CPU stays low while threads wait. Knative scales on concurrency for this reason.
7. **Cap the cap.** Set a max replica count that the database can survive. A scaler with no upper bound can turn a slow query into a self-inflicted outage.

---

## 7. Mapping to Rare.lab's stack

**What autoscaling Rare.lab already gets without running a controller:**

| Piece | Autoscaling status | Why |
|---|---|---|
| **Cloudflare Workers** (API, edge logic) | **Already autoscaled, per request.** | Isolates start in about 5 ms (Cloudflare's own figure), and shard-and-conquer routing keeps popular Workers warm. You have no replica count to tune. This is why you feel no "90 second gap" on the edge. |
| **Cloudflare R2 + content-addressed scene JSON** | **Effectively infinite read scale, zero egress.** | Immutable objects cache forever at the edge (Day 4, Day 23). A viral scene is one cache fill, then hits. Traffic spike on a scene costs almost nothing. |
| **Embeddable runtime with one shared WebGL context** | **Scales by distribution.** | The compute runs on the *viewer's* GPU, so 100x more viewers is 100x more GPUs, not more servers. The only server cost is the manifest and the JSON fetch. |
| **Supabase Postgres + RLS** | **Not autoscaled in the request sense.** | Fixed compute size. This is where the 90 second gap becomes "you cannot add a writer". |

**Where the next ceilings are (inference, not measured):**

1. **Postgres connections and writes, not Workers.** The first thing a Worker spike does is open more connections to Supabase. Use the pooled endpoint (Supavisor, Day 78), keep Workers reading from R2 and cache, and treat "Postgres CPU" as your real scaling signal. Autoscaling the edge will happily overrun an un-autoscaled database.
2. **The AI generation path.** If a user prompt triggers an LLM call and a shader compile, that stage has a **quota and a cost**, not just a latency. A traffic spike there is a spend spike and a rate-limit spike. Put a queue with a **bounded depth** and a per-user cap in front of it (Days 9, 13, 71), and shed with a clear message when full. Scaling it up is limited by the provider's rate limits, so shedding is the control you actually have.
3. **Cold start of heavy server-side work, if you add it.** If you later add server-side shader compile or preview rendering as a container or GPU job, expect the minutes-long actuator delay from section 3. Use a warm pool or a minimum replica count for it, and scale on queue depth, not CPU.

**Pattern checklist for Rare.lab:**

| Pattern | Do we use it? | Next action |
|---|---|---|
| Cache before scaling | Yes (R2 + CDN, immutable scenes) | Keep every hot read immutable and CDN-keyed |
| Stateless tier | Yes (Workers) | Keep session state in Postgres or KV, never in the Worker |
| Connection pooling | Partly | Always use the pooler endpoint from Workers |
| Load shedding / bounded queue | Not yet | Add on the AI generate endpoint before launch day |
| Retry budget and jitter | Not yet | Client SDK and Worker fetches: max 2 retries, jittered |
| Predictive or scheduled scale | N/A today | If you ship a launch or a Product Hunt day, pre-warm and raise limits **beforehand** |

**One-line lesson for Rare.lab:** autoscaling only fixes the stage that can scale, so push load onto the stages that scale for free (CDN, R2, the viewer's GPU), pool and cap the one that cannot (Postgres), and put a bounded queue plus load shedding in front of the expensive AI path, because the 90 seconds it takes capacity to arrive is the gap that breaks you.

---

## 8. Sources and what is actually in them

**Honest note on access:** direct page fetches were blocked by the network proxy this session (kubernetes.io, docs.aws.amazon.com, brooker.co.za did not resolve). Everything below was read through search result excerpts of the named pages, not the full pages. Where I add something from memory it is labeled. Verify numbers against the official pages before quoting them.

- [Kubernetes docs: Horizontal Pod Autoscaling](https://www.kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale). The primary source for the formula, the 0.1 tolerance, the "take the largest across metrics" rule and the stabilization windows (scale up 0 s, scale down 300 s). In plain language: the controller multiplies the replica count by how far over target you are, ignores tiny differences, scales up immediately and scales down only after five quiet minutes.
- [Scryer: Netflix's Predictive Auto Scaling Engine](http://techblog.netflix.com/2013/11/scryer-netflixs-predictive-auto-scaling.html) and [Part 2](https://netflixtechblog.com/scryer-netflixs-predictive-auto-scaling-engine-part-2-bb9c4f9b9385). Netflix explains why reactive AWS autoscaling struggled (spikes at 20% fleet size, rises at 80%, outages followed by retry storms, variable patterns) and how Scryer forecasts instead, with a data collector, predictor (linear regression plus FFT) and plan generator, scaling on requests per second. Not opened in full; the quantified results were not in the excerpts, so I cite none.
- [AWS docs: Default instance warmup](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-default-instance-warmup.html) and [Warm pools](https://docs.aws.eu/autoscaling/ec2/userguide/ec2-auto-scaling-warm-pools.html). Warmup keeps a booting instance's noisy metrics out of the average and counts it as capacity-in-progress. Warm pools keep pre-initialized instances (Stopped, Running or Hibernated) for fast scale-out. Plain language: do not panic about the new hire's first-day numbers, and keep some trained staff in the back room.
- [AWS Builders' Library: Using load shedding to avoid overload](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/) by David Yanacek, plus the [ALB target group load shedding post](https://aws.amazon.com/blogs/networking-and-content-delivery/target-group-load-shedding-for-application-load-balancer). Plain language: when you are overloaded, fast "no" to some beats slow "yes" to all, because slow answers cause timeouts and retries that make it worse. The ALB post states that cold starts are the reason elasticity alone is not enough.
- [Brooker: Metastable failures](https://brooker.co.za/blog/2021/05/24/metastable/) and [Bronson et al., Metastable Failures in Distributed Systems, HotOS 2021](https://doi.org/10.1145/3458336.3465286) (ACM DOI as reported by search). Define the vulnerable state, trigger and sustaining effect, with retries as the commonest sustaining effect. Brooker's angle: the root cause is often a feature that makes the system more efficient or reliable. Follow-up: [Metastable Failures in the Wild](https://www.usenix.org/conference/osdi22/presentation/huang-lexiang) (OSDI 2022) studied 22 incidents from 11 organizations.
- [Knative: Autoscaler types](https://knative.dev/docs/serving/autoscaling/autoscaler-types/) and [Demystifying the activator](https://knative.dev/blog/articles/demystifying-activator-on-path/). Concurrency-based scaling, stable and panic windows, scale-to-zero and the activator that buffers the first request. The window values (60 s, 6 s, 2x, 70%, 30 s) came from Alibaba Cloud documentation of Knative defaults via search, so confirm against upstream.
- [Cloudflare: Eliminating Cold Starts 2: shard and conquer](https://blog.cloudflare.com/eliminating-cold-starts-2-shard-and-conquer/) and the [InfoQ summary](https://www.infoq.com/news/2025/10/workers-shard-conquer-cold-start). Plain language: first they started the Worker during the TLS handshake so the wait was hidden. When Workers grew too big for that, they sent every request for a Worker to the same few servers using a hash ring, so the warm copy is reused. About 10x fewer cold starts, 99.99% warm.

**Confirmed vs inference.** Confirmed (from the excerpts above): the HPA formula, tolerance and stabilization defaults; AWS warmup and warm-pool behavior; Scryer's pipeline and algorithms; the metastable definition; Knative and Cloudflare figures as quoted. Inference, labeled: the 10,000 requests per second and 90 second arithmetic, the seven-step delay table values (some from memory), the retry-plus-new-pod spiral as a composite story, and all of section 7.

**About the video in your prompt:** the "Design LeetCode" mock interview is already fully taught in [Day 74](074-leetcode-online-judge-end-to-end.md). Its queue plus horizontally scaled container workers step is exactly the autoscaling question this lesson goes deep on: scale those workers on queue depth, with a warm pool, never CPU.

**Related lessons:** Day 9 (queue as shock absorber), Day 10 (consistent hashing), Day 13 (backpressure and load shedding), Day 19 (caching), Day 72 (chaos and failover), Day 74 (online judge), Day 78 (connection pooling), Day 83 (retry storms).
