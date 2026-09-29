# References: Canva design export (2026-09-29)

Kept for the Canva design export teardown.

## Primary and near primary

- LottieFiles case study, Canva x ThorVG (the strongest engineering fact):
  - https://lottiefiles.com/case-studies/canva
  - https://lottiefiles.com/blog/working-with-lottie-animations/canva-enhances-ios-rendering-faster-and-efficient-with-thorvg
  - Key facts: Canva used rLottie (C++ Lottie renderer) on iOS across video export and
    playback. After adding user imported Lottie, load rose. Benchmarked rLottie vs ThorVG
    in the iOS video export pipeline on a 1 minute all Lottie video: ~80% faster render,
    ~70% lower peak memory, fewer load/render errors. Canva switched to ThorVG.

- ThorVG (open source C++ vector graphics engine, SVG + Lottie):
  - https://www.thorvg.org
  - https://docs.lottiefiles.com/en/runtimes/overview/thorvg

- Canva Connect API, Exports (the documented async job model):
  - Create design export job (returns job id + status in_progress; becomes success or failed):
    https://www.canva.dev/docs/connect/api-reference/exports/create-design-export-job/
  - Get design export job (poll until success/failed; per page download URLs sorted by page
    order; URLs expire after 24 hours; premium unpurchased elements are a documented failure):
    https://www.canva.dev/docs/connect/api-reference/exports/get-design-export-job/
  - Exports overview: https://www.canva.dev/docs/connect/api-reference/exports/

- Canva Apps SDK, Exporting designs (formats PNG/JPG/PDF/GIF/SVG/video/PPTX; multi page in a
  single page format returns a ZIP or per page files; recommends downloading via the app
  backend; export URLs short lived): https://www.canva.dev/docs/apps/exporting-designs/

## Scale numbers

- 30 billion designs created by Dec 2024; ~12 billion in 2025; ~38.5 million designs/day;
  220M+ MAU (2024) rising to 260M+/265M (2025); 31M+ paid users; ~US$4B ARR.
  - https://backlinko.com/canva-users
  - https://en.wikipedia.org/wiki/Canva

## Labeling note

Canva has not published a full export/render architecture diagram. The queue plus
specialized worker fleet, pre warming for peak, and content hash dedupe are labeled
inference in the report: standard patterns for heavy render workloads, consistent with
Canva's visible async job API, not confirmed internals.
