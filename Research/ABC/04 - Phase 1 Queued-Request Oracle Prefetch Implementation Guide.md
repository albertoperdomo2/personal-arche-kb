---
title: "Phase 1 — Queued-Request Oracle Prefetch Implementation Guide"
date: "2026-08-16"
type: "implementation-guide"
experiment: "Activity-Based KV Cache Tier Placement"
team: "PSAP"
phase: "1 — naive proactive prefetching proof of concept"
status: "tutorial-ready"
supersedes-policy-in: "[[02 - Phase 1 Naive Prefetch Implementation Guide]]"
depends-on: "[[2026-08-14 - Phase 1 queued-request oracle prefetch plan]]"
validation: "[[2026-08-14 - Phase 1 NVMe prefetch validation]]"
codebase: "vLLM experimental/naive-proactive-prefetching @ 4614530b15 plus uncommitted old-policy cleanup"
repository: "/Users/aperdomo/workspace/redhat/vllm"
benchmark: "guidellm-nemotron-nvme-prefetch-poc"
---

# Phase 1 — Queued-Request Oracle Prefetch Implementation Guide

## How to use this guide

This is a tutorial for implementing the proof of concept yourself. Read it in order. Each implementation step starts by explaining the relevant existing code, then identifies the exact file and current line anchor, proposes a small change, explains the choice, and ends with a checkpoint.

The code snippets are intentionally implementation-sized, but they are not a substitute for reading the surrounding functions. Type them and adapt them rather than pasting the whole guide blindly. After each checkpoint, inspect the state in a debugger or with focused tests before continuing.

The line numbers were captured on **2026-08-16** from:

```text
/Users/aperdomo/workspace/redhat/vllm
branch: experimental/naive-proactive-prefetching
HEAD:   4614530b15
```

The worktree also contains the uncommitted cleanup that removed the rejected post-miss implementation. As soon as you add lines, later line numbers will move. Use the named class or function as the durable anchor; the line number tells you where it is in the starting tree.

Do not commit merely to advance through this tutorial. Keep each step reviewable with `git diff`, and commit only when you understand and are satisfied with the complete change.

## 1. Starting state

The repository is intentionally between implementations.

The rejected policy is gone:

- no `prefetch_chunks` configuration;
- no post-`MISS` read-ahead in `_maximal_prefix_lookup()`;
- no `_try_promote()` membership selector;
- no public `prefetch()` that queries secondary tiers.

The reusable substrate remains:

- CPU slot reservation through `_initiate_promotion()`;
- per-request/per-tier batching in `_pending_load_submissions`;
- asynchronous `submit_load()` and completion polling;
- the reactive demand path;
- useful/wasted/untracked outcome tracking;
- prefetch metric names and definitions.

The current uncommitted cleanup modifies these four files:

```text
tests/v1/kv_offload/tiering/test_tiering_offloading.py
vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py
vllm/v1/kv_offload/tiering/manager.py
vllm/v1/kv_offload/tiering/spec.py
```

This is the correct place to begin: old candidate selection is absent, while the expensive and delicate transfer machinery is still present.

## 2. What we are trying to prove

The research question is narrower than “can vLLM predict future KV reuse?”:

> If the identities and NVMe residency of a queued request’s blocks are known by construction, can starting NVMe→CPU promotion before demand reduce TTFT and improve pipeline throughput?

The benchmark supplies the oracle:

1. deterministic warm-up requests populate NVMe;
2. GPU and CPU cache state are churned or restarted while NVMe persists;
3. the measured phase replays the same prompts;
4. measured requests explicitly opt into prefetch;
5. vLLM blindly promotes their first `N` known keys from NVMe without checking NVMe membership.

For target request `i`:

$$
TTFT_{control} \approx Q_i + L_{NVMe\rightarrow CPU} + L_{CPU\rightarrow GPU} + L_{prefill}
$$

$$
TTFT_{prefetch} \approx Q_i + \max(0, L_{NVMe\rightarrow CPU} - Q_i) + L_{CPU\rightarrow GPU} + L_{prefill}
$$

`Q_i` is the queue interval while an earlier request occupies the only sequence slot. The proof succeeds when NVMe work overlaps `Q_i` instead of starting on the target request’s critical path.

## 3. Why the old policy was wrong

The old hook ran after a terminal demand `MISS` and selected later keys. vLLM block hashes are prefix-chained, and the filesystem tier is append-like. Under ordinary storage behavior, a later stored prefix block implies its predecessor was also stored. Therefore, once the predecessor is a resolved terminal miss, the later candidates are normally absent too.

Changing filesystem `RETRY` handling would improve the state machine but would not repair this candidate set. The new design moves the trigger earlier—from demand lookup to request admission—and makes residency a benchmark guarantee rather than something the prefetcher discovers.

## 4. End-to-end mental model

Follow one marked request through the intended implementation:

