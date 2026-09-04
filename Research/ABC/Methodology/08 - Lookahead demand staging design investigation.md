---
title: "Lookahead demand staging — design investigation for eager KV block loading"
date: "2026-09-03"
type: "research-design"
experiment: "ABC — Activity-Based KV Cache Tier Placement"
status: "proposed"
codebase: "vllm-project/vllm main @ 2db1c4dc31 (2026-09-03)"
scope: "Native OffloadingConnector: CPUOffloadingSpec and TieringOffloadingSpec; scheduler integration"
---

# Lookahead demand staging — design investigation for eager KV block loading

## 0. Verdict

The first iteration should not be a predictor and should not be a new
prefetch policy object. It should be the **existing reactive retrieval path,
started earlier and priced correctly**:

1. **M0 — memoized probe and stage telemetry.** Cache the per-request lookup
   verdict across scheduler steps and record the timestamp of every stage of
   the retrieval pipeline. This removes an O(chunks) rescan the scheduler
   already pays every step for every waiting request, and it produces the
   exposed-stall measurement every later decision depends on.
2. **L1 — CPU-side lookahead past the allocation barrier.** After the waiting
   loop stops (GPU blocks, token budget, or sequence slots), probe the next
   `K` waiting requests in scheduling order with the same demand lookup, so
   their secondary-tier existence checks and NVMe→CPU promotions start while
   they queue. Promotions are admitted under a **do-no-harm gate** (Cao,
   Felten, Karlin, Li 1995): never evict a CPU chunk that a request ahead in
   the queue will demand, and never promote when only such chunks are
   evictable.
3. **L2 — budget-independent GPU staging (flag, default off).** Let the
   existing zero-token async-load admission (`WAITING_FOR_REMOTE_KVS`) run
   even when the compute batch is full, bounded by GPU headroom, so
   CPU→GPU DMA overlaps the batch instead of starting after a slot frees.
4. **R1 — fix the reactive path's own bandwidth.** One NVMe→CPU promotion
   job is one sequential thread; measured aggregate read rates in the ABC
   runs were about 0.6 GB/s against a device that should do several GB/s.
   Eliminating that work is worth more than hiding it, and it lowers the
   lead time every prefetch policy needs.

The expected effect of L1/L2 is regime-dependent and small on the ABC AgentX
runs (order of a few percent of mean TTFT, more at the tail), zero where no
request queues or no request has secondary-tier data, and by construction
never negative if the do-no-harm gate and the per-step cost bound hold. That
matches the brief: a small, correct improvement with no regression.

