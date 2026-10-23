# Day 96: A 1 MB router buffer turns a 10 Mbps link into a 0.8 second delay. How TCP congestion control (CUBIC vs Google's BBR) decides how fast YouTube, Netflix and Cloudflare may send

**Date:** 2026-10-23
**Difficulty:** Advanced (the transport layer under every lesson so far)
**Topic:** Day 94 made the handshake cheap. Today is what happens AFTER the handshake: how fast is the sender allowed to push bytes before it breaks the network? Every CDN, video stream and R2 download rides on this answer.
**Stack relevance:** Rare.lab ships scene JSON, textures and a runtime bundle over Cloudflare. You do not write congestion control, but you choose payload size and request count, and those decide whether a user lives in the fast "slow start" zone or the slow steady zone. Section 9.
**A note on the pasted transcript:** the LeetCode mock interview is already taught in [Day 74](074-leetcode-online-judge-end-to-end.md) (sandbox Day 50, leaderboard Day 54). I did not repeat it. Section 0 uses its interview recipe on today's topic, and Section 10 explains it again in plain language, as you asked.
**Honesty labels:** CONFIRMED = read in a source today. Important limit: web fetch was not used, I only saw search excerpts and summaries. Numbers from the original ACM Queue BBR paper reached me through a secondary blog summary and Google's Linux kernel commit message, so check them against the paper. INFERENCE = my reasoning, labeled. MEMORY = standard textbook facts I know but did not re-read today.

---

## 0. The interview recipe applied to a congestion controller

The LeetCode candidate went: functional requirements, non-functional requirements, entities, API, naive design, then deep dives. Same recipe here.

1. **Functional.** Send N bytes from server to client as fast as the path allows, without a human tuning anything.
2. **Non-functional.** High throughput. Low delay (queueing delay is the enemy). Fair to other flows sharing the link. Works on Wi-Fi, LTE, fiber and satellite. Needs no change inside the network (you do not own the routers).
3. **Entities.** Sender, receiver, bottleneck link (the slowest hop), router buffer (a queue), congestion window (cwnd, how many bytes may be in flight), pacing rate, round trip time (RTT).
4. **API.** There is no REST API. The "API" is signals: an ACK arrived, a packet was lost, an ECN mark arrived. The algorithm turns those signals into one output: how many bytes may I send now?
5. **Naive design.** Send everything at once. Or: send at a fixed rate someone configured.
6. **Deep dives.** The breaking numbers, loss-based vs model-based control, bufferbloat, the fairness fight, scale tiers.

---

## 1. Named companies and the breaking numbers

- **Bufferbloat, the delay number (MEMORY plus arithmetic).** A home router with a 1 MB buffer, behind a 10 Mbps uplink. When the buffer is full, a new packet waits for 1 MB x 8 / 10 Mbps = **0.8 seconds** before it even leaves. Your video call has an 80 ms budget. Your ping to the router was 5 ms an hour ago. Nothing is broken. The queue is just full.
- **Google, YouTube and google.com (CONFIRMED, Linux kernel commit that added BBR, and the ACM Queue paper via a summary).** Google put BBR on its internal backbone, on google.com and on YouTube web servers. The kernel commit says BBR "significantly increased throughput and reduced latency." The paper summary I read reports 2x to 25x higher throughput than CUBIC in some tests, and a median RTT cut of 53 percent globally and more than 80 percent in developing regions on YouTube. Treat these exact figures as unverified until you read the paper.
- **The loss number that cripples loss-based control (MEMORY, the Mathis formula).** Classic TCP throughput is roughly `1.22 x MSS / (RTT x sqrt(loss))`. With MSS 1460 bytes, RTT 100 ms:
  - 1 percent random loss: about **1.4 Mbps**.
  - 0.01 percent loss: about **14 Mbps**.
  - The link might be 1 Gbps. The algorithm is the cap, not the pipe.

**The breaking number for the naive design: one full 1 MB buffer adds 0.8 seconds to every request, and 1 percent random loss caps a 1 Gbps path near 1.4 Mbps.**

---

## 2. Why the naive design dies

**Failure 1: Send everything at once (congestion collapse).**
- 1,000 senders each blast their full file. The bottleneck queue overflows, packets drop, every sender retransmits, which adds more load. In the 1980s this literally happened on the early internet: throughput on a link fell by a factor of about 1000 (MEMORY, Van Jacobson 1988). TCP congestion control exists because of that.
- Analogy: a highway on-ramp with no meter. Everyone merges at once, the highway stops, and the jam is made of cars all trying to get through.

