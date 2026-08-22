---
title: "Independent research audit and redirection — speculative KV-cache prefetching"
date: "2026-08-21"
type: "research-audit"
experiment: "Activity-Based KV Cache Tier Placement"
status: "research-reset"
scope: "Arche evidence, local vLLM branch, AgentX/Weka experiments, and external systems literature"
confidence: "high (0.85)"
---

# Independent research audit and redirection — speculative KV-cache prefetching

## 1. Executive conclusion

**Verdict: the broad opportunity is promising, but the current research path is partially misguided and the current V7 policy is a no-go. Confidence: 0.85.**

There is a real systems opportunity. Reactive NVMe/CephFS retrieval has produced multi-second request-visible delay, and realistic agent workloads expose reusable prefixes and workflow dependencies. However, the project has repeatedly treated three different questions as if they were one:

1. Can a secondary-tier block be moved into CPU before demand?
2. Can a policy choose data with enough lead time and low enough false-positive cost?
3. Does the movement remove exposed end-to-end stall without interfering with active inference?

The experiments answer the first question. They do **not** yet answer the third positively.

The current V7 mechanism is not a next-block predictor. It is deterministic lookahead for an already-arrived request: select a short contiguous prefix beyond the CPU frontier, estimate time until first scheduling, probe NVMe residency, then move at most 64 chunks from NVMe to CPU in eight 8-chunk jobs. That is a legitimate baseline, but it is too narrow to be the production architecture:

- it obtains lead time mainly from overload-induced queueing;
- it performs selection and policy work on the scheduler path;
- speculative and demand metadata lookups share a non-preemptible worker;
- it stages only NVMe -> CPU, not CPU -> GPU;
- it admits one speculative owner at a time;
- its `useful` metric means "later CPU hit," not "critical-path stall avoided";
- in V7, useful staging was tiny relative to workload progress, while the treatment's demand lookup delay doubled and service collapsed.

The strongest positive result often cited for the current direction is weaker than remembered. The serialized GuideLLM run (`max_num_seqs=1`) proved the wiring: every one of 25,344 promoted chunks was eventually demanded and 92.97% was ready at first demand. It did **not** prove a mean performance gain: request throughput and mean TTFT were essentially flat, the apparent tail improvement was a single cross-node pair, the control completed 255/256 requests, and JIT occurred during measurement. Serialization manufactured a long, clean prediction horizon and removed the continuous-batching problem we ultimately need to solve.

The next serious investment should therefore be a **clean oracle and data-readiness substrate**, not V8 of the current heuristic. The primary research question becomes:

> Given exact knowledge of a future request's prefix working set and deadline, how much exposed NVMe -> CPU and CPU -> GPU stall can vLLM actually hide under realistic continuous batching, without delaying demand work?

If that oracle shows material end-to-end headroom, the best production direction is a **scheduler-integrated, deadline-aware working-set staging system** with cache-aware admission and placement/eviction co-design. For agentic workloads, its highest-value signal is likely an **advisory event emitted before the HTTP inference request**—for example at tool execution, workflow-node completion, keystroke/session activity, or known DAG successor activation. Queue position at ordinary request admission remains a baseline, not the primary production signal.

Do not continue patching V7 unchanged. Preserve its tests, accounting, batched promotion, demand fallback, and priority lessons; retire its queue-EWMA/fixed-bundle/single-owner policy as the main research vehicle.

## 2. What the evidence actually says

### Evidence reconstruction

| Hypothesis | Mechanism | Workload | Expected benefit | Actual result | Bottleneck / failure mode | Confidence |
|---|---|---|---|---|---|---|
| Reactive secondary retrieval creates an opportunity | Native tiered demand lookup and promotion | Nemotron AgentX/Weka, CPU/NVMe/CephFS | Storage tiers should avoid recompute but expose retrieval stall | NVMe/CephFS asynchronous lookup distributions were multi-second; P99 hit a 10 s histogram ceiling and request-level means reached tens of seconds | Step-quantized lookup/promotion, queueing, CPU capacity; cells were not all image-matched | High that an opportunity exists; medium on exact magnitude |
| Post-miss forward read-ahead can prefetch later chunks | After a terminal prefix MISS, prefetch later keys | Phase-1 NVMe | Hide the next demand miss | 100% skipped; no promotion | Prefix-chained hashes mean later prefix keys cannot exist after an earlier missing prefix; `RETRY` was also collapsed | High; **falsified** |
| Blind admission first-N can prove movement | Assume first N keys resident and promote NVMe -> CPU | GuideLLM, `max_num_seqs=1`, N=100 | Long queue lead should make copies ready | 25,344/25,344 promoted chunks eventually useful; 7.03% late | Constructed serialization; mean performance flat; cross-node, one incomplete control, JIT | High for mechanism; low for performance |
| The same first-N policy transfers to realistic AgentX | Blind N=100 | Weka C32 | Improve TTFT | 90.99% attempted already CPU-resident; 87.08% of submissions failed; 98.50% late; only 1.16% attempts useful | Little queue horizon (mean waiting below 0.25), blind selection, stale/absent NVMe data | High negative |
| More queue lead improves timing | Blind N=100 | Weka C64 | More waiting should increase usefulness | useful/attempted rose to 15.81%; late/promoted fell to 42.39%; no positive throughput/TTFT result | Queue pressure provides horizon but also marks a stressed system; failures/waste remained | High for horizon sensitivity; high that usefulness alone was insufficient |
| Residency/deadline gating can avoid bad work | V2.1 shadow/live contiguous bundle | Weka | Submit only resident, timely prefixes | Shadow selected work; first live run submitted zero because CPU had no truly free speculative capacity | Correct selection could not allocate | High |
| A bounded CPU reserve makes live prefetch viable | V2.1 reserve, 64-chunk bundle | Weka | Promotions survive to demand and improve performance | Promotions executed; about 1.9% useful and 96.5% wasted; ~0.262 req/s and ~47 s mean TTFT | A self-eviction defect and lack of retention confounded the result | High that run was harmful; medium on attribution |
| A retention lease fixes waste | V2.1 one-bundle lease | Weka | Completed copies survive | 4.71% useful, 93.29% wasted; 31.71% pre-demand eviction; lease reclaimed by demand pressure; ~0.261 req/s and ~48 s mean TTFT | Reactive demand repeatedly consumed speculative headroom | High negative |
| JIT single-owner demand-safe policy fixes interference | V3 V6 | Weka | ~50% useful with no demand harm | 1,024 promoted; 512 useful; 448 wasted; no failures; apparent performance gain | Control/treatment nodes diverged during warmup before any speculative submission; 51 requests cancelled per cell; no GPU telemetry | High for accounting/retention, **invalid for benefit** |
| Batch-aware V7 works under continuous batching | Queue-round estimator, low-priority probes, one owner, 8-chunk slices | Weka C64, `max_num_seqs=8` | Preserve useful yield without slowing demand | 48.1% of submitted chunks useful, but request throughput -55.0%, mean TTFT +125.8%, mean ITL +180.3%; async lookup mean 37.84 -> 78.99 s | Current implementation/configuration harmful; cross-node pair and missing GPU/hook telemetry prevent single-cause attribution | High that V7 is a no-go; medium on root cause |

