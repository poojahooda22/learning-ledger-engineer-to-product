# Razorpay Smart Routing: picking the one payment path most likely to say yes

**Date:** 2026-06-26
**Product:** Razorpay (Optimizer)
**Feature:** Smart payment routing (intelligent terminal selection across many gateways and banks)

---

## 1. The user

It is 9:40 PM on the night of an IPL final. Rohit is on his couch in Pune with the
match paused at the toss. He opens Dream11, builds his team, and goes to add 499
rupees to his wallet so he can join a contest before the lineup locks. He taps
"Add Cash," picks UPI, and his bank app opens for him to approve.

There are two people standing behind that one tap, and Rohit is only one of them.
The other is the merchant, Dream11, whose revenue for the whole night depends on
that 499 rupees actually going through. On a normal Tuesday this is boring. On
final night, lakhs of people across India are doing the exact same thing in the
same ten minutes, hammering the same few banks. Some of those banks are quietly
buckling under the load right now.

Rohit does not know or care about any of that. He wants the money to move and the
contest to open. He has about fifteen seconds of patience before he gets annoyed.

## 2. The real problem

Here is the thing nobody tells you about online payments: a payment is not one
company saying yes. It is a relay race through four or five companies, and any one
of them can drop the baton.

When Rohit pays 499 rupees, the request travels from Dream11, to a payment
aggregator, to an acquiring bank, across a card network or the UPI rails (NPCI),
to Rohit's own bank (the issuer), and back. Each hop can fail. Rohit's bank might
be having a bad night. The UPI endpoint for that bank might be timing out. The
specific aggregator Dream11 picked might have a degraded link to that bank right
now, even though a different aggregator's link to the same bank is perfectly fine.

So the real pain is this: the same 499 rupee payment can succeed or fail purely
based on which path you sent it down, and the best path changes minute by minute.
A bank that was healthy at 9:35 PM can be dropping one in three transactions by
9:42 PM. If you hard-wire Dream11 to one gateway, every time that gateway has a
bad ten minutes, Dream11 loses real money and real users walk away. Failed
payments are not a small leak. Cart and checkout abandonment from a single failed
attempt is one of the largest silent revenue losses in Indian commerce.

The friend-level version: it is like trying to call someone when one cell tower is
congested. Your phone should quietly try a different tower instead of just showing
you "call failed." Most payment setups show you "call failed."

## 3. The feature in one sentence

Smart Routing looks at the live health of every possible path a payment could take
right now, and sends each transaction down the single path most likely to succeed,
retrying down the next-best path if the first one declines.

## 4. Jobs to be done

What is Rohit really hiring this for? He is not. Rohit never sees it. That is the
whole point of good plumbing.

The merchant, Dream11, is the one hiring it, and the jobs are:

- "When my default bank link is having a bad night, move my traffic somewhere else
  before I even notice, so my success rate does not crater during my biggest hour."
- "Do not make me, the merchant, manually watch dashboards and flip switches at
  9:40 PM. Decide per transaction, automatically."
- "When a payment soft-declines for a flaky reason, try again down another path
  instead of telling my customer it failed."
- "Give me one integration that hides five aggregators behind it, so I am not
  rebuilding plumbing every time I add a bank."

And the silent job for Rohit: "Let me add my 499 rupees on the first try and get
back to my team before the lineup locks."

The metric all of this moves is the payment success rate (also called the
authorization rate): out of every 100 genuine attempts, how many end in a captured
payment. Razorpay reports Smart Routing lifting this by roughly 4 to 6 percent in
their 2021 production system, and their Optimizer marketing cites up to around 10
percent uplift on the newer stack. For a large merchant, one percent of success
rate is crores of rupees a year. This is a revenue feature wearing an
infrastructure costume.

## 5. How it works for the user (the visible experience)

From Rohit's side: he taps "Add Cash," picks UPI, approves in his bank app, and a
second later sees "499 added." That is the entire visible experience. If the first
path had failed silently and a retry on a second path had succeeded, Rohit would
have seen the exact same thing, maybe a second slower. The feature's highest
praise is that it is invisible.

