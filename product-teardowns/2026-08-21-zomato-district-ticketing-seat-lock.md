# Zomato (District): movie and event ticket booking, the seat pick and the seat lock under a flash sale

Date: 2026-08-21
Product: Zomato, through its District app (movies, events, concerts, sports)
Feature: Seat selection and the seat lock that holds your chosen seats while you pay, and how that survives a flash sale

A note on scope. District is Zomato's going-out app. In August 2024 Zomato
bought Paytm's ticketing business (Paytm Insider and TicketNew) for about
2,048 crore rupees, roughly 244 million dollars, and folded it into District,
which sells movie tickets for chains like PVR Inox and Cinepolis plus concerts,
comedy shows, and IPL cricket. The market leader is still BookMyShow, which
holds roughly three quarters of online movie ticketing in India. Zomato and
District do not publish the internals of their seat engine. So the public,
provable numbers in this report come from the class leader (BookMyShow's
Coldplay sale) and from the well documented industry pattern that every
serious ticketing platform, District included, has to implement. Where I am
describing the standard solution rather than a confirmed District detail, I
say so and label it inference.

---

## 1. The user

Meet Aditya. It is a Tuesday, 11:58 AM. He has three browser tabs and the
District app open on his phone. In two minutes, tickets for Coldplay at the
DY Patil Stadium in Mumbai go on sale. He is not alone. On the real Coldplay
India onsale in September 2024, about 13 million people did exactly this at
the same minute, chasing about 178,000 tickets. That is a 1.3 percent chance
of success per person. Aditya knows the odds are bad. He still refreshes.

Or meet a calmer version of the same person. It is Friday evening, Aditya
wants two seats for the 9:45 PM show of a big new release at PVR Phoenix Mills,
he wants the aisle so he can get up for popcorn, and he wants seats H14 and
H15 specifically because row H is the sweet spot, not too close, not too far.

Both people are doing the same thing. They are trying to claim a specific,
physical, one of a kind object (seat H14 at that one show) before anyone else
on earth claims it.

## 2. The real problem

A seat is not like a product on Amazon. Amazon can sell you the same book a
million times because it just prints or ships another copy. Seat H14 at the
9:45 PM show exists exactly once. If two people pay for it, one of them shows
up to the cinema, finds a stranger sitting there, and now you have an angry
customer and a refund.

So the real pain, described like a friend would: "I picked my seat, I was
typing my card number, and the app told me the seat was gone." Or worse, the
opposite: two people both got a confirmation for H14. The first is annoying.
The second is a disaster.

Underneath that is a brutal timing problem. Between the moment Aditya taps H14
and the moment his payment clears, maybe 90 seconds pass. During those 90
seconds, is H14 available or not? If you say available, someone else grabs it
and you double book. If you say sold, and Aditya's card fails, the seat is now
frozen forever for no reason. The seat lock lives entirely inside that 90
second gap.

## 3. The feature in one sentence

When you tap a seat, the system quietly reserves it just for you for a few
minutes so you can pay in peace, and it guarantees that nobody else on the
planet can pay for that same seat during your window.

## 4. Jobs to be done

What is Aditya really hiring the seat lock to do?

- "Hold my seat while I pay." He picked H14 and H15. Do not let them slip away
  while he fumbles for his UPI PIN.
- "Never sell me a seat that is already taken." Do not show him a confirmation
  and then a stranger in his seat.
- "Do not freeze seats forever." If he changes his mind and closes the app,
  give H14 back to the next person quickly, do not leave it dead.
- "Be fair when everyone wants the same thing." On the Coldplay sale, he wants
  to feel that his place in line was honest, not that the person with the
  fastest refresh loop won.

Those four jobs pull in opposite directions. Holding a seat fights giving it
back fast. Fairness fights raw speed. The engineering is all about balancing
these.

## 5. How it works for the user

The visible experience is calm and hides all the fear.

1. Aditya opens the show. He sees a seat map, a grid of green (available),
   grey (taken), and a few colors for price tiers. Green H14 is there.
2. He taps H14 and H15. They turn to "selected" instantly.
3. He taps Pay. A timer appears. Something like "Complete payment in 4:59,
   4:58, 4:57." That countdown is the seat lock made visible.
4. He pays. Confirmation. QR code ticket. Done.

