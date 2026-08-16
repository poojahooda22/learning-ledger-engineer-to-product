# Day 55 — How does Cloudflare absorb a 31.4 terabit-per-second attack aimed at one IP address, with no load balancer anywhere in the picture?

**Date:** 2026-08-16
**Difficulty:** Expert
**Topic:** Anycast and BGP-based global routing. How the same IP address can be answered by 337 different physical buildings on 6 continents at once, why "which data center handles this request" is a decision no Cloudflare system ever makes, and how that same property turns a would-be catastrophic denial-of-service flood into hundreds of locally survivable ones, automatically, without anyone deciding to spread the load.
**Stack relevance:** Rare.lab already rides this for free, every R2 asset and every Worker Rare.lab ships is served from an anycast IP the moment it's deployed. The ceiling this lesson surfaces is what happens the moment a request needs something anycast can't give it: an authoritative answer from one place, like a write to Supabase Postgres.

---

## 1. The company and the breaking number

**Cloudflare**, whose network as of mid-2026 spans **337 cities across 100-plus countries**, all of it reachable through a **small number of IP address blocks that are announced, identically, from every single one of those 337 locations at once**. There is no dashboard anywhere inside Cloudflare that assigns "this user gets data center 214." No system decides it. And yet in the fourth quarter of 2025, Cloudflare's network absorbed a DDoS attack that peaked at **31.4 terabits per second, lasting 35 seconds**, aimed at a single customer's IP address, and the customer's origin server never saw a single packet of it. Four months earlier, in September 2025, a **11.5 Tbps attack** hit, also for about 35 seconds. A year before that, the record was a mere **5.6 Tbps**, during the last week of October 2024.

Put a number on what a single, well-provisioned data center can actually absorb on its own uplinks: a large facility today might carry on the order of a few hundred gigabits to low single-digit terabits of aggregate external bandwidth across all its links combined. A 31.4 Tbps flood aimed at one address is **roughly 10 to 50 times more traffic than a single site's total pipe can carry**, arriving in 35 seconds, not a slow ramp a human has time to react to. If that traffic all had exactly one physical building to reach, the building's uplinks saturate before anyone's pager even finishes buzzing. Every downstream customer behind that one IP goes dark at once, whether the attack was aimed at them specifically or not.

That is the breaking number this lesson is really about: **not "how many requests per second," but "how much of the internet's total physical capacity can converge on one point."** A single site, however large you build it, has a ceiling. Cloudflare's actual answer to that ceiling is a routing trick that is older than Cloudflare itself, and doesn't scale by making one place bigger.

## 2. Why the naive (demo) design dies

**Version one: one data center, one public IP, DNS points everyone at it.** This is how almost every service starts, and for a long time it's completely fine. It fails in three specific, compounding ways once traffic (legitimate or hostile) gets large and global.

**Physics puts a floor under latency that no amount of server capacity fixes.** Light in fiber travels at roughly two-thirds the speed of light in vacuum. A round trip between Mumbai and a data center in Virginia, even with a theoretically perfect straight cable, cannot be much faster than roughly 110 to 130 milliseconds; real-world fiber paths, which bend around geography and hop through multiple carriers, typically measure **around 230 milliseconds round trip** between those two regions. That number does not improve by adding servers in Virginia. It only improves by having a server closer to Mumbai. A single-site design forces every user on the planet who isn't near that one site to pay that floor on every request, no matter how fast the application code is.

**One physical location has a finite pipe, and a volumetric attack's entire job is to find and fill that pipe.** However many gigabits a single facility's uplinks can carry in aggregate, that number is public knowledge to anyone who wants to test it, and a 5, 11, or 31 terabit-per-second flood is not a subtle attack, it is a brute-force test of exactly that ceiling. There's no clever application-layer defense that helps once the physical link itself is full; packets are getting dropped indiscriminately, good traffic and bad traffic alike, before they ever reach a server that could tell the difference.

