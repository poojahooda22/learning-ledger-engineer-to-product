# References: Figma multiplayer cursors and presence

Keepers for the 2026-06-18 teardown.

## Primary (Figma engineering)

- How Figma's multiplayer technology works (Evan Wallace, cofounder). The canonical
  source. Document as a tree of objects, each with ID + properties; property level
  last writer wins ("like a last writer wins register, but no timestamp needed
  because the server defines the order of events"); reparenting via a parent
  property on the child to preserve identity; cursors/presence are ephemeral and
  never saved. https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
  Mirror: https://madebyevan.com/figma/how-figmas-multiplayer-technology-works/

- Realtime editing of ordered sequences (Evan Wallace). Fractional indexing:
  child position as a fraction in (0,1), order = sorted positions, insert =
  average of neighbors. Uses arbitrary precision strings in base 95 (printable
  ASCII) instead of 64-bit doubles so precision never runs out.
  https://www.figma.com/blog/realtime-editing-of-ordered-sequences/

- How Mozilla's Rust dramatically improved our server side performance.
  Multiplayer server rewritten TypeScript -> Rust. One process per document,
  fixed machines x workers, each document on one worker. TS was single threaded,
  one slow op stalled the whole worker -> latency spikes; Rust removed the stalls.
  https://www.figma.com/blog/rust-in-production-at-figma/

- Multiplayer editing in Figma (product framing).
  https://www.figma.com/blog/multiplayer-editing-in-figma/
- Making multiplayer more reliable (reconnection, journaling).
  https://www.figma.com/blog/making-multiplayer-more-reliable/

## Secondary (deep dives)

- Sujeet Jaiswal, Figma multiplayer infrastructure. Wire rate ~30 Hz vs 60 Hz
  render via requestAnimationFrame interpolation; ~200 visible cursor cap;
  presence broadcast over the same WebSocket but never journaled/written to S3.
  https://sujeet.pro/articles/figma-multiplayer-infrastructure
- Mark Skelton, Building Figma multiplayer cursors. Cursor message format,
  throttling, interpolation between the last two samples.
  https://mskelton.dev/blog/building-figma-multiplayer-cursors
- zknill, So you want to build Miro and Figma style collaboration. Why cursor
  data is ephemeral, pub/sub fan out, throttle 50-80 ms + interpolation.
  https://zknill.io/posts/ephemeral-collaboration/

## The one idea worth stealing

Two data problems wearing one feature. The document must be permanent and
conflict free (tree of objects, property level last writer wins, one server as
referee). The cursors must be fast and disposable (ephemeral broadcast, throttle
on the wire, interpolate on the client, cap the count). Never let the cheap path
touch the expensive path. For Rare.lab: graph = document pipeline, live
cursor/parameter scrubbing = ephemeral pipeline, commit only on mouse up, let the
GPU's existing per frame redraw carry the smoothness.
