# Google Search: PageRank and the web link graph (how the shape of the web ranks a page)

Date: 2026-07-18
Product: Google Search
Feature: PageRank and the link graph (the query-independent quality score that ranks web results)

A note on scope. The ledger already has two Google Search teardowns: autocomplete
(the typeahead trie) and spell correction (the noisy channel). Both live on the
front of the query. This one goes to the core that made Google beat AltaVista in
1998: not "which pages contain the words" but "which of those pages is worth
trusting." That answer came from a graph algorithm, not a text algorithm, and the
graph is the web itself. Nobody in this ledger has torn down a graph-centrality
computation before, so this is a genuinely different flavor from the match-then-rank
recsys reports.

---

## 1. The user

Meet Rohan, a second-year student in Pune. It is 11pm and he types one word into
Google: `jaguar`. He is not sure yet whether he means the animal (biology
assignment due tomorrow), the car brand (his uncle just bought a used one), or the
Jacksonville Jaguars (a friend keeps posting about them). He hits enter and expects
the ten blue links to already be sorted so that the best, most trustworthy page for
whatever "jaguar" usually means sits at the top. He will look at the first three
results, maybe scroll once, and if nothing good shows up he will retype. He gives
the page about four seconds of patience.

Rohan does not know that the web has roughly a hundred billion pages, that millions
of them contain the word "jaguar," and that some of them are spam farms built purely
to rank. He just wants the top result to be right.

---

## 2. The real problem

Here is the pain, told plainly. In 1997 the web was already tens of millions of
pages. The search engines of the day (AltaVista, Lycos, Excite) matched your words
against the page text and ranked mostly by how many times your word appeared. That
sounds reasonable until you realize the consequence: a page that says "jaguar jaguar
jaguar jaguar" three hundred times looks more relevant than the actual Wikipedia
article on jaguars. Ranking by on-page text is trivially gameable, and it was gamed.
Search in 1997 was full of junk floating at the top.

The real problem is that relevance and quality are two different things, and text
alone can only tell you about relevance. A page about jaguars and a spam page
stuffed with the word "jaguar" are equally relevant to the query by the text test.
You need a second signal, one that is hard to fake, that tells you which page the
rest of the web actually respects. In 1997 there was no such signal in production.

The insight that fixed it: the web is not just a pile of documents, it is a giant
directed graph. Every hyperlink from page A to page B is a small vote by A that B is
worth visiting. A page that thousands of respected pages link to is probably good.
And crucially, votes are weighted: a link from the New York Times home page should
count for far more than a link from Rohan's abandoned blog. That recursive idea
(your importance depends on the importance of who links to you) is PageRank.

---

## 3. The feature in one sentence

PageRank assigns every page on the web a single query-independent importance score,
computed once from the structure of the link graph, so that when Rohan searches
"jaguar" the pages that the rest of the web trusts most float to the top.

---

## 4. Jobs to be done

What is Rohan really hiring PageRank to do?

- "Sort the millions of matching pages so the trustworthy one is first, and do it
  before I even see the page."
- "Protect me from the guy who stuffed 'jaguar' three hundred times into a junk
  page to trick the ranker."
- "Give me the page the web itself vouches for, not just the page that mentions my
  word the most."
- "Do it in under a second, every time, for a query you may never have seen."

The deeper job: give Rohan a reason to trust the first result enough that he stops
searching and clicks. Trust is the product.

---

## 5. How it works for the user

Rohan types `jaguar`, hits enter, and in about 400 milliseconds gets ten results.
The Wikipedia article on the jaguar (the animal) and the official Jaguar cars site
sit near the top. A blog nobody links to that happens to repeat "jaguar" a lot sits
on page 4, or nowhere. Rohan never sees a score. He never sees the graph. He just
sees a good ordering and a little snippet under each link. That is the whole visible
experience: the ordering feels obvious and correct, which is exactly the point. Good
ranking is invisible; you only notice ranking when it is bad.

The thing he cannot see is that the "importance" half of that ordering was computed
days before he ever typed the word, over the entire web, with no idea that Rohan or
the word "jaguar" existed.

