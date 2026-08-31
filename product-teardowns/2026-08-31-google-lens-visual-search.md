# Google Lens visual search: search what you see (point the camera, find the object)

Date: 2026-08-31
Product: Google Lens (visual search / "search by image" / find-this-object)
Feature: Point-and-search visual lookup. You take a photo or point the camera at a
thing, and Google finds where that thing appears across the web: the same shoe, the
same plant, the same painting, the same product you can buy.

This is not text search with a picture bolted on. There are no words in the input. The
query is pixels, and the catalog is billions of images. That flips the whole machine
around. This teardown pulls that machine apart.

---

## 1. The user, in their day

Meet Aditi. She is at a friend's flat in Indiranagar on a Saturday evening. On the
balcony there is a plant with fat, waxy, split leaves. She loves it. She wants one for
her own place. But she has no idea what it is called. "The big split-leaf plant" is not
a search you can type and win.

Or take Ravi, standing in a mall in front of a store window. There is a pair of white
sneakers with a green tab on the heel. No brand visible from where he stands. He wants
to know what they are and whether they are cheaper online.

Or Meera on the sofa, TV paused on a movie, staring at a lamp in the background of the
scene. "I want that exact lamp."

In every case the person is looking at a real object in the real world and the thing
they want to know is locked inside how it looks, not inside any words they can produce.
Typing has failed them before they even start.

The concrete case we will walk end to end: Aditi points her camera at that plant. It is
a Monstera deliciosa. Hold that example. We will follow it all the way down to the
math.

---

## 2. The real problem

Text search has a cruel gap. To search for a thing, you have to already know the word
for the thing. But the whole reason you are searching is that you do not know the word.

That is the trap. "How do I look up a thing when I cannot name it?" Aditi cannot type
"Monstera deliciosa" because she has never heard those words in her life. She could try
"big green plant split leaves indoor," get a wall of blog posts, and scroll for ten
minutes comparing pictures by eye. That is not search. That is a chore.

The pain, said plainly like a friend would: your eyes already know the answer, your
fingers just cannot type it. You are holding all the information (the shape, the color,
the texture) in a form the search box refuses to accept. The gap between "I can see it"
and "I can name it" is exactly where you get stuck.

Visual search closes that gap. It lets the picture be the query.

---

## 3. The feature in one sentence

Google Lens turns an image (or a cropped region of one) into a compact numeric
fingerprint, then finds the images across a catalog of billions whose fingerprints sit
closest to it, and shows you what they are and where to get them.

---

## 4. Jobs to be done

What is Aditi really hiring Lens to do?

- "Tell me the name of this thing so I can act on it." (Identify.)
- "Find me the exact same object, not just similar-looking ones." (Match, not vibe.)
- "Show me where to buy it and for how much." (Shop.)
- "Do it in one motion, from the thing itself, without me typing a word." (Zero-friction.)
- "When it is text on a sign or a menu, read it and translate it." (Lens also does OCR
  and translation; this teardown focuses on the object-matching half, which is the
  harder engineering.)

The deep job: convert something I can only see into something I can name and buy.

---

## 5. How it works for the user (the visible experience)

Aditi opens the Google app (or Photos, or Chrome, or long-presses a photo; Lens is
wired into all of them). She taps the little camera icon in the search bar. The camera
opens. She points it at the plant and taps the shutter, or she picks a photo she
already took.

A set of white dots shimmers over the image for a moment. Then bounding boxes appear
around the objects Lens found: one around the plant, maybe one around a pot, one around
a book on the table behind it. A little chip says "Plant."

She taps the plant. The screen splits. Top half: her photo with the plant highlighted.
Bottom half: a scrollable tray of visual matches. Photo after photo of the same waxy
split leaves. Above them, the name she never had: "Monstera deliciosa" (also called the
Swiss cheese plant). Shopping results with prices. Care guides. All from one tap on a
picture.

Total time from opening the camera to knowing the plant's name: about two seconds.

---

## 6. The actual flow, step by step

1. Aditi taps the camera icon. The client opens the camera preview.
2. She frames the plant and taps capture. The client holds the full-resolution frame.
3. The client (or the server, depending on surface) runs object detection: it draws
   boxes around the distinct things in the frame and labels each with a coarse category.
   The plant gets a box and the tag "Plant."
