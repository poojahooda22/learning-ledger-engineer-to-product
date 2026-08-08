# Day 51 — How does WhatsApp deliver a message to five devices at once, exactly once, even if one of them has been dead for three weeks?

**Date:** 2026-08-08
**Difficulty:** Expert
**Topic:** Store-and-forward messaging at planetary scale: why a message is not "sent" until it survives a crash, why one logical message becomes up to five separate encrypted messages the instant a user links a laptop and a tablet, and how WhatsApp's Erlang connection layer plus the Signal Protocol's per-device sessions and group Sender Keys keep 100 billion messages a day both private and durable without a central plaintext copy anywhere.
**Stack relevance:** Rare.lab's runtime keeps a live WebGL context per session and will eventually need to sync a scene graph across a creator's own devices, or fan updates out to viewers watching a shared session. The moment "one edit" has to become "N consistent copies, delivered even to a client that was offline when it happened," this is the same problem: durability before delivery, per-recipient fan-out instead of one shared broadcast, and an acknowledgment model that survives a client reconnecting hours later with a stale view of the world.

---

## 1. The company and the breaking number

**WhatsApp**, now under Meta. The company that put this problem on the map published its own numbers rather than leaving them to estimates, which is rare enough in this space to lean on directly.

The connection-layer number comes from Rick Reed, a WhatsApp engineer, at Erlang Factory SF in March 2012, in a talk titled "Scaling to Millions of Simultaneous Connections." WhatsApp's internal target was **1 million concurrently open connections on a single server**. After a round of FreeBSD and BEAM (the Erlang virtual machine) tuning, reducing lock contention, retuning the scheduler and memory allocator, cutting cross-process chatter, they pushed a single box past that target to **over 2 million, peaking near 2.8 million concurrent TCP connections on one machine**. By 2014, High Scalability's widely cited writeup of WhatsApp's architecture put the whole fleet at roughly **500 million users, 11,000 server cores, and a peak of 70 million messages a second, run by a team of about 32 engineers**.

The volume number is more recent and just as concrete. WhatsApp's own head, Will Cathcart, said in October 2020 that the service was delivering **more than 100 billion messages a day**, roughly 1.1 million messages every second, around the clock, not a burst peak. That is the number the naive design has to survive continuously, not just at a headline moment.

The number that actually breaks a naive design, though, is not the raw volume. It is a second number that showed up only in July 2021, when Meta's engineering blog and a companion WhatsApp Security Whitepaper (version 5, updated September 2021) described WhatsApp's **multi-device architecture**: a user's account can now be live on their phone plus **up to four additional "companion" devices** (desktop, web, tablet) at the same time, each one holding its **own independent end-to-end encrypted session**, and a linked desktop can send and receive even while the phone is completely powered off. One logical message, from one sender, to one recipient, can now require **up to five separately encrypted physical copies** delivered to five separate device queues, each with its own delivery state. A design built around "one row per message, one delivered flag" was already wrong before multi-device; after it, it is wrong in a way that leaks message content or drops messages outright, not just slow.

## 2. Why the naive (demo) design dies

**Version one: a `messages` table with a `delivered` boolean, and a server that pushes over an open socket if the recipient looks connected.** This is the version every chat app tutorial builds. It fails on four separate axes, not one.

**Connection concurrency.** A conventional thread-per-connection server model, one OS thread blocked on one socket, the default shape in a lot of traditional application servers, runs out of memory and context-switch budget long before a million open connections on one box. Threads default to megabyte-scale stacks; a million of them is terabytes of stack space nobody is using, plus a scheduler thrashing between them. This is exactly the wall Rick Reed's team hit and tuned past: they did not switch languages to get faster CPU, they switched to Erlang's actor model, where each connection is a lightweight BEAM process (kilobytes, not megabytes, and cooperatively scheduled by the VM, not the OS), specifically because the OS thread model could not hold the connection count they needed on one machine.

**A shared boolean cannot represent multi-device state.** Once an account can be logged into a phone and four companions at once, "delivered" is not one bit, it is one bit *per device*. If two devices ack at slightly different times and both write to the same row, the naive design either overwrites one device's ack with the other's (lost update) or needs application-level locking on every message row just to survive concurrent writers, which turns every send into a point of lock contention exactly where you cannot afford one.

**Direct-push-or-nothing loses messages.** If "deliver" means "write to the currently-open socket," then the moment the recipient is not connected at the instant the sender hits send, and given phones sleep, lose signal, or sit in a pocket with the app backgrounded, the message simply has nowhere to go. There is no durable intermediate state. A server restart mid-push, a recipient closing the app half a second before the message arrived, a phone that has been off for three days while its owner is on a flight: all of these need the message to survive somewhere that is not "a live socket," or it is gone.

