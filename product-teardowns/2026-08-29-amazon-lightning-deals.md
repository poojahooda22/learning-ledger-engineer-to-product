# Amazon Lightning Deals: the countdown, the "% claimed" bar, the waitlist, and the contested-inventory machine underneath

Date: 2026-08-29
Product: Amazon
Feature: Lightning Deals (the time-boxed flash deal with a countdown timer, a "% claimed" progress bar, a 15-minute cart hold, and a first-come waitlist)

A note on sourcing before we start. The user-facing mechanics of Lightning
Deals are confirmed from Amazon's own help pages and consistent seller
documentation: the countdown, the progress bar, the 15-minute cart hold, the
first-come waitlist. Amazon has never published an engineering deep-dive on how
the deal counter is stored or decremented under load. So the "under the hood"
section is grounded inference. It is built on the well-documented flash-sale
engineering that Flipkart (Big Billion Days), Shopee, and Alibaba (Singles Day)
have written about publicly, plus the standard patterns for contested inventory.
Where I am inferring, I say so in plain words.

---

## 1. The user, and what they are doing when they hit this

Meet Anjali. It is 2:10 in the afternoon on a Tuesday. She has wanted a Fire TV
Stick 4K for two months but never at the right price. She is half-working,
half-scrolling, and she opens the Amazon app out of habit. On the "Today's
Deals" tab there is a red tile: Fire TV Stick 4K, was 5,999 rupees, now 3,499,
a Lightning Deal. Below the price is a thin bar that says "68% claimed" and a
clock ticking down: 2 hours, 14 minutes, 09 seconds. 08. 07.

Anjali is not a bargain hunter by trade. She is a normal person who got a small
jolt. Two feelings arrive at once. One: this is cheaper than she has ever seen
it. Two: the bar is almost full and the clock is moving. If she waits until
tonight to "think about it," the deal may be gone. She is not deciding whether
she wants the product anymore. That was decided months ago. She is deciding
whether to act in the next few minutes.

Now multiply Anjali by a few hundred thousand people, all looking at the same
tile, during a Prime Day or a Great Indian Festival sale, all within the same
sixty seconds. That crowd, pointed at one product with a fixed number of units,
is the entire engineering story.

---

## 2. The real problem, described like a friend would

There are two different people with two different pains here, and the feature
solves both at once. That is why it exists.

The shopper's pain is not price. It is doubt. "Is this actually a good deal or
is the 'was' price fake? Will it drop more next week? Should I wait?" This doubt
is what kills purchases. People put the thing in a cart and never come back. The
Lightning Deal removes the doubt by removing the option to wait. A real ticking
clock and a real depleting bar say: this window is closing, and other people are
taking the units, so your choice is now, not later. That is not a trick pop-up.
The clock and the count are true. That truth is what makes the nudge work.

The seller's pain is different. A seller sitting on 700 units of last season's
model wants a short, sharp burst of sales, a rush that also lifts the product's
rank so it keeps selling after the deal. A slow trickle of discounts does not do
that. A concentrated burst does. Amazon's pain is a third one: it wants that
burst to feel exciting without the site falling over when a hundred thousand
people tap "Add to Cart" on the same item in the same minute, and without ever
selling the 701st unit of a 700-unit deal.

So the feature has to do something quietly hard. It has to let a huge crowd
contend for a small, fixed pile of units, in real time, and be exactly right
about the count, and stay fast, and never oversell. That is the whole game.

---

## 3. The feature in one sentence

A Lightning Deal is a discount on a fixed number of units of one product for a
short window (4 to 12 hours, usually about 6), shown with a live countdown and a
"% claimed" progress bar, where adding it to your cart reserves a unit for 15
minutes, and when units run out you can join a first-come waitlist for reserved
units that get released.

---

## 4. Jobs to be done

What is Anjali really hiring this feature to do?

- "Tell me honestly that this is a good time to buy, so I can stop second-
  guessing." The bar and the clock are a trust signal, not just pressure.
