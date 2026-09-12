# Amazon Sponsored Products: the ad auction behind the "Sponsored" tiles

Date: 2026-09-12
Product: Amazon
Feature: Sponsored Products search-ad auction (the "Sponsored" results at the top of a search)

A note on what is confirmed and what is inferred. Amazon confirms the big
shape of this feature: it is a real-time auction, the winner is not simply
the highest bidder, and the price you pay depends on the runner-up. The U.S.
FTC lawsuit filings in 2024 and 2025 put hard numbers on how often the top
bid loses. Amazon does not publish the exact serving stack for the ad
auction. So the retrieval and ranking internals below are the well grounded
"this is how this class of system is built" version, drawn from Amazon
Science papers on product search and from public sponsored-search
engineering work at Amazon, Alibaba, Taobao, and Etsy. Every inference is
labeled inference.

---

## 1. The user

Two very different people hit this feature in the same tenth of a second.

Meet Ananya. It is a Tuesday night. Her old earbuds died on the metro this
morning. She opens the Amazon app, taps the search box, and types "wireless
earbuds". She wants something under 2,000 rupees, decent battery, and she
wants it tomorrow. She is not thinking about advertising at all. She is
thinking about her commute.

Meet Rajesh. He runs a small brand that sells the boAt-style "Airdopes 141"
earbuds. He is sitting in the Amazon Ads console at his desk. He has set a
bid of 12 rupees for the keyword "wireless earbuds" and a daily budget of
1,500 rupees. He wants his product to show up when someone like Ananya
searches, and he wants to pay as little as possible for each click that
turns into a sale.

The feature exists to serve both of them in one auction that finishes before
Ananya's screen finishes painting.

---

## 2. The real problem

For Ananya, the problem is choice overload. A search for "wireless earbuds"
can match hundreds of thousands of listings. She will look at maybe the top
six before she taps something. If the top six are junk, she leaves and buys
on Flipkart. So the top of the page is the most valuable real estate on the
internet for this query.

For Rajesh, the problem is being invisible. His product is good, but it is
new. It has few reviews. Organic ranking (the unpaid results) rewards
products that already sell well, so a new product is stuck at the bottom of
page four where nobody scrolls. He needs a way to buy his way to the top,
but only when it is worth it, and without overpaying.

For Amazon, the problem is money and trust at the same time. Amazon can sell
the top slots to the highest bidder and make a fortune. But if it shows
Ananya irrelevant junk that happened to bid high, she stops trusting search
and shops elsewhere. Amazon has to sell the slot and keep it relevant. Those
two goals fight each other. The whole design is about resolving that fight.

---

## 3. The feature in one sentence

When you search on Amazon, a real-time auction runs in a few milliseconds
that picks which paid products appear in the "Sponsored" slots, ordered not
by who bid most but by bid multiplied by how likely that product is to be
clicked and bought, and the winner pays just enough to have beaten the
product below it.

---

## 4. Jobs to be done

What Ananya is really hiring this feature to do:
- "Show me earbuds I would actually consider, fast, so I can stop looking."
- "Do not waste my top six slots on things I will never buy."

What Rajesh is really hiring this feature to do:
- "Put my product in front of people who are about to buy this category."
- "Charge me only for clicks, and only what I need to pay to win, not my
  whole maximum bid."
- "Stop spending the moment my daily budget runs out so I do not wake up
  broke."

What Amazon is really hiring this feature to do:
- "Turn the top of every search into a per-query auction that prints revenue
  while keeping results relevant enough that shoppers keep coming back."

Amazon's advertising business made about 68.6 billion dollars in 2025, the
third largest ad business on earth after Google and Meta, and Sponsored
Products is the biggest single piece of it. This one feature is a large
fraction of that number.

---

## 5. How it works for the user

Ananya's side is almost invisible, which is the point. She types "wireless
earbuds". The results page loads. The first two or three products carry a
small grey word, "Sponsored", under the title. They look exactly like normal
results. Same photo, same price, same star rating, same "Get it by tomorrow"
badge. She can tell they are ads only if she reads the fine print. She taps
one. If she buys, great. If she just clicks and leaves, the advertiser still
paid for that click.

Rajesh's side is a dashboard. He creates a campaign, picks his product,
picks keywords (or lets Amazon auto-target), sets a bid and a daily budget,
and turns it on. Later he sees reports: impressions, clicks, spend,
sales, and a number called ACOS (advertising cost of sales, spend divided
by sales). He never sees the auction. He sees the aftermath.

---

## 6. The actual flow, step by step

