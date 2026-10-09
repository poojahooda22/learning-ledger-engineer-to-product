# Day 85: How does one IP address spread 10 million packets a second across thousands of servers, without breaking a single open connection when a server dies? L4 load balancing, Maglev hashing and connection tracking

**Date:** 2026-10-09
**Difficulty:** Advanced (packet-level networking, consistent hashing with fixed tables, stateless vs stateful trade-offs)
**Topic:** Day 55 showed how anycast sends a user to the nearest data center. This lesson is the box right behind it: the load balancer that turns ONE virtual IP into thousands of real servers, at line rate, and survives its own machines and the backends changing mid-conversation.
**Stack relevance:** Rare.lab never runs this box (Cloudflare runs it for us), but the same idea, "pick a stable owner from a fixed table so nobody has to remember", shows up in how we should key our caches and rooms. Section 7 maps it.

---

## 0. The framework (same recipe as Day 74)

1. Functional: clients connect to one public IP and port. The system forwards every packet of a connection to the same healthy backend server, for the whole life of that connection.
2. Non-functional: handle millions of packets per second per machine, add or remove a backend with almost no broken connections, survive a load balancer machine dying, no single box on the path.
3. Entities: VIP (the one public address), Backend (a real server), Flow (one TCP connection, identified by 5 numbers), Lookup table, Connection table.
4. API: none for users. For operators: `add_backend`, `drain_backend`, `health_check`.
5. Naive design: one nginx or one hardware box in front, round robin.
6. Deep dives: ECMP (the router spreads load across load balancers), the 5-tuple hash, the Maglev lookup table, the connection tracking table, direct server return, and what happens when the backend set changes.

If you watched the LeetCode mock interview in your prompt: the interviewer asks "no single point of failure, consistent hashing so other nodes take the traffic". This lesson is the actual machine that does that, one layer below the API servers in that diagram.

---

## 1. The company and the breaking number

**Google (Maglev, running since 2008) and Meta, GitHub and Cloudflare, who each built their own copy.** Google's NSDI 2016 paper says a single Maglev machine can saturate a **10 Gbps** link **with small packets** (confirmed from the paper abstract).

Why "small packets" is the number that matters (my arithmetic, not a quote):

- The smallest Ethernet frame on the wire takes about 84 bytes including preamble and gap. 10 Gbps / (84 x 8 bits) is about **14.9 million packets per second**.
- One CPU core running at 3 GHz has about **200 cycles per packet** if it must handle 15 million packets per second. That is the time budget for: read the packet, find which service it belongs to, pick a backend, rewrite it, send it.
- A normal Linux network stack burns thousands of cycles per packet (interrupts, memory copies, system calls, lock handoffs). It cannot hit 15 million per second on one machine.

Now scale it up. A big service sees a **DDoS of tens of millions of packets per second** or a flash crowd of 1 million new connections per second. Every one of those packets must go to the right place, and every packet of an old connection must go to the **same** place as before.

The breaking number is **about 200 CPU cycles per packet**. A design that spends more than that falls behind, queues fill, packets drop, and TCP retries make it worse.

Analogy: a post office sorting room where letters arrive faster than one clerk can read the address. You cannot make the clerk read faster. You must make the sorting rule so simple that it takes one glance, and you must make sure the same family's letters always land in the same pigeonhole even if a clerk goes home sick.

---

## 2. Why the naive design dies

**Naive: one load balancer box (nginx, HAProxy or a hardware appliance), round robin over backends.**

