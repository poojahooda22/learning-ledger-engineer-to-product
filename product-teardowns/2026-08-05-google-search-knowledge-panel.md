# Google Search: the Knowledge Panel and the Knowledge Graph behind it (things, not strings)

Date: 2026-08-05
Product: Google Search
Feature: The Knowledge Panel (the fact box on the right) and the Knowledge Graph that fills it

---

## 1. The user

Ananya is on the sofa at 9pm, phone in hand, half watching a documentary about Paris.
The narrator says the Eiffel Tower is taller than she thought. She thumbs open Google and
types "how tall is the eiffel tower". She does not want ten blue links. She does not want to
open a website, wait for it to load, dodge a cookie banner, and scroll for the one number.
She wants the number. Now.

Or it is Rohit, a cricket fan, who hears a commentator mention Virat Kohli's age and does not
believe it. He types "virat kohli" and expects to instantly see: born 5 November 1988, the
teams he plays for, his height, a photo. One glance, done.

Both of them are asking about a real thing in the world (a monument, a person), not hunting for
a document. That is the whole story of this feature.

## 2. The real problem

Classic search is a document finder. You type words, it finds pages that contain those words,
it ranks them. That is perfect when you want "best noise cancelling headphones under 15000" and
you want to read reviews. It is a terrible fit when you want a single fact about a single thing.

Told like a friend: the old way made you do the last mile of work yourself. Google would hand
you a page that probably had the Eiffel Tower's height buried in it, and you had to go dig it
out. Worse, words are ambiguous. Type "jaguar" and the poor document engine has no idea if you
mean the animal, the car company, the Jacksonville NFL team, or the old Mac OS X release named
Jaguar. It just matches the letters j-a-g-u-a-r against a giant pile of text. It matches strings.
It does not understand things.

The real problem is that a string of characters is not a concept. "Eiffel Tower", "la tour
Eiffel", and "that iron tower in Paris" are three different strings that point at one real
object. And one string, "jaguar", points at many different real objects. A search engine that
only knows letters can never close that gap.

## 3. The feature in one sentence

The Knowledge Panel is the boxed summary of facts about one real world entity, assembled live
from Google's Knowledge Graph, a giant database that stores things (people, places, movies,
animals) and the relationships between them instead of just web pages.

## 4. Jobs to be done

- "Give me this one fact right now" (the height, the birthday, the CEO, the capital), no click.
- "Confirm you understood which thing I meant" (the jaguar the animal, not the car).
- "Show me the shape of this thing at a glance" (photo, a few headline facts, who is related to it).
- "Let me jump to the next related thing" (from the Eiffel Tower to its architect to Paris to France)
  without typing a new full query.
- Underneath all of them: "answer me without making me leave and read a webpage."

## 5. How it works for the user

Ananya types "how tall is the eiffel tower". Before she even finishes, a box appears. Not a list
of links. A card that says 330 metres (1,083 feet), with a photo of the tower, the architect
Gustave Eiffel, the year it opened (1889), and a short line about where it is (Champ de Mars,
Paris). She reads the number and locks her phone. She never clicked a single website.

Rohit types "virat kohli". On the right of the desktop results, or as a card on top on mobile,
a panel shows his photo, "Born: 5 November 1988 (age 37), Delhi", his height, his current teams,
and a row of related people ("People also search for") with faces he recognises. He taps one of
those faces and a new panel loads for that player. He is now browsing the graph, hopping from
thing to thing, and he never typed a second query in full.

When Ananya's daughter later types just "jaguar", Google has to make a choice. It shows the
animal panel (because most people who type the bare word want the cat), but it also offers a way
to switch to the car. The system picked the most likely thing, and admitted there were others.

## 6. The actual flow, step by step

1. Ananya types "how tall is the eiffel tower".
2. Google parses the query and spots that it is about a specific real world entity (the Eiffel
   Tower) and a specific attribute (height). This is the "understand the query" step.
