# Gmail Conversation Threading: how a pile of loose emails becomes one conversation

Date: 2026-08-10
Product: Gmail
Feature: Conversation view / message threading (grouping replies into one thread)

## 1. The user

Priya runs operations at a 30-person startup in Bengaluru. It is Tuesday
morning. She opens Gmail and there are 214 unread messages. One of them is a
question she asked yesterday: "Can we move the vendor call to Friday?" Since
then, four people have replied. Rahul said yes. Meera said she has a conflict.
The vendor's assistant proposed 3pm. Rahul replied again to lock 3pm.

Priya does not want to see five separate emails scattered across her inbox, each
titled "Re: Vendor call Friday," each demanding she remember what the other four
said. She wants to open one thing, read top to bottom, and know the call is
Friday at 3pm. That one thing is a conversation. She never thinks about how those
five separate emails found each other. They just did.

## 2. The real problem

Email was never designed as a chat. Every message is a standalone object that
gets mailed to a server, sits in a mailbox, and knows almost nothing about the
other messages around it. When Rahul hits reply, his mail client makes a brand
new message with its own unique id and sends it off. It is not "attached" to
Priya's original in any physical way. It is a fresh envelope dropped in the same
mailbox.

So the raw inbox is a flat list of thousands of unrelated envelopes. If you do
nothing, a five-person discussion looks like five random rows, often not even
next to each other because other mail arrived in between. You end up scrolling,
hunting, and re-reading. You lose the plot. Before Gmail, that was normal:
Hotmail and Yahoo showed you a flat list and you coped.

The pain is simple. A human conversation got shredded into loose pieces by the
mail system, and the user has to glue it back together in their head every single
time. The feature's whole job is to glue it back so the user never has to.

## 3. The feature in one sentence

Gmail looks at hidden headers each email carries (its own id, and the ids of the
messages it is replying to) plus the subject line, decides which loose emails
belong to the same discussion, and collapses them into one expandable
conversation row.

## 4. Jobs to be done

- "When I open my inbox, show me one line per discussion, not one line per email,
  so a 40-message party-planning thread is one row, not 40."
- "When a new reply lands, put the whole conversation back at the top and tell me
  how many messages are in it, so I know something moved."
- "Let me read a discussion top to bottom in order, so I do not have to
  reconstruct who said what."
- "Do not glue together two emails that just happen to share a boring subject like
  'Hi' or 'Update' but are actually unrelated."
- "Keep working even when someone's mail client sends slightly broken or
  out-of-order headers, which is most of them."

## 5. How it works for the user

The inbox shows conversations, not messages. A row reads "Priya, Rahul, Meera 5"
with the subject "Vendor call Friday." The number 5 is the message count. The row
is bold because there is something unread inside.

Priya clicks it. The conversation expands into a stack, oldest at the top, newest
at the bottom, each message collapsed to a one-line summary she can click open.
The most recent message is expanded by default. If two people replied while she
was reading, a small "New messages" bar appears. When she replies, her reply
joins the bottom of the same stack. She never picked a thread. She never linked
anything. The grouping already happened before she looked.

One concrete detail users feel: if Rahul changes the subject line from "Vendor
call Friday" to "3pm confirmed," Gmail often starts a new conversation, because
Gmail leans on the subject as part of its grouping rule. Pure email clients that
thread only on the hidden ids would keep it together. This is a real, visible
quirk of Gmail specifically, and we will see why under the hood.

## 6. The actual flow, step by step

Walk one real discussion. Four messages, four people.

1. Priya composes "Vendor call Friday" and sends it. Her mail server stamps it
   with a globally unique Message-ID, say `<a1@corp.example>`. No reply headers,
   because it starts the thread.
2. Rahul hits reply. His client makes a new message `<b2@rahul.example>`, keeps
   the subject as "Re: Vendor call Friday," and adds two hidden headers:
   `In-Reply-To: <a1@corp.example>` and `References: <a1@corp.example>`. That
   References list is the breadcrumb trail.
3. Meera replies to Rahul. Her message `<c3@meera.example>` carries
   `References: <a1@corp.example> <b2@rahul.example>`. The trail now has two
   crumbs, in order: the root first, the immediate parent last.
4. The vendor's assistant replies to the whole chain. Message
   `<d4@vendor.example>`, `References: <a1@corp.example> <b2@rahul.example>
   <c3@meera.example>`. Three crumbs.

Every one of those messages arrived at Gmail as a separate delivery, possibly
seconds or hours apart, possibly out of order if a server was slow. Gmail's job at
each delivery is to read the References trail, find the earlier messages already
sitting in Priya's mailbox, and stamp the new message with the same conversation
id. By the time Priya opens her inbox, all four wear the same thread id and show
as one row of "5" (four here, plus her own sent reply lands in the same thread).

## 7. Under the hood, like the engineer

