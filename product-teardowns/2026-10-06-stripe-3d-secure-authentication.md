# Stripe 3D Secure authentication: the "verify with your bank" step on a card payment

Date: 2026-10-06
Product: Stripe
Feature: 3D Secure (3DS2) cardholder authentication, the step where a card payment pauses and asks the bank "is this really the cardholder" before it moves money. This is NOT Stripe Checkout (08-09, the hosted payment page), NOT Stripe Radar (07-04, Stripe's own fraud score), and NOT Stripe idempotency keys (06-20). This is the authentication handshake that runs between the merchant, the card network, and the bank that issued the card.

---

## 1. The user, in the middle of their day

Priya is buying a pair of running shoes for 4,999 rupees on a Shopify store at 9pm, sitting on her sofa, HDFC debit card in hand. She has typed the 16 digit number, the expiry, the CVV. She taps Pay. The page does not immediately say "success." Instead the screen dims and a box slides up that says "HDFC Bank: enter the OTP we just sent to your phone ending 4432." Her phone buzzes. She types the 6 digit code. The box spins for a second. Then the shoes are hers.

That box is 3D Secure. Priya sees it on almost every card payment she makes in India, because the Reserve Bank of India has required a second factor on card payments for over a decade. She barely thinks about it. To her it is "the OTP step." To the three companies behind that one second of spinning it is a real time conversation across the planet.

A second user, Lukas in Berlin, buys the same shoes with the same store on the same Stripe account. He taps Pay and the payment just completes. No OTP, no box, nothing. Same feature, opposite experience. Understanding why Priya gets a box and Lukas does not is the whole teardown.

## 2. The real problem, like a friend would say it

Here is the honest problem. A card number is not a secret. It is printed on the front of the card, it sits in old database dumps, it gets skimmed, it gets phished. So "someone typed the right card number" is weak proof that the real owner is buying. For years that weak proof was all the internet had, and card fraud on "card not present" online payments was the single biggest loss in payments.

The bank that issued Priya's card carries the risk. If a thief uses her stolen number and she disputes it, the bank usually eats the loss through a chargeback. So the bank wants a way to say, during the payment, "prove you are Priya, not someone holding her card number." But there is a brutal tension. Every extra step the bank adds is a step where a real buyer gets annoyed, fumbles the OTP, or just gives up. The data is ugly and consistent: adding friction to a checkout always drops completion. So the bank wants more proof and the merchant wants less friction, and those two pull in opposite directions on the exact same screen.

3D Secure is the treaty between them. And the clever version, the one Lukas hit, is the bank quietly deciding it already has enough proof and letting the payment through with no box at all.

## 3. The feature in one sentence

3D Secure is a real time handshake between the merchant, the card network, and the card issuing bank that lets the bank authenticate the cardholder during a payment, either silently from data it already has (frictionless) or by challenging the shopper for something only they have, like an OTP or a banking app approval (challenge).

## 4. Jobs to be done

What is the shopper really hiring this step to do? They are not hiring it to "authenticate." Nobody wants an OTP. They are hiring it, without knowing it, to make the payment safe enough that the bank will stand behind it, so that if something goes wrong it is the bank's problem and not a drained account.

- "Let me buy the shoes and not get defrauded." (The shopper.)
- "Let me prove this is my real customer so if it is a fraudster, the loss shifts to the bank, not me." (The merchant. This is called liability shift and it is the entire commercial point of 3DS for a business.)
- "Let me approve payments I am confident about without annoying good customers, and only stop to ask when I am genuinely unsure." (The issuing bank.)
- "Let me satisfy the law. RBI in India and PSD2 in Europe both require strong authentication, and I must comply or I cannot operate." (Everyone.)

## 5. How it works for the user (the visible experience)

For Priya in India, the visible experience is four beats. She enters card details. She taps Pay. A box from her bank appears asking for an OTP (or asking her to approve in the bank app). She enters it, and the payment completes. The box is served by her bank, not by the shoe store, which is why it carries the HDFC name and look.

For Lukas in Berlin, the visible experience is two beats. He enters card details. He taps Pay. It completes. His bank ran the exact same 3D Secure handshake in the background, decided it trusted the transaction, and never showed him anything. He has no idea 3D Secure even happened.

That is the headline user fact: in modern 3D Secure (the version called 3DS2, built by EMVCo, the body owned by Visa, Mastercard, and the other networks), the default good experience is the silent one. The OTP box is the fallback for when the bank is not sure, not the default. India leans heavily to the box because RBI rules and issuer habits push almost everything to a challenge. Europe leans heavily to silent because the whole 3DS2 design was built to make PSD2's authentication rule survivable for merchants.

## 6. The actual flow, step by step

Walk Priya's 4,999 rupee shoe payment, tap by tap, but now including the parts she cannot see.

1. Priya enters her card and taps Pay. The Stripe.js code already running on the store page has quietly done one thing in the background before she even finished: it collected her browser and device data (screen size, time zone, language, browser headers) into a hidden invisible frame. This is the device fingerprinting step, and it happened before the Pay tap mattered. More on why below.

2. The store's server tells Stripe "confirm this PaymentIntent." Stripe, acting as the merchant side component called the 3DS Server, decides whether this payment needs 3D Secure at all. Sometimes Stripe is forced to run it (Indian cards, or European rules). Sometimes Stripe chooses to run it to get liability shift. Sometimes Stripe can skip it using an exemption (covered in the scale section).

3. Stripe builds an authentication request, the AReq message, and sends it to the card network's Directory Server. For an HDFC Visa card, that is Visa's directory. The AReq is stuffed with over 100 data elements: the card number, the amount (4,999 INR), the merchant, Priya's device fingerprint, her shipping address, her past behavior on this store. This fat message is the single biggest difference from the old 3DS1, which carried only a handful of fields.

4. The Directory Server looks at the card number's leading digits (the BIN, the bank identification number) and routes the AReq to the right bank's Access Control Server, the ACS. The ACS belongs to HDFC. The directory is basically a routing switch: given this card range, which bank's ACS do I forward to.

5. HDFC's ACS now runs its own fraud risk assessment on all 100+ fields in milliseconds. Is this device one Priya has used before? Is 4,999 rupees normal for her? Is the shipping address her usual one? The ACS produces one of two answers and sends it back as the ARes message.

6a. Frictionless path (what Lukas got). The ACS says "I am confident, authenticated, no challenge needed." It returns ARes with a result of Y. Stripe gets a cryptographic proof token (a cryptogram) that says the bank authenticated this, and the payment proceeds to the actual money movement (authorization) with liability shifted to the bank. The shopper saw nothing.

6b. Challenge path (what Priya got). The ACS says "I need to challenge," returning ARes with a result of C. Now a second mini conversation starts. The ACS serves its own OTP box into Priya's browser. Priya's browser and the ACS exchange the challenge request and challenge response messages (CReq and CRes) directly. Priya types the OTP, HDFC checks it, and the ACS finishes with a cryptogram proving she passed.

7. Either way, Stripe ends up holding a bank signed cryptogram (or a clear "authentication failed"). Stripe attaches that to the authorization request and sends it to get the 4,999 rupees actually moved. The shoes are confirmed.

The key structural fact: the box Priya typed into was never the shoe store's and never Stripe's. It was served by HDFC's ACS, directly, into her browser. Stripe orchestrated the handshake but the bank ran its own authentication screen. That separation is why it is called "3 domain" secure: the merchant domain, the network (interoperability) domain, and the issuer domain, three parties who do not fully trust each other, passing signed messages.

## 7. Under the hood, like the engineer

This is the heart of the report. 3D Secure is not a search or a feed. It is a distributed state machine plus a risk scoring problem plus a message routing problem, run across three companies under a hard latency budget, and the engineering lives in exactly those three places.

### It is a state machine, not a function call

A naive engineer models a payment as one synchronous function: `charge(card, amount) -> success`. 3D Secure breaks that model, because step 6b can pause for 30 seconds while a human finds their phone and types an OTP. You cannot hold an HTTP request open and a database transaction locked for 30 seconds while a person searches their couch for their phone.

So the real data structure is a persistent state machine, one record per payment attempt, that moves through states: `requires_payment_method` to `requires_confirmation` to `requires_action` (this is the "go show the bank's box" state) to `processing` to `succeeded` or `requires_payment_method` again on failure. Stripe exposes exactly this as the PaymentIntent object, and the `requires_action` status is literally "we paused, the shopper must go authenticate." (Confirmed, this is Stripe's documented PaymentIntent lifecycle.)

Why a stored state machine and not a blocking call? Because the authentication is asynchronous and can resume on a different server, after a redirect, even after the shopper switched to their banking app and came back. The payment's truth cannot live in one machine's RAM or one open socket. It must live in a durable row that any server can pick up. This is the same offline-think / online-lookup spine the whole ledger keeps hitting, but turned sideways: the "think" (the human finding their OTP) happens out of band, and the system just needs to durably remember where it was.

Concretely: when Priya's payment hits `requires_action`, Stripe stores that state and hands the browser a pointer to HDFC's challenge. If Priya's wifi drops mid OTP and she reloads, the store asks Stripe for the PaymentIntent, reads `requires_action`, and sends her right back to the bank box. No money was double moved because the state, not the connection, is the source of truth. (This is the same lesson as Stripe idempotency keys, 06-20, and Notion offline sync, 07-16: durable state beats a live connection.)

### The risk decision is a matching-then-ranking problem in disguise

The bank's ACS decision (frictionless or challenge) is the interesting algorithm, and it has the same two halves every ranking teardown in this ledger has had.

Matching (eligibility and hard rules first, cheap): is this card enrolled in 3DS, is the merchant region one where a rule forces a challenge, does a regulation require it. For Priya's Indian card, the matching stage almost always says "challenge required," because Indian issuers overwhelmingly demand the second factor. That is a rule, not a model. It is a hash map lookup on BIN range and region, microseconds, no machine learning needed.

Ranking (the risk score, expensive, only on survivors): for cards where a challenge is not forced, the ACS runs a learned fraud risk model over the 100+ data elements. This is where the device fingerprint earns its keep. The feature vector includes device fingerprint (browser, screen resolution, time zone, fonts), behavioral signals, transaction history (has this card bought at this merchant before, account age), and purchase details (4,999 INR, the currency, the IP, the shipping address). The model outputs a risk score, and a threshold turns that score into frictionless (Y) or challenge (C). (Confirmed that 3DS2 carries 100+ data elements versus roughly 8 to 15 in 3DS1; the exact model each bank runs is private, so the "learned model over a feature vector" description is well grounded inference labeled as such.)

This is why Lukas sailed through and Priya did not, even setting aside India's rules: a frictionless pass is the bank saying "the data already convinced me, I do not need to bother the human." The entire 3DS2 redesign exists to make that silent pass the common case. The old 3DS1 carried only about 8 fields, so banks had almost nothing to score on and defaulted to challenging everyone, which is why 3DS1 was infamous for killing conversion.

### Why the device fingerprint is collected BEFORE the Pay tap matters

Here is a subtle engineering move. The device data collection (the "3DS Method") runs early, in a hidden iframe, often while the shopper is still filling the form. (Confirmed: the 3DS Method URL loads a hidden iframe that runs JavaScript to gather device and browser data and hand it to the ACS before authentication.)

Why so early? Latency budget. If the bank had to request the device fingerprint only after the AReq, that is another browser round trip bolted onto the critical path while the shopper stares at a spinner. By pre collecting the fingerprint into a hidden frame during form fill, the heavy data is already sitting with the ACS the instant the AReq arrives, so the frictionless decision can come back fast. This is the classic move of pushing expensive work off the hot path and pre staging it, the same instinct as Netflix pre positioning bytes on its CDN (07-13) or a search engine building the index offline.

### The Directory Server is a routing table

The card network's Directory Server does one humble but critical job: given a card's BIN, find the right issuer's ACS and forward the message. Think of it as an interval map or a longest prefix match on BIN ranges, the same shape as IP routing. A card starting 4532 11 belongs to one bank's range, 5521 09 to another. The directory is sharded and replicated per network (Visa's, Mastercard's) and must be reachable in milliseconds globally, because every 3DS payment on earth for that network passes through it. It does not authenticate anything. It routes. Keeping it dumb and fast is exactly the Figma "keep the central server dumb" lesson (10-04) applied to payments: the brains (risk scoring) live at the edge (each bank's ACS), the center just routes.

### The scale story at three tiers

The thing that grows here is not a catalog of items. It is the rate of concurrent authentications and the number of issuer endpoints you must talk to, all under a latency budget where a slow handshake is a lost sale.

Tier 1, roughly 1,000 authentications a day (a single growing store). A synchronous model almost works. You could call the bank, wait, and show the box inline, and the occasional 30 second challenge is rare enough that holding some state in a simple database row is fine. At this tier building a full async state machine feels like over engineering. You build it anyway, because it is the only code path that survives to tier 3, and because even one shopper who reloads mid OTP will corrupt a naive design.

Tier 2, roughly 100,000 authentications a day (a mid size platform). Now the synchronous model breaks in three ways. First, challenges that pause for 30 seconds mean you cannot tie up a server thread or a database lock per in flight payment, so the durable PaymentIntent state machine stops being optional. Second, you are now talking to hundreds of different issuer ACS endpoints, and some banks' ACS servers are slow or flaky at 2am, so you need timeouts, retries, and graceful fallback (if the ACS does not answer, do you fail the payment or try without 3DS and eat the liability). Third, you now have enough volume that the frictionless-versus-challenge decision directly moves revenue, so you start caring about exemptions (below). This is the tier where the architecture earns its keep.

Tier 3, Stripe scale (millions of payments a day across 100+ countries and thousands of issuers). Four walls appear, each different from a search system's walls.

- Shard and route by issuer. You cannot hold global state about every bank in one place. The network's Directory Server shards BIN ranges, and Stripe's own 3DS Server fleet is horizontally scaled and stateless, reading each payment's state from a durable store (the PaymentIntent) rather than memory. This is shard-by-tenant again, where the "tenant" is effectively the issuing bank, the same pattern as Notion-by-workspace (06-25) and Stripe-by-account (09-01).

- The latency budget is the SLA. A frictionless authentication must complete in roughly a second or the shopper feels the payment is broken. So the device fingerprint is pre staged (above), the directory lookup is a fast prefix match, and the risk model at the ACS must score in milliseconds. Every hop is budgeted. A slow issuer ACS is a real operational problem Stripe monitors, because that bank's slowness becomes Stripe's apparent slowness.

- Retries and idempotency on an inherently flaky multi party call. You are orchestrating a message across two companies you do not control (the network and the bank) over the public internet. Messages time out. The shopper closes the tab mid challenge. So every step must be safe to retry, which is exactly why the PaymentIntent is idempotent and why the state, not the connection, is canonical. Without idempotency, a retried AReq could double authenticate or double charge.

- The abandonment cliff is the business scaling problem, not just a technical one. Every challenge shown is a chance the shopper bails. At millions of payments a day, a 1 percent swing in challenge rate is a huge amount of money. So the whole system is tuned to minimize unnecessary challenges: pass rich data so the bank can go frictionless, and use exemptions to skip 3DS entirely where the law allows.

### Exemptions: the legal shortcut that is pure engineering

Under Europe's PSD2 rules, strong authentication is required, but with carve outs, and exploiting those carve outs is a real time decision Stripe makes per payment.

- Low value exemption: a transaction under 30 euros can skip SCA. But there is a velocity counter: after 5 consecutive exempted low value payments on the same card, or once the cumulative exempted amount passes 100 euros, the exemption resets and the next one must be authenticated. (Confirmed PSD2 rule.) That counter is a per card running total someone has to maintain and check in real time, a tiny stateful aggregator per card.

- Transaction Risk Analysis (TRA) exemption: if the acquirer's overall fraud rate is low enough, it can skip SCA on higher value transactions (thresholds step down as fraud rises, with 500 euros as a hard ceiling where SCA is always required). This is the most valuable and most demanding exemption because it ties a per transaction decision to a fleet wide fraud rate you must continuously measure. (Confirmed.)

The sharp tradeoff Stripe weighs on every payment: an exemption boosts conversion (no box) but forfeits liability shift (if it was a fraudster, the merchant eats the chargeback, not the bank). So the decision "exempt or authenticate" is a live cost model: expected conversion gain versus expected fraud loss, decided in the hot path. That is the same shape as Rapido predicting whether a captain will accept before pinging (10-01): decide, in real time, whether the friction is worth it.

### What is confirmed versus inferred

Confirmed from primary and official sources: the 3 domain architecture (3DS Server, Directory Server, ACS); the AReq / ARes / CReq / CRes message flow; frictionless (Y) versus challenge (C) outcomes; 3DS2 carrying 100+ data elements versus roughly 8 to 15 in 3DS1; the 3DS Method hidden iframe device fingerprinting step; Stripe's PaymentIntent state machine with a `requires_action` state; PSD2 low value (30 euro, 5 payment / 100 euro reset) and TRA (up to 500 euro) exemptions; RBI's additional factor requirement in India. Inference, clearly labeled: the exact learned risk model each bank's ACS runs (private to each issuer), Stripe's internal sharding of its 3DS Server fleet, and the precise per payment exemption cost model (the general approach is public, the exact thresholds Stripe uses are not).

## 8. The retention and habit mechanic

3D Secure does not build a shopper habit the way a feed or an autoplay does. There is no loop pulling Priya back to the OTP box. Its retention power is the opposite: it is a trust and liability loop that keeps the merchant on Stripe and keeps the shopper transacting at all.

The real mechanic is confidence. When a payment is 3DS authenticated and goes frictionless, the merchant gets liability shift, meaning fraud chargebacks become the bank's problem. A merchant that loses less to fraud and gets fewer disputes is a merchant that keeps using Stripe and keeps accepting cards from more countries. The habit being protected is "I can accept any card on the internet without getting wiped out by fraud." That is a business retention loop, and it moves revenue (fewer losses, more approved good payments) far more than activation.

For the shopper, the mechanic is quieter: it is the absence of catastrophe. A real observed example of the trust-cracker is the Indian RBI experience. For years Indian issuers challenged nearly everything, and shoppers grew to accept the OTP as the price of safety. When OTP delivery fails (the SMS does not arrive, which is common), the payment fails, and that single failed payment does real damage, because the shopper does not blame the flaky SMS, they blame the store. That is the invisible craft trust pattern this ledger keeps meeting (Spotify loudness 08-28, Swiggy serviceability 09-28, YouTube search 10-02): the feature is noticed only when it breaks, and one break costs more than a hundred silent successes earned.

Which is exactly why the frictionless path is the real retention play. Every payment that authenticates silently is a payment where the safety happened and the shopper never felt it. The metric 3DS2 is built to move is authorization rate (the share of good payments that complete), by shrinking the challenge rate without shrinking the safety. The 3DS1 to 3DS2 jump, from 8 fields and challenge everyone to 100+ fields and challenge rarely, was the entire industry trading friction for data so that the safe outcome could also be the invisible outcome.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph of shaders and effects down to shippable code and runs it in an embeddable runtime on wildly different devices. The 3D Secure lesson maps straight onto a core runtime decision: when should an effect pay the cost of a full, correct, expensive path, and when can it be trusted to run the cheap path silently.

The direct transfer is risk-based gating with a staged challenge, not a blanket one. 3DS2 beat 3DS1 by refusing to challenge everyone. It collects rich data up front (the hidden iframe fingerprint, pre staged before the Pay tap), scores risk, and only interrupts when genuinely unsure. Rare.lab should do the same with quality and validation on a given device.

1. Pre stage the device fingerprint, do not probe on the hot path. Just as the 3DS Method collects device data in a hidden iframe before authentication so the frictionless decision is instant, Rare.lab's runtime should fingerprint the GPU, driver, and memory class once at init and cache it, so the per frame decision "can this device run the expensive bloom path" is a lookup, never a live probe. Probing GPU capability mid frame is the stall that 3DS1's late data request was.

2. Make the silent (frictionless) path the default, the challenge the fallback. The expensive full quality shader path is the "challenge": correct but costly, show it only when you must. The cheap approximation is the "frictionless" pass: if the cached device fingerprint plus the scene's measured cost say "this will hold 60fps," run the cheap path silently and never make the frame pay for the expensive validation. Only step up to the costly path when the risk score (predicted frame cost on this device class) crosses a threshold.

3. Model the whole thing as a durable state machine, not a blocking call, because compilation and asset load are asynchronous and can resume. 3DS refuses to hold a socket open for a 30 second human; Rare.lab must refuse to block a frame for a multi second shader compile. Put the compile off the render thread, store the graph's state (compiling, needs-fallback, ready), and let the frame read current state and use the fallback until the real thing is ready. The PaymentIntent `requires_action` state is the model: the frame renders the placeholder now and swaps in the compiled effect when the async work lands, exactly as 3DS shows the fallback and resumes when the OTP completes.

4. Keep the router dumb and the brains at the edge. The Directory Server just routes by BIN; each bank's ACS holds the risk model. Rare.lab's central service should likewise just route and store build artifacts, while the per device cost model and the frictionless/challenge quality decision live in the runtime on the device, where the real GPU is. Centralizing the quality decision would be as wrong as making the Directory Server score fraud.

5. Weigh the exemption tradeoff explicitly: conversion versus correctness. 3DS exemptions trade the OTP box (friction) for forfeited liability shift (correctness). Rare.lab's "skip the expensive validation to hit frame rate" is the same trade: smoother now, but if the cheap path is visibly wrong you own the artifact on screen. Make that a measured decision (expected visual error versus expected frame cost), decided per scene in the hot path, not a global quality slider, so the runtime skips the costly path exactly where it is safe and pays for it exactly where it shows.

One line: Stripe's 3D Secure wins by refusing to challenge everyone, pre staging rich device data so the bank can authenticate silently and only interrupting the shopper when a real time risk score says it must, all orchestrated as a durable asynchronous state machine rather than a blocking call; build Rare.lab's per device quality gating the same way, fingerprint once, default to the silent cheap path, step up to the expensive path only on measured risk, and never block a frame waiting for async work.

---

## Sources

- Stripe, "Authenticate with 3D Secure" (authentication flow): https://docs.stripe.com/payments/3d-secure/authentication-flow
- Stripe Support, "Frictionless flow for charges created using 3D Secure 2 (3DS2)": https://support.stripe.com/questions/frictionless-flow-for-charges-created-using-3d-secure-2-3ds2
- Stripe, "3D Secure 2" guide: https://stripe.com/en-gb-us/guides/3d-secure-2
- Gravitee, "3D Secure (3DS): Protocol, Flows, and Operational Integration" (3DS Server, Directory Server, ACS; AReq/ARes/CReq/CRes; frictionless vs challenge): https://gravitee.io/corpus/gen-694/payment-processor/3d-secure.html
- Netcetera, "3-D Secure Directory Server" (directory routing role): https://www.netcetera.com/payments/Payment-networks/3-D-Secure-Directory-Server.html
- Sardine, "3D Secure Method URL" (device fingerprinting via hidden iframe): https://www.sardine.ai/blog/3d-secure-method-url
- Mastercard, "Top 10 Things to Know About 3DS": https://www.mastercard.com/content/dam/public/mastercardcom/globalrisk/pdf/Top-10-Things-to-Know-About-3DS.pdf
- Checkout.com, "The sunsetting of 3DS1 and how businesses can prepare" (3DS1 ~8 fields vs 3DS2 100+ data elements): https://www.checkout.com/blog/post/the-sunsetting-of-3ds1-and-how-businesses-can-prepare
- Convesio knowledge base, "EMV 3D Secure" (100+ data elements, risk-based decisioning): https://convesio.com/knowledgebase/article/emv-3d-secure/
- PCI Proxy, "SCA exemptions under PSD2: a practical guide for payment teams" (low-value and TRA exemptions, velocity counters, liability shift tradeoff): https://www.pci-proxy.com/blog-posts/sca-exemptions-under-psd2-a-practical-guide-for-payment-teams
- SEON, "TRA Exemptions in PSD2": https://seon.io/resources/tra-exemptions/
- Business Standard, "RBI mandates stronger two-factor authentication in new guidelines" (India additional factor authentication): https://www.business-standard.com/finance/news/rbi-two-factor-authentication-digital-payments-guidelines-2026-125092501154_1.html
- RBI draft, "Framework on alternative authentication mechanisms for digital payment transactions": https://website.rbi.org.in/web/rbi/-/notifications/framework-on-alternative-authentication-mechanisms-for-digital-payment-transactions-draft
- "Investigation of 3-D Secure's Model for Fraud Detection" (arXiv 2009.12390): https://arxiv.org/pdf/2009.12390