---

## 6. The actual flow, step by step

From Rohan's tap to the ranked page:

1. Rohan types `jaguar` and hits enter. Request leaves his phone in Pune, hits the
   nearest Google front end.
2. The query is normalized (lowercased, tokenized) and sent to the index serving
   layer, which is sharded across thousands of machines.
3. Matching (the cheap wide half): for the term "jaguar," Google walks the inverted
   index. The posting list for "jaguar" is the set of document IDs that contain the
   word, perhaps 50 million of them. This is a set lookup, not a scan of the web.
4. Filtering: language, region, safe-search, and other filters trim the candidate
   set.
5. Ranking (the narrow expensive half): each surviving candidate gets a score that
   blends many signals. One of the oldest and most famous ingredients is PageRank,
   the precomputed per-page importance number, combined with the text relevance
   score (how well "jaguar" matches this specific page), freshness, and today
   hundreds of other signals.
6. The top ~10 are sorted server-side (never on Rohan's phone) and returned with
   snippets.
7. Rohan sees the Wikipedia jaguar article at the top and clicks.

The part this teardown cares about is step 5's PageRank number, and where it came
from: a huge offline computation that happened long before step 1. That is the
offline-think, online-lookup spine again. The web-scale graph math is done in
advance; the live query just reads a precomputed score off each candidate.

---

## 7. Under the hood, like the engineer

This is the heart. Three questions: what is the data structure, what is the
algorithm, and how does it survive at scale.

### The data structure: the web as a sparse directed graph

Model the web as a directed graph G. Every page is a node. Every hyperlink is a
directed edge. If Wikipedia's jaguar article links to the IUCN Red List page, there
is an edge from `en.wikipedia.org/wiki/Jaguar` to `iucnredlist.org/species/jaguar`.

How many nodes and edges? By the 1998 paper, Google's early link database was 322
million links (Brin and Page, 1998). Today the indexed web is well over 100 billion
pages, and each page has on average tens of outgoing links, so the edge count runs
into the trillions.

You do not store this as an adjacency matrix. A matrix of 100 billion by 100 billion
would have 10^22 cells, almost all of them zero, which is impossible and pointless.
The web is extremely sparse: a typical page links to maybe 10 to 100 others, not to
a billion. So you store it as adjacency lists: for each page, a list of the pages it
links to (its out-links). Concretely, node `wikipedia.org/Jaguar` stores an array
`[iucnredlist.org/..., britannica.com/..., nationalgeographic.com/...]`. To compute
PageRank you also want the reverse: for each page, who links TO it (its in-links),
because a page's rank flows in from its in-linkers. So the link graph is stored twice,
forward and reverse, as compressed adjacency lists. Google's early "BigFiles" and the
URL-to-docID mapping in the 1998 paper are exactly this machinery.

Why a graph and not a tree or a hash map? Because the web has cycles (A links to B
links to C links back to A), many roots, and no hierarchy. A tree cannot express a
cycle. A hash map alone cannot express "who points at whom." You need a general
directed graph, and the only affordable representation of a sparse one is adjacency
lists.

### The algorithm: PageRank as a random surfer's resting place

Here is the idea in one picture. Imagine a bored web surfer, "the random surfer."
He starts on some random page. He clicks a random out-link. From that page he clicks
another random out-link. He keeps doing this forever. PageRank of a page is the
long-run fraction of time the surfer spends on that page. Popular, well-linked pages
get visited often; orphan pages almost never.

The formula from the 1998 paper:

```
PR(A) = (1 - d) + d * ( PR(T1)/C(T1) + ... + PR(Tn)/C(Tn) )
```

where T1..Tn are the pages that link to A, C(Ti) is the number of out-links on page
Ti, and d is the damping factor, normally 0.85 (Brin and Page, 1998).

Read it in plain words. A page's rank is the sum of the ranks of its in-linkers, but
each in-linker only passes along its rank divided by how many links it has. So if
Wikipedia's jaguar page has a high rank but links to 200 places, each of those 200
gets only 1/200th of Wikipedia's rank. A link from a focused, high-rank page that
links to only a few things is worth more than a link from a high-rank page that links
to thousands. This is the "vote weighting" that makes PageRank hard to fake: to boost
a spam page you would need genuinely high-rank pages to link to it, and you do not
control those.

The damping factor d = 0.85 is the surfer's patience. With probability 0.85 he
clicks a link; with probability 0.15 he gets bored and teleports to a random page.
That teleport term (the `(1 - d)` part) is not a hack, it is load-bearing. It solves
two real failures of the pure random walk:

- Dead ends (pages with no out-links, called dangling nodes, like a PDF that links
  nowhere). Without teleport the surfer gets stuck there and rank leaks out of the
  system.
- Spider traps (a small clique of pages that only link to each other, a classic spam
  ring). Without teleport all the rank pools into the trap. The 15% random jump lets
  the surfer escape any trap, which is precisely why link farms cannot hoard rank.

### How it is actually computed: power iteration

You cannot solve that formula in closed form for 100 billion pages. You solve it by
iteration, called power iteration:

1. Give every page an equal starting rank, 1/N where N is the number of pages.
2. In each round, every page pushes its current rank, split evenly, out along its
   out-links to its neighbors.
3. Every page sums up what it received, applies the damping and teleport terms, and
   that is its new rank.
4. Repeat until the ranks stop changing (they converge).

Mathematically this is repeatedly multiplying a rank vector by the (sparse) link
transition matrix. The beautiful fact: it converges fast. On that 1998 database of
322 million links, PageRank "converged to a reasonable tolerance in 52 iterations"
(Brin and Page, 1998). Fifty-two passes over the graph, not thousands. The reason it
converges so fast is the damping factor: the error shrinks by roughly a factor of
d = 0.85 each iteration, so after ~50 iterations 0.85^50 is about 0.0003, tiny. The
teleport that fixed the spam traps also guarantees a unique answer exists and that
iteration finds it quickly. One constant, three jobs.

Concrete walk. Suppose after convergence, Wikipedia's jaguar article has PageRank
0.0009 and Rohan's abandoned blog has 0.00000001. Both contain the word "jaguar."
When Rohan searches, both are in the "jaguar" posting list, but at ranking time the
Wikipedia page carries a five-order-of-magnitude larger importance score, so it wins
the top slot even if the blog repeated "jaguar" more times. Text got them both into
the candidate set; the graph decided the order.

### Matching and ranking are still two different halves

Even though this report is about the ranking signal, keep the split clear, because
it is the reason the whole thing is affordable:

- Matching (inverted index, posting lists): "which pages contain jaguar?" This is a
  set lookup whose cost tracks the query, not the size of the web. The posting list
  for "jaguar" is fetched, not computed.
- Ranking (PageRank plus text score plus hundreds of signals): "of those, which are
  best?" This runs over a bounded candidate set (thousands to a few million after
  early pruning), not all 100 billion pages.