From Dream11's side: they integrated once with Razorpay Optimizer. Behind that one
integration sit many aggregators and bank connections (Razorpay's marketing talks
about routing across 15-plus gateway connections). Dream11 can set a few business
rules (for example, send high-value transactions a certain way, or keep a minimum
share with a particular provider for commercial reasons), and then let the system
decide everything else. They watch a success-rate dashboard, not a live war room.

## 6. The actual flow, step by step

Walk Rohit's 499 rupees, tap by tap.

1. Rohit taps "Add Cash," chooses UPI, confirms 499 rupees.
2. Dream11's app calls Razorpay to create an order and start a payment.
3. The routing service now has to answer one question in a few milliseconds:
   of all the ways I can send this UPI payment, which one do I pick first?
4. **Filter (the matching half).** Throw out every path that cannot process this
   payment at all. Wrong method (a card-only terminal cannot do UPI), wrong
   currency, a bank connection currently flagged as down. What is left is a small
   set of eligible "terminals" (think of a terminal as one specific lane: this
   aggregator, talking to this acquiring bank, for this method).
5. **Rank (the ranking half).** For each eligible terminal, look up its very
   recent success rate and predicted success probability for a payment like this
   one, right now. Sort them. Pick the top one.
6. Send the payment down that terminal. Rohit's bank app opens, he approves.
7. **If it declines for a soft, retryable reason** (a timeout, a flaky
   "try again later"), do not show Rohit a failure. Cascade: pick the next-best
   terminal and try once more. Razorpay says this failover happens in under 500
   milliseconds.
8. On success, capture the payment, tell Dream11, show Rohit "499 added."
9. **Quietly, in the background,** record what happened on that terminal (success
   or fail, how slow) so the next person's routing decision is a little smarter.
   This is the feedback loop, and it is the soul of the feature.

The key insight in this flow is that step 5 is not a fixed rule. The "best"
terminal at 9:35 PM is not the best at 9:42 PM, because step 9 keeps changing the
scores underneath.

## 7. Under the hood, like the engineer

This is the heart of it. Smart routing is the same two-act structure as search:
**matching** (which paths are even possible) then **ranking** (which possible path
is best right now). The twist that makes payments special is that the ranking
signal is non-stationary: it goes stale in minutes, because banks have good
minutes and bad minutes. Most of the engineering is about estimating a moving
target fast and cheap.

I will lean on three real, public papers, two of them from Razorpay's own team.

### The vocabulary: what is a "terminal"

A terminal is the unit of routing. It is one concrete lane the money can travel:
a specific aggregator plus a specific acquiring bank (a specific MID) for a
specific method. "UPI via Aggregator A to ICICI" and "UPI via Aggregator B to
ICICI" are two different terminals, and on final night one can be healthy while the
other is choking, even though the destination bank is the same. The router does
not pick a "bank," it picks a terminal. A large merchant might have dozens of
terminals live at once. That is a small set, and the smallness is what makes the
live path cheap. Hold that thought for the scale story.

### Act one: matching (cheap candidate filtering)

You start with all terminals and you need the ones that can serve this exact
payment. This is a filter, not a search over millions, because the set of terminals
is small (tens, not millions). A hash map from method to the terminals that support
it, plus a few boolean checks (currency, amount band, terminal enabled), gets you
the candidate set in microseconds.

The interesting part of matching is throwing out terminals that are technically
capable but currently sick. Razorpay's 2021 paper (Bygari et al, "An AI-powered
Smart Routing Solution for Payment Systems") calls this the **static module**: it
does "initial filtering of the terminals using static rules and a logistic
regression model that predicts gateway downtimes." So before you even rank, a small
model has already pulled the terminals it thinks are about to fail out of the pool.
For Rohit on final night, if Aggregator A's link to his bank has been timing out for
the last two minutes, the downtime predictor has likely already benched it.

The data structure here is unglamorous and correct: a hash map from
terminal id to a live health record, plus a set of currently-disabled terminal ids.
Lookups are O(1). The candidate set after filtering might be five terminals.

### Act two: ranking (predict success probability per terminal)

Now you have a handful of eligible terminals and you must order them by "most
likely to say yes for a payment like this, right now." Razorpay has shipped two
different brains for this over the years, and the contrast between them is the best
part of the teardown.