This is the heart of it. There are two questions and they are different questions:
matching (which emails belong together) and presentation (how to lay the thread
out and keep the inbox fast). Gmail's exact internal code is not public, so the
honest structure is: the published, canonical algorithm for the matching half is
Jamie Zawinski's threading algorithm from 1997, used in Netscape Mail and adopted
across mail clients and the IMAP THREAD extension. Gmail's observable behavior
matches that family with one documented twist (it also insists on the subject).
I will label fact versus inference as I go.

### The raw material: three headers (fact, from the mail standards)

Every email carries, per the internet message format standard (RFC 5322):

- `Message-ID`: a globally unique string for this one message, like
  `<a1@corp.example>`. Think of it as a primary key minted by the sender.
- `In-Reply-To`: the Message-ID of the single message this one directly answers.
- `References`: the ordered list of Message-IDs of the whole ancestor chain, root
  first, immediate parent last.

These ids are the entire basis for threading. The subject is a weak backup. The
body is never used. So the problem reduces to: given a bag of messages, each
knowing its own id and the ids of its ancestors, reconstruct the family tree.

### The data structure: a forest built through one hash map (fact, JWZ)

The core structure in JWZ's algorithm is the Container. Each Container has four
fields:

- `message`: the actual email, or null if we have only heard about this id but
  never seen the message itself (a dummy).
- `parent`: link up to the Container this one replies to.
- `child`: link down to the first reply.
- `next`: link sideways to the next sibling.

That is a general tree node. The magic that makes it fast is a single hash map
called the `id_table`, keyed by Message-ID, value is that message's Container.
This is the same move the ledger keeps seeing: a hash map turns "find the message
with this id" from a scan of the whole mailbox into an O(1) lookup. Without it,
linking a reply to its parent would be a linear search; with it, it is one probe.

### Building the tree, step by step (fact, JWZ)

Pass one, for each message:

1. Look up the message's own Message-ID in `id_table`. If a Container already
   exists there (because some earlier reply mentioned this id in its References
   before we had actually seen the message), reuse it and drop the real message
   into its empty `message` slot. If not, make a new Container and store it. If a
   Container with a real message is already there, this is a duplicate id, so mint
   a Container under a fake id instead. Duplicates happen and the algorithm just
   shrugs.

2. Walk the References list in order and, for each id, get-or-create its Container
   from `id_table`, then link each one as the parent of the next. When Meera's
   message says `References: <a1> <b2>`, we ensure a Container exists for `<a1>`,
   a Container for `<b2>`, and that `<a1>` is the parent of `<b2>`. Crucially, if
   `<b2>` had not arrived yet, we still create an empty dummy Container for it
   now. That is how out-of-order delivery is survived: the slot is reserved, and
   when `<b2>` finally lands, pass one fills its `message` field. The tree can be
   built before all the leaves exist.

3. Set the parent of this message's own Container to the Container of the last id
   in its References (its immediate parent). If it already had a presumed parent
   from some earlier reply's breadcrumb, throw that away and use this better one,
   because now we have the message itself talking.

Two guards keep garbage input from blowing up, and they are the interesting part:

- Loop guard: before linking A as the parent of B, check that A is not already a
  descendant of B. If it is, adding the link would create a cycle, so skip it. Mail
  headers in the wild are frequently malformed or forged, and without this check a
  crafted References field could make an infinite loop. JWZ notes the algorithm was
  field-tested by something like ten million users over six years and survived
  malicious input. The loop guard is why.

- Do-not-overwrite-good-links: if two ids are already linked, do not relink them
  based on a weaker signal. Existing structure wins.

After pass one, the root set is every Container with no parent.

### Pruning the dummies (fact, JWZ)

The tree is now full of empty Containers, the dummies we created for ids we had
heard of but not yet seen, plus ids we will never see (someone replied to a
message that was deleted or never delivered to this mailbox). Pass two walks the
tree and prunes:

- An empty Container with no children: delete it, it carries nothing.
- An empty Container with exactly one child: promote the child up, the dummy was
  just a placeholder.
- An empty Container with several children that is not at the root: splice all its
  children up to its grandparent, again removing a useless middle node.

Concrete case from our thread: suppose the vendor's message `<d4>` arrived first,
before `<a1>`, `<b2>`, `<c3>` had synced. Pass one would build three dummies for
the ancestors and hang `<d4>` off the bottom. As the real messages arrive, the
dummies fill in with real content and no pruning is needed. But if the very first
message `<a1>` had been sent from an account Priya cannot see and never lands in
her mailbox, its Container stays a dummy forever. Pruning promotes `<b2>` and its
descendants up so the thread still reads cleanly, just rooted at Rahul's reply
instead of Priya's original.

### The subject step, and where Gmail diverges (fact for the rule, inference for Gmail internals)

