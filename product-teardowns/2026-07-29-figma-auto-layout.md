# Figma Auto Layout: the frame that resizes itself

Date: 2026-07-29
Product: Figma
Feature: Auto Layout (the frame that stacks, spaces, and resizes its children automatically)

## 1. The user

Meet Riya. She is a product designer at a fintech startup, three coffees deep,
building the settings screen for their app. The screen has a list of rows:
"Profile", "Notifications", "Linked bank accounts", "Privacy". Each row is a
label on the left, a small chevron on the right, a thin divider under it.

She has just been told the row "Notifications" needs to become "Notifications
and alerts". Longer text. And a new row, "Two-factor authentication", has to
slot in between "Privacy" and the bottom. It is 4:40pm. Design review is at 5.

## 2. The real problem

If Riya had drawn those rows the old way, as free-floating rectangles and text
sitting at hand-placed x/y coordinates, this small change is death by a
thousand nudges. Longer text means the label box has to grow, so she drags its
right edge. The chevron now overlaps the text, so she drags the chevron. The
divider is the wrong width, so she stretches it. Then she inserts the new row,
which means every row below it has to move down by exactly the row height plus
the gap, one by one, and if she is off by two pixels the whole list looks
drunk. Multiply that by every screen, every dark-mode variant, every time
copy changes. That is the actual pain: in a hand-placed layout, content and
position are glued together, so changing content means re-doing position by
hand, forever.

She wants the design to behave like a real app screen: change the words, and
everything else just reflows.

## 3. The feature in one sentence

Auto Layout turns a Figma frame into a self-arranging container: you say
"stack these in a column, 12 pixels apart, padded 16 all around", and from then
on the frame positions, spaces, and resizes its children automatically whenever
their content or count changes.

## 4. Jobs to be done

- "When I change the text, don't make me move everything else." (self-healing
  layout)
- "When I add or delete a row, keep the spacing perfect without me counting
  pixels." (automatic stacking and gaps)
- "Let one button hug its label, and let one bar fill the whole width, in the
  same design." (per-child sizing intent)
- "Make my Figma file behave like the flexbox the engineers will actually
  write, so handoff is not a fight." (design that maps to code)

## 5. How it works for the user

Riya selects her four rows and presses Shift+A. They snap into an Auto Layout
frame: a vertical stack, even gaps, even padding. A panel appears on the right
with a few controls: direction (horizontal or vertical), the gap between items,
the padding inside the frame, and alignment (top, center, bottom, or
space-between).

Now the magic. She double-clicks "Notifications" and types "Notifications and
alerts". The row's label grows to fit the longer text, the row grows with it,
and because it is a vertical stack, nothing below shifts sideways. She drags
"Two-factor authentication" into the stack between two rows; the list opens a
gap exactly the right size, drops it in, and every row below slides down by the
correct amount. She never touched a coordinate. Total time: about 20 seconds.

The other half is per-child sizing. Each child can be set to one of three
modes:

