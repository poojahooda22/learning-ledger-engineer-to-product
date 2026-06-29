# Zomato: Food Preparation Time prediction (the "when will the food be ready" clock)

Date: 2026-06-29
Product: Zomato
Feature: Kitchen Preparation Time (KPT) / Food Preparation Time (FPT) prediction, the number that decides when to send a rider

---

## 1. The user

It is 9:10 pm on a Tuesday. Rahul in Indiranagar, Bangalore, just tapped "Place order" on a chicken biryani and a plate of kebabs from Meghana Foods. The app says "Delivery in 38 min." He puts the phone face down and goes back to the cricket.

He is not the only user of this feature. There are two more, and they never see the screen.

The first is the delivery partner, call her Lakshmi, riding a scooter three blocks away. The app is about to decide whether to send her to Meghana now or in eight minutes.

The second is Meghana Foods' kitchen itself, slammed with 14 live orders during the dinner rush, a single biryani station, and one person tapping a tablet.

The feature in question is the invisible clock that says: this specific biryani will be packed and ready at 9:31 pm. Get a rider to the counter at 9:31, not 9:22 and not 9:40.

## 2. The real problem

Food delivery has one number everyone obsesses over: the total time on the customer's screen. But that number is a sum of legs, and the leg that nobody can see directly is how long the kitchen takes.

Think about what goes wrong when the kitchen-time guess is off, in either direction.

Guess too low (say the food needs 22 minutes and you predicted 14): the rider arrives at 9:24, the biryani is still in the pot, and she stands at the counter for eight minutes doing nothing. Multiply that idle time across a city on a Friday night and you have burned a huge slice of your fleet's capacity on waiting. Riders waiting at counters is the single most expensive form of nothing in this business.

Guess too high (the food needs 22 minutes and you predicted 32): the food is packed and steaming at 9:32, but the rider was told to show up at 9:42. The biryani sits on the counter going cold and soggy for ten minutes. The customer's "38 min" becomes 48, and a cold order is a one-star order.

So this is not a "make the estimate accurate" problem in the usual symmetric sense. The cost of being five minutes early is completely different from the cost of being five minutes late. That asymmetry is the whole story, and it is why Zomato did not just train an ordinary regression and call it done.

## 3. The feature in one sentence

Predict, per order, how many minutes the restaurant will take to prepare it, so the dispatch system sends a rider to arrive exactly when the food is ready instead of too early (idle rider) or too late (cold food).

## 4. Jobs to be done

What is the user really hiring this number to do?

- The customer hires it for an honest promise: "tell me when it will actually arrive, and then keep that promise." Zomato frames this as setting correct expectations and then meeting them, measured by punctuality and a tolerance interval. (Source: Zomato, "The accurate ETA to customer satisfaction.")
- The rider hires it to not waste her shift standing at counters. Every minute she waits is a minute she is not earning on the next trip.
- Zomato the platform hires it to get more orders out of the same fleet. If you shave the average counter-wait, every rider completes more deliveries per hour, and you need fewer riders for the same volume. That is a direct cost line.
- The restaurant hires it to keep food fresh. A biryani that sits packed for ten minutes hurts the restaurant's rating as much as Zomato's.

## 5. How it works for the user

For Rahul: he sees one friendly number, "Delivery in 38 min," and a live tracker that later shows the rider approaching. He never sees that 38 was built from three separate predictions stacked together. (Source: Zomato states total delivery time = kitchen preparation time + rider-to-restaurant travel time + restaurant-to-customer travel time.)

For Lakshmi the rider: she gets a ping to head to Meghana at a moment chosen so she rolls up right as the food is bagged. When she arrives, ideally the order is on the counter. If Zomato got the kitchen time right, she is in and out.

For Meghana's kitchen: an order lands on their Restaurant Partner tablet. When the kebabs and biryani are bagged, a staffer taps a "Food Order Ready" (FOR) button. That tap is small for the restaurant but huge for Zomato, because it is the only honest ground-truth signal of when food was actually done. (Source: Zomato introduced the FOR button in the Restaurant Partner app; in early results it improved their within-5-minutes prediction accuracy by about 9 percent.)

## 6. The actual flow, step by step

