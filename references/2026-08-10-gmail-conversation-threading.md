# References: Gmail conversation threading (2026-08-10)

Keeper links for the email threading teardown.

## Primary / canonical algorithm

- Jamie Zawinski, "message threading." The canonical published algorithm behind
  mail threading, used in Netscape Mail and News 2.0/3.0 and Grendel. Defines the
  Container structure (message, parent, child, next), the id_table hash map keyed
  by Message-ID, linking via References and In-Reply-To, empty/dummy containers for
  unseen ids, the loop guard (do not link if it makes a cycle), pruning empty
  containers, root-set subject merge, and the final date sort. States it was
  field-tested by ~10M users over 6 years and survives garbage and malicious input.
  https://www.jwz.org/doc/threading.html

- IETF, "INTERNET MESSAGE ACCESS PROTOCOL - THREAD EXTENSION"
  (draft-ietf-imapext-thread). Standardizes two server-side threading algorithms:
  REFERENCES (the JWZ algorithm) and ORDEREDSUBJECT (a simpler subject+date
  grouping). The IMAP THREAD command returns threaded message sets.
  https://datatracker.ietf.org/doc/html/draft-ietf-imapext-thread-10

- RFC 5322, Internet Message Format. Defines the Message-ID, In-Reply-To, and
  References header fields that all threading is built on (root first, immediate
  parent last in References).
  https://datatracker.ietf.org/doc/html/rfc5322

## Reference implementations

- akuchling/jwzthreading. A clean Python implementation of the JWZ algorithm.
  https://github.com/akuchling/jwzthreading

- fdietz/jwz_threading. Another JWZ implementation, useful to cross-read the
  container/pruning logic.
  https://github.com/fdietz/jwz_threading

## Gmail-specific behavior (documented, not internals)

- Google Workspace Updates, "Threading changes in Gmail conversation view"
  (March 2019). The change that requires a "definite relationship": an incoming
  message's References header, if present, must reference IDs of previous messages
  in the thread to be grouped. Ends the old over-grouping where same subject within
  ~one week could thread unrelated mail.
  https://workspaceupdates.googleblog.com/2019/03/threading-changes-in-gmail-conversation-view.html

- 9to5Google, "Gmail Conversation View now requires 'definite relationship' to
  thread emails." Plain-language writeup of the same 2019 change.
  https://9to5google.com/2019/03/29/gmail-conversation-view-thread/

- cloudHQ Support, "How does Gmail decide to group emails into conversations?"
  The observable rules: Message-ID looped through References; grouping on same
  recipients/senders/subject or matching reference IDs; subject prefix handling
  (RE:, R:, FWD:); two same-subject same-sender mails will not thread unless one
  references the other.
  https://support.cloudhq.net/how-does-gmail-decide-to-group-emails-into-conversations/

- Gmail Help, "Group emails into conversations." The user-facing feature and the
  Conversation View toggle.
  https://support.google.com/mail/answer/5900

- Gmail API, "Manage threads." The programmatic threading contract: to append to a
  thread you must supply the threadId AND matching References/In-Reply-To headers
  AND a matching Subject; explains threadId semantics.
  https://developers.google.com/workspace/gmail/api/guides/threads

## The one-line insight

Reconstruct the conversation family tree from three headers by building a forest
of Containers through a single Message-ID hash map (O(1) parent lookup), reserve
empty dummy slots so out-of-order and missing mail never break the tree, guard
every link against cycles, prune the dummies, then (Gmail's twist) require both an
id match and a subject match. At billions of mailboxes, flip batch re-threading to
a delivery-time threadId stamp backed by a per-mailbox Message-ID index, so adding
one email is a few hash probes and reading the inbox is a grouped lookup:
think-at-write, look-up-at-read.