Ananya's tap-by-tap:
1. She opens the app and taps the search box.
2. She types "wireless earbuds" and hits search.
3. The app sends the query to Amazon's servers with her context: she is
   logged in, she is in Pune, it is 9pm, she has bought phone accessories
   before, she is a Prime member.
4. Two things now happen at once on the server. Organic search finds the
   best unpaid listings. The ad system runs an auction for the sponsored
   slots.
5. The results merge into one page. Sponsored slots sit at fixed positions
   (typically the first row, sometimes a slot mid-page and one near the
   bottom).
6. The page paints. Ananya sees six products. Two say "Sponsored".
7. She taps the boAt Airdopes 141 ad. Rajesh is charged for that click, the
   moment of the tap, not when the page loaded.
8. She buys. Amazon records a conversion and attributes it to Rajesh's
   keyword so his report shows a sale.

Rajesh's setup flow, once:
1. Open Ads console, create a Sponsored Products campaign.
2. Choose the product (the Airdopes 141 listing, one ASIN).
3. Choose targeting: manual keywords like "wireless earbuds", "earbuds under
   2000", or automatic targeting where Amazon picks queries for him.
4. Set a bid, say 12 rupees, and pick a bidding strategy (fixed, or dynamic
   up and down).
5. Set a daily budget, say 1,500 rupees.
6. Launch. From here it is automatic, running in every matching auction until
   the budget for the day is gone.

---

## 7. Under the hood, like the engineer

This is the heart of it. The core trick is the same trick every big search
system uses, and it is the one worth burning into memory: split the work
into matching and ranking, and never run the expensive step on the whole
catalog.

### The catalog you are searching

Amazon carries somewhere around 600 million active product listings (ASINs)
that shoppers can actually buy. The raw listing count behind the scenes was
far larger. Reporting in 2025 described an internal cleanup called "Bend the
Curve" aimed at removing about 24 billion junk or duplicate ASINs, pulling a
projected 74 billion down toward under 50 billion. Of the sellable 600
million, only a slice are advertising on any given query. For "wireless
earbuds" maybe tens of thousands of products have live ad campaigns that
could match. That is still far too many to score one by one in a few
milliseconds, and this request is one of billions per day (Amazon's own ad
engineering job postings describe "billions of requests per day" at
"millisecond" latency).

So the funnel has to shrink the problem hard and fast, in stages, cheap
first and expensive last.

### Stage 0: the inverted index (matching, the cheap half)

An inverted index is a dictionary from a word to the list of things that
contain that word. It is the single most important data structure in search.
Think of the index at the back of a textbook: the word "photosynthesis"
points to pages 44, 91, 210. Here the "word" is a keyword an advertiser
targeted, and the "pages" are the ads targeting it.

For the query "wireless earbuds", the ad server does not scan tens of
thousands of campaigns. It looks up the terms "wireless" and "earbuds" in
the inverted index and instantly gets back the posting lists: every ad that
targeted those terms, including Rajesh's Airdopes 141 ad, plus broad and
phrase-match variants. Cost of this lookup is proportional to how many ads
target those words, not to the size of the whole catalog. That is the whole
magic of an inverted index. A hash map gets you to the posting list in close
to constant time; the posting lists themselves are arrays you can intersect.

### Stage 1: retrieval beyond keywords (matching, the smart half)

Keyword matching alone misses too much. If a shopper types "buds for gym",
a pure keyword index may never surface the Airdopes ad, because Rajesh
targeted "wireless earbuds", not "gym". Modern systems add semantic
retrieval on top of the lexical index.

Inference, well grounded: this is done with embeddings and approximate
nearest neighbor (ANN) search, the same family used by the Spotify and
Instagram teardowns in this ledger. A "two tower" model learns to turn the
query into a vector and every product into a vector, so that a query and a
relevant product land close together in the same space. Crucially, every
product's vector is precomputed offline and stored. At request time the
server only has to (1) turn "buds for gym" into one vector and (2) find the
nearest product vectors with an ANN index (HNSW graphs or IVF/PQ are the
usual choices). ANN over precomputed vectors is what makes this survivable
at scale; brute-force cosine against millions of vectors per request would
blow the latency budget. Public work here includes Amazon's own product
search research on learning to rank and retrieve, and Taobao's
embedding-based product retrieval paper, which reports the tight real-time
constraints (a few milliseconds for the retrieval step). Amazon has not
published the exact ad-side retrieval stack, so treat the two-tower detail
as the standard pattern, not a confirmed Amazon internal.

After Stage 0 and Stage 1, the tens of thousands of possible ads are down to
maybe a few hundred or a couple thousand candidates for "wireless earbuds".

