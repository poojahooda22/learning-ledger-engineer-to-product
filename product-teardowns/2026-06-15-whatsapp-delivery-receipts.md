# WhatsApp Delivery Receipts (the ticks): a teardown

Date: 2026-06-15
Product: WhatsApp
Feature: Message delivery and read receipts (one grey tick, two grey ticks, two blue ticks)

---

## 1. The user

Priya is 27, in Pune, standing on a crowded platform at Shivajinagar at 9:10 pm.
She just left a friend's place and texts her flatmate Arjun: "reached station,
starting in 5, keep the door open." Arjun is on the Mumbai local heading home,
phone in pocket, signal flickering between two bars and zero as the train goes
through the tunnel near Sandhurst Road.

Priya is not reading a manual or thinking about distributed systems. She is doing
one tiny human thing: she wants to know if the message landed. Did it go? Did his
phone get it? Did he actually see it? She glances at the little check marks under
her message the way you glance at a traffic light. One grey tick. Then, a beat
later, two grey ticks. Ten minutes later, when Arjun surfaces from the tunnel and
opens the chat, two blue ticks.

That whole emotional arc, sent, delivered, seen, is carried by three tiny glyphs.
That is the feature.

## 2. The real problem

Texting into silence is stressful. Before receipts, every messaging app had the
same quiet anxiety: you send something important ("I am running late", "did you
transfer the rent?", "are you okay?") and then you stare at the screen with no
idea what happened. Did it fail? Is their phone off? Are they ignoring me? Should
I call?

The pain is not "I want a status icon." The pain is uncertainty about another
human being. A friend would describe it like this: "I texted her and heard
nothing for an hour and I was going slightly mad wondering if she even got it."
SMS gave you almost nothing. A delivery report on SMS was rare, slow, carrier
dependent, and never told you the person actually read it.

WhatsApp's ticks remove that uncertainty in stages. They turn an invisible
network event into something you can see and trust.

## 3. The feature in one sentence

A small set of check marks under each message that tells the sender, in near real
time, whether the message left their phone, reached the other phone, and was
opened by the other person.

## 4. Jobs to be done

What is Priya really hiring the ticks to do?

- "Tell me my message actually left my phone and is not stuck." (one grey tick)
- "Tell me it reached their device, so I know the ball is in their court."
  (two grey ticks)
- "Tell me they have actually seen it, so I can stop wondering and decide whether
  to call." (two blue ticks)
- "Do all of this without me having to ask 'did you get my message?', which is
  itself an annoying extra message."

The deeper job: reduce the anxiety of one-way communication and give the sender a
sense of control over a situation they cannot see.

## 5. How it works for the user

Three states, stacked in order. Each one is a promise about where the message is.

- One grey tick: the message left Priya's phone and was accepted by WhatsApp's
  servers. It has not reached Arjun's phone yet. (Per the WhatsApp Help Center,
  one tick means sent.)
- Two grey ticks: the message has been delivered to Arjun's phone. His device has
  it, even if the app is closed and the screen is locked. He has not necessarily
  looked.
- Two blue ticks: Arjun opened the chat and the message was shown on his screen.
  This is the actual "read receipt." Everything before it is delivery, not
  reading.

There is also a clock icon: message is still on Priya's phone, not even sent yet
(no signal, airplane mode). And read receipts are a two-way privacy deal: if Arjun
turns off "read receipts" in settings, he stops sending blue ticks, but he also
stops seeing other people's blue ticks. Voice notes are the one exception; their
blue ticks always show.

Group chats change the rule. In a 40-person flatmates-plus-friends group, two
grey ticks appear only when the message has been delivered to everyone in the
group, and two blue ticks appear only when every single member has read it. Long
press the message and tap "Info" to see the per-person breakdown of who got it and
who read it and when.

## 6. The actual flow, step by step

Walk Priya's "reached station" message tap by tap.

1. Priya types and hits send. Instantly a clock icon shows. The message is sitting
   in her phone's local database (SQLite on the device), queued to go out.
2. Her phone has signal. It pushes the message over its already-open encrypted
   socket to the nearest WhatsApp server. The server accepts it and sends a small
   acknowledgement back to Priya's phone. Her clock turns into one grey tick.
