# Figma multiplayer: the sync engine that merges two people editing the same object

Date: 2026-10-04
Product: Figma
Feature: The multiplayer sync and conflict-resolution engine (how concurrent edits to the same document get merged without stepping on each other)

A note on scope. This ledger already covered Figma's **multiplayer cursors and live presence** (2026-06-18), which is the ephemeral layer: the little colored arrows flying around that nobody saves. This report is the other half, the part that actually matters for your work: the engine that takes two people changing the same rectangle at the same time and produces one document both of them trust. Cursors are the fireflies. This is the ground truth underneath them.

---

## 1. The user

Priya is a product designer at a 30-person startup in Bangalore. It is 4:50pm. The founder pings her: "can we see the new onboarding screen before the 5pm call?" The screen is one frame in a Figma file called `Onboarding v3`, which already has about 1,200 layers in it.

She opens the file in Chrome. No app to install, no file to download to her Mac, no "someone else has this file open, open read-only?" dialog. The founder clicks the same link from his phone on the way to the meeting room. A second later Priya sees a cursor with his name on it land on the Sign In button. He drags the button two pixels to the right. She, at the same instant, changes that exact button's fill from blue to green. Neither change is lost. The button is now green AND two pixels to the right, for both of them, with no refresh and no "merge conflict" popup.

That last sentence is the whole report. Everything below is how that one thing is true.

---

## 2. The real problem

Here is the pain, described like a friend would.

For 30 years, design files were single-player. You had a `.sketch` or `.psd` or `.ai` file sitting on one person's laptop. If two people needed to work on it, you did one of two miserable things. Either you took turns ("don't touch it, I have it open"), or you both edited copies and then somebody spent an hour by hand merging `Onboarding_final_FINAL_priya_v2.sketch` back into the founder's version, eyeballing which rectangle moved. Tools like Dropbox and Abstract tried to put "version control" on top of binary design files, but a design file is not code. You cannot do a line-by-line three-way merge on a blob of shapes. So the honest answer was: last person to save wins, and everybody else's work silently dies.

The real problem is not "we want cute cursors." The real problem is **two people must be able to touch the same file at the same time and have both of their changes survive**, with no lock, no save button, no handoff, no lost work. That is hard because the two edits can collide in ugly ways: same object, same property, same second. Somebody has to decide who wins, and decide it in a way that never produces a broken or duplicated document.

---

## 3. The feature in one sentence

Figma syncs a shared design file by treating the document as a tree of objects, each with a bag of properties, and resolving concurrent edits with **per-property last-writer-wins where "last" means last to reach a single central server**, so two people editing the same object merge cleanly without the complexity of Google Docs style operational transforms.

---

## 4. Jobs to be done

What is the user really hiring this engine to do?

- "Let me and my teammate work on the same screen at the same time without talking about who has it open."
- "Never make me lose work because someone else saved after me."
- "Never show me a corrupted file, a duplicated layer, or a 'resolve conflict' dialog. I am a designer, not a git user."
- "Let me paste a link in Slack and have anyone open it instantly in a browser, no install."
- "When my wifi drops in the elevator, let me keep working and quietly catch up when it comes back."

Notice the jobs are mostly about things NOT happening: no lock, no loss, no conflict screen. A sync engine succeeds by being invisible, same invisible-craft trust as Spotify loudness (2026-08-28) or Netflix picture quality (2026-09-11). You only ever notice it when it fails.

---

## 5. How it works for the user

The visible experience is almost nothing, which is the point.

- You open a file link. It loads in the browser in a second or two.
- Other people's cursors appear with their names. You see their selection highlights.
- When someone moves or edits something, it just changes on your screen, smoothly, within a fraction of a second. No refresh.
- You never press Save. There is no Save. The file is always saved.
- If two of you grab the same layer, the last little change to land is the one that sticks, per property, so your color edit and their move edit both survive because they touched different properties of the same object.
- If your connection drops, you keep editing. When it returns, your screen reconciles to match everyone else, with the server's copy as the truth.

