# References: Amazon shopping cart and the Dynamo store

Keeper links for the 2026-07-28 teardown of the Amazon shopping cart and the
always-available key-value store (Dynamo / DynamoDB) underneath it.

## Primary sources

- Dynamo: Amazon's Highly Available Key-value Store (DeCandia et al., SOSP 2007). The paper opens with the shopping cart as its motivating example and defines consistent hashing, virtual nodes, N/R/W quorums, vector clocks, hinted handoff, and Merkle-tree anti-entropy: https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
- Werner Vogels, Amazon's Dynamo (blog post announcing the paper, plain-language framing of the always-writeable choice): https://www.allthingsdistributed.com/2007/10/amazons_dynamo.html
- Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service (Elhemali et al., USENIX ATC 2022). The modern managed evolution of Dynamo: https://www.usenix.org/conference/atc22/presentation/elhemali
- Consistent Hashing and Random Trees (Karger et al., STOC 1997). The original consistent-hashing ring the whole scheme is built on: https://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf

## Scale numbers

- AWS News Blog, AWS services scale to new heights for Prime Day 2025 (DynamoDB peaked at 151 million requests per second at single-digit millisecond latency): https://aws.amazon.com/blogs/aws/aws-services-scale-to-new-heights-for-prime-day-2025-key-metrics-and-milestones/
- AWS News Blog, Prime Day 2023 numbers (DynamoDB peaked at 126 million requests per second): https://aws.amazon.com/blogs/aws/prime-day-2023-powered-by-aws-all-the-numbers/
- Amazon DynamoDB developer guide, What is DynamoDB (single-digit millisecond performance at any scale): https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html

## Deep dives worth keeping

- How Amazon Scaled E-commerce Shopping Cart Data Infrastructure (System Design newsletter, walks the Dynamo cart design end to end): https://newsletter.systemdesign.one/p/amazon-dynamo-architecture
- ByteByteGo, A Deep Dive into Amazon DynamoDB Architecture: https://blog.bytebytego.com/p/a-deep-dive-into-amazon-dynamodb

## Key facts pulled for the teardown

- The cart is one key (user id) to one small value (the item list), so a key-value store beats a relational table: the only ops are get(user) and put(user, cart).
- Consistent hashing touches roughly 1/N of keys when a machine joins or leaves, versus hash-mod-N which reshuffles nearly everything.
- Virtual nodes (each physical machine placed on the ring 100 to 200 times) even out load and scatter a dead machine's slices across many neighbours.
- N = 3 replicas on a preference list across different racks/power domains; any of the three can accept a write, which is the root of always-writeable.
- R + W > N gives read-your-writes freshness; the cart favours write availability (W low) and heals the gap afterward.
- Vector clocks are (machine, counter) lists that detect when two cart versions branched instead of guessing by wall-clock timestamp.
- Conflict resolution for the cart is the union of items, so a deleted item can reappear; Amazon chose that on purpose because a reappearing item is a mild re-delete but a vanished item is a lost sale and broken trust.
- Hinted handoff plus Merkle-tree anti-entropy keep divergence rare and short-lived.
- Honest caveat: the 2007 Dynamo paper describes the confirmed original internal system; the 2012 managed DynamoDB is an evolution that leans on server-side conflict handling and Multi-Paxos and does not push client-side vector clocks by default.
