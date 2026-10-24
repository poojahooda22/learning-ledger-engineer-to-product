# Amazon: customer reviews, the star rating and the "Top reviews" order

Date: 2026-10-24
Product: Amazon
Feature: Customer reviews. Two linked things: the single "4.5 out of 5" star number near the top of a product page, and the order of the written reviews below it (the "Top reviews" list, the ones tagged "1,204 people found this helpful").

A quick note on what this is NOT, so it does not blur into earlier Amazon teardowns.
This is not how the search results page orders products (product search ranking, 2026-06-23).
This is not which seller wins the one-tap default (Buy Box / Featured Offer, 2026-08-12).
This is not "Customers who bought this also bought" (item-to-item collaborative filtering, 2026-07-12).
It is the review block: how one trustworthy star number gets made out of thousands of
opinions, and how the handful of reviews you actually read get chosen out of a pile most
people never scroll to.

This one is worth a teardown because it looks like a school problem (just average the
stars) and is not. A plain average quietly lies, and the fix is a small, beautiful piece
of statistics that most products get wrong. It is also the same spine we keep seeing:
do the expensive thinking offline and on write, serve a cheap cached number on read.

---

## 1. The user, and their day

Meet Meera. It is a Sunday afternoon in Indore. Her three-year-old keeps pulling the HDMI
cable out of the back of the TV, and she has decided she wants a streaming stick instead so
the whole mess lives behind the panel. She is on her phone, on the sofa, one eye on the
child. She searches "fire tv stick 4k" and taps the first real product.

She is not going to read a spec sheet. She does not know or care what codec it decodes. She
has about forty seconds of attention and one real question in her head: "is this thing
actually good, or am I about to waste 3,000 rupees and a Sunday." The first thing her eye
lands on is a row of orange stars and a number: "4.5 out of 5." Below that, in small grey
text, a count: a few hundred thousand ratings. Then a scroll of written reviews, the top one
with a photo of the stick plugged in behind someone's TV and a line underneath: "1,204 people
found this helpful."

That star number and those first three reviews are going to decide whether she buys. She will
never see the other three hundred thousand reviews. The product's whole job, in her forty
seconds, is to compress all of them into one honest number and three honest paragraphs.

---

## 2. The real problem

Here is the pain, said plainly, like a friend would say it.

You cannot try the thing before you buy it. You are trusting strangers. But strangers lie,
strangers are paid, strangers rate one star because the courier was late (nothing to do with
the product), and strangers rate five stars because they are the seller's cousin. And there
are too many of them to read. So you need someone to do three hard things for you, instantly,
for free:

1. Turn a crowd of opinions into one number you can trust in a glance.
2. Put the few genuinely useful reviews at the top, not the newest and not the loudest.
3. Keep the liars and the paid reviews out, or the first two jobs are worthless.

The trap is that the obvious answer to job one is wrong. "Just add up the stars and divide."
That is the plain average, and it fails in two ways that matter to Meera. A new gadget with
three ratings, all five star (two of them from the seller), shows a perfect 5.0 and sits above
the Fire TV Stick's hard-earned 4.5 from three hundred thousand people. And a one-star review
that only says "delivery was late, box was dented" drags the number down even though it says
nothing about whether the stick is any good. The plain average treats three votes like three
hundred thousand, and treats a complaint about the courier like a complaint about the product.

---

## 3. The feature in one sentence

Amazon reviews take a pile of ratings and written opinions about one product and compress them
into a single trustworthy star number plus a short ordered list of the most useful reviews,
with likely-fake ones filtered out, computed ahead of time so the product page can show it in a
blink.

---

## 4. Jobs to be done

What is Meera really hiring this feature to do?

- "Tell me in one glance whether this is good, and do not let a handful of fake five-stars fool
  me." (The trustworthy aggregate.)
- "Show me the three reviews actually worth reading, the ones that answer my real worry (does
  it keep disconnecting? is the remote any good?), not the newest or the angriest." (Useful
  ordering.)
- "Do not count the paid or fake reviews, because if you do I cannot trust any of it." (Fraud
  filtering.)
- "Do it instantly. I have forty seconds and a toddler."

