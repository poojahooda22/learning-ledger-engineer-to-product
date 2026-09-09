# Day 71 - How does Cloudflare enforce one rate limit across 300+ data centers with no central counter?

**Date:** 2026-09-09
**Difficulty:** Expert
**Topic:** Distributed rate limiting at the edge: per-datacenter local counting instead of one global synchronous counter (Cloudflare, since roughly 2017), built by combining three primitives this ledger has already taught for other problems and pointing them at a new one. Day 8 built a single-node rate limiter (one Redis, one token bucket, one GCRA timestamp). This lesson asks what happens to that idea the moment "one node" becomes "any one of 300-plus independent buildings around the planet, and a request can land at any of them." Day 10's consistent hashing comes back to shard a counter across servers inside one building instead of across the globe. Day 28's probabilistic counting comes back so a counter can be approximate instead of exact, on purpose. Day 55's anycast comes back as the reason a "local-only" counter is good enough at all: it is what keeps the same client hitting the same building, request after request.
**Stack relevance:** Rare.lab's runtime is meant to be embedded wherever a customer's page or app loads it, which is structurally the same shape as Cloudflare's problem: many independent points of contact (customer domains, edge regions, or eventually Rare.lab's own edge presence) all trying to enforce one shared policy (a compute quota on shader compiles, an API rate limit, a fair-use cap on the shared WebGL context) without a single choke point that every request has to phone home to first. The moment Rare.lab's embeddable runtime is live on enough customer sites that "check a central limiter before every shader compile request" becomes a real latency line item, this lesson's answer, count locally where the request already is, reconcile the picture slowly and out of the hot path, is the direct playbook.

---

## 1. The company and the breaking number

**Cloudflare, and the round trip a central counter cannot avoid.** Cloudflare's network is built on anycast: the same IP address is announced from every one of its data centers, reported in recent company materials as more than 300 cities across 100-plus countries, and BGP routing sends each client to whichever announcement is topologically closest. That is the whole trick behind Cloudflare being fast, and it is exactly what breaks a naive rate limiter. If Cloudflare wanted one authoritative, globally-consistent count of "how many requests has this API key made in the last minute," every one of those 300-plus locations would need to synchronously check and update one shared store before it could answer a single request. Speed of light alone puts a round trip between, say, a Singapore edge node and a US-based central store in the 150-300 millisecond range, before any processing happens. That is not a rounding error added to a request, it would often be several times the entire request's expected time-to-first-byte. A system built to make the internet faster cannot spend more time asking permission to count than it spends serving the page.

**The same wall shows up one layer down, in nanoseconds instead of milliseconds.** Cloudflare's L4Drop system, built on eBPF and XDP (eXpress Data Path, a hook that runs packet-processing code in the network card's driver layer before the kernel's normal networking stack ever touches the packet), is reported to sustain dropping more than 10 million packets per second on a single CPU core, adding roughly 10% total CPU overhead while absorbing an 8-million-packet-per-second attack. At that packet rate, the budget for an allow-or-drop decision is on the order of 50-100 nanoseconds per packet. There is no version of "make a network call to ask a counter somewhere else" that fits inside 100 nanoseconds; the decision has to be made with data that is already local, in memory, in the kernel, before the packet is allowed to go any further.

**The scale that makes the naive answer worse, not just slower.** Cloudflare's edge rate limiter has been described in the company's own materials as handling several billion requests per day and mitigating layer-7 floods reported as high as roughly 400,000 requests per second against a single domain. A single central counter is not just too slow for this traffic, it would also have to absorb that entire write volume, for every protected domain, as one hot, contended target, which is precisely the hot-key problem this ledger named on Day 16, now applied to a rate-limit counter instead of a viral post.

---

## 2. Why the naive (demo) design dies

**The obvious version:** every edge server, wherever it happens to be, sends an `INCR` to one shared Redis (or similar) instance that holds the authoritative count for each rate-limited key (an IP, an API token, a domain), and checks the result against the configured threshold before deciding to serve the request or return a 429. For a single office's API gateway, this is completely fine, and it is functionally what Day 8's lesson already built.

