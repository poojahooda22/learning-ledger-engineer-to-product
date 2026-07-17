# References: YouTube upload transcoding pipeline (chunked encoding + Argos VCU)

Keeper links for the 2026-07-17 teardown.

## Primary source (the chip)

- Ranganathan et al., "Warehouse-scale video acceleration: co-design and
  deployment in the wild," ASPLOS 2021. The VCU/Argos paper.
  - PDF: https://gwern.net/doc/cs/hardware/2021-ranganathan.pdf
  - ACM: https://dl.acm.org/doi/abs/10.1145/3445814.3446723
  - Key facts: VCU = Video Coding Unit, internally "Argos". Videos sharded into
    short chunks, processed in parallel across many VCUs. Encoder cores do 2160p
    realtime.

## Hardware deep-dives

- ServeTheHome, "Google YouTube VCU for Warehouse-scale Video Acceleration":
  https://www.servethehome.com/google-youtube-vcu-for-warehouse-scale-video-acceleration/
  - Two Argos ASICs per full-length PCIe card, aluminum heat sink.
  - 10 encoder cores per chip; each core encodes 2160p at up to 60 FPS with three
    reference frames.
  - First gen supports VP9 and H.264. A 20-VCU machine replaces multiple racks of
    CPU-only systems for VP9.

- DataCenterDynamics, "Google develops custom 'Argos' video-transcoding chips":
  https://www.datacenterdynamics.com/en/news/google-develops-custom-argos-video-transcoding-chips-for-youtube-processing/
  - 20-33x compute efficiency vs prior CPU (Skylake) systems. AV1 on roadmap.

- 9to5Google, "Google-developed 'Argos' VCU chip":
  https://9to5google.com/2021/04/22/youtube-google-custom-chip/

- SemiAnalysis, "Google New Custom Silicon Replaces 10 Million Intel CPUs":
  https://newsletter.semianalysis.com/p/google-new-custom-silicon-replaces
  - Analyst estimate: on the order of 10M Intel CPUs replaced (range 4-33M).

## Codec strategy

- Streaming Learning Center, "Which Codecs Does YouTube Use?":
  https://streaminglearningcenter.com/codecs/which-codecs-does-youtube-use.html
  - H.264 for low-view videos, VP9 as views climb (~low thousands), AV1 reserved
    for very popular videos. AV1 ~18x the encode cost of H.264; VP9 ~2x.

## Notes on inference

- GOP-aligned chunking (split only on I-frames so each chunk is independently
  decodable/encodable), the DAG-of-tasks orchestration, and the split/encode/stitch
  MapReduce shape are the standard, well-grounded way this class of pipeline is
  built. YouTube's exact production orchestrator internals are not public; those
  parts are labeled as inference in the teardown. The chunk length (a few seconds,
  keyframe every ~2s in the worked example) is illustrative, not a published
  YouTube constant.