Speculative session-continuation staging ("between tool calls") is the
highest-upside agentic signal but should wait for the upstream hint surface
(RFC #52113 / #51428); the engine-side action it needs is exactly L1/L2 driven
by an external intent instead of queue position. Dynamic CPU pool growth is
not required for any of this and should not be built now.

## 1. System reconstruction

Facts below were read from the code at `2db1c4dc31`; file references are to
that tree.

### 1.1 Execution path for an offload hit

| Stage | Where | What happens |
|---|---|---|
| Admission | `vllm/v1/engine/core.py:463`, `vllm/v1/core/sched/scheduler.py:2397` | `Request` is built with all prompt block hashes (`vllm/v1/request.py:219-224`), enqueued, and `connector.on_new_request` registers `RequestOffloadState`. No lookup happens here. |
| Waiting loop | `scheduler.py:786-812` | Entered only while `token_budget > 0`, `input_budget > draft_slots`, and `len(running) < max_num_running_reqs`. |
| Lookup | `scheduler.py:846-868` → `offloading/scheduler.py:937-992` → `_lookup` | Local prefix hit, then `get_num_new_matched_tokens`. `_maximal_prefix_lookup` calls `manager.lookup` once per chunk until the first `MISS`; `HIT_PENDING`/`RETRY` defer the request (returns `None`); `_touch` moves every key to MRU. |
| Tiering lookup | `vllm/v1/kv_offload/tiering/manager.py:337-418` | Primary (CPU) dict probe; on miss, secondary tiers in config order; FS tier answers `RETRY` on first sight (async existence check, `async_lookup.py`), `HIT` next step; a secondary `HIT` immediately allocates a CPU slot (`ref_cnt = -1`) and defers the I/O to `_flush_pending_promotions` at end of step. |
| Deferral | `scheduler.py:870-876` | `None` → request is popped and prepended to `step_skipped_waiting`; the loop **continues** with the next waiting request. |
| Allocation | `scheduler.py:1085-1106` | `allocate_slots(delay_cache_blocks=True, reserved_blocks=inflight)`; `None` → **`break`** (head-of-line: no request behind it is examined this step). |
| Pin + job | `offloading/scheduler.py:994-1105` | `prepare_load` pins CPU chunks (`ref_cnt += 1`); one `TransferJob` per request; `_chunks_being_loaded` dedups shared prefixes. Request → `WAITING_FOR_REMOTE_KVS`, joins `_inflight_prefills`, consumes **zero** token budget and no `running` slot (`scheduler.py:953-956`, `1135-1165`). |
| DMA | `offloading/worker.py:235-246` → `cpu/gpu_worker.py:512-682` | One batched `swap_blocks_batch` per job on a pooled stream; each load waits on the previous load's end event, so loads are FIFO per direction. |
| Completion | `worker.py:262-297` → `scheduler.py:2922-2945` → `2886-2901` | `finished_recving` → next `schedule()` promotes the request back to `WAITING` with `num_computed_tokens` set; `cache_blocks` publishes the loaded blocks to the GPU prefix cache. |

Minimum step count (established in [[../../Engineering/Learnings/vLLM KV offload retrieval path - lookup, promotion, and load|the retrieval-path learning]] and confirmed
here): a CPU hit costs the lookup step plus one to two steps for DMA and
completion polling; a secondary hit costs one more step for the async
existence check, one for the end-of-step promotion flush, `k` steps of tier
I/O, and then the CPU path.

### 1.2 Ownership, concurrency, memory

- Scheduler thread owns all manager state; secondary-tier I/O threads write
  into the shared `/dev/shm` mmap; the worker process DMAs from it. No KV
  bytes cross a process boundary.
- The CPU pool is fixed at init: `SharedOffloadRegion` sizes the mmap from
  `cpu_bytes_to_use`, and the whole range is `cudaHostRegister`ed
  (`cpu/gpu_worker.py:193-220`). Growth would require a remap in every
  process plus re-registration.
- Pinning: `ref_cnt > 0` blocks are never evicted; `ref_cnt = -1` means a
  write is in flight and reads as `HIT_PENDING`.
- One in-flight load per request, or one-or-more stores, never both
  (`offloading/scheduler.py:963-968`, `1481-1484`).
- GPU-side: async-load allocations are counted in
  `_inflight_prefill_reserved_blocks` so later admissions cannot consume
  blocks an in-flight prefill relies on (`scheduler.py:2836-2841`).
- Stores are deferred one step so they never delay sampling
  (`worker.py:248-260`).

### 1.3 Cost of the lookup itself (deduction, not measured)

`TieringOffloadingManager.lookup` per key does two `time.monotonic()` calls,
a dict probe, and `_metrics.on_lookup` bookkeeping (`tiering/metrics.py:80`).
A 50k-token AgentX prompt at 16-token chunks is about 3.5k keys. At a few
microseconds per key, one probe costs roughly 5–15 ms, and the current
scheduler repeats it **every step for every waiting request that is
examined**, plus an O(keys) `_touch`. With five or six waiting requests the
scheduler could be spending tens of milliseconds per step on rescans. The
working-set oracle report measured admission probing at p90 47.6 ms per
request, consistent with this order of magnitude. This is why M0 comes
first.

### 1.4 The NVMe promotion job is sequential

`FileSystemTierManager.submit_load` enqueues **one task** per job
(`fs/manager.py:233-269`, `enqueue_load(job_id, 1, [load_task])`), and the C
extension reads the batch in a single `for` loop under one
`Py_BEGIN_ALLOW_THREADS` (`csrc/fs_io.cpp:238-249`). A promotion for one
request therefore runs on one thread with one outstanding I/O at a time.
The 64 read threads only parallelize across requests. This is consistent
with the ABC observation of about 0.6 GB/s aggregate NVMe reads and a
p50/p90 time-to-ready after first lookup of 1.32 s / 10.76 s.

### 1.5 What is established, assumed, unknown

| Status | Claim |
|---|---|
| Fact | Lookups for waiting requests are gated by batch budget, sequence slots, and the first allocation failure (head-of-line). |
| Fact | Async-load admission consumes no token budget and no running slot, and reserves its GPU blocks against later admissions. |
| Fact | A secondary hit costs at least three extra steps before the CPU path starts. |
| Fact (ABC C32 corpus) | Admission-to-first-lookup horizon median 7.48 ms, p95 25.15 ms: at concurrency 32 there is no queue lead time to hide anything. |
| Fact (ABC C64) | Mean waiting depth ~6; admission-to-first-lookup p50 1.21 s, p90 8.73 s; time-to-ready after first lookup p50 1.32 s, p90 10.76 s. |
| Fact (ABC COSTAR) | Only 42/901 C32 requests and 789/1280 C64 requests reuse external KV at all; clairvoyant retention avoids 12/12 (C32) and 144/212 (C64) native reads. |
| Assumption | Per-key lookup cost of a few microseconds; unmeasured. |
| Unknown | What the 8.0 s "post-pressure reuse" on the A30 campaign is made of (DMA, steps, partial recompute). M0 telemetry answers this. |
| Unknown | Whether NVMe read rate is bound by the single-thread job or by the device/filesystem. R1's first experiment answers this. |

## 2. The underlying problem

Strip the feature away and the problem is **integrated prefetching and
caching over a partially known reference string across a three-level
hierarchy** (GPU, CPU, secondary):

- The reference string is the sequence of chunk keys demanded by requests in
  scheduling order. Its prefix is known exactly (waiting queue, FCFS or
  priority order, deterministic prompt hashes). Beyond the queue it is
  unknown; recency (LRU/ARC) is the estimator.
- Scarce resources: CPU slots (fixed), GPU blocks (fixed, shared with
  decode growth), secondary read bandwidth, PCIe bandwidth, and scheduler
  thread time (every microsecond per step is critical path).
- Decisions: when to fetch a chunk (secondary→CPU, CPU→GPU), which chunk,
  and which resident chunk to displace. These are one decision, not three.
- Hard constraints: correctness invariants above; no stall added to running
  requests; no preemption caused by staged blocks; bounded per-step cost.
- Objective: minimize exposed stall on the request critical path
  (admission → first token), measured, not inferred from hit counters.

Two formulations were tried in ABC and failed: "predict hot blocks and
prefetch N" (no signal at admission) and "stage the whole working set for
one owner" (coverage cannot be reached with 0.6 GB/s and seconds of lead
time). The formulation above shifts the objective to *never fetch or evict
worse than demand would*, then hide what lead time exists.

