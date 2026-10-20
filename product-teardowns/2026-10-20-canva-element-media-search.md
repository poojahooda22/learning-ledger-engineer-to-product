# Canva element and media search: typing "balloon" and getting the right balloon out of 100 million

Date: 2026-10-20
Product: Canva
Feature: The search bar inside the editor that finds photos, graphic elements,
videos, audio and templates, with its query understanding, keyword-plus-vector
candidate fetch, and re-ranking.

A note on sourcing before we start. Canva has published more about this than
most companies publish about their search. Its engineering blog has a two part
"Search Pipeline" series on rebuilding the whole search stack, and a separate
post on reverse image search for replacing an image in a design. Where I quote
those, I mark it CONFIRMED (Canva). Canva has not published every internal
number (exact index sizes, the exact embedding model, the exact latency
budget), so where I describe how this class of problem is actually built I lean
on the well-documented standard (BM25 and inverted indexes, dual-encoder
embeddings, approximate nearest neighbour libraries like Faiss, reciprocal rank
fusion, cross-encoder re-rankers) and mark it INFERENCE. Scale figures for
Canva the company (users, designs, library size) come from Canva Create 2025,
press coverage and Canva's own marketing pages; I mark those SECONDARY and treat
them as claims, not gospel.

This teardown is about the search INSIDE the editor (the left-hand panel where
you type and pictures appear), not about Canva's public website SEO. Different
problem entirely.

---

## 1. The user

Meet Ananya. It is 11:40pm and her niece's first birthday party is tomorrow at
4pm. She promised her sister she would make the invitation, print ten copies,
and put them in the party bags. She opens Canva on her laptop, picks a blank
square post, and now she needs art. She clicks the "Elements" panel on the left,
clicks into the search box, and types "balloon".

She is not browsing a gallery for fun. She has a picture in her head (a cluster
of pastel balloons, the cute cartoon kind, not a sad single party balloon) and
she wants to see it, click it, and have it land on her canvas. Then she will
type "confetti", then "birthday cake", then maybe a photo for the background.
Every search she runs tonight is a small transaction: word in, the right picture
out, drag onto the page, move on. If the first screen of balloons is wrong, or
slow, or all locked behind a Pro badge she does not want to pay for at midnight,
she feels the friction immediately.

## 2. The real problem

Here is the pain, said like a friend would say it. Ananya knows what she wants
but she does not know what it is CALLED or what some stock photographer TAGGED
it. She types "balloon". The library has an asset literally titled "Colorful
helium party balloons bunch vector", another tagged only "celebration decor",
and a photo whose filename is "IMG_2024_final_v3.jpg" with no useful words at
all. A dumb text match on the word "balloon" would miss two of those three good
results and would happily return a "water balloon fight" photo that is useless
for a baby's party.

So the real problem has two halves that fight each other.

