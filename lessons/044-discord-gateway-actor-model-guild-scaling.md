# Day 44 — How does one Discord server hold 21 million members live, when the engineers who built it thought 1 million was close to the ceiling?

*2026-07-30*

---

## 1. The company and the number that breaks a naive design

**Discord, 2017 to 2024.** Discord's real-time layer (chat, presence, typing indicators, voice signaling) runs on Elixir, on top of the Erlang VM (BEAM), because BEAM was built from the ground up for exactly this shape of problem: millions of small, independent, long-lived processes that need to talk to each other without one slow one blocking the rest. In 2017 Discord published that this architecture had scaled to **5,000,000 concurrent users** (Discord Engineering, "How Discord Scaled Elixir to 5,000,000 Concurrent Users"). By 2020, using a Rust extension to solve a specific bottleneck, they were at **11 million concurrent users**, and shortly after, Discord's own account (cross-posted on the official Elixir language blog) reported crossing **12 million concurrent users platform-wide, sustaining more than 26 million WebSocket events pushed to clients every second** (elixir-lang.org, "Real time communication at scale with Elixir at Discord").

Those are platform-wide numbers, spread across millions of separate Discord servers ("guilds" in Discord's internal vocabulary). The sharper, more specific breaking number is about a *single* guild. Before 2022, Discord's engineers believed a single guild topping out around **1,000,000 concurrently connected members** was close to the practical ceiling, because one guild is handled, internally, by one process. Then Midjourney's Discord server, driven by the sudden mainstream popularity of AI image generation, blew past that number and kept climbing. Discord formed a small team ("MaxJourney") specifically to raise that ceiling, and shipped changes that expanded a single guild's live capacity **15x** (Discord Engineering, "Maxjourney: Pushing Discord's Limits with a Million+ Online Users in a Single Server"). Midjourney's server sits today at roughly **21 million members**, with well over a million of them concurrently online at once, all inside what is, internally, still one logical process.

The number that breaks a naive design, in one line: **one process, handling one chat room, has to notify every currently-connected member every time anything happens, and that process choked once concurrent membership in a single room approached a million.**

## 2. Why the naive design dies

The demo version of "a chat server" is simple: one process (or one thread) owns the list of everyone connected to a room, and when a message arrives, it loops over that list and sends the message to each connection, one at a time. This is exactly the shape Discord actually used internally (a "guild process" holding a list of session PIDs to notify), so it is not a strawman, it is the real starting architecture, and here is where it breaks as the list grows into the hundreds of thousands and beyond.

**a. Broadcast cost is a straight line, and everything is serialized behind one process.** Discord measured the cost of one Erlang `send/2` call (the primitive that hands a message to another process) at roughly **70 microseconds** (discord/manifold, project README). That sounds trivial until you multiply it: a popular guild with 100,000 connected sessions means 100,000 individual `send/2` calls to notify everyone of one message, all issued sequentially by the single process that owns that guild, because that process is (by design, for correctness) single-threaded with respect to its own state. 100,000 x 70 microseconds is 7 seconds of pure send-dispatch time, for one message, before the next message in the queue can even start being processed. In a chat room, messages do not wait politely; they keep arriving, and the guild process's own mailbox backs up faster than it can drain.

**b. The membership data structure itself gets expensive to touch.** BEAM's core data structures (lists, maps) are immutable by default: any change produces a new copy of the structure containing that change, rather than mutating in place. Discord's own account of scaling to 11 million users describes exactly this trap: a guild's live member list, kept as a native Elixir data structure, meant that a single person joining a 100,000-member guild required constructing a brand-new list with 100,001 entries in it (Discord Engineering, "Using Rust to Scale Elixir for 11 Million Concurrent Users"). That is an O(n) copy triggered by an O(1) event, and it happens on every join, every leave, continuously, for the guild's entire lifetime.

**c. Full-fidelity fan-out to everyone, regardless of whether anyone is looking.** The naive design treats every connected member as equally deserving of every event: a typing indicator, a presence change, a reaction. In a guild with a million connected members, most of whom have that server open in a background tab they have not glanced at in twenty minutes, blasting every event to all million of them is pure waste, work performed for an audience that, in that instant, does not exist.

The analogy: imagine a single building superintendent who is contractually required to personally knock on the door of every one of a million residents, one at a time, on foot, every single time there is a building-wide announcement, and who must also personally recopy the entire tenant roster by hand every time someone moves in or out. It does not matter how fast the superintendent walks or writes; the job is structured as one person, doing one thing at a time, for a million-person building. No amount of individual speed fixes a fundamentally serial job.

## 3. The architecture, drawn top to bottom

```
CLIENTS (Discord desktop, mobile, web apps)
   open ONE persistent WebSocket connection each to the Gateway
   |
   v
GATEWAY / SHARD LAYER (public API: wss://gateway.discord.gg)
   each guild is assigned to a SHARD by (guild_id >> 22) % num_shards;
   a shard is capped at 2,500 guilds, ~1,000 recommended per shard
   job: accept the socket, authenticate, translate the wire protocol,
   run heartbeats, and support SESSION RESUME (session_id + last
   sequence number) so a brief network blip does not force a full
   re-sync of everything that client subscribes to
   analogy: a phone exchange operator who only patches you through
   to the extension for your own building, never the whole city,
   and who can pick your call back up mid-conversation after a drop
   |
   v
SESSION PROCESS (one lightweight Elixir process per connected user)
   holds this one user's subscriptions and owns the actual socket send
   an Erlang VM node can hold roughly 500,000 live sessions at once
   analogy: your own personal mail slot in the building; it only
   ever handles mail addressed to you
   |
   v
GUILD PROCESS (one process per Discord server; the source of truth)
   owns membership, presence aggregation, permissions, routing
   decisions for that one guild, and nothing else
   PASSIVE vs ACTIVE split: members who are connected but not
   actively viewing the guild are marked "passive" and excluded
   from the full-fidelity fan-out path; in Midjourney-scale guilds,
   roughly 90% of connections are passive at any given moment
   analogy: the building superintendent, who keeps the master
   tenant list but only actually knocks on doors of people who
   are home and paying attention right now
   |
   v
MANIFOLD + SENDER PROCESSES (the batching and fan-out layer)
   the guild process no longer calls send/2 itself; it hands the
   recipient list to Manifold, which groups recipient PIDs by their
   remote Erlang node (consistent hashing via :erlang.phash2/2),
   spins up one partitioner per node plus a worker pool sized to
   CPU cores, and each worker delivers to its slice in parallel;
   a further step (built for MaxJourney) hands the actual outbound
   network writes to separate SENDER processes, so a slow socket on
   one straggling client can never stall the guild process itself
   analogy: instead of the superintendent hand-delivering every
   flyer, they hand a stack to one floor captain per floor, who
   each hand smaller stacks to runners, who deliver in parallel
   |
   v
RUST NIF LAYER (a SortedSet, bridged in via Rustler)
   the guild's live member list moved out of native, immutable
   Elixir data structures into a Rust-backed sorted set; a join no
   longer means "copy a list of 100,000 into a list of 100,001",
   it means an in-place, ordered insert; Rustler's guarantees mean
   a bug in this Rust code cannot crash or corrupt the BEAM VM
   around it
   |
   v
ETS (Erlang Term Storage: shared, mutable, in-memory tables)
   used for the guild-to-node routing ring (so any process can look
   up "which node owns this guild" with a direct table read instead
   of a message round trip), and for parallelizing mass-notification
   work like an @everyone mention across many worker processes at
   once instead of one process touching every member serially
```

## 4. The transferable mechanisms

- **Actor-per-entity with bounded blast radius.** One lightweight process per stateful unit (one per user session, one per guild) instead of a shared lock or a shared thread pool. A hot guild's process backing up cannot stall an unrelated guild's process; each is its own island. Figma's per-document process (Day 3) is the same idea for a design file; this is the general pattern: give every independently-scaling entity its own actor, and let isolation, not raw speed, be what keeps the system alive under skew.

- **Tiered subscription by attention, not by membership.** Not every subscriber needs full fidelity. The passive/active split routes full event fan-out only to members who are plausibly looking right now, and coarser or no updates to everyone else. The mechanism generalizes to any push system: fan-out cost should scale with attention, not with raw audience size, the same instinct behind Day 43's star-tree index choosing what to precompute based on what actually gets queried.

- **Batch-and-partition instead of one-by-one dispatch.** Manifold turns "N individual sends from one caller" into "at most one send per remote node," then parallelizes inside that node with a bounded worker pool. Discord measured this cutting per-server packet rate in half immediately after deployment (discord/manifold README). This is directly reusable any time one process must notify many remote peers: group by destination first, then fan out in parallel per group, rather than looping over every individual recipient from a single thread.

- **Session resumption via an opaque ID plus a sequence number.** A WebSocket drop (mobile network hiccup, a deploy) does not have to mean a full state re-sync. The client keeps its session_id and the last sequence number it saw, and asks the gateway to resume: only the missed events replay. This is the WebSocket-world sibling of a Kafka consumer offset (Day 42) or an idempotency key (Day 12): give a stateless reconnect a cheap way to say "pick up exactly where I left off."

- **Drop to a narrower, safely-bridged native layer for one specific hot data structure, not the whole system.** BEAM's immutable-by-default cost model is right for almost everything Discord does, and wrong for one specific structure (a large, frequently-mutated ordered member list). Rather than rewriting the whole real-time layer in a faster language, Discord bridged in Rust for exactly that one structure via Rustler, whose safety guarantees mean the native code cannot destabilize the surrounding VM. The general move: measure which specific data structure has a cost curve that does not fit its access pattern, and replace only that, in a safely isolated way.

- **Separate "who decides" from "who does the slow I/O."** The guild process makes decisions (who gets what, in what order) and never itself blocks on a network write; sender processes do the actual, potentially-slow socket I/O. Decoupling decision-making from execution means a queue backing up on execution never backs up the decision-maker's own mailbox.

## 5. The trade-offs

CAP made concrete, per data type:

- **Presence, typing indicators, and passive members' view of guild state: availability and freshness win over strict consistency.** A passive member's client can lag behind the guild's true state for a while; they will catch up the moment they actually open that guild. Discord explicitly accepts staleness for backgrounded connections in exchange for not paying full fan-out cost for an audience that, right now, is not watching.

- **One guild, one process, one place to look: consistency and simplicity win over easy horizontal scaling.** Every decision about a given guild happens in one place, so there is never a question of which of several replicas has the "real" answer. The cost is a genuine single point of contention for any one hot guild; Discord accepts this the same way Figma accepts one process per document (Day 3), because it makes correctness trivial to reason about, and then spends its engineering effort shrinking and offloading what that one process has to do, rather than trying to split the undivided truth across multiple processes.

- **The Rust NIF bridge: safety and raw throughput, bought with operational complexity.** Rustler's compile-time guarantees mean a panic in the Rust code cannot crash the BEAM VM around it, but the team now maintains a compiled native dependency alongside interpreted Elixir code, a real ongoing cost, accepted because BEAM's own persistent-data-structure cost model could not deliver the throughput this one hot path needed.

- **Sharding guilds by a fixed hash formula: even distribution and operational simplicity, over graceful handling of skew.** `(guild_id >> 22) % num_shards` is cheap, predictable, and requires no coordination to compute, but it has no notion of "this particular guild is enormous, route it specially." A smarter, rebalancing-aware placement (closer to Day 34's cell-based routing) would handle a Midjourney-sized outlier more gracefully, at the cost of a much more complex system to build and operate. Discord's actual answer to the outlier case was not smarter placement, it was a dedicated engineering project (MaxJourney) to make the one process that outlier lands on capable of carrying far more weight.

## 6. The systems-thinking lens

The feedback loop here is a **hot-actor mailbox backlog**, the same shape as Day 16's hot-key problem, but for a single stateful process instead of a single cache key. As a guild's concurrently-online count grows, every incoming event (a message, a join, a reaction) generates proportionally more fan-out work for that guild's one process to perform. If the fan-out cost per event grows faster than the process can drain its own mailbox, the backlog grows without bound: each new message waits longer behind the ones ahead of it, clients start feeling "stuck" and reconnect or retry, and those reconnects generate more subscribe and re-sync traffic aimed at the very same already-backlogged process. This is structurally identical to the retry death spiral from Day 13, except the overloaded resource is one actor's mailbox rather than a database connection pool.

The naive response, "give it a bigger machine, more CPU cores," does not touch this problem, because a single Erlang process is inherently single-threaded with respect to its own mailbox; more cores sitting idle next to an overloaded single process do nothing for it. The senior fix, visible in every piece of MaxJourney's work, is not adding capacity, it is shrinking and offloading what has to pass through that one unavoidable single point:

- **Passive/active tiering** removes roughly 90% of connected members from the guild process's own fan-out decision entirely, before the decision is even made.
- **Manifold and dedicated sender processes** move the actual, potentially-slow byte-pushing work off the guild process, so its mailbox only ever holds "decide and dispatch," never "wait on a straggling client's socket."
- **The Rust-backed member list** removes the O(n) copy tax from membership churn, so a single join or leave costs roughly the same regardless of how large the guild has already grown.

The general lesson: when one logical entity genuinely must remain a single point of truth for correctness (a document, a guild, an account), the fix for that entity's scale ceiling is never to parallelize the untouchable core itself. It is to protect that core by removing everything from its critical path that does not strictly require its involvement, and delegating the rest to processes that can fail, queue, or run in parallel without threatening the one thing that has to stay singular.

---

## Sources

- Discord Engineering, ["How Discord Scaled Elixir to 5,000,000 Concurrent Users"](https://discord.com/blog/how-discord-scaled-elixir-to-5-000-000-concurrent-users)
- Discord Engineering, ["Using Rust to Scale Elixir for 11 Million Concurrent Users"](https://discord.com/blog/using-rust-to-scale-elixir-for-11-million-concurrent-users)
- elixir-lang.org, ["Real time communication at scale with Elixir at Discord"](https://elixir-lang.org/blog/2020/10/08/real-time-communication-at-scale-with-elixir-at-discord/)
- Discord Engineering, ["Maxjourney: Pushing Discord's Limits with a Million+ Online Users in a Single Server"](https://discord.com/blog/maxjourney-pushing-discords-limits-with-a-million-plus-online-users-in-a-single-server)
- discord/manifold, ["README: fast batch message passing between nodes for Erlang/Elixir"](https://github.com/discord/manifold/blob/master/README.md)
- Discord Developer Documentation, ["Gateway: sharding formula and guild-per-shard limits"](https://discord.com/developers/docs/events/gateway)
- ByteByteGo, ["How Discord Serves 15-Million Users on One Server"](https://blog.bytebytego.com/p/how-discord-serves-15-million-users)
