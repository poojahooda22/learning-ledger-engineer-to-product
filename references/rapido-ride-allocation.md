# References: Rapido ride allocation / dispatch

Saved 2026-10-01 for the Rapido ride-allocation teardown.

## Primary (Rapido's own)

- Rapido Labs, "Improving Ride Dispatch with Data at Rapido."
  https://medium.com/rapido-labs/improving-dispatch-with-data-6a307dab7ecc
  Key facts: old radial system (draw ~2 km circle, collect captains inside,
  crow-fly distance, ping in order). The crucial component of smart dispatch is
  reliable driving-time estimates, built from historical ride data. Metrics
  optimized: ETA, matching time, distance driven by captain to reach customer
  (dead km), cancellations from demand and supply side.

- Rapido Labs, "Data Platform @ Rapido (Part I): cheap, efficient and scalable
  analytics."
  https://medium.com/rapido-labs/data-platform-rapido-part-i-cheap-efficient-and-scalable-analytics-52662111b2d2
  Key facts: Kafka event bus, millions of events per day, at-least-once
  guarantee, bronze/silver warehouse layers. The data foundation the ETA and
  acceptance models train on.

- Google Cloud, "Rapido" customer story.
  https://cloud.google.com/customers/rapido-maps
  Key facts: 45M rides/month, 4M active captains, 100 cities, 1B+ cumulative
  rides, <5% market penetration. Captain-location capture improved from ~60% to
  ~95% via multi-signal GPS + device sensors (captains switch GPS off). The
  road-topology quote: a captain may be closest but across the road in traffic,
  unable to U-turn for several km, so distance alone is not the matching signal.
  Direction-related support calls fell from 5% to <1%. Two-wheeler routing, ETA,
  live traffic via Google Maps Platform (implementation partner Searce).

## Secondary (press / analysis)

- "From Coordinates to Hexagons: The Design Behind Rapido's Dispatch."
  https://medium.com/@iwasifirshad/from-coordinates-to-hexagons-the-design-behind-rapidos-dispatch-4e7f699e955e
  Radial system -> Uber H3 hexagon grid, 64-bit cell IDs, equidistant neighbors,
  WebSockets for rider/captain/server live comms.

- CIO Tech Outlook, "Rapido Tech Boosts Ride Matching, Captain Earnings."
  https://www.ciotechoutlook.com/news/rapido-tech-boosts-ride-matching-captain-earnings-nid-14645-cid-160.html
  15,000+ ride requests/min; allocation in <15-20s avg; matching on distance +
  ETA + acceptance rate; demand forecasting from weather, events, ride history;
  dead-km reduction; improved captain earning efficiency.

- Digital Terminal, "Rapido Enhances Ride Experience and Earnings with Advanced
  Matching and Forecasting Engine."
  https://digitalterminal.in/startup/rapido-enhances-ride-experience-and-earnings-with-advanced-matching-and-forecasting-engine
  Hyperlocal demand maps updated in near real time; 2-3s matching compute budget.

- YourStory, "Rapido set to expand to 500 cities" (Jan 2025).
  https://yourstory.com/2025/01/rapido-expand-500-cities
  ~3.3M (33 lakh) rides/day across bikes, autos, cabs by mid-2025; expansion to
  400+/500 cities.

- Explorist, "Case Study: Rapido - 'x/y captains didn't accept your ride.'"
  https://explorist.in/case-study-rapido-x-y-captains-didnt-accept-your-ridecase-study-rapido/
  The "N captains did not accept" message = the sequential ping-in-order
  broadcast model surfacing in the UI.

## Reference (standard methods)

- Uber Engineering H3, "A Hexagonal Hierarchical Spatial Index," and docs at
  https://h3geo.org . Hexagonal cells, 64-bit IDs, k-ring neighbor expansion.
- Assignment problem / Hungarian algorithm (Kuhn, 1955), optimal bipartite
  matching for batched dispatch. Also referenced in Uber DISCO teardown
  (2026-07-02).

## One-line insight

Straight-line distance to a captain is a lie; Rapido's dispatch evolution is the
work of replacing it with learned real driving time + predicted acceptance, after
a cheap H3 cull, with the learning offline and the live decision a lookup.