4. The white dots and boxes render. This is the client telling her: "I found things,
   pick one."
5. She taps the plant box. The client crops the image to that box. Now the query is not
   the whole cluttered balcony; it is just the plant.
6. The cropped region is sent to Google's servers (as compressed pixels, or as an
   on-device embedding on some surfaces).
7. The server runs the crop through a vision model that outputs an embedding: a fixed
   list of numbers, say 64 or 128 of them, that encodes what the crop looks like.
8. That embedding is used to fetch nearest neighbors from an index of billions of image
   embeddings. This returns a few hundred to a few thousand candidate images that look
   close.
9. Those candidates are re-ranked with a heavier, more precise model and blended with
   signals like page quality, product availability, and freshness.
10. The top results, their labels ("Monstera deliciosa"), and shopping links stream back
    and render in the bottom tray.

Steps 7, 8, and 9 are the whole ballgame. The rest is plumbing. Notice the shape: a
matching half (steps 7 and 8, get the candidates) and a ranking half (step 9, order
them). Same two-halves split we have seen in every search teardown in this ledger. Text
or pixels, the skeleton is identical.

---

## 7. Under the hood, like the engineer

This is the heart. We go deep. Where Google has published the internals I mark it
[FACT]. Where I am reasoning about how this class of problem is solved, I mark it
[INFERENCE] plainly.

### 7a. The core idea: turn a picture into a point in space

You cannot compare two images by comparing pixels. Two photos of the same Monstera,
one in morning light and one at night, have almost no pixels in common as raw numbers,
yet they are the "same" thing. And a red car and a red apple share lots of red pixels
yet are nothing alike. Raw pixels are the wrong language.

So the first move is to learn a better language. A deep vision model (a convolutional
network or a vision transformer) is trained so that it reads an image and emits a short
vector, say 64 or 128 floating point numbers. This vector is called an embedding. The
model is trained so that images of the same thing land close together in this vector
space, and images of different things land far apart. [FACT: Google's own patent on
this, US11782998B2 "Embedding Based Retrieval for Image Search," describes embedding
sub-networks that map visual, and optionally textual, features into a shared embedding
space for retrieval.]

Picture a 64-dimensional space (you cannot, nobody can, but pretend). Every image the
world has ever put on the web is a single dot in that space. All the Monstera photos
form a little cloud of dots huddled together. All the sneaker photos form a different
cloud somewhere else. Aditi's plant crop, once embedded, is a new dot. The job is:
find the dots nearest to her dot. Those are her answers.

