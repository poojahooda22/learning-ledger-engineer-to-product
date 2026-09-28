# References: Swiggy restaurant serviceability (2026-09-28)

Keeper links for the serviceability teardown (which restaurants are eligible to
deliver to your pin: point-in-polygon accelerated by a spatial index).

## Primary (Swiggy engineering)

- Somsubhra Bairi, "What Serviceability means at Swiggy?", Swiggy Bytes.
  https://bytes.swiggy.com/what-serviceability-means-at-swiggy-c94c1aad352a
  Point-in-polygon customer cluster detection, directional discovery of target
  restaurant/store clusters, geo-filtering pickup vs drop locations, the ~4-5 km
  delivery window.

- Somsubhra Bairi, "Designing the Serviceability Platform at Swiggy for High
  Scale, Part 1", Swiggy Bytes.
  https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-1-751a631f0379
  The GeoHash-keyed in-memory cluster index; convert point to geohash key, fetch
  the reduced candidate cluster set, run point-in-polygon only on that set;
  O(1) resolution with zero live database queries; horizontal scaling via
  replicas each holding the in-memory index.

- Somsubhra Bairi, "Designing the Serviceability Platform at Swiggy for High
  Scale, Part 2", Swiggy Bytes.
  https://bytes.swiggy.com/designing-the-serviceability-platform-at-swiggy-for-high-scale-part-2-ab20365fbc23

## Geospatial delivery zones and spatial indexing

- DoorDash Engineering, "Scaling DoorDash's Geospatial Innovation with a
  Location-Based Delivery Simulator".
  https://doordash.engineering/2020/08/12/scaling-geospatial-innovation-with-a-location-simulator/
  S2 cells for delivery zones (zone stored as a set of cell IDs), geofences
  around merchants, radius queries.

- Joud Wawad, "The Complete Guide to Location Indexing: Geohash, Quadtree,
  Google S2, and Uber H3".
  https://joudwawad.medium.com/location-indexing-complete-guide-36a143569555

- Uber H3, "Hexagonal hierarchical geospatial indexing system" (GitHub).
  https://github.com/uber/h3
  Resolution levels 0 to 15, equidistant hexagon neighbors, polyfill of a
  polygon into cells. Built by Uber for supply and demand / surge.

- H3 documentation (resolution and cell-size tables).
  https://h3geo.org/docs/

## The geometry

- Wikipedia, "Point in polygon" (ray casting / crossing number, winding number,
  even-odd rule, complexity).
  https://en.wikipedia.org/wiki/Point_in_polygon

- Wikipedia, "Even-odd rule".
  https://en.wikipedia.org/wiki/Even%E2%80%93odd_rule

## Isochrones vs radius (why polygons, not circles)

- Radar, "What is an isochrone map?".
  https://radar.com/blog/what-is-an-isochrone-map

- Maptive, "Drive Time Polygons vs a Simple Radius Explained".
  https://www.maptive.com/drive-time-polygons-vs-a-simple-radius-explained/

## Scale figures

- Wikipedia, "Swiggy" (over 3 billion orders delivered, 680-plus cities as of
  2024, 700-plus cities as of 2025).
  https://en.wikipedia.org/wiki/Swiggy

Note: bytes.swiggy.com, doordash.engineering, medium, wikipedia, h3geo.org and
github were not directly fetchable from this run's sandbox (network egress was
restricted to search only). Facts above were drawn from these sources via web
search result summaries of their indexed content. Links are recorded here for
direct reading when a run has open egress.
