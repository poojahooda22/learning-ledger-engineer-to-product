# Zomato push notifications and the home-screen nudge: the tap on the shoulder that makes you hungry

Date: 2026-09-30
Product: Zomato
Feature: Push notifications and the personalized nudge engine (the "Craving biryani? 🍗" that lands at 8:41pm, the meal-time reminders, the rainy-day and cricket-day nudges, and the machinery that decides who gets pinged, with what, at exactly what minute, without getting the app uninstalled)

A note on sourcing up front, because this feature has three layers and they
are documented very differently.

The bottom layer, how a push notification physically reaches a phone that has
the app closed (Apple's APNs and Google's FCM, device tokens, the single OS
socket), is public and precise. I treat it as confirmed fact.

The middle layer, how a food-delivery company builds a communication platform
that can send tens of millions of these reliably, is documented in real
engineering writing, but Swiggy's, not Zomato's. Swiggy published a full
teardown of its communication platform "Kabootar" and a series on "Smart Push
Notifications" using multi-armed bandits. Uber published its real-time push
platform "RAMEN", its "Consumer Communication Gateway", and a detailed post on
using machine learning plus linear programming to pick the send time. These are
Zomato's direct competitors solving the identical problem at the identical
scale, so I use them as the well-grounded "this is how this class of problem is
solved" reference, and I label it as such every time.

The top layer, Zomato's own targeting signals and its famously witty copy, is
documented at the marketing and behaviour level (which signals it uses, the
meme-style tone, the cricket and weather triggers). I treat that as confirmed
for what it claims, and I clearly mark anything about Zomato's internal
engineering as inference, because Zomato has not published it.

---

## 1. The user

It is 8:40pm on a Wednesday. Aisha is on her sofa in Indiranagar, Bangalore,
half-watching a show, phone face-down on the cushion. She is not hungry in the
loud way. She is in that soft, undecided evening state where she could cook the
leftover dal, or she could not. She has not opened any food app. She is not
thinking about dinner as a decision yet.

Her phone buzzes once. She flips it over. On the lock screen:

> Aisha, your Meghana biryani is getting lonely 🍗 Order in 2 taps?

She smiles at the line, taps it, and forty minutes later a Meghana Foods biryani
is at her door. She never "decided" to order. Something decided for her, at the
exact minute she was most persuadable, with the exact restaurant she orders from
most.

The other person in this story is the growth team at Zomato. They have hundreds
of millions of installed apps, most of them closed most of the time. Every one
of those closed apps is a door they cannot knock on, except through one narrow
slot: the push notification. They want to knock on Aisha's door at 8:40pm, not
at 3pm when she is in a meeting, and not five times tonight, because the fifth
knock gets the door slammed (the app uninstalled) forever.

Both sides want the same thing from opposite ends: the right tap on the
shoulder, at the right minute, no more than that.

---

## 2. The real problem

A food app makes money only when it is open. But an app spends almost its whole
life closed. Aisha opens Zomato maybe four times a week. The other roughly 160
waking hours a week, the app is a dead icon on a home screen.

So the company has a retrieval problem in the most literal sense. It has to
reach into a closed app and pull the user back. The only tool for that is the
push notification, and the push notification is a loaded gun. Get it right and
you manufacture an order out of thin air, like Aisha's biryani. Get it wrong and
you do real, permanent damage.

Here is the wrong, described like a friend would. You get a "50% OFF" push at
11am when you just ate. You get one at 3pm for a restaurant you would never
order from. You get three in one evening. By the fourth you long-press the app,
tap "Turn off notifications", and now Zomato has lost its only slot into your
closed phone forever. The numbers on this are brutal and real: by one widely
cited study, 46% of users opt out of notifications after getting just 2 to 5 in
a single week, and sending even one push a week already makes about 10% of users
disable notifications and 6% uninstall the app. Users getting 6 or more a week
uninstall at 3.4 times the rate of those getting 2 to 5.

So the real problem is not "how do we send a notification". Any intern can send
a notification. The real problem is: out of hundreds of millions of people, and
thousands of things we could say, and 1,440 minutes in the day, choose exactly
who to ping, with exactly what, at exactly which minute, such that the ping
earns an order and does not cost you the user. And then physically deliver tens
of millions of those to closed phones in a narrow dinner window without falling
over. That is two hard problems wearing one trench coat.