- **It is a single point of failure and a single point of capacity.** One box, one NIC. When it dies, the whole service is down. When traffic doubles, you buy a bigger box. Hardware load balancers cost six figures per pair and still top out.
- **Round robin breaks connections.** TCP is a conversation. Packet 1 of a connection went to server A. Packet 2 must also go to server A, or server B will say "I never heard of you" and reset it. Round robin sends packet 2 to server B.
- **Keeping a table of every connection does not scale and does not survive failover.** The fix for the point above is "remember which server each connection went to". At 1 million new connections per second, that table is huge, it must be updated on every packet, and when the box dies the replacement box has an empty table. Every user's download or video call breaks at once.
- **Plain modulo hashing is not safe.** Use `server = hash(5-tuple) % N` and you need no table. But change N from 100 to 101 (add one server) and about **99% of connections move to a different server** and break. This is the same problem Day 10 solved for caches. For a load balancer the damage is worse: you are breaking live user connections, not just cache hits.
- **Proxying everything is expensive.** An L7 proxy (nginx) terminates the TCP connection, reads the request, opens a second connection to the backend. That costs memory and CPU per connection and puts every response byte through the box again.

Analogy: a bank with one teller window. If the teller steps out, the bank stops. If you add a second window and send customers randomly, a customer who started a form at window 1 gets sent to window 2 halfway through.

---

## 3. The architecture, drawn top to bottom

```
Clients on the internet
   |   (all of them connect to ONE address: the VIP, e.g. 203.0.113.10:443)
   v
Edge routers  --- ECMP: split packets across N load balancer machines by hashing the 5-tuple
   |
   v
L4 load balancer fleet (Maglev, Katran, GLB Director, Unimog: dozens of identical machines)
   |   per packet:  5-tuple  ->  connection table hit?  ->  yes: use that backend
   |                                               ->  no:  Maglev lookup table -> backend
   |   then wrap the packet (GRE / IP-in-IP) and send it to the chosen backend
   v
Backend servers (thousands). They unwrap the packet and answer the CLIENT DIRECTLY
   |   (direct server return: the reply does NOT go back through the load balancer)
   v
Client
```

Layer by layer:

| Layer | Single job | Analogy |
|---|---|---|
| Anycast and BGP (Day 55) | Announce the VIP from many places, pull users to the nearest one | Many identical doorbells, you ring the nearest |
| **Edge router with ECMP** | Equal Cost Multi Path: spread incoming packets across all load balancer machines by hashing the 5-tuple. | A traffic cop who sends every car with a plate ending in 0-3 left, 4-7 right |
| **L4 load balancer machine** | Look only at IP and port numbers. Pick a backend. Forward the packet. Never read the request. | A mail sorter who reads only the zip code, never opens the envelope |
| **Connection table (per machine)** | Remember "this flow went to backend 17" so a later table change cannot move it | The sorter's sticky note: "the Sharma family goes to pigeonhole 17" |
| **Maglev lookup table** | A fixed array (about 65,537 or 655,373 slots) where each slot holds a backend. `slot = hash(5-tuple) % M`. | A phone book of 65,537 lines pre-assigned to branches |
| **Backend** | Do the real work, answer the client directly | The branch that actually serves you |
| **Health checker** | Remove dead backends from the table | The manager who crosses a closed branch off the book |

Key vocabulary:

- **L4 vs L7.** Layer 4 is the transport layer: IP addresses, TCP and UDP ports. Layer 7 is the application layer: HTTP paths, headers, cookies. L4 is fast and dumb. L7 is slower and smart. Real stacks use both: L4 in front, L7 (Envoy, nginx) behind.
- **5-tuple.** Source IP, source port, destination IP, destination port, protocol. These five numbers identify one connection. Every packet of the same connection has the same 5-tuple, so hashing it gives the same answer every time.
- **ECMP.** The router has several equal next hops and picks one by hashing the packet. It keeps no per-connection state.
- **Direct server return (DSR).** Requests are small (a GET line). Responses are large (a video). Only the request goes through the load balancer. The backend replies straight to the client, so the load balancer sees maybe a tenth of the bytes. Confirmed as Maglev's design in the paper.
- **Encapsulation (GRE).** To forward the packet to a backend on another subnet without changing the client's source address, the balancer wraps the whole packet inside a new packet addressed to the backend. The backend unwraps it. Confirmed: Maglev uses GRE.
- **Kernel bypass.** The network card writes packets straight into memory the Maglev process owns. No kernel stack, no interrupt per packet. This is how it fits in the cycle budget. Confirmed as Maglev's approach. Meta's Katran and Cloudflare's Unimog reach the same goal differently: they run an eBPF program at the XDP hook, the earliest point in the Linux kernel, before the normal stack touches the packet.

