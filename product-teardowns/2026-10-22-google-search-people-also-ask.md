# Google Search: People Also Ask (the expandable "related questions" box)

Date: 2026-10-22
Product: Google Search
Feature: People Also Ask (PAA), the accordion of related questions that sits in the results page and grows every time you open one.

A quick note on what this is NOT, so it does not blur into earlier teardowns.
This is not the ten blue links and how they get ordered (core ranking, 2026-10-10).
This is not the single boxed answer at the very top (Featured Snippets, 2026-08-20).
This is not the right-hand fact card about a person or place (Knowledge Panel,
2026-08-05). It is not the grey suggestions that drop down while you type
(Autocomplete, 2026-06-16). PAA is its own machine: a set of folded questions,
each of which opens to show a short extracted answer, and opening one quietly
appends more questions to the bottom of the list. The novel part, the part worth
a whole teardown, is that it behaves like a tree you grow by clicking, and that
each answer is lifted from a page using the same extraction brain as Featured
Snippets.

---

## 1. The user, and their day

Meet Arjun. It is 9:40 on a Tuesday night in Pune. His video call for tomorrow's
standup keeps freezing, the little buffering circle spinning while his manager's
face turns into a cubist painting. He is annoyed and a little embarrassed. He
opens Google on his phone and types what a frustrated person actually types:
"why is my wifi so slow at night".

He is not a networking engineer. He does not know the words "channel congestion"
or "QoS". He has a vague problem and a vague sentence, and he wants to stop the
freezing before the call drops again. He does not even know the right question to
ask yet. That last bit is the whole point of this feature.

---

## 2. The real problem, said plainly

Here is the honest truth about search: most people do not know the question that
would actually solve their problem. Arjun typed "why is my wifi so slow at
night", but the thing that will fix his call is probably "how do I change my
router's wifi channel" or "does my neighbour's wifi interfere with mine". He
cannot ask for those because he has never heard of them.

So the real pain is not "I cannot find an answer." The real pain is "I do not
know what to ask." A friend who happened to be good with routers would not just
answer the question Arjun asked. They would say, "Yeah, slow at night usually
means everyone on your street is streaming at once and your channel is crowded.
Have you tried switching the channel? Is your router sitting behind the TV? How
old is it?" The friend hands Arjun better questions than the one he walked in
with. People Also Ask is Google trying to be that friend.

---

## 3. The feature in one sentence

People Also Ask is a box of related follow-up questions on the results page,
each one folding open to a short answer pulled from a web page, and each one you
open spawning a few more questions underneath, so you can keep pulling the thread
as far as you want.

---

## 4. Jobs to be done

What is Arjun really hiring this box to do?

- "Tell me what I should have asked." He typed a weak question. He wants the
  strong ones surfaced. (Real example: he typed "why is my wifi so slow at
  night" and the box shows "How do I fix slow wifi in the evening?" and "Does
  router placement affect wifi speed?", which are the questions that actually
  lead to a fix.)
- "Answer it without making me leave." He wants a two line answer right here, not
  a click to a page covered in ads that buries the fix in paragraph nine.
- "Let me go deeper if I want, but only if I want." He wants to open one more
  question, then one more, following his own curiosity, without a second search.
- "Reassure me I am not alone." The literal words "People also ask" tell him
  thousands of others had the same stupid freezing call. That is quietly
  comforting and it builds trust in the answers.

---

## 5. How it works for the user, the visible experience

Arjun hits search. Under the first result or two, he sees a box. Inside it, four
questions, each with a small plus or chevron on the right:

- How do I fix slow wifi at night?
- Why is my wifi slow in the evening but fast in the morning?
- Does changing my wifi channel help?
- How many devices can a home router handle?

He taps the first one. It slides open and shows a short answer, three sentences,
with a blue link to the source page and that page's title underneath. Here is the
magic moment: the instant it opens, two or three new questions appear at the
bottom of the box. The list grew because he touched it.

