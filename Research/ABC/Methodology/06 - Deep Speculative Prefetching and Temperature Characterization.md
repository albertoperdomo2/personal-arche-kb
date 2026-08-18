---
title: "Deep Speculative Prefetching and Temperature Characterization for Multi-Tier KV Cache Offloading in Agentic LLM Workloads"
date: "2026-08-18"
type: "research-synthesis"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
status: "active"
related: "[[Methodology/01 - Experiment Definition]]"
---

# Deep Speculative Prefetching and Temperature Characterization for Multi-Tier KV Cache Offloading in Agentic LLM Workloads

> Author research doc by Alberto Perdomo, shared in chat on 2026-08-18. Staged verbatim for KB curation. Note: several inline LaTeX formulas were stripped when the doc was pasted into chat; where the prose unambiguously defines them, they have been reconstructed and marked [reconstructed].

## Introduction to the Memory-Bound Paradigm of Large Language Model Inference

The landscape of Large Language Model (LLM) inference has undergone a fundamental transformation, shifting from compute-bound, isolated query processing to memory-bound, stateful, and highly iterative execution patterns. As deployment paradigms pivot toward agentic workflows—characterized by autonomous tool utilization, iterative code execution, structured memory retrieval, and multi-step reasoning—the Key-Value (KV) cache has evolved from a transient decoding optimization artifact into a first-class, persistent global data structure. In these complex environments, LLMs repeatedly revisit overlapping contextual foundations, leading to KV cache hit rates that frequently exceed 95 percent. Consequently, the primary operational bottleneck constraining system throughput and Time-To-First-Token (TTFT) latency is no longer the pure matrix multiplication capacity of the GPU, but rather the Input/Output (I/O) capability required to fetch, transfer, and manage massive KV cache representations across heterogeneous storage networks.

Existing KV cache management frameworks, including native vLLM implementations, LMCache, and Mooncake, predominantly rely on reactive eviction policies, such as Least Recently Used (LRU) algorithms, alongside heuristic-based migration strategies. While these reactive systems efficiently handle standard, stateless conversational workloads, they suffer from critical TTFT stalls under heavy agentic loads due to network congestion and the high latency of synchronous storage fetches. When cache eviction is triggered dynamically upon memory exhaustion, or when prefill operations rely exclusively on admission-time cache lookups, the inference engine is forced to suspend computational progress while transferring gigabytes of tensor data over Peripheral Component Interconnect Express (PCIe) or Remote Direct Memory Access (RDMA) networks.

To circumvent these severe I/O bottlenecks and fully leverage the locality inherent in agentic access patterns, a proactive, activity-based KV cache management framework is strictly necessary. By systematically classifying KV cache blocks by an estimated "temperature"—a deterministic, multidimensional measure of access likelihood—distributed inference systems can asynchronously prefetch critical blocks across a multi-tiered storage hierarchy prior to request admission. This comprehensive report thoroughly examines the mechanics of characterizing temperature in agentic workloads, the design of an event-driven speculative prefetching heuristic, the rigorous cost-benefit thresholds governing multi-tier data migration, and the precise, code-level implementation pathways within the vLLM engine and the llm-d Endpoint Picker.

## The Evolution of KV Cache Architectures in LLM Serving

To establish a robust foundation for temperature-based speculative prefetching, it is essential to analyze the structural evolution of the underlying KV cache management systems that currently define state-of-the-art LLM serving. The architectural decisions made in these frameworks directly dictate the constraints and opportunities for predictive data placement.

### PagedAttention and the Virtualization of KV Memory

The foundational innovation enabling modern, high-throughput LLM serving is vLLM's PagedAttention mechanism. Prior to this development, inference engines managed the KV cache as contiguous memory tensors, pre-allocating the maximum possible sequence length for every request. This naive static batching led to severe internal and external memory fragmentation, frequently leaving upwards of 60 percent of GPU High Bandwidth Memory (HBM) unutilized.

PagedAttention directly mirrors the virtual memory architecture of traditional operating systems. It decouples the logical representation of a sequence from its physical storage by partitioning the KV cache into fixed-size, non-contiguous physical blocks—typically configured to 16 tokens per block. A block table maps logical token sequences to these physical blocks, enabling dynamic, on-demand allocation and facilitating block-level memory sharing across concurrent requests through a Copy-on-Write (CoW) mechanism. This block-level granularity is critical for temperature characterization, as the 16-token vLLM block represents the fundamental atomic unit of cache reuse and eviction within the GPU boundaries.