---

## 4. The transferable mechanisms

### 4.1 Hash instead of remember (stateless routing)

The best state is no state. `backend = table[hash(5-tuple) % M]` needs no per-connection memory. Any load balancer machine computes the same answer, so a second machine can take over a connection the first one was handling. ECMP can even change which machine gets a packet and the answer stays the same.

Real example: a client at 49.36.1.7:51512 connects to the VIP. Machine LB-3 hashes the 5-tuple, gets slot 40,211, and the table says backend 17. Next packet arrives at LB-9 (the router rebalanced). LB-9 hashes the same 5-tuple, same slot, same backend 17. No conversation between LB-3 and LB-9 was needed.

### 4.2 The Maglev lookup table: consistent hashing with a fixed-size array

The problem with `hash % N` is that N changes. Maglev hashes into a **fixed** table size M (a prime number, so the arithmetic spreads evenly) and instead changes **what is written in the slots**.

How the table is filled (algorithm from section 3.4 of the paper):

1. Each backend gets a **permutation** of all M slots, generated from two hashes of its name: an `offset` and a `skip`. The k-th slot in its preference list is `(offset + k x skip) mod M`. Because M is prime, this visits every slot exactly once.
2. Backends take turns, round robin. On its turn, a backend walks down its own list to its first slot that is still empty and claims it.
3. Stop when the table is full.

Every backend gets within one slot of an equal share. The turn-taking is what guarantees evenness. The permutations are what guarantee **stability**: when a backend leaves, most slots keep their old owner because each backend's preferences are unchanged.

Walked end to end with the paper's tiny example (M = 7, three backends; I ran this code and it matches the table in the paper's slides):

- B0 offset 3, skip 4 -> preference list 3, 0, 4, 1, 5, 2, 6
- B1 offset 0, skip 2 -> list 0, 2, 4, 6, 1, 3, 5
- B2 offset 3, skip 1 -> list 3, 4, 5, 6, 0, 1, 2
- Round 1: B0 takes 3. B1 takes 0. B2 wants 3 (taken), so takes 4.
- Round 2: B0 wants 0 (taken), 4 (taken), takes 1. B1 takes 2. B2 takes 5.
- Round 3: B0 wants 5 (taken), 2 (taken), takes 6. Table full.
- Result: `[B1, B0, B1, B0, B2, B2, B0]`. Shares: B0 gets 3, B1 gets 2, B2 gets 2. Perfectly even for 7 slots.
- Remove B1 and rebuild: `[B0, B0, B0, B0, B2, B2, B2]`. B1's slots 0 and 2 had to move, which is unavoidable. One slot that was NOT B1's also moved (slot 6, B0 to B2). The other four kept their owner. In a 7-slot toy that is 3 of 7 changed. At real sizes the extra churn is tiny, see next.

At real sizes it is tiny, see next.

At real size (**my own simulation, 100 backends, one removed, SHA-256 for the hashes, not numbers from the paper**):

| Table size M | Slot count per backend (ideal 655 or 6,553) | Slots owned by the removed backend | Slots of OTHER backends that also changed |
|---|---|---|---|
| 65,537 | 655 to 656 (flat) | 655 | 351 (0.54% of table) |
| 655,373 | 6,553 to 6,554 (flat) | 6,554 | 902 (0.14% of table) |

So: perfectly flat load, and removing one of 100 backends moves its own 1% share plus a small extra. The paper discusses exactly this trade (small table is faster to build and fits in cache, large table is more even and more stable). I recall the paper using M = 65,537 as the small size and 655,373 as the large size, from memory; the search tool could not return that page, so treat the exact constants as unconfirmed.

