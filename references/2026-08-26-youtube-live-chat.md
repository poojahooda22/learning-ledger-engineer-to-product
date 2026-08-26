# References: YouTube Live Chat and Super Chat (2026-08-26)

## Confirmed API contracts (primary, the reliable ground)

YouTube has not published an engineering deep-dive on Live Chat internals. The
public primary source is the YouTube Live Streaming API, whose fields leak the
shape of the system.

- LiveChatMessages: list. The `pollingIntervalMillis` field (server tells client
  how long to wait before polling again, adapts to chat activity), `nextPageToken`
  cursor, messages returned oldest-to-newest, "some or all" of history on first
  request. https://developers.google.com/youtube/v3/live/docs/liveChatMessages/list
- LiveChatMessages: streamList. Server push method that pushes new messages as
  they arrive, reduces polling and quota.
  https://developers.google.com/youtube/v3/live/docs/liveChatMessages/streamList
- LiveChatMessages resource. Message types: `superChatDetails`,
  `superStickerDetails`, `pollDetails`, membership gifting/receiving.
  https://developers.google.com/youtube/v3/live/docs/liveChatMessages
- YouTube Help, Learn about live chat. Top chat (filtered) vs Live chat
  (unfiltered), and the filtering signals (text, handle, channel, avatar).
  https://support.google.com/youtube/answer/15268877
- YouTube Help / moderation guides. Slow mode, blocked words, members-only,
  AutoMod. https://www.itechguides.com/moderate-live-chat-24-7-youtube-streams/

## Super Chat mechanics (secondary, consistent across creator guides)

- vidIQ, YouTube Super Chat guide. $1 to $500, tiered color and pin duration up to
  5 hours, 70/30 creator split.
  https://vidiq.com/blog/post/youtube-super-chat-guide/
- Gyre, Super Chat and Super Stickers guide. 2017 launch, pinned banner behavior.
  https://gyre.pro/blog/youtube-super-chat-super-stickers-everything-creators-need-to-know
- MIDiA Research, Super Chat as guerrilla marketing. 2017 launch context.
  https://www.midiaresearch.com/blog/youtubes-super-chat-is-a-guerrilla-marketing-gift-to-advertisers

## Fan-out and live-chat scale patterns (background, class-of-problem)

- vdocipher, Live streaming chat architecture. Pub-sub fanout, per-stream rooms,
  message brokers (Redis Pub/Sub, Kafka).
  https://www.vdocipher.com/blog/live-streaming-chat/
- System Design Handbook, Design a Live Comment System. Fan-out on read; the
  50,000 comments/sec at 10M concurrent viewers scale figure.
  https://www.systemdesignhandbook.com/guides/design-live-comment-system/

## What is confirmed vs inferred in the report

Confirmed: pollingIntervalMillis and its adaptivity, nextPageToken cursor,
oldest-to-newest ordering, streamList push, Top vs Live chat filtering, Super Chat
tiers/split/pin duration, the moderation controls.

Inference (clearly labeled inline, grounded in the standard broadcast-fan-out
solution, not in a YouTube post): one append-only partition per liveChatId,
read-side sampling on the busiest streams, absolute pin-expiry baked into the
Super Chat event checked lazily on read, moderation running as a pre-append filter
stage, and read-replica plus edge caching of the recent message window for a
single hot mega-stream.

## Cross-links to earlier teardowns in this ledger

- Fan-out on read for a celebrity object: 2026-06-27 Instagram Stories tray.
- The post-office store-and-forward inverse (few readers): 2026-06-15 WhatsApp
  delivery receipts.
- Idempotent payment that cannot be silently retried: 2026-06-20 Stripe
  idempotency keys.
- Move cost from per-interaction to per-title/per-effect: 2026-08-17 Netflix
  scrubbing thumbnails.
- Resolve the expensive check at write time, keep reads cheap: 2026-08-25 Notion
  search.
- Backpressure and load shedding: lesson 13. Hot-key / celebrity problem: lesson
  16. WAL and CDC: lesson 17. CDN zero origin egress: lesson 4.
