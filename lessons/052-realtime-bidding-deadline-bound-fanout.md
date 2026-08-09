# Day 52 — How does an ad exchange pick a winner from 100+ bidders in under 100 milliseconds, every single time?

**Date:** 2026-08-09
**Difficulty:** Expert
**Topic:** Real-time bidding (RTB) auction infrastructure: the IAB OpenRTB protocol's hard 100–300ms bid-response window, why a sequential waterfall of demand partners cannot fit inside it, and how exchanges like Google's Authorized Buyers and Index Exchange use deadline-bound parallel fan-out, per-bidder error-rate circuit breaking, and an async settlement path to run a fair auction across hundreds of strangers' servers without ever blocking the page load on the slowest one.
**Stack relevance:** Rare.lab's embeddable runtime renders inside a fixed frame budget (16ms for 60fps), and any multiplayer or asset-fetch step that needs to gather input from more than one source before it can render, another peer's live edit, a remote LOD asset, a compiled-shader variant, hits the exact same shape as an ad auction: you cannot wait for the slowest source, you take what answered inside the deadline and move on.

---

## 1. The company and the breaking number

**The IAB Tech Lab's OpenRTB specification**, the protocol nearly every programmatic ad exchange on the web speaks, defines the contract as: a **supply-side platform (SSP)** sends a bid request describing an ad slot to many **demand-side platforms (DSPs)** at once, and each DSP has to send back a bid, or nothing, before a clock runs out. Per OpenRTB 2.6, that clock is hard-capped between **100 and 300 milliseconds**, and Google's own Authorized Buyers RTB documentation is blunter still: Google places a **100 millisecond timeout** on the whole round trip, and any bid that lands after that is simply discarded from the auction, full stop, no partial credit for being close.

That 100ms is not per bidder, it is the *entire* budget: network round trip plus every bidder's own decisioning logic (should I bid, how much, on whose behalf) combined. And a single ad request is not going to one bidder. **Index Exchange**, one of the larger SSPs, now reports handling roughly **700 billion ad requests a day** globally across thousands of servers in its own purpose-built cloud infrastructure, up from a widely cited 2019 figure of 50 billion a day, and each of those requests can legitimately be offered to well over a hundred DSPs simultaneously through header bidding. **The Trade Desk**, a major DSP, has described its own infrastructure as evaluating on the order of **9 to 12 million bidding decisions per second**, across more than 100 exchanges and SSPs, while holding sub-100ms decision latency on each one.

Put the numbers together and the breaking constraint is obvious: **one publisher's page load has 100 milliseconds to get a bid back from potentially 100+ independent companies' servers, each running their own model, on their own infrastructure, with their own occasional outages, and the auction has to close and render an ad regardless of who did or didn't answer in time.**

## 2. Why the naive (demo) design dies

The demo version of this system is a simple loop: the ad server has a list of demand partners, and for each ad slot it calls partner 1, waits for a response, calls partner 2, waits for a response, and so on, a classic **waterfall**, which is in fact how header bidding actually worked in its earliest form before parallelized "header bidding" replaced it.

**Sequential calls blow the time budget almost immediately.** Even a generously fast average bidder response of 50ms, and only 10 bidders in the waterfall, is 500ms, five times over budget, before a single millisecond of actual network latency variance is added. Real deployments run 100+ demand partners. A sequential design cannot fit more than one or two bidders inside 100ms and still leave time to actually render the page.

**One slow bidder blocks everyone behind it in line.** This is head-of-line blocking: if bidder #3 in the waterfall hangs for 400ms because its own backend is having a bad afternoon, bidders #4 through #100 never even get called this round, not because they are slow, but because the position ahead of them in the queue is. The publisher loses revenue from bidders who were perfectly healthy and ready to bid.

**No isolation between bidders means one bad actor degrades everyone.** If the ad server naively opens a fresh connection per bidder per request and that connection setup, DNS lookup, TLS handshake, is itself slow for one particular bidder's endpoint (a common real failure mode when a DSP's own infrastructure is under-provisioned or geographically distant), that latency is paid by the shared request-handling thread pool on the exchange's side, not sandboxed away from other bidders' calls.

**A single-threaded "wait for everyone" design means the slowest response sets the total latency for every user on the page**, exactly like a group photo where the shutter waits for the one person who is still tying their shoe. If nobody enforces a hard cutoff, the auction's total latency degrades to the worst bidder present that millisecond, not the median one, which is the opposite of what a fast page load needs.

## 3. The architecture