Honest comparison with rendezvous hashing, which GitHub uses: rendezvous (highest random weight) gives the same minimal disruption but costs O(N) per lookup. GitHub's GLB Director instead precomputes a fixed forwarding table using rendezvous ordering so each lookup is O(1). Same idea, different fill rule. Cloudflare's Unimog also uses a fixed-size forwarding table (control plane builds it, data plane indexes it).

### 4.3 Connection tracking: a table of "who I already promised"

Even a 0.5% disruption is too much if each of those flows is a live video call. So Maglev adds a second line of defence:

- Each packet-processing thread keeps its **own** small connection table (no locks, no sharing). Confirmed from the paper excerpts.
- On the first packet of a flow, the thread uses the lookup table, then records `5-tuple -> backend` in its connection table.
- On later packets, the connection table is checked first. If the entry exists and the backend is still healthy, use it. The lookup table is only the fallback.
- The table is fixed size, so old entries are evicted. Katran (Meta) does the same with an LRU table.

Why you need BOTH:

- Connection table only: breaks when ECMP sends the next packet to a different load balancer machine that has no entry. The lookup table rescues it, because every machine would hash to the same backend.
- Lookup table only: breaks whenever the backend set changes between two packets of one flow.
- Together: the lookup table gives a good guess that every machine agrees on. The connection table pins the exception once it exists. That is "stateless by default, stateful as an optimisation".

### 4.4 Daisy chaining (Unimog): fix the broken flows after the fact

Cloudflare's Unimog (2020, by David Wragg) takes a different bet: do not keep a connection table at all. Instead, when the forwarding table changes, the **new** owner of a bucket forwards packets it does not recognise to the **previous** owner. If the old owner has the connection, it handles it. Per the search summary, this "daisy chaining" keeps established TCP connections alive during table updates. I could not open the original post this session, so treat the detail as second hand.

### 4.5 Shard the balancer itself: the fleet has no leader

All load balancer machines run the same code and have the same table. None is a boss. ECMP spreads packets. Lose one machine and the router rehashes the rest. Add one and capacity goes up linearly. This is the same "no single point of failure" the mock interview asked for, achieved by making the box stateless enough to be one of many.

### 4.6 Separate the control plane from the data plane

- Data plane: the fast path. Hash, lookup, wrap, send. No decisions, no health logic, no network calls. Fits in 200 cycles.
- Control plane: slow, smart, off to the side. Health checks, operator changes, rebuilding the table, pushing it to every machine.
- The data plane only ever reads a finished table. The control plane swaps in a new table atomically. This split is the reason it is fast and also the reason it is safe to change.

---

## 5. The trade-offs

**Consistency vs availability, per data type:**

| Data | Choice | Why |
|---|---|---|
| Lookup table contents | **Eventually consistent across balancer machines**, not strictly | During a push, machine A may have the new table and machine B the old. Briefly they disagree on a few slots. The connection table and daisy chaining absorb that. Strict agreement (a lock, consensus) would put a slow thing on the hot path. |
| Health state | **Availability first**: remove a flapping backend quickly, add it back slowly | A wrongly removed backend only costs capacity. A wrongly kept dead backend black-holes traffic. |
| Connection table | **Best effort cache, not truth** | If an entry is lost, the lookup table recomputes the same answer in most cases. |

**Cost vs latency:**

- L4 is far cheaper than L7. One Maglev class machine handles the packet rate of a whole fleet of nginx proxies, because it does not terminate TCP and does not touch response bytes (DSR).
- You pay for it in features. L4 cannot route by URL, cannot retry a failed request, cannot cache. You put an L7 tier behind it for that.
- Bigger lookup table = flatter load and less disruption, but more memory and slower rebuild. Small table = faster rebuild, but 1 to 2% unevenness (the paper discusses this trade).
- Kernel bypass or XDP costs engineering effort: you give up the normal Linux network tools (tcpdump, iptables) on that path.

