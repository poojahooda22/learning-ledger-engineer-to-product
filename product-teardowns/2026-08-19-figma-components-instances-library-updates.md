# Figma: Components, Instances, and Library Updates

Date: 2026-08-19
Product: Figma
Feature: Components and instances, and the team-library update that pushes one edit to thousands of copies

One line: A component is a master node, an instance is a thin recipe that says "be a copy of that master, except these few overrides." One edit to the master invalidates every instance and each one re-derives itself lazily. At team scale the master lives in a library file, publishing snapshots it, and every other file that uses it gets a "1 update available" nudge and pulls the new version on its own clock.

---

## 1. The user

Meet Aisha. She is a product designer at a mid-size fintech in Bangalore. It is Tuesday, 3 pm. She is working on the "add money" screen. The screen has eleven buttons on it: a big primary "Add money" button, a couple of secondary "Cancel" buttons, some little icon buttons, a disabled state, a loading state.

Every one of those buttons is a copy of the same thing: the team's official button. Same 8 pixel corner radius, same brand green (#00A86B), same 16 pixel horizontal padding, same font (Inter, 15 px, semibold). Aisha did not draw those buttons. She dragged them out of the team library, one by one, and typed new labels into each.

Now the brand team decides the corner radius should be 12, not 8. Softer corners. That decision touches every button in the entire product: hundreds of screens, tens of thousands of button copies, spread across forty different Figma files owned by twelve different designers.

Aisha is not the one making the brand change. But an hour later a small badge appears in her file: a little library icon with a blue dot and the text "1 update available." She clicks it, sees a preview ("Button, updated"), clicks "Update," and all eleven of her buttons quietly round their corners to 12. She did not touch a single one by hand.

That whole thing, the master, the copies, the one edit that reaches all of them, the little "update available" nudge, is the feature we are tearing down.

---

## 2. The real problem

Design used to be copy-paste, and copy-paste rots.

Picture the world before components. You draw a nice button. You need it in twenty places, so you copy-paste it twenty times. Fine for a day. Then the brand changes the green. Now you are hunting through twenty screens, finding every button, and editing each one by hand. You will miss some. Three months later the app has four slightly different greens because nobody could find all the copies. This is the real pain: a design system is only as good as your ability to change it everywhere at once, and manual copies make "everywhere at once" impossible.

The friend-level description: it is like having your company logo pasted into 300 Word documents, and then the logo changes, and there is no find-and-replace. You are doomed to a week of manual surgery, and you will still get it wrong.

There is a second, quieter pain. You do not want every copy to be identical. Aisha's buttons all share the shape and color, but each has a different label ("Add money," "Cancel," "Confirm"). So the system cannot be "all copies are the same." It has to be "all copies share the master, except for the specific things each copy deliberately changed." That exception list is the hard part. If a brand change to the master were to wipe out Aisha's custom labels, the feature would be worse than useless.

So the real problem has two halves that fight each other:

- Push one change to thousands of copies automatically.
- Without stomping on the deliberate local changes each copy made.

---

## 3. The feature in one sentence

A component is a reusable master element; every instance is a live link back to that master that inherits everything except a small set of local overrides, so one edit to the master flows to all instances at once, and across files a published library carries that master and offers each consuming file an opt-in "update available."

---

## 4. Jobs to be done

What is Aisha really hiring components for?

- "Let me change the button once and have it change everywhere." Single source of truth.
- "Let me still type my own label on each button without breaking the link." Local freedom on top of shared truth.
- "Do not surprise me. Tell me an update is waiting and let me pull it when I am ready." Control over timing.
- "Keep my file fast even though it has 4,000 instances in it." Performance is a job, not a nice-to-have. A design file that stutters is a broken tool.
- "Let me build big things out of small things." A card component made of a button component made of an icon component. Composition.

That fourth job matters more than people think. A single Figma file for a real product can hold hundreds of thousands of nodes. If updating one component forced a recompute of the whole file, the app would freeze every time somebody accepted a library update. The engineering has to make "one master changed" cost roughly "the instances that actually depend on it," not "the whole document."

---

## 5. How it works for the user

The visible surface is small and calm.

You select any element and press the "create component" button (the four-diamonds icon, or Ctrl/Cmd+Option+K). That element becomes a main component. It gets a purple diamond marker. Now you can drag out copies. Each copy is an instance and shows a hollow diamond.

