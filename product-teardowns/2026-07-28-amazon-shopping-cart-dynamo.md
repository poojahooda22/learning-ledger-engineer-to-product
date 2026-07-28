# Amazon shopping cart (the "Add to Cart" button and the always-available store underneath it)

Date: 2026-07-28
Product: Amazon
Feature: The shopping cart. Not checkout, not the buy button. The plain little basket that holds your items between now and when you decide, and the storage system that makes sure it never says "sorry, come back later" even on the biggest sale day of the year.

---

## 1. The user

Arjun is on the Bengaluru Metro at 6:40 PM on a Thursday, standing, one hand on
the rail, the other holding his phone. It is the first day of the Great Indian
Festival sale. He has ten minutes before his stop. He is doing what half the
train is doing: adding things to his Amazon cart.

A pair of running shoes. Tap "Add to Cart." A phone case. Tap. A 1.5 litre steel
water bottle. Tap. The train goes through a tunnel and his signal dies for
fifteen seconds. He taps "Add to Cart" on a set of resistance bands anyway,
because the button still looked alive. When the signal comes back the bands are
in the cart. Nothing spun forever. Nothing said "network error." The little
number on the cart icon just went from 3 to 4.

He is not going to buy any of this tonight. He is parking things. He will open
the cart at home on his laptop, look at the four items sitting there exactly as
he left them, remove the phone case, and check out the rest. The cart is his
scratch pad. It has to survive a tunnel, a dead signal, a switch from phone to
laptop, and a hundred million other people doing the same thing at the same
minute.

He never thinks about any of that. He taps a button and a number goes up.

## 2. The real problem

Here is the honest version, the way you would tell a friend.

The cart is the single most important object between browsing and money. Every
rupee Amazon makes passes through it. So the one thing it must never do is fail
to accept an add. If Arjun taps "Add to Cart" and gets an error, two bad things
happen at once. He loses a tiny bit of trust, and Amazon loses the sale, because
half the time he just closes the app.

Now make it hard. It is sale day. Lakhs of people are hammering "Add to Cart" in
the same few minutes. The servers that store carts are spread across many
machines in many buildings. On a normal day a machine dies every few hours; on
sale day, with ten times the fleet running hot, something is always broken
somewhere. A network cable between two data centre rows gets flaky. One rack
loses power. A whole availability zone hiccups.

The classic way to store data says: when things get uncertain, refuse the write
to stay safe. Lock the row, wait for the majority to agree, and if you cannot
reach the majority, return an error. That is the correct answer for a bank
transfer. It is the wrong answer for a cart. A cart that refuses adds during the
one hour that matters most is worse than a cart that occasionally shows you a
slightly stale list.

So Amazon faced a fork. Either keep the cart perfectly consistent and let it go
down when the network splits, or keep the cart always writeable and deal with
the mess of two versions of the same cart existing for a few seconds. They chose
always writeable. That single choice is the whole story, and the system they
built to make that choice safe, called Dynamo, changed how the industry thinks
about databases.

## 3. The feature in one sentence

The shopping cart is a small per-user bag of items that you can always add to or
remove from, backed by a storage system that would rather show you a slightly
merged version of your cart than ever tell you "no."

## 4. Jobs to be done

What is Arjun really hiring the cart to do?

- "Hold my stuff so I do not have to decide right now." A parking spot between
  wanting and buying.
- "Never lose what I put in." If he adds the water bottle, the water bottle is
  there later. Losing an item feels like the shop threw his basket away.
- "Always take my tap, even on a bad signal or a mad sale day." The add must
  land. A dead button is a broken promise.
- "Be the same cart everywhere." Phone on the train, laptop at home, same four
  items. One bag, many windows into it.
- "Remember me even when I am logged out." A guest cart that survives until he
  signs in, then merges into his real one.

