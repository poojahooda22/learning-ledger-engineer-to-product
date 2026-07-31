# Day 44 — How does one GPU serve thousands of simultaneous conversations without wasting most of its memory on empty seats?

*2026-07-31*

---

## 1. The company and the number that breaks a naive design

**Character.AI, 2023, serving tens of millions of chat conversations a day.** Every message a user sends triggers an autoregressive generation: the model reads the whole conversation so far, then produces the reply one token at a time, and every one of those tokens needs the attention "memory" (the Key and Value vectors, the KV cache) of every token that came before it to stay resident in GPU memory for as long as that conversation's request is alive. By mid-2023 Character.AI was serving roughly **20,000 inference queries per second, about 20% of the request volume Google Search handles**, almost entirely through its own inference stack rather than a third-party API (Character.AI Engineering, "Optimizing AI Inference at Character.AI").

The number that breaks a naive server: the UC Berkeley team that built vLLM measured that existing LLM-serving systems, the ones everyone was running in production before mid-2023, **waste 60% to 80% of a GPU's KV-cache memory to fragmentation and over-reservation**, while their PagedAttention design cuts that waste to under 4%. That memory difference translates directly into how many conversations one GPU can hold at once, and it is why vLLM measured **up to 24x higher throughput than Hugging Face Transformers, and up to 3.5x higher than Hugging Face's own TGI server, on the identical GPU** (Kwon et al., "Efficient Memory Management for Large Language Model Serving with PagedAttention," SOSP 2023; vLLM Blog, "vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention"). Character.AI's own optimizations, mostly int8 quantization and a shared prefix cache, cut their inference cost per query by roughly 33x over two years while holding quality steady (Character.AI Engineering, "Optimizing AI Inference at Character.AI" and "Part Deux"). The breaking number in one line: a GPU that has enough raw compute to serve dozens of concurrent conversations instead serves four or five, because the naive way of holding "memory of the conversation so far" for each request wastes most of the GPU's memory on empty, unusable space.

## 2. Why the naive (demo) design dies

The demo version of "serve an LLM to more than one user" is a for-loop: accept a request, run the model forward token by token until it produces an end-of-sequence marker, return the answer, then move to the next request. This dies for three concrete reasons.

**a. One-request-at-a-time starves the GPU.** A GPU is built to do thousands of matrix multiplications in parallel. Generating one token for one conversation barely uses a fraction of that parallelism, the same way running one truck at a time down an eight-lane highway wastes seven lanes. Serving requests sequentially leaves nearly all of the GPU's compute idle, so throughput per GPU-dollar is terrible even before memory becomes the problem.

**b. Static batching dies on head-of-line blocking.** The obvious fix is to batch requests together: collect N requests, run them through the model together as one batch, return all N answers when the batch finishes. This uses the GPU's parallelism properly, but conversations do not all take the same number of tokens to finish. One user asks for a one-line answer (20 tokens), another asks for a summary (2,000 tokens). With static batching, the whole batch is stuck running until the *longest* request in it finishes, because you cannot pull one row out of a matrix multiply mid-flight without restructuring the whole computation. A short request that finished in the first few steps sits there, done, silently wasting its GPU-cycles as padding, while a brand-new incoming request cannot join the batch until every seat in it is empty. This is the FasterTransformer-era baseline that Orca (Seoul National University and FriendliAI, OSDI 2022) measured against, and it is why Orca's iteration-level scheduling beat it by up to **36.9x throughput at equivalent latency**. The analogy: a restaurant that seats parties only in fixed groups and won't clear or refill any single table until every table in that group's booking has finished its multi-course meal, even though half the tables just wanted a quick coffee.

**c. Naive KV-cache allocation dies on fragmentation.** Every token generated needs its Key and Value attention vectors kept in GPU memory for the rest of that request's life; for a 7-billion-parameter model in 16-bit precision, a single sequence of 4,096 tokens needs roughly **2 GB of KV cache** on its own (2 bytes x layers x hidden size x 2 for K and V x sequence length). A naive server does not know in advance how long a conversation will run, so it reserves memory for the *worst case*, the maximum context length, as one contiguous block per request, the moment the request starts. That is like a hotel that books every guest for the maximum possible 30-night stay because it does not know in advance if they'll check out tomorrow, so the building looks fully booked while 80% of those reserved room-nights sit empty. Most real conversations are far shorter than the max, so most of that reserved memory never gets used, but it also can't be handed to anyone else, because it is one contiguous allocation. The GPU appears "out of memory" for new requests long before it is actually out of usable memory. That is the literal source of the 60-80% waste PagedAttention measured.

## 3. The architecture, drawn top to bottom

