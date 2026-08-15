# Gmail Undo Send: the five seconds that saved your career

Date: 2026-08-15
Product: Gmail
Feature: Undo Send (the "Message sent. Undo" toast, and the 5 to 30 second window behind it)

## 1. The user

Meet Ananya. She is a marketing manager in Bengaluru. It is 6:55 pm on a
Thursday. She is tired and wants to leave. She has been writing a reply to her
boss about the Q3 numbers. She types "Hi Rahul, here is the deck," attaches a
file, and hits Send.

One second later her stomach drops. She attached last quarter's deck. Or she
forgot the attachment entirely. Or she wrote "Rahul" but the email is going to
"Rohan." Or worse, she hit Reply All on a thread with 200 people and her reply
said "lol did you see this."

That half second of horror after you hit Send is one of the most universal
feelings in office life. Ananya is not editing a document. She fired a message
at another human, and she wants it back.

## 2. The real problem

Email has no take-backs. That is the whole pain in one line.

When you hit Send, in the old world, the message was gone. Not "gone from your
screen." Gone from your control. It left Gmail's servers, crossed the internet,
and landed in Rahul's inbox on his mail server. You cannot walk into his server
and pull the letter back out of his mailbox. It is his now.

People know this in their gut, which is why the panic is so physical. You want
an "are you sure?" But a confirmation popup on every single email is torture.
You send fifty emails a day. Forty nine of them are fine. A popup on all fifty
to catch the one mistake is a tax everyone hates.

So the real problem is sharper than "let me undo." It is: give me a safety net
for the one bad send without slowing down the forty nine good ones.

## 3. The feature in one sentence

Undo Send holds your outgoing email in a pending state for a few seconds
(5, 10, 20, or 30) so that if you catch a mistake in that window, one click
pulls it back into a draft before it ever leaves Gmail.

## 4. Jobs to be done

What is Ananya really hiring this feature to do?

- "Let me catch the obvious mistake I only see the instant after I send." The
  wrong attachment, the empty attachment, the typo in the first line, the wrong
  recipient.
- "Do not slow me down." No confirmation dialog on every email. The default
  send stays one click.
- "Give me my words back as a draft, not a apology." When she undoes, she wants
  to be back in the editor with everything intact, not staring at a lost
  message.
- "Let me stop being scared of the Send button." The deeper job. Take the fear
  out of sending so she moves faster.

Notice the tension. Job one wants a delay. Job two wants no friction. The genius
of the feature is that it serves both at once, and the user barely notices the
delay exists until they need it.

## 5. How it works for the user

Ananya hits Send. At the bottom of the screen a small black bar appears:
"Message sent." Next to it, one word: "Undo." And next to that, "View message."

The bar sits there for a few seconds. If she does nothing, it quietly
disappears and the email is truly on its way. If she clicks Undo, the bar
vanishes, and her email pops right back open in the compose window, exactly as
she left it, cursor and all. It is now a draft again. She fixes the attachment
and sends again.

That is the entire visible experience. There is a settings toggle too:
Settings, then General, then "Undo Send," where she can pick the cancellation
period. The choices are 5 seconds (the default), 10, 20, or 30. On the phone
app, the window is shorter and fixed, around 5 seconds, shown as an "Undo"
button on a toast at the bottom.

Real example of the choice mattering: a lawyer who reviews every word before
it leaves might set 30 seconds. A support agent firing 200 replies a shift
finds 30 seconds annoying (the email feels "stuck") and sets it to 5. The dial
is a personal speed-versus-safety setting.

## 6. The actual flow, step by step

1. Ananya clicks Send (or presses the keyboard shortcut).
2. The compose window closes immediately. To her, it looks sent. This is the
   important trick: the UI does not wait. It reacts instantly so sending still
   feels like one fast click.
3. Behind the scenes, the message is accepted by Gmail's servers and put into a
   pending, not yet delivered, state with a fire time a few seconds in the
   future.
4. The "Message sent. Undo." bar appears. A countdown is running that she cannot
   see.
5a. Path A, she does nothing. When the timer reaches zero, Gmail actually hands
    the message off to the delivery system. It goes out over SMTP to Rahul's
    mail server. Now it is gone for real. The bar had already faded.
5b. Path B, she clicks Undo within the window. Gmail cancels the pending send
    before the handoff ever happens. The message never went anywhere. It is
    moved back to Drafts and reopened in the composer.