**Brain one: the random forest (2021 paper).** After the static filter, a
**dynamic module** "computes a lot of novel features based on success rate, payment
attributes, time lag, etc. to model the terminal behaviour accurately." These
features are "updated using an adaptive time decay rate algorithm in real-time
using a feedback loop and passed to a random forest classifier to predict the
success probabilities for every terminal." So for each candidate terminal, build a
feature vector (recent success rate on that terminal, the method, the amount band,
the issuing bank, how stale the signal is) and ask a random forest: probability
this succeeds? Then sort the terminals by that probability and pick the top.

Why a random forest and not a single logistic regression for the ranking? Because
the success of a payment is full of interaction effects (this bank is bad for
high-value UPI specifically, but fine for small card payments), and a forest of
decision trees captures those "if this and this and this" splits naturally without
you hand-coding them. The paper reports the random forest beating the other models
they tried, with a 4 to 6 percent success-rate lift in production across cards, UPI,
and net banking, routing millions of live transactions.

**Brain two: the non-stationary bandit (2023 paper).** Razorpay's later paper,
"Maximizing Success Rate of Payment Routing using Non-stationary Bandits," reframes
the whole thing as a multi-armed bandit. Each terminal is a slot-machine arm with
an unknown, drifting payout (its current success probability). You want to pull the
best arm most of the time (exploit) while still occasionally testing the others, so
you notice the moment a sleeping terminal wakes up or a good one starts to fail
(explore). The "non-stationary" part is the entire point: a normal bandit assumes
each arm's payout is fixed, but a bank's success rate is a moving target, so you
must forget old evidence.

How do you forget? The paper found two recency tricks worked best:

- **Sliding-window UCB** with a window of the last 200 transactions per arm. You
  only count the most recent 200 outcomes on a terminal, so a bank that was great an
  hour ago but is failing now gets demoted within roughly 200 transactions, not days.
- **Discounted UCB** with a discount factor of 0.6, where each older outcome counts
  for less in a geometric decay. Same idea, smooth instead of a hard window.

Both gave the lowest cumulative regret in their simulator. "Regret" here is just the
success you left on the table by not always picking the truly-best arm. The UCB part
("upper confidence bound") is the elegant bit: you do not rank a terminal by its raw
recent success rate, you rank it by success rate plus an uncertainty bonus that is
larger when you have tested it less. That bonus is what forces a little exploration
without a clumsy "randomly send 5 percent of traffic to a coin flip" rule. A
terminal you have barely tried gets a benefit of the doubt, gets some traffic, and
quickly proves itself or not.

For Rohit: at 9:42 PM the bandit has been watching his bank's UPI terminal fail. The
sliding window of 200 has filled with declines, the score has dropped below a rival
terminal, and the very next person's payment (and a retry of Rohit's, if his soft-
declined) gets sent the other way. No human flipped a switch.