```
CLIENTS (chat apps, API callers, agent frameworks)
   send a prompt, stream back tokens as they're generated
   |
   v
API GATEWAY / ROUTER
   auth, rate limiting, routes to the right model + region
   analogy: the maitre d' who checks the reservation and
   points you to the right dining room
   |
   v
CONTINUOUS-BATCHING SCHEDULER (Orca's "iteration-level scheduling")
   decides, at EVERY single decode step (not just at request
   arrival), which requests are in this step's batch: finished
   requests leave immediately, freeing a seat; queued requests
   join immediately, filling it
   analogy: a shuttle van that lets a passenger off and picks
   a new one up at every stop, instead of a bus that must
   finish its whole scheduled route before anyone new boards
   |
   v
   +------------------------+------------------------+
   |                                                  |
PREFILL POOL                                    DECODE POOL
compute-bound: reads the whole prompt            memory-bandwidth-bound:
in one parallel forward pass, builds the         generates one token at a
initial KV cache for every prompt token          time, re-reading the whole
at once (fast, GPU-compute-heavy)                growing KV cache each step
   |                                                  |
   +------------------ KV cache handed off -----------+
   over the network (RDMA), so prefill and decode can be
   sized and scaled independently instead of forcing one
   GPU type to be good at two opposite workloads
   (Mooncake / Kimi's disaggregated design)
                       |
                       v
        PAGEDATTENTION KV-CACHE MEMORY MANAGER
        splits each request's KV cache into small FIXED-SIZE
        pages instead of one big contiguous block, keeps a
        per-request page table mapping logical token position
        to physical GPU memory page (borrowed directly from
        OS virtual memory)
        analogy: instead of reserving one giant reserved
        parking lot per car (most of it empty), hand out
        parking spots one at a time from a shared pool, and
        keep a ticket that remembers which spots are yours
                       |
                       v
        PREFIX CACHE (shared KV cache for identical prefixes)
        a system prompt, persona description, or few-shot
        example computed ONCE and reused by every request
        that starts with the same tokens; Character.AI's
        LRU, tree-structured cache hits ~95% of the time
        analogy: a call-center script the agent already has
        memorized, not re-read from scratch on every call
                       |
                       v
        TIERED KV-CACHE STORAGE (Mooncake Store)
        GPU HBM (fastest, smallest, most expensive) is the
        top tier; when it's full, cold KV cache spills to
        CPU RAM, then local SSD, then even another machine
        over the network, rather than being discarded and
        recomputed from scratch
                       |
                       v
        RESPONSE STREAM back to the client, one token at a
        time, as each decode step produces it
```

## 4. The transferable mechanisms

- **Iteration-level (continuous) scheduling instead of request-level batching.** The scheduler decides batch membership at every single generation step, not once when a batch is formed. A finished sequence exits and frees its slot the instant it's done; a queued sequence enters the very next step instead of waiting for the whole batch to drain. This is the single biggest lever: Orca measured up to 36.9x throughput over static batching at the same latency, and it generalizes to any pipeline where work items have wildly different completion times, don't force them to travel in lockstep.

- **Virtual-memory-style paging for a scarce, variable-size resource.** PagedAttention takes the exact trick operating systems have used since the 1960s, paging, and applies it to GPU memory: split a growable allocation into small fixed-size blocks, keep an indirection table (a page table) instead of requiring contiguity, and eliminate both internal fragmentation (reserved-but-unused space inside a block) and external fragmentation (unusable gaps between blocks). Anywhere a system pre-allocates for a worst case it rarely hits, this pattern applies.

- **Disaggregate stages with opposite resource profiles.** Prefill (read the whole prompt, one big parallel matrix multiply) is compute-bound. Decode (generate one token, re-read a growing cache) is memory-bandwidth-bound. Mooncake's architecture, which serves Kimi's production traffic across thousands of nodes and over 100 billion tokens a day, gives each phase its own independently-sized pool of GPUs instead of forcing one generic worker type to be good at both (Qin et al., "Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving," FAST 2025). The general lesson: when one pipeline has two stages that are bottlenecked on different resources, splitting them lets you buy exactly the capacity each stage needs, not a single averaged compromise.

- **Cache the expensive, exact-match part; recompute is the fallback, not the default.** A shared system prompt or persona is computed once and reused by every request that shares that exact prefix. Character.AI's tree-structured LRU cache hits about 95% of the time, meaning 95% of requests skip re-computing attention over a prefix they share with someone else. This is the same principle as HTTP caching or memoization, applied to a model's internal activations rather than a function's return value.

- **Choose the numeric representation deliberately, and match it to how you trained.** Character.AI trains its models natively in int8 rather than training in a high-precision format and quantizing afterward. That single choice avoids a train/serve mismatch (a model that behaves subtly differently at serving precision than it did during training) and cuts KV-cache size by more than 20x, because the cache stores one int8 byte instead of two bf16 bytes per value, and that memory saving is what lets more conversations fit on one GPU at once.

- **Tiered storage for cold state instead of a hard eviction cliff.** When the fastest, smallest, most expensive memory tier fills up, spill to the next tier down (GPU HBM to CPU RAM to SSD to network) rather than deleting and recomputing. Mooncake calls this explicitly "trading more storage for less computation": storage is cheap and computation is not, so keep the state around as long as you can afford to.

## 5. The trade-offs