---

## 6. The actual flow, step by step

Walking Priya's 4:50pm session, click by click.

1. Priya clicks the `Onboarding v3` link in Slack. Her browser opens Figma and a WebSocket connection is made to a Figma multiplayer server.
2. The server that owns this specific document sends her a full copy of the current document: the whole tree of ~1,200 objects, each with its properties. Her browser now holds the document in memory.
3. The founder opens the same link. His browser connects to **the same server process** (this matters, see under the hood) and also downloads a copy.
4. The founder drags the Sign In button 2px right. His client immediately moves it on his own screen (optimistic local update) and sends a tiny message to the server: "object `btn_signin`, property `position`, new value X."
5. At nearly the same moment Priya changes the same button's fill to green. Her client updates her screen instantly and sends: "object `btn_signin`, property `fillColor`, new value green."
6. The server receives both messages, in whatever order they arrive. It applies each one to its own authoritative copy. The two messages touch **different properties** of the same object, so there is no contest: position gets the founder's value, fillColor gets Priya's value.
7. The server broadcasts each change to everyone else connected. Priya gets "position = X" and her button jumps 2px right. The founder gets "fillColor = green" and his button turns green. Both documents now match the server. Both edits survived.
8. Now the harder case: suppose they had BOTH dragged the button at the same time (both editing `position`). The server applies them in arrival order. The second one to arrive wins and is the value broadcast to everyone. The button ends up in exactly one place for everyone. It never ends up as two buttons, and it never disagrees between the two screens. Somebody's drag is quietly overruled, but the document stays sane. That is the deliberate trade.
9. Priya's wifi blips for 4 seconds. Her client keeps letting her edit locally. On reconnect, her client re-syncs against the server, the server's state wins where they disagree, and she is back in lockstep.
10. Nobody ever pressed Save. The server periodically persists the document so it survives the server restarting.

---

## 7. Under the hood, like the engineer

This is the heart. The canonical primary source is Figma CTO Evan Wallace's own writeup, "How Figma's multiplayer technology works" (also hosted on the Figma blog). I will mark clearly what is confirmed there versus what is reasonable inference about the class of system.

### The document is a tree of objects, each a bag of properties

CONFIRMED. A Figma document is a tree. Every object (a frame, a rectangle, a text layer, a component instance) has a unique ID and a map of properties. Priya's Sign In button is one object, say `btn_signin`, with properties like `{ position, width, height, fillColor, cornerRadius, parent, ... }`. A frame like `Login Screen` is an object whose children are other objects.

Why a tree of property-maps and not, say, one big array or a geometry blob? Because the unit of editing is "one property of one object." A designer action is almost always "change property P of object O to value V." If the document is a tree of objects each holding an independent map of properties, then an edit is a tiny addressable message: `(object_id, property_key, new_value)`. That addressability is what makes clean merging possible at all. You cannot cleanly merge two edits to a flat binary `.sketch` blob because you cannot name the thing that changed. Here you can always name it.

Concrete data structure: think of it as a hash map from `object_id` to an object, where each object is itself a hash map from `property_key` to `value`. Lookups and updates are O(1). A single edit mutates one entry in one inner map.

### The one big decision: reject operational transforms, lean on the central server

CONFIRMED and this is the famous part. The "standard" way to do realtime collaboration, popularized by Google Docs, is **operational transforms (OT)**. OT is a set of rules for transforming one person's operation against another's so they can be applied in different orders on different machines and still converge. OT is powerful and OT is notoriously hard to get right. Evan Wallace's stated reasoning: as a startup Figma wanted to ship, and OT was "unnecessarily complex" for their problem.

The other famous family is **CRDTs** (conflict-free replicated data types), which are data structures designed so that peers can merge with no central authority at all. Figma's design is often described as "inspired by CRDTs but simpler" (HelloInterview frames it as "simplified CRDTs"). The simplification is the key insight:

> A real CRDT has to handle peers merging in any order with no coordinator, so it carries timestamps, vector clocks, and tombstones to make merges commutative. Figma does not need any of that, because **there is a central server and every edit already flows through it**. If the server decides the order, you can throw away the clocks and most of the bookkeeping. "Last writer wins" where **last means last to arrive at the server**.

This is the whole trick. CRDTs and OT both pay a heavy tax to work without a central authority. Figma has a central authority (one server owns each document, see below), so it refuses to pay that tax. The server is the single point that linearizes edits. Order is just arrival order at the server.

INFERENCE, clearly labeled: so the merge rule per property is a last-writer-wins register where the "timestamp" is simply the server's sequence of arrival. No Lamport clocks needed because the server IS the clock.

### Per-property last-writer-wins, and why it feels magical

CONFIRMED in spirit. Last-writer-wins is applied **per property, not per object**. This is why Priya's color edit and the founder's move edit both survived in section 6. They wrote different properties (`fillColor` vs `position`) of the same object `btn_signin`. Two writes to different properties never contest. Only two writes to the *same* property of the *same* object contest, and there the later arrival wins.

Walk it concretely. Two messages reach the server:

- `(btn_signin, fillColor, green)` from Priya
- `(btn_signin, position, X)` from the founder

These land in two different slots of `btn_signin`'s property map. Both stick. Broadcast both. Done. No conflict, because the resolution granularity is the property, and real human edits rarely hit the exact same property of the exact same object in the exact same instant. When they do (both dragging the button), last-to-arrive wins and the document stays consistent. The cost is that one person's simultaneous same-property edit is silently overruled, which Figma judged an acceptable and rare trade for enormous simplicity.

### The genuinely hard part: reparenting and ordering

CONFIRMED that Evan Wallace calls reparenting (moving an object to a new parent in the tree) the most complicated part of the system. Two sub-problems:

**(a) Parent and position must be one atomic property.** An object stores which parent it belongs to AND where it sits among that parent's children. These two must be a single property that updates together. Reason: if you stored "parent" and "position within parent" separately, a race could leave you using a position value that belonged to the old parent after the parent link already changed. Bundling them means an object can never be half-moved.

**(b) The server rejects reparenting cycles.** Picture two people at once: Priya drags frame A into frame B, the founder drags frame B into frame A. Apply both naively and A is inside B while B is inside A. The pair becomes a little ring that is detached from the document tree, a cycle, which is not a tree anymore and is a corrupted file. CONFIRMED: the server rejects any parent-property update that would create a cycle. The central server, because it sees every edit in order, is the perfect place to enforce this invariant. A peer-to-peer CRDT would have to detect and repair the cycle after the fact. The central server just refuses the second move. This is a concrete example of why having the authority is worth so much: tree-shape invariants are trivial to enforce at one chokepoint and painful to enforce without one.

### Fractional indexing: how two people reorder siblings without fighting

CONFIRMED and this is my favorite detail. Inside a parent, children have an order (z-order, or list order). How do you store "this layer is third"? If you store integer indices 0,1,2,3 and someone inserts between items 1 and 2, you have to renumber everything after it, and two people inserting at once produces a renumbering war.

Figma's answer: an object's position among its siblings is a **fraction between 0 and 1, exclusive**. Children are sorted by this fraction. To insert an object between two siblings at positions 0.5 and 0.75, you give it 0.625, the midpoint. Nobody else has to be renumbered. Two people inserting in the same gap at the same time pick slightly different fractions and both land, in some order, with no collision and no renumber.

CONFIRMED refinement: Figma uses arbitrary-precision fractions, not 64-bit floats, because repeatedly splitting a gap would exhaust float precision. They encode these fractions as strings in base-95 (the printable ASCII range) for compactness, and sort them as strings. So "position" is really a short string key like a point in a dense ordered space, and "insert between" is always "find a key that sorts between these two," which always exists because the space is dense. Figma wrote this up separately in "Realtime editing of ordered sequences."

