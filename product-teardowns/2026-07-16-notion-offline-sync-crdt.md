# Notion: offline mode and the sync engine (why the server-referee model broke, and the CRDT they bolted on)

Date: 2026-07-16
Product: Notion
Feature: Offline mode and conflict resolution (the "Available offline" toggle, the sync engine, and the CRDT data model behind it)

A note on scope. On 2026-06-25 this ledger tore down Notion's "/" slash command and
the block model it writes into. That report covered blocks as a storage and UI
abstraction, and how sharding by workspace_id let Notion scale the online path.
This report is about a different, harder problem that the same block model runs
into the moment the network goes away: what happens when two people edit the same
page and there is no server in the middle to decide who wins. That is a
concurrency and merge story, not a storage story. It is new ground.

---

## 1. The user

Meet Ananya. She is a product manager in Bangalore. Her entire second brain lives
in Notion: a "Q3 Roadmap" page, a running "1:1 notes" page for each report, a
personal "Reading list," and a shared team wiki. She opens the app forty times a
day. It is the first thing she reaches for when a thought lands.

Now put her on a Tuesday morning flight from Bangalore to Delhi. Wi-Fi is broken.
She pulls out her laptop over Nagpur to jot three roadmap changes before she
forgets them. Or put her on the Bangalore Metro Purple Line, which drops signal in
the tunnel between MG Road and Trinity every single day. Or put her in a client
basement with no bars.

For most of Notion's life, all three of those moments showed her the same thing: a
page that would not load, or an editor that silently refused her keystrokes. The
app that holds her whole brain went blank exactly when she reached for it.

---

## 2. The real problem

Here is the pain, said plainly, the way a friend would say it.

"The app is useless the second my Wi-Fi hiccups. I opened Notion on the plane to
write down three things and it just sat there spinning. I lost the thought. Now I
keep a backup note in Apple Notes for exactly these moments, which kind of defeats
the point of paying for Notion."

That last sentence is the dangerous one. A note app that fails when you pull it out
to take a note trains you to keep a second app. And a second app is the first step
to leaving.

But there is a deeper, hidden problem that only shows up when Notion tries to fix
the first one. Suppose Ananya does edit the roadmap offline for two hours. Meanwhile
her colleague Rohan, sitting online in the office, edits the same roadmap page.
Now there are two different versions of reality that were both "correct" when they
were written. When Ananya lands and reconnects, someone has to merge them. If the
merge is dumb, one of them silently loses two hours of work. For a tool people trust
with their thinking, silently eating work is worse than being offline.

So offline mode is really two problems wearing one coat. Let people work with no
network. Then merge divergent edits back together without losing anyone's work.
The first is plumbing. The second is computer science.

---

## 3. The feature in one sentence

"Available offline" lets you mark specific pages to be fully downloaded to your
device so you can read and edit them with no connection, and a sync engine merges
your offline edits with everyone else's changes when you reconnect.

---

## 4. Jobs to be done

What is Ananya really hiring this feature to do?

- "When I open a page I care about, let me read it and edit it, signal or no signal."
- "Never make me lose a thought because the network blinked."
- "When I get back online, put my changes and my teammates' changes together
  without me having to play detective about what got overwritten."
- "Make the app feel instant even when I am online, so typing never waits on a
  round trip to a server."
- "Let me trust that Notion is the safe place for my brain, so I can finally delete
  the backup note in Apple Notes."

Notice the last two. Retention hides there.

---

## 5. How it works for the user

Ananya opens her "Q3 Roadmap" page while she still has Wi-Fi at the gate. She taps
the three-dot menu at the top right and flips on "Available offline." A small
download happens. The page, and its subpages, are now pinned to her device.

On the plane, with the network dead, the page opens instantly. She edits a to-do
block from "Book flights" to "Book flights and hotel." She reorders two roadmap
items. She adds a new bullet. Everything responds the moment she types. Nothing
spins. There is no "saving" that never finishes.

