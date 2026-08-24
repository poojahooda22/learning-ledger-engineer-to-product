# Day 63 — How does Discord tell 30,000 people in the same server who's online, without broadcasting every status flip to every one of them?

**Date:** 2026-08-24
**Difficulty:** Expert
**Topic:** Presence at scale: the "green dot" problem, tracking and broadcasting who is online, idle, or offline for millions of concurrently connected users, in real time, without the update volume growing faster than the user base. The forcing example is Discord's own July 6, 2017 engineering blog post, "How Discord Scaled Elixir to 5,000,000 Concurrent Users," which disclosed the concrete number that breaks a naive presence design: its `/r/Overwatch` community guild carrying up to 30,000 concurrently connected members at once, all sharing one live "who's here" state. Slack's own "Real-time Messaging" engineering post is used as a second, independently arrived-at source, because Slack's presence architecture converges on the same core mechanism from a completely different codebase and language (Slack's stack is not Elixir), which is itself the evidence that the mechanism is a real, load-bearing pattern rather than one company's idiosyncratic choice. Why a naive "broadcast every status change to every member" design turns one person logging in into thousands of outbound messages, and why that cost does not shrink just because most of those recipients are not even looking at the member list right now. How the real fix is not a bigger broadcast, it is a smaller one: stop broadcasting to the whole guild and start delivering only to the specific member-list window a client has explicitly said it can currently see, plus batching the fan-out that remains by network destination instead of by recipient.
**Stack relevance:** Rare.lab does not have a presence problem today. There is no live multi-user view into the node-based shader editor described in this ledger, so there is no "who else is looking at this graph right now" state to broadcast, and saying that plainly matters before claiming anything transfers, the same discipline Day 62 applied to Rare.lab's current lack of a sharded-uniqueness problem. What does transfer, ahead of need, is the shape of the mechanism: if Rare.lab ever ships live collaborative editing on a shared node graph (the same feature Day 3's Figma teardown covered for cursors), or a "N people watching this shader right now" indicator on a viral public embed running through the shared WebGL runtime, the naive version of either feature is exactly this lesson's naive design, and the fix is the same one: subscribe to what is actually on screen, not to the whole audience.

---

## 1. The company and the breaking number

**Discord, July 6, 2017.** In a blog post titled "How Discord Scaled Elixir to 5,000,000 Concurrent Users," Discord's own engineering team (the post is attributed to co-founder and CTO Stanislav Vishnevskiy, per the byline carried by every mirror and discussion of the post found during this lesson's research; the original was not independently fetched, see the Sources note below) disclosed that the platform had grown to nearly five million concurrent users, with millions of events per second flowing through the system built on Elixir, a language running on the Erlang VM's actor model. In that model, every single connected client gets its own lightweight, isolated process (a GenServer, Erlang/Elixir's standard "one mailbox, one loop" actor abstraction) called a session process, and every Discord server, internally called a guild, gets its own guild process, also a GenServer, which acts as the single source of truth for that guild's live state, including who is currently online.

The number that makes this a genuinely hard fan-out problem, not just a big number, is the one the post uses as its own running example: the `/r/Overwatch` community's Discord guild, which the post describes as carrying up to 30,000 concurrently connected users at once, sharing one guild. Presence, the online/idle/offline/in-voice status shown next to every member's name, is exactly the kind of state that never sits still. Every login, every logout, every multi-minute idle timeout, every voice-channel join or leave is a presence event, and in a guild carrying 30,000 concurrent members, a meaningful fraction of them are flipping status at any given moment, more so around a synchronized trigger, a popular streamer going live, a game patch landing, a whole timezone logging on for the evening. One person's status flip is not a private fact in this design. The guild process holding that state has to inform every other session that is subscribed to it, and in the naive version of the architecture, "subscribed to it" means "every currently connected member of the guild," full stop.