---

## 3. The feature in one sentence

A system that reaches into a mostly-closed app and taps the right user on the
shoulder, with a message tuned to what they eat and when they are weakest, at
the single best minute, capped so it never becomes noise, and delivered
reliably to a closed phone through Apple's and Google's push pipes.

---

## 4. Jobs to be done

What is Aisha really hiring this notification to do, even though she would never
say so?

- "Decide dinner for me when I am too tired to decide." The push removes the
  cold-start of choosing. It names the restaurant she already trusts.
- "Remind me that the easy option exists." She knows she can order. The buzz
  moves that fact from the back of her mind to the front, at the hungry hour.
- "Make me feel seen, not spammed." The line "your Meghana biryani" lands
  because it is true. A generic "Hungry? Order now" would not have earned the
  tap.

What is the company hiring it to do?

- Convert a closed app into an open session (activation of a dormant user).
- Build the reflex that the evening hunger cue equals opening Zomato
  (retention).
- Do both without tripping the uninstall wire.

Two very different users, one tap.

---

## 5. How it works for the user

From Aisha's seat it is almost nothing. A single line of text appears on her
lock screen, sometimes with a tiny food image or an emoji, sometimes with two
buttons ("Order again", "Not now"). She taps the text. The app does not open to
the generic home screen. It opens straight to Meghana Foods, or straight to a
"Reorder your last meal" card, because the notification carried a deep link that
says exactly where to land.

The messages have a personality. This is a real, confirmed part of Zomato's
strategy: the copy is witty, often riffs on a cricket match that is happening
right then, on the weather, on a meme. "Rain + biryani = 😍" on a wet Bangalore
evening. A nudge tied to an India match wicket. The tone is the point: it reads
like a friend texting, not a brand blasting.

And the nudges that are not notifications, the ones inside the app: the
home-screen category strip that reshuffles to put "Biryani" first for Aisha, the
festival banner that animates during Diwali, the streak or reward nudge. These
are the same idea (a tap on the shoulder) delivered through a different door
(the app is already open).

---

## 6. The actual flow, step by step

Walk one real nudge, the 8:40pm biryani ping, end to end.

1. Long before tonight, Aisha's phone registered with the app. On first launch
   she tapped "Allow notifications". iOS handed the app an APNs device token,
   Android handed it an FCM registration token. The app sent that token to
   Zomato's servers. That token is Aisha's mailbox address. Without it, she is
   unreachable.

2. Tonight, a background job (a campaign) wants to nudge "lapsed-ish biryani
   lovers in Bangalore at dinner". It asks the targeting system for the set of
   users who match: city Bangalore, ordered biryani in the last 30 days, have
   not ordered in the last 3 days, notifications enabled. The system returns a
   candidate set, say 400,000 user ids.

3. For each candidate, the decision layer checks: has this user already hit
   their daily nudge cap? When is this user most likely to open (their personal
   best minute)? What is the single best thing to say to them? For Aisha the
   answers are: not capped yet, best minute is around 8:40pm, best message is
   her Meghana reorder.

4. The message is built. A template, "{name}, your {restaurant} biryani is
   getting lonely 🍗", is filled with Aisha's name and her top restaurant. A
   deep link to Meghana's page is attached. A tracking id is embedded so opens
   can be attributed back.

5. The built message is dropped onto a queue, not sent inline. A fleet of
   workers pulls from the queue.

6. A worker looks at Aisha's token, sees it is an iOS APNs token, and hands the
   payload to Apple's APNs (Android tokens go to Google's FCM). Zomato's server
   never talks to Aisha's phone directly. It cannot. It hands the message to
   Apple.

7. Apple's APNs has the one persistent socket to Aisha's iPhone. It pushes the
   payload down that socket. The phone wakes the notification, shows the line on
   the lock screen. Total time from queue to lock screen: a second or two.

8. Aisha taps. The deep link opens Meghana's page. The tracking id fires an
   "opened" event. The state of her message moves from sent, to delivered, to
   opened. That opened event is gold: it trains the model that 8:40pm and the
   Meghana line worked for her.