PageRank is precomputed and query-independent, so at query time it is a single
number read off each candidate, an O(1) lookup per candidate. None of the expensive
graph math happens while Rohan waits. This is the same offline-think/online-lookup
pattern as Discover Weekly, YouTube recs, and Amazon item-to-item in this ledger:
the whole-corpus computation lives offline; the live path is a keyed read plus a
small sort.

### The scale story at three tiers

Tier 1, about 1,000 pages. Trivial. Hold the whole graph in RAM as adjacency lists,
represent ranks as a 1,000-element array, run power iteration in a plain loop. A
single laptop does this in milliseconds. Nothing breaks. Every intro-to-algorithms
PageRank demo lives here.

Tier 2, about 100,000 to a few million pages. Still fits on one big machine's memory,
but now you care about the sparse representation (never the dense matrix) and about
cache-friendly iteration order. The 1998 system was here-ish: 322 million links, and
they noted it "would not be too difficult" to keep the whole thing efficient with
careful I/O because the graph, though large, was still sortable and streamable on the
hardware of the day. The break at the next tier is memory: soon the rank vector plus
the link graph no longer fit in one machine.

Tier 3, 100 million to 100 billion-plus pages. This is where the naive approach dies
and real engineering begins. Three problems and their fixes:

- The graph does not fit on one machine. Fix: partition (shard) the graph by page,
  spread it across thousands of machines. Each machine owns a slice of pages and
  their out-links.