- "Give me permission to act now instead of adding to a wishlist I will never
  reopen." The deadline converts intent into action.
- "Hold my unit while I find my card and my address, so I do not lose it at the
  last step." The 15-minute cart hold is the job here.
- "If I just missed it, give me a fair second chance instead of a dead end." The
  waitlist.

What the seller is hiring it to do: "Turn my excess stock into a rank-boosting
sales spike inside a known window, and let me predict roughly how many I will
sell." What Amazon is hiring it to do: "Make the storefront feel alive and
urgent every single day, and capture the buyer at the exact moment their
motivation peaks."

---

## 5. How it works for the user

Anjali sees the deal tile. It has four visible parts: the discounted price with
the struck-through original, a countdown timer, a progress bar labeled with a
percentage claimed, and a button. If units are still available the button says
"Add to Cart." She taps it. The unit is now hers to buy for the next 15 minutes,
and a small timer often appears in the cart reminding her. She checks out. Done.

If she had arrived at "100% claimed," the button would instead say "Join
Waitlist," and only if the waitlist itself still had room. She taps it and waits.
Somewhere ahead of her, someone who claimed a unit does not check out within
their 15 minutes. That unit is released back into the pool. The system walks the
waitlist in order, reaches Anjali, and sends her a notification: the deal is
available, add it to your cart within a short time limit. If she acts, she gets
it. If she ignores the notification past the limit, she is dropped and the unit
goes to the next person in line. When the promotion's clock hits zero, the deal
ends and everyone still on the waitlist loses the chance. The price returns to
normal.

The important thing about this experience: every number Anjali sees is real. The
"68% claimed" is a real count of real reservations. The clock is a real deadline.
Nothing here is theater. That honesty is load-bearing.

---

## 6. The actual flow, tap by tap

1. Anjali opens the app and lands on "Today's Deals." Amazon shows her a grid of
   deal tiles. The Fire TV Stick tile shows 3,499 rupees, a struck-out 5,999, a
   bar reading "68% claimed," and a countdown at 2h 14m.
2. She taps the tile. The product page loads with the same deal badge and timer.
   The button reads "Add to Cart."
3. She taps "Add to Cart." Behind that single tap, a unit is reserved for her.
   The progress bar for everyone else ticks up (say from 68% to 69%). Her cart
   now shows the item at the deal price with a note that she has about 15 minutes
   to check out.
4. She taps through to checkout, confirms her saved address, picks UPI, pays.
   The reservation converts to a confirmed order. The unit count is permanently
   spent.
5. Alternative branch: she got distracted for 16 minutes. Her reservation
   expired. The item is still in the cart list but no longer at the deal price,
   or it is removed from the deal. The unit she was holding went back into the
   pool for the next person.
6. Alternative branch: she arrived late and the bar read "100% claimed." Button
   says "Join Waitlist." She joins. Later she gets a push notification: "A Fire
   TV Stick 4K Lightning Deal is available." She has a short window to add it to
   her cart before her turn passes to the next person.

The two timers matter and are different. One is the deal's own countdown (hours),
the same for everybody. The other is Anjali's personal 15-minute hold, private
to her, that starts the instant she claims. Keeping these two clocks straight is
part of the engineering.

---

## 7. Under the hood, like the engineer

This is the heart of the report. Remember the honesty flag: Amazon has not
published how Lightning Deals are built internally. What follows is the standard
way this exact class of problem is solved, drawn from Flipkart's Big Billion
Days write-ups, Shopee's flash-sale engineering, and Alibaba's Singles Day
posts, mapped onto what Lightning Deals visibly do. I label the confirmed parts
and the inferred parts.

### The one hard requirement

Strip the feature down and it is this: a shared integer, "units remaining,"
starting at some number like 700, that a large crowd decrements concurrently,
and it must never go below zero, and reads of it (the "% claimed" bar) must be
cheap and roughly live for everyone. That is a contested counter. Everything
else is scaffolding around getting that counter right under load.

