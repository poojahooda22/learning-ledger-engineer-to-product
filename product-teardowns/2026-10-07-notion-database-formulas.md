# Notion database formulas: the tiny programming language and dependency graph that make a column do your math

How typing `prop("Hours") * prop("Rate")` into one column makes every row compute
itself, and keep itself correct the instant you change an input, without ever
re-running the whole database. This is the formula property (the computed column),
not the database views and filters (that was its own teardown on 2026-08-02) and
not the slash command and block model (2026-06-25).

Date: 2026-10-07
Product: Notion
Feature: Database formulas (the formula property: parse, evaluate, and the dependency graph that recomputes only what changed)

## 1. The user

Rohan is a freelance product designer in Pune. It is the last evening of the month
and he is doing the thing every freelancer hates: invoicing. He keeps a Notion
database called "Invoices", one row per job. Each row has the client, the hours he
worked, his hourly rate, the amount, the 18 percent GST, and the final total. He has
a second database called "Clients", and each invoice is linked to one client so he
can see how much he has billed each of them this year.

He just realized he undercounted one job. The "Website redesign for Chai Point" row
says 10 hours, but he actually did 12. He clicks the Hours cell, types 12, and
presses Enter. In the same blink, the Amount jumps, the GST jumps, the Total jumps,
and over in the Clients database the "Total billed this year" number for Chai Point
goes up too. He did not touch any of those. He changed one number and five other
numbers corrected themselves. He moves on without thinking about it. That not
thinking about it is the entire feature working.

## 2. The real problem

Rohan is not a programmer. He does not want a spreadsheet full of fragile cell
references like `=C2*D2` that break the moment he inserts a row. He wants to say,
in plain terms, "the amount is hours times rate, the tax is 18 percent of the
amount, the total is amount plus tax," write that rule once, and have it hold for
every invoice he will ever make, forever.

The pain the feature removes is the pain of keeping numbers in sync by hand. Change
the hours and you must remember to also fix the amount, the tax, and the total, and
the client summary, or your invoice is wrong and you lose money or trust. Humans are
terrible at this. We forget one of the five. A friend would describe the problem
like this: "I don't want to recalculate my whole life every time one thing changes.
I want to change the one thing and have everything that depends on it just follow."

That sentence, "everything that depends on it just follow," is secretly a computer
science problem. It is the exact problem a spreadsheet engine solves, and it is the
exact problem a shader node graph solves. Notion formulas are that engine wearing
the clothes of a friendly text box.

## 3. The feature in one sentence

A formula property lets you write one small expression for a column, and Notion
evaluates it for every row and automatically recomputes exactly the cells that
depend on whatever you just changed.

## 4. Jobs to be done

What Rohan is really hiring the formula column to do:

- "Let me write the rule once and never maintain the numbers by hand." Write
  `prop("Hours") * prop("Rate")` once, get a correct Amount on all 240 invoices.
- "Keep everything correct the instant I change an input." Fix the hours, and the
  tax and total fix themselves before he can blink.
- "Let me build a little app, not just take notes." A finance tracker, a habit
  tracker, a content calendar that computes "days until deadline" by itself. The
  formula is what turns a table into software.
- "Do not make me learn to code." Plain function names, a reference to a column by
  its human name, and an answer that appears live as he types.

## 5. How it works for the user

Rohan adds a new property to the Invoices database and picks the type "Formula". A
box opens. He types `prop("Hours") * prop("Rate")`. As he types, Notion shows him a
live preview of the result for the current row. He clicks Done. Instantly every row
in the table shows its own Amount: the Chai Point row shows 12 times his rate, the
Zomato row shows its own hours times rate, and so on. Each row ran the same rule on
its own data.

He adds a Tax formula: `prop("Amount") * 0.18`. He adds a Total:
`prop("Amount") + prop("Tax")`. Now he has a chain. Amount feeds Tax, Amount and Tax
feed Total. He never has to tell Notion the order to compute them in. He just states
the three rules and they settle into the right values.

