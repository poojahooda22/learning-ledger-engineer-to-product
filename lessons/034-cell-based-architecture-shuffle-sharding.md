# Day 34: How do you stop one bad thread, one bad customer, or one bad deploy from taking down everyone at once?

*2026-07-17*

---

## 1. The company and the number that breaks a naive design

**AWS Kinesis, November 25, 2020, US-EAST-1.** The trigger was a routine, small capacity add to the front-end fleet, started at 2:44 AM PST, finished by 3:47 AM PST. The actual root cause was structural: every front-end server in the Kinesis fleet held a dedicated OS thread for every *other* front-end server it needed to talk to, so per-server thread count grew as O(n) in the size of the whole fleet, and total connections across the fleet grew as O(n^2). Adding servers to help with load pushed every existing server's thread count up at the same time, and the fleet crossed an operating-system thread limit. The result was not "one server fell over." It was every server in the shared front-end fleet degrading together, because they were all counting toward the same shared ceiling.

The breaking number here is not a throughput number like "10,000 writes per second." It is a **topology number**: O(n) threads per box, in a fleet where n is the size of the entire regional front-end tier, hitting an OS-level ceiling. Because other AWS services (Cognito, CloudWatch, and more) called Kinesis in their own hot paths, the outage rippled outward into a roughly 17-hour event. AWS's own post-event summary states plainly that the medium-term fix was to "greatly accelerate the cellularization of the front-end fleet to match what they've done with the back-end," because cellularization is exactly the tool that keeps a shared resource, like a thread-per-connection count, from ever becoming a fleet-wide ceiling again.

---

## 2. Why the naive design dies

The naive version: one flat fleet, every server capable of serving every customer, and (in Kinesis's specific case) every server maintaining a live connection to every other server so any of them can answer questions about any shard. It collapses in three concrete, related ways.

**a. A limit crossed anywhere is a limit crossed everywhere.** Because every box's resource usage (threads, connections, memory) grows with the *same* shared variable (fleet size, total customer count, total shard count), there is no such thing as a partial failure. The whole fleet approaches its ceiling in lockstep, and when one server tips over, its neighbors are already close behind it.

**b. One noisy tenant degrades a resource every other tenant also depends on.** A hot key, a retry storm from one misbehaving client, or one customer's oversized query all draw from the same shared connection pool, the same shared thread budget, the same shared cache. There is no wall between "my problem" and "your problem" because there was never a wall built in the first place.

**c. The fix has the same blast radius as the failure.** Restarting or redeploying the shared fleet to patch the bug touches 100% of customers, not a slice of them, because there is only one fleet. Mitigation and damage are the same size.

The analogy: imagine one elevator shaft, with one motor and one set of cables, wired to serve every floor of every building in an entire city. It is efficient, one motor instead of thousands. But jam that one cable and no elevator moves anywhere in the city, not just in the one building where the jam happened.

---

## 3. The architecture, top to bottom

```
Clients (one workspace, one customer, one tenant per request)
   |
   v
Edge / DNS
   |  first hop, no tenant-specific logic yet
   v
Cell router (the ONE deliberately shared, deliberately thin layer)
   |  authenticates the request, looks up which cell owns this
   |  customer, then either proxies, redirects, or hands back a
   |  DNS name for that cell (three real implementation models:
   |  load-balancer, forwarding, or plain DNS)
   |  analogy: a hotel concierge who does none of the actual work,
   |  just tells you which floor your room is on
   v
   +--------------+--------------+--------------+
   |              |              |              |
   v              v              v              v
 Cell 1         Cell 2         Cell 3   ...   Cell N
   |              |
   |  each cell is a COMPLETE, independent replica of the whole
   |  stack: its own stateless app tier, its own cache, its own
   |  DB primary + replica, its own queue for async work
   |  each cell has a fixed maximum size (max customers, max QPS,
   |  max GB), decided BEFORE the cell fills up, not after
   |  analogy: separate apartment buildings, each with its own
   |  electrical panel and its own water main; knock one building's
   |  power out and the building next door never even flickers
   v
 within a cell: stateless app tier -> local cache -> DB primary
 + read replica -> local async queue
   |  same internal shape as any single-tenant service; a cell is
   |  just "the whole normal architecture," copied N times
```

A second, finer-grained mechanism, **shuffle sharding**, sits inside the router's assignment logic wherever a single cell still shares a pool of workers across many tenants (DNS resolvers, queue consumers, compile workers). Instead of putting every tenant wholly into one of a few buckets, each tenant gets a random *combination* of m workers drawn from a shared pool of n. Amazon Route 53 does this for real: n = 2048 virtual name servers, and each hosted zone (each customer's DNS zone) is assigned m = 4 of them.