1. 9:10 pm: Rahul places the order. Zomato needs a delivery promise instantly, before any rider is even assigned.
2. The system computes Kitchen Preparation Time (KPT) for this order at this restaurant, right now, during the dinner rush. Say it predicts 21 minutes, so food ready at about 9:31.
3. It separately predicts rider-to-restaurant travel time and restaurant-to-customer travel time, adds them to KPT plus a dynamic buffer, and shows Rahul a single ETA. (Source: Zomato ETA blog.)
4. The dispatch system uses the 9:31 ready-time to time the rider assignment: it wants a free rider whose travel time lands her at Meghana around 9:31, not at 9:22.
5. Lakshmi gets the ping, rides over, and arrives at 9:30.
6. 9:31: Meghana's staffer taps "Food Order Ready." That timestamp becomes the training label for what the kitchen actually took.
7. Lakshmi picks up, the tracker switches to "on the way," and the restaurant-to-customer leg takes over.
8. Overnight, the FOR timestamps from millions of orders flow back into training and the KPT model gets a little sharper for Meghana specifically, and for "biryani restaurants in Bangalore at 9 pm on weekdays" generally.

## 7. Under the hood, like the engineer

This is the heart of it. KPT looks like a plain "predict a number" regression, but three things make it hard: the input is a moving kitchen, not a static order; the cost of error is lopsided; and you must serve a fresh prediction the instant the order is placed across more than a million orders a day.

### Why a sequence model, not a flat row of features

The naive version is a tabular regression: features like restaurant id, item count, hour of day, day of week, current rush, feed into gradient boosted trees, predict minutes. That is a reasonable baseline and Zomato uses tree models (LightGBM) elsewhere in the ETA stack for speed and accuracy. (Source: Zomato switched its travel-time ETA from a map-graph model to a tree-based LightGBM model.)

But kitchen time has a property a flat row hides: it depends on what else the kitchen is doing right now and what it just finished doing. Meghana taking 21 minutes for a biryani when the kitchen is empty is a different animal from 21 minutes when 14 orders are stacked up. The recent past predicts the near future.

So Zomato's KPT model treats the kitchen as a short time series. Per the Zomato engineering blog ("The Deep Tech Behind Estimating Food Preparation Time"), the model takes two sequences:

- the running orders at the restaurant right now (capped at the most recent 5), and
- the last 5 completed orders.

Each sequence is fed through a stacked LSTM layer. An LSTM is a recurrent neural net that reads items in order and carries a memory state forward, so it can learn patterns like "the last three orders each took longer than predicted, this kitchen is falling behind, so add minutes." The LSTM output is a vector that summarizes the kitchen's current momentum.

That vector is then concatenated with:

- the present order's own features (how many items, which items, order value, time of day), and
- a Restaurant Embedding Vector.

The restaurant embedding is the quiet workhorse. Instead of one-hot encoding 150,000 restaurants (a giant sparse vector that learns nothing shared), each restaurant is mapped to a small dense learned vector. Meghana Foods becomes, say, 32 numbers that encode "fast biryani place, handles rush well, slows on weekends." Two restaurants with similar behavior land near each other in this space, so the model can generalize from busy biryani joints it has seen a lot to one it has seen rarely. This is the same trick YouTube and Amazon use to turn millions of discrete ids into learnable geometry (see the 2026-06-22 YouTube and 2026-06-23 Amazon teardowns); here it turns "which restaurant" into a few dozen meaningful dials.

The concatenated vector goes through a 2-layer dense network and the model regresses on FPT, the actual preparation time. (Source: Zomato engineering blog.)

Data structures in play, concretely:

- A bounded queue per restaurant for the last 5 running and last 5 completed orders. Bounded, because the LSTM reads a fixed window and because you do not want unbounded memory per restaurant across 150k restaurants. Old orders fall off the back.
- An embedding table (a hash map from restaurant id to a dense vector) looked up in constant time per order.
- The present-order features as a plain feature vector.

### The lopsided cost: quantile loss instead of MAE

Now the asymmetry. A normal regression trains on Mean Absolute Error (MAE) or squared error, which treat "5 minutes early" and "5 minutes late" as equally bad. For food prep they are not equally bad, and worse, the right trade-off depends on which side of the platform you are protecting at that moment.

Zomato uses a quantile loss. The idea: for a quantile q below 0.5, the loss penalizes over-prediction more than under-prediction, pulling the model to predict lower; for q above 0.5 it does the opposite; at q = 0.5 it collapses to plain MAE. (Source: Zomato, "Predicting your order's Food Preparation Time.")