**One encrypted blob cannot serve five devices with independent keys.** End-to-end encryption in WhatsApp's design (built on the Signal Protocol, the Double Ratchet algorithm) establishes a separate pairwise cryptographic session per device pair, not per account. There is no shared account-level key the server could use to encrypt once and fan out; encrypting once for "the account" either means the server can read the plaintext (breaking end-to-end encryption entirely) or means only the one device that shares that specific session can decrypt (breaking every other linked device). The naive single-copy design is not just inefficient here, it is architecturally incompatible with per-device end-to-end encryption. It has to become five separate encrypt-and-send operations, one per device, or it cannot be private and multi-device at the same time.

## 3. The architecture

```
[Sender's client: composes one message, but resolves the recipient's
 current device list first — phone + however many companions are linked]
   analogy: before mailing a letter to a household, checking how many
   people actually live there and writing one sealed envelope per person
   |
   v
[Client-side fan-out + per-device encryption: for EACH of the sender's
 own other devices AND each of the recipient's devices, encrypt the
 SAME plaintext separately using that device's own Signal session
 (1:1 chats), or once with the sender's current Sender Key for the
 group, itself distributed to member devices out of band (groups)]
   analogy: one letter, N envelopes, each sealed with a different
   recipient's own personal lock, not one shared building key
   |
   v
[Edge / connection layer: millions of long-lived, persistent, mTLS
 connections, one per logged-in device, terminated by a fleet sized
 in the thousands of cores]
   analogy: a phone exchange that keeps every subscriber's line open
   all day instead of dialing a fresh connection per call
   |
   v
[Chat/relay server tier: lightweight process-per-connection (Erlang
 actor per socket), NOT thread-per-connection — this is the layer
 that let one box hold 2M+ concurrent connections instead of a few
 thousand]
   analogy: a personal desk clerk assigned to every open mailbox for
   as long as that mailbox stays open, cheap enough to assign millions
   |
   v
[Store step — durable write BEFORE any delivery attempt: each of the
 N encrypted copies is written to a durable per-device queue with a
 server-assigned message id, and the server ACKs the sender ("single
 check") only once that durable write has happened]
   analogy: the post office stamping your letter as accepted into the
   system before a mail carrier ever leaves the building with it
   |
   v
[Delivery attempt: if the target device's process/socket is currently
 live, push immediately; if not, the copy just sits in that device's
 durable queue, no different in kind from "about to be pushed," just
 waiting]
   analogy: handing the letter straight to the person if they're home,
   otherwise leaving it in their labeled pigeonhole, not in a general
   pile
   |
   v
[Per-copy acknowledgment, not one shared flag: each device's delivery
 ("double check") and read ("blue double check") state is tracked
 independently, keyed by (message id, device id)]
   analogy: a separate signature card per recipient, not one shared
   sign-off sheet for the whole household
   |
   v
[TTL-bounded offline retention: an undelivered copy sits in a device's
 queue for up to 30 days (WhatsApp's publicly documented retention
 window); if that device never reconnects inside the window, the
 server drops its copy]
   analogy: an unclaimed-parcel shelf that gets cleared out on a
   schedule instead of growing forever
```

Two structural choices are doing the real work here, and they are worth separating from the box diagram.