## 3. Prior work and structural analogies

**Cao, Felten, Karlin, Li — A Study of Integrated Prefetching and Caching
Strategies (SIGMETRICS 1995).** Model: known reference string, cache of `k`
blocks, fetch time `F`. Four rules any optimal policy obeys:

1. *Optimal prefetching*: a prefetch brings in the next missing block in the
   reference stream.
2. *Optimal replacement*: it discards the block whose next reference is
   furthest.
3. *Do no harm*: never discard A to prefetch B if A is referenced before B.
4. *First opportunity*: never delay a prefetch-and-replace that could have
   been performed earlier.

`aggressive` (fetch the next missing block as soon as a legally replaceable
block exists) is within $\min(1 + F/k, 2)$ of optimal; `conservative` (same
replacements as offline MIN) within 2×. The paper also notes that ad hoc
throttling ("at most N prefetched-but-unreferenced blocks") "doesn't always
work well; the right approach is to follow the rules."

Correspondence: reference string ↔ queued requests' chunk keys in scheduling
order; cache ↔ CPU tier (`k` = `num_blocks`); `F` ↔ NVMe→CPU promotion time;
replacement ↔ `CachePolicy.evict`. What transfers: rules 1 and 3 directly;
rule 2 in the "LRU-sensible" form (evict LRU among blocks not referenced by
any queued request). What does not transfer: unit-time references, single
fetch at a time (we have parallel I/O), a single cache level (GPU prefix
cache also satisfies references), and a fully known string. The ABC
failures are explained by this paper: V2–V7 relied on owners, reserves and
leases (throttling) and violated rule 3 (586/966 ordinary CPU victims later
demanded; 2,684/4,670 proactive evictions regretted).

