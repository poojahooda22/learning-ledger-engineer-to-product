# Day 98: One million people keep a socket open to you all day, doing nothing. How connection gateways (WhatsApp, Discord, Slack Flannel, Supabase Realtime) hold them, and survive the minute they all reconnect at once

**Date:** 2026-10-25
**Difficulty:** Advanced (the tier under presence Day 63, push fanout Day 53, messaging Day 51, load shedding Day 13)
**Topic:** Every "real-time" lesson so far assumed a pipe to the user already exists. Today is that pipe: a long-lived connection (WebSocket) held open for hours. It breaks the normal web rules, because the cost is not requests per second. It is connections held, and the danger is everyone returning at the same moment.
**Stack relevance:** Rare.lab will want live collaboration, live preview and "your shader finished compiling" pushes. Section 9.
**A note on the pasted transcript:** the LeetCode mock interview is already taught in [Day 74](074-leetcode-online-judge-end-to-end.md) (sandbox Day 50, leaderboard Day 54). I did not repeat it. Section 0 reuses its interview recipe, and Section 10 summarizes it again in plain language, as you asked. The one live link to today: its contest leaderboard was "client re-fetches periodically". The upgrade to "server pushes" is exactly the gateway below.
**Honesty labels:** CONFIRMED = read in a search excerpt today. Important limit: direct page fetches failed (network blocked), so I never read a full primary article. Everything below comes from search excerpts and secondary summaries. INFERENCE = my reasoning, labeled. MEMORY = standard knowledge I did not re-read today. Arithmetic is shown so you can check it.

---

## 0. The interview recipe applied to a connection gateway

The LeetCode candidate went: functional requirements, non-functional requirements, entities, API, naive design, deep dives. Same recipe.

1. **Functional.** Client opens one connection and keeps it. Server can push to any connected user at any time. Client can send events. After a drop, the client resumes without missing messages.
2. **Non-functional.** 1 million or more concurrent connections. Push latency under about 200 ms. No single point of failure. A server restart must not cause an outage. Cheap per idle connection.
3. **Entities.** Connection, session (the logical user state that outlives one connection), topic or channel, gateway node, routing table (who is connected where).
4. **API.** `CONNECT(token, last_seen_id)`, `SUBSCRIBE(topic)`, `PUBLISH(topic, msg)`, server frame `EVENT(id, topic, payload)`, `ACK(id)`.
5. **Naive design.** One app server, one WebSocket per user, an in-memory list of sockets, loop over the list to broadcast.
6. **Deep dives.** Memory per idle socket, routing a message to the right box, fanout cost, the reconnect storm, deploys.

---

## 1. Named companies and the breaking numbers