Half one is MATCHING. Out of a library that Canva describes as more than 100
million photos, videos, graphics and audio tracks (SECONDARY, Canva Pro
marketing pages), find the few hundred that are even plausibly about balloons.
This has to survive messy input: typos ("ballon"), vague intent ("something
cute for a party"), and assets with thin or wrong text.

Half two is RANKING. Of those few hundred, decide which 30 to show on the first
screen, in order, because Ananya will almost never scroll past the first two
rows. The best balloon has to be in the top-left corner.

Matching and ranking are genuinely different jobs. Matching is "is this even
about balloons, yes or no". Ranking is "given a pile of balloon results, which
is the best one for THIS person making THIS design right now". The old Canva
made this worse in a way that is very relatable to any engineer: CONFIRMED
(Canva Search Pipeline Part I) the company had grown at least four SEPARATE
search systems over the years, one for media, one for templates, and others,
each with its own codebase. So the balloon you found in photo search and the
balloon you found in template search were ranked by two different piles of code
that did not agree and could not share improvements. Fixing relevance in one did
nothing for the others.

## 3. The feature in one sentence

You type a few words (or drop in a picture) and Canva fetches the handful of
relevant assets out of a library of 100 million-plus, then ranks them for you
specifically, and does it fast enough that the pictures feel like they were just
sitting there waiting.

## 4. Jobs to be done

What is Ananya really hiring this search box to do.

- "Turn the word in my head into the picture in my head." She thinks "balloons",
  she wants balloons on screen, without learning anyone's tagging system.
- "Forgive me." Typo, plural, Hindi-English mix, half a word. Still work.
- "Read my mind a little." When she types "cute party", give her playful pastel
  things, not a corporate clip-art handshake.
- "Do not waste my first screen." Put the best, most usable, most on-brand-for-a-
  baby-party result where my eye lands first.
- "Do not make me leave." If I cannot find it here in a few seconds, I will open
  a new tab and go to a free stock site, and some of those trips I never come
  back from.
- "Let me point instead of type." Sometimes I have the picture already (a red
  sneaker photo) and I just want "more like this one".

That last one is a real shipped feature: CONFIRMED (Canva) reverse image search,
used so you can swap an image in a design for a visually similar one.

## 5. How it works for the user

From Ananya's side it feels like nothing at all, which is the point.

She clicks "Elements", types "balloon", and within a blink a grid of balloons
fills the panel. Cartoon balloon bunches, single balloons, balloon line icons,
balloon photos. Some have a small crown or "Pro" marker in the corner. She
scrolls a little, sees a pastel cluster she likes, drags it onto the canvas, and
it drops in at full resolution. She never sees a "Search" button; results update
as she types and as she finishes the word. She types "confetti" next and the
grid flips to confetti just as fast.

If she had instead dragged her own photo of balloons into the design, right-
clicked, and chosen to find a similar image, Canva would show her stock images
that LOOK like her photo (same kind of scene, same colours) even if nobody ever
typed the word "balloon" on them. That is the reverse image search path.

The whole visible experience is: word or picture in, ranked grid out, drag,
done. All the hard work is hidden.

## 6. The actual flow, step by step

Tap by tap, and then what each tap sets off.

1. Ananya clicks the Elements panel and the search box focuses. The app may
   already show "recent" or popular categories (a cheap cached list, no search
   engine touched yet).
2. She types "ballon" (missing an "o"). The client waits a beat (debounce, a few
   hundred milliseconds, so it does not fire a search on every keystroke) and
   sends the query string to Canva's servers. The phone or browser does NOT
   search anything itself. It is a dumb window onto a server result.
3. The server runs QUERY UNDERSTANDING. CONFIRMED (Canva Search Pipeline Part
   II): the query is validated, then rewritten if needed. "ballon" becomes
   "balloon" (spell correction). A query like "bday" might expand to include
   "birthday". This is where messy human input is cleaned into something the
   indexes can use.
4. The server runs CANDIDATE GENERATION. CONFIRMED (Canva): multiple candidate
   generators can run at once under the pipeline's coordination, each with a time
   or result-count deadline, and their results plus metadata are merged into an
   in-memory structure. In plain terms: several different "finders" race to pull
   plausible balloons, and whoever answers in time gets merged.
5. The server RE-RANKS the merged pile. CONFIRMED (Canva): re-ranking happens
   through dense vector spaces and also applies business rules (like result
   diversity and free-versus-Pro content) and personalization or machine-learned
   models. This is where the best balloon is chosen for the top-left slot.
6. The server returns a page of results (say the top 30 to 50) as a list of
   asset IDs plus thumbnails. Ananya's browser just lays them out in the grid.
   Sorting already happened on the server. The client never sorts 100 million of
   anything.
7. She scrolls. The client asks for the next page (pagination), it does not
   re-run the whole search or download everything up front.
8. She drags one onto the canvas. Now a different system (the design document)
   takes over. Search's job ended when it handed her the ID.

The reverse image path swaps step 2 and 3: instead of a word, she gives a
picture, the server turns that picture into an embedding (a list of numbers that
captures what it looks like), and candidate generation becomes "find the nearest
pictures in that number space".

## 7. Under the hood, like the engineer

This is the heart of it. I will build the mental model in layers, mark CONFIRMED
versus INFERENCE as I go, and walk the real "balloon" query the whole way.

### 7.1 Two halves, said precisely: matching then ranking

Every big search is two machines bolted together.

MATCHING (also called retrieval or candidate generation) answers "which items
are even in the running". It must look at the whole 100 million-item library but
is allowed to be cheap and a little sloppy, because its only job is to not miss
the good stuff and to hand the next stage a short list (a few hundred to a few
thousand items).

RANKING (here, re-ranking) answers "of this short list, what order". It is
allowed to be expensive and smart, because it only ever looks at a few hundred
survivors, never the whole library.

CONFIRMED (Canva Search Pipeline Part II) that Canva's pipeline is exactly this
shape: Query Understanding, then Candidate Generation (plural generators), then
Re-ranking. The genius of the split is that you spend cheap compute on the many
and expensive compute on the few.

### 7.2 Candidate generation, finder number one: the inverted index (BM25)

The first finder is a classic keyword search. CONFIRMED (Canva) that the design
"allows us to use BM25 search" for retrieval.

The data structure under BM25 is an INVERTED INDEX. Think of the index at the
back of a textbook. Instead of "page 3 contains these words", it stores "the
word balloon appears in assets 17, 240, 988, 10452, ...". That list of asset IDs
for a word is called a POSTING LIST.

Why this structure and not, say, scanning every asset's text. Because scanning
100 million text blobs for the word "balloon" on every keystroke is impossible
in the few hundred milliseconds Ananya will tolerate. With an inverted index you
jump straight to the "balloon" posting list and you are instantly holding the
IDs of only the assets that mention balloon. If she typed two words, "party
balloon", you fetch both posting lists and INTERSECT them (walk two sorted lists
of IDs and keep the ones in both), which is fast because the lists are sorted.