**Which choice the system makes:** availability and throughput over precision. A few percent of unevenness is accepted. A tiny fraction of flows breaking on a backend change is accepted. Both are far cheaper than a single point of failure or a stateful bottleneck.

---

## 6. The systems-thinking lens

**The feedback loop that actually kills load balancers: the rehash cascade (a self-inflicted thundering herd).**

1. One backend gets slow or flaps (health check fails, passes, fails).
2. Each flap changes the table. With plain modulo hashing, almost every connection moves.
3. Every moved connection is a new connection to a new backend: a fresh TCP handshake, a cold cache, a TLS handshake. That extra work makes the other backends slower.
4. Slower backends miss their health checks. More flaps. Go to step 2.

This is a metastable failure (Day 84): once the loop is running, removing the original trigger does not stop it.

How the senior fix **breaks** the loop instead of adding capacity:

- **Make change local.** Maglev hashing means a backend flap moves roughly its own share (about 1% of flows for 1 in 100), not everything. The loop's gain is below 1, so it dies out.
- **Pin what already exists.** The connection table means a table change cannot move a flow already in progress.
- **Damp the sensor.** Health checks require N consecutive failures to remove and M consecutive successes to add back. This is the same stabilization window idea as Day 84's autoscaler.
- **Drain, don't kill.** For planned changes, stop sending NEW flows to a backend but let existing flows finish. Zero broken connections by design.
- **Cap the blast radius.** Day 34 (cells and shuffle sharding): a bad backend should only be reachable by a slice of clients, not everyone.

---

## 7. Mapping to Rare.lab

Rare.lab runs on Supabase Postgres with RLS, Cloudflare R2 for content addressed scene JSON plus a manifest, Cloudflare Workers, and an embeddable runtime with one shared WebGL context.

**What we already get for free:** Cloudflare operates an L4 layer built on exactly these ideas (Unimog, plus anycast) under every Worker and R2 request. We never configure it. That is why a 100,000 person launch day does not need us to run a load balancer.

**Where the idea still helps us:**

| Idea from today | Where it applies to Rare.lab | Action |
|---|---|---|
| Hash into a fixed table instead of `% N` | Any time we shard by key: per-scene realtime rooms, a render worker pool for AI generation, a cache keyed by scene hash | Use a stable owner scheme (Durable Object per scene id, or Maglev or rendezvous hashing for a worker pool) so adding a worker does not reshuffle every scene |
| Content addressed names are already a perfect hash | Our scene JSON in R2 is named by content hash | Immutable and cache-key stable. A hash of the content never moves. This is "hash instead of remember" already in use |
| Stateless by default, stateful as a pin | Editor sessions | Do not keep session state in a Worker. Keep a small pin (Durable Object or a KV entry) only for the multiplayer room, like the connection table |
| Drain, don't kill | Deploying a new embeddable runtime version | Serve the old manifest version to existing embeds until they re-check; let new embeds take the new one. Never swap under a running page |
| Damped health | AI generation backends | Require several failures before removing a provider, several successes before returning it |

**Next ceiling:** the database. A Worker fleet is free to scale. Postgres has a hard connection and write ceiling (Day 78). Put the same discipline there: pool, cap, shed. And the AI generation path needs a bounded queue (Day 84).

**One-line lesson for Rare.lab:** route by a hash of a stable key into a fixed table so that adding capacity moves almost nothing, pin only what is already in flight, and never let a health flap rehash the world.

---

## 8. Sources and what is actually in them

**Honest note on access:** direct page fetches were blocked by the network proxy this session (usenix.org, research.google, github.blog, blog.cloudflare.com, the-paper-trail.org, blog.acolyer.org all failed to resolve). Everything cited below was read through search result excerpts. The 200 cycle and 14.9 million packets per second arithmetic, the toy table walkthrough and the 100 backend simulation are mine and labelled as such. The table sizes 65,537 and 655,373 are from my memory of the paper and are unconfirmed here.

