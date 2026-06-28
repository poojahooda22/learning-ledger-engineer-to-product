# Canva: Template search and ranking (the "search a template, start designing" box)

Date: 2026-06-28
Product: Canva
Feature: Template search and ranking (the search box on the templates page and inside the editor)

---

## 1. The user

Meet Priya. She runs a small bakery in Pune and handles her own Instagram.
It is 9pm, the shop is closed, and she wants to post about tomorrow's fresh
croissants. She is not a designer. She has no brand kit, no Photoshop, no idea
what a "1080x1080 canvas" is. She opens Canva on her laptop, and in the search
box she types one word: `croissant`. Or maybe `instagram post`. Or
`bakery sale`. She wants a ready made design she can change in two minutes and
post before she goes to bed.

She is one of hundreds of millions doing the same thing. A teacher making a
worksheet. A founder making a pitch deck. A student making a birthday card for
a friend. None of them want a blank page. They all want a head start.

## 2. The real problem

A blank canvas is terrifying for a normal person. If Canva opened to an empty
white rectangle, Priya would close the tab. The hard part of design is not
dragging boxes. It is the first move: what should this even look like?

So the real problem is not "let me edit a design." It is "show me something
good that is already 90 percent done, that matches what I am making, so I only
have to swap the words and the photo." The search box is the front door to
that. If she types `birthday` and gets ugly, generic, or wrong-sized results,
she leaves. If she gets ten gorgeous birthday cards she can pick from, she
stays for twenty minutes and makes something she is proud of.

The catch: Canva has roughly 600,000 templates and over 100 million stock
images, videos, and graphics behind that one box (Canva, The Social Shepherd
stats). Priya's one word has to reach into that pile and pull back the right
ten, in under a blink, while 20,000 other people per second do the exact same
thing (Better Stack, Canva Engineering).

## 3. The feature in one sentence

You type a few words, and Canva pulls the best-matching, correctly-sized,
high-quality templates out of a catalog of hundreds of thousands and ranks
them so the most useful ones land at the top.

## 4. Jobs to be done

What Priya is really hiring this search box to do:

- "Save me from the blank page. Give me a starting point, not a tool."
- "Read my intent from one or two words. I typed `bakery`, show me bakery
  things, not random food clips."
- "Give me the right shape. If I am making an Instagram post, do not show me
  A4 posters."
- "Only show me good ones. I cannot tell good design from bad, so do that
  filtering for me."
- "Be instant. I am tired and it is late. Do not make me wait."

Notice that most of these jobs are about trust and taste, not keyword
matching. The search has to act like a design-savvy friend who already knows
what looks good.

## 5. How it works for the user

Priya sees a single search bar. She types `instagram post bakery`. As she
types, suggestions drop down. She hits enter. A grid of templates appears,
maybe a few dozen, all already shaped as square Instagram posts, all looking
polished, the most relevant ones first. She can keep narrowing with filters on
the side (color, style, "free" vs "pro", format). She clicks one she likes. It
opens straight in the editor with her cursor ready. She swaps the headline,
drops in a photo of her croissants, and posts.

The whole "find something good" part took maybe eight seconds. That is the
product.

## 6. The actual flow, step by step

