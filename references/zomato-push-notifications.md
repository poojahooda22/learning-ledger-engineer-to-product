# References: Zomato push notifications and the personalized nudge engine (2026-09-30)

Saved for the 2026-09-30 teardown. Sourcing is layered: APNs/FCM delivery is
public fact; the platform engineering is drawn from Swiggy and Uber (direct
competitors solving the identical problem at identical scale) and used as the
well-grounded "how this class is solved" reference; Zomato's own targeting
signals and copy strategy are confirmed at the marketing/behaviour level, and
anything about Zomato's internal engineering is inference, clearly labeled in
the report.

## Delivery fundamentals (APNs / FCM / device tokens) [CONFIRMED FACT]
- back4app glossary, "Push Notifications: APNs, FCM, Device Tokens, Web Push (2026)":
  https://www.back4app.com/glossary/push-notifications-apns-fcm/
  Key facts: each device keeps exactly one persistent, battery-cheap socket to
  its vendor gateway (APNs for Apple, FCM for Android); the delivery chain is
  backend -> APNs/FCM -> the one OS socket -> app; the device token is the
  address; push reaches closed apps, websockets serve open ones.

## Communication platform (Swiggy Kabootar) [CLASS REFERENCE]
- Swiggy Bytes, "Kabootar, Swiggy's Communication Platform":
  https://bytes.swiggy.com/kabootar-swiggys-communication-platform-e5a43cc25629
  Self-contained MESSAGE object so any node can process it; pipeline of
  microservices Event -> Campaign -> Templating -> Router (channel: push/SMS/
  email); message state machine continuously persisted so non-final messages are
  re-picked after a crash; retry/fallback handled internally to the message;
  transactional vs marketing at different priorities.

## Smart push / weekly plan (Swiggy) [CLASS REFERENCE]
- Swiggy Bytes, "Smart Push notifications (Multi-Armed Bandits at Swiggy, Part 4)":
  https://bytes.swiggy.com/smart-push-notifications-multi-armed-bandits-at-swiggy-part-4-f5698f2af0a6
  Multi-armed bandits orchestrate eligible pushes into an optimized weekly plan
  per user under constraints (frequency capping), balancing exploration vs
  exploitation.

## Real-time push platform (Uber RAMEN) [CLASS REFERENCE]
- Uber Engineering, "Uber's Real-Time Push Platform":
  https://www.uber.com/us/en/blog/real-time-push-platform/
  RAMEN = Realtime Asynchronous MEssaging Network. Fireball decides when to push;
  Streamgate (Netty) holds the live connections. Gen1: Node.js + Ringpop
  consistent hashing sharded by user UUID + Redis; wall at ~30K RPS / 1.5M
  concurrent connections (gossip convergence slowdown, single-thread event-loop
  GC pauses). Gen2 (rebooted early 2017): Netty + Zookeeper + Apache Helix
  (sharding/rebalancing) + Redis + Cassandra; 1.5M -> 15.5M concurrent
  connections; ~70,000 messages/sec at peak.

## Send-time optimization + CCG (Uber) [CLASS REFERENCE]
- Uber Engineering, "How Uber Optimizes the Timing of Push Notifications using ML
  and Linear Programming":
  https://www.uber.com/us/en/blog/how-uber-optimizes-push-notifications-using-ml/
  XGBoost predicts P(order within 24h | push, time) = value of each (push, time)
  pair. Scheduling framed as an assignment problem, solved with an integer linear
  program. Constraints encode business rules: push expiry, send window (e.g.
  morning only), daily frequency cap (e.g. <=2/day), minimum gap (e.g. 8h),
  restaurant open hours. Consumer Communication Gateway (CCG) = centralized layer
  controlling relevance, order, timing, and frequency of pushes at the user level.

## Zomato targeting signals + copy strategy [CONFIRMED at marketing/behaviour level]
- PushPilot, "Zomato and Swiggy Push Notifications: Why They Convert So Well":
  https://pushpilot.ai/blog/zomato-swiggy-push-notification-teardown
- Juno School, "Zomato's Push Notification Strategy":
  https://www.junoschool.org/article/zomato-push-notification-strategy/
  Four per-user signals: cuisine preferences, amount spent, time of ordering,
  browsing history. Triggers: meal-time (morning/lunch/dinner), weather (soup on
  rainy days), cricket match days tied to the emotional high. Copy is witty /
  meme-style, reads like a friend texting.

## Retention / uninstall statistics [CONFIRMED]
- Business of Apps, push notifications research:
  https://www.businessofapps.com/marketplace/push-notifications/research/push-notifications-costs/
  1 push/week -> ~10% disable notifications, ~6% uninstall; 6+/week uninstalls at
  3.4x the rate of 2-5/week.
- Mobiloud, "50+ Push Notification Statistics" (compiling Localytics):
  https://www.mobiloud.com/blog/push-notification-statistics
  46% opt out after 2-5 pushes in a week; 32% after 6-10.
- Airship, "How Push Notifications Impact Mobile App Retention Rates":
  https://www.airship.com/newsroom/urban-airships-mobile-app-retention-study-for-key-industry-verticals/
  A major reason new users churn is NOT receiving push at all; relevance-to-
  frequency ratio matters more than raw frequency.
