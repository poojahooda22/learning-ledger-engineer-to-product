# Stripe API Versioning: how a 2014 integration still works in 2026

Date: 2026-10-26
Product: Stripe
Feature: Date-based API versioning with version change transformations

A note on sourcing. Stripe's core versioning design is described in their own
engineering post "APIs as infrastructure: future-proofing Stripe with
versioning" and in their public docs. The real field-change examples below are
from Stripe's own public changelog. Brandur Leach, a former Stripe engineer,
wrote the most detailed public account of the pinning model on brandur.org.
Where I describe internals Stripe has not published line by line (the exact
code of a transformation, the gate mechanism), I label it inference and say so.

---

## 1. The user

Meet Ravi. He is the one backend engineer at a 12-person D2C mattress brand in
Bangalore. Two years ago, on a Tuesday afternoon in early 2024, he wired up
Stripe to take card payments on the checkout page. He copied the sample code,
pasted his secret key, read the `charge.succeeded` webhook, flipped the order to
"paid", and shipped it. It worked. He moved on to the next fire.

It is now October 2026. Ravi has not touched that payment code in over two
years. He has not read a single Stripe changelog. He does not know what version
of the Stripe API his code is talking to. He does not want to know. The company
did 4 crore in sales last quarter and every rupee of it flowed through code he
wrote once and forgot.

Ravi is not lazy. He is normal. The person hitting this feature is every
developer who integrated a payment API once, under deadline, and then spent the
next two years on everything except that integration.

## 2. The real problem

Here is the pain, told plainly.

An API is a promise. When Ravi's code reads `response["status"]` from a Stripe
charge, he is trusting that the field will still be called `status`, still hold
the values he coded against, tomorrow and next year. His whole payment flow is a
tower built on the exact shape of Stripe's JSON.

Now put yourself on the other side. Stripe has thousands of engineers who want
to improve the API every week. Someone realizes a boolean field was a mistake
and should have been an enum. Someone wants to split one object into two.
Someone wants to rename a field so it finally makes sense. Every one of those
improvements, shipped naively, is a live grenade rolled under Ravi's checkout
page. He will not see it coming. His code will throw at 2am on the biggest sale
day of the year, and he will have no idea why, because he changed nothing.

The classic bad answer is "v1, v2, v3". You freeze v1, build v2 with the fixes,
and tell everyone to migrate. But v2 is a big-bang rewrite. It bundles fifty
unrelated changes into one scary jump. Ravi will never do it. So you are stuck
maintaining v1 forever anyway, and now you have two code paths that drift apart.

Stripe's own framing, from their engineering blog: an API is "a contract for
communication" that "can't be changed without considerable cooperation and
effort", and "to date, we've maintained compatibility with every version of our
API since the company's inception in 2011." That is the real problem stated as a
goal. Fifteen years of API changes, and nobody's integration broke because
Stripe changed something.

## 3. The feature in one sentence

Stripe pins every account to the exact API version it first used, names versions
by date instead of v1/v2/v3, and keeps old integrations working by transforming
fresh responses backward through a chain of small, dated changes until the shape
matches whatever version the caller is pinned to.

## 4. Jobs to be done

What is Ravi really hiring this feature to do?

- "Let me integrate once and not babysit it." He wants to write payment code and
  forget it, the way he forgot the Postgres driver version.
- "Never surprise me." He wants a guarantee that no response shape changes under
  him without an explicit choice on his part.
- "When I do upgrade, make it small and reversible." The day he finally needs a
  new field, he wants the jump to be one tidy step with a rollback, not a rewrite.

What is Stripe hiring it to do, from the other side?

- "Let us keep improving the API every week without a change-review committee
  that answers to fifteen years of legacy integrations."
- "Let a new engineer add a breaking change safely, in isolation, without
  understanding the whole history of the API."

## 5. How it works for the user

From Ravi's seat, the feature is almost invisible, and that is the point.

When he made his very first API call in 2024, Stripe silently stamped his
account with the current version on that date. Say it was `2024-09-30.acacia`.
That stamp is now the default for every request from his account, forever, until
he changes it himself. Stripe's docs: "Your account's default API version gets
set the first time you make an API request." New accounts get pinned to the
latest version at signup so they never "accidentally receive a breaking change".

Every response he reads, and every webhook event Stripe sends him, comes shaped
exactly the way the API looked on that 2024 date. Stripe has shipped dozens of
changes since. Ravi sees none of them. His `status` field still reads the way it
read in 2024.

When Ravi one day wants a new field that only exists in a newer version, he has
three visible controls:

1. Upgrade his account's default version in the Stripe dashboard (Workbench).
2. Override the version on a single request with the `Stripe-Version` HTTP
   header, so he can test the new shape on one call without touching everything
   else.
