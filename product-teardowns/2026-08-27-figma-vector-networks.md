# Figma: Vector Networks (the pen tool that treats a shape as a graph, not a loop)

Date: 2026-08-27
Product: Figma
Feature: Vector networks, the geometry model behind Figma's pen and bend tools, where a shape is a graph of vertices and edges instead of one closed chain of anchor points

One line: Every other vector tool since 1987 stores a shape as a single ordered chain of anchor points (a linked list that has to loop back on itself), and that one choice is the source of most pen-tool pain. Figma threw out the chain and stored the shape as a graph: an array of vertices, an array of segments that can connect ANY two vertices, and an array of fillable regions. A vertex can now have three or five edges, not just two, so a plus sign or a wireframe joint is one object instead of overlapping hacks. Deleting a point "heals" the graph instead of ripping the loop open, and fill starts by auto-filling every enclosed region so you never fight winding direction. The heavy triangle mesh the GPU actually draws is a derived, cached byproduct, re-tessellated only where you edited.

---

## 1. The user

Meet Meera. She designs icons at a design studio in Pune. It is Thursday, 11 am, and she is drawing a small "network" icon for a settings screen: three little circles connected by lines, like a hub with three spokes. The center circle has three lines coming out of it to the three outer circles.

She reaches for the pen tool, the way she has in Illustrator for eight years. And here is the thing she has done a thousand times and always hated: in a normal pen tool, that center point cannot have three lines coming out of it. A path is a single strand. It goes point to point to point in one unbroken chain, like beads on a string. A point in the middle of the string touches exactly two neighbors, the bead before and the bead after. It physically cannot touch three.

So in the old world Meera would draw this as three separate overlapping lines, all starting at roughly the same center spot, and then spend five minutes nudging their endpoints so they meet cleanly, and the "center point" would actually be three different points stacked on top of each other. Move one by accident and the icon splits open.

In Figma she just clicks the center, then clicks each of the three outer points. The center vertex now has three edges. It is one shape. She drags the center and all three spokes follow, because they are genuinely attached to the same vertex. That difference, one point holding three lines, is the whole feature we are tearing down. It sounds tiny. It rewrites the data structure under the entire drawing surface.

---

## 2. The real problem

The pen tool as we know it was shipped in Adobe Illustrator 88, in 1987, and the mental model has barely changed since. It is a bezier path: a chain of anchor points, and between each pair of anchors a curve controlled by two handles. It is a beautiful tool for drawing one clean outline, a letter "S", the silhouette of a leaf. It is a miserable tool the moment your shape is not a single loop.

Three real pains, described the way a friend who draws for a living would describe them:

Pain one, the T-junction. You are drawing a tree, or a subway map, or the "network" icon above. You want three lines meeting at one joint. A single-chain path cannot express that. So you draw overlapping separate paths and pray they stay aligned. The tool is forcing you to lie about the shape.

Pain two, fill direction and the winding number. In a normal vector tool, whether the inside of a shape fills depends on which direction you drew the outline, clockwise or counter-clockwise, and how many times the outline wraps around a spot. This is the "winding number." Draw a donut and the hole only becomes a hole if the inner circle winds the opposite way from the outer circle. Get the direction wrong and your donut fills solid. Designers have lost hours to "why is my hole not a hole," and the honest answer is a math concept they never asked to learn.

Pain three, deleting a point breaks everything. In a chain, every point is load-bearing. Delete a point in the middle of your path and the path either snaps shut across the gap in a way you did not want, or tears open into two pieces. You wanted to remove one bump from a curve; you got a broken outline.

Under all three pains sits one root cause: the shape is stored as a single ordered loop. Every frustration is that loop refusing to be anything other than a loop.

---

## 3. The feature in one sentence

A vector network stores a shape as a graph (a set of vertices, a set of edges between any two vertices, and a set of fillable regions) instead of as a single closed chain of points, so a point can have any number of edges, deleting a point heals the graph, and fill is chosen per enclosed region instead of by winding direction.

---

