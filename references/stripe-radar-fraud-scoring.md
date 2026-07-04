# References: Stripe Radar real-time fraud scoring

Keeper links for the 2026-07-04 teardown.

## Primary (Stripe engineering / product)

- **How we built it: Stripe Radar** (Stripe.dev engineering blog). The main source:
  Wide-and-Deep (XGBoost + DNN) migrating to a pure DNN in mid-2022, the
  ResNeXt-inspired multi-branch architecture, 10x training data, why XGBoost was
  dropped (not parallelizable, slow retrain, incompatible with newer techniques).
  https://stripe.dev/blog/how-we-built-it-stripe-radar
- **A primer on machine learning for fraud detection** (Stripe guide). Feature
  engineering as two jobs (formulation + production availability), why every feature
  must be computable live, score calibration, precision-recall, class imbalance.
  https://stripe.com/guides/primer-on-machine-learning-for-fraud-protection
- **Stripe Radar product page.** Scale figures ($1.9T/yr, millions of businesses,
  thousands of partner banks), ~100ms budget, fraud-reduction and false-positive
  claims, the three components (rules engine, manual review, ML scoring).
  https://stripe.com/radar
- **Updates to Stripe's advanced fraud detection** (Stripe blog).
  https://stripe.com/blog/advanced-fraud-detection-updates
- **Risk evaluations** (Stripe docs). The 0-99 risk score and how blocks/reviews work.
  https://docs.stripe.com/radar/risk-evaluation

## Counterfactual evaluation (the censored-labels problem)

- **Counterfactual Evaluation of Machine Learning Models** (InfoQ / QCon, Stripe talk
  by Michael Manapat). Why you must estimate outcomes for payments you blocked to get
  an honest precision-recall curve. Same off-policy problem as a contextual bandit.
  https://www.infoq.com/presentations/stripe-ml-models-fraud/

## 2025 Payments Foundation Model

- **Stripe unveils AI foundation model for payments, deeper Nvidia partnership**
  (TechCrunch, May 2025). Transformer, self-supervised on tens of billions of
  transactions, behavioral embedding vector; card-testing detection 59% -> 97% with
  no false-positive increase.
  https://techcrunch.com/2025/05/07/stripe-unveils-ai-foundation-model-for-payments-reveals-deeper-partnership-with-nvidia/

## Secondary deep-dives

- **How Stripe Detects Fraudulent Transactions Within 100 ms** (ByteByteGo). Latency
  budget breakdown, feature store, live scoring flow.
  https://blog.bytebytego.com/p/how-stripe-detects-fraudulent-transactions

## Key facts to remember

- Radar = three components: rules engine, manual review, ML scoring of every payment.
- ~100ms total budget in the authorization path; DNN forward pass is single-digit ms;
  heavy features are precomputed running aggregates read O(1) from a feature store.
- High-cardinality entities (card, IP, email, device) -> learned embeddings, not
  one-hot, so never-before-seen values generalize via nearby known behavior.
- Model history: Wide-and-Deep (XGBoost + DNN) -> pure ResNeXt-style multi-branch DNN
  (mid-2022).
- Labels come from disputes/chargebacks, 30-90 days late; blocked payments are
  censored (no outcome), which forces counterfactual evaluation.
- Block threshold is the core product dial; must be validated per business segment.
- Data network effect across $1.9T/yr is the moat; 2025 foundation model pushed
  card-testing detection 59% -> 97%.