```
[Publisher's page / app: an ad slot needs to render]
   analogy: a newspaper classifieds page with one blank box, open to
   whoever's ad copy arrives at the print deadline
   |
   v
[Edge / SSP front door: receives the bid request, attaches context
 (page category, user segment, floor price), starts the auction clock]
   analogy: the auctioneer's gavel taps once, starting a countdown
   every bidder in the room can see
   |
   v
[Load balancer -> stateless auction orchestrator tier, horizontally
 scaled, one instance per incoming request, holds no state between
 requests]
   analogy: a bank of identical ticket windows, any window can serve
   any customer, so adding a window (a machine) adds capacity linearly
   |
   v
[DEADLINE-BOUND PARALLEL FAN-OUT: the orchestrator fires the SAME bid
 request to all eligible DSPs (100+) AT ONCE over already-open
 connections, not one after another]
   analogy: one auctioneer shouting the lot description to the whole
   room simultaneously, not walking bidder to bidder
   |
   v
[Per-bidder circuit breaker / callout quota: each DSP has its own
 rolling error-rate counter; a bidder whose timeout/error rate crosses
 a threshold (Google's documented figure: 15%) gets its callout volume
 throttled down automatically, rather than continuing to be hammered]
   analogy: a maitre d' who stops sending new tables to a kitchen
   station that's already sending back burnt plates, instead of
   overloading it further
   |
   v
[Cache: bidder health state, floor prices, campaign pacing counters,
 read on every auction, updated asynchronously, not written
 synchronously inside the 100ms window]
   analogy: a scoreboard everyone glances at mid-game, updated between
   plays, never blocking the play itself
   |
   v
[Collect-by-deadline: at T=~70-90ms (leaving headroom inside the
 100ms budget for the response to travel back and the ad to render),
 the orchestrator stops waiting. Bids that arrived are scored; bids
 that didn't are simply absent from this auction, not retried, not
 waited on]
   analogy: a game show's "time's up" buzzer; contestants who hadn't
   buzzed in yet just don't get to answer this round
   |
   v
[Second-price (or first-price, exchange-dependent) auction logic
 picks the winner from whichever bids actually arrived]
   |
   v
[DB primary + read replicas: campaign budget/pacing state, sharded by
 advertiser or campaign ID at the billions-of-events tier, read
 heavily, written asynchronously after the fact]
   |
   v
[ASYNC PIPELINE, off the hot path entirely: win notification to the
 winning DSP, billing ledger update, impression logging for
 analytics — all fire only AFTER the ad has already been decided and
 is already rendering]
   analogy: the receipt prints after you've already walked out with
   the bag, it doesn't block you at the register a second time
```

## 4. The transferable mechanisms

**Deadline-bound scatter-gather, not wait-for-everyone.** Fire the same request to every candidate in parallel, then collect whatever has returned by a fixed deadline and proceed without the stragglers. This is the single load-bearing idea in this lesson: latency becomes a property of the deadline you set, not of the slowest participant you happened to call. It generalizes to any fan-out: search federating across shards, a dashboard pulling from five microservices, a game server polling multiple matchmaking pools.