**Failure 2: Fixed rate (the configured pipe).**
- Set 50 Mbps and the user is on a 20 Mbps hotel Wi-Fi: you create a queue. User is on 500 Mbps fiber: you waste 90 percent of it.
- The path changes every second (someone else starts a download, you walk into another room). No fixed number stays right.

**Failure 3: Treat packet loss as the only sign of trouble (the classic design).**
- CUBIC (the Linux default for years) and Reno grow the window until a packet drops, then cut it. Two bugs follow.
  - With a deep buffer, no loss happens until the buffer is full, so the algorithm keeps the queue FULL. That is bufferbloat, caused by the algorithm itself.
  - On a lossy path (Wi-Fi, cellular, a flaky transoceanic link) random loss is not congestion, but the algorithm backs off anyway. That is the 1.4 Mbps problem.
- Analogy: driving by feeling for the car in front only after you hit it.

---

## 3. The architecture, top to bottom

This is a control loop, not a server stack. Draw it as a feedback cycle.

```
Application (YouTube player, curl, browser fetching R2 object)
   |  writes bytes into the socket
   v
Sender TCP/QUIC stack  (runs INSIDE the kernel or the QUIC library)
   |   Congestion controller: CUBIC or BBR       <- the brain of today's lesson
   |        inputs: ACKs, RTT samples, loss, ECN marks, delivery rate
   |        outputs: cwnd (max bytes in flight), pacing rate (spacing between packets)
   |   Pacer: releases packets one by one at the pacing rate
   |        job: no bursts. (Analogy: a metered on-ramp light, one car every 3 seconds.)
   v
Network path: many hops
   |   BOTTLENECK link = slowest hop, with a BUFFER (a waiting line)
   |        job of the buffer: absorb tiny bursts. Becomes a problem when it stays full.
   v
Receiver
   |  sends ACK for each packet (the sender's only eyes)
   v
Back to the sender: each ACK = "one packet left the network, you may send another"
```

Annotations:
- **The sender is blind.** It cannot see the router queue. It sees only when ACKs return and how late they are. Everything is inference from timing. (CONFIRMED in concept: BBR "requires only changes on the sender side, not in the network or the receiver side," per the kernel commit.)
- **Two knobs, not one.** `cwnd` caps bytes in flight. `pacing rate` spaces them. BBR uses pacing as the main control; classic TCP used cwnd and sent in bursts.
- **Bandwidth-delay product (BDP).** The pipe's volume: bandwidth x round trip time. Link of 100 Mbps, RTT 50 ms: 100,000,000 x 0.05 / 8 = **625 KB**. If you keep exactly 625 KB in flight, the pipe is full and the queue is empty. More than that is pure queue (delay with zero extra speed).

---

## 4. The transferable mechanisms (6)