3. Pin a specific webhook endpoint to its own version, independent of the
   account default, so his new "reporting" endpoint can use a modern shape while
   his old "fulfillment" endpoint keeps the 2024 shape.

That is the entire visible surface: a stamp he can see, a header he can set, a
per-webhook override. Everything hard happens underneath.

## 6. The actual flow, step by step

Walk one real request end to end. Ravi's server asks Stripe to retrieve a bank
account object attached to a connected account.

1. Ravi's code sends `GET /v1/accounts/acct_123/external_accounts/ba_456`. He
   sets no version header. His SDK pins no version. So three fallbacks resolve in
   order: request header, then SDK-pinned version, then account default. The
   account default wins: `2024-09-30.acacia`.
2. Stripe's API server authenticates the key, loads the bank account record from
   its datastore, and builds the response object at the newest internal version.
   Call today's newest `2026-09-30.endive`. Internally there is exactly one
   canonical, modern shape of a bank account. Everything is computed against it.
3. Now the server has a mismatch. It holds a 2026-shaped object. The caller is
   pinned to 2024. It must hand back the 2024 shape.
4. The server looks up the ordered list of every backwards-incompatible change
   between `2024-09-30.acacia` and `2026-09-30.endive`. For each change that
   touches the bank account object, it applies that change's downgrade
   transformation, walking from newest back toward the pinned version.
5. The transformed object now matches the 2024 contract to the byte. Stripe
   serializes it to JSON and sends it.
6. Ravi's code reads the fields it has always read. It has no idea any
   transformation happened. The webhook for the same object goes through the
   identical pipeline, except the target version comes from the webhook
   endpoint's own pin if it has one.

The magic is step 4. That is the whole feature, and the next section is about how
it does not fall over.

## 7. Under the hood, like the engineer

This is the heart of it. I will build up the design the way you would if you had
to invent it, name the data structures, then tell the scale story.

### The naive design and why it dies

Start with the obvious approach. Every endpoint checks the caller's version and
branches.

```
if version <= "2014-09-07":
    out["disabled"]  = (status == "errored")
    out["validated"] = (status == "validated")
    out["verified"]  = (status in ("verified", "validated"))
else:
    out["status"] = status
```

This is a real change. On `2014-09-08` Stripe actually did this. From their
changelog, the change "replaces the `disabled`, `validated`, and `verified`
properties on the Bank Account object with a `status` enum property." Three
booleans collapsed into one enum. Older code read the three booleans. Newer code
reads the enum.

The naive branch works for one change. The problem is the second change, and the
fiftieth. Every endpoint's serializer grows a thick ladder of `if version`
blocks. The blocks interleave. The bank account serializer ends up knowing about
the `status` enum change, and a later currency change, and a later metadata
change, all tangled in one function. Add a new breaking change and you must
understand every prior branch to not break it. A new engineer cannot safely
touch it. This is the tier-two wall, and it is a code-maintainability wall, not a
performance one. It arrives around a few dozen versions.

### The design Stripe actually uses: version change modules

Stripe's fix, from their engineering blog, is to make each backwards-incompatible
change a self-contained object. When Stripe needs a breaking change, it "wraps
that change in a version change module." Each module bundles three things:

- Documentation of the change, in plain English. This is not a comment. It is
  data, and it is what generates the public changelog entry you read, like the
  `2014-09-08` bank account note above. Docs cannot drift from behavior because
  they are the same object.
- The transformation itself: the code that rewrites a response (and, where
  needed, a request) from the newer shape into the older shape.
- The set of resource types the change can touch, for example "Bank Account", so
  the server can skip modules that are irrelevant to the object in hand.

All the modules are registered in one ordered master list, keyed by date. That
list is the single source of truth for the entire history of the API. (Inference,
grounded in the blog's description of a "master list" and dated versions: the
list behaves like a sorted array of dated entries. Given a pinned date, you
select the contiguous slice of modules newer than it, and apply them.)

So the request flow in section 6, step 4, is really this: compute the response at
the latest version, resolve the target version, then "walk back through time,
applying each version change module" whose resource type matches, until you reach
the target. The newest change is undone first, then the next newest, down to the
pinned date. It is a pipeline, a chain of small reversible functions composed in
reverse chronological order.

### Why this is the right shape, by data structure

- The master list is an ordered list sorted by date. Finding "all changes after
  2024-09-30" is a slice. You can binary-search the start point. In practice the
  list is short enough (see scale below) that a linear scan with a date compare
  is fine.
- Each response object is a map (a JSON object, a dictionary). A transformation
  is a function from map to map. It reads a few keys, writes a few keys, deletes
  a few keys. For the `status` example, the downgrade reads `status`, writes
  three booleans, deletes `status`. Cheap. Pure. No I/O.