### Persistent Caching and Prefill-Decode Disaggregation

While vLLM resolved intra-node memory fragmentation, its native cache remains ephemeral, tied to the lifecycle of active requests or limited to the capacity of local GPU HBM. To address cross-session reuse and scale beyond a single node, frameworks like LMCache and Mooncake introduced persistent, disaggregated KV caching architectures.

LMCache functions as a distributed KV cache layer that extracts and stores KV caches out of GPU memory, sharing them across engines and queries. It supports both prefix cache offloading and cross-node cache transfers, demonstrating up to a 15-fold improvement in throughput for multi-round document analysis. Crucially, LMCache addresses the I/O inefficiencies of transferring small 16-token blocks by grouping multiple pages from multiple transformer layers into larger "chunks"—defaulting to 256 tokens—to saturate network bandwidth strata. This introduces a significant architectural consideration for speculative prefetching: the impedance mismatch between vLLM's 16-token execution granularity and LMCache's 256-token optimal transfer granularity.

Mooncake further advances this paradigm by implementing a strictly KVCache-centric disaggregated architecture, permanently decoupling the prefill (compute-bound) and decoding (memory-bound) clusters. The core of Mooncake is a global scheduler, or "Conductor," which treats the entire cluster's CPU Dynamic Random Access Memory (DRAM) and Solid State Drives (SSDs) as a unified, disaggregated cache pool. Mooncake's implementation highlights a fundamental tension in LLM serving Service Level Objectives (SLOs): maximizing overall throughput by reusing remote KV caches prolongs the Time-To-First-Token (TTFT), while maximizing batch sizes degrades the Time-Between-Tokens (TBT). Consequently, any speculative prefetching heuristic must strictly balance these competing SLOs, ensuring that the act of prefetching does not induce network congestion that delays active decoding tasks.

## The Unique Dynamics of Agentic Memory Access Patterns

Agentic workloads deviate structurally and behaviorally from traditional, linear Retrieval-Augmented Generation (RAG) tasks. In a standard RAG pipeline, raw text passages are retrieved and appended to the prompt, and the dominant source of reusable KV states is cleanly localized to the static system prefix or the discrete retrieved document. In contrast, agentic memory systems actively curate knowledge through iterative operations, enriching retrieved representations with LLM-generated metadata such as summaries, hierarchical tags, and relational graph nodes.

### The Failure of Partial Recompute and Token-Selection Methods

The distinction between raw RAG passages and metadata-rich agentic memory is not merely semantic; it fundamentally alters attention dynamics. Recent empirical research, specifically the AgentKVShift methodology, demonstrates that existing training-free KV cache reuse techniques—which attempt to selectively recompute only a small fraction of tokens—suffer severe accuracy degradation when applied to agentic memories.

The structural cause of this degradation lies in the distribution of the KV reuse residual. In agentic contexts, the error induced by reusing a cached token does not manifest as isolated, token-wise anomalies. Instead, the per-memory KV reuse residual decomposes into a shared, chunk-level offset combined with minor token-wise fluctuations. When an agent retrieves a curated memory unit, the presence of new metadata alters the aggregate representation formed within each transformer layer. Because conventional token-selection methods leave unrefreshed tokens completely stale, they fail to account for this shared memory-level bias, destroying the cross-chunk global attention relationships required for high-fidelity reasoning.

This phenomenon heavily influences the design of a temperature-based prefetching heuristic. It dictates that KV cache access and the definition of "hotness" in agentic environments exhibit dense spatial locality at the chunk level. Individual 16-token vLLM blocks cannot be evaluated for temperature in strict isolation; their reuse probability and semantic validity are inextricably linked to the broader curated memory unit to which they belong. When predicting cache temperature, the framework must treat these structured memory graphs as cohesive spatial units.

### Multi-Agent Collaboration and Deterministic Redundancy

In multi-agent collaborative frameworks, complex tasks such as mathematical reasoning, iterative code generation, and simulated software engineering involve cascading LLM calls where the output generated by one agent serves as the input prompt for the subsequent agent in the pipeline. The RelayCaching framework addresses this specific dynamic, noting that redundant prefill computation for shared content generated by previous agents creates a critical memory and latency bottleneck.