**DNS-based failover is slow because it depends on millions of independent caches expiring on their own schedule.** The standard fix for "what if the one data center dies" is DNS: point a domain at a second IP and lower the TTL. But a DNS answer, once handed to a resolver, sits cached wherever that resolver (an ISP's, a corporate network's, a public one like a phone carrier's) decides to keep it, TTL is a suggestion many resolvers underhold or overhold in practice. A real failover this way is measured in minutes, sometimes tens of minutes, while a meaningful fraction of the internet is still holding the dead IP in cache and retrying it, which, under an active attack, is retry traffic piling onto a link that is already failing, the classic shape of a system's own users making the outage worse while it's happening.

## 3. The architecture

```
[Client anywhere on Earth: Mumbai, São Paulo, Lagos, Tokyo]
   sends a packet to ONE IP address, e.g. 1.1.1.1 or a Cloudflare
   customer's anycast IP
   |
   v
[The internet's own BGP mesh: tens of thousands of independent
 networks, each running its own routers]
   analogy: not a switchboard operator connecting your call, more
   like every post office in the world independently keeping its
   own "which direction is closest to this ZIP code" table, and
   Cloudflare has simply told every post office "this ZIP code is
   local to you, no matter where you are"
   single job: EVERY router along the path independently picks its
   own next hop toward that IP, using its own local BGP table
   (shortest AS-path, local routing policy, peering agreements) —
   no single router, and no Cloudflare system, sees or controls
   the global decision
   |
   v
[The topologically nearest of 337 Cloudflare cities gets the
 packet, because every network on the path independently decided
 that direction was shortest]
   analogy: mail addressed to "the nearest branch of a chain store"
   with no chain-wide dispatcher, every delivery driver on the
   route just picks whichever branch is fewest turns away
   |
   v
[Inside that one city: ECMP (equal-cost multi-path) hashing
 across many physical edge machines, all answering the same
 anycast IP locally]
   single job: spread the city's share of traffic across dozens of
   real servers without breaking any one connection mid-stream
   analogy: one building, many teller windows, a doorman hashes
   your ticket number so you always return to the same window for
   the rest of your visit
   |
   v
[L3/L4 packet-level DDoS scrubbing, automated, no human in the
 loop for the first response]
   single job: drop flood traffic (bad packet shapes, known attack
   signatures, volumetric floods) before it costs a full TCP
   handshake or an application-layer cycle
   |
   v
[L7 edge: reverse proxy, cache, Workers compute]
   single job: serve from local cache when possible, run edge
   logic, this is the layer that answers most legitimate requests
   without going anywhere else
   |
   v (cache miss / dynamic request only)
[Regional / origin-shield tier -> the actual origin, which is
 usually NOT anycast, one or a few real addresses, deliberately
 shielded from ever being hit directly]
   analogy: the newsstand (edge) sells the paper; only a restock
   truck (a cache miss) ever goes back to the one printing plant
```

The property that makes this whole thing work: **nobody, anywhere, decided which of the 337 cities serves a given user.** It falls out of tens of thousands of independently-operated routers each doing ordinary, local, self-interested path selection on a route that happens to be announced identically everywhere. That's also exactly why an attack aimed at one anycast IP doesn't concentrate on one building: the attacking traffic's source machines are scattered across the globe too, and BGP routes each attacking machine's packets to whichever city is nearest to it, the same way it routes a real user's. A 31 Tbps flood doesn't arrive as 31 Tbps at one door, it arrives as some fraction of that at each of 337 doors, and each of those fractions is a problem an individual site's scrubbing capacity can actually chew through.

## 4. The transferable mechanisms

**Anycast: one IP, many physical answers, routing delegated to infrastructure you don't own or operate.** The same IP prefix is announced from every location. There is no proprietary global load balancer making the routing call, the load balancer *is* the internet's own BGP mesh, already running, already maintained by thousands of other organizations for their own reasons. This is the most extreme form of "don't build what already exists": Cloudflare doesn't route packets to the nearest city, it convinces the internet's existing routing fabric to do that on its own.

**ECMP as anycast's little sibling, one level down.** The same trick, spread identical work across many equal-cost paths, repeats inside a single building: dozens of physical machines answer the one anycast IP, and a hash of the connection's five-tuple (source/destination IP and port, protocol) pins a given TCP connection to one machine for its lifetime while spreading different connections across all of them. Same mechanism, smaller radius.

**BGP route withdrawal as failover, not DNS TTL expiry.** If a city becomes unhealthy, overwhelmed, or needs to be pulled for maintenance, Cloudflare simply stops announcing that route from that location. Every router on the internet that was pointing traffic there recalculates its next-best path within seconds, because BGP convergence, while not instant, is an order of magnitude faster than waiting out a DNS TTL held across millions of independent caches. No client-side logic, no cache to expire, the internet's own routers do the failover.

**Distributed absorption as the actual DDoS defense, not a bigger single link.** Because attacking machines are geographically scattered like real users, their traffic naturally splits across the same 337 doors real traffic uses. A flood that would flatten one site's uplinks in seconds becomes, divided across the network, a load each individual scrubbing point has a real chance of filtering. This is the same "spread the hot key across many replicas" idea from Day 16, applied not to a cache key but to physical network ingress points.

**Trust-based routing has no built-in correctness check, which is a real, documented failure mode.** In February 2008, Pakistan Telecom (AS17557), trying to block YouTube locally for domestic censorship, announced a more specific route for a slice of YouTube's address block (208.65.153.0/24, nested inside YouTube's own 208.65.152.0/22). BGP prefers more specific prefixes, so once that announcement leaked to Pakistan Telecom's upstream provider PCCW and from there to the rest of the internet, much of the world started sending YouTube's traffic to Pakistan Telecom instead, and YouTube went dark globally for roughly two hours. Nothing in the base protocol asked "does this network actually own this address block." The real, deployed fix is RPKI (Resource Public Key Infrastructure): a cryptographic Route Origin Authorization that lets a network's real owner sign which autonomous system is allowed to announce their prefixes, so participating routers can reject a forged announcement instead of believing it by default. Anycast rides on the same trust substrate; the same openness that lets any of 337 cities legitimately answer for an IP is the same openness a hijack abuses.

