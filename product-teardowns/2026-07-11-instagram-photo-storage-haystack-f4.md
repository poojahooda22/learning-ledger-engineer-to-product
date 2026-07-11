# Instagram photo storage and delivery: Haystack, f4, and the CDN in front

Date: 2026-07-11
Product: Instagram (running on Facebook/Meta photo infrastructure)
Feature: How one uploaded photo is stored forever and served back in a blink

A quick, honest framing note up top. Instagram was bought by Facebook in 2012 and its
images live on Facebook's photo storage stack. The deep, published engineering is in two
Facebook papers: Haystack (OSDI 2010) and f4 (OSDI 2014). So the confirmed internals below
are Facebook's storage engine. The mapping to Instagram specifically (your Instagram photo
sits in this same Haystack/f4 pipeline) is a well-grounded inference from the acquisition and
Meta's shared infrastructure, and I label it as inference where it matters.

---

## 1. The user

Meet Aditi. She is standing at a rooftop cafe in Bandra at 7pm, the light is gold, and she
takes a photo of her cold coffee against the skyline. She taps share. Two seconds later it is
live on her profile. Her friend Rohan, sitting on a train in Pune, opens Instagram and the
photo is just there, sharp and instant, as he scrolls.

Neither of them thinks about storage for one second. That is the whole point. The photo Aditi
posts today has to load exactly this fast in 2031 when she is scrolling her old posts, and it
has to do that for a photo library that, across all users, runs into the hundreds of billions
of images.

## 2. The real problem

Here is the pain, described like a friend would.

Photos are small and there are an obscene number of them. A single coffee photo is maybe
100KB to 3MB. That does not sound hard. The hard part is that there are billions of them, most
of them are old, and people still occasionally look at the old ones.

The naive way to store photos is one file per photo on a normal filesystem, served over
network storage (this is literally what Facebook did first, on NFS-mounted NAS appliances).
That works great for a thousand photos. It falls apart at scale for one brutal reason: reading
one photo file is not one disk read. To open `/2013/aditi/coffee.jpg`, the filesystem has to
walk the directory tree, read the directory's metadata, read the file's inode to find where
the bytes physically live, and only then read the actual bytes. That is three or more disk
seeks for one photo (the Haystack paper measured this as the core disease). A spinning disk
does maybe 100 to 200 random seeks per second. Wasting two of every three seeks on metadata
means you have thrown away two thirds of your most precious resource.

You cannot cache your way out of it either. You could cache the metadata in RAM so lookups are
free, but at billions of photos the metadata alone is too big to fit in memory. And a CDN in
front only helps for popular photos. The killer is the long tail: Aditi's 2013 coffee photo
gets requested rarely, so it is never in the CDN, so every one of those rare requests punches
straight through to the storage disks. Old, unpopular photos are exactly the ones that hurt.

## 3. The feature in one sentence

Store every photo as a tiny record appended into a big shared file, keep a small in-memory map
from photo id to byte offset so any photo is one disk read, and tier cold photos onto
cheap erasure-coded storage while a CDN absorbs the popular ones.

## 4. Jobs to be done

What is Aditi really hiring this system to do?

- "Make my photo appear the instant I post it, and never lose it." (durability + write speed)
- "Load my feed and my old albums with no wait, forever." (read latency on hot and cold)
- "Do all this without my paid plan, for free." (cost per photo must be tiny)

And what is Meta hiring it to do? Serve over a million images per second at peak (2010 figure)
without the disk-seek budget exploding, and keep the storage bill sane while photos accumulate
forever and almost never get deleted.

## 5. How it works for the user

Aditi taps share. A progress bar fills, the app says posted, done. Rohan opens the app and the
image is instant. If he taps into her profile grid and scrolls to a post from three years ago,
that loads instantly too. There is no "loading older photos" spinner. From the outside it is
featureless. That featurelessness is the product.

The one visible artifact of the design is the URL of the image itself. If you ever inspect an
Instagram or Facebook image URL you see a long opaque string with numbers and a token in it.
That token is not decoration. It is the cookie that stops someone from guessing photo URLs, and
the numbers are the routing address of where the bytes live. More on that below.

## 6. The actual flow, step by step