3. It resolves the words "eiffel tower" to a single entity ID in the Knowledge Graph. Not the
   text. An ID. In the old Freebase world that ID looked like /m/02j81w. Today think of it as a
   stable machine handle, the same idea as Wikidata's Q243 for the Eiffel Tower.
4. With the ID in hand, it looks up that entity's stored facts (its property bag): height,
   architect, opening date, location, image, related entities.
5. It picks the facts worth showing for this query (height first, because she asked for height),
   assembles the panel server side, and ships a small block of structured data to her phone.
6. Her phone renders the card. All the thinking already happened on Google's servers. The phone
   just draws the box.
7. She taps nothing, or she taps "Gustave Eiffel" and step 3 to 6 repeat for a new entity, a
   graph hop, no full re typing.

The whole thing feels instant because almost none of the work happens at the moment she types.
It was all precomputed and stored, waiting to be looked up by key.

## 7. Under the hood, like the engineer

This is the heart of the report. The Knowledge Panel is search wearing a completely different
skeleton from the inverted index. The old teardowns in this ledger (Amazon search, Swiggy search,
Google autocomplete) all fetch documents from an inverted index and rank them. The Knowledge
Graph does not store documents at all. It stores a graph of facts. So the data structure is the
whole story.

### The core data structure: a triple is an edge in a labeled graph

The atom of the Knowledge Graph is the triple: (subject, predicate, object). Three parts. For
example:

- (Eiffel Tower, height, 330 metres)
- (Eiffel Tower, architect, Gustave Eiffel)
- (Eiffel Tower, located in, Paris)
- (Paris, capital of, France)
- (Gustave Eiffel, also designed, internal frame of the Statue of Liberty)

Read those again and notice something. Each triple is an edge in a directed graph. The subject
and object are nodes. The predicate is the label on the arrow between them. "Eiffel Tower" is a
node. "Paris" is a node. "located in" is the arrow. So the entire Knowledge Graph is one huge
directed labeled graph: nodes are things, edges are the relationships. This is exactly why Google
called it a graph and not a table.

