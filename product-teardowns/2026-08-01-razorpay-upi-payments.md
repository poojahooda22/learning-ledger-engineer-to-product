# Razorpay UPI payments: the collect-and-intent flow, the async switch, deemed success, and reconciliation

Date: 2026-08-01
Product: Razorpay (payment gateway) riding the UPI rail (NPCI)
Feature: UPI as a checkout method: the collect request, the intent deep link, the asynchronous switch underneath, and how a "did it go through" that no one can answer instantly gets made correct later.

A note on grain. This is one feature: paying by UPI at a checkout. Razorpay is the gateway a merchant plugs in. The rail underneath is UPI, run by NPCI. Razorpay's exact internal code is not public, so I split this carefully. NPCI-level mechanics (ReqPay, deemed success, the mapper, TCC codes) are documented fact. Where I describe Razorpay's own queues and state machine I label it inference and give the well-grounded "this is how a gateway must solve it" version.

---

## 1. The user

Aarav is buying a ₹499 phone case at 9:40pm on a small Shopify-style store that uses Razorpay for checkout. He is on his phone, one hand, half watching a match. He taps "Pay", picks UPI, and a spinner appears. He does not have a card out. He does not want to type a 16 digit number on a phone keyboard. He just wants the little Google Pay sheet to pop, wants to press his UPI PIN, and wants the order to say "confirmed".

The other Aarav is at a kirana store the same evening, buying ₹340 of groceries. He points his camera at the paper QR taped to the counter, his UPI app opens with the amount already filled, he enters his PIN, and the shopkeeper's phone says "₹340 received" out loud in Hindi before he has put his own phone back in his pocket.

Same rail, two doors. The website is a collect. The QR is an intent. The whole teardown lives in the difference between those two doors and in what happens in the two seconds after Aarav presses his PIN.

---

## 2. The real problem

Money has to move between two bank accounts that have never heard of each other, through apps built by different companies, and everyone has to agree on one fact: did the ₹499 leave Aarav's HDFC account and land in the merchant's ICICI account, yes or no.

The pain, described like a friend would: before UPI, paying online in India meant a card number, an expiry, a CVV, then an OTP page from your bank that timed out half the time and threw you back to a blank screen where you had no idea if you had been charged. You would refresh, panic, check your SMS, maybe pay twice. The merchant had no idea either. Both of you were staring at a spinner that meant nothing.

The deeper problem is that the honest answer to "did it go through" is often "we do not know yet." Aarav's phone talks to his UPI app, which talks to NPCI, which talks to two banks, and any of those five hops can time out. A network can drop the success message on the way back. The money can move while the "it worked" reply gets lost. So the system cannot just ask once and trust the answer. It has to be built so that a lost reply does not become a lost payment or a double charge.

---

## 3. The feature in one sentence

UPI checkout turns "pay this merchant" into either a deep link that opens your bank-authorised app with the amount pre-filled (intent) or a push request that lands in your app as a notification you approve (collect), and then rides an asynchronous switch that resolves who you are, moves the money, and, when the confirming reply gets lost, still settles the truth through deemed success and overnight reconciliation.

---

## 4. Jobs to be done

- "Let me pay without typing a card number on a phone." The core hire. UPI is address-based, not card-number-based.
- "Pre-fill the amount and the payee so I cannot fat-finger it." The intent flow exists almost entirely for this.
- "Tell me the truth about whether it worked, and do not charge me twice if the wifi hiccups." The reliability job, which is most of the engineering.
- "Do not make me wait more than a couple of seconds." UPI's own rule is an end-to-end response in under 2 seconds. The user hired speed too.
- For the merchant: "Confirm the order the instant money is truly mine, and reconcile every paisa at night so my books are provable."

---

## 5. How it works for the user

Collect (the website case). Aarav picks UPI, maybe types or picks his UPI ID (aarav@okhdfcbank), and presses pay. A few seconds later his Google Pay buzzes with a notification: "Merchant XYZ requests ₹499." He opens it, sees the amount already filled, enters his 6 digit UPI PIN, and the app says success. The website spinner, which was waiting the whole time, flips to "Order confirmed."

Intent (the QR / app case). Aarav taps a UPI button or scans a QR. His phone does not ask him to type anything. It directly opens Google Pay with the merchant and the ₹499 already filled in. He enters his PIN. Done. No notification to hunt for, no ID to type.