The write path, when Aditi posts the coffee photo:

1. The app uploads the image bytes to a Facebook web server.
2. The web server asks the Haystack Directory: "where should this go?" The Directory picks a
   logical volume that is currently writable and hands back the physical machines behind it.
3. The photo is written as a "needle" (one small record) appended to the end of a big volume
   file on each of typically three machines at once, for redundancy. Append only. No seeking
   to find a slot.
4. Each machine updates its in-memory index: photo id maps to (offset in the file, size).
5. The server hands the app back a URL that encodes how to fetch this photo later.

The read path, when Rohan views it:

1. His app requests the URL. It first hits an external CDN (like Akamai). If the photo is
   popular and cached at the edge, it returns from a server near Pune. Done, no Facebook disk
   touched.
2. On a CDN miss, the request goes to the Haystack Cache, Facebook's own internal CDN. If the
   photo is there, return it.
3. On a Cache miss, the request reaches a Haystack Store machine. The machine reads its
   in-memory index to get the byte offset, then does exactly one disk read at that offset to
   pull the needle. It returns the bytes.
4. Later, when the photo has cooled off and is rarely requested, a background process migrates
   it from Haystack (3x replicated) to f4 (erasure coded, much cheaper). Reads then come from
   f4 instead.

Three stops (CDN, Cache, Store), and the deepest stop is still just one disk read.

## 7. Under the hood, like the engineer

This is the heart of it. Three problems stack: the small-file problem, the read path, and the
temperature problem.

### The core idea: pack many photos into one big file, index in RAM

Haystack's central move is to stop treating a photo as a file. Instead, a physical volume is
one enormous file, around 100GB, living on one Store machine. Inside that 100GB file, photos
are stored back to back as "needles."