If instead he is slow, the timer hits zero, the seats are released, and he is
bounced back to the map. If instead this is a flash sale, before he even sees
the seat map he first sees a waiting room: "You are in line. 240,000 people
ahead of you." He waits, then gets pushed through to the map when it is his
turn.

## 6. The actual flow, step by step

Normal Friday show:

1. Tap show and time (9:45 PM, PVR Phoenix Mills).
2. App loads the seat map. This is a read. It is cached hard, because
   thousands of people are staring at the same map and most of them are just
   looking, not buying.
3. Tap H14, H15. The app marks them selected locally. Note: at this moment
   nothing is reserved on the server yet. This is just UI.
4. Tap Pay. Now the app sends the real request: "reserve H14 and H15 for me."
   The server tries to lock them. If it succeeds, the countdown starts. If in
   the half second since the map loaded someone else grabbed H14, the server
   says no, and the app shows "seat no longer available, pick another."
5. Payment window. The seats are held. The clock runs, commonly 5 to 10
   minutes in the industry.
6. Pay. On success the hold is converted to a permanent booking, the seats go
   grey for everyone, and Aditya gets his ticket.
7. On failure or timeout, the hold is dropped and the seats go green again.

Flash sale (Coldplay):

0. Before step 1, everyone is dumped into a virtual waiting room. Entry into
   the actual booking site is throttled. You are admitted in controlled
   batches. For the second Coldplay show in Ahmedabad, BookMyShow publicly
   switched to randomized queue position instead of pure first come first
   served, to stop the "who clicked at the exact millisecond" lottery from
   deciding everything.

## 7. Under the hood, like the engineer

This is the heart of it. Three sub problems: represent the seats, lock a seat
safely, and survive when 13 million people want the same map. Since District's
own internals are not public, I am describing the standard architecture every
platform of this class uses, grounded in BookMyShow's documented Coldplay
numbers and in the well known seat lock pattern. I label the confirmed public
facts and the inferred standard design as I go.

### The data structure: the seat map

A show is a grid. The natural structure is simple: a map (a hash map) from
seat id to seat state. Seat "H14" points to a small record: status, price
tier, and who holds it.

    seat "H14" -> { status: AVAILABLE, tier: GOLD, held_by: null, hold_expires: null }

Status is a tiny state machine with three states:

- AVAILABLE (green). Anyone can grab it.
- LOCKED or RESERVED (yellow, held). One specific user is paying. Time boxed.
- BOOKED (grey). Sold. Permanent.

The whole seat map for one show is small. A big auditorium is a few thousand
seats. The DY Patil stadium for Coldplay is far bigger, but still on the order
of tens of thousands of seats, not millions. So the entire live state of one
show fits comfortably in memory. That is the key insight that makes the fast
path possible: you do not need a heavyweight database query to know if H14 is
free, you need one in memory hash map lookup, which is O(1).

Concrete example. When Aditya loads the map, the server reads a few thousand
seat records and paints the grid. When he taps H14, the app just needs to flip
one entry.

### The lock: the actual hard part

Here is the naive version and why it breaks. You could store seats in a SQL
table and, when Aditya pays, run:

    SELECT * FROM seats WHERE seat_id = 'H14' FOR UPDATE;

`FOR UPDATE` takes a row lock so nobody else can touch H14 until Aditya's
transaction ends. This is correct. It genuinely stops double booking. And it
is fine at small scale. It falls apart under a flash sale, because a database
row lock is a held, blocking thing. If 300,000 people all target the popular
front rows at the same instant, they all queue up on the same few rows, the
database threads pile up waiting, connection pools drain, and the whole system
grinds. Row locks are the right tool at 1,000 requests and the wrong tool at
300,000 per second.

The standard fix is a two layer lock: a fast in memory lock in Redis for
speed, backed by a final correctness check in the database. This is the widely
documented BookMyShow style pattern.

Layer 1, the Redis hold. To reserve H14, the server runs one atomic Redis
command:

    SET seat:show789:H14 aditya_user_id NX PX 300000

Read that carefully, because every word earns its place:

- `NX` means "only set this if the key does not already exist." This is the
  whole game. If H14 is unheld, the key gets created and the command returns
  OK, Aditya has it. If someone already holds H14, the key exists, `NX` makes
  the command do nothing and return nil, Aditya is told to pick another. And
  because Redis runs this as one atomic operation, there is no gap where two
  people both see "unheld" and both win. One request wins, the rest lose,
  cleanly. Modern Redis (2.6.12 and later) does the set and the expiry in this
  single atomic command. The older two step SETNX then EXPIRE pattern is
  deprecated precisely because a crash between the two steps could leave a
  seat locked with no expiry, frozen forever.
