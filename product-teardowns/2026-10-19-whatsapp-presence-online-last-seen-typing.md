# WhatsApp presence: how "online", "typing...", and "last seen today at 9:41" stay almost free at two billion people

How the little grey line under a contact's name is produced and delivered. This is
the live presence signal (online / typing / recording / last seen), not the blue
double ticks of delivery and read receipts (that was its own teardown on 2026-06-15),
not the voice note (2026-10-05), and not Figma's in-document cursors (2026-06-18),
which are a different problem on a different product.

Date: 2026-10-19
Product: WhatsApp
Feature: Presence (online status, "typing..." indicator, and "last seen")

## 1. The user

Ananya is in Pune, sitting on her bed at 9:40 on a Tuesday night. She wants to ask her
older brother Vikram, who is in Bengaluru, whether he booked the train tickets for
Diwali. She opens his chat. Under his name it says "online". Good, he is awake and on
his phone. She types "tickets done?" and sends it. Three dots appear: the word
"typing..." shows up under his name. She waits, because she can see he is in the middle
of answering. Twelve seconds later his reply lands. The whole thing felt like talking,
not like leaving a note in a mailbox.

The next morning she opens the same chat to ask one more thing. This time it says "last
seen today at 7:12 am". He is not on the app right now, so she decides to send the
message and not call. That one grey line changed her behavior: whether she calls, waits,
or lets it go.

## 2. The real problem

Messaging has a coordination problem that plain text does not solve. When you send a
message you do not know if the other person is there. You stare at the screen wondering:
did it even reach them, are they ignoring me, are they asleep, should I call instead?
That uncertainty is uncomfortable, and it makes people either spam follow-ups ("??",
"you there?") or give up.

Presence fills that gap with one honest signal. "Online" means they are on the app right
now, so a reply might be seconds away. "Typing..." means hold on, an answer is literally
being written. "Last seen today at 7:12 am" means they are not here now, so do not expect
an instant reply. The honest pain it removes, said like a friend: "I just want to know if
you are around before I decide whether to wait, nudge, or call."

There is a second problem that is pure engineering, and it is the real reason this
feature is interesting. Presence changes constantly. Every time Vikram opens the app,
closes it, locks his phone, or taps a key, his status changes. Multiply that by two
billion people, each with dozens or hundreds of contacts who might care. If WhatsApp
naively told all of your contacts every time your status flickered, the amount of
"Vikram is online now, Vikram is offline now, Vikram is typing" traffic would dwarf the
actual messages. So the real problem is also: show a live, constantly changing status to
the people who care, right now, without that firehose of updates melting the servers.

## 3. The feature in one sentence

A thin, live, best-effort status line under a contact's name that says whether they are
on the app right now, whether they are typing to you this second, and if not, roughly
when they were last around.

## 4. Jobs to be done

- "Tell me if it is worth waiting." Ananya sees "online" and decides to wait for a reply
  instead of calling.
- "Tell me an answer is coming." The "typing..." line stops her from firing off a second
  message or assuming she is being ignored.
- "Tell me not to expect much right now." "Last seen at 7:12 am" sets her expectation so
  she is not refreshing the chat for an hour.
- "Make the conversation feel alive." Seeing the other person react in real time turns a
  slow message exchange into something closer to a conversation.
- "Let me hide, both ways." Vikram, who finds "last seen" stressful, can turn his off,
  and WhatsApp's rule is that if he hides his, he also loses the ability to see other
  people's. The job here is control over a signal that can cause real social anxiety.

## 5. How it works for the user

Ananya opens Vikram's chat. The status appears under his name at the top. If he has the
app open and in the foreground, she sees "online". The instant he starts typing a reply
to her, "online" is replaced by "typing...". If he is recording a voice note instead, it
says "recording audio...". When he puts his phone down and the app goes to the
background, the line changes to "last seen today at 7:12 am", a timestamp of the last
moment he was active, as long as his privacy settings allow it.

She never sees a status for a contact whose chat she has not opened. The home screen list
of chats does not show a dozen little "online" dots next to everyone. Presence is
something she sees when she is actually looking at one person's conversation. And it is
reciprocal and private: Vikram can set "last seen" and "online" visibility to Everyone,
My Contacts, or Nobody, and if he hides his own last seen, WhatsApp hides everyone
else's from him too.

## 6. The actual flow, step by step

1. Ananya taps Vikram's chat to open it.
2. Her phone, on opening that specific chat, tells the server "I am now interested in
   Vikram's presence", a subscribe request for just him.
