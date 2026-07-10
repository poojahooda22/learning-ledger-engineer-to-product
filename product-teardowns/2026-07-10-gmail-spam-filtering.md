# Gmail spam filtering: the Spam folder, "Report spam," and the classifier behind them

Date: 2026-07-10
Product: Gmail
Feature: Spam filtering (the Spam folder, the connection-time reputation checks, and the ML text classifier)

A note on shape. Most of this ledger is "match then rank" (search, recommendations, dispatch). Spam filtering is a different animal. There is no catalog to fetch candidates from and nothing to sort. It is a yes/no decision made on every single message under an adversary who edits the input on purpose to fool you. So this teardown wears a different coat: cheap-filter-first, then features-then-model, under a false-positive rule that is close to sacred, against a moving target. It is the cousin of the Stripe Radar fraud teardown (2026-07-04), but with a faster feedback loop and a text-obfuscation twist that produced one of Google's cleverest recent pieces of engineering, RETVec.

---

## 1. The user

Priya opens Gmail at 8:02am on her phone, coffee in one hand, before the 8:15 standup. She has 14 new emails overnight. She wants the two that matter: the Razorpay settlement report and a reply from her landlord. She does not want to think about the other twelve. She taps into the inbox, sees exactly those two plus a newsletter she half-reads, and moves on. She never opens the Spam folder. That is the whole point: the feature worked precisely because she did not notice it working.

Somewhere in the twelve that did not reach her inbox was an email that said "Yоur Аpple ID has been locked, verify at paypa1-secure.com" with the account name spelled using Cyrillic letters that look Latin. If that had landed in her inbox at 8:02am, half-awake, she might have tapped it.

## 2. The real problem