3. The server looks up Arjun. Right now his train is in the tunnel. No live
   connection. The server parks the message in Arjun's offline queue and keeps it.
4. Arjun's train clears the tunnel near Dockyard Road. His phone reconnects. The
   server sees the connection come alive, pushes the queued message down to his
   device, and his phone stores it and fires back a "delivered" receipt.
5. The server relays that "delivered" signal to Priya. Her one grey tick becomes
   two grey ticks. She relaxes a little: it is on his phone now.
6. Arjun pulls out his phone, taps the chat open. The message renders on screen.
   His device fires a "read" receipt. The server relays it to Priya. Her two grey
   ticks turn blue. She stops worrying and goes to find an auto.

Notice the shape: every tick is a round trip. Priya's phone never talks to Arjun's
phone directly. Each glyph is a tiny status message that traveled phone, to server,
to other phone, and a confirmation that traveled back the same way.

## 7. Under the hood, like the engineer

This is the heart of it. The ticks look trivial. The machinery that makes them
fast and reliable for billions of people is not.

### The backbone: store and forward over persistent connections

WhatsApp was built on Erlang/OTP, on top of a heavily modified version of the
ejabberd XMPP server. The wire protocol is a custom, compressed binary variant of
XMPP that the community reverse-engineered and named "FunXMPP" (the name appears
in early decompiled WhatsApp Android clients). XMPP already had a notion of
message receipts; the open standard is XEP-0184, "Message Delivery Receipts,"
where a sender attaches a request element and the recipient returns a received
element carrying the same message id. WhatsApp's tick system is the same idea,
hardened and extended to three states.

The core pattern is store and forward, the same idea as a post office. Priya's
phone does not hold an open connection to Arjun's phone. Both phones hold a
long-lived encrypted socket to WhatsApp's servers. The server is the post office
in the middle:

- Receive the message from the sender.
- Look up where the recipient is (which server, which process, or "offline").
- Forward it if the recipient is connected, or store it if not.
- Delete it once the recipient confirms receipt.