## 4. Jobs to be done

What is Meera really hiring vector networks to do?

- "Let me draw the shape that is actually in my head, including joints where three or more lines meet, without stacking fake overlapping paths." (A wireframe cube, a flowchart node, a leaf with veins.)
- "Let me edit without fear. If I remove a point, keep my shape sensible instead of tearing it open."
- "Stop making me think about draw direction. If a region is enclosed, let me just say fill or do not fill."
- "Do not make me relearn anything. If I already know the pen tool, let me keep working the way I always have and only notice the difference when the old limits are gone."

That last job is real and load-bearing. Figma ran user studies on the first vector networks build. Many people did not even notice it was different from a normal pen tool; it just worked as expected. They noticed only when they went back to another tool and hit the old walls again. A new data structure that most users never consciously see is a rare and deliberate design win.

---

## 5. How it works for the user

The visible experience, in plain terms:

You pick the pen tool and click to drop points, same as always. The difference is you can click back onto any existing point to start a new edge from it. Click the center of the network icon, then click an outer node: that is one edge. Click the center again, then a second outer node: a second edge from the same center. The center now holds two, then three edges. No stacking, no alignment ritual.

You get a bend tool. Hover an edge or a point and drag to curve it, without switching modes or fiddling two separate handles. Under the hood each curved segment still has bezier handles (Figma calls them tangents), but the bend tool lets you shape the curve directly.

Delete behaves like "delete and heal." Select a point in the middle of a line and hit Shift+Delete: the point vanishes and the two lines it connected join into one clean line across the gap. The shape does not tear.

Fill is a paint bucket that works per region. Figma starts by treating the entire enclosed part of your network as filled. If you want a hole, you point at the region you want empty and punch it out. You never rotate a sub-path to flip a winding number. A concrete case: draw a big circle, then a small circle inside it. In a classic tool you would fight winding to make the small circle a hole. In Figma, the ring between them and the inner disc are just two regions; you fill the ring and leave the inner disc empty, and you are done.

---

## 6. The actual flow, step by step

Let us draw Meera's network icon, tap by tap, and watch the data change.

1. Pen tool active. Meera clicks at (100, 100). Figma creates vertex v0 at (100,100). The network so far: one vertex, no edges, no regions.
2. She clicks the center at (150, 150). Figma creates vertex v1, and because a previous point was active, a segment s0 from v0 to v1. Now: two vertices, one edge.
3. She clicks (200, 100). Vertex v2, segment s1 from v1 to v2. The center v1 now touches two edges.
4. Here is the move a normal pen tool cannot make. She clicks back onto v1 (the center) to re-anchor there, then clicks (150, 200). Figma adds vertex v3 and segment s2 from v1 to v3. The center vertex v1 now has three edges: s0, s1, s2. In a chain model this is impossible; in a graph it is just another entry in the segments array.
5. She wants the top spoke curved. She grabs the bend tool and drags segment s0. Figma writes a tangentStart and tangentEnd onto s0, the two bezier control vectors that bow the line. The endpoints do not move; only the curve between them changes.
6. She realizes the top-left node is one click too many and deletes it with Shift+Delete. Figma removes that vertex and heals: the two segments that met there are replaced by one segment spanning the gap. No tear.
7. She closes a little triangle between three of the spokes and clicks the paint bucket inside it. Figma records a region: a loop (the ordered list of the three segment ids bounding that triangle) plus a winding rule, and paints it.

The finished object is not seven separate strokes. It is one vector node holding three arrays: vertices [v0..v3], segments [s0..s2], regions [the triangle]. That is the whole thing.

---

## 7. Under the hood, like the engineer

This is the heart of it. The feature is a data-structure swap, so we go structure first, then the algorithms that structure enables, then rendering, then scale.

### The old structure: a path is a linked list that has to close