### The strongest negative evidence

The V7 pair is the most important result, not because it conclusively proves one bug, but because it falsifies "a high useful ratio implies useful prefetch." The treatment promoted 2.019 chunks/s, recorded 0.972 useful chunks/s, and still cut request throughput from 0.362 to 0.163 requests/s. NVMe busy time was lower, not saturated. Running depth remained near eight and there were no preemptions, OOMs, or request failures. Generation rate fell from 621.6 to 212.8 tokens/s and mean ITL rose from 22.32 to 62.58 ms.

This result is conditionally valid: one run per cell, different H100 nodes, missing GPU utilization/clocks/memory-bandwidth/PCIe telemetry, and unrecorded initial NVMe state. It rejects V7 as tested. It does **not** prove that speculative KV prefetching as a class is a dead end.

### Key experiment registry

| Experiment | Control / comparator | Treatment | Disposition |
|---|---|---|---|
| Phase-1 post-miss NVMe | `988f03995bb745659749110472019c6b` | `96d01b33a71f4f1bbb2d55a53a8aaacd` | Structural negative; mechanism impossible under prefix-chain invariant |
| Repaired GuideLLM serialized oracle | `3581db3f82d7427c883ff72113390121` | `b28bd1db0836406a94c31c2e3faa7c35` | Mechanism valid; performance inconclusive |
| AgentX/Weka C32 blind N=100 | `d82302a3769541cd9f98ad91bd8c3a69` | `915dac9e54d54b18b9b5a79ac8f69c2b` | Valid negative policy result |
| AgentX/Weka C64 blind N=100 | `beaf48bcd79d46a1b155ba9af508ec2c` | `6febe03b9d1f4b4e95f628a34e59c038` | Horizon sensitivity supported; no benefit proof |
| V2.1 five-cell selection/allocation | `4111b847dba14ae0a8f6b6617aec939e` | `afce8c...`, `7a5ba9...`, `3deeb0...`, `6ccc8c...` | Accounting valid; live allocation blocked |
| V2.1 bounded reserve | `4111b847dba14ae0a8f6b6617aec939e` | `493bcd6ad3e741789339e9a1a6f6e543` | Harmful; self-eviction/retention confounded |
| V2.1 retention lease | `4111b847dba14ae0a8f6b6617aec939e` | `7d56be15fa04496db9ef76c7b539b235` | Valid harmful treatment; 93.29% wasted |
| V3 V6 JIT demand-safe | `19c4d1be0d0b4bbeb6358da05c32721f` | `5be11650e5a34043a3940c2e57dded74` | Mechanism accounting valid; causal benefit invalid |
| V3.1 V7 continuous batching | `fc55e9b024a345c096cb29a63853559a` | `68ecf349407f4b68a6f232e24e2ad2b6` | Conditionally valid severe negative; treatment no-go |

### Claims that should be corrected

| Previous interpretation | Measured result | Current interpretation |
|---|---|---|
| `max_num_seqs=1` showed performance benefit | Mean throughput +0.08%, mean TTFT -0.12%; provisional p95/p99 tail improvement; invalid/incomplete control | It proved mechanical timeliness under serialization, not production benefit |
| V3 V6 demonstrated a good direction | 50% useful, but node performance diverged during warmup while submissions were zero | It validated state/accounting, not end-to-end utility |
| 48% useful in V7 means the predictor is promising | Useful CPU hits coexisted with 55% throughput loss and 126% mean-TTFT increase | `useful` is not a causal performance metric |
| More queueing is the desired production signal | C64 improved useful/late ratios | Queueing supplies horizon precisely when the server is stressed; it is a diagnostic signal, not a sufficient policy |
| The original 80/20 hot-block/XGBoost framing is evidence-based | No ABC experiment has measured a stable Pareto hot set or trained/evaluated that predictor | This is an inherited hypothesis, not an established fact |

## 3. Root-cause decomposition

### Predictor quality