- `PX 300000` means "auto delete this key after 300,000 milliseconds," that is
  5 minutes. This one flag solves the "do not freeze seats forever" job for
  free. If Aditya abandons checkout, closes the app, or his phone dies, nobody
  has to run a cleanup job to release H14. The key just evaporates after 5
  minutes and H14 goes green again on its own. No cron sweep, no reaper, no
  expiry index to scan. The time to live is the janitor.

Layer 2, the database as the final judge. Redis is fast but it is a cache, and
you should not trust your only copy of "this seat is sold" to live only in a
cache. So when Aditya's payment succeeds, the permanent booking is written
with a conditional update that is itself atomic:

    UPDATE seats SET status = 'BOOKED', booked_by = 'aditya'
    WHERE seat_id = 'H14' AND status = 'AVAILABLE';

The `AND status = 'AVAILABLE'` is optimistic locking. If somehow two payment
flows both reached this line for H14, the first one flips the row to BOOKED and
the update affects 1 row. The second one finds status is no longer AVAILABLE,
the `WHERE` matches nothing, and it affects 0 rows. The server sees "0 rows
changed" and knows it lost the race, so it refunds rather than double books.
The database, not Redis, has the final word. Redis makes the common case fast
and shields the database from the flood, the database guarantees you never
actually sell H14 twice.

So the full lifecycle of seat H14, tap by tap:

1. Aditya taps Pay. Server runs `SET ... NX PX 300000`. Returns OK. H14 is now
   yellow, held by Aditya, for 5 minutes.
2. Everyone else who tries H14 in those 5 minutes gets nil and is told to pick
   another. The seat map broadcasts H14 as unavailable.
3a. Aditya pays in time. Conditional `UPDATE` flips H14 to BOOKED in the
    database. The Redis key is deleted or left to expire. H14 is grey forever.
3b. Aditya does not pay. After 5 minutes the Redis key auto expires. H14 goes
    green. The next person can grab it.

### Matching versus committing, the two halves

Even here the familiar split shows up, though it is not search ranking. There
is a cheap wide read path (show me the map, is H14 free) that 100 people hit
for every 1 who buys, and a narrow expensive write path (actually lock and
sell H14) that must be perfectly correct. Industry write ups on this exact
system note that seat viewing traffic can be around 100 times higher than seat
booking traffic. So you design them separately. The read path is cached and
disposable and can be a little stale. The write path is atomic and authoritative
and is the only thing that must never be wrong. Keeping the huge lazy read
crowd off the tiny sacred write path is most of the battle.

### The scale story at three tiers

Tier 1, about 1,000 seats or requests. A single indie play, a small comedy
show at Canvas Laugh Club, maybe 200 seats, a trickle of buyers. Here you do
not need any of the fancy machinery. A single Postgres table with
`SELECT ... FOR UPDATE` row locking is completely fine. Contention is near
zero because two people rarely fight over the same seat in the same
millisecond. Total simplicity. This is the tier most shows on District live at
every single day.

Tier 2, about 100,000 concurrent buyers. A Friday blockbuster first day first
show, thousands of screens across the country, lakhs of people booking at
once but spread across thousands of different shows and seat maps. Now the
naive row lock starts to hurt on the popular shows, and a single database is a
bottleneck. What breaks: the one database, and hot rows on popular seats. What
you do to survive:
- Move the live lock to Redis with the `SET NX PX` pattern above, so the hot
  path is in memory, not a blocking database row lock.
- Shard by show id. Each show's seat map is independent, so you can spread
  shows across many machines and databases. Aditya's PVR show and someone
  else's Cinepolis show never touch the same data. This is embarrassingly
  parallel across shows.
- Split the read path from the write path. Serve the seat map from cache and a
  CDN, because most requests are people just looking. Only the Pay tap hits
  the real lock service.
This tier is the everyday peak. A real posted example from this class of
system: a big Hindi release selling on the order of 1 lakh tickets in an hour
across the country. Sharded by show, that load spreads out and the system
barely notices.