The nodes are the deep idea. A node is not the text "Eiffel Tower". It is a stable ID, and the
text is just one of its many labels ("Eiffel Tower", "la tour Eiffel", "Tour Eiffel"). That is
what Google meant by the 2012 slogan "things, not strings" (Google, "Introducing the Knowledge
Graph: things, not strings", 16 May 2012). The thing is the node with the ID. The strings are
just aliases hanging off it. Once you store the thing as an ID, "Eiffel Tower" and "la tour
Eiffel" collapse into the same node, and "jaguar" the animal and "jaguar" the car become two
different nodes that happen to share a label.

### Why a graph and not a relational table

You could try to store this in a normal SQL table: one row per fact. It works at small size. It
falls apart on the query that makes the feature magic: "show me things related to this thing, and
things related to those." That is a multi hop traversal. In a relational table each hop is a JOIN,
and JOINs across hundreds of billions of rows get brutal fast (this ledger's Lesson 46 is
literally about how painful distributed joins are). In a graph, a hop is just following an edge.

Concrete: "who else did the Eiffel Tower's architect design?" In a graph you go Eiffel Tower to
(architect) to Gustave Eiffel to (designed) to his other works. Two hops, two edge follows. The
"People also search for" row and the "related entities" strip are exactly these one and two hop
neighbours of the current node. A graph makes neighbours cheap. A table makes them a join storm.

### Storage: the property bag as a keyed lookup

Here is where it connects to the pattern this whole ledger keeps finding. You do not walk the
graph live when Ananya types. You precompute, for each entity, its adjacency list: the full set
of (predicate, object) pairs pointing out of that node, plus its top related nodes. You store that
as a value keyed by the entity ID, in a key value store or a graph store that behaves like one.

So the live operation is: entity ID goes in, one property bag comes out. That is a hash map
lookup, O(1) in the size of the graph. "Everything about the Eiffel Tower" is one keyed read of a
precomputed blob, not a live crawl over 500 billion facts. Offline think, online lookup, again,
the same spine as Discover Weekly, YouTube recommendations, and every other feature in this ledger.
The expensive part (building and cleaning the graph) is a batch job. The live part is a lookup.

### The two halves: matching (entity linking) then assembly

Just like document search splits into matching then ranking, the panel splits into two halves, and
the first half is the hard, interesting one.

Half one is matching, but here matching means entity linking: turn the ambiguous query string into
the one correct entity ID. This is a two step process, well studied in the literature (see the
entity linking survey work and Wikipedia's "Entity linking" article).

- Step 1, candidate generation. Take the mention "jaguar" and look it up in an alias dictionary, a
  precomputed map from surface strings to every entity that has ever gone by that string. "jaguar"
  returns a candidate set: the animal (Panthera onca), Jaguar Cars, the Jacksonville Jaguars, the
  Atari Jaguar console, Mac OS X 10.2 Jaguar, and more. This alias table is itself an inverted
  index, string to list of entity IDs, so the cost tracks the number of candidates, not the 5
  billion entity catalogue. Same trick as every posting list in this ledger.
- Step 2, disambiguation (ranking the candidates). Now pick the right one. Signals: prior
  popularity (bare "jaguar" most often means the animal, so it wins by default), the rest of the
  query ("jaguar price" leans car, "jaguar habitat" leans animal), the user's context and location,
  and recent news spikes. The candidate with the best combined score wins, and if nothing clears a
  confidence bar, Google shows no panel at all and falls back to plain links. That silence is the
  same confidence gate as Gmail Smart Compose and Google's own "Did you mean": only speak when sure.

Half two is assembly. With the winning entity ID, read its property bag, choose which facts fit the
query (height first for a height question), choose the image, choose the related entities row, and
build the card. This is cheap. It is a lookup plus a small ranking over a handful of attributes.

### Where the graph comes from: Freebase, Wikidata, and harvesting the web

The graph did not appear from nowhere, and its history is a lesson in itself. Google bought Metaweb
in July 2010 (announced 16 July 2010). Metaweb ran Freebase, a community built open graph of
entities whose final data dump held about 1.9 billion RDF triples. That seeded the Knowledge Graph,
which launched on 16 May 2012 with about 500 million objects and more than 3.5 billion facts, drawing
from Freebase, Wikipedia, and the CIA World Factbook among others. Google later wound Freebase down
(shutdown announced 16 December 2014, service closed 2 May 2016) and helped migrate its data toward
Wikidata, the open graph that now backs a lot of public entity data.

By May 2020, Google said the Knowledge Graph had grown to more than 500 billion facts about 5 billion
entities (Google Search blog, "How Google's Knowledge Graph works", 2020). That is the scale the
serving system has to survive.

Curated sources only get you so far. The web has facts that no one has typed into Wikidata. So Google
built the Knowledge Vault (Dong et al., "Knowledge Vault: A Web-Scale Approach to Probabilistic
Knowledge Fusion", KDD 2014) to harvest facts automatically. Two ideas from that paper matter here:

- Multiple extractors read the messy web in different ways: free text, HTML DOM trees, HTML tables,
  and human annotations. Each proposes candidate triples like (Barack Obama, born in, Honolulu).
- Every extracted fact gets a calibrated probability of being true, computed by fusing the noisy
  extractors with graph based priors. The prior asks: given everything else the graph already knows,
  how plausible is this new edge? One method named in that line of work is the Path Ranking Algorithm,
  which learns that certain paths in the graph predict certain edges (if X was born in a city that is
  in country Y, X's nationality is probably Y). So the graph helps guess its own missing edges, and a
  shaky fact with low probability never gets shown to Ananya.

This is the same asymmetric caution as the rest of the ledger: a wrong fact in a Knowledge Panel is
far more damaging than a missing one, so the confidence threshold guards the panel like a bouncer.

### Entity resolution: the dedup problem

One real thing shows up in many sources. Wikipedia has "Barack Obama". Freebase had a node for him.
A government dataset has him. If you naively imported all three you would get three nodes for one man,
and his facts would scatter. So a huge offline job called entity resolution (or reconciliation) decides
"these three records are the same real person" and merges them into one node with one ID. This is a
dedup and clustering problem: compare records, score how likely two are the same, union the matches.
Get it wrong one way and you split one person into two half empty panels. Get it wrong the other way
and you merge two different people named John Smith into one nonsense panel. This merge step is quietly
where a lot of the quality lives.

### The scale story at three tiers

Tier one, 1,000 entities. Imagine a single company's internal "who and what" graph: 1,000 people,
teams, and projects. Store it as one table or one in memory map. A couple of JOINs answer any query.
No ambiguity worth worrying about, no sharding, no confidence scores. Building a graph database here
would be over engineering. A spreadsheet almost does it.

Tier two, 100,000 entities. Now think of every notable monument, or every listed company in a country.
Two things break. First, ambiguity becomes real: many entities share names ("Cambridge" the UK city vs
Cambridge Massachusetts vs Cambridge the company), so you now need the alias dictionary and the
disambiguation step, not just a name lookup. Second, freshness and duplicates bite: the same entity
arrives from three feeds, so you need entity resolution to merge them. A relational store with good
indexes on the alias table still holds at this tier, and one machine can keep the whole graph in memory.

Tier three, 5 billion entities and 500 billion facts. Now nothing fits on one machine. The survival
moves are the ledger's usual toolkit:

- Shard the graph by entity ID across many machines, so entity ID routes you to the one shard that
  holds its property bag (same self routing key idea as Notion's workspace shards).
- Replicate hot shards for reads, because reads massively outnumber writes.
- Cache the head entities. When a celebrity dies or a country trends after news, one entity gets a
  flood of lookups. That is the classic hot key or celebrity problem (this ledger's Lesson 16). The
  fix is cache plus replicate that one node, and precompute its panel so the surge hits a cache, not
  a cold graph walk.
- Kill live multi hop traversal on the hot path. A query that crossed shards to walk three hops live
  would be slow and unpredictable. So precompute each entity's property bag and its top related
  entities offline and store them denormalised, so the live request is one keyed read, never a walk.
  The graph is walked in batch, in the background, not while Ananya waits.
- Run the whole Knowledge Vault fusion (extract, score, resolve, merge) as offline batch. The serving
  path only ever reads the finished, cleaned, precomputed result.

What breaks at the jump to each next tier is always the same shape: at 1k nothing breaks, at 100k
ambiguity and duplicates break (fix with alias tables and entity resolution), at billions the single
machine and the live graph walk break (fix with sharding, replication, hot key caching, and
precomputed property bags). The candidate set size in disambiguation is the dial that keeps matching
cheap no matter how big the catalogue grows, exactly like the candidate set in Amazon and Canva search.

## 8. The retention and habit mechanic

The Knowledge Panel's habit loop is the zero click answer. Ananya got her fact without leaving Google.
She did not visit a website, so she did not get distracted by another website. The next time she has a
quick question, the reflex is stronger: just google it, the answer will be right there. That reflex is
the product. The verb "to google" is the retention mechanic made language.

Which metric does it move? Engagement and retention, and defensively the core search habit and the ad
real estate around it. Every fast, correct instant answer deepens trust, and trust is what keeps a user
from ever trying a different search box.

The real observed example, and the controversy: multiple third party studies (for instance analyses by
SparkToro and Rand Fishkin) have reported that a majority of Google searches now end without a click to
any external website, a large share of them satisfied right on the results page by panels, featured
snippets, and similar boxes. Treat the exact percentage as a third party estimate, not a Google confirmed
number, but the direction is not in dispute: the panel keeps people on Google. That is great for the
habit loop and contentious for publishers, whose facts fill the box while their traffic drops.

There is a second, two sided flywheel for businesses and public figures. A restaurant or a person can
claim and correct their own Knowledge Panel (through Google Business Profile and verification). That pulls
entity owners onto Google to curate their own node, which makes the graph richer, which makes the panels
better, which pulls in more users. The users feed the graph with their queries, the owners feed it with
corrections, and Google sits in the middle owning the loop.

## 9. The lesson for Rare.lab

Rare.lab is a node based shader and visual effects editor that compiles to shippable code, plus an
embeddable runtime. The Knowledge Graph is a direct blueprint for how to model your asset and effect
library, and it hardens a lesson this ledger keeps circling.

Model your library as an entity graph of IDs, not a catalogue of strings. Give every shader, node,
texture, preset, and device capability bucket a stable machine ID, and store the relationships between
them as first class edges: (bloom effect, uses, gaussian blur node), (this preset, derives from, that
preset), (this variant, is compatible with, Adreno 7xx GPU family), (this node, samples, that noise
texture). Then "everything about this effect" (its parameters, its dependencies, its compatible device
buckets, its known good compiled variants) becomes one keyed lookup of a precomputed property bag, not a
live crawl over your whole graph. That is the offline think, online lookup spine again, and it is what
keeps the editor responsive when the library grows from 1,000 nodes to millions.

Three concrete moves fall out of the graph model:

1. Resolve references to IDs once, like entity linking. When a user or a template refers to "cinematic
   bloom", link that string to a node ID at author time and store the ID, not the string. Then renaming
   the effect never breaks a single reference, exactly like Notion storing 4 character property keys
   instead of column names. The string is a label on the node, never the pointer.

2. Precompute the neighbour set and the compiled variant bag offline. Do not walk the dependency graph
   live to answer "what does this effect need and where does it run". Batch build each node's property
   bag (dependencies plus per device compiled variants plus compatibility edges) and serve it as a
   single keyed read. Cache the hot presets (the shared "cinematic bloom" everyone imports) because they
   are your celebrity entities and will get the request flood.

3. Gate the instant suggestion behind a confidence threshold. When Rare.lab auto suggests or auto applies
   an effect from a user's rough intent, treat it like Google deciding whether to show a panel: only act
   when the match clears a confidence bar, otherwise stay silent and show plain options. A wrong auto
   applied effect that recompiles the graph and wrecks the frame budget costs far more trust than showing
   nothing. Asymmetric caution, the same bouncer that guards the Knowledge Panel and Smart Compose.

The one line: store your effects library as a graph of stable IDs with first class relationship edges,
precompute each node's property bag so the hot path is a keyed lookup and never a live graph walk, and
let a confidence gate decide when to speak. Things, not strings, applied to shaders.

---

## Sources

- Google, "Introducing the Knowledge Graph: things, not strings" (official blog, 16 May 2012): https://blog.google/products/search/introducing-knowledge-graph-things-not/
- Google Search, "A reintroduction to Google's Knowledge Graph" / "How Google's Knowledge Graph works" (2020, the 500 billion facts about 5 billion entities figure): https://blog.google/products-and-platforms/products/search/about-knowledge-graph-and-knowledge-panels/
- Wikipedia, "Knowledge Graph (Google)" (launch date, initial 500 million objects and 3.5 billion facts, sources, Freebase history): https://en.wikipedia.org/wiki/Knowledge_Graph_(Google)
- Google acquires Metaweb / Freebase (announced 16 July 2010): https://www.seobythesea.com/2010/07/google-gets-smarter-with-named-entities-acquires-metaweb/
- Freebase history, final 1.9 billion RDF triple dump, shutdown (16 December 2014) and migration to Wikidata (closed 2 May 2016): https://en.wikipedia.org/wiki/Freebase_(database)
- Dong, Gabrilovich, Heitz, Horn, Lao, Murphy, Strohmann, Sun, Zhang, "Knowledge Vault: A Web-Scale Approach to Probabilistic Knowledge Fusion" (Google, KDD 2014): https://dl.acm.org/doi/10.1145/2623330.2623623
- Wikipedia, "Entity linking" (candidate generation then disambiguation, the two step process): https://en.wikipedia.org/wiki/Entity_linking
- Wikidata, the open knowledge base that succeeded Freebase (entity IDs such as Q243 for the Eiffel Tower): https://www.wikidata.org/
- SparkToro / Rand Fishkin, zero click search analyses (third party estimates of searches ending without a click): https://sparktoro.com/blog/