**DNS-based geo-routing (GSLB) is a different, complementary mechanism, not a competitor to be replaced.** Services like AWS Route 53 latency-based routing or Akamai's classic CDN model pick a *different IP per region* by resolving DNS differently depending on where the query appears to come from. It buys fine-grained control (weight traffic, run health checks, steer by business logic) that pure anycast doesn't give you, since anycast's routing decision is emergent, not something you can dial. The trade is TTL-bound failover speed, and a structural accuracy problem: the location DNS sees is the *resolver's* location, not the end user's, which is exactly why a large share of the world routes through a small number of centralized public resolvers (like 8.8.8.8), and why the industry had to invent EDNS Client Subnet just to pass a hint of the real client's location through.

## 5. The trade-offs

**Control versus resilience.** Anycast gives you almost no dial to turn, you cannot tell it "send 30% of Mumbai's traffic to Singapore instead of Chennai this week." What you get instead is failover and load spreading that requires zero orchestration and survives outages BGP-fast. DNS-based GSLB gives you that dial, weighted routing, staged rollouts, custom health checks, at the cost of TTL-bound reaction time and imprecise client-location data. Cloudflare runs anycast for the edge network itself and layers DNS-based tools on top for customers who need the finer control; neither replaces the other.

**Cost versus latency, paid as fixed physical presence.** Being real infrastructure in 337 cities, with peering relationships and hardware in every one, is an enormous fixed cost compared to a handful of big regional data centers. It buys the latency floor down near tens of milliseconds for almost everyone on Earth (Cloudflare states 95% of the internet-connected population sits within 50ms of one of its locations) and it buys DDoS capacity that scales with the size of the whole network rather than the size of any one site. Fewer, bigger regions are cheaper per unit of compute, but every distant user pays the physics tax from Section 2, and every attack has fewer, larger doors to concentrate on.

**Availability of the edge versus consistency of the state behind it.** Anycast solves "get the packet to somewhere nearby, fast, resiliently." It does not solve "make every one of those 337 locations agree on the current value of something that changes." A cache read or a static asset is trivially fine to answer from whichever nearby city the packet happened to land on. A write, a balance update, a rate-limit counter that must not double-count, cannot be answered correctly by 337 independent locations each with their own local view; that has to funnel back to a smaller, deliberately non-anycast set of authoritative places, exactly the root/leaf split Day 47's Quicksilver lesson covers for Cloudflare's own config data. Anycast is the reason the read side of Cloudflare's network can be almost infinitely distributed while the write side still has to be small and careful.

## 6. The systems-thinking lens

**The feedback loop a single-site design walks into: centralization makes the site the physical choke point for both legitimate load and hostile load, and adding server capacity at that one site doesn't remove the choke point, it just moves the ceiling up by a fixed amount before the next attack clears it too.** Trace it through: all traffic, real and attacking, is forced through one address that maps to one place → an attacker doesn't need to out-clever the application, they only need to fill the one physical pipe every request already has to cross → once the pipe saturates, real users' retries and reconnect attempts add to the exact same saturated link, worsening the outage the moment it starts, the same self-reinforcing shape as a retry storm → and DNS-based recovery is slow precisely because the "fix" (point the domain elsewhere) has to propagate through millions of independent caches that don't all listen at once, so the system stays broken well past the moment the fix was issued.

