# References: Figma vector networks

Keepers for the 2026-08-27 teardown (the pen/bend geometry model, a shape stored as a
graph instead of a single closed chain of anchor points). Distinct from the four earlier
Figma teardowns: multiplayer cursors (2026-06-18), canvas rendering engine (2026-07-09),
Auto Layout (2026-07-29), and components/instances/library updates (2026-08-19).

## Primary (Figma engineering + Evan Wallace)

- Introducing Vector Networks (Evan Wallace, Figma co-founder). The canonical source. The
  pen tool "was originally introduced in 1987 and has remained largely unchanged since."
  A vector network "improves on the path model by allowing lines and curves between any
  two points instead of requiring that they all join up to form a single chain." Fill:
  Figma "starts with the entire closed part of the network filled in, then lets people
  punch holes in the fill," avoiding the "confusing" winding-number concept. Superset of
  paths, backwards compatible. User study: many people "didn't even notice a difference,"
  the tool "worked as expected," and only noticed when they went back to other tools.
  https://www.figma.com/blog/introducing-vector-networks/
  Mirror: https://madebyevan.com/figma/introducing-vector-networks/
  Medium: https://medium.com/figma-design/introducing-vector-networks-3b877d2b864f

- Delete and Heal for Vector Networks (Jamie Wong, Figma). Vector objects are treated as
  "undirected graphs (or more precisely, as undirected multigraphs with edge identity),
  not as paths of vertices." Delete-and-heal rules by vertex degree: degree 1 = throw away
  the single edge (no sensible heal); degree 2 = join the two adjacent vertices; even
  degree > 2 = "edges are healed by replacing pairs of 'opposite' edges with a new edge";
  odd degree > 2 = "no sensible resolution, so all incident edges are deleted." User
  gesture: Shift + Delete = "Delete and Heal Selection."
  https://www.figma.com/blog/delete-and-heal-for-vector-networks/
  Medium: https://medium.com/figma-design/delete-and-heal-for-vector-networks-d92f176805fb

- Figma Developer Docs, VectorNetwork / VectorNode. The confirmed data structure = three
  arrays. vertices: VectorVertex with x, y, and optional strokeCap, strokeJoin,
  cornerRadius, handleMirroring. segments: VectorSegment with start and end (indices into
  vertices) and optional tangentStart, tangentEnd (bezier control vectors). regions:
  VectorRegion with windingRule ("NONZERO" or "EVENODD") and loops (arrays of segment
  indices). VectorNode is "Figma's most general representation of shape." Stated: a vector
  network is a superset of paths (a vector network can represent everything paths can,
  paths cannot represent everything a vector network can).
  https://developers.figma.com/docs/plugins/api/VectorNode/

## Deep-dives and format

- Alex Harri, "The Engineering behind Figma's Vector Networks": graph structure, regions
  as loops, winding rules, and how each region is defined by loops (ordered edges,
  clockwise or counter-clockwise) plus a winding rule to unambiguously define the region.
  https://alexharri.com/blog/vector-networks

- Grida, "Kiwi Schema for the .fig Format": reverse-engineered binary save format showing
  a vector node serialized as its vertex, segment, and region arrays (the compact graph is
  what is stored and synced, kilobytes, not the derived triangle mesh).
  https://grida.co/docs/wg/feat-fig/glossary/fig.kiwi

- Infinite Canvas tutorial, Lesson 22 (VectorNetwork): an implementation walkthrough of
  the vertices/segments/regions model on an infinite canvas.
  https://infinitecanvas.cc/guide/lesson-022

## Rendering background (inference basis for tessellation to triangles)

- NVIDIA GPU Gems 3, Chapter 25, "Rendering Vector Art on the GPU" (Loop and Blinn):
  tessellating bezier curves into triangles; GPUs "excel at rendering triangles and
  triangular approximations to smooth objects."
  https://developer.nvidia.com/gpugems/gpugems3/part-iv-image-effects/chapter-25-rendering-vector-art-gpu

- Loop and Blinn, "Resolution Independent Curve Rendering using Programmable Graphics
  Hardware" (Microsoft Research).
  https://www.microsoft.com/en-us/research/wp-content/uploads/2005/01/p1000-loop.pdf

- "GPU-friendly Stroke Expansion" (arXiv 2405.00127): stroke-to-fill conversion of stroked
  bezier paths (with caps and joins) into fillable geometry for GPU rasterization.
  https://arxiv.org/pdf/2405.00127

## The one-line takeaway

A shape is a graph (vertices, segments-between-any-two-vertices, fillable regions), not a
single closed chain. That one swap removes the degree-two constraint (points can hold any
number of edges), makes fill a per-region toggle instead of a winding-direction puzzle,
and makes delete "heal" the graph. The compact graph is authored and synced; the GPU
triangle mesh is a derived, cached, incrementally re-tessellated byproduct.