Around half of all email sent in the world is spam. Recent traffic estimates put it near 47 to 52 percent, on the order of 170 billion junk messages a day across the whole internet ([Statista spam share](https://www.statista.com/statistics/420391/spam-email-traffic-share/), [EmailTooltester 2026](https://www.emailtooltester.com/en/blog/spam-statistics/)). If Gmail did nothing, one in every two things in Priya's inbox would be garbage, and a slice of that garbage is not just annoying, it is a scam that costs real money or steals her password.

Described like a friend would: your mailbox is a public street corner. Anyone on Earth can walk up and drop a letter, for free, with no ID check. Most days it is fine. But every day thousands of strangers also drop letters designed to look exactly like the one from your bank. The job is to let the real bank letter through and quietly bin the fakes, on every letter, without ever once binning the real bank letter. That last clause is the hard one.

## 3. The feature in one sentence

Gmail scores every incoming message for how spammy or dangerous it is, and routes the bad ones to a Spam folder instead of the inbox, learning continuously from what senders do and from what you tap.

## 4. Jobs to be done

- "Give me back my inbox." Keep the 50 percent junk out so the real mail is findable.
- "Do not let me get scammed at 8am." Catch phishing and malware, not just ads for weight-loss pills.
- "Never lose my real mail." If my landlord's reply lands in Spam, the feature has failed worse than if it let one ad through.
- "Do it silently and instantly." No CAPTCHA, no quarantine review queue, no waiting. The message is already sorted before I look.

## 5. How it works for the user

Priya sees almost nothing, and that is the design. The visible surface is thin:

- Two folders: Inbox and Spam. Bad mail is in Spam, which auto-deletes after 30 days.
- A "Report spam" button (the little stop sign) in the inbox, and a "Not spam" button in the Spam folder.
- Occasional yellow or red banners: "This message seems dangerous," "Be careful with this message," or "This message was not sent by the person it claims to be from."
- A tiny "Why is this message in Spam?" line at the top of a spam message that gives a one-sentence reason ("It is similar to messages that were detected by our spam filters").

That is the entire user-facing feature. Everything else is machinery she never sees.

## 6. The actual flow, step by step

Follow one real message: the fake Apple/PayPal phishing mail aimed at Priya.

1. A sending server somewhere opens a TCP connection to Gmail's mail servers and says "I have mail for priya@gmail.com."
2. Before the body is even accepted, Gmail looks up the sender's IP address and domain reputation. This is the cheapest, earliest gate. If the IP is a known spam cannon, the connection can be refused or deferred right here, and the body is never read.
3. If the sender is not obviously bad, the message body is accepted. Authentication checks run: does the mail pass SPF, DKIM, and DMARC, which prove the sender really controls the domain it claims? The phishing mail claims to be Apple but is sent from a domain registered yesterday. DMARC fails.
4. The message is now turned into features: header signals, the sending domain's age and reputation, the URLs in the body (paypa1-secure.com), whether the body is an image with no text, and the text itself.
5. The ML classifier scores it. The obfuscated text "Yоur Аpple ID" is fed through the vectorizer, which recognizes it as "your apple id" despite the Cyrillic swaps, so the impersonation signal fires. The score comes back high.
6. The score crosses the spam threshold. The message is filed in Spam, tagged with a "looks like phishing" reason, and a red banner is attached in case Priya ever opens it.
7. Priya never sees it. Two weeks later it auto-deletes.

Now the other direction, the correction loop:

8. A newsletter Priya actually wants got filed in Spam by mistake. She opens Spam, taps "Not spam." That message moves to the inbox, and that click becomes a training label: "this sender, to this user, is wanted." Future mail from that sender to her is more likely to land in the inbox.

## 7. Under the hood, like the engineer

This is the heart. Spam filtering is not one model. It is a funnel of filters ordered cheapest-first, ending in a machine-learned classifier, wrapped in a feedback loop against an adversary. Let me build it up the way the load forces you to.

### The core reframe: this is classification, and the false-positive cost is asymmetric

Search ranks. Spam decides. Every message gets a probability P(spam), and a threshold turns that number into an action: deliver, deliver-with-warning, or send to Spam. The two errors are not equal. A false negative (a spam ad reaches the inbox) is a minor annoyance Priya clears in one tap. A false positive (her landlord's reply, or a job offer, or a Razorpay settlement, lands in Spam) can cost her a house or a paycheck, and she may never find it. So the whole system is tuned to keep false positives extremely low even at the cost of letting some spam through. The threshold is the real product dial, exactly like the block-vs-insult tradeoff in the Stripe Radar teardown. This is why Gmail leans toward "when unsure, deliver."

### Layer 0: connection-time reputation, the cheapest filter, does the heavy lifting

Gmail receives on the order of 121 billion messages a day across roughly 1.8 billion users ([Gmail statistics 2026](https://www.demandsage.com/gmail-statistics/)). You cannot afford to run a neural network on all of that. So the first gate is the cheapest possible check, done before the message body is even accepted: the reputation of the sending IP address and domain.

The data structure here is a reputation store: a hash map (and in practice Bloom-filter-style membership structures for the blocklists) keyed by IP and by domain, answering "have we seen abuse from this source" in microseconds. The industry baseline for this is the DNS blocklist (DNSBL). Spamhaus runs the best-known public ones and describes them plainly as "your first line of defense" ([Spamhaus DNSBL basics](https://www.spamhaus.org/resource-hub/email-security/dns-blocklist-basics/)). Gmail uses public DNSBLs plus its own far larger internal reputation signals built from the whole fleet's history.

Why this ordering matters: if half of all incoming volume is spam and much of it comes from known-bad IPs, this O(1) lookup sheds a huge fraction of the load before you spend a single feature-extraction cycle on it. It is the same principle as pruning a search branch before you rank it. Concrete example: a botnet blasting from a range of residential IPs that Spamhaus flagged an hour ago gets rejected at the door, and the expensive classifier never sees those millions of messages.

This is the "matching half" analog. In search, the cheap wide filter is the inverted-index posting-list merge. In spam, the cheap wide filter is reputation at connection time. Both exist so the expensive model only ever runs on a small, pre-filtered set.

### Layer 1: authentication and rules

For messages that get past reputation, cheap deterministic checks run next. SPF, DKIM, and DMARC are cryptographic and DNS-based proofs that the sender controls the domain in the From line. They are close to free to check and they catch a whole class of impersonation: our fake Apple mail fails DMARC because it is not actually sent from apple.com's authorized servers. Gmail has pushed hard on this, requiring bulk senders to authenticate. Alongside authentication sit heuristic rules, the oldest form of spam filtering, still useful as fast pre-filters even though they are brittle on their own.

### Layer 2: the machine-learned classifier, features then model

Now the message is a candidate that survived reputation and authentication. This is where the real learning happens, and it is a "features then model" problem, the same spine as fraud scoring.

The history is worth naming because it is a clean progression:

- Rules (1990s to early 2000s): humans write patterns like "contains the word Viagra." Spammers just misspell it. Brittle.
- Bayesian (2002): Paul Graham's "A Plan for Spam" made naive Bayes the standard. You compute P(spam) from the frequency of each word in known spam versus known ham. It learns from data instead of hand-written rules, and it can be per-user. Still the textbook baseline ([ScienceDirect ML spam review](https://www.sciencedirect.com/science/article/pii/S2405844018353404)).
- Linear ML, then deep learning: Google reported that adding neural networks moved Gmail's catch rate from 99.5 percent to 99.9 percent. That last 0.4 percent sounds tiny until you multiply by 121 billion.
- In 2019, Google added a TensorFlow-based deep model specifically to catch the hard tail, the less-than-0.1-percent of spam that everything else missed, blocking around 100 million additional messages a day ([Google Workspace blog, 2019](https://workspace.google.com/blog/product-announcements/ridding-gmail-of-100-million-more-spam-messages-with-tensorflow)). Google named exactly what it catches that rules and older models missed: image-based messages with no real text, emails with hidden embedded content, and mail from newly created domains that hide a low volume of spam inside otherwise legitimate-looking traffic. They monitor training with TensorBoard to iterate models quickly, which matters because the adversary keeps moving.

The features fed to this model: sender and domain reputation scores (carried down from Layer 0), header anomalies, the URLs and their reputations, structural signals (is the body just one big image, is there hidden text set to font-size zero or white-on-white), and above all the text content.

### The adversarial text problem, and RETVec (the deepest and freshest idea here)

Text is where spammers fight hardest, because text is what carries the scam. And text is where standard machine learning is most fragile.

Here is the failure. Every normal text model starts by tokenizing: chop the string into known words or sub-word pieces from a fixed vocabulary, then map each to an embedding. Spammers break this on purpose. They write "V1@gr@," "PayPa1," "𝓯𝓻𝓮𝓮 money," or swap the Latin "A" in "Apple" for the Cyrillic "А" that renders identically. To a human eye it reads fine. To a tokenizer it is a pile of out-of-vocabulary junk: "paypa1" is not "paypal," the Cyrillic "Аpple" is a different token from "Apple," and the model that learned "paypal is a phishing target" never fires because it never sees the word. This is a genuine adversarial attack on the input representation, and there are effectively infinite spellings of any word.

Google's 2023 answer is RETVec (Resilient and Efficient Text Vectorizer), published at NeurIPS 2023 and open-sourced ([Google Security blog](https://security.googleblog.com/2023/11/improving-text-classification.html), [arXiv 2302.09207](https://arxiv.org/abs/2302.09207), [google-research/retvec](https://github.com/google-research/retvec)). It kills the vocabulary entirely. Two pieces:

1. A vocabulary-free character encoder. Take a word (up to 16 characters). For each character, get its UTF-8 codepoint (the Integerizer), then write that codepoint as a compact 24-bit little-endian binary string (the Binarizer). A word becomes a small fixed grid of bits. There is no lookup table and no fixed vocabulary size, so it works on 100-plus languages out of the box, and there is no such thing as out-of-vocabulary. The paper's own point: this binary representation is both easy for a network to learn and far more compact than one-hot encoding (one-hot over all of Unicode would be enormous; 24 bits per character is tiny). That compactness is not a nicety, it is the performance win, as you will see in the numbers.

2. A small embedding model trained to be typo-proof. A tiny neural network (about 200,000 parameters, which is featherweight) projects that bit grid into a dense 256-dimensional embedding. The crucial part is how it is trained: pairwise metric learning on a typo-augmented dataset across 157 languages. The training deliberately corrupts words with insertions, deletions, typos, homoglyphs, and LEET substitutions, and teaches the model that the corrupted version must land near the clean version in the 256-dimensional space. So "paypa1," "PayPal," and "Pаypal" (Cyrillic a) all map to nearly the same vector. The downstream spam classifier sees essentially the same input whether the spammer obfuscated the word or not. The adversary's favorite trick stops working.

The measured result in the actual Gmail spam classifier, from Google's own post: swapping in RETVec improved the spam detection rate by 38 percent over the previous baseline, cut the false positive rate by 19.4 percent, and reduced the model's TPU usage by 83 percent. Read that last number twice. They got more accurate and 5x cheaper to serve at the same time, because a compact 24-bit binary encoding plus a 200k-parameter model is dramatically less compute than a big vocabulary embedding, and at 121 billion messages a day the serving cost is the whole ballgame. This is the offline-think, online-lookup pattern in a new dress: the expensive, adversarially-augmented training runs offline once, the live path is one cheap forward pass through a tiny model.

### The label problem and the feedback loop (why this beats Radar's timing)

A classifier is only as good as its labels, and spam is a moving target: today's clean campaign is next week's blocked one (concept drift). Gmail has an advantage the Stripe Radar fraud model does not. In Radar, the truth (a chargeback dispute) arrives 30 to 90 days late, and blocked payments have no outcome at all. In Gmail, the label loop is fast and honest: every "Report spam" and every "Not spam" tap is a labeled example arriving in near real time, from real users, at massive volume. Priya's "Not spam" tap on the newsletter is a training signal within minutes. That fast loop plus TensorBoard-monitored fast retraining is how Gmail keeps up with an adversary who adapts weekly. Personalization rides on the same loop: the same newsletter can be spam for one user and ham for another, so per-user signals (a lineage back to per-user Bayesian filtering) blend with the global model.

### The scale story at three tiers

Tier 1, about 1,000 messages a day (a small mail server). A single box runs regex rules plus a naive Bayes filter over a word-count table. Sorting-by-score, storage, everything fits in memory. Nothing breaks. This is a college department's mail server in 2005.

Tier 2, about 100,000 messages a day (a mid-size company). Per-user Bayesian tables get large, and you start needing a shared reputation cache so you are not re-deciding the same known-bad sender for every recipient. You add DNSBL lookups at connection time to shed the obvious junk before scoring. The bottleneck becomes feature extraction cost, so you cache sender and URL reputation. Rules alone now have too many false positives, so ML earns its place.

Tier 3, 10 million-plus and up to Gmail's roughly 121 billion a day. Everything changes. You cannot run a neural net on every message, so the funnel ordering becomes survival, not elegance: connection-time reputation (Layer 0) must kill the bulk of volume with an O(1) lookup before any body is parsed, or you drown. The classifier itself must be cheap to serve, which is the entire reason RETVec's 83 percent TPU cut matters (a 5x heavier model at 121 billion messages a day is a datacenter you cannot afford). Serving runs on TPUs. Reputation stores are distributed and sharded. The label pipeline ingests billions of user "report spam" signals and retrains fast to track drift. What breaks at each step up is compute: the naive answer (score everything with the best model) is linear in a number that is astronomically large, so the whole architecture exists to make sure the expensive model only runs on a tiny, pre-filtered fraction, and even then runs as cheaply as possible.

Concrete walk-through at Tier 3: a spammer registers 500 fresh domains overnight and sends 2 million phishing mails, 4,000 from each domain to stay under volume alarms, using homoglyph text to dodge word filters. Layer 0 lets them through at first because the domains are brand new with no reputation yet (this is exactly the "newly created domains hiding low volume" case Google called out). But the TensorFlow model flags the image-plus-hidden-text structure and the RETVec-decoded impersonation text, files them as spam, and the first users who tap "Report spam" push those 500 domains' reputation negative within minutes, so Layer 0 now rejects the rest at the door. The cheap filter learns from the expensive one. That handoff is the system working as designed.

## 8. The retention and habit mechanic

Spam filtering is not an engagement loop. It does not try to make Priya open the app more. It is a defensive, trust-based retention mechanic, the same family as WhatsApp encryption and Stripe Radar in this ledger.

The loop is: quiet, reliable filtering builds trust, trust makes Gmail the default place her mail lives, and the accumulated mail plus that trust becomes a switching cost. The day a competitor's inbox is 30 percent spam, or worse, the day it drops one real invoice into a junk folder and she misses a payment, is the day she stops trusting it. Gmail's near-invisible correctness is exactly what keeps her from ever evaluating alternatives. The metric it moves is retention, by protecting trust, and secondarily activation (a clean inbox on day one is why new users stay).

There is a small active loop layered on top: the "Report spam" and "Not spam" buttons give users a feeling of control and, more importantly, feed the label pipeline. Every tap both satisfies the user and improves the product. That is a rare two-for-one: the retention mechanic and the training-data mechanic are the same button.

A real observed example of the trust dynamic: Gmail's 2019 announcement led with the phrase "spam does not bring us joy," and the entire framing was about how much junk they remove that you never see. The marketing of the feature is the absence of the feature. You are meant to notice only when it fails.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph to shippable shader code and runs an embeddable runtime. The sharpest transferable idea here is RETVec's insight that your input representation is a performance and robustness decision, not a detail, and that a compact encoding can be both more robust and 5x cheaper at once.

Concretely, two moves:

First, encode identifiers and tokens in a compact, structural form instead of a brittle lookup table. RETVec threw out the vocabulary and encoded characters as 24-bit binary, which is why it handles 100-plus languages and infinite misspellings with a 200k-parameter model. In Rare.lab, node types, parameter names, and user-authored expressions in the graph are your "tokens." If you key everything off a fixed string-table or an enum vocabulary, you get the tokenizer's brittleness: an unknown or slightly-off node id becomes an out-of-vocabulary crash or a silent miss, and every new node type forces a table migration. Represent nodes and their signatures in a compact structural encoding (stable content hashes, or a small fixed binary schema per node) so the compiler and the runtime never choke on an input they have not seen a literal string for, and so lookups stay O(1) and tiny.

Second, order your pipeline cheapest-filter-first, and reject at ingress. Gmail's survival trick at 121 billion messages a day is that the expensive model only ever runs on what the cheap reputation gate did not already kill. Rare.lab's expensive step is compilation and shader validation. Do the analog of Layer 0: a cheap, near-O(1) validation and dedup pass at graph-ingress (reject malformed graphs, cache-hit on graphs whose content hash you have already compiled, flag cycles and type mismatches) before you spend a single cycle in the real compiler. At scale, most submitted graphs are small edits of graphs you already compiled; a content-hash cache turns the common case into a lookup, exactly like reputation-caching a known sender. The pattern is identical to the rest of this ledger: push the expensive thinking offline and behind a cheap gate, and make the live path a small, bounded, cheap operation.

And the meta-lesson from the 83 percent TPU cut: when you must run something on every single item at massive volume, the compact representation is not a micro-optimization, it is what makes the good model affordable to ship at all. Choose the encoding first, then the model.

---

## Sources

- Google Security Blog, "Improving Text Classification Resilience and Efficiency with RETVec," Nov 2023: https://security.googleblog.com/2023/11/improving-text-classification.html
- RETVec paper (NeurIPS 2023), arXiv 2302.09207: https://arxiv.org/abs/2302.09207
- RETVec open-source repo, google-research/retvec: https://github.com/google-research/retvec
- Google Workspace Blog, "Ridding Gmail of 100 million more spam messages with TensorFlow," Feb 2019: https://workspace.google.com/blog/product-announcements/ridding-gmail-of-100-million-more-spam-messages-with-tensorflow
- Spamhaus, "DNS Blocklist Basics" (DNSBL as first line of defense): https://www.spamhaus.org/resource-hub/email-security/dns-blocklist-basics/
- Paul Graham, "A Plan for Spam" (naive Bayes baseline), 2002: https://www.paulgraham.com/spam.html
- ScienceDirect / Heliyon, "Machine learning for email spam filtering: review, approaches and open research problems": https://www.sciencedirect.com/science/article/pii/S2405844018353404
- Statista, spam share of global email traffic: https://www.statista.com/statistics/420391/spam-email-traffic-share/
- EmailTooltester, spam statistics 2026: https://www.emailtooltester.com/en/blog/spam-statistics/
- Gmail usage statistics (users and daily volume): https://www.demandsage.com/gmail-statistics/

Fact vs inference. Confirmed by primary Google sources: the RETVec architecture (UTF-8 to 24-bit binary encoder, 16-char words, ~200k-parameter model, 256-dim metric embedding, 157-language typo-augmented metric-learning training), the RETVec Gmail metrics (+38 percent detection, -19.4 percent false positives, -83 percent TPU), the 2019 TensorFlow model blocking ~100 million more spam per day and the kinds it catches, and the >99.9 percent overall block rate. Inference (labeled as such in the text): the exact internal ordering and data structures of Gmail's connection-time reputation store, the precise blend of per-user and global models today, and the specific botnet walk-through, which are presented as the well-grounded "how this class of system is built" version because Google has not published those internals in full.
