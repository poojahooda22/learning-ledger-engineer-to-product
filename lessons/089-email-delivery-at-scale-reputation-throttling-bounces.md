# Day 89: How do you send 50 million emails on a sale day and still land in the inbox, when the receiver, not you, decides how fast you are allowed to go? Email delivery at scale: authentication, reputation, per-domain throttling, bounces

**Date:** 2026-10-15
**Difficulty:** Advanced (a system where the downstream is an outside party with its own rules, and your own past behavior is state that limits your throughput)
**Topic:** Earlier lessons covered queues (Day 9), backpressure (Day 13), webhooks (Day 83) and timers (Day 87). Email is the same shape with one twist: the receivers (Gmail, Yahoo, Outlook) rate-limit you by how much they trust you, and trust is slow to earn and fast to lose.
**Stack relevance:** Rare.lab sends auth emails through Supabase and will soon send invites, share links and render-complete notices. All of it goes through someone else's sender reputation unless you decide otherwise.

---

## 0. The framework (same recipe as Days 74, 86, 87, 88)

The LeetCode mock interview in your prompt follows one recipe. Full walk-through is in [Day 74](074-leetcode-online-judge-end-to-end.md). Short version:

1. Functional requirements plus an out-of-scope list.
2. Non-functional requirements (availability vs consistency, latency, scale, isolation).
3. Core entities, then API routes.
4. Simplest high-level design, one requirement at a time.
5. Break it, then optimize. The interviewee's move "API server runs the code = bad, VM = heavy, container + queue = right" is the same move as "web server sends the email = bad, queue + isolated sender fleet = right".
6. Say the trade-off out loud every time.

Today's recipe:
- Functional: accept a "send this email" request, deliver it, tell the caller what happened (delivered, bounced, complained), honor unsubscribes.
- Non-functional: transactional mail (password reset) arrives in seconds; bulk mail never harms the transactional lane; complaint rate stays under 0.3 percent; no duplicates when we retry.
- Out of scope: composing UI, template editors, inbound parsing.
- Core entities: Message, Recipient, Sending identity (domain + IP), Suppression entry, Event (queued, deferred, delivered, bounced, complained).
- API: `POST /messages` (with an idempotency key, Day 12), `GET /messages/{id}`, webhook out for events, `POST /unsubscribe`.
- Naive design: the web handler opens an SMTP connection and sends. We will break it below.

---

## 1. The company and the breaking number