- [Maglev: A Fast and Reliable Software Network Load Balancer, NSDI 2016 (Eisenbud et al.)](https://research.google/pubs/maglev-a-fast-and-reliable-software-network-load-balancer/) and the [PDF](https://www.usenix.org/sites/default/files/nsdi16-paper-eisenbud.pdf). Plain language: Google explains the box that has fronted its services since 2008. Routers spread packets to Maglev machines by ECMP. Each machine looks only at the 5-tuple, uses a connection table first and a consistent hashing table second, wraps the packet and sends it to a backend. One machine can fill a 10 Gbps link even with tiny packets. Section 3.4 is the table filling algorithm.
- [Maglev conference slides (USENIX)](https://www.usenix.org/sites/default/files/conference/protected-files/nsdi16_slides_eisenbud.pdf). Plain language: the same story as the talk, with the small B0, B1, B2 table example. It also notes that when ECMP changes which machine a packet lands on, a machine without the entry relies on consistent hashing to pick the same backend.
- [Adrian Colyer, the morning paper: Maglev](https://blog.acolyer.org/2016/03/21/maglev-a-fast-and-reliable-software-network-load-balancer/) and [Henry Robinson, Network Load Balancing with Maglev](https://www.the-paper-trail.org/post/2020-06-23-maglev/). Plain language: two readable walkthroughs of the paper. Start here if the PDF is dense.
- [Meta Katran on GitHub](https://github.com/facebookincubator/katran) and [Meta engineering post](https://code.facebook.com/posts/1906146702752923). Plain language: Meta replaced the old kernel IPVS balancer with an eBPF program at the XDP hook, uses a modified Maglev hash, and keeps a fixed LRU connection table. Good proof the idea works without a custom user space stack.
- [GitHub: Introducing the GitHub Load Balancer](https://github.blog/engineering/introducing-glb/) and [GLB Director open source release](https://github.blog/engineering/infrastructure/glb-director-open-source-load-balancer/). Plain language: GitHub's variant. A fixed forwarding table filled using rendezvous hashing ordering, shared to all directors, in front of HAProxy or nginx. Shows the same pattern with a different hash.
- [Cloudflare: Unimog, Cloudflare's edge load balancer (David Wragg, 2020)](https://blog.cloudflare.com/unimog-cloudflares-edge-load-balancer/). Plain language: Cloudflare's own L4 layer for every data center. eBPF and XDP data plane, a control plane that builds forwarding tables, and daisy chaining to keep live TCP connections alive when the table changes.
- [Katran overview, "How Meta turned the Linux Kernel into a planet-scale Load Balancer"](https://softwarefrontier.substack.com/p/how-meta-turned-the-linux-kernel). Plain language: a long-form walkthrough of the same design for engineers who prefer prose.

**Confirmed vs inference.** Confirmed from excerpts: 10 Gbps with small packets; in service since 2008; ECMP into Maglev machines; per-thread connection table plus consistent hashing fallback; Katran uses modified Maglev hashing and an LRU table; GLB uses a fixed rendezvous-ordered table; Unimog uses eBPF/XDP, forwarding tables and daisy chaining. Inference, labelled: cycle budget arithmetic; the exact table constants; the simulated disruption percentages; the rehash cascade as a named loop.

**About the video in your prompt:** the LeetCode mock interview is taught in [Day 74](074-leetcode-online-judge-end-to-end.md). Its "distributed, consistent hashing, no single point of failure" line is this lesson one layer down.

**Related lessons:** Day 10 (consistent hashing), Day 13 (backpressure), Day 34 (cells), Day 55 (anycast), Day 76 (tail latency, power of two choices), Day 78 (connection pooling), Day 84 (autoscaling and metastable failure).