The visible difference is one screen. In collect, Aarav waits for a notification and has to go find it. In intent, the app just opens. That one screen is worth a lot: intent flows benchmark around 92 to 95 percent success, roughly 10 to 15 percent higher than collect, mostly because there is no typed ID to get wrong and no notification to miss or ignore.

---

## 6. The actual flow, step by step

Collect, tap by tap:

1. Aarav presses "Pay ₹499" and picks UPI on the Razorpay checkout.
2. Razorpay creates a payment record with a unique id (order_id / payment_id) and fires a collect request into UPI naming the merchant's VPA, Aarav's VPA, and ₹499.
3. NPCI routes that collect to Aarav's PSP (the bank behind Google Pay). His phone buzzes.
4. Aarav opens the notification, sees ₹499, enters his UPI PIN.
5. His PIN is verified by his bank's system, his bank debits ₹499, NPCI credits the merchant's bank, and a success flows back.
6. Razorpay hears the result, updates the payment to "captured", and calls the merchant's server with a webhook. The website flips to "confirmed".
7. If Aarav ignores the notification, the collect expires. NPCI's default collect expiry is 3 minutes. The payment record moves to "failed / expired".

Intent, tap by tap:

1. Aarav taps the UPI app icon or scans the QR.
2. The button carries a UPI deep link: `upi://pay?pa=merchant@icici&pn=MerchantXYZ&am=499.00&tr=<txn ref>&cu=INR`.
3. Android's intent system (or iOS deep linking) hands that link to a UPI app. Google Pay opens with everything filled.
4. Aarav enters his PIN. Debit, credit, success, same as steps 5 to 6 above.

Notice: in intent, Razorpay does not have to know Aarav's VPA at all. The link carries the merchant address and the amount, and Aarav's own app supplies who he is. That is why intent has nothing to type and nothing to mistype.

---

## 7. Under the hood, like the engineer

This is the heart of it. Three hard things stack: addressing (who is aarav@okhdfcbank), the two-second synchronous path (move the money and reply), and the truth-repair path (what happens when the reply is lost). I will take them in order, then the scale story.

### 7a. Addressing: a VPA is a key, resolved by a mapper, not a scan

A card number encodes the bank in its first digits. A UPI address does not. `aarav@okhdfcbank` is a handle. Something has to turn that handle into an actual account at an actual bank. That something is the NPCI mapper.

The data structure is a distributed hash map: key = VPA (or the newer UPI Number, resolved as `upinumber@mapper.npci`), value = the routing needed to reach the account. Resolution is an O(1) keyed lookup, not a search over banks. When Razorpay's collect request enters UPI, NPCI's resolver exchanges `ReqAuthDetails` / `RespAuthDetails` messages with the payer's PSP to resolve the virtual address before any money logic runs. This is the same lesson as every other teardown in this ledger: turn "find the right account among hundreds of banks and billions of handles" into "look up a key." The mapper is the inverted index of identity.

Concrete: `aarav@okhdfcbank` resolves to an HDFC account handle; `aarav@okicici` to ICICI. The `@` suffix is the shard hint for which bank's PSP handles auth. Aarav never exposes his real 14 digit account number to the merchant. The handle is the privacy layer and the routing layer at once.

### 7b. The synchronous path: ReqPay, RespPay, and a 12 digit RRN

Once resolved, the actual money instruction is a `ReqPay` message from the payer's PSP to the NPCI switch, and a `RespPay` back. The switch is the post office (same shape as the WhatsApp ticks teardown): it receives, routes to the payer bank for debit and the payee bank for credit, and relays the result. Crucially the core switch is stateless, which is what lets NPCI run many of them behind a load balancer and scale horizontally. State lives in the banks and in the settlement records, not in the switch box that happens to handle this one message.

Every transaction gets a 12 digit RRN (Retrieval Reference Number). This is the join key for the entire lifetime of the payment: the debit at HDFC, the credit at ICICI, Razorpay's record, and tomorrow's settlement file all carry the same RRN. It is also the idempotency guard at the rail level: NPCI rejects a duplicate RRN, so a retried `ReqPay` cannot double-move money. Aarav pressing pay twice, or a switch re-sending on a timeout, collapses to one transaction because the RRN is the same.

The whole synchronous path has a hard budget: UPI targets an end-to-end response in under 2 seconds. That budget is why nothing heavy runs inline. VPA resolution is a keyed lookup. PIN verification is a check at the bank, not a round trip to a fraud model. The debit and credit are two atomic ledger operations at two banks. Anything that cannot fit in the budget is pushed off the hot path.

