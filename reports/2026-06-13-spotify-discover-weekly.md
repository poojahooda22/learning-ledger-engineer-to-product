# Spotify Discover Weekly

Date: 2026-06-13
Product: Spotify
Feature: Discover Weekly (the personalized Monday playlist of 30 songs)

---

## 1. The user

Picture Aditi, 27, on her Monday morning commute on the Bangalore metro. Earbuds in, phone in hand. She is tired of her own playlist. She has played the same 40 songs for three weeks. She does not want to spend 20 minutes hunting for new music. She just wants something fresh to start her week, music that sounds like her taste but that she has not heard before.

She opens Spotify. There, near the top, is a playlist called Discover Weekly with her face stamped on it. Thirty songs. Made for her. Refreshed today.

## 2. The real problem

Finding new music you actually like is hard work. There are tens of millions of songs. Searching means you already know what you want, so search cannot help you find the unknown. Browsing genre pages dumps you into a sea of strangers. Friends' recommendations are rare and hit or miss.

The pain in plain words: "I want new songs that feel like me, but I do not want to do the digging, and I do not want to risk 30 minutes of music I will skip." Most discovery before Discover Weekly meant either effort or disappointment. Usually both.

## 3. The feature in one sentence

Every Monday, Spotify hands you a private playlist of 30 songs you have probably never heard, chosen to match your taste, with zero effort from you.

## 4. Jobs to be done

What Aditi is really hiring this feature to do:

- "Bring me new music without making me search for it."
- "Make the new stuff feel safe, like it belongs in my library, not random noise."
- "Give me a reason to come back and a small weekly ritual."
- "Let me discover privately, without judgment, no social pressure."

## 5. How it works for the user

Aditi does nothing to set it up. She just listens to music normally over the week. She plays songs, she skips songs, she adds some to playlists, she lets others run to the end. That ordinary behavior is the only input.

Every Monday the playlist quietly regenerates. Old songs roll off, 30 new ones roll in. If she loves a track she saves it. If she skips one, that signal feeds back into next Monday's list. The playlist gets sharper the more she uses Spotify.

## 6. The actual flow, step by step

1. Aditi opens the Spotify app on Monday.
2. On Home, under "Made For You," she sees the Discover Weekly cover with her name.
3. She taps it. Thirty tracks load, roughly two hours of music.
4. She hits play and lets it run during her commute.
5. She skips track 4 after ten seconds. She saves track 9 to her library. She lets track 12 play to the end and replays it.
6. Those actions (skip, save, full play, replay) become fresh signals.
7. Next Monday, the playlist rebuilds. Track 12's "neighbors" are more likely to show up. Track 4's style is slightly down-weighted.

## 7. Under the hood, like the engineer

This is the heart of it. Discover Weekly is a recommendation problem, not a search problem. Search starts from a query you type. Discovery starts from your behavior and finds things you did not ask for. The engine has three documented pillars, then a final assembly step. (Three-pillar model confirmed by Sophia Ciocca's widely cited write-up and by Spotify's own engineers; exact current production blend is not fully public, so the assembly stage below is labeled inference where it is.)

### Pillar 1: Collaborative filtering (the big one)

Start with one giant table. Rows are users. Columns are songs. A cell is 1 if that user played that song, near 0 if not. With roughly hundreds of millions of users and tens of millions of songs, this table has trillions of cells, and almost all of them are empty. Aditi has touched maybe a few thousand songs out of 30 million plus.

Storing that table as a dense 2D array is impossible. A full matrix of, say, 500 million users by 30 million songs would need more memory than exists in any cluster. So the real structure is a **sparse representation**: store only the cells that are non-zero, as lists of (user, song, play-count) entries. That is the right data structure because the data is over 99.99 percent empty. A hash map or adjacency list of "user to songs they played" holds the same information in a tiny fraction of the space.

Then comes **matrix factorization**. The idea: squeeze that giant sparse table into two small dense tables.

- A user table: every user becomes a short vector, for example 40 to 200 numbers. Aditi is now a list like [0.3, -1.1, 0.7, ...].
- A song table: every song becomes a vector of the same length. The track "Kesariya" is now [0.2, -0.9, 0.8, ...].

These numbers are learned, not assigned. Each number is a hidden "taste dimension" that no human labels. Multiply a user vector by a song vector (a dot product) and you get a predicted affinity score. High score means likely to love it.

Concrete example. Aditi and a stranger in Pune both heavily played the same 60 indie tracks. Matrix factorization places their two user vectors close together in this space. The stranger also loves a song Aditi has never played. That song's vector sits near both of them. So it scores high for Aditi. That is the recommendation: "people whose taste vector looks like yours played this, you have not, here it is."