"Nearest" is measured by distance, usually cosine similarity or inner product. Two
embeddings that point in nearly the same direction are near. [FACT: ScaNN, Google's
search library discussed below, is built for Maximum Inner Product Search, MIPS, which
is exactly "find the database vectors with the largest inner product against my query
vector."]

Concrete: Aditi's plant crop becomes something like [0.02, -0.13, 0.88, ...] with 64
entries. A stock photo of a Monstera on a nursery website became [0.03, -0.11, 0.86,
...] months ago when Google indexed that page. Those two lists are almost the same list.
That closeness is the whole answer. The name "Monstera deliciosa" is just the caption
attached to the nursery photo that turned out to be nearest.

### 7b. The two halves: matching (get candidates) then ranking (order them)

Half one, matching: out of billions of image embeddings, pull the few thousand nearest
to the query. This must be brutally fast and can be approximate. It just has to not miss
the real answer.

Half two, ranking: take those few thousand, score them with a heavier model and extra
signals (is this a shopping page with the item in stock, is the source reputable, is it
recent), and produce the final order. This is slower per item but only runs on a small
set. [INFERENCE, but this is the standard shape of every large retrieval system Google
has published, from web search to YouTube recommendations, and matches the "double
embedding" observation in the Lens patent: a small 64-dim vector for fast filtering, a
larger 128-dim vector for precise re-ranking of the survivors.]

Do not merge these halves in your head. Matching optimizes recall (do not lose the
answer). Ranking optimizes precision (put the best answer first). They use different
models and run at different scales.

### 7c. Half one in detail: why you cannot just check every image

The naive way to find nearest neighbors: compute the distance from Aditi's dot to every
other dot and keep the smallest. This is brute-force k-nearest-neighbor. It is a linear
scan. Cost grows straight-line with the catalog size.

Do the arithmetic. Each distance is a dot product over 64 numbers, roughly 64 multiply
and add operations. At a catalog of:

- 1,000 images: 64,000 operations. A microsecond. Trivial. Just scan everything.
- 100,000 images: 6.4 million operations per query. Still a few milliseconds. Fine.
- 10 million images: 640 million operations per query. Tens of milliseconds, per query,
  and you have thousands of queries a second. Now it hurts.
- Billions of images (Lens's real scale): a brute-force scan is flatly impossible inside
  a two-second answer. Hundreds of billions of operations per query, times the query
  load. It falls over.

Brute force dies somewhere past a few million vectors. This is the wall. Everything
interesting in visual search is what you do at the wall.

### 7d. Surviving the wall: approximate nearest neighbor with ScaNN

You stop insisting on the exact nearest neighbor and accept the almost-nearest, in
exchange for going orders of magnitude faster. This is Approximate Nearest Neighbor
search, ANN. Google's public, open-source library for it is ScaNN (Scalable Nearest
Neighbors). [FACT: ScaNN is open-sourced on GitHub under google-research/scann, and its
method is published in Guo et al., "Accelerating Large-Scale Inference with Anisotropic
Vector Quantization," ICML 2020. Google's Vertex AI Vector Search is built on ScaNN.]
ScaNN, or a close in-house relative, is exactly the class of engine a billion-image
visual search needs. Its pipeline has three stages. [FACT: partitioning, then
anisotropic vector quantization, then rescoring.]

Stage one, partitioning (prune the space). Before any query arrives, cluster all the
billions of embeddings into groups using k-means. Say 100,000 clusters, each with a
center point. This is a one-time offline build. Now when Aditi's query dot arrives, you
do not compare it to all billion dots. You compare it to the 100,000 cluster centers
(cheap), pick the few dozen closest clusters, and only search inside those. You just
threw away 99% of the catalog without looking at it. This is a tree/inverted-index idea:
the cluster centers are the index, the members are the postings. Concrete: Aditi's plant
dot lands near the "leafy green houseplant" clusters, so the nursery Monstera photos
(which live in those same clusters) are in the searched set, while photos of cars,
faces, and invoices are in clusters you never open.

Stage two, quantization (shrink each vector). Storing billions of 64-float vectors in
full precision is enormous, and reading them from memory is slow. So compress each
vector into a tiny code, a handful of bytes, using product quantization: chop the vector
into chunks and replace each chunk with the id of the nearest entry in a small learned
codebook. Now a vector that was 256 bytes becomes maybe 16 bytes. The whole index fits
in RAM and streams fast. The distance is computed on these compressed codes.

Here is ScaNN's actual contribution, and it is clever. Ordinary quantization tries to
reconstruct each vector as accurately as possible on average. ScaNN says: for search,
not all errors matter equally. The vectors that will end up as top results are the ones
with a large inner product against the query. So it uses an anisotropic loss that
penalizes error in the direction that changes the inner-product score (the parallel
component of the residual) much more than error in directions that do not. [FACT: from
the ScaNN paper, the loss "more greatly penalizes the parallel component of a
datapoint's residual relative to its orthogonal component." The intuition, from Google's
write-up: the quantization error is shaped so it is largest exactly when the query is
dissimilar to the point, where it cannot flip a result, and smallest where it matters.]
In plain words: be sloppy where sloppiness is harmless, be precise where a wrong nudge
would drop the true match. [FACT: this let ScaNN beat other similarity-search libraries
by roughly a factor of two and post state-of-the-art numbers on the public
ann-benchmarks glove-100 dataset.]

Stage three, rescoring (fix the approximation). The compressed distances are fast but
fuzzy. So take the top candidates from stage two, a few hundred, and recompute their
distances using the full, uncompressed vectors. This cleans up the ordering right where
it counts, on the handful that might actually win. Cheap, because it runs on hundreds,
not billions.

Put together: Aditi's query touches 100,000 cluster centers, opens a few dozen clusters,
scans their compressed member codes, and exactly rescores a few hundred survivors.
Maybe a few hundred thousand cheap operations instead of hundreds of billions. That is
the difference between impossible and two seconds.

### 7e. One more scale trick: SOAR (redundancy so you never miss the answer)

Clustering has a failure mode. If Aditi's dot sits right on the boundary between two
clusters, its true nearest neighbor might live in the cluster you did not open. You miss
it. Recall drops.

Google's fix is SOAR (Spilling with Orthogonality-Amplified Residuals). [FACT: published
as "SOAR: Improved Indexing for Approximate Nearest Neighbor Search," 2024, and shipped
in ScaNN.] The idea: assign each vector to more than one cluster, a primary and one or
two backups, but choose the backup cleverly so it covers exactly the query directions
where the primary assignment is weak (the orthogonal directions). A redundancy factor
controls how many copies. [FACT] Cost: the index gets a bit bigger. Payoff: a vector can
now be found through multiple paths, so boundary cases stop getting lost. [FACT: ScaNN
with SOAR posts querying throughput several times higher than comparable libraries at
matched index-build time on ann-benchmarks glove-100 and the Big-ANN 2023 benchmarks.]

So the Monstera nursery photo might live in both the "leafy houseplant" cluster and, as
a backup, the "large tropical foliage" cluster. Whichever way Aditi's dot leans, the
photo is reachable.

### 7f. Where the sorting happens (say it out loud)

None of the ranking happens on Aditi's phone. The phone's job is capture the frame, help
her crop, maybe compute a small embedding, and render results. The billion-vector index,
the ANN search, the rescoring, the final ranking, all of it lives server-side in
Google's data centers, on machines with the whole quantized index held in RAM and
sharded across many servers. The phone could never hold a billion vectors, and you would
not want to ship that index to a billion phones. [INFERENCE on the exact serving split
for Lens specifically, but this is how every web-scale retrieval system is built, and
the Lens patent describes server-side embedding retrieval.]

### 7g. Sharding, the last tier

At billions of vectors, even the compressed index does not fit on one machine. So it is
sharded: split across many servers, each holding one slice of the catalog. A query fans
out to all shards in parallel, each shard returns its local top candidates, and a
gathering layer merges them into a global top list, then rescores. [INFERENCE, standard
distributed-search design, e.g. the same scatter-gather pattern behind Google web search
and the inverted-index sharding we covered in the Amazon search and Swiggy search
teardowns.] This is why the catalog can 10x again without the per-query latency
exploding: you add shards, not depth. Each shard still only searches its own few dozen
clusters.

Scale story, compressed:
- 1,000 vectors: brute force, one machine, microseconds. No cleverness needed.
- 100,000 vectors: brute force still fine, single box, low milliseconds.
- 10 million vectors: brute force starts to hurt; move to partitioned ANN (cluster and
  prune), quantize to fit RAM.
- Billions (Lens): partition + anisotropic quantization + rescoring + SOAR redundancy +
  sharding across many servers with the index in RAM. Each new tier breaks a different
  thing: first raw scan speed (fixed by clustering), then memory footprint (fixed by
  quantization), then recall at cluster boundaries (fixed by SOAR), then single-machine
  capacity (fixed by sharding).

---

## 8. The retention and habit mechanic

The loop Lens builds is "the camera is a search box." Once you have pointed your phone
at a plant and gotten its name in two seconds, a small rewiring happens. The next time
you see something you cannot name, a menu in a language you cannot read, a jacket you
like, a landmark, your thumb reaches for the camera icon without a decision being made.
The world becomes queryable. Every unnamed object is now a potential search.

Which metric does it move? Two, stacked.

- Activation and engagement for Google Search: Lens manufactures brand-new searches that
  would never have been typed, because they could not be typed. Aditi was never going to
  search "Monstera deliciosa." Lens created a query out of thin air. [FACT: Google has
  repeatedly said Lens is used billions of times a month and has framed multimodal and
  visual search as a major growth surface for total Search queries, especially with
  younger users.]
- Revenue, through shopping: a huge fraction of visual searches are on products (shoes,
  furniture, clothes). Point at Ravi's sneakers, get "buy it here for X." That is a
  commercial query with clear intent, the most monetizable kind there is. Visual search
  quietly turns the camera into the top of a shopping funnel.

The real observed example: Google has pushed "Circle to Search" on Android, where you
long-press the home button and circle anything on your screen (a bag in a friend's
Instagram photo, a word in a video) to Lens it in place. That is the habit loop made
frictionless: no app switch, no screenshot, search from inside whatever you are already
looking at. The whole design is to make the distance from "I see it" to "I searched it"
approach zero, because a shorter loop is a loop you run more often.

---

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable
code, plus an embeddable runtime. The Lens lesson maps almost too cleanly.

Build embedding-based nearest-neighbor search over your shader graphs and effects, and
make it the way users find things. Concretely:

Give every shader graph, node preset, and finished effect a rendered thumbnail, then run
that thumbnail (and the graph structure) through an embedding model so each asset becomes
a vector. Now "find effects that look like this" becomes a nearest-neighbor query, not a
keyword hunt. A user who made a shimmering water caustic and wants "more like this, but
for lava" should be able to point at their own node and get visually and structurally
similar graphs back. Same trap Lens solved: your users can see the effect they want
long before they can name it. Tags and keywords will always fail the person who does not
know your vocabulary. The thumbnail is the query.

And take the scale discipline seriously from day one, because it is exactly ScaNN's
three-stage lesson:

- While your library is small (1,000 presets), just brute-force the cosine similarity in
  memory. Do not build infrastructure you do not need.
- As it grows to 100,000 community-shared graphs, switch to partitioned ANN: cluster the
  embeddings, search only the near clusters. Quantize the vectors so the whole index
  stays in RAM in the browser or on a modest server.
- Design the boundary case out early with a SOAR-style redundant assignment, so a graph
  that sits between "fire" and "smoke" is reachable from both. In a creative tool,
  missing the one perfect match a user was hoping for is the difference between "this
  search is magic" and "this search is useless," and recall at cluster edges is precisely
  where that magic lives.

One sharper, performance-flavored point, since that is where everything connects for
you: adopt ScaNN's anisotropic insight as a design attitude, not just a library. Spend
your precision budget where a wrong answer costs the user, and be cheap everywhere else.
In the shader editor that means: compute exact, expensive similarity only for the top
handful of candidates a user is about to see, and use a fast, lossy, quantized score to
throw away the thousands they will never scroll to. Approximation is not a compromise you
apologize for. Placed correctly, it is the only reason the feature can exist at all.

---

## Sources

- Guo, Sun, Lindgren, Geng, Simcha, Chern, Kumar. "Accelerating Large-Scale Inference
  with Anisotropic Vector Quantization." ICML 2020. https://proceedings.mlr.press/v119/guo20h.html
  and arXiv:1908.10396 https://arxiv.org/abs/1908.10396
- Google Research. "Announcing ScaNN: Efficient Vector Similarity Search."
  https://research.google/blog/announcing-scann-efficient-vector-similarity-search/
- Google Research. "SOAR: New algorithms for even faster vector search with ScaNN."
  https://research.google/blog/soar-new-algorithms-for-even-faster-vector-search-with-scann/
  and "SOAR: Improved Indexing for Approximate Nearest Neighbor Search," arXiv:2404.00774
  https://arxiv.org/abs/2404.00774
- ScaNN open-source library, Google Research. https://github.com/google-research/google-research/tree/master/scann
- Google patent US11782998B2, "Embedding Based Retrieval for Image Search."
- RESONEO, "How Google Lens works" (patent-based teardown). https://think.resoneo.com/google-lens/
- Google Developers Blog. "See the Similarity: Personalizing Visual Search with
  Multimodal Embeddings." https://developers.googleblog.com/see-the-similarity-personalizing-visual-search-with-multimodal-embeddings/
- Google Cloud, Vertex AI Vector Search (built on ScaNN) documentation.
- ann-benchmarks.com, public ANN benchmark suite (glove-100 dataset).

Fact vs inference note: ScaNN's three-stage pipeline, the anisotropic quantization loss,
the "factor of two" and benchmark claims, SOAR's redundant assignment, and embedding-
based image retrieval are all published facts from the sources above. The specific
serving split for Google Lens (exact embedding dimensions, on-device vs server split,
whether Lens uses ScaNN itself or an in-house sibling, and the exact sharding layout) is
labeled inference where stated: it follows the standard design of Google's published
retrieval systems and the Lens patent, not a confirmed Lens architecture diagram.
