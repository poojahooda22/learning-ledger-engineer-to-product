# Amazon Buy Box (the Featured Offer): how one box picks which seller you buy from

Date: 2026-08-12
Product: Amazon
Feature: The Buy Box, now officially called the Featured Offer. The single box on a product page with the price, the "Add to Cart" button, and the "Buy Now" button, and the quiet algorithm that decides which of many sellers gets to sit inside it.

A note on sourcing before we start. Amazon has never published the Featured Offer algorithm. The exact scoring is a trade secret. So this teardown separates two things carefully. The facts come from real sources: Amazon Seller Central documentation, a 2021 investigation by The Markup that audited 3,492 popular products, a Northeastern University study of price changes, a Profitero price-tracking report, and the public patent US8630923. The internals of the scoring and the scale story are labeled inference, built from how this exact class of problem is solved in practice. Every inference is marked.

---

## 1. The user

Meet Ananya. It is a Tuesday night. Her wireless mouse just died mid-email, the little Logitech M185 she has used for three years. She opens the Amazon app, types "logitech m185", and taps the first result.

She lands on the product page. There is a photo, a price of 649 rupees, a delivery promise of "Get it by Thursday", and a big yellow "Add to Cart" button with an orange "Buy Now" under it. She taps Buy Now. Done. Thirty seconds, start to finish.

What Ananya never saw: eleven different sellers were offering that exact same mouse at that exact moment. Cloudtail at 649, another seller at 641 but with slower shipping, one at 629 but shipping from a warehouse three states away, one out of stock, one with a poor rating. Amazon picked one of the eleven and put only that one behind her Buy Now button. She bought from a seller whose name she never read.

That box is the Featured Offer. Ananya is the user, and her whole job that night was: stop the annoyance, get a working mouse, move on. She did not want a spreadsheet of eleven sellers.

---

## 2. The real problem

Here is the pain, told plainly.

The same product on Amazon is often sold by many different people. A pack of Duracell AA batteries might have twenty sellers behind it. They are selling the identical item, just from different warehouses at slightly different prices with slightly different delivery speeds.

If Amazon showed Ananya all twenty and said "pick a seller", she would freeze. Which one is trustworthy? Is the 629 one a scam? Why is the 641 one slower? That is twenty tiny decisions to buy one boring pack of batteries. Most people would give up, or bounce to another tab, or just close the app. A confused shopper is a lost sale.

The other side of the pain is the seller's. Twenty sellers cannot all get the sale. Somebody has to be the default. Whoever is the default gets almost all the orders. Whoever is not is nearly invisible. So the rule that picks the default is the single most important rule on the whole platform for a seller's livelihood.

Amazon needs one answer to a hard question, computed for hundreds of millions of products, refreshed constantly as prices and stock change: for this product, right now, which single seller should be the default?

---

## 3. The feature in one sentence

The Featured Offer is the algorithm that looks at every eligible seller of one product and picks one to sit behind the Buy Now button, so the shopper buys with one tap and never has to compare sellers.

---

## 4. Jobs to be done

What is Ananya really hiring this box to do?

- "Choose a trustworthy seller for me so I do not have to." She is outsourcing the vetting.
- "Give me a fair price without making me shop around." She trusts that the default is not a rip-off.
- "Let me buy in one tap." Speed is the point.

What is the seller hiring the box to do? "Route the order flow to me." Winning the box is the seller's entire growth lever.

What is Amazon hiring the box to do? "Turn a crowd of competing sellers into one clean, fast, cheap default, so the shopper converts and the sellers keep fighting each other on price and speed." The box is Amazon's referee, and the prize it hands out keeps the whole marketplace competing.

---

## 5. How it works for the user

From Ananya's seat, there is almost nothing to see, and that is the design.

She sees one price. One delivery date. One "sold by" line she probably ignores. One Buy Now button. The eleven-way contest happened before the page finished loading. She experiences a simple product, not a marketplace.

If she is curious and scrolls down, she finds a small link that says something like "Other sellers on Amazon" or "See All Buying Options". Tapping it reveals the full list: all eleven offers, their prices, their delivery dates, their seller ratings. Almost nobody taps it. Industry estimates put roughly 82 percent of Amazon sales through the Featured Offer itself, not through that "other sellers" list (this 82 percent figure is a widely cited industry estimate, not a number Amazon has confirmed).