- Fixed: this stays exactly this size (a 24 by 24 icon).
- Hug contents: the box shrinks or grows to exactly fit what is inside (a
  button that hugs its label, so "OK" is a small button and "Continue to
  payment" is a wide one, automatically).
- Fill container: the box stretches to take all the space left over (the label
  fills the row so the chevron gets pushed to the far right).

(Fact: the Fixed / Hug / Fill modes and the direction, gap, padding, and
alignment controls are Figma's documented Auto Layout model, and Figma states
Auto Layout was designed to mirror how the web renders layout and to match CSS
flexbox behavior. See Sources.)

## 6. The actual flow, step by step

1. Select the rows. Press Shift+A. Figma wraps them in an Auto Layout frame and
   guesses direction from how they were arranged (they were stacked, so it
   picks vertical).
2. The right panel now shows the Auto Layout controls. Riya sets gap to 12 and
   padding to 16.
3. She sets each row's label to "Fill container" and each chevron to "Fixed",
   with the frame itself set to "Hug contents" vertically (so the frame is
   exactly as tall as its rows) and "Fixed" width (the screen is a fixed 375
   wide).
4. She edits the middle row's text. The label, being Hug inside a Fill slot,
   reports a new intrinsic width; the row re-flows; the frame re-hugs its new
   height; rows below move down. All in one frame, instantly.
5. She drags in the new row. Figma shows a blue insertion line between two
   children, she drops, and the stack re-spaces.
6. Handoff: an engineer opens Dev Mode and sees the frame described as
   `display: flex; flex-direction: column; gap: 12px; padding: 16px;`, which
   maps almost one to one onto the CSS they will write.

## 7. Under the hood, like the engineer

This is where a designer sees "it just reflows" and an engineer sees a layout
engine running a tree algorithm on every edit. Let me separate what is
confirmed from what is well-grounded inference.

### What Figma has confirmed (fact)

Figma's canvas is not the browser's HTML/CSS engine. The document is a scene
graph written in C++ and compiled to WebAssembly with Emscripten. It draws
straight to the GPU through WebGL, and now WebGPU, bypassing the browser's own
HTML layout and paint pipeline entirely. The 2018 rewrite of this renderer,
plus fixing WebAssembly bugs, made Figma about 3x faster. Figma runs its own
world so it can push millions of state changes through a design file without
the browser falling over. (Sources: Figma "Figma Rendering: Powered by
WebGPU", the Config 2022 talk "A Tour of the C++ Engine which Powers That
Design MMO", and independent write-ups.)

Figma has also described how it avoids recomputing everything on every change.
It keeps an explicit dependency graph with push-based invalidation. When a
source value changes, Figma marks its dependents as dirty and recomputes them
later, lazily, only when needed. Their "Materializer" records these
dependencies automatically as nodes read data during materialization, so
engineers do not hand-declare who depends on whom. The payoff they state
plainly: only the relevant subsections of the tree update, and subtree-level
computations can be cached. Expanding one frame recomputes only that frame's
subtree and skips the rest. (Source: Figma "How We Rebuilt the Foundations of
Component Instances".)

Put those two facts together and you have the skeleton of why Auto Layout feels
instant: a GPU scene graph plus a dirty-flag dependency graph so a text edit
touches only the branch of the tree that actually changed.

### The data structure: a tree of boxes

The document is a tree. The root is the page. Frames are internal nodes.
Shapes, text, and icons are leaves. Riya's settings screen is:

```
Frame "Settings" (vertical, gap 12, pad 16)
 - Row "Profile" (horizontal)
    - Text "Profile" (Fill)
    - Icon chevron (Fixed 24x24)
 - Row "Notifications and alerts" (horizontal)
    - Text "Notifications and alerts" (Fill)
    - Icon chevron (Fixed 24x24)
 - Row "Two-factor authentication" (horizontal)
    - ...
 - Row "Privacy" (horizontal)
    - ...
```

A tree is the right structure because layout is naturally recursive: a frame's
size depends on its children's sizes, which depend on their children, all the
way down to the text and icons. Auto Layout is a rule attached to each internal
node saying "arrange my direct children along this axis with this gap and
padding". Nesting frames inside frames (a row inside a list inside a card) is
just deeper tree levels, and the same rule runs at each level.

### The algorithm: two passes over the tree (inference, grounded in how flexbox engines universally work)

Figma has not published its exact layout pass, so this next part is inference,
but it is safe inference because Figma states Auto Layout mirrors CSS flexbox,
and every flexbox-style engine (browsers, Yoga, the standalone implementations)
solves this with the same shape of algorithm: measure bottom-up, then place
top-down.

Pass one, bottom-up (resolve intrinsic sizes). Start at the leaves. A text node
measures the pixels its string needs: "Notifications and alerts" in the chosen
font is, say, 172 pixels wide. An icon reports its fixed 24 by 24. Then walk up.
A "Hug contents" frame sets its own size to exactly the sum of its children plus
gaps plus padding. So a button that hugs "Continue to payment" ends up wider
than one hugging "OK", with zero manual work, because its width is computed from
the child text's measured width. This pass answers "how big does each box want
to be".

Pass two, top-down (distribute leftover space and place). Now the parent knows
its own final width (fixed 375 for the screen). It subtracts padding and the
Fixed children (the 24-pixel chevron) and the gaps, then hands the leftover
space to the "Fill container" children. In the row, the label is Fill, so it
takes all the width the chevron did not, which is what pushes the chevron to the
far right. If several children are Fill, the leftover is split among them (the
flexbox flex-grow idea). This pass answers "where does each box actually go".

Two passes because you cannot place things until you know how big they want to
be, and a parent's size can depend on children (Hug) while a child's size can
depend on the parent (Fill). Real flexbox implementations often use two or three
passes for exactly this reason: one top-down traversal to build the work order,
one bottom-up to resolve automatic/hug sizes, one top-down to resolve fill and
apply positions. (Source: standard flexbox-engine write-ups, listed below.)

Now connect the algorithm to the dirty-flag fact. When Riya edits one row's
text, only that text leaf's intrinsic size changed. The dependency graph marks
that leaf dirty, which marks its row frame dirty, which marks the settings frame
dirty (because it Hugs its height), and stops there. The four other rows are not
dirty, so their subtrees are not re-measured. The engine re-runs the two passes
on the dirty path only, which is a handful of nodes, not the whole document.
That is the difference between "instant" and "janky".

### Where the work happens: on your machine, in WASM, not on a server

Unlike a search-ranking feature, layout is not a database query. The whole
scene graph lives in memory in the browser tab, and layout runs locally in the
WebAssembly C++ engine on every edit, targeting 60 frames per second so
dragging feels live. The server's job is different: it stores and syncs the
document (the multiplayer layer, covered in the 2026-06-18 cursors teardown and
the 2026-07-09 canvas-rendering teardown), and it streams other people's edits
in as operations. Each incoming edit is applied to the local tree, marks the
touched nodes dirty, and triggers the same incremental relayout. So "someone
else renamed a row" and "I renamed a row" go through the identical dirty-flag
path.

### The scale story, three tiers

Here the "catalog of millions" is the number of nodes in the document tree and
the number of edits per second, not rows in a database.

Tier 1, about 1,000 nodes (one screen, a few components). Brute force is fine.
Re-run the full two-pass layout over the entire tree on every keystroke. A
thousand boxes is nothing for a C++/WASM engine; it finishes in well under a
frame. A dirty-flag system here is almost over-engineering. This is Riya's
settings screen.

Tier 2, about 100,000 nodes (a real design-system file: hundreds of components,
each with variants, spread over many frames). Now a full relayout on every
keystroke starts to miss the 16-millisecond frame budget, and typing feels
laggy. This is exactly where the dirty-flag dependency graph earns its keep:
mark only the edited subtree dirty and re-layout just that branch, and cache the
subtree results so untouched components are never recomputed. Figma's stated
"only the affected subtree recomputes" is the survival move at this tier.
Component instances make this sharper: change the main component once, and every
instance must update, so the dependency graph has to fan the dirty flag out to
all instances efficiently, which is a big part of why Figma rebuilt the
instance foundations.

Tier 3, millions of nodes and many concurrent editors (a giant multi-page file,
a whole product's design system, a live design jam). What breaks is memory and
the cost of ever walking the whole tree. Survival moves, some confirmed by
Figma, some standard practice: keep the engine in C++/WASM with tight manual
memory control instead of garbage-collected JavaScript; load pages
incrementally and lazily (Figma's "Speeding Up File Load Times, One Page At A
Time" and "Incremental Frame Loading") so you never materialize what is off
screen; render only visible tiles on the GPU; and never, ever do an O(all
nodes) operation on the hot path. Layout stays local and incremental; the tree
is the unit of caching; the dirty set is the unit of work. The rule at every
tier is the same: make the cost of an edit proportional to what changed, not to
how big the file is.

## 8. The retention and habit mechanic

Auto Layout is a productivity flywheel that quietly becomes a switching cost.

The loop: Riya makes a change (new text, new row), the layout self-heals in a
second, she is rewarded with a correct-looking screen for almost no effort. That
reward makes her build the next thing with Auto Layout too. Over weeks, her
whole file, and then her team's design system, is built out of Auto Layout
frames and components. Now the file is not just drawings, it is a little
program that reflows itself.

Which metric it moves: retention, and underneath it, expansion revenue. Not by a
dopamine notification but by lock-in through capability. Once a team's design
system is expressed as nested Auto Layout components, the cost of leaving Figma
(or of going back to hand-placed layouts) is enormous, because you would be
throwing away a self-maintaining system and going back to nudging pixels. That
is the stickiest kind of retention: the product got more valuable the more the
user invested in it.

A real observed example of the loop paying off: renaming a navigation menu item
from "Docs" to "Documentation" in an Auto Layout navbar reflows the entire bar,
re-centers it, and re-spaces every other item, with no manual cleanup. The first
time a designer sees that, they stop drawing rectangles by hand for the rest of
their career. That single "oh" moment is the activation event, and every
self-healing edit after it is the retention drip.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader editor that compiles to shippable code, plus an
embeddable runtime. Figma's Auto Layout hands you a precise, performance-first
lesson: model your node graph as a dependency graph with push-based
invalidation and dirty flags, and make the cost of any edit proportional to the
affected subtree, never to the whole graph.

Concretely. When a Rare.lab user tweaks one parameter, say the frequency on a
single noise node, do not recompile or re-evaluate the whole shader graph.
Follow Figma's Materializer pattern: as nodes read their inputs during
evaluation, record the dependency edges automatically, so you build the
dependency graph for free instead of hand-declaring it. On an edit, mark only
that node and its downstream nodes dirty, and recompute only that path. Cache
each node's compiled output keyed by a hash of its inputs, so an untouched
subtree is a cache hit and never recompiles. This is the exact move that lets
Figma stay at 60fps in a 100,000-node file, and it is what will let a Rare.lab
graph with hundreds of nodes recompile on every slider drag without stalling the
preview.

Two more transfers. First, borrow the two-pass idea for anything in Rare.lab
that has a "hug" versus "fill" tension, for example a node canvas that should
grow to fit its contents while individual panels fill available space: resolve
intrinsic sizes bottom-up, then distribute leftover space top-down, so you never
get circular size dependencies. Second, and most important for your runtime:
keep the hot evaluation loop in a tight, GC-free, memory-controlled layer
(WASM/C++ or compiled native), exactly as Figma keeps its engine out of
JavaScript, and keep the "chrome" (your editor UI) in whatever is convenient.
The heavy per-frame math belongs in the engine; the UI just sends edits and
reads results. Ship the dirty-flag dependency graph on day one; retrofitting
incrementality into a graph engine after it is slow is the hardest rewrite there
is.

## Sources

- Figma Blog, "Behind the feature: the making of the new Auto Layout": https://www.figma.com/blog/behind-the-feature-the-making-of-the-new-auto-layout/
- Figma Learn, "Use auto layout with CSS Flexbox in mind": https://help.figma.com/hc/en-us/articles/42031586813719-Use-auto-layout-with-CSS-Flexbox-in-mind
- Figma Blog, "How We Rebuilt the Foundations of Component Instances" (dependency graph, push-based invalidation, Materializer, subtree recompute): https://www.figma.com/blog/how-we-rebuilt-the-foundations-of-component-instances/
- Figma Blog, "Figma Rendering: Powered by WebGPU": https://www.figma.com/blog/figma-rendering-powered-by-webgpu/
- Figma Blog, "Keeping Figma Fast": https://www.figma.com/blog/keeping-figma-fast/
- Figma Blog, "Speeding Up File Load Times, One Page At A Time": https://www.figma.com/blog/speeding-up-file-load-times-one-page-at-a-time/
- Figma Blog, "Improving Performance with Incremental Frame Loading": https://www.figma.com/blog/incremental-frame-loading/
- Joey Liaw, "A Tour of the C++ Engine which Powers That Design MMO Called Figma", Config 2022: https://www.youtube.com/watch?v=opGoe7yHHkk
- MDN, "Basic concepts of flexbox": https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Flexible_Box_Layout/Basic_Concepts_of_Flexbox
- tchayen, "How to Write a Flexbox Layout Engine" (multi-pass measure/layout): https://tchayen.com/how-to-write-a-flexbox-layout-engine

Note on fact versus inference: the C++/WASM scene graph, GPU rendering, the
2018 3x speedup, and the dependency-graph/dirty-flag/Materializer invalidation
model are stated by Figma. The specific two-pass "measure bottom-up, place
top-down" layout algorithm is inference, grounded in Figma's own statement that
Auto Layout mirrors CSS flexbox and in how every flexbox-class engine is built;
Figma has not published its exact layout pass.
