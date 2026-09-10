# Day 72 — How does Netflix evacuate an entire AWS region in under 7 minutes, without the two regions left standing collapsing under the traffic they inherit?

**Date:** 2026-09-10
**Difficulty:** Expert
**Topic:** Chaos engineering and regional failover: not a new data structure or protocol, but a discipline this ledger hasn't taught yet, deliberately breaking a production system on purpose, on a schedule, to find out whether its redundancy actually works before a real disaster asks the same question. This lesson sits on top of machinery this ledger already built. Day 14 laid the active-active foundation (accept writes in more than one place, reconcile later). Day 22's quorum reads and vector-clock thinking are exactly what lets Netflix's Cassandra rings run independently per region. Day 13's backpressure and load-shedding ideas are the reason a careless failover doesn't just relocate the outage. Day 55's anycast/GSLB is the DNS-and-routing cousin of the traffic-shifting problem here. What's new is the idea that redundancy nobody has tested is not actually redundancy, and that the fix is a control plane whose entire job is to keep proving it, in production, against real user traffic.
**Stack relevance:** Rare.lab does not need three AWS regions yet, and building them now would be premature. What transfers immediately is the discipline, not the topology. Rare.lab has its own single points of contact today: one Supabase Postgres project behind RLS, one Cloudflare R2 bucket holding content-addressed scene JSON, one embeddable runtime sharing a single WebGL context across everything a customer's page renders through it. This lesson's real lesson is that "we have a backup" and "we have a tested backup" are different claims, and only deliberately, regularly exercising the failure path (killing a shader-compile worker on purpose, cutting the runtime off from R2 on purpose, watching what a tenant's failed render does to the shared WebGL context) turns the first claim into the second, long before three-region failover is anywhere on the roadmap. Ties directly to Day 50 (sandboxed multi-tenant execution) and Day 67 (multi-tenancy and the noisy-neighbor problem): a failure in one tenant's shader job must not be allowed to cascade into the shared context the way an unscaled survivor region collapses under a neighbor's traffic.

---

## 1. The company and the breaking number

**Netflix, and the 7 hours it could not fix itself.** On December 24, 2012, Netflix streaming broke for large parts of the United States, Canada, and Latin America, most visibly on TV-connected devices, for roughly seven hours on the single highest-traffic night of the year. The proximate cause was not Netflix's own code. It was Amazon's Elastic Load Balancing (ELB) service: an internal AWS maintenance process was run against production ELB state data and deleted it, corrupting the load-balancer configuration that routed traffic to Netflix's services in that region. Netflix has been explicit in its own retrospective that the root cause sat entirely inside AWS's control plane, a layer Netflix does not own and cannot patch. Waiting for someone else to fix a broken control plane, on Christmas Eve, is not a strategy.

**The number that makes "just fail over" harder than it sounds.** By the mid-2010s Netflix was running active-active across three AWS regions (us-east-1 in Virginia, us-west-2 in Oregon, and eu-west-1 in Dublin), each one taking live production traffic all the time, not sitting cold as a backup. Roughly a third of Netflix's global traffic normally lands on any one of those three regions. If one of them disappears in an instant, whether from an ELB control-plane failure like Christmas Eve 2012 or from a full outage of the region itself, the other two do not each need to find a little spare room. Each of them needs to instantly absorb something on the order of 50% more traffic than it was sized for a moment earlier, because a third of the world's requests just landed on the remaining two-thirds of the fleet. Ordinary reactive autoscaling, the kind that watches live CPU and request-rate metrics and launches more instances in response, realistically takes minutes, not seconds, to notice the surge, boot new EC2 instances, and get an application warmed up and serving. A naive failover has minutes of exposure with nothing standing in the gap.

**The scale this has to survive.** By 2015, industry measurement firm Sandvine reported that Netflix alone accounted for roughly 37% of all downstream internet traffic in North America during peak evening hours, more than any other single service on the internet at the time. A regional outage for a service carrying more than a third of a continent's peak downstream bytes is not a minor incident to route around quietly; it is immediately visible, at internet-measurement scale, to millions of people mid-stream.

---

## 2. Why the naive (demo) design dies

**The obvious version:** run active-passive. Keep one region hot, handling all production traffic, and keep one or two other regions as cold or lightly-provisioned standbys. If the primary region ever goes down, a human notices, spins up capacity in a standby region, and manually repoints DNS once things look ready.

**Death one: a cold region takes hours to become a hot one, and Christmas Eve 2012 is the proof.** Booting a fleet of instances from zero, letting autoscaling groups converge, warming JVM heaps and application caches, and replicating enough recent data into a region that was not actively serving traffic a moment ago is not a five-minute job; it is realistically a multi-hour one when it has to start from a standing stop under pressure. Netflix's own Christmas Eve incident ran for about seven hours precisely because the machinery to move real traffic between regions in minutes did not yet exist at that point; the company had other regions available in principle, but no fast, rehearsed way to actually use them.

**Death two: an instant, full cutover just relocates the outage to whichever region catches it.** If an operator (or a script) simply flips 100% of the dead region's DNS to the survivors the moment trouble is noticed, those survivors, each sized for roughly their own third of global traffic, are suddenly asked to serve about 150% of what they were provisioned for, immediately, with no ramp. Connection pools exhaust, caches that were sized and warmed for the old load thrash, and the "rescuer" region falls over from the exact same kind of overload that may have taken down the first one, this time self-inflicted. This is the thundering-herd failure Day 13 already named, arriving here in the shape of a well-intentioned failover.

**Death three: DNS is not a switch, it is a slow and partially unreliable diffusion process.** Updating a DNS record does not instantly move traffic. Clients cache resolved addresses, and a meaningful fraction of ISP resolvers ignore or extend the record's advertised time-to-live, so some slice of users keep getting routed to the dead region for an unpredictable extra stretch of time no operator directly controls. Treating "change the DNS record" as the entire failover mechanism means accepting an uncontrolled, long tail of continued failures on top of the two failure modes above, precisely while everyone believes the incident is already resolved.

**The real-world version:** Netflix's own Christmas Eve 2012 outage is this scenario, not a hypothetical. The company had cloud infrastructure and, in principle, other regions, but no rehearsed, fast, production-tested path to actually move live traffic off a broken region in minutes. The direct consequence was a multi-year investment, starting with a two-region stopgap called Isthmus in mid-2013, then full three-region active-active later that year, then years of hardening the evacuation mechanism itself (Chaos Kong in 2014-2015, later reworked as "Project Nimble") into something exercised on a schedule against live production traffic, not something read off a runbook for the first time during an actual disaster.

---

## 3. The architecture

```
Clients worldwide (subscribers on TVs, phones, browsers)
  - job: connect to whichever region the current DNS answer points
    to, and retry politely if a connection fails
  - analogy: dialing a company's published phone number and trusting
    today's receptionist to route the call correctly

        |
        v
Global DNS layer (geo- and weighted-routing DNS, e.g. UltraDNS)
  - job: hold the current mapping from "which region should this
    user's traffic go to," changeable by automation without any
    change on the client side, but propagating on the order of
    minutes because of client and resolver caching
  - analogy: a company repainting the address on its storefront
    sign, even though some regular customers won't notice the new
    sign for a while and will still show up at the old address

        |
        v
Regional edge gateway (Zuul, one deployment per region, capable of
tunneling to peer regions over a private backend link)
  - job: absorb all inbound traffic addressed to its region, and
    during an evacuation, proxy a controlled, steadily increasing
    slice of that traffic to a healthy peer region before DNS has
    even finished propagating the official change
  - analogy: a store whose front doors stay exactly where they are,
    while staff quietly walk a growing share of customers through a
    back hallway to a sister store while the new sign is still
    being painted

        |
        v
Stateless application tier (per region, autoscaled independently)
  - job: serve every request using only in-region dependencies, so
    normal operation never requires a cross-region network hop
  - analogy: each branch office fully equipped to do business on
    its own, not merely an extension line ringing through to a
    head office somewhere else

        |
        v
Service discovery and client-side routing (Eureka, made aware of
peer regions' registries; Ribbon doing client-side IPC routing)
  - job: let a caller find a live instance directly from a cached,
    client-held registry, defaulting to in-region but able to route
    to a peer region's registered instances on command, without
    depending on a regional load balancer's own control plane
  - analogy: employees keeping their own address books of direct
    lines to coworkers, so a company phone directory outage doesn't
    stop anyone from reaching who they need to reach

        |
        v
Predictive plus reactive autoscaling (Scryer forecasting from
recurring historical weekly/daily traffic curves; AWS Auto Scaling
reacting to live metrics on top of that forecast)
  - job: have most of a survivor region's extra needed capacity
    already booted and warm before an evacuation even starts,
    then fill in whatever the forecast under- or over-shot
  - analogy: a restaurant calling in extra staff ahead of a holiday
    rush it has seen every year, then calling a couple more in only
    if the night runs busier than even that forecast expected

        |
        v
Multi-region data layer (per-region Cassandra ring, asynchronous
cross-region replication, LOCAL_QUORUM reads and writes)
  - job: let each region read and write against its own local
    replicas without waiting on a round trip to another region for
    every operation, while quietly catching every other region up
    in the background
  - analogy: three branch offices each keeping their own current
    ledger and comparing notes with each other overnight, rather
    than phoning another branch before recording every transaction

        |
        v
Evacuation control plane (Chaos Kong / later Project Nimble)
  - job: run the full sequence, pre-scale the survivors, ramp the
    proxied traffic percentage, then cut DNS, on a schedule, against
    live production traffic, whether or not anything is actually
    broken that day
  - analogy: a fire drill that empties the real building during a
    real work day, not a tabletop exercise walked through on a
    whiteboard once a year
```

---

## 4. The transferable mechanisms

- **N+1 capacity as insurance you pay for continuously.** Every region is provisioned so that if any one peer disappears, the others can absorb its full share, not merely "some" of it. This is not free: it means running meaningfully more total capacity, all the time, than steady-state traffic alone would require. The insurance only pays off on the rare day it's needed, which is exactly why it is tempting, and dangerous, to quietly under-fund it when nothing has gone wrong in a while.

- **Progressive traffic migration instead of an instant cutover.** Zuul's cross-region tunnel lets traffic move from a sick region to a healthy one gradually, as a rising percentage, rather than as a single all-at-once DNS flip. This gives reactive autoscaling in the survivor region time to actually catch up to demand as it grows, turning what would otherwise be an instant 50% surge into a ramp the system can absorb without collapsing. It's the same instinct as Day 64's virtual waiting rooms: don't let demand arrive faster than the system can be told to grow.

- **Bypass the exact control plane that just failed.** Eureka's client-side service discovery, paired with Ribbon doing the routing decision on the caller's own machine, means a service can find and call a live instance directly from a locally cached registry, without depending on the health of the regional load balancer itself. Christmas Eve 2012 failed specifically because the ELB control plane broke; a mechanism that routes around ELB entirely survives exactly that failure mode. This is the same "decide locally, don't phone a shared authority mid-request" instinct Day 71 used for edge rate limiting, applied here to service discovery instead of counting.

- **Predictive scaling to remove the boot-time tax from the critical path.** Scryer forecasts near-term capacity needs from recurring historical traffic curves (the same weekday tends to look like the same weekday the week before and the week after) and pre-provisions ahead of time, so an evacuation's reactive autoscaling only has to correct a forecast, not build capacity from a standing start. This is the same pre-warm-before-the-spike idea this ledger has used for sale-day and inventory-contention scenarios, aimed at infrastructure capacity instead of stock.

- **Deliberately breaking production, on a schedule, is itself the reliability mechanism.** Chaos Kong doesn't wait for an incident to test whether regional evacuation works; it runs the real evacuation against real live traffic on a recurring cadence (Netflix has described running full region-level failover tests on a roughly monthly rhythm), turning "we believe our redundancy works" into "we watched it work, again, this month." A failure discovered by a scheduled drill costs an engineering afternoon. The same failure discovered during a real regional outage costs the incident this whole lesson opened with.

---

## 5. The trade-offs

**Consistency versus availability, chosen per data type, exactly the way this ledger has seen before.** Netflix's Cassandra rings replicate asynchronously across regions and serve reads and writes at LOCAL_QUORUM, meaning each region can keep reading and writing its own local replicas without ever blocking on a cross-region round trip, and it may take a short window (typically well under a second in normal operation, longer during an actual regional event) for another region's replicas to catch up. For viewing history, a "continue watching" position, or a recommendation input, that staleness window is a non-event; nobody notices their watch progress lagging by a moment across regions. Contrast this directly with Day 6's Stripe ledger, where the same looseness applied to a money balance would be unacceptable, because double-spending is a different order of problem than a slightly stale "resume point." Same CAP dial. Netflix sets it toward availability here because the data type allows it; Stripe sets it toward consistency for its data type because it doesn't.

**Cost versus latency and recovery time, paid as standing idle capacity.** Running three full regions, each provisioned with enough headroom to absorb a peer's entire load rather than just its own, is strictly more total compute purchased and paid for around the clock than a single-region deployment, or even three regions sized only for their own individual share, would cost. That extra spend buys a roughly 6-to-7-minute recovery time on the rare day an entire region needs to be evacuated, instead of the multi-hour recovery Christmas Eve 2012 actually delivered when that headroom and machinery didn't yet exist. For a company whose product is watched live, minute by minute, by tens of millions of concurrent households at peak, that trade is not a close call.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is **latent-failure normalization**: a system's redundancy quietly rots the moment nobody is forced to prove it still works. Configuration drifts, capacity margins shrink as traffic grows against a scaling policy nobody revisited, a runbook goes stale as the architecture changes underneath it, and none of this shows up anywhere, because the failure path it would expose simply never gets exercised. Confidence keeps accumulating on a foundation nobody is actually testing, right up until the day a real regional outage asks the question for the first time in production, under pressure, with actual customers watching, which is the worst possible moment to discover the answer is no.

There is a second, faster-acting loop layered on top of it: the thundering-herd risk that shows up the moment a naive, untested failover actually gets triggered, exactly as Death Two in Section 2 described. An instant, all-at-once cutover doesn't just risk failing; it actively manufactures the overload that makes the survivor regions fail too, converting one region's bad day into three regions' bad day.

The senior fix does not add more capacity and hope, and it does not just write a better runbook. It breaks both loops directly by making failure-path exercise routine rather than exceptional: Chaos Kong runs the actual evacuation, against actual live production traffic, on a recurring schedule, which forces configuration drift, stale capacity margins, and outdated runbooks to surface as a mildly annoying finding this month instead of a multi-hour customer-facing outage some month. And the evacuation mechanism itself is engineered, via progressive proxying rather than an instant DNS flip, so that the drill (and the real event it's rehearsing for) never becomes the second failure mode it's supposed to be testing for. The system doesn't get bigger. It gets continuously, deliberately re-proven.

---

## Sources

- [A Closer Look at the Christmas Eve Outage, Netflix TechBlog](https://netflixtechblog.com/a-closer-look-at-the-christmas-eve-outage-d7b409a529ee): Netflix's own retrospective on the December 24, 2012 outage, the primary source for the ELB control-plane root cause (production ELB state data deleted by an internal AWS maintenance process) and the framing that the failure sat outside Netflix's own ability to fix. Direct fetch was blocked by this session's network egress policy; the account here is drawn from search-indexed summaries and contemporaneous press coverage rather than a full direct read.
- [Updated: Netflix Crippled On Christmas Eve By AWS Outages, TechCrunch](https://techcrunch.com/2012/12/24/netflix-crippled-on-christmas-eve-by-aws-outages/): contemporaneous, independent press corroboration of the outage's timing, duration, and user-facing impact (primarily TV-connected device playback across the US, Canada, and Latin America). Direct fetch blocked this session.
- [Isthmus — Resiliency against ELB outages, Netflix TechBlog](http://techblog.netflix.com/2013/06/isthmus-resiliency-against-elb-outages.html): primary source for the first, two-region stopgap architecture built directly in response to the Christmas Eve outage, including Zuul as the cross-region traffic bridge, Eureka made aware of a peer region's registered instances, and Ribbon doing client-side routing so callers can bypass a failed regional load balancer entirely. Direct fetch blocked this session; described here from search-indexed summaries.
- [Active-Active for Multi-Regional Resiliency, Netflix TechBlog](http://techblog.netflix.com/2013/12/active-active-for-multi-regional.html): primary source for the full three-region (us-east-1, us-west-2, eu-west-1) active-active architecture that followed Isthmus, and for the Cassandra multi-region replication approach. Direct fetch blocked this session.
- [Chaos Engineering Upgraded, Netflix TechBlog](http://techblog.netflix.com/2015/09/chaos-engineering-upgraded.html): primary source for Chaos Kong, the tool and practice of deliberately evacuating a full, live production AWS region on a recurring basis to verify regional failover actually works. Direct fetch blocked this session.
- [Evolving Regional Evacuation, Netflix TechBlog (Niosha Behnam)](https://netflixtechblog.com/evolving-regional-evacuation-69e6cc1d24c6): primary source for the later, hardened three-step evacuation sequence (pre-scale survivor regions, progressively proxy traffic via Zuul, then cut DNS) and for Scryer-style predictive pre-scaling ahead of an evacuation. Direct fetch blocked this session; described here from search-indexed summaries and the associated Hacker News discussion.
- [Project Nimble: Region Evacuation Reimagined, Netflix TechBlog, mirrored on Medium](https://medium.com/netflix-techblog/project-nimble-region-evacuation-reimagined-d0d0568254d4): primary source for the later reworking of the evacuation orchestrator itself. Direct fetch blocked this session.
- [How Netflix does failovers in 7 minutes flat, Opensource.com (write-up of a PyCon 2018 talk)](https://opensource.com/article/18/4/how-netflix-does-failovers-7-minutes-flat): secondary source corroborating the roughly 6-to-7-minute end-to-end failover time, the three-step sequence (scale up survivor regions, proxy traffic, then change DNS), and the role of DNS TTL-driven client caching in why the final cutover step alone takes several minutes. Direct fetch blocked this session; used here via search-indexed summary.
- [Streaming services now account for over 70% of peak traffic in North America, Netflix dominates with 37%, VentureBeat](https://venturebeat.com/media/streaming-services-now-account-for-over-70-of-peak-traffic-in-north-america-netflix-dominates-with-37/): secondary source reporting Sandvine's 2015 measurement of Netflix's roughly 37% share of North American peak downstream internet traffic, used here for scale context on why a Netflix regional outage is immediately visible at internet-measurement scale.
- [Netflix boasts 37% share of Internet traffic in North America, compared with 3% for Apple's iTunes, AppleInsider](https://appleinsider.com/articles/16/01/20/netflix-boasts-37-share-of-internet-traffic-in-north-america-compared-with-3-for-apples-itunes): a second secondary source corroborating the same Sandvine figure.
- Day 6 (Stripe correctness under load), Day 13 (backpressure and load shedding), Day 14 (multi-region active-active), Day 22 (leaderless replication, quorums, and vector clocks), Day 50 (sandboxed multi-tenant code execution), Day 55 (anycast global server load balancing), Day 64 (inventory contention and virtual waiting rooms), Day 67 (multi-tenancy and the noisy-neighbor problem), Day 71 (distributed rate limiting at the edge): the ledger's own prior lessons this one directly reuses and recombines.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of every Netflix engineering blog post cited above, so the specific architectural sequence (Isthmus's client-side bypass of a failed ELB, the three-region active-active layout, Chaos Kong's recurring live-region evacuations, and the later pre-scale-then-proxy-then-DNS sequence) is summarized here from search-indexed excerpts, a conference-talk write-up, and independent press coverage rather than quoted from a full direct read of the primary sources, consistent with how this ledger has flagged network-blocked sources before (see Day 69, Day 70, and Day 71). The core claims are treated as solid because they are corroborated across Netflix's own multiple posts on the subject over several years, an independently reported PyCon talk describing the same mechanism, and contemporaneous press coverage of the originating Christmas Eve 2012 incident, even where the primary documents could not be fetched directly this session.