Concrete: the `Login Screen` frame has children ordered by keys. The logo is at `"V"`, the title at `"V5"`, the button at `"W"`. Priya drags a new subtitle between the title and the button. Her client picks a key that sorts between `"V5"` and `"W"`, say `"V8"`, and sends it. No other child's key changes. The founder, simultaneously dropping a divider in the same gap, picks `"Vf"`. Both land. Both ordered correctly. Zero renumbering.

### The server is deliberately dumb

INFERENCE, well grounded in the confirmed design. The sync server does not need to understand what a rectangle is, or how to render Bezier curves, or how Auto Layout reflows. It stores objects and properties and it enforces a few invariants (ordering, no cycles). Evan Wallace notes the server can sync properties it does not understand, which means **clients can add new property types without deploying a new server**. The heavy, product-specific brains (rendering, the C++/WebAssembly canvas engine covered on 2026-07-09, layout, vector math) all live in the client. The sync server is a small, fast, general-purpose merge-and-broadcast machine. Keeping it dumb is what lets it be tiny and fast, which sets up the scale story.

### The Rust rewrite: the performance story that made per-document isolation possible

CONFIRMED, from Evan Wallace's 2018 post "Rust in production at Figma." The multiplayer server was first written in TypeScript on Node.js. Two problems bit hard:

1. **Latency spikes.** Node is single-threaded. One slow operation (or a garbage-collection pause) locks up the whole worker until it finishes, so syncing had unpredictable latency spikes. For a realtime tool, a random 200ms freeze is very visible.
2. **Memory overhead blocked isolation.** They wanted to give each document its own isolated process so one heavy document could not hurt others. But a separate Node.js process per document was impractical because the JavaScript VM's per-process memory overhead was too high. You cannot run tens of thousands of Node VMs on a box.

They rewrote the server in Rust. Reported result: roughly an order of magnitude improvement in server-side performance, dramatically lower memory, and about 10x faster serialization. Crucially, Rust's low per-task memory footprint made **per-document isolation actually affordable**, so each document could live in its own lightweight unit without the JS VM tax. This is the same lesson as other teardowns in this ledger: the engine you pick IS the performance ceiling (Canva's rasterizer swap, 2026-09-29). Here the language runtime was the ceiling.

### The scale story at three tiers

What grows here is not a catalog of items. It is **(a) the size of a single document in objects, (b) the number of concurrent editors on one document, and (c) the number of documents open across the whole fleet.** These three scale very differently, and the third one hides a nasty wall.

**Tier 1: a 1,000-object file, 2 or 3 editors.** Honestly, almost anything works. Hold the whole document tree in memory on one server. On every edit, you could even re-broadcast a big chunk of state and nobody would notice. Per-property LWW and fractional indexing are arguably over-engineering at this size, but you build them anyway because they are the code path that survives to Tier 3. A single-threaded Node server is completely fine here. This is Priya's file at 1,200 layers: trivially inside Tier 1.

**Tier 2: a 100,000-object file, dozens of editors.** Now the shortcuts break.
- Re-sending big chunks of state on every edit saturates the WebSocket and lags everyone. Fix: send only tiny per-property deltas, `(object_id, property_key, value)`, never the whole tree after the initial load.
- Concurrent edits to the same objects become common (a design critique with 20 people poking at one screen), so you genuinely need per-property LWW to merge, and fractional indexing so simultaneous reorders do not renumber-war.
- The single-threaded server's latency spikes start to show. A GC pause during a busy critique is now a felt stutter. This is exactly the pain that pushed Figma off Node.
This is the tier where the architecture earns its keep. Below it you could fake collaboration; at this size you need the real engine.

**Tier 3: millions of documents open across the fleet, plus the occasional monster single file.** CONFIRMED architecture: the multiplayer service runs on a fixed number of machines, each with a fixed number of workers, and **each document lives on exactly one worker.** So a worker owns some fraction of all currently-open documents. This is sharding, but the shard key is the document, and it falls out naturally from "one central authority per document." Routing a client to the right worker is just "look up which worker owns this doc id." This is the same shard-by-tenant move as Notion by workspace (2026-06-25) and Stripe by account (2026-09-01): here the tenant is the document.

