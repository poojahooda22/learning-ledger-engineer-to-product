# References: Notion database formulas (the formula engine and dependency graph)

Keeper links for the 2026-10-07 teardown. Notion confirmed facts, plus the
well-documented spreadsheet-engine class used for the clearly-labeled inference about
Notion's unpublished recalculation internals.

## Notion (confirmed: data model and scale)

- The data model behind Notion's flexibility (everything is a block, consistent Postgres structure, block attributes, parent/workspace): https://www.notion.com/blog/data-model-behind-notion
- Herding elephants: Lessons learned from sharding Postgres at Notion (workspace ID partition key, app-level sharding, VACUUM stall + TXID wraparound trigger): https://www.notion.com/blog/sharding-postgres-at-notion
- Storing 200 billion entities: Notion's data lake project (ByteByteGo summary; ~20B blocks 2021 -> 200B+ 2024, doubling every 6-12 months, 32x15 then 96x5 shard layout): https://blog.bytebytego.com/p/storing-200-billion-entities-notions

## Notion (confirmed: the formula feature)

- Formulas 2.0, what's changed (new extensible language, rich types pages/dates/people/lists, lets() and map(), cross-database access, text-migration behavior): https://www.notion.com/help/guides/new-formulas-whats-changed
- Notion Help, Formulas (rollups read-only, recalc automatically when related entries change): https://www.notion.com/help/formulas

## The spreadsheet recalculation class (grounding for the inference half)

- Microsoft, Excel recalculation (PRIMARY, authoritative for this class): dependency tree, calculation chain "in the order in which they should be calculated," dirty cells and transitive marking, chain reordering, circular-reference detection, smart vs full recalc: https://learn.microsoft.com/en-us/office/client-developer/excel/excel-recalculation
- HotXLS incremental recalculation and dependency graph (build graph once, propagate dirtiness, evaluate dirty subgraph once in topological order via Kahn, cost tracks dirty cells not workbook size): https://blog.loslab.com/en-us/hotxls-component/hotxls-incremental-recalculation-dependency-graph.html
- grid.is recalculation algorithm (pull/priority-queue, mark dirty then process from a dependency-ordered queue, dynamic dependencies reorder the queue): https://docs.grid.is/apiary/explain/recalculation-algorithm/
- lord.io, Spreadsheets (heap-scheduled dirty-only vs full recompute trade-off; modern Excel combines dirty marking with topological sorting): https://lord.io/spreadsheets/

## Scale/context (secondary, treat as reported)

- Notion 100M users milestone (2024) and revenue/valuation context (aggregator, not Notion-confirmed): https://synergystartup.substack.com/p/notion-growth-strategy-behind-100m

## Note on confirmed vs inference

Confirmed: Notion's block data model, workspace-ID sharding and scale numbers, the
Formulas 2.0 language and its rich types, rollups recomputing automatically. Not
published by Notion and therefore labeled inference in the report: the parser/AST,
the dependency-graph dirty-marking and topological recompute, the client-side
evaluation split, and incremental rollup aggregation. The inference is grounded in
the spreadsheet-engine class above, and Notion's observable behavior (live updates as
you type, only-affected cells recompute, self-reference rejected) matches it.