After pruning, JWZ does a subject-based merge on the root set only. It normalizes
each root's subject (strip leading "Re:", "Fwd:", "R:", and so on), buckets roots
by normalized subject in a `subject_table`, and merges roots that share a subject
but were not linked by ids, on the theory that they are probably the same
discussion where someone lost the References header. Then it sorts: siblings by
date, the root set by the date of each thread's earliest message.

Here is the Gmail-specific twist, and it is documented behavior, not internals.
Gmail's conversation view historically grouped messages when they shared a
subject and arrived within a roughly one-week window, or replied to each other.
That over-grouped: two unrelated emails both titled "Hi" could land in one
conversation. In March 2019 Google tightened the rule. Now, if an incoming
message carries a References header, that header must actually reference the ids
of earlier messages in the thread for Gmail to group it. A "definite
relationship" is required. Subject alone is no longer enough to merge two
messages that do not reference each other.

The net Gmail rule, as users experience it, is an AND, not the pure-id OR that
strict clients use. Gmail groups when the ids line up AND the normalized subject
still matches. That is why changing the subject line in Gmail can split a thread
even though the References chain is intact: Gmail is also checking the subject,
where a pure JWZ client would keep it together on the ids alone. The documented
subject handling even allows prefixes ("test" and "re: test" thread; valid
prefixes include RE:, R:, FWD:), and two same-subject same-sender emails will not
thread unless one explicitly references the other.

The inference, clearly labeled: Gmail at billions of users almost certainly does
not run batch JWZ over a whole mailbox on every inbox open. That would be far too
slow. The observable rules are JWZ-family for the matching logic, but the
mechanism underneath is near-certainly incremental, described next.

### Where the sort and the grouping happen, and at what scale

The sort and grouping happen on the server, in the store, not on the phone. The
phone receives an already-threaded list. Now the scale story, three tiers.

Tier one, about 1,000 messages (a single person's folder, or one mailing-list
archive). Run full JWZ in memory. Build the `id_table`, link, prune, subject-merge,
sort. It is O(n) in the number of messages because every step is a hash-map
operation or a single tree walk. On a thousand messages this is a few milliseconds.
This is exactly what Netscape Mail did on the client in 1997 with far slower
machines. Nothing breaks here.

Tier two, about 100,000 messages (a heavy user's multi-year archive, or a busy
public mailing list like the Linux kernel list). Full re-threading on every inbox
open is now wasteful: you would rebuild the same forest thousands of times a day.
The 100k Containers still fit in memory (tens of megabytes), so memory is not the
wall, latency is. The fix is to stop treating threading as a read-time batch job
and make it a write-time decision that you persist. Store a `threadId` on each
message. Keep an index from Message-ID to `threadId` for the mailbox. When a new
message arrives, look up its References ids in that index, find the existing
thread, and stamp the new message with that `threadId` once. Reading the inbox
becomes "group by threadId, order by newest," served from an index. What broke at
this tier was repeated recomputation; the fix was compute-once-on-write.

Tier three, 10 million plus messages and, more to the point, billions of mailboxes
with constant delivery (Gmail). Re-threading a mailbox is O(n) and you have
billions of them receiving mail every second. You cannot afford it, so threading
is purely a delivery-time act (inference, but the only design that fits the
numbers). When one email is delivered:

1. Extract its `References` and `In-Reply-To` ids, a handful of strings.
2. Probe the per-mailbox Message-ID to threadId index for those ids. A few hash
   lookups, cost tracks the size of this one message's ancestor list, not the size
   of the mailbox.
3. Apply Gmail's rule: if a referenced id is found and the normalized subject
   matches, adopt that existing `threadId`. Otherwise mint a new one.
4. Write the message with its `threadId` and index its own Message-ID pointing at
   that thread, so the next reply can find it.

That is the whole hot path: constant work per delivered email, independent of how
big the mailbox is. Reading the inbox is then a cheap grouped query on an indexed
`threadId` column, newest thread first. The expensive-thinking-offline,
cheap-lookup-online spine the ledger keeps finding shows up here as
think-at-write, lookup-at-read.

Two things break at this tier and both are handled. First, a runaway thread: a
company-wide "reply all" storm, or a mailing list where a single subject collects
thousands of replies, would create one unbounded conversation object that is slow
to open and expensive to update. Gmail caps a conversation at 100 messages, then
starts a fresh conversation with the same subject. That bound keeps any single
thread object small and predictable. Second, sharding: mailboxes are partitioned
by user, so the Message-ID index and the threading decision are entirely local to
one user's shard. No cross-user coordination is ever needed to thread, because a
conversation lives inside one mailbox. That makes the whole system embarrassingly
parallel across users, the same partition-by-owner move as sharding a shopping
cart by account.