He taps "Does changing my wifi channel help?". It opens, gives him a crisp
answer ("Yes. On 2.4 GHz, channels 1, 6, and 11 do not overlap. Switching to a
less crowded one reduces interference."), and again the box grows with questions
like "How do I change my wifi channel on a TP-Link router?". Now he is somewhere
he could never have typed his way to in one shot. He went from "why is it slow"
to "how do I change the channel on my exact router" in three taps, and he never
typed a second query.

---

## 6. The actual flow, step by step

1. Arjun types "why is my wifi so slow at night" and taps the search icon.
2. His phone sends one HTTPS request to Google with the query string and some
   context (language, rough location, device type).
3. Google's servers do the normal search, and in parallel decide: does this query
   deserve a PAA box, and if so, which questions go in it? (Decision made
   server-side, never on the phone.)
4. The results page comes back. The PAA box arrives with its first set of
   questions as collapsed items. Crucially, the answers are usually NOT all sent
   yet. The questions are cheap text; the answers are fetched when needed.
5. Arjun taps a question. The phone fires a small background request (an
   asynchronous fetch) asking for that question's answer AND for the next batch of
   related questions.
6. The server returns the extracted answer snippet (text, source URL, source
   title) plus a fresh list of follow-up questions tied to the one he opened.
7. The box animates open to show the answer and appends the new questions to the
   bottom.
8. Repeat from step 5 as long as Arjun keeps tapping. Most chains dry up after a
   handful of expansions; a few have been observed to go nearly forever.

The two things to hold onto: the questions load first and cheap, the answers load
lazily on tap, and tapping a question is what fetches the next generation of
questions. That is a tree being walked one node at a time, expanded on demand.

---

## 7. Under the hood, like the engineer

This is the heart of it. I will split confirmed fact from well grounded
inference, because Google has published a patent and a launch history but not the
live system's internals.

### 7a. Two halves, like every search feature: matching then ranking

Every teardown in this ledger keeps hitting the same spine: first you MATCH (find
a candidate set out of something huge), then you RANK (order the small survivor
set with an expensive model). PAA is really two matching-then-ranking problems
stacked on top of each other.

- Problem A: given the query, which QUESTIONS should appear?
- Problem B: given a chosen question, which PAGE and which PASSAGE answers it, and
  what are the next questions?

Hold these apart. They use different data and different structures.

### 7b. Where the questions come from (CONFIRMED patent, inference on live use)

Google holds a patent, US9213748B1, "Generating related questions for search
queries," with Yossi Matias and Ziv Bar-Yossef named as inventors (both real,
senior Google Search people). The patent describes the mechanism cleanly, and it
maps so well onto PAA's observed behaviour that it is the most likely basis for
it. I label the patent as fact and the "this is what PAA runs" link as inference.

The patent's core idea is a QUESTION GRAPH:

- Take a large database of questions. Each distinct question is one NODE in a
  graph. Think of millions of question strings: "how do I fix slow wifi",
  "why is my wifi slow at night", "how do I change my wifi channel", and so on.
- Draw an EDGE between two questions when they share at least a threshold number
  of common "selected resources." A selected resource is a page that users
  clicked after asking that question. In plain words: if lots of people who asked
  "why is my wifi slow at night" ended up clicking the same pages as people who
  asked "how do I change my wifi channel," those two questions get linked.

So the graph is built from CLICK CO-SELECTION. Two questions are neighbours not
because their words look alike, but because they send people to the same answers.
This is beautiful, because it connects "why is it slow" to "change the channel"
even though the words barely overlap. The words are different; the destinations
are the same.

Why a graph and not a list? Because the whole feature is "walk from a question to
its neighbours." A graph is the exact data structure for "what is adjacent to
this node." Clicking a PAA question and getting more questions underneath is
literally a walk to that node's neighbours in this graph. The tree Arjun grew by
tapping is a breadth-first expansion of one region of the question graph, one hop
at a time.