- Each iteration must push rank across machine boundaries (Wikipedia lives on machine
  17, but it links to a page on machine 4,102). Fix: this is a distributed
  message-passing computation. Google first expressed PageRank as a chain of MapReduce
  jobs (Dean and Ghemawat, 2004): the Map step emits, for each page, its rank divided
  among its out-links as messages keyed by the target page; the Reduce step sums all
  incoming messages per target and applies damping. One MapReduce job is one power
  iteration; you run ~50 of them in sequence, each reading the previous round's
  output. MapReduce made "iterate over the whole web ~50 times" a routine batch job.
- MapReduce is wasteful for graphs, because it re-reads and re-shuffles the entire
  graph structure from disk every single iteration even though the structure never
  changes; only the rank numbers change. Fix: Google built Pregel (Malewicz et al.,
  SIGMOD 2010), a vertex-centric graph system. You "think like a vertex": each page
  is a vertex that, in each superstep, receives messages (incoming rank), updates its
  own rank, and sends new messages along its edges. The graph structure stays
  resident in memory across supersteps, so only the small messages move, not the
  whole graph. Pregel checkpoints for fault tolerance (a machine dies mid-computation,
  you restart from the last checkpoint, not from scratch). Pregel is the system that
  powered PageRank at Google scale and inspired Apache Giraph, which Facebook used on
  its social graph.