An instance looks and behaves like the master, but the moment you try to restructure it, Figma gently pushes back. You can change "surface" things freely: the text, the color, a swapped icon, toggling a layer on or off. These are overrides. You cannot easily add or delete child layers, because the instance's structure is owned by the master. If you want a different structure, you detach the instance (it becomes plain shapes and forgets its master) or you edit the master.

Overrides show up with a subtle "modified" indication, and there is a "Reset all changes" option that throws your overrides away and snaps the instance back to a pure copy of the master.

At team scale, one more surface appears: the Assets panel and the "Libraries" dialog. You publish a file as a library. Other people enable that library. When you publish a new version, everyone who uses it sees "Updates available," reviews a diff-like preview, and clicks to accept. That is the entire visible feature. Everything hard is underneath.

---

## 6. The actual flow, step by step

Let us walk Aisha's real Tuesday, tap by tap.

Making the master (done once, weeks ago, by the design-system team):

1. A designer draws the button: a rectangle, 8 px radius, green fill, an Inter text layer "Button," 16 px padding.
2. Selects it, presses Cmd+Option+K. It becomes the main component "Button/Primary." Purple diamond appears.
3. They open the "Publish" dialog, write a changelog note ("initial button"), and publish the "Design System" library.

Aisha using it (today):

4. In her "Add money" file, she opens the Assets panel, finds "Button/Primary," and drags it onto the canvas. This creates an instance. Hollow diamond.
5. She double-clicks the label and types "Add money." That keystroke does not edit the master. It writes a text override onto this one instance.
6. She repeats for ten more buttons, each with its own label. Eleven instances, eleven text overrides, all still linked to the one master.

The brand change (an hour later, elsewhere):

7. In the Design System file, a brand designer selects the "Button/Primary" main component and changes corner radius from 8 to 12.
8. They press Publish, write "softer corners," and confirm. Figma snapshots the component's new definition into the library's published version.

The update reaching Aisha:

9. Figma's servers know Aisha's file subscribes to that library and that it contains instances of that component. Her open file (or the next time she opens it) receives a signal: an update is available.
10. A badge appears: "1 update available." She clicks it. A panel lists "Button" with a before/after thumbnail.
11. She clicks "Update." Figma walks every instance of that component in her file, points them at the new master definition, and re-derives each one.
12. All eleven buttons now have 12 px corners. Every custom label ("Add money," "Cancel") is untouched, because labels are overrides and the change was to radius, which none of them overrode.

The quiet magic is step 12. The radius flowed in. The labels stayed. That is the whole promise kept.

---

## 7. Under the hood, like the engineer

This is the heart of it. We will build up from the document model to the instance recipe to the update-propagation problem, and cite the real Figma engineering where it is public. Where internals are not fully documented, it is labeled as inference.

### 7.1 The document is a tree of property maps (confirmed)

A Figma file is a tree of objects with a single root, very much like the HTML DOM. Figma's own multiplayer write-up describes the document as, in effect, a `Map<ObjectID, Map<Property, Value>>`: every object has a stable id, and under that id lives a flat bag of properties and their values. (Source: Figma's "How Figma's multiplayer technology works.")

Concretely, Aisha's green button instance is one node id, say `12:47`, whose property map holds things like `{ parent: 12:40, type: INSTANCE, x: 24, y: 300, componentId: 3:12, overrides: {...} }`. The master button is another node, `3:12`, whose property map holds the real geometry: `{ type: COMPONENT, cornerRadius: 8, fill: #00A86B, ... }` plus child node ids for its rectangle and text layer.

Two facts about this model matter for us:

- Parent and child are just properties. The tree structure lives inside the property maps (a `parent` pointer and an ordering key), not in a separate rigid structure. That is why moving a layer is just a property write.
- Conflicts resolve per property, last writer wins. If two people edit the same property of the same object, the server keeps the last value it received. Two people editing different properties, or the same property on different objects, never conflict. This is a deliberately simplified, CRDT-inspired scheme, not full OT and not a full CRDT. (Source: Figma multiplayer blog.)

Hold onto the "tree of property maps" picture. Components are a second tree layered on top of it.

### 7.2 An instance is a recipe, not a copy (this is the key idea)

Here is the misconception to kill: an instance is not a deep copy of the master's subtree sitting in the document. If it were, a file with 4,000 buttons would literally store 4,000 full button subtrees, and an edit to the master could not find them cheaply.

Instead, an instance is a small node that says three things:

1. "My master is component `3:12`."
2. "Derive my children by copying the master's subtree."
3. "But apply this override map on top."

The override map is the exception list from section 2. It is keyed by a path into the instance's subtree and holds only the properties this instance deliberately changed. Aisha's "Add money" button stores roughly `overrides = { "text-layer": { characters: "Add money" } }`. That is a handful of bytes, not a whole button.

