# WhatsApp end-to-end encryption: the Signal Protocol (X3DH, the Double Ratchet, and Sender Keys)

Date: 2026-07-01
Product: WhatsApp
Feature: End-to-end encryption of one-to-one and group messages

A note on scope. On 2026-06-15 this ledger tore down WhatsApp delivery receipts,
the ticks, and found that the server is a post office: it does receive, look up,
forward, ack, delete, and it never needs to understand a message to move it.
Today we open the envelope, or rather we find that we cannot. This teardown is
about the layer that sits on top of that same post office and makes the letters
unreadable to the postman. Different feature, same product, and the two fit
together like a lock fits a door.

---

## 1. The user

Meet Aisha. She is a 29-year-old journalist in Delhi. It is 9:40 pm and she is
texting a source about a story. She sends a photo of a document and the line
"can you confirm this is real?" She is not a cryptographer. She has never heard
the words "elliptic curve." But she has a very specific feeling in her stomach:
she wants exactly two people to be able to read this, her and her source, and
nobody else. Not WhatsApp. Not her phone company. Not someone who steals the
server logs next year.

She is also in three group chats at the same time: her family group of 14
people, her newsroom group of 60, and a neighborhood group of 512. She expects
the same privacy in all of them, and she never once thinks about the fact that
"send this to 512 people privately" is a wildly harder problem than "send this
to one person privately."

---

## 2. The real problem

Here is the pain, said plainly. When Aisha hits send, her message does not fly
straight to her source. It goes up to a WhatsApp server sitting in a data
center, waits there, and then gets pushed down to the source's phone. That is
how every chat app works, because the two phones are almost never online at the
same instant (this is the store-and-forward post office from the ticks
teardown).

So there is a middle. And the middle is the problem. Whoever controls that
server, or hacks it, or subpoenas it, could in principle read everything. The
old promise of "we encrypt your data" usually meant "we encrypt it between your
phone and our server, and then we hold the keys." That is a locked box where the
company keeps a copy of the key. Aisha does not want that. She wants a box only
her source can open, where even the company that built the box cannot open it.

There is a second, sneakier problem. Phones get stolen and seized. Suppose
someone grabs the source's phone six months from now and pulls every key off it.
Aisha wants tonight's message to still be safe, because the key that locked
tonight's message should have been destroyed months ago. The system should
forget how to read old messages the instant it is done reading them.

And the hardest problem: the source is asleep. Their phone is off. Aisha cannot
do a live handshake with a phone that is not there. She needs to set up a
private channel with someone who is not online. That is not how locks usually
work.

---

## 3. The feature in one sentence

End-to-end encryption locks every WhatsApp message with a key that lives only on
the two (or few) phones in the conversation, generated fresh per message, so the
WhatsApp server relays ciphertext it cannot read and could not read even if
compelled to.

---

## 4. Jobs to be done

- "Make sure only the person I am texting can read this, not WhatsApp, not
  anyone in the middle." (Confidentiality.)
- "If someone steals my phone later, don't let them read the messages I already
  sent and deleted." (Forward secrecy.)
- "If my phone gets briefly hacked, let the conversation heal itself so the
  hacker loses access going forward." (Post-compromise recovery, the self-heal.)
- "Let me start a private chat with someone who is asleep right now."
  (Asynchronous setup.)
- "Make me confident the person on the other end is really them, not an
  impostor the server swapped in." (Authentication, the safety number.)
- "Do all of this without me thinking about any of it." (Zero friction. The user
  never sees a key.)

---

## 5. How it works for the user

Aisha sees almost nothing, and that is the point. She opens a chat and there is a
small line at the top: a lock icon and "Messages and calls are end-to-end
encrypted. No one outside of this chat, not even WhatsApp, can read or listen to
them." She types, she sends, the ticks turn blue. It feels identical to an
unencrypted app.

The only two moments encryption becomes visible:

1. If she taps the contact name and scrolls to "Encryption," she sees a 60-digit
   number and a QR code. That is the safety number. She can scan her source's QR
   code in person to confirm nobody is impersonating them.