Two different jobs live inside it, and it helps to name them separately, the
same way search has matching and ranking as two halves:

- The write path: claiming a unit. Rare, sacred, must be exactly correct. This
  is the atomic decrement.
- The read path: showing the bar and the timer. Constant, enormous in volume,
  and allowed to be a little stale. This is a cached lookup.

Confusing these two is the classic mistake. If you serve the "% claimed" bar by
hitting the source-of-truth counter on every page view, you melt the counter
with reads before a single purchase happens. So you split them hard.

### The naive version, and exactly why it dies

The obvious first build: a row in a SQL database, `deal_stock`, one column
`remaining`. To claim a unit you run `UPDATE deal_stock SET remaining =
remaining - 1 WHERE deal_id = 'firetv' AND remaining > 0`. The `WHERE remaining
> 0` guard is what stops overselling. If the update affects one row, you got a
unit. If it affects zero rows, you are sold out.

This is correct. It is also a single hot row. When a hundred thousand people
tap "Add to Cart" in the same minute, they all want to write to the same row of
the same table at the same time. The database serializes them with row locks:
each writer waits for the previous one to commit before it can touch the row.
The row becomes a single-file turnstile. At a few hundred contending writers per
second the queue behind that lock explodes, transactions time out, and the whole
deal page starts throwing errors. The count stays correct but the site is down.
This is the "single hot row" or "hot key" problem, and it is the wall you hit
somewhere between the 100,000-item tier and the 10-million tier below.

### Move the counter to memory and make the decrement atomic

The fix everyone converges on: hold the live "remaining" count in an in-memory
store like Redis, not in the SQL database, on the hot path. Redis does one
operation at a time on a single key, so a decrement is naturally atomic. But a
plain read-then-write still races: two clients both read `remaining = 1`, both
decide "there is stock," both decrement, and you just sold unit 701. The fix is
to make check-and-decrement a single indivisible step.

Redis gives you that with a small Lua script run via `EVAL`. Redis runs each Lua
script atomically, start to finish, with nothing else interleaved on that key.
The script is tiny in spirit:

```
if redis.call('GET', stock_key) > 0 then
    redis.call('DECR', stock_key)
    return 1        -- you got a unit
else
    return 0        -- sold out
end
```

Now the check and the decrement happen together or not at all. No two clients
can both see the last unit. This is the confirmed canonical pattern in every
serious flash-sale write-up (Flipkart, Shopee, and the widely-cited flash-sale
system-design articles all describe exactly this Lua check-and-decrement). It is
inference that Amazon uses precisely this, but the shape of the problem forces
something equivalent: an atomic compare-and-decrement in fast memory.

### The reservation, not a sale: the 15-minute hold

Here is the subtlety that makes Lightning Deals more than a counter. Tapping
"Add to Cart" does not sell you the unit. It reserves it for 15 minutes while
you find your card. So the decrement is really a reservation, and reservations
can expire.

The clean way to build a self-expiring reservation is a key with a TTL (time to
live). When Anjali claims, you write something like
`reservation:firetv:anjali` with a 15-minute TTL, and you decrement the pool.
If she checks out, you convert the reservation into a confirmed order and the
unit is gone for good. If she vanishes, Redis deletes the reservation key when
the TTL fires, and a listener puts the unit back by incrementing the pool. No
human, no cron sweep needed for the common case: the expiry is baked into the
data. This is the same self-releasing-hold trick used for movie seat locks
(covered in the 2026-08-21 District ticketing teardown) and for warehouse holds.
Flipkart's public design uses a 5-minute reservation TTL with the same idea;
Amazon's is 15 minutes, confirmed from the user-facing behavior.

This is why the "% claimed" bar can go down as well as up. An abandoned cart
releases its unit and the bar ticks back. The count is a live tally of active
reservations plus confirmed sales, not a one-way ratchet.

### The read path: the bar and the clock are a cheap cached lookup

