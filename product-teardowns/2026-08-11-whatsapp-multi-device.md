# WhatsApp Multi-Device: using your laptop while your phone is dead in your bag

Date: 2026-08-11
Product: WhatsApp
Feature: Multi-device (companion devices, client-fanout encryption, and App State Sync via LTHash)

---

## 1. The user

Priya is a product designer in Pune. She works all day on a laptop. Her phone
sits in her bag on the desk, screen down, often at 4% and dying by lunch.

She lives in WhatsApp. Her design team coordinates there. Her mom sends her
photos of lunch there. Her landlord, her CA, her dentist, all there. During the
work day she wants to type long messages on a real keyboard, not thumb them on
a phone. So she keeps WhatsApp open on her laptop in a browser tab.

At 2 pm her phone battery hits zero and shuts off. She does not notice, because
her hands are on the laptop. She keeps chatting on the laptop for three more
hours. She archives an old chat with a client. She stars a message with the
new office address. She mutes a noisy group. Her phone is a dead brick in the
bag the entire time, and none of it matters to her.

That last sentence is the whole feature. Four years ago it would have been a
lie. Her laptop would have gone dark the moment the phone died.

## 2. The real problem

For most of WhatsApp's life, your phone was WhatsApp. WhatsApp Web and the
desktop app were not real clients. They were mirrors. The laptop showed you a
live picture of what was happening on your phone, and every message you "sent"
from the laptop was actually relayed through the phone over an encrypted
channel. The phone did the real work: it held the one and only encryption
identity for your account, it stored the truth about your chats, and it
physically sent and received every message.

So the laptop had one humiliating dependency. If your phone was off, out of
battery, or on a plane, WhatsApp Web was a frozen screenshot. You would be
typing a reply to your boss, hit send, and watch the spinner turn forever
because the phone in your bag could not hear you.

This was the single most common complaint, and it was the one thing rivals beat
WhatsApp on for years. Telegram's desktop app was a real client. It worked with
your phone off, because Telegram is not end-to-end encrypted by default, so the
server holds your messages and any device can just log in and fetch them. iMessage
worked across your Mac and iPhone because Apple controls the whole key system
inside one account.

WhatsApp had painted itself into a corner with its own best feature. It is
end-to-end encrypted, which means the server is a blind post office. It cannot
read your messages, so it cannot just hand your history to a new device. And the
encryption keys lived only on the phone. Adding a second independent device that
could send and receive on its own, while keeping the promise that the server
never sees plaintext, and while keeping all your devices showing the same chat
list, is genuinely hard. That is why it took years.

## 3. The feature in one sentence

Multi-device lets you run WhatsApp on your phone plus up to four other devices
at the same time, each with its own encryption identity, each able to send and
receive on its own even when the phone is off, all showing the same synced
chats.

## 4. Jobs to be done

What is Priya really hiring this feature to do?

- "Let me type on my laptop keyboard without my phone being the bottleneck."
- "When my phone dies, do not kill my work."
- "Keep my chat list identical everywhere. If I archive a chat on the laptop, it
  should be archived on my phone too, without me touching the phone."
- "Do not make me feel less safe. I do not want to trade my encryption away for
  the convenience of a desktop app."
- "Do not make me re-scan a QR code every hour or re-verify safety numbers every
  time I add a device."

Notice the tension baked into that list. She wants independence (works with the
phone off) AND consistency (same state everywhere) AND security (still end to
end). Those three pull against each other. The engineering is all about not
having to pick.

## 5. How it works for the user

Priya opens web.whatsapp.com on her laptop. It shows a QR code. She opens
WhatsApp on her phone, taps Linked Devices, and points the camera at the screen.
A second later the laptop fills with her chats. Done.

From then on:

- She can send and read messages on the laptop.
- Her phone can be off. The laptop still works, for up to 14 days of the phone
  being idle.
- Anything she does on one device shows up on the others. Archive on the laptop,
  archived on the phone. Star a message on the phone, starred on the laptop.
- She can link up to four companion devices total. A second laptop, a tablet, a
  work desktop. Five surfaces including the phone.
- The little "safety number" that proves her chat with her mom is private still
  works. She never had to verify each new device by hand.

The only rule she has to obey: unlock her phone at least once every 14 days. If
the phone goes fully dark for two weeks, all the companions log out. This is a
safety timer, not a technical limit, and we will see why it exists.