The key: there is no "recall." In path B nothing is chased down and pulled back,
because nothing had left yet. The email spent its whole short life sitting
inside Gmail, waiting.

## 7. Under the hood, like the engineer

This is the heart of it, and the lesson is beautiful because the hard version
of this feature is impossible, and they shipped the easy version that works
perfectly.

### The impossible version everyone imagines

When people first hear "undo send," they picture recall. You sent a letter, and
somehow a hand reaches into the other person's mailbox and takes it back. That
is how Microsoft Outlook's "Recall This Message" tries to work, and it is why
Outlook recall is famous for failing. It only works if both people are on the
same Exchange server inside the same company, and even then only if the reader
has not opened it. The moment the mail crosses to a different server, recall is
dead.

Here is the engineering reason, and it is worth saying plainly. Email delivery
runs on SMTP. When Gmail sends Ananya's mail to Rahul at, say, an Outlook
address, Gmail's mail transfer agent opens a connection to Microsoft's incoming
server and runs a short conversation: HELO, MAIL FROM, RCPT TO, DATA, then the
whole message, then a final dot. Microsoft's server says "250 OK, I have it."
That is a one way door. SMTP has no "un-send" command. There is no verb in the
protocol for "please forget the message I just handed you." Once the receiving
server says 250 OK, the message is legally and technically theirs. You cannot
get it back. Confirmed fact, this is how SMTP works (RFC 5321).

So a true recall across the open internet is not hard, it is impossible. Any
feature that promised it would be lying.

### The version they actually built: do not send yet

The insight is a reframe. Do not try to reverse the send. Just do not perform
the send until later. Put a small delay between the click and the real SMTP
handoff, and let the user cancel during that delay.

This is confirmed by Google's own framing. When the feature launched in Gmail
Labs in March 2009, engineer Yuzo Fujishima and designer Michael Leggett
described it as holding your message for a few seconds before actually sending
it. Fujishima's original idea was even bigger: a special outbox where a message
could sit in limbo for up to five minutes so you could still edit it. The
shipped feature is the same idea with a shorter, tunable window. It graduated
from Labs to a real Gmail setting in June 2015, about six years later, with the
5, 10, 20, 30 second choices.

So the mental model is: hitting Send does not mean "deliver now." It means
"schedule delivery for now plus N seconds, and let me cancel until then."

### The data structures in play (grounded inference)

Google has not published the exact internals of the pending queue, so the next
part is the well grounded "this is how this class of problem is solved" version,
labeled as inference.

The message is already stored. Gmail persists your mail the moment you send
(that is why Sent and Drafts survive a browser crash). Undo Send does not need a
second copy. It needs one extra piece of state on the message: a status
(pending versus sent) and a fire_at timestamp equal to now plus the window. On
undo, flip the status to draft and clear the timestamp. This is a cheap mutation
of a row that already exists, not a new heavy object. That is why the feature
costs almost nothing to store.

The scheduler that fires the send at fire_at is the interesting part. You have
hundreds of millions of little "send at time T" jobs in flight at any moment.
The classic ways to hold "do X at time T" jobs are:

- A min-heap (priority queue) keyed by fire_at. The soonest job is always on
  top. A worker peeks the top, sleeps until that time, pops it, sends. Good for
  one machine, but a single global heap does not shard across thousands of
  machines cleanly, and every insert is log(n).
- A timing wheel. Imagine a clock with slots, one slot per second. A 7 second
  delay drops the job into the slot 7 ticks ahead. Every tick, a hand moves one
  slot and fires everything in it. Insert and delete are O(1), not log(n). This
  is exactly the structure built for "huge numbers of short lived timers," the
  same idea Linux kernel timers and Kafka's delay queues use. For a fixed small
  window like 5 to 30 seconds, a timing wheel is close to ideal.
- A time sorted scan. Write pending sends to a store indexed by fire_at, and run
  a sweeper every second that grabs all rows whose fire_at has passed and
  dispatches them. Simple, shards well, and a second of slack does not matter
  here.

Any of these turns "wait 10 seconds" into cheap bookkeeping. The concrete
example: Ananya's email at 6:55:03 pm with a 10 second window gets a fire_at of
6:55:13. It lands in the "13 seconds past the minute" slot. At 6:55:13 the hand
sweeps that slot and hands her mail to the SMTP layer, unless an undo already
plucked it out.