### 7c. The truth-repair path: deemed success, TCC, and overnight reconciliation

Here is the deep part, the reason UPI is a distributed-systems problem and not a form.

Picture the worst case. Aarav's HDFC account is debited ₹499. NPCI tells the merchant's bank to credit it. The credit happens, but the beneficiary bank's "yes, credited" reply does not come back to NPCI in time (its system is slow, a link flapped). Now three parties disagree: Aarav's money is gone, the merchant may or may not have it, and the switch has no confirmation. If UPI simply failed the transaction here, it would either lose the merchant a real payment or leave Aarav debited with nothing to show.

UPI's answer is deemed success. On a timeout from the beneficiary bank, NPCI treats the transaction as deemed approved and settles it the same as an approved one. The disagreement is recorded with a TCC code:

- TCC-102: the merchant account was credited online but the bank failed to respond online to the switch.
- TCC-103: the merchant account was not credited online and shall be credited post-reconciliation.

So "deemed" is not a guess. It is a promise with a paper trail: settle now on trust, then make the books match later. The later part is URCS, the UPI reconciliation and clearing system. In the next settlement cycle the beneficiary bank raises a TCC (confirming it credited) or a RET (return, it could not, reverse the money). NPCI's URCS matches these against the deemed transactions by RRN and either finalises the credit or reverses it back to Aarav. As of the Feb 2025 rules, chargebacks even auto-accept or auto-reject based on the TCC/RET the beneficiary bank raised, inside URCS, in the settlement cycle after the dispute.

This is the same spine as the Stripe balance/ledger teardown: the live number can be provisional, but there is an immutable, RRN-keyed record and a nightly re-derivation that makes it provably correct. Money that was debited but not credited comes back as a reversal, which is why Aarav sometimes sees "amount debited, will be reversed in 1 to 2 days." That message is the deemed-success machinery being honest with him.

### 7d. Where Razorpay sits, and why the gateway needs its own state machine (inference, well grounded)

Razorpay does not move the money. It orchestrates the request and, more importantly, it insulates the merchant from every one of the ugly states above. This part is inference from documented gateway behaviour, not Razorpay's private code.

A gateway must model a UPI payment as an explicit state machine, because "did it work" is genuinely unknown for a window of time:

`created -> pending (collect sent / intent opened) -> [authorized -> captured] | expired | failed | (debited, awaiting recon) -> reversed`

Why a state machine and not a boolean: the synchronous reply can be lost even when the money moved. So the gateway cannot trust one answer. It needs a reconciliation loop that polls a "get transaction status" API for any payment stuck in pending, because a UPI payment can flip from pending to success or to failure minutes later. Documented behaviour: webhook callbacks can lag by up to ~15 minutes, and there is an explicit status-fetch API precisely for the case where the callback never arrived.

Two safety rails fall directly out of this, both already familiar from the Stripe idempotency and webhooks teardowns:

- Idempotency on the merchant id. The gateway keys each attempt by a unique merchant transaction id and rejects duplicates, so a double-tap or a client retry does not create two payments. The RRN guards the rail; the merchant id guards the merchant.
- At-least-once webhooks, dedupe on the receiver. Razorpay delivers the "payment captured" event to the merchant's server with retries and signing (HMAC), and the merchant must dedupe by event id and treat the webhook as a nudge to go fetch authoritative status, never as ordered or exactly-once. A lost 200 means the merchant hears it twice.

The clean mental model: the rail (NPCI) guarantees the money is eventually made correct via deemed success and URCS; the gateway (Razorpay) guarantees the merchant eventually learns the correct outcome via a state machine, a polling reconciler, and idempotent signed webhooks. Neither layer trusts a single reply.

### 7e. The scale story at three tiers

Tier 1, about 1,000 payments a day (one small merchant). A single database table of payments and a synchronous "wait for the callback" is fine. You could almost run the reconciler by hand. The deemed-success and RRN machinery still exists at the NPCI level, but at this volume the gateway never feels it. The trap: everything looks like a simple request-response, so a naive engineer models it as a boolean and gets away with it. The bug is invisible until volume finds the lost-reply case.