### Stage 2: relevance filtering (the second-pass gate)

Now a heavier model, too expensive to run on tens of thousands but fine on a
few hundred, throws out candidates that are matched but not truly relevant. A
listing for "wireless earbud cleaning putty" might match the keywords but is
not what Ananya wants in a top slot. This gate protects Ananya's trust. The
sponsored-search literature calls this the relevance or quality filter, a
second pass over the retrieved set. Amazon has publicly emphasized a "high
relevance bar" for ads in its own job descriptions for this team.

### Stage 3: predict clicks and conversions (scoring the survivors)

For each surviving candidate, the system predicts two numbers for this
specific shopper and this specific query:
- pCTR: the probability Ananya clicks this ad.
- pCVR: the probability she buys if she clicks.

These come from machine-learned models fed with features: the ad's history,
the shopper's history, price, star rating, image, time of day, whether it is
Prime, and the query itself. Rajesh's Airdopes 141 has a 4.1 star rating,
sells well, and ships next-day to Pune, so its pCTR for Ananya might come out
at 6 percent. A worse listing might come out at 1 percent.

### Stage 4: the auction (ranking, the money half)

Here is where bid meets relevance. Amazon runs a generalized second-price
(GSP) auction. Two facts define it, both confirmed by Amazon and by the FTC
case.

Fact one: the winner is chosen by an "ad rank" score, not by bid alone. The
score is roughly

    ad rank = bid  x  relevance

where relevance bundles pCTR, pCVR, and listing quality. So a product with a
lower bid but much higher relevance can beat a product that bid more. Walk
the real query. Suppose three sellers compete for the top "wireless earbuds"
slot:

    Seller A: bid 20 rupees, pCTR 1%  ->  ad rank score 0.20
    Rajesh:   bid 12 rupees, pCTR 6%  ->  ad rank score 0.72
    Seller C: bid 15 rupees, pCTR 3%  ->  ad rank score 0.45

Rajesh wins the top slot with the lowest of nobody's-highest bid, because
his product is the one shoppers actually click and buy. Amazon makes more
money in the long run by showing the ad that gets clicked, even at a lower
per-click bid, because no click means no revenue at all.

Fact two: the winner pays a second-price, not their own bid. Rajesh does not
pay his 12 rupee maximum. He pays only just enough to have out-ranked Seller
C below him, plus a tiny increment. Working it back through the ad rank
formula: he needed a score above 0.45, and with his 6% pCTR that means a bid
around 7.5 rupees, so he is charged roughly 7.5 rupees plus a cent, not 12.
The gap between his max bid and what he pays is real money saved on every
click.

How real? The FTC lawsuit against Amazon forced out the numbers. In 2024,
about 92 percent of selected Sponsored Products ads were not the highest bid,
and the winning advertiser's bid was on average around the 128th highest bid
by raw amount. Read that twice. The ad that wins the slot is, on average, the
128th highest bidder, because relevance dominates raw bid. That single stat
is the best proof that "highest bid wins" is a myth here.

### Stage 5: budget pacing, the contested-counter problem

This is the part engineers underrate, and it is the closest thing here to
the atomic-stock-decrement problem from the Zepto and Amazon Lightning Deals
teardowns.

Rajesh set a daily budget of 1,500 rupees. That budget is a shared counter.
Ananya's auction is one of thousands happening every second, across many
data centers, all of which might charge against Rajesh's budget at the same
time. If each auction naively reads the remaining budget, decides "there is
room", and charges, then a thousand simultaneous auctions can all see "room"
and blow far past 1,500 rupees. That is the classic race condition, the same
one that oversells the last item in a flash sale.

How this class of system survives it (inference, standard practice):
- Do not do a global lock per auction. That would serialize billions of
  requests and kill latency. Locks are the wrong tool at this scale.
- Instead, split the budget into slices and hand a slice to each serving
  shard, so most decrements are local and cheap. A shard spends its slice,
  then asks for more. This is token-bucket style pacing.
- Accept small, bounded overspend, then reconcile. Amazon's public budget
  rules even formalize a controlled overshoot: daily spend is averaged over
  the month, and the system may spend up to 25 percent over the daily cap on
  a high-intent day and pull back on quiet days, so the monthly total still
  holds. That is pacing turned into a product feature, not a bug.
- Dynamic bidding rides on top. With "dynamic bids, up and down", Amazon
  raises Rajesh's effective bid by up to 100 percent for a top-of-search slot
  when the shopper looks likely to convert, and lowers it when they do not.
  So the 12 rupee bid is really a range the system moves in real time inside
  each auction.