3. The server checks whether Vikram is currently connected and allowed (by his privacy
   setting) to be shown to Ananya. If he is online, it sends her phone a tiny "Vikram is
   available" update, and her screen shows "online". If he is offline, it sends his last
   seen timestamp, and her screen shows "last seen today at 7:12 am".
4. Vikram taps a key to start replying. His phone sends a small "composing" signal to the
   server, debounced so it is not one signal per keystroke.
5. The server looks up who is currently subscribed to Vikram's presence (Ananya is, right
   now, because she has his chat open) and pushes "Vikram is typing" to her. Her screen
   swaps "online" for "typing...".
6. Vikram stops typing for a few seconds, or sends the message. His phone sends a "paused"
   or "active" signal, the "typing..." line disappears, and his actual message arrives.
7. Vikram locks his phone. The app moves to the background and the phone sends an
   "unavailable" signal (or the connection simply goes idle and the server notices). The
   server records the current time as Vikram's new "last seen" value and pushes
   "unavailable" to his current subscribers.
8. Ananya closes the chat or leaves the app. Her phone drops the subscription to Vikram's
   presence, so the server stops sending her his updates. If she never reopens, the
   subscription also expires on its own after a while.

## 7. Under the hood, like the engineer

This is the heart of the report. Presence looks like the simplest possible feature, a
single word under a name, and it is one of the sneakiest scaling problems in all of
messaging. The reason is fan-out. A message goes from one person to one person (or one
small group). A presence change can potentially concern everyone who knows you. Get the
design wrong and presence traffic buries the actual messages.

First, a note on what is confirmed and what is inference, because WhatsApp has published
a lot about its scale and almost nothing about presence specifically.