Two clean halves already visible here. Steps 2 to 4 are the decision (who, what,
when). Steps 5 to 7 are the delivery (get the bytes to the phone). They are as
different as matching and ranking are in search, and mixing them up is the
classic mistake.

---

## 7. Under the hood, like the engineer

The whole feature splits into two halves that fail for different reasons and
scale for different reasons. Keep them apart or you will build something that is
either dumb or fragile.

- The DECISION half: out of hundreds of millions of users and thousands of
  possible messages and 1,440 minutes, pick the (user, message, minute) triples
  worth sending. This is a relevance and optimization problem.
- The DELIVERY half: take the chosen messages and physically land them on closed
  phones, reliably, at huge and spiky volume. This is a distributed-systems
  problem.

### The addressing problem, and why a phone is not a server

First, the hard constraint that shapes everything. You cannot open a socket to a
closed app. iOS and Android kill background sockets to save battery. Each device
instead keeps exactly one persistent, battery-cheap connection open to its
vendor's gateway: APNs for Apple, FCM for Google. That connection is idle almost
always and wakes only when the gateway pushes something.

So to reach Aisha you must go through Apple. The chain is always: your backend,
then APNs or FCM, then the one OS socket, then the app. The device token is the
address Apple needs. This single fact is why you need a token registry, why you
cannot just "call the phone", and why dead tokens are a real cost (more on that
below).

### Data structures in the decision half, and why those shapes

The token registry. A key-value store mapping user_id to the set of that user's
device tokens and their platform. Aisha with an iPhone and an iPad has two
tokens. Lookup must be O(1) by user_id, because at send time you will do this
hundreds of thousands of times in a few minutes. A hash-map-shaped store
(Redis, a wide-column table keyed by user_id) is exactly right. A relational
join per user would die here.

The segment index. "Biryani lovers in Bangalore" needs to be fetched fast. The
shape that makes this cheap is the same one search uses: an inverted index.
Instead of scanning 200 million user rows and testing each, you keep lists keyed
by attribute: city:bangalore points to a list of user_ids, cuisine:biryani
points to a list, active_last_7d points to a list. The candidate set is the
intersection of those lists. This is matching, pure and simple, and it is
identical in spirit to fetching candidates from a catalog of millions in the
Amazon search teardown (2026-06-23): never scan the whole corpus, intersect
precomputed lists.

The per-user feature row. For each user, a compact record: last order time, a
cuisine-preference vector, home city, and crucially a 24-bucket array of "open
probability by hour". Aisha's array might peak at bucket 20 and 21 (8pm to
10pm). This is precomputed offline from her history, so at send time picking her
best minute is an argmax over 24 numbers, not a live computation. This is the
offline-think, online-lookup spine that runs through this whole ledger
(Discover Weekly 2026-06-13, Uber upfront fares 2026-09-13): do the expensive
thinking in a batch job, make the live path a lookup.

The frequency-cap counter. Per user, a small rolling-window counter of how many
nudges they got today and this week, with a TTL. Before any send, check it. This
is the single most important safety structure in the system, because it stands
between you and the uninstall statistics above.

### Ranking and timing: which nudge, which minute

Once you have Aisha in the candidate set, two questions remain: what to say, and
when. This is the ranking half, and it is where the real machine learning lives.

The confirmed Zomato layer: Zomato builds its nudges on four signals per user,
cuisine preferences, amount typically spent, time of ordering, and browsing
history. So the "what" for Aisha leans biryani, mid-range price, evening. That
is real and published at the strategy level.

The well-grounded engineering version, from Uber's published work on the exact
same problem (labeled as the class reference, not Zomato internals): Uber trains
an XGBoost model that predicts, for a given (push, time) pair, the probability
the user places an order within 24 hours of receiving that push at that time.
Now every candidate message at every candidate minute has a score. Picking the
schedule becomes an assignment problem: assign pushes to time slots to maximize
the total score. That is solved with an integer linear program. The beauty of
the linear program is that the business rules drop in as constraints:

- a daily frequency cap (for example at most 2 pushes per day),
- a minimum gap between pushes (for example 8 hours apart),
- a send window (only during waking or meal hours),
- an expiry (a lunch offer is worthless at 4pm),
- real-world validity (do not push a restaurant that is closed right now).

So "send Aisha the Meghana biryani line at 8:40pm and nothing else today" is not
a guess. It is the solution to a scored optimization with hard safety
constraints baked in. Swiggy describes the same idea from a different angle:
its Smart Push system uses multi-armed bandits to build an optimized weekly plan
of notifications per user under constraints, balancing trying new messages
(exploration) against sending the known winners (exploitation). Both companies
independently landed on the same shape: score the options, then optimize a
constrained schedule per user. I infer Zomato does something in this family; it
would be strange if it did not, given the identical problem and the published
playbooks next door.

### The delivery half: Kabootar, a real blueprint

Now the messages are chosen. Getting them out is a distributed-systems problem,
and here Swiggy's published platform "Kabootar" is the concrete blueprint
(competitor, same scale, same job; I use it as the class reference).

Kabootar treats each communication as a self-contained MESSAGE object that
carries everything needed to send it. Because the object is self-sufficient, any
node in the cluster can process any message, which is what lets the system scale
out horizontally. The message flows through a pipeline of microservices:

1. An Event service receives raw events (an order shipped, a campaign fired),
   classifies them, and enriches them with user info.
2. A Campaign service applies campaign context and can group many messages under
   one campaign.
3. A Templating service renders the template (fills in Aisha's name and
   restaurant) and embeds tracking ids.
4. A Router service decides the channel (push, SMS, email) and pushes the
   message to the right gateway (APNs, FCM, an SMS provider).

Two design choices in Kabootar matter most for reliability. First, each message
has a state machine that is continuously persisted to storage. A message sits in
states like created, rendered, routed, sent, delivered. If a worker crashes
mid-flight, any message left in a non-final state is simply picked up again and
handed to the right processor. Nothing is lost because nothing lives only in a
worker's memory. Second, transactional messages (your order is out for delivery)
and marketing messages (biryani is lonely) ride the same platform but at
different priorities, so a flood of marketing pushes can never delay the "your
rider is here" message that the user actually needs.

This is the same forward-only, persist-the-state instinct as Stripe webhooks
(2026-07-31) and idempotency (2026-06-20): treat delivery as a state machine you
can resume, not a fire-and-pray call.

### The real-time lane: a different animal

The 8:40pm biryani nudge is batch. But "your rider has arrived" or "wicket! 20%
off for the next 30 minutes" is real-time, and needs a different path. Uber built
exactly this and published it: RAMEN (Realtime Asynchronous MEssaging Network).
The pieces, as a class reference:

- A service called Fireball whose entire job is to answer "should we push this,
  and to whom, right now?" by listening to the event firehose.
- A connection layer (Streamgate, built on Netty) that holds millions of live
  connections to apps that are currently open, so updates stream instantly
  without going through APNs or FCM.

The scale numbers Uber published are the useful part. The first generation was
Node.js with Ringpop (their consistent-hashing library), connections sharded by
user UUID, Redis for persistence. It hit a wall around 30,000 requests per
second and 1.5 million concurrent connections: the gossip protocol that kept the
cluster in sync got slow to converge, and the single-threaded Node event loop
stalled during garbage-collection pauses. The rebuild (early 2017) moved to
Netty with Apache Zookeeper, Apache Helix (which manages sharding and rebalances
those 1.5 million connections across the cluster when a node joins or dies),
Redis, and Cassandra. Concurrent connections scaled from 1.5 million to 15.5
million, pushing on the order of 70,000 messages per second at peak. The lesson
buried in that story is that the bottleneck was never raw bandwidth; it was
cluster coordination and GC, the unglamorous middle layer.

### The scale story, three tiers

What grows here is not a catalog. It is the number of sends and, worse, how
spiky they are, because everyone gets hungry at roughly the same hour.

Tier 1, about 1,000 users. A nightly cron loops the user list, calls FCM once
per user, writes a row per send to one Postgres table. No queue, no state
machine, no ML. Send time is a fixed 8pm for everyone. This is correct and
shipping anything fancier is a waste. The whole thing is 200 lines.