The "68% claimed" bar and the countdown are shown to everyone who so much as
glances at the deal. That is orders of magnitude more traffic than actual
claims. You never serve that from the sacred write counter. You serve it from a
cache, refreshed every few seconds, fanned out through Amazon's CDN and edge
caches. If Anjali sees "68%" when the true number is "69%," nothing breaks. A
one-second-stale bar is fine. A wrong final unit sold is not. So the read path
is deliberately allowed to be slightly stale and made very cheap; the write path
is kept exactly correct and rare. This read/write split is the same spine as
half this ledger: keep the expensive-to-be-correct thing small and rare, serve
the high-volume thing from a cache.

The countdown itself is not streamed from the server tick by tick. The server
sends one deal end-timestamp; the phone counts down locally against it. That is
why the seconds animate smoothly even on a weak connection: it is just the
device's own clock subtracting from a fixed future time. The server is the
source of truth for when the deal ends, not for each visible second.

### The waitlist is a queue for fairness

When the pool hits zero, new arrivals cannot claim. But units keep leaking back
in as 15-minute holds expire. Handing those released units out fairly is a
classic queue problem, and Amazon solves it the classic way: a first-in-first-
out waitlist (confirmed first-come-first-served in Amazon's help docs). Join the
waitlist and you take a numbered spot at the back. When a unit is released, the
system pops the head of the queue, notifies that person, and gives them a short
window to add it to their cart. Miss the window, you are dropped and the next
person is popped. When the deal's own clock ends, the queue is discarded.

A queue here is doing what a queue does everywhere: it turns a chaotic scramble
for a scarce released unit into an orderly line, so the fastest bot does not win
every crumb. It also smooths load, because waitlisted users are not hammering
"Add to Cart" in a loop; they are parked, waiting to be called.

### The async twist for the very top tier: take the request, answer later

At Prime-Day scale the claim itself can be too spiky to handle synchronously. A
common pattern (documented by Flipkart and Shopee, inferred for Amazon at peak)
is to not make the user wait for the full order to be written. You accept the
tap, do the fast atomic decrement in Redis, immediately return a "you got it,
completing your order" response (an HTTP 202 Accepted in spirit), and drop the
heavier work (writing the order row, charging, updating the seller) onto a
durable message queue that worker pools drain at the database's own comfortable
pace. Poison messages fall to a dead-letter queue. This decouples the sharp
spike of taps from the steadier speed of the order database. The user feels an
instant yes; the paperwork settles behind the scenes.

### Data structures in play, and why each one

- A single integer counter per deal, held in memory (Redis), for "units
  remaining." Chosen because the whole feature reduces to one contended number,
  and an in-memory integer with atomic ops is the cheapest correct home for it.
- Short-lived keys with TTLs for the 15-minute reservations. Chosen because
  "hold this, auto-release if not confirmed" is exactly what a TTL expresses,
  with no sweeper needed for the common path.
- A FIFO queue for the waitlist. Chosen because "fair second chance in arrival
  order" is the literal definition of a first-in-first-out queue.
- A durable message queue for order processing at peak. Chosen because it
  absorbs a spike the order database cannot, and gives retries for free.
- A hash-map-style cache, fanned out to the edge, for the "% claimed" and timer
  reads. Chosen because reads dwarf writes and tolerate slight staleness.

### The scale story at three tiers

Walk the same Fire TV Stick deal through three sizes.

Tier one, 1,000 shoppers, 700 units. Almost anything works. A single SQL row
with `UPDATE ... WHERE remaining > 0` is correct and fast enough. A few hundred
claims spread over six hours is a trickle. Do not over-engineer. The hot-row
lock never gets hot because writers rarely collide. You could ship this on a
laptop.

Tier two, 100,000 shoppers in the first minute, 700 units. Now the single row is
a problem. A hundred thousand taps aimed at one row in sixty seconds means
thousands of writers per second queued behind one row lock; timeouts begin. This
is the tier where you move the counter into Redis and make the decrement an
atomic Lua check-and-decrement, and you split the read path off into a cache so
the "% claimed" bar (viewed by far more than 100,000 people) never touches the
counter. Reservations become TTL keys so abandoned carts self-heal. What broke
at this tier: the single hot row under write contention. What saved it: move to
in-memory atomic decrement plus a cached read path.

Tier three, millions of shoppers on a Prime Day, one blockbuster deal, still a
few hundred to a few thousand units. Now even a single Redis key is a hot key:
every claim in the world for this deal hits the same key on the same Redis node,
and that one node's CPU and network become the ceiling. The move (confirmed as
the standard solution across Flipkart, Shopee, Alibaba, and the flash-sale
design literature) is inventory sharding: split the 700 units into, say, 10
logical buckets of 70 each, `firetv:stock:0` through `firetv:stock:9`, spread
across Redis nodes. Each incoming claim picks a random bucket and decrements
that. Ten buckets means one-tenth the contention per key, and the load divides
by ten. To know if the whole deal is sold out you check whether all buckets are
empty. There is a small cost: buckets can deplete unevenly, so one bucket can
show empty while another still has a few. Real systems handle that by having a
claim that hits an empty bucket retry a different bucket a couple of times before
declaring sold out. On top of sharding you add the async 202-Accepted queue so
the order database is never the bottleneck, a virtual waiting room or rate limit
in front to shed obvious bot floods, and randomization so the "released unit
lottery" is not won by whoever polls fastest to the microsecond. What broke at
this tier: the single hot key. What saved it: shard the counter into buckets,
absorb the write spike with a queue, and throttle the crowd at the door.

