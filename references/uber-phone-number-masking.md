# References: Uber phone number masking (anonymized calling + VoIP)

Saved 2026-10-17 for the Uber phone-number-masking teardown.

## Confirmed, Uber (consumer newsroom / blog)
- Uber Newsroom India, "Keep your number private with phone anonymisation": https://www.uber.com/en-IN/newsroom/keep-your-number-private-with-phone-anonymisation
  - Masks BOTH driver and rider numbers; calls connected by software as a safety precaution.
- Uber blog, phone anonymisation: https://www.uber.com/blog/phone-anonymisation
- Uber Newsroom Bangladesh, "Uber VoIP call" (Oct 2019): https://www.uber.com/bd/en/newsroom/uber-voip-call/
  - Two options when calling: regular voice call, or free "Uber Call" = VoIP over the internet inside the app.

## Vendor-reported scale (treat specific numbers as vendor claims, not Uber-confirmed)
- Exotel customer story, Uber: https://exotel.com/case-study/uber/
  - Powers Uber India since 2016, ~2 million daily calls; 200K-300K calls/hour at peak; temporary proxy numbers hide both parties; vendor reports connectivity rising into the ~70% range.
- Infobip customer story, Uber: https://www.infobip.com/customer/uber
  - Built "Number Masking" for Uber; ~90% global call anonymization target.

## Industry design patterns (architecture + the pool math)
- Plivo docs, virtual number allocation: https://www.plivo.com/docs/number-masking/concepts/virtual-number-allocation
  - Pool sizing formula: Virtual Number Count = 2 x (concurrent sessions) - 1, using the higher-concurrency party. Example: 10 concurrent orders -> 19 numbers.
- Yelp Engineering, "Phone Number Masking for Yelp Services Projects" (Mar 2024): https://engineeringblog.yelp.com/2024/03/phone-number-masking-for-yelp-services-projects.html
  - Recycle/reuse proxy numbers; short-lived sessions -> small pool + aggressive recycling policy.
- Exotel docs, number masking setup: https://developer.exotel.com/docs/advanced-config/number-masking-setup
  - Pool strategies: dedicated (low volume), shared (high volume / short engagements), regional (region-local caller ID). "Busy" state when a number is in use. Allocation lifecycle: create, active, update, deactivate.
- Arpit Bhayani, "Designing a Scalable Phone Number Masking System for Hyperlocal Apps": https://arpitbhayani.me/videos/designing-scalable-phone-number-masking-system/
  - 50M accounts would need 50M numbers if static; components: Order Service, event bus (Kafka), Virtual Number Service, telecom partners; numbers bound to the life of an active order and returned to the pool at end.
- Twilio glossary, "What are masked phone numbers": https://www.twilio.com/docs/glossary/what-are-masked-phone-numbers
  - Common pattern: parties communicate through a third proxy number that forwards the call to the real destination.

## Key takeaways reused in the teardown
- Two data structures: a free-list POOL of virtual numbers + an O(1) hash-map MAPPING (proxy number -> {trip, rider, driver, expires_at}), direction disambiguated by the calling party's real number.
- Hard half is RELEASE: grace window after trip end, then cooldown/quarantine before reuse, or a late callback reaches the wrong fresh pairing.
- Scale wall is the FINITE, regulated, rented number inventory -> size pool to concurrency, recycle hard, shard by region, go multi-provider per country.
- VoIP "Uber Call" is the escape hatch: leases ZERO numbers (voice over data through Uber servers), the cheap default/fallback that spares the scarce PSTN pool.
