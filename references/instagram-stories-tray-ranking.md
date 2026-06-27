# References: Instagram Stories tray ranking and 24-hour expiry

Saved 2026-06-27 for the Instagram Stories teardown.

## Primary sources (Instagram / Meta engineering)

- **Lessons Learned at Instagram Stories and Feed Machine Learning** (Thomas Bredillet,
  Instagram Engineering, 2018). The value-function design: combine ranking-loss models and
  point-wise models for fine control over engagement trade-offs. Position-bias correction by
  adding a sparse position feature to the last fully connected layer (so the top of the tray
  does not freeze). Caffe2 as the modeling framework. Co-learned sparse embeddings for
  interests.
  https://instagram-engineering.com/lessons-learned-at-instagram-stories-and-feed-machine-learning-54f3aaa09e56

- **Journey to 1000 models: Scaling Instagram's recommendation system** (Engineering at Meta,
  May 2025). Names the Stories tray model `ig_stories_tray_mtml` (multi-task, multi-label).
  The operational challenge of running 1000+ ranking models: training flows, checkpoints,
  inference services.
  https://engineering.fb.com/2025/05/21/production-engineering/journey-to-1000-models-scaling-instagrams-recommendation-system/

- **Instagram Stories AI system** (Meta Transparency Center). The named predictions used to
  order the tray: likelihood to tap into a story, to reply, to move on. Concrete input signals:
  average story-viewing time, device platform, number of times you have viewed this account's
  stories, total time spent on the author's stories, connection strength (Facebook friend /
  friend-of-friend), reply-to-view ratio, first interaction type.
  https://transparency.meta.com/features/explaining-ranking/ig-stories/

- **Sharding & IDs at Instagram** (Instagram Engineering). The 64-bit packed id: 41 bits ms
  timestamp (custom epoch), 13 bits logical shard, 10 bits per-shard sequence (mod 1024, so
  1024 ids per shard per ms). Timestamp in the high bits means ORDER BY id equals
  ORDER BY created_at, and the creation time (hence the 24h deadline) is readable straight from
  the id with no extra column or index. This is the backbone of the lazy-expiry argument.
  https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c

- **Scaling the Instagram Explore recommendations system** (Engineering at Meta, Aug 2023).
  Meta's two-stage retrieval-then-ranking pattern (two-tower retrieval, ANN, lightweight
  first-pass ranker, heavy second-pass ranker, multi-task heads). Context for how the tray
  model fits the broader recommendation stack.
  https://engineering.fb.com/2023/08/09/ml-applications/scaling-instagram-explore-recommendations-system/

- **Instagram Ranking Explained** (Adam Mosseri, About Instagram). Stories top signals stated
  plainly as likelihood to tap, like, reply.
  https://about.instagram.com/blog/announcements/instagram-ranking-explained

## What is fact vs inference in the report

- **Fact:** the named predictions (tap/reply/move-on), the input signals, the
  `ig_stories_tray_mtml` model name, the value-function + point-wise design, the position-bias
  sparse feature, Caffe2, and the full 41-13-10 bit id layout. All from the sources above.
- **Inference (labeled in the report):** that Stories specifically rides on TAO and on
  Haystack/f4 blob storage (these are published Meta systems but not documented as the Stories
  store); that the tray uses fan-out-on-read with a push/pull hybrid; the exact value-function
  weights; and that expiry is lazy-on-read with async physical reclamation. These are the
  standard solutions for this class of problem, grounded in Meta's published feed/storage work,
  but not Stories-specific published facts.