Figma made this explicit in March 2026 in "How We Rebuilt the Foundations of Component Instances." They describe a generic system called the **Materializer** whose job is to "create and maintain derived subtrees, subtrees whose structure and properties are computed from other sources of truth." For a component instance, a **blueprint** "describes how an instance resolves itself from its main component: what properties it inherits, which overrides apply, and which children should exist." (Source: Figma blog, "How We Rebuilt the Foundations of Component Instances," March 2026.)

Read that carefully. The instance's real children are not stored. They are *materialized*: computed on demand from (master subtree) + (override map). This is the same idea as a spreadsheet cell holding `=A1*2` instead of a frozen number. The formula is tiny; the value is derived.

So the resolution of one instance is, in plain terms:

```
materialize(instance):
    base = clone_structure(master_of(instance))   # shape and children from the master
    for (path, props) in instance.overrides:       # layer the exceptions on top
        apply(base, path, props)
    return base
```

Walk Aisha's button: clone the master (rectangle with 12 px radius after the brand change, green fill, a text layer reading "Button"), then apply her one override (`text = "Add money"`). Out comes a 12 px green button that says "Add money." The radius came from the master (fresh), the label came from the override (preserved). That single function is why section 12 of the flow works.

### 7.3 Nesting and composition: instances inside instances

Real design systems nest. A "Card" component contains a "Button" instance which contains an "Icon" instance. So materializing a Card means materializing its Button child, which means materializing that Button's Icon child. It is recursion down a graph of masters.

This is exactly where the old system hurt and why Figma rebuilt it. Overrides on nested instances (change the icon inside the button inside the card) were fragile. You have seen the symptoms in the wild: forum threads titled "overrides not preserved when library is pushed" and "nested variant instances are reset after the library update," and the error "Some components were reparented to avoid a dependency cycle." Those are the sharp edges of resolving a derived tree whose sources are themselves derived trees. The Materializer rebuild was aimed at making this override-resolution consistent and reactive rather than a pile of special cases. (Sources: Figma forum threads; Figma Materializer blog.)

The data structure in play here is a **dependency graph**, not just a tree. Instance `12:47` depends on master `3:12`. If `3:12` is itself an instance-bearing component, it depends on its masters too. Edges point from "derived thing" to "source of truth." A cycle in that graph (A contains an instance of B, B contains an instance of A) is illegal, which is precisely why Figma sometimes reparents to "avoid a dependency cycle." You cannot define something in terms of itself.

### 7.4 Propagating one edit: invalidate, then re-materialize lazily

Now the money question. The brand designer changes the master's radius from 8 to 12. How do 4,000 instances update without recomputing the entire document?

The answer Figma describes is **dependency tracking plus targeted invalidation**. As nodes read data during materialization, the Materializer records those relationships implicitly. Developers do not declare "instance X depends on master Y" by hand; the system learns it as materialization runs. When the master changes, the system marks the dependents dirty and re-materializes "only what's necessary," rather than recomputing everything. (Source: Figma Materializer blog.)

In plain steps, when master `3:12` gets `cornerRadius = 12`:

1. Look up who depends on `3:12`. That is the set of instances (and nested instances) that read from it. This is a lookup in the dependency index, not a scan of all 200,000 nodes in the file.
2. Mark each of those dependents dirty (stale).
3. Re-materialize the dirty ones. Because a dirty instance is just "run `materialize(instance)` again," each one re-clones the now-12 px master and re-applies its own override map. Radius updates, label survives.

There is a real design choice between doing the recompute eagerly the instant the master changes (push), or lazily when the instance is next read for layout or rendering (pull). The public descriptions of the Materializer emphasize invalidating stale data and re-materializing only what is necessary, which is the lazy, demand-driven flavor: mark dirty now, do the actual work when the value is needed. (Inference, grounded in the blog's language: the exact eager-vs-lazy split for every path is not fully public.) Either way, the cost scales with "instances that depend on the changed master," not with document size. That is the property that keeps the app usable.

Compare this to the naive alternative: on any master edit, walk the whole document, find instances by scanning, deep-recompute each. That is O(document) per edit. The dependency-index approach is roughly O(affected instances + their subtree size). For a 200k-node file where a given button is used 300 times, that is the difference between touching 200,000 nodes and touching a few thousand.

### 7.5 Crossing files: publishing is a snapshot, and the update is opt-in

Everything above is within one file. Aisha's file and the Design System file are different documents on different servers-worth of state. A live pointer from her instance into the brand designer's open editing session would be chaos: she would see half-finished edits, renamed layers, experiments.

So the cross-file link is not live. It is versioned. Publishing a library takes a **snapshot** of the current components and stamps it with a version. Aisha's instances point at "Button/Primary, version N," a frozen definition, not at whatever the brand designer is doing right now. (This is the same instinct as depending on `react@18.2.0` in a lockfile rather than on whatever is on someone's laptop.)

