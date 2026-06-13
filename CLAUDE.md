# CLAUDE.md — operating instructions for the daily teardown ledger

This repo is a daily product teardown ledger. Each run produces ONE deep product
case study and commits it here. The repo is the only durable memory. Read this
file and `_history.md` before doing anything.

## The routine, every run

1. Read `_history.md` first. It lists every product+feature pair already covered.
   NEVER repeat a pair that appears there.
2. Read `_index.md` to see the newest-first table.
3. Pick ONE specific feature from ONE real mass-market product (not the whole
   product). Rotate across the pool so no single product dominates a week:
   Amazon, Google Search, Gmail, YouTube, Netflix, Spotify, Stripe, Razorpay,
   Notion, Figma, WhatsApp, Uber, Instagram, Canva, Zepto, Swiggy, Zomato, Rapido.
4. Research with REAL primary sources (engineering blogs, GitHub repos, conference
   talks, papers). Separate fact from inference and label inference clearly.
5. Write the report to `reports/YYYY-MM-DD-product-feature.md` using the fixed
   framework below.
6. Append one line to `_history.md` (date, product, feature, one-line insight).
7. Add the new entry to the TOP of `_index.md` (newest first).
8. Save keeper links under `references/`.
9. Commit and push, then land on `master` (see "How to ship" below).

## The report framework (fixed sections)

1. The user — who they are, what they are doing in their day when they hit this.
2. The real problem — the actual pain, described like a friend would.
3. The feature in one sentence.
4. Jobs to be done — what the user is really hiring this feature to do.
5. How it works for the user — the visible experience.
6. The actual flow, step by step — tap by tap, screen by screen.
7. Under the hood, like the engineer — the heart of the report. Go deep:
   data structures and algorithms in play and why; for search/ranking explain
   candidate fetch from a catalog of millions, filtering, ranking (matching and
   ranking are two different halves), inverted indexes, where sorting happens
   (server-side, not on the phone). Scale story at three tiers: 1,000 items,
   100,000, and 10 million plus — name what breaks at the next tier and what
   they did to survive it. Cite real engineering; if internals are not public,
   say so and give the clearly-labeled inference version.
8. The retention and habit mechanic — what loop brings the user back; which
   metric it moves (activation, retention, or revenue); a real observed example.
9. The lesson for Rare.lab — one concrete, specific, actionable lesson. Rare.lab
   is an AI shader and visual-effects product: a node-based editor that compiles
   to shippable code, plus an embeddable runtime. Bias toward scalability and
   performance.

End with a Sources list (real links). Save keepers under `references/`.

## Hard format rules

- No em dashes. No en dashes. Anywhere.
- No jargon walls. Short sentences. Concrete numbers and real examples over abstract description.
- Every section carries at least one concrete real-world example (a real watch
  listing, a real song, a real ride, a real query walked end to end).
- Thorough but readable. Depth over padding.
- Everything stays in the repo. No Gmail, no email.

## How to ship (this is settled, do it the same way each run)

The GitHub app now has write access. The ledger lives on `master`.

1. Set identity once: `git config user.email noreply@anthropic.com && git config user.name Claude`
   (avoids the "Unverified" commit warning; amend the tip with
   `git commit --amend --no-edit --reset-author` if the hook flags it).
2. Commit the new report + updated `_history.md` + `_index.md` + any references.
   Commit message names the product and feature, e.g. "Teardown: Amazon search ranking".
3. Push the working branch: `git push -u origin <branch>`.
4. Land it on `master`: open a PR (base `master`), then squash-merge it via the
   GitHub MCP tools (`mcp__github__create_pull_request`, `mcp__github__merge_pull_request`).
   The user wants everything on `master`. Do the PR and merge yourself; do not
   wait for the user to merge.

## Notes from earlier runs

- 2026-06-13: First run. Created repo structure and the Spotify Discover Weekly
  teardown. Direct push to `master` is not allowed from the session; the working
  pattern that succeeded was: commit on the feature branch, push, open a PR into
  `master`, squash-merge. The git proxy 403'd until the GitHub app was connected
  with write access, then push worked.
