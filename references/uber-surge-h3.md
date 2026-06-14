# Uber surge pricing + H3 geospatial index (keeper links)

Saved 2026-06-14 for the Uber surge pricing teardown.

## Primary sources

- Uber Engineering, "H3: Uber's Hexagonal Hierarchical Spatial Index"
  https://www.uber.com/blog/h3/
  Why hexagons over squares (uniform neighbor distance, minimizes quantization
  error for moving users), hierarchical grid, surge uses per-hexagon supply/demand.

- H3 open-source repo (Uber): https://github.com/uber/h3
  16 resolutions (0 coarsest to 15 finest), hexagonal hierarchical indexing.

- H3 docs: https://h3geo.org/
  Resolution table, 122 base cells, 12 pentagons (icosahedron basis), 64-bit index,
  gridDisk / k-ring neighbor lookups, approximate parent-child nesting.

- "Real-time Data Infrastructure at Uber," arxiv 2104.00087
  https://arxiv.org/pdf/2104.00087
  Kafka as event backbone, streaming aggregation at millions of events/sec.

- Garg & Nazerzadeh, "Driver Surge Pricing," arxiv 1905.07544 (Management Science 2022)
  https://arxiv.org/abs/1905.07544
  Multiplicative surge is not incentive compatible in a dynamic setting; additive
  (flat-amount) surge is closer to incentive compatible. Basis for Uber's shift.

- Uber Marketplace surge page: https://www.uber.com/us/en/marketplace/pricing/surge-pricing/
- Uber Drive "How Surge Works" (driver heat map): https://www.uber.com/us/en/drive/driver-app/how-surge-works/

## Key facts to reuse

- Hexagon has 6 equidistant neighbors; square has 4 edge + 4 corner (1.41x farther).
- Only triangles, squares, hexagons tile a plane; hexagon is closest to a circle.
- Cannot tile a sphere with hexagons alone, so H3 has exactly 12 pentagons (placed over ocean).
- ~7x more cells per finer resolution (aperture 7).
- Surge is computed and frozen server-side at request time; phone is a thin client.
- Counting pattern: hash map keyed by H3 index, supply = sum over X + k-ring neighbors.
- Scale survival: Kafka streaming -> geo-sharded workers -> precompute + cache (Redis) keyed by H3 index, live path is one keyed read.

## Labeled inference (not officially published)

- Exact surge formula and the precise H3 resolution surge runs at.
- Exact cache cadence / Redis sweep interval (secondary blogs claim ~5s; treat as illustrative).
