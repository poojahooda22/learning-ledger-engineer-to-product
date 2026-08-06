# Netflix homepage rows: the wall of shelves that rearranges itself for you

Date: 2026-08-06
Product: Netflix
Feature: The personalized homepage (the grid of rows: which titles fill each row, the order inside each row, and the order of the rows themselves)

A note on what this teardown adds to the ledger. Almost every ranking feature so
far produces one ordered list: a search results page, a Stories tray, an Explore
grid. The Netflix homepage is a new shape. It is not a list, it is a page: a
two dimensional grid of about 40 rows, each row a mini ranked list of up to 75
titles, and the rows themselves are ranked against each other. So there are
three ranking problems stacked in one screen, and they interact. That is the
whole story.

---

## 1. The user

Ananya, 29, Bengaluru, just finished the last episode of "Stranger Things" at
10:40 on a Tuesday night. The credits roll. She is not ready to sleep and she is
not ready to commit to a two hour movie. She opens the Netflix home screen on her
TV, leans back, and starts thumbing the remote sideways and down through the wall
of thumbnails.

She is not searching. She has no title in mind. She is grazing. The average
Netflix member in exactly this moment spends 60 to 90 seconds browsing, looks at
roughly 10 to 20 titles, considers about 3 of them in any detail, and scans one
or two screenfuls before they either start something or give up and put the phone
down (Gomez-Uribe and Hunt, 2015). Ninety seconds. That is the entire budget the
homepage has to work with.

---

## 2. The real problem

Netflix has a catalog in the low thousands of titles in any one country. That
sounds small next to Amazon's 350 million items. It is not the problem. The
problem is the opposite of search. In search you typed "watch" and told the
system what you want. Here Ananya has typed nothing. She wants to be shown
something she did not know to ask for, fast, before the 90 second clock runs out
and she opens a different app.

A plain A to Z list of 3,000 titles is useless: she would scroll for ten minutes.
A single "most popular" list is useless too: it shows the same 20 blockbusters to
everyone, and Ananya, who just watched a whole season of supernatural horror, does
not want the same grid as her uncle who watches cricket documentaries. And a
single long personalized list still fails, because a good recommendation for her
right now is not one thing. It is "more supernatural horror like the show you just
finished" AND "that thriller you paused halfway through last week" AND "new this
week in India" AND "because you liked dark comedies." Those are different reasons,
and a flat list cannot show the reason. The row is the reason. The homepage
problem is: pick the handful of reasons that matter to this person tonight, fill
each reason with the right titles, and stack the reasons so the best one is under
her thumb first.

---

## 3. The feature in one sentence

The Netflix homepage is a per member two dimensional page where the titles inside
each row, the order of titles in a row, and the order of the rows down the screen
are all ranked live for you, so the thing you are most likely to press play on
lands in the top left.

---

## 4. Jobs to be done