**CachedAttention / AttentionStore (USENIX ATC 2024).** Scheduler-aware
fetching from the job queue into host memory, layer-wise preloading to
overlap with compute. Direct precedent for queue-driven staging; up to 87%
TTFT reduction in multi-turn serving.

**SGLang HiCache.** Prefetch from L3 storage to L2 host when the matched
prefix exceeds 256 tokens; termination policies `best_effort` (stop waiting
the moment the GPU could prefill), `wait_complete`, `timeout` (2 s + 0.1 s
per 1k tokens, max 30 s). Precedent for the demand-cutoff semantics: staging
must never delay admission.

**LMCache.** Hint-based prefetch from disk/remote into pinned CPU RAM, with
LRU making room. Same tier and same failure mode (CPU pressure).

**KVFlow (arXiv 2507.07400), SYMPHONY (NSDI 2026), CacheScout (arXiv
2608.14624).** Agent-step-graph or advisory-driven prefetch of next-step KV
from CPU to GPU in background threads, steps-to-execution eviction. These
are the "between tool calls" mechanisms; all rely on a signal that arrives
before the HTTP request. [[../Continuation Readiness/06 - A4 offline Weka summary and decision|The A4 audit]] found the Weka traces carry no such
signal, and upstream RFCs #52113 (`agent_hint` with `prefetch`) and #51428
(`KvHint`) are defining the surface.

