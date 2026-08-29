# References: Amazon Lightning Deals and flash-sale contested inventory

Saved 2026-08-29 for the Amazon Lightning Deals teardown
(product-teardowns/2026-08-29-amazon-lightning-deals.md).

## Confirmed user-facing mechanics (Amazon + seller docs)

- Amazon Customer Service, "Amazon Lightning Deal Waitlist" (first-come waitlist,
  Join Waitlist button active only when spots exist and deactivated when full,
  notification when a spot opens, short time limit to add to cart, drop if you
  miss it, waitlist expires when the promotion expires):
  https://www.amazon.com/gp/help/customer/display.html?nodeId=201894810
- Amazon Customer Service, "Amazon Lightning Deals on Prime Day":
  https://www.amazon.com/gp/help/customer/display.html?nodeId=GW3L8JX7Q9FH8ALB
- SellerApp, "How Do Amazon Lightning Deals Work" (4-12hr window usually ~6,
  countdown timer, progress bar showing % claimed, add-to-cart claims a unit,
  15-minute checkout window): https://www.sellerapp.com/blog/amazon-lightning-deals/
- Tinuiti, "Amazon Lightning Deals: Everything Sellers Need to Know":
  https://tinuiti.com/blog/amazon/what-are-amazon-lightning-deals/
- The Krazy Coupon Lady, "Amazon Lightning Deals" (shopper walkthrough of the
  bar and waitlist): https://thekrazycouponlady.com/tips/money/amazon-lightning-deals
- Groupon, "Amazon Lightning Deals Explained and When They Are Worth It":
  https://www.groupon.com/coupons/blog/amazon-lightning-deals-guide-when-worth-it

## Flash-sale engineering (the class of problem; Amazon internals not public)

- Himanshu Singour, "Big Billion Sale 2025: System Design" (Flipkart: Redis stock
  counter, distributed lock on the SKU, 5-minute reservation TTL, Kafka events,
  auto-release on TTL expiry):
  https://medium.com/@himanshusingour7/big-billion-sale-2025-system-design-282f84d28e8f
- Ajit Singh, "Flash Sale System Design: Architecture, Scale, and Oversell"
  (atomic Lua check-and-decrement, durable queue + 202 Accepted, cache the read
  path, dead-letter queue): https://singhajit.com/flash-sale-system-design/
- tanhdev, "Shopee Architecture Chapter 2: Flash Sale Engine, Solving Overselling
  and Hot Keys" (single hot key on one SKU, inventory sharding into buckets,
  Redis EVAL atomicity): https://tanhdev.com/series/shopee-architecture/02-flash-sale-engine/
- Umesh Kushwaha, "Designing a Flash Sale System That Never Oversells, From 1 User
  to 1 Million Users (Without Crashing Redis)" (split stock into N keys, decrement
  a random bucket, aggregate to check global empty):
  https://medium.com/@umesh382.kushwaha/designing-a-flash-sale-system-that-never-oversells-from-1-user-to-1-million-users-without-8426db0f1ad0
- Sujeet Jaiswal, "Design a Flash Sale System" (one-per-user enforcement,
  reservation TTL, message-queue decoupling):
  https://sujeet.pro/articles/design-flash-sale-system
- Cracking Walnuts, "System Design: E-Commerce Flash Sales (10M Users, Coupon
  System, One-Per-User Enforcement)": https://crackingwalnuts.com/post/flash-sale-system-design

## Key takeaways carried into the teardown

- The feature reduces to one contested integer: "units remaining," decremented
  concurrently by a crowd, must never go below zero.
- Split write path (rare, atomic, exactly correct) from read path (huge, cached,
  slightly stale). Never serve the "% claimed" bar from the sacred counter.
- Naive single SQL row with `WHERE remaining > 0` is correct but becomes a single
  hot row under contention. Move to Redis, make check-and-decrement one atomic Lua
  step.
- Add-to-cart is a reservation with a 15-minute TTL, not a sale. Expired holds
  self-release and the bar can tick back down.
- Waitlist is a FIFO queue for fairness (confirmed first-come-first-served).
- At the top tier a single Redis key becomes a hot key: shard inventory into N
  buckets, decrement a random bucket, retry another bucket before declaring sold
  out. Absorb spikes with a durable queue (202 Accepted) and a virtual waiting
  room.
