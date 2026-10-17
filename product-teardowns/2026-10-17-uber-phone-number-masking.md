# Uber phone number masking: the anonymized call between driver and rider

Date: 2026-10-17
Product: Uber
Feature: Phone number anonymization (masked proxy calling), plus the in app VoIP "Uber Call" that sidesteps the number pool entirely

A note on scope. This is NOT dispatch (02 July), NOT surge pricing (14 June), NOT ETA
prediction (15 July), NOT map matching (08 August). This is the small phone icon on the
trip screen. You tap it, you talk to the other person, and neither of you ever sees the
other's real number. One tap. Underneath it is a classic systems problem: how to connect
two strangers by phone, millions of times a day, using a pool of rented numbers that is
tiny compared to the number of people, and hand every number back clean a few minutes later.

---

## 1. The user

Meera is standing outside Phoenix Marketcity in Bengaluru at 9:10 pm. She booked an Uber
two minutes ago. The app says Rajesh is 3 minutes away in a white Swift Dzire, KA 05 MX
1123. The mall has three gates and a confusing service road. Rajesh does not know which
gate she is at. Meera does not know he just turned into the wrong lane.

She needs to talk to him for fifteen seconds. "I am at Gate 2, near the Starbucks." That is
the whole conversation. She does not want to give a stranger her personal number. He does
not want to give her his.

The same moment plays out for a Swiggy delivery rider trying to find a flat, an Amazon
courier at a locked gate, a Rapido captain two buildings away. Short call, two strangers,
nobody wants to hand over a permanent phone number.

## 2. The real problem

Before masking, ride apps simply showed the other person's real number, or the driver
called from his own SIM. That caused real harm. Drivers kept rider numbers and texted them
later. Riders got "hi maam" messages at midnight days after a trip. Women stopped sharing
pickup points. In India this was bad enough that it became a safety and trust problem, not
a convenience problem.

The friendly way to say it: you need to talk to the delivery guy for ten seconds, but you
do not want him to have your number forever. And he probably does not want you to have his
either. The call should work now and then the line between you should disappear.

## 3. The feature in one sentence

When you tap call on a trip, Uber connects the two of you through a temporary virtual number
(or a free in app internet call) so the call goes through while both real numbers stay hidden,
and the connection is torn down and forgotten once the trip is over.

## 4. Jobs to be done

What Meera is really hiring this feature to do:

- "Let me reach my driver right now without digging for his number."
- "Do not let this stranger keep my number after tonight."
- "Make the call just work even if my data is flaky outside the mall."

What Uber is hiring it to do, quietly:

- Keep riders (especially women) willing to share a pickup point, because that is what
  keeps them booking.
- Keep the relationship on the platform. If driver and rider swap real numbers, the next
  ride can happen off app with no commission. Masking is also an anti disintermediation moat.

## 5. How it works for the user

Meera taps the phone icon on the trip screen. Her phone shows two choices (this is the real
flow Uber shipped in India and Bangladesh around 2018 to 2019):

1. A regular voice call.
2. A free "Uber Call" that goes over the internet inside the app.

If she picks the regular call, her phone dials what looks like an ordinary number, something
like +91 80 4718 XXXX. It does not look like Rajesh's number because it is not. It is a
rented virtual number. Rajesh's phone rings, and on his screen he sees that same unfamiliar
virtual number, not Meera's. They talk. "Gate 2, near Starbucks." They hang up. Twenty
minutes later that same virtual number is quietly routing a call between two completely
different people in another part of the city.

If she picks "Uber Call," no phone number is involved at all. The voice travels as data over
her internet connection, the way a WhatsApp call does, routed through Uber's servers. It is
free and there is simply no number to leak.

## 6. The actual flow, step by step

Regular masked call, tap by tap:

1. Meera taps the phone icon. The app sends a request to Uber's backend: "connect rider R
   and driver D for trip T."
2. The backend asks its telephony layer for a proxy number to use for this trip in this city.
   The telephony layer picks a free virtual number from a pool, say +91 80 4718 5566, and
   records a short lived mapping: "for the next while, calls on 5566 belong to trip T."
3. Uber tells Meera's app to dial 5566. Her phone dials a normal looking number. To her OS,
   her carrier, and her call log, it is just a phone call.