Notice what is NOT on this list: "be perfectly, instantly consistent to the
millisecond." Arjun does not care if, for two seconds during a network blip, one
server thinks he has 3 items and another thinks 4, as long as when the dust
settles nothing he added is missing. That gap between what he needs (nothing
lost) and what a strict database gives (perfect agreement, or an error) is the
room Dynamo lives in.

## 5. How it works for the user

From the outside it could not be simpler.

- Every product page has a yellow "Add to Cart" button. Tap it. The cart icon in
  the corner ticks up by one. A little slide-in confirms "Added to Cart."
- The cart icon shows a running count. It sits in the top corner of every page,
  always visible, so the bag is never more than one tap away.
- Open the cart. You see a list: each item, a thumbnail, the price, a quantity
  stepper, a "Delete" and a "Save for later" link, and a running subtotal.
- Change quantity, remove an item, move it to "Save for later." Each change
  sticks instantly. No "Save" button. The cart is the saved state.
- Close the app. Come back tomorrow on a different device. The cart is exactly
  as you left it.

The magic is the absence of friction. It never spins for long. It never errors
on an add. It is always there. That reliability is not luck. It is the product
of a very deliberate engineering trade.

## 6. The actual flow, step by step

Walk Arjun's water bottle into the cart, tap by tap.

1. Arjun is on the product page for the "Milton Thermosteel 1 litre" bottle. He
   taps the yellow "Add to Cart."
2. The app fires a request to Amazon: "for user Arjun, add SKU B07XYZ, quantity
   1." This is a write against the cart store, keyed by Arjun's user id.
3. The request lands on a coordinator machine responsible for Arjun's cart. It
   does not go hunting through millions of carts. Arjun's user id is hashed, and
   the hash points directly at the handful of machines that hold his cart. More
   on that hash in a moment.
4. The coordinator writes the new cart, "3 items plus the bottle," to itself and
   fires copies to two sibling machines. It does not wait for all three. As soon
   as enough of them say "got it," it tells the phone "done." The phone updates
   the cart count to 4. Total time: a few milliseconds.
5. The tunnel. Suppose during this write one of the sibling machines is
   unreachable because a network link is down. The coordinator does not fail. It
   writes the copy to the next healthy machine on the ring instead, with a note
   attached: "this really belongs to the machine that was down; hand it back when
   it returns." The add still succeeds.
6. Arjun gets home, opens the laptop. The laptop asks for his cart. The request
   reads from a couple of the machines holding it, takes the newest version, and
   returns the four items. He sees the bottle. He removes the phone case. That
   remove is another write, same path.
7. Days later Amazon runs a background gossip between the machines to make sure
   every copy of Arjun's cart agrees, and the machine that was down during the
   tunnel gets its handed-off copy back and catches up.

The whole visible experience is "tap, number goes up." Everything in steps 3
through 7 is the hidden machinery that makes that tap never fail.

## 7. Under the hood, like the engineer

This is the heart of the report. The cart is the textbook use case for a system
that Amazon published in 2007 in one of the most influential systems papers ever
written: "Dynamo: Amazon's Highly Available Key-value Store." The paper opens by
naming the shopping cart as the motivating example. So we are not guessing at the
class of solution. Amazon told us. I will be clear about what is confirmed from
the paper, what is the modern managed evolution, and what is my inference.

### What the cart actually is as data

Strip the cart down and it is one key and one value.

- Key: the user id (or a session id for a guest). Say `user:arjun-8842`.
- Value: a small blob. A list of line items, each a SKU, a quantity, a
  timestamp, maybe a "saved for later" flag. Arjun's cart value is roughly
  `[{shoes, 1}, {case, 1}, {bottle, 1}, {bands, 1}]`. A few hundred bytes.

That is it. No joins. No "find all carts containing this SKU." The only two
operations that matter are `get(user)` and `put(user, cart)`. This is why a
key-value store, not a relational database, is the right tool. When your access
pattern is always "look up one thing by its id," you do not need the machinery of
tables, foreign keys, and a query planner. You need a giant, reliable hash map
spread across thousands of machines. Dynamo is exactly that: a distributed hash
map that never refuses a write.