## 6. The actual flow, step by step

Linking the laptop:

1. Laptop shows a QR code. The QR is not just a login token. It carries the
   laptop's freshly generated public identity key and a reference the phone can
   use to talk back.
2. Phone scans it. Now the phone knows the laptop's public identity key, and the
   laptop learns the phone's.
3. The phone signs the laptop's public identity key. This signature is called
   the Account Signature. It is the phone saying, on behalf of the account, "I
   vouch that this laptop is mine."
4. The laptop signs the phone's public identity key. This is the Device
   Signature. It is the laptop saying "and I acknowledge this phone is my
   primary."
5. Both signatures get published to the server as part of the account's device
   list. Only when both exist will other people's phones agree to encrypt to the
   laptop.
6. The phone hands the laptop the account-level secrets it needs to decrypt the
   shared app state (the keys for the encrypted chat-settings blobs). The laptop
   pulls down the current state and reconstructs the chat list.

Sending a message from the laptop, phone off:

1. Priya types "moved to the new office, address is starred above" to her client
   Rahul and hits enter.
2. The laptop looks up the full device list for Rahul (say his phone plus his
   own laptop, so 2 devices) and for Priya herself (her phone plus her second
   work desktop, so her 2 other devices).
3. The laptop encrypts the message separately for each of those 4 devices, using
   a separate pairwise encrypted session with each one. Four sealed envelopes,
   four different keys.
4. It uploads all four to the server. The server, which cannot read any of them,
   drops each envelope into the queue for its target device.
5. Rahul's phone and laptop each pull their envelope and decrypt. Priya's phone
   (whenever it wakes up) and her desktop each pull the copy addressed to them,
   so her own sent message appears on all her devices too.
6. Priya archives the Rahul chat. The laptop writes a tiny "archive this chat"
   mutation, encrypts it, and uploads it as an app-state patch. Her phone and
   desktop download the patch later and apply it, so the chat is archived
   everywhere.

No step in that flow needed the phone to be awake.

## 7. Under the hood, like the engineer

This feature is really three separate hard problems wearing one coat. Identity
(how do other people trust a new device), messaging (how does a message reach
every device without the phone in the middle), and state (how does the boring
metadata, archive/mute/star/contacts, stay identical everywhere). Let us take
them one at a time, because they use completely different tricks.

### The old design and why it had to die

Before 2021, the data model was simple and doomed. One account, one identity
key, and that key lived on the phone. WhatsApp Web was a proxy. Think of it as a
puppet: the laptop showed frames streamed from the phone and forwarded your
keystrokes back to the phone, which did the actual Signal Protocol encryption
and the actual sending.

The plus side: there was only ever one real cryptographic identity, so trust was
trivial. The fatal side: the puppet is dead when the puppeteer sleeps. Phone off,
web off. There was no getting around it as long as exactly one device held the
keys.

The fix is structural, not a patch. Stop having one identity. Give every device
its own.

(This teardown builds directly on two earlier ones. The WhatsApp ticks teardown,
2026-06-15, explained the server as a store-and-forward post office that receives,
looks up, forwards, acks, deletes. The WhatsApp end-to-end encryption teardown,
2026-07-01, explained the Signal Protocol: X3DH to set up a shared secret, the
Double Ratchet for the per-message keys, and Sender Keys for groups. Multi-device
sits on top of both. If those two are the foundation, this is the second storey.)

### Problem one: identity. Each device is its own Signal citizen.

In the new model, your account is not a device. Your account is a *list* of
devices, and each device has its own Signal identity key pair, its own prekeys,
its own sessions. Priya's phone, laptop, and desktop are three separate Signal
identities that happen to belong to one phone number.

The data structure here is a per-account device list held on the server: a small
set of records, each record being (device id, public identity key, account
signature, device signature). It is basically a hash map from device id to that
device's public credentials. When anyone wants to message Priya, they fetch this
list.

Now the danger. If the server owns the device list, a malicious server could
quietly add a fifth device (its own) to Priya's list, and then every message
sent to Priya would also get encrypted to the attacker's device. That would break
end-to-end encryption completely and silently. This is the classic "ghost device"
attack.

The defense is **Automatic Device Verification**, and it is a neat piece of
cryptographic bookkeeping. Recall the two signatures from the linking flow:

- Account Signature: the phone (the account's root of trust) signs the new
  device's public identity key.
- Device Signature: the new device signs the phone's public identity key.

A device is only trusted by others if it carries a valid Account Signature made
by the phone's identity key. The server cannot forge that signature, because it
does not have the phone's private key. So the server can *store* the device list,
but it cannot *fabricate* a member of it. When Rahul's phone downloads Priya's
device list, it checks each device carries a phone-signed Account Signature
before it agrees to encrypt to it. A ghost device injected by the server has no
valid Account Signature, so Rahul's phone refuses it.

That is what "Automatic" means. Priya never re-verified anything by hand, but the
trust chain is still anchored to her phone's key, not to the server's word.

This is the same shape as certificate signing. The phone is a mini certificate
authority for its own account. It issues signed "this device is mine"
certificates. Real example: when Priya links her second work desktop, the phone
mints an Account Signature over the desktop's identity key. From then on, her
mom's phone, seeing that signature, will happily encrypt to the desktop without
Priya's mom ever being asked to check a safety number for it.

### Problem two: messaging. Client-fanout, one sealed envelope per device.

A one-to-one chat is no longer one-to-one under the hood. It is one-to-many,
because "Rahul" is several devices and "me" is several devices.

WhatsApp chose **client-fanout**. The sending client encrypts the message
separately for every device that should receive it, using a distinct pairwise
Signal session with each one, and ships N ciphertexts. Not one ciphertext with a
shared key. N ciphertexts, N keys.

Walk the concrete count. Priya (3 devices: phone P1, laptop L1, desktop D1) messages
Rahul (2 devices: phone P2, laptop L2). Priya is sending from L1. The recipients
of this one message are:

- Rahul's P2 and L2 (so Rahul reads it), and
- Priya's own P1 and D1 (so her sent message mirrors onto her other devices).

That is 4 target devices, so L1 produces 4 separate ciphertexts and uploads all
4. The server fans them into 4 device queues. This is why "sent from laptop" also
appears on your phone: your own other devices are in the recipient set.

Why client-fanout and not a single shared group-style key for the pair? Because
pairwise sessions give you the full Double Ratchet guarantees per device
(forward secrecy, break-in recovery) independently. If one device is compromised,
the blast radius is that one session, not the whole conversation. The cost is
that encryption work grows with device count. We will come back to that in the
scale story, because it is the load-bearing tradeoff.

Groups do NOT use client-fanout for the message body, or the numbers would
explode. A 200-person group where everyone has 3 devices would need 600
encryptions per message. Instead groups keep the **Sender Keys** trick from the
E2E teardown, now stretched across devices. Each *device* has its own sender key
for the group. When a device sends its first group message, it distributes its
sender key once, pairwise, to every other device in the group (that pairwise
distribution is the only client-fanout-shaped cost, and it is rare). After that,
each message is encrypted exactly once with the sender key, signed, and handed to
the server, which fans out the single ciphertext to everyone. So per-message
encryption cost is O(1) for the sender no matter how big the group. The pairwise
key setup is O(devices) but happens once, not per message.

That split is the whole scalability story of messaging in one line: pay O(devices)
rarely (key setup, one-to-one small fanout), pay O(1) on the hot path (each group
message).

### Problem three: state. The quiet nightmare of archive, mute, star, contacts.

Messages are the glamorous part. The part that actually made this take years is
boring metadata. Which chats are archived. Which are pinned. Which are muted.
Which messages are starred. What you named each contact. The unread markers.

On the old design, all of that lived on the phone and only the phone. Now every
device needs it, needs it to stay identical, and has to get it while the phone
might be asleep. And the server must never be able to read it, because "Priya
muted her boss" and "Priya's contact named Doctor Sharma" is private too.

This is **App State Sync**, and it is a small distributed database problem hiding
inside a chat app.

The model. Your app state is split into a handful of named **Collections**, each
a bag of key-value settings:

- `critical_block`: your push name and locale.
- `critical_unblock_low`: your contact list (the names you gave people).
- `regular_low`: chat pins, archive status, the unarchive-on-new-message setting.
- `regular_high`: mute status and starred messages.
- `regular`: protocol housekeeping like key expiration.

The names encode priority. Critical collections sync first and matter for
correctness. Regular ones can lag a beat.

Each change is a **Mutation**: a SET or REMOVE on one indexed value, for example
SET archive=true for chat-with-Rahul, or SET starred=true for message id
0xAB12. Mutations are bundled into a **Patch**. A device writes a patch, encrypts
each mutation (the server must not read "muted boss"), attaches integrity tags,
and uploads it. Every other device downloads patches in version order and applies
them. The server stores an opaque encrypted log per collection and, on request,
hands a device a full **Snapshot** (the current state rolled up) plus any patches
since.

So far this is just an encrypted operation log, like a tiny event-sourced system.
The hard question is the one every replicated system faces: **how does a device
know it is correctly in sync?** How does it know it applied every patch, in the
right order, with nothing dropped, reordered, or replayed by a buggy or hostile
server? The server is untrusted and blind, so it cannot just assert "you are up
to date." The device has to be able to *check*.

The naive check is: hash the entire collection and compare to an expected hash.
But Priya's contact list might have 5,000 entries. Re-hashing all 5,000 every
time one contact name changes is wasteful, and it means the server would have to
ship the whole set to verify a one-entry change. That does not scale.

### The clever core: LTHash, a hash you can update by subtraction

WhatsApp uses a **homomorphic hash** called **LtHash16** to solve exactly this.
This is the single most interesting idea in the whole feature, so let us go slow.

A normal hash like SHA-256 is a blender. Change one byte of input and the whole
output changes with no relationship to the old one. Useful for tamper detection,
useless for incremental updates. If you have the SHA-256 of a 5,000-item set and
you add one item, you must re-hash all 5,001 items. There is no shortcut.

A homomorphic hash is built so that the hash of a *set* can be updated
incrementally when the set changes, without re-reading the whole set. LtHash
works roughly like this: each item is expanded (via a keyed hash) into a long
vector of numbers, 1024 bits worth, and the hash of the whole set is the
component-wise sum of all those vectors, with each component reduced modulo a
fixed size (16-bit lanes, hence LtHash*16*). Addition is the combining rule.

Because the set hash is just a big sum, it has a magic property:

- To add an item, add its vector into the running sum.
- To remove an item, subtract its vector out of the running sum.
- Order does not matter, because addition commutes.

So the hash after applying a batch of mutations equals the hash you would get by
applying them one at a time, in any order. That is the "homomorphic" promise. In
the open-source implementations the core operation is literally named
`SubtractThenAdd(base, subtract, add)`: take the current 1024-bit hash, subtract
out the vectors for the values you are removing, add in the vectors for the
values you are setting, and you have the new hash. The WhatsApp instance is keyed
with the HKDF label "WhatsApp Patch Integrity" and a 128-byte (1024-bit) hash
size.

Now the payoff. Every device keeps one 1024-bit LtHash value per collection. When
Priya archives the Rahul chat on her laptop:

1. Laptop computes the new LtHash by SubtractThenAdd: subtract the old
   archive=false vector for that chat, add the new archive=true vector. One
   subtract, one add. It does not touch the other 4,999 settings.
2. Laptop puts the new expected LtHash into the patch and uploads it.
3. Phone downloads the patch, applies the mutation locally, and independently
   does the same SubtractThenAdd on its own copy.
4. Phone compares its freshly computed LtHash to the expected one in the patch.
   Match means "I am correctly in sync." Mismatch means "I dropped or misapplied
   something, I must resync from a full snapshot."

The cost of verifying a change is O(number of changed items), not O(size of the
whole collection). One archived chat costs one subtract and one add, whether
Priya has 5 chats or 5,000. That is the property that lets app state sync scale.

On top of LtHash for ordering-and-completeness integrity, each patch also carries
HMACs: a value MAC per mutation, and a patch MAC over the whole patch, keyed by
secrets only the devices hold. So the server cannot forge a patch, cannot flip
archive to unarchive, and cannot silently drop one. If it tries, either the HMAC
fails or the LtHash diverges, and the device falls back to a clean snapshot
resync. (Real-world proof this is load-bearing: open-source WhatsApp libraries
routinely hit "mismatching LTHash" errors when a patch chain gets confused, and
the only cure is a full app-state resync. The check really does fire.)

### The scale story at three tiers

Tier one, a handful of devices, small groups. Naive everything works. Client-
fanout to 3 or 4 devices is nothing. App state is tiny. You could even keep the
old phone-as-proxy model and survive. Nothing forces cleverness yet. This is
Priya on day one with just her phone and one laptop, 12 total chats.

Tier two, big groups and busy accounts, the pairwise fanout wall. Now imagine a
250-person WhatsApp group, and multi-device means each member has up to 5
devices. If you tried to client-fanout the message body, one message becomes up
to 250 x 5 = 1,250 encryptions and 1,250 server deliveries, *per message*, and a
chatty group sends thousands of messages a day. That is the quadratic-ish blowup
that kills naive group encryption. The survival move is Sender Keys: encrypt the
body once, distribute the per-device sender key once, let the server fan out a
single ciphertext. Encryption cost per message drops from 1,250 to 1. The
O(devices) cost is paid once at key-distribution time and amortized away. This is
the same "do the expensive thing rarely, do the cheap thing on the hot path"
spine that runs through the whole ledger.

Tier three, 2.5 billion users and 100 billion-plus messages a day, with device
counts multiplying everything. Three things break and three things save them.

- Message volume multiplies by device count. A world where every user has 3
  devices is a world with up to 3x the delivery events. Survival: the store-and-
  forward post office from the ticks teardown already shards per device and keeps
  a queue per device; adding devices adds queues, which is horizontal and cheap.
  Each envelope is idempotent and keyed by target device, so it self-routes.

- App state cannot be re-shipped in full. If every archive/mute re-downloaded a
  5,000-contact snapshot to 5 devices, the metadata traffic would dwarf the
  messages. Survival: incremental patches plus the LtHash homomorphic check, so a
  one-setting change costs one subtract-add and a tiny patch, and full snapshots
  happen only on first link or after a detected divergence.

- Prekeys can run dry. Every device needs a stock of one-time prekeys on the
  server so strangers can start sessions with it while it is offline (the X3DH
  setup from the E2E teardown). Multiply devices by users and the prekey pool is
  under constant drain; a popular account whose devices are offline can have its
  prekeys exhausted, forcing a fallback path. Survival: devices replenish prekeys
  when they come online, and the protocol has a reduced-security fallback when the
  one-time prekey is missing, exactly as in the single-device case.

And the 14-day rule is a scale-and-safety valve, not a UX whim. A companion
device that never re-touches the phone can drift: its device signature is stale,
its state may be far behind, and if the phone was lost or stolen you want dead
devices to auto-expire rather than linger forever holding a valid identity.
Requiring the phone to check in every 14 days bounds how long a forgotten or
stolen companion can keep decrypting, and lets the server garbage-collect device
lists and queues for accounts that have gone fully dark.

### Fact versus inference

Facts, from WhatsApp's own engineering post and Security Whitepaper and the open-
source clients that reimplement the wire protocol: the client-fanout model and
one ciphertext per device; up to four companion devices; the phone-can-be-offline
design with the 14-day idle limit; Automatic Device Verification via Account and
Device signatures; the App State Sync collections (critical_block,
critical_unblock_low, regular_low, regular_high, regular); and LtHash16 with the
"WhatsApp Patch Integrity" HKDF label and 1024-bit values updated by subtract-
then-add, verified alongside HMACs.

Inference, clearly labeled: the exact server-side sharding of device queues, the
precise prekey pool sizes, and the internal batching of patches are not published
in detail. Where I described those, I used the well-grounded "this is how this
class of problem is solved" version, consistent with the ticks and E2E teardowns
already in this ledger, and I flagged it as such.

## 8. The retention and habit mechanic

Multi-device is not an engagement loop. There is no streak, no daily nudge, no
red dot. It is a **switching-cost and lock-in** play, the same defensive
retention mechanic we saw in the WhatsApp encryption teardown, and it moves
retention by removing a reason to leave rather than manufacturing a reason to
return.

Here is the real, observed mechanic. For years WhatsApp's biggest desktop
weakness, the phone-must-be-on dependency, was the exact thing Telegram beat it
on. Telegram's desktop app was a full client because Telegram is not end-to-end
by default, so people who lived on a laptop all day had a concrete reason to keep
Telegram open in a tab. Every "ugh, my phone died and WhatsApp Web froze" moment
was a small push toward a rival. Multi-device deleted that push. Now the person
who works on a laptop all day, Priya, has no functional reason to prefer the
competitor's desktop experience, and she keeps her whole network on WhatsApp.

The loop that builds is quiet but strong: the more surfaces WhatsApp runs on
(phone, two laptops, tablet, work desktop), the more woven into your whole
digital day it becomes, and the more painful it is to move any of it elsewhere,
because moving means moving all five surfaces and re-anchoring the trust in a new
app. Each linked device is another root that has to be pulled up before you can
leave. That is retention as entanglement.

The metric it moves is retention and churn resistance, especially among the
highest-value, most-active users (people at a desk all day, businesses running
support from a computer). It does not directly move activation (new signups
still come from the phone) and it does not directly move revenue for consumer
WhatsApp. It moves the boring, decisive number: the probability that a heavy user
still opens WhatsApp next month instead of drifting to a competitor with a better
desktop story. WhatsApp closed that gap and took the reason-to-drift off the
table.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader editor that compiles to shippable code plus an
embeddable runtime. That is inherently a multi-surface product. Someone edits a
shader graph on a desktop, previews it on a phone, and ships it into an embedded
runtime on a customer's site. The temptation is to make the desktop editor the
one true source and have every other surface proxy through it or re-fetch the
whole graph on every change. That is exactly the pre-2021 WhatsApp mistake, and
it fails the same way: the moment the editor is closed or offline, the other
surfaces freeze.

Two concrete, actionable moves, both biased to scalability:

First, give each surface an independent, verifiable replica of the graph, not a
puppet view of the editor. The embedded runtime should keep working with the
editor offline, the way a companion device keeps working with the phone off. Sync
through an incremental patch log of mutations (SET node param, ADD connection,
REMOVE node), not by re-shipping the whole graph. A shader graph with 400 nodes
should never re-send 400 nodes because one float parameter changed.

Second, and this is the piece to steal outright, use a **homomorphic rolling
checksum per collection** to detect divergence in O(changed), not O(graph size).
Split the graph state into collections (nodes, connections, parameters, uniforms)
and keep a running LtHash-style sum for each. When a single uniform changes on
one surface, update that collection's checksum with one subtract-then-add, ship
the tiny patch plus the new expected checksum, and let the receiving surface
verify with a subtract-then-add on its own copy. If the cheap checksum matches,
you are provably in sync for the cost of one item. Only when it disagrees do you
pay for a full snapshot resync. For a live-editing tool where a designer drags a
slider and expects the phone preview and the embedded runtime to track in real
time, that difference, O(1) verification per edit versus O(whole graph), is the
difference between a preview that feels instant at 400 nodes and one that hitches
every time you touch a knob. Cheap verification of large replicated state is a
scalability primitive, and WhatsApp shipped the textbook version of it inside a
chat app.

---

## Sources

- Engineering at Meta, "How WhatsApp enables multi-device capability" (2021):
  https://engineering.fb.com/2021/07/14/security/whatsapp-multi-device/
- WhatsApp Security Whitepaper (Encryption Overview, covers Automatic Device
  Verification, client-fanout, app state sync):
  https://www.whatsapp.com/security/WhatsApp-Security-Whitepaper.pdf
- InfoQ, "WhatsApp Adopts the Signal Protocol for Secure Multi-Device
  Communication" (2021):
  https://www.infoq.com/news/2021/07/WhatsApp-signal-protocol/
- whatsmeow appstate package (open-source reimplementation of WhatsApp app state
  sync; collections and patch handling): https://pkg.go.dev/go.mau.fi/whatsmeow/appstate
- LTHash package documentation (SubtractThenAdd, "WhatsApp Patch Integrity" HKDF,
  1024-bit LtHash16): https://pkg.go.dev/github.com/testovoleg/whatsapp/appstate/lthash
- whatsmeow issue "mismatching LTHash" (real-world app-state resync behavior):
  https://github.com/tulir/whatsmeow/issues/651
- Signal Protocol Sender Keys (group messaging scheme):
  https://en.wikipedia.org/wiki/Sender_Keys
- Prior ledger teardowns this builds on: 2026-06-15 WhatsApp delivery receipts
  (store-and-forward post office) and 2026-07-01 WhatsApp end-to-end encryption
  (X3DH, Double Ratchet, Sender Keys).