2. If her source reinstalls WhatsApp or switches phones, she gets a small yellow
   system message in the chat: "Your security code with [source] changed." That
   is the app telling her the keys were reset, which is normal after a
   reinstall, but is also exactly the signal you would want to see if someone
   tried a swap attack.

That is the whole visible surface. Everything else is underneath.

---

## 6. The actual flow, step by step

Setup, done once, long before tonight's message:

1. When the source first installed WhatsApp, their phone generated a batch of
   keys and uploaded the public halves to the WhatsApp server: one long-term
   Identity Key, one medium-term Signed Pre Key (signed by the identity key),
   and a batch of around 100 One-Time Pre Keys. The private halves never leave
   the phone. The server stores this bundle like a stack of pre-addressed
   envelopes waiting to be used.

Sending tonight's first message to a sleeping source:

2. Aisha's phone asks the server for the source's "prekey bundle." The server
   hands back the source's Identity Key, the Signed Pre Key, and pops one
   One-Time Pre Key off the stack (and deletes that one so nobody reuses it).
3. Aisha's phone runs X3DH (Extended Triple Diffie-Hellman) locally, combining
   those public keys with her own keys plus a fresh ephemeral key, to compute a
   shared secret. The source's phone, whenever it wakes up, can compute the exact
   same secret from its private keys. No secret ever crossed the wire.
4. That shared secret seeds the Double Ratchet. Aisha's phone derives a message
   key, encrypts "can you confirm this is real?" and the photo with it, and hands
   the ciphertext to the server. The server forwards it (the post office), waits,
   delivers it when the source's phone reconnects.
5. Every subsequent message ratchets forward to a brand new key. Message 2 uses a
   different key than message 1. Yesterday's keys are already deleted.

Sending to the 512-person neighborhood group:

6. The first time Aisha posts in the group, her phone makes a Sender Key for
   herself, then sends that Sender Key to each of the other 511 members over
   their existing one-to-one encrypted channels. This is the expensive step, and
   it happens once.
7. After that, every group message is encrypted one time with her Sender Key,
   handed to the server as a single ciphertext, and the server fans it out to all
   511 recipients. She does not encrypt 511 times per message.

---

## 7. Under the hood, like the engineer

This is the heart of it. The problem splits into three engines, each solving a
different hard thing: how two absent strangers agree on a secret (X3DH), how a
live conversation keeps rolling forward with fresh keys (the Double Ratchet), and
how you do all that for a group without paying an N-squared tax (Sender Keys).
WhatsApp licensed and shipped the Signal Protocol, written by Trevor Perrin and
Moxie Marlinspike at Open Whisper Systems, and finished rolling it out to all of
its (then) one billion users on 5 April 2016. The crypto primitives are
Curve25519 for key agreement, AES-256 in CBC mode for the message body, and
HMAC-SHA256 for authentication. Those specifics are public, published in the
WhatsApp Encryption Overview white paper (now on version 9, February 2026) and
in Signal's own specifications.

### Engine A: X3DH, agreeing on a secret with someone who is asleep

The core trick of Diffie-Hellman is that two people can each mix their own
private key with the other's public key and land on the exact same number, a
number an eavesdropper who saw only the public keys cannot compute. The problem
is that plain Diffie-Hellman needs both parties present and needs long-term keys
to authenticate, and long-term keys alone give you no forward secrecy. X3DH fixes
all three at once by doing not one Diffie-Hellman but four, over a mix of
long-term, medium-term, and one-time keys, and by letting one side publish its
public keys to the server ahead of time so it can be offline during setup.

The keys, concretely, as WhatsApp names them:

- Identity Key (IK): long-term, the phone's stable public identity. Think of it
  as the source's face. It rarely changes (only on reinstall).
- Signed Pre Key (SPK): medium-term, rotated periodically, signed by the Identity
  Key so a recipient can trust it came from that identity.
- One-Time Pre Keys (OPK): a stack of roughly 100 single-use keys. Each is handed
  out exactly once and then deleted from the server.