### The core data structure: the consistent hashing ring

Question one: with a billion carts and thousands of machines, which machine holds
Arjun's cart? The naive answer is `machine = hash(user) % number_of_machines`.
That works until you add or remove a machine. Change the machine count and the
`% N` shifts for almost every key, so almost every cart would have to move. On a
fleet where machines come and go constantly, that is a non-starter.

The fix is consistent hashing, and the mental picture is a clock face.

- Imagine a ring numbered 0 up to a huge number, then wrapping back to 0.
- Hash each machine and drop it onto the ring at that point. Machine A at 12
  o'clock, Machine B at 4 o'clock, Machine C at 8 o'clock.
- Hash a cart key the same way. `hash("user:arjun-8842")` lands at, say, 1
  o'clock. To find its home, walk clockwise from 1 o'clock until you hit the
  first machine. That is Machine B at 4 o'clock. Arjun's cart lives there.

Now the payoff. Add a new machine D at 2 o'clock. Only the keys between 12 and 2,
a thin slice of the ring, move to D. Every other key stays put. Remove a machine
and only its slice reshuffles to the next machine clockwise. Adding or removing
one machine touches roughly `1/N` of the keys instead of all of them. On sale day
when Amazon spins up hundreds of extra machines, this is the difference between a
smooth scale-up and a stampede.

Real Dynamo adds one refinement: virtual nodes. Instead of putting each physical
machine on the ring once, it puts each machine on the ring at many points, say
100 or 200 tokens each. Why? Two reasons, both concrete. First, load evens out.
With one point per machine, a powerful new machine gets the same slice as a weak
old one, and slices vary wildly in size. With 200 points each, the law of large
numbers smooths it, so every machine ends up holding roughly the same number of
carts. Second, when a machine dies, its 200 slices scatter to 200 different
neighbours instead of dumping its entire load onto one unlucky successor. The
recovery load spreads out. This is confirmed in the Dynamo paper.

### Replication: never trust one machine

Arjun's cart is not stored on just Machine B. It is copied to N machines, and in
the paper N is typically 3. Starting from B, you keep walking clockwise and the
next two distinct physical machines, C and then A, each hold a copy. That set of
three is the "preference list" for Arjun's cart. Any of the three can serve a
read. Any of the three can accept a write. There is no single primary that must
be up. That is the root of the always-writeable property. If B is down, C
coordinates. If C is down too, A does. You need all three of a specific,
independently-failing set of machines to be down at once before Arjun's add can
fail, and Dynamo pushes those three onto different racks and power domains so
they do not fail together.

### Quorum: the R plus W greater than N knob

How many copies must answer before a read or a write counts as done? Dynamo
tunes this with two numbers.

- W is how many machines must acknowledge a write before it is "done."
- R is how many machines must answer a read before you return an answer.
- N is the number of copies, usually 3.

The rule Dynamo can use for freshness is `R + W > N`. With N = 3 you might set
W = 2 and R = 2. Two plus two is four, which is greater than three, so any read
of two machines and any write to two machines must overlap on at least one
machine, and that overlap machine has the latest write. That guarantees a read
can see the newest write.

But here is the deliberate choice for the cart. For maximum write availability
you can lower W to 1: a single machine acknowledging is enough to call the add
done. That means an add almost never fails, because you only need one of three
machines alive. The cost is that a later read might briefly miss the newest add
if it reads the two machines that have not caught up yet. Amazon accepted that
cost for the cart, because the anti-entropy machinery below heals the gap in
seconds and nothing is ever lost, only briefly delayed. Fact from the paper:
Dynamo lets each application pick its own N, R, W. Inference: the cart runs
write-heavy settings that favour accepting the add.

### The hard part: two carts, same user, at the same time

