# References: Zepto dark-store inventory and order routing (2026-06-17)

Keeper links for the quick-commerce inventory + geospatial-routing teardown.

## Geospatial / nearest-store / nearest-rider
- Tile38, real-time geospatial and geofencing engine (in-memory, Redis RESP,
  NEARBY/WITHIN/INTERSECTS, points/bboxes/geohashes/GeoJSON, WHERE filters,
  leader/follower replication): https://github.com/tidwall/tile38

## Atomic inventory / oversell prevention
- Redis official tutorial: real-time inventory reservation with WATCH/MULTI
  and audit streams: https://redis.io/tutorials/inventory-reservation-in-real-time-with-redis/
- Redis INCR/DECR atomic counters explained:
  https://oneuptime.com/blog/post/2026-03-31-redis-incr-decr-atomic-counters/view
- Flash-sale system that never oversells (Lua atomic check-and-decrement,
  inventory sharding into N sub-counters, token-list LPOP, ~100k ops/sec):
  https://medium.com/@umesh382.kushwaha/designing-a-flash-sale-system-that-never-oversells-from-1-user-to-1-million-users-without-8426db0f1ad0
- Flash sale system design (architecture, scale, oversell):
  https://singhajit.com/flash-sale-system-design/
- Shopify Engineering: replaced Redis with MySQL for inventory reservations
  using SKIP LOCKED + composite keys (2026):
  https://shopify.engineering/scaling-inventory-reservations

## Zepto / quick-commerce operations (reported, not Zepto eng blog)
- Zepto delivery, algorithmic brilliance behind 10-minute deliveries
  (reported Tile38 use; ARIMA/Prophet/LSTM forecasting; picking benchmarks):
  https://medium.com/@aryanakm01/zepto-delivery-the-algorithmic-brilliance-behind-10-minute-deliveries-daea75f0604f
- Deconstructed: Zepto's 10-minute model (store 2,000-4,400 sq ft, within
  2-3 km, ~76s picking, median ~8m47s, Min-Max replenishment):
  https://www.42signals.com/blog/zepto-business-model-explained/
- Dark store management in quick commerce (real-time stock sync, 99%+
  accuracy, oversell challenge):
  https://kissflow.com/solutions/retail/dark-store-management-in-quick-commerce/

## Note on fact vs inference
Zepto has not published a granular engineering blog. Store sizes, picking
times, and median delivery are reported figures. Tile38 attribution comes
from secondary write-ups, not Zepto. The atomic-decrement, sharding, and
match-then-rank patterns are industry-standard and well sourced above;
applied to Zepto they are grounded inference, labeled as such in the report.
</content>
