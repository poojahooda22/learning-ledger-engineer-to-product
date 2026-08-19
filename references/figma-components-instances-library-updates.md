# References: Figma components, instances, and library updates

Saved for the 2026-08-19 teardown.

## Primary (Figma engineering / official)

- How We Rebuilt the Foundations of Component Instances (March 2026) — the Materializer system, blueprints, derived subtrees, push/pull dependency tracking, invalidation, re-materialization. The single best internal source.
  https://www.figma.com/blog/how-we-rebuilt-the-foundations-of-component-instances/
- Components in Figma (blog) — main component vs instance, overrides, publishing, update flow.
  https://www.figma.com/blog/components-in-figma/
- How Figma's multiplayer technology works (blog) — document = tree of objects, Map<ObjectID, Map<Property, Value>>, per-property last-writer-wins, simplified CRDT-inspired model.
  https://www.figma.com/blog/how-figmas-multiplayer-technology-works/
- Evan Wallace mirror of the multiplayer post (Figma co-founder):
  https://madebyevan.com/figma/how-figmas-multiplayer-technology-works/

## Official help / best practice

- Guide to components in Figma (Help Center):
  https://help.figma.com/hc/en-us/articles/360038662654-Guide-to-components-in-Figma
- Explore component properties (Help Center):
  https://help.figma.com/hc/en-us/articles/5579474826519-Explore-component-properties
- Component architecture in Figma (Best Practices):
  https://www.figma.com/best-practices/component-architecture/

## Real-world failure modes (Figma forum)

- Library changes not updating instances in consuming files:
  https://forum.figma.com/ask-the-community-7/changes-published-in-library-are-not-updating-component-instances-in-other-files-that-use-the-library-19374
- "Some components were reparented to avoid a dependency cycle" (the DAG/cycle constraint surfacing):
  https://forum.figma.com/ask-the-community-7/error-message-some-components-were-reparented-to-avoid-a-dependency-cycle-26738
- Nested variant instances reset after a library update (override-preservation edge cases):
  https://forum.figma.com/ask-the-community-7/nested-variant-instances-within-a-component-are-reset-after-the-library-update-35195

## Key facts pulled

- Document is a DOM-like tree, each object is a flat property map, keyed by stable id.
- Conflict resolution is per-property last-writer-wins; different properties / different objects never conflict.
- An instance is NOT a deep copy: it is a pointer to a master + an override map (the exception list). Children are materialized on demand.
- Materializer = generic system that builds and maintains derived subtrees from other sources of truth; instance blueprint says what it inherits, which overrides apply, which children exist.
- One master edit -> look up dependents in a dependency index -> mark dirty -> re-materialize only those. Cost scales with affected instances, not document size.
- Cross-file: publish takes an immutable versioned snapshot; server-side subscription table notifies consuming files; each file pulls opt-in and re-materializes locally. Load-shedding by design.
- Cycles (A embeds B embeds A) are illegal; Figma reparents to avoid dependency cycles.