The current mechanism's main problem is not classic prediction accuracy. Once an HTTP request has arrived, its prompt keys are deterministic and will be demanded if the request reaches execution. V7 predicts **when** the already-known working set will be needed and whether a short prefix is resident in NVMe. The 48.1% useful/submitted ratio is dominated by eviction/retention and timing, not uncertainty about prompt contents.

Blind V1 selection was poor: most keys were already in CPU, absent from the filesystem, or late. V2/V3 residency and frontier checks corrected those selection errors. That improvement did not yield performance.

### Insufficient prediction horizon

Admission-time queue position gives no horizon for requests that fit available sequence slots. At `max_num_seqs=1`, every queued request waits behind a full request and the mechanism receives an artificial, long horizon. At `max_num_seqs=8`, lead time depends on dynamic batch occupancy, prefill/decode mix, external lookup stalls, and completion order. An EWMA of "admission rounds" is only a proxy.

More fundamentally, ordinary HTTP request arrival is late compared with agent/workflow signals. Tool execution, a workflow predecessor, a session keystroke, or a known successor can occur seconds before the next inference request. That is the signal class demonstrated by recent systems and the one AgentX dependencies can expose.

### Transfer/compute overlap

Current ABC proactively moves NVMe -> CPU only. The normal request still needs CPU -> GPU transfer after scheduling. Therefore even perfect V7 execution cannot hide the complete retrieval path.

V7 also slices a 64-chunk bundle into at most eight 8-chunk jobs. With its measured fitted cost, one 8-chunk job costs approximately:

$$
T_8 = 0.239 + 8(0.00164) \approx 0.252\ \text{s}.
$$

The full 64-chunk bundle costs:

$$
T_{64} = 8(0.239) + 64(0.00164) \approx 2.017\ \text{s}.
$$

That is the minimum promotion horizon after residency is known, before lookup queueing, scheduler-step polling, CPU -> GPU transfer, and safety margin. A single 64-chunk bundle is 1,024 tokens or 128 MiB; AgentX requests average roughly 50k-58k input tokens. The bundle covers only a small prefix fraction and the policy stops at first demand.

### Bandwidth

One chunk is approximately 2 MiB and 16 tokens in this Nemotron TP8 deployment. V7 submitted about 4.0 MiB/s and usefully staged about 1.9 MiB/s. Equivalently, it submitted about 32 prompt-token-equivalents/s and usefully staged about 15.6, versus 10,591 input tokens/s reported by the treatment. These rates are not a perfect traffic decomposition, but their orders of magnitude show that V7 covered only a tiny fraction of the active prompt working set. Its treatment NVMe read rate was about 0.520 GB/s. Speculative **data bytes** were therefore far too small to explain a 55% throughput collapse by device saturation alone.

This is important evidence for control-plane, lookup, cache, or node effects. It also means the current mechanism moves too little useful data to create a large throughput improvement.

At the control's observed rates, moving a 50k-token (about 6.1 GiB) working set would require on the order of 7-8 seconds at 0.824 GB/s NVMe -> CPU plus about 1.3-1.4 seconds at 4.54 GB/s CPU -> GPU if done serially. Exact values depend on what fraction is already resident and on queueing. These magnitudes justify an oracle experiment: sufficiently early knowledge could matter, but a 1,024-token CPU-only bundle cannot establish that upside.

### Cache pressure and eviction

V2.1 demonstrated that allocation, reserve, and retention cannot be bolted on independently:

- zero free CPU slots prevented all live submissions;
- reserving capacity enabled promotions but early code self-evicted speculative blocks;
- a one-bundle lease reduced one failure mode but demand-critical pressure reclaimed it;
- V7 still recorded 0.65 evicted-before-demand chunks/s, a large subset of 0.998 wasted chunks/s.

Prefetch, placement, admission, and eviction are one policy problem. A prefetched copy has an opportunity cost: it occupies CPU/GPU capacity that might retain a more valuable demand-owned prefix.

### Scheduler behavior and hot-path interference

The current branch places several operations in or adjacent to the scheduler critical path:

- `OffloadingConnectorScheduler.on_new_request()` calls `_maybe_prefetch_on_admission()`, builds all offload keys, and enters the policy (`.../offloading/scheduler.py`, around lines 809-868).
- `AdmissionPrefetchPolicy.on_request_admitted()` scans up to `max_candidate_chunks=1024` primary keys synchronously (`tiering/prefetch/admission.py`, around lines 319-387).
- `TieringOffloadingManager.on_schedule_end()` invokes `policy.step()` before flushing promotions and tier lookups (`tiering/manager.py`, around lines 1277-1305).
- `AdmissionPrefetchPolicy.step()` performs lifecycle work, scans queued bundles to select an owner, probes secondary residency, gates, records metrics, and submits slices (`admission.py`, around lines 389-683).
- `has_pending_work()` keeps the engine stepping while policy state exists (`admission.py`, lines 773-774; manager lines 1315-1328).

The filesystem metadata path is asynchronous but not isolated. `AsyncLookupManager` has one background lookup worker and one priority queue. Demand batches bypass queued speculative batches, but an in-progress speculative batch is non-preemptible. If demand reaches a key whose speculative lookup has already been flushed, the manager enqueues a duplicate demand-priority lookup (`tiering/async_lookup.py`, around lines 199-220). This is a concrete interference mechanism worth instrumenting. V7's mean asynchronous lookup delay doubled, consistent with it, but the existing data cannot prove causality.

The data-load pool similarly provides queue priority, not preemption. An already-running 8-chunk `batch_load_block()` remains non-preemptible. This is acceptable only if the micro-batch bound is validated against demand SLOs.

