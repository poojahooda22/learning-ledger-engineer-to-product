# References: Uber Upfront Fares (2026-09-13 teardown)

Keeper links for the upfront-fares teardown. Uber's own domains (uber.com, help.uber.com)
were blocked by the network egress proxy during this run, so details below were pulled from
search result snippets of those pages plus the accessible community summary. Links are kept
for future re-fetch when access allows.

## Primary (Uber)

- Upfront Fares: No Math and No Surprises (India blog)
  https://www.uber.com/in/en/blog/upfront-fares-no-math-and-no-surprises-3/
  The consumer framing and the "no math" promise.

- Ride Prices and Rates: How It Works, Upfront Pricing
  https://www.uber.com/us/en/ride/how-it-works/upfront-pricing/
  What goes into the price: estimated time and distance, demand for the route, tolls, taxes,
  surcharges; wait-time fees are the exception.

- Uber Marketplace: Upfront Pricing
  https://www.uber.com/us/en/marketplace/pricing/upfront-pricing/
  "Upfront fares help create certainty for riders, which leads to more trip requests." The
  conversion claim.

- Uber Help: How do upfront fares work? (riders)
  https://help.uber.com/en/riders/article/how-do-upfront-fares-work?nodeId=5073140f-3d5f-4046-80da-2db9ed7b11b3

- Uber Help: My upfront fare was not honored (riders)
  https://help.uber.com/en/riders/article/my-upfront-fare-was-not-honored?nodeId=ff65490e-2ffb-41cf-a709-4611521c7b24
  Honored vs not-honored rules: faster route still pays the upfront fare; address changes,
  detours, and material time blowouts can adjust it; the receipt explains why.

- Uber Help: How is the price of a trip determined?
  https://help.uber.com/riders/article/how-are-fares-calculated/?nodeId=d2d43bbc-f4bb-4882-b8bb-4bd8acf03a9d
  Base fare + per-minute + per-distance + booking fee + surcharges, minimum fare, dynamic.

## Primary (Uber Engineering, DeepETA and routing)

- DeepETA: How Uber Predicts Arrival Times Using Deep Learning
  https://www.uber.com/us/en/blog/deepeta-how-uber-predicts-arrival-times/
  Routing engine sums segment-wise traversal times along the best path (Dijkstra); DeepETA
  predicts the RESIDUAL between routing ETA and observed outcomes; uRoute frontend calls
  Michelangelo Online prediction service.

- Engineering More Reliable Transportation with ML and AI at Uber
  https://www.uber.com/us/en/blog/machine-learning/
  Michelangelo serves up to 10 million predictions per second at peak across matching,
  pricing, and routing.

- Scaling Real-Time Traffic Forecasting with a Graph-Aware Transformer
  https://www.uber.com/us/en/blog/scaling-real-time-traffic/
  Live edge-weight (traffic) forecasting that feeds ETA.

## Secondary (accessible, corroborating)

- Community architecture summary of DeepETA (GitHub)
  https://github.com/shubacca/EngineeringArchitectureSummaries/blob/main/Uber%20DeepETA%20prediction%20service.md
  Confirms: residual prediction; shallow encoder-decoder with linear self-attention; seven
  architectures tested; continuous features quantile-bucketed; geohashing plus multiple
  independent hash functions for location; categorical embeddings; asymmetric Huber loss
  (underprediction penalized differently from overprediction); "a few milliseconds at most"
  latency target; Canvas framework on Michelangelo; uRoute serving path.

## Fact vs inference

- CONFIRMED: fare components; upfront guarantee behavior and honored/not-honored rules;
  DeepETA residual approach, linear self-attention, quantile bucketing, geohash + feature
  hashing, asymmetric Huber loss, few-ms latency, Michelangelo up to 10M predictions/sec.
- INFERENCE (clearly labeled in the report): the internal fare-quote object format (signed
  token, fare id, TTL, silent re-quote); contraction hierarchies as the routing speedup.