4. The call lands at the telephony provider (in India this has been Exotel since 2016; Uber
   has also used Infobip globally). The provider sees "incoming call to 5566, from Meera's
   real number."
5. The provider asks Uber (or its own mapping store): "5566 plus this caller, who do I bridge
   to?" The answer: Rajesh. The provider places a second leg to Rajesh's real number and
   bridges the two legs. Neither leg ever sees the other's real number. Each sees 5566.
6. They talk. The provider relays audio between the two legs.
7. The trip ends. After a short grace window, Uber tells the telephony layer to release the
   mapping. 5566 goes back into the pool, cleaned, ready for the next pair. If Rajesh tries
   to call 5566 the next day, it no longer maps to Meera. He reaches a dead end or a new
   stranger, never her.

Uber Call (VoIP), tap by tap: Meera taps, the app opens a data voice session, signaling and
audio go through Uber's servers over the internet, no PSTN number is leased at all, and when
the call ends the session is just closed. Confirmed by Uber: it is a free "call over an
internet connection rather than a phone's cellular service."

## 7. Under the hood, like the engineer

The honest disclaimer first. Uber has not published a deep engineering post on its masking
internals. What is confirmed is the user facing behavior (Uber newsroom, 2018 to 2019) and
the scale, from the telephony vendors (Exotel and Infobip case studies, which are vendor
reported). The architecture below is the standard way this exact class of problem is solved,
drawn from public designs by Plivo, Twilio, Exotel and Yelp's engineering blog. I label the
Uber specific guesses as inference.

### The core data structures

Two structures do almost all the work.

**A pool of virtual numbers, as a free list.** Think of a stack or queue of available
numbers: `[5566, 5567, 5568, ...]`. When a trip needs a proxy, you pop one. When the trip
ends, you push it back. This is the same pattern as a database connection pool or a thread
pool. The numbers are a scarce, rented, reusable resource, so you lease and return them.

**A mapping table, as a hash map keyed by the proxy number.** Something like:

```
5566 -> { trip: T, rider: +91-98xxxxxx (Meera), driver: +91-99xxxxxx (Rajesh),
          expires_at: 21:40 }
```

When a call arrives on 5566, the telephony provider needs to answer one question in a few
hundred milliseconds: who does this bridge to? That is an O(1) hash map lookup on the number,
then a check of who is calling. If the caller is Meera, bridge to Rajesh. If the caller is
Rajesh, bridge to Meera. One proxy number, both directions, disambiguated by the calling
party's real number. This is why the lookup has to be a hash map and not a scan: the inbound
call is a live phone ringing, the provider's webhook is waiting, and a slow answer is a
dropped call.

Linked around these: a telephony webhook endpoint (the provider calls Uber's service, or
Uber preloads the mapping into the provider), and an event that fires on trip end to trigger
release.

### Matching versus returning, the two halves

Like search has matching and ranking, masking has allocation and release, and the hard half
is release.

Allocation is easy: pop a free number, write the mapping with a TTL. Release is where systems
break. If you release too early, the driver calls back to say "I am at the gate" and reaches
nobody. If you release too late, you leak: 5566 is still mapped to Meera when a new rider
across town gets handed 5566, and now that new rider's callback rings Meera. So you add a
grace window after trip end, and then a cooldown or quarantine before the number is reusable,
so a late caller on an old trip never lands on a fresh pairing. Exotel's own lifecycle names
these states plainly: create, active, update, deactivate.

### Why not just give everyone a permanent number