1. The OpenAI request contains `kv_transfer_params.abc_admission_prefetch=true`.
2. vLLM constructs the scheduler `Request`, including all prompt block hashes.
3. `OffloadingConnectorScheduler.on_new_request()` creates the normal `ReqContext` and manager request state.
4. The connector derives the request’s ordered offload keys with the existing `RequestOffloadState.update_offload_keys()` method.
5. It slices the first `N` keys and calls `TieringOffloadingManager.prefetch_assume_resident()`.
6. The manager checks CPU only. It never asks NVMe whether a candidate exists.
7. For every CPU miss, `_initiate_promotion()` reserves a CPU slot immediately and appends the key to a pending batch.
8. At scheduler-step end, `_flush_pending_promotions()` submits one asynchronous NVMe load for the request.
9. The filesystem threads copy the blocks while the request waits behind another request.
10. Later demand lookup sees either `HIT` (ready), `HIT_PENDING` (late), or `MISS` (failed/evicted) in CPU.

The key distinction is:

```text
candidate selection: first N known request keys, blind to NVMe membership
data movement:       existing tiered-offload promotion path
correctness fallback: existing reactive lookup path
```

## 5. Repository map and current line anchors

| Responsibility | Repository path | Current anchor |
|---|---|---:|
| Metric names and transfer job shape | `vllm/v1/kv_offload/tiering/base.py` | `TieringOffloadingMetrics`, lines 37–48; `JobMetadata`, lines 51–59 |
| Manager state | `vllm/v1/kv_offload/tiering/manager.py` | `PendingPromotion`, lines 66–72; `TieringOffloadingManager.__init__`, lines 183–243 |
| Async completion | `vllm/v1/kv_offload/tiering/manager.py` | `_process_finished_jobs`, lines 264–297 |
| Retained outcome tracking | `vllm/v1/kv_offload/tiering/manager.py` | `_track_prefetched`, lines 302–334; `_observe_prefetch_outcome`, lines 336–362 |
| Reactive demand lookup | `vllm/v1/kv_offload/tiering/manager.py` | `lookup`, lines 364–435 |
| CPU reservation and batching | `vllm/v1/kv_offload/tiering/manager.py` | `_initiate_promotion`, lines 465–512; `_flush_pending_promotions`, lines 514–536 |
| End-of-step flush | `vllm/v1/kv_offload/tiering/manager.py` | `on_schedule_end`, especially lines 820–834 |
| Configuration and metric definitions | `vllm/v1/kv_offload/tiering/spec.py` | metrics lines 136–199; config lines 211–230; manager construction lines 298–302 |
| KV group shape | `vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py` | `GroupOffloadConfig`, lines 85–104 |
| Key derivation | same scheduler file | `RequestOffloadState.update_offload_keys`, lines 312–326 |
| Request parameter propagation | same scheduler file | `_create_req_context`, lines 431–442 |
| Demand prefix lookup (leave unchanged) | same scheduler file | `_maximal_prefix_lookup`, lines 545–577 |
| Admission hook | same scheduler file | `on_new_request`, lines 804–814 |
| OpenAI request field already available | `vllm/entrypoints/openai/chat_completion/protocol.py` | field lines 465–468; propagation lines 690–693 |
| Manager tests | `tests/v1/kv_offload/tiering/test_tiering_offloading.py` | fixture lines 254–293; promotion tests from line 422; retained accounting tests from line 447 |
| Scheduler lookup tests | `tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py` | focused lookup section from line 1060 |
| Filesystem batching | `vllm/v1/kv_offload/tiering/fs/manager.py` | `submit_load`, lines 221–231; completion lines 234–252 |

Read these anchors before writing code. In particular, trace `_initiate_promotion()` through `_flush_pending_promotions()` and `_process_finished_jobs()` until you can explain why CPU blocks are `HIT_PENDING` before I/O completion.

## 6. Implementation order

Implement in this order:

1. add configuration;
2. add manager-side direct prefetch and a happy-path unit test;
3. add admission-time scheduling and scheduler tests;
4. add per-key prefetch provenance and job-failure accounting;
5. add late-prefetch accounting and lifecycle cleanup;
6. connect the benchmark request gate;
7. run the full validation matrix.

This order separates “can I queue a blind promotion?” from “can I trigger it correctly?” and from “can I explain every outcome?”

---

## Step 0 — Establish your checkpoint

From the repository root:

```bash
cd /Users/aperdomo/workspace/redhat/vllm
git status --short
git diff --check
.venv/bin/python -m pytest \
  tests/v1/kv_offload/tiering/test_tiering_offloading.py -q
.venv/bin/python -m pytest \
  tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py::TestMaximalPrefixLookup \
  tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py::TestSlidingWindowLookup \
  -q
```

Expected starting result from the cleanup:

```text
51 tiering tests passed
28 focused scheduler tests passed
```

The complete scheduler test file may try to contact Hugging Face while constructing model configuration. In a network-restricted environment, run the two focused classes above; do not misclassify DNS setup failures as scheduler regressions.

Checkpoint question: can you identify the four cleanup files in `git diff` and explain why each changed?

---

## Step 1 — Add the configuration knob

### 1.1 Understand the two controls

Use two independent controls:

- server configuration: `admission_prefetch_chunks=N`;
- per-request opt-in: `abc_admission_prefetch=true`.

The server knob determines the maximum work. The request flag separates warm-up from measurement without restarting or mutating global state.

Do not restore the name `prefetch_chunks`. Its old meaning was “post-miss read-ahead,” so reusing it would make old deployment files silently activate different behavior.