Tier 2, about 100,000 users. The synchronous loop breaks. Each FCM call has real
latency and some time out; looping 100,000 of them serially does not finish
before the dinner window closes, so half your users get the biryani nudge at
10:30pm when they have already eaten. Three fixes, and they are exactly the
Kabootar shape. First, decouple: drop chosen messages onto a queue (Kafka) and
run a pool of workers that fan out in parallel, and batch the gateway calls (FCM
lets you send to up to 500 tokens in one multicast call, so 100,000 sends become
200 calls, not 100,000). Second, persist each message's state so a crashed
worker's in-flight messages get re-picked instead of lost. Third, put the
per-user feature rows and token registry on read replicas so the send does not
hammer the primary. This tier is where the platform architecture earns its keep,
and where the user first notices the difference (the nudge lands at their hour,
not whenever the loop reached them).

Tier 3, 10 million and up (Zomato and Swiggy both operate at hundreds of
millions of installs, with festival evenings firing tens of millions of pushes
in a narrow window). Four new walls appear, and each has a known survival move:

- Sharding. The token registry and feature store cannot live on one database
  when tens of millions of lookups must happen inside a 30-minute send window.
  Shard by user_id so the load spreads, the same shard-by-tenant trick used for
  Notion workspaces and the Stripe ledger.
- The thundering herd. The demand itself is synchronized: everyone is hungry at
  8pm. If you send all 10 million dinner nudges at 8:00:00 sharp, you hammer your
  own workers and you hammer APNs and FCM at once. The elegant fix is that
  send-time optimization doubles as load spreading. Because each user has their
  own best minute (Aisha at 8:40, someone else at 7:55, another at 9:10), the
  sends naturally smear across the window instead of spiking. What was a
  personalization feature is also a capacity-planning feature. On top of that
  you rate-limit per gateway and apply backpressure when a gateway pushes back.
- Global frequency capping across campaigns. This is the subtle killer. At scale
  you have many campaigns running at once (a biryani campaign, a reactivation
  campaign, a cricket campaign). Each one, checking its own local cap, might
  decide Aisha is fair game. Nobody is counting the total, so Aisha gets 9
  pushes tonight from 9 campaigns and uninstalls. The fix is a centralized
  gatekeeper that owns the per-user budget across every campaign. Uber built
  exactly this and named it the Consumer Communication Gateway (CCG), a single
  intelligence layer that controls the relevance, ordering, timing, and
  frequency of pushes at the user level. Without it, the frequency cap is a lie.
- Dead tokens. At 100 million tokens, millions go stale every week (app deleted,
  token rotated). If you do not prune them, you spend your whole send budget and
  your gateway rate limit pushing into the void. FCM and APNs tell you when a
  token is unregistered; you must consume that signal and delete the token, or
  the ghosts slowly eat the system.

