# References: Stripe API versioning

Saved 2026-10-26 for the Stripe API versioning teardown.

## Primary (Stripe's own)

- APIs as infrastructure: future-proofing Stripe with versioning (the core design: version change modules, master list, date versions, account pinning, "compatible with every version since 2011").
  https://stripe.com/blog/api-versioning
- How API versioning works (pinning, Stripe-Version header, major vs monthly releases, codenames Acacia/Basil/Dahlia/Endive).
  https://docs.stripe.com/api-versions
- API reference, Versioning (per-language SDK pinning, per-request override).
  https://docs.stripe.com/api/versioning
- Upgrade your integration (72 hour rollback window, how upgrades are applied).
  https://docs.stripe.com/upgrades

## Real changelog entries used as walked examples

- 2014-09-08: Bank Accounts include a status enum that replaces the disabled, validated, and verified properties.
  https://docs.stripe.com/changelog/2014-09-08/bank-accounts-include-status-enum
- 2015-07-13: Accounts replace the verification[contacted] boolean with a verification[disabled_reason] string.
  https://docs.stripe.com/changelog/2015-07-13/accounts-include-verification-disabled-reason-field

## Secondary / commentary

- Brandur Leach (ex-Stripe), Why Doesn't Stripe Automatically Upgrade API Versions? (pinning model, "tens of new versions per year", a version is cut only for a backwards-incompatible change).
  https://brandur.org/api-upgrades
- Hacker News thread with a Stripe engineer discussing the versioning tradeoffs.
  https://news.ycombinator.com/item?id=13708927

## Key facts to reuse

- Versions are dated (2017-05-24), modern ones add a codename suffix (2024-09-30.acacia).
- Each version "contains a small set of changes that make incremental upgrades relatively easy."
- A version change module bundles: documentation + transformation + the resource types it may modify. All modules live in one ordered master list.
- Response pipeline: build at latest, then walk back through time applying matching modules until the target version.
- Target version resolution order: request Stripe-Version header, then SDK-pinned version, then account default (set on first API request). Webhook endpoints can pin their own version.
- Since 2024-09-30.acacia: monthly releases are non-breaking; breaking changes batched into major releases about twice a year.
- Inference, not confirmed internals: "gates" as per-version feature flags for pure behavior changes; exact value mappings inside any single transformation.