A classic vector path is an ordered sequence of anchor points. Between consecutive anchors sits a cubic bezier: start anchor, two control handles, end anchor. Model it as a linked list (or an array walked in order) where node i connects to node i+1, and a "closed" path adds one edge from the last node back to the first. The defining constraint: degree two. Every interior anchor has exactly one predecessor and one successor. That is why a point cannot hold three lines, and why deleting an interior node either bridges its two neighbors or splits the list. The single-loop pain is the linked list showing through.

### The new structure: a shape is a graph

Figma's public plugin API exposes the exact shape of a vector network, and it is three arrays (confirmed, Figma Developer Docs, VectorNetwork / VectorNode):

- vertices: an array of points. Each VectorVertex has an x and a y, plus optional per-point properties: strokeCap, strokeJoin, cornerRadius, handleMirroring.
- segments: an array of edges. Each VectorSegment has a start and an end that are indices into the vertices array, plus optional tangentStart and tangentEnd vectors. The tangents are the bezier control offsets; if both are zero the segment is a straight line, otherwise it is a curve.
- regions: an array of fillable areas. Each VectorRegion has a windingRule ("NONZERO" or "EVENODD") and a list of loops, where each loop is an ordered list of segment indices that bound the region.

So a segment is not "the space between point i and point i+1." It is an explicit edge that names its two endpoints by index. Two segments can name the same vertex as an endpoint. A vertex can appear in five segments. The degree-two prison is gone. In graph terms this is an undirected multigraph with edge identity: undirected because a segment from v1 to v2 is the same edge as v2 to v1, multigraph because two vertices can be joined by more than one edge, and "edge identity" because each segment is a distinct object even if it duplicates another's endpoints (this matters for delete-and-heal below).

Figma states the relationship to paths plainly: a vector network is a superset of paths. It can represent everything a path can, and paths cannot represent everything a vector network can. That is why the old files still open: a single-chain path is just a vector network where every vertex happens to have degree two. Backwards compatible by construction.

The concrete win, in bytes: Meera's network icon in the old world is three overlapping paths, each a little chain, with three near-duplicate center points that must be kept in sync by hand. In the new world it is four vertices, three segments, one shared center vertex. Fewer points, one object, and moving the center is one edit to one vertex that all three segments read.

### Algorithm one: finding fillable regions (planar graph face enumeration)

Once a shape is a graph, "what areas can I fill?" becomes a real graph question: find the enclosed faces of the planar graph. This is the classic planar face traversal. Sort the edges around each vertex by angle. To trace one face, start on an edge and repeatedly take the "next edge clockwise" at the vertex you arrive at; keep going until you return to the start. Each such closed walk is one minimal cycle, one fillable region. The cost is dominated by the angular sort around vertices, on the order of O(E log E) for E edges, which for a hand-drawn icon of a few dozen edges is nothing, and even for a dense imported map of tens of thousands of edges is a one-time computation you cache, not a per-frame cost.

This is why Figma can offer the paint-bucket-per-region experience at all: the regions are not something you declare by drawing in the right direction, they are computed from the graph, and then you toggle each one on or off. A region is stored as its bounding loops plus a winding rule so the fill is unambiguous once chosen, but you, the user, never pick the winding rule by hand.

### Algorithm two: fill without the winding-number headache

Traditional engines decide inside-versus-outside with the winding number: shoot a ray from a point to infinity, count how many times the outline crosses it and in which direction, and fill where the signed count is nonzero (NONZERO rule) or where the raw count is odd (EVENODD rule). Correct, but it makes fill depend on the direction you drew, which is the donut-hole trap.

Figma keeps the winding rule in the data (regions carry NONZERO or EVENODD, because the renderer still needs an unambiguous answer and imported SVGs bring their own rules), but changes the interaction. It starts with the entire enclosed part of the network filled, and lets you punch holes out of specific regions. Because the regions were already enumerated from the graph, "punch this hole" is just flipping one region off. You get the donut by leaving the inner disc region empty, never by reversing a circle's direction. The math still runs; the user is lifted out of it.

### Algorithm three: delete and heal on a graph

In a chain, deleting an interior point is trivial: connect its two neighbors. In a graph a vertex can have any degree, so "heal" needs a real rule. Figma's approach (confirmed, Figma Blog, "Delete and Heal for Vector Networks," Jamie Wong):