Why this is exactly the right tool here: predicting a higher KPT means the rider is told to arrive later, which risks cold food but protects against idle waiting; predicting lower means the rider arrives sooner, protecting freshness but risking idle riders. Quantile q is the dial between those two business outcomes. Zomato went further and modified the quantile loss with tunable penalty factors (the blog calls them m and n) so they get MAE-like smoothness while still tilting the penalty asymmetrically toward whichever error hurts the business metric they are optimizing. (Source: Zomato FPT blog.)

This is the deep point: the loss function, not just the features, is where the product decision lives. Two teams could ship the identical LSTM and get opposite rider behavior purely by choosing q = 0.3 versus q = 0.7.

### The label problem: when is "ready" actually ready

A model is only as honest as its labels. The label here is "how long did the kitchen really take," and for a long time Zomato did not truly know it. The proxy was often the moment the rider picked the food up, which is contaminated: if the rider was late, the food was "ready" long before pickup, and the model learns the kitchen is slower than it is.

That is what the FOR button fixes. When the restaurant taps "Food Order Ready," you get a clean timestamp of completion independent of when the rider showed up. The honest label is worth more than a fancier model: just adding FOR improved within-5-minutes accuracy by about 9 percent. (Source: Zomato FPT blog.)

It is still imperfect, and Zomato says so plainly. Some merchants tap FOR only when the rider arrives, not when the food is done (rider-influenced marking). There is no signal for kitchen-wide rush beyond Zomato's own orders, and there is human and operational bias in manual tapping. Labeled clearly as Zomato's own stated limitation, not my inference. The takeaway: the team invested in better ground truth before chasing model complexity, which is usually the higher-leverage move.

### The scale story at three tiers

Tier 1, about 1,000 orders a day, one small city. A single LightGBM or even a per-restaurant rolling average works. You could almost compute KPT as "this restaurant's median prep time for this item count over the last week." The sequence model is overkill. Nothing is on fire.

What breaks going up: per-restaurant medians stop capturing rush dynamics, and cold-start restaurants (new joins, of which Zomato adds many) have no history at all.

Tier 2, about 100,000 orders a day, dozens of cities. Now you need the model to share knowledge across restaurants, which is exactly what the restaurant embedding buys you: a brand-new biryani place inherits sensible behavior from similar embedded restaurants on day one instead of guessing blind. Serving must be online: the prediction is needed the instant the order is placed, so it cannot be a heavy batch job. Real-time features (current running orders) must be available in milliseconds, which is why platforms in this space keep live state in Redis fed by a stream. (Source: Zomato describes real-time features computed via Kafka and Flink and served from a Redis cluster, with batch features in Spark and a key-value store.)

What breaks going up: a single hot restaurant during a cricket-final dinner rush has its running-orders queue mutated dozens of times a minute; and the model must serve predictions under tight latency at peak.

Tier 3, 1.3 to 2 million orders a day, 150,000 restaurants, peaks over 2 million orders on a single day (about 1,400 orders every minute on 20 June 2024). (Source: Zomato investor data and reporting.) Here the architecture matters:

- Keep the expensive thinking offline. The LSTM plus embeddings are trained overnight on the full history of FOR timestamps. The live path is one forward pass through a fixed-size network: look up the embedding (hash map), read the bounded running/completed queues (already in memory), run a small LSTM and 2 dense layers, output one float. Constant work per order, independent of how many restaurants exist. This is the same offline-think / online-lookup spine as Discover Weekly, YouTube, and Razorpay routing in earlier teardowns.
- Shard the live kitchen state by restaurant. Each restaurant's running-order queue is independent, so partition by restaurant id and you scale horizontally for free; no global lock, no cross-restaurant coordination on the hot path.
- Bound everything. Max 5 running, last 5 completed. Fixed window keeps per-restaurant memory and per-prediction compute flat even when a restaurant has 40 live orders.
- For the assignment loop specifically, even the small neural net can be too slow when you are scoring many rider-order pairs per second. Zomato built an equation-based ultra-fast ETA (from delivery-partner speed and "medal" features) for the dispatch system's inner loop, which raised assignment-prediction compliance by about 4 percent. (Source: Zomato ETA blog.) The lesson: the model you show the customer and the model you run inside the matching loop do not have to be the same model. Use the cheap one where you call it a million times.