Tier 2, about 100,000 payments a day (a growing platform). Now the lost-reply case happens hundreds of times a day, and the pending state is common, not rare. What breaks: synchronous waiting ties up a worker for the full collect window (up to 3 minutes) per payment, so the pool starves. The fixes: make the flow asynchronous (fire the collect, return, let the webhook or the poller resolve it), run a dedicated reconciliation job that sweeps pending payments against the status API on a timer, and shard the payments table by merchant so one busy merchant does not lock the others. This is also where one slow beneficiary bank starts producing a steady drip of deemed / reversed transactions that must be surfaced to merchants correctly, so the reversal path stops being theoretical.

Tier 3, UPI-wide, 10 million-plus and then billions. This is NPCI's problem, and the numbers are real: July 2026 saw 23.66 billion UPI transactions in one month, worth ₹29.88 trillion, about 737 million a day, on a system engineered for on the order of 100,000 API requests per second and hundreds of millions of users. Full-year FY25-26 was 24,162 crore transactions, up about 30 percent year on year. What breaks at this tier and how they survive it:

- The switch cannot be stateful, or you cannot add boxes. Solution: the core switch is stateless, so NPCI scales it horizontally behind load balancers and the mapper is a sharded, replicated key-value store.
- The database is the bottleneck, not the CPU. Solution: shard and replicate by bank / by key so a lookup and a write touch one shard, not the whole store; read replicas absorb the status-check reads.
- The 2-second budget forbids inline heavy work. Solution: settlement and reconciliation are batch, run in cycles overnight through URCS, off the hot path entirely. Deemed success exists precisely so the live path never blocks waiting for a slow bank; it settles now and reconciles in a later cycle.
- Serialization overhead adds up at billions of messages. Reported solution: compact binary formats (Avro / Protobuf style) rather than fat XML/JSON on the wire.

The invariant across all three tiers is the ledger's recurring one: the expensive truth-making (settlement, reconciliation, reversal) is done offline in cycles, and the live path is a cheap keyed operation (resolve the VPA, move the money, or read a status) that is allowed to be provisional because a correct offline process will fix it.

---

## 8. The retention and habit mechanic

UPI's retention is not a dopamine loop, it is a habit that hardens into a reflex, and Razorpay's slice of it is trust plus success rate.

For the end user, the loop is: pay by UPI, it works in two seconds, no card, no OTP page, no double charge. Do that twice a day for groceries, chai, autos, and rent, and reaching for UPI becomes automatic. The habit is manufactured by removing every reason to hesitate. The reversal message ("debited amount will be returned in 1 to 2 days") is part of this: even the failure case is handled visibly and made whole, so the user never learns to fear the button. That is the deemed-success and URCS machinery doubling as a retention mechanic. India ran 737 million of these a day in mid-2026 because the button is boring and reliable.

For the merchant, and this is where Razorpay's business lives, the metric is success rate, and it moves revenue directly. A failed checkout is an abandoned cart. This is why intent flows matter commercially: pushing users to the intent door (app opens pre-filled, ~92 to 95 percent success) instead of the collect door (type an ID, hunt for a notification, ~10 to 15 percent lower) is worth real percentage points of completed payments. A gateway that squeezes success rate higher, by defaulting to intent, by retrying intelligently, by resolving pending payments fast so the merchant confirms orders sooner, keeps the merchant on its rails. Razorpay publicly markets exactly this: case studies of lifting a merchant's UPI success rate into the high 80s. Higher success rate is the switching cost. The metric moved is revenue (merchant) and retention (both sides).

---

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph to a shader and runs it in an embeddable runtime on someone else's site. When a creator hits "publish" or when the embedded runtime asks the backend to compile-and-cache a variant, you have the exact UPI problem: an operation whose true outcome is not knowable inside the request window, across systems you do not fully control (the CDN, the client GPU, the tenant's page).

The concrete lesson: do not model publish/compile as a synchronous boolean. Model it as a UPI-style state machine with deemed success and a reconciler.