So the visible experience is: the winner is loud, everyone else is one quiet link away.

---

## 6. The actual flow, step by step

Tap by tap, the night Ananya bought the mouse:

1. She searches "logitech m185" and taps the first result.
2. The product page loads. Behind the scenes, Amazon has already decided the Featured Offer for this product. The page just reads that decision and renders it.
3. She sees 649 rupees, "Get it by Thursday", "Sold by Cloudtail India", and the buttons.
4. She taps "Buy Now".
5. Because her address and card are already saved, the order is placed against Cloudtail's offer. (This is where the Featured Offer hands off to the one-tap ordering machine covered in the 2026-07-06 teardown.)
6. She gets a confirmation. She never saw the other ten sellers.

Now the same page from a seller's seat. Imagine "TechBazaar", one of the eleven. TechBazaar runs a repricing bot. Every few minutes the bot checks the current Featured Offer price and lowers TechBazaar's price by one rupee to try to win the box. The moment the bot changes the price, Amazon re-evaluates the offer for that product. If TechBazaar now wins, the very next shopper who loads the page buys from TechBazaar instead of Cloudtail. No human touched anything. The box changed hands in the time between two page loads.

---

## 7. Under the hood, like the engineer

This is the heart of the report. The Featured Offer is the same two-half pattern the ledger keeps finding, matching then ranking, but with the sizes flipped from a search engine.

### Matching and ranking, with the sizes inverted

In Amazon product search (teardown 2026-06-23) the candidate set is enormous. One word like "watch" pulls thousands of products out of a catalog of 350 million. The whole game there is fetching candidates cheaply.

The Featured Offer is the opposite. The candidate set is tiny. For one product, the offers are a handful to maybe a few hundred sellers, not millions. Fetching them is trivial. The hard part is somewhere else: there are 350 million-plus products, each with its own little contest, and the inputs to every contest (price, stock, seller health) change constantly. The difficulty is not the size of one contest. It is the number of contests times how often each one must be recomputed.

So the two halves are:

Half one, matching, is eligibility. It is a filter. Take all offers for this product and throw out the ones that cannot compete. This part is public. From Amazon Seller Central and seller documentation, the eligibility gate has required:

- A Professional seller account. Individual sellers are never eligible, full stop.
- The item in "new" condition for most categories. Used items compete in their own separate box.
- The seller in stock. Zero inventory means instant removal.
- Healthy account metrics. The Order Defect Rate must stay under 1 percent.

Half two, ranking, is the scoring. Among the offers that survive the filter, pick a winner. This part is the trade secret. What is publicly documented as the major inputs:

- Landed price. This is the number that matters, not the sticker price. Landed price is item price plus shipping plus tax. A seller at 641 with 40 rupees shipping loses to a seller at 649 with free shipping, because 649 beats 681.
- Delivery speed. How fast can this seller get it to this buyer's pin code.
- Fulfillment reliability and seller performance. Late shipments, cancellations, and returns all hurt.

### The recent architectural change (this is real and recent)

Two documented shifts matter, because they show the algorithm is a living system, not a fixed formula.

First, in 2023 Amazon renamed "Buy Box" to "Featured Offer" across Seller Central. Not just cosmetic. The system moved toward rotating several qualified sellers through the spot rather than freezing one winner, and it leaned harder on service quality.

Second, and bigger, around November 1, 2025 Amazon went "fulfillment-neutral". For years, Fulfillment by Amazon (FBA) carried a structural built-in advantage in the score. A seller who let Amazon warehouse and ship their goods got a thumb on the scale. That default thumb was removed. Now, per seller reporting on the change, eligibility is checked first, then offers are sorted by condition tier, then price competitiveness and seller metrics are weighed together, with delivery speed weighing very heavily. Since October 2025, a seller who ships their own goods but promises same-day handling can beat a slower FBA seller. Note the nuance: even after going fulfillment-neutral, FBA sellers still win far more often (commonly cited as three to five times more often on identical listings), because FBA sellers tend to be faster and more reliable, which is what the algorithm now rewards directly instead of rewarding the FBA label itself.

The lesson inside that change: they moved from rewarding a proxy (the FBA badge) to rewarding the real outcome (fast, reliable delivery). Same spirit as Zomato swapping a contaminated label for an honest one in the food-prep teardown.