Two more scale realities worth naming. First, dangling nodes (pages with zero
out-links, like a lone image or PDF) are common at web scale; their rank has nowhere
to go, so implementations collect the "leaked" dangling rank each iteration and
redistribute it uniformly (equivalent to treating a dead end as linking to everyone).
Second, the hot-node problem: a few pages (Google's own home page, Facebook, YouTube)
have hundreds of millions of in-links, so the machine holding one of them receives a
flood of messages every superstep. This is the celebrity/hot-key problem from Lesson
16 in this ledger, and the fix is the same shape: combine messages locally before
sending (Pregel "combiners" sum partial rank contributions on the sender side so a
target receives one merged message per machine, not millions of tiny ones).

### One honest caveat about the present

Everything above is real and published, but be precise about the timeline. The
recursive random-surfer PageRank is exactly as the 1998 paper describes and it was
the signal that launched Google. Google today does not rank on 1998 PageRank alone;
ranking is hundreds of signals (relevance, freshness, page quality, the learned
systems Google has confirmed such as RankBrain and BERT, plus spam defenses). Google
has publicly said classic PageRank is still one signal among many and that they moved
off the exact original algorithm years ago; the precise modern link-based scoring is
not public. So: the algorithm and its scaling story here are fact and primary-sourced;
"this exact formula is what ranks your query in 2026" is not a claim I am making.
What is durable and worth stealing is the idea, and it is very much alive: a
query-independent, graph-derived quality score, computed offline, read in O(1) at
query time.

---

## 8. The retention and habit mechanic

PageRank's retention mechanic is not a notification or a streak. It is trust
compounding into a default.

The loop: Rohan searches "jaguar," the top result is right, he clicks and gets what
he wanted. That success trains a tiny expectation: "Google's first result is usually
correct." Repeat that a few hundred times across a year and the expectation hardens
into a reflex. Rohan stops evaluating whether to use Google. He just Googles. The
verb is the retention metric. When a product name becomes the verb for the whole
category, retention is effectively solved: the user no longer makes a choice.

Which metric does it move? Retention, and through retention, revenue. Every reliable
answer is a reason to come back, and coming back is what Google monetizes through
ads on the same results page. But the causal root is quality: AltaVista and Excite
had ads too; Google won because the organic results were better, and they were better
because the link graph beat text stuffing. Trust was the growth engine, and better
ranking was how trust was earned. The real observed example is the language itself:
"to google" entered the Oxford English Dictionary as a verb in 2006. No marketing
campaign puts your brand in the dictionary; only a decade of the top result being
right does that.

There is a defensive side too, like the WhatsApp-encryption and Stripe-Radar reports
in this ledger. Better ranking created a data flywheel: more users means more click
and query-reformulation data, which trains better ranking, which brings more users.
The link graph was the seed that started that flywheel spinning.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader editor that compiles to shippable code plus an
embeddable runtime. The node graph a creator builds is, literally, a directed graph:
nodes are operations, edges are data flow, and it can have shared subgraphs and
diamond dependencies just like the web has shared links. PageRank's lesson maps
almost one to one.

The concrete, actionable move: compute a query-independent importance score per node
at compile time, once, offline, and read it in O(1) at runtime. Right now a runtime
scheduler or an optimizer that has to decide "which nodes matter most" (which to keep
at full precision, which to cache, which to evaluate first, which to down-res under a
frame-budget squeeze) would be tempted to analyze the graph live, every frame. That
is the AltaVista mistake: doing whole-graph work on the hot path. Instead, run a
PageRank-style propagation over the shader DAG at compile time. Weight each node by
how many downstream nodes depend on it and how important those are (importance flows
backward from the final output nodes, exactly like rank flows in from in-linkers), so
a color-ramp node feeding the final blend of a hero effect scores high, while a
debug-visualization branch that feeds nothing scores near zero. Bake that per-node
importance into the compiled artifact. At runtime, when the frame budget is tight,
the scheduler makes its "what to protect, what to degrade" decision as a single array
lookup per node, not a graph walk. Offline-think, online-lookup, on your own graph.

Two riders, both straight from PageRank:

- Add a damping/teleport analog so importance cannot pool into a self-referential
  cluster. A tight feedback loop of nodes (a physics or feedback-buffer subgraph that
  reads its own output) is Rare.lab's spider trap: naive importance propagation would
  let that loop hoard the whole budget. Bleed a fixed fraction of importance back to
  the output nodes each iteration so no internal cycle can starve the pixels the user
  actually sees.
- Use combiners for hot nodes. A single "time" or "UV" input node that feeds hundreds
  of downstream nodes is your celebrity node. When you propagate importance, sum the
  contributions locally before applying them rather than touching the hot node once
  per dependent. Same message-combining trick Pregel uses to keep google.com from
  melting a machine, at a scale a thousand times smaller but the same shape.

The one-line version: the structure of your graph already encodes what matters. Spend
the compute to extract that once, offline, into a per-node number, so the runtime
never has to think about the whole graph while a frame is due.

---

## Sources

- Sergey Brin and Lawrence Page, "The Anatomy of a Large-Scale Hypertextual Web
  Search Engine," 7th International World Wide Web Conference (WWW7), 1998. The
  primary source for PageRank, the random-surfer model, d = 0.85, and the "52
  iterations on 322 million links" convergence result.
  http://infolab.stanford.edu/pub/papers/google.pdf
- Lawrence Page, Sergey Brin, Rajeev Motwani, Terry Winograd, "The PageRank Citation
  Ranking: Bringing Order to the Web," Stanford InfoLab Technical Report, 1999. The
  fuller treatment of the algorithm, dangling nodes, and convergence.
  http://ilpubs.stanford.edu:8090/422/
- Jeffrey Dean and Sanjay Ghemawat, "MapReduce: Simplified Data Processing on Large
  Clusters," OSDI 2004. Google's batch framework; PageRank expressed as iterated
  Map/Reduce jobs is a canonical example.
  https://research.google.com/archive/mapreduce-osdi04.pdf
- Grzegorz Malewicz et al., "Pregel: A System for Large-Scale Graph Processing,"
  SIGMOD 2010. The vertex-centric ("think like a vertex") superstep model, combiners,
  and checkpointing that powered PageRank at web scale.
  https://15799.courses.cs.cmu.edu/fall2013/static/papers/p135-malewicz.pdf
- Amy Langville and Carl Meyer, "Google's PageRank and Beyond: The Science of Search
  Engine Rankings," Princeton University Press, 2006. The standard reference on the
  linear-algebra view (power iteration, teleport, convergence rate ~ d).