BM25 is the scoring formula on top of that. For the balloon query it gives each
matching asset a relevance number from three ideas:
- Term frequency: an asset whose text says "balloon" three times is more about
  balloons than one that says it once (with diminishing returns, so ten
  mentions is not ten times better).
- Inverse document frequency: a rare word counts more than a common one. If
  Ananya types "helium balloon", the word "helium" is rarer across the library
  than "balloon", so matching "helium" is stronger evidence this is the right
  asset.
- Length normalization: a short, tightly-tagged asset that says "balloon"
  outranks a giant wall of text that happens to include "balloon" once.

The real win for Ananya's "ballon" typo and vague words is that query
understanding (step 3) fixes the input BEFORE it hits this index, so the posting
list she actually looks up is "balloon", not the empty "ballon" list.

What BM25 cannot do: it cannot match MEANING. If an asset is tagged only
"celebration decor" and never says "balloon", BM25 will never find it for the
word "balloon". That is the job of the second finder.

### 7.3 Candidate generation, finder number two: embeddings and vector search

The second finder searches by MEANING, not words. This is what makes "cute
party" return playful pastel things.

The idea is an EMBEDDING: a model reads an asset (its pixels, or its text, or
both) and boils it down to a list of numbers, say 512 of them, a point in a
512-dimensional space. Assets that mean similar things land near each other in
that space. The pastel balloon bunch and the "celebration decor" asset sit close
together even though they share no words, because they LOOK and MEAN similar
things.

For the balloon query, Canva turns Ananya's cleaned query "balloon" into a point
in the same space (a text encoder), then asks "which asset points are nearest to
this query point". Nearness is usually cosine similarity, which for normalized
vectors is just a dot product: multiply the two lists of numbers together and
add them up. A big number means close in meaning.

INFERENCE (standard practice, strongly grounded): this is a two-tower or dual-
encoder setup. One tower turns an asset into a vector, the other turns a query
into a vector, trained so that matching pairs score high. The entire reason you
use TWO separate towers is CACHEABILITY. Every asset's vector is computed ONCE,
when the asset is uploaded to the library, and stored. It never depends on the
query. So at search time you only have to compute ONE fresh vector (the query)
and then look up neighbours. You never run the heavy image model on 100 million
assets per search. This is the "think offline, look up online" pattern that
shows up in every hard-scale feature.

For reverse image search this tower setup is CONFIRMED in spirit by Canva's own
post: they rejected text and metadata search because metadata "does not capture
nuanced visual features" like how many subjects are in the image or its colours,
and instead generated image EMBEDDINGS and compared them in a vector index.
CONFIRMED (Canva): they prototyped with 50,000 library images loaded into an
in-memory Faiss vector store, compared several embedding models on a 200-image
sample, and found that a roundabout approach (asking GPT-4o to describe the
image in words, then embedding the words with a CLIP text index) performed the
WORST. The lesson they learned is the lesson of this whole section: to search
pictures by how they look, embed the PICTURE, do not launder it through words
first, because words throw away exactly the visual nuance you were trying to
match.

### 7.4 What Faiss and ANN are actually doing (and why a flat scan dies)

Finding the nearest vectors sounds easy: compute the dot product of the query
vector against every asset vector and keep the top few hundred. At 50,000 images
that is fine, which is why Canva's PROTOTYPE used a flat in-memory Faiss store
(CONFIRMED). A flat index literally compares against everything. For 50,000
vectors that is 50,000 dot products, a couple of milliseconds, nobody notices.