**Death one: the round trip is the whole latency budget.** As above, once "every edge server" spans multiple continents, the network round trip to one central store dwarfs the time it takes to actually serve the request. Users on the far side of the world would feel their rate limit check before they felt their page load.

**Death two: one store becomes a single global point of contention and failure.** Every protected domain's counter lives on the same store, so its write throughput has to absorb the sum of Cloudflare's entire protected traffic, not any one domain's share of it. And because every single request now depends on that store answering, its outages stop being a local problem and become a global one: if it is slow or down, Cloudflare has to choose between failing open (letting all traffic through everywhere, including the DDoS traffic the limiter exists to catch) or failing closed (blocking legitimate traffic on every domain, everywhere, because one shared dependency hiccuped). Neither answer is acceptable at this scale.

**Death three: it defeats the very topology that makes the system fast.** Anycast's entire value is that a request gets served by the nearest location without any cross-datacenter coordination. Routing every request's rate-limit check back to one hub, wherever that hub is, reintroduces the cross-continent hop anycast was built to avoid, for every single request, just to answer a yes-or-no question.

**The real-world version:** a team stands up a shared Redis-backed limiter for a service running out of two regions and it works well. They add six more edge regions to cut latency for users worldwide, and average response time gets worse, not better, because now most requests are served fast by the nearest region but every one of them still pays a cross-region round trip just to ask "am I still under my limit."

---

## 3. The architecture

```
Clients (worldwide)
  - job: send requests to one anycast IP, unaware of which data
    center will actually answer
  - analogy: dialing one nationwide phone number and being connected
    to whichever local branch office is closest to you, automatically

        |
        v
Anycast routing (BGP announces the same IP from 300+ locations)
  - job: send each client to its nearest data center, and keep
    sending that same client back to the same one on later requests
    under normal network conditions
  - analogy: a phone company's call routing that reliably connects
    you to your neighborhood branch every time you dial, so that
    branch, not head office, ends up knowing your recent history

        |
        v
In-kernel fast path (eBPF / XDP hash map, per network card)
  - job: for raw packet-flood traffic, make an allow/drop decision
    in tens of nanoseconds using a local, in-memory table, before
    the packet reaches any application code at all
  - analogy: a bouncer at the door turning away an obvious flood of
    fake tickets without radioing the box office to check each one

        |
        v
Per-data-center sharded counter store (Twemproxy in front of many
Memcached nodes, consistent hashing across shards)
  - job: hold the request counters for this one data center only,
    spread across many local machines so no single memory cache
    becomes its own hot spot, using the same consistent-hashing
    trick Day 10 used to spread cached data across a cluster
  - analogy: a single branch office's own filing system, split
    across several clerks by customer name so no one clerk's
    inbox drowns, but never mailed to another branch

        |
        v
Local allow/deny decision (this data center only, this request only)
  - job: compare this data center's local count for the key against
    the configured threshold and answer immediately, no network hop
    outside the building
  - analogy: the local branch manager approving or declining a
    withdrawal using only that branch's own ledger

        |
        v
Async, out-of-band aggregation and rule propagation (eventual,
never in the request's critical path)
  - job: slowly roll up approximate, cross-data-center signals
    (which keys are trending hot everywhere) and push updated
    thresholds or block-lists back out to every location, on the
    order of seconds to minutes later, using the same
    push-configuration pattern Day 47 described for Cloudflare's
    own Quicksilver config-distribution system
  - analogy: branch offices mailing end-of-day summaries to head
    office, which occasionally wires back an updated company-wide
    policy, never something a customer standing at the counter
    has to wait for
```

---

## 4. The transferable mechanisms

- **Decide locally, reconcile eventually.** The core move is refusing to put a cross-location network call anywhere in the request's hot path. Every enforcement decision is made with data that already lives where the request landed; anything that needs a global view is computed later, out of band, and only ever changes future decisions, never blocks the current one. This is Day 14's active-active pattern (accept writes locally everywhere, reconcile later) aimed at a counter instead of a database row.