### Cost-model incompleteness

`TransferCostModel` learns secondary -> CPU promotion completion time and correctly charges each 8-chunk job's fixed term. The policy gate does not price:

- secondary residency-probe latency and duplicate probes;
- scheduler/frontier-scan time;
- demand lookup interference;
- CPU -> GPU transfer;
- eviction opportunity cost;
- GPU batch perturbation;
- the counterfactual stall actually removed.

The policy can therefore accept work with a positive transfer deadline margin and negative end-to-end utility. V7's low policy deadline-miss share alongside severe regression is direct evidence that its objective is incomplete.

### Implementation bugs versus research limitations

Historical implementation bugs were real and materially invalidated runs: wrong configuration attribute, structurally impossible post-miss selection, unreachable trim accounting, shadow/live mismatch, reserve/bundle inconsistency, self-eviction, multi-slice cost undercharge, and incomplete lifecycle accounting. These are documented negative results, not evidence against the whole hypothesis.

For current V7, no model-output correctness failure is observed. Reactive fallback remains intact and requests complete. The demonstrated failure is performance correctness: the policy does not preserve service quality. A specific current bug causing all of the 55% regression is **not proven**. Cross-node placement and missing GPU/hook metrics prevent that claim.

### Workload and evaluation

AgentX/Weka is the correct production-oriented workload family: long multi-turn coding-agent traces, prefix reuse, tool/subagent dependencies, and trace-derived idle gaps. Its cache-bust mode intentionally removes cross-play reuse, so it tests within-play reuse rather than a globally warm repeated corpus.

The evaluation methodology remains weaker than the effect claims:

- most cells have one repetition and use different nodes;
- cache cleaning/fingerprinting is inconsistent;
- GPU metrics are often absent;
- some earlier runs had incomplete/cancelled requests or JIT in measurement;
- MLflow revision tags have sometimes disagreed with manifests;
- current metrics classify block outcomes but do not measure saved exposed stall.

## 4. Incorrect or weak assumptions

| Assumption | Classification | Decision |
|---|---|---|
| Secondary-tier demand latency is large enough to matter | Supported | Keep; quantify with oracle |
| `max_num_seqs=1` performance validates production prefetch | Contradicted | Use only as a correctness/serialization control |
| A later CPU hit is proof of benefit | Contradicted | Replace with deadline readiness and exposed-stall metrics |
| Queue position at request admission is the best signal | Plausible baseline, weak production signal | Retain as baseline only |
| More queueing means more desirable prefetch opportunity | Partly supported mechanically, contradicted as sufficient utility | Treat overload as a cost as well as a horizon |
| Predicting individual next blocks is necessary | Unnecessary | Predict/rank request or prefix working sets and deadlines |
| First-N contiguous chunks adequately represent a request | Contradicted for long AgentX prompts | Use working-set coverage and staged pipeline |
| NVMe -> CPU prefetch fulfills the project objective | Contradicted/incomplete | Separate storage -> CPU and CPU -> GPU opportunity |
| Transfer time alone is a sufficient cost gate | Contradicted | Use end-to-end utility including contention and eviction |
| One owner is the safest production design | Safe but too restrictive | Replace with bounded in-flight bytes/jobs and EDF priority |
| Agentic workloads are automatically predictable | Plausible but unproven | Measure advisory horizon/recall on real workflow signals |
| The original 80/20 temperature/XGBoost model is justified | Unsupported/inherited | Do not build ML before deterministic/oracle baselines |
| vLLM must own physical placement and movement | Supported | Keep; external orchestrator provides advisory intent only |

## 5. State-of-the-art research map

The external literature changes the novelty and design assessment.

### Directly transferable systems