**The senior fix doesn't make the one site's pipe bigger; it removes the idea that there is one site at all.** Anycast doesn't defend the choke point better, it deletes it, by making "the location that answers this IP" a question with 337 simultaneously true answers instead of one. That's the same structural move as Quicksilver's root/leaf split (Day 47, separate the thing that must be singular from the thing that can be everywhere) and as consistent hashing spreading a hot key across replicas (Day 16): don't add capacity to the bottleneck, remove the requirement that there be a bottleneck at all. The remaining risk isn't "not enough capacity," it's trust: BGP will happily route traffic toward a false announcement exactly as fast as a true one, which is why the mature version of this architecture pairs anycast's decentralization with RPKI's cryptographic route validation, decentralize the routing decision, but don't decentralize away the ability to tell a real announcement from a forged one.

---

## Map to Rare.lab's stack

**What's already anycast, for free, without anyone at Rare.lab configuring it.** Every asset Rare.lab serves out of Cloudflare R2, and every Cloudflare Worker in front of the embeddable runtime, is already answered from whichever of Cloudflare's 337 cities is nearest to whoever embedded that runtime's page. A scene JSON blob or a compiled shader asset fetched by a viewer in Singapore never has to cross the ocean to wherever Rare.lab's "home" region is, it's served from an edge cache that's already local. This is Day 55's whole lesson, already bought and paid for, sitting underneath the content-addressed R2 manifest pattern from Day 23, without Rare.lab having built a single line of routing logic.

**The next ceiling: everything that isn't a cache hit still has exactly one home.** Anycast fixes the read path for content that can be cached at the edge. It does nothing for a request that has to be answered authoritatively by Supabase Postgres, an RLS-gated read, a license check, a write. Those requests, no matter how close the anycast edge got the user, still have to travel the rest of the way to wherever Supabase's single region actually lives, paying close to the full Section 2 physics tax on every single one. As Rare.lab's embed count grows and a larger share of runtime traffic needs a dynamic answer rather than a static asset, that gap between "edge already solved" and "origin still one place" is exactly where the next latency complaint comes from, and it's the same gap Day 47's lesson names for Cloudflare's own config data: push what can be cached and validated at the edge (Workers KV, Durable Objects for narrowly-scoped coordination) out to where anycast already reaches, and keep the smallest possible slice of genuinely authoritative state, everything else that still funnels through one Postgres region, as small and rare as the workload allows.

---

## References and summaries

**Cloudflare Learning Center: "What is Anycast? How does Anycast work?"**
https://www.cloudflare.com/learning/cdn/glossary/anycast-network/
Cloudflare's own plain-language explainer of anycast: the same IP address advertised from multiple locations, with the internet's routers using BGP path selection (shortest AS-path, local routing policy) to decide which location handles a given user, rather than any central dispatcher making that call. Used here as the baseline definition the rest of the lesson builds on.

**Cloudflare Blog: "A Brief Primer on Anycast"**
https://blog.cloudflare.com/a-brief-anycast-primer/
Cloudflare's own technical background piece on why they moved to anycast for their entire network early on, and how it differs from unicast-plus-DNS-based load balancing. Direct fetch was blocked by this session's network egress policy (blog.cloudflare.com is not reachable from this environment); the explanation in Sections 3 to 4 is built from the Learning Center glossary entry above plus independent corroborating summaries, not a direct quote from this post, and it's worth a first-hand read before citing specific wording from it elsewhere.

**Cloudflare network page**
https://www.cloudflare.com/network/
Source for the current footprint figures used in Section 1: 337 cities, 100-plus countries, 8 regions, 13,000-plus network interconnections, and the claim that 95% of the internet-connected population sits within 50 milliseconds of a Cloudflare location. Cloudflare's own marketing/status page rather than an independently audited figure; used here for general scale context, consistent with how Day 47's lesson treated the same page.

**Cloudflare DDoS threat report for 2024 Q4**
https://blog.cloudflare.com/ddos-threat-report-for-2024-q4/
Primary source for the 5.6 Tbps DDoS attack mitigated during the last week of October 2024, at the time the largest ever publicly reported. Direct fetch blocked in this session (blog.cloudflare.com egress-blocked); figure corroborated across multiple independent news summaries (BleepingComputer, SecurityWeek, The Hacker News) that all cite this report.