Now the scale trick. Once everyone is a vector, the question becomes "find the songs whose vectors are closest to Aditi's vector." Comparing Aditi against all 30 million song vectors one by one, every week, for hundreds of millions of users, is far too slow. Brute force nearest neighbor is O(N) per user, which is billions of operations per playlist.

This is where **Annoy** comes in, a real open-source library Spotify built and published (github.com/spotify/annoy, by Erik Bernhardsson). Annoy stands for Approximate Nearest Neighbors Oh Yeah. It does not check every song. It builds a **forest of binary trees**. At each tree node it picks a random hyperplane that splits the song vectors into two halves, then splits again, and again, until each leaf holds only a few songs. To find Aditi's neighbors, you walk down the trees following which side of each split she falls on. You only ever look at a small bucket of candidates, not all 30 million. This turns an O(N) scan into roughly O(log N) per tree.

Two engineering details from Annoy's own docs that matter at scale:
- The index is a **read-only file that is memory-mapped (mmap)**. Many worker processes on a machine share one copy of the index in RAM instead of each loading its own. You need only enough RAM to fit the index once, then fan out lookups across all CPUs.
- Index building is decoupled from index serving. You build the big tree file in a batch job, ship the static file out, and any process can mmap it and start answering queries instantly. That is how you push billions of lookups a day.

### Pillar 2: Natural language processing (what the world says)

Spotify crawls the web: blogs, articles, playlist titles, reviews. It treats each song and artist like a document and pulls out "cultural" terms and adjectives that cluster around them, things like "chill," "mellow," "monsoon," "breakup." Each song gets a profile of these terms with weights, often modeled as **term vectors**, the same family of idea as a TF-IDF weighted vector over a vocabulary. This is essentially an **inverted-index style** view: for the term "lo-fi," which songs and how strongly. Two songs that share the same descriptive language end up near each other even if few people have played both yet.

### Pillar 3: Raw audio models (rescue the unknown songs)

Collaborative filtering has a fatal gap called the **cold start problem**. A brand-new song that nobody has played yet has an empty column. No plays means no vector means it can never be recommended, a chicken and egg trap. A small new artist stays invisible.

Spotify's documented fix (Sander Dieleman's NeurIPS 2013 paper "Deep content-based music recommendation," work he continued during his Spotify internship): use a **convolutional neural network** that listens to the raw audio. The input is a mel-spectrogram, a time-by-frequency picture of the sound, in the paper 599 time frames by 128 frequency bins. A 7 or 8 layer CNN reads that picture and predicts what the song's collaborative-filtering vector *would* be if lots of people had played it. So a fresh upload with zero plays still gets a usable vector from its sound alone, and can land in Discover Weekly next to the tracks it sonically resembles.

### The final assembly (largely inference, clearly labeled)

The exact production recipe is not fully public. Based on the three pillars above plus how this class of system is built, the plausible flow is:

1. Generate a large candidate set for Aditi from her neighbors' playlists and her own affinity vectors (Annoy nearest-neighbor lookups). This is the **matching** half: cheaply gather a few thousand plausible songs from tens of millions.
2. Filter out anything she has already heard, anything she explicitly disliked, and apply rules (limit songs per artist, freshness).
3. **Rank** the survivors with a model that scores each candidate, then take the top 30. Matching and ranking are two different halves: matching is "which few thousand are even worth looking at," ranking is "of those, which 30, in what order."
4. Spotify has publicly described BaRT (Bandits for Recommendations as Treatments), a multi-armed-bandit approach used across Home, which balances exploitation (songs it is confident you will like) against exploration (slightly riskier picks to learn more). It is reasonable to infer similar explore-exploit logic shapes the final list. (Inference, not a confirmed Discover Weekly internal.)

### The scale story at three tiers

- **1,000 songs.** Trivial. You could brute-force compare Aditi's vector against all 1,000 song vectors in milliseconds. No trees, no sharding. A simple array scan and a sort by score is enough.
- **100,000 songs.** Brute force per user is getting heavy once you multiply by millions of users every Monday. Here approximate nearest neighbors starts to earn its keep. You build Annoy trees so each lookup touches a few hundred candidates, not 100,000. What broke at the previous tier: the O(N) scan times user count. What saved it: tree-based candidate pruning.
- **10 million plus songs and hundreds of millions of users.** Now everything is a batch and distributed problem. The matrix factorization runs as a large offline job (Spotify historically ran this on Hadoop and Spark across a cluster). The Annoy index is built once, memory-mapped, and shared. Playlists are precomputed in big weekly batches and cached, not generated live when you tap. What breaks here: you cannot recompute on demand, you cannot fit the matrix in one machine, and live nearest-neighbor over 30 million vectors per request is impossible. What survives it: sparse storage, offline batch factorization, approximate nearest neighbors, precompute-and-cache, and sharding the work across many machines.

