# References: Gmail Undo Send (2026-08-15)

## Primary sources

- Official Gmail Blog, "New in Labs: Undo Send" (March 19, 2009). The original
  announcement (Michael Leggett, Yuzo Fujishima). Framing: Gmail holds your
  message for a few seconds before actually sending it.
  https://gmail.googleblog.com/2009/03/new-in-labs-undo-send.html
- Google Blog, "How to undo send in Gmail for up to 30 seconds." Official
  description of the 5/10/20/30 second windows.
  https://blog.google/products/gmail/how-to-unsend-email-gmail/
- RFC 5321, Simple Mail Transfer Protocol. The technical reason there is no
  recall: SMTP has no un-send command; once the receiving server returns 250 OK
  the message is delivered.
  https://datatracker.ietf.org/doc/html/rfc5321

## History and launch

- Vice, "The Wonder of Undo Send." History of the feature, Michael Leggett's
  idea and Yuzo Fujishima's engineering, and Fujishima's original five minute
  limbo outbox concept.
  https://www.vice.com/en/article/the-wonder-of-undo-send/
- NPR, "Gmail now features a way to ease sender's remorse" (June 24, 2015). The
  graduation from Labs to a standard setting.
  https://www.npr.org/sections/thetwo-way/2015/06/24/417117823/gmail-now-features-a-way-to-ease-senders-remorse
- CBC, "Gmail users get undo send option in new email settings" (June 2015).
  https://www.cbc.ca/news/canada/british-columbia/gmail-users-get-undo-send-option-in-new-email-settings-1.3125272
- Fortune, "Google's Gmail now lets you unsend emails" (June 23, 2015).
  https://fortune.com/2015/06/23/google-gmail-undo-send

## Engineering background (the scheduling primitive)

- Varghese and Lauck, "Hashed and Hierarchical Timing Wheels: Efficient Data
  Structures for Implementing a Timer Facility." The O(1) timer structure for
  large numbers of short lived timers; the class of structure a bounded delayed
  send would use.
  https://www.cs.columbia.edu/~nahum/w6998/papers/ton97-timing-wheels.pdf

## Key facts captured

- Not a recall, a deferred send. The message sits pending inside Gmail for
  5/10/20/30 seconds (web, default 5, hard cap 30) or about 5 seconds fixed on
  mobile, then hands off to SMTP. Undo cancels the pending send before handoff.
- Born in Gmail Labs March 2009, promoted to a default setting June 2015.
- The delay is server side (strong inference from observable behavior: closing
  the tab after Send still sends the mail seconds later).
- Storage cost is near zero because the message is already persisted for
  Sent/Drafts; Undo Send only adds a status plus a fire_at timestamp.