**Cloudflare Blog: "Defending the Internet: how Cloudflare blocked a monumental 7.3 Tbps DDoS attack"**
https://blog.cloudflare.com/defending-the-internet-how-cloudflare-blocked-a-monumental-7-3-tbps-ddos/
Source for the May 2025 7.3 Tbps attack figure, part of the same escalating pattern of record attacks referenced in Section 1. Not directly fetchable in this session; relayed via search-indexed summaries.

**The Hacker News / SecurityWeek: coverage of the 11.5 Tbps DDoS attack (September 2025)**
https://thehackernews.com/2025/09/cloudflare-blocks-record-breaking-115.html
https://www.securityweek.com/cloudflare-blocks-record-11-5-tbps-ddos-attack/
Independent security-press coverage of Cloudflare's announcement that it autonomously detected and mitigated an 11.5 Tbps DDoS attack lasting roughly 35 seconds, without human intervention, used here as the second data point in Section 1's escalation from 5.6 to 31.4 Tbps.

**Cloudflare DDoS threat report for 2025 Q4**
https://blog.cloudflare.com/ddos-threat-report-2025-q4/
Primary source for the most recent record cited, a 31.4 Tbps attack lasting 35 seconds, part of a described 700%-plus year-over-year escalation in peak attack size through 2025. Direct fetch blocked in this session; relayed via search summaries corroborated across BleepingComputer and Cloudflare Radar's own report landing page (https://radar.cloudflare.com/reports/ddos-2025-q4).

**Renesys / Google Research: "YouTube Hijacking (February 24th 2008): Analysis of BGP Routing Dynamics"**
https://research.google/pubs/youtube-hijacking-february-24th-2008-analysis-of-bgp-routing-dynamics/
Primary technical analysis of the Pakistan Telecom BGP hijack used in Section 4: Pakistan Telecom (AS17557) announced the more specific prefix 208.65.153.0/24 (a subset of YouTube's 208.65.152.0/22) intending to block YouTube domestically, the announcement leaked to upstream provider PCCW (AS3491) and propagated globally, and because BGP prefers more specific prefixes, much of the internet's YouTube traffic was misrouted to Pakistan Telecom for roughly two hours.

**MANRS: "What is BGP prefix hijacking? (Part 1)"**
https://manrs.org/2020/09/what-is-bgp-prefix-hijacking-part-1/
General background on BGP hijacking as a class of failure and on RPKI (Resource Public Key Infrastructure) and Route Origin Authorizations as the deployed, cryptographic fix, cited in Section 4 and the systems-thinking lens as the real answer to "BGP trusts announcements by default."

**APNIC Blog: search-indexed coverage of anycast site-selection accuracy (including "Autocast: Automatic anycast site optimization" and related posts)**
https://blog.apnic.net/2025/10/08/autocast-automatic-anycast-site-optimization/
Used for the nuance in Section 4 that BGP's path selection is driven by AS-path length and provider routing policy, not literal geography, so anycast does not always route a client to the physically nearest site; provider-specific policy and peering relationships can and do produce a farther, though still BGP-optimal, choice.

---

*Inference vs. fact, stated plainly: the 337 cities / 100-plus countries / 13,000-plus interconnections / 95%-within-50ms figures come from Cloudflare's own current network page. The 5.6 Tbps (Oct 2024), 7.3 Tbps (May 2025), 11.5 Tbps (Sept 2025), and 31.4 Tbps (Q4 2025) DDoS figures come from Cloudflare's own DDoS threat reports and press announcements, cross-corroborated through independent security-press coverage since blog.cloudflare.com could not be fetched directly in this session. The Pakistan Telecom/YouTube hijack details (the specific prefixes, AS numbers, and the roughly two-hour outage) come from Google Research's and Renesys's contemporaneous technical write-ups of the incident. The "a single site's aggregate uplink capacity is roughly a few hundred gigabits to low single-digit terabits" figure in Section 1, and the "10 to 50 times" comparison to the 31.4 Tbps attack, are this lesson's own illustrative estimate, not a Cloudflare-published number, offered to make the scale of the gap concrete rather than as a precise industry benchmark. The Mumbai-to-Virginia ~230ms round-trip figure is a commonly measured typical range for that city pair on real-world internet paths, not a single authoritative source's number. The architecture diagram, the transferable-mechanisms framing, the GSLB-versus-anycast contrast, the trade-offs section, the systems-thinking feedback-loop framing, and the Rare.lab mapping are this lesson's own analysis layered on top of the documented facts above, not claims made verbatim by Cloudflare.*