Because the math does not work. A platform with 50 million registered accounts would need
50 million rented numbers if each person kept one, even though only a small slice are ever on
a call at the same moment (this exact framing is from Arpit Bhayani's masking design writeup).
Numbers cost money every month and are regulated per country. So you size the pool to
concurrency, not to users.

Plivo publishes the formula: virtual numbers needed = 2 x (concurrent sessions) - 1, using
whichever side has more concurrency. Their worked example: 10 concurrent orders need 19
numbers, not 20 users worth, not 50 million. Yelp's engineering team says the same thing in
one line: for short lived delivery or rideshare calls, a relatively small pool plus a very
aggressive recycling policy is enough. A trip call lasts maybe 30 seconds, so each number can
serve dozens of pairs an hour.

### One real number walked through the pool

Take 5566 across one evening in Bengaluru:

- 9:10 pm: leased to trip T1 (Meera and Rajesh). They talk 20 seconds. Trip ends 9:35 pm.
- 9:35 pm: grace window starts. 9:40 pm: released, enters a short cooldown.
- 9:48 pm: leased again to trip T2 (a different rider and driver near Indiranagar).
- and so on, maybe 30 to 60 pairs on that one number across the night.

That reuse is the whole trick. The pool is small, the city is huge, and each number is a
hot reusable slot.

### The scale story at three tiers

**1,000 trips a day.** One city, a handful of concurrent calls. A pool of maybe 20 numbers,
one mapping table in a single database or even in memory, a nightly process that sweeps
expired rows. Recycling barely matters. You could almost hand out semi permanent numbers.
Sharding here would be over engineering.

**100,000 trips a day.** Now concurrency bites. At peak you might have a few thousand calls
live at once, so by the Plivo rule you need a few thousand numbers, pooled and recycled
hard. New things appear:
- Numbers become regional. A Bengaluru rider should see a Bengaluru looking number, because
  a local caller ID gets picked up and a strange international one gets ignored. So the single
  pool splits into per region pools (Exotel documents exactly this: dedicated, shared, and
  regional pool strategies).
- The mapping lookup must be fast and shared across many telephony workers, so it moves into
  an in memory store like Redis, not a slow disk query, because an inbound call cannot wait.
- You must handle the "busy" state: the pool for a region is momentarily empty. Exotel calls
  this out directly. The fix is a bigger pool or a shared pool, plus a fallback.

**10 million plus trips a day (real Uber scale).** Uber India alone has run about 2 million
masked calls a day through Exotel since 2016, with 200,000 to 300,000 calls an hour at peak
(vendor reported). At this tier:
- The mapping store is sharded, naturally by the proxy number itself or by region, because
  one Redis or one table cannot hold the whole world's live calls. A call on an Indian number
  only ever needs the Indian shard.
- Telephony is multi provider and multi country: Exotel in India, Infobip globally (Infobip
  reports building Uber's "Number Masking" and hitting a 90 percent global call anonymization
  target). Each provider is a separate number inventory to manage and recycle.
- Regional backup numbers act as a failsafe when a provider or a pool has trouble, so a
  connectivity blip does not mean a dropped pickup call. (Exotel's Uber work reports call
  connectivity rising after they came on, from the low to mid 60s in percent, vendor reported.)
- Allocation must be idempotent. A rider taps call twice on a flaky network. You must not
  lease two numbers for one trip and you must not double bridge. Key the allocation by trip
  id so a retry returns the same number. This is the same idempotency key idea as Stripe
  (20 June) and the UPI RRN join key (01 August), applied to phone numbers.
- The finite inventory is the real wall. You cannot buy 10 million numbers. So you recycle
  aggressively, quarantine briefly, shard by region, and, crucially, you build an escape
  hatch that does not consume a leased number at all.

### The VoIP escape hatch

That escape hatch is "Uber Call," the free in app internet call. It is not a nicer version of
the masked PSTN call. It is architecturally different and it is the scale pressure valve.
A data call routes voice over the internet through Uber's servers, so it leases zero numbers
from the finite pool. Inference, clearly labeled: the signaling is almost certainly a standard
real time stack (SIP or WebRTC style) and the audio is relayed through Uber's own media
servers or TURN relays, the same shape as a WhatsApp call. Confirmed part: Uber describes it
as a free VoIP call made over the internet rather than the cellular network, offered as the
second option next to a regular call. The business beauty: every rider who taps Uber Call
instead of the regular call frees a scarce leased number for someone whose data is too weak
for VoIP. The masked PSTN pool and the VoIP path together are a cheap default with an expensive
fallback, sized so the scarce resource is spent only when it has to be.

## 8. The retention and habit mechanic

This is not a dopamine loop. There is no streak, no badge, no red dot. It is a trust and
safety feature, and its retention effect is the kind you only notice when it is missing.

The loop is negative space. Because Meera knows her number will not leak, she keeps sharing
her exact pickup and keeps booking late at night from the mall. The feature removes a reason
to churn rather than adding a reason to return. The metric it moves is retention, specifically
the retention of the most privacy sensitive riders, and platform integrity.

There is a second, colder business loop: anti disintermediation. If the driver cannot keep the
rider's number, the next ride cannot quietly move off app to dodge the commission. A real
observed signal of how much this matters: Uber, Infobip and Exotel all publicly treat the
anonymization rate as a headline number (Infobip cites the 90 percent global target). You do
not put a number on a feature you consider optional. The habit it protects is the booking
habit itself.

## 9. The lesson for Rare.lab

Rare.lab has its own finite, rented, reusable resources: GPU compile workers, live preview
render slots, collaboration session ids, shareable embed preview URLs. The masking pattern is
the exact playbook for all of them, and it is a scalability pattern first.

1. **Pool and recycle the scarce resource, size it to concurrency not to users.** You will
   never have one dedicated GPU compile worker per project, just as Uber never has one number
   per rider. Keep a free list of workers, map `session -> worker` in an O(1) table with a TTL,
   and the Plivo rule holds: you need about two times peak concurrent compiles, not one per
   user. A thousand users sharing a graph do not need a thousand warm workers.

2. **The hard half is release, not allocation.** A compile worker or a preview slot handed back
   dirty, or handed back too early while a straggler frame is still rendering, is your leaked
   number: the next user lands on stale state. Add a grace window and a short quarantine
   (flush caches, reset GPU state) before a worker rejoins the pool. Design the teardown as
   carefully as the setup.

3. **Make allocation idempotent and keyed.** A flaky client that taps "render preview" twice
   must not lease two GPU slots. Key the lease by `(project, version)` so a retry returns the
   same slot, the same way masking keys by trip id.

4. **Shard the pool by region or edge, and match locality.** Uber hands a Bengaluru rider a
   Bengaluru number because a local caller ID actually connects. Hand a user the compile
   worker or edge runtime nearest them, and shard the mapping store so a session only ever
   touches its own region's shard. One global table of every live session will not survive.

5. **Build the VoIP escape hatch.** This is the sharp one. When the scarce leased resource is
   contended (every dedicated GPU worker is busy), do not just queue and stall. Have a cheaper
   data path that does not consume the scarce resource at all: a shared pooled CPU or WASM
   fallback compile, a lower fidelity preview, a cached variant. Uber's free VoIP call frees a
   leased number for the person whose network truly needs it. Your fallback render frees a GPU
   worker for the job that truly needs it. Design the cheap default plus expensive fallback so
   the scarce thing is spent only when nothing else will do. The wall is never your code, it is
   the finite external resource, so plan the recycle, the cooldown, the regional shard and the
   escape hatch from day one.

---

## Sources

Confirmed, Uber:
- Uber Newsroom India, "Keep your number private with phone anonymisation": https://www.uber.com/en-IN/newsroom/keep-your-number-private-with-phone-anonymisation
- Uber blog, phone anonymisation: https://www.uber.com/blog/phone-anonymisation
- Uber Newsroom Bangladesh, "Uber VoIP call" (the free in app internet call option): https://www.uber.com/bd/en/newsroom/uber-voip-call/

Vendor reported scale (treat specific numbers as vendor claims):
- Exotel customer story, Uber (about 2 million daily calls in India since 2016, 200K to 300K calls per hour at peak): https://exotel.com/case-study/uber/
- Infobip customer story, Uber (Number Masking, 90 percent global anonymization target): https://www.infobip.com/customer/uber

Industry design patterns for number masking (the architecture and scale math):
- Plivo docs, virtual number allocation and the 2 x concurrent - 1 pool sizing formula: https://www.plivo.com/docs/number-masking/concepts/virtual-number-allocation
- Yelp Engineering, "Phone Number Masking for Yelp Services Projects" (aggressive recycling for short lived sessions): https://engineeringblog.yelp.com/2024/03/phone-number-masking-for-yelp-services-projects.html
- Exotel docs, number masking setup and pool strategies (dedicated, shared, regional, busy state, allocation lifecycle): https://developer.exotel.com/docs/advanced-config/number-masking-setup
- Arpit Bhayani, "Designing a Scalable Phone Number Masking System for Hyperlocal Apps" (the 50 million numbers framing, pool and return to pool): https://arpitbhayani.me/videos/designing-scalable-phone-number-masking-system/
- Twilio glossary, "What are masked phone numbers" (the third proxy number forwarding pattern): https://www.twilio.com/docs/glossary/what-are-masked-phone-numbers