- **Shard within the blast radius you actually have, using consistent hashing again.** Day 10 used consistent hashing to spread cached data across a cluster without a full reshuffle on every resize. The same algorithm reappears here at a smaller radius: inside one data center, Twemproxy hashes each rate-limit key to one specific Memcached shard, so the counter for a given key always lands on the same machine, and adding or removing a shard only reshuffles a small fraction of keys, not all of them.

- **Trade exactness for O(1) memory and zero coordination with approximate counting.** Rather than storing an exact per-key counter forever, a probabilistic structure like a CountMin sketch (Day 28) can track approximate frequency for an effectively unbounded number of keys in fixed memory, with a known, bounded error rate instead of perfect accuracy, reported in some of Cloudflare's own engineering write-ups as being tuned for roughly single-digit-percent overcounting in exchange for counting at line rate.

- **Let the routing layer do half the consistency work for free.** Anycast (Day 55) is not just a performance optimization here, it is what makes local-only counting a reasonable approximation of a global count in the first place: because the same client's traffic reliably lands at the same data center, that data center's local view is, most of the time, actually a good proxy for that client's real global rate, without anyone coordinating anything.

- **Push configuration, don't pull it on the hot path.** Rate-limit thresholds and rules are distributed to every location ahead of time through an asynchronous propagation system (Day 47's Quicksilver pattern), so the local, per-request decision never needs to fetch a rule over the network either, only to compare against a rule it already has in memory.

- **Push the cheap check as close to the wire as the traffic requires.** As attack volume rises past what a userspace process can evaluate per request, the same style of decision moves further down the stack, from an application-level check, to a shared-memory lookup, to an in-kernel eBPF/XDP hash-map lookup before the packet is even fully parsed. This is the same instinct as a CDN (Day 4) pushing content physically closer to the reader: push the decision as close as possible to where the cost is actually being paid.

---

## 5. The trade-offs

**Consistency versus availability, and it is a deliberate choice about this specific data type.** A rate-limit counter is allowed to be wrong for a little while. An attacker who sends 900 requests per second to each of 50 different data centers, staying under any single location's threshold, can slip past a limit that was conceptually meant to cap a client at 1,000 requests per second globally, at least until the slower, asynchronous aggregation layer notices the pattern and pushes a tightened rule back out. Cloudflare accepts this gap because the cost of a false negative here (a burst of extra abusive traffic gets through for seconds to minutes) is far smaller than the cost of the alternative: a synchronous global counter that adds hundreds of milliseconds to every legitimate request, or that takes the whole network down with it when it has a bad day. Contrast this with Day 6's Stripe ledger, where the same kind of looseness on a payment balance is never acceptable, because the data type there (money that must never be double-spent) demands the opposite choice. Same CAP dial, deliberately set in opposite directions for two different kinds of data.

**Cost versus latency, paid in duplicated memory.** Holding a full local counter shard in every one of 300-plus data centers means the same logical rate-limit key can exist in memory in dozens or hundreds of places at once, which is strictly more total memory across the fleet than one central store would use. That cost buys sub-millisecond local decisions everywhere instead of a 150-300 millisecond round trip anywhere, which for a company whose entire product is "faster" is not a close call.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is **threshold slicing**: a distributed variant of the hot-key problem, run in reverse. Day 16 was about one key getting too hot for one shard to bear. Here, a motivated actor can deliberately keep every individual shard's view of their traffic under its local threshold by spreading requests across many locations, exploiting exactly the local-only decision that makes the system fast in the first place. Left alone, this is not self-correcting: unlike a retry storm, which feeds on itself and gets worse, a slicing attack is stable as long as the attacker keeps their per-location rate low enough, and no single data center ever sees enough evidence, on its own, to act.