### 1.2 Parse the knob in `TieringOffloadingSpec`

File:

```text
vllm/v1/kv_offload/tiering/spec.py
```

Current anchors:

- module configuration documentation: lines 11–27;
- `TieringOffloadingSpec.__init__`: lines 211–238;
- manager construction: lines 298–302.

Immediately after secondary-tier parsing and before `prefetch_track_capacity`, read the extra config:

```python
admission_prefetch_chunks = self.extra_config.get(
    "admission_prefetch_chunks", 0
)
if (
    not isinstance(admission_prefetch_chunks, int)
    or isinstance(admission_prefetch_chunks, bool)
    or admission_prefetch_chunks < 0
):
    raise ValueError(
        "admission_prefetch_chunks must be a non-negative int, got "
        f"{admission_prefetch_chunks!r}"
    )
self.admission_prefetch_chunks = admission_prefetch_chunks
```

Why reject `bool` explicitly? In Python, `bool` is a subclass of `int`. Without the explicit check, `true` silently becomes `1`, which is a surprising configuration for a block-count knob.

Pass the value at manager construction:

```python
tiering_manager = TieringOffloadingManager(
    primary_tier=primary_tier,
    secondary_tiers=secondary_tiers,
    admission_prefetch_chunks=self.admission_prefetch_chunks,
    prefetch_track_capacity=self.prefetch_track_capacity,
)
```

Update the module documentation to describe the new key and its default.

### 1.3 Store and expose it in the manager

File:

```text
vllm/v1/kv_offload/tiering/manager.py
```

Current anchors:

- constructor signature: lines 183–188;
- assignments: lines 200–209;
- place for a property: immediately before `_next_job_id`, line 245.

Add the constructor parameter, assign it, and expose a read-only property:

```python
@property
def admission_prefetch_chunks(self) -> int:
    return self._admission_prefetch_chunks
```

For this controlled PoC, fail fast if `N > 0` but no secondary tier exists. An enabled server without a source tier is a deployment error, not a per-request cache miss:

```python
if self._admission_prefetch_chunks > 0 and not self.secondary_tiers:
    raise ValueError(
        "admission_prefetch_chunks requires at least one secondary tier"
    )
```

This deliberately improves on the original guide’s “skip without crashing” suggestion. Fail-fast configuration gives a clearer experiment: a treatment deployment cannot start while incapable of performing the mechanism.

Log the enabled value and source tier 0 at startup. Keep the log experimental and concise.

### 1.4 Test configuration before adding behavior

Add tests near the top-level spec tests in:

```text
tests/v1/kv_offload/tiering/test_tiering_offloading.py
```

At minimum cover:

- default `N=0`;
- positive integer accepted;
- negative integer rejected;
- Boolean rejected;
- enabled manager with no secondary tier rejected.

Checkpoint:

```bash
rg -n "admission_prefetch_chunks" vllm tests/v1/kv_offload/tiering
.venv/bin/python -m pytest \
  tests/v1/kv_offload/tiering/test_tiering_offloading.py -q
```

Do not continue until the knob reaches the manager and no prefetch happens merely because it is configured.

---

## Step 2 — Implement the blind manager API

### 2.1 Define the contract first

Add this method to `TieringOffloadingManager`:

```python
def prefetch_assume_resident(
    self,
    keys: Sequence[OffloadKey],
    req_context: ReqContext,
    tier_idx: int = 0,
) -> int:
    """Queue assumed-resident secondary blocks for promotion to CPU."""
```

Place it in:

```text
vllm/v1/kv_offload/tiering/manager.py
```

Current insertion anchor: after `_tier_label()` at lines 299–300 and before `_track_prefetched()` at line 302.

Its contract is deliberately different from reactive `lookup()`:

| Question | Reactive `lookup()` | `prefetch_assume_resident()` |
|---|---|---|
| Who selects the key? | Demand prefix scan | Admission policy |
| Check CPU? | Yes | Yes |
| Check NVMe membership? | Yes | **No** |
| On assumed source absence | `MISS` | Async job failure/oracle violation |
| Correctness role | Required | Best-effort optimization |

### 2.2 Implement the decision loop

The method should:

1. process prior completions once, so newly freed CPU slots are visible;
2. resolve the configured source tier;
3. honor the request’s tier filter;
4. increment `PREFETCH_ATTEMPTED` for every supplied key;
5. query CPU only;
6. count `HIT`/`HIT_PENDING` as redundant;
7. directly initiate a promotion for CPU misses;
8. count capacity failures as skipped;
9. return the number queued.

An implementation skeleton is:

```python
def prefetch_assume_resident(
    self,
    keys: Sequence[OffloadKey],
    req_context: ReqContext,
    tier_idx: int = 0,
) -> int:
    if not keys:
        return 0
    if tier_idx < 0 or tier_idx >= len(self.secondary_tiers):
        raise IndexError(f"secondary tier index out of range: {tier_idx}")

    self._maybe_process_finished_jobs()
    tier = self.secondary_tiers[tier_idx]
    label = self._tier_label(tier_idx)
    initiated = 0

    for key in keys:
        self._stats.increase_counter(
            TieringOffloadingMetrics.PREFETCH_ATTEMPTED,
            labelvalues=label,
        )

        if not req_context.load_tier_filter.allows(
            tier.medium, tier.locality
        ):
            self._stats.increase_counter(
                TieringOffloadingMetrics.PREFETCH_SKIPPED,
                labelvalues=label,
            )
            continue

        primary_result = self.primary_tier.lookup(key, req_context)
        if primary_result in (
            LookupResult.HIT,
            LookupResult.HIT_PENDING,
        ):
            self._stats.increase_counter(
                TieringOffloadingMetrics.PREFETCH_REDUNDANT,
                labelvalues=label,
            )
            continue

        assert primary_result is LookupResult.MISS
        if self._initiate_promotion(
            tier, key, req_context, is_prefetch=True
        ):
            initiated += 1
            self._stats.increase_counter(
                TieringOffloadingMetrics.PREFETCH_PROMOTED,
                labelvalues=label,
            )
            self._track_prefetched(key, tier_idx)
        else:
            self._stats.increase_counter(
                TieringOffloadingMetrics.PREFETCH_SKIPPED,
                labelvalues=label,
            )

    return initiated
```

The `is_prefetch=True` parameter does not exist yet; add it in Step 3 before expecting the skeleton to compile.

Why check `HIT_PENDING` as redundant? A CPU slot is already reserved and a load is in flight. A second load would duplicate I/O and could target the same CPU key twice.

Why not call `tier.lookup()`? Avoiding that call is the defining property of the oracle experiment. The benchmark—not the selection method—asserts residency.

Why use the actual `1:fs` label? The source tier is already known. The ambiguous old `tier="prefetch"` label is unnecessary and should never return.

---

## Step 3 — Teach the batch which keys were prefetched

The existing transfer job knows direction only:

```text
JobMetadata.is_promotion=True  means secondary→CPU
```

Both reactive demand and proactive prefetch are promotions, and different keys from both origins can be combined into one `(tier, request)` batch. Therefore, provenance must be **per key**, not one Boolean on `JobMetadata`.

### 3.1 Extend `PendingPromotion`

File and anchor:

```text
vllm/v1/kv_offload/tiering/manager.py
PendingPromotion, lines 66–72
```

Add:

```python
prefetch_keys: list[OffloadKey] = field(default_factory=list)
```

Do not add `is_prefetch: bool` to the whole batch. A mixed batch would make that flag ambiguous.

### 3.2 Extend `_initiate_promotion()`

Current anchor: lines 465–512.

Add a keyword-only parameter with a safe default:

```python
def _initiate_promotion(
    self,
    tier: SecondaryTierManager,
    key: OffloadKey,
    req_context: ReqContext,
    *,
    is_prefetch: bool = False,
) -> bool:
```

The default preserves every existing reactive caller, including line 418 in `lookup()`.

After extending `entry.keys` and `entry.block_ids`, record only the keys the CPU tier actually reserved:

```python
if is_prefetch:
    entry.prefetch_keys.extend(primary_write_result.keys_to_store)
```

Using `keys_to_store` instead of blindly appending the input key keeps provenance aligned with the actual CPU allocation result.

### 3.3 Record provenance when a batch becomes a job

In `TieringOffloadingManager.__init__`, near `_transfer_jobs` at lines 211–224, add manager-local state:

```python
self._prefetch_job_keys: dict[
    JobId, tuple[int, tuple[OffloadKey, ...]]
] = {}
```

Keeping this in the manager avoids changing the shared `JobMetadata` interface used by filesystem, object-store, P2P, and example tiers.

In `_flush_pending_promotions()` at lines 523–534, after assigning `job_id` and before `submit_load()`:

```python
if entry.prefetch_keys:
    tier_idx = self.secondary_tiers.index(tier)
    self._prefetch_job_keys[job_id] = (
        tier_idx,
        tuple(entry.prefetch_keys),
    )
```

Now the manager can answer, on completion, “which subset of this promotion job came from proactive prefetch?”

### 3.4 Write the manager happy-path test now

The current test fixture is at:

```text
tests/v1/kv_offload/tiering/test_tiering_offloading.py
TestTieringOffloadingManager.manager_setup, lines 257–283
```

The existing temporary helper `_queue_tracked_promotion()` at lines 447–455 was added only to test retained plumbing after the old selector was removed. Replace its manual calls with the new public method as soon as the method exists.

First test:

1. put several keys directly in `secondary_tier1.blocks`;
2. wrap `secondary_tier1.lookup` with `MagicMock`;
3. call `prefetch_assume_resident(keys, _CTX, tier_idx=0)`;
4. assert the return count equals the key count;
5. assert `secondary_tier1.lookup` was never called;
6. call `_simulate_on_schedule_end()` twice;
7. assert CPU lookup returns `HIT` for every key;
8. assert attempted and promoted use the `1:example` label.

That test proves the central mechanism without involving the connector scheduler.

Also add small tests for:

- a CPU `HIT` is redundant;
- a CPU `HIT_PENDING` is redundant;
- primary capacity exhaustion is skipped;
- a filtered source tier is skipped;
- `attempted = promoted + redundant + skipped`.

Checkpoint:

```bash
.venv/bin/python -m pytest \
  tests/v1/kv_offload/tiering/test_tiering_offloading.py -q
```

Before moving on, put a breakpoint in `_flush_pending_promotions()` and verify that 100 selected keys become one `submit_load()` job, not 100 jobs.

---

## Step 4 — Trigger prefetch at admission

### 4.1 Reuse the existing request field

No OpenAI protocol change is necessary.

The field already exists in:

```text
vllm/entrypoints/openai/chat_completion/protocol.py
ChatCompletionRequest.kv_transfer_params, lines 465–468
to_sampling_params propagation, lines 690–693
```

The connector already copies it into `ReqContext` in:

```text
vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py
_create_req_context, lines 431–442
```

Add a constant near `KV_LOAD_TIERS_KEY` at lines 61–63:

```python
ABC_ADMISSION_PREFETCH_KEY = "abc_admission_prefetch"
```

The `ABC_` prefix makes the experiment-specific nature obvious and reduces collision risk with established disaggregated-serving parameters.

### 4.2 Understand key derivation

`RequestOffloadState.update_offload_keys()` at lines 312–326 converts prompt block hashes into group-qualified `OffloadKey` values. It is incremental: it starts after `len(group_state.offload_keys)`, so calling it at admission and again during demand lookup does not duplicate keys.

The PoC must use only hashes that already exist. It cannot prefetch future decode blocks because their chained hashes have not been computed yet.

### 4.3 Add a small scheduler helper

Place a private helper immediately before `on_new_request()` at current line 804. Keeping the policy in a helper makes `on_new_request()` readable and independently testable.

Suggested structure:

```python
def _maybe_prefetch_on_admission(
    self, req_status: RequestOffloadState
) -> None:
    params = req_status.req_context.kv_transfer_params or {}
    if params.get(ABC_ADMISSION_PREFETCH_KEY) is not True:
        return

    n = getattr(self.manager, "admission_prefetch_chunks", 0)
    prefetch = getattr(self.manager, "prefetch_assume_resident", None)
    if n <= 0 or prefetch is None:
        return

    if len(self.config.kv_group_configs) != 1:
        logger.warning(
            "Admission prefetch supports exactly one KV group; skipping %s",
            req_status.req.request_id,
        )
        return

    group_config = self.config.kv_group_configs[0]
    if (
        group_config.sliding_window_size_in_chunks is not None
        or group_config.is_eagle_group
    ):
        logger.warning(
            "Admission prefetch requires one stable full-attention group; "
            "skipping %s",
            req_status.req.request_id,
        )
        return

    req_status.update_offload_keys()
    keys = req_status.group_states[0].offload_keys[:n]
    prefetch(keys, req_status.req_context, tier_idx=0)
```

Why use `getattr` here? `OffloadingConnectorScheduler` is intentionally typed against generic `OffloadingManager`/`OffloadingSpec`, while this experimental API belongs only to the tiering manager. Adding a tier-indexed method to every offloading manager would pollute the general interface. For the PoC, a guarded optional capability is the narrower change.

Why restrict group shape? Nemotron uses the simple full-attention layout needed by this experiment. Sliding-window, Mamba, hybrid, and EAGLE groups have different reachability or hash-stability rules. Generalizing them before proving the mechanism would add unrelated correctness risk.

### 4.4 Call it in the correct order

Current `on_new_request()` is lines 804–814. Call the helper only after:

1. `_create_req_context()`;
2. `manager.on_new_request(req_context)`;
3. `RequestOffloadState(...)` construction;
4. insertion into `_req_status`.

Then:

```python
self._maybe_prefetch_on_admission(req_status)
```

The manager request state must exist because promotion completion and request lifecycle code use `_req_state`. Registering connector state first also prevents later callbacks from observing an unknown request.

The first marked request may have no queue cover and therefore little benefit. Do not special-case it. With `max_num_seqs=1` and eight client streams, most measured requests have a natural overlap window; the first-request cost is part of the honest aggregate result.

### 4.5 Add scheduler tests

File:

```text
tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py
```

Add a focused class near the lookup-unit section beginning at line 1060. Build the smallest scheduler object possible rather than using network-sensitive model fixtures.

Cover:

1. marked request and `N=100` passes the first 100 ordered keys;
2. fewer than `N` complete prompt chunks passes all available keys;
3. missing flag does nothing;
4. Boolean `false` does nothing;
5. integer `1` does **not** count as Boolean `true`;
6. `N=0` does nothing;
7. unsupported group layout does nothing;
8. manager `on_new_request` occurs before prefetch;
9. `_maximal_prefix_lookup()` on `MISS` still does no prefetch.

The strict `is True` behavior matters. Request JSON values such as `1` or `"true"` should not silently enable an experimental storage operation.

Checkpoint:

```bash
.venv/bin/python -m pytest \
  tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py::TestAdmissionPrefetch \
  tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py::TestMaximalPrefixLookup \
  -q
```

---

## Step 5 — Account for failed and late prefetches

The happy path is enough to show data movement, but not enough to trust a benchmark. You need to distinguish:

- the candidate was assumed resident but its load job failed;
- the load succeeded but demand arrived before it completed;
- the load completed and demand used it;
- the block was evicted before demand;
- the outcome tracker overflowed.