- **WhatsApp (CONFIRMED via search excerpt of the official Erlang Factory 2012 slides, Rick Reed, "Scaling to Millions of Simultaneous Connections").** The slides list roughly 200,000 connections per server at the start and a goal of 1 million per server, reached by tuning the Erlang VM (BEAM) and the FreeBSD kernel. A secondary write-up reports a test of 2.5 million-plus on one box. The famous "2 million per server" is a widely repeated approximation I could not tie to a WhatsApp statement. Another summary says the team later stopped maximizing connections per box, because headroom for load matters more (secondary).
- **Discord (CONFIRMED via excerpts of Discord's 2017 post "How Discord Scaled Elixir to 5,000,000 Concurrent Users" and a 2020 Elixir case study).** Each connected user gets a session process. Each server (called a guild) has one guild process that fans every event out to its sessions. Big servers like /r/Overwatch, with up to 30,000 concurrent users, made guild processes fall behind their message queues at peak, and engineers sometimes turned off message-generating features by hand. Sending messages between Erlang processes was "not as cheap as expected". They open-sourced Manifold (batch message passing between nodes). The 2020 case study reports 12 million-plus concurrent users and 26 million-plus WebSocket events per second to clients.
- **Slack (CONFIRMED via search excerpts).** Slack's own Flannel post describes the failure: when an entire office loses its network and reconnects at once, the combined weight of requests means "slower connections, more reconnects, more failures, and more retries". Flannel (an application-level edge cache) ran at the edge from January 2017, serving 4 million simultaneous connections at peak and 600,000 client queries per second (StackShare summary). Slack's status page for 27 June 2018 says a bug in an offline batch process caused network spikes and left all customers disconnected and unable to reconnect, fixed by restricting new connections and adding capacity. A secondary article claims a Flannel bug dropped 1.6 million WebSockets in two minutes and the reconnects overloaded surviving hosts. I could not verify that last one.
- **Supabase Realtime (CONFIRMED via the official architecture page excerpt).** A globally distributed Elixir cluster; clients connect to any node over WebSockets; channels use Phoenix Channels on Phoenix.PubSub. The official benchmark page excerpt showed a 32,000-user test. A 250,000-user, 800,000 msg/s figure appeared only on a mirror domain, so treat it as unverified. Do not confuse it with Supavisor's 250,000 connections to a Postgres pooler (Day 78).

**The breaking number for the naive design (INFERENCE arithmetic, labeled):** 1,000,000 idle sockets at about 10 KB of kernel plus app memory each is 10 GB of RAM doing nothing. The real killer: if all 1,000,000 reconnect in 60 seconds after a blip, that is 16,700 new TLS handshakes and auth checks per second, against a steady state of maybe a few hundred. The login path was sized for the steady state. That is a 30x to 100x spike aimed at the most expensive endpoint you own.

---

## 2. Why the naive design dies

**Failure 1: One process per connection and one thread per process.**
- Classic thread-per-connection servers cost a few MB of stack each. 10,000 threads is already trouble (the old C10K problem, MEMORY). One million is impossible.
- Analogy: hiring one receptionist per visitor who is just sitting in the lobby. You need one receptionist who watches a thousand people sit.

**Failure 2: The broadcast loop.**
- One chat room of 30,000 members. Each message loops over 30,000 sockets. At 50 messages per second that is 1.5 million socket writes per second from one loop on one process. Discord's guild process hit this wall (CONFIRMED above). A slow client blocks the loop for everyone behind it.
- Analogy: one teacher shouting to a stadium by walking to every seat.

**Failure 3: Routing, "which box is Ravi connected to?"**
- With one server it is a list. With 200 servers, a message for Ravi arrives at box 17 but Ravi is on box 142. The naive fix, broadcast to all 200, makes cost grow with servers times messages.
- Analogy: phone numbers without an exchange.

**Failure 4: The deploy and the blip.**
- Restarting a box drops its 50,000 sockets. All 50,000 clients retry immediately, find the same neighbors, and overload them. Their sockets drop too. This is the retry spiral from Day 13, now carried by connection state instead of requests.
- Analogy: a theatre fire alarm where everyone runs to the same two exits.

---

## 3. The architecture, top to bottom

```
Clients (phone, browser, desktop app)
   |   keep ONE WebSocket open; reconnect with backoff + jitter; send last_seen_id
   v
Edge / anycast + TLS termination  (Cloudflare, cloud LB)
   |   job: absorb TLS handshakes close to the user; cap new-connection rate (admission control)
   |   Analogy: the lobby doorman who lets in 500 an hour, not 50,000 a second.
   v
L4 load balancer  (Day 85, consistent hashing / least connections)
   |   job: spread connections evenly; keep a socket pinned to its box for life
   v
GATEWAY TIER  (stateful about sockets, stateless about business)
   |   job: hold sockets, do auth once, heartbeat, parse frames, nothing else.
   |   one event loop / lightweight process per connection (epoll, BEAM, Go, Rust)
   |   ~100k to 1M sockets per box, sized with 50 percent headroom
   |   Analogy: switchboard operators. They connect calls. They do not answer them.
   v
SESSION LAYER  (per-user logical state: subscriptions, last_seen_id, small replay buffer)
   |   job: survive a socket dying. The session outlives the connection.
   |   Analogy: the hotel reservation survives you leaving the lobby.
   v
ROUTING / PUBSUB  (topic -> set of gateway nodes; user -> gateway node)
   |   job: deliver to ONLY the boxes that have a subscriber. Redis pubsub, NATS, Phoenix.PubSub, Kafka.
   |   Analogy: the exchange that knows which switchboard holds your line.
   v
Backend services  (chat, presence, compile jobs, leaderboards) publish events here
   |   job: they never touch sockets. They publish to a topic and forget.
   v
Durable store  (Postgres / log)
       job: the truth. A socket is a delivery hint, never the record. Replay from here after a gap.
```

Annotations:
- **Gateways are the one stateful tier.** Everything behind them is a normal stateless app. Keep the gateway dumb so it can be replaced at will.
- **Fanout is two-level.** A message goes to the gateway nodes that have subscribers (one hop per node, not per user), then each node writes to its local sockets. This is the Discord relay idea (the exact relay details are SECONDARY in my search, so the two-level form is partly INFERENCE).
- **The log closes the gap.** A client that reconnects asks "give me everything after id 8841". That makes a dropped socket a non-event.

---

## 4. The transferable mechanisms (6)

1. **Cheap idle connections (event loop, not thread).** Use an event loop (epoll, kqueue) or a runtime with tiny green processes (Erlang BEAM, used by WhatsApp and Discord). Memory per idle socket drops from megabytes to kilobytes. Example: WhatsApp's talk was about taking ~200k sockets per box to a million by tuning the VM and kernel (CONFIRMED title and goal).
2. **Session outlives socket plus resume token.** Give every client a session id and a monotonic `last_seen_id`. On reconnect, replay from a short buffer or the durable log. Example: Ravi's train enters a tunnel for 40 seconds, reconnects, sends `last_seen_id=8841`, and gets messages 8842 to 8850 instead of a full reload. This removes the biggest cost of a blip: re-downloading all state (Slack's pre-Flannel pain: huge bootstrap payloads, CONFIRMED in concept).
3. **Backoff with full jitter.** On disconnect, wait `random(0, min(cap, base * 2^attempt))`. Example: base 1 s, cap 60 s. One million clients then spread over about a minute, not the same second. This is the single cheapest protection against the reconnect storm. MEMORY: the AWS Architecture Blog "Exponential Backoff and Jitter" shows full jitter finishes sooner and with fewer calls than plain backoff.
4. **Admission control at the front (shed new, protect old).** Cap new connections per second per node. Excess clients get "try again in N seconds" with a server-chosen delay. Example: Slack's 2018 fix was literally restricting new connections while adding capacity (CONFIRMED on the status page). Existing healthy sockets stay up. Day 13's rule: protect the work in flight.
5. **Two-level fanout with batching.** Publish once to a topic; the routing layer sends one copy per gateway node; the node writes to local sockets. For huge rooms add a relay tier and batch sends (Discord's Manifold, CONFIRMED to exist and to batch cross-node messages). Add **passive sessions**: users not looking at a big server do not get every message, only when they open it (SECONDARY summary, treat as direction not detail).
6. **Per-connection send buffer with a drop policy.** Every socket gets a bounded outbound queue. A slow phone must not stall a room. When its queue is full, drop it, or send a "resync" marker and let it replay from the log. Example: a phone on 2G in a 30,000-person room gets disconnected after 256 queued frames and resumes later. This is backpressure (Day 13) applied per socket.

---

## 5. The trade-offs accepted

- **Consistency vs availability, per data type.**
  - Chat or event delivery: at-least-once with ids, so duplicates are possible and the client dedups by id. Availability wins, because a late message beats a dead app. Ordering only per topic (Day 31, Day 51).
  - Presence ("online now"): eventually consistent and lossy on purpose (Day 63). A 20-second-stale green dot is fine.
  - Anything that moves money or permissions: never trust the socket. Re-check on the durable store.
- **Cost vs latency.** Idle sockets cost RAM, heartbeats and NAT keepalive traffic. Polling every 5 seconds costs more requests than it saves sockets once you pass a few thousand users. Push gives sub-second delivery; polling is cheaper to operate under about 1,000 users (INFERENCE, the crossover is workload-dependent, measure it).
- **Density vs headroom.** Packing 2.5 million sockets on one box is a benchmark result; running at 40 to 50 percent of that is operations. When a box dies, its neighbors absorb its clients. WhatsApp's own shift away from maximum connections per box is the real-world version (SECONDARY).
- **Sticky vs stateless.** A socket pins a user to a box for hours, so deploys and autoscaling must drain gracefully (tell clients "reconnect to someone else in 30 to 90 seconds, jittered"). Autoscaling cannot rebalance existing sockets, only new ones (INFERENCE, standard consequence).

---

## 6. The systems-thinking lens

**The loop: the reconnect storm (a metastable failure).**
1. A blip drops N sockets (deploy, network flap, a bad batch job, as Slack's 2018 status page says).
2. All N clients retry at once. Each retry costs a TLS handshake plus an auth call plus a bootstrap payload.
3. The login path and surviving gateways slow down. Handshakes time out.
4. Clients treat the timeout as a failure and retry again. Load goes up while capacity effectively goes down.
5. Even after the original cause is gone, the system stays stuck because the retry load alone exceeds capacity. That is the definition of metastable.

**Adding servers does not fix this.** Servers you add also start cold (empty caches, JIT warm-up), and the storm is 30x to 100x the steady state. You break the loop instead:
- **Jitter** spreads the arrival in time.
- **Admission control** caps the arrival rate and tells the client when to come back.
- **Resume tokens plus replay** shrink the cost per reconnect (no full bootstrap).
- **Cache the expensive part at the edge** (Slack's Flannel idea: serve the big "who is in my workspace" state from an edge cache instead of the central database, CONFIRMED in concept).
- **Drain on deploy**: roll one box at a time and give each client a randomized reconnect delay.
- A second, smaller loop: the **hot room**. One 30,000-member room's guild process falls behind, queues grow, memory grows, latency grows (Discord, CONFIRMED). Break it with relays, batching and passive sessions, not a bigger box.

---

## 7. Scale story at three tiers

- **1,000 concurrent users.** One Node or Go process, an in-memory `Map<userId, socket>`, broadcast by loop. This is correct. Add heartbeats (ping every 25 s so proxies do not kill idle sockets) and client reconnect with jitter now, since they cost nothing. Example: a small collaborative editor.
- **100,000 concurrent users.** One box is no longer safe (a restart drops everyone) and one process cannot loop over big rooms. Add: 3 to 10 gateway nodes behind an L4 balancer, a pub/sub (Redis or NATS) so a message published on node A reaches subscribers on node B, resume ids and a replay buffer, a per-socket send queue cap, and a managed edge for TLS. What broke from the last tier: single box, in-memory room list, broadcast loop.
- **10 million plus (Discord 12M+, Slack 4M at edge, WhatsApp hundreds of millions of users).** Walls in order: (1) memory and file descriptors per box, so a VM or runtime built for cheap concurrency and tuned kernel limits; (2) routing, so a user-to-node map or pub/sub with topic-to-node sets rather than broadcast to all; (3) giant rooms, so relays, batching, passive sessions; (4) the reconnect storm, so jitter, admission control, resume, edge caches; (5) deploys, so graceful drain and cell-based rollout (Day 34); (6) cost, since idle connections and keepalives are a fixed bill. Real numbers: Discord 26M-plus events per second to clients (CONFIRMED via case study excerpt); Slack 600,000 client queries per second at the edge cache (CONFIRMED via StackShare summary).

---

## 8. A worked example, end to end

Ravi, a designer in Pune, has a live shared canvas open at 8:10 pm. 100,000 other users are online across 12 gateway nodes.

**Connect:**
1. His app opens `wss://rt.example.com`. The edge terminates TLS (one handshake, 2 round trips on TLS 1.3 after the TCP one). The balancer picks gateway 7 (least connections).
2. Gateway 7 checks his token once (a signed JWT, no database call), creates session `s_91f` with `last_seen_id=0`, and subscribes him to `canvas:abc` in the pub/sub. The routing table now says `canvas:abc -> {gw3, gw7, gw11}`.

**A message:**
3. A teammate on gw3 moves a layer. The app service publishes `canvas:abc {id: 8842, ...}` and also appends it to the log (the truth).
4. Pub/sub sends ONE copy each to gw3, gw7, gw11. Each writes to its local sockets. Ravi sees the move in about 80 ms (target; INFERENCE).

**The blip:**
5. Ravi's train enters a tunnel. The socket dies. Heartbeat misses 2 pings (50 s), gw7 closes it. Session `s_91f` is kept for 2 minutes.
6. Ravi's client waits `random(0, 1 s * 2^attempt)`, retries, and reconnects 40 s later via gw9 (a different box). It sends `session=s_91f, last_seen_id=8841`.
7. The session layer replays 8842 to 8850 from the buffer or the log. If the gap is larger than the buffer, the server sends `RESYNC` and the client fetches a snapshot once. No full reload, no database stampede.

**The storm, prevented:**
8. A deploy later restarts gw7 with 8,000 sockets. Gw7 first sends each client `RECONNECT in random(30..90 s)`, then stops accepting new connections, then exits. 8,000 reconnects spread over 60 s is about 133 per second, which the other 11 nodes absorb. Compare with 8,000 at once.

---

## 9. Map to Rare.lab's stack

**Already used, without calling it a gateway:**
- **Cloudflare (edge).** TLS termination, anycast and rate-limited admission are free on your front door. Cloudflare Durable Objects (MEMORY, not re-read today) give a single-threaded, addressable object per room that can hold WebSockets, which is the "session plus room" layer for small rooms with the routing problem solved by naming (room id maps to one object).
- **Supabase Postgres with RLS and Supabase Realtime.** CONFIRMED architecture: Elixir cluster, Phoenix Channels, any node reachable. This is your gateway tier at the 1,000 to 100,000 tier for free. Remember RLS applies to what Realtime may send (MEMORY), and the durable truth stays in Postgres.
- **R2 content-addressed immutable scene JSON plus a manifest.** This is the best "log closes the gap" design: a reconnecting client asks "what is the manifest version?" and fetches only missing immutable objects. Replay becomes a diff of two manifests (Day 23, Day 93).
- **One shared WebGL context in the runtime.** Incoming real-time events must never run heavy work on the frame path.

**The ceiling (INFERENCE, measure before acting):**
1. **Do not push per-frame state through the socket.** At 60 fps with a few cursors and a few edits, a naive "send every change" is 60 messages per second per user. In a room of 20 people that is 1,200 per second fanned to 20 sockets each. Coalesce to 10 to 15 Hz for cursors and send edits as ops with ids. This is exactly Discord's "work per event times fanout" problem at tiny scale.
2. **Write the reconnect code first.** Full-jitter backoff and `last_seen_id` resume on the client costs a day and removes your biggest future outage. Add a `RECONNECT in N s` message to your server for deploys.
3. **Use manifest versions as the resume token.** On reconnect send the manifest hash you hold. The server answers with the missing object hashes. No replay buffer is needed.
4. **Know your numbers before choosing.** Supabase Realtime's official benchmark page excerpt shows a 32,000-user test; larger claims were only on a mirror. Load-test your own plan with k6 (the tool Supabase used per the excerpt) before you promise a 10,000-person live event.
5. **Next ceiling** is not sockets, it is the hot room: one live-streamed workshop with 5,000 viewers on one canvas. Plan for relays: viewers are read-only and can be served from a cached snapshot plus a 5 Hz delta channel instead of full sessions.

Concrete first step: write one `connect()` helper in the runtime that implements full-jitter backoff, sends the manifest hash on reconnect, and honors a server `RECONNECT in N` hint. Test it by killing the server and checking that 1,000 simulated clients reconnect over at least 30 seconds.

---

## 10. References, summarized in plain language

1. **Rick Reed, "Scaling to Millions of Simultaneous Connections," Erlang Factory SF, 30 March 2012 (WhatsApp).** Seen via search excerpts of the official slide listing, not read in full. Story: about 200k sockets per server at the start, goal of 1 million, reached by patching the Erlang VM and tuning FreeBSD. Plain version: the language runtime and the kernel were the limit, not the hardware. Secondary summaries claim 2.5M-plus in a test and CPU growing almost linearly with connections.
2. **High Scalability, "How WhatsApp Grew to Nearly 500 Million Users, 11,000 cores, and 70 Million Messages a Second."** Secondary summary. Lesson: very few engineers plus a runtime built for cheap concurrency. I could not fetch it, so I did not use its specific numbers.
3. **Discord, "How Discord Scaled Elixir to 5,000,000 Concurrent Users" (2017) and the 2020 Elixir case study.** Seen via excerpts. Story: one process per user, one per guild, fanout from guild to sessions; huge guilds (30,000 online) fell behind; Erlang message passing cost more than expected; Manifold batches cross-node sends. The 2020 case study reports 12M-plus concurrent users and 26M-plus WebSocket events per second. Lesson: the cost is work per event times fanout, so cut one of them.
4. **Slack Engineering, "Flannel: An Application-Level Edge Cache to Make Slack Scale."** Seen via excerpts and the StackShare summary only. Story: a mass reconnect (an office coming back online) produced slower connections, more reconnects and more retries, so Slack served workspace state from edge caches (4 million simultaneous connections, 600,000 client queries per second at peak, running since January 2017). Lesson: the reconnect bootstrap is the expensive part, so cache it near the user.
5. **Slack status page, 27 June 2018 incident.** An offline batch process bug caused network spikes, all customers were disconnected and could not reconnect; recovery came from restricting new connections and adding capacity. Lesson: a storm needs admission control, not just more servers.
6. **Slack API changelog on `rtm.start` vs `rtm.connect`.** The old call returned a huge payload that got harder to deliver as teams grew, so Slack told developers to use the light `rtm.connect`; the member list was being truncated at 1,500. Lesson: keep the connect call tiny and load state lazily.
7. **Supabase Realtime architecture and benchmark docs.** Official: globally distributed Elixir cluster, Phoenix Channels, Phoenix.PubSub. The benchmark page excerpt showed a 32,000-user test. A 250,000-user claim appeared only on a mirror; I treat it as unverified. Lesson: ask for the load profile behind any big number.
8. **ByteByteGo, "How Discord Serves 15-Million Users on One Server"; The New Stack, "How Discord scales up to millions of users on a single guild server."** Secondary. Mention passive sessions and relays. The text I got was cut off, so I used these only as a direction.
9. **Not retrieved today, would read next:** AWS Architecture Blog "Exponential Backoff and Jitter" (Marc Brooker); Bernstein-style resume protocols in the Phoenix Channels docs; Dan Kegel's "The C10K problem"; Cloudflare Durable Objects and WebSocket hibernation docs; Marc Brooker and Bronson et al., "Metastable Failures in Distributed Systems" (HotOS 2021). Listed so you can find them, not summarized as if I read them.
10. **The pasted LeetCode mock interview (YouTube).** A Google engineer designs LeetCode. Functional: list problems, view and code in any language, submit with an instant verdict, a live leaderboard for weekly and biweekly contests. Non-functional: availability over consistency, low latency (2 to 5 seconds), isolation of untrusted code, about 100k contest users and millions overall, no single point of failure. Entities: problem, user, submission, leaderboard. REST APIs with pagination. DynamoDB, because no joins. Running user code on the API server was rejected (security, CPU, no isolation), VMs were heavy, containers per language chosen, with the interviewer noting VMs isolate better and suggesting serverless. A queue sits between API and runners with exponential retry, a cache serves the leaderboard instead of polling the database, test cases are stored language-neutral and serialized per language, and the database gets a primary with replicas. Full teaching: [Day 74](074-leetcode-online-judge-end-to-end.md). Link to today: the candidate's leaderboard "client refetches every 5 to 10 seconds" is polling. With 100k contest users at one request per 5 seconds that is 20,000 requests per second for a page that changes a few times a second. The upgrade is to push the top 100 over a socket, which brings in the reconnect-storm rules above. The candidate's "exponential retry" has the same missing piece: add jitter.

---

## 11. One-line takeaway

**A connection gateway is cheap when it only holds sockets, safe when sessions outlive sockets, and survivable when clients reconnect with jitter and the front door refuses a stampede, because the thing that kills real-time systems is not the million connections but the minute they all come back at once.**

## Sources

- [Erlang Factory SF 2012, Rick Reed (WhatsApp)](https://www.erlang-factory.com/conference/SFBay2012/speakers/RickReed)
- [Erlang Factory 2012 slides, WhatsApp scaling (PDF)](https://www.erlang-factory.com/upload/presentations/558/efsf2012-whatsapp-scaling.pdf)
- [High Scalability, How WhatsApp Grew to Nearly 500 Million Users](https://highscalability.com/how-whatsapp-grew-to-nearly-500-million-users-11000-cores-an/)
- [Discord, How Discord Scaled Elixir to 5,000,000 Concurrent Users](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
- [Elixir blog, Real time communication at scale with Elixir at Discord](https://elixir-lang.org/blog/2020/10/08/real-time-communication-at-scale-with-elixir-at-discord/)
- [The New Stack, How Discord scales up to millions of users on a single guild server](https://thenewstack.io/how-discord-scales-up-to-millions-of-users-on-a-single-guild-server/)
- [ByteByteGo, How Discord Serves 15-Million Users on One Server](https://blog.bytebytego.com/p/how-discord-serves-15-million-users)
- [Slack Engineering, Flannel](https://slack.engineering/flannel-an-application-level-edge-cache-to-make-slack-scale)
- [Slack status, 27 June 2018](https://status.slack.com/2018-06-27)
- [Slack API changelog, use rtm.connect not rtm.start](https://api.slack.com/changelog/2017-04-start-using-rtm-connect-and-stop-using-rtm-start)
- [Supabase Realtime architecture](https://supabase.com/docs/guides/realtime/architecture)
- [Supabase Realtime benchmarks](https://supabase.com/docs/guides/realtime/benchmarks)
- [Supabase, Supavisor 1 million connections](https://supabase.com/blog/supavisor-1-million)