- Degree 1 (the vertex touches one edge): there is no sensible heal, so throw the single edge away with the vertex.
- Degree 2 (an ordinary point on a line): heal by replacing the two incident edges with one new edge across the gap. This is the familiar case, the one every tool does.
- Even degree greater than 2: pair up "opposite" edges and replace each pair with a new bridging edge, so a smooth crossing stays smooth.
- Odd degree greater than 2: there is no clean pairing, so all incident edges are deleted.

None of these branches even exist in a single-chain editor, because a chain never lets a point reach degree three. Handling them is the tax Figma pays for the graph, and paying it is what makes "remove a point without tearing the shape" feel effortless.

### How bezier curves live inside the graph

Each segment carries its own curve. tangentStart and tangentEnd are vectors relative to the segment's start and end vertices, exactly the two control handles of a cubic bezier. Straight segment: both tangents zero. This keeps curvature local to the edge, so bending segment s0 of Meera's icon touches only s0's tangents and leaves the two spokes sharing that center vertex untouched. Curve data attached to edges, not to points, is what lets one vertex host several edges that each curve independently.

### Rendering: the graph is authored, triangles are drawn

A GPU does not draw bezier graphs. It draws triangles. So the vector network is the source of truth, and the thing actually rendered is a derived triangle mesh (tessellation). The standard pipeline, and the one consistent with Figma running its canvas as C++ compiled to WebAssembly driving a WebGL/WebGPU tile renderer (see the 2026-07-09 teardown of Figma's canvas engine):

1. Split each cubic bezier segment into simpler pieces (cubics into quadratics, then flatten into short line chords fine enough that the curve looks smooth at the current zoom).
2. Triangulate each filled region's flattened outline into a triangle mesh.
3. For strokes, run stroke-to-fill (stroke expansion): turn the centerline plus width plus caps and joins into a filled outline, then triangulate that too. This is why per-vertex strokeCap and strokeJoin live on the vertices.
4. Upload the triangles to the GPU and draw.

This split, compact editable graph in, heavy triangle mesh out, is the load-bearing engineering decision for performance, and it drives the scale story.

### The scale story at three tiers

The "catalog" here is not a server of a million listings; it is the number of vertices and segments in a document, and the cost is tessellation and region-finding, mostly on the client.

Tier one, about 1,000 points (a detailed single icon or logo). Trivial. On every edit you can re-run region enumeration and re-tessellate the whole node from scratch, re-upload the triangle buffer, and still hit 60 frames per second. Do not over-engineer. A few thousand triangles is a rounding error for any GPU.

Tier two, about 100,000 points (a complex illustration, or an imported SVG world map with every country as a filled region). Now re-tessellating the entire node on every mouse-move during a drag is too slow; you would blow past your 16 millisecond frame budget. What breaks is the "recompute everything per frame" habit. Survival moves: cache the tessellated mesh and mark only the edited part dirty, so dragging one vertex re-tessellates the handful of segments and regions that touch it, not all 100,000. Re-run region-finding incrementally around the change rather than globally. Keep the triangle buffer resident on the GPU and patch the changed slice. The compact graph is what makes this cheap: you edit four numbers on one vertex, and only its neighbors need new triangles.

Tier three, 10 million plus (a whole design file: thousands of vector nodes across many frames, plus imports, plus other people editing live). Now the problem is the document, not one shape. What breaks is trying to keep every node's mesh live at once; you would run out of GPU memory and time. Survival moves: only tessellate what is visible in the viewport at the current zoom, and pick chord fineness by zoom (level of detail), because a country outline zoomed to fit the screen needs far fewer chords than the same outline at 400 percent. The tile renderer (from the canvas-engine teardown) redraws only dirty screen tiles. And crucially for collaboration and file size, the thing synced and saved is the compact graph, not the triangles: Figma serializes documents in a binary Kiwi schema (see Grida's reverse-engineered .fig Kiwi glossary), where a vector node is its vertex, segment, and region arrays, kilobytes, not the megabytes a baked mesh would cost. Every client re-derives its own triangles locally. Send the recipe, not the cake.

