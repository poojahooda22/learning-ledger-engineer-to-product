# Figma multiplayer cursors and live presence

Date: 2026-06-18
Product: Figma
Feature: Multiplayer cursors and live presence (the little colored arrows with names that float across the canvas while other people edit)

---

## 1. The user

Meet Priya. She is a product designer at a 30 person startup in Bengaluru. It is
4:45 pm and the founder wants the new checkout screen "looking real" for an
investor call at 6. Priya opens the Figma file. Two other people are already in
it: Rahul the front end engineer, who is poking at the button styles to copy
the exact corner radius, and Anita the PM, who is leaving sticky comments.

Priya does not message them "hey I am in the file now." She does not have to.
The moment she opens the file she sees two little arrows already moving on the
canvas. One is labeled "Rahul" in green, hovering over the pay button. One is
labeled "Anita" in orange, parked near the top. She can literally watch Rahul's
cursor drift to the corner radius field and change it from 8 to 12. Nobody
narrated it. She just saw it happen.

That is the feature. Not the editing. The *seeing each other*.

---

## 2. The real problem

Design used to be a file you owned and then mailed. You worked alone in Sketch
or Photoshop, exported a PNG, dropped it in Slack, and waited. Someone replied
"can you move the button" and you opened the file again, made the change,
re-exported, re-uploaded. The file on your laptop and the file in the engineer's
head drifted apart constantly. The classic disaster: two people both edit "the
latest version," email it back, and now there are two latest versions and nobody
knows which is real.

The pain underneath is not "we cannot edit together." It is "I cannot tell what
you are doing right now, so I either interrupt you or I duplicate your work."
Without presence you are working blind next to other people. You hover over the
same button Rahul is already fixing. You ask "are you in the file?" five times a
day. The friction is not technical, it is social: collaboration without presence
feels like sharing a desk in the dark.

Figma's cursor is the light switch. It makes the other people in the room
visible.

---

## 3. The feature in one sentence

Live multiplayer cursors show every other person's pointer, name, current
selection, and viewport on the shared canvas in real time, so editing together
feels like standing at the same whiteboard.

---

## 4. Jobs to be done

What is Priya really hiring the cursor to do?

- "Tell me who else is here right now, without me asking." (awareness)
- "Show me what they are touching so I do not collide with them." (coordination)
- "Let me follow along when the senior designer is doing the thing I need to
  learn." (the click-on-the-avatar follow mode)
- "Make this feel alive, like a real room, not a dead file." (the emotional job,
  and it is the one that actually sells Figma)

Notice the last one. The cursor moving smoothly across the screen is doing
almost no functional work. It is doing enormous emotional work. A file with
moving cursors feels occupied. That feeling is the product.

---

## 5. How it works for the user

You open a file. In the top right you see a row of colored avatar circles, one
per person in the file. On the canvas you see their cursors: a colored arrow
with a name tag. When Rahul selects the pay button, you see a colored selection
box appear around it in Rahul's color, so you know "that is Rahul's, hands off."

Click on Rahul's avatar and you enter "follow" mode: your viewport locks to his.
When he pans, you pan. When he zooms into the corner radius, you zoom with him.
It is screen sharing without the screen share, and it is built out of the same
presence data that draws the cursor.

When Anita closes her laptop, her cursor and avatar simply vanish. No "Anita has
left" toast, no cleanup. She was a light that was on, and now it is off.

---

## 6. The actual flow, step by step

1. Priya double clicks the file. The browser opens a WebSocket connection to
   Figma's multiplayer server (not many small HTTP requests, one long-lived
   pipe). Source: Figma engineering blog, "How Figma's multiplayer technology
   works."
2. The server hands her the current document and adds her to the list of editors
   connected to this one file.
3. Priya moves her mouse. The client samples the pointer position and sends a
   tiny message up the WebSocket: roughly "user Priya, cursor at canvas
   coordinate (1240, 880)." It sends these at a throttled rate, not on every
   single pixel of mouse movement.
4. The server receives Priya's cursor message and fans it out to everyone else
   connected to this file. It does not save it anywhere. Cursor data is treated
   as ephemeral: never written to the journal, never written to S3 storage.
   Source: Figma blog and Sujeet Jaiswal's infrastructure write-up.