When Aisha initiates, she generates a fresh Ephemeral Key (EK) and computes four
shared values (using the Signal X3DH spec's naming):

- DH1 = DH(IK_Aisha, SPK_source)
- DH2 = DH(EK_Aisha, IK_source)
- DH3 = DH(EK_Aisha, SPK_source)
- DH4 = DH(EK_Aisha, OPK_source)

Then SK = KDF(DH1 || DH2 || DH3 || DH4). That single strong secret SK is the seed
for everything.

Why four and not one? Each Diffie-Hellman buys a different guarantee. DH1 mixes
Aisha's identity with the source's signed prekey, which authenticates Aisha. DH2
mixes Aisha's ephemeral with the source's identity, which authenticates the
source. DH3 and DH4 mix ephemerals with the medium and one-time keys, which is
what gives forward secrecy: because EK and OPK are throwaway, once they are
deleted nobody can reconstruct SK, even holding every long-term key. The
one-time prekey DH4 is the strongest forward-secrecy ingredient, which is exactly
why the server hands out each OPK only once.

Data structures here are humble but exact. On the server, a user's prekey bundle
is a small record: one identity public key, one current signed prekey, and a
queue (first-in-first-out) of one-time prekey public blobs. "Hand out a prekey"
is a dequeue plus a delete. On the phone, the private keys sit in secure local
storage and are never uploaded.

Failure mode, named honestly and grounded in real research: the one-time prekey
queue can run dry. A popular account (say a public figure) gets contacted by
thousands of new people, each conversation pops one OPK, and the queue empties
faster than the phone refills it (the phone only refills when it comes online).
When the queue is empty, the server falls back to serving the bundle without an
OPK, so that first session loses the DH4 ingredient and its initial forward
secrecy is slightly weaker until the ratchet takes over. This "prekey exhaustion"
behavior is real and has been studied (see the Prekey Pogo paper on WhatsApp's
handshake, 2025). The mitigation is simply to keep the queue topped up and to
treat the fallback as rare and temporary.

### Engine B: the Double Ratchet, a new key for every single message

X3DH gives you the first shared secret. The Double Ratchet is what turns that one
secret into an endless supply of one-time-use message keys, so that message 500
is protected by a key with no computable relationship to the key for message 1.
It is two ratchets working together, which is where the name comes from.

The symmetric ratchet is a one-way chain. Picture a Key Derivation Function
(KDF), which is a hash-like function that takes one input key and produces two
outputs: a new chain key and a message key. You feed the current Chain Key in,
you get out (next Chain Key, this message's Message Key). Encrypt the message with
the Message Key, throw the Message Key away, keep only the next Chain Key. Because
the KDF is one-way, holding Chain Key number 6 tells you nothing about Chain Key
number 5, and holding one Message Key tells you nothing about any other. This is
the forward-secrecy engine: the chain only walks forward and cannot be rewound.
Concretely, if Aisha sends "on my way," then "5 min," then "here," those three
messages ride keys MK0, MK1, MK2, each derived and then immediately erased.

But a chain alone never re-injects new randomness, so a one-time break-in could
let an attacker follow the chain forward forever. That is what the second ratchet
fixes. The Diffie-Hellman ratchet rides on the messages themselves: every message
header carries a fresh ratchet public key. When Aisha receives a message from her
source with a new ratchet key she has not seen, she does a fresh Diffie-Hellman
with it, feeds the result into a Root Chain, and that Root Chain spits out a brand
new receiving chain key. Then she generates her own new ratchet key pair, does
another DH, feeds the Root Chain again, and gets a new sending chain key. Every
back-and-forth turn of the conversation injects fresh secret material. This is
the self-heal, the post-compromise recovery: if an attacker stole the state at
one moment, the very next round-trip mixes in a Diffie-Hellman value they never
saw, and they are locked back out.

So each side juggles three KDF chains at once:

- The Root Chain: fed by Diffie-Hellman outputs, produces the seeds for the two
  chains below.
- The Sending Chain: produces message keys for outgoing messages.
- The Receiving Chain: produces message keys for incoming messages.

Now the real-world wrinkle: messages arrive late and out of order, because the
network and the store-and-forward post office do not promise order. Say the source
sends messages numbered 0, 1, 2, but 2 arrives before 1. If the symmetric ratchet
had to advance strictly in order, message 2 would be undecryptable until 1 shows
up. The fix is in the header. Every message carries two small integers: N, its
number in the current sending chain, and PN, the length of the previous sending
chain. When Aisha receives message 2 before message 1, she advances the receiving
chain to derive MK1 and MK2, uses MK2 now, and stores MK1 in a small dictionary
of skipped message keys, keyed by (ratchet public key, message number). When
message 1 finally arrives, she looks up its key, decrypts, and deletes it from the
dictionary.

That skipped-keys dictionary is a genuine attack surface, and the defense is a
named constant: MAX_SKIP. It caps how many message keys may be skipped in one
chain. Set it too low and a few dropped messages break the chat; set it too high
and a malicious sender could claim message number 2,000,000,000 and force the
recipient to derive two billion keys, a denial-of-service by arithmetic. MAX_SKIP
is the guard rail that bounds the work any one incoming header can force. Hold
that idea; it comes back in the Rare.lab lesson.

One more elegance worth naming: the header also lets the receiver know when a new
DH ratchet key has appeared, so PN tells them how many keys were in the old chain
before the ratchet turned, so they can finish deriving any stragglers from the
old chain before moving to the new one. The two integers N and PN are all the
bookkeeping the whole out-of-order machine needs.

### Engine C: Sender Keys, doing this for 512 people without N-squared pain

Here is the scaling cliff. The Double Ratchet is a two-person protocol. A group
is not two people. The naive way to encrypt for a group is client-side fan-out:
Aisha runs a separate Double Ratchet with each of the other 511 members and
encrypts her message 511 times, once per member. For one message that is 511
encryptions and 511 uploads. For a chatty group it is brutal, and it is
quadratic across the group: N members each sending to N-1 others is on the order
of N-squared pairwise encryptions for a single broadcast round.

Sender Keys breaks the quadratic. The idea: separate "agree on a key" (do it once,
pairwise, expensive) from "send a message" (do it many times, once, cheap). Each
member has a Sender Key, which is a tuple of a symmetric Chain Key (32 random
bytes) and a Curve25519 Signature Key pair. The first time Aisha speaks in the
group, she generates her Sender Key and distributes it to the other 511 members
inside a SenderKeyDistributionMessage, sent over each member's existing one-to-one
encrypted channel. That distribution is the O(N) step, and it happens once (and
again only when the membership changes).

After that, sending is cheap and constant work for the sender. To post "anyone
home?" Aisha:

1. Derives a Message Key from her Sender Key's Chain Key using a KDF (HMAC-based),
   then ratchets the Chain Key forward (same symmetric-ratchet trick as the
   Double Ratchet, so group messages also get forward secrecy).
2. Encrypts the message once with AES-256-CBC under that Message Key.
3. Signs the ciphertext with her Signature Key, so all 511 recipients can verify
   it truly came from her and not from another group member forging her.
4. Uploads one ciphertext. The server fans it out to all 511 recipients.

Each recipient already has Aisha's Sender Key, so each can derive the same Message
Key and decrypt. One encryption, one upload, server-side fan-out. The sender's
per-message cost is now independent of group size.

The cost of Sender Keys is paid at membership change, and it is paid on purpose.
When someone leaves the neighborhood group, everyone resamples and redistributes
a fresh Sender Key, so the person who left cannot read anything sent after their
exit (that is O(N) work again, but rare). Security researchers have poked at the
edges of this trade (see "WhatsUpp with Sender Keys?", IACR 2023, which formally
analyzes and proposes improvements), and the honest summary is: Sender Keys trade
a little of the Double Ratchet's per-message self-healing for a massive efficiency
win that is what makes 512-member (now up to 1024-member) encrypted groups
practical at all.

### The scale story at three tiers

Tier 1, a one-to-one chat (2 endpoints). One X3DH handshake, one Double Ratchet
session, a new key per message. The cost is trivial, a few Diffie-Hellman
operations at setup and one KDF plus one AES operation per message. Nothing
breaks here. If WhatsApp only ever did two-person chats, the story would end at
Engine B.

Tier 2, a large group (hundreds of endpoints). The thing that breaks is the
per-message cost of client-side fan-out: encrypting every message N-1 times is
quadratic across the group and murders battery and bandwidth on big active
groups. What survives it is Sender Keys: move the O(N) cost to a one-time
distribution and make each message O(1) for the sender plus server fan-out. A
second thing that breaks at this tier is membership churn: every join or leave
forces a key redistribution, so very large, very churny groups pay a real cost,
which is one reason group sizes are capped (256, later 1024) rather than
unbounded.

Tier 3, the whole network (billions of endpoints, roughly 100 billion messages a
day as of 2020, across 2 to 3 billion users). What breaks here is not the crypto
math, which is per-conversation and embarrassingly parallel, but the supporting
infrastructure. The prekey server must hand out and replenish one-time prekeys for
billions of accounts, and the OPK queue exhaustion problem becomes a real
operational concern for popular accounts (Engine A's failure mode, at planetary
scale). The delivery layer must still relay every ciphertext through the
store-and-forward post office (the ticks teardown), now with the extra rule that
it can never read, cache-inspect, or transform a payload. And multi-device makes
the endpoint count explode: each of a user's linked devices (phone, laptop web,
tablet) is its own cryptographic identity with its own keys, so a 5-member group
where everyone has 3 devices is really 15 endpoints, and Sender Key distribution
scales with device count, not just human count. WhatsApp's multi-device design
gives each companion device its own identity key and treats it as a separate
member of the fan-out, which keeps the per-message crypto model unchanged while
the endpoint graph grows. The saving grace at every tier is the same one this
ledger keeps finding: the expensive asymmetric work is rare and per-session, the
per-message work is cheap symmetric ratcheting, and none of it needs a global
lock or a central brain, so it shards perfectly by conversation.

### Fact versus inference

Fact, from the WhatsApp white paper and the Signal specs: the key types and names
(Identity Key, Signed Pre Key, One-Time Pre Keys, Sender Key), Curve25519 /
AES-256-CBC / HMAC-SHA256, the Double Ratchet chains, MAX_SKIP and the
N/PN header fields, Sender Keys with server-side fan-out, and the April 2016
rollout. Inference, clearly labeled: exact server-side queue implementations, the
precise size of the one-time prekey batch (commonly cited around 100, order of
magnitude is public but the exact live value is an operational detail), and the
precise internal storage layout of the skipped-message-key dictionary. Those
follow the standard way this class of problem is solved and match the specs, but
the byte-level internals of WhatsApp's servers are not public.

---

## 8. The retention and habit mechanic

Be honest here, because the CLAUDE rules demand the real mechanic, not a guess.
End-to-end encryption is not an engagement loop like Discover Weekly's Monday
refresh or Instagram's unseen-stories ring. Nobody opens WhatsApp because they are
excited to ratchet a key. So which metric does it move? Retention, specifically
the defensive kind: it lowers churn and cements WhatsApp as the default, trusted
place your entire social graph already is.

The mechanic is trust as a switching cost. Once Aisha believes WhatsApp is the
private option, and once all 512 of her neighbors, her whole family, and her whole
newsroom are already there under the same promise, the cost of moving everyone
somewhere else is enormous. The lock icon and the "not even WhatsApp can read
this" banner are quiet reassurance that runs under every single session, all day,
forever. It is the kind of feature you never notice until a competitor makes you
doubt it.

The one genuinely visible re-engagement touch is the security-code-change
notification. When a contact reinstalls or swaps phones, that yellow "your
security code changed" line appears in the chat. It is a tiny trust ritual: most
of the time it is benign, and the user learns to shrug, but it keeps the promise
visible and gives the security-conscious user a moment to re-verify by safety
number. Real observed example: after the January 2021 WhatsApp privacy-policy
scare, tens of millions of users flirted with Signal and Telegram, and WhatsApp's
public response leaned hard on the encryption story ("your personal messages are
end-to-end encrypted") precisely because that promise was the retention anchor
that kept the exodus from becoming permanent. The feature's job is not to pull you
back daily; it is to make sure you never have a reason to leave.

---

## 9. The lesson for Rare.lab

The sharpest lesson here is the split between expensive-and-rare and
cheap-and-constant, and it maps almost one-to-one onto a node-based editor that
compiles to a runtime.

The Signal Protocol pays a heavy asymmetric cost exactly once per session (X3DH,
four Diffie-Hellmans, the whole handshake) and produces a cheap seed. After that,
every message is a single symmetric ratchet step: one KDF, one AES, done. The
expensive thinking happens at setup; the hot path is trivial and deterministic.
Rare.lab already has this shape hiding in it. The node-graph compile is your
X3DH: it can be as expensive as it needs to be, because it happens once and
produces a compact runtime artifact. Each frame of the embeddable runtime should
then be your ratchet step: a cheap, deterministic advance of state from a seed,
with no re-parsing of the graph and no per-frame allocation. If a frame ever has
to re-walk the node graph, you have put asymmetric cost on the symmetric hot path,
which is the exact mistake Signal refused to make.

Two concrete, performance-biased takeaways:

First, borrow the Sender Keys pattern for fan-out. When your embeddable runtime
has to broadcast visual state to N viewers (a shared shader scene, a collaborative
canvas, N clients watching the same effect), do not do per-viewer work every
frame. Distribute a derivable seed once at join time (your SenderKeyDistribution
step, O(N) and rare), then each frame serialize or encrypt the delta exactly once
and let the transport fan it out (O(1) sender work plus fan-out). Per-frame cost
should be independent of viewer count. Pay the O(N) tax only on membership change,
just like the group does when someone leaves.

Second, and most important for a product that runs untrusted graphs: ship a
MAX_SKIP. Signal caps how much compute a single incoming header can force, so a
malicious sender cannot ask the recipient to derive two billion keys. Your runtime
compiles and executes user-authored node graphs, which is precisely a situation
where a hostile or malformed input can try to blow up compute: a graph that
expands to millions of nodes, a shader loop with an enormous bound, a catch-up
path that tries to replay a huge backlog of frames. Put an explicit, named budget
on every place where one input decides how much work happens, and reject or clamp
past it. Bound the blast radius of a single input at compile time, before it
reaches the GPU. That one constant is the difference between a runtime that
degrades gracefully under a bad graph and one that a single crafted input can
freeze.

Compile once, ratchet forever, and cap what any one input can cost you.

---

## Sources

- WhatsApp Encryption Overview, Technical White Paper (version 9, February 2026): https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf
- Signal, The X3DH Key Agreement Protocol (Marlinspike, Perrin): https://signal.org/docs/specifications/x3dh/
- Signal, The Double Ratchet Algorithm (Perrin, Marlinspike): https://signal.org/docs/specifications/doubleratchet/
- Double Ratchet Algorithm, Wikipedia: https://en.wikipedia.org/wiki/Double_Ratchet_Algorithm
- Sender Keys, Wikipedia: https://en.wikipedia.org/wiki/Sender_Keys
- Balbas, Collins, Gajland, "WhatsUpp with Sender Keys? Analysis, Improvements and Security Proofs" (IACR ePrint 2023/1385): https://eprint.iacr.org/2023/1385.pdf
- "Prekey Pogo: Investigating Security and Privacy Issues in WhatsApp's Handshake Mechanism" (2025): https://arxiv.org/pdf/2504.07323
- Signal blog, "Open Whisper Systems partners with WhatsApp to provide end-to-end encryption": https://signal.org/blog/whatsapp/
- TechCrunch, "WhatsApp completes end-to-end encryption rollout" (5 April 2016): https://techcrunch.com/2016/04/05/whatsapp-completes-end-to-end-encryption-rollout/
- TechCrunch, "WhatsApp is now delivering roughly 100 billion messages a day" (29 October 2020): https://techcrunch.com/2020/10/29/whatsapp-is-now-delivering-roughly-100-billion-messages-a-day/
- Open Whisper Systems, Wikipedia: https://en.wikipedia.org/wiki/Open_Whisper_Systems
- "A Dive into WhatsApp's End-to-End Encryption" (arXiv 2209.11198): https://arxiv.org/pdf/2209.11198