- "Show me something I will actually watch tonight without me having to think."
- "Remind me of the thing I already started so I can just continue."
- "Give me a reason, not just a title, so I trust the pick" (the row header "Because
  you watched Stranger Things" is the reason).
- "Make the wall feel fresh every time I open it so it is worth opening."
- "Do not make me scroll. Put the best bet where my thumb already is."

---

## 5. How it works for the user

Ananya opens the app. The screen paints top to bottom. The first thing she sees
without scrolling is a big billboard at the top (a single promoted title with
its own art), then a "Continue Watching" row with the thriller she paused, then
"Top Picks for Ananya," then a row literally titled "Because you watched Stranger
Things" full of "Dark," "The Haunting of Hill House," "Midnight Mass," then
"Trending Now," then "Indian Movies and TV," then genre rows going down for a
long way (about 40 rows in all).

She never sees the catalog. She sees reasons, each reason a row, each row already
sorted so its strongest title sits on the left where the row starts. She moves
right inside a row to see more of that one reason, or down to switch to a
different reason. Every account gets a different set of rows in a different order.
Her uncle's first non billboard row might be "Documentaries" and "Because you
watched a cricket film." Same app, same catalog, two completely different walls.

---

## 6. The actual flow, step by step

1. Ananya's TV app asks the Netflix backend for her home page for this profile,
   on this device, in this country.
2. The backend does not compute 3,000 titles times 40 rows from scratch in that
   instant. Almost all of the heavy thinking already happened offline and is
   sitting precomputed. The live request is mostly assembly and lookup.
3. Each candidate row has already been built by its own algorithm: Continue
   Watching, Top Picks (from the Personalized Video Ranker), Trending Now,
   Because You Watched Stranger Things, genre rows, and so on. Netflix generates
   far more candidate rows than fit, thousands of them, per member.
4. A page generation step chooses which of those thousands of rows to actually
   show, in what order, removes duplicate titles across rows, and keeps the page
   diverse so two near identical horror rows do not sit back to back.
5. For each title that survives, a separate system picks the single best artwork
   image for Ananya (covered in the 2026-06-19 artwork teardown, out of scope
   here but it plugs into the same page).
6. The finished page, a small ordered structure of rows and title ids and image
   urls, is sent to the TV. The TV just draws it. The TV never sorts anything and
   never sees the catalog.
7. As she scrolls down, lower rows can be fetched lazily so the first screen paints
   fast and the rest fills in as she goes.

The one line that matters: by the time Ananya's remote wakes up, the ranking is
already done. The live path is a keyed read plus a light assembly, not a
computation over the catalog. This is the offline-think, online-lookup spine that
runs through this whole ledger.

---

## 7. Under the hood, like the engineer

### The page is three ranking problems, not one

Break the screen apart and you get three distinct questions:

1. Within a row: given this row's reason (say "Because you watched Stranger
   Things"), what order do the titles go in? This is a 1D ranked list, the same
   shape as every other ranking feature in this ledger.
2. Filling a row: which titles even belong in this row? This is candidate
   generation, the matching half.
3. Across rows: of the thousands of candidate rows we could show, which ~40 do we
   show, and in what vertical order? This is the genuinely new problem, ranking a
   set of lists against each other on a 2D page.

Netflix built a named algorithm for each. Facts here are from Gomez-Uribe and
Hunt's "The Netflix Recommender System" (ACM TMIS, 2015) and the Netflix Tech
Blog post "Learning a Personalized Homepage" (2015), with the 2026 update from
the "GenPage" post.

### The row builders (matching, the candidate generators)

- Personalized Video Ranker (PVR). This is the workhorse. PVR takes the whole
  catalog and produces a personalized order of it for one member. It runs
  per genre and per member, so when the "Indian Movies" row needs filling, PVR
  hands back Ananya's personal ordering of Indian titles. PVR deliberately blends
  personal signals with unpersonalized popularity, so it never strays too far
  from broadly good titles. Think of PVR as a giant precomputed sort key per
  member: the row just reads the top slice.
- Top-N Video Ranker. The same idea but tuned only for the very head of the
  ranking, because the "Top Picks for Ananya" row only ever shows the top handful,
  and getting positions 1 to 10 exactly right matters more than the shape of the
  long tail. Optimizing the head is a different objective than optimizing the
  whole order, so it is a different model.
- Trending Now. Short window popularity, blended with personalization. It catches
  two kinds of bursts: seasonal (romance titles the week of Valentine's Day) and
  event driven (a news event that sends people to a related documentary). The time
  window is the whole point, so this is a streaming aggregate over recent plays,
  not the same slow monthly signal PVR uses.
- Continue Watching. Titles Ananya started but did not finish, ordered by how
  likely she is to resume: signals include how long since she watched, where she
  stopped (finished the season versus bailed 8 minutes in), and which device.
  This is the feature the 2026-07-27 teardown covered from the storage side; here
  it is just one more row builder feeding the page.
- Because You Watched (BYW) and video-video similarity. Netflix precomputes, for
  each title, a ranked list of similar titles (an unpersonalized "people who
  watched A also watched B" table, the same item-to-item co-occurrence idea as the
  2026-07-12 Amazon teardown). BYW seeds a row from a specific title Ananya
  actually finished, which is why the row can honestly say "Because you watched
  Stranger Things" and fill it with "Dark" and "Midnight Mass."

Each of these is a candidate row. Crucially, each row builder can score its own
titles independently and cheaply, and most of that work is precomputed offline.
A member has thousands of candidate rows available before they ever open the app.

### The hard half: page generation (ranking rows against each other)

Now the new problem. You have thousands of candidate rows. The screen holds about
40. Which 40, in what order?

The naive answer is greedy: score every row by how good its best titles are, sort
the rows by that score, take the top 40. This is wrong, and the reason it is wrong
is the entire lesson of this teardown: rows interact.

- Duplication. PVR's "Top Picks" row and the "Because you watched Stranger
  Things" row might both want to show "Dark." Seeing "Dark" twice on one screen
  wastes a slot and looks broken. So the page cannot rank rows independently. The
  value of a row depends on what the rows above it already used.
- Diversity. Two horror rows stacked back to back are redundant even if each is
  individually excellent. The second one adds almost nothing, because Ananya
  already saw the best horror in the first. A good page spreads reasons out.
- Stopping power. The job of the page is not to be uniformly good, it is to make
  her stop and press play somewhere in the first screen or two. A row that is a
  perfect fit but sits at row 30, below the 90 second cliff, is worth almost
  nothing tonight.

So Netflix models the page, not the row, as the unit of quality. Two design
choices came out of the 2015 work:

Choice one, stage-wise versus page-wise. The simple build is stage-wise: first
pick and order the rows using only row level scores, then fill each row. The
better build is page-wise: score a candidate row in the context of the partial
page built so far, so "how much does adding this row improve the whole page right
now" replaces "how good is this row in isolation." Page-wise is what lets the
system say "skip the second horror row, its marginal value given the first horror
row is low."

Choice two, a 2D metric that respects how people actually browse. You cannot
tune a page you cannot measure, and the usual 1D ranking metrics (NDCG, MRR)
assume a single top to bottom list. A page is browsed differently: eyes go across
a row, then drop to the next row, in a cascade. So Netflix extended the standard
metrics to two dimensions and adapted Expected Reciprocal Rank (ERR) to a 2D
navigation model, assigning each cell (row i, column j) a probability that the
member's eyes actually reach it, decaying as you go right and as you go down, and
crediting a title by how likely it is to be both seen and played. In their
published comparison, the personalized page layout (the blue line) beat the old
rule based layout (the red line) on recall at a fixed screen position. The metric
is the product, because it encodes the 90 second cliff directly: a title at row 30
gets almost no "probability of being seen" weight, so the optimizer stops caring
about it, which is exactly correct.

Concrete walk. Ananya's page is being built. Billboard is fixed at top. Continue
Watching goes next because its "probability of resume" is very high (she paused a
thriller 40 minutes in yesterday, that is nearly a guaranteed play). Then Top
Picks. Then the page generator considers a BYW Stranger Things row: high value,
add it, and mark "Dark," "Midnight Mass," "The Haunting of Hill House" as used so
no later row repeats them. Next it considers a second horror-heavy genre row:
individually strong, but most of its best titles are already used or redundant
with the BYW row above, so its marginal page value is low and it gets pushed far
down or dropped, and a "Trending in India" row (fresh, non redundant, decent
personal fit) takes the slot instead. That swap is page-wise generation doing its
job.

### 2026: GenPage, from a modular pipeline to one generative model

The classic pipeline above is modular: separate row builders, then a page
generator that greedily lays rows down one at a time. It works, but greedy
sequential layout is still myopic. Choosing row 3 cannot fully anticipate how it
constrains rows 5 through 40, and hand tuning the interactions (dedup rules,
diversity penalties, business constraints like "surface a new Netflix Original")
gets brittle.

Netflix's 2026 Tech Blog post "GenPage: Towards End-to-End Generative Homepage
Construction" describes the next step: treat building a homepage the way a
language model treats writing a sentence. Represent the page as a sequence of
tokens (rows and the titles inside them), and generate it autoregressively: the
model computes a value for every candidate next token, greedily picks the highest
value token, appends it to the page built so far, and repeats, token by token,
until the whole page exists. Because each new token is chosen conditioned on the
entire page prefix already generated, cross-row interactions (dedup, diversity,
the rising cost of lower positions) are learned by the model instead of hand coded
as rules on top.

They train it with reinforcement learning, framing page construction as a
sequential decision problem that optimizes an aggregate page-level reward, not a
sum of independent row scores. The reward bundles the things that always fought
each other in the rule based system: diversity across rows, stopping power (does
the page actually make you start something), and page-level business constraints.
This is the same "the objective is where the product lives" theme as Zomato's
quantile loss (2026-06-29) and Amazon's delivery quantile (2026-07-25): the
interesting engineering is in what you optimize, not just how fast you compute it.

Honest labeling. PVR, Top-N, Trending, Continue Watching, BYW, video-video
similarity, the 2D page metrics, and the stage-wise versus page-wise framing are
confirmed from Netflix's own 2015 paper and blog. GenPage is confirmed from
Netflix's 2026 post. The exact model sizes, the precise number of candidate rows
per member, the serving latency budget, and the precise offline versus online
split for GenPage are not fully public. Where I said "about 40 rows" and "up to 75
titles per row," those are widely reported figures for the current UI, not
guaranteed constants. The claim that most scoring is precomputed offline and the
live path is assembly is grounded inference from the shape of the system and from
how every other feature in this ledger at this scale is built, and it is consistent
with a page that paints inside a 90 second attention window for hundreds of
millions of members.

### Data structures actually in play

- Per member per genre sorted lists (PVR output): read the top K slice, O(K), the
  expensive sort is offline. Same "precomputed sort key, cheap head read" pattern
  as Spotify Discover Weekly's memory-mapped lookup.
- A video-video similarity table: item id to a short array of neighbor ids, a hash
  map of small posting lists, so BYW is a keyed lookup, not a live similarity
  computation.
- A "used titles" set built up during page generation, for O(1) dedup as each row
  is placed. This is the one bit of genuinely stateful work on the page path, and
  it is tiny (a hash set of a few hundred ids).
- The finished page: a small ordered list of rows, each a short ordered list of
  title ids plus image urls. Kilobytes. That is all the TV receives.
- Trending Now: a streaming counter over a recent time window (the same streaming
  aggregate shape as Uber surge counts per hex or Instagram's engagement counters),
  because "trending" is defined by a moving window, not an all time total.

### The scale story at three tiers

Tier 1, about 1,000 titles, thousands of members. You could score every title for
every member on request and sort in memory. A single box handles it. Page
generation as a fancy 2D optimization is over-engineering here: a couple of fixed
rows and a popularity list would look fine. Nothing breaks. This is why a small
streaming startup does not need any of this machinery, and copying it early would
be pure waste.

Tier 2, about 100,000 members, catalog still small. Now the break is not catalog
size, it is member count times freshness. You cannot recompute PVR for 100,000
members on every home screen open, and you cannot A/B test row-ranking ideas fast
enough if each test needs millions of member-weeks to reach significance. Two
fixes appear. First, precompute: PVR, Top-N, similarity tables, and most candidate
rows are batch computed offline and cached per member, so the live open is a read.
Second, measure cheaply: this is where interleaving earns its place. Instead of
serving algorithm A to one group and B to another and waiting weeks, Netflix
interleaves both rankers inside the same row for the same member (team-draft: a
coin toss decides who contributes the first title, then A and B alternate
contributing their next best not-yet-used title), and credits whichever algorithm
contributed the titles she actually played. Netflix reports this needs over 100
times fewer members than their most sensitive A/B metric to pick the better
ranker. Interleaving prunes the field fast, then a real A/B confirms the winner.
Faster iteration is itself a scaling tool, because at this tier the bottleneck is
how quickly you can improve the ranker, not how quickly you can serve it.

Tier 3, hundreds of millions of members (Netflix is past 300 million), catalog in
the low thousands, tens of thousands of candidate rows per member, a hard 90
second attention budget. The catalog is still small, so the enemy was never
catalog size. The enemies are: the volume of precompute (per member per genre
orderings for hundreds of millions of people), the fan-out of serving a fresh
page fast to all of them, and the combinatorial page generation problem. Survival
moves: push essentially all scoring offline into batch and stream jobs and keep
the live path a keyed read plus assembly; cache the assembled page and refresh it
on a cadence rather than rebuilding from zero on every open; lazy load lower rows
so the first screen paints fast; and, in 2026, replace the brittle hand tuned page
rules with one learned generative model (GenPage) that internalizes dedup,
diversity, and stopping power so engineers stop maintaining a pile of interacting
heuristics. The candidate-row count is the dial that decouples page cost from
catalog size, exactly as candidate-set size did for Amazon and Canva search: you
never optimize over the whole catalog live, only over a bounded set of prebuilt
rows.

### Matching and ranking, restated for a page

Every ranking teardown in this ledger splits into matching (cheap, wide, get a
bounded candidate set) then ranking (expensive, narrow, order the survivors). The
homepage does it twice. Inside a row: matching is "which titles belong in this
row," ranking is PVR or Top-N ordering them. Across rows: matching is "which
candidate rows exist for this member" (the row builders), ranking is page
generation ordering those rows on the 2D screen. The second layer is the part
most ranking systems never need, because most surfaces are one list. A page needs
a ranker whose items are themselves ranked lists.

---

## 8. The retention and habit mechanic

The loop is the 90 second graze itself. Ananya opens Netflix with no plan, the
wall has already rearranged to put her most likely play in the top left, she
starts something within the attention window, and the act of watching feeds new
signal (she finished Stranger Things, so tomorrow's page leans harder into
supernatural horror and her BYW rows reseed). Every session personalizes the next
session. A brand new competitor app has none of this history, so its wall feels
generic to her where Netflix's feels like it read her mind. That accumulated
per member page is the switching cost.

Which metric does it move: retention, primarily, through churn. Gomez-Uribe and
Hunt attribute "several percentage points" of reduced monthly churn to
personalization and recommendations, and put the combined value at more than
1 billion dollars per year, mostly because a member who reliably finds something
in 90 seconds does not cancel, and a lower churn rate both raises the lifetime
value of every existing member and shrinks how many new members you must acquire
just to stand still. Roughly 80 percent of hours streamed on Netflix start from a
recommendation, not from search, so the homepage is not a nice-to-have surface, it
is the main way the product is consumed.

The real observed example is the failure it prevents. Netflix measured that a
member who does not find something in 60 to 90 seconds is at "substantially"
increased risk of abandoning the session, and abandoned sessions are the leading
edge of churn. The homepage exists to win that 90 seconds, session after session,
for 300 million people.

---

## 9. The lesson for Rare.lab

Rare.lab has a page-shaped surface hiding in plain sight: the effects and presets
gallery a creator lands on, and the node palette they scan when building a graph.
Treat it like the Netflix homepage, not like a flat search list.

Concrete moves:

1. Make it a page of rows, and rank the rows, not just the items. Do not show one
   long list of presets sorted by popularity. Show rows with reasons: "Because you
   used a bloom node," "Trending shaders this week," "Runs at 60fps on your GPU,"
   "Continue editing" (the graph they left open). The row header is the reason,
   and the reason is what makes a creator trust the suggestion. Build far more
   candidate rows than you show and pick per creator.

2. Do page-wise selection with a used-set and a diversity rule, exactly like page
   generation. If the top row already surfaced three bloom-based presets, the next
   row should not be more bloom. Keep a used-effects set as you lay rows down and
   drop a row whose marginal value given the rows above is low. Independent
   per-row scoring will duplicate and bore; the interaction between rows is where
   the quality is.

3. Encode your own 90 second cliff into the objective. A creator scanning the
   gallery has a short attention budget too. Weight a preset by probability-of-being
   -seen (decaying down the page) times probability-of-being-used, a 2D cascade
   metric like Netflix's adapted ERR, so your ranker stops wasting effort on row 30
   and fights for the first screen. Optimize the page reward (did they start
   building), not the average item score.

4. Fold your hard performance constraint into the page reward, do not bolt it on
   as a filter. Rare.lab's non-negotiable is that a shipped shader hits frame
   budget on the target device. So make "predicted to hit 60fps on this creator's
   GPU family" a first class term in the row/preset value, the way GenPage folds
   business constraints into its RL reward, rather than filtering after ranking.
   A preset that looks great and blows the frame budget should lose in the ranker
   itself. Pair this with the per-device capability bucket idea from the Swiggy and
   Instagram Explore lessons: precompute, per device bucket, which presets are
   known-good, so the "runs fast on your GPU" row is a keyed lookup, not a live
   compile.

5. Keep the spine: offline-think, online-lookup. Precompute per creator preset
   orderings and item-item similarity ("creators who used this node also used
   that") in batch, so opening the gallery is a keyed read plus light page
   assembly, never a live scan or a live compile. The gallery must paint instantly
   or the creator's own 90 second graze ends in a blank stare.

The one sentence: a gallery is a page, not a list, so rank the rows against each
other with a used-set and a see-probability metric, bake your frame-budget
constraint into the page reward, and precompute everything so the live open is a
lookup.

---

## Sources

- Carlos A. Gomez-Uribe and Neil Hunt, "The Netflix Recommender System:
  Algorithms, Business Value, and Innovation," ACM Transactions on Management
  Information Systems, 2015. PDF:
  https://dl.acm.org/doi/pdf/10.1145/2843948 and mirror
  https://ailab-ua.github.io/courses/resources/netflix_recommender_system_tmis_2015.pdf
  (PVR, Top-N, Trending Now, Continue Watching, BYW, video-video similarity, the
  90 second window, ~80% of hours from recommendations, "several percentage
  points" of churn, more than 1 billion dollars per year).
- Netflix Technology Blog, "Learning a Personalized Homepage," April 2015:
  http://techblog.netflix.com/2015/04/learning-personalized-homepage.html (rows as
  the page structure, page generation, stage-wise versus page-wise, 2D page-level
  metrics, adapted Expected Reciprocal Rank and cascade browsing, rule-based versus
  personalized layout).
- Netflix Technology Blog, "GenPage: Towards End-to-End Generative Homepage
  Construction at Netflix," June 2026:
  https://netflixtechblog.com/genpage-towards-end-to-end-generative-homepage-construction-at-netflix-77146fba8a08
  (autoregressive token-by-token page construction, greedy max-value token
  selection, reinforcement learning with a page-level reward: diversity, stopping
  power, business constraints).
- Netflix Technology Blog, "Innovating Faster on Personalization Algorithms at
  Netflix Using Interleaving":
  https://netflixtechblog.com/interleaving-in-online-experiments-at-netflix-a04ee392ec55
  (team-draft interleaving, over 100x fewer members than the most sensitive A/B
  metric, two-stage experimentation).
- Quartz, "Netflix knows exactly how long you'll look for something to watch
  before giving up": https://qz.com/622316 and NBC News,
  https://www.nbcnews.com/business/business-news/netflix-knows-how-long-you-ll-search-they-lose-you-n521766
  (the 60 to 90 second browsing window, ~10 to 20 titles reviewed, ~3 in detail).