5. On Rahul's screen, his client receives Priya's cursor samples at maybe 30 per
   second over the wire, but redraws the cursor at 60 frames per second by
   interpolating between the last two known positions inside
   requestAnimationFrame. That is why her arrow glides smoothly instead of
   teleporting in jumps.
6. When Priya selects an object, the same channel carries her selection (a list
   of object IDs) and her viewport rectangle. That is what powers the colored
   selection outline and the follow feature.
7. Priya closes the tab. The WebSocket drops. The server notices the
   disconnect and tells everyone else to remove her cursor. Gone.

Concrete walk: the message that draws Rahul's green arrow on Priya's screen is
not stored in the file. If Priya reloads the page, the document comes back from
storage exactly as saved, but every cursor is freshly rebuilt from whoever
happens to be connected at that second. The cursors are a live readout of "who
is here now," nothing more.

---

## 7. Under the hood, like the engineer

This is the heart of it. There are two completely different data problems hiding
inside one feature, and Figma solved them in two completely different ways.

### Problem A: the document (must be permanent and conflict free)

Every Figma document is a tree of objects, very much like the HTML DOM. Source:
Evan Wallace, Figma cofounder, "How Figma's multiplayer technology works." A
frame contains a button, the button contains a text layer and a rectangle. Each
object has a unique ID and a bag of properties: x, y, width, fill color, corner
radius, and a pointer to its parent.

Data structure: a tree of objects, where each object is basically a hash map of
property name to value, plus a parent ID. Concretely, the pay button might be
object `12:5` with properties `{ cornerRadius: 8, fill: #2D7FF9, parent: 12:1 }`.

How do two people editing at once not corrupt this tree? Figma uses an approach
*inspired by* CRDTs (conflict free replicated data types) but it is not a full
CRDT. The rule is property level last writer wins. The server keeps the latest
value any client has sent for a given property on a given object. Source: Evan
Wallace, Figma blog.

Walk the real collision. Rahul sets the pay button corner radius to 12 at the
same instant Priya sets it to 16. Both messages race to the server. The server
processes them in the order they arrive and keeps whichever landed last, say 16.
Both clients converge on 16. No merge math, no vector clocks. The blog says it
plainly: this is like a last writer wins register from CRDT literature, except
"we don't need a timestamp because the server can define the order of events."
The single server process for that document is the referee, so the order is
never ambiguous.

Two changes only conflict if they touch the same property on the same object.
Rahul changing the fill while Priya changes the corner radius on the same button
do not conflict at all. Both stick. That is why editing together feels safe: 99
percent of edits are on different properties or different objects.

Two clever tree problems they had to solve, both real and both in the blog:

- Ordering children. Layers have an order (z order and the layers panel). If you
  store order as integer indexes 0, 1, 2, and two people both insert "at
  position 1," you get a fight. Figma instead stores each child's position as a
  fraction between 0 and 1, and the order is just the sorted positions. To
  insert between two siblings, average their two positions. To avoid running out
  of precision after thousands of inserts, the position is an arbitrary
  precision fraction stored as a string in base 95 (the printable ASCII range),
  not a 64 bit double. Source: Evan Wallace, "Realtime editing of ordered
  sequences." Concrete: dropping a new icon between layers at positions "V" and
  "W" gives it a string position roughly halfway between, like "Vd", forever
  insertable.
- Reparenting without duplication. When you drag the text layer out of button A
  into button B, Figma does not delete and recreate it (that would change its
  ID and lose its history). It just changes the `parent` property on the child.
  Identity is preserved. Source: Figma blog. The edge case: two people reparent
  such that A becomes a child of B and B becomes a child of A at the same time,
  which would make a cycle (an impossible tree). The server detects this and
  rejects one of the changes, popping the offending object out until it lands
  somewhere legal.

Why not Operational Transformation (the Google Docs approach)? Figma says OT did
not fit their problem well. OT shines for linear text where you transform
"insert at index 5" against "delete at index 3." A design document is a tree of
typed properties, not a string of characters, so the last writer wins register
per property is simpler and good enough. Why not a real CRDT? Full CRDTs carry
metadata overhead to merge without a central authority. Figma *has* a central
authority (one server per document), so they took the idea (per property
registers, fractional indexing) and dropped the expensive parts.

### Problem B: the cursors (must be fast and can be thrown away)

