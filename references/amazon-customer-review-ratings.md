# References: Amazon customer reviews (star aggregate + "Top reviews" order)

Saved 2026-10-24 for the teardown `product-teardowns/2026-10-24-amazon-customer-review-ratings.md`.

## Amazon's 2015 move off the plain average (CONFIRMED)
Amazon replaced the unweighted mean star rating with a machine-learned number that weights a
review by recency, helpful votes, and verified-purchase status; the same factors also influence
review ordering. Formula and weights not published. Rolled out US first. Keeps learning.

- GeekWire: https://www.geekwire.com/2015/amazon-changes-its-influential-formula-for-calculating-product-ratings/
- The Register: https://www.theregister.com/2015/06/21/amazon_customer_reviews_machine_learning/
- InformationWeek: https://www.informationweek.com/software-services/amazon-applies-machine-learning-to-improve-reviews
- Retail Dive: https://www.retaildive.com/news/amazon-installs-new-machine-learning-reviews-system/401086

## Sorting reviews by helpfulness, done right (CONFIRMED)
Net-score and raw-fraction sorts both fail. Sort by the lower bound of the Wilson score
confidence interval (Edwin B. Wilson, 1927). Balances the observed helpful fraction against the
uncertainty from a small vote count.

- Evan Miller, "How Not To Sort By Average Rating": https://evanmiller.org/how-not-to-sort-by-average-rating.html
- Instacart wilson_score gem (continuity correction for n < 30): https://github.com/instacart/wilson_score

Wilson lower bound (z = 1.96 for 95%), p = helpful fraction, n = total votes:
( p + z^2/(2n) - z*sqrt( (p*(1-p) + z^2/(4n)) / n ) ) / ( 1 + z^2/n )

## Bayesian shrinkage for the small-sample lie (CONFIRMED example, IMDb)
WR = (v/(v+m))*R + (m/(v+m))*C
R = this item's mean, v = its vote count, m = min votes to qualify, C = global mean. Low v leans
on C, high v drifts to R. IMDb says current constants and "regular voter" criteria are secret.

- IMDb / weighted rating: https://en.wikipedia.org/wiki/IMDb

## Fake-review detection (CONFIRMED Amazon statements; graph method academic)
Amazon runs ML on all reviews 24/7, screens pre-publication, reported blocking 200M+ suspected
fakes in 2022, says 90%+ of inauthentic reviews are machine-generated. Strong public method:
bipartite product-reviewer graph, fake-buying products cluster, unsupervised clustering finds them.

- Amazon (aboutamazon.in), "Detecting fake reviews with advanced AI": https://www.aboutamazon.in/news/retail/detecting-fake-reviews-with-advanced-ai
- He, Hollenbeck, Overgoor, Proserpio, Tosyali (MSI working paper, 2022): https://www.msi.org/working-paper/detecting-fake-review-buyers-using-network-structure-direct-evidence-from-amazon/
- Storyboard18 (SECONDARY, GNN claim, treat cautiously): https://www.storyboard18.com/quantum-brief/how-amazon-is-using-ai-to-to-combat-fake-reviews-17812.htm
