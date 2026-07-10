# References: Gmail spam filtering and RETVec (2026-07-10 teardown)

## Primary sources (Google)

- Google Security Blog, "Improving Text Classification Resilience and Efficiency with RETVec" (Nov 2023).
  https://security.googleblog.com/2023/11/improving-text-classification.html
  Key facts: RETVec is vocabulary-free and multilingual (100+ languages), resilient to insertion/deletion/typos/homoglyphs/LEET. In the Gmail spam classifier it improved spam detection rate by 38% over baseline, cut false positive rate by 19.4%, and cut TPU usage by 83%. Open-sourced; details in the NeurIPS 2023 paper.

- RETVec paper, arXiv 2302.09207, "RETVec: Resilient and Efficient Text Vectorizer" (NeurIPS 2023).
  https://arxiv.org/abs/2302.09207
  Architecture: character encoder = Integerizer (chars -> UTF-8 codepoints) + Binarizer (codepoint -> compact 24-bit little-endian binary). Max 16 characters per word. A small embedding model (~200k parameters) projects to a 256-dimensional metric embedding. Pretrained via pairwise metric learning on a typo-augmented 157-language dataset. Binary representation is easier to learn and far more compact than one-hot.

- google-research/retvec (open-source repo).
  https://github.com/google-research/retvec

- Google Workspace Blog, "Ridding Gmail of 100 million more spam messages with TensorFlow" (Feb 2019).
  https://workspace.google.com/blog/product-announcements/ridding-gmail-of-100-million-more-spam-messages-with-tensorflow
  Key facts: Gmail blocks >99.9% of spam/phishing/malware with all protections combined; the TensorFlow model blocks ~100M additional messages per day, the <0.1% tail. Catches image-based messages, emails with hidden embedded content, and mail from newly created domains hiding low volume in legitimate traffic. Uses TensorBoard to monitor training and evaluate models quickly.

## Supporting / context sources

- Spamhaus, "DNS Blocklist Basics" (DNSBL as "first line of defense" against spam).
  https://www.spamhaus.org/resource-hub/email-security/dns-blocklist-basics/

- Paul Graham, "A Plan for Spam" (2002), the naive-Bayes baseline that made statistical filtering standard.
  https://www.paulgraham.com/spam.html

- ScienceDirect / Heliyon, "Machine learning for email spam filtering: review, approaches and open research problems."
  https://www.sciencedirect.com/science/article/pii/S2405844018353404

- Statista, spam share of global email traffic (~47-52%).
  https://www.statista.com/statistics/420391/spam-email-traffic-share/

- Gmail usage statistics (~1.8B users, ~121B emails/day).
  https://www.demandsage.com/gmail-statistics/

## The one-line insight

Spam filtering is classification under an adversary, not match-then-rank. Order the pipeline cheapest-filter-first (connection-time reputation kills the bulk before a body is parsed) so the expensive ML model runs on almost nothing. RETVec's win is that the input representation is a performance decision: a compact 24-bit binary char encoding plus a 200k-param model got MORE accurate AND 5x cheaper (83% less TPU) at 121B messages/day, and by dropping the vocabulary it made spammer text obfuscation (paypa1, homoglyph Apple) stop working.