The update flow is therefore a classic publish/subscribe with human-gated pull:

- Publish: the Design System file writes a new immutable version (N+1) of its components to Figma's backend, with the changelog note.
- Notify: the backend knows which files subscribe to that library. It signals those files "a newer version exists." That is the "1 update available" badge. (Figma keeps this mapping server-side; the exact index is not public, but it must be a subscription table keyed by library so the notify step is a lookup, not a crawl of every file in the org. Inference.)
- Pull: Aisha accepts. Her file swaps its instances' target from version N to N+1 and re-materializes them locally using the same `materialize()` machinery from section 7.4. The heavy work happens in her file, on her accept, on her clock.

Opt-in matters for a human reason and a systems reason. Human: a designer mid-deadline does not want the ground shifting under them; they pull when ready. Systems: making it pull-and-local means one brand edit does not trigger a synchronous write storm across 40 files and 12 editing sessions at once. The work is spread out in time as each file accepts. That is load-shedding by design.

The failure modes people report (published but "half the components did not change," or a component with a broken property silently failing to publish) are the seams of exactly this pipeline: snapshot, subscription notify, per-file re-materialize. When one stage half-fires, you get a partial update, which is why those forum threads exist. (Sources: Figma forum threads on library updates not propagating.)

### 7.6 The scale story at three tiers

Tier one, 1,000 nodes, a single file, one designer. Everything is trivial. An instance could even be a naive deep copy and nobody would notice. Materialize on every edit, re-render, done. The whole document fits in memory and recomputes in a frame. There is no problem to solve yet.

Tier two, 100,000 nodes, one big file, a handful of multiplayer collaborators. Now naive deep copies bloat memory and naive "recompute the document on any master edit" drops frames. This is where the recipe-not-copy model earns its keep: instances stay tiny (a master pointer plus an override map), and a master edit re-materializes only the dependent instances via the dependency index, not the whole tree. This is also where multiplayer's per-property last-writer-wins matters: two designers editing two different instances of the same button touch different objects, so they never conflict; the server only arbitrates when they hit the same property of the same node. The thing that breaks going into this tier is "recompute everything," and the fix is targeted invalidation.

Tier three, 10 million plus nodes across an organization: think a giant design system used by hundreds of files, thousands of designers, one icon like "search-24" instanced tens of thousands of times across the whole company. Now the problems are cross-file, not in-file. A single edit cannot be allowed to fan out synchronously to every file. The survival moves are the ones in section 7.5: versioned snapshots so nobody points at a live in-progress master; a server-side subscription index so "who needs to know about this publish" is a keyed lookup, not an org-wide crawl; and opt-in per-file pull so the re-materialization cost is sharded across files and spread across time. What breaks at this tier is synchronous global propagation; the fix is publish/subscribe with lazy, local, human-gated application. It is the same shape as CDN cache invalidation: you do not push new bytes to every edge at once, you mark stale and let each edge refetch on demand.

The through-line across all three tiers is one principle: keep the derived thing cheap to store (a recipe) and cheap to update (invalidate a dependency edge, re-run the recipe), and never let the cost of one change scale with the size of the whole system.

---

## 8. The retention and habit mechanic

Components are a lock-in loop disguised as a convenience.

The loop: a designer adopts the team library because dragging a ready-made button is faster than drawing one. Every instance they place deepens their file's dependence on that library. The more of their work is built from components, the more painful it is to leave Figma, because the value is not in any one button, it is in the living web of masters and instances that only Figma resolves. This is switching-cost retention, and it compounds silently with every instance placed.

The metric it moves is retention, at two levels. Individual retention: the designer keeps coming back because their whole file is wired to update itself; abandoning it means losing the auto-propagation. Team and revenue retention: components are what make an *organization* standardize on Figma. Once the design system lives as a Figma library and forty product files subscribe to it, switching tools means rebuilding the entire system and re-linking thousands of instances. That is why libraries sit behind Figma's paid Organization and Enterprise tiers: the feature that creates the deepest lock-in is the one they charge companies for. Component adoption is a leading indicator of a team that will renew.