Tier 3, 10 million plus, all aimed at ONE map. This is the Coldplay tier, and
it is a different beast, because sharding by show does not save you. Thirteen
million people (a confirmed BookMyShow figure) all want the same handful of
shows, the same seat maps, in the same 10 minutes. Sharding by show id does
nothing when everyone is on one show id. You have a single, unavoidable hot
partition. What breaks:
- The origin melts. BookMyShow's site buckled and was widely reported to have
  crashed in the first roughly 10 minutes of the Coldplay onsale. About a
  million requests hit in the first 10 minutes.
- The thundering herd at T equals zero. Every one of 13 million clients fires
  at 12:00:00 exactly.
- Fairness collapses. First come first served becomes a millisecond lottery
  that rewards bots and fast connections.

What you do to survive tier 3, the standard playbook, confirmed in part by
BookMyShow's own later changes:

- A virtual waiting room in front of everything. Instead of letting 13 million
  requests reach the booking core, you park them in a queue and admit them in
  controlled batches at a drain rate the core can actually handle. The waiting
  room itself is cheap: at the CDN edge, a visitor without a valid "your turn"
  token gets a 302 redirect to the waiting page. Only holders of a valid token
  reach the seat map. This is exactly how dedicated systems like Queue-it and
  SeatGeek's own waiting room work, and it is what BookMyShow effectively
  runs for these onsales. The waiting room is pure admission control: it turns
  an unsurvivable 13 million wide spike into a steady, boring stream the
  booking service can serve all day.
- Queue state in a fast store, not the main database. Positions are tracked in
  something like a Redis sorted set or a DynamoDB table built for hundreds of
  thousands of writes per second, keyed by a token, never touching the seat
  database until it is actually your turn.
- Fairness by randomization. For the Ahmedabad show, BookMyShow publicly moved
  to randomized queue positions. Everyone who joins in the opening window gets
  a random spot, so being 5 milliseconds faster does not matter, which kills
  the bot advantage and feels fairer.
- Split the seat inventory so the hot counter is not one single number. If the
  bottleneck becomes "how many of the 178,000 tickets are left," you do not
  keep that as one hot row that every request contends on. You split it into
  many sub counters (block A has X left, block B has Y left) and sum them, so
  the contention spreads. Same idea as splitting one hot stock counter into N
  sub counters under a Zepto sale.
- Serve everything static from the CDN. The seat map image, the layout, the
  event page, all cached at the edge so the origin only ever sees the tiny
  atomic "lock this seat" writes.

The pattern rhymes with every other teardown in this ledger: keep the
expensive, contended thing tiny and atomic, push the huge lazy crowd onto a
cache or a queue, and never let 13 million people touch one database row at the
same time.

### One honest caveat on labels

Confirmed public facts: the Coldplay 13 million and 178,000 numbers, the
crash, the switch to randomized queueing, and that seat viewing dwarfs seat
booking in volume. The specific `SET NX PX 300000` command, the 5 minute hold,
the two layer Redis plus conditional SQL design, and the sub counter split are
the standard, widely documented pattern for this class of system, which
District must implement in some form, but not a published District internal.

## 8. The retention and habit mechanic

Ticketing is a strange retention beast. Nobody books a concert every day. So
the loop is not daily engagement like a feed. The mechanic here is scarcity and
FOMO, and it moves revenue, not daily active use.

- The drop calendar. "Sale opens Tuesday 12 PM" is a manufactured event. It
  concentrates demand into one minute, which creates the queue, which creates
  the scarcity, which creates the urgency. The waiting room screen showing
  "240,000 people ahead of you" is itself the retention hook: people
  screenshot it and post it. During Coldplay, those queue screenshots were all
  over social media. The pain became the marketing.
- The countdown timer during checkout ("pay in 4:58") is a small commitment
  device. Once you have seats held and a clock ticking, you are far more likely
  to finish paying. It lifts conversion, which is revenue.
- District's real trick is different from BookMyShow's. Ticketing is low
  frequency, but food delivery is high frequency. So Zomato's bet is to use
  its daily active food app as the front door to ticketing, and to use
  ticketing occasions to pull you back into food. Book an evening movie on
  District, get nudged toward a dining deal near the cinema. The low frequency
  ticket occasion becomes an entry point back into the high frequency Zomato
  habit. The metric it really targets is frequency through cross sell, gluing a
  once a month behavior onto a several times a week one.

The honest read: for the ticket engine itself, the primary metric is revenue
per transaction and successful conversion under load. The retention play lives
one level up, in bundling ticketing into Zomato's existing daily habit.