When the counter hits zero, Rajesh's ad simply stops entering auctions for
the rest of the day. Ananya, searching at 11pm, never sees it. Rajesh sees
"budget exhausted" in his console the next morning.

### The scale story at three tiers

Tier one, 1,000 candidate ads for a query. You do not need any of this. Loop
over all 1,000, compute bid times relevance, sort, done. A single server
handles it in microseconds. An inverted index is a nicety, not a need.

Tier two, 100,000 candidate ads and thousands of queries per second. Now the
brute-force loop hurts. You need the inverted index so you touch only the ads
that match the query terms, not all 100,000. You need to cache the pCTR
models' features. You precompute what you can. Sorting still happens
server-side inside the ad service, never on the phone; the phone only draws
the final six results. Read replicas serve the index so reads do not fight
writes.

Tier three, tens of millions of eligible ads and billions of requests per
day, which is Amazon's real world. Now single-word keyword matching is not
enough and single-machine anything is not enough. You add embedding
retrieval with ANN so semantic matches survive. You shard the index (by term,
by category) across many machines and scatter-gather the candidates. You run
the funnel in strict stages so the expensive deep models only ever see a few
hundred survivors, never the millions. You precompute every product
embedding offline and refresh it in batch, so the live path is a lookup plus
a vector search, not a model run over the catalog. And the budget counters
become distributed, sliced, and eventually-consistent with reconciliation,
because a global lock at billions of requests per day is impossible. What
breaks at each tier is always the same thing: doing expensive work on too
many items. What saves you is always the same shape: match cheap to shrink
the set, then rank expensive on the survivors, and precompute anything that
does not depend on the live request.

---

## 8. The retention and habit mechanic

The clever part is that the habit loop is aimed at the advertiser, not the
shopper, and it is nearly self-sustaining.

Rajesh checks his Ads console the next morning. He sees his campaign ran out
of budget by 2pm. Right next to that, Amazon shows him an estimate: "you
missed X impressions" or "increase budget to capture more sales". This is the
core nudge. A seller who is making money is told, in numbers, exactly how
much more money is sitting on the table behind a budget cap he set himself.
The rational move is to raise the budget. He does. The next day the same
thing happens at a higher spend. The loop ratchets.