1. **Slow start (exponential probing).** Start tiny (modern default 10 packets, about 14.6 KB, MEMORY) and double every RTT until a signal says stop. Example: to download a 1 MB file on a 50 ms path you need about 6 or 7 round trips just to ramp up, about 300 ms, even on a gigabit link. **Small payloads never leave slow start. That is why the first 14 KB of a page matter so much.**
2. **Additive increase, multiplicative decrease (AIMD).** Grow by a little each RTT, cut by a lot on trouble. It is what makes many flows converge to a fair split (MEMORY, Chiu and Jain analysis). Analogy: everyone creeps forward in a doorway, everyone backs off sharply when jammed.
3. **Model the path instead of reacting to loss (BBR).** Estimate two numbers: bottleneck bandwidth (BtlBw, the highest delivery rate seen recently) and round trip propagation time (RTprop, the lowest RTT seen recently). Operate at their product. Analogy: instead of driving until you crash, measure how wide the road is and how long it is, then fill it exactly.
4. **Probe, then drain (BBR's gain cycling).** MEMORY of the paper: BBR cycles its pacing rate around the estimate through 8 phases of one RTT each: 1.25x (push, look for more room), 0.75x (drain any queue we just made), then six phases at 1.0x. Every ~10 seconds if RTT never reached a new minimum, it dips cwnd to 4 packets for about 200 ms (ProbeRTT) to re-measure the true propagation delay.
5. **Pacing.** Send smoothly, not in bursts. A burst of 40 packets from one sender hits a shallow buffer all at once and drops some, even when the average rate was fine.
6. **Explicit signals (ECN, L4S).** Instead of dropping a packet to say "slow down," a router can mark it. The sender sees a congestion signal with no loss. This is how data centers (DCTCP) keep queues tiny (MEMORY). BBRv2 added reacting to loss and ECN (CONFIRMED via an IETF-related review excerpt).

---

## 5. The trade-offs accepted

- **Throughput vs delay.** CUBIC picks throughput: it keeps the buffer full so the pipe never idles, and your latency suffers. BBR targets full pipe with an empty queue. YouTube mostly wants throughput, a video call wants delay. One algorithm cannot be ideal for both, so tuning differs by workload.
- **Fairness vs performance (the big open fight).** BBR version 1 could take more than its share when competing with CUBIC on shallow buffers, because it ignores loss (INFERENCE from the BBRv2 motivation). CONFIRMED via search excerpts: BBRv2 added loss and ECN reactions to share better with CUBIC and Reno; BBRv3 is described as a tuning release for coexistence; a 2024 measurement slide deck still reports BBRv3 unfair toward multiple CUBIC flows. So even after three versions, fairness is not settled. The IETF draft of BBRv3 (CONFIRMED to exist, draft-cardwell-ccwg-bbr) is the sign that this is still evolving.
- **Consistency vs availability, per data type (applied to this layer).** The congestion state is per-connection, in the sender's memory. It is never replicated: if the process dies, the connection dies and it restarts at slow start. Availability wins, and nothing needs consistency here. The one shared piece is path metadata cached per destination (some stacks remember an RTT or bandwidth per client IP to skip slow start, INFERENCE, varies by stack).
- **Cost vs latency.** BBR needs precise pacing timers (more CPU per packet, or hardware offload). A CDN doing millions of flows pays that in cores. Google moved this into the kernel's fq pacing and TCP-layer pacing so the cost is small (CONFIRMED in concept via the kernel commit; I did not read the commit's details).
- **Where this lives.** Because it is sender-side, the operator who owns the server (Google, Netflix, Cloudflare) can deploy it alone, with zero help from ISPs or users. That is why it spread quickly.

---

## 6. The systems-thinking lens

**The loop that causes failure: the bufferbloat standing queue.**
- Loss-based control grows the window until loss. A deep buffer delays loss. So the window grows far past the BDP. Now the extra bytes are all queue. RTT rises (0.8 s in our example). Higher RTT means ACKs come back later, which slows how fast the window can react. Eventually the buffer overflows, a huge burst of drops hits, every flow cuts at the same instant, and the pipe goes idle. Then everyone restarts together. (Reinforcing delay, then a synchronized crash, then repeat. Related to the thundering herd in Day 16.)
- A second loop: **retransmission storms.** Loss triggers retransmit, retransmit adds load, load triggers more loss. This is the retry death spiral from Day 13 at packet level.

**The senior fix BREAKS the loop, it does not buy a bigger pipe:**
- Control the QUEUE, not the pipe: keep inflight near BDP so the queue stays near zero. That is BBR's whole idea.
- Make the network say it early: ECN marks and active queue management (CoDel, fq_codel) so the signal arrives before the buffer fills (MEMORY).
- Desynchronize: pacing and gain cycling stop every flow from reacting at the same instant.
- Do not add a 100 MB buffer "to avoid drops." That is exactly what created the problem.

---

## 7. Scale story at three tiers

- **1,000 connections a day (a small site):** the default stack (CUBIC, 10 packet initial window) is fine. Nothing to tune. The break: a user on a bad mobile link sees stalls, but you cannot tell why, because you have no transport metrics.
- **100,000 concurrent connections (a busy API or file host):** now the shape of your responses matters. Many small objects stay in slow start and never reach full speed, so the fix is fewer, bigger requests and connection reuse (Day 94, Day 78 for databases). The break: one tenant's huge download fills a shared egress queue and everyone else's latency rises (noisy neighbor, Day 67). Fix: per-flow fair queueing and pacing on the host.
- **10 million+ concurrent flows (Google, YouTube, Netflix, Cloudflare):** the algorithm is a product decision measured in watch time. Deploy BBR on a small percentage first and compare (CONFIRMED: the paper summary says a small percentage of YouTube users were assigned BBR playback). Pacing needs cheap timers at millions of flows, so use kernel fq pacing or NIC offload. The break at this tier: coexistence. Your BBR flows compete with everyone else's CUBIC flows on the same last-mile link, and a measurement study can show you are unfair. Fix: BBRv2 and BBRv3 style loss and ECN reaction. Netflix and Cloudflare publish their own transport tuning; I did not retrieve those posts today, so I make no numeric claim about them.

---

## 8. A worked example, end to end

