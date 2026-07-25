# References: Amazon delivery date promise (Get it by / PDD / countdown)

Saved 2026-07-25 for the teardown on Amazon's delivery date promise engine.

## Primary / Amazon research
- Amazon Science, SPEEDY: Framework for sharpening promise time estimates in sub-same day delivery. Multiple heterogeneous views of order-to-delivery data + deep view interaction network; narrower slots for 60%+ of eligible orders. https://www.amazon.science/publications/speedy-framework-for-sharpening-promise-time-estimates-in-sub-same-day-delivery
- Amazon Science, Improving forecasting by learning quantile functions. Learn the whole quantile function at once (spline QF); example of lowering a 24h promise from 95% to 94% to unlock inventory reduction. https://www.amazon.science/blog/improving-forecasting-by-learning-quantile-functions
- US Patent 11,334,845, System and method for generating notification of an order delivery (delivery promise engine + optimizer; QR/QRF/GBT/XGBoost/NN/ensemble named). https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/11334845
- AWS Blog, How to Predict Shipments' Time of Delivery with Cloud-based ML Models. zip-to-zip transit modeling shape. https://aws.amazon.com/blogs/industries/how-to-predict-shipments-time-of-delivery-with-cloud-based-machine-learning-models/

## Academic
- INFORMS M&SOM, Real-Time Delivery Time Forecasting and Promising in Online Retailing: When Will Your Package Arrive? https://pubsonline.informs.org/doi/10.1287/msom.2022.1081
- Quantile Regression for Delivery Promise Optimization (asymmetric loss in split-point selection for quantile regression forests). https://www.academia.edu/124686682/QUANTILE_REGRESSION_FOR_DELIVERY_PROMISE_OPTIMIZATION

## Product behavior / context
- Amazon Customer Service, Order with Prime FREE Same-Day Delivery (countdown timer, cutoff, slot can become unavailable). https://www.amazon.com/gp/help/customer/display.html?nodeId=GFUT24ALVAVC6VFD
- Veeqo, Delivery Day Prediction on Amazon (PDD sets the "Get it by" date and is measured against seller On-Time Delivery Rate). https://www.veeqo.com/blog/delivery-day-prediction-on-amazon-what-it-is-how-it-works-and-why-it-matters

## Scale figures (2024-2025 estimates)
- Capital One Shopping, Amazon Logistics Statistics 2026: ~20-25M packages/day globally; 350+ US fulfillment centers; 1,000+ logistics facilities; 750k+ robots; 9B+ items same/next-day in 2024. https://capitaloneshopping.com/research/amazon-logistics-statistics/

## Key facts vs inference (for future runs)
- FACT: PDD is the measured commitment (seller OTDR); zip-to-zip transit models; quantile methods (QR/QRF/GBT/XGBoost/NN/ensemble); SPEEDY 60%+ narrower slots; learn-whole-quantile-function; countdown accounts for capacity/inventory draining.
- INFERENCE (labeled in report): the live render is a precomputed keyed lookup with a light capacity adjustment; match(candidate lanes via item-node and zip-node indexes) then rank(pick soonest date at chosen quantile); hot same-day slot handled by atomic reserve + sub-counter split (Zepto/Razorpay pattern).