Notice the pattern across tiers. The count of units barely grows (700 stays
700). What explodes is the crowd contending for it. So the scaling work is not
about storing more data; it is about spreading contention: one row becomes one
key becomes ten keys, and reads get pushed out to caches while the sacred write
gets smaller and faster.

---

## 8. The retention and habit mechanic

Lightning Deals are a habit engine wearing a discount costume. The loop is a
textbook variable-reward schedule, the same mechanism that makes slot machines
and social feeds sticky. Every day the deal grid is different. You do not know
what will be on sale, at what price, or how long it will last. So the only way to
find out is to open the app and look. That uncertainty, refreshed daily, is what
pulls the repeat open. It is the same category nudge Swiggy and Zomato use with
rotating home-screen tiles, just aimed at price instead of food.

Three specific hooks reinforce it:

- The countdown creates loss aversion. "2h 14m left" tells you inaction has a
  deadline. Fear of missing the price is a stronger motivator than desire for
  the product.
- The "% claimed" bar is social proof plus scarcity in one thin line. "68%
  claimed" says other people are validating this deal and the units are running
  out, so your window is real and shrinking.
- The waitlist keeps you engaged even after you "lose." Instead of a dead end
  that makes you close the app, it parks you in a line and can pull you back with
  a push notification, which is another reason to reopen.

Which metric does it move? Primarily activation and revenue, with a retention
tail. Activation, because it converts a lurker into a buyer at the exact moment
motivation peaks. Revenue, because it drives concentrated purchase bursts and
lifts average order value when people add "while I am here." Retention, because
the daily-changing grid manufactures a reason to reopen tomorrow. For sellers
the observed effect is a real sales-and-rank spike inside the window that often
keeps the product selling above baseline afterward, which is exactly why sellers
pay Amazon to run these deals. The habit is not the shopper's alone; the seller
is hooked on the spike too.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to
shippable code, plus an embeddable runtime. The Lightning Deal lesson is about
contended, limited resources under a burst of concurrent demand, and it maps
cleanly onto two Rare.lab situations.