1. Priya lands on the templates page (or the editor's template panel).
2. She types `birthday card` into the search box.
3. With each keystroke, an autocomplete dropdown suggests completions
   (`birthday card`, `birthday card for kids`, `birthday card funny`). This is
   a separate typeahead system, close cousin of the Google autocomplete
   teardown from 2026-06-16.
4. She presses enter. The browser sends the query plus context (her locale,
   the format she is in, whether she is a Pro user) to Canva's search service.
5. The query is cleaned up and understood: lowercased, tokenized into
   `birthday` and `card`, language detected, maybe spell-corrected
   (`brithday` to `birthday`), maybe expanded with synonyms.
6. The service fetches a candidate set: a few hundred to a few thousand
   templates that contain or relate to those words, pulled from an inverted
   index.
7. Those candidates get filtered: drop the ones that are the wrong format,
   drop ones she has no license for, keep only live templates.
8. The survivors get re-ranked by predicted usefulness, not just text match.
9. The top results (say 50) are returned. The grid renders. Sorting already
   happened on the server. Her laptop just paints the pictures.
10. She scrolls; pagination fetches the next page on demand, not all at once.

Steps 5 through 9 are the search pipeline, and that is where the engineering
lives.

## 7. Under the hood, like the engineer

The cleanest way to understand Canva's search is the same split that ran
through the Amazon, YouTube, and Google teardowns: **matching and ranking are
two different halves.** Matching is cheap and wide (find a few thousand
plausible templates fast). Ranking is expensive and narrow (carefully order a
few hundred). You never run the expensive half over the whole catalog.

### The catalog and the core data structure: the inverted index

Canva built its search on top of a Lucene-based engine. For years that was
Apache Solr. Over 2021 onward they migrated to Elasticsearch 7.10 running on
AWS OpenSearch (Canva Engineering, "Migrating from Solr to Elasticsearch").
Both are Lucene under the hood, and the heart of Lucene is the **inverted
index**.

An inverted index flips the obvious layout. Instead of "template -> its words,"
it stores "word -> list of templates that contain it." So the engine keeps,
for the word `birthday`, a posting list:

```
birthday  -> [t_4412, t_9981, t_10342, t_55012, ... ]
card      -> [t_9981, t_22719, t_55012, ... ]
```

Each template's searchable text is its title, tags, category, and creator
metadata. A real template like "Pastel Cute Birthday Card" lands in the
posting lists for `pastel`, `cute`, `birthday`, and `card`.

When Priya searches `birthday card`, the engine does not scan 600,000
templates. It grabs the posting list for `birthday`, grabs the list for
`card`, and **intersects** them (or unions, depending on the match mode). The
cost tracks the length of those two lists, not the size of the whole catalog.
This is the single most important reason search is fast: the work is
proportional to how rare your words are, not to how big the library is. A
posting-list intersection over two terms touching a few thousand templates is
milliseconds. A full scan of 600,000 would not be.

Inside each posting list, Lucene also stores how often the term appears and how
important it is, which feeds the relevance score.

### Matching half, step by step

**Query understanding (annotation).** Canva's pipeline first turns the raw
string into a structured query. Lowercase, tokenize, detect language,
spell-correct, expand synonyms. Canva describes itemizing nearly 50 distinct
entry points into search (template results, editor ingredients, fonts, photos,
help, and more) and having grown at least four separate search systems over
time, which is exactly why they rebuilt around one shared pipeline with a
shared vocabulary (Canva Engineering, "Search Pipeline: Part I"). The query
understanding stage is the first pipeline component.

**Candidate generation.** The annotated query hits the inverted index and
pulls back candidates. This is the recall step: cast a wide net, accept some
junk, just do not miss the good ones. Canva explicitly supports
**overfetching** here, grabbing more candidates than they will finally show, so
the ranking step downstream has room to work (Canva Engineering, "Search
Pipeline: Part II"). If Priya needs 50 results, candidate generation might pull
2,000.

A purely lexical (word-matching) index has a famous blind spot: it cannot match
meaning. Someone searching `birthday` will miss a perfect template tagged only
`anniversary` or `celebration`. This is the same "sneakers equals running
shoes" gap Amazon's two-tower model solved. Canva's move to Elasticsearch was
partly to unlock **vector search**: Elasticsearch lets you map fields as dense
and sparse vectors and encode them as binary doc_values (Canva Engineering,
migration post). A dense vector is a list of numbers that captures meaning, so
`birthday` and `celebration` end up close together in vector space even though
they share no letters. That gives a hybrid retrieval: lexical BM25 for exact
words, vectors for meaning, results fused. (Inference, well grounded: Canva
states the migration unlocked dense and sparse vector fields and that they run
hybrid lexical-plus-vector retrieval is the standard Elasticsearch pattern;
the exact production blend for templates is not fully public.)

**Filtering.** Now constrain. Priya is making an Instagram post, so drop every
template whose format is not square social. Drop Pro-only templates if she is
on the free plan (or mark them locked). Keep only live, non-deleted templates.
In Elasticsearch these are filter clauses, which are cached and do not affect
the relevance score, so they are cheap to apply over and over.

### Ranking half: from a few thousand to the right order

Now the expensive thinking runs, but only on the survivors, a few hundred to a
few thousand. The base layer is Lucene's **BM25** text score. BM25 (Best
Matching 25) is the decades-old workhorse that scores a document higher when
the query terms appear often in it (term frequency) but discounts terms that
appear in tons of documents (inverse document frequency) and normalizes for
length. So a template literally titled "Birthday Card" outscores one that just
mentions birthday in a tag.

But BM25 alone gives bland results, because hundreds of templates match
`birthday card` about equally on text. Text cannot tell a beautiful template
from an ugly one. So Canva layers **re-ranking** on top: reorder the narrowed
set by predicted usefulness using behavioral and quality signals (Canva
Engineering, Part I and Part II). Plausible signals (inference, this is how the
class of problem is solved and matches what Canva describes as post-fetch
re-rankers): how often this template gets opened and actually used after
appearing for this kind of query, how recent and fresh it is, its quality
score, personalization to Priya's locale and past picks, and creator
performance. This is the same shape as YouTube ranking on expected watch time
or Amazon ranking on behavior: when the text of the candidates looks nearly
identical, you rank on what people actually do.

Canva built these as **componentized, post-fetch re-rankers** behind common
interfaces, so the same re-ranker built for one content type can be reused for
another (Part II). The pipeline can be deployed in different topologies: query
understanding can be a remote service call, candidate generators can be their
own microservices. This is an architecture choice, not just a model choice. It
means an engineer can improve the birthday-card ranker without understanding
the whole system.

### Where the sorting happens

Important and easy to get wrong: **the sort happens on the server, inside the
search query, not on Priya's laptop.** Her browser never sees 2,000
candidates. It receives the final ordered top 50 and paints them. If you tried
to ship 2,000 templates to the phone and sort there, you would burn her data,
her battery, and her patience. Sorting lives next to the index, in the database
query, every time.

### Interleaving: how they know a new ranker is better

Canva uses **interleaving** for ranking experiments (Part I and II). Instead of
the classic A/B test where group A sees ranker A and group B sees ranker B,
interleaving blends the two rankers' results into one list for the same user
and watches which side's results get clicked and used. It needs far less
traffic to reach significance, so they can test many ranking ideas quickly. The
pipeline was deliberately built to make interleaving easy: two pipelines, one
blended result list.

### Testing search without reading your private designs

A neat and genuinely Canva-specific detail. They cannot freely look at user
queries and designs for privacy reasons, and online interleaving experiments
are slow. So they built an offline evaluation tool that uses LLMs to generate
**synthetic** templates and queries, a static reproducible dataset, and they
run code changes against it locally. It outputs results on over 1,000 test
cases in under 10 minutes, letting one engineer run more than 300 offline
evaluations in the time a single online interleaving experiment would take
(Canva Engineering, "How to improve search without looking at queries or
results"). A clever trick in there: to test ranking, they take a known-relevant
design and create degraded copies with weaker text match and worse freshness,
then check the ranker still surfaces the good one. That is how you measure a
ranker without ever seeing a real customer.

### The scale story at three tiers

**1,000 templates.** Trivial. A single Lucene index on one box. You could
almost scan it. BM25 over a thousand documents is instant. No sharding, no
caching needed. This is Canva in its first months.

**100,000 templates.** Still one or a few Elasticsearch shards, fits in memory,
fast. The inverted index is now clearly paying off: you touch only the posting
lists for the typed words, not all 100,000 docs. You start caring about
relevance, because now many templates match `birthday` and order matters. The
ranking half starts to matter more than the matching half. Add read replicas so
search reads do not fight with index updates.

**10 million plus searchable items, 20,000 searches per second.** This is
Canva's real world: 600,000 templates plus 100M+ media elements, 1 million
searches per minute (Better Stack, Canva Engineering). Here is what breaks and
what survives it:

- A single index node cannot hold it or serve that QPS. So **shard** the index
  across nodes and **replicate** each shard. A query scatters to the shards,
  each returns its local top results, and a coordinator gathers and merges
  them (scatter-gather, the same pattern as Amazon search). Replicas multiply
  read throughput, which is what 20,000 QPS demands.
- Self-managed Solr needed a literal team of engineers just to keep it patched
  and alive. That operational weight is part of why they moved to managed
  Elasticsearch on AWS OpenSearch (migration post). At this scale, who operates
  the cluster is an engineering decision as real as which algorithm ranks.
- Popular queries (`instagram post`, `resume`, `birthday`) repeat constantly.
  **Cache** their result sets. A huge fraction of that 1M-per-minute is a small
  set of head queries, so a cache hit turns most searches into a memory lookup
  and never touches the index.
- The expensive thinking (training rankers, building vector embeddings,
  computing template quality and popularity) runs **offline**, on a schedule.
  The live query just reads precomputed scores and does one bounded sort. This
  is the same offline-think / online-lookup split as Discover Weekly, YouTube,
  and Razorpay. The live path stays cheap on purpose.
- **Pagination**, not full dumps. Return 50, fetch the next 50 only if she
  scrolls. Deep pagination is expensive in a scatter-gather index, so the UI
  leans on "load more" and good first-page ranking instead of letting anyone
  jump to page 500.

The dial that makes all of this work is the **candidate-set size**. Ranking
cost is tied to how many candidates you rank (a few thousand), not to how many
templates exist (600,000) or how big the media catalog is (100M+). Keep that
set bounded and your ranking cost is decoupled from catalog growth. Canva can
10x its template library and the live search cost barely moves.

## 8. The retention and habit mechanic

The loop here is subtle and powerful. Search is the **activation** moment: the
first search that returns something Priya loves is the moment she goes from
"curious" to "Canva is my design tool." If that first `birthday card` search
returns junk, she never comes back. If it returns ten beautiful options, she
makes something, feels capable, and that feeling is the hook.

Then it compounds into **retention and revenue**. Every good search result
that happens to be a Pro template ("you can use this, just upgrade") is a
gentle, well-targeted upsell placed exactly when she already wants the thing.
The freshness and personalization signals in ranking keep the grid feeling
alive: search `instagram post` in December and seasonal, holiday-flavored
templates rise, so the library feels current every time she returns, the same
"app feels alive" trick Swiggy and Zomato use with festival animations. And
because Canva pays creators based on billions of content usages per month, good
ranking is also what feeds the creator economy that keeps the library growing,
which keeps search results fresh, which keeps Priya coming back. Search quality
is the flywheel's hub.

The metric it moves: primarily **activation** (first-design success), then
**retention** (the library feels worth returning to) and **revenue** (Pro
templates surfaced at the moment of intent).

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime. Your library of starter
effects, materials, and node graphs is your version of Canva's templates, and
it will be the front door for every new user who does not yet know how to
build a shader from scratch.

The concrete lesson: **build your effect search as the same two-half pipeline,
and decouple ranking cost from library size with a bounded candidate set from
day one.** When a user types `water` or `glow` or `holographic foil`, do not
scan every shader graph. Index each effect's title, tags, the node types it
uses, and a short text description into an inverted index, and match on that
first. Then, because two `water` shaders look identical by text, re-rank on
what actually matters for your users: does it compile clean, what is its
measured GPU cost on a mid-range device, how often is it used, does it match
the target platform (WebGL vs native). That GPU-cost-aware ranking is your
unfair advantage. Canva ranks on "is it beautiful and used"; you can rank on
"is it beautiful, used, **and** does it run at 60fps on the user's hardware,"
which is a signal only you can compute.

Two more steals, both pure scalability:

- **Add a meaning layer early.** Canva's whole Solr-to-Elasticsearch migration
  was partly to unlock vector fields so `birthday` matches `celebration`. For
  shaders, embed each effect so `water` also surfaces `ripple`, `caustics`,
  and `ocean`. Plan the index to hold dense vectors before you need them, not
  after.
- **Steal the synthetic offline-eval trick.** Canva tests ranking changes
  against LLM-generated synthetic designs, 1,000 cases in 10 minutes, no
  private data, no slow online test. You can generate synthetic node graphs and
  queries and test "does my new ranker still surface the good shader" locally in
  minutes, so you iterate on search quality 300x faster than waiting on real
  usage. For a small team, that speed is the moat.

Keep the expensive thinking (compiling, GPU-cost profiling, embedding, usage
ranking) offline and precomputed, so the live search a user fires while editing
is one bounded index lookup and one small sort. That is how Canva serves a
million searches a minute, and it is how your editor stays instant as your
effect library grows from 100 to 100,000.

---

## Sources

- Canva Engineering Blog, "Search Pipeline: Part I" : https://www.canva.dev/blog/engineering/search-pipeline-part-i/
- Canva Engineering Blog, "Search Pipeline: Part II" : https://www.canva.dev/blog/engineering/search-pipeline-part-ii/
- Canva Engineering Blog, "Migrating from Solr to Elasticsearch, and their differences" : https://www.canva.dev/blog/engineering/migrating-from-solr-to-elasticsearch-and-their-differences/
- Canva Engineering Blog, "How to improve search without looking at queries or results" : https://www.canva.dev/blog/engineering/how-to-improve-search-without-looking-at-queries-or-results/
- Canva Engineering Blog (index) : https://www.canva.dev/blog/engineering/
- Better Stack newsletter, "How Canva Scaled Their Search to Handle 1M+ Searches Per Minute" : https://newsletter.betterstack.com/p/how-canva-scaled-their-search-to
- The Social Shepherd, "19 Essential Canva Statistics" (catalog and design counts) : https://thesocialshepherd.com/blog/canva-statistics
- Canva Design School, "Using search and Magic Design" : https://www.canva.com/design-school/resources/using-search-and-magic-design
- Elasticsearch / Lucene background on inverted index and BM25 : https://www.elastic.co/search-labs/blog/hybrid-search-elasticsearch
</content>
</invoke>