**The real rule change.** Since February 2024, Google (Gmail) and Yahoo require senders of roughly **5,000 or more messages a day** to Gmail to: authenticate with SPF and DKIM and publish DMARC, support one-click unsubscribe (RFC 8058) on marketing mail and honor it within 2 days, and keep the user-reported **spam rate under 0.3 percent**, with **0.1 percent** as the healthy target. (Confirmed by vendor summaries of the rule, see sources. I did not open Google's own page this session.)

**The breaking number (my arithmetic, a worked example, not a company figure):**

- A retailer sends a flash-sale email to **50,000,000** subscribers.
- Gmail alone is a large share. Say 20,000,000 go to Gmail.
- If Gmail lets a trusted sender open, say, 50 parallel connections each pushing 100 messages per connection, and each connection cycle takes 2 seconds, that is 50 x 100 / 2 = **2,500 messages per second**, or 20,000,000 / 2,500 = **8,000 seconds, about 2.2 hours** for Gmail alone. (The 50 and 100 are illustrative; real limits are not published and change per sender.)
- Now the reputation twist. The sale email is clicked "Report spam" by 1 in 200 people. That is **0.5 percent, above the 0.3 percent ceiling**. Gmail starts deferring and filtering you. Retries pile up. Your next email, the password reset a user is waiting for, is stuck behind the same pile if it shares the same IP and queue.

Analogy: you are a delivery driver and every building (Gmail, Yahoo, Outlook) has its own doorman. Each doorman decides how many parcels per hour he takes from you, based on how many of your past parcels the residents threw away. Drive faster and the doorman slows you down further.

---

## 2. Why the naive design dies

Naive design: the web request handler calls `smtp.send(to, body)` directly using the cloud provider's shared IP.

- **Blocking the request path.** One SMTP exchange to a distant server takes 0.3 to 3 seconds. At 2,500 sends/sec you need thousands of open handlers. A slow receiver stalls your checkout page.
- **No memory of failure.** A "4xx temporary failure" (try later) is treated as an error and dropped, or retried instantly in a loop. A "5xx permanent failure" (mailbox does not exist) is retried forever. Bounce rate climbs, reputation falls.
- **Shared IP, shared fate.** Marketing and password resets leave from one IP. When marketing earns complaints, the IP is throttled, and password resets queue behind it. A real example of the pain: a user waits 20 minutes for a "reset password" mail and gives up.
- **Cloud shared IPs are often pre-blocked.** Big receivers distrust brand-new or cloud-hosted IPs with no history (inference, widely reported by deliverability vendors; not a single primary source).
- **Duplicates on retry.** Request times out after the SMTP server accepted the message. Handler retries. Customer gets two "your order shipped" emails.

---

## 3. The architecture, drawn top to bottom

```
Clients / app servers  (checkout, auth, marketing tool)
        |  POST /messages  {to, template, idempotency_key, lane: transactional|bulk}
        v
Edge API (stateless)
   - validates, checks suppression list, checks idempotency key, writes message row
        |
        v
Durable queue(s), separated by LANE
   transactional queue         bulk queue
        |                          |
        v                          v
Scheduler / rate governor  (the brain)
   - one logical queue per (recipient domain, sending IP pool)
   - token bucket per queue: msgs/sec and max concurrent connections
   - slows down on 4xx deferrals, speeds up on success
        |
        v
MTA fleet (Mail Transfer Agents: Postfix/Haraka/custom)
   - signs with DKIM, opens TLS SMTP to the receiver's MX host
   - reuses connections (many messages per connection)
        |
        v
Receiver (Gmail / Yahoo / Outlook)  ---- replies 250 OK | 4xx defer | 5xx bounce
        |
        v
Event pipeline
   - response codes + async bounces (DSN) + feedback loop (complaints) + webhooks (opens/clicks)
   - updates message state, suppression list, reputation dashboards
        |
        v
Reputation & compliance layer
   - suppression list, unsubscribe endpoint, Postmaster Tools / feedback-loop monitoring
```

| Layer | Single job | Analogy |
|---|---|---|
| **Edge API + idempotency key** | Accept once, never twice | A ticket stub so the same order cannot be filed twice |
| **Lane-separated queues** | Keep urgent mail away from bulk mail | A priority lane and a cargo lane at the airport |
| **Per-domain rate governor** | Send at the speed each receiver tolerates | Each doorman gets his own conveyor belt with its own speed |
| **MTA fleet** | Speak SMTP, sign DKIM, reuse connections | The truck fleet |
| **Sending IP pools** | Isolate reputation (transactional IPs vs bulk IPs) | Separate licence plates for the ambulance and the junk-mail van |
| **Event pipeline** | Turn receiver replies into state | The return-receipts mailbox |
| **Suppression list** | Never email a bad or opted-out address again | The "do not knock" list |

Vocabulary:
- **MTA**: Mail Transfer Agent, the server that speaks SMTP to other mail servers.
- **MX record**: DNS record naming which server receives mail for a domain.
- **SPF**: DNS list of IPs allowed to send for your domain. **DKIM**: a cryptographic signature on each message, public key in DNS. **DMARC**: policy saying what to do when SPF/DKIM do not align with the From domain, and where to send reports.
- **Soft bounce / deferral (4xx)**: try again later. **Hard bounce (5xx)**: do not retry; suppress.
- **Feedback loop (FBL)**: the receiver tells you when a user clicked "Report spam".
- **IP warming**: starting a new IP at low volume and ramping, so receivers learn you are not a spammer.

---

## 4. The transferable mechanisms

### 4.1 Queue per downstream, not one global queue (bulkheads)

One global FIFO means one slow receiver blocks everyone. Instead keep a logical queue **per recipient domain (and per IP pool)**. A Gmail slowdown only backs up the Gmail queue; Outlook flows at full speed. This is the bulkhead idea from Day 34 (cell-based architecture) and the head-of-line blocking fix from Day 9.

Walked example: 3 queues, `gmail.com` (20M), `outlook.com` (8M), `yahoo.com` (5M). Gmail starts returning `421 4.7.0 try again later`. The governor halves Gmail's rate; Outlook and Yahoo finish on schedule. The sale email reaches 60 percent of people on time instead of 0 percent.

### 4.2 Adaptive rate control (AIMD) driven by the receiver's replies

The receiver tells you its limit through the response code. Treat each queue like TCP congestion control: **additive increase** while replies are `250`, **multiplicative decrease** (cut by half) on `421/451` deferrals. Mail transfer agents in the wild expose per-domain throttles in messages or bytes per minute and connection counts (MailerQ documents this style of per-domain/per-IP limits). The mechanism is general: when the downstream cannot be configured, let it steer you with its error signals (Day 13 backpressure, Day 84 control loops).

### 4.3 Lane separation: protect the transactional path

Two lanes, two IP pools, often two subdomains (`mail.example.com` for receipts, `news.example.com` for marketing). Marketing earns complaints; transactional mail earns opens. Keeping them apart means a bad campaign cannot delay a password reset. This is priority isolation, the same idea as Day 67 noisy neighbors.

### 4.4 Reputation is state; warm it like a cache (IP warming)

A new IP has no history. The standard practice is a ramp over roughly three weeks, for example from about 50 per day to about 1,000,000 per day with automatic slowdown on trouble, and a hard stop if bounces pass about 3 percent (one SendGrid-based provider's public warm-up page describes exactly this ramp; it is a vendor page, not SendGrid engineering). Mechanism: when a downstream scores you on history, you cannot burst to full load; you ramp, and you send to your **most engaged** users first so early signals are good.

### 4.5 Idempotency and the suppression list (exactly-once-ish)

- **Idempotency key** on `POST /messages`: same key within 24 hours returns the same message id, never a second send (Day 12). SMTP itself has no exactly-once; if a connection drops after `DATA` but before the reply, you do not know if it was accepted. You accept a tiny duplicate risk and prefer it over silent loss (at-least-once).
- **Suppression list**: a hard bounce, a complaint or an unsubscribe writes the address to a per-tenant list that the Edge API checks **before** queueing. Sending to bad addresses is the fastest way to wreck reputation, so this check is a gate, not a report.

### 4.6 Authentication as a precondition, not a feature

SPF + DKIM + DMARC alignment is how the receiver knows the message is really yours. Without it you are not just filtered, you are rejectable. DKIM signing is CPU work done by the MTA per message (sign the headers and body hash), so it scales horizontally with the fleet. Rotate DKIM keys with selectors: publish the new key in DNS under a new selector, switch signing, retire the old one later (an expand-then-contract migration like Day 58).

### 4.7 One-click unsubscribe is a throughput feature

RFC 8058 adds `List-Unsubscribe` and `List-Unsubscribe-Post` headers so the mail client shows an "Unsubscribe" button that makes a single POST to you. A user who can unsubscribe in one tap does not hit "Report spam". Every spam report you avoid protects the 0.3 percent budget, which protects your sending rate. Handle the POST idempotently and honor it within 2 days (the stated requirement).

---

## 5. The trade-offs

**Consistency vs availability, per data type:**

| Data | Choice | Why |
|---|---|---|
| Message accepted by API | Durable write before returning 202 (consistency) | Losing a "receipt" email silently is worse than a 50 ms write |
| Suppression list | Strongly consistent on write, cached on read with short TTL | Emailing someone who unsubscribed one second ago is a legal and reputation risk |
| Delivery status shown to the user | Eventually consistent | Bounces can arrive minutes later; "sent" is not "delivered" |
| Open and click counts | Approximate, async | Apple Mail privacy features inflate opens, so they are weak signals anyway (widely reported; treat as inference) |

**Cost vs latency:**
- Dedicated IPs cost money and need warming; shared pools are cheaper but share fate with strangers. Rule of thumb many vendors state: dedicated IPs only make sense with steady volume. (Vendor guidance, not a measured fact.)
- Lower latency for transactional mail means over-provisioning that lane (idle capacity). Bulk lane can run at 100 percent utilization all night.
- Faster bulk sending costs reputation. Sending the sale email over 2.2 hours instead of 20 minutes is cheaper than being filtered.

**The choice email makes:** at-least-once, durable, slowly and politely. Throughput is governed by the receiver's trust, not your hardware.

---

## 6. The systems-thinking lens

**The feedback loop: the retry-and-reputation death spiral.**

1. A campaign sends fast to a list with stale addresses and sleepy users. Bounces and spam reports rise.
2. The receiver lowers your trust and starts deferring (4xx) or routing to spam.
3. Your queue grows because deferred messages are retried. Retries add volume against an already-angry receiver.
4. Messages age past the point they matter (a "flash sale ends in 1 hour" mail arrives after it ends); users who see junk-folder mail click spam.
5. Spam rate rises. Trust falls further. Loop closed. Meanwhile the transactional lane, if it shares the IP, is trapped inside the same loop.

How the senior fix **breaks** the loop instead of adding servers:
- **Never retry harder against a slowing downstream**: exponential backoff with jitter per domain, and an overall retry window (for example give up after 24 to 72 hours; RFC 5321 suggests giving up after at least 4 to 5 days for ordinary mail, but marketing mail should expire much sooner because its value decays).
- **Message TTL tied to business value**: expire a flash-sale email after its sale ends rather than delivering it late.
- **Gate at the source**: the suppression check and "send only to engaged users first" reduce bounces and complaints before they happen.
- **Circuit breaker per domain**: when a domain's deferral rate passes a threshold, pause that queue and alert, rather than hammering it.
- **Isolate lanes and IP pools** so the loop cannot spread.
- **Alert on leading indicators**: complaint rate vs 0.1 percent target, hard bounce rate, deferral rate per domain, queue age of the oldest transactional message.

---

## 7. Mapping to Rare.lab

Rare.lab: Supabase Postgres with RLS, Cloudflare R2 (immutable content-addressed scene JSON plus manifest), embeddable runtime with one shared WebGL context.

**Already doing, whether named or not:**
- **Immutable, content-addressed share links.** A "share this scene" mail can carry a URL to an immutable hash, so a retry or a late-opened email never shows a different scene than the one intended. That is the idempotent-content idea (4.5) at the object layer.
- **Supabase Auth emails.** You already depend on a sender. Supabase's own docs advise configuring a custom SMTP provider for production because the default sender is rate-limited (I did not open the page this session; verify the current limit in the Supabase docs before launch).

**Where this bites next (my inference from your stack description, not from reading your code):**

| Risk | Why it matters | Action |
|---|---|---|
| **Auth and invite emails on the default sender** | Hard rate limit and shared reputation; a sign-up spike silently drops codes | Move to a dedicated provider via custom SMTP now; separate subdomain such as `auth.rare.lab` |
| **No SPF/DKIM/DMARC on your domain** | Mail lands in spam or is rejected; also lets others spoof you | Publish all three; start DMARC at `p=none` with reports, then tighten |
| **Marketing and transactional on one domain** | One bad launch email hurts password resets | Two subdomains, two streams, even if the same vendor |
| **"Render complete" or "comment" notifications** | A burst of 10,000 renders finishing at once becomes 10,000 emails | Put them through a queue with per-user digest batching (Day 53), not a direct send in the render worker |
| **Unsubscribe and complaint handling** | The 0.3 percent ceiling applies once you pass about 5,000 a day to Gmail | Add List-Unsubscribe headers, a suppression table in Postgres (unique on tenant+address), and a webhook consumer for bounces |

**Next ceiling:** not sending speed, but sender reputation on a new domain at launch. Plan a 3-week warm-up for launch-day volume instead of mailing the whole waitlist at once.

**One-line lesson for Rare.lab:** treat email as a queue into a downstream you do not control: separate transactional from marketing on their own subdomains, authenticate with SPF/DKIM/DMARC, throttle per receiving domain, and never let a bulk send share a lane with a password reset.

---

## 8. Sources and what is actually in them

**Honest note on access:** this session I used web search summaries only and did not open the full pages. The search did not surface SendGrid, Mailgun or Postmark engineering posts, so internals of those providers are NOT sourced here; the per-domain queue, AIMD and lane designs are my synthesis of standard MTA practice, labeled as inference. Section 1 numbers are my own arithmetic. Verify against Google's and Yahoo's own pages before relying on the thresholds.

- [Bird: Gmail and Yahoo sender requirements](https://bird.com/docs/knowledge-base/deliverability/gmail-yahoo-requirements). Plain language: a sending-platform vendor's summary of the 2024 rule: authenticate, one-click unsubscribe on marketing mail within 2 days, keep spam complaints low.
- [Bird: what are the Gmail and Yahoo bulk sender requirements](https://bird.com/explained/deliverability/what-are-the-gmail-and-yahoo-bulk-sender-requirements). Plain language: the same rules in explainer form; says DMARC policy `none` is acceptable as long as the From domain aligns with SPF or DKIM.
- [Close: Requirements for bulk email senders](https://help.close.com/technical-support/email-deliverability/requirements-for-bulk-email-senders). Plain language: a CRM vendor's help page; gives the 0.3 percent spam ceiling, 0.1 percent target, Postmaster Tools and Yahoo complaint feedback loop, and the seven-day recovery detail.
- [Warmy: Google and Yahoo sender requirements](https://www.warmy.io/guides/google-yahoo-sender-requirements/). Plain language: a deliverability-tool vendor's guide; treats passing both SPF and DKIM as the bulk-sender bar (slightly stricter than the Bird reading, so check the primary source).
- [MailerQ: email throttling](https://www.mailerq.com/blog/email-throttling-with-mailerq). Plain language: how a commercial MTA lets you cap messages, bytes and connections per recipient domain or IP. This is the closest thing in the results to the per-domain governor in section 3.
- [Suped: handling throttling from new IPs and recipient domains](https://www.suped.com/email-deliverability/answers/what-are-the-best-practices-for-handling-throttling-from-new-ips-and-recipient-domains). Plain language: practitioner advice: slow down per ISP when you see connection limits or specific bounce codes; remove invalid addresses promptly after 5xx errors.
- [SiteVibes: Understanding our IP warming process](https://support.sitevibes.com/support/solutions/articles/47001271345-understanding-our-ip-warming-process). Plain language: a vendor's public ramp: about three weeks from roughly 50 a day to about 1,000,000 a day, automatic slowdown on problems, halt when bounces exceed 3 percent.
- Standards worth reading directly (from my own knowledge, not fetched today): RFC 5321 (SMTP, including retry guidance), RFC 7208 (SPF), RFC 6376 (DKIM), RFC 7489 (DMARC), RFC 8058 (one-click unsubscribe).
- Video context: the LeetCode mock interview in your prompt (Day 74 has the full summary). Takeaway for today: its fix "put a queue between the API and the workers, add exponential retries" is exactly right, and email adds the missing piece: the retry policy must be per downstream and capped, because the downstream punishes you for retrying.

**Confirmed (from search summaries):** the 5,000/day threshold, 0.3 percent spam ceiling and 0.1 percent target, one-click unsubscribe within 2 days, SPF/DKIM/DMARC requirement; per-domain/per-IP throttling exists in commercial MTAs; a roughly three-week IP warm-up ramp with a 3 percent bounce stop at one vendor.
**Inference (mine):** the exact queue topology, AIMD controller, TTL policy, lane/subdomain advice, and every number in section 1.

**Related lessons:** Day 9 (queue as shock absorber), Day 12 (idempotency), Day 13 (backpressure), Day 53 (push fan-out and digests), Day 67 (noisy neighbors), Day 83 (webhook delivery), Day 87 (timers and delay queues).