Availability has a price, and this is it. Because any of three machines can
accept a write, and because a network split can briefly cut them off from each
other, you can end up with two different versions of Arjun's cart at the same
moment. Picture it concretely.

- Arjun's cart is `[shoes, case, bottle]`.
- On the train he adds bands. That write lands on Machine B: cart becomes
  `[shoes, case, bottle, bands]`.
- At the exact same time, a network blip means his laptop's earlier "remove
  case" reached Machine C, which had not yet heard about the bands: cart becomes
  `[shoes, bottle]`.
- Now B says four items, C says two items. Neither is wrong. They are two
  branches of history. Which one is the cart?

A last-writer-wins database would pick one by timestamp and silently throw the
other away. If it kept C, Arjun's bands vanish. If it kept B, his removed case
comes back. Dynamo refuses to silently pick, and the tool it uses to detect the
split is the vector clock.

### Vector clocks: knowing which version came after which

A vector clock is a tiny piece of bookkeeping attached to each cart version. It
is a list of `(machine, counter)` pairs that records who updated the cart and how
many times. It answers exactly one question: did version X happen before version
Y, or are they cousins that branched?

- Start: cart at `([B,1])`. Machine B has touched it once.
- Arjun adds bands via B: `([B,2])`. B has now touched it twice. This clock
  contains the old one, so the new version cleanly descends from the old. No
  conflict.
- Meanwhile the remove goes through C on a stale copy: `([B,1],[C,1])`. This
  clock and `([B,2])` each contain something the other does not. B counter 2 is
  ahead in one, C counter 1 is ahead in the other. Neither descends from the
  other. That is the signal: these two branched. Conflict detected.

When Dynamo sees two versions where neither clock dominates the other, it does
not throw either away. It keeps both and hands both back to the application on
the next read, saying "you resolve this."

### Resolution: the union, and the famous reappearing item

For a cart the resolution rule is simple and business-driven: take the union of
the items. Merge the two branches into `[shoes, case, bottle, bands]`. The bands
Arjun added are kept. The case he removed is also kept, so it reappears.

That is the famous, slightly funny consequence Amazon accepted on purpose. In the
worst merge case a deleted item can come back into your cart. Amazon reasoned it
out plainly: an item that reappears is a mild annoyance the customer can just
delete again, but an item that silently vanishes is a lost sale and a broken
trust. Given the choice between "you might have to remove something twice" and
"we might lose what you added," they chose never to lose. This exact shopping
cart example, reappearing deleted items and all, is spelled out in the Dynamo
paper. It is the cleanest illustration in all of distributed systems of letting
the business rule, not the database, decide how to merge.

### Healing: hinted handoff and Merkle trees

Two background mechanisms keep the branches rare and short-lived.

- Hinted handoff. Remember the tunnel, when a target machine was down and the
  write landed on a substitute with a note attached. When the real owner comes
  back, the substitute hands the data back and deletes its copy. This is how a
  brief outage does not turn into lost data. Confirmed in the paper.
- Anti-entropy with Merkle trees. To catch any drift between replicas, Dynamo
  builds a Merkle tree over the keys each machine holds. A Merkle tree is a tree
  of hashes: leaves hash individual values, parents hash their children, up to a
  single root hash. Two machines compare roots first. If the roots match, their
  whole datasets match and they are done in one comparison. If the roots differ,
  they walk down only the branches that differ, so they find the exact carts out
  of sync while transferring almost no data. This turns "are our millions of
  carts identical" into a cheap logarithmic check instead of shipping everything
  across. Confirmed in the paper.

### The scale story at three tiers

Tier one, 1,000 carts. A single Postgres table, one row per user, a JSON column
for the items. `get` and `put` by user id, both indexed. It is fast, it is
consistent, and honestly for a small shop this is the right answer. One machine
holds everything. If it goes down for a minute, your thousand users wait a
minute. At this scale nobody needs Dynamo, and reaching for it would be
over-engineering. Name the item concretely: user 442's cart is one row,
`{items: [{sku: "shoes", qty: 1}]}`.

