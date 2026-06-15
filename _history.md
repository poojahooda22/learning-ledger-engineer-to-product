# History

Running log of every teardown. Newest at the bottom. Never repeat a product+feature pair listed here.

| Date | Product | Feature | One-line insight |
|------|---------|---------|------------------|
| 2026-06-13 | Spotify | Discover Weekly | Keep the expensive thinking offline and static (matrix factorization + Annoy index), serve the live path as a cheap memory-mapped lookup; the Monday refresh manufactures a weekly habit loop. |
| 2026-06-14 | Uber | Surge pricing | Turn "where" into one integer (H3 hex index), count supply/demand per hexagon via streaming + geo-sharding, then precompute and cache so the rider's live request is a single keyed lookup; hexagons beat squares because all 6 neighbors are equidistant. |
| 2026-06-15 | WhatsApp | Delivery receipts (the ticks) | Phones never talk directly; the server is a post office that does receive, look up, forward, ack, delete. Each tick (sent, delivered, read) is a tiny idempotent status event keyed by message id, relayed back the same path; store-and-forward queues plus 2M+ connections per Erlang box keep the hot path five steps long at any scale. |