### 5.1 Add metric names

File:

```text
vllm/v1/kv_offload/tiering/base.py
TieringOffloadingMetrics, lines 37–48
```

Add:

```python
PREFETCH_LOAD_FAILED = "vllm:kv_offload_tiering_prefetch_load_failed"
PREFETCH_LATE = "vllm:kv_offload_tiering_prefetch_late"
```

Do not change `JobMetadata` at lines 51–59. Manager-local provenance is sufficient and avoids widening the interface for all secondary tiers.

### 5.2 Define the counters

File and anchor:

```text
vllm/v1/kv_offload/tiering/spec.py
build_metric_definitions(), lines 136–199
```

Both counters need `labelnames=("tier",)`.

Definitions:

- `PREFETCH_LOAD_FAILED`: proactively selected chunks in an asynchronous promotion job that completed unsuccessfully;
- `PREFETCH_LATE`: proactively promoted chunks whose first demand lookup saw `HIT_PENDING`.

`LOAD_FAILED` is a subset of promoted attempts. `LATE` is orthogonal: a late prefetch may still become useful on the next scheduler step.

Extend the metric-definition unit test currently near lines 165–183 of `test_tiering_offloading.py` so label arity cannot drift.

### 5.3 Understand filesystem failure granularity

The filesystem tier batches all keys into one `batch_load_block()` call:

```text
vllm/v1/kv_offload/tiering/fs/manager.py:221–231
vllm/v1/kv_offload/tiering/fs/io.py:192–213
```

`JobResult` has one Boolean for the whole job. A read error makes the batch unsuccessful, and `primary_tier.complete_write(..., success=False)` invalidates the batch’s CPU reservations. Consequently, count **all prefetched keys in the failed job** as failed. The current API cannot identify only the offending file.

This is not a flaw in the experiment: a failed batch did not deliver usable proactive blocks. Document the job-level granularity when interpreting the metric.

### 5.4 Handle completion

Current anchor:

```text
vllm/v1/kv_offload/tiering/manager.py
_process_finished_jobs(), lines 264–297
```

For every completed promotion job:

1. pop its prefetch provenance from `_prefetch_job_keys`;
2. call `primary_tier.complete_write()` exactly as today;
3. if the job failed, increment `PREFETCH_LOAD_FAILED` by the number of prefetched keys;
4. remove those keys from `_prefetched` so a later demand `MISS` does not relabel an I/O failure as ordinary waste;
5. remove them from late-state tracking.

`OffloadingConnectorStats.increase_counter()` accepts the increment as its second argument; see `vllm/distributed/kv_transfer/kv_connector/v1/offloading/metrics.py`, lines 275–286.

Suggested shape:

```python
prefetch_job = self._prefetch_job_keys.pop(job_id, None)

if job_metadata.is_promotion:
    self.primary_tier.complete_write(
        job_metadata.keys,
        job_metadata.req_context,
        completed_job.success,
    )
    if prefetch_job is not None and not completed_job.success:
        tier_idx, prefetch_keys = prefetch_job
        self._stats.increase_counter(
            TieringOffloadingMetrics.PREFETCH_LOAD_FAILED,
            len(prefetch_keys),
            self._tier_label(tier_idx),
        )
        for key in prefetch_keys:
            self._prefetched.pop(key, None)
            self._late_prefetches.discard(key)
```

Be careful with argument order: `increase_counter(name, value, labelvalues)`.

### 5.5 Count lateness once per key

In manager initialization, near `_prefetched` at lines 204–209, add:

```python
self._late_prefetches: set[OffloadKey] = set()
```

Extend `_observe_prefetch_outcome()` at lines 336–362:

- `HIT`: useful, remove tracked key, discard late marker;
- `MISS`: wasted, remove tracked key, discard late marker;
- first `HIT_PENDING`: increment late and remember the key;
- repeated `HIT_PENDING`: do nothing.

Reset the late marker when `_track_prefetched()` begins a new promotion of the same key. Discard it when tracking overflow removes a key.

This preserves two different facts:

```text
useful: demand eventually consumed the proactive CPU copy
late:   that copy was not ready at the first demand observation
```

### 5.6 Clean lifecycle state on cache reset

Review `reset_cache()` in `manager.py`, lines 864–904.

The primary cache reset destroys proactively promoted copies. Before clearing the primary tier:

- resolve currently tracked prefetches as wasted or explicitly untracked;
- clear `_prefetched` and `_late_prefetches`;
- ensure drained job provenance is gone;
- clear pending submissions as the existing code already does.

For this experiment, counting a completed-but-never-demanded copy destroyed by reset as wasted is reasonable. More important than the exact label is avoiding stale keys that could be attributed to a later run.

The benchmark should still avoid resets during the measured phase.

### 5.7 Add failure and lateness tests

In `test_tiering_offloading.py`, extend the retained tests beginning near line 463:

- successful completion then demand `HIT` → useful;
- first demand while in flight → late exactly once, still tracked;
- completion then next demand → useful and no second late increment;
- failed proactive job → load_failed, not later only wasted;
- mixed demand/prefetch batch → only proactive keys counted failed;
- tracking overflow → untracked, not wasted;
- reset clears all outcome state.