Undo is the cheap operation on purpose. Cancelling is "remove this one job from
its slot," which is O(1) in a timing wheel and a single keyed delete in a sorted
store. Compare that to the impossible recall path, which would be an unbounded
chase across foreign servers. This is the whole trick: they chose a design where
the "undo" is a small local delete instead of a large remote reversal.

### Client side or server side, and why it must be server side

A lot of blogs say the delay is a client side timer in your browser. That is
almost certainly wrong for the real feature, and the reasoning is a nice test of
understanding.

If the delay lived only in the browser tab (the tab waits 10 seconds, then
sends), then closing the tab right after clicking Send would break it. Either
the message would never go out, or it would fire instantly with no undo window.
Neither happens. In practice you can hit Send, close the tab, and the mail still
goes out a few seconds later. That behavior only makes sense if the server is
holding the scheduled send. So the authority for "has this been sent yet" is
server side. The browser's job is only to show the "Undo" toast and, if clicked,
fire one tiny cancel request to the server. Labeled as inference from observable
behavior, but it is strong inference.

### Matching and ranking? No. This is a timing problem, not a search problem

Most features in this ledger split into matching then ranking. Undo Send does
not. There is no catalog to fetch from and nothing to rank. The hard part is not
"which item," it is "when," and the correctness rule is "never deliver before
the window closes, and never fail to deliver after it." It is a scheduling and
state machine problem: draft, then pending with a deadline, then either sent or
back to draft. That state machine is the feature.

### The scale story at three tiers

Tier one, 1,000 users, a startup's internal mail server. Trivial. A single min
heap of pending sends in memory on one box. A background thread pops the top and
sends. If the box reboots you might lose a few pending sends, but at this scale
you would just send immediately on restart and no one notices.

Tier two, 100,000 users, a mid size company. Now pending sends must survive a
crash, because losing "in flight" mail is unacceptable. You persist the pending
state (status plus fire_at) to a durable store and run sweeper workers. What
breaks at the next tier: a single sweeper scanning one global table cannot keep
up, and a single machine's timing wheel is a single point of failure.

Tier three, Gmail scale. Gmail serves on the order of 1.8 billion users and
moves hundreds of billions of messages a day. At any instant, a large but
bounded set of messages are inside their 5 to 30 second window. What they do to
survive, grounded in standard practice:

- Bounded window is your friend. Because the delay is at most 30 seconds, the
  pending set never grows without limit. It is roughly "messages sent in the
  last 30 seconds," which is a fixed size working set, not an ever growing
  backlog. This is why the feature is cheap at planetary scale. Contrast with
  Fujishima's original 5 minute idea, which would have held 10x more mail in
  limbo at once. The short window is a scale decision, not just a UX one.
- Shard the pending sends the same way mailboxes are sharded, by user. Ananya's
  pending send lives on the shard that owns Ananya's mailbox. No global queue,
  no cross shard coordination. Each shard runs its own timing wheel or sweeper.
  This is the same partition by user id pattern seen in the Notion and Stripe
  teardowns.
- No extra copy of the message. The mail is already persisted for Sent and
  Drafts. Undo Send adds a status field, not a duplicate. Storage cost is a
  rounding error.
- Idempotent, exactly once handoff. When the timer fires, the send must happen
  once and only once, even if a worker crashes and a backup worker retries. So
  the dispatch is guarded the same way payments are: mark the message "handed to
  SMTP" atomically before the network call, so a retry after a crash does not
  double send. This is the idempotency lesson from the Stripe teardown applied
  to mail.

The thing that would break if you got it wrong: a slow undo path. If clicking
Undo had to search for the message across the system, undo would sometimes miss
the window and you would send the email you were trying to kill. They avoided
that by making undo a direct keyed cancel of a job you already know the location
of. Fast path stays fast.

### One honest limit

The window is a hard wall. Confirmed fact: after the 30 second cap on web (or
about 5 seconds on mobile), there is no undo, because the message has already
gone out over SMTP and is now someone else's property. The feature does not
extend your control past the delay. It only uses the delay. That honesty is why
it never overpromises the way Outlook recall does.

## 8. The retention and habit mechanic

Undo Send is not an engagement loop. There is no feed to scroll, no streak, no
red dot pulling Ananya back. It sits in the same family as WhatsApp encryption
and Stripe Radar from this ledger: defensive retention built on trust, not
attention.

