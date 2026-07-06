# References: Amazon 1-Click ordering (Buy Now)

Saved 2026-07-06. Keepers for the 1-Click / Buy Now teardown.

## Primary sources

- **US Patent 5,960,411**, "Method and system for placing a purchase order via a
  communications network." Amazon.com, granted September 1999.
  https://patents.google.com/patent/US5960411A/en
  - The mechanism: customer enters address + payment once, is handed an
    identifier stored in a client cookie. On a later single action the client
    sends the item plus the identifier; the server's "single-action ordering
    component" retrieves the previously stored info and generates the order.
  - Order consolidation: the server may combine single-action orders placed
    within a time period (patent gives 90 minutes as an example) when their
    expected ship dates are similar, to save on shipping and reduce confusion.
  - Expired in 2017; the surviving public form is the "Buy Now" button.

- **DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store," SOSP
  2007.** The canonical primary source for Amazon's checkout/cart at scale.
  https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
  - Shopping cart / session is the motivating "always writeable" example: an
    "add to cart" must never be rejected even under disk, network, or data
    center failure.
  - 99.9th-percentile latency SLA (about 300ms) rather than average; averages
    hide the tail and the tail is real customers.
  - Consistent hashing with virtual nodes; sloppy quorum with N/R/W (typical
    N=3, R+W>N); hinted handoff so writes still succeed under failure.
  - Vector clocks: list of (node, counter) pairs to capture causality. Concurrent
    "sibling" versions are handed to the application to reconcile.
  - Shopping-cart reconciliation = union merge: keep every item from every
    sibling. Documented tradeoff: a deleted item can resurface, accepted because
    a lost "add to cart" (missed sale) is worse than a stale row.
  - Runs on tens of thousands of servers across many data centers.

- **Werner Vogels, "Amazon's Dynamo," All Things Distributed (2007).**
  https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html

## Operating-scale numbers

- **AWS, "How AWS powered Prime Day 2024 for record-breaking sales."**
  https://aws.amazon.com/blogs/aws/how-aws-powered-prime-day-2024-for-record-breaking-sales/
  - DynamoDB peaked at 146 million requests/second, no latency issues.
  - Order processing at 500,000 orders/minute; peak ~2 million orders/hour
    (2.5x and 2x the 2023 figures respectively).
  - CloudFront over 500M HTTP req/min peak, 1.3 trillion total.

## Pattern / supporting

- **AWS Step Functions, state machine structure** (order-processing reference:
  a wait state after order entry so the customer can fix the address, then
  parallel reserve-inventory / capture-payment / notify-fulfillment states).
  https://docs.aws.amazon.com/step-functions/latest/dg/statemachine-structure.html
- **Baymard Institute**, cart abandonment (~70%) and checkout-length as a top
  cause. https://baymard.com/lists/cart-abandonment-rate

## The one-line insight

Split the decision from the data-entry. A client identifier keys an O(1) lookup
of stored defaults so one tap becomes a full order; an idempotency token plus a
hold-and-merge window make the fast path safe; and the cart/session underneath is
an always-writeable Dynamo object whose union-merge encodes the business's risk
appetite (never lose a sale, tolerate a resurrected item). Return "placed" first,
do the expensive reserve/charge/fulfill work async after. Offline-think,
online-lookup, one more time.