For a failure test, the example tier reports a failed job when a requested key is absent. That is enough for a manager unit test. Add a real filesystem integration case later to confirm the same behavior through `batch_load_block()`.

---

## Step 6 — Preserve the reactive fallback

Do not modify these paths:

```text
vllm/distributed/kv_transfer/kv_connector/v1/offloading/scheduler.py
_maximal_prefix_lookup(), lines 545–577

vllm/v1/kv_offload/tiering/manager.py
lookup(), lines 364–435
```

Normal demand lookup remains responsible for correctness. If proactive work is skipped, late, evicted, filtered, or failed, the demand path still checks secondary tiers and can initiate a normal promotion.

This fallback is why the feature can be best-effort. Prefetch changes timing, not the model’s KV correctness contract.

Add a regression test proving a failed proactive attempt can still be recovered reactively when the source becomes available again.

---

## Step 7 — Wire the GuideLLM request gate

Only do this after the vLLM unit tests pass.

### 7.1 Measured requests opt in

File:

```text
/Users/aperdomo/workspace/redhat/benchflow/profiles/benchmark/nemotron-nvme-prefetch-poc.yaml
backend body, current lines 7–15
```

Add:

```yaml
kv_transfer_params:
  abc_admission_prefetch: true
```

### 7.2 Pre-warmup explicitly opts out

The current pre-warmup begins at line 37. Override its GuideLLM backend so it preserves the deterministic generation fields but sends `false`:

```yaml
pre_warmup:
  rate: 8
  backend:
    kind: openai_http
    request_format: /v1/chat/completions
    stream: true
    extras:
      body:
        temperature: 0
        seed: 20260814
        kv_transfer_params:
          abc_admission_prefetch: false
  constraint:
    - kind: max_requests
      count: 1024
```

Do not use a request-count threshold inside vLLM. The explicit request flag makes phase separation observable and deterministic.

### 7.3 Configure control and treatment deployments

The active NVMe deployment profile is:

```text
/Users/aperdomo/workspace/redhat/benchflow/profiles/deployment/rhoai/multi-tier-offloading-nvme.yaml
vLLM arguments, current lines 45–55
```

Its commented line 48 still contains the rejected `prefetch_chunks` name. Delete or update that comment so it cannot be copied accidentally.

Use two otherwise identical deployment cells:

```text
control:   admission_prefetch_chunks = 0
treatment: admission_prefetch_chunks = 100
```

Do not compare a treatment that differs in NVMe threads, CPU bytes, image, TP, model revision, or cache cleaning.

The active experiment already pins Nemotron FP8 to `--max-num-seqs=1` in:

```text
/Users/aperdomo/workspace/redhat/benchflow/experiments/rhoai/cpu-offloading.yaml
lines 29–38
```

### 7.4 Validate BenchFlow configuration

From the BenchFlow repository, use its virtual environment and validator. Confirm the rendered GuideLLM commands for both phases, not only YAML syntax. Specifically verify that warm-up contains `false` and measurement contains `true`.

---

## Step 8 — Test plan

### Manager unit tests

File:

```text
tests/v1/kv_offload/tiering/test_tiering_offloading.py
```

Required behaviors:

1. direct assumed-resident promotion never calls secondary `lookup()`;
2. 100 CPU misses queue 100 promotions in one request/tier batch;
3. CPU `HIT` and `HIT_PENDING` are redundant;
4. CPU capacity failure is skipped;
5. tier-filter rejection is skipped with the real tier label;
6. successful completion followed by demand `HIT` is useful;
7. first demand `HIT_PENDING` is late exactly once;
8. failed load is load_failed and not misreported only as wasted;
9. mixed demand/prefetch batch preserves per-key provenance;
10. tracking-capacity accounting remains valid;
11. reset removes stale attribution state.

### Scheduler unit tests

File:

```text
tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py
```

Required behaviors:

1. marked request takes the first `N` ordered keys;
2. short prompt takes all available complete chunks;
3. absent/false/mistyped flag does nothing;
4. `N=0` does nothing;
5. unsupported group layout does nothing;
6. manager request state precedes prefetch;
7. demand `MISS` never invokes the admission prefetch API.

### Filesystem integration test

Use the existing filesystem test area:

```text
tests/v1/kv_offload/tiering/test_fs_tier.py
```

Test:

1. pre-populate files for known keys;
2. call the manager direct API;
3. flush and wait for completion;
4. verify CPU hits;
5. verify candidate discovery did not call filesystem `lookup()`;
6. omit one file and verify the batched job-level failure metric;
7. verify failed CPU reservations are released.

### Commands

```bash
.venv/bin/python -m pytest \
  tests/v1/kv_offload/tiering/test_tiering_offloading.py -q

.venv/bin/python -m pytest \
  tests/v1/kv_offload/tiering/test_fs_tier.py -q

.venv/bin/python -m pytest \
  tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py::TestAdmissionPrefetch \
  tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py::TestMaximalPrefixLookup \
  tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py::TestSlidingWindowLookup \
  -q

git diff --check
```

Run formatting and linting through the repository’s configured `uv`/virtual-environment workflow. Do not use system Python or bare `pip`.