A third piece, **static stability**, is what keeps a cell's own failover from depending on a live decision made by something outside the cell during an incident: each cell holds pre-provisioned spare capacity for its own zone failures ahead of time, so surviving a zone loss does not require a control-plane call to a shared capacity granter while that same shared system may itself be under stress.

---

## 4. The transferable mechanisms

**a. Bulkhead / cell isolation.** Partition the entire stack, app tier, cache, and database, not just the data, into N independent, complete copies, each with a hard ceiling on size. A failure inside one cell is contained to at most 1/N of customers by construction, not by luck or by a lucky routing decision made in the moment.

**b. Shuffle sharding.** Assign each tenant a random m-of-n combination of shared workers instead of putting every tenant wholly into 1 of a few contiguous shards. In contiguous sharding, two customers in the same shard share 100% of their fate: whatever breaks their shard breaks both of them, always. In shuffle sharding, two customers share *all* m workers only with probability roughly 1/C(n,m). AWS's own worked example: 8 workers, m = 2 per customer, gives C(8,2) = 28 distinct combinations, about seven times better isolation than plain 4-way contiguous sharding (1-in-28 fully-shared-fate odds versus 1-in-4). A denser pick, 4-of-8 using AWS's ordered-assignment variant, pushes the number of distinct assignments up to 8 x 7 x 6 x 5 = 1,680.

**c. Static stability.** Pre-provision the exact spare capacity a cell needs for its own worst-case failover *inside* that cell, ahead of time, rather than depending on a live API call to an external capacity-granting system during the incident that triggered the need for spare capacity in the first place. The system should stay standing even if every control plane it normally talks to goes silent.

**d. A thin, close-to-stateless router.** Keep the one genuinely shared component, the layer that decides which cell owns a request, as small and simple as possible. It is the single piece of the architecture that, if it breaks, breaks every customer at once, so it should do as little as possible: look up an assignment, hand back an address, nothing more.

**e. Bounded cell size as a day-one design input.** Decide the maximum customers, QPS, or gigabytes a single cell may hold *before* building it, not after it becomes a problem. Salesforce's production instances ("pods") each carry on the order of 8,000 to 10,000 customer organizations; when an instance nears that ceiling, Salesforce performs a deliberate "org split," moving some organizations to a fresh or less-loaded instance, rather than letting a single pod grow open-ended.

**f. Blast-radius math as a first-class reliability metric.** Express a reliability target not only as an uptime percentage but as "what fraction of customers does failure class X affect." AWS's own illustration: if a single database wipe (human error) affects one entire cell out of many, and each cell holds roughly an equal slice of a 1,000-user platform, ten cells of 100 users each turn a would-be 100%-of-users incident into a 10%-of-users incident, and at real scale (thousands of cells) that same class of failure shrinks toward a fraction of a percent of the whole platform.

---

## 5. The trade-offs

**Consistency vs. availability, per data type.** A customer's data lives in exactly one cell. Cross-cell transactions are either banned outright or made deliberately expensive and rare, because keeping them cheap would require exactly the kind of shared, tightly-coupled coordination layer that cells exist to avoid. Moving a customer between cells (a capacity rebalance, an "org split") is an explicit, one-time, out-of-band operation, not a live, continuously-consistent cross-cell read or write path. The system trades away live cross-cell strong consistency in exchange for hard, structural fault isolation.