The through-line across tiers is the one this ledger keeps finding: push the
expensive, whole-population thinking (segmenting, scoring every user, solving
each user's schedule) into offline batch jobs, and keep the live send path a
cheap lookup and a queue drain. The decision half thinks slowly and in advance;
the delivery half acts fast and dumb.

---

## 8. The retention and habit mechanic

This feature is a habit machine, and it is worth naming the loop precisely.

The loop is cue, routine, reward. The cue is the buzz at 8:40pm. The routine is
open, tap, order. The reward is food at the door forty minutes later. Run that
loop enough Wednesdays and the cue stops being the push; the cue becomes the
8:40pm hunger itself, and Aisha opens Zomato on her own. That is the real prize.
The notification is training wheels for a reflex. Once the reflex is built, the
evening hunger opens the app without any push at all.

Which metric does it move? Primarily retention and order frequency, with revenue
right behind. Airship's retention research makes the counterintuitive point that
one of the biggest reasons new users churn is not getting push notifications at
all: the user who opts out never gets re-engaged and quietly disappears. So the
push is not just an upsell; it is the lifeline that keeps a dormant user from
going fully dark.

A real, observed example of the mechanic beyond the meal nudge: the in-app
nudges that make the app feel alive. Zomato (and Swiggy) rotate festival
animations and reshuffle the home-screen category strip so the app looks
different on Diwali than on a normal Tuesday, and the witty, timely copy
(biryani-and-rain, cricket-day lines) makes each open feel like a small event.
This is the same retention instinct, delivered through the already-open door
instead of the closed one.

But the mechanic has a hard ceiling made of the uninstall numbers from section
2, and that ceiling is the whole reason the ML-plus-linear-program machinery
exists. Recall: 46% opt out after 2 to 5 pushes in a week, and 6-plus a week
uninstalls at 3.4 times the rate of 2 to 5. And Airship's sharper finding is
that frequency alone is not the villain; it is the relevance-to-frequency ratio.
An irrelevant push at low frequency loses users as fast as a relevant push at
high frequency. So the job of the decision half is not "send fewer". It is
"every single push must be worth the slot". The frequency cap protects the
slot's scarcity; the relevance model protects the slot's value. Cross either
line and the retention machine flips into a churn machine, permanently, because
an uninstall or a disabled notification is a door that does not reopen.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime that lives inside many other people's
apps and sites. The structural twin of "push notifications" is the moment
Rare.lab needs to reach out to a fleet it does not control: pushing a recompiled
effect bundle, a hotfix, or a live event to thousands of embedded runtimes
running in the wild. The notification playbook maps onto that almost exactly,
and the lessons are all about scalability and performance.

1. Split decide from deliver, hard, and give them different SLAs. Zomato and
   Swiggy keep the "who gets what, when" brain separate from the "get the bytes
   there" plumbing, because they fail for different reasons. Rare.lab should do
   the same: one system decides which embedded runtimes need an update and at
   what urgency (a critical shader crash fix versus a cosmetic tweak), a separate
   delivery fleet ships it. Never let the decision logic and the fanout logic
   live in one tangled service, or a slow decision will stall an urgent
   delivery, the way a marketing flood would delay a "rider is here" message if
   Kabootar had not separated priority lanes.

2. Precompute the per-client decision offline so the hot path is a lookup.
   Aisha's best send minute is a 24-number array computed in a nightly batch, so
   the live send is an argmax, not a computation. Rare.lab should precompute, per
   device tier or per client, which compiled variant it should get (the low-end
   Android gets the cheaper shader LOD, the desktop GPU gets the full one) and
   store it as a lookup keyed by client id. When a push goes out, the delivery
   path reads the answer; it does not recompute which variant fits which GPU at
   send time. Same offline-think, online-lookup spine that runs through this
   whole ledger.

3. Keep two lanes: a persistent-connection real-time lane and a batched lane.
   Uber's RAMEN holds millions of live connections for instant updates to open
   apps, and uses APNs/FCM for closed ones. Rare.lab's runtime should hold a
   lightweight live channel (like RAMEN's Streamgate) for the rare urgent case
   (pull this broken effect right now), and a cheap batched revalidation path for
   the common case (new version available, fetch it when convenient). Do not pay
   the cost of a live connection for updates that can wait, and do not force an
   urgent fix through a slow batch.

4. Build one central budget gatekeeper, because local caps lie. The subtle
   scale killer for Zomato is many campaigns each independently deciding to ping
   Aisha until she uninstalls; the fix is Uber's CCG, a single authority that
   owns the per-user budget across all senders. Rare.lab has the exact same trap
   one level down: if five effects on a page each independently decide to
   recompile or re-fetch or fire a GPU job, no single effect is wrong, but
   together they blow the frame budget and the page stutters. Build a central
   per-frame and per-session budget authority that every effect must ask before
   spending, so the total is capped even though each requester thinks locally.
   The frame budget is Rare.lab's frequency cap.

5. Stagger the herd; let personalization double as load spreading. Zomato does
   not send 10 million dinner nudges at 8:00:00; each user's own best minute
   smears the load across the window for free. When Rare.lab publishes a new
   version to thousands of embeds, do not let them all re-fetch at 8:00:00 and
   melt the CDN. Jitter the revalidation across a window, ideally using a
   per-client signal you already have (its traffic pattern, its timezone), so the
   spread is also smart, not just random. A synchronized thundering herd is the
   default failure mode of any fanout, and the cure is the same one that makes
   the nudge feel personal.

6. Prune dead endpoints or they eat the system. Millions of stale device tokens
   waste the whole send budget if you keep pushing into the void; APNs and FCM
   tell you, and you must listen and delete. Rare.lab's equivalent is embeds that
   have gone offline, been removed from a page, or upgraded past your reach. Track
   liveness, consume the "gone" signal, and stop spending fanout and connection
   slots on ghosts, or your delivery fleet slowly fills with dead weight exactly
   when a real push needs the capacity.

One line to carry away: a push notification system is two systems, a slow careful
brain that decides who to reach and when (scored, capped, precomputed offline)
and a fast dumb pipe that reaches them reliably at spiky scale (queued, state-
machined, sharded, herd-staggered), and Rare.lab's update-and-delivery path to
its embedded runtimes should be built as those same two halves, with a central
budget cap standing in for the frequency cap so the fleet is never flooded by
many local decisions that were each individually reasonable.

---

## Sources

- APNs and FCM delivery model, device tokens, the single OS socket, closed-app
  reach: back4app glossary, "Push Notifications: APNs, FCM, Device Tokens, Web
  Push (2026)". https://www.back4app.com/glossary/push-notifications-apns-fcm/
- Swiggy Kabootar communication platform (MESSAGE object, event/campaign/
  templating/router microservices, persisted state machine, internal retry and
  fallback, transactional vs marketing priority): Swiggy Bytes, "Kabootar,
  Swiggy's Communication Platform".
  https://bytes.swiggy.com/kabootar-swiggys-communication-platform-e5a43cc25629
- Swiggy Smart Push Notifications (multi-armed bandits, optimized weekly plan per
  user under constraints): Swiggy Bytes, "Smart Push notifications (Multi-Armed
  Bandits at Swiggy, Part 4)".
  https://bytes.swiggy.com/smart-push-notifications-multi-armed-bandits-at-swiggy-part-4-f5698f2af0a6
- Uber real-time push platform RAMEN (Fireball, Streamgate/Netty, gen1 Node.js +
  Ringpop consistent hashing sharded by UUID + Redis, the 30K RPS / 1.5M
  connection wall, gen2 Netty + Zookeeper + Helix + Redis + Cassandra, 1.5M to
  15.5M connections, ~70K QPS peak): Uber Engineering, "Uber's Real-Time Push
  Platform". https://www.uber.com/us/en/blog/real-time-push-platform/
- Uber send-time optimization (XGBoost predicting P(order in 24h | push, time),
  assignment problem solved by integer linear program, constraints for frequency
  cap, min gap, send window, expiry, restaurant hours) and the Consumer
  Communication Gateway (CCG): Uber Engineering, "How Uber Optimizes the Timing
  of Push Notifications using ML and Linear Programming".
  https://www.uber.com/us/en/blog/how-uber-optimizes-push-notifications-using-ml/
- Zomato targeting signals (cuisine, spend, order time, browsing history), meal-
  time, weather, and cricket triggers, and the witty copy strategy: PushPilot,
  "Zomato and Swiggy Push Notifications: Why They Convert So Well"
  (https://pushpilot.ai/blog/zomato-swiggy-push-notification-teardown); Juno
  School, "Zomato's Push Notification Strategy"
  (https://www.junoschool.org/article/zomato-push-notification-strategy/).
- Opt-out and uninstall statistics (46% opt out at 2 to 5 per week, 32% at 6 to
  10, 1 per week causes ~10% to disable and ~6% to uninstall, 6-plus per week =
  3.4x uninstall rate): Business of Apps push notifications research
  (https://www.businessofapps.com/marketplace/push-notifications/research/push-notifications-costs/);
  Localytics figures as compiled by Mobiloud, "50+ Push Notification Statistics"
  (https://www.mobiloud.com/blog/push-notification-statistics).
- Retention research, including "not receiving push" as a major churn driver and
  the relevance-to-frequency ratio: Airship, "How Push Notifications Impact
  Mobile App Retention Rates".
  https://www.airship.com/newsroom/urban-airships-mobile-app-retention-study-for-key-industry-verticals/