She lands. The phone finds signal. Without her doing anything, a little "Saving"
indicator flickers and settles. Her three changes are now on the server. Rohan's
change, made while she was in the air, is also there, sitting next to hers. Nothing
was lost. She never opened a merge dialog or picked a winner.

That invisibility is the whole product. The best conflict resolution is the one the
user never notices happened.

Notion shipped this in version 2.53 on 19 August 2025. Before that date, offline
Notion simply did not work in any reliable way, which is why "finally" showed up in
most of the press headlines.

---

## 6. The actual flow, step by step

1. Ananya opens "Q3 Roadmap," taps the three-dot menu, toggles "Available offline."
2. The client downloads every block in that page tree and writes it into a local
   store on her device. Notion has used SQLite for this local cache for years. For
   offline mode they turned that best-effort cache into a durable, tracked store
   that knows which pages are fully present.
3. The page, and every subpage under it, is added to her "offline sync set." The
   toggle is the source of truth for why the page is on her device.
4. Network dies. She edits the "Book flights" to-do block. The client turns that
   edit into one or more small operations, groups them into a transaction, and
   applies the transaction to her local copy immediately. The screen updates before
   anything leaves the device.
5. Because there is no network, the transaction cannot be sent. It is parked in a
   local TransactionQueue. She makes two more edits. Two more transactions stack up
   in the queue, in order.
6. She lands. The client detects connectivity. It drains the TransactionQueue,
   posting the parked transactions to the server's /saveTransactions endpoint, in
   order.
7. The server validates and commits each transaction, then reconciles Ananya's
   offline edits against whatever changed on the server while she was gone
   (Rohan's edit).
8. The server notifies every other client subscribed to that page through a
   WebSocket system called MessageStore. Those clients call back to fetch the
   changed record values and update their own local caches.
9. For every offline page, the client also keeps a lastDownloadedTimestamp. On
   reconnect it compares that timestamp with the server's lastUpdatedTime for the
   page and only pulls pages whose server version is newer. Unchanged pages are not
   refetched.

Steps 4 and 5 are why the app feels instant. Step 7 is where the computer science
lives.

---

## 7. Under the hood, like the engineer

### 7a. The online model, and the one assumption it rests on

Start with how Notion works when you are online, because offline is defined by what
it takes away.

Every piece of content is a block. A block is a small record: a UUID, a type
("to_do", "paragraph", "page"), a properties hash map (the text, the checked state),
an ordered array of child block IDs (its content), and a pointer to its parent.
A page is a tree of these blocks. This is confirmed from Notion's own "data model
behind Notion" post and is the same model the slash-command teardown walked.

When you type, the client does not send your keystroke straight to a database. It
creates operations, groups them into a transaction, and applies that transaction
optimistically to a local cache called RecordCache (an LRU cache sitting on SQLite
on desktop and mobile, IndexedDB on web). The screen updates from the local cache.
Only then does the client POST the transaction to /saveTransactions. The server
validates it, commits it, and fans a notification out to other clients over the
MessageStore WebSocket, which makes them re-sync the changed records. This five-step
pipeline is confirmed from Notion's engineering writing.

Here is the quiet assumption baked into that design: the server is always reachable,
and it is the single referee. Two people editing the same block both send their
transactions to the same server. The server applies them one after another in the
order they arrive. Whoever's transaction lands second wins on any field they both
touched. That is last-writer-wins, and it is fine, because the server picks the
order and everyone hears the result within milliseconds. There is never really any
divergence to resolve, because there is never really any parallel history. The
server linearizes everything.

Walk a concrete conflict online. Ananya and Rohan both edit the "Book flights"
to-do at the same second. Ananya's client sends "set text = Book flights and hotel."
Rohan's sends "set text = Book flights AND car." Both hit the server. The server
applies them in arrival order. Say Rohan's lands last. The block ends as "Book
flights AND car." Ananya's screen, which optimistically showed her version, gets
corrected within a moment when MessageStore tells her client the real value. Slightly
annoying, rarely happens, nobody loses hours of work, because the window of
divergence is milliseconds.

### 7b. Why offline detonates that assumption

Offline removes the referee. Now Ananya's device and the server (carrying Rohan's
edits) both accept writes to the same page for two hours with no communication. When
she reconnects, there are two real histories that ran in parallel. There is no
single arrival order to defer to, because the edits did not arrive at one place.