Since the 2023 "Formulas 2.0" release (confirmed, Notion's own help guides),
formulas can do much more than numbers. They can return a date, a person, a page,
or a whole list. They can reach across a relation into the linked Clients database
and pull back a field, for example a client's email or the sum of all their
invoices. He can define a reusable intermediate value with `lets()` and transform a
list with `map()`. The formula is a small language now, not just arithmetic.

## 6. The actual flow, step by step

Here is the real edit, tap by tap, when Rohan changes 10 to 12 in the Chai Point row.

1. He clicks the Hours cell, deletes 10, types 12, presses Enter.
2. The Hours value for that one row changes. This is a plain data write, not a
   formula.
3. Notion now has to answer one question: which other cells are now wrong? Not the
   whole table. Just the ones that read Hours, directly or through a chain.
4. Amount in that row reads Hours, so Amount is now stale. Amount recomputes: 12
   times the rate.
5. Tax reads Amount, so Tax is now stale. It recomputes from the new Amount.
6. Total reads Amount and Tax, both just changed, so Total recomputes last.
7. The relation to Clients means the Chai Point client page has a rollup, "Total
   billed this year", that sums the Amount of all linked invoices. That sum now
   includes a bigger Amount, so the rollup recomputes too.
8. The screen updates all of these together. To Rohan it looks instant and
   simultaneous. Under the hood it happened in a strict order: Hours, then Amount,
   then Tax, then Total, then the client rollup. Order matters, and the engine
   figured out the order so he did not have to.

The one thing that did not happen is important: the other 239 invoices were not
recomputed. The Zomato row, the Swiggy row, every untouched row, sat still. Only
the cells downstream of the one change did any work.

## 7. Under the hood, like the engineer

A formula column is two problems stitched together, and they fail and scale for
different reasons, so keep them separate in your head.

- The language half: turn the text `prop("Amount") * 0.18` into something runnable,
  then run it against one row's data. Parsing and evaluation.
- The dependency half: when one cell changes, recompute exactly the cells that
  depend on it, in the right order, without re-running everything, and refuse any
  rule that depends on itself. A dependency graph and incremental recomputation.

Notion has published a great deal about its block data model and its database
sharding, but it has not published the internals of the formula recalculation
engine. So below, the data model and scale facts are confirmed and cited, and the
recalculation mechanics are clearly labeled inference, grounded in the very
well documented class of spreadsheet engines (Microsoft's own description of Excel
recalculation, plus several open calc engines). This is how this class of problem
is solved, and Notion's observable behavior matches it exactly.

### The language half: text to syntax tree to value

When Rohan types `prop("Amount") + prop("Tax")`, the engine does what every language
does. It tokenizes the text into pieces (the function `prop`, the string `"Amount"`,
the operator `+`), then parses those tokens into an abstract syntax tree, an AST. The
AST for his Total formula is a small tree: a plus node at the root, with two children,
each a `prop` lookup. This is inference, but it is the only sane way to run a
language, and Notion's Formulas 2.0 announcement explicitly says they built a "new,
more extensible formula language" with real data types, which is exactly what you do
when you move from string hacking to a proper parser and typed AST.

Why a tree and not just left-to-right text evaluation? Because precedence and nesting
need structure. `prop("Amount") * 0.18 + 5` must multiply before it adds. The tree
encodes that: the plus is the root, the multiply is a child. Walk the tree bottom up
(post-order traversal) and you evaluate children before parents, so multiply happens
before add, automatically. The same tree, evaluated against each row's data, gives
each row its own answer. The Chai Point row binds `prop("Amount")` to its Amount, the
Zomato row binds it to a different Amount, one tree, 240 evaluations.

Parsing happens once when he saves the formula. Evaluation happens many times, once
per row, and again whenever an input changes. Parse once, evaluate often. That split
is the first performance idea and it matches the whole ledger's spine: do the
expensive structural thinking rarely, keep the hot path cheap.

### The dependency half: the graph that knows what follows what

This is the heart. Picture every cell that holds a formula as a node, and draw an
arrow from a cell to each cell it reads. In Rohan's row:

- Amount reads Hours and Rate. Arrows: Hours to Amount, Rate to Amount.
- Tax reads Amount. Arrow: Amount to Tax.
- Total reads Amount and Tax. Arrows: Amount to Total, Tax to Total.
- The Chai Point client rollup reads the Amount of every linked invoice. Arrow:
  Amount to that rollup.

That is a directed graph. Because a sane formula cannot ultimately depend on itself,
it is a directed acyclic graph, a DAG. The DAG is the data structure that makes "just
follow" possible.

When Hours changes, the engine does three moves (this is the standard spreadsheet
recalc, confirmed for Excel by Microsoft's documentation, inferred for Notion):

1. Mark dirty. Follow the arrows out of Hours and mark every reachable cell as
   needing recomputation. Hours points to Amount. Amount points to Tax, Total, and
   the client rollup. Tax points to Total. So the dirty set is {Amount, Tax, Total,
   rollup}. The marking is transitive: dirtiness flows downstream. Microsoft says
   exactly this for Excel, "if B1 depends on A1 and C1 depends on B1, changing A1
   marks both B1 and C1 as dirty." The 239 untouched invoices are never reached by
   an arrow, so they are never marked, so they do no work.

2. Order the work. You cannot compute Total before Amount, or you would use a stale
   Amount. So sort the dirty set into an order where every cell comes after the cells
   it reads. This is a topological sort of the dirty subgraph. A clean way is Kahn's
   algorithm: repeatedly take a cell whose inputs are all already computed, compute
   it, remove it, repeat. Here it yields Amount, then Tax, then Total, with the
   rollup slotting in after Amount. Microsoft calls Excel's version the "calculation
   chain", the list of formula cells "in the order in which they should be
   calculated", and notes Excel reorders the chain when a formula turns out to read a
   cell not yet computed. Same idea.

3. Evaluate in order. Walk the ordered dirty set, evaluate each cell's AST against
   its row's current data, write the result. Because of the order, every cell reads
   already-fresh inputs. Each dirty cell is computed exactly once. The cost of the
   update is proportional to the number of cells that actually changed, not the size
   of the database. This is the single most important property. An open calc engine
   (HotXLS) states the same claim plainly: evaluate the dirty subgraph once in
   topological order so cost "tracks the number of dirty cells rather than the
   workbook size."

### Why a DAG and not a list or a hash map

A plain list of formulas loses the ordering information. You would have to guess or
re-sort the entire table on every change. A hash map from cell to formula gives you
O(1) lookup of a formula but says nothing about what depends on what, so you still
cannot know that Total must wait for Amount. The graph is the only structure that
stores the relationships, and the relationships are the whole point. In practice you
keep both: a hash map from each cell to its node for fast lookup, and the graph edges
for the ordering. Hash map for "where is this", graph for "what follows this".

### Cycles: the one rule the engine must refuse

What if Rohan accidentally writes a Total that references itself, or an A that reads
B while B reads A? Now there is a cycle in the graph, and the topological sort has no
valid starting point, there is no cell whose inputs are all ready. Every serious calc
engine detects this and refuses rather than looping forever. Microsoft: Excel
"detects the circular reference and warns the user." Notion likewise rejects a
self-referential formula with an error instead of hanging. Detecting a cycle is cheap
at the moment of the edit (try the topological sort, if cells remain that can never
become ready, you have a cycle), and detecting it at edit time is far better than
discovering it mid-recompute. This is the same lesson as Figma refusing a reparent
that would create a cycle in the document tree (2026-10-04): a single authority
checking for cycles at write time keeps the whole structure sane.

### Relations and rollups: edges that cross databases

Rohan's "Total billed this year" is a rollup: it reaches through the relation into
every linked invoice and sums their Amount. In graph terms this is an edge from each
invoice's Amount cell to the client's rollup cell, and the rollup is an aggregation
node (a sum over many inputs). Notion confirms rollups "are read-only and recalculate
automatically when related entries change," which is the dependency graph doing its
job across two databases. Formulas 2.0 made these cross-database reads first class:
a formula can pull a related page's property directly.

This is also where the real scaling pain hides, below.

### Where the work actually runs (well grounded inference)

Notion has not published the client-versus-server split for formula evaluation, so
this is labeled inference, but the behavior is strongly observable. Notion is a
client-heavy app. When Rohan opens the Invoices view, the rows for that view are
loaded to his browser or app, and the formula results update live as he types,
before any round trip could complete. That means the formula language runs in the
client, evaluating the AST over the rows currently loaded. The server stores the
inputs (the Hours, the Rate) as blocks, and stores or caches the derived values so
other views and the public API can read them without every reader re-running the
math. The derived value is a cache; the inputs are the truth. This matters for sync,
below.

### The data model underneath (confirmed)

Everything in Notion, confirmed by Notion's engineering blog, is a "block" stored in
Postgres "with a consistent structure", even though block types look different in the
editor. A database row is a block (a page), and each property value hangs off it. A
formula property is a definition stored on the database schema; the per-row value is
derived. Blocks carry a type, their properties, their content, and a parent, and
every block belongs to exactly one workspace. That last fact, confirmed in Notion's
"Herding elephants" sharding post, is the key to the scale story: the workspace is
the unit of isolation.

### The scale story at three tiers

What grows here is not a catalog of millions of shared items. It is three things at
once: the number of rows in a database, the number of formula properties (the width
and depth of the dependency graph), and the fan-out of relations and rollups that
cross databases. And it all sits inside a service that, confirmed, passed 100 million
users in 2024 and stores more than 200 billion blocks in Postgres, up from about 20
billion in 2021, with block data doubling every 6 to 12 months (Notion engineering,
via its data lake and sharding posts).

Tier 1, about 1,000 rows (Rohan's invoices, a personal tracker). Honestly, you could
recompute the entire table on any change and nobody would notice. A full re-evaluation
of a small DAG is microseconds. Building a dirty-marking incremental engine here is
over-engineering. But you build it anyway, because it is the same code path that has
to survive tier 3. The naive "recompute everything" is correct and fast at this size,
and a real example proves it: changing one habit-tracker checkbox and re-running 30
rows of formulas is invisible.

Tier 2, about 100,000 cells (a 10,000-row project database with 10 formula columns,
or a busy team wiki's task database). Now "recompute everything on every keystroke"
starts to jank. If every edit to one cell re-evaluated 100,000 formula cells, the UI
would stutter on every keypress. This is the tier where the dependency graph earns
its keep: an edit to one Hours cell must touch only its downstream cone (a handful of
cells), not the whole table. Relations and rollups now create cross-row and
cross-database edges, so the graph is no longer per-row-independent. And because
evaluation runs client-side over loaded rows, the client has to have the referenced
data loaded; a rollup that sums a related 10,000-row database needs those 10,000
Amounts available or summarized. The fix is exactly the spreadsheet fix: mark dirty,
sort the dirty subgraph, evaluate only that.

Tier 3, 100 million users and 200 billion blocks. The walls are different in kind,
not just degree:

- Multi-tenancy is the first wall, and it is a gift. Every block, and therefore every
  formula and every dependency edge, lives inside exactly one workspace, and Notion
  shards Postgres by workspace ID (confirmed: application-level sharding, grown from
  32 database instances with 15 logical shards each in 2021 to 96 instances with 5
  logical shards each by 2023, which is 480 logical shards). A formula recompute is
  contained inside one workspace's shard. There is no global recompute across 200
  billion blocks, ever, because there is no global dependency graph. The graph that
  matters is tiny: it is Rohan's workspace, a few thousand cells. This is the same
  shard-by-tenant move as Notion search (2026-08-25), Stripe by account, and Gmail by
  mailbox: the tenant is the unit, and inside one tenant the hard problem shrinks back
  to tier 1 size.

- The aggregation monster is the second wall. A rollup that sums a 50,000-row related
  database cannot re-sum 50,000 rows on every keystroke. The survival move is
  incremental aggregation and caching: keep the running sum, and when one linked
  Amount changes by a delta, adjust the cached sum by that delta instead of re-summing
  from scratch. This is the contested-hot-value pattern from the double-entry ledger
  teardown (2026-10-06): do not recompute the aggregate, apply the change to it. (The
  exact mechanism inside Notion is not published, so this is the class-standard
  inference.)

- Sync and conflict is the third wall, and it is why "the inputs are the truth, the
  derived value is a cache" matters. Notion works offline (2026-07-16). Two people, or
  one person on two devices, can edit the same database while disconnected. You must
  never ship the derived Total as the source of truth and try to merge two Totals.
  You sync the inputs (the Hours, the Rate), let each client recompute the formulas
  locally from the merged inputs, and treat the stored derived value as a cache to be
  recomputed, not a fact to be merged. Derived data is never authoritative. Merge the
  causes, recompute the effects.

- Near-real-time feel is the fourth wall. The recompute has to land before the user
  looks away. Because of multi-tenancy the working set is small, because of dirty
  marking the work is proportional to what changed, and because evaluation is
  client-side the result appears without a server round trip. The three ideas
  together are why a 200-billion-block service can make Rohan's five cells update in
  one blink.

The through-line across all three tiers: the expensive structural thinking (parse
the formula, build the graph, shard by workspace) happens rarely or offline, and the
hot path on every edit stays a small dirty-subgraph recompute. Offline think, online
lookup, in a column of numbers.

## 8. The retention and habit mechanic

Formulas are what turn Notion from a notes app into a thing people cannot leave.

The loop: you build a self-updating system once (an invoice tracker, a reading list
with "days since started", a content calendar with "status" computed from dates), and
from then on it maintains itself. Every time you come back, it is already correct. You
did not have to tidy it. That reliability is the reward, and the reward is quiet: it
is the absence of broken numbers. This is invisible-craft trust, the same loop as
Netflix picture quality (2026-09-11) and Spotify loudness (2026-08-28). You do not
praise it; you just stop considering alternatives.

Which metric it moves: retention and lock-in, and through them revenue. A database
full of formulas is a small app you wrote, and rewriting it in another tool is real
work, so the switching cost climbs with every formula you add. That is the deepest
moat Notion has. It also drives team-seat revenue, because the person who builds the
finance dashboard pulls their whole team into the workspace to use it.

A real observed example of the mechanic working: Notion's template gallery is full of
formula-heavy templates, habit trackers, finance dashboards, CRMs, that creators
share and thousands of people duplicate. The moment you duplicate one and see its
numbers update themselves on your own data, you are activated. That shared,
self-computing template is a viral invite, the same growth engine as a Figma share
link (2026-10-04). The activation "aha" is not writing your first note; it is watching
your first rollup update itself.

The trust-cracker is specific and brutal: one wrong total, or one value that is
visibly stale after you changed its input, and you stop trusting every number in the
database. Derived data has to be right every single time, because its whole value was
that you did not have to check it. The cycle-rejection, the correct ordering, and the
incremental recompute are not nice-to-haves; they are the product.

## 9. The lesson for Rare.lab

A node graph is a dependency DAG of expressions. This is not an analogy, it is the
same data structure. Each node in Rare.lab's editor reads some inputs and produces an
output that feeds downstream nodes, exactly like Rohan's Amount feeding Tax feeding
Total. So Notion's formula engine is the closest structural twin to Rare.lab's
compiler that this ledger has, and its lessons transfer almost directly.

1. Recompile incrementally, never wholesale. When a creator tweaks one node's
   parameter (a color, a frequency, a strength), do not re-evaluate or recompile the
   whole graph. Mark the downstream cone dirty, topologically sort just that cone, and
   recompute only it. The cost of an edit should track the number of affected nodes,
   not the size of the graph, exactly as the dirty-cell cost tracks dirty cells not
   workbook size. On a 500-node graph, dragging one slider should touch ten nodes, not
   five hundred.

2. Split "structure changed" from "value changed". Changing a node's parameter is a
   value change: keep the compiled shape, re-run the affected subgraph, cheap and
   live at 60 frames per second. Adding or rewiring a node is a structure change:
   rebuild the dependency graph and the evaluation order, more expensive, done once at
   edit time. This is precisely Excel rebuilding the calculation chain only on a
   structural change versus just re-running dirty cells on a value change. Do not pay
   the structural cost on every slider drag.

3. Parse and plan offline, look up online. Parse each node's expression into a typed
   AST once, compute the topological evaluation order once per structural change, and
   emit that order plus per-node cost estimates as a build artifact. The per-frame
   runtime should walk a precomputed order and do lookups, never re-plan. This is the
   parse-once-evaluate-often split, and it is the whole ledger's spine applied to
   pixels.

4. Detect cycles at edit time and refuse them, loudly and immediately. The moment a
   creator wires an output back into its own input chain, reject the edge with a clear
   error, the way Notion rejects a self-referential formula and Figma rejects a
   cyclic reparent. A cycle discovered at render time is a hang or a garbage frame; a
   cycle caught at wiring time is a tooltip. Check for the cycle in the one place that
   owns the graph.

5. Treat compiled output as a cache, keep the graph as the truth. When two people
   co-edit a shader graph, or one person edits on two machines, sync the node
   definitions and parameters (the inputs), not the compiled shader (the derived
   value). Merge the causes and recompile the effects locally. Never try to merge two
   compiled outputs, the same reason Notion never merges two Totals.

6. Make aggregation incremental where a node sums or reduces many inputs. A node that
   accumulates over many particles, samples, or child nodes should keep a running
   result and apply deltas when one input changes, not re-reduce the whole set every
   frame. That is the rollup-over-50,000-rows problem, and the fix is the same: adjust
   the cached aggregate by the delta.

One line: build Rare.lab's graph evaluation and recompilation as a dirty-subgraph,
topologically-ordered, incremental recompute with edit-time cycle detection, parse
and plan offline and look up online, and treat the compiled artifact as a cache of a
graph that is the real truth, because a shader node graph and a Notion formula column
are the same dependency-DAG problem wearing different clothes.

## Sources

- Notion, "The data model behind Notion's flexibility" (engineering blog): everything is a block, stored in Postgres with a consistent structure; block attributes and parent/workspace relationships. https://www.notion.com/blog/data-model-behind-notion
- Notion, "Herding elephants: Lessons learned from sharding Postgres at Notion": workspace ID as the partition key, application-level sharding, the VACUUM stall and transaction-ID wraparound trigger. https://www.notion.com/blog/sharding-postgres-at-notion
- Notion data lake write-up (ByteByteGo summary of Notion engineering): ~20 billion block rows in 2021 growing to 200 billion+ by 2024, doubling every 6 to 12 months, shard layout 32x15 then 96x5. https://blog.bytebytego.com/p/storing-200-billion-entities-notions
- Notion Help, "Formulas 2.0: what's changed": the new extensible formula language, rich output types (pages, dates, people, lists), lets() and map(), access to related databases' properties, migration converting rich references to text. https://www.notion.com/help/guides/new-formulas-whats-changed
- Notion Help, "Formulas": rollups are read-only and recalculate automatically when related entries change. https://www.notion.com/help/formulas
- Microsoft, "Excel recalculation" (client developer docs): the dependency tree, the calculation chain "in the order in which they should be calculated," chain reordering, dirty cells and transitive marking, circular-reference detection, smart vs full recalculation. https://learn.microsoft.com/en-us/office/client-developer/excel/excel-recalculation
- HotXLS, "Incremental recalculation and dependency graph": build the graph once, propagate dirtiness along edges, evaluate the dirty subgraph once in topological order (Kahn's algorithm), cost tracks dirty cells not workbook size. https://blog.loslab.com/en-us/hotxls-component/hotxls-incremental-recalculation-dependency-graph.html
- grid.is, "Recalculation algorithm": pull/priority-queue recompute, mark dirty then process from a queue ordered by dependencies, handle dynamically discovered dependencies by reordering. https://docs.grid.is/apiary/explain/recalculation-algorithm/
- lord.io, "Spreadsheets": the trade-off between heap-scheduled dirty-only recompute and full recompute; modern Excel combines dirty marking with topological sorting. https://lord.io/spreadsheets/
- Notion 100 million users milestone (2024) and scale/revenue context (secondary aggregators; treat exact figures as reported, not Notion-confirmed). https://synergystartup.substack.com/p/notion-growth-strategy-behind-100m
