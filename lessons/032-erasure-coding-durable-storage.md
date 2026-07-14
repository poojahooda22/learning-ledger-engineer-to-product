# Day 32 — How do you store 65 petabytes of photos without paying for three separate hard drives per photo?

*2026-07-14*

---

## 1. The company and the number that breaks a naive design

Facebook's **f4**, the "warm" BLOB storage system described in Muralidhar et al.'s 2014 OSDI paper, was built to hold photos and videos once they stop being hot. At the time of the paper it already stored **more than 65 petabytes** of logical (non-redundant) data, on top of an even larger hot tier called Haystack, and the corpus was growing continuously as uploads never stop and nothing gets deleted.

Haystack, the hot tier that serves freshly-uploaded, frequently-viewed photos, keeps **three full replicas** of every BLOB, and once you account for the RAID-6 padding Haystack machines use underneath each replica, the true physical-to-logical ratio is closer to **3.6x**. That number is fine for hot data: you need three live copies anyway so any of them can answer a read instantly. The number stops being fine the moment you apply it to data nobody is actively reading. Facebook's own access logs showed that the overwhelming majority of photos go "warm," their read rate drops to a trickle, within days of being posted. Multiply 3.6x by 65+ petabytes and you are paying for roughly **234 petabytes of physical disk** to store 65 petabytes of content that mostly just sits there. And that ratio does not shrink as the corpus grows: it is a permanent 3.6x tax on every future petabyte too, forever, on data that will almost never be read again after this week.

That is the breaking number: not a latency figure, not a request rate, a **cost curve that scales linearly and unboundedly with total bytes stored**, applied uniformly to data whose actual access pattern no longer justifies uniform treatment. A single-server demo never hits this wall, because a demo never stores enough data, for long enough, for the gap between "how this data is accessed" and "how this data is protected" to show up on an invoice.

---

## 2. Why the naive design dies

The naive design is: replicate every object N times (commonly 3, "the number that feels safe"), keep every replica on a fast disk, and never revisit that decision once the object stops being popular. It fails in three concrete ways.

**a. The cost is a straight line with no ceiling.** Every additional petabyte of uploads costs 3.6 additional petabytes of physical disk, forever, whether or not anyone will ever look at it again. There is no point at which the system gets cheaper per byte as it gets bigger; replication factor does not amortize. This is like renting three separate warehouses and putting one identical copy of every box you own in each of them, including boxes you packed away two years ago and have not opened since, "just in case" one warehouse burns down.

**b. Hot-tier infrastructure is the wrong shape for cold-tier access.** Haystack's three-full-copy design exists so that *any* of the three machines can answer a read with zero extra work, which matters when a photo is being hit thousands of times an hour right after posting. Once a photo's read rate drops to a handful of times a year, that architecture is solving a problem the data no longer has, while still charging the data full price for the privilege.

**c. Just cutting the replica count is not a fix, it is a different failure.** The obvious naive "optimization" is to drop from 3 replicas to 2. That does reduce cost, and it also means a single disk or rack failure leaves the data with **zero remaining redundancy**: one more failure, on the remaining copy, and the object is gone permanently. You cannot buy back cost by simply spending less on durability when durability was the entire point of replicating in the first place. The naive design has exactly one lever, "how many full copies," and every position of that lever is either too expensive or too fragile.

---

## 3. The architecture, top to bottom

```
Upload clients (photo / video upload)
   |
   v
Haystack — hot tier, full triple replication + RAID-6 padding (effective RF ~3.6x)
   |  any of 3 machines answers instantly: keep every morning's fresh newspapers on the
   |  front counter, all three stacks, everywhere, so a walk-in customer is served with zero delay
   v
Temperature tracker / BLOB age-out job
   |  watches per-object read rate; once a BLOB's reads drop below a threshold, it is
   |  "warm" and gets migrated out of the hot tier instead of staying there by default
   v
f4 router / coordinator tier
   |  stateless lookup: given a BLOB id, which encoded volume and which block holds it,
   |  a phone-book lookup, not a search
   v
Reed-Solomon(10,4) encoder — single-datacenter erasure coding
   |  splits each volume's data into 10 blocks, computes 4 parity blocks from them,
   |  spreads all 14 blocks across different racks/failure domains, expansion factor 1.4x,
   |  survives losing ANY 4 of the 14 blocks
   |  like sending a 10-page fax plus 4 extra "checksum pages" that let the recipient
   |  reconstruct any 4 pages that got lost in transmission, without resending anything
   v
Cross-datacenter XOR layer
   |  XORs each block against the matching block of a paired copy in a second datacenter,
   |  so losing an entire datacenter is still recoverable from the other site;
   |  pushes total effective replication factor to about 2.1x, still far under 3.6x,
   |  while surviving a failure Reed-Solomon alone (confined to one site) cannot
   v
Background rebuilder / scrubber
   |  continuously scans for missing or corrupted blocks (bad disk, dead rack, silent
   |  bit rot), reconstructs them from the surviving blocks, writes the replacement,
   |  racing to restore the safety margin before a second failure lands
   v
Read path
   |  normal read: fetch the ONE block that directly holds the requested BLOB, done
   |  degraded read (that block's node is down): fetch enough of the remaining
   |  blocks to reconstruct it on the fly, slower, but still correct, invisible to the
   |  user except for extra latency
```