Now make it 100 million. A flat scan is 100 million dot products of 512 numbers
each, on every single search, for every one of hundreds of millions of users.
That does not fit in the latency budget and the hardware bill is absurd. This is
the tier where a flat scan dies.

The fix is APPROXIMATE nearest neighbour (ANN). You give up the promise of the
exact nearest neighbours and accept "almost always the right near ones", in
exchange for sub-linear search. Two common shapes, both in Faiss:
- IVF (inverted file): cluster the 100 million vectors into, say, 100,000
  buckets by rough region of the space. At search time, figure out which few
  buckets the query falls near and only scan those. You compare against maybe a
  few hundred thousand vectors instead of 100 million.
- HNSW (a navigable small-world graph): build a graph where each vector links to
  its near neighbours, with a few long-range "express" links layered on top. To
  search, you walk the graph greedily toward the query, hopping closer each
  step, touching only a few hundred nodes to find good neighbours out of
  millions. It is the same instinct as a skip list: a few express lanes make the
  walk logarithmic instead of linear.

INFERENCE (grounded): production Canva media search at 100 million-plus items is
an ANN index of this family, not a flat scan. The flat Faiss store they describe
was explicitly a prototype over 50,000 images.

### 7.5 Merging the two finders

Now the balloon query has two candidate lists: a BM25 list (word matches) and a
vector list (meaning matches). CONFIRMED (Canva) that results from multiple
candidate generators are merged into one in-memory structure, each generator
bounded by a deadline so a slow one cannot hold up the whole search.

INFERENCE (standard): the merge is usually something like reciprocal rank fusion,
a dead-simple rule that says an item's combined score is the sum of 1 divided by
its rank in each list. An asset that is number 2 in BM25 and number 5 in the
vector list beats one that is number 1 in only a single list. The beauty of this
rule is it does not need the two finders to produce comparable scores (BM25
numbers and cosine numbers are on totally different scales), it only needs each
list's ORDER. The pastel balloon bunch that scored well on both words and
meaning floats to the top of the merged pile; the "water balloon fight" photo
that only matched the word drops.

### 7.6 Re-ranking: the expensive, smart stage on the few

Now there is a merged short list, maybe 200 to 500 balloon candidates. This is
where the expensive brain runs, because it only sees a few hundred items.

CONFIRMED (Canva Search Pipeline Part II) that re-ranking goes through dense
vector spaces AND applies business rules and personalization or ML models. The
business rules they name include result diversity and free-versus-Pro content.
Read into that for the balloon query:
- Relevance and personalization: a learned model scores each candidate for "how
  good is this for this user and this query". INFERENCE: at this stage you can
  afford a heavier model than retrieval, even a cross-encoder that looks at the
  query and the asset together rather than two precomputed vectors, precisely
  because there are only a few hundred to score, not 100 million.
- Diversity: do not fill the first row with eight near-identical red balloon
  photos from the same shoot. Spread the kinds (a vector bunch, a line icon, a
  photo) so Ananya's eye has real choices. This is a sequence decision, not a
  per-item one, the same "do not stack six of the same thing" logic good feeds
  use.
- Free versus Pro: Canva makes money when free users upgrade to reach Pro
  content, so the business rule can weave Pro results into the grid rather than
  burying them. This is a revenue lever living inside the ranker, done carefully
  so it does not wreck relevance (show the perfect Pro balloon, not a bad one).

Everything here is SERVER-SIDE. The ordered list of 30 IDs is the final answer;
the browser just paints it.

### 7.7 The scale story at three tiers

Walk the same balloon search as the library grows, naming what breaks and the
fix. (Library sizes are illustrative tiers; Canva's real library is the 100
million-plus top tier, SECONDARY.)

Tier one, 1,000 assets (a weekend stock site). One database table. Search is a
text LIKE or a tiny BM25 over a small inverted index, sort in the query, return.
No embeddings, no ANN, no sharding. A vector index here would be pure over-
engineering. A flat scan for "balloon" across 1,000 rows is instant. Everything
is fine; do the simple thing.