A blunt honesty note: the precise current architecture (which models, what cadence, exact infra) is not all published. The pillars, Annoy, the CNN audio model, and the matrix factorization are documented facts. The candidate-to-rank assembly and the bandit blend are well-grounded inference about how this class of system is built.

## 8. The retention and habit mechanic

The genius is not only the algorithm. It is the **Monday cadence**.

Discover Weekly refreshes every Monday and stays frozen for a week. That single design choice manufactures a habit loop:

- **Trigger:** it is Monday, there is new music waiting, just for you.
- **Action:** open the app, tap the playlist.
- **Reward:** a couple of genuinely good new songs (variable reward, the strongest kind, because not every track lands and the hits feel earned).
- **Investment:** you save the ones you love and skip the ones you do not, which makes next Monday's list better and pulls you back to find out.

Scarcity drives it too. The list is finite and expires. You have a week before it changes, which nudges you to come listen before you "lose" it. The "Made For You" framing and your photo on the cover make it feel personal and a little flattering.

Which metric it moves: primarily **retention**, with a strong assist to **engagement**. It is a recurring reason to open the app on a slow day, and it deepens the relationship by proving Spotify "gets" you. Discover Weekly became one of Spotify's most powerful retention surfaces after its 2015 launch, reaching tens of millions of listeners within its first years.

Compare the same trick elsewhere: Swiggy and Zomato rotate festival animations and home-screen category nudges so the app feels alive on every open. Same family of mechanic, a fresh-on-arrival reward that rewards the habit of opening.

## 9. The lesson for Rare.lab

Rare.lab is a node-based shader and visual-effects editor that compiles to shippable code, plus an embeddable runtime. The Discover Weekly lesson that maps hardest to your scalability goals: **precompute and memory-map your heavy assets so the live path is a cheap lookup, never a fresh computation.**

Concretely. When a creator builds a shader graph, do not recompile or re-resolve the whole node tree on every preview frame or every embed load. Borrow Annoy's pattern:

- Compile the graph once into a static, read-only artifact (the shipped code plus a packed asset blob: compiled shader variants, lookup textures, baked constant buffers).
- Make that artifact **memory-mappable**, so many runtime instances on one machine or one GPU context share a single copy instead of each re-parsing and re-uploading. This is exactly Annoy's "build the index once, mmap it, fan out reads" move, applied to shader artifacts.
- Split your pipeline the way they split matching from ranking: an expensive **offline/author-time** stage (graph compile, variant generation, optimization) and a cheap **runtime** stage (bind the prebuilt artifact and draw). Keep anything O(graph size) out of the per-frame hot path.

Second, smaller lesson on the cold start problem: when a brand-new effect or template has no usage data to recommend it, do not let it stay invisible. Generate a content-based descriptor from the graph itself (node types, output look, a rendered thumbnail embedding), the same way Spotify's CNN predicts a vector from raw audio. That lets a fresh shader surface in "you might like" before anyone has used it.

The through-line: keep the expensive thinking offline and static, keep the live path a thin shared read. That is what lets one weekly batch serve hundreds of millions, and it is what will let one compiled shader serve thousands of embeds without melting.

---

## Sources

- Sophia Ciocca, "How Does Spotify Know You So Well?" (the three-pillar explainer, Medium / HackerNoon): https://medium.com/@sophiaciocca/spotifys-discover-weekly-how-machine-learning-finds-your-new-music-19a41ab76efe
- Spotify Annoy library, official GitHub repo (random-projection forest, mmap, distance metrics): https://github.com/spotify/annoy
- Erik Bernhardsson, "Annoy" (origin and design of the ANN library): https://erikbern.com/2013/04/12/annoy.html
- Erik Bernhardsson, ANN benchmarks: https://erikbern.com/2018/06/17/new-approximate-nearest-neighbor-benchmarks.html
- Aaron van den Oord, Sander Dieleman, Benjamin Schrauwen, "Deep content-based music recommendation," NeurIPS 2013 (CNN on audio, cold start, mel-spectrogram 599x128): http://papers.neurips.cc/paper/5004-deep-content-based-music-recommendation.pdf
- Sander Dieleman, "Recommending music on Spotify with deep learning": https://sander.ai/2014/08/05/spotify-cnns.html
- Chris Johnson (Spotify), "Algorithmic Music Recommendations at Spotify" (matrix factorization, ANN, Hadoop): https://www.slideshare.net/MrChrisJohnson/algorithmic-music-recommendations-at-spotify
