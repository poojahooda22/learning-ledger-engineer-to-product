# References: Swiggy live order tracking

Saved keepers for the 2026-06-24 teardown on Swiggy's live order tracking (the
moving rider map and the "Arriving in X min" timer).

## Primary (Swiggy engineering)

- **How ML powers When is my Order coming, Part I** (Swiggy Bytes). The four-leg ETA
  decomposition: O2A (Order to Assignment), FM (First Mile), WT (Wait Time), LM (Last
  Mile), each its own model, refreshed at fixed intervals. Features: restaurant type
  (cloud kitchen vs dine-in), item count/variety, restaurant stress (orders placed vs
  prepared ratio), rider availability in the hyperlocal zone, historical and
  near-real-time FM/LM speed from DE GPS pings.
  https://bytes.swiggy.com/how-ml-powers-when-is-my-order-coming-part-i-4ef24eae70da

- **The OSM Distance Service Part 1** (Swiggy Bytes). GraphHopper map matching on
  OpenStreetMap to snap noisy GPS to the road network; two-wheeler (motorcycle) routing
  profile because Swiggy deliveries are predominantly two-wheelers; evaluation metrics
  for routing configurations.
  https://bytes.swiggy.com/the-osm-distance-service-part-1-evaluation-metrics-and-routing-configurations-6e8686ca814f

- **Designing the Serviceability Platform at Swiggy for High Scale, Part 1** (Swiggy
  Bytes). Geohash-bucketed in-memory index of clusters served from memory.
  https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-1-751a631f0379

- **Architecture and Design Principles Behind the Swiggy Delivery Partners app** (Swiggy
  Bytes). Background location for 4-5 hour sessions; event-driven delivery flow honoring
  a finite state machine per delivery.
  https://bytes.swiggy.com/architecture-and-design-principles-behind-the-swiggys-delivery-partners-app-4db1d87a048a

## Streaming / infrastructure

- **Confluent customer case study, Swiggy.** Kafka for location and order event
  streaming at large scale. https://www.confluent.io/customers/swiggy/

- **How Platforms Like Zomato, Swiggy, Uber, and Ola Update Rider's Location in Real
  Time** (DEV Community). The canonical streaming-not-storage pipeline: phone -> Kafka
  -> location processor -> in-memory/Redis state -> WebSocket gateway -> customer app;
  GPS every 3-5s; location as event stream not DB row.
  https://dev.to/rachit_avasthi/how-platforms-like-zomato-swiggy-uber-and-ola-update-riders-location-in-real-time-3ic5

## Routing / academic

- **Batching and Matching for Food Delivery in Dynamic Road Networks** (arXiv 2008.12905).
  Road-network graph model for batching orders and matching to delivery agents.
  https://arxiv.org/abs/2008.12905

- **GraphHopper** map matching (Hidden Markov Model + Viterbi) and contraction-hierarchy
  routing for sub-millisecond shortest paths. https://www.graphhopper.com/

## Scale context

- **Swiggy Annual Report FY 2024-25.** ~923M orders in the year (up 22% YoY), 690k+
  delivery partners, 260k+ restaurants, 1,100+ dark stores, 653 cities, ~23M monthly
  transacting users; 3-4M orders/day across food and quick commerce.
  https://www.swiggy.com/corporate/wp-content/uploads/2025/07/Swiggy-Annual-Report-FY-2024-25.pdf

## Key takeaways carried into the report

1. Live location is a streaming problem, not a storage problem. The GPS firehose never
   touches the primary DB on the live path; it flows through Kafka, memory, and a socket.
2. Sparse truth, smooth fiction: send a position every 3-5s, interpolate to 60fps on the
   client. Same split as Figma multiplayer cursors.
3. Map matching (HMM + Viterbi over the OSM road graph) turns lying GPS into a believable
   rider on real roads.
4. The ETA is four refreshed leg-models (O2A, FM, WT, LM), not one. Decompose so each leg
   is measurable and fixable. The rider fleet's own pings are the traffic sensor.
5. Precompute offline (contraction hierarchies, geohash buckets), keep the live query a
   cheap lookup; partition by geography to spread load and isolate hot cities.