Tier two, 100,000 carts, growing traffic. The single database starts to strain on
reads during busy hours. The standard moves: add read replicas so the many "show
me my cart" reads spread across copies, and put a cache like Redis or Memcached
in front keyed by user id so a repeat cart view never touches the database. Now
the write still goes to one primary, which is fine at this volume. What breaks at
the next tier is exactly that single primary. Every write funnels through it, and
if it dies, writes stop. For a bank you accept that pause. For a cart on sale day
you cannot.

Tier three, hundreds of millions of carts, 100 million-plus requests per second
at peak. This is where the single primary has to die as a concept. There is no
one machine that owns your cart. The consistent hashing ring spreads carts across
thousands of machines. Each cart is replicated to three machines on different
racks. Any of the three takes writes, so losing one, or a whole rack, or an
availability zone, does not stop adds. Quorum settings favour accepting the write.
Vector clocks detect the rare branches, union resolves them, hinted handoff and
Merkle trees heal the drift. The system that Amazon built for this, Dynamo, and
its managed successor DynamoDB, is what the live cart runs on today. The public
numbers are staggering and real: during Prime Day 2023 DynamoDB peaked at 126
million requests per second, in 2024 at 146 million, and in 2025 at 151 million,
all while holding single-digit millisecond response times. That is the always
writeable promise, kept, at a scale where something in the fleet is always broken.

### A note on fact versus the modern system

Two honest clarifications so this is not misleading.

- The 2007 Dynamo paper describes the original internal system that ran the cart,
  and everything above about consistent hashing, N/R/W, vector clocks, hinted
  handoff, and Merkle trees is straight from that paper. Confirmed fact.
- The public Amazon DynamoDB service launched in 2012 is an evolution, not the
  same code. Amazon's 2022 USENIX ATC paper, "Amazon DynamoDB: A Scalable,
  Predictably Performant, and Fully Managed NoSQL Database Service," describes the
  managed version. It kept the always-available spirit and consistent hashing but
  changed internals: it does not push client-side vector-clock reconciliation on
  users by default, leaning instead on server-side conflict handling and stronger
  options, and it uses techniques like Multi-Paxos on some replication paths. So
  when I say "the cart runs on this today," the accurate statement is: the cart
  runs on DynamoDB, whose design descends directly from the Dynamo paper but has
  moved on in its details. The teaching value of the vector-clock cart example is
  timeless even though the current production internals are more refined.

## 8. The retention and habit mechanic

The cart is not just storage. It is one of the quietest, most powerful retention
tools Amazon has, and it works precisely because it never loses anything.

The loop is this. You add things you are not ready to buy. The cart holds them,
perfectly, across days and devices. Every time you open Amazon, that little
number on the cart icon is a small unfinished-business signal pulling you back:
"you have four things waiting." The cart becomes a saved intention, and saved
intentions are the strongest reason to return. This is the same psychology as a
to-do list you have not cleared.

Which metric does it move? All three, in order. It moves activation, because a
new user who successfully parks an item has taken the first real step toward
buying. It moves retention, because a full cart is a standing reason to come back
tomorrow. And it moves revenue most directly of all, because the cart is the
literal on-ramp to checkout, and every item that survives in it is a sale that can
still happen. If the cart lost items, all three collapse. A user who finds his
cart emptied does not trust it with his next intention, and an untrusted cart is
an unused one.

The real observed example is the abandoned-cart nudge, which the reliable cart
makes possible. Because the four items Arjun left are still sitting there
perfectly two days later, Amazon can surface "still interested? your cart is
waiting" on the home page, in notifications, sometimes in a reminder message. That
nudge only works because the cart is a durable, trustworthy record. The engineering
choice, never lose an add, is what turns the cart from a scratch pad into a
retention engine. You cannot remind someone about a cart you dropped.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader editor that compiles to shippable code, plus an
embeddable runtime. The cart's lesson is about the graph the user is building, and
it is a lesson about a choice, not a feature.