The layer that does not exist in the naive design, and is the entire point of this lesson, is the **temperature tracker feeding an erasure-coded warm tier**. Replication never goes away entirely, hot data still needs it, but it stops being the *only* tool, and it stops being applied to data that no longer needs it.

---

## 4. The transferable mechanisms

**a. Temperature-based tiering.** Track actual measured access frequency per object, not a guess or a fixed age cutoff, and move data between an expensive-fast tier (full replication, low latency) and a cheap-durable tier (erasure coding, more latency on the rare read) once its real access pattern justifies the move. The tracker, not a human, decides.

**b. Reed-Solomon (k, m) erasure coding.** Encode k data chunks into k+m total chunks such that *any* k of the k+m chunks reconstruct the original. Tolerate losing any m chunks simultaneously, at a storage cost of (k+m)/k instead of a full extra copy per replica. Facebook's f4 uses (10,4): 1.4x overhead for surviving 4 losses. Backblaze's open-sourced implementation for its Vault architecture uses (17,3): 20 shards total, about 1.18x overhead, rated at 11 nines of annual durability. Bigger k means lower overhead but also more chunks to fetch on every reconstruction, that trade is a design choice, not a law of nature.

**c. Failure-domain-aware placement.** Spread the k+m chunks of any single object across independent racks, hosts, and power circuits, so one correlated event (a rack losing power, a bad batch of drives from one vendor) cannot take out more chunks of the same object than the code's tolerance budget m allows.

**d. Layered redundancy sized to the blast radius of the threat.** f4 uses one code (Reed-Solomon within a datacenter) for disk- and rack-scale failures, and a second, cheaper code (XOR across a paired datacenter) for datacenter-scale failures. Small, frequent threats get an efficient dedicated mechanism; large, rare threats get a second, separately-tuned mechanism layered on top, instead of over-provisioning one mechanism to cover both.

**e. Continuous background repair, not repair-on-discovery.** Treat "one chunk just went offline" as routine and urgent at the same time: a scrubbing process races to reconstruct the missing chunk from the surviving ones and write a fresh replacement before a second, independent failure can land and cross the tolerance threshold. The system's safety margin shrinks the instant a chunk is lost, and the fix is to restore it fast, not to wait for a human to file a ticket.

**f. Degraded-read reconstruction on the read path itself.** A read for an object whose home chunk is temporarily unavailable is not treated as a failure returned to the caller. It is served by fetching enough surviving chunks and decoding on the fly, correct, just slower, exactly the same "pay latency, not correctness" trade this ledger has used before (Day 19's cache stampede protection, Day 31's read-after-write gate).

---

## 5. The trade-offs

**CAP made concrete, per data type.** For a storage system, "C" is not abstract, it means: whatever bytes you read back are exactly the bytes you wrote, or the read fails loudly, it never silently returns wrong or partial data. That is treated as non-negotiable at every tier in this architecture, hot or warm. What actually gets traded is **availability of the fast path under failure**: a degraded read costs materially more I/O than a healthy one, and a large-scale rebuild event temporarily competes with live traffic for network and disk bandwidth. The system chooses "occasionally slower" over "occasionally wrong" or "occasionally gone," the same choice this ledger keeps finding at every tier of every system.

**Cost vs. latency, with real numbers on both sides.** Triple replication is nearly free at read time (grab any of three identical copies) and expensive at rest (3.6x). Reed-Solomon(10,4) is roughly 60% cheaper at rest (1.4x) but every degraded read means reconstructing from 10 surviving chunks instead of reading 1 direct copy, an order of magnitude more I/O per byte recovered. That gap is exactly why nobody stops at "whatever code has the lowest storage overhead": Microsoft's Azure Storage team found that plain Reed-Solomon reconstruction meant reading roughly 6 other fragments to rebuild one lost fragment, and built **Local Reconstruction Codes** specifically to cut that down, their (12,4,2)-style LRC scheme accepts slightly higher storage overhead (about 1.33x instead of the bare Reed-Solomon minimum) in exchange for cutting reconstruction reads roughly in half. Repair cost is a first-class design variable, not a rounding error you discover after the code is already in production.

**The three real ratios, side by side, are the whole lesson in one line.** Full replication: 3.6x, cheap to read, free to reason about, brutally expensive at rest. f4 single-datacenter Reed-Solomon(10,4): 1.4x, still cheap enough to read that it's viable for "warm" data. f4 with cross-datacenter XOR added: 2.1x, buys back an entire class of failure (losing a datacenter) for less than half the cost of just replicating a third full copy. Backblaze Vaults (17,3): about 1.18x, tuned for a workload that reads even less often than Facebook's warm tier. None of these is "the correct" ratio; each is a deliberate point chosen for a specific access pattern and a specific failure model.