**Per-downstream circuit breaking on error rate, not on a single failure.** Google's documented callout quota system throttles a bidder's traffic down once its error/timeout rate crosses roughly 15%, rather than either tolerating it forever or cutting it off after one bad response. This is the classic circuit breaker pattern (as popularized by Netflix's Hystrix), applied here to a business relationship between companies instead of internal microservices: the exchange protects its own request-handling capacity from a struggling bidder without permanently blacklisting a partner over one bad millisecond.

**Parallel-over-sequential for anything with a shared deadline.** Header bidding's shift from waterfall to parallel calls, and Prebid.js firing requests to all configured demand partners simultaneously rather than one after another, is the same fix applied at the publisher's layer instead of the exchange's: whenever multiple independent lookups all have to complete before one shared next step, doing them concurrently instead of in sequence turns "sum of all latencies" into "latency of the slowest one you chose to wait for."

**Decouple the synchronous decision from the asynchronous consequences.** The auction has to decide a winner inside 100ms; billing that winner, notifying them they won, and logging the impression for analytics do not. Moving everything that isn't strictly required to answer "which ad renders right now" onto an async path after the decision is made is what keeps the hot path's latency budget from being eaten by bookkeeping work.

**Stateless, horizontally-scaled orchestration tier.** The auction orchestrator holds no per-request state between calls, so adding capacity is just adding more identical instances behind the load balancer, no coordination needed between them, no sticky sessions, no shared mutable state on the hot path.

**Adaptive, feedback-driven throttling instead of a fixed cap.** PubMatic's publicly described QPS-management layer reacts to real-time fluctuations and machine-learning-driven inventory throttling rather than a single static per-bidder rate limit set once and left alone; the healthy version of backpressure adjusts to current conditions, not to a number picked at design time.

## 5. The trade-offs

**Auction completeness vs. availability: availability wins, explicitly and by design.** An auction that closes without a bid from a bidder who was too slow this round is not treated as a failed auction, it is a complete one, just with fewer participants than the theoretical maximum. The alternative, refusing to close the auction until every eligible bidder has answered, would mean a single struggling DSP could stall ad rendering for every publisher using that exchange. The system chooses "always render something on time" over "always include every possible participant."

**Budget pacing consistency vs. latency: pacing is allowed to be approximate.** Advertiser daily/hourly budgets are tracked via counters that are read from cache and updated asynchronously, not locked and strictly serialized on every single bid decision across a globally distributed fleet handling millions of auctions a second. This means a campaign can occasionally slightly overspend or underspend its exact budget for a short window before pacing catches up, an accepted eventual-consistency trade, because strict global serialization of a budget counter across that request volume would itself become the bottleneck the whole architecture is built to avoid.

**Cost vs. latency: parallel fan-out is more expensive per request, on purpose.** Calling 100+ bidders simultaneously for every single ad slot costs more compute, bandwidth, and connection overhead than calling just a handful. The exchange accepts that cost because it is what buys a genuinely competitive auction (more bidders, more price competition, better yield for the publisher) inside a fixed, non-negotiable latency budget; cutting the bidder list to save cost directly costs the publisher auction revenue.

## 6. The systems-thinking lens

The failure mode this architecture is built to avoid is **a cascading overload triggered by a single struggling bidder**, the same underlying shape as a retry storm, just with the retry loop crossing a company boundary instead of staying inside one system.

Picture the mechanism without a circuit breaker: a DSP's backend starts degrading, maybe a deploy, maybe a downstream dependency of theirs is having its own bad day, and its response times creep past what would normally win auctions. If the exchange's fan-out logic has no per-bidder error tracking, it keeps sending that bidder the same full volume of callouts every single auction, each one now more likely to time out, each timeout still having consumed a connection slot and a slice of orchestrator thread-pool time on the exchange's side for nothing. At the exchange's scale, hundreds of billions of requests a day, that wasted capacity is not trivial: it is capacity that could have gone to healthy bidders, and if enough bidders are simultaneously degraded (a shared cloud region having a bad day, for instance), the exchange's own request-handling tier can start queuing, which slows down auctions for bidders who were never the problem in the first place. That is a **bulkhead failure**: one tenant's degradation leaking into shared infrastructure that every other tenant depends on.

The senior fix is exactly the callout quota system described in Google's own RTB documentation: track each bidder's error/timeout rate independently, and once it crosses a threshold, **automatically reduce how much traffic that bidder receives**, rather than either continuing to send full volume (which wastes capacity and makes the bidder's own overload worse) or cutting them off entirely on one bad response (which is too blunt and punishes transient blips). This breaks the loop at its source: the exchange stops feeding load into a system that is already struggling to handle the load it has, instead of scaling up connection pools or timeouts to merely tolerate the symptom for longer.

The general principle, worth carrying past ad tech entirely: whenever a system depends on many independent, uncoordinated downstreams that it does not control, the failure to defend against is not "a downstream is slow," it is "my own system keeps sending a slow downstream the same volume of work it would send a healthy one." The fix is never more capacity on your own side alone; it is feedback-driven throttling that reduces the load a struggling dependency receives, proportional to how struggling it actually is right now.

---

## Sources

- [OpenRTB 2.6 bid response timing, discussed with production timeout figures](https://www.researchgate.net/publication/321832827_Timeout_Analysis_Troubleshooting_and_Notification_in_Real_Time_Bidding_Advertising_System_with_Implementation)
- [Build the Response, Real-time Bidding, Google for Developers (Authorized Buyers RTB response guide, describes the ~100ms timeout window)](https://developers.google.com/authorized-buyers/rtb/get-started/response-guide)
- [Callout Quota System, Real-time Bidding, Google for Developers (the error-rate-based throttling / circuit-breaker mechanism)](https://developers.google.com/authorized-buyers/rtb/callout-quota-system)
- [Best Practices for RTB Applications, Google for Developers](https://developers.google.com/authorized-buyers/rtb/practices-guide)
- [Introducing Index Cloud, Index Exchange (current request-volume scale and purpose-built infrastructure)](https://www.indexexchange.com/2026/04/21/introducing-index-cloud/)
- [Scale: billions of bid requests per day, Index Exchange deck via Brave (2019 figures, cited for the earlier-scale comparison point)](https://brave.com/static-assets/files/Scale-billions-of-bid-requests-per-day-RAN2019061811075588.pdf)
- [Trade Desk customer story, Aerospike (queries-per-second and sub-100ms decision latency figures)](https://aerospike.com/resources/customer-stories/trade-desk/)
- [Prebid.js Timeouts documentation, Prebid.org (parallel header-bidding fan-out and client-side auction timeout behavior)](https://docs.prebid.org/features/timeouts.html)
- [Automated Bid Throttling & Machine Learning for Bid Requests, PubMatic engineering blog (adaptive, feedback-driven QPS shielding)](https://pubmatic.com/blog/bid-throttling-efficient-infrastructure/)

---

*Inference vs. fact, stated plainly: the OpenRTB 100–300ms response window, Google's 100ms Authorized Buyers timeout, the callout quota system's error-rate-based throttling (with the 15% figure), Index Exchange's request-volume figures, and Prebid.js's parallel-fan-out timeout behavior are all drawn from the vendors' own public documentation or widely corroborated secondary reporting on it, and are cited directly above. The Trade Desk's specific queries-per-second and sub-100ms latency figures come from a third-party (Aerospike) customer case study rather than The Trade Desk's own engineering blog, so they are reported here as that vendor's published account rather than a primary company statement. The framing of Google's callout quota system as "the circuit breaker for this domain," the bulkhead-failure description of one bidder's degradation leaking into shared exchange capacity, and the Rare.lab frame-budget analogy are this lesson's own reasoned inference about how these documented mechanisms function and generalize, not quoted claims from any of the sources above.*
