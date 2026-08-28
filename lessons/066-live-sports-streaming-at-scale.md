# Day 66 — How does JioHotstar keep a cricket match live for 25 million people at once, when a new server takes 90 seconds to boot and Dhoni can double the traffic in half that time?

**Date:** 2026-08-28
**Difficulty:** Expert
**Topic:** Live video streaming at national-broadcast scale, and why it is a fundamentally different engineering problem from the video-on-demand storage and delivery this ledger already covered in Day 5 (YouTube) and Day 4 (CDN egress). A VOD system has minutes or hours to transcode a file and pre-warm a CDN before anyone watches it. A live sports broadcast has none of that: the same second of cricket has to be encoded, packaged, and pushed to tens of millions of screens inside a latency budget of a few seconds, while the audience size itself swings by tens of millions of people inside a single over, driven by events inside the match, not by anything an engineer can forecast on a normal traffic curve. The forcing numbers are real and disclosed: Disney+ Hotstar (now JioHotstar after the Reliance-Disney India merger) recorded 25.3 million concurrent viewers during the India vs New Zealand 2019 World Cup semifinal, a live-stream concurrency record at the time, and the same match is independently documented as the source of the load-bearing operational fact this lesson is built around, that a standard AWS Auto Scaling Group takes roughly 90 seconds for a freshly launched instance to pass health checks and start serving traffic, while live-cricket traffic has been observed to double in well under that window the instant a wicket falls or a marquee batter walks in. Disney+ Hotstar later served 59 million concurrent viewers for the 2023 ODI World Cup final and 53 million for the 2024 T20 World Cup final; JioCinema separately drew over 32 million concurrent viewers for the 2023 IPL final. The pattern repeats every season, at a bigger number each time, which is exactly what makes it worth studying as a system, not a one-off event.
**Stack relevance:** Rare.lab's runtime shares one WebGL context across an embedded session, the same shared-expensive-resource shape Day 49 (WebRTC SFU) already mapped for audio and video. Live sports streaming adds the piece Day 49 and Day 5 each only had half of: Day 49 solved fanning out one live resource to many viewers cheaply; Day 5 solved encoding once and caching the result at the edge for later, cold, unpredictable demand. Live sports has to do both at once, encode continuously and fan out immediately, while the audience curve itself is driven by events inside the content, not outside it. Rare.lab's own closest future analog is a live, shared runtime session, several viewers watching one shader scene render in real time inside an embed, where a burst in viewers is driven by something happening inside the page (a scene going viral, a live demo link shared on stage) rather than by a scheduled traffic ramp. The lesson to bank now, before that need exists: never scale the thing people are watching (the live compute or render) on the same timeline as the thing you scale for the number of people watching it (the fan-out tier), because those two problems have completely different failure clocks.

---

## 1. The company and the breaking number

**Disney+ Hotstar (now JioHotstar), India vs New Zealand, 2019 Cricket World Cup semifinal.** The platform recorded 25.3 million concurrent viewers, the largest concurrent live-stream audience recorded anywhere at the time, on a single match, on a single day, with a documented zero-downtime outcome. That headline number is well corroborated across independent reporting and Hotstar's own AWS re:Invent presentation from the same year (session CMY302, "Scaling hotstar.com for 25 million concurrent viewers"). The number is not the interesting part on its own; plenty of systems serve tens of millions of requests a second in steady state. The interesting part is the shape of the curve inside that match.

**The shape, not just the peak, is the actual engineering problem.** Independent accounts of that same match describe MS Dhoni walking in to bat and traffic spiking sharply as viewers who had the app open in the background, or were watching on TV and switching over, converged on the stream within seconds of the moment being announced on commentary and on social media. When Dhoni was dismissed shortly after, viewership is reported to have collapsed from the 25.3 million peak down to under 1 million within minutes, as a very large fraction of the audience was there for one specific batter's innings and left the instant it ended. A system built to handle "25 million people, arriving smoothly over two hours" is not the same system as one built to handle "25 million people, most of whom arrive or leave inside a five-minute window, more than once, at times nobody can schedule in advance, because they are triggered by what happens on the field."