User Meera in Jaipur, on 4G, opens a YouTube video. Path to the nearest edge server: bottleneck 20 Mbps, RTT 40 ms, carrier buffer around 1 MB (a plausible bloated buffer, INFERENCE).

BDP = 20,000,000 x 0.04 / 8 = **100 KB**.

**With CUBIC:**
1. Slow start from 10 packets (14.6 KB). It doubles each RTT: 14.6, 29, 58, 117 KB. Within four RTTs (160 ms) it has passed the 100 KB BDP.
2. It keeps growing, because no packet has dropped. The 1 MB buffer starts to fill. The window reaches about 1.1 MB in flight.
3. RTT is now 40 ms plus 1 MB x 8 / 20 Mbps = 40 + 400 = **440 ms**. Meera's throughput is still 20 Mbps (the pipe is full), but every tap, every seek, and the ad request that shares this link waits 400 ms behind video data.
4. The buffer overflows, a burst of loss, the window halves, and the cycle repeats.

**With BBR:**
1. Startup finds the delivery rate plateau (about 20 Mbps) within a few RTTs (the startup gain is about 2.89 per RTT, MEMORY) and then drains the extra queue.
2. It settles at about 100 KB in flight. RTT stays near 40 ms. Throughput is still 20 Mbps.
3. Every 8 RTTs it probes at 1.25x for one RTT; if rate rose it adopts a higher estimate, then 0.75x to drain whatever it added.
4. She seeks to 12:30. The request returns in about 40 ms plus server time, not 440 ms.

Same video, same pipe, **about 10x lower delay** at identical throughput. That is the point: the pipe was never the problem, the standing queue was. (The 53 percent median RTT cut in the paper summary is the real-world version of this effect.)

Failure case: Meera's neighbor's CUBIC download shares the same 20 Mbps link and has a shallow buffer. BBRv1 holds its rate through the drops, so it can take more than a fair share. That is exactly why BBRv2 and v3 exist.

---

## 9. Map to Rare.lab's stack

**Already used (without writing a line):**
- **Cloudflare in front of R2.** Cloudflare terminates TCP and QUIC at the edge close to the user, so the lossy last-mile RTT is short (low RTT means faster slow-start and fast recovery). Day 94 covers h3.
- **Content-addressed immutable scene JSON.** Because it caches at the edge forever, the long haul from a far origin is not on the user's critical path. Only the short edge-to-user hop pays congestion control.
- **One shared WebGL context.** Network is separate from GPU, but the same lesson holds: a shared resource should be fed smoothly, not in bursts.

**The ceiling (INFERENCE, measure before acting):**
1. **Slow start is your enemy for small chained fetches.** Manifest, then scene JSON, then textures are sequential round trips, each starting small. Numbers: a 600 KB scene over a 60 ms RTT path with a 10 packet initial window needs about 6 RTTs of ramp up, roughly 360 ms, before the pipe is full. Fix: shrink the first payload (compress with Brotli, quantize numbers), preload hints (`<link rel=preload>`), or inline the manifest with the HTML so the scene fetch starts at once. HTTP/2 or h3 multiplexing over ONE connection beats opening six, because one warmed-up window is shared.
2. **Parallel texture downloads fight each other.** Ten textures at once on one phone link create the same standing queue that Meera's video did. Prioritize: fetch what the first frame needs, defer the rest (HTTP priorities). The embeddable runtime should expose a "priority" per asset.
3. **Do not tune congestion control yourself.** Cloudflare owns that layer. Your levers are: object size, request count, ordering, cacheability, and connection reuse.
4. **Measure the right thing:** track time-to-first-frame split into DNS, TLS, first byte, last byte. If last byte minus first byte is large for a small file, you are paying slow-start or queueing cost, not server cost.

Concrete first step: pick the top scene by traffic, run it through WebPageTest on a throttled 4G profile, and read the waterfall. If the textures begin after the scene JSON finishes, the chain is the ceiling and you inline or preload.

---

## 10. References, summarized in plain language