Tier two, 100,000 assets (a serious niche library). The naive text LIKE now gets
slow and, worse, dumb (it still misses "celebration decor"). The inverted index
with BM25 earns its keep here: it is the difference between scanning everything
and jumping to a posting list. Embeddings start to pay off because synonyms and
vague queries ("cute party") now cost real conversions when they return nothing.
A FLAT vector scan over 100,000 is still totally fine (this is the size of
Canva's own 50,000-image prototype, doubled), so ANN is still optional. What
broke going from tier one to here: raw scanning and the absence of semantic
matching. The fix: inverted index plus the first embeddings, still exact.

Tier three, 10 million and then 100 million-plus (real Canva). Now several walls
hit at once.
- Wall 1, the vector scan. A flat vector comparison against 100 million per
  search is dead (section 7.4). Fix: ANN (IVF or HNSW) in Faiss-family indexes,
  and SHARD the index across many machines so each holds a slice, search all
  slices in parallel, merge the top results. Sharding also lets the index grow
  past one machine's memory.
- Wall 2, the keyword index. One inverted index for 100 million assets is too
  big for one box and too slow to update. Fix: shard the inverted index too,
  scatter the query to all shards, gather and merge.
- Wall 3, computing embeddings. You cannot embed 100 million assets at search
  time. Fix: embed every asset ONCE at upload (offline), store the vector, and
  at query time compute only the single query vector and look up. Canva's own
  reverse-image work is the small version of this: embeddings generated and
  stored, queried later. This offline-think, online-lookup split is the whole
  ballgame.
- Wall 4, repeated popular queries. Millions of people search "instagram post",
  "resume", "birthday", "background". Fix: cache the ranked results for hot
  queries so the engine is not re-derived from scratch each time; a cache hit
  for "birthday" skips retrieval and ranking entirely.
- Wall 5, four codebases. CONFIRMED (Canva) the real historical wall was not
  only data size but the four separate search systems. The fix was the rebuild:
  one pipeline of ordered, isolated steps (Query Understanding, Candidate
  Generation, Re-ranking) with a query representation decoupled from any single
  engine, so a relevance win is made ONCE and every content type gets it.
  CONFIRMED that they migrated audio, font, video and media search onto the new
  pipeline, with templates next. That is an architecture-scaling fix, not a
  data-scaling fix, and it is the one most teams forget counts.

Through-line: the catalog being huge is only half the story. The deeper move is
to push every expensive computation (embedding each asset, training the ranker)
OFFLINE and precomputed, so the live balloon query is reduced to cheap online
steps: clean the text, look up two sharded indexes, merge a few hundred
survivors, score them with a model that only ever sees a few hundred, and cache
the popular ones.

## 8. The retention and habit mechanic

Search does not look like a retention feature. It has no streak, no badge, no
notification. But it is one of the strongest, because it is INVISIBLE CRAFT.

The loop is: want an asset, type a word, get the right thing in two seconds,
drop it in, the design looks good, finish it, feel capable, come back next time
you need a design. Every fast, accurate search is a tiny hit of "this tool
works, I am good at this", and that feeling is what pulls Ananya back for the
next birthday, the next Instagram post, the next flyer. A rebuild of search
feeds the north-star metric that matters most: designs completed. Canva reports
that users created one billion designs in March 2025 alone, and about 34 billion
cumulatively since 2013 (SECONDARY). Every one of those designs is a pile of
successful searches. Search is upstream of the activation metric (did a new user
finish a first design) and the retention metric (how many designs per user per
month).

The failure mode proves the point. If "balloon" returns junk, or takes four
seconds, or shows only locked Pro results at midnight, Ananya opens a new tab
and goes to a free stock site. Some of those trips she does not return from. A
slow or lying search is a trust-cracker exactly like a grocery app showing an
out-of-stock item as available: one bad experience and the rival tab stays open.
So the honest, fast, relevant grid IS the retention mechanic. There is no
separate loop to bolt on.

It also quietly moves REVENUE, not just retention. CONFIRMED (Canva) that free-
versus-Pro is a re-ranking business rule. Weaving the perfect Pro balloon into
Ananya's results at the moment she most wants a balloon is the single best
upgrade nudge Canva has, far better than a banner ad, because it is contextual
and she is already in flow. That is the rare feature that touches activation,
retention and revenue on the same ranked list.

The concrete example of "feel alive" freshness is Canva's seasonal behaviour:
search "diwali" in October or "christmas" in December and the library and
rankings surface timely, freshly added templates and elements, so the tool feels
current the way Swiggy and Zomato rotate festival home screens. A library that
feels stale is a library you stop opening.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime. Its own "library" is the set of
nodes, effects, presets and shader snippets a creator searches to build a scene.
Here is the specific, actionable lesson, biased to scale and performance.

Build node and asset discovery as ONE pipeline, two finders, expensive stage on
the few, and make "will it run in budget" a first-class part of the ranking, not
an afterthought.

1. One pipeline, not four. Canva's real pain was four search codebases that
   could not share a relevance fix. Rare.lab will be tempted to grow a separate
   search for nodes, another for presets, another for the asset marketplace.
   Resist it. Build one Query Understanding plus Candidate Generation plus Re-
   ranking pipeline with a query representation decoupled from any engine, so one
   improvement lifts node search, preset search and marketplace search at once.

2. Two finders. Keep a keyword inverted index with BM25 for the creator who
   knows the exact name ("fbm noise", "chromatic aberration"), AND a dense
   embedding index for the creator who only knows the FEELING ("make it look
   underwater", "glitchy VHS"). Embed every node, preset and shader example ONCE
   at publish time into an ANN index (Faiss-family, HNSW or IVF), never on the
   hot path, and compute only the single query vector live. Merge the two lists
   with reciprocal rank fusion so you do not have to make BM25 and cosine scores
   comparable.

3. Make performance a MATCHING or ranking signal, not a label. This is the
   Rare.lab-specific twist. An effect that will not compile or will not hit the
   frame budget on the creator's current target should either disappear from the
   candidate set (like an out-of-stock item) or be heavily penalized in re-
   ranking, the same slot where Canva applies its free-versus-Pro and diversity
   rules. Showing a gorgeous shader that drops to 12fps on the target device is
   the visual-effects version of a search result that is actually out of stock:
   it cracks trust the first time the creator drops it in and the preview stutters.

4. Offline-think, online-lookup, because your cost blows up per TARGET not per
   query. Precompute each asset's embedding, its compiled variants, and its
   measured cost on each device class OFFLINE. Then a live search is cheap: clean
   the query, hit two sharded indexes, re-rank a few hundred survivors with a
   model that can afford to be heavy because it only sees a few hundred, and
   cache the popular queries ("fire", "water", "portal"). The expensive compile
   and profiling already happened at publish.

5. Ship reverse search too. Canva lets a creator point at a picture and get
   visually similar stock. Rare.lab should let a creator point at a look (a
   reference frame, or an existing node graph) and get compatible building blocks
   back, via the same embedding index. And learn Canva's hard-won lesson: to
   match a look, embed the LOOK, do not describe it in words first and embed the
   words, because the words throw away exactly the visual nuance you wanted.

One line: scope the candidate set with a cheap keyword-plus-vector fetch over an
index built offline, make "runs in budget on this target" a first-class ranking
signal, and spend your expensive model only on the few survivors, so the live
search stays a fast lookup that never lies about performance and never stalls a
frame.

## Sources

Primary (Canva Engineering Blog):
- Search Pipeline: Part I (why Canva rebuilt search; at least four separate
  search systems; query representation decoupled from the engine):
  https://www.canva.dev/blog/engineering/search-pipeline-part-i/
- Search Pipeline: Part II (the new pipeline; Query Understanding, Candidate
  Generation, Re-ranking; BM25 retrieval with dense-vector re-ranking; business
  rules including diversity and free-versus-Pro; migrated audio, font, video,
  media, templates next):
  https://www.canva.dev/blog/engineering/search-pipeline-part-ii/
- Image replacement in Canva designs using reverse image search (why metadata
  misses visual nuance; image embeddings; 50,000 images in an in-memory Faiss
  store; 200-image model comparison; GPT-4o-to-CLIP approach performed worst):
  https://www.canva.dev/blog/engineering/image-replacement-in-canva-designs-using-reverse-image-search/

Background on the class of problem (inverted indexes, BM25, embeddings, ANN):
- Faiss (Facebook AI Similarity Search), the vector index library Canva used:
  https://github.com/facebookresearch/faiss
- HNSW approximate nearest neighbour (Malkov and Yashunin):
  https://arxiv.org/abs/1603.09320
- CLIP (Radford et al.), the multimodal text-and-image embedding family:
  https://arxiv.org/abs/2103.00020
- Reciprocal rank fusion (Cormack, Clarke, Buettcher), for merging result lists:
  https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf

Scale figures (secondary, treat as company-reported claims):
- Canva Create 2025 coverage (around 230 million monthly active users; one
  billion designs in March 2025; about 34 billion cumulative designs).
- Canva Pro premium-content marketing page (more than 100 million stock photos,
  videos, graphics and audio): https://www.canva.com/pro/premium-content/