**The 90-second number is the one that actually breaks a naive design.** A standard AWS Auto Scaling Group, reacting to a CPU or request-count threshold, provisions a new EC2 instance, waits for it to boot, waits for application code to initialize, and waits for a load balancer health check to pass, before that instance is allowed to take a single request. Documented accounts of Hotstar's infrastructure work put that end-to-end window at roughly 90 seconds under realistic conditions. Cricket does not wait 90 seconds. A wicket, a six, a marquee player coming to the crease, each of these is a single discrete event broadcast simultaneously on television, on social media, and via push notification, and each one is capable of roughly doubling concurrent traffic in well under 90 seconds. A capacity mechanism that takes 90 seconds to add one more increment of capacity, reacting to a threshold that traffic can blow past in 30 or 45 seconds, is not slow, it is structurally unable to keep up with the thing it is supposed to protect against, the same category of failure this ledger named for the AWS ASG step-size mechanism in general terms back in Day 13 (backpressure and load shedding), now with a specific, disclosed number attached to a specific, real match.

Later seasons pushed the same shape further. Disney+ Hotstar served 59 million concurrent viewers for the 2023 Cricket World Cup final, a record at the time; JioCinema drew over 32 million concurrent viewers for the 2023 IPL final on a separate platform, and Disney+ Hotstar recorded 53 million concurrent for the 2024 T20 World Cup final. Every one of these events has the same internal shape as the 2019 Dhoni spike: a national audience, a shared real-time trigger, and a system that has to absorb a demand curve set by cricket, not by capacity planning.

---

## 2. Why the naive (demo) design dies

**The obvious version:** a broadcast truck at the stadium encodes the camera feed and pushes it as a single RTMP stream to one origin server, which transcodes it into a handful of bitrates and hands playback manifests directly to client apps, all served from one region, scaled the same way any other stateless web tier is scaled, an Auto Scaling Group watching CPU and adding instances on a threshold. This is exactly how live streaming starts for any product that first needs "a live tab," because it works fine for a feature with a few thousand viewers, and video-on-demand infrastructure the company may already have (Day 5's transcoding and CDN pipeline) looks, on paper, reusable for live.

**Death one: the encode-and-package pipeline is a single point of failure with no time to recover.** In VOD, if a transcoding job fails, it retries and the video is available a few minutes later than planned, invisible to anyone. In live, if the transcoding pipeline for the primary feed fails for even 10 seconds, that 10 seconds of cricket is gone forever for every single viewer simultaneously, there is no "try again," because there is no earlier copy of a moment that has not happened yet. A design with one encoder, one packager, and no redundant standby pipeline turns any transient fault, a process crash, a bad deploy, a single flaky host, into a simultaneous outage for the entire audience, at whatever moment in the match it happens to occur.

**Death two: reactive autoscaling cannot outrun an event-driven demand curve.** The 90-second Auto Scaling Group boot time from Section 1 is the concrete version of this death. A threshold-based scaler is built for gradual traffic growth, the pattern most web applications actually see, where a slow ramp gives the scaler time to add capacity ahead of demand. Cricket delivers the opposite pattern deliberately: traffic doubling in under a minute, driven by an event visible to the entire audience at the same instant (a wicket falling is announced on commentary, on the scoreboard, and on social media all at once). By the time a CPU-threshold alarm fires, a new instance is requested, boots, and passes a health check, the spike that triggered the alarm has often already passed its peak or the service has already degraded for the users caught in the gap.

**Death three: a single-region origin cannot serve a national audience inside a live latency budget.** Live sports has almost no queryable local cache the way a VOD library does, because the content that matters is the content that just happened, seconds ago, everywhere at once. A design that transcodes and packages in one region and serves every viewer in the country from that one region's edge either accepts round-trip latency that makes the stream lag noticeably behind live television (a real, user-visible failure for sports, where a neighbor's shout at a six landing a second before your own screen shows it is an immediate, visceral sign that the product is broken), or it overloads that single region's egress capacity trying to serve a national audience from one place, the exact zero-origin-egress problem Day 4's CDN lesson described, now compounded by the fact that live content cannot be pre-warmed at the edge before anyone watches it the way a VOD catalog can.