Rick Reed, a WhatsApp engineer, gave talks at Erlang Factory (2012, "That's
'Billion' with a 'B'", and a companion talk on scaling connections) describing how
they tuned the FreeBSD kernel and the Erlang runtime to hold on the order of two
million-plus simultaneous TCP connections on a single server. That number matters
for receipts: a tick is only fast if the recipient's socket is already open and
the server can write to it in milliseconds.

### The data structures actually in play

Think about what each piece of the tick journey needs.

1. Routing table: "where is Arjun right now?" This is a key-value lookup. Given a
   user id, return either a live location (server and process handle) or "offline."
   WhatsApp used Mnesia, Erlang's built-in distributed, in-memory database, for
   this kind of session and routing state. Conceptually it is a hash map keyed by
   user id, replicated across nodes so a lookup is fast and survives a node dying.
   A hash map is the right pick because the only operation that matters is "look up
   one user by id," and that needs to be O(1), not a scan.

2. The offline queue: a durable per-recipient queue. When Arjun is in the tunnel,
   Priya's message goes into Arjun's queue. A queue is exactly right because order
   matters (messages should arrive in roughly the order they were sent) and you
   drain it front to back when he reconnects. This queue is persisted and
   replicated to a backup so a server crash does not lose Priya's message.
   Undelivered messages are held for up to 30 days, then dropped. (The 30-day
   retention is widely reported; the exact storage layout is not fully public, so
   treat the "it is a replicated per-user FIFO queue" description as well-grounded
   inference from the XMPP/ejabberd lineage rather than a published spec.)

3. The pending-ack table: when the server hands the message to Arjun's phone, it
   does not immediately forget it. It keeps the message in a "waiting for
   acknowledgement" state, keyed by message id, and retries if the ack does not
   come. Once Arjun's device acks, the server deletes the message. This is the
   classic reliable-delivery pattern: do not throw away the letter until the
   recipient signs for it.

4. Message id: every message carries a unique id. All three receipts (server ack,
   delivered, read) reference that id, so Priya's phone knows exactly which bubble
   to upgrade from one tick to two to blue. Without a stable id you could not match
   a "read" signal to the right message in a fast-moving chat.

### Walk one message through the machine

Priya sends "reached station" with message id `3EB0...A91`.

- Her phone writes it to local SQLite, shows a clock.
- Phone sends it up the socket. Server receives, assigns it to Arjun's mailbox,
  and writes a server ack back down to Priya. Her client flips id `3EB0...A91` to
  one grey tick.
- Lookup in the routing map: Arjun = offline. Server enqueues `3EB0...A91` in
  Arjun's offline queue, replicated to a backup node.
- Arjun reconnects. Server drains his queue, writes `3EB0...A91` to his socket,
  moves it into pending-ack state.
- Arjun's phone stores it, sends "delivered" for `3EB0...A91`. Server deletes it
  from the queue and relays "delivered" to Priya. Her client flips `3EB0...A91` to
  two grey ticks.
- Arjun opens the chat. His phone sends "read" for `3EB0...A91`. Server relays it.
  Priya's client flips `3EB0...A91` to blue.

Each flip is a separate, tiny, idempotent status event keyed by the same id.

### The encryption wrinkle

Since 2016 WhatsApp messages are end-to-end encrypted with the Signal protocol.
The server cannot read "reached station." But receipts are metadata, not content.
The "delivered" and "read" signals are about the envelope, not the letter, so the
server can still route and relay them without ever decrypting the body. This is
why the ticks keep working even though the company genuinely cannot read your
chats.

### The scale story at three tiers

The interesting thing about receipts is that the naive version works fine small
and quietly explodes large.

Tier 1, around 1,000 messages. Trivial. One server. Keep an in-memory map of "who
is online," forward messages, relay acks. A single Postgres table and a websocket
loop would do it for a college project. Nothing breaks.

Tier 2, around 100,000 messages and tens of thousands of users online. Now two
things start to hurt. First, connection count: holding tens of thousands of open
sockets per box pushes you into kernel and runtime tuning (file descriptors,
memory per connection). This is exactly the problem WhatsApp solved with FreeBSD
tuning and Erlang's cheap lightweight processes, one process per connection.
Second, the routing lookup ("where is this user?") cannot be a scan; it has to be
a hash lookup in a shared, replicated store. A single database doing a row lookup
per message is already feeling warm. The fix is in-memory routing state (Mnesia
style) plus durable per-user queues for the offline case.

Tier 3, 10 million plus, and really the WhatsApp reality of 100 billion-plus
messages a day across 2 billion-plus users. Here the receipts problem becomes a
fan-out and amplification problem, and group chats are the killer.

- Amplification: one message can produce up to three receipt events (server ack,
  delivered, read). So 100 billion messages a day is not 100 billion network
  events; it is several times that just for the ticks. Each receipt is itself a
  routed message that has to find the original sender and update their client.
- Group fan-out: a single message to a 256-person (now up to 1,024) group is not
  one delivery. It is up to 1,024 separate deliveries, each of which can generate a
  delivered receipt and a read receipt. And the group-level rule ("two grey ticks
  only when everyone has it, blue only when everyone has read it") means the server
  or client has to aggregate N individual receipts before it can flip the sender's
  single visible tick. That is a counting problem: track a set of "who has
  delivered" and "who has read" per message, and only promote the visible state
  when the set is complete. Naive fan-out here is the thing that breaks at this
  tier.

How they survive it: shard users across many servers so no single box owns too
many people; keep routing and session state in memory with replication so the
"where are you" lookup never hits slow disk; use durable per-user queues so an
offline recipient does not block or lose anything; delete messages the instant
they are acknowledged so storage does not grow without bound; and lean on Erlang's
model of millions of tiny cheap processes so each connection and each in-flight
message is isolated and a failure in one does not stall the rest. The genius is
that the hot path stays simple: receive, look up, forward, ack, delete. The same
five steps whether 1 million or 1 billion people are online.

(Confirmed from public sources: Erlang/OTP, ejabberd lineage, Mnesia, FunXMPP,
the two-million-connections-per-server figure, 30-day undelivered retention,
Signal end-to-end encryption, and the three-state tick meanings. Inference, clearly
labeled: the exact internal layout of the offline queue and the group receipt
aggregation are not fully published, so those are described as the standard way
this class of problem is solved given WhatsApp's known stack.)

## 8. The retention and habit mechanic

The ticks are a quiet retention engine, and they work on both people in the
conversation.

For the sender, blue ticks create a small obligation loop. Once Priya sees Arjun
read her message and not reply, the silence becomes socially loud. That pressure
pulls people back into the app to follow up, and pulls the reader back to respond
so they are not "the person who left it on read." The feature manufactures
back-and-forth.

For the reader, the two-way privacy rule is a clever lock-in. To hide your own
blue ticks you must give up seeing everyone else's. Almost nobody takes that deal,
because knowing whether your message was read is too valuable. So the default
(read receipts on) sticks, and the anxiety-and-reassurance loop stays switched on
for the whole network.

The metric it moves is retention and engagement, specifically messages sent per
user and daily opens. Receipts make every conversation feel live and accountable,
which is exactly what keeps a messaging app open many times a day. A real, observed
example is the widely documented "left on read" social phenomenon: the blue ticks
turned "did you see my message" into a visible, emotionally charged event, to the
point that articles, relationship advice, and an entire vocabulary grew around it.
That cultural weight is free engagement; the product made checking the app feel
necessary.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime. The WhatsApp lesson is about how you
report progress on slow, invisible work.

A shader graph compile, an asset bake, or a remote render is exactly like Priya's
message into the tunnel: the user fires it off and then stares at nothing,
wondering if it is stuck, working, or dead. Do not make them wonder. Model the job
as an explicit state machine with a stable id and report each transition the way
WhatsApp reports ticks: queued (clock), compiling/uploading (one tick), built and
delivered to the runtime (two ticks), live and rendering on the target device
(blue). Each state is a cheap status event keyed by the job id, relayed back to the
editor, not the heavy artifact itself.

Two concrete carryovers, both biased toward performance and scale:

1. Separate the status channel from the payload, like receipts versus message body.
   When a designer publishes a graph to 50 embedded runtimes (50 game clients, 50
   ad placements), do not wait on the big compiled blob to know how things are
   going. Fan out lightweight "received / running" acks per target, aggregate them
   into one visible status the way group chats aggregate per-member ticks, and only
   show "live everywhere" when the set is complete. The heavy compiled shader
   travels once and gets cached; the status is tiny and travels constantly.

2. Acknowledge then delete. On the publish path, keep a compiled artifact in a
   "pending ack" state until the runtime confirms it loaded and rendered a first
   frame, then you can safely retire the previous version. That gives you reliable
   rollouts (no half-updated clients) and bounded storage, the same discipline that
   lets WhatsApp delete a message the instant it is delivered instead of hoarding
   billions of them.

The headline: invisible work needs visible, staged, cheap status. Three glyphs
removed a global wave of anxiety. A clear job state machine can do the same for
every designer waiting on a Rare.lab build.

---

## Sources

- WhatsApp Help Center, read receipts and message ticks (single, double, blue):
  https://faq.whatsapp.com/665923838265756
- "How WhatsApp Works: Architecture Deep Dive on 100 Billion Messages," GetStream:
  https://getstream.io/blog/whatsapp-works/
- "How WhatsApp Handles 40 Billion Messages Per Day," ByteByteGo:
  https://blog.bytebytego.com/p/how-whatsapp-handles-40-billion-messages
- "Designing WhatsApp," High Scalability:
  https://highscalability.com/designing-whatsapp/
- "Erlang in Large-Scale Messaging Systems (e.g. WhatsApp)," Software Patterns Lexicon:
  https://softwarepatternslexicon.com/erlang/case-studies-and-practical-applications/erlang-in-messaging-systems-e-g-whatsapp/
- "WhatsApp Erlang Architecture, Scaling to 2 Billion Users," scalewithchintan:
  https://scalewithchintan.com/blog/whatsapp-erlang-architecture-2-billion-users
- XEP-0184: Message Delivery Receipts (the XMPP receipt standard WhatsApp's scheme resembles):
  https://xmpp.org/extensions/xep-0184.html
- FunXMPP in early decompiled WhatsApp client (community reverse engineering):
  https://github.com/JaapSuter/Niets/blob/master/external/decompiled/android/WhatsApp_2_0_7.src_fun_xmpp/WhatsApp_2_0_7.src/com/whatsapp/client/FunXMPP.java
- "WhatsApp Read Receipts Explained," Blueticks Blog:
  https://blueticks.co/blog/whatsapp-read-receipts-explained