That framing produces the breaking arithmetic. A single presence change, naively broadcast, costs the guild process on the order of N outbound sends, where N is the number of concurrently connected members, up to 30,000 for `/r/Overwatch` on this design. That alone is a real cost, but it is not yet the death blow, because 30,000 sends is a large but finite, one-time piece of work per event. The actual collapse comes from what happens when many members flip state near-simultaneously, which is the common case, not the rare one, because logins and reconnects cluster in time (an evening rush, a patch drop, everyone's client reconnecting together after a brief network blip). If K members change state within the same short window, the guild process's total outbound send volume for that window is roughly K times N, and because K itself tends to scale with N in a large, active guild (more members means more people plausibly flipping status in the same minute), the total broadcast work trends toward the square of guild size, not linear in it. That specific K-times-N framing is this lesson's own arithmetic laid over Discord's disclosed 30,000-concurrent-member figure, not a complexity bound Discord's post states in those terms, and it is flagged as this lesson's synthesis for that reason, the same way Day 62 flagged its own signups-per-second arithmetic as inference layered on a sourced milestone. But the underlying shape, one popular guild's presence traffic growing much faster than its membership count, is exactly the wall Discord's own post describes hitting, and it is why a single-threaded guild process (an Erlang GenServer processes its mailbox one message at a time, by design, so that its own state never races itself, a discipline this ledger's Day 11 consensus lesson and Day 29 write-skew lesson both already established the value of) can become the one place in the whole pipeline that cannot keep up, even while every other part of a nearly-five-million-concurrent-user system is healthy.

---

## 2. Why the naive (demo) design dies

**The obvious version:** one guild holds one shared "who's online" set, a list or set of currently connected member IDs living inside the guild's own process or a shared table. On any presence change, the system looks up every other member currently connected to that guild and pushes each of them a message saying so. This is not a bad idea on a small guild, a ten-person study group's Discord server broadcasting ten members' status to ten other members is trivially fine, in the same way Day 61 pointed out a client-side trie is the *correct* architecture for a small catalog, not a naive stand-in for a better one. The naive design only becomes naive once the guild is large and busy at the same time.

**Death one: broadcast fan-out is O(guild size) per event, and the design cannot tell who is actually looking.** The naive version has no concept of "this particular client currently has its member-list sidebar closed" or "this client is showing a completely different channel and has no member list rendered at all right now." It pays the full 30,000-recipient fan-out cost for a status change regardless of whether even one connected client's UI is presently displaying that information, because the design's unit of subscription is "member of the guild," not "client that can currently see this specific piece of state." Nearly all of that fan-out work, for nearly every event, in a large guild, is thrown away the instant it lands on a client that never renders it.

**Death two: the single guild-actor becomes a serialized bottleneck under exactly the load pattern that matters most.** Discord's own actor-per-guild design is a deliberate, correct choice for consistency, exactly one process owns a guild's live state, so two simultaneous status changes for the same guild can never race each other into a corrupted view, the same single-writer discipline this ledger's Day 11 lesson named for consensus systems generally. But that correctness guarantee has a cost: a GenServer's mailbox is processed strictly one message at a time. If the naive fan-out logic makes the guild process itself issue one `send/2` call per recipient PID, and those PIDs are scattered across dozens of remote Erlang nodes (a large guild's connected members are, in a real distributed deployment, not all sitting on the same machine as the guild process), each of those sends is a cross-node network operation, not a cheap local one. Thirty thousand of those, sequentially, on the one process that also has to process the *next* presence event, the next chat message, everything else that guild does, turns the correctness-preserving single-writer design into a queueing bottleneck precisely when the guild is busiest, which is the worst possible moment for it to slow down.

**Death three: naive polling for presence dies on staleness or on hammering the same hot rows, the mirror image of the push failure.** If a design avoids push entirely and instead has every client poll "who's online in this guild" on a timer, it faces the same fork Day 61 already named for autocomplete under load: poll too slowly and the green dot lags reality by long enough that users stop trusting it, which defeats the entire point of a presence indicator, or poll frequently enough to feel live and turn "is my friend online" into a repeated query against the same busy guild-state row from every one of 30,000 concurrently connected clients, a self-inflicted read storm against exactly the data structure death two already showed struggling under write load. Neither push-without-scoping nor poll-without-scoping survives a guild at this size; both fail because neither one asks the one question that actually matters, which of the people who could theoretically care are actually watching right now.

---

## 3. The architecture

```
Client (Gateway WebSocket connection)
  - opens one persistent connection, authenticates (IDENTIFY) or
    resumes a prior session (RESUME), then explicitly subscribes to
    only the member-list ranges its own UI is currently rendering,
    typically ~100 members at a time, requesting more as the user
    scrolls
  - job: ask only for what you can actually see
  - analogy: subscribing to the two house numbers you're walking past
    right now, not to every address in the city, on the assumption
    the city will tell you anyway

        |  WebSocket: explicit per-range subscription request
        v
Gateway edge / connection load balancer
  - terminates the websocket, assigns the connection to a gateway
    node, and on a mass-reconnect event (an outage recovering, a
    deploy rolling), deliberately throttles how many clients are
    allowed to resume or re-identify at once rather than accepting
    every retry at full force
  - job: turn a stampede of reconnects into a metered line
  - analogy: a bouncer re-admitting a venue's crowd in an orderly
    line after a fire-alarm evacuation, not letting everyone surge
    the door together

        |
        v
Session process (one lightweight actor per connected client)
  - a single-mailbox actor holding just that one connection's state
    and its current subscriptions
  - job: represent exactly one client's view; receive only the
    events that client actually asked for
  - analogy: one mail slot, addressed, not a megaphone aimed at a
    crowd

        |  connects to / receives from
        v
Guild process (one actor per server, the presence source of truth)
  - every status change for that guild (login, idle timeout, voice
    join or leave) lands here first, and here alone, so the guild's
    presence state can never race itself
  - fan-out is scoped to sessions that hold an active subscription
    to the affected member-list range, not to every connected member
  - job: be the one place a guild's live state is authoritative;
    process its mailbox one event at a time, by design
  - analogy: one venue announcer reading the guest list once, who
    only pages the specific section of doorstaff whose gate the
    change actually affects

        |  batched by destination, not by recipient
        v
Cross-node batched transport (Discord's own Manifold library)
  - groups the destination session PIDs by which remote Erlang node
    they live on before sending, so the guild process calls send
    once per NODE involved, not once per member, no matter how many
    thousands of members happen to be hosted on that node
  - job: keep the guild process's own outbound work bounded by node
    count, which grows slowly, instead of member count, which does
    not
  - analogy: one delivery truck per neighborhood block, loaded with
    every package for that block, instead of one truck making a
    separate trip for every single house

        |
        v
Shared read-mostly state (Discord's own FastGlobal library)
  - guild metadata every session process reads constantly (channel
    lists, permission overwrites) but that changes rarely, compiled
    into Erlang constant term pools instead of copied on every read;
    FastGlobal's own published benchmark shows a get at 0.33
    microseconds per op versus 7.64 for a plain ETS table and 12.67
    for a GenServer-backed Agent
  - job: make a value that changes rarely and is read by thousands
    of processes effectively free to read, so it never competes for
    the same attention as the presence fan-out path above it
  - analogy: printing a poster once and pinning it to the wall,
    instead of running a fresh photocopy for every person who wants
    to read it
```

The scoped-subscription layer in the middle of this diagram, deliver presence only for the member-list range a client explicitly asked for, is documented under the community-reverse-engineered name "Lazy Guilds" (this specific label and its wire-level detail, Gateway Opcode 14 "Guild Subscriptions" carrying an explicit member-list range such as `[0, 99]`, come from unofficial, community-maintained Discord API documentation, not from an official Discord engineering post found during this lesson's research, and are flagged here as lower-authority than the primary sources above, the same caveat Day 62 applied to a fan-maintained wiki). What matters for this lesson is not the exact opcode number, it is that Slack's own, independently built presence system converges on the identical mechanism from a completely different codebase: Slack's engineering blog post "Real-time Messaging" describes dedicated, in-memory Presence Servers that users are hashed to, reached through a Gateway Server acting as a websocket proxy, and states plainly that a Slack client receives presence notifications only for the subset of users actually visible in that client's app screen at any given moment, the same viewport-scoped delivery Discord's community docs describe for guild member lists. Two companies, two languages, two actor models (Elixir's GenServers versus whatever Slack's Presence Servers are built on, not disclosed in the source used here), landing on the same "subscribe to what's visible, not to everyone" answer is the strongest available evidence that this is a real, convergent pattern rather than one team's implementation quirk, the identical kind of cross-company convergence Day 62 used compare-and-set writes across Manhattan, Cassandra, DynamoDB, and MongoDB to establish.

---

## 4. The transferable mechanisms

- **Subscribe to the visible window, not to the whole population.** The single idea underneath this entire lesson: presence delivery cost should scale with what is currently rendered on someone's screen, a member-list page, a viewport, a visible cursor, not with the size of the population that could theoretically be shown. Discord's per-range guild subscriptions and Slack's viewport-scoped presence notifications are the same mechanism, arrived at independently, and it is the one change that actually breaks the O(guild size) fan-out cost this lesson's Section 2 described, rather than just making that same O(guild size) broadcast faster.

- **Actor-per-entity with a strictly serialized mailbox, to buy correctness before you optimize throughput.** One guild process owning one guild's presence state, processing one message at a time, is what makes it safe to *then* go optimize the fan-out path (batching, scoping) without ever risking two concurrent status changes corrupting each other's view. This is Day 11's single-writer intuition and Day 29's write-skew lesson, both made concrete as a specific actor-model implementation choice rather than an abstract database isolation level.

- **Batch fan-out by destination locality, not by recipient identity.** Manifold's group-by-remote-node-first approach turns a guild process's outbound work from "one send per member" into "one send per node," which is the same shape as Day 53's flash-broadcast lesson batching a push-notification fan-out by pre-warmed worker rather than dispatching one job per recipient, and the same shape gossip protocols (Day 25) use when a node tells its neighbors about a change instead of every affected node individually.

- **Make rarely-changing, widely-read data effectively free to read, separately from the hot write path.** FastGlobal's 0.33-microsecond get, versus 7.64 for ETS and 12.67 for an Agent, exists specifically because guild metadata is read by thousands of processes far more often than it changes, and paying a copy or lookup cost on every one of those reads would compete for the same CPU the presence fan-out path needs. This is Day 19's caching-strategy lesson, specialized to the case where the "cache" is a language-level constant pool rather than a separate service.

- **Treat presence as bounded-staleness by contract, not as a value that must be instantaneously correct.** A green dot that is a second or two stale is not a bug, it is the entire acceptable operating range for this data type, the same eventual-consistency posture Day 31's session-guarantees lesson argued for causal, not linearizable, reads. Slack's own documented 10-minute auto-away timeout is this same discipline made into a product rule: rather than trying to precisely detect the exact moment someone stepped away, a coarse timer accepts a bounded amount of staleness in exchange for never having to track truly continuous activity.

- **Throttle the reconnect path deliberately, don't just try to absorb it.** Discord's own Gateway documentation describes Opcode 9, Invalid Session, being used specifically to tell a fraction of reconnecting clients "not yet" after an outage, rather than accepting every simultaneous reconnect attempt at full force, paired with Opcode 6, Resume, letting a client that reconnects quickly replay only the sequence-numbered gap in events it missed instead of re-subscribing to full state from scratch. This is Day 13's backpressure lesson applied to the specific moment presence systems are most fragile, the instant after an outage, when every client's natural instinct is to reconnect at once.

---

## 5. The trade-offs

**Consistency vs. availability, and presence sits at the opposite pole from Day 62's uniqueness write.** A username claim had to be strongly consistent because a visible, unrecoverable-feeling bug results the moment it isn't, two people both believing they own the same handle. Presence is the mirror case: nothing meaningfully bad happens if two different clients briefly disagree about whether a given member is online, and trying to make presence strongly consistent, coordinating every status flip through a consensus round before showing it to anyone, would spend real latency and availability defending a guarantee the feature does not need. The deliberate choice here is to bias hard toward availability and low latency, and let brief, self-correcting staleness be the accepted cost, because the alternative (a green dot that takes noticeably longer to update, or a guild page that hangs waiting for a coordinated write) is strictly worse for what this feature is actually for.

**Cost vs. latency, paid as bookkeeping instead of as bandwidth.** Scoped, per-range subscription tracking (Lazy Guilds, Slack's viewport-scoped delivery) is more complex to build and operate than a flat broadcast-to-everyone design, the server now has to track which specific ranges which specific sessions currently care about, and update that bookkeeping as clients scroll or switch channels. That complexity is a real, ongoing engineering cost. It is worth paying because the alternative cost, bandwidth and CPU that scale with guild membership rather than with what is actually on screen, grows without bound as guilds grow, while the bookkeeping cost grows only with how many distinct ranges are actively being watched at once, a number that stays roughly proportional to concurrently open UI surfaces, not to total membership.

**Freshness vs. update volume, resolved by accepting a coarse timer over a precise one.** Detecting "away" precisely, the exact moment a person's attention left the app, is both hard to define and expensive to track continuously. A flat idle timeout (Slack's documented 10 minutes) trades a small, bounded amount of freshness, someone might still be at their desk, unmarked as away, for a large reduction in how often the system needs to evaluate and broadcast a status change at all. The trade-off this lesson keeps returning to, spend precision only where the feature actually needs it and buy back efficiency everywhere else, shows up here at the level of a single product-facing timer value, not just in the architecture diagram.

---

## 6. The systems-thinking lens

The feedback loop worth naming here is a **reconnect storm**, a specific, well-documented instance of the thundering-herd pattern this ledger has already named twice, in Day 13's backpressure lesson and Day 57's distributed-cron lesson. It plays out like this: an outage, a deploy, or even a brief network blip drops a large number of gateway connections simultaneously. Every one of those disconnected clients has, by design, the same instinct: reconnect immediately, then re-establish full presence visibility for every guild it cares about. If reconnection and re-subscription are naive, every client's clock ticks to "retry now" on approximately the same beat, which means the exact infrastructure that just came back up, or is still recovering, gets hit by a spike of full IDENTIFY handshakes and full guild-subscription requests all landing in the same few seconds, precisely the load pattern this lesson's Section 2 already showed a single guild process struggling to absorb even under ordinary conditions. If that spike knocks the gateway back down, or even just slows it enough that clients start timing out and retrying again, the loop reinforces itself: more failed connections produce more simultaneous retries, which produce more load, which produces more failures.

The naive instinct, give the gateway more capacity so it can absorb a bigger spike, does not break this loop, it only raises the herd size at which the same synchronized-retry pattern eventually overwhelms it again, because the underlying cause is not insufficient throughput, it is that thousands of independent clients are all deciding to act at the same instant. Discord's own documented Gateway behavior breaks the loop with two changes, neither of which is "more capacity." First, Opcode 9, Invalid Session, deliberately rejects a fraction of reconnect attempts during a recovery window, an explicit, honest "not yet" that spreads the retry load out over time rather than accepting it all up front, the identical "fail fast and redirect load" instinct Day 62's inverted-hot-key lens already named for a different kind of stampede. Second, Opcode 6, Resume, lets a client that does get back in replay only the sequence-numbered gap in events it missed, instead of re-subscribing to full guild presence state from scratch, which shrinks the size of the work each successful reconnect generates, on top of throttling how many reconnects happen at once. Client-side exponential backoff with jitter, the standard companion to both, is what actually desynchronizes the herd's clock, ensuring the next wave of retries does not arrive as one coordinated spike either. The senior fix, across all three, is the same shape this ledger keeps returning to: break the synchronization that turns independent, reasonable individual behavior into a coordinated system-wide spike, don't just build a bigger door for the spike to hit.

---

## Map to Rare.lab's stack

Rare.lab does not have a presence problem today, and that is worth stating plainly before claiming anything transfers. The product described in this ledger is a node-based editor that compiles shader graphs to shippable code, plus an embeddable runtime sharing one WebGL context, and nothing in that description implies a live, multi-user view into a shared piece of state the way a Discord guild's member list or a Figma canvas's live cursors are. There is no "who else is looking at this graph right now" state to broadcast, and building the scoped-subscription machinery this lesson describes ahead of any actual multi-user feature would be exactly the premature-infrastructure mistake Day 62 warned against for uniqueness constraints Rare.lab does not yet need either.

The gap that would open this lesson's mechanism is a specific, plausible future feature, not a hypothetical abstraction: live collaborative editing on a shared node graph, the same category of feature Day 3's Figma teardown covered for multiplayer cursors, or a "N people are viewing this shader" indicator surfaced on a popular public embed running through the shared WebGL runtime. Either one is, structurally, this lesson's problem in miniature: a single popular canvas or embed, watched by many concurrent viewers at once, where naively broadcasting every cursor move, every selection change, or every viewer-join event to every other connected viewer reproduces `/r/Overwatch`'s exact fan-out shape, just at a smaller absolute scale that Rare.lab is unlikely to hit soon, but at the same *structural* risk, one popular shared object attracting a crowd that a flat broadcast design cannot serve cheaply.

The concrete, actionable lesson worth banking now, before either feature exists, is a design constraint rather than new infrastructure: if and when Rare.lab builds live multi-viewer state for a shared node graph or a viral embed, scope delivery to what a given client's viewport is actually rendering, which nodes are currently visible in the graph editor, which portion of a crowded viewer list is actually being shown, from the very first version, rather than shipping a flat broadcast-to-everyone implementation and needing a Lazy-Guilds-style retrofit later once one popular public shader accumulates the kind of concurrent audience that makes the naive version expensive. The mechanism this lesson teaches, subscribe to what's visible, not to who exists, costs nothing to design in from day one and is exactly the kind of retrofit that is expensive to bolt on after a broadcast-shaped system is already load-bearing in production.

---

## Sources

- [How Discord Scaled Elixir to 5,000,000 Concurrent Users, Discord Blog](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users) (July 6, 2017): primary source for the nearly-five-million concurrent user figure, millions of events per second, the session-process/guild-process GenServer architecture, the `/r/Overwatch` up-to-30,000-concurrent-member figure used as this lesson's breaking number, and the Manifold cross-node batching library described in Section 3. Direct fetch to discord.com was blocked by this session's network egress policy; relayed via search-indexed summaries and secondary discussion of the post (Elixir Forum, Hacker News, and syndicated mirrors), not a first-hand read, and worth re-verifying directly.
- [discord/fastglobal, GitHub](https://github.com/discord/fastglobal): primary source, fetched directly in this session, for FastGlobal's stated purpose (large, infrequently-changing data read by thousands of processes, avoiding the per-read copy cost of a plain process mailbox or an ETS table) and its published benchmark numbers (0.33 microseconds per get versus 7.64 for ETS and 12.67 for an Agent), referenced in Section 3 and Section 4.
- Lazy Guilds, unofficial Discord API documentation (community-maintained, not an official Discord source): source for the Gateway Opcode 14 "Guild Subscriptions" mechanism, the explicit member-list-range subscription model (for example `[0, 99]`), and the description of Lazy Guilds being rolled out to all guilds rather than only guilds Discord classifies as "large," referenced in Section 3. Flagged explicitly as lower authority than Discord's own primary engineering and API documentation, the same caveat Day 62 applied to a fan-maintained wiki; direct fetch was blocked, relayed via search-indexed summary.
- Discord's official Gateway documentation (docs.discord.com): primary documentation source for Opcode 9, Invalid Session, and Opcode 6, Resume, and their role in throttling mass-reconnect events and replaying missed events via sequence number rather than a full re-subscription, referenced in Section 3, Section 4, and Section 6. Direct fetch was blocked; relayed via search-indexed summary of the documentation's own described behavior.
- [Real-time Messaging, Slack Engineering Blog](https://slack.engineering/real-time-messaging/): primary source for Slack's Presence Server architecture, hashing users to individual presence server instances, the Gateway Server's role as a websocket proxy, the statement that presence notifications are scoped to only the users visible in a client's current app screen, and Slack's disclosed scale figures (tens of millions of channels per host, tens of millions of connected clients, message delivery worldwide in roughly 500ms), referenced in Section 3. Direct fetch to slack.engineering was blocked; relayed via search-indexed summary, worth re-verifying directly.
- Slack's user presence and status documentation (docs.slack.dev / api.slack.com): primary documentation source for the roughly 10-minute automatic away-timeout rule referenced in Section 4 and Section 5. Direct fetch was blocked; relayed via search-indexed summary.

---

*Inference vs. fact, stated plainly: Discord's concurrent-user count, events-per-second figure, the session-process/guild-process GenServer architecture, the `/r/Overwatch` 30,000-concurrent-member figure, and the existence and purpose of the Manifold library are documented claims from Discord's own named engineering blog post, but that post was relayed through this session's web search and secondary discussion of it rather than a first-hand read, because direct fetch to discord.com was blocked by this session's network egress policy; it was not independently re-verified and should be treated as worth confirming directly. FastGlobal's stated purpose and benchmark numbers were fetched directly from the discord/fastglobal GitHub repository in this session and are treated as a confirmed primary source. The Lazy Guilds mechanism and its Opcode 14 wire-level detail come from unofficial, community-maintained Discord API documentation, explicitly lower authority than an official Discord source, and are flagged as such in Section 3 and the Sources list rather than presented as Discord's own confirmed description of its architecture. Discord's Gateway Opcode 9 and Opcode 6 behavior comes from Discord's own official developer documentation as relayed through search, not a first-hand read, and is treated as reliable given its documentation-primary origin while still being flagged as unverified directly by this session. Slack's Presence Server architecture, its viewport-scoped presence delivery, its disclosed scale figures, and its away-timeout rule all come from Slack's own named engineering blog and documentation as relayed through search, not a first-hand read, for the same network-policy reason. The K-times-N quadratic-trend arithmetic in Section 1, the specific framing of Section 2's three deaths, the architecture diagram's exact layering, and the entire Rare.lab mapping in the final section are this lesson's own synthesis built on top of the documented mechanics above, not a claim that Discord, Slack, or any cited source describes their systems in these exact terms.*