- **Aggregate throughput vs. any single request's latency.** Continuous batching lets a scheduler insert a brand-new prefill mid-stream among ongoing decode steps, which raises overall GPU utilization but can momentarily slow the in-flight decodes that were already running, because that prefill briefly competes for the same GPU cycles. Production schedulers cap how much prefill work can be injected per step (a token or compute budget) specifically to protect decode latency. More throughput per GPU and a tighter latency guarantee for every individual request pull in opposite directions, and the budget is a dial, not a solved problem.

- **Extra memory-management complexity vs. the cost it avoids.** Page tables, per-block bookkeeping, and prefix-cache eviction policies are real engineering overhead. The alternative, recomputing an attention state from scratch on every cache miss, costs a full forward pass over however many tokens are missing, which is dramatically more expensive than an O(1) cache lookup. The complexity buys back GPU-hours that would otherwise be spent redoing work.

- **Correctness of the cache is exact-match, not eventually-consistent.** Unlike Day 43's Pinot segments, where a query that's off by a few recent events is an acceptable trade for availability, a KV-cache entry is only valid if the token sequence, model version, and decoding parameters match exactly. One different token anywhere in the shared prefix invalidates the cache entry outright; there is no "close enough" here, because a wrong cached attention state produces a wrong answer, not a stale count.

- **Disaggregation buys right-sized pools at the cost of a network hop.** Splitting prefill and decode into separate GPU pools means every request's KV cache has to move across the network (over RDMA) from the machine that computed it to the machine that will generate from it. That's real latency and bandwidth cost, accepted because it prevents a burst of large prompts from stealing GPU cycles away from users who are already mid-conversation and waiting on their next token.

## 6. The systems-thinking lens

The feedback loop here is a **metastable failure triggered by prefill-decode interference**: a burst of new, long prompts arrives, so the scheduler admits more compute-heavy prefill work into the batch to keep up. That prefill work steals GPU cycles from in-flight decode steps, which slows down every conversation already running. Those slowed-down conversations stay resident in the KV cache longer than they would have, which eats more GPU memory, which shrinks how large the next batch can be, which lowers overall throughput, right when demand is at its highest. Unlike a simple overload, this state doesn't recover on its own once the burst passes, because the pile-up of half-finished, memory-resident requests is now the bottleneck, not the original traffic spike. That is exactly the shape of a metastable failure: a trigger that comes and goes, but a self-reinforcing internal state that persists after it.

The senior fix does not mean buying more GPUs generically, it means breaking the specific coupling that let one workload steal from the other:

- **Disaggregating prefill and decode into separate pools** so a burst of new prompts physically cannot take GPU cycles away from users who are mid-generation; the two workloads no longer share a resource that one can starve the other of.
- **An explicit per-step token or compute budget in the scheduler** (admission control), so the batch can never overcommit past what the KV-cache memory can actually hold, the same principle as Day 13's backpressure: refuse to accept work you cannot promise to finish, rather than accepting it and stalling everyone already in the system.
- **Queueing and shedding new requests at the edge with a visible signal** (a queue-depth response or an explicit "try again shortly") instead of silently accepting unbounded work and letting the failure surface later as every conversation slowing down at once.

The general lesson, the same one Day 13 teaches from the angle of overload and Day 43 teaches from the angle of scatter-gather fan-out: whenever two workloads with different resource profiles are forced to share one undifferentiated pool, a spike in one degrades the other, and the fix is architectural separation and admission control, not raw capacity thrown at the symptom.

---

## Sources

- Kwon et al. (UC Berkeley), ["Efficient Memory Management for Large Language Model Serving with PagedAttention"](https://arxiv.org/abs/2309.06180) (SOSP 2023)
- vLLM Project Blog, ["vLLM: Easy, Fast, and Cheap LLM Serving with PagedAttention"](https://blog.vllm.ai/2023/06/20/vllm.html)
- Yu et al. (Seoul National University / FriendliAI), ["Orca: A Distributed Serving System for Transformer-Based Generative Models"](https://www.usenix.org/conference/osdi22/presentation/yu) (OSDI 2022)
- FriendliAI, ["Iteration Batching (a.k.a. Continuous Batching): Accelerate LLM Inference Serving with Flexible Scheduling"](https://friendli.ai/blog/llm-iteration-batching)
- Character.AI Engineering, ["Optimizing AI Inference at Character.AI"](https://blog.character.ai/optimizing-ai-inference-at-character-ai/)
- Character.AI Engineering, ["Optimizing AI Inference at Character.AI (Part Deux)"](https://blog.character.ai/optimizing-ai-inference-at-character-ai-part-deux-2/)
- Qin et al. (Moonshot AI / Tsinghua), ["Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving"](https://arxiv.org/abs/2407.00079) (FAST 2025)
- Mooncake Project, [GitHub README](https://github.com/kvcache-ai/Mooncake/blob/main/README.md)
- Anyscale, ["Achieve 23x LLM Inference Throughput & Reduce p50 Latency"](https://www.anyscale.com/blog/continuous-batching-llm-inference)
- NVIDIA Technical Blog, ["Mastering LLM Techniques: Inference Optimization"](https://developer.nvidia.com/blog/mastering-llm-techniques-inference-optimization/)