**Durability is the server's job; privacy is the client's job.** The server holds queued, encrypted-but-opaque-to-it bytes per device. It never sees plaintext and it never decides who reads what; it just guarantees "once accepted, this copy will be handed to that specific device, or it will visibly expire." The client decides who gets a copy at all (device list resolution) and how each copy is sealed (per-device Signal session, or the group's Sender Key). This split is why "the server was compromised" and "a message leaked" are not the same incident for WhatsApp's design; the server's queue is not a decryption point.

**Groups do not scale by pairwise fan-out.** If every message to a 200-person group had to be individually encrypted to every member's every device, a 5-device member turns one group send into up to 1,000 separate encryptions just from that one member's send, and that cost repeats for every message from every sender. Signal's Sender Keys protocol, used by WhatsApp for group messages (Signal itself uses it for large attachments like photos and video), fixes this by distributing a per-sender symmetric key once, pairwise, to each member device when someone joins or the sender rotates the key, and after that, ordinary messages are encrypted **once** with that symmetric key and fanned out as ciphertext, not re-encrypted per recipient. The expensive pairwise step happens on membership change, which is rare; the cheap symmetric step happens on every message, which is constant. That is the same shape as "precompute the expensive thing once, look it up cheaply on the hot path" that shows up in this ledger's search-ranking and recommendation lessons, just applied to cryptography instead of an index.

## 4. The transferable mechanisms

**Store-and-forward, durability before delivery.** The server never attempts a push until the message has already survived a durable write. This is the same ordering discipline as a write-ahead log: commit first, act second, so a crash between the two loses nothing, it just means the "act" step gets retried against a write that already happened. Applied outside messaging: any system that both records an event and tries to notify someone about it should record first, notify second, and treat "notify" as retryable, idempotent, and separable from "record."

**Per-recipient fan-out at the edge instead of a shared broadcast copy.** Encrypting once for a group of recipients and letting the server distribute the single ciphertext is cheaper, but it also means whoever holds that single copy holds something every recipient can open, which is a bigger blast radius if that copy leaks or if you need per-recipient revocation. Fanning out at the sender, one distinct copy per recipient, trades some redundant compute and bandwidth for isolation: compromising or losing one copy does not expose or lose the others. The general lesson is: broadcast is cheap but couples every recipient's exposure together; fan-out is more expensive but keeps recipients' blast radii independent. Pick based on which property you actually need.

**Idempotent, per-copy acknowledgment instead of one shared status flag.** Keying delivery state by (message id, recipient/device id) rather than by message id alone means concurrent acks from different recipients never race each other or overwrite one another's state. This generalizes directly: whenever "did this reach its target" can have more than one target, the acknowledgment key needs to include the target, not just the thing being delivered.

**Lightweight-process-per-connection concurrency instead of thread-per-connection.** The jump from roughly a million to multiple million connections on one WhatsApp box was not new hardware, it was swapping a heavyweight OS-scheduled unit of concurrency (a thread, megabyte stack, kernel-scheduled) for a lightweight, VM-scheduled one (an Erlang process, kilobyte footprint, scheduled cooperatively by BEAM). Any system holding a very large number of mostly-idle, long-lived connections, chat, notification push, live collaboration cursors, hits the same ceiling and has the same fix available: make the per-connection unit of concurrency as cheap as possible, because the bottleneck is idle-connection overhead, not active work.

**A bounded retention window converts "guarantee forever" into "guarantee within a window."** Thirty days of offline queueing, not unlimited, is a deliberate cap: storing undelivered messages for accounts that may never come back again would grow storage without bound. Bounding the guarantee ("delivered, or definitively expired by day 30") is what makes the guarantee affordable to keep at all. This is a pattern worth naming on its own: an unbounded promise is usually not a stronger version of a bounded one, it is a different, much more expensive system, and most products don't actually need the unbounded version.

**Amortize the expensive step, keep the hot path cheap.** Sender Keys pay the pairwise cryptographic cost only when group membership changes, and let every routine message ride on a cheap, already-negotiated symmetric key. This is the same shape as precomputing an index offline and doing cheap lookups online, just applied to key distribution instead of search candidates.

## 5. The trade-offs

**Message acceptance: availability over consistency.** The server accepts and durably queues a send even if it cannot immediately prove delivery to every device. It chooses "never refuse to accept a send" over "block until every recipient has acknowledged," the same choice this ledger's Amazon cart lesson describes for "add to cart": refusing a write because the system cannot instantly guarantee its downstream effect is usually worse for the user than accepting it and resolving the guarantee asynchronously.

**Delivery-state ticks: eventual consistency is fine.** A user seeing a single gray check for a second before it flips to a double check, or a slight delay before a read receipt appears, costs nothing real. This is a UI-visible value that is allowed to be briefly stale because nobody's safety or money depends on the exact millisecond it updates.

**Message confidentiality: never traded for availability.** If a device's Signal session cannot be established (say, a new device's key hasn't been fetched and verified yet), the correct behavior is to block or retry that copy's send, not to silently fall back to a weaker guarantee, such as sending unencrypted or reusing a stale session key, just to keep the "message accepted instantly" promise alive. This is the one place the system is willing to be less available in order to hold a stronger consistency-of-guarantee: it would rather delay a copy than ship a private message insecurely. *(This preference is not from a single WhatsApp-published incident report; it follows directly from how the Signal Protocol's session establishment is documented to work, and is the standard, expected trade-off in that design, so it's labeled here as a reasoned inference rather than a quoted company statement.)*

**Cost vs. latency: storage cost is accepted so the client never has to retry-forever.** Keeping a durable per-device queue for up to 30 days, multiplied by an average fan-out of somewhere between one and five devices per account, is a real, ongoing storage bill. WhatsApp accepts that cost specifically so that the sending client's job stops at "handed off to the server," instead of the client itself needing to hold, retry, and re-encrypt an unsent message for weeks. Pushing durability into the server, rather than onto every client, is a direct cost-for-simplicity trade, and it is also why a message can arrive even if the sender uninstalled the app five minutes after hitting send.

## 6. The systems-thinking lens

The failure mode this architecture has to defend against by design, not just by adding servers, is a **reconnect thundering herd**.

Picture the mechanism: a phone goes offline for a day, a flight, a dead battery, a region-wide carrier outage, and during that day dozens of chats accumulate queued messages across all its devices. The instant connectivity returns, the client doesn't trickle back in, it reconnects immediately and asks for everything at once: every queued message, across every conversation, decrypted and acknowledged as fast as the client can process it. That is one device. Now widen the lens: a whole region's connectivity flaps back after an outage, and millions of devices reconnect within the same few seconds, each one immediately demanding its full backlog. The connection layer, the exact layer tuned to hold millions of *idle* long-lived sockets, now faces millions of simultaneously *active* reconnect-and-drain requests at once. That is a load spike shaped nothing like steady-state traffic; it's correlated by the same external event, which is what makes it dangerous, ordinary spikes are usually somewhat independent across users, but an outage recovery synchronizes them.

The naive response, add more connection-handling capacity, doesn't break the loop, it just raises the threshold at which the next flap re-triggers it. The senior fix breaks the correlation itself:

- **Jittered, staggered client reconnect.** Instead of every client retrying on a fixed interval (which synchronizes retries into repeating waves), each client backs off with randomized jitter, spreading the reconnect storm across seconds instead of concentrating it in one instant.
- **Paginated backlog drain, not a single burst.** A reconnecting device pulls its queued messages in bounded pages, not everything at once, so no single reconnect event demands the server (or the device's own decrypt pipeline) do a huge burst of work in one instant.
- **Admission control at the edge, not deep in the stack.** If the connection layer is saturated, the right place to shed or queue new reconnect attempts is at the edge, before they've consumed a chat-server process and a downstream queue read, not after they've already fanned out work into the storage tier.

The general principle: a feedback loop where recovery from a failure *creates* a new, correlated failure (everyone retries at once, which overloads the thing that was recovering, which causes more retries) is exactly what backoff, jitter, and load shedding exist to break. Buying more servers raises the ceiling on how big an outage has to be before this loop kicks in; it does not stop the loop from existing.

---

## Sources

- [Scaling to Millions of Simultaneous Connections, Rick Reed, Erlang Factory SF 2012 (slides)](https://www.slideshare.net/milkers/scaling-to-millions-of-simultaneous-connections-by-rick-reed-from-whatsapp)
- [How WhatsApp Grew to Nearly 500 Million Users, 11,000 Cores, and 70 Million Messages a Second, High Scalability](https://highscalability.com/how-whatsapp-grew-to-nearly-500-million-users-11000-cores-an/)
- [Erlang Factory SFBay2012 speaker page, Rick Reed](http://www.erlang-factory.com/conference/SFBay2012/speakers/RickReed)
- [WhatsApp is now delivering roughly 100 billion messages a day, TechCrunch, Oct 29, 2020](https://techcrunch.com/2020/10/29/whatsapp-is-now-delivering-roughly-100-billion-messages-a-day/)
- [How WhatsApp enables multi-device capability, Engineering at Meta, July 14, 2021](https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/)
- [WhatsApp Signal Protocol multi-device coverage, InfoQ, July 2021](https://www.infoq.com/news/2021/07/WhatsApp-signal-protocol/)
- [Sender Keys, Wikipedia (overview of the group-encryption protocol used by WhatsApp and Signal)](https://en.wikipedia.org/wiki/Sender_Keys)
- [Private Group Messaging, Signal Blog, Moxie Marlinspike, May 2014](https://signal.org/blog/private-groups/)

---

*Inference vs. fact, stated plainly: the connection counts (1M target, 2M+ achieved, 2.8M peak), the 2014 fleet numbers (500M users, 11,000 cores, 70M msg/s, 32 engineers), the 100 billion messages/day figure, the up-to-four-companion-devices multi-device design, and the client-fanout-plus-Sender-Keys encryption split are all drawn from WhatsApp's or Meta's own public talks, blog posts, and whitepaper. The 30-day offline retention window is WhatsApp's long-documented, widely reported support-facing policy rather than a single primary engineering source. The reconnect-thundering-herd mechanism and the specific claim that confidentiality is never traded for availability during session establishment are this lesson's reasoned inference about how a system with these documented properties would have to behave, not a quoted incident report or company statement.*