### What is confirmed versus inferred

Confirmed from primary sources: the three-array structure and every field name (Figma Developer Docs); the superset-of-paths claim and the auto-fill-then-punch-holes fill model and the 1987 pen-tool framing and the user-study finding (Figma Blog, "Introducing Vector Networks," Evan Wallace); the delete-and-heal rules by vertex degree (Figma Blog, Jamie Wong); the binary Kiwi save format shape (Grida .fig glossary). Clearly labeled inference: the exact planar-face-traversal algorithm and its O(E log E) cost, the incremental-tessellation and level-of-detail specifics, and the cubic-to-quadratic tessellation pipeline are the standard, well-established way this class of problem is solved (NVIDIA GPU Gems chapter 25, Loop and Blinn's GPU curve rendering, recent GPU stroke-expansion work), matched to Figma's known WebAssembly-plus-WebGL renderer, not internals Figma has published line by line.

---

## 8. The retention and habit mechanic

Be honest about what kind of feature this is. Vector networks are not a dopamine loop like Discover Weekly's Monday drop or Instagram Stories' 24-hour timer. There is no notification, no streak. This is a craft tool, and its retention mechanic is a different animal: it removes friction so completely that the user stops noticing the tool and stays in flow.

The metric it moves is activation and retention, not revenue directly. Two concrete mechanisms:

First, activation for people who never mastered the classic pen tool. The pen tool has, for decades, been the wall beginners bounce off. By letting you connect any two points, heal on delete, and fill without understanding winding numbers, vector networks let a newer designer succeed at a task that used to take years of muscle memory. A user who successfully draws the thing they pictured, on the first try, is a user who comes back. Figma's own user study is the real observed example: testers using vector networks did not notice anything unusual, the tool "just worked," and they only felt the difference when they returned to a legacy tool and hit the old limits again. That "ugh, I have to go back to that" reaction is switching-cost retention in its purest form. The tool earns loyalty by being invisible.

Second, retention through reduced rage-quits on the exact operations people do constantly. Delete-and-heal is the sharpest example: in older tools, deleting a point and then manually reconnecting the two loose ends is a small tax paid dozens of times an hour, and every one of those is a tiny moment of friction where a frustrated pro might think about their old app. Removing that reconnection ritual removes dozens of daily papercuts. Papercuts are what erode retention slowly; closing them is what quietly holds it.

Zoom out and this feeds Figma's real habit engine, which is multiplayer and the browser. Vector networks are one of the reasons a serious illustrator or icon designer can move to Figma and not feel downgraded, which is the precondition for the whole team living in Figma, which is where the real retention loop (shared files, comments, presence) takes over. The geometry model is not the hook by itself; it is what removes the reason to leave.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based editor that compiles to shippable shader code plus an embeddable runtime. Vector networks are a near-perfect parable for you, because Figma faced your exact tension: a rich, editable graph that a human manipulates, and a flat, heavy, GPU-friendly artifact that the machine actually runs. Their answer is the lesson.

Keep the authored graph as the single source of truth, and treat the compiled output as a derived, cached, incrementally-updated byproduct.

Concrete moves:

1. Make your editable representation a true graph, and a superset. Figma's vector network can represent everything a path can plus more, which is why old files still open and nothing was lost in the switch. Design Rare.lab's node graph so that simpler forms (a single linear chain of shader passes, a plain material) are just special cases of the general graph. Never build a second, weaker internal model for the "simple" case; a degree-two vertex and a five-way joint are the same structure. One model that degrades gracefully beats two models you must keep in sync.

2. Separate the source graph from the compiled artifact, and sync the graph, not the artifact. Figma stores and transmits the kilobyte vertex/segment/region arrays and lets every client tessellate its own triangles locally. For you, the shippable shader code and the GPU draw setup are the "triangles": derive them, do not treat them as the document. Save, version, and collaborate on the compact node graph. Recompile locally. This keeps files small, diffs meaningful, and multiplayer cheap, and it means a shipped title carries a pinned compiled output while the editor keeps the live graph.

3. Recompile only the dirty subgraph, never the whole thing. This is the tier-two survival move. Figma does not re-tessellate 100,000 points because you dragged one; it re-tessellates the segments and regions that touch the edited vertex. Your equivalent: when a user tweaks one node's parameter or rewires one edge, mark exactly the downstream nodes that read from it dirty and recompile only that slice of the shader, reusing cached output for everything upstream and untouched. Incremental re-tessellation is incremental recompilation. A 500-node effect graph where one node changed should recompile a handful of nodes, not 500, or the editor stutters and the flow state (the very thing that retains Meera) is gone.

4. Choose fidelity by context, like level-of-detail tessellation. Figma flattens a curve into more chords when zoomed in and fewer when zoomed out, spending compute only where the eye can see it. In the Rare.lab editor, a live preview thumbnail does not need the full-resolution, fully-optimized shader; compile a cheaper preview variant for editing and the fully optimized variant only on export or publish. Match the cost of the derived artifact to how it is being looked at right now.

The one-sentence version: hold the human's editable node graph as the compact source of truth, compile the heavy GPU artifact as a cache that only the changed subgraph ever rebuilds, and ship or sync the graph rather than the artifact. That is how Figma made an icon with a three-way joint feel weightless, and it is how a node editor stays at 60 frames per second when the graph gets big.

---

## Sources

- Figma Blog, "Introducing Vector Networks" (Evan Wallace, Figma co-founder): the graph-vs-path model, the 1987 pen-tool framing, connecting any two points, auto-fill-then-punch-holes, backwards compatibility with paths, and the user study. https://www.figma.com/blog/introducing-vector-networks/
- Evan Wallace mirror of "Introducing Vector Networks": archived copy of the above. https://madebyevan.com/figma/introducing-vector-networks/
- Figma Blog, "Delete and Heal for Vector Networks" (Jamie Wong): vector objects as undirected multigraphs with edge identity, and the delete-and-heal rules by vertex degree (degree 1 discard, degree 2 bridge, even degree pair opposite edges, odd degree delete all). https://www.figma.com/blog/delete-and-heal-for-vector-networks/
- Figma Developer Docs, VectorNetwork and VectorNode: the confirmed data structure = vertices (x, y, strokeCap, strokeJoin, cornerRadius, handleMirroring), segments (start, end indices, tangentStart, tangentEnd), regions (windingRule NONZERO or EVENODD, loops of segment indices), and the "superset of paths" statement. https://developers.figma.com/docs/plugins/api/VectorNode/
- Alex Harri, "The Engineering behind Figma's Vector Networks": deep walkthrough of the graph structure, regions as loops, and winding rules. https://alexharri.com/blog/vector-networks
- Grida, "Kiwi Schema for the .fig Format": reverse-engineered binary save schema showing vector nodes serialized as vertex, segment, and region arrays. https://grida.co/docs/wg/feat-fig/glossary/fig.kiwi
- NVIDIA GPU Gems 3, Chapter 25, "Rendering Vector Art on the GPU" (Loop and Blinn): tessellating bezier curves into triangles for GPU rendering (inference basis for the rendering pipeline). https://developer.nvidia.com/gpugems/gpugems3/part-iv-image-effects/chapter-25-rendering-vector-art-gpu
- Charles Loop and Jim Blinn, "Resolution Independent Curve Rendering using Programmable Graphics Hardware" (Microsoft Research): GPU-side bezier fill, background for curve-to-triangle rendering. https://www.microsoft.com/en-us/research/wp-content/uploads/2005/01/p1000-loop.pdf
- "GPU-friendly Stroke Expansion" (arXiv 2405.00127): converting stroked bezier paths (with caps and joins) into fillable geometry, background for stroke-to-fill. https://arxiv.org/pdf/2405.00127