If you keep last-writer-wins at the transaction or block level here, you get a
disaster. Ananya rewrote a paragraph offline. Rohan reordered the list online. If
"last writer wins" means her whole reconnect batch overwrites the block Rohan
touched, Rohan's reorder vanishes. If it means his wins, her paragraph vanishes.
Someone loses hours. Last-writer-wins is acceptable when the loser loses a
half-second of typing. It is unacceptable when the loser loses a flight's worth of
work.

So the moment Notion decided to let a page diverge for hours, they signed up for
true concurrent merge. That is what CRDTs are for.

### 7c. The fix: migrate offline pages to a CRDT data model

A CRDT (conflict-free replicated data type) is a data structure designed so that
two replicas can be edited independently and then merged into the same result with
no coordination and no central referee. The merge is baked into the data structure's
math, not decided by a server.

Notion's confirmed move: pages marked "Available offline" are dynamically migrated
to a new CRDT data model for conflict resolution. Not the whole workspace. Only the
pages you flagged. That is a deliberate cost boundary, and it is the same trick this
ledger keeps seeing. Netflix only puts a handful of images through its bandit.
Amazon only ranks a few thousand candidates. Notion only pays CRDT overhead on the
small set of pages that actually go offline. The expensive machinery is fenced to
the set that needs it.

Why fence it? Because CRDTs are not free. To merge without a referee, every element
needs a stable unique identity, and deletions usually cannot be true deletions.
They become tombstones (markers that say "this used to exist, do not resurrect it")
so that a delete on one replica and an edit on another do not fight. That metadata
adds up. Running all 200 billion-plus blocks in Notion under full CRDT bookkeeping
would be a lot of tombstones and IDs to carry for pages nobody will ever edit on a
plane. So you only migrate what you must.

### 7d. The two-halves pattern, again, inside a single page

A page is not one kind of data. It is three, and each needs a different merge rule.
This split is the heart of the engineering.

Half one: the text inside a block. This is confirmed to merge automatically. If
Ananya edits paragraph A offline and Rohan edits paragraph B online, both survive.
Even within one paragraph, character-level merge is possible. The right tool is a
sequence CRDT. (Inference, clearly labeled: Notion has not published which one, but
this class of problem is solved by RGA-style or Yjs-style sequence CRDTs, where each
character gets a unique, ordered identifier so two insertions at "position 5" do not
collide, they interleave deterministically.) The concrete payoff: Ananya changes the
to-do text to "Book flights and hotel" while Rohan, in the same block a beat earlier,
fixed a typo three words away. Both edits land. Neither clobbers the other.

Half two: the block tree structure. Reordering, inserting, deleting, and the nasty
one, reparenting (moving a subtree under a different parent). This is the genuinely
hard half, and it is where naive merges produce corruption, not just a lost edit.

Picture the danger. Offline, Ananya drags the "Timeline" block to sit under the
"Q3 Roadmap" heading. Online, at the same time, Rohan drags the "Q3 Roadmap"
heading to sit under the "Timeline" block. Each move is legal on its own. Merge them
naively and you get a cycle: Timeline is under Roadmap, and Roadmap is under
Timeline. A tree with a cycle is not a tree. The subtree can vanish from the page
entirely, or the app can loop forever trying to render it.