A concrete observed example of the habit: the "N updates available" badge is a small, recurring dopamine-free-but-satisfying nudge. It reappears every time the design system evolves, pulling the designer back into the file to accept changes, keeping the file fresh and keeping the designer in the tool. It is a gentle, non-spammy version of the same "make the app feel alive, pull the user back" pattern you see in consumer apps, tuned for a professional audience that would resent anything louder. The reward is real (your work stays consistent for free), so the loop sustains itself without gimmicks.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code, plus an embeddable runtime. The Figma component model is almost a blueprint for how a shared shader or effect should live inside Rare.lab and inside every product that embeds it. Here is the specific, actionable lesson.

Treat a reusable shader node as a **master with instances resolved by recipe, not by copy, and propagate updates by versioned publish/subscribe, not by live link.**

Break that into concrete moves:

1. Make an embed a thin recipe. When someone drops your "glass distortion" effect into their scene, do not deep-copy the whole node subgraph into their document. Store a pointer to the master graph plus a small override map of the parameters they tuned (blur radius, tint, intensity). This keeps embeds tiny and, crucially, keeps the link intact so updates can flow. Aisha's button label survived the brand change for exactly this reason: overrides are stored separately from the master, so a master edit and a local tweak do not collide.

2. Build a dependency index and invalidate, do not recompute. When a master shader graph changes, you must not re-resolve every embed in a scene, and above all you must not recompile every shader on the GPU. Track which embeds and which downstream nodes read from the changed node, mark exactly those dirty, and re-resolve (and recompile) only those. This is Figma's Materializer move applied to shader compilation, where the cost of a recompute is far higher than re-laying-out a button, so targeted invalidation matters even more for you. A scene with 500 effect instances where one master changed should recompile roughly one shader, not 500.

3. Cross-boundary updates must be versioned and opt-in, never live. When a game studio embeds your runtime and ships it, they cannot have their shipped title's visuals shift because you edited a master at 3 am. Publish immutable versions of an effect. Let each consuming project pin a version and pull `v2` on its own clock with a visible "update available," the way Figma libraries work. Live cross-document links would be a support nightmare and a trust killer; snapshot-and-subscribe is what makes shared assets safe to depend on at scale.

4. Forbid cycles explicitly and cheaply. A node graph that compiles to a shader is a DAG. If your composition system lets effect A embed effect B which embeds effect A, you get an infinite resolution loop and a compile that never terminates. Detect cycles at edit time and refuse the edge, the way Figma "reparents to avoid a dependency cycle." Cheaper to block at authoring than to discover in a stuttering runtime.

The one-sentence version: store the derived thing as a formula over a master plus overrides, invalidate along a dependency graph so one change costs only its dependents, and cross every boundary (file, project, shipped build) with versioned publish/subscribe rather than a live wire. That is how one edit safely reaches ten thousand copies without melting the machine or breaking the ten thousand small things each copy did on purpose.

---

## Sources

- Figma Blog, "How We Rebuilt the Foundations of Component Instances" (March 2026): the Materializer system, blueprints, derived subtrees, dependency tracking, invalidation, and re-materialization. https://www.figma.com/blog/how-we-rebuilt-the-foundations-of-component-instances/
- Figma Blog, "Components in Figma": main components vs instances, overrides, publishing, and instance updates. https://www.figma.com/blog/components-in-figma/
- Figma Blog, "How Figma's multiplayer technology works": the document as a tree of objects, `Map<ObjectID, Map<Property, Value>>`, per-property last-writer-wins, and the simplified CRDT-inspired model. https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
- Evan Wallace, "How Figma's multiplayer technology works" (mirror of the above by Figma's co-founder): document-model internals. https://madebyevan.com/figma/how-figmas-multiplayer-technology-works/
- Figma Help Center, "Guide to components in Figma" and "Explore component properties": the user-facing model of components, instances, overrides, and reset. https://help.figma.com/hc/en-us/articles/360038662654-Guide-to-components-in-Figma
- Figma Best Practices, "Component architecture in Figma": nesting and composition patterns. https://www.figma.com/best-practices/component-architecture/
- Figma Forum threads on library-update propagation and override preservation (real-world failure modes: partial updates, overrides reset, "reparented to avoid a dependency cycle"): https://forum.figma.com/ask-the-community-7/changes-published-in-library-are-not-updating-component-instances-in-other-files-that-use-the-library-19374 and https://forum.figma.com/ask-the-community-7/error-message-some-components-were-reparented-to-avoid-a-dependency-cycle-26738