**Death four: treating every feature as equally important means everything degrades together.** A monolithic app tier that serves live video playback, personalized recommendations, match statistics, social features, and login all through the same undifferentiated request path has no way to shed the parts of the product that do not matter during a capacity crunch. When that shared tier saturates, a request for the actual video stream waits in the same queue as a request for a personalized recommendation carousel, and both degrade together, even though only one of those two things is the reason 25 million people opened the app in the first place.

---

## 3. The architecture

```
Broadcast source (stadium cameras, commentary feed)
  - job: produce the single highest-quality video/audio feed, the
    one and only source of truth for "what actually happened," with
    no substitute if it is lost even briefly
  - analogy: the live orchestra itself, not any recording of it

        |  primary + redundant standby encoder paths
        v
Redundant ingest and encode tier (contribution encoding)
  - at least two independent encode paths from the venue, often via
    different network routes, so a single encoder crash or a single
    link failure does not blank the feed for the entire audience;
    output is one high-bitrate mezzanine stream, not yet the
    consumer-facing bitrate ladder
  - job: get one durable, redundant copy of the raw live feed off
    the truck and into the cloud, with automatic failover if the
    primary path drops
  - analogy: two independent phone lines to the newsroom, so a cut
    cable on one street doesn't go dark, not one line with a backup
    plan nobody has tested

        |  mezzanine stream
        v
Real-time transcoding into an adaptive bitrate (ABR) ladder
  - the single incoming feed is transcoded, continuously, into
    several parallel bitrate/resolution rungs (for example roughly
    6 Mbps HD down to a few hundred kbps for a poor connection),
    each cut into short segments (commonly 2-6 seconds) as they are
    produced, not after the whole video exists, because in live
    there is no "whole video" yet
  - job: give every device, on every network quality, a rung it can
    play smoothly, the same adaptive-bitrate mechanism Day 5 named
    for stored video, now computed continuously instead of once
  - analogy: a print shop running the same breaking headline
    simultaneously on the giant press, the tabloid press, and the
    single-page flyer copier, all from the one story as it's typed

        |  segments + manifest, published continuously
        v
Origin packaging (HLS/DASH manifest + segment store)
  - wraps each new segment into the streaming protocol's manifest
    (the playlist telling a player which segment to fetch next),
    updated every few seconds as new segments land
  - job: the one place that knows, at any given second, which
    segment is "now," so every downstream cache and every client
    player agrees on the same live edge
  - analogy: a newsroom's live wire feed, stamped with the exact
    order stories go out, so no two desks report events out of order

        |  fanned out, not fetched individually per viewer
        v
Multi-tier CDN (edge caches, geo-distributed)
  - the same zero-origin-egress shape Day 4 described for VOD,
    except every cached segment is only useful for a few seconds
    before it's stale, so cache hit rate depends on many viewers
    requesting the same fresh segment inside a tiny window, not on
    long-lived popularity
  - job: absorb the actual viewer count without every one of 25
    million players hitting the origin directly for the same few
    seconds of video
  - analogy: a village crier's message relayed street by street,
    instead of the mayor personally repeating it to every household

        |  playback + non-video API calls
        v
Stateless app tier, split by criticality (playback vs everything else)
  - playback-critical APIs (auth, entitlement checks, the manifest
    URL, the CDN token) are served by a dedicated, minimally-loaded
    path; personalization, recommendations, social features, and
    match-stats overlays are served by a separate tier that can be
    shed independently
  - job: make sure the thing 25 million people actually came for,
    the video, never queues behind the thing that is nice to have
  - analogy: an airport keeping the runway and the gate desk staffed
    at full strength while closing the duty-free shop during a storm

        |  request-rate/concurrency-driven, not CPU-driven
        v
Predictive + custom autoscaling ("pre-warm before kickoff")
  - capacity for a known live event is provisioned ahead of time
    from forecast models and coordinated cloud-provider capacity
    reservations made months in advance, then a custom scaling
    engine, keyed on request rate and concurrency rather than raw
    CPU, adds increments fast enough to matter inside a live event's
    actual reaction window, rather than the 90-second ASG default
  - job: have the next unit of capacity ready inside single-digit
    seconds of a demand spike, not 90 seconds after one starts
  - analogy: a stadium opening every turnstile and having extra
    staff already standing by 30 minutes before kickoff, rather than
    calling more staff in only once the queue is visibly backed up

        |  when load still exceeds capacity
        v
Tiered graceful degradation (load shedding by feature priority)
  - a standing, pre-agreed priority order for what gets turned off
    first under load: personalized banners, recommendations, and
    viewing history first; match statistics and secondary content
    APIs second; live video delivery, the playback API, and
    authentication last, never
  - job: fail cheaply and by design, on the parts of the product
    that do not matter in the moment, instead of failing expensively
    and by accident, on the part that does
  - analogy: a ship's crew sealing off flooded compartments in a
    fixed order, decided long before any water gets in, so the parts
    that keep it afloat are never the parts sacrificed first

        |  rehearsed continuously, not assumed
        v
Load testing and chaos engineering ("game day" before match day)
  - synthetic traffic generated from a large fleet of load-generating
    machines simulates tens of millions of concurrent sessions
    against the real production stack before a marquee match,
    combined with deliberate fault injection (killing hosts,
    degrading a region) to verify the degradation tiers and failover
    paths actually work under realistic concurrency, not just in a
    design document
  - job: find the 90-second gap, the untested failover path, and the
    silently-broken degradation tier days before 25 million real
    people find it for you, live, with no replay
  - analogy: a fire drill run at full capacity, with the actual
    doors and the actual crowd size, not a tabletop exercise
```

