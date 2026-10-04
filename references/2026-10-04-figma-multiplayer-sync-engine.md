# References: Figma multiplayer sync and conflict-resolution engine (2026-10-04)

Keeper links for the Figma multiplayer sync/merge teardown.

## Primary sources (Figma / Evan Wallace)

- **How Figma's multiplayer technology works** (Evan Wallace, Figma CTO). The canonical explainer: rejecting OT, tree of objects with properties, per-property last-writer-wins ordered by the central server, atomic parent-plus-position, server-side cycle rejection, fractional indexing.
  - Figma blog: https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
  - Original on madebyevan.com: https://madebyevan.com/figma/how-figmas-multiplayer-technology-works/

- **Rust in production at Figma** (Evan Wallace, 2018). The TypeScript/Node to Rust multiplayer server rewrite: single-threaded latency spikes, JS VM memory overhead blocking per-document process isolation, ~order-of-magnitude performance gain, ~10x faster serialization, per-document isolation made affordable.
  - https://www.figma.com/blog/rust-in-production-at-figma/
  - Medium mirror: https://medium.com/figma-design/rust-in-production-at-figma-e10a0ec31929

- **Realtime editing of ordered sequences** (Figma blog). Fractional indexing deep dive: fraction between 0 and 1 exclusive, arbitrary-precision fractions, base-95 ASCII string encoding, insert-between always exists.
  - https://www.figma.com/blog/realtime-editing-of-ordered-sequences/

- **Multiplayer editing in Figma** (public launch post).
  - https://www.figma.com/blog/multiplayer-editing-in-figma/

## Secondary syntheses

- HelloInterview, "How Figma Built Multiplayer Editing on Simplified CRDTs." Useful system-design framing (calls the approach "simplified CRDTs").
  - https://www.hellointerview.com/learn/system-design/in-the-wild/figma-multiplayer

- Sujeet Jaiswal, "Figma: Building Multiplayer Infrastructure for Real-Time Design Collaboration."
  - https://sujeet.pro/articles/figma-multiplayer-infrastructure

## Key confirmed facts to reuse

- Document = tree of objects, each object = ID + map of properties. Edit = (object_id, property_key, new_value).
- Rejected operational transforms (OT, Google Docs style) as unnecessarily complex for a startup. Not a full peer-to-peer CRDT either; central server linearizes edits.
- Per-property last-writer-wins. "Last" = last to arrive at the central server. No vector clocks; the server is the clock.
- One central server process/worker per document. Each worker owns a fraction of currently-open documents. Shard key = the document.
- Reparenting is the hardest part: parent-link + position is a single atomic property; server rejects parent updates that would create a cycle.
- Sibling ordering = fractional indexing, arbitrary-precision fractions encoded as base-95 strings, sorted as strings.
- Server is dumb: syncs properties it does not understand, so clients add new property types without a server deploy. Rendering/layout/vector math live in the client (WASM canvas engine, see 2026-07-09 teardown).
- Scale wall: a single document cannot be sharded across workers (one authority per doc), so a huge single file is a vertical scaling ceiling.

## Not public (labeled inference in the report)

- Exact wire format of delta messages.
- Precise persistence / snapshot cadence.
- Current worker counts and per-worker memory figures.
