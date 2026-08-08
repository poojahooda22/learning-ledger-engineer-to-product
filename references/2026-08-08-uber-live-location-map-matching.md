# References: Uber live driver location, map matching, smooth marker

Saved 2026-08-08 for the teardown on the car moving on the rider's map.

## Primary / canonical

- Paul Newson and John Krumm, "Hidden Markov Map Matching Through Noise and Sparseness," ACM SIGSPATIAL GIS 2009 (Microsoft Research). The canonical HMM + Viterbi map-matching method. Emission probability = Gaussian on the great-circle distance from the GPS point to the candidate road segment; measured GPS noise standard deviation about 4 meters. Transition probability = exponential on the difference between the straight-line distance between two GPS points and the driving-route distance between their snapped points; widely used implementations default the decay beta to about 3. Viterbi finds the single most likely whole sequence of segments. Accuracy holds up to roughly 30-second sampling, then degrades.
  https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/map-matching-ACM-GIS-camera-ready.pdf

- Uber Engineering, "Improving Uber's Mapping Accuracy with CatchME." Uber snaps trip GPS traces to the road with an HMM, treats aggregated trip GPS as ground truth, and flags map errors when matched routes disagree with the map.
  https://www.uber.com/en-IN/blog/mapping-accuracy-with-catchme/

- Uber Developer Documentation, driver-location webhook reference. States driver location webhooks are sent at a 4-second interval by default. Primary-source confirmation of the ping cadence.
  https://developer.uber.com/docs/guest-rides/references/api/webhooks/driver-location

## Secondary / class-of-problem

- Marie Douriez, "A New Real-Time Map-Matching Algorithm at Lyft," Lyft Engineering. Online HMM map matching in production; the online-matcher vs offline-reprocessing split.
  https://eng.lyft.com/a-new-real-time-map-matching-algorithm-at-lyft-da593ab7b006

- Or Nachmias, "Map Matching: The Viterbi Snapper," Gett Engineering. Viterbi choosing the single most likely sequence of road edges that jointly explains a whole GPS trace.
  https://medium.com/gett-engineering/map-matching-the-viterbi-snapper-e32d11f0d130

- "Designing Uber," High Scalability. Real-time dispatch and location architecture: Ringpop consistent hashing, geo-sharding, persistent connections, DISCO.
  https://highscalability.com/designing-uber/

- Ndriqim Muhadri, "The Tech Behind Uber's Smooth Real-Time Map Experience," Medium. Client-side interpolation (tweening over 300 to 500 ms with requestAnimationFrame) and dead reckoning between sparse GPS pings.
  https://medium.com/@ndriqim.muhadri99/the-tech-behind-ubers-smooth-real-time-map-experience-55f4cccee2b2

## Cross-links inside this ledger

- 2026-06-14 Uber surge pricing: the same "turn where into a shard key" idea (H3 hexes / geographic partitioning).
- 2026-07-02 Uber batched matching (DISCO): dispatch, the step before this one.
- 2026-07-15 Uber ETA prediction (DeeprETA): the number, this report is the moving dot.
- 2026-06-13 Spotify Discover Weekly: keep the expensive thinking off the live path.