**Cost vs. isolation.** Smaller cells shrink blast radius but multiply fixed overhead: N control planes, N sets of idle standby capacity sitting there for static stability, N times the operational surface to patch, monitor, and upgrade. Larger cells are cheaper per customer but widen the blast radius of any one failure. There is no cell size that is simultaneously cheapest and safest; every real deployment picks a point on this line deliberately, the way AWS's own guidance frames it: smaller cells are "easier to test and deploy" but costlier per customer served; larger cells are more cost-efficient but riskier per incident.

**Velocity vs. isolation.** Every schema change, every deploy, every migration has to roll out N times, usually gradually, one cell at a time, canary-style, rather than once to a single shared fleet. This is slower than a monolithic rollout on purpose: a bad deploy caught in the first canary cell never reaches the other N-1 cells at all, which is the entire point, but it means "ship to everyone" is structurally no longer a single action.

---

## 6. The systems-thinking lens

**The feedback loop that actually kills a non-cellularized system is a shared-resource amplifier, a specific flavor of metastable failure.** Trace Kinesis's actual loop: the fleet is stable at normal load -> a small, routine trigger (a capacity add) increases fleet size n -> because thread count per server scales with n, every server's thread usage rises at the same moment, not just the new servers' -> the shared OS thread ceiling is crossed fleet-wide -> affected servers start failing requests -> clients and internal callers retry -> those retries add more connection attempts onto an already-saturated shared resource -> the system does not recover on its own even after the original trigger (the capacity add) is long finished, because the retry traffic is now self-sustaining the same shared bottleneck that caused it. This is the signature of a metastable failure: a system that is perfectly fine at low load, and that a small trigger can push into a self-reinforcing bad state that does not exit on its own, because the system's own recovery behavior (retries) is feeding the same shared choke point that needs to recover.

**The senior fix does not add more threads.** That was AWS's own immediate stopgap, raising the OS thread-count limit, and it is a legitimate short-term patch, but it does not change the shape of the problem: the fleet still shares one fault domain, so the next resource that happens to scale with n will eventually hit its own ceiling the same way. The fix that actually breaks the loop is cellularization: split the fleet into N independent cells small enough that a ceiling crossed inside one cell is structurally invisible to the other N-1 cells, and where a shared worker pool still exists inside a cell, shuffle-shard the assignment so that even tenants sharing that pool rarely share full, correlated fate with each other. Capacity still grows over time, but it grows cell by cell, so the blast radius of the *next* thread-limit-shaped bug is bounded at 1/N instead of 100%, permanently, not just until the next surprise shared variable is discovered.

---

## Map to Rare.lab's stack

Rare.lab today runs on a shared Supabase Postgres instance with row-level security (RLS) doing the tenant isolation, one shared logical database, many workspaces, walled off from each other by policy rather than by physical separation, plus Cloudflare R2 holding content-addressed, immutable scene JSON and a manifest. That is the honest, correctly-sized answer for the current scale: a single shared Postgres instance with RLS is exactly the "one flat fleet" shape this lesson describes, and at low tens or hundreds of workspaces that is the right trade, not a mistake. Building N cells for a handful of customers would be paying the cost side of section 5's trade-off with none of the benefit.

The ceiling is predictable, though, and it is the same shape as Kinesis's: RLS isolates *data access*, not *resource consumption*. One workspace running an expensive query, a runaway shader compile job, or a pathological scene graph does not violate any RLS policy, but it can still saturate the shared Postgres connection pool or the shared compile-worker fleet that every other workspace also depends on, exactly the "one noisy tenant degrades a resource everyone shares" failure mode from section 2b. The embeddable runtime's one-shared-WebGL-context constraint is a browser-level, per-client concern and sits outside this lesson's scope; the backend compile service and the asset/manifest API are where cell-shaped thinking actually applies.