The critical insight from multi-agent systems is that decoding-phase KV caches from a preceding agent are highly consistent with the prefill-phase requirements of a subsequent agent. Prefix-induced deviations are typically sparse and localized within a limited range of specific attention layers. Therefore, the temperature of a KV cache block in these environments is not solely a function of user-driven temporal recency, but is heavily influenced by the deterministic execution graph of the multi-agent orchestrator. If a central routing agent assigns a sub-task to a specialized coding agent, the orchestrator's output blocks should immediately be classified as possessing maximum temperature, irrespective of historical frequency, because their future consumption is guaranteed by the application logic.

## Designing the Event-Driven Heuristic Framework

To proactively migrate KV blocks across a multi-tiered storage hierarchy, the framework eschews black-box machine learning in favor of a deterministic, event-driven scheduling architecture. This framework co-optimizes scheduling and memory management by leveraging the strict state-machine behavior of autonomous agents, drawing heavily from recent KV-centric serving systems like TokenCake and KVFlow.

### Event-Driven Temporal Scheduling

Agentic workloads do not exhibit random access patterns; they follow a highly predictable execution loop: LLM Inference → External Function Call → LLM Inference.

The heuristic exploits this deterministic loop as the primary driver for temperature modeling:

- **Proactive Offloading:** When an agent issues a function call (e.g., executing Python code, querying a database), its corresponding KV cache instantly becomes idle. Instead of relying on a generalized LRU metric to eventually evict this memory, the framework intercepts the Function Call Start event and proactively offloads the session's KV cache to the CPU (Warm tier) or NVMe (Cool tier), freeing GPU HBM opportunistically.
- **Predictive Prefetching:** Once the function call completes and the agent is re-admitted to the scheduling queue, the heuristic registers an imminent access event. LMCache and TokenCake demonstrate that the idle interval between queue admission and actual computation execution is the ideal window for prefetching. The framework triggers background daemon threads to pull the required KV tensors from the lower tiers back into the GPU before the prefill phase stalls on a cache miss.

### Kinetic Average Eviction Time (AET)

For the background management of system prompts, general conversational history, and non-event-driven cache blocks, the framework utilizes a kinetic Average Eviction Time (AET) model rather than probabilistic decay.

AET tracks the "travel speed" of a block moving down the LRU stack. Because this travel speed is monotone and non-increasing on average, the framework can accurately measure acceleration toward the eviction threshold in linear time with O(1) space overhead. If a block's AET trajectory indicates imminent eviction, the heuristic classifies it as "Cooling" and gracefully demotes it to NVMe or CephFS asynchronously, preventing the system from ever hitting hard capacity limits that force synchronous, blocking evictions.

### Bandwidth-Aware Spatial Chunking

A critical flaw in naive heuristic prefetching is the impedance mismatch between vLLM's execution granularity (16 tokens) and hardware I/O efficiency. Transferring 16 tokens at a time across PCIe or NVMe fails entirely to saturate available bandwidth strata.

When making a prefetch or offload decision, the framework groups 16-token physical blocks with their semantic neighbors to form contiguous 256-token chunks. If the temporal heuristic dictates a block must be fetched, the connector pulls the entire 256-token chunk into an intermediate streaming GPU buffer. This guarantees that the prefetch operation fully saturates the network or PCIe link, amortizing the I/O overhead and accelerating the migration of functionally cohesive agentic memories.

## Cost-Aware Tier Placement and Multi-Hop Migration

Upon generating the predictive temperature profile for the cache blocks, the framework classifies the data into four distinct hierarchical storage tiers. This multi-tiered approach allows the system to balance extreme latency requirements against vast storage capacity limits.

| Storage Tier | Target Hardware | Profile | Ideal Block Classification |
|---|---|---|---|
| Hot | GPU HBM | Immediate access required. Highly sensitive to TTFT and ITL SLOs. Capacity strictly limited. | Active generation contexts, imminent multi-agent handoffs, and highly reused root prefixes. |
| Warm | CPU DRAM | Highly probable access within the short-term window. Fast transfer to GPU via PCIe. | Near-term conversational history, blocks for sessions executing fast external API calls. |
| Cool | NVMe SSD | Probable access in the medium-term. Excellent capacity-to-cost ratio. | Persistent sessions awaiting delayed user input, dormant agent processes. |
| Cold | CephFS / Distributed FS | Long-term archival. High latency overhead for retrieval, but effectively infinite capacity. | Historical sessions, offline batch processing contexts. |