And the reviewer, the other user, is hiring it to do something too: "when I write a careful
review, let it matter, let people see it and tell me it helped." That second user is what keeps
the pile growing. Hold that thought for the retention section.

---

## 5. How it works for the user (the visible experience)

Meera sees, from top to bottom:

- The stars and the number: "4.5 out of 5," orange stars, and a tappable count like
  "3,40,000 ratings."
- A little histogram: 5 star 71%, 4 star 15%, 3 star 6%, 2 star 3%, 1 star 5%. She can tap "1
  star" to read the worst ones (smart shoppers do this first).
- A line of auto-pulled phrases Amazon extracted from the text: "easy setup," "good value,"
  "remote quality." Tap one and the reviews filter to that topic.
- The reviews themselves, under a dropdown that says "Top reviews" by default. She can switch it
  to "Most recent." The default is the important one.
- Each review has stars, a title, "Verified Purchase" in orange if the person actually bought
  it here, the date, a photo sometimes, and at the bottom "1,204 people found this helpful" with
  a "Helpful" button and a "Report" link.

The two decisions the system is quietly making for her: what number to show at the top, and
which reviews count as "Top." Both were decided before she ever opened the page.

---

## 6. The actual flow, step by step

Tap by tap, for Meera the reader:

1. She taps the product. The page loads. The "4.5" and the ratings count are already there, in
   the first paint. They were not calculated when she tapped. They were read, as one small
   stored value, the same way the price is read.
2. She scrolls to reviews. The first ten "Top reviews" come down, already in order. Her phone did
   not sort anything. It received a list that was already sorted, on a server, and cached.
3. She taps "Helpful" on the review with the photo. That is a write. It does not re-sort the page
   in front of her. It drops a tiny event into a queue ("+1 helpful on review X") and moves on.
   The new order shows up later, for the next shopper, not as a live reshuffle for her.
4. She buys the stick.

And the other flow, days later, for Meera the writer:

5. The stick arrives. Two days after delivery, Amazon sends her a gentle nudge: "How was your Fire
   TV Stick 4K? Leave a review." She taps it, gives four stars, writes two lines, uploads the same
   kind of photo.
6. Her review does not appear on the page instantly. It goes through a screening step first (job
   three, the fraud filter). If it passes, it is published, and it quietly nudges the product's
   star number and may enter the "Top reviews" race.

The whole visible experience hangs on one architectural choice you cannot see: writing a review or
a helpful vote is cheap and asynchronous, reading the aggregate is cheap and instant, and all the
expensive work (scoring, sorting, fraud checks) happens in between, off to the side, not while
anyone is waiting.

---

## 7. Under the hood, like the engineer

This is the heart of it. Three problems, each with its own data structures and its own scale wall.

### Half one: the trustworthy star number (aggregation)

**The naive version and why it dies.** Store every rating as a row: `(product_id, user_id, stars,
timestamp)`. To show the star number, run `SELECT AVG(stars) WHERE product_id = X`. For the Fire TV
Stick that is an average over a few hundred thousand rows, run every single time anyone opens the
page, which is millions of times a day. That is a huge amount of repeated arithmetic to produce a
number that barely changes between two page views. It also gives you the plain average, which as we
said lies about small samples and about off-topic one-stars.

**What Amazon actually changed (CONFIRMED, 2015).** In June 2015 Amazon publicly moved off the plain
unweighted average. The company said the overall star rating would now be a machine-learned number
that weights a review by how recent it is, how many people found it helpful, and whether it is from a
verified purchaser. The same learned factors also help decide where a review sits in the list. Amazon
did not publish the formula or the weights, and said the model keeps learning over time. It rolled out
in the United States first. (GeekWire, The Register, InformationWeek, Retail Dive, all June 2015.)

Read that back in plain words. The one-star "delivery was late" review from a person who did not buy it
here, written three years ago, now counts for much less than a recent, helpful, verified-purchase
review. That is the fix for the "off-topic one-star drags the number down" problem. The number at the
top of Meera's page is a weighted, learned estimate, not a mean.

**The small-sample problem and the beautiful fix (CONFIRMED math, labeled as the standard solution).**
There is a second, separate lie in the plain average: a 5.0 from three ratings should not outrank a 4.6
from two thousand. The classic, published way to handle this is a Bayesian weighted rating, and the
cleanest public example is the IMDb Top 250. IMDb documents its weighted rating as:

```
WR = ( v / (v + m) ) * R  +  ( m / (v + m) ) * C
```

where `R` is this title's own mean rating, `v` is how many votes it has, `m` is a minimum vote count to
qualify, and `C` is the mean rating across all titles. When `v` is tiny, the term leans almost entirely
on `C`, the global average, so a movie with three votes is dragged back toward the middle of the pack.
As `v` grows, the number drifts toward the title's real mean `R`. (IMDb help pages and Wikipedia; IMDb
says the current constants and the definition of a "regular voter" are secret.)

Concretely: give that brand-new gadget its three five-star ratings and a global average `C` of about 4.0,
and its weighted score sits near 4.1, not 5.0. The Fire TV Stick with three hundred thousand ratings sits
almost exactly at its true 4.5. The shrinkage only touches the titles that have not earned their number
yet. Amazon has not published whether its learned model uses exactly this Bayesian form (INFERENCE,
clearly labeled), but every serious ratings system solves the small-sample lie this way, and Amazon's
stated goal (do not let a few votes dominate) is the same goal this math exists to serve.

**The data structure that makes the read instant (INFERENCE, labeled, but this is how this class of
problem is always solved).** You do not keep recomputing. You keep a small per-product summary record and
update it on write:

```
product_rating_summary[product_id] = {
  count:            340127,
  sum_of_stars:     1518000,
  histogram:        [17006, 10204, 20407, 51019, 241491],  // 1..5 star buckets
  displayed_rating: 4.5,                                     // the learned/weighted number
  updated_at:       ...
}
```

This is a hash map keyed by product id. Reading Meera's star number is one keyed lookup, O(1), no matter
how many ratings the product has. When a new rating lands, a worker does `count++`, `sum += stars`, bumps
the right histogram bucket, recomputes the displayed number, and writes the summary back. The page read
never touches the three hundred thousand raw rows. This is the offline-think / online-lookup spine again,
the same one behind Discover Weekly (2026-06-13), the Buy Box hash map (2026-08-12), and Netflix downloads
(2026-10-08): compute the expensive thing away from the hot path, serve a cached scalar on the hot path.

### Half two: the "Top reviews" order (ranking by helpfulness)

Now the second visible thing: which reviews sit at the top. The wrong answers are "newest first" (then a
useless "arrived on time!" five-star buries the one careful review) and "most helpful votes first." That
second one sounds right and is a classic trap.

**Why most-helpful-votes-first is wrong (CONFIRMED argument, Evan Miller).** Sorting by raw helpful votes,
or by helpful minus unhelpful, lets volume beat quality. Evan Miller's "How Not To Sort By Average Rating"
walks the exact failure: an item with 600 positive and 400 negative votes (60% positive) loses to an item
with 5,500 positive and 4,500 negative (only 55% positive) if you sort by net score, because 1,000 beats
200. And sorting by the raw positive fraction lets a review with 2 of 2 helpful votes (100%) leap over one
with 1,900 of 2,000 (95%). You need to balance the fraction of "helpful" votes against how sure you are of
that fraction given how few votes it has.

**The fix (CONFIRMED math).** Use the lower bound of the Wilson score confidence interval as the sort key.
The math was worked out by Edwin B. Wilson in 1927. For a review with `n` total helpful/unhelpful votes and
an observed helpful fraction `p`, the lower bound at confidence level `z` (z = 1.96 for 95%) is:

```
( p + z^2/(2n)  -  z * sqrt( ( p*(1 - p) + z^2/(4n) ) / n ) )  /  ( 1 + z^2/n )
```

It answers "given these votes, how low could the true helpful fraction plausibly be?" A review with 2 of 2
helpful has a high fraction but huge uncertainty, so its lower bound is modest. A review with 1,900 of 2,000
has almost no uncertainty, so its lower bound stays near 0.95 and it wins. The Instacart `wilson_score` gem
adds a continuity correction that helps when `n` is under 30. (Evan Miller, evanmiller.org; Instacart gem.)

So the review Meera reads first, "1,204 people found this helpful," is not first because 1,204 is a big
number. It is first because 1,204 helpful out of (say) 1,260 total votes gives a very high, very confident
lower bound. A newer review with 3 of 3 helpful cannot beat it, because three votes are not enough to be sure.

