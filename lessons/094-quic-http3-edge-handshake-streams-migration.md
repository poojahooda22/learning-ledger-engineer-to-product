# Day 94: A phone on a bad network opens your page in one round trip and does not stall when one packet dies. How QUIC and HTTP/3 (Google, Meta, Cloudflare) beat TCP at the edge

**Date:** 2026-10-20
**Difficulty:** Advanced (transport layer, the part under every other lesson)
**Topic:** Every earlier lesson assumed "the client talks to the edge". Today is what happens on the wire. Day 4 (CDN), Day 55 (anycast), Day 76 (tail latency) and Day 79 (uploads) all sat on top of TCP. QUIC is the replacement, and it is the reason Cloudflare, Google and Meta feel fast on mobile.
**Stack relevance:** Rare.lab's embeddable runtime fetches a manifest and scene JSON from Cloudflare R2 behind Cloudflare's edge. Cloudflare already speaks HTTP/3 to browsers. Section 9 shows what that gives you for free and what you can tune.
**A note on the pasted transcript:** the LeetCode mock interview is already taught in [Day 74](074-leetcode-online-judge-end-to-end.md) (sandbox in Day 50, leaderboard in Day 54). I did not repeat it. I reused its recipe (requirements, entities, API, naive design, deep dives) in Section 0, and summarized it again in Section 10 because you asked for every reference to be explained.
**Honesty labels:** CONFIRMED = read in a source today (search excerpts, not full pages; several sites were not fetchable). INFERENCE = my reasoning, labeled.

---

## 0. The interview recipe applied to a transport protocol

1. **Functional.** Carry bytes reliably and in order between a client and a server. Multiplex many requests (HTML, JSON, images) over one connection. Encrypt everything.
2. **Non-functional.** Low latency to first byte. Survive packet loss. Survive the user walking from Wi-Fi to 4G. Resist abuse (spoofed floods). Be deployable on the real Internet, full of old routers.
3. **Entities.** Connection, connection ID, stream, packet, congestion window, session ticket.
4. **API.** `open connection -> open stream -> send/recv bytes -> close`. For HTTP/3: `GET /manifest.json` on stream 4.
5. **Naive design.** TCP plus TLS plus HTTP/2. This is the "one server, one database" of networking: it works fine on a wired office network.
6. **Deep dives.** The break numbers, the three fixes (handshake, streams, connection ID), then the rollout trap.

---

## 1. Named companies and the breaking numbers

**Google, Meta, Cloudflare.**

- **250 ms**: a round trip from India to a US server (speed of light in fibre plus routers). TCP plus TLS 1.3 needs about 2 round trips before the first byte of your request can be sent, so you pay ~500 ms before anything happens. (Round trip counts: CONFIRMED in general by the search results, "QUIC merges transport and TLS setup"; exact arithmetic is mine.)
- **1 lost packet in 50** on a weak mobile link (2 percent loss is a common figure for poor cellular, INFERENCE as a typical value). Under HTTP/2 over TCP, ONE lost packet freezes ALL the multiplexed requests on that connection until it is resent.
- **Google's measured payoff (CONFIRMED, SIGCOMM 2017 paper, via search excerpt):** QUIC cut Google Search latency by 8.0 percent on desktop and 3.6 percent on mobile, and cut YouTube rebuffer rates by 18.0 percent on desktop and 15.3 percent on mobile. QUIC was about 7 percent of global Internet traffic then and over 30 percent of Google's egress bytes.
- **Meta's measured payoff (CONFIRMED, Meta engineering blog Oct 2020, via search excerpt):** versus HTTP/2, 6 percent fewer request errors, 20 percent lower tail latency, and 5 percent smaller response headers on Facebook. Video: 8 percent fewer errors and 20 percent fewer stalls, with the biggest wins on poor networks in emerging markets. More than 75 percent of Meta's Internet traffic uses QUIC and HTTP/3.
- **Adoption (CONFIRMED but varies by source):** roughly 30 percent of HTTP requests seen by Cloudflare use QUIC per Radar analyses; one older Cloudflare post says ~12 percent when RFC 9000 was published. Treat 20 to 30 percent as the honest band.

**The breaking number for the naive design: 2 round trips of setup at 250 ms, plus ONE lost packet freezing every request on the connection.**

---

## 2. Why the naive design (TCP + TLS + HTTP/2) dies

