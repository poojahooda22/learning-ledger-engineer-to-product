# References: Stripe Webhooks (event delivery system)

Saved 2026-07-31 for the teardown at
`product-teardowns/2026-07-31-stripe-webhooks-event-delivery.md`.

## Primary (Stripe official)

- Receive Stripe events in your webhook endpoint. Delivery model, retries up to
  3 days with exponential backoff in live mode, endpoint auto-disable after
  prolonged failure, at-least-once delivery.
  https://docs.stripe.com/webhooks
- Check webhook signatures. `Stripe-Signature` header format (`t=`, `v1=`),
  signed payload = `timestamp + "." + raw_body`, HMAC-SHA256 with the `whsec_`
  endpoint secret, 300-second default timestamp tolerance, constant-time
  comparison, must use the raw request body.
  https://docs.stripe.com/webhooks/signature
- List all events (Events API). 30-day full retention then summary, `types`
  filter up to 20, the Event object shape (`id`, `type`, `data.object`,
  `api_version`, `created`), polling as an alternative to webhooks.
  https://docs.stripe.com/api/events/list
- Stripe Support: event retention period (30 days full, summary after).
  https://support.stripe.com/questions/stripe-event-retention-period

## Deep dives / corroborating (secondary)

- Svix, Stripe Webhooks Review. Retry schedule and signature scheme walk-through.
  https://www.svix.com/resources/webhook-reviews/stripe-webhooks-review/
- WebhookWatch, Stripe Webhook Retry Policy Explained. Any non-2xx (including
  4xx) triggers retries, not only 5xx.
  https://www.webhookwatch.com/article/stripe-webhook-retry-policy-explained
- HookRelay, Stripe Webhook Retry Behavior Explained. Approx 16 attempts across
  3 days, the backoff ladder (immediate, ~5m, ~30m, ~2h, then every several h).
  https://www.hookrelay.io/guides/stripe-webhook-retry
- DEV Community, At-least-once vs exactly-once delivery guarantees. Why the
  industry settles on at-least-once plus idempotent consumers.
  https://dev.to/letanure/at-least-once-vs-exactly-once-understanding-message-delivery-guarantees-4bhj
- Hookdeck, Webhook Infrastructure at Scale. Queues, retries, partitioning to
  isolate slow consumers, Stripe billions-per-month scale context.
  https://hookdeck.com/webhooks/guides/webhook-infrastructure-guide

## Key facts pulled

- Delivery is at-least-once; events can arrive more than once and out of order.
  Dedupe by `evt_id`; do not assume ordering.
- Live-mode retries: up to 3 days, exponential backoff, roughly 16 attempts.
  Sandbox: roughly 3 attempts over a few hours. Any non-2xx retries.
- Endpoint auto-disabled and owner notified after ~3 days of continuous failure.
- Signature: `signed_payload = t + "." + raw_body`, HMAC-SHA256 with `whsec_`,
  300s tolerance, constant-time compare, raw body required.
- Events retained 30 days in full via `GET /v1/events`; the append-only event
  log is the source of truth, webhooks are one delivery mechanism, polling is
  another.
- Inference (labeled in the report): the delivery backbone is a durable queue
  plus a time-ordered delay structure, partitioned per endpoint/account to
  isolate head-of-line blocking. Stripe runs large Kafka deployments publicly
  but has not published webhook internals in detail.
