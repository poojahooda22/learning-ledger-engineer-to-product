# References: Razorpay Smart Routing (Optimizer)

Saved 2026-06-26 for the teardown on intelligent payment routing.

## Primary sources (papers)

- **Razorpay 2021, random forest smart routing.** Ramya Bygari et al, "An AI-powered
  Smart Routing Solution for Payment Systems," arXiv 2111.00783.
  https://arxiv.org/abs/2111.00783
  Key facts: static module (rules + logistic regression to predict gateway
  downtimes) filters terminals; dynamic module computes recency-decayed features
  (success rate, payment attributes, time lag) updated via a real-time feedback loop
  with an adaptive time-decay rate; a random forest classifier predicts per-terminal
  success probability. Random forest beat other models. 4-6% production success-rate
  lift across cards, UPI, net banking; routing millions of live transactions.

- **Razorpay 2023, non-stationary bandits.** "Maximizing Success Rate of Payment
  Routing using Non-stationary Bandits," arXiv 2308.01028 (AI-ML Systems 2023).
  https://arxiv.org/abs/2308.01028 | ACM: https://dl.acm.org/doi/10.1145/3639856.3639883
  Key facts: each terminal is a bandit arm with a drifting, unknown success prob.
  Best performers in their simulator: sliding-window UCB with window 200 txns, and
  discounted UCB with discount factor 0.6 (lowest cumulative regret). UCB adds an
  uncertainty bonus so under-tested terminals get exploration without random spray.
  Routing Service uses a Ray-based implementation scaling to 10,000+ TPS under PCI
  DSS. Live experiment on Dream11: 0.92% success-rate improvement vs traditional.

- **Juspay 2025, control-theoretic routing.** "A Control-Theoretic Approach to
  Dynamic Payment Routing for Success Rate Optimization," arXiv 2510.16735.
  https://arxiv.org/abs/2510.16735
  Closed-loop feedback controller; reward/penalize loop inspired by a PID controller
  maintains per-gateway health scores that converge toward live success rate.

- **PayU 2018, prior art.** "Stochastic Multi-path Routing Problem with Non-stationary
  Rewards," WWW 2018. https://dl.acm.org/doi/fullHtml/10.1145/3184558.3191630

## Product/marketing (claims, not paper-verified)

- Optimizer AI/ML routing blog: https://razorpay.com/blog/boost-payments-success-rates-with-optimizers-ai-ml-routing/
- Smart Routing explainer: https://razorpay.com/blog/learn-all-about-smart-routing-on-optimizer/
- Optimizer product page: https://razorpay.com/optimizer-intelligent-payments-routing/
- Optimizer performance report: https://razorpay.com/blog/how-do-you-know-that-your-payments-router-is-actually-creating-value-for-your-business-introducing-the-optimizer-performance-report/
- Marketing claims: ~150 parameters, 600M data points (last 6 months), 1B+ historical
  transactions, up to ~10% SR uplift, failover under 500ms, circuit breakers, 99.99%
  uptime. Treat as vendor claims, not peer-reviewed.

## The reusable pattern

Filter then rank (matching vs ranking), but the ranking signal is non-stationary so
you must forget old evidence (sliding window or geometric decay). Explore via an
uncertainty bonus, not random traffic. Keep expensive learning offline; the live
decision is a memory read of a precomputed per-terminal score plus a tiny sort over
tens of candidates. At scale the bottleneck is the hot per-terminal counter, not the
decision: shard sub-counters and stream outcomes back asynchronously.
