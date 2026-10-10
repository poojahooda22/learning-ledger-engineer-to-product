# Google Search: core ranking (how the ten blue links get ordered, Navboost and the deep models behind the list)

Date: 2026-10-10
Product: Google Search
Feature: Core ranking, the ordering of the organic web results (NOT crawling/indexing 09-08,
NOT PageRank the link-graph signal 07-18, NOT autocomplete 06-16, NOT spell correction 07-05,
NOT featured snippets 08-20, NOT the knowledge panel 08-05, NOT Lens 08-31). This is the
step after the index is built and after the query is understood: given a matching set that
can be hundreds of thousands of pages deep, which ten go on page one, and in what order.

A note on sourcing before we start. For years almost nothing about live ranking was public.
Two events changed that and most of the concrete facts in this teardown come from them: the
US v. Google antitrust trial (2023 to 2024), where Google's then head of Search ranking Pandu
Nayak testified and internal slides became court exhibits, and the May 2024 leak of Google's
internal "Content Warehouse API" documentation, analyzed publicly by Michael King (iPullRank)
and Rand Fishkin. Where a fact comes from Google's own public pages I say so. Where it comes
from the trial or the leak I say so. Where I am reasoning about the shape of the system rather
than quoting it, I label it inference.

---

## 1. The user

Meet Rohan. It is 11pm, he is on his laptop, and both his feet ache after a half marathon he
had no business running. He types into the Google box:

> best running shoes for flat feet

He has never searched that exact phrase before. Neither, probably, has anyone else in quite
that wording. He is not going to scroll. He will look at the first two or three results, click
one, and if it does not answer him in about ten seconds he will hit back and click the next.
He does not know the word "ranking." He just knows that Google either puts the right page at
the top or it does not.

The second user is Meera, who at 8am types a single word:

> gmail

She does not want ten pages about email. She wants one link, the login page, first, and she
wants it so reliably that she has stopped typing the full URL. That is also ranking. The same
machine has to serve both the never-seen sentence and the worn-smooth single word, from the
same index, in the same fraction of a second.

---

## 2. The real problem

Here is the pain, said plainly. The web is not a library with a card catalog. It is hundreds
of billions of pages, most of them junk, many of them lying, a lot of them actively built to
trick whatever ranks them. When Rohan types his sentence, the number of pages that contain
the words "best," "running," "shoes," "flat," and "feet" somewhere is enormous. Easily
hundreds of thousands. Many are spammy affiliate pages stuffed with those exact words. The
page that would actually help Rohan, a podiatrist-reviewed guide that explains overpronation
and names three real shoes, might use the phrase "flat feet" only twice.

So keyword matching alone is worse than useless. It rewards the stuffers. The real problem is
ordering under adversarial conditions: take a matching set that is too big to read, where the
loudest pages are often the worst, and put the genuinely most useful one first, in under a
second, for a query the system may be seeing for the first time. Get it wrong and Rohan goes
to Reddit, or to an AI chatbot, and Google loses the one thing it sells.

---

## 3. The feature in one sentence

Core ranking is the system that takes every page in the index matching a query, scores each
one on how topical and how trustworthy it is for that specific query and user, and returns a
sorted list so the most useful result sits at position one.

---

## 4. Jobs to be done

What is Rohan really hiring this feature to do?

- "Decide for me." He will not evaluate a hundred pages. He is hiring Google to do the
  judging and hand him a winner.
- "Punish the fakers for me." He cannot tell a keyword-stuffed affiliate farm from a real
  guide at a glance. He wants that filtering done upstream, invisibly.
- "Understand what I meant, not what I typed." He typed "flat feet." He meant "I have fallen
  arches and overpronate and need stability or motion-control shoes." He is hiring ranking to
  cross that gap.
- Meera's job is different: "Take me to the thing I always want, instantly, so I never have
  to think about it again." Navigational certainty.

---

## 5. How it works for the user