The concrete, actionable move, when workspace count or compile-worker load makes this worth doing (not before): treat the shader/scene compile pipeline as the first candidate for shuffle sharding rather than full cellularization, since it is the shared worker pool most likely to see one workspace's expensive compile job degrade every other workspace's compile latency. Assign each workspace a random m-of-n slice of the compile-worker pool instead of a first-come-first-served shared queue, the same move Route 53 makes with its 2048 name servers, so one workspace's pathological shader never fully correlates with any other specific workspace's fate. Full cell-based partitioning of Postgres itself (separate Supabase projects per customer band, with a thin router in front) is the next rung up, worth reaching for only once a single shared Postgres instance's connection pool or RLS-policy overhead is the measured bottleneck, not before.

---

## References and summaries

**AWS Solutions Library: "Guidance for Cell-Based Architecture on AWS"** (GitHub)
https://github.com/aws-solutions-library-samples/guidance-for-cell-based-architecture-on-aws
Fetched directly. This is the primary technical source for section 3's architecture and section 5's cell-size trade-off. It defines a cell as a "standalone, independent replica of an entire system," describes three router implementation models (load-balancer, forwarding, DNS), and states the cell-size trade-off explicitly: smaller cells reduce blast radius and are easier to test and deploy but raise operational overhead, larger cells are cheaper per customer but widen blast radius. It gives the concrete illustration used in section 5: a human-error database wipe in a cell-based system serving 1,000 users, spread across many cells, affects a fraction (its own worked number: about 0.1% of users) instead of the whole platform.

**AWS: "Summary of the Amazon Kinesis Event in the Northern Virginia (US-EAST-1) Region"** (AWS post-event summary, November 25, 2020)
https://aws.amazon.com/message/11201/
This is AWS's own official root-cause writeup for the outage that anchors section 1 and section 6. Direct automated fetch returned HTTP 403 (AWS's site blocks this session's fetcher), so the specific facts used here, the O(n) per-server thread growth tied to fleet size, the 2:44-3:47 AM PST capacity-add trigger, and AWS's stated commitment to "greatly accelerate the cellularization of the front-end fleet," are corroborated through search-indexed excerpts of the official document plus independent secondary write-ups (Arpio's "Outage Tales: The 17-Hour AWS Kinesis Outage" and Evan Jones's "Lessons from the AWS Kinesis Outage") rather than a direct full-text read. The event's real-world duration is reported consistently across sources as roughly 17 hours.

**AWS Builders' Library (Colm MacCarthaigh): "Workload isolation using shuffle-sharding"**
https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/
This is the canonical primary source for shuffle sharding, the mechanism in section 4b. Direct automated fetch returned HTTP 403. The specific numbers used here (8 workers taken 2-at-a-time giving C(8,2) = 28 combinations, roughly 7x better isolation than plain 4-way sharding; Route 53's real production parameters of n = 2048 virtual name servers with m = 4 assigned per hosted zone; the 8-choose-4 ordered-assignment variant giving 8x7x6x5 = 1,680 distinct assignments) are corroborated via a technical explainer that walks through the same combinatorics (Kakashi's blog, "AWS Shuffle Sharding") and AWS's own Well-Architected Framework fault-isolation guidance, both of which cite the identical figures back to this original source.

**AWS Well-Architected Framework: fault isolation and bulkhead guidance** (REL_10, "How do you use fault isolation to protect your workload?")
https://docs.aws.amazon.com/wellarchitected/2022-03-31/framework/rel_fault_isolation_use_bulkhead.html
Referenced for the standard vocabulary used throughout this lesson: "bulkhead" as the general pattern, with data partitions and service cells as its two concrete forms. Direct automated fetch returned HTTP 403; content used here is corroborated via indexed search excerpts.

**Salesforce architecture explainers (Cirra, FocusOnForce): pod / instance / superpod model**
https://cirra.ai/articles/salesforce-database-architecture-explained and https://focusonforce.com/platform/salesforce-instances-vs-orgs-vs-environments/
The source for section 4e's non-AWS, independently-arrived-at real-world example of cell-based architecture. Salesforce's production "pods" (instances) each hold on the order of 8,000 to 10,000 customer organizations, with "org splits" used to relieve a pod nearing capacity, and "superpods" grouping several instances behind shared infrastructure while keeping failure isolated from other superpod groups. Read via search-indexed excerpts, not a direct full-text fetch.