A 2025 paper from a competitor, Juspay ("A Control-Theoretic Approach to Dynamic
Payment Routing"), frames the same problem as a feedback controller: a reward-and-
penalize loop, explicitly inspired by a PID controller, that nudges each gateway's
health score up on success and down on failure so the score tracks the live success
rate. Different vocabulary, identical instinct: a closed loop that senses, scores,
and steers, continuously.

### Where does the sorting happen

On a server, never on Rohit's phone. The phone sends one request. A routing service
holds the live per-terminal scores in memory, filters to the candidate set, sorts
that tiny set (sorting five things is free), and returns the chosen terminal. The
expensive thinking, training the random forest, tuning the bandit's window and
discount on a simulator, happens offline on logs. The live path is a filter, a
scan over a few terminals, and a pick. That separation (expensive learning offline,
cheap keyed decision online) is the same pattern that shows up in Spotify Discover
Weekly and YouTube recommendations in this very ledger. It is the pattern.

### The scale story, three tiers

**1,000 transactions a day (a small merchant).** You barely need this. One default
gateway, maybe one backup, a hand-written rule: "if it fails, try the backup once."
Per-terminal statistics are too sparse to be meaningful (you cannot estimate a
bank's current success rate from three data points). The honest engineering at this
tier is static rules. Razorpay's static module alone (rules plus a downtime
predictor) covers most of the value.

What breaks going up: the rules go stale and nobody updates them, and you have too
few data points per terminal to be smart.

**100,000 transactions a day.** Now per-terminal success rates are statistically
real, and the non-stationarity bites: you can clearly see banks having bad hours.
This is where the dynamic brain earns its keep. You keep a recent-history estimate
per terminal (a sliding window of the last N outcomes, or a time-decayed running
average), update it on every transaction via the feedback loop, and rank with the
forest or the bandit. The data structures are small: a hash map from terminal id to
a ring buffer of recent outcomes (or a single decayed float), plus the model. State
fits in memory on one box and is cheap to update.

What breaks going up: a single in-memory counter per terminal becomes a write
hotspot, and one box cannot hold the request rate.

**10 million plus transactions, peaks of 10,000 a second (final night).** Razorpay's
2023 paper is explicit that their Routing Service uses a "novel Ray-based
implementation for optimally scaling bandit-based payment routing to over 10,000
transactions per second," while staying inside PCI DSS constraints. Ray is a
distributed-actor framework, which is the tell: the per-terminal state is held in
distributed actors so that updating ICICI-UPI's score and reading it happen across
many workers without a single global lock. Two specific things to survive this tier:

- **The hot-counter problem.** On final night, one popular bank's terminal is being
  written to thousands of times a second as outcomes stream in. A single counter
  there is a contention point (the same hazard as Zepto's last-carton SKU counter or
  a flash-sale stock decrement in this ledger). You shard the counter into sub-
  counters and aggregate, or you push updates through a stream and recompute the
  decayed estimate in small batches, so no single lock is on the hot path.
- **Read-cheap, write-streamed.** The routing decision must read a fresh score in
  well under the latency budget (the whole failover including a retry is quoted at
  under 500 milliseconds, so a single decision is a few milliseconds). So the live
  read is an in-memory keyed lookup of a precomputed score, and the outcomes flow
  back asynchronously to update that score. You never recompute a model on the hot
  path; you only read a number and occasionally write an outcome.

Across all tiers the candidate-set size stays tiny (tens of terminals), which is
exactly why routing cost does not grow with transaction volume the way a naive
design would. The volume grows the write rate on the score updates, not the size of
the per-decision sort. You scale the bookkeeping, not the decision.

### One honest caveat on what is public

The two Razorpay papers and the Juspay paper are real and specific about
algorithms, hyperparameters, and the Ray-based serving system. The exact current
production blend inside Optimizer (which model runs for which method, the precise
feature set, the live thresholds) is not fully public, and the "150-plus parameters,
600 million data points, 1 billion transactions" figures come from Razorpay's
marketing rather than a paper. I have labeled the paper-backed mechanics as fact and
kept the marketing numbers as their claims. The class-of-problem reasoning (filter
then rank, recency-weighted estimates, explore-exploit, shard the hot counter) is
standard and well grounded even where a specific internal is not disclosed.

## 8. The retention and habit mechanic

This feature has two loops, one technical and one commercial, and they feed each
other.

The technical loop is the feedback loop inside the algorithm. Every transaction's
outcome updates the score of the terminal it used, which changes the next routing
decision. The system literally gets smarter with every payment, and it self-heals:
the moment a bank degrades, the falling score routes traffic away within a couple
hundred transactions, and the moment it recovers, the exploration bonus lets a
trickle back in to notice the recovery. It is a control loop that never stops
running. That is the machine's habit.

The human habit it builds is the merchant's, and the metric it moves is revenue,
through retention of the merchant on the platform. The loop goes: better routing
means a higher success rate, a higher success rate means more captured revenue and
fewer abandoned checkouts, more revenue means the merchant has a concrete, monthly,
dashboard-visible reason to stay and to push more volume through Razorpay. Razorpay
even ships an "Optimizer performance report" so the merchant can see the recovered
transactions the router saved them. That report is the habit hook: it turns an
invisible algorithm into a visible monthly number the merchant comes back to check.
A 5 percent success-rate lift on a Dream11-sized volume is a number a CFO reopens
the dashboard for.

And the end-user retention is the quietest loop of all: Rohit's 499 rupees went
through on the first try, the contest opened before lineup lock, and he never had a
reason to rage-quit the app over a failed payment. The feature retains users by
removing a reason to leave, which is the hardest kind of retention to measure and
the most valuable.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph to shippable shader code and ships an embeddable
runtime that then runs on a wild zoo of devices: a high-end desktop GPU, a mid
Android phone, WebGPU here, WebGL fallback there, integrated graphics on a cheap
laptop. The performance of a given shader path on a given device is exactly a
non-stationary, per-lane success problem. The same effect that hits 60 fps on a
gaming laptop can drop frames on a three-year-old phone, and even on one device it
shifts as the GPU thermally throttles mid-session.

So borrow the whole structure: **filter then rank, with a recency-weighted health
score per backend, and cascade on failure.**

- **Filter (matching).** At runtime, throw out shader variants the device cannot run
  at all (no WebGPU, no compute shaders, texture limits, missing extensions). This is
  the static module: a capability hash map, an O(1) check, done once at init.
- **Rank (the bandit).** For the variants that survive, keep a recency-weighted
  health score per device-and-backend from real measured frame time, not a guess at
  build time. Use a sliding window or a discounted average exactly like the 200-
  transaction window and 0.6 discount in Razorpay's bandit, so a backend that starts
  dropping frames (thermal throttle, a backgrounded tab losing GPU priority) gets
  demoted within seconds and the runtime drops to a cheaper quality tier or a simpler
  variant automatically. Add a small exploration bonus so you occasionally retest the
  higher-quality variant and climb back up when the device cools down.
- **Cascade.** When a frame budget is blown (the equivalent of a soft decline), fall
  forward to a simpler shader instead of janking. The user keeps a smooth picture,
  the way Rohit kept a working payment.

And steal the cost split that makes it scale: do the expensive learning offline
(profile the shader graph across a device matrix in CI, precompute per-tier variants
and their expected cost) and make the live decision a cheap keyed lookup of a
precomputed score plus a tiny sort over a handful of variants. The per-frame decision
must be a few microseconds, never a model run, the same way Razorpay's per-payment
decision is a memory read and not a forest evaluation on the hot path. Ship the
recovered-performance number back to the developer too, the way Optimizer ships its
performance report: "we kept you at 60 fps on 94 percent of sessions by adapting
quality per device" is the dashboard a developer reopens, and the reason they keep
the runtime in their app.

---

## Sources

- Ramya Bygari et al., Razorpay, "An AI-powered Smart Routing Solution for Payment Systems," arXiv 2111.00783 (2021): https://arxiv.org/abs/2111.00783
- Razorpay, "Maximizing Success Rate of Payment Routing using Non-stationary Bandits," arXiv 2308.01028, AI-ML Systems 2023: https://arxiv.org/abs/2308.01028
- ACM proceedings entry for the bandits paper: https://dl.acm.org/doi/10.1145/3639856.3639883
- Juspay, "A Control-Theoretic Approach to Dynamic Payment Routing for Success Rate Optimization," arXiv 2510.16735 (2025): https://arxiv.org/abs/2510.16735
- Razorpay Blog, "Achieve ~10% Payments Success Rates with Optimizer's robust AI/ML Routing": https://razorpay.com/blog/boost-payments-success-rates-with-optimizers-ai-ml-routing/
- Razorpay Blog, "Learn All About Smart Routing on Optimizer!": https://razorpay.com/blog/learn-all-about-smart-routing-on-optimizer/
- Razorpay Blog, "Razorpay Optimizer: India's first AI-powered payments router": https://razorpay.com/blog/razorpay-optimizer-indias-first-ai-powered-payments-router/
- Razorpay Blog, "Introducing the Optimizer performance report": https://razorpay.com/blog/how-do-you-know-that-your-payments-router-is-actually-creating-value-for-your-business-introducing-the-optimizer-performance-report/
- Razorpay product page, Optimizer (intelligent payments routing): https://razorpay.com/optimizer-intelligent-payments-routing/
- Earlier related work, PayU, "Stochastic Multi-path Routing Problem with Non-stationary Rewards" (WWW 2018): https://dl.acm.org/doi/fullHtml/10.1145/3184558.3191630