The direct engineering lesson: split every contended resource into a sacred,
tiny, atomic write path and a cheap, cache-served, slightly-stale read path, and
never let reads touch the write path. In Rare.lab this shows up wherever many
clients contend for one scarce thing. Concrete example: a shared collaborative
shader graph where several editors and thousands of embedded runtime players all
read the same live parameter set, but only a few writes happen. Serve the reads
from an edge-cached snapshot refreshed every few seconds (the "% claimed" bar
model), and route the rare writes through one atomic compare-and-set on the
authoritative value (the Lua check-and-decrement model). If you ever have a
genuinely contended counter, for example a limited pool of GPU compile workers
or a rate-limited "render this on our cloud" quota during a launch spike, do not
guard it with a database row that becomes a hot row. Hold it as an atomic in-
memory counter, and when one key becomes a hot key, shard the quota into N
buckets and decrement a random bucket, accepting slightly uneven depletion for a
tenfold drop in contention. That single trick, one key becomes ten keys, is the
cheapest way to survive the next tier of load, and it is worth building the quota
system so this split is possible from day one rather than retrofitting it when
the launch-day spike is already melting one node.

The product lesson: a real, honest scarcity signal converts intent into action
better than any amount of persuasion copy. Rare.lab should surface true, live
counts where they exist (compile queue position, remaining free render minutes,
seats left in a live collab session) rather than hide them, because an accurate
"you are number 4 in the compile queue" both sets expectations and gently pushes
the paid upgrade at the moment of peak motivation, the same way the Lightning
Deal clock pushes the buy. The rule from Amazon is that the number has to be
true. A fake scarcity bar erodes trust the first time a user catches it; a real
one earns the nudge.

One line: split every contended resource into a rare atomic write and a cheap
cached read, shard the counter into buckets before one key becomes one hot node,
and show users real live scarcity numbers because honest scarcity converts far
better than persuasion.

---

## Sources

- Amazon Customer Service, "Amazon Lightning Deal Waitlist" (help page describing
  the first-come waitlist, active/deactivated Join Waitlist button, notification,
  time limit, and expiry): https://www.amazon.com/gp/help/customer/display.html?nodeId=201894810
- Amazon Customer Service, "Amazon Lightning Deals on Prime Day":
  https://www.amazon.com/gp/help/customer/display.html?nodeId=GW3L8JX7Q9FH8ALB
- SellerApp, "How Do Amazon Lightning Deals Work" (duration, countdown, progress
  bar, 15-minute cart hold, waitlist): https://www.sellerapp.com/blog/amazon-lightning-deals/
- Tinuiti, "Amazon Lightning Deals: Everything Sellers Need to Know":
  https://tinuiti.com/blog/amazon/what-are-amazon-lightning-deals/
- The Krazy Coupon Lady, "Amazon Lightning Deals" (shopper-facing % claimed and
  waitlist walkthrough): https://thekrazycouponlady.com/tips/money/amazon-lightning-deals
- Himanshu Singour, "Big Billion Sale 2025: System Design" (Flipkart flash-sale
  design: Redis stock counter, distributed lock, 5-minute reservation TTL, Kafka
  events): https://medium.com/@himanshusingour7/big-billion-sale-2025-system-design-282f84d28e8f
- Ajit Singh, "Flash Sale System Design: Architecture, Scale, and Oversell"
  (atomic Lua check-and-decrement, queue/202 pattern, cache for reads):
  https://singhajit.com/flash-sale-system-design/
- tanhdev, "Shopee Architecture Chapter 2: Flash Sale Engine, Solving Overselling
  and Hot Keys" (hot-key problem, inventory sharding into buckets, Lua atomicity):
  https://tanhdev.com/series/shopee-architecture/02-flash-sale-engine/
- Umesh Kushwaha, "Designing a Flash Sale System That Never Oversells, From 1
  User to 1 Million Users" (inventory sharding into N keys, random-bucket
  decrement): https://medium.com/@umesh382.kushwaha/designing-a-flash-sale-system-that-never-oversells-from-1-user-to-1-million-users-without-8426db0f1ad0
- Sujeet Jaiswal, "Design a Flash Sale System" (one-per-user enforcement,
  reservation TTL, message queue): https://sujeet.pro/articles/design-flash-sale-system