## 9. The lesson for Rare.lab

Rare.lab is an AI shader and visual effects product: a node based editor that
compiles to shippable code, plus an embeddable runtime. The seat lock looks
unrelated. It is not. It is the textbook answer to any contested, scarce
resource under a sudden spike, and you will have exactly that.

Your scarce resource is not seats, it is GPU time for AI shader generation and,
if you offer them, limited live collaboration or compute sessions. Picture a
Product Hunt launch or a viral tweet: 50,000 people hit "generate this shader
with AI" in the same 10 minutes, all wanting a slot in a finite GPU pool. That
is Aditya and the front row seats.

Three concrete, actionable moves, biased to scale and performance:

1. Never guard a hot contended resource with a synchronous blocking lock on the
   request path. If a GPU slot is claimed with a held database row lock, your
   pool becomes the Coldplay database, threads pile up, everything stalls.
   Claim the slot with a single atomic in memory operation, the exact
   `SET slot NX PX <ttl>` move. One command decides winner and loser with no
   gap.

2. Make the time to live do the cleanup. Give every claimed GPU slot or compile
   job a TTL, say 120 seconds. If a user's browser tab dies mid generation, the
   slot must free itself with zero cleanup job, exactly like an abandoned seat
   hold auto expiring after 5 minutes. Build the TTL in from day one. Do not
   write a reaper cron, let the expiry be the janitor.

3. Put an admission gate in front of anything flash mobbable. Before your GPU
   pool is a smoking crater on launch day, front it with a virtual waiting
   room: a cheap edge check that admits users at the rate your pool can
   actually serve and parks the rest with an honest position and estimate.
   Drain at your true capacity, not at whatever the crowd throws at you, and
   randomize admission in the opening window so the fastest retry loop does not
   win. And split the two paths: browsing the shader gallery and previewing
   effects is your 100x read traffic, cache it hard at the edge; actually
   reserving compute is the tiny sacred write path, keep it atomic and keep the
   browsers off it.

The one line version: the moment two users can want the same indivisible thing
at the same instant, you are running a seat lock, so reach for an atomic in
memory claim with a self expiring TTL and an admission queue in front, and keep
the lazy readers far away from the one hot write.

---

## Sources

- Pollstar, "BookMyShow Buckles Under Coldplay Onsale In India As 13 Million Try To Secure Tickets" (Sept 2024): https://news.pollstar.com/2024/09/26/bookmyshow-buckles-under-coldplay-onsale-in-india-as-13-million-try-to-secure-tickets/
- Malay Mail, "Coldplay's three concerts in India sell out in 30 minutes" (Sept 2024): https://www.malaymail.com/news/showbiz/2024/09/29/coldplays-three-concerts-in-india-sells-out-in-30-minutes-tickets-resold-for-as-high-as-rm49261/151988
- BookMyShow.Live on X, waiting room and randomized queue for the added India show: https://x.com/Bookmyshow_live/status/1857681822085427257
- TechCrunch, "Zomato buys Paytm's entertainment ticketing business for $244 million" (Aug 2024): https://techcrunch.com/2024/08/21/zomato-buys-paytms-entertainment-ticket-business-for-244-million
- District by Zomato: https://www.district.in/
- Queue-it, "How Queue-it Works" (virtual waiting room, edge 302, token verification, FIFO plus randomization): https://www.queue-it.com/developers/how-queue-it-works/
- Queue-it on AWS, virtual waiting room with DynamoDB queue state at high throughput: https://aws.amazon.com/blogs/apn/how-to-manage-peak-traffic-on-aws-using-queue-its-virtual-waiting-room
- techinterview.org, "System Design: Ticketing System (Ticketmaster)" (seat states, holds, waiting room): https://www.techinterview.org/post/3233463451/system-design-ticketing-system-ticketmaster/
- OneUptime, "How to Implement a Booking Lock System with Redis" (SET NX PX, TTL auto release): https://oneuptime.com/blog/post/2026-03-31-redis-booking-lock-system/view
- Redis SET command docs (atomic NX PX, why SETNX plus EXPIRE is deprecated): https://redis.io/docs/latest/commands/set/
- DevelopersVoice, "BookMyShow Seat Selection Architecture: Distributed Locks, Payment Sagas and Zero Double-Booking at Scale": https://developersvoice.com/blog/practical-design/scalable-net-ticketing-architecture-distributed-locks/