The naive instinct, make the per-data-center counters perfectly synchronized so no slicing pattern can hide, reintroduces exactly the wall Section 2 already ruled out: a synchronous cross-continent call on every request. The senior fix does not add capacity or add consistency to the hot path at all; it adds a second, deliberately slow, deliberately out-of-band signal that only exists to catch the pattern the fast path structurally cannot see. Local decisions stay local and stay fast, but an asynchronous aggregation layer periodically rolls up approximate counts across locations, and when it spots a key that looks fine everywhere but is actually large in aggregate, it pushes a tightened, temporary rule back out to every data center through the same async propagation path that distributes ordinary configuration. The loop is broken not by making the fast path smarter or bigger, but by giving the system a second, slower loop whose entire job is to watch for the failure mode the fast loop is blind to by design.

---

## Sources

- [How we built rate limiting capable of scaling to millions of domains, Cloudflare Blog](https://blog.cloudflare.com/counting-things-a-lot-of-different-things/): the primary source for the per-data-center architecture (Twemproxy sharding a Memcached cluster with consistent hashing inside each location, counters kept local to the data center rather than synchronized globally) and for anycast being the reason local-only counting is a workable approximation of a global count. Direct fetch was blocked by this session's network egress policy; the architecture description here is drawn from search-indexed summaries of the post and from the Hacker News discussion of it, rather than a full direct read.
- [How we built rate limiting capable of scaling to millions of domains, mirrored on Medium](https://medium.com/cloudflare-blog/how-we-built-rate-limiting-capable-of-scaling-to-millions-of-domains-3bcc875e16a6): same article, used as a secondary mirror; also blocked from direct fetch this session.
- [How Cloudflare built rate limiting that scaled to millions of domains, Hacker News discussion](https://news.ycombinator.com/item?id=26393706): secondary discussion corroborating the architecture summary above.
- [Raking the floods: my intern project using eBPF, Cloudflare Blog](https://blog.cloudflare.com/building-rakelimit/): primary source for the CountMin-sketch-based approximate counting approach to rate limiting per subnet, and the accuracy-versus-throughput trade-off (tolerating a small, bounded overcount in exchange for constant memory and lock-free, line-rate counting). Direct fetch blocked this session; figures here are drawn from search-indexed summaries.
- [L4Drop: XDP DDoS Mitigations, Cloudflare Blog](https://blog.cloudflare.com/l4drop-xdp-ebpf-based-ddos-mitigations/): primary source for the eBPF/XDP in-kernel drop path, the reported figure of sustaining over 10 million packets per second on a single CPU core, and the roughly 10% CPU overhead observed while absorbing an 8-million-packet-per-second attack. Direct fetch blocked this session; figures drawn from search-indexed summaries of the post.
- [Mitigating a 754 Million PPS DDoS Attack Automatically, Cloudflare Blog](https://blog.cloudflare.com/mitigating-a-754-million-pps-ddos-attack-automatically/): cited for scale context on the packet-per-second range Cloudflare's edge decision path has to survive; direct fetch blocked this session.
- Day 4 (this ledger, CDN and zero origin egress), Day 6 (Stripe correctness under load), Day 8 (single-node rate limiting), Day 10 (consistent hashing and sharding), Day 13 (backpressure and load shedding), Day 14 (multi-region active-active), Day 16 (the hot-key/celebrity problem), Day 28 (probabilistic data structures), Day 47 (Cloudflare Quicksilver config distribution), Day 55 (anycast global server load balancing): the ledger's own prior lessons this one directly reuses and recombines.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of every Cloudflare engineering blog post cited above, so the specific figures drawn from them (the per-data-center Twemproxy/Memcached architecture, the CountMin-sketch overcount tolerance, the L4Drop packet-per-second and CPU-overhead numbers, and the 754-million-pps attack) are summarized here from search-indexed excerpts and a Hacker News discussion rather than quoted from a full direct read of the primary sources, consistent with how this ledger has flagged network-blocked sources before (see Day 69 and Day 70). The general architectural claims, local-only per-data-center counting backed by anycast routing stability, and the accuracy-for-throughput trade behind approximate counting, are treated as solid, since they are corroborated across the original company's own multiple posts and independent secondary discussion, even where the primary documents could not be fetched directly this session.