Cursors are the opposite kind of data. They must never be saved. They are only
interesting for the half second they are fresh. So Figma runs them on a totally
different track from the document.

- Transport: same WebSocket as the document, but the cursor message is treated
  as ephemeral presence. It is broadcast to the other editors and then dropped.
  Never appended to the journal, never written to S3. Source: Sujeet Jaiswal,
  Figma multiplayer infrastructure write-up, and Figma blog.
- Data structure on the receiving client: a small hash map of user ID to "last
  known cursor state" (position, color, name, selection, viewport). When a
  sample arrives you overwrite that user's entry. When the user disconnects you
  delete the entry. Drawing the cursors is just iterating this small map each
  frame.
- The smoothness trick. The wire rate is throttled (samples arrive at roughly 30
  per second to save bandwidth), but the screen refreshes at 60 per second. If
  you drew the cursor only when a sample arrived, it would stutter. So each
  client stores the last two samples and, inside requestAnimationFrame,
  interpolates the cursor's position between them every frame. Source: secondary
  analyses of the cursor implementation (Mark Skelton, "Building Figma
  multiplayer cursors"; Sujeet Jaiswal). The arrow you see is a smooth lie
  stitched from 30 real data points per second.

Why split them at all? Because mixing them would poison both. If cursors went
through the same durable, ordered, conflict resolved pipeline as document edits,
you would be writing tens of thousands of throwaway points per second to durable
storage and forcing them through conflict resolution they never need. If
document edits went through the cheap fire and forget cursor path, you would
lose people's work. Different durability needs, different pipelines. Same wire.

### The server shape

Clients connect over WebSockets to a cluster of servers. Each document gets its
own dedicated server process, and every editor of that document connects to that
same process. Source: Figma blog and Rust write-up. That single owner per
document is exactly what makes the "server defines the order of events" trick
work, and it is what lets cursor fan out be a simple in-memory loop over the
sockets connected to that one process.

Figma originally wrote this server in TypeScript and rewrote it in Rust. The
reason is directly relevant to this feature: the single threaded TypeScript
server had unpredictable latency spikes, because one slow operation would lock
up the whole worker and freeze syncing for everyone on that document. Cursors
that freeze for 400 ms feel broken even if no data is lost. Rust let them
process work without those stalls. Source: Figma blog, "How Mozilla's Rust
dramatically improved our server side performance."

### The scale story (three tiers)

Tier 1, a small file with 1,000 objects and 3 editors (Priya, Rahul, Anita).
Trivial. The document tree fits in memory easily. Cursor fan out is 3 people
times 30 samples per second, a rounding error. Everything is instant. Nothing
special needed.

Tier 2, a big design system file with 100,000 objects and 50 people in a
workshop. Two pressures appear. First, the document tree is now large, so
sending the whole thing on join is slow; Figma sends it once over the socket and
then only sends deltas (the changed properties), not the whole tree on every
edit. Second, cursor fan out is now 50 senders times 49 receivers times 30
samples per second, on the order of 70,000+ messages per second through one
document's process. This is where throttling the wire rate and keeping cursor
handling as a cheap in memory broadcast (no storage, no conflict resolution)
stops it from melting. The render side survives because each client interpolates
locally, so the wire can stay sparse.

Tier 3, the pathological case, a popular community template or an all hands
FigJam with thousands of people opening the same file. Now the fan out is
quadratic: every cursor must reach every viewer, so N senders times N viewers.
At N = 2,000 that is millions of cursor messages per second through one process,
and the screen would be a useless swarm of arrows anyway. Figma's answer is a
hard cap: it stops rendering beyond roughly 200 visible cursors per file.
Source: Sujeet Jaiswal's write-up. This is the key engineering lesson hiding in
a tiny product decision. Past a couple hundred cursors the feature has zero
added value (you cannot track 2,000 arrows) but quadratic cost, so they simply
refuse to pay for value that does not exist. The document edits still all apply;
only the *presence rendering* is capped.

What breaks at each jump: at tier 2 the naive "send the whole document on every
change" breaks, fixed by deltas. At tier 3 the quadratic cursor fan out breaks,
fixed by the 200 cursor cap plus the single-process-per-document model that
keeps fan out local instead of cross machine.

---

## 8. The retention and habit mechanic

The loop is social presence. You open Figma and someone is already there, a
cursor already moving. That does three things:

1. It pulls you in. An occupied room invites you to join; an empty file does
   not. Seeing Rahul's cursor live makes Priya stay and engage instead of
   exporting a PNG and leaving.
2. It creates a reason to come back at a specific time. "Let us all hop in the
   file at 5" becomes a live session, like a meeting, with the cursors as the
   proof everyone showed up.
3. It removes the need to leave. Because you can see and follow each other, the
   conversation that used to happen in Slack or email stays inside Figma. The
   tool that holds the collaboration holds the user.

Which metric does it move? Primarily activation and retention, and through them,
revenue. The mechanic is the famous one: Figma's growth was driven by
multiplayer turning design from a single player file into a multiplayer team
habit. The cursor is the visible spark of that. Once your whole team's daily
design conversation lives where the cursors are, switching tools means losing the
room, not just the files. That lock in is the business. Sources: Figma's own
multiplayer posts and widely cited product-led growth analyses of Figma.

Real observed example: the "follow" feature (click an avatar, ride their
viewport) turned design reviews and onboarding into a live guided tour. A new
hire follows the lead designer's cursor through the file and learns the system in
minutes. That is a retention feature dressed up as a cursor.

---

## 9. The lesson for Rare.lab

Rare.lab is a node based shader editor that compiles to shippable code, with an
embeddable runtime. The direct lesson: split your collaborative state into two
pipelines by durability, exactly as Figma split document edits from cursors, and
never let the cheap one touch the expensive one.

Concretely for Rare.lab:

- The node graph is your "document." Treat each node and each wire as an object
  with an ID and a hash map of properties (a node at position, with its shader
  parameter values, and a parent group). Resolve concurrent edits with property
  level last writer wins under a single authoritative session owner per graph.
  Two artists tuning two different uniforms on the same node must both stick;
  only a true same-property collision falls back to last writer wins. You do not
  need full CRDTs if you keep one server (or one host peer) as the order
  referee, and that one decision deletes a mountain of merge complexity.
- Live cursors, live parameter scrubbing, and "what node is this person hovering"
  are your ephemeral presence. Run them on a separate fire and forget channel,
  never written to the saved graph, throttled on the wire (around 30 per second)
  and interpolated to 60 fps in your render loop. Here is the Rare.lab specific
  twist: a shader preview is already redrawing every frame on the GPU, so when a
  collaborator drags a "glow intensity" slider, do not re-run your conflict
  pipeline per frame. Stream the scrub value down the ephemeral channel, let
  each viewer's runtime apply it straight to the live uniform, and only commit
  the final value to the durable graph on mouse up. You get buttery live preview
  of someone else's edit at zero storage cost, and a single clean commit when
  they let go.
- Steal the cap. Past a few hundred live participants, presence rendering is
  quadratic and worthless. Cap visible cursors, keep applying the actual graph
  edits. Pay only for value that exists.

The one line: keep the permanent thing (the graph) correct and ordered through
one referee, keep the live thing (cursors and scrubbing) cheap and disposable,
and let the GPU's existing per frame redraw carry the smoothness so collaboration
costs you almost nothing.

---

## Sources

- How Figma's multiplayer technology works, Figma engineering blog (Evan
  Wallace): https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
  (author mirror: https://madebyevan.com/figma/how-figmas-multiplayer-technology-works/)
- Realtime editing of ordered sequences (fractional indexing), Figma blog (Evan
  Wallace): https://www.figma.com/blog/realtime-editing-of-ordered-sequences/
- How Mozilla's Rust dramatically improved our server side performance, Figma
  blog: https://www.figma.com/blog/rust-in-production-at-figma/
- Multiplayer editing in Figma, Figma blog:
  https://www.figma.com/blog/multiplayer-editing-in-figma/
- Making multiplayer more reliable, Figma blog:
  https://www.figma.com/blog/making-multiplayer-more-reliable/
- Figma: Building Multiplayer Infrastructure for Real-Time Design Collaboration,
  Sujeet Jaiswal: https://sujeet.pro/articles/figma-multiplayer-infrastructure
- Building Figma multiplayer cursors, Mark Skelton:
  https://mskelton.dev/blog/building-figma-multiplayer-cursors
- So you want to build Miro and Figma style collaboration (ephemeral presence),
  zknill: https://zknill.io/posts/ephemeral-collaboration/