### Category weights (documented, mechanism inferred)

The relative importance of price versus speed versus seller rating is not one global setting. It is weighted per category. For a commodity like AA batteries, landed price dominates, because every pack is identical and the only thing that separates sellers is price and speed. For a high-value or sensitive category, seller performance and reliability weigh more. That per-category weighting is documented in seller guidance. The exact weights are not public. Inference: this is a small config table keyed by category id, read at scoring time, so the same scoring code serves every category by looking up its weight vector. That is the standard way to make one ranker behave differently across categories without writing a ranker per category, the same trick Amazon search uses with per-category rankers.

### The data structures, and why (inference, clearly labeled)

The exact storage is not public. Here is how this class of problem is solved, item by item, with the real objects named.

- The offers for one product are an array of offer records. For our Logitech M185, that is eleven small structs, each holding seller id, landed price, delivery estimate to a region, stock count, and a cached seller-health score. Eleven items. You can scan them in nanoseconds.

- The result of the contest is one value: the winning offer id, plus maybe a short ordered list for the "other sellers" page. For the M185, the answer is "Cloudtail" plus a runner-up list. This answer is the thing the product page reads.

- The map from product to its current Featured Offer is a giant hash map, keyed by product id (the ASIN, Amazon's ten-character product identifier like B07GBXML8B). Key is the ASIN, value is the winning offer id and a short cache of the ranked list. The live page load is one keyed lookup into this map. That is O(1), constant time, no scan, no sort at request time.

This is the ledger's spine again, offline-think and online-lookup. The shopper's page load never runs the contest. It reads a precomputed answer. The contest runs earlier, off the hot path, triggered by change.

- What triggers a recompute is an event stream. Every price change, every stock change, every seller-metric update is an event. Picture it as a queue (a Kafka-style log). TechBazaar's bot drops the price by one rupee, that emits a "price changed for ASIN B07GBXML8B" event, a worker picks it up, re-scores the eleven offers for that one ASIN, and writes the new winner back into the hash map. Only that one product's contest reran. The other 350 million were untouched.

Why a queue and not a cron job that recomputes everything every minute. Because recomputing all 350 million contests every minute is insane waste when only a few million actually changed. The event stream means you do exactly as much work as the world changed, no more. Same reason Swiggy pushes GPS pings through Kafka instead of polling every rider.

- A second index maps seller id to the set of ASINs that seller offers. When a seller's account health drops (say TechBazaar gets a run of late shipments and its metric crosses a threshold), you need to re-score every product TechBazaar competes on. That reverse index tells you which ASINs to re-enqueue. Without it you would have no idea which of the 350 million contests this one seller touches.

### The scale story, three tiers

Tier one, 1,000 products, a handful of sellers each. Trivial. You could recompute the Featured Offer live on every page load and nobody would notice. A small shop could run this in a single database query. There is no problem to solve yet.

Tier two, 100,000 products, and now sellers run repricing bots. This is where it breaks. Over 60 percent of marketplace sellers use automated repricers that change prices every 5 to 15 minutes. Suddenly the inputs never sit still. If you still recompute on page load, two problems hit at once. First, the same contest gets recomputed thousands of times per hour by page loads even when nothing changed, pure waste. Second, a hot product loaded by many shoppers at once recomputes the same answer many times in parallel. The fix is the split described above: stop computing on read, precompute on change, cache the winner in the hash map, and serve reads as lookups. Move the work from "every page load" to "every actual change".

Tier three, 350 million-plus products, billions of events per day. The scale is real. Back in December 2013 Profitero measured Amazon making more than 2.5 million price changes per day on its own retail prices alone, a tenfold jump from 269,113 per day a year earlier. For comparison that same month Walmart made 54,633 price changes and Best Buy 52,956, in the whole month. Add roughly 1.9 million active third-party sellers, most running repricers, and the change events run into the millions per day and beyond. What breaks and what you do:

- One box (one server) cannot hold the map or the event load. So shard by ASIN. Product B07GBXML8B and its eleven offers live on one shard; another product lives on another. Each shard owns a slice of the catalog and runs its own contests. Because each contest is self-contained (one product's offers never affect another product's winner), sharding by ASIN gives near-perfect horizontal scaling. Add shards, add capacity, no coordination between them. This is the same property that let Notion shard by workspace and Stripe shard by account.

- Hot products are a hot key. On Prime Day, one viral deal (say a discounted Fire TV Stick) gets loaded millions of times in an hour, and its price and stock churn every few seconds. That single ASIN can swamp its shard. The survival moves are the ones from the hot-key lesson: cache the winner hard and serve it from many read replicas, and rate-limit how often that one contest reruns so a frantic repricing war does not melt the shard. You accept the winner being a second or two stale for a wildly popular item, because a slightly stale default is fine and a fallen-over shard is not.

- Stock and the winner must stay roughly honest. If the featured seller just went to zero stock, you do not want to feature them for long. Inventory changes are high-priority events that jump the queue, so an out-of-stock seller drops out of the box fast. This is softer than the atomic stock decrement Zepto needs for the last carton of milk, because featuring a just-sold-out seller for one extra second only costs a fallback to the next seller, not an oversell.

### Rotation, and why the box is sticky but not frozen

The audited numbers tell a story about how the winner is chosen among near-ties.

The Markup's 2021 investigation found that Amazon's own retail arm won the Featured Offer for about 40 percent of the popular products in its sample, while the next most common seller won it for only about half of one percent. That is a very top-heavy distribution.

But the box is not permanently frozen on one seller. A Northeastern study found that back in 2016 the winning seller changed for seven in ten products over a six-week window. When The Markup looked again over a twelve-week window years later, the winner changed for fewer than three in ten. So the box does rotate, just less than it used to.

Inference on the mechanism, grounded in seller reporting: when several eligible offers are close, commonly cited as within about 5 percent of the lowest landed price, the algorithm does not always hand the box to the single cheapest. It rotates the tied-enough sellers through the spot, giving each a share. That is weighted sampling among near-equals, not a hard argmax. Two reasons this is the right design. First, fairness: if only the rock-bottom price ever won, sellers would race to the bottom and many would just quit the product, thinning the marketplace. Letting a seller win at a price within 5 percent keeps more sellers in the game. Second, it avoids winner-freeze, the same failure mode YouTube and Instagram fight with position-bias correction: if the top spot never moved, the system could never learn whether a slightly different seller would convert better.

So the honest picture of the contest is: filter to eligible (matching), score by landed price and delivery and seller quality with per-category weights (ranking), and among the near-ties, rotate for a share rather than freeze on the single best. All of it precomputed on change, cached, and served to the shopper as a one-key lookup.

---

## 8. The retention and habit mechanic

This feature runs two loops at once, on two different users, and they feed each other.

The buyer loop moves revenue. The habit is "trust the default". Ananya learns, without ever thinking about it, that tapping Buy Now gives a fair price from a decent seller. So she stops comparison shopping. She stops opening other tabs. The one-tap default becomes muscle memory, which is exactly why an estimated 82 percent of sales flow through the box and not the "other sellers" link. The metric moved is conversion, straight revenue. The box removes the twenty-way decision that would otherwise leak sales.

The seller loop is the real flywheel, and it looks like Uber's liquidity flywheel, not an engagement loop. The box is a prize. To win it, roughly 1.9 million sellers cut prices and speed up delivery, many of them with bots repricing every few minutes. That competition drives prices down and delivery up. Lower prices and faster delivery pull in more buyers. More buyers make the box worth more to sellers, so they compete harder. Round and round. The rotation rule is what keeps the seller side from giving up: because you can win by being within about 5 percent of the lowest price, not only by being the single cheapest, more sellers stay in the fight instead of quitting a product they can never win. A marketplace with more live sellers is a stickier marketplace.

A real observed example of the loop's power: The Markup documented that once Amazon's own offer took the box, it tended to keep it, and the winner changed for fewer than three in ten products over twelve weeks. That stickiness is the flywheel's inertia. The default, once earned, keeps earning.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based editor that compiles visual effects to shippable code, plus an embeddable runtime. The Featured Offer teaches one sharp, specific thing about where to spend compute.

Precompute the winning variant off the hot path, cache it keyed by the inputs that actually change, and make the runtime a one-key lookup, never a solver.

Here is the concrete mapping. A single shader graph in Rare.lab does not compile to one shader. It compiles to a family of variants: a high-quality path for a desktop GPU, a cut-down path for a mid-tier phone, a fallback for a weak GPU, maybe different precision or feature-toggle permutations. At runtime, on a specific device, exactly one variant should run. That is the same shape as the Featured Offer: many candidates, one product, pick the one that wins for this context.

Do not pick it at frame time. Do what Amazon does with page loads. Precompute the best variant for each (device class, quality tier, feature set) offline or at compile time, store it in a map keyed by that context, and let the runtime do an O(1) lookup: given this GPU and this quality setting, load this precompiled variant. The embeddable runtime should never be running a cost model or a compile at the moment a frame needs to draw, the same way Amazon's product page never runs the seller contest at the moment it renders.

Then borrow the event-driven recompute. When an artist edits the node graph, or you add a new target device profile, emit a change event and recompute only the affected variants, then update the cache. Do not recompile the whole variant matrix on every edit, the same reason Amazon does not recompute 350 million contests every minute. Recompute exactly what changed. Keep a reverse index too: "which compiled variants depend on this shared sub-graph or this device profile", so when a shared node changes you know precisely which variants to rebuild, the way Amazon's seller-to-ASIN index tells it which contests to rerun when one seller's metric moves.

And take the rotation idea as a warning against premature freezing. When two variants are near-tied on measured cost or quality for a device, do not hard-freeze on the one that won your first benchmark. Keep a small rotation or A/B so you keep measuring real performance on real hardware in the field, because the benchmark that picked the winner was made on your machine, not the user's phone. The single argmax you trust today can quietly be wrong on a device you never tested.

The one-line version: make selection expensive and rare, make the runtime lookup cheap and constant, invalidate by event not by clock, and never let the hot path solve a problem you could have precomputed.

---

## Sources

- Amazon Seller Central and official seller guidance on the Featured Offer (formerly Buy Box): eligibility, landed price, delivery, and performance factors. https://sell.amazon.com/blog/buy-box-featured-offer
- The Markup, "When Amazon Takes the Buy Box, It Doesn't Give It Up" (Oct 14, 2021): audit of 3,492 popular products, the ~40 percent Amazon-wins figure, and rotation/stickiness numbers. https://themarkup.org/amazons-advantage/2021/10/14/when-amazon-takes-the-buy-box-it-doesnt-give-it-up
- Profitero, "Amazon.com Makes More Than 2.5 Million Price Changes Every Day" (Dec 2013): the 2.5M/day figure, the tenfold jump from 269,113/day, and the Walmart/Best Buy comparison. https://www.profitero.com/blog/2013/12/profitero-reveals-that-amazon-com-makes-more-than-2-5-million-price-changes-every-day
- US Patent 8,630,923, "Virtual shelf with single-product choice and automatic multiple-vendor selection." https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/8630923
- GoDataFeed, "Amazon sunset the Buy Box eligibility gate: feed architecture became a continuous ranking input" (on the move from a boolean eligibility gate to continuous unified ranking). https://www.godatafeed.com/blog/amazon-buy-box-eligibility-changes
- Sentrykit, "Amazon Featured Offer Went Fulfillment-Neutral in 2025" (the ~Nov 1, 2025 removal of FBA's built-in weight and the delivery-speed emphasis). https://sentrykit.com/blog/amazon-featured-offer-fulfillment-neutral-2026/
- Feedvisor, "Amazon Buy Box: How to Win the Featured Offer": factor list, category weighting, and the ~82 percent-of-sales industry estimate. https://feedvisor.com/university/amazon-buy-box/
- "Antitrust, Amazon, and Algorithmic Auditing" (arXiv 2403.18623): academic audit of featured-offer selection signals. https://arxiv.org/pdf/2403.18623

Note on inference: the eligibility gate, the documented factors (landed price, delivery speed, fulfillment reliability, seller metrics), the ~82 percent figure, the audited win rates, the 2.5M price-changes/day figure, the 2023 rename, and the 2025 fulfillment-neutral shift are from the sources above. The data structures (per-ASIN offer arrays, the ASIN-to-winner hash map, the event queue, the seller-to-ASIN reverse index), the sharding-by-ASIN scale story, and the weighted-rotation mechanism are labeled inference: they are how this class of problem is solved in practice, not published Amazon internals.