One needle (the on-disk record for Aditi's coffee photo) is roughly:

- Header: a magic number, a cookie (a random secret baked into the URL), the key (the photo
  id), an alternate key (which size variant, since Instagram stores several resized copies),
  flags, and the data size.
- Data: the actual JPEG bytes.
- Footer: a checksum and padding.

The data structure that makes reads fast is a plain in-memory hash map. Key is (photo id, size
variant). Value is (offset into the 100GB file, needle size). That is it. To serve a photo the
machine hashes the id, gets the offset, and issues one `pread` at that offset for that many
bytes. One disk operation. No directory walk, no inode read, because from the operating
system's point of view there is only one giant file that was opened once.

Why a hash map and not a tree? Because the only query is exact match by photo id ("give me
needle 8123456789"), never a range scan ("give me all photos between two ids"). Exact-match
point lookups are what hash maps are perfect at: average O(1). A B-tree would buy you ordered
scans you never use and cost you log(n) per lookup. So: hash map.

Concrete size check. An index entry is tiny, on the order of 20 to 40 bytes (a couple of keys,
an offset, a size). A 100GB volume holds maybe a few hundred thousand to a couple million
photos depending on size. So the index for a whole volume is only tens of megabytes. That fits
in RAM easily, which is the entire trick: the metadata that was too big to cache in the old
NFS design becomes tiny once you stop storing a separate inode per photo. You collapsed
billions of inodes into a handful of open files plus compact in-memory maps.

### Writes are appends, updates are appends, deletes are lazy

Because the volume file is append only, writing Aditi's photo is just "seek to end, write
needle, update the map." Fast and sequential, which disks love.

What about editing or replacing a photo? You do not overwrite in place. You append a new needle
with the same key and just point the in-memory map at the new offset. The old needle becomes
dead bytes. Same with deletes: you flip a delete flag in the in-memory map and mark the needle,
but you do not immediately reclaim the space. A periodic compaction job later rewrites the
volume, dropping dead needles, to reclaim space. This is the same log-structured, append-then-
compact pattern that LSM-tree databases use (see this ledger's Day 21 lesson on LSM-trees).
Appending is cheap, cleanup is batched and offline.

### The three serving tiers and why the Cache exists

The Directory, Cache, and Store split the work:

- Haystack Directory: the brain. It maps a logical volume to the physical volumes (machines)
  that hold its replicas, balances writes across volumes, marks a volume read-only once it
  fills, and decides whether a photo's URL should route through the external CDN or through the
  internal Cache. When it builds the URL, it stamps in the machine id and volume so the read
  request is self-routing.
- Haystack Cache: an internal CDN, structured as a distributed hash table keyed by photo id. It
  exists for one specific reason, and it is subtle. It only caches a photo when two things are
  true: the request came from a user directly (a CDN miss, meaning the external CDN could not
  help), and the photo was fetched from a write-enabled Store machine. Why that second rule?
  Because a machine that is actively taking writes is the one whose disks are busiest, and the
  long tail of reads to recently written photos is exactly what would thrash those disks. The
  Cache shields the machines that can least afford extra random reads.
- Haystack Store: the actual bytes plus the in-memory index. One read per photo.

Walk Rohan's request as a real address. The URL looks conceptually like
`http://<CDN>/<Cache>/<machine-id>/<logical-volume>,<photo-id>`. The external CDN tries first.
Miss. The Cache tries next, keyed by photo id. Miss. Now the machine id and volume in the URL
send the request straight to the right Store machine, which reads its map for photo-id, gets
offset 4,214,880,256, reads that needle, returns the JPEG. The cookie in the needle header is
checked against the cookie in the URL so a stranger cannot enumerate photo ids and scrape.

### The temperature problem, and f4

Here is the second published system. Photos have a temperature. A photo is hot the day it is
posted: everyone's feed pulls it, it is requested thousands of times. Within days to weeks the
request rate collapses. Aditi's coffee photo is red hot tonight and stone cold by August.

Keeping cold photos in Haystack is wasteful. Haystack replicates each photo 3x (three full
copies on three machines) because that gives read throughput and durability, which hot photos
need. But a cold photo requested once a month does not need three whole copies for throughput.
It only needs durability. Paying 3x storage (actually higher once you count the historic RAID
overhead the f4 paper cites, roughly 3.6x effective) for bytes nobody reads is pouring money
on a shelf.

So f4 (OSDI 2014) is the warm and cold tier. It uses Reed-Solomon(10,4) erasure coding. Take
10 blocks of photo data, compute 4 parity blocks, spread all 14 across different racks. You can
lose any 4 of the 14 and still rebuild every byte. The storage cost of that is a 1.4x expansion
(14/10) instead of 3.6x. Reads normally just read the data block directly (cheap). Only when a
disk or rack has failed do you pay the expensive reconstruction that reads 10 blocks and solves
for the missing one. Since failures are rare, you almost always pay the cheap path. f4 layers an
XOR geo-replication trick across datacenters on top, landing at roughly 2.1x effective
replication for full multi-datacenter durability, versus 3.6x before.

The numbers Facebook published: f4 stored over 65PB of logical BLOBs and saved over 53PB of
storage versus the old scheme, covering 400 billion plus photos as of February 2014. That saved
53PB is the entire reason the tier exists. (This ledger's Day 23 lesson on content-addressed
storage and Day 22 on leaderless replication both touch the erasure-coding-vs-replication
tradeoff.)

### The scale story at three tiers

1,000 photos. A normal filesystem, one file per photo, is completely fine. All the metadata
fits in the OS page cache, seeks are plentiful. Do not build Haystack. This is Aditi's phone.

100,000 photos and up. Now the small-file problem bites. Per-photo inodes stop fitting in RAM,
so every read of a not-recently-used photo pays multiple disk seeks just to find the bytes.
Your seek budget evaporates and read latency spikes on exactly the old photos. The fix is the
Haystack move: pack many photos into large volume files and keep a compact in-memory
offset index, turning three-plus seeks into one. What broke: metadata did not fit in memory.
What survived it: collapse the metadata by collapsing the files.

10 million plus, into the billions. One machine cannot hold it and one copy is not durable
enough. You shard into logical volumes spread across many machines, replicate hot volumes 3x
for throughput, and put a Directory in front to route and load-balance writes. Reads are
absorbed by an external CDN plus the internal Cache so the Store disks only see the long tail.
Then storage cost itself becomes the ceiling: at hundreds of billions of mostly-cold photos,
paying 3.6x for cold bytes is the thing that breaks the budget. What survived it: tier cold
photos to f4's erasure coding to cut effective replication from 3.6x toward 2.1x, and reclaim
dead space with background compaction. The 2010 scale the Haystack paper cites: 260 billion
images, 20PB, one billion new photos (about 60TB) uploaded per week, one million-plus images
served per second at peak.

## 8. The retention and habit mechanic

This is infrastructure, so the loop is not a notification or a streak. It is data gravity, and
it is a defensive moat, not an engagement hook.

The mechanic works like this. Because storage is effectively free and unlimited to the user,
Aditi never deletes anything. Every photo she has ever posted, since 2013, just accretes on the
platform. Because the read path is one disk read behind a CDN, all of that history stays
instantly loadable forever, so browsing her own past never feels slow enough to make her stop.
Over years, her entire visual autobiography lives inside Instagram and loads like it was posted
today. That accumulated, always-fast history is the switching cost. Leaving means abandoning a
decade of instantly-accessible memories. No competitor can import that feeling.

The metric it moves is retention, specifically long-horizon retention and the switching cost
that suppresses churn. It does not directly move activation or revenue; it quietly makes leaving
unthinkable. A real observed example of the same force: people who keep their entire photo
history on a platform for years are dramatically less likely to churn, which is exactly why
every photo product (Google Photos, Apple iCloud, Instagram) races to be the free, infinite,
instantly-loading home for your images. The storage engineering is what makes "free and
instant forever" affordable enough to offer, and the offer is the moat.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code, plus
an embeddable runtime. You will accumulate a huge number of small assets: individual textures,
node-graph documents, compiled shader binaries, thumbnails, and many versioned copies of each as
users iterate. That is the small-file problem waiting to happen, and it is the same shape as
Instagram photos.

One concrete, actionable lesson: do not store one file per asset, and tier by temperature.

Practically:

- Pack many small assets into large append-only pack files and keep an in-memory (or Redis)
  map from a content-addressed asset id to (pack file, byte offset, size). Serving an asset to
  the runtime then becomes one read at a known offset, not a directory walk. This is the single
  most important move for keeping the embeddable runtime's asset fetch fast as a project grows
  from 1,000 to 1,000,000 assets. Your seek budget is your latency budget.
- Address assets by content hash (a needle key), which also gives you free deduplication:
  two projects using the same 2K noise texture point at the same needle. See this ledger's Day
  23 lesson on content-addressed storage.
- Tier by temperature exactly like Haystack to f4. The shader graph a user is actively editing
  and the assets in the project they shipped last week are hot: keep them 3x replicated and
  CDN-fronted for instant load in the editor and runtime. Old project versions and the long tail
  of published-but-quiet projects are cold: move them to erasure-coded cold storage to cut your
  storage bill from roughly 3x toward roughly 1.4x, and only pay reconstruction cost on the rare
  failure. At scale that saved multiple is the difference between an affordable free tier and a
  bleeding one.
- Put a CDN in front of the runtime's asset fetches so a shader shipped inside someone's website
  loads its textures from an edge near the end user, and your origin only ever sees the long
  tail. One read per asset, CDN first, cold tier cheap. That is the whole recipe.

Bias to performance, the way the whole ledger keeps repeating: do the expensive thinking offline
(packing, indexing, compaction, tiering) so the live path a user or a runtime hits is a single
cheap keyed lookup.

---

## Sources

- Beaver, Kumar, Li, Sobel, Vajgel. "Finding a Needle in Haystack: Facebook's Photo Storage."
  OSDI 2010. https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf
- USENIX listing for the Haystack paper.
  https://www.usenix.org/conference/osdi10/finding-needle-haystack-facebooks-photo-storage
- Muralidhar et al. "f4: Facebook's Warm BLOB Storage System." OSDI 2014.
  https://www.usenix.org/system/files/conference/osdi14/osdi14-paper-muralidhar.pdf
- f4 paper mirror (Princeton).
  https://www.cs.princeton.edu/~wlloyd/papers/f4-osdi14.pdf
- The Morning Paper summary of f4 (Adrian Colyer).
  https://blog.acolyer.org/2014/12/16/f4-facebooks-warm-blob-storage-system/
- Stephen Holiday's notes on Haystack (clear secondary walkthrough).
  https://stephenholiday.com/notes/haystack/
