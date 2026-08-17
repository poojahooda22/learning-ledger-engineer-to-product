# Day 56: How does an ad exchange run a live auction among dozens of bidders and settle it before the page you're loading finishes painting?

**Date:** 2026-08-17
**Difficulty:** Expert
**Topic:** Real-time bidding (RTB) and OpenRTB ad auctions: how Google's Ad Exchange, The Trade Desk, and dozens of other demand-side platforms run a sealed-bid, second-price auction for a single ad impression, fan it out to every interested bidder in parallel, and close it inside a deadline of 100 to 160 milliseconds, hard-capped by a field called `tmax` that shrinks every time the request hops through another layer. The mechanism (parallel fan-out against a shrinking deadline, discard whoever answers late, no retries) shows up anywhere real-time work has to finish inside a fixed budget: a rendering frame, a live search-ranking call, a multiplayer tick.
**Stack relevance:** Rare.lab's embeddable runtime already lives inside a version of this exact deadline: one shared WebGL context, one frame budget of roughly 16.6 milliseconds at 60 frames per second, and multiple shader passes competing for that budget on every single frame. The ceiling this lesson surfaces is what happens the moment work that misses its deadline can't just be silently dropped, the way a late ad bid is, because it represents money, usage, or state that has to land somewhere durable.

---

## 1. The company and the breaking number