This is not hypothetical. Martin Kleppmann and co-authors documented exactly this
class of bug in Google Drive and Dropbox, where concurrent moves of the same folders
led to duplicated or lost subtrees. Their 2021 paper, "A highly-available move
operation for replicated trees," gives a CRDT algorithm that handles arbitrary
concurrent moves, guarantees no cycles are ever introduced, and guarantees all
replicas converge to the same tree, with no synchronous coordination. They proved it
correct in the Isabelle/HOL proof assistant. (Inference: Notion has not said it uses
this exact algorithm, but this is the published state of the art for the precise
problem their block tree faces, and any correct solution must satisfy the same two
invariants: no cycles, and convergence.)

The ordering of siblings inside a parent is a smaller cousin of the same problem.
If Ananya inserts a bullet "between item 2 and item 3" offline and Rohan inserts a
different bullet "between item 2 and item 3" online, both need a home and a stable
order. The clean trick (inference, same one Figma uses for layer order) is
fractional indexing: give each child a fractional position key like a string between
"a2" and "a3," so a new item slots in as "a2m" without renumbering anyone. Two
concurrent inserts get two different keys and simply sort next to each other.

Half three: non-text scalar properties. A "Status" select set to "In progress," a
due date, a relation to another page. These are confirmed NOT to merge. One version
survives, last-writer-wins. And that is the correct call, because a scalar has no
meaningful merge. If Ananya sets Status to "Done" offline and Rohan sets it to
"Blocked" online, there is no sensible blend of "Done" and "Blocked." You pick one.
LWW is right here precisely because merging would invent a value neither person
chose. The engineering maturity is knowing which half deserves the expensive CRDT
and which half deserves cheap LWW.

### 7e. Managing the offline forest without rebuilding it

There is a bookkeeping problem underneath all this. Ananya might mark thirty pages
offline. Those pages form a forest of trees on her device. Pages get added and
removed from that forest as she toggles them, and as sharing and permissions change
on the server. Keeping her local forest consistent with the live workspace, without
rebuilding the whole thing every time, is its own engineering.

Two confirmed mechanisms do the heavy lifting, and both are the "offline-think,
online-lookup" spine this ledger keeps finding.

First, the offline toggle (the offline_action) is treated as the source of truth for
"why is this page on this device." Reconciliation reads from that and adjusts, rather
than recomputing the forest from scratch. Incremental, not full rebuild.

Second, the lastDownloadedTimestamp per page is a cheap delta cursor. On reconnect,
the client asks: for each offline page, is the server's lastUpdatedTime newer than
my lastDownloadedTimestamp? If no, skip it entirely. If yes, pull just that page's
changes. So a reconnect after a flight is not "re-download thirty pages." It is
"re-download the two pages that actually changed." Bandwidth and battery track the
delta, not the size of the offline set. This is the same instinct as Notion only
fetching changed records over MessageStore online, applied to the reconnect path.

### 7f. The scale story at three tiers

Tier one, about 1,000 records, one small page offline, single user. Everything fits
in the SQLite cache. On reconnect you could re-download the whole page and diff it
by hand and nobody would notice. Conflicts are rare because one person rarely
collides with themselves. You barely need CRDTs. This is essentially the old
best-effort RecordCache that Notion shipped for years. It worked because the stakes
and the concurrency were both tiny.

Tier one hundred thousand records, hundreds of offline pages, the same user across a
laptop and a phone, plus a few collaborators. Now the naive approach breaks in two
places. First, you cannot re-download everything on reconnect. That is where
lastDownloadedTimestamp earns its keep, turning a full sync into a delta sync.
Second, real divergence appears, because a page can now sit edited on the laptop for
hours while the phone or a teammate edits it too. This is the tier where you are
forced to move from "best-effort cache plus last-writer-wins" to "durable offline
store plus CRDT merge." The break is not performance. The break is correctness. That
is exactly the jump Notion made to ship 2.53.

