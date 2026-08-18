# References: Canva Magic Resize

Keeper links from the 2026-08-18 teardown.

## Canva engineering (primary)
- How we see groups in design (design = collection of positioned elements; meaning from visual composition): https://www.canva.dev/blog/engineering/how-we-see-groups-in-design/
- CSS: Absolutely positioning things relatively (responsive rendering via relative/percentage placement, absolute-relative to each other): https://www.canva.dev/blog/engineering/css-absolutely-positioning-things-relatively/
- Adding responsiveness to Canva's Design System: https://www.canva.dev/blog/engineering/adding-responsiveness-to-canvas-design-system/
- 5 Visual Effects Canva Uses to Thrill Users (GPU compositing on the canvas): https://medium.com/canvatech/5-visual-effects-canva-uses-to-thrill-users-ebbdf18c51ce
- Subscription analytics at scale: https://www.canva.dev/blog/engineering/subscription-analytics-at-scale/

## Canva product / SDK / docs (the data model, primary)
- Apps SDK, Positioning elements (top/left/width/height/rotation; width or height "auto" preserves aspect ratio; default position = center): https://www.canva.dev/docs/apps/positioning-elements/
- Apps SDK, Elements (an element is a container of content with dimensions and position): https://www.canva.dev/docs/apps/elements/
- Help Center, Resize designs and size limits (custom 40 to 8000 px per side, max area 25,000,000 px squared, e.g. 5000x5000): https://www.canva.com/help/resize/
- Canva Pro, Magic Resize product page: https://www.canva.com/pro/magic-resize/

## History (paid-tier gating)
- Canva Newsroom, Canva for Work launches to 4 million users (Aug 2015): https://www.canva.com/newsroom/news/canva-for-work-launches-to-4-million-users/
- TechCrunch, Now With 4 Million Users, Canva Launches To Businesses (Aug 10, 2015): https://techcrunch.com/2015/08/10/now-with-4-million-users-design-platform-canva-launches-to-businesses/

## Scale numbers
- Expanded Ramblings, Canva Statistics (2026): 220M MAU 2024, ~260M 2025, ~30B designs, ~420-445 designs/sec, 26M paid subs, $2.65B revenue: https://expandedramblings.com/index.php/canva-statistics-and-facts/
- Skillademia, Canva Statistics 2026: https://www.skillademia.com/statistics/canva-statistics/

## Key takeaways
- The design is structured data (a scene-graph tree of positioned elements), not a rasterized image. That is the entire precondition for Magic Resize.
- Resize = coordinate-system change from (Wo,Ho) to (Wn,Hn) over the tree. sx=Wn/Wo, sy=Hn/Ho.
- Do NOT multiply size by (sx,sy) independently (two-axis stretch = funhouse mirror). Scale each element by ONE uniform factor to preserve aspect ratio; reposition using anchors (edges/center/percentages); then a local overflow/spacing constraint pass. (Exact algorithm not published by Canva; this is labeled inference grounded in their positioning writing + observed behavior.)
- Layout computed server-side: fonts/assets live there, consistency + caching + fan-out batching matter. Client only renders the returned tree.
- Two scale axes: elements-per-design (O(n^2) collision pass breaks at ~100k, fix with quadtree/grid bucketing + viewport virtualization; page area capped at 25M px^2) vs the fleet (shard document store, CDN assets, queue the 9x resize fan-out and reuse fetched assets).
- Business: value-gated paywall placed at peak motivation (publish everywhere). Moves revenue. Paid since 2015.