The loop is emotional and it is real. Every time Ananya clicks Undo and it saves
her from a wrong attachment, the feature earns a little more of her loyalty. The
memory of "Gmail caught my mistake" is sticky. It shows up constantly in
"favorite Gmail features" lists and in the relieved way people talk about it.
The metric it moves is retention through trust, and underneath that, confidence.
A user who is not scared of the Send button sends more email and leans on Gmail
harder, which raises switching cost. Moving to a mail app without a safety net
would feel like driving without a seatbelt once you have had one.

There is a quieter activation angle too. The feature lowers the anxiety of a new
or nervous user. Knowing you can take it back for a few seconds makes you braver
in the product from day one.

The concrete tell: this feature lived in Labs for six years as an opt in
experiment, and enough people fell in love with it that Google promoted it to a
default setting in 2015. Features graduate from Labs when usage and affection
prove out. Undo Send passed that bar precisely because the people who used it
would not give it up.

## 9. The lesson for Rare.lab

Rare.lab compiles a node graph into shippable shader code and pushes it to an
embeddable runtime. Some of those actions are irreversible in the same way an
SMTP handoff is: publishing a shader to production, shipping a runtime update to
live sites, overwriting a deployed asset viewers are already loading. Once it is
out on a customer's page, you cannot pull it back.

The lesson: do not try to build "rollback after the fact" for your irreversible
actions. Build a deferred commit with a cancel window, exactly like Undo Send.

Concretely:

- Make the irreversible step (the actual publish or deploy) the last step, and
  bracket it. Everything before it (compile the graph, validate, stage the
  artifact) is reversible and can happen instantly. The one way door is the
  final handoff, and only that.
- Insert a short, tunable hold between "user clicked Publish" and the real
  push. Default it to a few seconds. Show a "Published. Undo." toast. During the
  hold, the deploy is a scheduled job with a fire_at, not a done deal.
- Make cancel an O(1) delete of that scheduled job, not a reverse migration.
  Canceling should be "do not fire the deploy," which is cheap and always
  correct, versus "un-deploy what already shipped," which is expensive and
  sometimes impossible.
- Keep the hold server side, not in the editor tab. If the designer closes the
  browser after clicking Publish, the deploy should still complete. The client
  only renders the undo affordance and fires the cancel request.
- Keep the window bounded and short so the pending set stays a fixed size
  working set, and shard pending deploys by project so there is no global queue.
  This is the same "bounded window keeps it cheap at scale" move Gmail made when
  it chose seconds over Fujishima's original five minutes.

The one line to tape to the monitor: when an action cannot be undone, do not
build undo, build a delay. The cheapest correct way to reverse something is to
not have done it yet.

## Sources

- Official Gmail Blog, "New in Labs: Undo Send" (March 2009), the original
  announcement by Michael Leggett and Yuzo Fujishima:
  https://gmail.googleblog.com/2009/03/new-in-labs-undo-send.html
- Google Blog, "How to undo send in Gmail for up to 30 seconds":
  https://blog.google/products/gmail/how-to-unsend-email-gmail/
- Vice, "The Wonder of Undo Send" (history, Leggett and Fujishima, the five
  minute limbo outbox idea): https://www.vice.com/en/article/the-wonder-of-undo-send/
- NPR, "Gmail now features a way to ease sender's remorse" (June 2015 launch):
  https://www.npr.org/sections/thetwo-way/2015/06/24/417117823/gmail-now-features-a-way-to-ease-senders-remorse
- CBC, "Gmail users get undo send option in new email settings" (June 2015):
  https://www.cbc.ca/news/canada/british-columbia/gmail-users-get-undo-send-option-in-new-email-settings-1.3125272
- Fortune, "Google's Gmail now lets you unsend emails" (June 2015):
  https://fortune.com/2015/06/23/google-gmail-undo-send
- RFC 5321, Simple Mail Transfer Protocol (why there is no un-send verb once a
  receiving server returns 250 OK): https://datatracker.ietf.org/doc/html/rfc5321
- Background on timing wheels for large numbers of short lived timers
  (Varghese and Lauck, "Hashed and Hierarchical Timing Wheels"):
  https://www.cs.columbia.edu/~nahum/w6998/papers/ton97-timing-wheels.pdf