1. Give every publish a single stable id (your RRN) that every downstream system carries: the compile job, the CDN artifact, the runtime cache entry, the audit log. One key, joinable end to end, so you can always reconcile "did this version actually land everywhere" later.
2. Keep the live path under a hard budget and let it be provisional. When the runtime requests a shader variant, resolve it as a keyed lookup (graph-hash + device-bucket, the same capability-bucket idea from earlier teardowns). If the exact variant is not warm yet, return a known-good fallback variant immediately and mark the precise one "deemed pending", exactly as UPI settles on trust and reconciles later. Never block a frame waiting for a cold compile.
3. Build the reconciler as a first-class background loop, not an afterthought. Sweep every "pending" compile against real signals (did the CDN store it, did a real device report it hit 60fps), and finalise or reverse. Reversal here means: mark the variant bad, fall the tenant back to the previous good version, and record why. This is UPI's TCC/RET applied to shader artifacts.
4. Make every publish-to-embed notification at-least-once, signed, idempotent, keyed by (graph_id, version). The embed dedupes and treats it as a nudge to fetch authoritative state, never as ordered or exactly-once. That is the webhook lesson, and UPI's collect callback (which can lag 15 minutes) is the same warning.

The one-line version: the rail makes the money eventually-correct with deemed success and nightly reconciliation, and the gateway makes the merchant eventually-informed with a state machine and idempotent webhooks. Rare.lab should make the artifact eventually-correct (a reconciler over graph-hash-keyed compiles) and the embed eventually-informed (idempotent signed pushes), so a lost reply or a cold cache never becomes a broken frame or a lost publish.

---

## Sources

- NPCI monthly UPI volume, July 2026 (23.66 billion transactions, ₹29.88 trillion): Business Standard, "UPI transactions hit record 23.66 billion in July." https://www.business-standard.com/finance/news/upi-transactions-hit-record-23-66-billion-in-july-value-rs-29-88-trillion-126080100548_1.html
- UPI FY25-26 annual volume and growth, May 2026 record: ANI News. https://www.aninews.in/news/business/upi-hits-new-high-in-may-2026-with-232-billion-transactions-worth-rs-299-trillion-npci-data-shows20260602155337/
- UPI intent vs collect success rates (92 to 95 percent, +10 to 15 percent), mechanism: Razorpay Blog, "UPI Intent vs Collect." https://razorpay.com/blog/upi-intent-vs-collect-success-rates/
- UPI collect vs intent business guide (collect push flow, VPA validation): Razorpay Blog. https://razorpay.com/blog/upi-collect-vs-intent-flow-business-guide/
- Deemed success, payout states, NPCI reconciliation (created / processing / deemed / processed / reversed): RazorpayX Blog, "Payout Processing for IMPS and UPI: Deemed Success and NPCI." https://razorpay.com/blog/business-banking/payout-processing-imps-upi-transactions-deemed-success-npci
- TCC-102 / TCC-103 deemed codes and beneficiary-bank timeout settlement: RMA Bhutan, "Guidelines for UPI Global QR 2024." https://www.rma.org.bt/media/Laws_By_Laws/Guidelines%20for%20UPI%20Global%20QR%202024.pdf
- URCS auto chargeback accept/reject on TCC/RET in next settlement cycle (Feb 2025 rule): Business Standard. https://www.business-standard.com/finance/personal-finance/new-upi-rule-on-automatic-acceptance-rejection-of-chargebacks-from-feb-15-125021200820_1.html
- ReqPay / RespPay / ReqAuthDetails resolution and mapper, NPCI API protocol: NPCI API Descriptions (public PDF). https://s3-ap-southeast-1.amazonaws.com/he-public-data/NPCI%20API%20Descriptionsb9bceb7.pdf
- UPI Number mapper resolution (`upinumber@mapper.npci`): PayU docs, "UPI Number Mapper API." https://docs.payu.in/reference/upi-number-mapper-api
- UPI error and response codes, RRN and duplicate-RRN handling: NPCI "UPI Error and Response Codes v2.9" (public PDF). https://dth95m2xtyv8v.cloudfront.net/tesseract/assets/upi-tpap-sdk/UPI_Error_and_Response_Codes_2_9-HHLrJ.pdf
- Stateless switch, horizontal scaling, sharding, ~100,000 req/sec, Avro/Protobuf: Medium, "The Data Behind UPI: Real-Time Financial Pipelines" (Ankit Gochhayat) and "NPCI (UPI) handles ~600M users daily and up to 100,000 API requests/sec." https://medium.com/@ankit.gochhayat90/the-data-behind-upi-understanding-the-architecture-of-real-time-financial-pipelines-e8d5c6f89b16
- Collect expiry default 3 minutes, callback lag up to 15 minutes, status-fetch API, idempotency on merchant txn id: UPI integration guides (Decentro / Happay webhook docs / NxtBanking guide). https://webhooks.isac.happay.in/upi-webhooks/upi-payments-webhooks
