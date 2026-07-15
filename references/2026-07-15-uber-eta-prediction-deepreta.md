# References: Uber ETA prediction (routing engine + DeeprETA)

Saved 2026-07-15 for the Uber ETA teardown.

## Primary sources

- Uber Engineering, "DeepETA: How Uber Predicts Arrival Times Using Deep Learning" (2022).
  https://www.uber.com/blog/deepeta-how-uber-predicts-arrival-times/
  Company post announcing the deep-learning ETA post-processing model, the
  residual-on-router idea, self-attention encoder, and Michelangelo serving.

- Hu, Wang, et al., "DeeprETA: An ETA Post-processing System at Scale" (arXiv:2206.02127, 2022).
  https://arxiv.org/abs/2206.02127  (PDF: https://arxiv.org/pdf/2206.02127)
  The academic paper. Key facts pulled: highest-QPS model at Uber; predicts the
  residual over the routing engine; bucketize all continuous features; quantile
  buckets beat equal-width (entropy argument); multiple feature hashing beats
  single hash on collisions; linear self-attention with O(Kd^2) vs O(K^2 d);
  DeeprETANet = 2 layers, linear-transformer K/Q/V dim 4, fully-connected layer
  size 2048; asymmetric Huber loss (delta = robustness, omega = under vs over
  penalty); calibration layer per request segment; latency median 3.25 ms, p95 4 ms.

- Uber Engineering, "ETA Phone Home: How Uber Engineers an Efficient Route."
  https://www.uber.com/blog/engineering-routing-engine/
  The routing-engine side: Gurafu (launched ~April 2015, replaced OSRM), road
  graph as directed weighted graph, tried A* then settled on Contraction
  Hierarchies, map partitioning into cells with boundary-node stitching, live +
  historical speeds as edge weights.

## Secondary / explainer sources

- "How Uber Computes ETA at Half a Million Requests per Second," System Design newsletter.
  https://newsletter.systemdesign.one/p/uber-eta
  Source for the ~500,000 ETA requests/second scale figure and a clean walkthrough.

- Charles Yuan, "Uber's ETA Prediction System (DeeprETA)," deMISTify / Medium.
  https://medium.com/demistify/ubers-eta-prediction-system-478026c96b95

- MarkTechPost, "Uber Explores Deep Learning To Develop DeepETA" (2022-02-15).
  https://www.marktechpost.com/2022/02/15/uber-explores-deep-learning-to-develop-deepeta-a-low-latency-deep-neural-network-architecture-for-eta-prediction/

## Notes on fact vs inference

- Facts (from Uber's own paper/blog): residual prediction, quantile bucketing,
  multiple feature hashing, linear transformer dims (2 layers / dim 4 / FC 2048),
  asymmetric Huber, 3.25 ms median / 4 ms p95, Contraction Hierarchies, Gurafu,
  cell partitioning, ~500k req/s.
- Not fully public: exact bucket counts per feature, embedding dimensions per
  field, precise MAE improvement percentages. Left unquoted in the report.
- Illustration only: specific Bengaluru corridors (Koramangala, Hebbal flyover,
  airport road) are my worked example, not Uber-published data.