---

## 4. The transferable mechanisms

- **Pre-warm known-demand events instead of reacting to them.** For a scheduled live event, the demand curve's timing is known even though its exact magnitude is not; the fix is to provision capacity ahead of the event from forecasts and pre-negotiated cloud capacity, so the system starts the match already carrying most of the load it will need, rather than discovering the need reactively. This generalizes past sports to anything with a known start time and an unknown but large audience: a product launch, a live product demo, a scheduled sale (this ledger's Day 13 covered the same pre-warming instinct for e-commerce spikes).

- **Decouple the scaling clock of the produced resource from the scaling clock of its consumers.** The encode-and-package pipeline (one feed, continuously transcoded) scales on a completely different axis and timeline than the CDN and app tier serving viewers (millions of independent, bursty consumers of that one feed). Conflating the two, scaling the expensive shared thing being watched in lockstep with the number of people watching it, is the mistake this lesson's stack-relevance section flags for Rare.lab's own future live-viewing feature; Day 49's SFU lesson already established the fan-out half of this split for WebRTC, this lesson adds the production half.

- **A fixed, pre-agreed priority order for what gets shed under load.** Deciding in a war room, live, during an incident, what to turn off is slower and more error-prone than deciding it weeks in advance and encoding it as an automated, tiered policy (Tier 1 shed first: personalization and recommendations; Tier 2: secondary content APIs; Tier 3 never shed: playback and auth). This is the same principle Day 13's backpressure lesson named generally, load shedding as a deliberate, ranked policy rather than an undifferentiated collapse, now applied with a concrete, disclosed real-world tier list.

- **Redundant, independently-failing paths for anything with no replay.** Two independent encode paths from the venue exist because a live feed, unlike a stored file, cannot be re-fetched or retried after the fact; the general principle is that anything which is only available once, in a narrow real-time window, needs active-active redundancy built in from the source, not just retries downstream, because downstream retries have nothing to retry against once the moment has passed.

- **Custom, workload-aware autoscaling signals over generic ones.** Scaling on request rate and concurrency, signals that move in lockstep with the actual thing being protected against, rather than CPU utilization, a signal that can lag the real bottleneck, is what let a custom scaling engine react inside single-digit seconds instead of the 90-second default. The general lesson: pick the autoscaling signal that is causally closest to the failure you are trying to prevent, not the signal that happens to be easiest to read off a dashboard.