Rohan sees nothing of the machinery. He types, he hits enter, and in roughly half a second a
page of results appears. The top result is a guide titled something like "The 8 Best Running
Shoes for Flat Feet (2026), Tested by Physical Therapists." Below it, a running-magazine
review. Below that, a shoe brand's stability-shoe page. The keyword-stuffed affiliate farm
that mentions "flat feet" fourteen times is on page four, where he will never see it.

Meera types "gmail," and the Gmail login page is result one, every single time, with the
other nine results barely registering in her mind. The list feels obvious, almost boring. That
boredom is the product working. The invisible work is the entire point: a good ranking looks
like there was nothing to decide.

---

## 6. The actual flow, step by step

1. Rohan types the query. Autocomplete (a different feature, 06-16) offers suggestions as he
   types. He ignores them and hits enter on his own sentence.
2. The query hits a Google front-end server. Before ranking, the query is understood: spell
   correction (05-07's feature) runs, synonyms are added ("sneakers" might join "shoes"),
   and language models including BERT read the whole sentence to grasp that "for flat feet"
   is a constraint, not just three more words.
3. Retrieval (the matching half). The query is turned into terms, and the inverted index is
   consulted to pull the set of pages that contain those terms. This is the candidate set. It
   is big, and it is cheap to get, and it is NOT yet ordered by quality.
4. Scoring and primary ranking (the ranking half). Each candidate gets a base relevance score
   from the core ranking algorithm. The set is cut down hard, from hundreds of thousands to a
   few thousand to a few hundred.
5. Deep re-ranking. Expensive machine-learned models (RankBrain, DeepRank, RankEmbed) look
   at the survivors and re-order them using learned meaning, not just word overlap.
6. Re-ranking adjustments (Twiddlers). A second layer of functions nudges the order: enforce
   diversity (do not show six results from the same domain), boost fresh pages for queries
   that deserve freshness, demote pages flagged for spam or other problems.
7. Click-signal re-ranking (Navboost). For a query popular enough to have click history, a
   system called Navboost adjusts the order based on what real users clicked and how long
   they stayed, learned over the past 13 months.
8. The whole page is assembled (a system called Glue handles the non-web pieces: images, a
   video block, "People also ask"), and the final ordered list is serialized and sent to
   Rohan's browser. Total time: a fraction of a second. The sort happened on Google's
   servers, across thousands of machines, never on the phone.

Everything from step 3 to step 8 is the subject of the next section.

---

## 7. Under the hood, like the engineer

The one idea that organizes everything here: matching and ranking are two different halves,
and they have completely different costs. Matching has to be cheap because it touches a huge
set. Ranking can be expensive because by the time it runs the set is small. Every scaling
decision in Google ranking is some version of "do the cheap thing on the big set, save the
expensive thing for the small set."

### The two halves, concretely

Start with the data structure. The index is an inverted index. (Fact, Google's own "How
Search Works" describes the index as organized so that each word maps to the list of pages
containing it, "like the index at the back of a book.") For the term "flat," there is a
posting list: the sorted list of document IDs of every page that contains "flat," along with
positions and metadata. Same for "feet," "running," "shoes," "best."

Retrieval for Rohan's query is, at its core, intersecting and unioning these posting lists.
The pages that matter most contain all the terms, so the engine walks the five sorted posting
lists together and finds the document IDs that appear across them. Because the lists are
sorted by document ID, this is a merge, not a search: you advance pointers, you skip ahead
using skip pointers, you never scan the whole web. The cost is roughly the length of the
posting lists you touch, not the size of the index. That is the whole reason an inverted index
exists. Rohan's candidate set, say 200,000 pages, falls out of this step cheaply.

Now the ranking half. You cannot run a heavy neural network on 200,000 pages in half a
second. So ranking is staged, cheapest first.

### Stage one: the base relevance score (topicality, the "ABC" signals)

The primary ranking algorithm (internal name Ascorer, surfaced in the 2024 API leak, so treat
the name as leak-sourced, not Google-confirmed) computes a base relevance score for each
candidate. Court exhibits from the antitrust trial gave this base score a shape, and SEO
engineers summarized it as the "ABC" signals that build "topicality," written internally as
T*. (Fact that these terms appeared in trial exhibits and the leak; the exact math is not
public.)

- A is Anchors: the text other pages use when they link to this page. If fifty running blogs
  link to a page with the words "flat feet shoe guide," those anchor words are strong evidence
  of what the page is about, and crucially they come from other people, not from the page
  itself, so they are harder to fake than on-page text.
- B is Body: the terms in the document itself, their frequency, their positions, whether
  "flat feet" appears in the title and headings or just once in a footer.
- C is Clicks: click signals for this query and page, which is where Navboost feeds in (more
  below).

Topicality is deliberately built so that no single component can be gamed into dominance. A
page can stuff its body all it wants; if no one links to it with relevant anchors and no one
clicks it, its topicality stays low. This stage is cheap enough to run across the full
candidate set, and it does the heavy cutting: 200,000 candidates down to a few thousand.

Worth naming a tension the trial exposed. Many of these signals are hand-crafted and
hand-tuned, not learned end to end. In testimony, Google's engineers described a deliberate
preference for signals they can understand and debug, because a fully black-box ranker is
impossible to fix when it does something stupid on a visible query. (Testimony-sourced; the
phrasing is my summary.) So core ranking in 2026 is a hybrid: interpretable hand-built signals
for the base, machine learning layered on top.

### Stage two: the deep models (RankBrain, DeepRank, RankEmbed)

Now the set is small enough, a few thousand, to spend real compute. Three machine-learned
systems do the understanding that keyword overlap cannot.

RankBrain (launched 2015). This was Google's first deep-learning ranking signal. It embeds
words and queries into vectors, points in a high-dimensional space where "sneakers" lands near
"running shoes" and "fallen arches" lands near "flat feet." That is how Rohan's never-seen
sentence gets handled: RankBrain does not need to have seen it before, it just needs the
vector for the sentence to land near vectors of queries it has seen. (Fact, Google via
Bloomberg, 2015: roughly 15% of daily queries are ones Google has never seen before, which is
exactly the problem RankBrain was built for. Google called it, at the time, the third most
important of the hundreds of ranking signals.)

DeepRank. In Nayak's testimony, DeepRank is described as the internal name for BERT applied to
ranking. BERT is the 2018 language model that reads a sentence in both directions at once, so
it understands that in "running shoes for flat feet" the word "for" ties the shoes to the
feet, a relationship bag-of-words ranking throws away. (Fact: BERT rolled out in Search in
October 2019, affecting about 1 in 10 queries in English in the US at launch per Google's
announcement, later expanding to all languages and to featured snippets. DeepRank as the
internal name is trial/leak-sourced.)

RankEmbed and RankEmbedBERT. A dual-encoder model: it embeds the query into a vector and the
document into a vector in the same space, and relevance becomes how close the two vectors sit.
What makes it interesting is the training data. Per the court's findings, RankEmbed and
RankEmbedBERT are trained on two things: search logs (what people searched and clicked) and
human rater scores (the IS score, below). It is fast and strong on common queries. (Fact:
trial findings describe the two training sources and that rater-trained RankEmbedBERT improved
Google's handling of complex, long-tail queries.)

There is also MUM (2021), described by Google as roughly a thousand times more capable than
BERT and multimodal, but Google has said it is used narrowly, not as a broad live ranker, so
it is not doing the ordering for Rohan's query today. (Fact, Google blog, plus absence of
evidence it is a general ranking signal.)

### The human rater loop (IS score)

How does Google know its ranking is good, so it can train these models and tune the signals?
It does not ask the algorithm. It asks people. Google employs thousands of external Search
Quality Raters who follow a published document, the Search Quality Rater Guidelines (a real,
public PDF, over 170 pages). For a sample query, raters look at the results and score them on
Needs Met (did this answer the query) and Page Quality (is this page trustworthy and
expert, the E-E-A-T idea: Experience, Expertise, Authoritativeness, Trust).

Those judgments roll up into an Information Satisfaction score, called IS (and a variant IS4),
confirmed as a real internal metric in the trial. Critically, the IS score is NOT a live
ranking signal that lifts your individual page. It is the measuring stick. Engineers propose a
ranking change, run it, and see whether the IS score goes up on a held-out set of
rater-judged queries. And, as RankEmbed's training shows, those same rater scores become
training labels for the deep models. So the loop is: humans define "good" on a sample, the
system learns to predict "good," the learned system ranks the whole web. (Fact: SQRG is
public; IS/IS4 and rater-score training confirmed in trial.)

### Navboost: the click memory

Here is the piece Google denied in public for a decade. Navboost is a re-ranking system driven
by aggregated, anonymized click data. (Fact: Nayak testified to it; it was a central reveal of
the trial.) For a query that has enough history, Navboost remembers what users clicked and, as
importantly, how they behaved after clicking: a "good click" where the user stayed on the page
versus a click where the user bounced straight back to the results and clicked something else
(a "bad click," sometimes called pogo-sticking). A long last click, where the user clicked a
result and did not come back, is strong evidence that result satisfied them.

Two concrete details from testimony. Navboost works on a rolling window of about 13 months of
click data (it had been 18 months and was cut back). And it is sliced by country and by
device, so the clicks that teach ranking for "football" in the US on mobile are kept separate
from "football" in the UK on desktop, because the right answer differs. (Fact, trial.)

This is why Meera's "gmail" query is rock solid. It is one of the most-clicked queries on
Earth, and essentially everyone clicks the Gmail login result and stays. Navboost has
memorized that answer so hard that nothing dislodges it. Navboost is, in effect, a giant
memorization layer for the head of the query distribution, where behavior is the truth. For
Rohan's rare query there may be little or no Navboost data, which is exactly why the deep
semantic models (RankBrain, DeepRank) carry more of the load on the long tail. The two
mechanisms cover for each other: memorize the head, generalize the tail.

### Twiddlers and Glue: the finishing layer

After the primary ranking, re-ranking functions the leak calls Twiddlers run (leak-sourced
name). A Twiddler does not re-score from scratch; it adjusts the existing order: boost, demote,
or filter. One enforces host diversity so page one is not six results from the same site. One
boosts freshness for queries that deserve it (a query like "earthquake" needs today's page,
"pythagorean theorem" does not). One demotes pages caught by spam systems. These are layered
and cheap because they operate on the final few dozen results, not the candidate set.

Glue is the sibling of Navboost for the whole results page. Navboost orders the web links;
Glue decides where the image block, the video, the "People also ask" box, and other features
slot into the page, using the same kind of interaction data. (Fact, trial.)

### The scale story, three tiers

Tier one, 1,000 pages. You do not need any of this. Score every page against the query with a
simple formula (TF-IDF or BM25), sort the 1,000 numbers, take the top 10. A laptop does it in
milliseconds. No inverted index strictly required, you could scan every document. At 1,000
documents the "clever" parts are overkill.

Tier two, 100,000 pages. Scanning every document per query now hurts. You build an inverted
index so retrieval touches only posting lists, not all 100,000 pages. It still fits in memory
on one machine. You retrieve the matching few thousand, score them, sort, return top 10.
Single box, single index, straightforward. What is starting to break: a single relevance
formula cannot tell a good page from a keyword-stuffed one, so you begin layering signals
(links, maybe early click data). Still, one machine holds the whole thing.

Tier three, 10 million to hundreds of billions of pages (the real Google: the index is well
over 100,000,000 gigabytes per Google's own docs, spanning hundreds of billions of pages).
Everything about one machine fails. The fixes, each one a direct answer to a specific break:

- The index does not fit on one machine, so it is sharded across thousands of machines by
  document (each shard holds the full inverted index for its slice of the web). A query is
  scatter-gathered: a root server fans the query to all shards, each shard finds and scores
  its own local top-k candidates in parallel, and the root merges the per-shard winners into
  one list. This is the only way to search hundreds of billions of pages in half a second:
  do it in thousands of places at once.
- Running heavy models on the full candidate set is impossible, so ranking is staged as
  described: cheap topicality (Ascorer, the ABC signals) cuts hundreds of thousands to a few
  thousand, then the expensive deep models (DeepRank, RankEmbed) re-rank only the survivors.
  The expensive compute per page is paid on hundreds of pages, not hundreds of billions. This
  "retrieve wide and cheap, rank narrow and expensive" cascade is the heart of all web-scale
  ranking.
- The same hot queries repeat constantly, so results and sub-computations for popular queries
  are cached, and Navboost's memorized orderings turn the entire head of the distribution into
  what is effectively a lookup. Meera's "gmail" barely ranks anything live; it recalls an
  answer.
- Storage is tiered: the pages and posting lists for popular, high-quality documents live in
  fast storage (RAM, flash), the long cold tail on slower, cheaper disk, so the common case
  stays fast and the rare case stays affordable.
- And the sort itself always happens server-side, across the shards and the merge, never on
  the client. The phone receives an already-ordered list. Sorting hundreds of billions of
  anything on a phone is a non-idea; the architecture exists precisely so the device does
  nothing but display ten links.

The through-line across all three tiers is the same sentence from the top: cheap work on the
big set, expensive work on the small set, and push every heavy computation you can off the
live path into something precomputed or memorized.

---

## 8. The retention and habit mechanic

The loop here is a trust reflex, and it is self-reinforcing in a way that is almost unfair.

Rohan types a sentence, gets the right answer at position one, and clicks it. That click,
aggregated with millions of others, feeds Navboost, which makes the ranking for that query
even better, which makes the next person's result even more likely to satisfy, which earns
another good click. Clicks improve ranking, better ranking earns more clicks. The data is the
product and the product generates the data. This is why click history over 13 months is such a
jealously guarded asset: a competitor can copy the algorithm but cannot copy the decade of
behavior that tuned it.

The metric this moves is retention, specifically return searches and the speed of each
session. When ranking is good, "I have a question" compiles directly into "just Google it,"
with no conscious choice of tool. That reflex is the entire franchise.

And here is the real-world example that shows how much the company believes this. The whole
antitrust trial was about Google paying to be the default search engine in browsers and on
phones. Court evidence put the 2021 figure at roughly 26 billion dollars in total payments for
default placement, with around 20 billion of that going to Apple alone. Why pay that much for
a default, when switching a search engine takes two taps? Because defaults feed the click
flywheel. Every default query is another good click, another 13 months of behavior, another
increment of ranking quality that a rival starting from zero cannot match. The default buys
the data, the data buys the ranking, the ranking buys the habit. Retention is not a feature
bolted onto ranking; good ranking IS the retention mechanic, and the default deals are Google
spending tens of billions to keep feeding it.

The honest tension: this same flywheel means the rich get richer, and a genuinely better
upstart search engine is fighting not just Google's code but Google's accumulated click
memory. That moat is exactly what the trial was arguing about.

---

## 9. The lesson for Rare.lab

Rare.lab has a library problem coming, the same shape as Rohan's query. As the node-based
editor grows, there will be a huge catalog of effects, presets, node templates, and
community-shared graphs. When a user types "cheap mobile-friendly bloom" into the asset
search, you face Google's exact situation: a matching set too big to rank with the expensive
tool, where the loudest-named asset is often not the best one.

Steal the two-stage cascade directly. Do not run a heavy relevance model over the whole
library. Build the cheap half first: an inverted index over asset names, tags, and
descriptions, plus approximate-nearest-neighbor lookup over precomputed embeddings, to cut
10 million assets down to a few hundred candidates in a couple of milliseconds. Only then run
the expensive model (a learned quality-and-fit scorer) on those few hundred. Cheap and wide,
then expensive and narrow. Keep the shortlist size as a single tuning dial, so the live search
stays fast whether the library is 1,000 assets or 10 million.

Then steal Navboost, because it is the part most teams miss. Your single best relevance signal
is not the asset's name or its author's description, it is what users actually did with it:
which presets they dropped into a graph, kept, compiled, and shipped versus which they previewed
and deleted in two seconds. That is your "good click" versus "bad click." Log it per query and
per context (mobile target versus desktop, 2D versus 3D scene, the way Navboost slices by
device and country). Aggregate it offline into a memorized re-rank table. At edit time, reading
that table is an O(1) lookup, not a live computation. The heavy learning happens in a nightly
batch job; the editor path, which has a frame budget to respect, does nothing but a cheap
lookup and a small sort. That is the same offline-think, online-lookup discipline that lets
Google answer Meera's "gmail" by recalling an answer instead of computing one.

One more, from the trial's quieter lesson: keep some of your ranking signals hand-tunable and
interpretable, not a single black box. When a user complains that a terrible preset keeps
surfacing for "glass refraction," you want to be able to open the hood and see which signal
over-weighted it, and turn that knob. A fully learned ranker that you cannot debug will, on
your most visible queries, eventually do something indefensible with no handle to fix it.
Google, with the most ranking talent on the planet, chose the hybrid on purpose. So should you.

---

## Sources

- US v. Google LLC (DOJ antitrust trial, 2023 to 2024). Testimony of Pandu Nayak and trial
  exhibits are the primary source for Navboost (13-month click window, country/device slicing),
  Glue, the IS / IS4 information-satisfaction score, DeepRank as the internal name for BERT in
  ranking, and RankEmbed / RankEmbedBERT training on search logs plus rater scores. Summaries:
  https://searchengineland.com/how-google-search-ranking-works-445141
  https://www.hobo-web.co.uk/google-vs-doj/
- Michael King (iPullRank), "Secrets from the Google Algorithm Leak," analysis of the May 2024
  Content Warehouse API documentation leak. Primary public analysis of Ascorer, Twiddlers, the
  ABC signals (Anchors, Body, Clicks) and RankEmbed. https://ipullrank.com/google-algo-leak
- Search Engine Land, "The ABCs of Google ranking signals: what top search engineers revealed."
  On topicality (T*) built from Anchors, Body, Clicks as surfaced in court.
  https://searchengineland.com/google-abc-ranking-signals-455360
- Google, "How Search Works" (official). The inverted index description ("like the index at the
  back of a book"), index size (well over 100,000,000 gigabytes, hundreds of billions of pages),
  and the retrieve-then-rank framing. https://www.google.com/search/howsearchworks/
- Bloomberg (Jack Clark), "Google Turning Its Lucrative Web Search Over to AI Machines," October
  2015. RankBrain: embeds queries as vectors, handles the ~15% of never-before-seen daily
  queries, described by Google as the third most important ranking signal at the time. Greg
  Corrado quoted. (Coverage: TheNextWeb, Search Engine Land.)
  https://thenextweb.com/google/2015/10/26/how-google-handles-search-queries-its-never-seen-before/
- Google Search blog / Pandu Nayak, "Understanding searches better than ever before," October
  2019. BERT in Search: roughly 1 in 10 English US queries at launch, later all languages and
  featured snippets. (Coverage: TechCrunch, Search Engine Journal.)
  https://techcrunch.com/2019/10/25/google-brings-in-bert-to-improve-its-search-results/
- Google Search Quality Rater Guidelines (public PDF). The Needs Met and Page Quality (E-E-A-T)
  framework that produces the human judgments behind the IS score.
- Resoneo, "Google leak Part 4: the Neural Revolution," on DeepRank, RankEmbed-BERT, and the
  2019-onward neural ranking stack as described in the leak. https://www.resoneo.com/?p=21264