## 8. The retention and habit mechanic

This feature does not have a flashy hook like an unseen-stories ring. Its retention power is quieter and arguably stronger: trust in the number.

The loop is: Zomato promises 38 minutes, the food arrives in 37, hot. Rahul does not consciously notice. But the next time he is hungry and weighing Zomato against cooking or against a competitor, the friction is low because the app has never lied to him. An app whose ETA is consistently honest gets opened on reflex. An app that says 30 and delivers cold food at 55 gets uninstalled after a few burns.

The metric this moves is primarily retention, with a direct line to revenue through fleet efficiency. Every minute shaved off rider counter-waiting lets the same fleet complete more orders, which lowers cost per order and lets Zomato keep delivery fees low, which keeps people ordering. Accurate KPT is upstream of both "customer trusts the promise" (retention) and "fleet does more with less" (margin). Zomato itself centers ETA on punctuality and a tolerance interval as the customer-satisfaction lever. (Source: Zomato ETA blog.)

Real observed example of the loop working: on 20 June 2024 Zomato sustained over 2 million orders in a day, roughly 1,400 a minute, without the promise system collapsing. That kind of peak only holds if the prediction-and-dispatch spine degrades gracefully, which is the retention mechanic doing its job invisibly at the worst possible moment.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into shippable shader code and runs it in an embeddable runtime. The KPT lesson that maps cleanest is: pick your loss to match the asymmetric cost, and let it be a tunable product dial, not a fixed default.

Concretely, when Rare.lab's compiler or runtime estimates the cost of a node graph (to decide LOD, to budget a frame, to pick a fallback path on a weak GPU), the two errors are not symmetric. Predicting a shader is cheaper than it really is means you schedule too much work and drop a frame, which users see as jank. Predicting it is more expensive than it really is means you needlessly downgrade quality, which is annoying but not a stutter. On a 60 fps budget those costs are wildly different, just like early-rider versus cold-food.

So do not estimate frame cost with plain squared-error regression that treats overshoot and undershoot equally. Use a quantile-style objective tilted to be conservative about the expensive direction (never blow the frame budget), and expose the quantile as a single knob: a "smooth, never drop a frame" setting (predict high, downgrade early) versus a "max fidelity, occasional hitch" setting (predict closer to the mean). One dial, two product personalities, same estimator.

Two supporting lessons that fall straight out of the KPT design:

- Keep the expensive estimator offline and make the live path a fixed-size forward pass. Profile shader-node costs offline across real device classes, bake the result into small per-node cost vectors (the analog of the restaurant embedding), and at runtime do a constant-time lookup plus a tiny sum, not a live simulation. Constant work per frame regardless of graph size.
- Invest in honest ground truth before model cleverness. Zomato's single biggest accuracy jump came from the FOR button, a better label, not a better network. For Rare.lab that means instrumenting the runtime to report actual measured GPU time per node on real hardware and feeding it back, rather than tuning the cost model against synthetic benchmarks that lie the way rider-pickup timestamps lied.

---

## Sources

- Zomato Engineering, "The Deep Tech Behind Estimating Food Preparation Time": https://www.zomato.com/blog/food-preparation-time/ (Medium mirror: https://medium.com/zomato-technology/the-deep-tech-behind-estimating-food-preparation-time-e5068807acb0)
- Zomato Engineering, "Predicting your order's Food Preparation Time" (quantile loss, FOR button results): https://blog.zomato.com/predicting-fpt-optimally
- Zomato Engineering, "The accurate ETA to customer satisfaction" (Part One and Part Two): https://blog.zomato.com/the-accurate-eta-to-customer-satisfaction-part-one and https://blog.zomato.com/the-accurate-eta-to-customer-satisfaction-part-two
- Zomato Engineering, "The elements of scalable machine learning": https://blog.zomato.com/elements-of-scalable-machine-learning
- Business Standard, "What is Zomato's daily order count?" (2 million orders on 20 June 2024, ~1,400/min): https://www.business-standard.com/companies/news/what-is-zomato-s-daily-order-count-company-s-new-feature-spills-the-beans-124062200356_1.html
- Statista, Zomato number of orders FY2023 (647M orders, +21%): https://www.statista.com/statistics/1110238/zomato-number-of-orders/