---

## 6. The systems-thinking lens

**The feedback loop here is a rebuild storm, the storage-system twin of a thundering herd.** Trace it: a rack or a batch of disks fails → every erasure-coded chunk that happened to live on that hardware needs simultaneous reconstruction → each reconstruction reads several other chunks over the network from several other machines → thousands of simultaneous reconstructions compete with live production reads for the same finite network and disk bandwidth → live reads get slower, and degraded reads (already the expensive path) get slower still → if reconstruction cannot finish before a **second**, independent correlated failure lands, say another rack, or another disk from the same bad manufacturing batch, the object crosses below its tolerance threshold m and the data is permanently lost. One failure manufactures a burst of expensive recovery work that can make a healthy system look degraded, or in the worst case, turn a survivable single failure into an unsurvivable double failure, purely because the repair itself became the bottleneck.

**The senior fix is not "add more redundancy."** That just re-inflates the cost problem the whole design exists to solve. The fix breaks the loop three ways: (1) rate-limit and prioritize repair traffic so a rebuild can never fully saturate the shared network and disk bandwidth that live reads also need, a backpressure decision with the same shape as Day 13's load shedding; (2) choose a code whose *repair* cost, not just its storage overhead, is a design target, the way Azure's LRC deliberately spends a little extra storage to roughly halve the fragments read per reconstruction; and (3) place chunks so that no single correlated failure domain, one rack, one power circuit, one bad drive batch, can ever threaten more than a small slice of any object's tolerance budget at once. The failure you actually have to survive is not "one chunk goes down," it is "can the system finish repairing chunk one before chunk two goes down too," and every one of these fixes is aimed at winning exactly that race.

---

## Map to Rare.lab's stack

Rare.lab stores content-addressed, immutable scene JSON plus a manifest in Cloudflare R2 (Day 23 covered the content-addressing half of this). R2's own durability underneath that is Cloudflare's problem to solve, not something Rare.lab needs to reimplement with a hand-rolled Reed-Solomon layer; that would be solving a problem R2 already solves. The mechanism from this lesson that transfers directly is the **temperature tracker**, not the erasure code itself.

Every save in a node-based scene editor mints a new immutable, content-addressed blob. That means the corpus of stored versions only grows, and the overwhelming majority of it, every historical revision of every scene except the current HEAD, becomes "warm" almost immediately: a user is far more likely to load the current state of a project than to fetch a version from three weeks ago. That is exactly the access-frequency shape that made Haystack's uniform triple-replication wasteful for Facebook's photo corpus. The proportionate move as storage volume grows, not today, but the ceiling worth planning for: track access frequency per content hash, and once volume justifies it, move rarely-fetched historical scene blobs into R2's lower-cost storage classes instead of paying hot-storage rates for bytes nobody has requested in months, while keeping current-HEAD manifests and recently-touched scene blobs on the fast path. The transferable rule is the one this whole lesson has been building toward: measure temperature, don't assume it, and let the measured curve, not a fixed age cutoff, decide what stays expensive-fast and what moves to cheap-durable.

---

## References and summaries

**Muralidhar, Lloyd, Roy, et al.: "f4: Facebook's Warm BLOB Storage System"** (OSDI 2014)
https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-muralidhar.pdf
The primary source for this lesson's breaking number and architecture: Haystack's ~3.6x effective replication factor for hot BLOBs, f4's use of Reed-Solomon(10,4) single-datacenter encoding (1.4x overhead) layered with cross-datacenter XOR coding (pushing effective replication to about 2.1x), and the 65+ petabyte logical-data figure for the warm tier at time of publication. Also the source for the router/coordinator/rebuilder shape used in section 3.

**Huang, Simitci, et al. (Microsoft): "Erasure Coding in Windows Azure Storage"** (USENIX ATC 2012, Best Paper)
https://www.usenix.org/conference/atc12/technical-sessions/presentation/huang
The source for section 5 and 6's repair-cost argument: plain Reed-Solomon reconstruction requiring roughly 6 fragment reads to rebuild one lost fragment, and Azure's Local Reconstruction Codes (LRC) design, which accepts a modestly higher storage overhead (about 1.33x) in exchange for roughly halving the number of fragments read per reconstruction, making repair cost, not just storage overhead, an explicit design target.

**Backblaze: "Erasure Coding: Backblaze Open Sources Reed-Solomon Code"**
https://www.backblaze.com/blog/reed-solomon/
The source for this lesson's third real-world ratio: Backblaze Vaults' (17,3) Reed-Solomon configuration, about 1.18x storage overhead, rated at 11 nines of annual durability, plus a plain-language walkthrough of how Reed-Solomon reconstructs a message from any k of its n+k encoded pieces, used here as the concrete intuition behind the "10-page fax with 4 checksum pages" analogy in section 3.