**Upstream vLLM.** Issue #41784 documents the same gating problem for
LMCache ("prefetch only triggered inside the waiting loop, skipped when the
running queue consumes the budget") and converges on a bounded early pass
over the first N waiting requests with cached lookup results. PR #54366
(waiting-queue-informed GPU eviction) is the GPU-side rule 3. No open PR
implements L1/L2 for the native connector (checked 2026-09-03).

## 4. Solution families considered

| Family | Verdict | Why |
|---|---|---|
| A. Predictive per-block prefetch (temperature, XGBoost) | Reject | No admission-time signal (exact-key history absent for 100% of valuable C32 arrivals); killed by the 2026-08-21 audit. |
| B. Whole-working-set staging for one owner | Reject for now | Coverage cannot be reached: 0.76% of requests fully ready at first lookup; bandwidth-bound. |
| C. Post-miss forward read-ahead | Reject permanently | Prefix-chained hashes: a later key cannot exist after an earlier terminal miss. |
| D. CPU reserve / retention lease | Reject | Throttling instead of rules; 93% wasted, −61% throughput. |
| E. Reactive path started earlier (L1/L2) | **Adopt** | Deterministic keys, existing invariants, bounded cost, do-no-harm gate. |
| F. Eliminate work in the reactive path (R1, memoization) | **Adopt** | Lowers `F`; benefits every regime; no policy risk. |
| G. Placement/retention (queue-informed CPU and GPU eviction) | Adopt as the gate in E, full form later | COSTAR shows retention headroom ≥ prefetch headroom; PR #54366 covers GPU. |
| H. Session-continuation staging on hints | Later | Needs the hint surface; engine action is E driven by intent. |
| I. Dynamic CPU pool growth | Do not build | Capacity is not the binding constraint under rule 3; pinned RAM cost up front; multi-process remap. |
| J. Layer-wise or chunk-pipelined CPU→GPU load | Later, after M0 shows the CPU→GPU stage is material | Invasive in the model runner. |

## 5. Recommended design: lookahead demand staging

### 5.1 M0 — memoized probe and stage telemetry

Add to `RequestOffloadState` a probe cache: `(probe_step, verdict)` where
verdict is one of `deferred(reason)`, `hit(num_tokens)`, `miss`. The
connector recomputes a request's verdict only when

- the request has never been probed,
- a promotion or load job carrying any of its keys completed
  (`update_connector_output` / `_process_finished_jobs` already see these
  keys), or
- `probe_step + PROBE_TTL_STEPS < current_step` (safety valve, default 8).

`get_num_new_matched_tokens` at real admission always revalidates: the
existing `prepare_load` asserts remain the correctness backstop, and the
revalidation is a scan that stops at the first change. `_touch` runs once per
verdict, not once per step, which also stops repeated touches from inflating
ARC's frequency list (`policies/arc.py:75-87`).

Telemetry per request: `t_admit`, `t_first_probe`, `t_verdict_hit`,
`t_alloc`, `t_dma_submit`, `t_dma_done`, `t_scheduled`, and per key the
tier that served it. Export histograms of each adjacent gap. This is the
"exposed stall" metric the audit asked for and it replaces `useful` as the
success criterion.

Hot-path cost: negative (removes rescans). Memory: one small record per
waiting request.

### 5.2 L1 — CPU-side lookahead with a do-no-harm gate

In `Scheduler.schedule()`, after the waiting loop has stopped for any
reason other than an empty queue, call `connector.probe_waiting(reqs)` with
the next `K` requests in scheduling order that are in `WAITING`, have
`num_computed_tokens == 0`, and are not already probed this epoch. The
connector runs `_lookup` for each (memoized per 5.1) and **does not
allocate GPU blocks**. Side effects are exactly the demand path's: async
existence checks are batched, LRU keys are touched, and secondary hits
initiate promotions.

The gate lives in `TieringOffloadingManager._initiate_promotion`. Each probe
records the CPU-resident chunk count of requests earlier in scheduling
order. A promotion for request $i$ is allowed only if

$$
\text{evictable\_blocks} - \text{protected\_ahead}(i) \ge \text{blocks\_needed}
$$

where $\text{protected\_ahead}(i)$ is the resident chunk count of requests
ahead of $i$ in the queue plus chunks pinned by in-flight jobs. The CPU
policy already receives a `protected` set in `evict()`; L1 passes the union
of the keys of requests ahead of $i$ (bounded: at most `K` requests' key
lists). This is rule 3 restricted to the known prefix of the reference
string, with LRU as the estimator for the unknown tail (Cao's
"LRU-sensible"). No reserve, no lease, no owner.

Demand cutoff: if request $i$ reaches real admission while its lookahead
promotion is in flight it sees `HIT_PENDING` and defers exactly as it would
have on the reactive path; no new wait is introduced.

I/O priority: lookahead jobs are enqueued behind demand jobs in the FS pool
(a third deque in `DualQueueThreadPool` drained after `_load_q`). With one
thread per job this only matters when more than `n_read_threads` jobs are
in flight, but it is the interference channel the V7 audit flagged, so it
is closed explicitly.

Hot-path cost: at most `K` memoized probes per step; with memoization the
steady-state cost is one dict lookup per request per step. Default `K = 4`.

Expected gain: for a request that waits $W$ seconds behind the allocation
barrier and has a secondary-tier working set with promotion time $P$, TTFT
falls by $\min(W, P)$ minus one step. On the C64 corpus that is a minority
of requests with $P$ of seconds and $W$ p50 ≈ 1.2 s, p90 ≈ 8.7 s; a few
percent of mean TTFT and more at p90/p95 is the realistic ceiling until R1
lowers $P$. On C32 the gain is zero and the cost is zero.

### 5.3 L2 — budget-independent GPU staging (flag, default off)

Allow the `load_kv_async` admission branch to run for probed requests with
a `hit` verdict when the waiting loop stopped because of `token_budget`,
`input_budget`, or `max_num_running_reqs`, but **not** when it stopped
because `allocate_slots` failed (that is the block-limited regime; nothing
to gain). Conditions:

- `allocate_slots` succeeds with `reserved_blocks = inflight + staging_headroom`,
  where the headroom approximates decode growth during the load
  (conservative default: one block per running request per step of
  expected load time).
- At most `S` staged requests beyond what the batch can admit (default 1).
- The GPU do-no-harm rule for prefix-cache blocks is approximated by
  requiring the free pool to hold at least `blocks_needed` blocks without
  scanning cached candidates; PR #54366 exposes the primitive
  (`has_cached_block_in_first_n`) and L2 should build on it when merged.

Expected gain: the whole CPU→GPU stage (DMA plus one to two steps) for
requests queued behind long prefills or a full sequence-slot table, with GPU
memory to spare. This is issue #41784's regime and the regime the A30
campaign hints at ("post-pressure reuse 8.0 s vs immediate repeat 2.31 s"),
but the ABC C64 regime is block-limited and gains nothing, which is why L2
ships off by default and is validated separately.

### 5.4 R1 — parallelize the promotion job

Split a promotion job into `ceil(len(keys) / chunks_per_task)` tasks
(`JobState` already aggregates multi-task jobs), default 32 chunks per
task, so a 3.5k-chunk request uses many read threads. Keep partial-failure
accounting per task. Validate first with a one-request microbenchmark:
single-thread vs 8-way vs 32-way read of 7 GiB of 2 MiB O_DIRECT files on
the target NVMe. If the single-thread rate is already at device capability,
R1 is dropped.

### 5.5 What this deliberately does not do

- No speculative keys: every key staged belongs to an admitted request.
- No CPU reserve, lease, or owner; capacity decisions go through `evict()`
  with a protected set.
- No policy work at `on_new_request`; all lookahead is in `schedule()`
  after the demand loop, bounded by `K`.
- No change to hashing, keys, transfer kernels, or the one-load-per-request
  invariant.

## 6. Adversarial analysis

| Scenario | What happens | Why it is safe |
|---|---|---|
| CPU tier thrashing (working sets ≫ capacity) | Gate finds `evictable − protected_ahead < needed`; no promotion. | Rule 3; degenerates to the reactive path. |
| Queued requests share a prefix (subagent fan-out) | Second probe sees `HIT_PENDING` and is memoized as deferred; `_chunks_being_loaded` gates the load as today. | Existing invariant. |
| Request aborted while its lookahead promotion is in flight | Promotion completes into an evictable block; `on_request_finished` clears lookup state. | Same as an aborted reactive promotion. |
| Priority scheduling reorders the queue | Probe order follows `_select_waiting_queue_for_scheduling`; a reorder changes `protected_ahead` next step; a promoted chunk for a demoted request is merely evictable. | Rule 3 is evaluated at promotion time; later reorders cannot cause a stall. |
| Burst of arrivals | `K` bounds work per step; the rest are probed when they enter the window. | Bounded per-step cost. |
| `reset_cache` (sleep/wake, weight update) | `_pending_load_submissions` and verdict caches are cleared; stale jobs are filtered by `_stale_job_threshold`. | Existing reset contract; verdict cache must be cleared in `reset_cache`. |
| L2 stages blocks, then decode growth needs them | Headroom under-estimated → running request preempted. | The only true regression path; `S = 1`, conservative headroom, and the falsification test in §8 guard it; L2 is off by default. |
| Async scheduling (batch queue) | `schedule()` runs one step ahead; probes see state one step stale. | No correctness dependence on freshness; memoization TTL bounds staleness. |
| Hybrid / SWA / Mamba / EAGLE groups | `_lookup` already handles them; L1 reuses it unchanged. | Unlike the ABC branches, no single-group restriction. |
| Tier filter per request (`kv_load_tiers`) | Probes honor `ReqContext.load_tier_filter`. | Existing path. |

## 7. Established versus novel

Established: rules 1–4 (Cao et al.), scheduler-aware fetching
(CachedAttention), demand cutoff (HiCache best-effort), bounded early pass
over the waiting queue (issue #41784 discussion), all existing vLLM
invariants.

Novel to this system, and to be treated as hypotheses:

- applying the do-no-harm rule to a **two-level** hierarchy where the
  reference is satisfied by either GPU or CPU residency and only a prefix of
  the reference string is known;
- the `protected_ahead` accounting derived from memoized verdicts of
  earlier-queued requests as a near-free implementation of rule 3;
- stage-attributed exposed-stall telemetry as the acceptance metric.

## 8. Validation plan and falsification

Order matters; each gate is a stop/go.

1. **M0 alone, A/A.** Same node, three repetitions, AgentX C32 and C64,
   stock reactive vs M0. Pass: scheduler step p99 not worse; TTFT within
   the 2% mean / 5% tail A/A band; the stage histograms populated. This also
   measures how much per-step time the rescans currently cost.
2. **R1 microbenchmark, then R1 e2e.** Falsified if single-thread read rate
   equals multi-thread rate; otherwise expect time-to-ready after first
   lookup to fall proportionally on C64.
3. **L1 with K ∈ {0, 2, 4, 8}.** Metrics: exposed stall per stage,
   promotions gated/admitted, victims later demanded (regret), demand
   lookup p95, ITL. Falsified if regret rises above the reactive baseline or
   if any TTFT/ITL mean or tail worsens beyond the A/A band. Accept if p90
   TTFT for secondary-hit requests falls and everything else is flat.
4. **L2 in a budget-limited regime.** Synthetic: `max_num_seqs` small or
   `max_num_batched_tokens` small with long prefills, GPU KV ≤ 70%
   occupancy; then C64. Falsified if preemption rate rises at all, or if
   the block-limited C64 shows any change (it should be inert there).
5. **Regime map.** Report gains as a function of measured lead time $W$
   and promotion time $P$; the design predicts $\min(W, P)$; a systematic
   shortfall points at step quantization or I/O queueing, which the stage
   histograms localize.

Acceptance for iteration 1: no metric worse than the A/A band on any run,
and a measured reduction of exposed stall on at least the secondary-hit
subset. Hit-count ratios are not acceptance evidence.

## 9. Answers to the specific questions in the brief

**"CPU physical blocks cannot be expanded once the engine runs."** True
(`SharedOffloadRegion` fixed at init and fully `cudaHostRegister`ed), but
it is not the binding constraint: under rule 3 a prefetch only ever
replaces a block that would be evicted later anyway, so capacity growth
buys hit rate, not prefetch safety. If elasticity is wanted later, the
viable route is virtual over-provisioning of the mmap with a soft usable
block cap raised at runtime and slab-wise host registration on demand,
which costs a scheduler→worker growth message; it is not needed for
iteration 1.

**"Agentic, between tool-call turns."** The signal that makes this work
arrives before the HTTP request (tool start, workflow predecessor) and is
absent from the traces ABC has; upstream is defining the hint surface. The
engine-side action a hint needs is L1/L2 driven by an intent instead of
queue position, plus soft retention of the finished turn's chunks in CPU.
Build L1/L2 first so the hint has something correct to drive.

**"GPU or CPU."** CPU-side lookahead (L1) is safe in every regime and is
the first thing to ship. GPU-side staging (L2) is only beneficial in
budget-limited regimes with free GPU memory and carries the single real
regression risk (preemption), so it is a flag validated on its own.

## 10. Open unknowns and the smallest experiments

| Unknown | Smallest experiment |
|---|---|
| Per-step cost of the current rescans | M0 A/A with scheduler step timing. |
| Whether NVMe rate is thread-bound | R1 microbenchmark, one request, one node. |
| Composition of the 8.0 s post-pressure reuse on A30 | M0 stage histograms on the A30 exact-prefix reuse sequence. |
| Fraction of C64 requests with $W \ge P$ | Join M0 telemetry with the COSTAR C64 corpus. |
| Whether ARC's touch semantics need a lookahead-specific touch | Compare ARC vs LRU under L1 on the A30 pressure sequence. |

## Sources

- Code: `vllm-project/vllm` at `2db1c4dc31` (files cited inline).
- Cao, Felten, Karlin, Li. *A Study of Integrated Prefetching and Caching
  Strategies.* SIGMETRICS 1995. https://homes.cs.washington.edu/~karlin/papers/sigmetrics.pdf
- Gao et al. *CachedAttention.* USENIX ATC 2024. https://arxiv.org/abs/2403.19708
- SGLang HiCache design. https://docs.sglang.io/advanced_features/hicache_design.html
- LMCache CPU RAM docs. https://docs.lmcache.ai/kv_cache/cpu_ram.html
- KVFlow. https://arxiv.org/abs/2507.07400 · CacheScout. https://arxiv.org/abs/2608.14624
- vLLM blog, KV offloading connector. https://vllm.ai/blog/2026-01-08-kv-offloading-connector
- vLLM issue #41784, PR #54366, RFCs #52113 and #51428.
- Arche: [[../2026-08-21 - Independent research audit and redirection for speculative KV prefetching|Independent research audit]],
  [[../2026-08-23 - ABC prefetch research brief for feedback|research brief]],
  [[../Reports/2026-08-23 - Working-set oracle AgentX first comparison|working-set oracle comparison]],
  [[../Future-Value Placement/00 - Index|Future-Value Placement]],
  [[../Continuation Readiness/06 - A4 offline Weka summary and decision|A4 decision]],
  [[../Campaigns/2026-08-29 - A30 COSTAR campaign final update|A30 COSTAR final update]],
  [[../../Engineering/Learnings/vLLM KV offload retrieval path - lookup, promotion, and load|vLLM KV offload retrieval path]].