**Where Amazon goes past Wilson (CONFIRMED, 2015).** Amazon's 2015 change said the same learned factors that
set the star number also order the list: recency, verified purchase, and helpfulness all feed in. So "Top
reviews" is not pure Wilson. It is a learned ranking where a confident helpfulness signal (Wilson-style) is
one strong feature among several, blended with recency (a two-year-old review about an older firmware matters
less) and trust (verified purchase outranks a drive-by). The exact blend is not public (INFERENCE on the
blend; CONFIRMED that those three factors are in it).

**The data structure (INFERENCE, labeled).** Reviews for one product are an array keyed by product id. Each
helpful vote is a counter on a review row. The "Top reviews" order for each product is precomputed and cached
as a short ordered list of review ids (you only ever show the first page), re-scored when votes or new reviews
accumulate, not on every page view. The sort happens server-side, in a batch or a streaming job, and lands in
a cache. The phone receives a ready list. Nothing is sorted on the device, the same as search results are
sorted in the database and not on the phone (2026-06-23).

### Half three: keeping the liars out (fraud filtering)

If jobs one and two run on poisoned data, they are worse than useless, because now they lend false
confidence. So there is a third pipeline, and it runs constantly.

**What Amazon says (CONFIRMED, 2022 to 2023).** Amazon runs machine learning on all incoming and existing
reviews 24 hours a day, and screens reviews before they are published. It reported blocking more than 200
million suspected fake reviews in 2022 alone, and says more than 90% of inauthentic reviews it finds are
computer generated. It combines automated models with human investigator teams. Signals it names include odd
reviewing patterns, review history, whether a seller paid for ads, and customer reports. (aboutamazon.in,
Amazon's own posts; reported by Storyboard18 and others.)

**The graph move (CONFIRMED for the academic version, SECONDARY/cautious for Amazon's own).** The strongest
public method treats this as a graph, not as single-review text scoring. Build a bipartite graph: products on
one side, reviewers on the other, an edge for each review. Fake-review-buying products turn out to be highly
clustered in that graph, because the same rings of paid accounts review the same suspicious products. A 2022
working paper (He, Hollenbeck, Overgoor, Proserpio, Tosyali, published via MSI) uses unsupervised clustering on
exactly this product-reviewer network to find fake-review buyers, trained on real observed fake-review
purchases rather than lab fakes. A secondary news report (Storyboard18) says Amazon itself uses deep graph
neural networks to find groups of bad actors; treat that as plausible but not an Amazon publication. Either way
the shape is the same: catch the ring, not the single review, because a single paid five-star is hard to tell
from a happy customer, but a cluster of accounts that only ever review the same ten obscure products together
lights up like a flare.

When this pipeline suppresses a batch of reviews, that is itself a write event: the product's summary record
(half one) and its top-reviews list (half two) get recomputed without them. Fraud filtering is upstream of the
other two.

### The scale story at three tiers

The thing that grows is not one number. It is three: reviews per product, products in the catalog, and the
stream of write events (new reviews, helpful votes, fraud suppressions) per second.

**1,000 reviews on one product** (a mid-list paperback with a thousand ratings). You can be naive and survive.
Store one row per review. Compute the star number with a live `AVG` query, sort the reviews by a Wilson score
computed on the fly, all in memory, in a few milliseconds. No caching, no queue, no separate fraud pipeline
beyond simple rules. Building the full machine here would be over-engineering. Ship the simple version. The one
thing you should still do even at this size is the Bayesian shrinkage, because the 5.0-from-3 lie shows up at
any scale and costs almost nothing to fix.

**100,000 reviews on one product** (a bestselling phone case, viewed millions of times a day). The live `AVG`
and the live re-sort now break, not because they are hard but because you are doing them millions of times to
produce a number that changed by 0.001. This is the tier that forces offline-think / online-lookup. You switch
to a stored per-product summary updated on write (the hash map above), so reads become O(1). You precompute the
"Top reviews" order and cache it, refreshing it on a schedule or when enough new votes pile up, not per page
view. You paginate hard: the phone gets 10 reviews, not 100,000, and "page 2" is another server-side slice. This
is exactly where "the sorting happens on the server, not the phone" stops being a style choice and becomes
survival.

**10 million plus** (the real Amazon: a catalog of hundreds of millions of products, some single products
carrying several hundred thousand ratings, billions of reviews in total, and a constant storm of new reviews and
helpful-vote taps). Four walls, and the survival moves for each:

- You cannot compute anything synchronously on the write path either. A helpful tap cannot re-sort a 300,000
  review list inline. So writes become events on a queue (Meera's "Helpful" tap is one tiny message), and async
  workers fold them into the cached summary and the cached top-reviews list. (Same queue-the-write shape as the
  Amazon cart's Dynamo writes, 2026-07-28, and Stripe webhooks, 2026-07-31.)
- One machine cannot hold the reviews. Shard the reviews by product id, so all of the Fire TV Stick's reviews
  (and only those) live together and its recompute is a local job on one shard, never a scan across the catalog.
- The read path is the hottest path on the site. Serve the star number and the top-ten from read replicas and
  edge caches, so a hundred million shoppers reading "4.5" never touch the primary store. The displayed rating is
  a denormalized cached scalar sitting right next to the price.
- Fraud cannot be a per-review check at this volume; 90%+ of fakes are machine-generated and arrive in waves. So it
  is its own near-real-time pipeline running 24/7 over the product-reviewer graph, sharded, flagging rings and
  triggering recomputes. Amazon blocking 200 million fakes in one year is this pipeline's output.

What breaks if you skip a tier's move: skip the stored summary and the read path melts under repeated `AVG`
queries; skip sharding and no single box can hold or recompute a hot product; skip the async queue and a viral
product's helpful votes back up the whole write path; skip the graph fraud pass and the star number becomes a
number nobody trusts, which destroys the entire feature no matter how fast it is.

---

## 8. The retention and habit mechanic

Reviews run two loops at once, one for each user.

**The reader loop moves revenue.** The star number is the single biggest convert-or-leave moment on the page.
Meera's forty-second decision swings on "4.5 from hundreds of thousands" versus "4.1 from twelve." A trustworthy
aggregate turns a browser into a buyer, and it does it on nearly every product page on the site, so the lift is
enormous even if each page moves conversion only a little. The habit is quiet: shoppers learn that Amazon's star
number is usually honest (the histogram is there, the one-stars are readable, the fakes are mostly gone), so they
stop checking other sites and come back to Amazon to decide. Trust is the retention mechanic here, the same quiet
invisible-craft loop as Netflix picture quality (2026-09-11) or Zepto showing only in-stock items (2026-10-18):
you only notice it when it breaks, and when it breaks you stop trusting the whole product.

**The writer loop feeds the moat.** The reviews only exist because people write them for free, and people write
them because the system rewards contribution. Two days after Meera's stick arrives, she gets a nudge: "review your
recent purchase." When she writes one, the "X people found this helpful" counter is a tiny social reward that pulls
her back to see if her review helped anyone. Amazon has leaned hard on this loop for years: the post-delivery review
request, the old Top Reviewer rankings and Hall of Fame, and the Vine program that gives trusted reviewers free
products in exchange for honest reviews. Which metric does this loop move? Not revenue directly. It is a
contribution-and-engagement loop whose output is the content that powers the revenue loop. No writers, no reviews,
no trustworthy stars, no conversion lift. The feature quietly manufactures its own fuel.

The real observed example that ties both loops together: a shopper who reads the "1 star" bucket first (a very
common power-shopper move) is trusting that the one-stars are real complaints about the product and not courier
rage or competitor sabotage. The 2015 move to down-weight off-topic and unverified reviews, and the fraud pipeline
that strips the rings, are what make reading the one-star bucket useful instead of noise. That usefulness is the
retention.

---

## 9. The lesson for Rare.lab

Rare.lab will have a gallery: community shaders, node presets, visual-effect building blocks, each with a rating and
a "used 2,000 times" count, each shown in an editor that must stay at 60fps. That gallery is this exact feature. Four
concrete lessons, biased toward scale and performance.

1. **Serve the aggregate as a cached scalar, never recompute on the hot path.** The "4.6 stars, 2,000 uses" on a
   preset tile must be one keyed lookup, computed on write by a queue worker, exactly like Amazon's per-product
   summary record. If opening the gallery triggers an average over every rating, the editor stalls. A rating is a
   denormalized number that sits next to the preset, updated asynchronously when a vote lands, read in O(1) while
   the user scrolls.

2. **Use confidence-aware ranking, or new good-looking presets will dominate and lie.** Rank gallery items by the
   lower bound of a Wilson score (for up/down "this worked for me" votes) or a Bayesian shrinkage estimate (for
   1-to-5 star ratings), never by the raw average or the raw vote count. Otherwise a brand-new node with three
   five-star ratings from the author's friends outranks a battle-tested 4.6-from-2,000 node, and your gallery
   recommends the untested thing. Shrink small samples toward the global mean, the same move that keeps a 5.0-from-3
   gadget off the top of Amazon.

3. **Split the write path from the read path and run abuse detection offline as a graph.** A rating or a "ran great
   on my GPU" vote is a tiny event on a queue; workers fold it into the cached rank later. Fake-popularity rings (a
   cluster of sockpuppet accounts all boosting the same handful of presets) are caught the Amazon way, as clusters
   in a user-preset graph, in an offline pass, not by judging one vote at a time. A poisoned rating is worse than no
   rating, because it lends false confidence to a shader someone is about to ship.

4. **Make measured performance a first-class part of the "rating," aggregated the same confidence-aware way.** This
   is the Rare.lab-specific twist. A community shader's quality is not just "do people like it," it is "does it hold
   frame budget on real devices." Collect the median fps it actually hit across N real runs on real GPUs, and
   aggregate that with the same Wilson/Bayesian confidence weighting. A shader that hit 120fps on two high-end
   machines must not outrank one that holds a steady 60fps across two thousand mid-range phones, for the same
   statistical reason a 5.0-from-3 must not outrank a 4.5-from-300,000. Then a creator dropping an effect into their
   scene sees a rating that already encodes "this will survive on the devices your users actually have," which is
   the single most useful thing a visual-effects gallery could tell them, and it is the same honest-aggregate
   discipline that makes Meera trust Amazon's 4.5.

One line: a star rating looks like "add them up and divide" and is really a confidence-weighted aggregate plus a
confidence-weighted sort plus an offline graph fraud pass, all precomputed on write so the page reads one cached
number in a blink. Build Rare.lab's gallery the same way, and fold measured on-device performance into the same
honest, confidence-aware aggregate.

---

## Sources

- GeekWire, "Amazon changes its influential formula for calculating product ratings" (June 2015): https://www.geekwire.com/2015/amazon-changes-its-influential-formula-for-calculating-product-ratings/
- The Register, "Amazon tweaks its customer reviews with machine learning" (June 21, 2015): https://www.theregister.com/2015/06/21/amazon_customer_reviews_machine_learning/
- InformationWeek, "Amazon Applies Machine Learning To Improve Reviews" (2015): https://www.informationweek.com/software-services/amazon-applies-machine-learning-to-improve-reviews
- Retail Dive, "Amazon installs new machine learning reviews system" (2015): https://www.retaildive.com/news/amazon-installs-new-machine-learning-reviews-system/401086
- Evan Miller, "How Not To Sort By Average Rating" (the Wilson score lower bound): https://evanmiller.org/how-not-to-sort-by-average-rating.html
- Instacart `wilson_score` Ruby gem (continuity correction for small n): https://github.com/instacart/wilson_score
- IMDb weighted-rating / Bayesian estimate (Top 250 formula): https://en.wikipedia.org/wiki/IMDb
- Amazon, "Detecting fake reviews with advanced AI" (aboutamazon.in): https://www.aboutamazon.in/news/retail/detecting-fake-reviews-with-advanced-ai
- He, Hollenbeck, Overgoor, Proserpio, Tosyali, "Detecting Fake-Review Buyers Using Network Structure" (MSI working paper, 2022): https://www.msi.org/working-paper/detecting-fake-review-buyers-using-network-structure-direct-evidence-from-amazon/
- Storyboard18 (secondary), "How Amazon is using AI to combat fake reviews": https://www.storyboard18.com/quantum-brief/how-amazon-is-using-ai-to-to-combat-fake-reviews-17812.htm