### The Cost-Benefit Migration Equation

Migration decisions within this four-tier hierarchy do not universally execute just because an event fires. If the network cost of the transfer severely impacts concurrent, latency-sensitive decoding requests, an opportunistic offload is skipped. Therefore, the framework dictates that a migration is performed only when the expected benefit strictly outweighs the systemic cost, multiplied by a configurable load threshold.

The migration inequality is defined as [reconstructed]:

$$\text{Benefit} > N \times \text{Cost}$$

Where $N$ is a dynamic threshold parameter tuned in real-time based on current system load, queue depths, and interconnect utilization.

**Calculating Systemic Cost:** The $\text{Cost}$ term is a composite function of bandwidth consumption, hardware opportunity cost, and interconnect contention [reconstructed]:

$$\text{Cost} = \frac{S_{block}}{B_{link}} \times \lambda_{queue} + C_{evict}$$

In this formulation, $S_{block}$ represents the physical size of the KV block (e.g., approximately 320 KB per token for a model like Llama2-70B). $B_{link}$ represents the available bandwidth of the targeted interconnect (such as NVLink, PCIe, or RDMA Ethernet). $\lambda_{queue}$ is a dynamic penalty weight that scales linearly with current network queue depths, preventing prefetch operations from starving active forward passes. Finally, $C_{evict}$ quantifies the opportunity cost of evicting a currently resident block in the higher tier to accommodate the promoted block.

**Calculating Expected Benefit:** The $\text{Benefit}$ term quantifies the expected reduction in latency and the avoidance of redundant floating-point operations (FLOPs) [reconstructed]:

$$\text{Benefit} = P_{access} \times (T_{prefill}(L) - T_{fetch}) + V_{compute}$$

Here, $P_{access}$ is the definitive probability of access. $T_{prefill}$ is the massive TTFT penalty incurred by re-running the prefill phase through the attention layers, an operation that scales quadratically $O(L^2)$ with the sequence length $L$. $T_{fetch}$ is the anticipated latency of pulling the block from the lower tier. $V_{compute}$ represents the holistic value of avoiding redundant GPU computation.

### Asynchronous Multi-Hop Prefetching Dynamics

Empirical benchmarking indicates that naive prefetching at admission time yields marginal systemic improvements because the prefill phase must still block while the network transfers the data, ultimately stalling TTFT. To circumvent this limitation, the framework utilizes an asynchronous priority scheduler designed to perform multi-hop promotions in the background.

When a session feature indicates an upcoming interaction—for example, an agent initiates an external database query that typically takes 5 seconds to resolve—the framework proactively initiates a multi-hop promotion chain. A "Cool" block residing in NVMe is promoted to CPU DRAM (Warm) completely asynchronously. This effectively hides the I/O latency of the slowest storage tiers. By the time the database query resolves and the request is officially submitted to the inference engine, the required KV cache chunks are already physically resident in CPU memory, requiring only a final, high-bandwidth PCIe hop to the GPU HBM—or, in optimal scenarios, a direct allocation into the GPU if capacity permits.

Furthermore, leveraging architectural patterns observed in the DualPath and FlowKV frameworks, multi-hop prefetching can be routed optimally to avoid network saturation. Loading massive KV caches from external storage often saturates the NICs of prefill engines, while the NICs on decoding engines remain idle. The multi-hop prefetching system can optionally route "Warm" blocks directly into the DRAM of decoding nodes via an RDMA storage-to-decode path, thereby preserving the prefill node's bandwidth strictly for incoming, un-cached user prompts.

## System Integration: Implementation within vLLM and the KV Connector API

Integrating complex heuristics into vLLM requires a nuanced understanding of its execution pipeline, specifically the PagedAttention block manager and the KVConnectorBase_V1 interface. The KVConnectorBase_V1 is the established API contract within vLLM for interacting with distributed and multi-tiered KV cache stores.

### Intercepting the KVConnectorBase_V1 Hooks

The multi-tiered offloading logic is implemented as a specialized connector class inheriting from KVConnectorBase_V1, leveraging daemon threads and pipelined data transfer architecture analogous to LMCache.

