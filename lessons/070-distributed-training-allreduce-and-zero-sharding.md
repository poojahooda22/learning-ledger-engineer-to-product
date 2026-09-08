# Day 70 — How do you average gradients across 16,384 GPUs every single training step without one machine's network card becoming the bottleneck, and without keeping a spare copy of a 700GB optimizer state on every one of those cards?

**Date:** 2026-09-08
**Difficulty:** Expert
**Topic:** Distributed model training at GPU-fleet scale: ring all-reduce for gradient synchronization (Baidu SVAIL, 2017) and ZeRO memory sharding (Microsoft DeepSpeed, 2020). Two separate walls show up the moment you try to train one model across more than a handful of GPUs: a **communication** wall (someone has to average everyone's gradients every step, and the naive way to do that gets slower, not faster, as you add machines) and a **memory** wall (the optimizer's own bookkeeping, not the model, is what runs a GPU out of room first). This ledger has already built the individual pieces these two fixes assemble, aimed at a different unit each time: Day 10 covered a ring topology spreading *data* evenly so no one node is overloaded (consistent hashing); this lesson reuses the same ring shape to spread *network traffic* evenly instead. Day 13 covered breaking a retry-amplification loop with backpressure instead of raw capacity; this lesson hits the same class of loop when one straggling GPU stalls an entire synchronous fleet. Day 17 covered a durable, replayable log as the recovery boundary; this lesson uses the same idea, a periodic checkpoint, to bound how much a GPU failure costs. Day 45/46 covered sharding a big stateful thing across nodes instead of replicating it everywhere; ZeRO is that exact move, aimed at an optimizer's momentum and variance tensors instead of database rows.
**Stack relevance:** Rare.lab is an AI shader product, which means the "AI" part is, at some point, either a model Rare.lab fine-tunes itself or one it depends on someone else having trained at exactly this scale. The day Rare.lab trains or fine-tunes its own shader-generation or scene-understanding model on more than one GPU, both walls in this lesson apply immediately: naive multi-GPU training hits the communication wall past a handful of workers, and any optimizer with per-parameter state (Adam, the default for nearly everything) hits the memory wall long before the model itself would. The more durable lesson for Rare.lab's own runtime, independent of ever training anything, is the shape of the failure mode in Section 6: a global barrier that makes every unit wait for the slowest one is exactly the risk in a shared WebGL context serving many concurrent scene evaluations, one slow or stuck shader compile can stall everyone sharing that context the same way one stuck GPU stalls a 16,384-GPU training step.

---

## 1. The company and the breaking number