Layer on suggested bids ("the going rate for this keyword is 14 to 22
rupees"), automatic targeting that discovers new keywords for him, and newer
auto-bidding that optimizes to a target ROAS. Each feature lowers the effort
to spend more. The metric this moves is revenue, directly and mostly
Amazon's. It also moves the seller's own sales, which is what keeps the
seller coming back rather than churning. The genius is that Amazon does not
have to convince the seller to spend. It just has to show the seller,
honestly, the sales he is leaving on the table, and let the seller talk
himself into a bigger budget. The FTC number, winners paying near the 128th
bid, is even part of the retention story: sellers keep advertising partly
because the second-price mechanic means they usually pay less than they
feared, so the return on ad spend stays tolerable.

For the shopper, the retention mechanic is quieter and defensive. Sponsored
slots have to stay relevant (that is the whole point of the relevance filter
and the ad-rank formula) or shoppers learn to distrust the top of the page
and scroll past it. Amazon guards shopper retention by refusing to let raw
bid override relevance, which is exactly why the top bid loses 92 percent of
the time.

---

## 9. The lesson for Rare.lab

The reusable idea is the funnel: never run your most expensive computation on
your largest set. Amazon takes tens of millions of ads down to a handful with
cheap matching first (inverted index, then ANN over precomputed vectors), and
only runs the heavy click-and-convert models on the few hundred survivors,
then the auction on maybe a dozen. Cheap-first, expensive-last, and
precompute anything that does not depend on the live input.

Map that straight onto Rare.lab. In the node-based editor and the embeddable
runtime you will face the same shape: a graph of many nodes, most of which do
not need expensive work on any given frame.

- Build the cull before the shade. Every frame, cheaply decide which nodes
  actually changed and which pixels or objects the effect can even touch
  (a dirty-node check, a bounding volume, a frustum or scissor test, a
  level-of-detail pick). Only compile or evaluate the survivors. This is the
  inverted-index-then-rank move in graphics clothing: shrink the set with a
  near-free test, then spend the GPU only on what is left.
- Precompute anything that does not depend on this frame's live input. Amazon
  precomputes product embeddings offline so the live path is just a lookup.
  You should precompile and cache shader permutations, bake constant
  subgraphs, and hoist anything time-invariant out of the per-frame path, so
  a running effect is mostly a cached lookup plus the small changed part, not
  a recompile.
- Steal the budget-pacing pattern for the runtime. Give each frame a compute
  budget the way Rajesh has a daily budget. Meter it with a cheap local
  counter, not a global lock. When a frame is about to overspend, degrade
  gracefully (drop to a cheaper LOD, skip a non-critical pass, lower sample
  count) instead of blowing the frame time and stuttering. Amazon accepts a
  small bounded overspend and reconciles later; a real-time runtime should
  accept a small quality dip and recover next frame, never a dropped frame.

One concrete, specific rule to carry out of today: put a hard staged budget
in the compiled runtime. Stage 1 is a near-free visibility and dirty cull,
Stage 2 is cached-permutation lookup, Stage 3 is full shader evaluation, and
each stage has a fixed millisecond ceiling. If a stage would exceed its
ceiling, it sheds the lowest-value work (the equivalent of the lowest ad
rank) rather than running long. That is how Amazon keeps a billions-per-day
auction inside a few milliseconds, and it is how an embeddable VFX runtime
holds 60 frames per second on hardware you do not control.

---

## Sources

Auction mechanics and the second-price / ad-rank facts:
- How Amazon's ad auction actually works (not highest bid wins): https://rel.ai/blog/how-amazons-ad-auction-works
- Cornell INFO 2040, the complexities of advertising bidding on Amazon: https://blogs.cornell.edu/info2040/2022/09/17/the-complexities-of-advertising-bidding-on-amazon/
- How the Amazon PPC auction works: https://www.aihello.com/resources/blog/how-does-the-amazon-ppc-auction-work/

The FTC lawsuit numbers (92 percent not highest bid, ~128th bid):
- Karooya, FTC v. Amazon sponsored ads auction-pricing: https://www.karooya.com/blog/ftc-v-amazon-what-the-sponsored-ads-auction-pricing-lawsuit-means-for-advertisers/
- Amazon's own response to the FTC sponsored-ads lawsuit: https://www.aboutamazon.com/company-news/amazon-ftc-sponsored-ads-lawsuit-response

Retrieval and ranking pipeline (matching vs ranking, multi-stage funnel):
- Amazon Science, from structured search to learning to rank and retrieve: https://www.amazon.science/blog/from-structured-search-to-learning-to-rank-and-retrieve
- Amazon, extreme multi-label learning for semantic matching in product search: https://arxiv.org/pdf/2106.12657
- SIGIR eCom 2025, model-based performance filtering for ad quality: https://sigir-ecom.github.io/eCom25Papers/paper_5.pdf
- PCDF, a parallel distributed framework for sponsored search serving: https://arxiv.org/pdf/2206.12893
- Alibaba, constrained optimization of auction mechanisms in sponsored search: https://arxiv.org/pdf/1807.11790

Two-tower and ANN retrieval (the standard pattern, labeled inference above):
- Taobao, embedding-based product retrieval in search: https://arxiv.org/pdf/2106.09297
- Google Cloud, scaling deep retrieval with two-tower architecture: https://cloud.google.com/blog/products/ai-machine-learning/scaling-deep-retrieval-tensorflow-two-towers-architecture

Dynamic bidding and budget rules:
- Amazon Ads, guide to dynamic bidding up and down: https://advertising.amazon.com/library/guides/dynamic-bidding-sponsored-products
- Sellermetrics, Amazon's 2025 budget rules update: https://sellermetrics.app/amazon-2025-budget-rules-update/
- Feedvisor, default, suggested and dynamic bids: https://feedvisor.com/university/amazon-sponsored-products-default-suggested-and-maximum-bids/

Scale (catalog size, revenue, request volume):
- Redstag, how many products Amazon carries: https://redstagfulfillment.com/how-many-products-does-amazon-carry/
- Slashdot, Amazon purges billions of listings ("Bend the Curve"): https://slashdot.org/story/25/05/30/1954240/amazon-purges-billions-of-product-listings-in-cost-cutting-drive
- Marketing Dive, Amazon annual ad revenue passes 68 billion: https://www.marketingdive.com/news/amazon-annual-ad-revenue-passes-68b-boosted-by-full-funnel-strategy/811569/
- eMarketer, Amazon retail media ad revenue past 60 billion in 2025: https://www.emarketer.com/content/amazon-retail-media-ad-revenues-will-pass-60-billion-2025
- Amazon Jobs, ML Engineer II, Sponsored Products Search Sourcing (billions of requests per day, millisecond latency): https://amazon.jobs/en/jobs/3061807/machine-learning-engineer-ii-sponsored-products-search-sourcing-amazon-advertising