Tier ten million-plus records, a real workspace on Notion's 480 logical shards, with
thousands of devices going offline and back. Three things carry the load, all from
the confirmed online architecture. The write path, /saveTransactions, is sharded by
workspace_id, so each workspace's reconnect storm routes to its own shards and does
not contend with anyone else's. The notify path, MessageStore over WebSocket, fans
server-ordered updates out to subscribed clients, so a device coming back online is
brought current by a push, not a poll. And the CRDT metadata cost, the tombstones
and per-element IDs, is bounded by only ever migrating offline-marked pages, plus
(inference) periodic compaction and garbage collection of tombstones once all
replicas have seen a delete, so the bookkeeping does not grow without limit.

The thing that breaks at the next tier is the hot page. Imagine one company wiki
page that five hundred people mark offline and edit. That page is a celebrity. Its
transaction stream and its MessageStore fan-out are hot in a way a private page never
is. The general survival move (inference, but it is the same pattern as Instagram's
Stories fan-out and the hot-key lessons in this repo) is to bound and shard that
fan-out, and to keep the per-page CRDT state compact so that merging a five-hundred-
way-edited paragraph does not walk a mountain of tombstones. The candidate-set trick
returns one more time: the CRDT cost per page is decoupled from the size of the
workspace, because the merge only ever touches the blocks that actually changed on
that one page, never the ten million blocks around it.

---

## 8. The retention and habit mechanic

Offline mode is not an engagement loop. There is no streak, no Monday drop, no
notification pulling Ananya back. It is a defensive retention feature, and it moves
the retention metric by removing a reason to leave.

The mechanic is trust and data gravity. Before August 2025, every dead-signal
moment was a small betrayal. Ananya reaches for Notion in the Metro tunnel, it fails,
and she learns to keep Apple Notes as a backup. Every backup note is a crack. Given
enough cracks, the backup becomes the primary and Notion becomes the thing she is
paying for but not using. TechCrunch and most coverage framed the launch as Notion
finally closing a gap that Apple Notes and Obsidian had held over it for years,
which is another way of saying: this feature was plugging an active leak toward
those competitors.

Close the leak and the gravity flips the other way. Once Notion is reliably there on
the plane and in the tunnel, Ananya deletes the Apple Notes backup. Now her whole
brain, a decade of pages, lives in one place that never fails her. That
accumulated, always-available history is the switching cost. It is the same defensive
mechanic this ledger found in WhatsApp encryption (trust as a moat) and Instagram
photo storage (a decade of memories as gravity). The metric is retention, specifically
the reduction of churn triggered by the app failing at the exact moment of need.

There is a second, sneakier retention gift hiding in the same machinery. The
optimistic local apply and RecordCache that make offline possible are the same thing
that makes online Notion feel instant. Every keystroke commits to the local cache
before any network round trip, so typing never waits. That snappiness is itself
retention. The offline engine and the "it just feels fast" engine are one engine.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable
code, plus an embeddable runtime. A node graph is a collaborative tree-shaped
document, which puts it in exactly Notion's problem class the day two people edit the
same graph, or one person edits it on a laptop that lost Wi-Fi. Take three concrete
lessons.