One more real detail on the index. Message-ID is a string like
`<CAF8=xY...@mail.gmail.com>`, high cardinality and effectively random, so the
index is a hash index, not a range scan. Looking up "does `<b2@rahul.example>`
exist in this mailbox and what thread is it in" is one probe. This is the same
inverted-index-style idea used everywhere in the ledger: precompute the map from
key to location so the live question is a lookup, never a scan.

## 8. The retention and habit mechanic

Conversation view is a clutter compressor, and clutter is the enemy of opening an
app. A 40-message "Diwali party planning" discussion is one inbox row, "Priya,
Rahul, Meera 40," not 40 rows. The inbox stays short, so it stays openable. That
is the activation and retention lever: Gmail launched in 2004 with three
differentiators against Hotmail and Yahoo's flat lists, search, a full gigabyte of
storage, and conversation view. The threaded inbox was a signature reason people
switched and stayed.

The repeat-open loop is the unread bump. When a new reply lands, the whole
conversation jumps back to the top of the inbox, goes bold, and the message count
ticks up (that "40" becomes "41"). An unanswered thread sitting bold at the top is
an open loop, the same psychological nag as an unseen Instagram story ring or an
unread WhatsApp badge. It pulls the next open. The metric it moves is retention
through reduced cognitive load, plus engagement through the bump. A real observed
example: reply to any active Gmail thread and watch it rise to the top of both
your inbox and every other participant's, bold, counted. That rise is the hook.

## 9. The lesson for Rare.lab

A Rare.lab shader graph is exactly this problem wearing different clothes. Each
node references upstream nodes by id; the editor has to reconstruct a directed
graph from a bag of nodes that each know only their own id and the ids they pull
from. That is the JWZ Container problem. Steal three specific moves.

First, build the graph through one id-keyed hash map and tolerate dangling
references with placeholder nodes. When a user pastes a partial subgraph, or loads
a project file where one node type is missing because it came from a newer
version, do not crash and do not drop the wire. Create a dummy node for the
missing id, hang the dependents off it, and fill it in if the real node shows up,
exactly like JWZ's empty Containers absorbing out-of-order and missing messages.
A node editor that survives half-broken input feels solid; one that throws on a
missing reference feels fragile.

Second, guard against cycles before you connect, not after. JWZ refuses to add a
parent link if it would make a loop, checked by asking "is the proposed parent
already a descendant of the child." A shader DAG must stay acyclic or the compiler
loops forever. Run that same descendant check the instant a user drags a wire from
node A's output toward node B, before committing the edge, and reject the
connection with a clear reason. The check is a cheap graph walk and it is the
difference between a compiler that cannot hang and one that can.

Third, and most important for scale, thread at write time, not at render time.
Gmail assigns a threadId once when a message arrives and the inbox read is a cheap
grouped lookup; it does not re-thread the mailbox on every open. Do the same with
the shader graph. Resolve the topological evaluation order once, when the graph is
edited, and cache it. The compiled, shippable runtime should carry the
pre-resolved order as a flat array, so the per-frame path is a straight walk of
that array, constant work per node, with zero graph reconstruction at 60 or 120
frames per second. Reconstructing dependencies every frame is the mail client that
re-threads the whole mailbox on every open: correct, and far too slow. Think at
edit time, look up at frame time.

## Sources

- Jamie Zawinski, "message threading" (the canonical JWZ algorithm, Netscape Mail
  and News): https://www.jwz.org/doc/threading.html
- IETF, IMAP THREAD extension (the REFERENCES and ORDEREDSUBJECT threading
  algorithms, standardized JWZ):
  https://datatracker.ietf.org/doc/html/draft-ietf-imapext-thread-10
- akuchling/jwzthreading, a reference Python implementation of the JWZ algorithm:
  https://github.com/akuchling/jwzthreading
- fdietz/jwz_threading, another JWZ implementation:
  https://github.com/fdietz/jwz_threading
- Google Workspace Updates, "Threading changes in Gmail conversation view" (the
  March 2019 definite-relationship change):
  https://workspaceupdates.googleblog.com/2019/03/threading-changes-in-gmail-conversation-view.html
- 9to5Google, "Gmail Conversation View now requires 'definite relationship' to
  thread emails": https://9to5google.com/2019/03/29/gmail-conversation-view-thread/
- cloudHQ Support, "How does Gmail decide to group emails into conversations?":
  https://support.cloudhq.net/how-does-gmail-decide-to-group-emails-into-conversations/
- Gmail Help, "Group emails into conversations":
  https://support.google.com/mail/answer/5900
- Gmail API, "Manage threads" (threadId, References, In-Reply-To, subject rules for
  programmatic threading):
  https://developers.google.com/workspace/gmail/api/guides/threads
- RFC 5322, Internet Message Format (Message-ID, In-Reply-To, References headers):
  https://datatracker.ietf.org/doc/html/rfc5322