- [SYMPHONY (NSDI 2026)](https://www.usenix.org/conference/nsdi26/presentation/agarwal) is the closest prior work. It sends advisory requests from user interaction or agent call graphs before inference, moves session KV toward GPU, couples placement with serving-memory pressure, uses layer-wise loading, and tolerates spurious/missed advisories. Its ShareGPT advisories arrive 11.3 seconds early on average; its agent implementation issues advisories for downstream call-graph nodes. A simple "agent DAG triggers KV prefetch" claim is therefore not novel.
- [CachedAttention (USENIX ATC 2024)](https://www.usenix.org/conference/atc24/presentation/gao-bin-cost) uses scheduler-aware fetching/eviction and layer-wise preloading to overlap slow-tier access with GPU computation. This supports working-set/layer scheduling rather than one-shot block promotion.
- [Strata (OSDI 2026)](https://www.usenix.org/conference/osdi26/presentation/xie-zhiqiang) shows that hierarchical KV systems fail when schedulers ignore cache-load latency, when layouts create small I/O, and when concurrent requests produce delay hits. It co-designs large transfers with cache-aware scheduling and complementary work overlap. This maps directly to our lookup-delay and 8-chunk fixed-cost problem.
- [Mooncake (FAST 2025)](https://www.usenix.org/conference/fast25/presentation/qin) treats KV placement and scheduling as one SLO-constrained global optimization over GPU, CPU, SSD, and network resources. It is evidence against an isolated local prefetch knob.
- [KV Cache in the Wild (USENIX ATC 2025)](https://www.usenix.org/conference/atc25/presentation/wang-jiahao) finds skewed reuse, important single-turn as well as multi-turn reuse, and category-conditioned predictability. This supports request-category/session reuse ranking, but not a universal 80/20 claim.

### Agent/workflow-specific prior art

- [KVFlow](https://arxiv.org/abs/2507.07400) represents agent execution as a step graph, uses steps-to-execution for eviction, and prefetches next-step CPU KV to GPU.
- [CacheScout](https://arxiv.org/abs/2608.14624) learns agent transition structure online and applies it to eviction and prefetching outside the serving critical path on vLLM.
- [PBKV](https://arxiv.org/abs/2605.06472) predicts several future workflow steps for dynamic agent graphs and conservatively applies those predictions to eviction and prefetch.
- [InferCept](https://arxiv.org/abs/2402.01869) treats tool/external execution as an interception lifecycle and co-designs swap/recompute/retention decisions.

These works mean "learn the next agent" is not a fresh research direction by itself. A defensible contribution would need a materially different problem—such as deadline- and cost-aware **multi-tier** staging in native vLLM, causal stall attribution, and realistic production workflow signals—or stronger empirical evidence.

### Adjacent but different prefetch problems

- [InfiniGen (OSDI 2024)](https://www.usenix.org/conference/osdi24/presentation/lee) predicts important token KV one transformer layer ahead for long-context decode. [ECHO (OSDI 2026)](https://www.usenix.org/conference/osdi26/presentation/liu-guangda) exploits native sparse-attention index predictability. These have deterministic intra-model compute windows and different accuracy/granularity constraints; they do not validate request-arrival prediction.
- [DirectKV (OSDI 2026)](https://www.usenix.org/conference/osdi26/presentation/luo) is a counterexample to the premise that data must always be prefetched into GPU: on coherent high-bandwidth CPU-GPU systems, zero-copy attention may be stronger.
- [Preble](https://arxiv.org/abs/2407.00023) and [SGLang/RadixAttention](https://papers.nips.cc/paper_files/paper/2024/file/724be4472168f31ba1c9ac630f15dec8-Paper-Conference.pdf) optimize cache-aware request ordering/routing. A ready-prefix scheduler may outperform speculative movement when admission choice is available.

### What still appears underexplored

The promising gap is not generic agent prediction. It is a vLLM-native mechanism that jointly reasons about:

1. exact prefix working sets already known to the scheduler;
2. earlier probabilistic workflow/session advisories;
3. deadlines and tier-specific completion distributions;
4. demand interference and eviction opportunity cost;
5. staged NVMe -> CPU -> GPU readiness;
6. cache-aware admission when staging misses a deadline;
7. counterfactual exposed-stall reduction as the objective.

Novelty must be rechecked after the oracle establishes value.

## 6. Candidate research directions

### A. Deadline-aware request working-set staging and cache-aware admission

Treat known future KV demand as an I/O scheduling problem. For every waiting request, derive a prefix working set, current residency vector, estimated ready deadline, and byte cost. Rank transfers with earliest-deadline-first plus positive expected utility. If a request is not data-ready, schedule complementary ready work rather than allow a delay hit.

Mapping to vLLM:

- scheduler state and deterministic keys: `OffloadingConnectorScheduler`, `vllm/v1/core/sched/scheduler.py`;
- tier residency/promotion: `TieringOffloadingManager`, `AsyncLookupManager`;
- working-set allocation/eviction: `CPUOffloadingManager`, GPU `BlockPool`/KV cache manager;
- execution: current batched secondary loads and CPU GPU worker;
- policy computation: outside the synchronous schedule loop, with only bounded command application on `on_schedule_end()`.

This direction does not require ML and is easy to falsify with an oracle.

### B. Workflow/session advisory staging

Emit a pre-request advisory when a tool starts, a workflow predecessor completes, a subagent branch becomes likely, a session becomes active, or a known successor is released. The advisory identifies session/prefix candidates and an arrival interval. vLLM executes cost-aware staging; an orchestrator/router supplies intent but does not move blocks.

For AgentX, the replay scheduler already knows SPAWN/JOIN dependencies and recorded offsets. This offers an oracle-quality first experiment without inventing a predictor. Production integration could later use LangGraph/AutoGen-style execution events or llm-d session/routing metadata.

This is high-upside but externally crowded by SYMPHONY, KVFlow, CacheScout, and PBKV.

### C. Reuse-likelihood placement and eviction, without proactive movement

Use session/category/transition signals to preserve likely prefixes in CPU/GPU and avoid future transfers. This may capture most benefit with less bandwidth amplification. It should be evaluated before claiming prefetch is necessary.

The policy target is expected future reuse density per byte, conditioned on deadline and recompute/fetch cost—not raw access frequency. Integrate with CPU LRU/ARC and GPU prefix-cache retention.

### D. Cost-aware opportunistic staging during explicit slack

Prefetch only when the data plane reports genuine slack: no demand lookup/load queue, spare transfer engine/worker capacity, and sufficient GPU/CPU memory headroom. Use advisories to rank work but never authorize it solely from probability.

This is safer but may starve under sustained load. It is a useful baseline and a component of A/B, not a complete strategy.

### E. Layer-wise or chunk-pipelined CPU -> GPU staging

Once a request is selected, overlap later-layer/later-chunk CPU -> GPU transfers with useful computation rather than wait for the full prefix. This attacks the part V7 leaves exposed. It is architecturally invasive and should be attempted only if the CPU-resident versus GPU-resident oracle shows a material gap.

### F. Learned next-block/temperature prediction

Train a predictor over recency, frequency, session, and spatial features to rank individual blocks. This is the inherited ABC end-state.

Do not invest now. The target is too fine-grained, current opportunity is unquantified, and external work already supports higher-level deterministic/structural signals. If revisited, predict expected utility or time-to-next-use for a working set, not binary next-block access.

### G. Avoid movement: recompute, route, or directly access

For some tiers/hardware, recomputing a short prefix, routing to the resident endpoint, or directly accessing CPU KV may dominate transfer. The decision should compare these alternatives per request. This is not a failure of ABC; it is a stronger framing of the placement problem.

## 7. Ranked recommendation

| Rank | Direction | Expected upside | Evidence | Novelty | Risk | Implementation cost | Verdict |
|---:|---|---|---|---|---|---|---|
| 1 | Deadline-aware working-set staging + cache-aware admission | High if oracle confirms storage stall is hideable | Internal reactive delays; CachedAttention; Strata; Mooncake | Medium as native multi-tier vLLM co-design | Medium | Medium-high | **PRIMARY BET**, oracle first |
| 2 | Workflow/session advisory staging | High; longest natural horizon | AgentX DAGs; SYMPHONY; KVFlow | Low-medium unless combined with multi-tier deadlines/cost | Medium | Medium | **INVESTIGATE**, likely signal source for #1 |
| 3 | Reuse-likelihood placement/eviction | Medium-high with lower bandwidth risk | V2/V3 eviction failures; KV Cache in Wild; CacheScout/PBKV | Medium-low | Medium | Medium | **INVESTIGATE** as co-design |
| 4 | Explicit-slack opportunistic staging | Medium, robust | V7 byte volume not saturating; demand interference concern | Low | Low | Low-medium | **BASELINE ONLY** |
| 5 | Layer/chunk-pipelined CPU -> GPU staging | Potentially high | CachedAttention, SYMPHONY, InfiniGen; current V7 leaves this stage exposed | Medium | High | High | **INVESTIGATE only after GPU oracle** |
| 6 | Cache-aware routing/recompute/direct-access alternative | Workload/hardware dependent | Preble, Mooncake, DirectKV | Medium | Medium | High cluster-wide | **INVESTIGATE as comparator** |
| 7 | Learned per-block temperature/next-block predictor | Unknown | No internal validation; target likely too fine | Low-medium, crowded | High | High | **KILL for now** |
| 8 | Current queue-EWMA, fixed-64, single-owner V7 | Low | Severe negative V7; tiny coverage | Low | High performance risk | Continued patch cost | **KILL as primary path; retain baseline artifact** |
| 9 | Post-miss same-request forward read-ahead | None under prefix-chain invariant | Phase-1 NVMe structural failure | None | Certain failure | Low | **KILL permanently** |

## 8. Recommended next architecture

The evidence justifies an **experimental architecture**, not yet a production algorithm.

### Data-readiness service

```text
workflow/session advisory (optional, early)
                    |
ordinary request admission + exact prefix keys
                    |
                    v
     bounded intent/event queue (no filesystem work)
                    |
                    v
 off-hot-path planner: residency + deadline + utility
                    |
                    v
  EDF working-set queue with demand-preemptible budgets
         |                         |
         v                         v
   NVMe -> CPU stage          CPU -> GPU stage
         |                         |
         +------ readiness --------+
                    |
                    v
 cache-aware scheduler admits ready/complementary work
```

### Planner contract

The unit is a request prefix/working set, internally represented by chunks. A candidate has:

- request/session identity;
- ordered contiguous key range;
- current tier residency and bytes;
- earliest and latest likely use time;
- probability of use, one for an admitted deterministic request;
- estimated lookup, transfer, polling, CPU -> GPU, and eviction cost;
- request priority/SLO and alternative cost (recompute, route, reactive fetch).

A first utility score can be simple:

$$
U = P_{use}\,E[\text{exposed stall removed}]
  - \lambda_{io}E[\text{demand delay}]
  - \lambda_{evict}E[\text{future miss cost}]
  - \lambda_{mem}(\text{bytes}\times\text{residency time}).
$$

Only positive-utility work enters the EDF queue. ML is unnecessary until simple estimates fail.

### Critical-path rule

The scheduler may:

1. publish a compact immutable intent;
2. consume at most a bounded number of ready planner commands;
3. update readiness/deadline state;
4. make a cache-aware admission choice.

It must not scan thousands of keys, perform storage existence checks, fit models, or rank hundreds of bundles synchronously. Export `on_new_request`, policy, connector, and total schedule-step wall time.

### Data-plane rule

Demand always has strict priority **and a bounded preemption delay**, not merely queue order. Use bounded I/O micro-batches selected from measurements, multiple owners subject to global in-flight-byte/job budgets, cancellation, and explicit deadline expiry. Avoid 8-job fixed-cost amplification when larger reads can run without violating the demand preemption SLO.

### Placement rule

Prefetch and eviction share one controller. A staged copy's retention priority increases as its deadline approaches and falls on expiry. Demand may borrow capacity, but the planner must account for the lost speculative work and must not immediately restage it.

### Correctness contract

The existing reactive path remains authoritative:

`_maximal_prefix_lookup()` -> `TieringOffloadingManager.lookup()` -> normal promotion or recompute.

Speculation changes readiness and scheduling, never token identity or model KV semantics. Any absent, late, cancelled, evicted, or failed staged range falls back reactively.

## 9. Critical experiments

Run these in order. Do not begin the next architecture until the preceding gate passes.

### Experiment 0 — Reproducibility and V7 interference localization

- **Hypothesis:** the severe regression is caused by active policy work, not ordinary node variance.
- **Independent variable:** same-node, counterbalanced arms: reactive; manager/policy constructed but request opt-in absent; key/frontier scan only; speculative lookup only; active promotion.
- **Metrics:** scheduler-step and hook p50/p95/p99, demand/spec lookup queue/service time and duplicates, load queue/service time, GPU SM/memory/PCIe/clocks, TTFT/ITL/throughput.
- **If true:** one ablation introduces lookup/hook/ITL regression.
- **If false:** A/A or node crossover reproduces the large difference without active work.
- **Effort:** 1-2 days after instrumentation.
- **Decision:** identifies a current implementation defect or invalidates V7 causal attribution.

### Experiment 1 — Perfect CPU-residency oracle

- **Hypothesis:** hiding secondary -> CPU fetch materially improves realistic AgentX.
- **Independent variable:** exact future prefix working set pre-positioned in CPU outside the measured scheduler path versus reactive tiering, matched capacity.
- **Metrics:** exposed lookup stall, TTFT, throughput, ITL, CPU hit bytes, evictions.
- **If true:** p95 TTFT or throughput improves materially with no demand interference.
- **If false:** storage -> CPU prefetch is not worth further work for this workload/architecture.
- **Effort:** 1-3 days using trace-known keys or a controlled prewarm phase.
- **Decision:** go/kill the central opportunity.

### Experiment 2 — Perfect GPU-residency oracle

- **Hypothesis:** CPU -> GPU remains a material exposed floor after CPU prepositioning.
- **Independent variable:** exact prefix held GPU-resident versus CPU-resident, with matched effective capacity/pressure.
- **Metrics:** TTFT, CPU -> GPU transfer wait/copy, GPU memory pressure, throughput.
- **If true:** invest in staged/layer-wise GPU loading.
- **If false:** keep scope at secondary -> CPU and scheduler admission.
- **Effort:** 2-4 days.
- **Decision:** determines architecture depth.

### Experiment 3 — Minimum useful horizon curve

- **Hypothesis:** there is a predictable horizon threshold above which exact staging hides transfer.
- **Independent variable:** advisory lead time H in `{0, 0.25, 0.5, 1, 2, 4, 8, 16}` seconds; exact working sets; `max_num_seqs=8`; realistic Weka demand.
- **Metrics:** ready-before-demand bytes, exposed stall saved, late bytes, demand slowdown, utility.
- **If true:** benefit has a knee near measured lookup+transfer+poll cost.
- **If false:** queueing/cache/GPU interactions dominate simple timing.
- **Effort:** 1-2 days once oracle exists.
- **Decision:** sets required advisory horizon and cost model.

### Experiment 4 — Scheduler-known EDF working-set baseline

- **Hypothesis:** deterministic scheduler knowledge plus EDF is sufficient; a behavioral predictor is unnecessary.
- **Independent variable:** FCFS reactive versus queue-rank V7 baseline versus exact-deadline EDF working-set staging, bounded by in-flight bytes.
- **Metrics:** fraction of oracle stall savings realized, deadline accuracy, demand interference, TTFT/throughput.
- **If true:** build the simple production baseline.
- **If false:** identify whether horizon or cost/ranking failed.
- **Effort:** 3-5 days for a minimal experimental planner.
- **Decision:** establishes the non-ML floor.

### Experiment 5 — Agent workflow advisory oracle

- **Hypothesis:** tool/DAG events provide longer and more useful horizons than HTTP admission.
- **Independent variable:** no advisory; request-admission advisory; AgentX predecessor/SPAWN/JOIN advisory; controlled tool pause duration.
- **Metrics:** advisory recall, false positives, horizon distribution, ready bytes, stall reduction, workflow completion time.
- **If true:** integrate advisory API/session metadata.
- **If false:** do not assume agent structure solves timing.
- **Effort:** 2-4 days using replay metadata.
- **Decision:** validates the agentic-specific contribution.

### Experiment 6 — Placement versus movement

- **Hypothesis:** reuse-aware retention captures most benefit with less interference than prefetch.
- **Independent variable:** LRU/ARC; oracle next-use retention; simple session/category retention; retention+prefetch.
- **Metrics:** avoided transfers, hit density/byte, TTFT, bandwidth amplification, eviction regret.
- **If true:** prioritize placement/eviction and use prefetch sparingly.
- **If false:** movement is necessary.
- **Effort:** 3-5 days.
- **Decision:** chooses the balance of policy components.

## 10. Stop/go criteria

### Evaluation/infrastructure gate

- Same image digest and configuration manifest; record actual config, not revision tag.
- Same-node counterbalanced runs where possible, then a node crossover.
- At least three repetitions per accepted arm.
- Clean or fingerprinted NVMe/cache starting state.
- No missing GPU/scheduler/lookup telemetry.
- A/A and no-op policy deltas within 2% at the mean and 5% at tails before interpreting a smaller benefit.

### Opportunity gate

**Kill secondary -> CPU speculative prefetch for this workload/architecture** if the perfect CPU-residency oracle fails to deliver either:

- at least 10% p95 TTFT reduction, or
- at least 5% useful-token/request throughput improvement,

at `max_num_seqs>=8` with TTFT/ITL SLOs preserved across three matched repetitions.

### Mechanism gate

Proceed from oracle to a policy only if it realizes at least 50% of oracle stall savings while:

- keeping mean and p95 ITL within 5% of control;
- keeping demand lookup p95 within 5%;
- adding under 1% scheduler-step time at p99;
- keeping wasted speculative bytes below 10% of demand-read bytes;
- avoiding sustained capacity reclamation/lease churn;
- showing benefit in workflow completion time or TTFT, not only a hit counter.

### Signal gate

Use a workflow/session signal only if its real horizon distribution exceeds the measured minimum useful horizon for at least 80% of the working-set bytes it selects. If a simple deterministic graph/recency/category baseline is within 5% of a learned predictor, do not deploy ML.

### Direction kill rules

- Permanently kill post-miss forward read-ahead under prefix-chained storage.
- Keep blind fixed-N and current V7 only as negative/baseline arms.
- Do not revive per-block XGBoost/temperature prediction unless oracle value is proven and higher-level policies leave material headroom.
- After two redesigns fail the same oracle-realization gate, stop attributing the result to implementation quality and reassess the architecture/hardware opportunity.

## 11. Immediate next actions

1. **Freeze the current branch as an evidence artifact.** Do not add another heuristic to `AdmissionPrefetchPolicy`. Preserve tests and tag the code/run mapping.
2. **Instrument before changing behavior.** Add policy/hook duration; scheduler iteration duration; demand/spec lookup queue, service, duplicate, and batch metrics; load queue/service/bytes; GPU metrics; per-request readiness and exposed-stall timing.
3. **Run the same-node V7 ablation ladder.** This decides whether the current severe regression is a real implementation interference path or cross-node confounding.
4. **Build the smallest perfect-residency oracle.** Exact CPU prepositioning on realistic Weka at `max_num_seqs=8`; no prediction and no max-sequence serialization.
5. **Measure the GPU-residency delta.** Do not design CPU -> GPU prefetch until the oracle proves it matters.
6. **Create a clean experimental planner surface.** Compact intent events, off-hot-path planning, bounded scheduler command application, EDF working-set jobs, and demand-preemptible budgets. Reuse current reactive fallback, async/batched transfer primitives, and exact accounting.
7. **Exploit AgentX dependency metadata as an oracle advisory source.** Measure actual predecessor/tool/SPAWN/JOIN horizons before designing a learned signal.
8. **Compare movement with retention and cache-aware scheduling.** Prefetch is one action in a broader data-readiness policy.
9. **Revisit novelty after Experiments 1-5.** SYMPHONY, KVFlow, CacheScout, PBKV, CachedAttention, and Strata already cover much of the obvious design space.

## Code disposition: keep versus retire

### Keep/reuse

- reactive correctness path in `OffloadingConnectorScheduler._maximal_prefix_lookup()` and `TieringOffloadingManager.lookup()`;
- batched `_flush_pending_promotions()` and asynchronous tier managers;
- demand/speculative queue separation, after measuring preemption delay;
- exact per-key accounting and lifecycle/failure tests;
- transfer-cost observations, expanded to a complete pipeline cost;
- request/tier filters and explicit experiment opt-in;
- reset/cancellation/provenance invariants.

### Retire from the primary design

- `LeadTimeEstimator` queue-round EWMA as the sole deadline oracle;
- fixed first contiguous 64-chunk bundle as the working-set abstraction;
- single speculative owner as the throughput controller;
- reserve/lease behavior as a separate policy instead of integrated placement;
- `useful` as the success criterion;
- admission-time synchronous frontier scans and policy decisions;
- the assumption that the inference HTTP request is the earliest available signal.

## Arche evidence reviewed

Primary ABC materials:

- [[Reports/2026-08-10 - ABC Nemotron no-offload versus CPU-offload KV lookup report]]
- [[Reports/2026-08-14 - Phase 1 CPU prefetch validation]]
- [[Reports/2026-08-14 - Phase 1 NVMe prefetch validation]]
- [[Reports/2026-08-17 - Phase 1 admission prefetch first execution report]]
- [[Reports/2026-08-18 - Phase 1 admission prefetch repaired-image validation]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch first exploratory run]]
- [[Reports/2026-08-18 - AgentX Weka admission prefetch concurrency 64]]
- [[Version2/04 - Theoretical Validation]]
- [[Version2/05 - V2.1 Implementation Record]]
- [[Version2/06 - 2026-08-19 - V2.1 Implementation Deep Dive]]
- [[Version2/Reports/2026-08-19 - V2.1 first five-cell comparison]]
- [[Version2/Reports/2026-08-20 - V2.1 bounded-reserve live validation]]
- [[Version2/Reports/2026-08-20 - V2.1 retention-lease Weka failure]]
- [[Version3/01 - Current JIT Demand-Safe Speculative Prefetch Mechanism]]
- [[Version3/02 - Continuous-Batching Remediation and v7 Implementation]]
- [[Version3/2026-08-21 - V3 JIT demand-safe AgentX comparison]]
- [[Version3/2026-08-21 - V3.1 continuous-batching AgentX v7 comparison]]

Broader Arche materials:

- [[../KV Cache Offloading/Activity-Based KV Cache Offloading|Activity-Based KV Cache Offloading]]
- [[../KV Cache Offloading/Cross-Model Synthesis|Cross-model KV Cache Offloading synthesis]]
- [[../KV Cache Offloading/AgentX Workload Definition|AgentX Workload Definition]]
- [[../../Engineering/Learnings/vLLM KV block prefetch architecture|vLLM KV block prefetch architecture]]
- [[../../Engineering/Learnings/vLLM KV offload retrieval path - lookup, promotion, and load|vLLM KV offload retrieval path]]
- [[../../Engineering/Learnings/vLLM and llm-d-router KV cache responsibility split|vLLM and llm-d-router responsibility split]]

## Final decision

ABC should continue, but its thesis should change from **"predict hot blocks and prefetch N of them"** to **"make future request data ready by its deadline, using the earliest reliable structural signal and a scheduler/placement policy that can prove it removes exposed stall."**

The current mechanism is a valuable failed baseline. It should not be the foundation that every future version patches.