**Baidu SVAIL, and the parameter server that got slower as you added more GPUs.** In 2017, Baidu's Silicon Valley AI Lab published "Bringing HPC Techniques to Deep Learning" and open-sourced `baidu-allreduce`, describing a problem every lab training on more than a handful of GPUs was already hitting: the standard way of synchronizing gradients across workers, a central parameter server that every worker pushes its gradients to and pulls updated weights from, requires that server's network interface to handle traffic that grows with the number of workers. Add more GPUs to go faster, and you also add more load on the one machine everyone depends on every single step. Past some worker count, that server's bandwidth, not GPU compute, decides how fast training runs, and adding a 129th GPU can make the average step slower than it was with 128, not faster. Baidu's fix, adapting a decades-old HPC collective-communication algorithm (ring all-reduce) into deep learning, is now the default underneath both Horovod (Uber, 2018) and NCCL (NVIDIA's own collective library), used in essentially every large-scale training job running today.

**Microsoft DeepSpeed, and the model that ran out of memory before it ran out of parameters.** The 2020 ZeRO paper (Rajbhandari, Rasley, Ruwase, He, "ZeRO: Memory Optimizations Toward Training Trillion Parameter Models," SC20) opens with a number that surprises anyone who assumes GPU memory limits are about the model: training GPT-2's 1.5-billion-parameter version with mixed-precision Adam, the standard optimizer for almost all large-model training, requires roughly 24GB of memory just for the optimizer's own bookkeeping (fp16 parameters, fp16 gradients, and fp32 master parameters, momentum, and variance), before a single activation or batch of data is loaded. The math is `16Ψ` bytes for `Ψ` parameters (2 bytes fp16 params + 2 bytes fp16 gradients + 4+4+4 bytes fp32 master weights/momentum/variance = 16 bytes per parameter). At 1.5B parameters that is already 24GB, more than a 16GB V100 has in total, and standard data parallelism makes it worse by replicating that entire 24GB on every single GPU in the job, identically, whether you have 4 GPUs or 4,000.

**Meta, Llama 3, and the number that makes "someone is always failing" the normal operating condition, not the exception.** Meta's Llama 3 herd-of-models report describes training the 405-billion-parameter model on a cluster of 16,384 NVIDIA H100 GPUs over roughly 54 days. In that window the cluster recorded 466 job interruptions, 419 of them unexpected (the rest planned maintenance): about 148 from GPU failures, 72 from GPU HBM3 memory faults, 19 from GPU SRAM errors, 17 from GPU processor issues, 35 from network switch and cable problems, and 2 CPU failures, averaging out to roughly one interruption every three hours across the whole run. Training is synchronous: every GPU must finish its part of the gradient computation and participate in the all-reduce before any GPU moves to the next step. At 16,384 GPUs running for weeks, the probability that at least one of them is unhealthy at any given moment stops being an edge case and becomes the default state of the system, and because the whole fleet shares one synchronization barrier, one failed GPU does not cost that GPU's share of the work, it costs all 16,384 GPUs' worth of idle waiting until something intervenes.

---

## 2. Why the naive (demo) design dies

**The obvious version:** one GPU computes the forward and backward pass on the whole batch, or, once that's too slow, you add more GPUs each with a copy of the model, and put one machine in the middle, a parameter server, that every worker sends its gradients to and receives the averaged, updated weights from. This is exactly how the earliest multi-GPU training setups worked, and for 4 or 8 GPUs it is fine.

**Death one: the parameter server's bandwidth is a fixed budget, and total traffic grows with the number of workers.** Every worker must push its full gradient to the server and pull the full updated weights back, every step. With `N` workers and a model of size `M`, the server sees on the order of `N × M` bytes of inbound traffic and `N × M` bytes of outbound traffic per step. Double the number of GPUs and you double the load on the one machine nothing else depends on, without adding any capacity to handle it. This is Baidu's exact motivating problem: past a worker count that depends on model size and network speed, adding GPUs stops helping because the bottleneck moved off the GPUs entirely and onto one NIC.

**Death two: replicating the optimizer's state on every worker means memory, not model size, sets the ceiling.** Standard data parallelism gives every GPU a full copy of the model, a full copy of the gradients, and a full copy of the optimizer's own state (Adam's momentum and variance tensors, one value per parameter, in addition to the parameter itself). None of that is sharded, all of it is duplicated identically on every worker. Doubling the number of GPUs does not reduce this per-GPU cost by one bit, because it was never divided among the workers in the first place, it was copied to each of them. A model whose weights alone would comfortably fit in memory can still be untrainable, because the optimizer bookkeeping around those weights does not fit.

**Death three: a synchronous barrier means the fleet moves at the speed of its slowest or most broken member, not its average member.** Because every worker must finish and participate in the all-reduce before anyone proceeds, one GPU running slow (a thermal throttle, a flaky NVLink connection) or outright dead (a fault, per Meta's numbers, roughly every three hours at 16,384-GPU scale) does not just lose its own progress, it holds the entire fleet at the barrier until it either recovers or gets detected and evicted. Nothing about the naive design distinguishes "wait a normal amount for the slowest healthy worker" from "wait indefinitely because one worker died," so without explicit handling, a single dead GPU can silently stall a job costing tens of millions of dollars in reserved compute per day.

**The real-world version:** a lab trains a small model on 8 GPUs with a central parameter server and everything works. They scale the same architecture to 128 GPUs to train faster. Step time barely improves, sometimes gets worse, and nobody changed a line of the model, because the server's network card, not the 128 GPUs' combined compute, is now the thing setting the pace.

---

## 3. The architecture

```
Data sharding (per-worker dataloader)
  - job: split the training corpus so each of the N GPUs sees a distinct
    slice of the batch, never the same examples as its neighbors
  - analogy: a printing press splitting one phone book across many
    typesetters, each responsible for a different, non-overlapping
    range of pages

        |
        v
GPU compute workers (forward pass, backward pass, local gradient)
  - job: each worker independently computes a full forward and backward
    pass on its own slice of data, producing a local gradient that, on
    its own, only reflects 1/N of the batch
  - analogy: many chefs in separate kitchens cooking their own dish from
    the same recipe; each dish is correct on its own but the final
    result should reflect everyone's cooking, not just one kitchen's

        |
        v
Ring all-reduce collective (NCCL / Horovod, arranged in a logical ring)
  - job: average every worker's local gradient into one identical,
    globally-averaged gradient, using bandwidth that stays constant
    per worker no matter how many workers join the ring: each GPU
    sends and receives roughly 2(N-1)/N of the gradient's total size,
    not N times that size
  - analogy: a bucket brigade passing partial sums around a circle of
    firefighters instead of everyone running their own bucket across
    the field to one central fire chief

        |
        v
Sharded optimizer state (ZeRO stage 1/2/3, partitioned across the
same N workers doing the compute)
  - job: instead of every GPU carrying a full copy of momentum,
    variance, and (at stage 3) the parameters themselves, split that
    bookkeeping so each GPU owns and updates only its 1/N shard,
    fetching other shards only when a step actually needs them
  - analogy: a library system where each branch stocks one shelf of a
    collection instead of every branch owning a full duplicate set;
    a reader who needs a book outside their branch's shelf requests it
    on demand instead of every branch pre-buying every book

        |
        v
Checkpoint store (durable, periodic snapshot of sharded state to
persistent storage)
  - job: bound how much a failure costs to "time since the last
    checkpoint," not "the entire run so far," by writing a resumable,
    consistent snapshot of every shard on a fixed cadence
  - analogy: a word processor's autosave; a crash costs you the last
    few unsaved minutes, not the whole document

        |
        v
Health-check / straggler-detection / elastic scheduler
  - job: watch every worker's step timing and hardware telemetry,
    detect a stalled or failed GPU fast, and evict-and-replace it
    (or reroute its share of work) instead of leaving the whole fleet
    parked at the barrier waiting for it indefinitely
  - analogy: a relay-race coach who notices a runner has collapsed
    mid-lap and immediately signals the team to adjust, instead of the
    entire stadium sitting in silence waiting for that one lane
```

---

## 4. The transferable mechanisms

- **Ring topology for spreading network traffic evenly, the same shape Day 10 used to spread cached data evenly.** Consistent hashing puts a ring between *data* and the *nodes* storing it, so adding or removing a node only reshuffles a small fraction of keys. Ring all-reduce puts a ring between *workers* and the *traffic* synchronizing them, so each worker only ever talks to its two ring neighbors, sending and receiving a bounded 2(N-1)/N share of the total gradient regardless of how many workers are in the ring. Same primitive, aimed at bandwidth instead of storage: turn an operation that naively costs O(N) at one central point into one that costs O(1) at every point.

- **Shard the stateful thing instead of replicating it, the same move as sharding a database, aimed at an optimizer's tensors.** ZeRO's three stages are a menu of how much to shard: stage 1 (`Pos`) shards only the optimizer's momentum and variance and already gets roughly 4x memory reduction over full replication; stage 2 (`Pos+g`) also shards gradients, roughly 8x; stage 3 (`Pos+g+p`) shards the parameters themselves too, and memory reduction scales linearly with the number of data-parallel workers `Nd` (64 GPUs, 64x reduction). Each stage trades a little more communication (fetching a shard you don't own, on demand, when a step needs it) for a lot less memory per GPU, exactly the general "shard it and fetch on demand instead of copying it everywhere" trade this ledger has made before for rows and files, applied here to gradients and weights.

- **Bounded, periodic checkpoints as the recovery boundary, not the whole run.** A training job that only ever holds state in GPU memory loses everything on any failure. Writing a consistent, resumable snapshot of every shard on a fixed cadence turns "how much work does a failure cost" from "all of it" into "whatever happened since the last checkpoint," the same durability boundary Day 17's write-ahead log gives a database and Day 37's durable execution gives a long-running workflow.

- **Fast failure detection and eviction beats waiting at a barrier, the same move backpressure makes against a retry storm.** A synchronous barrier has no built-in notion of "this worker is dead, move on without it"; left alone, it waits. Meta's mitigation, purpose-built health-check and flight-recorder tooling (NCCLX) to identify and localize a stalled collective operation fast, exists specifically to shrink the gap between "a GPU dies" and "the scheduler notices and reroutes around it," because every second in that gap is 16,384 idle GPUs, not one.

- **Mixed precision and gradient compression reduce the bytes actually moving, buying headroom before the next wall.** fp16/bf16 parameters and gradients halve the bytes an all-reduce has to move compared to fp32, and gradient compression techniques (quantization, top-k sparsification) push further, at the cost of some numerical precision. This is the same "compress before you replicate or transmit it" move this ledger has made with erasure coding: fewer bytes moved means the same ring, the same bandwidth, and the same architecture scale further before hitting the communication wall again.

- **Synchronous-versus-asynchronous update is a straight consistency-versus-availability dial, not a detail.** Synchronous SGD (every worker's gradient counted, every step, via the barrier described above) gives deterministic, reproducible convergence, exactly as if training on one giant GPU, at the cost that the whole fleet's availability depends on every single worker showing up on time. Asynchronous or bounded-staleness variants let fast workers proceed without waiting for a slow one, trading a small amount of gradient staleness (a slightly stale, slightly noisier signal) for a fleet that keeps moving even when one member lags.

---

## 5. The trade-offs

**Consistency versus availability, made concrete in the barrier itself.** Synchronous all-reduce is a strong-consistency choice: every worker's gradient is counted, every step, producing one deterministic, reproducible average, the training-time equivalent of requiring every replica to acknowledge a write before it commits. The cost is availability: the whole fleet's forward progress depends on the least available worker at that instant, exactly why Meta needed dedicated tooling to detect and route around failures fast rather than simply waiting. Relaxing to asynchronous or bounded-staleness updates buys availability, training keeps moving without the slow or dead worker, at the cost of consistency: the gradient average is now slightly stale or noisier, and convergence behavior is less deterministic run to run.

**Cost versus latency, in the specific currency of memory versus network traffic.** ZeRO's sharding is not a free win: every stage that shards more state (gradients at stage 2, parameters at stage 3) also adds more communication, because a GPU that doesn't own a shard has to fetch it on demand instead of already having a local copy. The trade is real and it is the paper's own headline number: roughly 4x memory savings at stage 1, up to 8x at stage 2, and up to `Nd`x at stage 3, purchased with additional all-gather and reduce-scatter traffic per step, worth paying when memory, not network, is what stops you from training the model at all.

**More GPUs is not free scaling once failure probability enters the picture.** Assume each GPU has a fixed, independent probability of failing in any given hour. As the fleet size `N` grows, the probability that *at least one* GPU fails in that hour grows with it, even though each individual GPU's reliability hasn't changed. Meta's 16,384-GPU, 54-day run absorbing 419 unexpected interruptions, one roughly every three hours, is that arithmetic showing up in production: the fix is not "buy more reliable GPUs," it is architectural, fast detection and cheap recovery, because at this scale some GPU somewhere is always in a degraded state and the system has to be designed around that being normal.

---

## 6. The systems-thinking lens

The feedback loop here is **synchronization-barrier amplification**, a specific, well-understood shape of the general "the tail, not the average, sets the pace" failure this ledger has already named twice (Day 13's backpressure lesson, Day 16's hot-key problem): a global barrier means every step's wall-clock time is bounded not by the average GPU's speed but by the single slowest or most broken GPU in that step. As fleet size `N` grows, the probability that *someone* is slow or dead on any given step climbs toward certainty, so "wait for the straggler" stops being an occasional cost and becomes the steady-state cost of running at that scale at all. Left unaddressed, this compounds the same way a retry storm does: a stalled step looks like a hang, operators or auto-recovery logic may restart the job from scratch rather than from a checkpoint, discarding hours of already-completed, correct work on every one of the other 16,383 healthy GPUs, which is a far more expensive failure than the original stall.

The naive fix, throwing more GPUs at the problem to "go faster," makes this specific failure mode worse, not better: more GPUs means more independent chances for one of them to be the straggler on any given step, so the barrier's expected wait time grows with fleet size even as raw compute grows too. The senior fix, visible in both Baidu's and Meta's engineering, does not add capacity to the loop, it changes the loop's shape: ring all-reduce bounds each worker's communication cost independent of `N` so adding GPUs doesn't add proportional network load; frequent, cheap checkpointing bounds the cost of a failure to "since the last checkpoint" instead of "the whole run," so a stall never has to become a full restart; and fast, purpose-built failure detection (Meta's NCCLX health checks and flight recorder) shrinks the time between "a GPU dies" and "the scheduler routes around it," so the barrier's wait time is bounded by detection speed instead of by however long it takes a human to notice a stuck job. Each of these is the same move Day 13 made against retry storms: don't wait longer or add more machines to the thing that's already saturated, remove the step whose cost scales with the thing that's growing.

---

## Sources

- [Bringing HPC Techniques to Deep Learning, Andrew Gibiansky, originally the Baidu Research / SVAIL technical blog (2017)](https://andrew.gibiansky.com/blog/machine-learning/baidu-allreduce/): the primary source for the parameter-server bandwidth problem (traffic to/from the server scaling with worker count) and for ring all-reduce as Baidu SVAIL's fix, adapted from HPC collective-communication algorithms. Direct fetch was blocked by this session's network egress policy; the description here is drawn from search-indexed summaries of the post and from contemporaneous reporting (HPCwire, Tom's Hardware) rather than a direct read of the full original post.
- [baidu-research/baidu-allreduce, GitHub](https://github.com/baidu-research/baidu-allreduce): the open-sourced reference implementation Baidu released alongside the blog post.
- [HPC Technique Propels Deep Learning at Scale, HPCwire (2017)](https://www.hpcwire.com/2017/02/21/hpc-technique-benefits-deep-learning/): secondary, contemporaneous coverage corroborating the ring-allreduce motivation and its adoption path into Horovod and NCCL.
- [ZeRO: Memory Optimizations Toward Training Trillion Parameter Models, Rajbhandari, Rasley, Ruwase, and He, SC20 (2020)](https://arxiv.org/abs/1910.02054): the primary academic source for the `16Ψ`-bytes-per-parameter mixed-precision Adam memory formula, the roughly 24GB figure for a 1.5-billion-parameter (GPT-2 scale) model under standard data parallelism, and the ZeRO stage 1/2/3 partitioning scheme and its memory-reduction factors (roughly 4x, 8x, and linear in `Nd`). Direct fetch of the paper and of the DeepSpeed project's own tutorial pages was blocked by this session's network egress policy; the specific figures here are drawn from search-indexed excerpts of the paper and of Microsoft's own ZeRO/DeepSpeed blog posts rather than a full direct read.
- [ZeRO & DeepSpeed: New system optimizations enable training models with over 100 billion parameters, Microsoft Research blog (2020)](https://www.microsoft.com/en-us/research/blog/zero-deepspeed-new-system-optimizations-enable-training-models-with-over-100-billion-parameters/): corroborating source for the same memory-reduction figures, in Microsoft's own framing.
- [The Llama 3 Herd of Models, Meta AI (2024)](https://ai.meta.com/research/publications/the-llama-3-herd-of-models/): the primary source for the 16,384 H100 GPU training cluster, the roughly 54-day training window, and the reliability data (466 total job interruptions, 419 unexpected, with the GPU/HBM3/SRAM/network breakdown cited above). Direct fetch of the paper was blocked by this session's network egress policy; the specific interruption counts and percentages here are drawn from search-indexed summaries and contemporaneous technical press coverage (Tom's Hardware, DataCenterDynamics) rather than a full direct read of the original report.
- [Faulty Nvidia H100 GPUs and HBM3 memory caused half of failures during Llama 3 training, Tom's Hardware (2024)](https://www.tomshardware.com/tech-industry/artificial-intelligence/faulty-nvidia-h100-gpus-and-hbm3-memory-caused-half-of-the-failures-during-llama-3-training-one-failure-every-three-hours-for-metas-16384-gpu-training-cluster): secondary source aggregating and summarizing the Llama 3 paper's interruption-cause breakdown; used here because direct access to the primary paper was blocked, as noted above.
- Day 10 (this ledger, consistent hashing and sharding), Day 13 (backpressure and load shedding), Day 16 (the hot-key/celebrity problem), Day 17 (WAL and change data capture), Day 37 (durable execution and workflow orchestration), Day 45/46 (secondary indexes and joins on sharded databases): the ledger's own prior lessons this one directly reuses, generalized here from database rows, cache keys, and workflow steps up to gradients, optimizer tensors, and GPU workers.

**A note on sourcing for this lesson:** this session's network egress policy blocked direct retrieval of the original Baidu SVAIL blog post, the ZeRO paper and DeepSpeed's own documentation, and the Llama 3 paper itself, so the specific numbers drawn from them (the `16Ψ`-byte memory formula and 24GB figure, the ZeRO stage reduction factors, and the 466/419 interruption counts and their cause breakdown) are summarized here from search-indexed excerpts and secondary technical reporting rather than quoted from a full direct read of the primary sources, consistent with how this ledger has flagged network-blocked sources before (see Day 69). The general architectural claims, ring all-reduce's bandwidth-independent-of-N property and ZeRO's partition-instead-of-replicate design, are treated as solid and well corroborated across multiple independent sources (the original authors' own project documentation, academic citation, and independent technical press), even where the exact original document could not be fetched directly.