---

## 9. Metric accounting

Retain:

- `prefetch_attempted`;
- `prefetch_promoted`;
- `prefetch_redundant`;
- `prefetch_skipped`;
- `prefetch_useful`;
- `prefetch_wasted`;
- `prefetch_untracked`.

Add:

- `prefetch_load_failed`;
- `prefetch_late`.

Primary partition:

```text
attempted = promoted + redundant + skipped
```

Outcome accounting is time-dependent:

```text
promoted = useful + wasted + untracked + still_tracked
```

`load_failed` is a labeled subset of promoted attempts and should be removed from the effective-success denominator. `late` is orthogonal and indicates lost overlap, even when the block later becomes useful.

Useful benchmark ratios:

```text
effective_promoted = promoted - load_failed
useful_rate = useful / effective_promoted
late_rate = late / effective_promoted
```

Interpret `late_rate` together with `useful_rate`. A high useful rate and high late rate means prediction was correct but timing was insufficient.

For the controlled PoC, every attempt has a known source tier. Expect `tier="1:fs"`; the old aggregate `tier="prefetch"` must not appear.

## 10. Benchmark matrix

Start with:

| Cell | Request flag | `admission_prefetch_chunks` | Purpose |
|---|---:|---:|---|
| Control | true | 0 | Same request body, proactive work disabled server-side |
| Treatment | true | 100 | Blind first-100 admission prefetch |

Warm-up sends the request flag as `false` in both cells.

Required conditions:

- Nemotron FP8;
- tensor parallelism 8;
- one replica;
- `max_num_seqs=1`;
- GuideLLM concurrent streams 8;
- deterministic seed 20260814;
- 4,096 prompt tokens;
- 64 output tokens;
- 1,024 warm-up requests;
- 256 measured requests;
- persistent NVMe state across warm-up and measurement;
- cold or churned GPU/CPU for the replayed target prefixes;
- at least five paired repetitions.

After mechanism validation, sweep:

```text
N ∈ {25, 50, 100, 200}
```

Do not tune `N` before the mechanism gates pass.

## 11. Acceptance gates

### Mechanism

- each marked request attempts exactly `min(N, available_complete_prompt_chunks)`;
- `promoted{tier="1:fs"}` is nonzero and close to attempted minus redundancy;
- no `tier="prefetch"` series;
- zero `prefetch_load_failed` after NVMe preparation is verified;
- useful/effective-promoted at least 0.9;
- most demanded prefetched blocks are not late;
- no material CPU-capacity skips;
- one batched NVMe load per marked request/tier/step, not one job per block.

### Performance

- paired target TTFT improves consistently;
- aggregate request throughput improves;
- request errors and cancellations do not increase;
- output-token throughput is reported but secondary for this prompt-heavy workload;
- the analysis records the overlap/queue interval rather than reporting only global means.

## 12. Common mistakes to avoid

### Accidentally querying NVMe

If `prefetch_assume_resident()` calls `tier.lookup()`, the PoC has recreated reactive membership discovery and no longer tests the intended mechanism.

### Triggering from `_maximal_prefix_lookup()`

That restores the rejected policy. Admission and demand lookup must remain separate.

### Marking the whole job as prefetch

A promotion batch may contain different keys from reactive and proactive origins. Track a key subset.

### Treating `HIT_PENDING` as success-before-demand

It is likely to become useful, but it was late for the first demand observation. Count both facts separately.

### Prefetching during warm-up

The files do not exist yet. This produces oracle failures and contaminates the mechanism counters.

### Generalizing group layouts too early

Nemotron’s single full-attention group is enough for Phase 1. Sliding-window and hybrid correctness belongs after the mechanism proof.

### Conflating queueing with TTFT

GuideLLM TTFT includes time visible to the client. Record queue depth/overlap context so an improvement can be attributed to hidden storage latency rather than changed arrival order.

## 13. Interpretation boundary

A successful result proves:

> Given known queued-request block identities, residency guaranteed by benchmark construction, and enough queue overlap, proactive NVMe→CPU promotion can remove storage work from the request-critical path and improve TTFT or pipeline throughput.

It does not prove:

- that `N=100` is optimal;
- that a production system can predict the next request;
- that blind prefetch remains beneficial under arbitrary concurrency;
- that the policy is correct for hybrid/sliding-window models;
- that NVMe bandwidth contention cannot erase the gain.

The admission hook and direct manager API are experimental seams. Once the mechanism is accepted, Phase 2 can replace “first N marked-request keys” with a real speculative predictor while reusing the same promotion, provenance, and measurement substrate.

## 14. Recommended working rhythm

For each tutorial step:

1. read the named existing function end to end;
2. state its current invariant in your own words;
3. make the smallest change;
4. inspect `git diff`;
5. run the focused test;
6. explain the observed state before continuing.

Good discussion checkpoints with an assistant are:

- after configuration reaches the manager;
- after the direct manager test proves no secondary lookup;
- after you observe a batched job in the debugger;
- before choosing failure-accounting state;
- after scheduler admission tests pass;
- before the first container build and benchmark run.

That keeps the implementation yours while still using review and design discussion where it is most valuable.