Treat the user's shader graph like Amazon treats the cart: never lose an edit,
even if that means briefly tolerating two versions.

Here is the concrete move. A creator is wiring up a shader graph, dragging nodes,
tweaking a noise frequency, connecting a colour ramp. Same as Arjun's cart, this
is a bag of small operations against one keyed object, `graph:project-9981`. Now
the same failure appears. They edit on a laptop, the tab loses connection for
fifteen seconds, they keep dragging nodes because the canvas still felt alive,
and meanwhile a collaborator moved a node on their machine. When the connection
returns you have two branches of the same graph.

The wrong instinct, the one most editors ship, is last-writer-wins: the server
picks whichever save arrived last by timestamp and silently discards the other
branch. That is the "silently vanishing item," and for a creative tool it is
worse than for a cart. Losing ten minutes of node wiring makes a creator distrust
the whole tool, and a creator who does not trust autosave stops using your editor.

Do what Amazon did. Attach a small vector clock to each graph version so you can
detect when two edits branched instead of guessing by wall-clock time. When they
branched, do not throw either away. Resolve by a domain rule you choose on
purpose, the way the cart chose "union of items." For a shader graph the natural
rule is: union the added nodes and edges, keep both sides' new work, and only
surface a visible "you both edited this same node, pick one" prompt for the
genuinely conflicting node, which is rare. Reappearing-deleted-item is your
acceptable worst case too: a node someone deleted might come back, and that is a
one-click fix, whereas an hour of lost wiring is not.

The performance payoff maps straight across. An always-accepts-the-edit graph
store means your editor never blocks the user's drag on a round trip to the
server. The edit lands locally and syncs in the background, exactly like the add
that succeeds on one replica and heals later. Your canvas stays at 60 frames per
second during a bad connection instead of freezing on a spinner. And for the
embeddable runtime, the same "keyed object, replicate to a few, any can serve"
shape means a shader's compiled state can be read from the nearest edge copy
without a single origin that must be up.

The one-line version: for the thing your users are actively building, choose
always-writeable over always-consistent, detect branches with vector clocks
instead of guessing with timestamps, and merge with a business rule that never
silently loses work. A deleted node that reappears is an annoyance. An hour of
lost work is why someone leaves. Amazon decided a reappearing item beats a lost
sale, and that one decision is why the cart icon on the Metro never showed Arjun
an error.

---

## Sources

- Dynamo: Amazon's Highly Available Key-value Store (SOSP 2007, the primary source, opens with the shopping cart): https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
- Werner Vogels, Amazon's Dynamo (blog announcing the paper): https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html
- Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service (USENIX ATC 2022, the modern managed evolution): https://www.usenix.org/conference/atc22/presentation/elhemali
- AWS News Blog, AWS services scale to new heights for Prime Day 2025 (DynamoDB peaked at 151 million requests per second): https://aws.amazon.com/blogs/aws/aws-services-scale-to-new-heights-for-prime-day-2025-key-metrics-and-milestones/
- AWS Blog, Amazon Prime Day 2023 metrics (DynamoDB 126 million requests per second): https://aws.amazon.com/blogs/aws/prime-day-2023-powered-by-aws-all-the-numbers/
- How Amazon Scaled E-commerce Shopping Cart Data Infrastructure (System Design newsletter deep dive on Dynamo): https://newsletter.systemdesign.one/p/amazon-dynamo-architecture
- ByteByteGo, A Deep Dive into Amazon DynamoDB Architecture: https://blog.bytebytego.com/p/a-deep-dive-into-amazon-dynamodb
- Consistent Hashing (original Karger et al. 1997 paper the ring is built on): https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf
- Amazon DynamoDB developer guide, What is DynamoDB (single-digit millisecond performance at any scale): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html
