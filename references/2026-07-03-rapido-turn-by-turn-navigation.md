# References: Rapido turn-by-turn navigation (the routing engine)

Keeper links for the 2026-07-03 teardown.

## Rapido primary sources

- Rapido Labs (Inder Singh): "Rapido Navigation: Getting You There, Wherever You're Going" — https://medium.com/rapido-labs/rapido-navigation-getting-you-there-wherever-youre-going-867e1993c80f
  - Confirms the multi-provider in-app Navigation SDK for the captain app: real-time route optimization, turn-by-turn, traffic-aware routing. Three stated reasons for multi-provider: reduce single-vendor dependency, customization/flexibility (region/pricing/two-wheeler), foster competition/innovation.
- Rapido Labs (Siddharth Panchanathan): "Improving Ride Dispatch with Data at Rapido" — https://medium.com/rapido-labs/improving-dispatch-with-data-6a307dab7ecc
  - Driving-time estimates for dispatch built from the historical ride store.
- Rapido Labs index: https://medium.com/rapido-labs

## Rapido scale and outcomes (case studies)

- Google Maps Platform: "How Rapido is building customer and rider trust in India with Google Maps Platform" — https://mapsplatform.google.com/resources/blog/how-rapido-is-building-customer-and-rider-trust-in-india-with-google-maps/
  - Key numbers: ~4 million captains; ~60 million rides per month. Fleet Analytics raised captain location visibility from 60% to 95%, which improved matching and fulfilled orders per day. Two-wheeler routing, route planning, dispatch, ETA, live traffic.
- Google Cloud Rapido case study: https://cloud.google.com/customers/rapido-maps

## The routing-engine class (how this is universally solved)

- Geisberger, Sanders, Schultes, Delling: "Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks" (WEA 2008) — https://link.springer.com/chapter/10.1007/978-3-540-68552-4_24
  - Under 200 microseconds per exact query on the ~18M-node Western Europe graph, visiting a few hundred nodes; preprocessing minutes to hours, offline.
- "Exact Routing in Large Road Networks Using Contraction Hierarchies," Transportation Science (2012) — https://pubsonline.informs.org/doi/10.1287/trsc.1110.0401
- CH lecture notes (Geisberger et al., "Faster and Simpler," PDF) — https://turing.iem.thm.de/routeplanning/hwy/contract.pdf
- Project OSRM (open-source routing on OSM, contraction hierarchies) — https://github.com/Project-OSRM/osrm-backend
- GraphHopper (CH, car/bike/foot and custom profiles) — https://github.com/graphhopper/graphhopper
- Valhalla (tiled hierarchical routing, multimodal/two-wheeler costing) — https://github.com/valhalla/valhalla
- GraphHopper vs OSRM vs Valhalla comparison (2026) — https://www.pistack.xyz/posts/2026-04-25-graphhopper-vs-osrm-vs-valhalla-self-hosted-routing-engines-guide-2026/

## Map matching (keeping the icon on the road)

- Newson and Krumm: "Hidden Markov Map Matching Through Noise and Sparseness" (Microsoft Research, ACM SIGSPATIAL 2009) — https://www.microsoft.com/en-us/research/publication/hidden-markov-map-matching-noise-sparseness/

## Two-wheeler routing specifics (why car navigation is wrong)

- Google for Developers: "Get a two-wheeled vehicle route" (Routes API two-wheeler mode, India) — https://developers.google.com/maps/documentation/routes/route_two_wheel
- OpenStreetMap Wiki: Routing / Narrow Roads (standard engines like OSRM ignore road width) — https://wiki.openstreetmap.org/wiki/Routing/Narrow_Roads
- OpenStreetMap Wiki: India/Roads (Indian road hierarchy tagging) — https://wiki.openstreetmap.org/wiki/India/Roads
- Organic Maps discussion: request for a motorized two-wheeler profile (car mode routes onto banned expressways in India) — https://github.com/orgs/organicmaps/discussions/11932