- The account-to-version binding is a hash map lookup: `account_id -> pinned
  version`, read once per request. Webhook endpoint to version is the same, a
  per-endpoint field.
- The composition of transformations is function composition. Because each
  module only has to turn version N into version N minus 1, no module needs to
  know the whole history. It needs to know exactly one hop. That is what lets a
  new engineer add a breaking change in isolation: write one module, drop it at
  the top of the list, done. This is the real win, and it is an architectural
  property, not a performance trick.

The deep pattern here is the same one that shows up everywhere in this ledger:
compute the canonical thing once, adapt at the boundary. Stripe keeps exactly one
modern internal representation of every object. It never stores fifty shapes. It
stores one shape and a chain of adapters. This is the write-side-simple,
read-side-adapt version of denormalization. The expensive, messy variety (all the
old shapes) is generated on demand from one clean source, instead of being stored
and kept in sync.

### A second real example, walked

Take the `2015-07-13` change. From the changelog, it "replaces the
`verification[contacted]` boolean property on the Account object with a
`verification[disabled_reason]` string." An account pinned to a 2015-era version
or earlier expects `verification.contacted: true/false`. Modern Stripe has no
such field; it has a richer `disabled_reason` string like `"fields_needed"`.

The module for this change owns the downgrade: read `disabled_reason`, map any
non-null reason to `contacted: true` (inference on the exact mapping; the shape
of the transformation is public, the precise value mapping is not), delete
`disabled_reason`. An account pinned before 2015-07-13 hits this module on the
way down and gets its boolean back. An account pinned after it never sees the
module at all, because the slice of the list it walks does not include it. Same
code, two accounts, two correct shapes, zero branches in the serializer.

### The scale story, three tiers

The thing that scales here is not a catalog of items. It is the product of three
numbers: how many versions exist, how many object types, and how many requests
per second. Name what breaks at each tier.

Tier one, roughly 10 versions, thousands of integrations (Stripe around 2012 to
2013). A ladder of `if version` branches per endpoint is survivable. Nothing
breaks on performance. The code is ugly but readable. Most companies never leave
this tier and never need the module system.

Tier two, roughly 50 versions, dozens of object types. This is where the naive
design dies, and it dies on humans, not CPUs. The branch ladders become
unmergeable. Changes interact. The test matrix, every version times every object
type, explodes, and nobody can be sure a new change did not silently alter an old
version's output. Stripe's survival move is the version change module: isolate
each change, let it carry its own documentation and its own tests, and compose
them. The maintainability cost goes from "understand the whole history" to
"understand one hop." Brandur's public account puts the rate at "on the order of
10s of new versions in a year", so this tier arrives within a few years.

Tier three, roughly 100 versions accumulated since 2011, billions of API requests
and webhook deliveries, every object type in the platform. Now two performance
facts matter. First, the hot path must be free. The vast majority of request
volume is from accounts on recent versions, and a request already at the latest
version walks zero modules. The transformation chain only costs for old
integrations, and even then it is a handful of cheap map rewrites, no database
round trips, so it never dominates the request. Second, webhooks multiply the
work: the same event may fan out to many endpoints each pinned to a different
version, so the object gets transformed down to several different shapes from the
one canonical source. Because the transforms are pure and in-memory, this is fine.
The genuinely hard scaling problem at this tier is correctness, not latency: the
test and documentation surface. Stripe's answer is structural. Each module owns
its own documentation and its own behavior, so the changelog, the docs, and the
runtime cannot disagree, and each change can be tested against the one hop it
performs rather than the full cross product.

Modern Stripe has also reshaped the release cadence to keep the chain shallow for
most people. From the docs: since `2024-09-30.acacia`, monthly releases carry
"no breaking changes", and breaking changes are batched into a major release
roughly twice a year with a codename like Acacia, Basil, Dahlia, Endive. "You can
safely upgrade to any monthly release without updating your code." So the
expensive backward chain only grows at the major releases. Most version bumps add
nothing to the pipeline at all. Upgrades even get a 72 hour rollback window, per
Stripe's upgrade docs, which is only safe because rolling back is just
re-pinning the account to the earlier date and letting the same transformation
chain do its work.

### Honest limits of what is public

Stripe has published the design (modules, master list, walk-back pipeline, date
versions, pinning) but not the source. The "gate" idea, a small per-version
feature flag that toggles a behavior for a given version, appears in good
third-party write-ups and is a reasonable inference for how a pure behavior
change (not a shape change) is expressed, but I did not find it spelled out in
Stripe's own docs, so treat the gate detail as inference. The exact value
mappings inside any one transformation are not public either. The shape of the
system is confirmed; the bytes are not.