**Google's Ad Exchange (AdX)** and **The Trade Desk** run the same basic game, at the same rough scale: somewhere north of **10 million bid requests per second at peak**, according to public reporting on how programmatic exchanges size their infrastructure, with each individual request needing an answer inside a window of **80 to 300 milliseconds**, depending on the ad format and exchange. Google's own documentation for Authorized Buyers states the response deadline varies by format and is carried in the `tmax` field of the OpenRTB bid request (or `response_deadline_ms` in Google's own RTB protocol), typically landing in the **80 to 1,000 millisecond range** depending on the surface; the widely cited number for Google Ad Exchange's standard display auction is a **100 millisecond** budget, and Google's own Open Bidding product explicitly extends that to **160 milliseconds** as a deliberate trade, more time for bidders to think, in exchange for a slightly slower page. The IAB's OpenRTB 2.6 specification hard-caps the whole exercise: bid responses that arrive after the exchange's `tmax` are dropped from the auction, full stop, no matter how good the bid was.

Put a real event next to that number. A person opens a news article. Before a single ad pixel paints on their screen, the publisher's page has already: sent a bid request out to a dozen or more advertising exchanges and demand-side platforms in parallel, each of those exchanges has fanned that same request out to dozens of its own bidders, each bidder has pulled up whatever it knows about this specific person (a hashed cookie or device ID, their inferred interests, whether they're in the retargeting list for a shoe brand that's still annoyed they abandoned a cart), scored every active campaign against that profile, picked a price, and shipped a response back across the internet, and the exchange has run a full auction over every bid that made it back in time, picked a winner, and handed the ad creative's URL back to the page. All of that, done, before the ad slot itself finishes rendering. If it isn't done in roughly a tenth of a second, the auction closes anyway, with or without a bid from anyone who was still thinking.

The breaking number isn't "how many ads can one server evaluate." It's **how much real, useful computation can survive a hard deadline measured in tens of milliseconds, at global scale, without the system either timing out constantly or accepting stale, cached-guess bids instead of live ones.** A naive version of this, one server evaluating one auction at a time with a generous timeout, doesn't scale down gracefully to that number, it simply cannot answer in time, and every millisecond of slack the naive version might have used to be more accurate or more careful is directly, measurably, money the whole ecosystem loses. This is also where Akamai's 2017 State of Online Retail Performance report, built from roughly 10 billion anonymized retail visits, becomes relevant even though it isn't about ads specifically: it found a **100 millisecond delay in page load time cuts conversion rate by about 7%.** Amazon's own internal testing, as described by engineer Greg Linden from work done around 2006, found that **every 100 milliseconds of added latency cost about 1% of sales.** An ad auction that blows its budget by even one of those 100 millisecond increments isn't a rounding error, it's revenue the publisher and the advertiser both feel.

## 2. Why the naive (demo) design dies

**Version one: one ad server, evaluate bidders one at a time, wait for each to answer before asking the next.** This is the "waterfall" model, and it is, historically, exactly how ad serving actually started. It fails in three specific, compounding ways once the deadline and the bidder count both get real.

**Sequential calls burn the deadline linearly, and the deadline doesn't move.** If the exchange has a 100 millisecond budget and 20 potential demand partners, asking them one after another at even a generous 20 milliseconds each blows the entire budget on four partners, leaving sixteen who never got asked at all. Those sixteen aren't a rounding error, in the waterfall era publishers ran this exact model, offering inventory to ad networks in a fixed priority order (usually ranked by historical average revenue), and it structurally meant lower-ranked networks, even ones that might have paid more for this specific impression, effectively never got a real look. The publisher wasn't running an auction. They were running a queue with a cutoff, and the cutoff was arbitrary, not price-driven.

**A single evaluator becomes a shared queue the moment two ad requests arrive close together, and ad requests never arrive one at a time.** A popular publisher isn't serving one reader, it's serving tens of thousands of concurrent page loads, each of which needs its own independent, time-boxed auction. One evaluator, or even one thread pool sized for calm traffic, means request number two waits behind request number one's already-tight 100 millisecond budget. Under real traffic (a news event breaks, a video goes viral, a Black Friday sale opens) this doesn't degrade gracefully, it collapses: every auction that was going to be tight becomes an auction that's late, and every late auction is unsellable inventory, either shown blank, filled with a low-value fallback ad, or, worst case for the user, not shown until after the surrounding content has already rendered and shifted.

**A hop that doesn't shrink its own deadline for the next hop silently eats the whole budget.** This is the specific one, unique to nested, multi-party systems like ad exchanges: the publisher's page asks an exchange, the exchange asks several DSPs (demand-side platforms, the systems that represent advertisers), a DSP's own gateway asks its internal bidding microservices, which might call a feature store or a machine learning model server for a click-through-rate prediction. If any layer in that chain forwards the *same* 100 millisecond deadline it received, instead of subtracting the time it already spent and the network hop still ahead of it, the innermost service thinks it has 100 milliseconds when in reality 60 are already gone. It answers "on time" from its own point of view and misses the real deadline anyway. Nobody's component was slow. The chain was.

## 3. The architecture

```
[Publisher's webpage / app, e.g. a news article]
   a reader opens the page, an ad slot needs to be filled
   analogy: a homeowner puts a "room for rent" sign in the
   window and opens the door to whoever's fastest, not whoever
   knocks first
   |
   v
[Ad server / header bidding wrapper, in the browser or app]
   fires bid requests to MULTIPLE exchanges/SSPs simultaneously,
   not one after another (this is "header bidding," which
   replaced the old sequential "waterfall" through the mid-2010s)
   job: start every auction's clock in parallel, not in series
   analogy: instead of calling estate agents one by one, you
   text all of them the listing at the same moment and take
   whichever verified offer lands first
   |
   +----------------+----------------+----------------+
   v                v                v                v
[Exchange A]    [Exchange B]    [Exchange C]    [Exchange D]
(Google AdX)    (The Trade      (PubMatic)      (Xandr / etc.)
                 Desk-side)
   each exchange independently fans the SAME impression out to
   its own roster of registered bidders
   job: multiply the fan-out by another layer, and set (and
   shrink) its own tmax before forwarding
   analogy: each estate agent has their own list of buyers they
   personally call
   |
   v (inside one exchange, e.g. Exchange A)
[Callout to N demand-side platforms (DSPs) in parallel]
   each DSP gets: the ad slot's size/format, the page's category,
   a hashed user/device ID, viewability signals, and a shrinking
   tmax (say, 60ms left, not the original 100ms)
   job: give every interested buyer a real, simultaneous chance
   analogy: everyone on the buyer list gets the same phone call
   at the same time, phone doesn't ring twice for anyone
   |
   v (inside one DSP, e.g. The Trade Desk)
[Bidder logic: feature lookup + ML scoring + budget/pacing check]
   pulls the user's profile from an in-memory feature store
   (O(1) key lookup, not a live database query, there is no time
   for a query), scores every eligible advertiser campaign against
   it with a pre-trained model, checks the campaign still has
   budget left today (pacing), returns ONE price
   job: turn "who is this person, probably" into "what is this
   specific impression worth to me, right now, in single-digit
   milliseconds"
   analogy: an appraiser who has already memorized the whole
   market and just needs one glance at the house
   |
   v
[Auction clears at the exchange: second-price sealed-bid auction]
   collects every bid that arrived before tmax, drops every bid
   that arrived after it no matter how high, ranks the rest,
   winner pays the SECOND-highest bid plus a cent (or the reserve
   price, whichever is higher), not their own bid
   job: pick a winner AND a fair-ish clearing price in one pass,
   using only data that showed up on time
   analogy: a silent auction where you write one sealed number,
   the highest number wins but only pays what the runner-up wrote
   |
   v
[Ad creative served to the page; async win-notice + billing pipeline]
   the WINNING bid's creative URL goes back to the browser
   immediately (synchronous, on the critical path); the "you won,
   here's what to bill" notification and the actual ledger entry
   happen on a SEPARATE, async path that does not block the ad
   from rendering
   job: keep the user-visible path fast by moving anything that
   doesn't need to finish in 100ms off of the 100ms path
   analogy: you get your coffee the moment it's made, the receipt
   and the accounting entry happen behind the counter, not in
   the queue
```

## 4. The transferable mechanisms

- **Hard deadline with silent discard, not retry.** `tmax` isn't a suggestion, it's an eviction rule: a bid that arrives even one millisecond late is treated exactly like a bid that never arrived. There's no retry, because a retry can't finish before the deadline either, and retrying would only make the next auction's queue longer. This is the same discipline a real-time rendering loop uses: a shader pass that can't finish inside the frame budget doesn't get an extra frame, it gets skipped or downgraded, because "late" and "wrong" cost the same to the viewer.

- **Parallel fan-out beats sequential polling under a fixed deadline, every time.** Header bidding's whole reason for existing is that a sequential waterfall structurally can't give everyone a fair, real-time look inside a shared budget, only parallel fan-out can, because the deadline is shared, not per-bidder. The general form: whenever N independent parties need to be asked the same question inside one fixed time budget, ask them all at once and take what comes back, don't queue them.

- **Shrink the deadline at every hop (deadline propagation).** An exchange that forwards its own remaining time, not the original time, to each downstream call is what keeps the whole nested call chain honest. This is the same pattern gRPC calls "deadline propagation" and it's the direct fix for the silent-budget-eating failure described in Section 2: every layer subtracts what it already spent, plus a margin for the next network hop, before it asks anyone downstream.

- **Second-price sealed-bid auction for incentive-compatible pricing.** Paying the winner's own bid (a first-price auction) rewards bidders who guess the market instead of stating their true value, and pushes everyone toward shading their bids down defensively, worse information for the seller. Paying the *second*-highest bid instead makes a bidder's own honest true value their best possible bid, mathematically, so the exchange gets truthful price signals without anyone needing to bluff. (In practice much of the industry has drifted toward first-price auctions since header bidding made "the highest bid across many exchanges" directly comparable, which is its own trade-off, covered below, but the underlying mechanism design lesson, that auction *rules* shape what bidders honestly reveal, is the transferable one.)

- **Precomputed, O(1) feature lookups instead of live queries on the hot path.** A bidder scoring a real person in under 10 milliseconds is not running a database query against that person's history, it's reading a pre-materialized feature vector out of an in-memory key-value store that some separate, slower, offline pipeline already built and kept warm. The expensive work (aggregating a user's behavior into a feature) happens off the clock; only the cheap read happens on it. This is the same move Day 54's leaderboard lesson made for ranking, and Day 47's for config distribution: never compute on the request path what you can precompute and just look up.

- **Decouple the synchronous critical path from the async settlement path.** The ad creative renders the instant a winner is picked. The billing ledger entry, the win notification back to the losing bidders, the analytics event, none of that has to happen before the user sees their ad, and none of it is allowed to block that render. Splitting "what the user needs right now" from "what needs to be durably true eventually" is what lets the fast path stay fast while the slow, correctness-critical bookkeeping still happens, just off to the side.

## 5. The trade-offs

**Consistency vs. availability, and it's a different answer for different data in the very same request.** The user profile a bidder scores against is allowed to be stale, eventually consistent, built from a feature pipeline that might be minutes or hours old, because being slightly wrong about someone's interests costs a slightly worse ad, not a broken system. The auction's winner-selection and the resulting billing entry cannot be eventually consistent in the same way, real money changes hands, advertisers set hard daily budgets that must not be overspent, and a DSP that double-counts an impression or double-charges a campaign has a real, auditable, financial bug, not a UX nitpick. So the same pipeline runs AP (available, eventually consistent) for profile data and something much closer to CP (consistent, willing to reject or delay) for spend accounting, on purpose, because the cost of being wrong is wildly different for the two.

**First-price vs. second-price is its own live trade-off the industry is still fighting over.** Header bidding exposed the same impression to many exchanges at once, and once a publisher can directly compare the *actual highest bid* across all of them, a second-price auction inside any single exchange starts looking like it's leaving money on the table relative to a first-price one, so most major exchanges (including Google AdX, since 2019) switched to first-price. That fixes cross-exchange comparability but reintroduces the bid-shading problem second-price was designed to avoid: bidders now have to guess how much to shave off their true value to avoid overpaying, adding noise and cost (both sides now run their own pacing and shading models) back into the system in exchange for simpler, comparable pricing.

**Cost vs. latency, paid for in infrastructure a naive design wouldn't need.** Hitting single-digit-millisecond bidder response times at 10 million requests per second isn't free: it means keeping warm, persistent, HTTP/2 connections open to every exchange instead of paying a TCP and TLS handshake on every request, it means running bidder infrastructure physically colocated inside or next to the exchange's own data centers to avoid a long network hop eating the budget, and it means keeping the entire feature store in memory rather than on disk. All of that is real, ongoing infrastructure spend, taken on specifically to buy latency, because in this system latency isn't a UX nicety, it's the line between "this bidder gets to compete" and "this bidder's response gets silently dropped, forever, on every single auction."

## 6. The systems-thinking lens

The feedback loop that actually kills fill rate at scale here isn't "too much traffic," it's a **hidden timeout-budget cascade**: a chain where no single hop is provably broken, but the deadline quietly evaporates before it reaches the layer that actually needs it. A DSP's own internal call graph, gateway to bidder logic to feature store to ML model server, can each individually respond "fast" by their own local clock and still, in aggregate, blow the exchange's real deadline, because nobody in the chain was subtracting what upstream already spent. The failure doesn't look like an error. It looks like the DSP simply stops winning auctions, or stops being asked at all, because from the exchange's point of view it "timed out" over and over, with no exception thrown anywhere, no alert fired, just a slow, silent drop in win rate that a naive dashboard reads as "this bidder must be uncompetitive today" rather than "this bidder's internal deadline math is broken."

The senior fix isn't adding capacity to make every hop individually faster, that only raises the ceiling the same loop hits again at higher scale. It's **breaking the loop structurally**: propagate a shrinking, explicit deadline through every layer of the call chain (not the original deadline, the *remaining* one), and pair it with a circuit breaker that stops calling a downstream dependency once it's chronically blowing its allotted slice, rather than paying the full timeout cost on every single request forever. This is the same discipline behind gRPC's deadline propagation and the Google SRE playbook's guidance on cascading failures: a system that keeps politely waiting out a timeout on a dependency that's already proven itself unreliable this second isn't being resilient, it's donating its own scarce time budget to a problem it can't fix from where it's standing. The fix is to stop asking, temporarily, and let that dependency recover off the hot path instead of inside it.

## Map to Rare.lab's stack

Rare.lab doesn't run ad auctions, but it already runs the exact structural pattern underneath one: a **hard, recurring deadline shared by many competing pieces of work**. Every frame the embeddable runtime renders has a budget of roughly 16.6 milliseconds at 60 frames per second, one shared WebGL context, and potentially many shader passes or effects that all want time inside that single frame. That's `tmax`, just renamed. The transferable lesson from Section 4 applies directly: a shader pass that can't finish inside the frame budget should be dropped or downgraded for that frame, not queued to "catch up" next frame, the same way a late ad bid is discarded rather than retried, because retrying inside a fixed, recurring budget only pushes the backlog forward and compounds it.

The node-based editor's compile step is a second, more literal echo: if Rare.lab ever fans a shader graph out to multiple compile targets or candidate generations in parallel (say, trying a couple of AI-suggested node arrangements and taking whichever compiles clean and fastest inside a time budget), that's structurally a parallel fan-out auction, header bidding's core move, not a sequential waterfall through candidates one at a time. The same argument that makes header bidding beat the old waterfall (a shared deadline can't be fairly divided across sequential attempts) applies to that compile fan-out too.

The ceiling this lesson surfaces for Rare.lab is the one place the ad-auction analogy stops holding: **billing, usage metering, and durable scene saves cannot be treated like a late ad bid.** An ad exchange can drop a late bid for free, nobody's data was lost, the auction just runs without it. But if Rare.lab's runtime ever needs to record "this user's session used N seconds of compute" or persist a finished scene to R2 and it misses some tight synchronous window, that work cannot simply be discarded the way a late bid is, it has to fall onto Section 4's other pattern instead: decouple it onto an async path (a queue, a durable write-ahead log) that's allowed to take longer than one frame, precisely because it's the kind of state that has to end up durably, exactly-once true, not the kind of state that's fine to lose.

---

## Sources

- [OpenRTB 2.6, IAB Tech Lab (GitHub)](https://github.com/InteractiveAdvertisingBureau/openrtb): the current formal spec; establishes `tmax` as the field carrying the exchange's maximum wait time for bid responses, and that responses arriving after it are excluded from the auction. Summarized here as fact based on multiple independent technical write-ups of the spec (direct GitHub fetch not attempted in this session).
- [IAB Tech Lab: OpenRTB standard page](https://iabtechlab.com/standards/openrtb/): background on OpenRTB as the industry-standard protocol connecting publishers/exchanges (the sell side) to DSPs/bidders (the buy side).
- [Deep Dive Into OpenRTB: The Backbone of Online Ad Auctions, Evangelos Pappas, Medium (Ad-tech publication)](https://medium.com/ad-tech/deep-dive-into-openrtb-the-backbone-of-online-ad-auctions-e518bdfef216): practitioner walkthrough of the bid request/response cycle and typical 100 to 150 millisecond response windows in production exchanges.
- [Authorized Buyers Real-Time Bidding (RTB) Protocol, Google for Developers](https://developers.google.com/authorized-buyers/rtb/get-started/start) and the [Callout Quota System page](https://developers.google.com/authorized-buyers/rtb/callout-quota-system): Google's own documentation that the response deadline is carried per-request in `tmax`/`response_deadline_ms` and varies by format (roughly 80 to 1,000 milliseconds depending on surface), and that bidders who are chronically slow or unresponsive have their callout quota (their share of traffic) throttled down. Direct fetch of developers.google.com was blocked by this session's network egress policy; figures relayed via search-indexed summaries of the same pages.
- Reporting citing Google Ad Exchange's roughly 100 millisecond standard auction window versus Open Bidding's extended 160 millisecond window, and industry figures describing Google AdX, Amazon's APS, and The Trade Desk each processing upward of 10 million bid requests per second at peak, drawn from aggregated ad-tech industry write-ups surfaced via search rather than one single primary document; presented as a rough, well-corroborated order of magnitude rather than an audited figure.
- [Header Bidding vs Waterfall: Key Differences Explained, BidsCube](https://bidscube.com/blog/header-bidding-vs-waterfall-key-differences/) and similar industry explainers: background on the historical shift from sequential waterfall ad serving (offer inventory to networks one at a time, in a fixed priority order) to parallel, in-browser header bidding auctions across multiple exchanges at once, and the widely cited 20 to 40% publisher revenue lift attributed to that shift.
- [Akamai Online Retail Performance Report, Spring 2017 (press release)](https://www.akamai.com/newsroom/press-release/akamai-releases-spring-2017-state-of-online-retail-performance-report): source for the "100 millisecond delay cuts conversion rate by about 7%" figure, based on an analysis of roughly 10 billion anonymized retail site visits.
- [Latency Cost: Amazon Loses 1% Sales Per 100ms](https://latencycost.com/) and multiple independent secondary write-ups tracing back to Amazon engineer Greg Linden's original 2006 internal findings and public talks: source for the "100 milliseconds of added latency cost roughly 1% of sales" figure. The original internal Amazon study itself was never published as a formal paper; this figure is relayed through Linden's own public blog posts and conference talks rather than a primary Amazon document.
- Google's shift from second-price to first-price auctions on Google Ad Exchange (2019), and the general industry drift toward first-price auctions following the rise of header bidding, drawn from aggregated ad-tech industry reporting rather than one single primary announcement.
- Deadline propagation as a distributed-systems pattern (the general practice of forwarding a shrinking, remaining deadline through a call chain rather than the original one) and cascading-failure mitigation via circuit breakers, both widely documented patterns in distributed-systems and SRE literature (for example, Google's Site Reliability Engineering book's treatment of cascading failures and load shedding); referenced here as established general practice rather than a claim specific to any one ad-tech company's internals, since no ad exchange publishes its own internal deadline-propagation implementation.

---

*Inference vs. fact, stated plainly: the `tmax`/response-deadline mechanics, the second-price-to-first-price shift, and the waterfall-to-header-bidding history are documented industry facts, corroborated across the OpenRTB spec, Google's own developer documentation, and multiple independent ad-tech trade publications. The Akamai 7%-per-100ms and the Amazon 1%-per-100ms figures are each real, specific, publicly reported findings, from two different studies, at two different companies, roughly a decade apart, presented side by side deliberately to show that the "click adds up" nature of latency cost has been independently observed more than once, not to imply the two numbers measure the same thing. The "10 million-plus bid requests per second at peak" figure for AdX/APS/The Trade Desk is a rough, well-corroborated industry order of magnitude relayed through search-indexed secondary sources, not a single company's own audited disclosure, and should be read as illustrative of scale rather than a precise, citable statistic. The architecture diagram, the deadline-propagation-as-cascading-failure framing, the coffee/receipt and estate-agent analogies, the CAP-per-data-type trade-off framing, and the Rare.lab mapping are this lesson's own analysis built on top of the documented mechanics above, not claims made verbatim by Google, The Trade Desk, or the IAB. Direct fetches of developers.google.com and akamai.com were blocked by this session's network egress policy; the figures attributed to them were relayed through search-indexed summaries rather than a first-hand read of the source page, and are worth verifying directly before citing elsewhere.*