Confirmed from primary sources: WhatsApp's server backend runs on Erlang/OTP (the BEAM
virtual machine) on FreeBSD, and the whole engineering culture is built around holding a
massive number of long-lived connections cheaply. In a 2012 post ("1 million is so
2011") WhatsApp reported pushing a single server past 2 million simultaneous TCP
connections on FreeBSD 8.2 on a 24-core Xeon X5675, with CPU and memory still to spare.
By a 2015 FreeBSD Foundation testimonial they were running between 2 and 3 million
concurrent TCP connections per FreeBSD server. In Rick Reed's 2014 Erlang Factory talk
("That's 'Billion' with a 'B'") the fleet was over 8,000 cores pushing more than 70
million Erlang messages per second, running Erlang R16B01 with their own BEAM patches on
FreeBSD 9.2, using Mnesia for in-memory state, with on the order of 150 chat servers each
holding roughly a million phones, and peak traffic around 342,000 inbound and 712,000
outbound messages per second.

Inference, clearly labeled: the WhatsApp wire protocol is a heavily modified, binary
XMPP (community reverse-engineering calls it FunXMPP), so the presence model almost
certainly mirrors XMPP's. In XMPP, presence is a publish-subscribe broadcast: a bare
presence stanza marks you available, an "unavailable" type marks you offline, chat states
like "composing" (XEP-0085) carry the typing indicator, and "last seen" has its own
extension (XEP-0318). Third-party WhatsApp client libraries and the system-design
community consistently report one crucial behavior: you must explicitly subscribe to a
contact's presence, the server then sends you their updates, and the subscription stops
after a while if not renewed. I treat the XMPP lineage and the subscribe-on-demand detail
as well-grounded inference, not as published WhatsApp internals.

### The one decision that makes this whole feature possible: subscribe on demand

Here is the naive design almost everyone writes first. Keep a list of each user's
contacts. Whenever a user's status changes, push that change to every contact. Simple,
and it detonates.

Walk the arithmetic, using numbers from the system-design literature. Take just 100,000
concurrent users. Say each has 50 contacts who could see their presence, and each user's
status flips about 20 times an hour (open app, lock phone, type, stop typing, and so on).
That is 100,000 times 50 times 20, which is 100 million presence events an hour, for a
tiny slice of the user base, carrying zero real conversation. Now imagine a popular person
with 2,000 contacts: every time they glance at their phone, 2,000 push notifications of
"online" go out. This is a fan-out bomb.

WhatsApp's answer (and the standard answer for this class of problem) is to flip the
model. Do not broadcast presence to the whole contact graph. Instead, push a person's
presence only to the people who currently have that person's chat open on their screen.
When Ananya opens Vikram's chat, her phone subscribes to just Vikram. When she closes it,
she unsubscribes. At any instant, the set of people watching Vikram's presence is tiny,
usually a handful, often zero. If nobody has Vikram's chat open, nobody needs his "online"
or "typing", so the server sends nothing at all.

This single scoping decision is the difference between a fan-out that melts and one that
is nearly free. Presence stops being "tell everyone who knows you" and becomes "tell the
two people looking at your chat right this second". The 100-million-events-an-hour monster
collapses to almost nothing, because the vast majority of status changes have no live
audience.

### The data structures actually in play

- A connection registry: a hash map from user id to the live connection, think
  `userId -> connectionProcess`. On WhatsApp each phone holds one long-lived TCP socket to
  a chat server, and in Erlang each socket is handled by its own lightweight process (an
  actor). So the registry maps Vikram's id to the Erlang process holding his socket. Looking
  up "where is Vikram connected" is an O(1) hash lookup, not a scan. This is exactly the
  kind of in-memory routing table Mnesia is used for.
- A subscriber set per user: for each user, the small set of connections currently
  interested in their presence, think `userId -> {set of watcher connections}`. When Ananya
  opens Vikram's chat, her connection is added to Vikram's watcher set. When she leaves, it
  is removed, or it ages out. This set is almost always tiny. Publishing a presence change is
  "walk this small set and send each a stanza", an O(number of current watchers) operation,
  where that number is usually single digits.
- A single "last seen" timestamp per user: one value, `userId -> lastActiveAt`, overwritten
  each time the user goes offline. This is the only piece of presence that is durable. Online
  and typing are never stored; they are transient.
- The presence stanza itself: a tiny struct, a few bytes, carrying `{userId, state}` where
  state is available, unavailable, composing (typing), recording, or paused. No payload, no
  history. It is the lightest message the system sends.

### Push versus pull: the quiet trick inside the trick

Notice two different delivery styles living in one feature.

"Online" and "typing..." are push. They change many times a second across the fleet and
must feel instant, so the server pushes them to the small live subscriber set the moment
they happen. They are also lossy on purpose: if a "typing" stanza is dropped under load,
nothing breaks, the indicator just does not show for a moment. Presence is best-effort,
never journaled, never retried. That lossiness is what makes it cheap. Contrast this with
an actual chat message, which must be durably stored offline and retried until delivered
(the delivery-receipt machinery from the 2026-06-15 teardown). Presence gets to be sloppy
precisely because losing it costs nothing.

"Last seen" is pull. It is a single stored timestamp that only matters when someone opens
your chat and finds you offline. So it is read lazily, on demand, at the moment Ananya
opens Vikram's chat and he is not online, subject to his privacy setting. It is not pushed
to anyone, because a timestamp that is not changing does not need a live stream. Splitting
the hot, changing signal (push) from the cold, rarely-read one (pull) keeps the push path
carrying only what is genuinely live.

### The typing indicator, specifically

"typing..." is a chat state (XEP-0085's "composing" in XMPP terms). Two engineering details
make it behave well. First, it is debounced on the phone: it does not send a signal per
keystroke, which would be a storm, it sends one "composing" when Vikram starts and
suppresses the rest. Second, it self-expires: if Vikram stops typing and does not send,
after a few seconds the phone sends "paused" (or the indicator just times out on the
watcher's side), so you never get stuck showing "typing..." forever for someone who walked
away mid-sentence. The indicator is a small state machine, not a durable fact.

### Where the work happens

Almost nothing durable happens on the server per presence change. The phone decides its own
state (foreground means available, a keystroke means composing) and sends a tiny signal. The
server's job is a fast lookup (who is watching this person) and a fan-out to a tiny set. The
only write is overwriting one "last seen" timestamp. There is no database row per "online",
no history of every time you opened the app. Keeping presence stateless and transient is
what lets it ride on top of a messaging core already running at 70 million Erlang messages a
second without doubling the load.

### The scale story at three tiers

What grows here is not a catalog of items. It is the number of concurrent online users and,
far more dangerously, the fan-out of presence updates per change.

Tier 1, about 1,000 users, a chat app you build this weekend. Anything works. Keep one
hash map of `userId -> socket` in a single process, and when someone's status changes, just
loop over their contacts and tell everyone. Even the naive full-contact-graph broadcast is
fine at this size; 1,000 users flipping status a few times a minute is nothing. You would not
even bother with subscribe-on-demand yet, though it is the pattern that survives to tier 3.

Tier 2, about 100,000 concurrent users, a real regional product. The naive broadcast is the
first thing that breaks, and it breaks loudly, the 100-million-events-an-hour calculation
above. The survival move is subscribe-on-demand: push presence only to users who have the
subject's chat open right now. The second thing that breaks is that one server can no longer
hold every connection, so you run several chat servers, and now a watcher and the person
they are watching can be on different servers. You need a cross-server routing table: a
registry that knows which server holds Vikram, so a "Vikram is typing" event on his server
can be routed to Ananya's server. In WhatsApp's world this is the meta-clustering and Mnesia
routing layer. You also move the connection and subscriber maps fully into fast in-memory
state, because a disk round trip per presence lookup would never keep up.

Tier 3, two billion users with hundreds of millions online at once, the real WhatsApp.
The walls, and the moves:

- Holding the connections at all is a wall. Each of hundreds of millions of online phones
  needs a live socket, each nearly idle. The move was years of tuning the BEAM VM and the
  FreeBSD kernel (and using kqueue-style event handling) so a single box holds 2 to 3 million
  mostly-idle TCP connections cheaply, with an Erlang process per connection costing very
  little memory. About 150 chat servers each holding around a million phones is the confirmed
  shape. Presence lives on top of these same sockets, so it costs no new connections.
- Fan-out is the wall that would kill presence specifically. At two billion users, a popular
  account could have tens of thousands of contacts, and broadcasting presence to all of them
  on every flicker is unthinkable. Subscribe-on-demand is the survival move, made back at
  tier 2 and absolutely required here: presence goes only to the handful of people actively
  viewing your chat, so a celebrity's "online" reaches maybe five watchers, not fifty
  thousand. The fan-out is bounded by who is looking, not by who knows you.
- Cross-server routing is a wall. With the fleet sharded across 150+ servers, Vikram's typing
  event has to find Ananya wherever she is connected. The move is an in-memory distributed
  registry (Mnesia plus Erlang process groups) that maps users to their current server and
  routes small presence stanzas between servers. Because the stanzas are tiny and lossy, this
  inter-server chatter stays light.
- Load spikes are a wall. On a big event (a cricket final, a festival, New Year midnight),
  hundreds of millions of people open the app in minutes, a storm of "available" and "typing".
  The move that saves you is that presence is best-effort: under pressure you can throttle or
  drop presence updates without losing any real message, because presence is never journaled.
  The system sheds the cheap, lossy traffic first and protects the durable message path. This
  is the same load-shedding instinct seen in the Stripe rate-limiting teardown (2026-09-01),
  applied by making presence deliberately droppable.

Through-line: presence is engineered to be cheap by being transient, lossy, pull-where-possible,
and above all scoped to a live audience of nearly zero. The hard part was never computing a
status; it was refusing to deliver it to anyone who is not looking right now.

## 8. The retention and habit mechanic

Presence is one of the most powerful engagement loops in all of messaging, and it works
partly through a little anxiety. The loop is the check-back. Ananya sends "tickets done?",
sees "online", and now she keeps the chat open, watching, waiting for "typing..." to appear.
If she sees "last seen 2 minutes ago" but no reply, she wonders why, and she comes back to
check again. The status line turns a one-shot message into a reason to keep opening the app.

Which metric it moves: retention and session frequency (daily active use and opens per day),
not revenue directly, since WhatsApp does not charge for it. Presence makes conversations feel
synchronous, which pulls both people to reply faster, which keeps them both in the app longer,
which is the whole game for a messenger whose value is how many of your conversations live
inside it.

A real observed signal of how potent and sensitive this is: WhatsApp built an entire layer of
privacy controls around presence, letting you set "last seen" and "online" visibility to
Everyone, My Contacts, or Nobody, and enforcing a reciprocity rule where hiding your own last
seen hides everyone else's from you. You only build that much control, and that much "you lose
it if you hide yours" pressure, around a signal people feel strongly about. The reciprocity
rule is itself a retention nudge: most people want to see others' last seen, so they leave
their own on, which keeps the signal flowing across the network and keeps the check-back loop
alive for everyone. The feature that could make people anxious is kept switched on by making
its visibility a two-way street.

## 9. The lesson for Rare.lab

Treat live, ephemeral state as a separate, lossy, subscribe-on-demand channel, and scope every
live update to exactly what a viewer is looking at right now. WhatsApp's entire presence design
survives two billion users on one decision: never stream a changing signal to anyone who is not
actively watching the thing that changed. That principle maps straight onto both halves of
Rare.lab, the collaborative node editor and the embeddable runtime, where everything connects to
scalability and performance.

1. Split durable graph state from ephemeral live state, and make the live state lossy. In the
   collaborative editor, the saved shader graph (nodes, edges, parameters) is the durable,
   must-never-be-lost layer, the equivalent of a chat message. But live collaboration signals
   (who is editing which node this second, a live uniform value being dragged, a collaborator's
   cursor, a preview frame) are presence: transient, best-effort, never journaled. Build them on
   a channel you are allowed to drop under load without corrupting anything, so a spike in live
   activity sheds cheap updates and never threatens a save. This is WhatsApp protecting the
   message path by making presence droppable.

2. Subscribe to live updates only for what is on screen, and unsubscribe the moment it scrolls
   off. Do not stream every node's live state to every collaborator. A collaborator viewing one
   corner of a 2,000-node graph should receive live updates only for the nodes currently in their
   viewport, and should subscribe or unsubscribe as they pan and zoom, exactly as Ananya's phone
   subscribes to Vikram only while his chat is open. This is viewport culling applied to realtime
   sync, and it bounds fan-out by who is looking, not by graph size, so a huge graph with many
   editors does not become a fan-out bomb.

3. Carry the same rule into the embeddable runtime. On an end user's device, only run and update
   effects that are actually visible: cull offscreen passes, pause shaders for elements scrolled
   out of view, and subscribe to any live or networked uniforms only while the effect is on
   screen. At ten million end-user installs, each device paying only for its visible effects
   rather than the whole scene is the difference between a runtime that is cheap and one that
   drains batteries and melts low-end GPUs.

4. Separate push from pull. Push the tiny, fast-changing signals (someone is editing here, this
   value is moving) to the small live audience. Pull the expensive or rarely-read state (the full
   contents of a node, the last-saved timestamp, a heavyweight preview) lazily, on demand, when a
   viewer actually focuses it, the way "last seen" is read only when someone opens your chat.
   Keep the hot realtime path carrying only what is genuinely live and genuinely watched.

One line: WhatsApp makes a signal that changes billions of times a second nearly free by making
presence transient, lossy, pull-where-it-can, and above all delivered only to the handful of
people looking at your chat this instant; build Rare.lab's live collaboration and its embeddable
runtime the same way, stream ephemeral state on a droppable channel scoped tightly to what each
viewer and each device is actually looking at, and never let a live signal fan out to anyone who
is not watching.

## Sources

- "1 million is so 2011," WhatsApp engineering blog (single FreeBSD 8.2 box past 2 million TCP
  connections on a 24-core Xeon X5675): https://blog.whatsapp.com/1-million-is-so-2011
- Rick Reed, "That's 'Billion' with a 'B': Scaling to the Next Level at WhatsApp," Erlang Factory
  SF Bay 2014 (8,000+ cores, 70M+ Erlang messages/sec, Erlang R16B01 + BEAM patches, FreeBSD 9.2,
  Mnesia): https://www.erlang-factory.com/sfbay2014/rick-reed
- Rick Reed, "Scaling to Millions of Simultaneous Connections," Erlang Factory SF 2012 (starting
  ~200k connections/server, goal 1M, BEAM and FreeBSD kernel tuning):
  https://www.erlang-factory.com/conference/SFBay2012/speakers/RickReed
- "How WhatsApp Grew to Nearly 500 Million Users, 11,000 cores, and 70 Million Messages a Second,"
  High Scalability (write-up of the 2014 talk: ~342k inbound / 712k outbound messages/sec, ~150
  chat servers each ~1M phones, Mnesia):
  https://highscalability.com/how-whatsapp-grew-to-nearly-500-million-users-11000-cores-an/
- FreeBSD Foundation testimonial, WhatsApp (December 2015; 2 to 3 million concurrent TCP
  connections per FreeBSD server): https://freebsdfoundation.org/testimonial/whatsapp/
- XEP-0085: Chat State Notifications (the "composing" typing-indicator state and its paused /
  active / inactive / gone siblings): https://xmpp.org/extensions/xep-0085.html
- XEP-0318: Last User Interaction in Presence (the "last seen" style last-presence-change time):
  https://xmpp.org/extensions/xep-0318.pdf
- "User Presence System Low-Level Design: Online Status, Heartbeat, and Real-Time Fan-out,"
  TechInterview (the 100k-users x 50 contacts x 20 changes/hour = 100M events/hour fan-out
  arithmetic, and the subscribe-to-the-open-chat scoping fix):
  https://www.techinterview.org/post/3233468381/lld-user-presence/