- **build_connector_meta(scheduler_output):** Executed during the scheduling phase, prior to the forward pass, this method builds metadata for KV cache transfers between the backend and GPU memory. The event-driven framework injects its prefetch triggers here. If an agent function call has resolved, the connector marks the required logical block hashes for immediate async retrieval.
- **start_load_kv(forward_context):** This method triggers the actual loading of the KV cache from the lower-tier storage into vLLM's paged buffer. By enabling async loading between scheduler steps, this operation allows the engine to pull data concurrently with GPU execution.
- **Layer-wise Pipelining and wait_for_layer_load(layer_name):** To fully hide I/O latency, the framework strictly enforces layer-wise pipelining (use_layerwise = True), supported natively in the V1 scheduler. start_load_kv initiates the fetch, but the GPU only blocks when wait_for_layer_load is explicitly called from within a specific attention layer. The framework spawns LayerSendingThread and LayerRecvingThread daemon threads to stream KV blocks sequentially. This allows layer $i$ to transfer over PCIe concurrently while the GPU is computing layer $i-1$, effectively overlapping computation and hiding 60 to 85 percent of latency.
- **save_kv_layer(layer_name, kv_layer, attn_metadata):** Following the attention computation, this hook asynchronously saves a layer of KV cache from vLLM's paged buffer back to the connector. The temporal heuristic uses this hook for proactive offloading; if a Function Call Start event fired, save_kv_layer streams the data to the CPU tier directly out of the forward pass.

### Managing Exact Matches Versus Semantic Shifts

A highly critical engineering consideration when implementing speculative prefetching for agentic workloads is managing the discrepancy between exact-match hashing and semantic relevance. Native vLLM utilizes a strict BlockHash—a cryptographic hash of the token sequence—to identify reusable prefix cache blocks. However, agentic workloads frequently feature slight permutations in retrieval order, timestamp injections, or metadata formatting that completely invalidate exact hashes, despite the underlying KV tensors remaining highly relevant for attention computation.

Drawing directly from the methodologies proposed in C^2KV and AgentKVShift, the framework must not indiscriminately discard warm, prefetched blocks simply because of a hash miss. Instead, the heuristic framework tracks blocks via a broader, session-aware logical ID. When a slight semantic drift is detected, the framework can utilize probe-guided residual correction. By estimating the chunk-level offset from a small probe set of tokens, the system applies a weighted mean-shift to the reused tokens. This correction operation is strictly compute-bound and substantially faster than either fetching cold exact-match blocks from CephFS or forcing a full recomputation through the prefill engine.

## Session-Aware Routing via the llm-d Endpoint Picker

The systemic effectiveness of proactive KV cache placement is exponentially magnified when coupled with intelligent, multi-node request routing. The llm-d inference gateway provides "Intelligent Inference Scheduling," utilizing an advanced Endpoint Picker that routes requests based on prefix-cache affinities and active cluster load.

Currently, standard load balancers, and even naive affinity routers, often scatter related conversational turns across disparate pods based strictly on instantaneous CPU/GPU utilization, thereby destroying cache locality and forcing expensive remote network transfers. The proposed activity-based framework transforms this paradigm by exporting the continuously updated AET and session metadata directly to the llm-d Endpoint Picker.

### Cross-Replica Prefix-Affinity Routing

When a request arrives at the llm-d gateway in Gateway Mode, the routing logic consults the exported temperature map. The scheduling formula evolves from a simple utilization check into a composite function of node queue depth, exact prefix match ratio, and the cumulative predicted temperature of resident blocks [reconstructed]:

$$\text{Score}(n) = w_1 \cdot \text{QueueDepth}(n) + w_2 \cdot \text{PrefixMatch}(n) + w_3 \cdot \text{TemperatureSum}(n)$$

This scoring mechanism ensures that agentic sessions are routed to the specific replica most likely to possess a warm cache, drastically improving hit rates and maximizing cost-efficiency in distributed deployments.

### Predictive Session Migration

If an agentic conversation must inevitably be migrated to another inference node due to strict load balancing constraints or hardware failure, the framework utilizes the exported temperature data to initiate a Session-Aware Prefetch. Before the next user or agent turn fully reaches the destination node's prefill engine, the framework identifies the hottest KV cache blocks for that session, ranks them by predicted temperature, and proactively streams them directly into the destination node's CPU DRAM.

