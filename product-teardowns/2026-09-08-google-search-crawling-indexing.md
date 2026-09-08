# Google Search: web crawling and index building

Date: 2026-09-08
Product: Google Search
Feature: Web crawling and the index build (Googlebot's crawl frontier, plus the Caffeine / Percolator incremental indexing pipeline that turns crawled pages into the inverted index a query reads)

A note on scope. Earlier teardowns in this ledger took apart the parts of Google
that happen when you press Enter: autocomplete (2026-06-16), spell correction
(2026-07-05), PageRank (2026-07-18), the Knowledge Panel (2026-08-05), Featured
Snippets (2026-08-20). Every one of those reads an index that already exists.
This one is about the machine that builds that index in the first place, and
keeps it fresh. It is the plumbing under all the rest. When Amazon search
(2026-06-23) or Canva template search (2026-06-28) "merge posting lists from an
inverted index," this is the story of who wrote those posting lists and how they
stay current.

---

## 1. The user

Two people, at opposite ends of the same pipe.

The first is Priya, a reader. It is 10:04 on a Wednesday. The RBI just cut the
repo rate. She opens Google and types "rbi repo rate today". She expects the top
result to be a story published minutes ago, not last month's rate.

The second is Arjun, who runs the newsroom that broke that story. He hit Publish
at 10:00. His whole job depends on Google knowing his page exists, reading it
correctly, and showing it to Priya while the news is still news. If Google finds
his page next Tuesday, he has lost the traffic and the ad money that came with it.

Neither of them thinks about crawling or indexing. Priya just wants a fresh
answer. Arjun just wants to be found fast. The feature sits invisibly between
them, and its whole reason to exist is to make those four minutes possible.

---

## 2. The real problem

The web is not a database Google owns. It is a few hundred billion pages sitting
on other people's servers, changing every second, with no one sending Google a
notification when something changes. Google's search results are only as good as
its copy of that web, and that copy is always going stale.

Described like a friend would: imagine you are trying to keep a phone book for a
country where people change their numbers constantly and never tell you. If you
reprint the whole book once a month, most numbers are wrong by week two. If you
try to reprint it the instant any one number changes, you spend your entire life
reprinting and never deliver a book. The real problem is freshness against scale.
You cannot rebuild everything every time one thing changes, and you cannot ignore
the change either.

There is a second, quieter problem underneath. When Googlebot fetches Arjun's
page, it is a guest on his server. If Google sends ten thousand requests a second
to one small newsroom, it knocks the site offline. So the crawler has to be
polite, which means slow, which fights directly against freshness. Fast enough to
catch the news, gentle enough not to break the site it is reading. That tension
is the whole engineering problem.

---

## 3. The feature in one sentence

Googlebot continuously discovers and downloads web pages at a rate each server
can survive, and an incremental indexing pipeline (Caffeine, built on the
Percolator system) threads each freshly crawled page into a giant inverted index
within minutes, so a query typed seconds later can find it.

---

## 4. Jobs to be done

- For the reader: "When I search for something that happened five minutes ago,
  show me the page that was written five minutes ago, not last month's version."
- For the publisher: "Find my new page quickly, read it the way a browser would,
  and put it in the running so it can rank."
- For Google itself: "Keep a fresh mirror of a few hundred billion pages without
  reprocessing the whole web every time one page changes, and without taking down
  the servers I am borrowing from."

Notice the reader is hiring the crawler for freshness and the publisher is hiring
it for discovery, and both are hiring the indexer for the same thing underneath:
turn a raw HTML page into something a query can match in a few milliseconds.

---

## 5. How it works for the user

For Priya it is invisible. She types, she gets a fresh result, she never sees the
crawl. The only visible trace is the little grey line some results used to show,
the "cached" copy and the date, which was literally Google's stored snapshot of
the page from the last time Googlebot visited.

For Arjun it is visible, because Google gives publishers a dashboard over the
crawl: Google Search Console. He can see when Googlebot last fetched a URL, submit
a sitemap (a plain list of his URLs so Google does not have to discover them by
following links), ask for a recrawl of a single page, and read the "Crawl Stats"
report showing how many requests Googlebot made per day and how fast his server
answered. He can also write a `robots.txt` file that tells Googlebot which paths
to stay out of. That file is the polite handshake: the crawler reads it before
touching the site.

---

## 6. The actual flow, step by step

Follow Arjun's repo-rate story end to end.

1. 10:00:00. Arjun publishes `/rbi-cuts-repo-rate`. His CMS automatically adds the
   URL to his sitemap and pings Google that the sitemap changed. It also appears
   as a link on his homepage, which Googlebot already crawls often.
2. 10:00:30. Google's scheduler notices the sitemap change and the new link on a
   frequently crawled page. The URL goes into the crawl frontier, the giant
   to-fetch queue, with a high priority because this host publishes fresh news
   often (crawl demand is high for it).
3. 10:01:10. Before fetching, Googlebot checks its cached copy of the newsroom's
   `robots.txt`. The path is allowed. It also checks the host's recent health:
   fast responses lately, so it is safe to fetch now.
4. 10:01:12. Googlebot sends one HTTP GET for the page. The server answers `200 OK`
   with the HTML in about 200 milliseconds.
5. 10:01:13. The raw HTML goes into Google's store. A parser pulls out the visible
   text, the title, and every link on the page. Any new links (a related-story
   link, say) are fed back into the frontier. This is the loop: pages discover
   more pages.
6. 10:01:40. Because the page uses some JavaScript, it is queued for the Web
   Rendering Service (WRS), a headless Chromium that runs the page like a real
   browser so Google sees the same text a human would. WRS fetches the page's
   extra resources (its CSS and JS), reusing cached copies where it can.
7. 10:02:30. The rendered text flows into the indexer. The page is tokenized into
   words. For each word, the indexer records "this word appears in this document,
   at these positions." That is the inverted index write.
8. 10:03:00. Because the pipeline is incremental, this single document is stitched
   into the live index without rebuilding anything else. Its signals (freshness,
   the newsroom's authority, the words it contains) are now queryable.
9. 10:04:00. Priya searches "rbi repo rate today." The matching half of search
   walks the inverted index, finds Arjun's document in the posting lists for
   "rbi," "repo," and "rate," the ranking half scores it high on freshness and
   authority, and it lands near the top. Four minutes, publish to reader.

Pre-2010, step 8 was the killer. The index rebuilt in slow batches, so Arjun's
page might not be queryable for hours or days. The whole point of Caffeine was to
make step 8 take minutes.

---

## 7. Under the hood, like the engineer

There are two distinct halves here, and it helps to keep them apart the way we
kept matching and ranking apart in the search teardowns.

- The **crawl**: discover URLs and download pages, politely, at web scale.
- The **index build**: turn those raw pages into an inverted index and keep it
  fresh incrementally.

The crawl feeds the index. The index feeds the query. Let us go through each.

### Half one: the crawl and its frontier

The core data structure of any crawler is the **frontier**: the set of URLs known
but not yet fetched. Naively it is a queue. You pop a URL, fetch it, extract its
links, push the new ones, repeat. That is breadth-first traversal of the web
graph, where pages are nodes and links are edges. The web is a graph, so the
crawler is a graph traversal. The classic public description of a real
web-scale crawler is the Mercator paper (Heydon and Najork, 1999); Google's
current internals are not fully public, so treat the specific structures below as
the standard way this class of problem is solved, grounded in Mercator and in
Google's own 2024 "Crawling December" posts, and labeled as inference where it is
inference.

Three things turn that simple queue into a real crawler:

**1. The seen-URL test (deduplication).** The web is full of the same URL linked
from a million places. Before you push a URL into the frontier you must ask "have
I seen this already?" With a few hundred billion URLs, you cannot keep a plain
hash set of full URL strings in memory; the strings alone would be tens of
terabytes. The standard fix is a **Bloom filter**, a bit-array probabilistic set
that answers "definitely new" or "probably seen" using a handful of hash
functions and a few bits per URL, so the membership test costs a constant number
of bit lookups instead of storing the string. It can give a rare false "seen"
(you skip a genuinely new URL), which is an acceptable trade for shrinking the
seen-set from terabytes to gigabytes. Concretely: the homepage link to
`/rbi-cuts-repo-rate` and the sitemap entry for the same URL both hash to the same
bits, so the second one is recognized as a duplicate and not fetched twice.

**2. Politeness (per-host scheduling).** You must never hammer one server. So the
frontier is not one queue, it is thousands of **per-host queues**, and a scheduler
that enforces a minimum gap between two fetches to the same host. Google calls the
governing budget the **crawl rate limit**, and it is set by "host load," which is
Google's own phrase for how much it can fetch "politely, without hurting the
server." Google's documented rule: Googlebot watches the server's response time,
and if Time To First Byte climbs or the server starts returning `429 Too Many
Requests`, Googlebot "immediately reduces its parallel connections." If the site
stays fast, the limit rises. So the crawl rate is a feedback loop measured off the
site's own health. A real example: a big fast site like Wikipedia can absorb many
parallel Googlebot connections; Arjun's small newsroom on shared hosting gets a
gentle trickle, because the moment its TTFB rises Googlebot backs off.

**3. Priority (crawl demand).** You cannot crawl everything equally often. Google
splits the budget into crawl rate limit (what the host can take) and **crawl
demand** (how much Google wants the page). Demand is driven by how important a URL
looks and how often it changes. A news homepage that posts twenty stories a day is
recrawled constantly; a static "About us" page from 2015 is visited rarely. So the
frontier is really a **priority queue** keyed by expected value of a recrawl, per
host, rate-limited by host health. That is why Arjun's fresh URL jumped the line:
high demand host, page allowed, host healthy.

One more real cost that surprises people. Modern pages need JavaScript to show
their text, so Google renders them in a headless Chromium (the WRS). Rendering one
page means fetching all its sub-resources (CSS, JS files), and Google's December
2024 post is blunt that this "chips away from the crawl budget of the hostname."
Their survival trick is caching: WRS "caches everything for up to 30 days," and
that cache deliberately ignores the site's own HTTP cache headers, precisely so
one page's fifty script fetches do not eat the crawl budget of the whole site on
every visit. Caching to protect a scarce resource: the same instinct as every
other teardown here.

### Half two: building the inverted index

Now the page is fetched and rendered. What gets built?

The heart is the **inverted index**. Start with the obvious "forward" direction:
document to words. Doc 8801 (`/rbi-cuts-repo-rate`) contains the words {rbi, repo,
rate, cut, monetary, policy, ...}. A forward index is great for "what is on this
page" and useless for search, because a query gives you words and wants documents.
So you invert it. For each word you store a **posting list**: the sorted list of
document IDs that contain that word, usually with positions.

```
"repo"  -> [ 8801, 9004, 9210, ... ]     (docs containing "repo")
"rate"  -> [ 12, 8801, 8842, 9004, ... ] (docs containing "rate")
```

A query for "repo rate" then becomes: fetch the two posting lists, walk them
together, keep the doc IDs that appear in both (a merge intersection), and you
have your candidate set. Doc 8801 is in both, so it survives. This is exactly the
"matching" half from the Amazon and Canva teardowns; this teardown is where those
lists are written. The structure that maps a word to its posting list is a
**lexicon**, essentially a hash map or sorted term dictionary from word to a
pointer into the postings.

This is not new. The original public blueprint is Brin and Page's 1998 paper,
"The Anatomy of a Large-Scale Hypertextual Web Search Engine," which describes
Google's first index in this exact shape: a repository of raw pages, a lexicon of
words, "barrels" holding the posting lists, a forward index, and a **sorter** that
turns the forward index into the inverted index by sorting billions of (word,
document) pairs by word. That sort is the expensive heart of index building. Get
comfortable with that word "sort," it is where the whole scale story lives.

The critical point: **the sort and the index build happen on Google's servers,
offline, ahead of the query.** By the time Priya searches, the posting lists are
already written and sorted. Her query does no crawling and no sorting of the
corpus. It reads a prebuilt structure. This is the offline-think, online-lookup
pattern that shows up in nearly every teardown in this ledger (Discover Weekly,
YouTube, Amazon, Razorpay). Crawling and indexing are the offline think for all of
search.

### The scale story, three tiers

**Tier 1: about 1,000 pages (one small site, a crawler on a laptop).**
One queue, a plain hash set of seen URLs, fetch pages one at a time, build the
inverted index as an in-memory hash map from word to a list of doc IDs. The whole
index fits in RAM. Nothing breaks. You would be over-engineering to add anything.
Example: crawling one newsroom of a thousand articles is a weekend script.

**Tier 2: about 100,000 to a few million pages (a large site, or a vertical
crawler).** Two things break. First, you are now fetching from many hosts at once,
so a single global queue would either hammer some servers or starve others; you
need the per-host politeness queues and a scheduler. Second, the inverted index no
longer fits in memory. You spill it to disk as sorted segments (Brin and Page's
"barrels") and merge them, and the merge sort of billions of (word, doc) pairs
becomes the dominant cost. The seen-URL set is getting big, so the Bloom filter
starts earning its place. This tier is a classic external-sort and
merge-many-segments problem.

**Tier 3: the real web, hundreds of billions of pages.** Two hard walls.

The first wall is the seen-set and the frontier at global scale. You cannot keep
either on one machine. You **shard** by URL host, so all URLs for one site live on
one crawler shard, which conveniently makes politeness a local decision (one shard
owns one host's rate limit). Bloom filters keep the per-shard seen-set small.

The second wall is freshness, and this is the interesting one, because it is the
wall Google actually hit and publicly climbed. Google's original index was built
in **batches** with MapReduce: crawl a big chunk of the web, then run a giant
multi-stage sort-and-merge job to rebuild the index, then ship it. The problem is
that batch is all-or-nothing. As Google described it when launching **Caffeine**
in June 2010, the old index was built in layers, some refreshed fast and some
slow, and "to refresh a layer of the old index, we would analyze the entire web,
which meant there was a significant delay between when we found a page and made it
available to you." One new page could not be added cheaply; you waited for the
next rebuild of its layer. That is why Arjun's page could sit invisible for hours.

The fix was to stop rebuilding and start **updating incrementally**. Caffeine, per
Google's own numbers, "takes up nearly 100 million gigabytes of storage in one
database, and we add new information at a rate of hundreds of thousands of
gigabytes per day," and it delivered "50 percent fresher" results than the old
index. Under Caffeine sits **Percolator**, described in Peng and Dabek's 2010
paper "Large-scale Incremental Processing Using Distributed Transactions and
Notifications" (OSDI 2010). The trick of Percolator:

- It stores the whole index and crawl state in **Bigtable** (Google's giant
  distributed sorted key-value table).
- It adds **cross-row transactions** with snapshot isolation, which raw Bigtable
  did not have, so an index update that touches many rows stays consistent.
- It adds **observers**, which are like database triggers. When a crawled document
  changes a cell, Percolator sets a "notify" marker on that cell, and observers
  wake up and process just the rows affected by that change, then possibly notify
  further observers, rippling the update outward only as far as it needs to go.

The result Google reports: "the median document moves through [the pipeline] over
100 times faster" than the old MapReduce system, cutting the average age of a
document in results by about 50 percent. That is the whole game. Instead of
recomputing the web to add one page, you touch only the rows that page affects.
Batch became incremental, and freshness went from days to minutes.

The trade Percolator accepted is worth naming, because it is a real engineering
choice. Doing per-document transactions costs far more machine work per document
than a batch job does (the paper is candid that Percolator uses roughly 30 times
more CPU per document than the equivalent MapReduce at peak). Google spent CPU to
buy latency. For a batch report you would never make that trade. For a search
index where freshness is the product, spending more compute to make each document
land 100 times sooner is exactly right. The lesson is that "incremental" is not
free; it is latency bought with throughput, and you only buy it where latency is
the thing users feel.

Serving is the last shard story. The index is **partitioned** (sharded) across
thousands of machines, so no one machine holds all the posting lists, and each
shard is **replicated** for read throughput and failure tolerance. A query is
scattered to the shards, each returns its best local matches, and the results are
gathered and merged. Scatter-gather over sharded, replicated posting lists is the
same pattern the Amazon and Canva search teardowns described from the query side.
Crawling fills the shards; querying reads them.

---

## 8. The retention and habit mechanic

This feature has no button and no streak, so its habit loop is subtle and it is
the strongest kind: it moves **retention** by protecting trust.

The loop is "search returns fresh, correct results, so I search again." Every time
Priya searches for something that happened minutes ago and gets a minutes-old
answer, Google earns one more unit of the belief that Google is where live
questions get answered. That belief is the entire moat. The day results feel
stale ("I searched for the score and it showed yesterday's game"), the habit
cracks and she tries something else. Freshness is not a feature users praise; it
is a promise they only notice when it breaks, exactly like Spotify's loudness
normalization (2026-08-28) or its shuffle spread (2026-09-05). Invisible papercut
removal at planet scale.

There is a real, observed number behind this. Google's own 2010 launch claim that
Caffeine made results "50 percent fresher" was a retention and quality move, not a
vanity metric: fresher results mean more searches land on a satisfying answer,
which means more searches. And there is a two-sided loop with publishers. Arjun
keeps publishing to Google's web because Google finds and rewards fresh pages fast;
the more publishers race to be fresh, the fresher the web Google can show readers,
which brings more readers, which makes publishers chase Google harder. The crawler
being fast is what keeps that flywheel spinning. Google Search Console, the
publisher dashboard over the crawl, exists precisely to keep publishers engaged
with that loop.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles graphs to
shippable code, plus an embeddable runtime. The whole product is a pipeline from a
source graph to a compiled artifact. That is structurally the same shape as
crawl-to-index: a source that keeps changing, and an expensive build that turns it
into something fast to run. So the Caffeine lesson lands directly.

**The lesson: make your compile incremental, not batch, and buy that with a
dependency graph plus dirty-marking, the same move Percolator made.**

Concretely. When a Rare.lab user drags one node and changes one parameter, say the
frequency on a single noise node in a graph of two hundred nodes, do not recompile
the whole graph. That is the MapReduce-era mistake: any one change forces a full
rebuild, so edit latency scales with graph size, and a big graph starts to feel
laggy exactly when the artist is iterating hardest.

Instead, do what Caffeine did:

1. **Model the graph as a dependency DAG and keep it explicit.** Each node knows
   its inputs. That is your Bigtable of state.
2. **Dirty-mark on edit.** Changing the noise node marks it and everything
   downstream of it as dirty, and nothing else. This is Percolator's "notify"
   marker: the change ripples outward only as far as it truly reaches. A node on a
   parallel branch that does not depend on the noise is never touched.
3. **Recompile only the dirty subgraph, then splice it in.** Rebuild the affected
   nodes' code and stitch it into the already-compiled artifact, the way Caffeine
   threads one document into a live index without rebuilding the rest.
4. **Cache aggressively across edits.** Every clean node's compiled output and its
   intermediate buffers are still valid; keep them, the way WRS caches resources
   for 30 days to protect a scarce budget. Here the scarce budget is the artist's
   patience and the frame time.

And accept Percolator's honest trade. Incremental compilation costs more
bookkeeping per edit than a clean batch compile: you maintain the DAG, the dirty
sets, the cached buffers. You are spending memory and complexity to buy edit
latency. Make that trade only where latency is the product, which for an editor is
the live-edit path, and keep a simple clean batch compile for the final shippable
export where throughput matters and latency does not. Fresh-feeling edits are your
freshness metric. The artist who sees the shader update the instant they move a
slider keeps iterating, the same way Priya keeps searching when the news is
minutes old. Incremental build is the feature that makes iteration feel alive.

---

## Sources

- Sergey Brin and Lawrence Page, "The Anatomy of a Large-Scale Hypertextual Web
  Search Engine" (1998). The original public description of Google's crawler,
  repository, lexicon, barrels, forward index, sorter, and inverted index.
  http://infolab.stanford.edu/~backrub/google.html
- Google Search Central Blog, "Our new search index: Caffeine" (June 2010). The
  incremental-vs-layered explanation, "100 million gigabytes," "hundreds of
  thousands of gigabytes per day," and "50 percent fresher."
  https://developers.google.com/search/blog/2010/06/our-new-search-index-caffeine
- Daniel Peng and Frank Dabek, "Large-scale Incremental Processing Using
  Distributed Transactions and Notifications" (Percolator), OSDI 2010. Bigtable,
  cross-row transactions, observers/notify columns, the over-100x median-latency
  result, and the CPU trade.
  https://research.google/pubs/pub36726/
- Google Search Central Blog, "Crawling December" series (December 2024),
  including the resources/rendering post. Crawl budget as crawl rate limit plus
  crawl demand, host load and politeness, `429`/TTFB backoff, and the WRS 30-day
  resource cache.
  https://developers.google.com/search/blog/2024/12/crawling-december-resources
- Google, "What Crawl Budget Means for Googlebot" (January 2017). Crawl rate limit,
  crawl demand, and host load defined by Google.
  https://developers.google.com/search/blog/2017/01/what-crawl-budget-means-for-googlebot
- Allan Heydon and Marc Najork, "Mercator: A Scalable, Extensible Web Crawler"
  (1999). The classic public design for a web-scale crawl frontier, per-host
  politeness, and URL deduplication.
- The Register, "Google Caffeine jolts worldwide search machine" (June 2010) and
  "Google Percolator, global search jolt sans MapReduce comedown" (September 2010).
  Contemporary reporting on the Caffeine and Percolator numbers.