First, split your document by merge semantics, and pay CRDT cost only where merge is
real. Do not wrap the whole project in one expensive conflict-free blob. A shader
project has the same three halves as a Notion page. The graph structure (add a node,
wire an edge, delete a node, group nodes) is the tree half. Text-like fields (a GLSL
snippet typed into a custom node, a parameter's name) are the sequence half. Scalar
params (a float on a slider, an RGBA color, an enum on a dropdown) are the scalar
half. Give the structure a cycle-safe convergent CRDT, give the text a sequence CRDT
so two people can edit the same expression, and give the scalars plain
last-writer-wins because there is no sane blend of two float values. Trying to CRDT
the sliders would be pure overhead for zero user value.

Second, treat the graph's structural correctness as sacred, more sacred than
Notion's, because your tree has a compiler behind it. When Notion's tree merge goes
wrong and creates a cycle, a page renders badly. When your node graph merge creates
a cycle, the shader fails to compile, and if it is live in an embedded runtime it can
freeze a customer's product. A shader graph is usually a directed acyclic graph, and
a merge that introduces a back-edge is not a cosmetic bug, it is a broken build. So
adopt the Kleppmann move-operation guarantees as a hard requirement: concurrent moves
must never create a cycle and all replicas must converge to the same graph. Do not
hand-roll this. It is the part people get wrong, and there is a formally verified
algorithm to copy.

Third, bound the expensive machinery to an explicit set, and sync by delta, not by
project. Only "shared" or "live-collaborated" projects should carry CRDT metadata and
tombstones. A solo draft stays on a cheap best-effort local cache, exactly like
Notion's pre-2025 model, and only migrates to the CRDT model when it is shared or
marked for offline, exactly like Notion's "Available offline" toggle. And on
reconnect, use a per-project, ideally per-node, version cursor like Notion's
lastDownloadedTimestamp, so you resync only the nodes that changed. This matters more
for you than for Notion, because a shader project drags heavy baggage: compiled
outputs, cached textures, baked lookup tables. You never want a reconnect to re-pull
a hundred megabytes of assets to discover one float moved. Do the heavy analysis at
compile time (your offline-think), and keep the live merge path a bounded incremental
delta over a known-good baseline (your online-lookup). It is the same spine as the
routing-engine-plus-residual lesson from the Uber ETA teardown, pointed at
collaboration instead of prediction.

---

## Sources

Primary (Notion engineering and product):
- Notion, "How we made Notion available offline." https://www.notion.com/blog/how-we-made-notion-available-offline
- Notion, "Exploring Notion's data model: a block-based architecture." https://www.notion.com/blog/data-model-behind-notion
- Notion, "Herding elephants: lessons learned from sharding Postgres at Notion." https://www.notion.com/blog/sharding-postgres-at-notion
- Notion, "The Great Re-shard: adding Postgres capacity (again) with zero downtime." https://www.notion.com/blog/the-great-re-shard
- Notion release notes, "August 19, 2025 - Notion 2.53: Offline mode." https://www.notion.com/releases/2025-08-19

Primary (the tree-move CRDT problem):
- Martin Kleppmann, Dominic P. Mulligan, Victor B. F. Gomes, Alastair R. Beresford, "A highly-available move operation for replicated trees," IEEE TPDS, 2021. https://martin.kleppmann.com/2021/10/07/crdt-tree-move-operation.html
- Reference implementation: https://github.com/trvedata/move-op
- Martin Kleppmann, "CRDTs: The Hard Parts" (talk). https://martin.kleppmann.com/2020/07/06/crdt-hard-parts-hydra.html

Secondary and coverage:
- TechCrunch, "Finally, Notion now works without an internet connection" (20 Aug 2025). https://techcrunch.com/2025/08/20/finally-notion-now-works-without-an-internet-connection/
- XDA Developers, "Notion's offline mode is finally here, but it comes with a strange limitation." https://www.xda-developers.com/notion-offline-mode-launched/
- Educative, "Notion System Design Explained." https://www.educative.io/blog/notion-system-design
- ByteByteGo, "Storing 200 billion entities: Notion's data lake project." https://blog.bytebytego.com/p/storing-200-billion-entities-notions

Fact vs inference note: The block model, the transaction pipeline (operations,
TransactionQueue, /saveTransactions, RecordCache, MessageStore, syncRecordValues),
the SQLite local cache, the 480 logical shards, the "Available offline" toggle, the
lastDownloadedTimestamp delta cursor, the offline_action source of truth, the
migration of offline pages to a CRDT data model, automatic text merge, and
non-text-property last-writer-wins are all confirmed from Notion's own writing and
release notes. The specific CRDT algorithms (sequence CRDT for text, Kleppmann-style
move operation for the tree, fractional indexing for sibling order, tombstone garbage
collection) are inference, clearly labeled where used: Notion has confirmed "a new
CRDT data model" and the merge behavior, but not the internal algorithm choices.
Those inferences are the published state of the art for the exact invariants Notion's
block tree must satisfy.
