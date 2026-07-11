# References: Instagram / Facebook photo storage (Haystack + f4 + CDN)

Keeper links for the 2026-07-11 teardown on how one photo is stored forever and served in a blink.

## Primary sources (the two papers)

- Beaver, Kumar, Li, Sobel, Vajgel. "Finding a Needle in Haystack: Facebook's Photo Storage."
  USENIX OSDI 2010. The core system: needles, in-memory index, one disk read, Directory + Cache
  + Store, cookies in the URL.
  https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf
- USENIX OSDI 2010 listing for the Haystack paper.
  https://www.usenix.org/conference/osdi10/finding-needle-haystack-facebooks-photo-storage
- Muralidhar, Facebook, et al. "f4: Facebook's Warm BLOB Storage System." USENIX OSDI 2014.
  Temperature tiering, Reed-Solomon(10,4), XOR geo-replication, ~2.1x effective vs 3.6x.
  https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-muralidhar.pdf
- f4 paper mirror (Princeton, Wyatt Lloyd).
  https://www.cs.princeton.edu/~wlloyd/papers/f4-osdi14.pdf

## Secondary walkthroughs worth keeping

- Adrian Colyer, The Morning Paper, summary of f4 (clear, correct).
  https://blog.acolyer.org/2014/12/16/f4-facebooks-warm-blob-storage-system/
- Stephen Holiday's notes on the Haystack paper.
  https://stephenholiday.com/notes/haystack/

## Key numbers to remember

- Haystack (2010): 260B images, 20PB, ~1B new photos/week (~60TB), 1M+ images/sec at peak.
- Old NFS design pain: 3+ disk seeks per photo read (dir walk, inode, data); billions of
  inodes do not fit in RAM to cache; CDN cannot help the long tail of old photos.
- Volume file ~100GB, one giant file, photos appended as needles; index entry ~20 to 40 bytes,
  so a whole volume's index is tens of MB and fits in RAM.
- Needle = header (magic, cookie, key=photo id, alt key=size variant, flags, size) + data +
  footer (checksum). Cookie stops URL enumeration.
- f4 (2014): Reed-Solomon(10,4) = 1.4x expansion; +XOR geo => ~2.1x effective vs 3.6x; stored
  65PB logical, saved 53PB, 400B+ photos as of Feb 2014.
- Serving tiers: external CDN -> internal Haystack Cache (DHT by photo id, caches only
  CDN-miss reads from write-enabled Stores) -> Store (one pread at the indexed offset).

## Framing note

The confirmed internals are Facebook's storage engine. Instagram (acquired 2012) runs on Meta's
shared photo infrastructure, so "your Instagram photo lives in Haystack/f4" is a well-grounded
inference from the acquisition and Meta's shared infra, not a separately published Instagram paper.