1. **"BBR: Congestion-Based Congestion Control," Cardwell, Cheng, Gunn, Yeganeh, Jacobson, ACM Queue 2016.** I read it only through a summary on blog.acolyer.org. The paper's claim: instead of waiting for loss, estimate the bottleneck bandwidth and the minimum RTT, and run at their product. Per the summary: 2x to 25x throughput over CUBIC in some tests, the receive-buffer limit matters, and on YouTube a small share of users got BBR with median RTT down 53 percent globally and over 80 percent in developing regions. Lesson: the network gave you a queue and the old algorithm filled it.
2. **Linux kernel commit "tcp_bbr: add BBR congestion control" (Google).** The commit message states BBR needs only sender-side changes, was deployed on Google's backbone, google.com and YouTube, and "significantly increased throughput and reduced latency." Lesson: an operator who owns the server can change the whole internet's experience alone.
3. **IETF draft "BBR Congestion Control" (draft-cardwell-ccwg-bbr, BBRv3 spec).** The standards-track version. It says BBR gives much higher throughput on shallow buffers and random loss, and much lower queueing delay on deep buffers. Lesson: even Google's own algorithm needed three versions to be good citizens.
4. **IETF 121 ICCRG slides, "Promises and Potential of BBRv3," and the 2024 ANRW / PAM measurement decks.** An outside look at BBRv3. Summary from excerpts: it improves coexistence but still shows unfairness toward multiple CUBIC flows. Lesson: "deployed at Google" does not mean "settled."
5. **Academic comparisons (CUBIC vs BBR; a 2025 preprint, alphaxiv 2510.22461).** Excerpt: BBR gets high throughput broadly and is especially strong with other BBR flows and on high bandwidth paths. Lesson: results depend on the path, so always A/B on your own traffic.
6. **Blog on BBR bandwidth estimation and pacing (mohakchugh.is-a.dev).** An explainer of the same model and pacing ideas. I only saw its title and a search excerpt; treat my Section 4 gain cycle numbers as MEMORY of the paper, not from this post.
7. **Not retrieved today, would read next:** Van Jacobson, "Congestion Avoidance and Control" (1988), the original collapse and slow-start paper; Jim Gettys and Kathleen Nichols, "Bufferbloat: Dark Buffers in the Internet" (ACM Queue 2011); the CoDel paper. Listed so you can find them, not summarized as if I read them.
8. **The pasted LeetCode mock interview (YouTube).** A Google engineer designs LeetCode. Functional: list problems, view and code in any language, submit with an instant verdict, a live contest leaderboard. Non-functional: availability over consistency, low latency (about 2 to 5 seconds), isolation of untrusted code, 100k contest users and millions overall, no single point of failure. Entities: problem, user, submission, leaderboard. REST APIs with pagination. DynamoDB because there are no joins. Running user code directly on the API server was rejected (security, CPU hogging, no fault isolation), VMs were heavy, containers per language chosen, with the interviewer noting VMs isolate better and suggesting serverless. A queue sits between API and runners with exponential retry, a Redis cache serves the leaderboard instead of polling the database, test cases are stored language-neutral and serialized per language, and the database gets a primary with replicas. Full teaching: [Day 74](074-leetcode-online-judge-end-to-end.md). Link to today: when the judge returns the verdict to the browser, a small response lives entirely in slow start. And inside the judge, the queue-between-API-and-runners is the same idea as BBR: do not let work pile up in a standing queue, control the inflow to match the drain rate.

---

## 11. One-line takeaway

**A sender cannot see the queue, so it must infer the pipe from timing: aim for the bandwidth-delay product, keep the standing queue near zero, and for small payloads remember you never get out of slow start, so send fewer, smaller, better-ordered requests.**

## Sources

- [Adrian Colyer, "BBR: Congestion-based congestion control" (summary of Cardwell et al., ACM Queue 2016)](https://blog.acolyer.org/2017/03/31/bbr-congestion-based-congestion-control/)
- [Linux kernel commit, "tcp_bbr: add BBR congestion control"](https://android-kvm.googlesource.com/linux/+/0f8782ea14974ce992618b55f0c041ef43ed0b78)
- [IETF draft, BBR Congestion Control (draft-cardwell-ccwg-bbr)](https://datatracker.ietf.org/doc/html/draft-cardwell-ccwg-bbr-00)
- [Geoff Huston, IETF 120 BBR notes (potaroo.net)](https://ispcol.potaroo.net/2024-07/ietf120-bbr.html)
- [IETF 121 ICCRG, Promises and Potential of BBRv3](https://datatracker.ietf.org/meeting/121/materials/slides-121-iccrg-promises-and-potential-of-bbrv3-00)
- [ANRW 2024 slides on BBRv3](https://irtf.org/anrw/2024/slides-120-anrw-17.pdf)
- [2025 preprint, comparative study of congestion control (alphaxiv)](https://www.alphaxiv.org/abs/2510.22461)
- [Through the RTT Lens: CUBIC and BBR](https://doi.org/10.32628/CSEIT2390652)
- [TCP BBR bandwidth estimation and pacing explainer](https://mohakchugh.is-a.dev/blog/tcp-bbr-congestion-control-bandwidth-estimation-pacing)