**Failure 1: Setup tax.**
- TCP handshake (SYN, SYN-ACK, ACK) = 1 round trip. TLS 1.3 handshake = 1 more. Only then does `GET /` leave your phone.
- Analogy: you must ring the doorbell, wait, show ID at the intercom, wait, and only then say why you came.

**Failure 2: Head-of-line blocking.**
- HTTP/2 multiplexes 20 requests onto ONE TCP connection (good: one handshake). But TCP only knows "one ordered byte pipe". If packet 7 is lost, packets 8 to 40 sit in the kernel buffer until 7 is resent, even if they belong to a different, perfectly fine request.
- CONFIRMED (search excerpts): "TCP stalls every multiplexed stream when one packet is lost, while QUIC confines the stall."
- Analogy: one lane, one broken-down car, every van behind it waits, even the ones going to a different street.

**Failure 3: The connection is its address.**
- TCP names a connection by the four-tuple: source IP, source port, destination IP, destination port. CONFIRMED in the excerpts: "change any one and the connection dies."
- You leave home Wi-Fi, your phone gets a new IP, every TCP connection dies, and you pay failure 1 again for every site.
- Analogy: a phone call that is identified by the handset's physical location, so walking out the door hangs up.

**Failure 4: Ossification (the real killer).**
- TCP lives in the operating system kernel and is read by every router and firewall. Changing it means waiting years for phones and middleboxes to update. Google tried TCP improvements for years; the way out was to put the new protocol on top of UDP, in user space, so apps can ship it with an app update. (INFERENCE on motivation, consistent with the Google paper's framing that QUIC runs in user space and is deployed with Chrome.)

---

## 3. The architecture, top to bottom

```
 Client (Chrome, Safari, mobile app, embedded runtime in a page)
        |   first visit: HTTP/2 over TCP + TLS. Server replies with
        |   Alt-Svc: h3=":443"   ("next time, talk QUIC to me")
        |   later visits: UDP port 443, QUIC v1 (RFC 9000)
        v
 [1] ANYCAST EDGE (Day 55)       nearest data centre answers
        |
        v
 [2] QUIC TERMINATION            one UDP socket, many connections.
        |                        crypto (TLS 1.3 inside QUIC, RFC 9001),
        |                        loss recovery, congestion control per connection
        v
 [3] HTTP/3 (RFC 9114)           each request = one QUIC stream
        |                        header compression (QPACK)
        v
 [4] EDGE CACHE / WORKER         serve from cache or fetch from origin (Day 4)
        |
        v
 [5] ORIGIN (R2, your API)       usually still HTTP/1.1 or HTTP/2 over TCP
                                 (the lossy last mile is client to edge, not edge to origin)
```

Layer jobs and analogies:
- **Alt-Svc discovery:** a business card you hand out on the first visit saying "call my new number next time". CONFIRMED in the Cloudflare excerpt: the server returns an `Alt-Svc` header listing supported versions, so only the second and later connections use QUIC.
- **UDP socket:** a postbox with no promises. QUIC builds its own reliability on top, because UDP passes through old routers better than any brand-new IP protocol.
- **QUIC connection:** the delivery contract. Carries encryption, loss recovery, flow control.
- **Streams:** separate lanes in one motorway. Each has its own ordering. A crash in lane 3 does not stop lane 5.
- **Connection ID:** a ticket number, not an address. The ticket stays valid when you change address.
- **Edge to origin on TCP:** a short, clean, datacentre-grade link. QUIC's wins are on the long, lossy, mobile hop, so you do not need QUIC there. (INFERENCE, standard practice.)

---

## 4. The transferable mechanisms (6)

1. **Merge the handshakes (transport + crypto in one).**
   - QUIC carries the TLS 1.3 handshake inside its own first packets. New connection: 1 round trip until the request can go. CONFIRMED in excerpts.
   - Reusable lesson: when two protocols each add a round trip, fuse them. Cost in the wild is the same idea as batching.

2. **Resumption with 0-RTT (and its replay price).**
   - Returning client sends the request in its FIRST packet using a saved session ticket: 0 extra round trips.
   - The price: an attacker can record and replay that first packet. 0-RTT has NO replay protection (CONFIRMED, RFC 9001 section 5.6 as cited in RFC 9308 and the IETF slides). So limit it to safe, idempotent methods like GET and HEAD; servers can answer 425 Too Early to force a full handshake. HTTP/3 itself restricts 0-RTT to safe methods (CONFIRMED in IETF slide summary).
   - Reusable lesson: it is Day 12 (idempotency) again. Speed shortcuts are only safe for operations that survive being done twice.

3. **Independent streams (kill head-of-line blocking).**
   - Loss recovery is per stream. A lost packet delays only the stream it belongs to. CONFIRMED.
   - Reusable lesson: do not share one ordered queue across unrelated work. This is the same instinct as Day 9 (queues) and Day 76 (tail latency).

4. **Connection ID (identity independent of the network path).**
   - After the Wi-Fi to 4G switch, the client sends packets from a new IP with the same connection ID, and the server runs a path validation exchange before trusting the new path. CONFIRMED in excerpts.
   - Reusable lesson: name things by a stable ID, not by where they currently live. Same shape as consistent hashing (Day 10) and content addressing (Day 23).

5. **Anti-amplification limit (do not become a DDoS gun).**
   - UDP lets an attacker forge the source address. A server that answers a tiny packet with a big handshake reply would blast a victim. The limit: before the client's address is validated, the server may send at most 3 times what it has received. CONFIRMED from IETF slide excerpt. An NDSS 2023 paper found some of 13 tested implementations did not enforce it properly (CONFIRMED in excerpt).
   - Reusable lesson: a cheap request that triggers an expensive response is an attack surface. Cap the ratio. (Same family as Day 13, load shedding.)

6. **Fallback and gradual rollout.**
   - Chrome falls back to TCP when UDP is blocked or the path is too small. Google measured about 0.3 percent of users behind networks that rate-limit or block QUIC/UDP (CONFIRMED, excerpt of SIGCOMM 2017).
   - Meta started with only dynamic GraphQL requests in the Facebook app, then widened (CONFIRMED in Gigazine summary of the Meta post).
   - Reusable lesson: ship the new path next to the old path, with an automatic escape hatch. Day 68 (feature flags) applied to a protocol.

---

## 5. The trade-offs accepted

- **Latency vs CPU (cost).** QUIC moved packet handling from the kernel to user space and encrypts every packet. The Google-era CPU load was higher than TCP/TLS; one survey says about 3.5 times, another summary says about 2 times. The sources conflict, so treat as "meaningfully more CPU" and read the paper for the exact multiple (CONFIRMED conflict, resolved nowhere). Cost buys latency. Hardware offload and kernel-bypass work have narrowed this since (INFERENCE).
- **Speed vs safety (0-RTT).** Faster first byte, but replayable. Choice: allow only for GET-like requests. Never for "charge card".
- **Availability vs novelty.** Some networks drop UDP. The choice is availability: always keep TCP as a fallback. A fraction of users silently get the slower path.
- **Consistency per data type.** Transport has no CAP problem, but the principle holds: the manifest (must be fresh) and a cached texture (can be stale) have different rules. QUIC does not change that. It just delivers both sooner.
- **Visibility vs privacy.** QUIC encrypts almost everything including transport headers, so operators lose the ability to inspect and shape it. RCR Wireless / Enea commentary says mobile operators worry about exactly this (CONFIRMED as an opinion piece, vendor-written, treat as biased).

---

## 6. The systems-thinking lens

**The loop: the congestion-and-retry spiral on a lossy link.**
- Packet lost -> TCP stalls the whole connection -> app sees a timeout -> app retries the request or opens a NEW connection -> more packets and a new handshake on a link that is already lossy -> more loss. A metastable failure (Day 13) at the user's edge.

**Meta's real-world version (CONFIRMED, Meta post excerpt):** even though QUIC was enabled only for dynamic requests, error rates for static content over TCP went UP. Cause: app logic changed the type and number of requests based on how other requests were performing, and bandwidth estimation tuned for TCP overestimated network quality under QUIC. One fix to one transport changed the behavior of a feedback loop elsewhere.

**Senior fixes break the loop; they do not add capacity:**
- Per-stream loss recovery, so one loss does not trigger app-level retries of everything.
- Congestion control per connection with pacing, so a loss slows the sender instead of triggering a burst.
- Retries with jittered backoff in the app (Day 13, Day 83).
- Measure the whole system when you change a lower layer. Meta's lesson is to watch the metrics of the traffic you did NOT change.

---

## 7. Scale story at three tiers

**1,000 users (one small site):**
- TCP + TLS 1.3 + HTTP/2 behind any CDN is fine. Handshake cost is invisible when users are close and on good links. Turning on HTTP/3 in Cloudflare is one toggle, no engineering. Do not build anything.

**100,000 users (a global product):**
- Users are spread across continents and half are on mobile. Handshake round trips and loss show up in your p95 (Day 76). HTTP/3 pays off here. What breaks next: you start caring about connection reuse, fallback rate, and the fraction of users that never get HTTP/3.

**10 million plus (Google, Meta):**
- UDP packet processing in user space dominates CPU (CONFIRMED trend, higher CPU than TCP). Meta built its own stack (`mvfst`) and Google built theirs; Cloudflare built `quiche` (the search did not return quiche internals, so I state only that it exists as Cloudflare's open source QUIC library). Cloudflare Radar-scale deployments need load balancers that route by connection ID so packets from a migrated client still reach the same machine (INFERENCE, grounded in how Day 85 L4 balancing with connection tracking works; a stateless hash of the destination connection ID is the usual approach).
- Walls: CPU per packet, load balancers that only understand TCP, middleboxes dropping UDP, and observability (encrypted headers).

---

## 8. A worked example, end to end

A user in Pune on a train opens a page that uses Rare.lab's runtime. Cloudflare serves the page.

1. First visit: TCP + TLS 1.3 + HTTP/2. About 2 round trips at 120 ms = 240 ms before the request leaves. Response has `Alt-Svc: h3=":443"; ma=86400`.
2. The runtime asks for `manifest.json` plus 12 scene blobs. Over HTTP/2, 13 requests share 1 TCP connection. One packet drops in the tunnel. All 13 pause ~200 ms (one retransmit timeout, assumed value).
3. Second visit (next day): browser tries UDP 443 with QUIC. 1 round trip (120 ms) instead of 2. With a saved ticket and a GET, 0-RTT sends the request immediately: 0 extra round trips (replay-safe because GET and the manifest is read-only).
4. Same tunnel, same drop. The 13 requests are 13 QUIC streams. Only the stream that lost a packet pauses. The other 12 finish.
5. The train switches towers. The phone's IP changes. Connection ID stays. After path validation the transfer continues without a new handshake.
6. A corporate Wi-Fi blocks UDP. The browser falls back to TCP in the background. Page still works, just slower.

---

## 9. Map to Rare.lab's stack

| Pattern from today | Do we use it? | Where is our ceiling? |
|---|---|---|
| HTTP/3 to the browser | **Free.** Cloudflare terminates QUIC. Enable and verify HTTP/3 on the zone. | Check that the embed host page and the R2 custom domain BOTH advertise `Alt-Svc`; a different asset domain means a new connection and a new handshake. |
| Streams without head-of-line blocking | **Free once on h3.** Manifest + N scene blobs load in parallel. | Keep blobs small and immutable (you already do, content-addressed). A 4 MB single blob is one stream and still slow; split by node group so partial scenes can render. |
| 0-RTT | **Safe for you** because manifest and scene GETs are read-only. | Never route writes (Supabase RLS inserts) over 0-RTT paths. Cloudflare handles this per request, but check Workers that proxy writes. |
| Immutable content-addressed blobs (Day 23) | **Yes.** Perfect for cache plus `Cache-Control: immutable`. | The manifest is the one mutable pointer (Day 93). Give it a short TTL or a version check, because transport speed does not fix staleness. |
| Fallback | Automatic in browsers. | Your embeddable runtime inside a customer's page may run behind a customer's firewall with UDP blocked. Measure the TCP-fallback share with Cloudflare analytics or a Real User Monitoring beacon; do not assume everyone gets h3. |
| One shared WebGL context | Not a network item. | Network streams finishing out of order is now normal. Make the runtime accept scene pieces in any order and render when the manifest's required set is complete. |

**Next ceiling (INFERENCE):** not the protocol. It is the number of round trips your app logic chains: manifest -> scene -> textures -> shaders. Each step is a full round trip even on QUIC. Fix by inlining the manifest in the embed snippet, using `103 Early Hints` or `<link rel=preload>` for the scene hash, and bundling the first screen of nodes. Fewer dependent fetches beats a faster protocol.

---

## 10. References, summarized in plain language

(Most pages could not be fetched; summaries come from search excerpts and my reading of titles. Where I could not verify, I say so.)

1. **Langley et al., "The QUIC Transport Protocol: Design and Internet-Scale Deployment", SIGCOMM 2017 (Google).** Explains why Google built QUIC on UDP, in user space, shipped via Chrome. Gives the real numbers: search latency down 8.0 percent desktop and 3.6 percent mobile; YouTube rebuffers down 18.0 percent desktop and 15.3 percent mobile; 0.3 percent of users on networks that block or throttle it. Lesson: measure at Internet scale and keep a TCP fallback.
2. **Meta Engineering, "How Facebook is bringing QUIC to billions" (Oct 2020).** Meta built `mvfst`, rolled it out starting with dynamic API calls, saw 6 percent fewer errors, 20 percent less tail latency, and found a surprising regression in TCP static traffic caused by app logic reacting to the changed timings. Lesson: a lower-layer change shifts higher-layer feedback loops.
3. **RFC 9000 (QUIC transport), RFC 9001 (TLS in QUIC), RFC 9114 (HTTP/3), RFC 9308 (applicability).** The standards. 9001 section 5.6 is the 0-RTT replay discussion; 9000 has the 3x anti-amplification limit and connection ID/migration rules. Excerpts only; verify wording on rfc-editor.org.
4. **NDSS 2023 paper on QUIC implementations (amplification).** Tested 13 servers and found some did not enforce the amplification limit. Lesson: a spec is not a guarantee, test the implementation.
5. **IETF 115 slides, "Challenges of 0-RTT in IETF QUIC".** Why 0-RTT is hard: replay, address validation, safe methods only in HTTP/3.
6. **Cloudflare blog, "Examining HTTP/3 usage one year on" and "QUIC version 1 is live on Cloudflare".** Explains Alt-Svc discovery (second visit uses QUIC) and HTTP/3 share of traffic. I only saw excerpts; the percentages in the market vary, so use a band.
7. **APNIC, "Triggering QUIC" (Geoff Huston, 2025).** A measurement view of how often clients actually use QUIC. Not read in detail; included as further reading.
8. **Enea / RCR Wireless piece on mobile operators and QUIC.** Vendor commentary about operators losing visibility into encrypted transport. Treat as opinion.
9. **The pasted LeetCode mock interview (YouTube).** A Google engineer designs LeetCode: functional requirements (list problems, view and code a problem, submit and get instant verdict, live contest leaderboard), non-functional (availability over consistency, low latency, isolation of user code, 100k contest users, no single point of failure), entities (problem, user, submission, leaderboard), REST APIs with pagination, DynamoDB for problems, code run in language-specific containers (the interviewer pushed back that VMs isolate better but cost more, and suggested serverless), a queue between API and runners with retries, Redis cache for the leaderboard, serialized test cases per language, and primary-replica databases. Full teaching is in [Day 74](074-leetcode-online-judge-end-to-end.md). Link to today: the "instant feedback" requirement is a latency budget, and a QUIC/HTTP/3 front door shaves the handshake off every submit.

---

## 11. One-line takeaway

**Fuse the handshakes, give every request its own lane, name the connection by an ID instead of an address, only take speed shortcuts on safe-to-repeat requests, and always keep the old road open.**

## Sources

- [Google Research, The QUIC Transport Protocol: Design and Internet-Scale Deployment](https://research.google/pubs/the-quic-transport-protocol-design-and-internet-scale-deployment/)
- [SIGCOMM 2017 paper PDF](https://networks.cs.northwestern.edu/CS497-s22/papers/sigcomm17-quic.pdf)
- [Meta Engineering, How Facebook is bringing QUIC to billions](https://engineering.fb.com/2020/10/21/networking-traffic/how-facebook-is-bringing-quic-to-billions/)
- [Gigazine summary of Meta QUIC rollout](https://gigazine.net/gsc_news/en/20201024-facebook-quic-replacement)
- [NDSS 2023 QUIC amplification paper](https://www.ndss-symposium.org/wp-content/uploads/2023-72-paper.pdf)
- [IETF 115, Challenges of 0-RTT in IETF QUIC](https://datatracker.ietf.org/meeting/115/materials/slides-115-tdd-sessa-challenges-of-0-rtt-in-ietf-quic-linked-from-deployment-talk-00)
- [RFC 9000 summary](https://ipfyi.com/rfc/9000/)
- [Cloudflare, Examining HTTP/3 usage one year on](https://blog.cloudflare.com/http3-usage-one-year-on)
- [Cloudflare, QUIC version 1 is live on Cloudflare](https://blog.cloudflare.com/quic-version-1-is-live-on-cloudflare/)
- [APNIC, Triggering QUIC](https://blog.apnic.net/2025/07/15/triggering-quic/)
- [QUIC and HTTP/3 explainer](https://karolbroda.com/series/networking/03-transport-mechanics/quic-and-http3)
- [Enea on QUIC and mobile operators](https://www.enea.com/insights/heres-why-quic-is-still-worrying-mobile-operators)