This predictive, queue-informed management—conceptually aligned with mechanisms proposed in frameworks like PEEK and KVFlow—minimizes cache warm-up latency. By ensuring that the destination node's cache is primed before the request is even admitted, the system maintains strict TTFT Service Level Objectives even across highly volatile, distributed inference environments.

## Future Research Directions and Optimization Synergies

While the immediate implementation of the event-driven speculative prefetching pipeline within the KVConnectorBase_V1 ecosystem carries a high confidence of success due to its reliance on mature API hooks, the domain of KV cache optimization presents several highly lucrative avenues for future research.

### Hardware-Aware Adaptive Recomputation

As demonstrated by the CacheTune system, under extreme I/O constraints (e.g., massive network congestion to the CephFS tier), it is occasionally mathematically optimal to simply recompute specific semantic-critical tokens rather than waiting for an I/O transfer to complete. Future iterations of the cost-benefit migration model should dynamically evaluate the real-time bandwidth ($B_{link}$) of the storage NICs. If bandwidth drops below a critical threshold, the framework should dynamically fall back to recomputing the KV cache locally, updating the threshold parameter $N$ in real-time based on hardware telemetry, sustaining speedups even when storage is heavily I/O bound.

### Integration with FP8 and INT4 Quantization

The profound I/O bottleneck of multi-tier KV cache transfers can be significantly mitigated through the application of post-training quantization. Storing and transferring KV caches in FP8 (e.g., the e4m3 format) or INT4 formats effectively halves or quarters the bandwidth requirements and storage footprint.

Future architectures should investigate the viability of asymmetric quantization tiers. For instance, retaining FP16 in GPU HBM for maximum accuracy during sensitive autoregressive generation, while compressing blocks to FP8 or INT4 prior to offloading them to the Cool (NVMe) and Cold (CephFS) tiers. The asynchronous multi-hop prefetching pipeline would then include a rapid, fused dequantization kernel as blocks transition from Warm (CPU) to Hot (GPU). Crucially, research from AgentKVShift demonstrates that advanced residual correction methods compose orthogonally with aggressive 2-bit and 4-bit KV quantization, retaining high accuracy even under severe context drift.

### Cross-Model Prefix Caching via Activated LoRA

The recent emergence of frameworks proposing Activated LoRA (aLoRA) introduces the unprecedented capability to share prefix caches across base and adapted models during inference. In complex agentic workflows where multiple specialized LoRA adapters process the exact same foundational context or retrieved knowledge base, the temperature prediction engine could be expanded to track block affinity not just by session, but by adapter type. A "Hot" block generated by the base model could be proactively fetched and directly mapped into the memory space of an incoming LoRA-specific request, effectively multiplying the value of a single prefetch operation across the entire adapter ecosystem and further driving down the aggregate TTFT of the cluster.

## Works cited

1. vLLM Explained: PagedAttention and Continuous Batching - Runpod, https://www.runpod.io/articles/guides/vllm-pagedattention-continuous-batching
2. LMCache: An Efficient KV Cache Layer for Enterprise-Scale LLM Inference - arXiv, https://arxiv.org/html/2510.09665v2
3. KVFlow: Efficient Prefix Caching for Accelerating LLM-Based Multi-Agent Workflows - arXiv, https://arxiv.org/html/2507.07400v1
4. TokenCake: A KV-Cache-centric Serving Framework for LLM-based Multi-Agent Applications - arXiv, https://arxiv.org/html/2510.18586v3
5. Kinetic Modeling of Data Eviction in Cache - USENIX, https://www.usenix.org/system/files/conference/atc16/atc16_paper-hu.pdf
6. [RFC]: Add Mooncake Store Connector for Shared KV Cache Reuse · Issue #38474 - GitHub, https://github.com/vllm-project/vllm/issues/38474
7. vllm/vllm/distributed/kv_transfer/kv_connector/v1/lmcache_connector.py at main - GitHub, https://github.com/vllm-project/vllm/blob/main/vllm/distributed/kv_transfer/kv_connector/v1/lmcache_connector.py
8. Mooncake: A KVCache-centric Disaggregated Architecture for LLM Serving - ResearchGate, https://www.researchgate.net/publication/397645965_Mooncake_A_KVCache-centric_Disaggregated_Architecture_for_LLM_Serving