Concrete walk: Arjun's query "why is my wifi so slow at night" maps to a node.
Its highest-weight neighbours (most shared clicked pages, most asked) are the
first four questions in the box. When he opens "Does changing my wifi channel
help?", Google jumps to THAT node and shows ITS neighbours, which is why the new
questions suddenly get very channel-specific ("How do I change my wifi channel on
a TP-Link router?"). Each tap re-roots the walk on the node you opened.

### 7c. Matching, step one: the inverted index does the first cut

Before any graph walk, Google has to turn "why is my wifi so slow at night" into
candidate questions and candidate pages out of a corpus of hundreds of billions
of web pages. This is the classic INVERTED INDEX job, the same one from the
Autocomplete (2026-06-16), Amazon search (2026-06-23) and Gmail search
(2026-10-03) teardowns.

An inverted index is a hash map from a word to the list of documents that contain
it. The word "wifi" points to a posting list of every page (and every stored
question) mentioning wifi. The word "slow" points to its own posting list. To
find candidates for the query, you fetch the posting list for each important
query word and intersect them: pages that have wifi AND slow AND (night or
evening). That intersection is a few thousand candidates out of hundreds of
billions, computed in milliseconds, because you never scan the corpus, you only
walk short precomputed lists. This is the MATCHING half for Problem B (which
pages could answer).

The key engineering fact, and the framework asks me to stress it: the sorting and
intersecting happen SERVER-SIDE, inside Google's index serving systems, not on
Arjun's phone. The phone receives a tiny, already-ordered result. A phone could
never hold the index or do the intersection. The expensive work is done far away,
on sharded index servers, and the device just renders the answer.

### 7d. Ranking, step two: pick the question order, then extract the answer

Two rankings happen.

Ranking the questions: the candidate related questions from the graph get ordered
by signals the patent and Google's public hints point at: how often that question
is actually asked (popularity), how relevant it is to the original query, and how
confidently Google can answer it well. A question Google cannot answer cleanly is
a bad box item, so answerability is itself a ranking signal. The top four (or so)
survive into the visible box; the rest wait to be shown as you expand.

Ranking and extracting the answer: this is CONFIRMED to share machinery with
Featured Snippets. Once Google picks the best page to answer "Does changing my
wifi channel help?", it must find the exact PASSAGE on that page that answers it
and lift two or three sentences out. That extraction brain got a big upgrade in
2019 when Google applied BERT (a transformer language model) to Search and
specifically to Featured Snippets, rolled out in 70 languages in December 2019,
to better understand the question and match it to the right passage. In 2021
Google shipped "passage ranking" (sometimes called passage indexing), which lets
it rank a single relevant passage deep inside an otherwise-unfocused page, rather
than judging the whole page as one blob. PAA answers ride on exactly this: find
the page, find the passage, extract the sentences, show the source link.

Walked example of extraction: the source page for the channel question might be a
2,000 word router guide. The answer Arjun sees ("On 2.4 GHz, channels 1, 6, and
11 do not overlap...") is three sentences from somewhere in the middle of that
page. BERT-style passage understanding is what lets Google reach into paragraph
nine and grab precisely those sentences instead of the page's intro.

### 7e. The lazy, grow-on-tap design (CONFIRMED behaviour, inference on plumbing)

The observed behaviour is confirmed by many trackers: the box loads with a few
questions, opening one appends new ones, and the answers are fetched as you
expand. The engineering reason is obvious once you see it. If Google eagerly
computed and sent the answer for every question, and every question's children,
and their children, it would be computing an unbounded tree for a box most people
tap zero or one times. So it does the sane thing: send cheap question text now,
and fetch the expensive extracted answer and the next question generation only
when a finger actually lands on a question. This is lazy evaluation of a tree,
and it is the same instinct as pagination (do not compute page 50 until someone
asks for it). Inference, but the only sane build.

### 7f. The scale story at three tiers

What grows here is not one catalogue. Two things grow: the size of the QUESTION
GRAPH (how many question nodes and edges you maintain) and the QUERY RATE (how
many PAA boxes you build per second). Google runs on the order of billions of
searches a day (widely cited around 8.5 billion per day, very roughly 99,000 per
second), and a large share of result pages have carried a PAA box, so both
numbers are enormous.

Tier 1, about 1,000 questions (a tiny vertical, say a single product's help
site). You do not need a graph or an index. Keep a small table of questions, and
for each, a hand-curated or simple list of a few related questions. Answer by
grepping the help docs. Precomputing, sharding, BERT: all over-engineering. Ship
the table and a text search. A startup's in-app help "People also ask" would
rationally live here.

Tier 2, about 100,000 questions (a big single-domain Q and A site). Now the
naive "compare this question to every other question" is 100,000 comparisons per
lookup, too slow to do live. Two moves. First, build the inverted index so
matching a query to candidate questions is a posting-list intersection, not a
scan. Second, PRECOMPUTE the related-question edges in a nightly batch job: for
each question, find and store its top neighbours offline, so the live request is a
keyed lookup ("give me the stored neighbours of node X") instead of a graph
computation. This is the first appearance of the pattern that shows up in almost
every teardown here: do the expensive thinking offline, make the live path a
cached lookup. Discover Weekly (2026-06-13) did it with song vectors; Autocomplete
(2026-06-16) did it with top-k per trie node; PAA does it with precomputed
question neighbours.

Tier 3, hundreds of billions of pages and billions of queries a day (real
Google). Four walls appear, and each has a known survival move.

- The index does not fit on one machine. Survival: SHARD the inverted index
  across thousands of servers by document, fan the query out to all shards, merge
  the top results. Same shard-and-merge as every web-scale index.
- Reads vastly outnumber writes. Survival: REPLICATE each shard many times so
  billions of read queries spread across replicas, and the index only rebuilds on
  a slower crawl-and-index cadence (crawling and indexing was the 2026-09-08
  teardown).
- The question graph plus answer extraction is too expensive to run fully live for
  every box. Survival: precompute the question neighbours offline (as in Tier 2
  but now sharded by question id), and cache popular PAA boxes hard. "Why is my
  wifi slow" is asked constantly, so its box is almost certainly served warm from
  cache, not recomputed. The long-tail weird query is the one that actually hits
  the live path.
- The expand-on-tap tree could be unbounded. Survival: lazy fetch (only on tap)
  plus practical caps and dedup, so the same question does not appear twice in one
  growing chain and the chain is allowed to dry up. This is why most PAA chains
  stop after a few expansions instead of going forever.

The through-line, identical to the rest of this ledger: push the graph building,
the edge precomputation, the index building and the answer extraction OFFLINE,
and reduce the live request to a cached lookup plus a short posting-list
intersection plus, at most, one passage extraction. The box must appear in the
same fraction of a second as the ten blue links, whether the question graph has a
thousand nodes or billions.

### 7g. A quick honesty box

Confirmed: PAA exists since 2015; the US9213748B1 patent and its question-graph,
click-co-selection method; BERT applied to Featured Snippets in December 2019 in
70 languages; passage ranking shipping in 2021; PAA answers sharing the Featured
Snippet extraction path; the lazy grow-on-tap behaviour. Inference, clearly
labelled: that the live PAA system runs exactly the patent; the inverted-index
and sharding specifics (standard class-of-problem design, not published for PAA);
the lazy-fetch plumbing details; the caching specifics. Google does not publish
the live internals, so where I describe plumbing I am describing how this class
of problem is universally solved, not a leaked design doc.

---

## 8. The retention and habit mechanic

PAA is a session-depth and re-query machine, and it quietly serves Google's core
business.

The loop: cue (you have a fuzzy problem and a weak query) to action (you tap a
question that is sharper than what you typed) to reward (a clean two line answer,
right now, no page load) to a NEW cue (the box just grew, and one of the new
questions is exactly the next thing you were starting to wonder). That last step
is the hook. The feature manufactures your next question before you have fully
formed it, so instead of leaving Google you keep tapping. It turns one search into
a ten minute exploration without a single new query typed.

Which metric does it move? Mostly ENGAGEMENT and RETENTION of the search session,
and indirectly REVENUE. More time inside the results page, more pages per visit,
and more trust that Google is the place where a vague worry becomes a concrete
fix. That trust is why you come back tomorrow with the next fuzzy problem instead
of asking a chatbot or a forum. Real observed example of the scale of this habit:
Moz's Dr. Pete Meyers, tracking the MozCast data set, watched PAA go from a minor
feature to appearing on a very large share of result pages (his trackers reported
it on the order of 40 percent early on, climbing past 80 percent at its peak,
before settling back). Google does not spread a feature to that many result pages
unless it is reliably pulling people deeper into search. A feature on most of your
pages is a feature doing a retention job.

There is a sharp edge worth naming: the reward (a good enough two line answer) can
satisfy Arjun so completely that he never clicks a publisher's site. That "answer
without a click" is great for Google's engagement and controversial for the web's
publishers. The same mechanic that retains the user is the one that worries the
sources it quotes.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime. The day you build "suggest the next
node" or "ask the canvas a question" assist, PAA is your exact blueprint, and the
lessons are biased to scale and performance as requested.

1. Model "what connects to this node" as a precomputed neighbour graph, not a
   live computation. PAA's question graph has one node per question and edges from
   click co-selection, all built offline, so the live tap is a lookup of stored
   neighbours. Build the same for nodes: every shader graph your users create is a
   training document, every node a word, and co-occurrence (which nodes get wired
   together) defines edges. When a creator drops a Noise node and you want to
   suggest "Color Ramp, Displace, Fresnel," that must be a keyed lookup of the
   Noise node's precomputed neighbours, never a model run or a corpus scan at
   interaction time. The editor has a 60fps budget; a live graph walk on the hot
   path will blow it.

2. Make expansion lazy and grow-on-demand, exactly like PAA's grow-on-tap. Do not
   eagerly compute the whole suggestion tree ("if they add Noise, then maybe Color
   Ramp, then maybe..."). Compute and show the first few suggestions cheap, and
   fetch the next generation only when the creator actually accepts one and
   re-roots the context on it. An unbounded suggestion tree computed up front is
   wasted work for a feature most users tap once. Lazy fetch plus dedup plus a
   practical cap is the whole trick.

3. Split matching from ranking and keep the heavy half offline. PAA matches
   candidate questions and pages with a cheap inverted-index intersection, then
   ranks the survivors, then extracts an answer with the expensive BERT-class
   model only for the handful shown. Mirror it: a cheap candidate step (neighbour
   lookup, a few dozen nodes) feeds a light ranker ordered by the creator's recent
   actions, and you only run anything expensive (a code-gen check that the
   suggested subgraph compiles) on the top few. Never run the expensive validator
   across all candidates live.

4. Cache the popular boxes hard and serve the long tail live. "Why is my wifi
   slow" is served warm from cache; the weird query hits the live path. For
   Rare.lab, the common starting graphs and the common next-node suggestions
   should be precomputed and cached (they are asked for constantly), and only a
   creator's unusual, bespoke graph should trigger fresh computation. Your cache
   hit rate on suggestions is your CPU bill and your frame budget, the same way
   Google's PAA cache hit rate is its serving cost.

One line: PAA is a precomputed question graph you walk one lazy hop at a time, with
cheap matching up front and expensive extraction only for the few answers shown,
so copy the shape (offline graph, lazy grow-on-tap, cheap-then-expensive, cache
the common path) for any "suggest the next step" feature you bolt onto a realtime
editor.

---

## Sources

- US Patent US9213748B1, "Generating related questions for search queries," Google
  LLC, inventors Yossi Matias and Ziv Bar-Yossef (the question-graph and
  click-co-selection method): https://patents.google.com/patent/US9213748
  and its continuation US9679027B1: https://patents.google.com/patent/US9679027
- Go Fish Digital, analysis of Google's related-questions and contextual-questions
  patents: https://gofishdigital.com/blog/contextual-questions/
- Search Engine Land, "Could Google passage indexing be leveraging BERT?":
  https://searchengineland.com/could-google-passage-indexing-be-leveraging-bert-342975
- Search Engine Land, "Google People Also Ask boxes not showing as often now?"
  (MozCast share dropping from 93% to 87% of queries):
  https://searchengineland.com/google-people-also-ask-boxes-not-showing-as-often-now-330321
- Search Engine Roundtable, early People Also Ask coverage and launch timing:
  https://seroundtable.com/people-also-ask-google-26104.html
- Ahrefs blog, People Also Ask overview (dynamic expansion, occasionally
  "infinite," most chains stop after a few): https://ahrefs.com/blog/people-also-ask
- Semrush study on People Also Ask (observed appearance and behaviour):
  https://www.semrush.com/blog/people-also-ask/
- Hallme, "Google's Passage Ranking Launched in February" (passages ranked within
  whole pages, not indexed separately):
  https://www.hallme.com/blog/googles-passage-ranking-launched-in-february-heres-what-that-means/
- Dr. Pete Meyers (Moz / MozCast), tracking of People Also Ask growth across the
  SERP data set: https://drpete.co/who-is-dr-pete

Note on sourcing: Google does not publish the live internals of People Also Ask.
The patent, the launch history, the BERT and passage-ranking announcements, and
the MozCast tracking are the solid facts. The inverted-index, sharding,
precomputation, caching and lazy-fetch plumbing are described as the standard way
this class of problem is solved and are labelled as inference in the text, not as
confirmed Google design.