- **Rehearse the failure, don't just design for it.** Load testing at real target concurrency, combined with deliberate chaos engineering (killing hosts, degrading regions on purpose), before the real event is what turns "we designed a degradation tier" into "we know the degradation tier actually works," the same gap this ledger's Day 34 (backpressure and load shedding's sibling lesson on cascading failure) implicitly assumed was already closed by testing, made explicit here as its own mechanism.

---

## 5. The trade-offs

**Latency vs. absolute correctness of "live," and the industry has settled on a few seconds of lag as the accepted price.** A live stream is never truly zero-latency; segment-based protocols like HLS and DASH batch video into multi-second chunks specifically because that batching is what makes CDN caching, and therefore serving tens of millions of viewers, possible at all. The trade-off is explicit: a fully real-time protocol (closer to what a video call uses) would remove that lag but would also remove the ability to cache segments at the edge for many viewers at once, forcing something closer to one connection per viewer back to a central point, the exact scaling wall this ledger's Day 49 WebRTC lesson described peer-to-peer mesh calls hitting past four to six participants. A few seconds of delay behind the TV broadcast is accepted as the cost of serving a national audience from a cache-friendly protocol instead of a real-time one.

**Cost vs. headroom, paid months in advance rather than reactively.** Coordinating with cloud providers to reserve extra capacity ahead of a marquee match, and running large-scale synthetic load tests against production infrastructure, both cost real money for capacity and compute that may sit partly idle outside of marquee events. This is accepted because the alternative, discovering a capacity ceiling live, during the one match with the largest audience of the year, converts a cost line item into a headline outage in front of the exact audience the entire product exists to serve.

**Personalization vs. availability, decided in advance, not negotiated in the moment.** Every feature that makes the product richer day to day, recommendations, personalized banners, viewing history, is explicitly the first thing sacrificed the moment capacity is threatened. This is a deliberate ranking of what the product is for during a crisis: during a World Cup final, the product is a live video stream with authentication, nothing else, and every other feature is optional by design, decided ahead of time so nobody has to argue about it while traffic is doubling.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is a **reactive-autoscaling death spiral triggered by a correlated, external event, not by the system's own load.** Most of this ledger's prior capacity-failure lessons (Day 13's backpressure, Day 63's presence reconnect storm, Day 64's inventory contention, Day 65's cardinality explosion) describe loops the system itself generates internally, retries causing more retries, reconnects causing more reconnects. This one is different in an important way: the trigger is external and instantaneous, a wicket falling is a single real-world event visible to the entire audience at the same second, so the demand spike it causes has none of the gradual ramp-up that lets a reactive threshold-based scaler catch up. The loop runs like this: a wicket falls, tens of millions of already-open or newly-opening app sessions converge on the stream within seconds, request volume crosses the autoscaling threshold, a new instance is requested, and by the time that instance boots and passes health checks roughly 90 seconds later, the spike has already saturated existing capacity, degraded playback for viewers caught in the gap, and, in a worse version of this failure than Hotstar's own documented outcome, could trigger cascading retries from client players re-requesting manifests and segments, adding load on top of the load that caused the problem in the first place, the same retry-storm shape Day 13 named generally.

The naive fix, provision more baseline capacity and hope it's enough, does not break this loop, it only raises the size of spike needed to overwhelm it, the same point this ledger has made about every prior version of this lesson: adding capacity does not fix a loop whose root cause is a reaction time mismatched to the trigger's actual speed. The senior fix here is the two mechanisms Section 3 named together: pre-warm capacity ahead of a known event using forecasts and reserved capacity, so the system is not starting from a cold reactive posture at all when the spike arrives, and replace the generic CPU-threshold trigger with a custom scaler reacting to request rate and concurrency fast enough to add real capacity inside single-digit seconds, closer to the actual speed of the trigger than to the speed of a general-purpose cloud default built for gradual web traffic. Combined with the tiered degradation policy from Section 3 as the last line of defense, the loop is broken in two places at once: before it starts (pre-warming removes the cold-start gap) and during it (fast custom scaling plus pre-agreed shedding keeps a saturating spike from cascading into every feature at once).

---

## Sources

- [CMY302 — Scaling hotstar.com for 25 million concurrent viewers, AWS re:Invent 2019](https://d1.awsstatic.com/events/reinvent/2019/Scaling_Hotstar.com_for_25_million_concurrent_viewers_CMY302.pdf): the primary AWS conference source naming the 25.3 million concurrent viewer figure and Hotstar's own account of scaling its infrastructure for that match; referenced here via search indexing, direct fetch was blocked by this session's network egress policy, so details drawn from it are cross-checked against the independent secondary sources below rather than quoted from the deck directly.
- [Hotstar IPL 2019: Scaling to 25 Million Concurrent Users, techlogstack.com](https://techlogstack.com/explore/hotstar-ipl-scaling-2019/): secondary engineering write-up corroborating the 2019 concurrency record and the general shape of Hotstar's scaling approach for that season.
- [T20 World Cup 2024: India-South Africa final match records peak concurrent viewership of 5.3 cr, Deccan Herald](https://www.deccanherald.com/sports/cricket/t20-world-cup-2024-india-south-africa-final-match-records-peak-concurrent-viewership-of-53-cr-3086917): source for the 53 million (5.3 crore) concurrent viewer figure for the 2024 T20 World Cup final on Disney+ Hotstar.
- [India's T20 World Cup final win draws 53m on Disney+ Hotstar, SportsPro](https://www.sportspro.com/news/t20-world-cup-final-2024-india-south-africa-disney-plus-hotstar-viewership-tv/): independent corroboration of the 2024 final's 53 million figure, and the reference point noting the 2023 ODI World Cup final's 59 million concurrent viewers as the prior record it fell short of.
- [How Disney Hotstar (now JioHotstar) Scaled Its Infra for 60 Million Concurrent Users, ByteByteGo](https://blog.bytebytego.com/p/how-disney-hotstar-now-jiohotstar): secondary engineering explainer discussed in search results as the source for the 90-second Auto Scaling Group provisioning-to-healthy window, the custom request-rate/concurrency-driven autoscaling engine built to react faster than that window, and the tiered graceful-degradation policy (Tier 1: recommendations/personalization/social features shed first; Tier 2: match stats and secondary content APIs; Tier 3: live video delivery, playback, and auth never shed); direct fetch of this URL was blocked by this session's network egress policy, so these details are presented as the clearly labeled industry-account version, corroborated by the independent secondary sources below rather than quoted from the original article.
- [Dhoni! The Nightmare to Streaming Services, Medium (Search It)](https://medium.com/search-it/dhoni-the-nightmare-to-streaming-services-755ac1715e8e): secondary account, surfaced via search, describing the specific traffic spike when Dhoni came to bat during the 2019 semifinal and the sharp drop in concurrent viewers reported after his dismissal, used here as the clearly labeled illustrative account of the event-driven demand curve this lesson is built around; direct fetch was blocked by this session's network egress policy.
- [JioCinema draws record 32 million concurrent viewers for the IPL 2023 final, reported across Indian business press]: figure corroborated across multiple independent search results for the 2023 IPL final's concurrent viewership on JioCinema, used as a second, separate platform's data point showing the same event-driven live-sports scaling problem recurs across companies, not just one.
- Day 4 (this ledger, CDN and zero-origin-egress) and Day 5 (this ledger, YouTube video storage and streaming): the ledger's own prior lessons this one builds directly on, for the adaptive-bitrate-ladder and edge-caching mechanisms reused here in a live rather than stored context.
- Day 13 (this ledger, backpressure and load shedding) and Day 49 (this ledger, WebRTC SFU real-time video at scale): the ledger's own prior lessons providing the general shape of, respectively, tiered load shedding under a demand spike and fan-out of one live resource to many simultaneous viewers, both specialized here to the live-sports case.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of several of the most detailed secondary sources (the ByteByteGo deep-dive, the AWS re:Invent PDF, and the Medium account of the Dhoni spike), so those sources are cited and summarized here based on the passages surfaced through search indexing rather than a full read of the original page, and are flagged as such rather than presented as directly verified quotes. The core numbers this lesson leans on hardest, the 25.3 million, 53 million, and 59 million concurrent-viewer figures, are independently corroborated across multiple unrelated outlets (AWS's own conference program, Deccan Herald, and SportsPro) and are treated as solid; the 90-second ASG figure and the specific tiered-degradation policy rest on a single secondary source surfaced via search and are treated as the labeled industry-account version rather than a company-disclosed fact with the same level of independent corroboration.

---