## 8. The retention and habit mechanic

This feature's "retention" is not a dopamine loop. It is trust, and trust is the
strongest retention mechanic a payments company has.

The loop is this. Ravi integrates once. Two years pass. Nothing breaks. That
silence is the product. Every month that his code keeps working without him
touching it, his trust compounds, and the switching cost of ever leaving Stripe
grows, because the one thing he knows for certain is that this integration is
stable. A competitor who is 2% cheaper cannot overcome "I will have to touch my
payment code again and I am not sure it will keep working."

Which metric does it move? Retention and revenue, through churn, not activation.
Activation is Ravi's first successful charge. Versioning does almost nothing for
that. What it does is prevent the silent, deadly churn where an integration
breaks, a founder gets burned, and quietly migrates to Adyen or Razorpay over the
next quarter. A real observed signal: Stripe advertises, as a headline feature,
that it has "maintained compatibility with every version of our API since the
company's inception in 2011." They put a reliability guarantee in marketing
copy, which is only worth doing if it demonstrably keeps customers. The habit it
builds is the best kind: the habit of not thinking about you at all, because you
never gave them a reason to.

There is a smaller, sharper loop for the engaged developer too. Because each
upgrade is small, dated, documented, and rollback-safe within 72 hours, the act
of upgrading stops being scary. A developer who upgrades once, sees the tidy
changelog, tests with the `Stripe-Version` header on one call, and rolls forward
cleanly, learns that upgrading is safe. That turns upgrading into a routine
instead of a dreaded big-bang, which keeps more accounts near the current version,
which keeps Stripe's own backward-compatibility chain shallower. The mechanic
feeds itself.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into shippable shader code and ships an embeddable
runtime. The moment someone embeds a Rare.lab effect on their live site or in a
shipped game build, you have signed exactly Ravi's contract. Their build is
pinned to the runtime and graph format that existed the day they shipped. If a
later Rare.lab release changes the compiled output format or the runtime API, you
have rolled a grenade under a game that is already in players' hands and that
nobody is going to rebuild for you.

Steal Stripe's design directly:

1. Keep one canonical internal representation. The compiler and editor should
   work against a single, latest graph IR (intermediate representation). Never
   store or special-case old graph formats inside the compiler. Store one shape.
2. Pin every shipped project. When a project compiles, stamp the saved file and
   the embed with the exact IR version and runtime version it targeted, the way
   Stripe stamps an account on first request. Default new projects to latest.
3. Make each breaking format change a self-contained, dated migration module that
   owns its own transformation and its own test, and register them in one ordered
   list. Loading an old `.rare` project becomes: parse at its pinned version, then
   walk the ordered migration chain forward to the current IR, applying one tiny
   module per hop. No `if version` ladders in the compiler. A new engineer adds a
   breaking change by writing one module, not by understanding the whole history.
4. Bias to performance on the hot path, exactly as Stripe does. A project already
   at the latest IR walks zero migrations and pays nothing. The migration chain
   only costs when loading an old project, and because each migration is a pure
   in-memory graph rewrite with no I/O, even a long chain is cheap and runs once
   at load, never per frame. The runtime's per-frame path must never contain
   version branches; resolve the version once at load, compile to the current
   shape, then run clean.

The one concrete, actionable move: before you ship the embeddable runtime
publicly, build the ordered migration-module system first, even with just one
module in it. Retrofitting backward compatibility after a thousand sites have
embedded v1 is the big-bang-v2 trap that Stripe spent fifteen years avoiding.
Pay the small cost now so that a shader someone embeds in 2026 still renders,
untouched, when your runtime is on v9 in 2029.

---

## Sources

- Stripe engineering blog, "APIs as infrastructure: future-proofing Stripe with versioning": https://stripe.com/blog/api-versioning
- Stripe docs, "How API versioning works": https://docs.stripe.com/api-versions
- Stripe docs, API reference "Versioning": https://docs.stripe.com/api/versioning
- Stripe docs, "Upgrade your integration" (rollback window, major vs monthly releases): https://docs.stripe.com/upgrades
- Stripe changelog, "Bank Accounts now include a status enum that replaces multiple fields" (2014-09-08): https://docs.stripe.com/changelog/2014-09-08/bank-accounts-include-status-enum
- Stripe changelog, "Accounts include verification disabled reason field" (2015-07-13): https://docs.stripe.com/changelog/2015-07-13/accounts-include-verification-disabled-reason-field
- Brandur Leach, "Why Doesn't Stripe Automatically Upgrade API Versions?": https://brandur.org/api-upgrades
- Hacker News, Stripe engineer comments on versioning (secondary): https://news.ycombinator.com/item?id=13708927