Three walls appear at this tier and here is what each one costs and how Figma survives it:

- **Memory per document.** Every open document's full tree sits in its worker's RAM. Millions of open docs means you cannot afford fat per-doc overhead. Survival: Rust's small footprint, and only loading a document into a worker while it is actually open.
- **One slow document hurting its neighbors.** On single-threaded Node, a heavy document sharing a worker could freeze others. Survival: the Rust rewrite made cheap per-document isolation possible, so documents stop interfering.
- **The un-shardable monster file. This is the real ceiling.** A single document lives on ONE worker by design, because that worker is the central authority that orders its edits. So a 2-million-object file with 80 people editing it at once is a vertical scaling problem: you cannot split that one document across two machines without reintroducing exactly the multi-authority coordination (OT/CRDT merge across servers) that the whole design was built to avoid. The survival move is to make the single-document path as cheap as humanly possible (dumb server, tiny deltas, Rust, no geometry parsing on the server) and to push all the expensive per-object work (rendering those 2M objects) onto the client's WebAssembly canvas engine. The sync server never has to render or understand the 2M objects; it only merges property writes. That separation is what keeps even a huge file's server cost small.

The through-line across all three tiers, and across this whole ledger: the realtime hot path stays a tiny lookup-and-merge (`object_id -> property_key -> value`, apply, broadcast), and everything expensive (rendering, layout, compile) is kept off that path. Offline-think, online-lookup, applied to a document instead of a search index.

---

## 8. The retention and habit mechanic

The sync engine is not just a feature. It is Figma's growth engine, and the mechanic is a loop, not a notification.

The loop: you are working in a file and you need another human's eyes or hands. You copy the link and paste it in Slack. Your teammate clicks. Because Figma runs in the browser, there is no install gate, they are in the file in two seconds. They see your cursor move live. They make one edit and watch it appear on your screen. **That moment, watching a stranger's change land on your own document in real time, is the activation event.** They are now, functionally, a Figma user, and the next time they need to design or comment, they open Figma, because that is where the shared truth lives.

Which metric does it move? All three, in sequence, which is rare:
- **Activation:** the first "aha" is seeing live co-editing work, not finishing a design.
- **Viral acquisition:** every shared link is an invite. Multiplayer made the file itself the growth channel. This is widely credited as a core reason Figma won against Sketch, which was a single-player desktop app where collaboration meant mailing files around.
- **Retention and revenue:** once a team's shared source of truth lives in Figma, leaving means migrating everyone, so the shared file is a switching-cost moat, and seats convert to paid.

Real observed example: the standard behavior of a design review where the PM, during a live call, renames a button's label directly in the file while the designer watches, instead of writing "change 'Sign In' to 'Log In'" in a comment thread that gets actioned next week. The feedback-to-change latency collapses to zero. Teams that experience that once do not go back to mailing files. The engine's correctness (never losing an edit, never a conflict dialog) is what makes people trust it enough to work *in* the file together rather than *around* it. The trust-cracker is the mirror image: one lost edit, one corrupted file, one "two copies of my layer" bug, and the whole "we can all just work in here" promise cracks, which is why the cycle-rejection and atomic-reparent invariants are not nice-to-haves, they are the product.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based editor for shaders and visual effects that compiles to shippable code, plus an embeddable runtime. A node graph is structurally the same thing as a Figma document: **a tree (really a graph) of objects, each node a bag of properties (inputs, params, position, connections).** So the moment two people want to co-edit a shader graph, or even one person edits on a laptop and tweaks on a tablet, you face the exact problem this teardown is about. The concrete, actionable lessons, biased to scalability and performance:

1. **Do not reach for OT or a full CRDT. Put a central authority per graph and use per-property last-writer-wins.** For a shader graph where the edits are "change this node's `blurRadius` to 8" or "connect output A to input B," property-granular LWW with the server as the order is almost certainly enough, and it is an order of magnitude less code than OT. Reserve CRDT complexity for the genuinely peer-to-peer case, which you probably do not have. Spend the saved complexity on the compiler instead.

2. **Store node ordering and list positions with fractional indexing, not integer indices.** The instant two people reorder passes in a render stack, or drop nodes into the same group, integer indices force renumbering wars. A dense ordered key (a base-N string between two neighbors) means an insert touches exactly one node and never collides. This is cheap to build now and miserable to retrofit later.

3. **Keep the sync server dumb and keep compilation off the realtime path.** This is the sharpest performance lesson. Figma's sync server does not render or understand geometry. Rare.lab's sync server must NOT compile shaders or run the GPU preview. The realtime layer moves tiny property deltas and merges them. Compiling the graph to shippable code and running the 60fps preview is a separate pipeline (async, like Canva export on 2026-09-29, or client-side like Figma's WASM canvas). If you let a graph edit trigger a compile on the sync hot path, one expensive compile stalls every collaborator. Separate the merge engine from the compile engine with a hard wall.

4. **Pick the runtime that makes per-graph isolation affordable, and learn Figma's Node-to-Rust lesson before you pay for it.** One heavy graph should never be able to freeze other people's sessions. That requires cheap per-document isolation, which requires a low-overhead runtime (Rust, or WASM workers), exactly the constraint that forced Figma off Node. Choose that substrate for the collaboration server from the start.

5. **Recognize the un-shardable-single-graph wall early and design the ceiling in.** One graph = one authority = one worker, by the same logic that gives you clean merges. So a giant 10,000-node megagraph with 50 editors is a vertical scaling problem you cannot shard away without giving up the simplicity. Plan for it: make the single-graph path as lean as possible (dumb server, tiny deltas, no compile on the hot path), and push the expensive per-node work (cost estimation, preview rendering) onto the client and the offline compiler, so even the monster graph costs the server almost nothing. The server's job is to merge small writes, not to understand your shaders. Guard that boundary and the engine scales from a two-person doodle to a studio's shared effect library on the same code path.

One line: Figma won multiplayer by refusing the hard algorithms, putting one central authority per document and resolving edits with per-property last-writer-wins plus fractional indexing, keeping the server dumb and fast so the real work stays on the client; build Rare.lab's graph collaboration the same way and never let a compile touch the realtime merge path.

---

## Sources

- Evan Wallace (Figma CTO), "How Figma's multiplayer technology works," madebyevan.com and the Figma blog. https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
- Evan Wallace, "Rust in production at Figma" (the TypeScript to Rust multiplayer server rewrite, latency spikes, per-document isolation, memory overhead), Figma blog, 2018. https://www.figma.com/blog/rust-in-production-at-figma/
- Figma, "Realtime editing of ordered sequences" (fractional indexing, arbitrary-precision fractions, base-95 string encoding). https://www.figma.com/blog/realtime-editing-of-ordered-sequences/
- Figma, "Multiplayer editing in Figma" (the public launch post). https://www.figma.com/blog/multiplayer-editing-in-figma/
- HelloInterview, "How Figma Built Multiplayer Editing on Simplified CRDTs" (secondary synthesis of the above). https://www.hellointerview.com/learn/system-design/in-the-wild/figma-multiplayer

Fact vs inference: the tree-of-objects-with-properties model, the rejection of OT, per-property last-writer-wins ordered by the central server, atomic parent-plus-position, server-side cycle rejection, fractional indexing with base-95 arbitrary-precision keys, one-worker-per-document, and the Node-to-Rust rewrite with its performance and isolation motivations are all CONFIRMED in Figma and Evan Wallace primary sources above. The exact wire format of delta messages, the precise persistence and snapshot cadence, and current worker counts and memory figures are not fully public and are labeled as inference where used.
