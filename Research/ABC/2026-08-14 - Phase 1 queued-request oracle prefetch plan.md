---
title: "ABC Phase 1 — queued-request oracle prefetch proof of concept"
date: "2026-08-14"
type: "research-plan"
experiment: "Activity-Based KV Cache Tier Placement"
phase: "1 — naive proactive prefetching"
status: "proposed"
prefetch_policy: "blind first-N blocks of an admitted waiting request"
prefetch_direction: "NVMe to CPU"
initial_n: 100
---

# ABC Phase 1 — queued-request oracle prefetch proof of concept

## Objective

Demonstrate that correctly timed proactive NVMe→CPU promotion can reduce target-request TTFT and raise pipeline throughput, without attempting to build the final speculative heuristic.

This is deliberately an oracle-like toy. It does not discover a reusable prefix at demand-lookup time. Instead, when a target request is admitted while another request occupies the only scheduling slot, it blindly submits the first $N=100$ target keys to the configured NVMe tier. The benchmark guarantees those keys were populated during a separate warm-up phase.

## Policy

For a request explicitly marked for the experiment:

1. At admission, construct its normal ordered offload keys.
2. Take the first exactly $N$ keys; do not run maximal-prefix lookup and do not select only demand misses.
3. Submit those keys directly to secondary tier 0 as assumed-resident promotions.
4. Let the transfers run while an earlier cover request occupies the GPU.
5. When the target request becomes schedulable, normal demand lookup should find the prefetched blocks in CPU.

Every KV prefetch mechanism requires block identities. The toy uses identities already computed for the queued request; its simplification is that candidate selection and secondary membership prediction are treated as a perfect oracle.

## Why this can show benefit

With the control, the target request's NVMe existence lookup and NVMe→CPU reads begin when the request reaches the demand path. With oracle prefetch, that work overlaps the preceding request's GPU execution.

For target request $i$:

$$
TTFT_{control} \approx Q_i + L_{NVMe\rightarrow CPU} + L_{CPU\rightarrow GPU} + L_{prefill}
$$

$$
TTFT_{prefetch} \approx Q_i + \max(0, L_{NVMe\rightarrow CPU} - Q_i) + L_{CPU\rightarrow GPU} + L_{prefill}
$$

where $Q_i$ is the natural queue/cover interval available for overlap. Repeated pairs can also remove storage stalls from the pipeline critical path and improve request throughput.

## Controlled workload

Use repeated ordered pairs:

- **Cover request $A_i$:** disjoint prompt, enough decode work to provide an NVMe prefetch window.
- **Target request $B_i$:** long, unique, prompt-heavy request whose first 100 offload chunks were pre-populated on NVMe; short output so TTFT/storage dominates.

Run with `max_num_seqs=1` or an equivalent deterministic scheduling constraint. Admit $B_i$ immediately after $A_i$ starts, so $B_i$ waits naturally while its prefetch overlaps $A_i$ execution. Ensure ordering with request priority or a controlled submission handshake, not a fixed sleep.

Use unique $B_i$ prefixes so one measured target does not warm GPU or CPU for another.

## Cache preparation and isolation

1. Warm every $B_i$ through a cache-builder deployment using the identical model revision, TP topology, offload chunk size, file mapping, and `PYTHONHASHSEED`.
2. Preserve the NVMe directory.
3. Restart the benchmark deployment to clear GPU and CPU caches while retaining NVMe.
4. Start each control and prefetch repetition from the same immutable NVMe snapshot and cold GPU/CPU state.
5. Verify the snapshot outside the timed measurement; do not perform membership-driven candidate selection inside the experiment.

CPU capacity must fit the one 100-block lookahead window plus active-request traffic. Avoid admitting many marked targets simultaneously, which would turn the toy into uncontrolled deep-queue prefetch.

## Minimal implementation

Add an explicitly experimental API such as:

```python
prefetch_assume_resident(
    keys: Sequence[OffloadKey],
    req_context: ReqContext,
    tier_idx: int = 0,
) -> int
```

It should:

- bypass secondary `lookup()`;
- check CPU residency only to avoid duplicate allocation;
- call `_initiate_promotion()` for each of the first $N$ keys;
- label successful promotions `1:fs`;
- label primary-capacity failures `1:fs` skipped;
- retain normal completion and useful/wasted tracking;
- expose load-job failures separately because an assumed-resident missing file is an oracle violation.

Trigger it from request admission after offload keys are available, guarded by an experiment-only request parameter or configuration. Do not overload the existing post-miss hook.

This direct path also avoids the current `prefetch()` defect where an asynchronous filesystem `RETRY` is immediately collapsed into a generic skip.

## Experimental matrix

Begin with paired repetitions:

| Cell | Admission policy | $N$ |
|---|---|---:|
| Control | no proactive promotion | 0 |
| Toy oracle | blind assumed-resident promotion of queued target | 100 |

Run at least five paired repetitions with identical pair ordering. After mechanism validation, add $N \in \{25, 50, 100, 200\}$ to demonstrate scaling and possible over-prefetch pressure.

## Acceptance gates

Mechanism acceptance:

- exactly $N$ attempts per marked target when at least $N$ keys are available;
- nonzero `promoted{tier="1:fs"}`, ideally close to $N$;
- `useful/promoted \ge 0.9`;
- no oracle/file-load failures;
- prefetch completes before target demand lookup for most targets;
- no material allocation failures or unexpected CPU eviction pressure.

Performance acceptance:

- paired target TTFT improves consistently;
- aggregate request throughput improves across repeated $A_i,B_i$ pipelines;
- output-token throughput is reported but is secondary for a prompt-heavy, short-output target;
- results include the overlap interval and target-level paired distributions, not only global averages.

## Interpretation boundary

A successful result would prove:

> Given accurate future block identities, confirmed residency by construction, and enough overlap time, proactive NVMe→CPU promotion can remove storage work from the request-critical path.

It would not prove that a production heuristic can predict those blocks, nor that $N=100$ is optimal. That is the intended bridge to later trajectory/session or learned-successor heuristics.