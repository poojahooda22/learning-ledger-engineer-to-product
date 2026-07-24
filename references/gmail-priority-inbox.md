# References: Gmail Priority Inbox (importance ranking)

Saved 2026-07-24 for the Gmail Priority Inbox teardown.

## Primary (Google paper, the core source)
- Douglas Aberdeen, Ondrej Pacovsky, Andrew Slater. "The Learning Behind Gmail Priority Inbox." Google, 2010. The definitive source: behavioral label (importance = probability the user acts on a mail within a time window of delivery, given client activity), plain linear logistic regression, a global model plus a thin per-user additive model combined as a sum of log-odds (global weights frozen during the personal update), online Passive-Aggressive PA-II regression updates to tolerate a noisy label, a per-user threshold treated as ranking-not-classification and tuned in near real time, ~80% (+/- 5) accuracy on a control group, false-negative rate ~3-4x false-positive rate, and ~6% less time reading mail overall / ~13% less on unimportant mail.
  https://research.google/pubs/pub36955/
  PDF: https://research.google.com/pubs/archive/36955.pdf
- Semantic Scholar entry (metadata, citations).
  https://www.semanticscholar.org/paper/The-Learning-Behind-Gmail-Priority-Inbox-Aberdeen-Pacovsky/5882cc9ac2380f13016c5aa6c81422de9e47b311

## Product (how it behaves for the user)
- Google Workspace Help, "Tips to optimize your Gmail inbox" (Priority Inbox sections, the yellow importance marker, the mark-important / mark-not-important buttons).
  https://support.google.com/a/users/answer/9282734

## Launch-era press (the headline numbers, secondary)
- TechCrunch (2010), "Priority Inbox Is Working; Users Spending 15 Percent Less Time Reading Email" (press quoted ~15%; the paper's controlled numbers are 6% overall / 13% unimportant).
  https://techcrunch.com/2010/12/06/gmail-priority-inbox-stats/
- Fast Company (2010), "Google's Priority Inbox Will Save 13% of Your Time."
  https://www.fastcompany.com/1714786/googles-priority-inbox-will-save-13-your-time
- Softpedia, "Gmail's Priority Inbox Explained in Google Research Paper."
  https://news.softpedia.com/news/Gmail-s-Priority-Inbox-Explained-in-Google-Research-Paper-176954.shtml

## Key facts to reuse
- Importance is defined behaviorally, not by committee: did the user act on the mail soon after delivery, and only counted when the user was active (no click while away is not a real negative). This makes importance a predictable probability.
- Two models added, not one: score = logistic(sum of feature * (global_weight + personal_weight)). Personal weights encode only the DEVIATION from the average user, so per-user storage is a small sparse vector, which is what makes a model-per-user affordable at hundreds of millions of accounts.
- Learning is online per message (PA-II), because you can never retrain from scratch at Gmail scale and the label is noisy. Each mail updates the global model once and the recipient's model once. PA-II makes the smallest weight change that fixes the current example, damped by regularization, so one mislabeled mail cannot yank the model.
- Two timescales on purpose: slow careful weight learning, plus a cheap fast per-user THRESHOLD that jumps in near real time when the user marks a few in a consistent direction. The threshold is what the user feels; the weights are what is actually accurate.
- Asymmetric threshold: false negatives ~3-4x false positives, because a wrong yellow marker on junk destroys trust in the "Important" section, whereas a missed-important mail still sits safely in "Everything else." Protect the top section like a bouncer.
- Cold start from the global model gives ~80% accuracy on day one; personalization then sharpens it and becomes the switching-cost moat (your trained model exists nowhere else). Same defensive/data-gravity retention as spam filtering.
- Feature families: social (sender-recipient interaction history, e.g., % of a sender's mail this user opens/replies to), content (recently predictive header/subject terms), thread (has the user engaged this thread), label (filters/categorization applied).
- This is ranking, not matching: the candidate set is just your own inbox (already fetched), so nearly all the engineering is the ranking half. Cheap linear + sparse features is a feature, not a compromise: scoring is a short dot product, updating is a short vector add, no GPU on the mail path.
- Scale: shard by user id (a user's mail, personal model, and threshold are self-contained). The one shared object is the global model, the classic hot-key; the grounded-inference fix is sampled/aggregated global updates while personal models carry the fast per-user adaptation.
- Offline-think / online-lookup: scoring runs server-side as mail arrives; the phone renders sections and markers and does zero ranking.
