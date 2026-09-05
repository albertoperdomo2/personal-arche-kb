---
title: Selective KV loading and offloading - initial implementation plan
date: 2026-09-03
type: implementation-plan
status: proposed
topic: KV cache routing and offloading
repos:
  - vllm-project/vllm
  - llm-d/llm-d-router
vllm_commit: 0e3ac4907d21e77cb4781338c49fef17bfea8f2b
router_commit: b1bf63da5e9a52dc8815264809d00f45f5b5e966
---

# Selective KV loading and offloading — initial implementation plan

## Executive summary

The first implementation should let llm-d-router skip low-value KV loads using a statically configured threshold and should establish independent router control over the existing vLLM offload cap. The smallest safe delivery is cross-repository: vLLM must first interpret `kv_load_tiers=[]` as “do not load from any offload tier,” including CPU; llm-d-router can then compute a decision for the selected endpoint and inject `kv_load_tiers` plus, independently, `max_offload_tokens`.

Do not start with dynamic calibration, strict CPU-versus-storage selection, or a single boolean controlling both directions. Loading answers “is reuse worthwhile for this request now?” Offloading answers “how much of this request is worth retaining for possible future reuse?” Those decisions can legitimately disagree.

## Normative stakeholder direction

1. A load-value threshold exists below which loading provides no benefit. The threshold may eventually be dynamic, but the initial implementation should support static profiling and calibration.
2. The router should be able to override both loading and offloading.
3. The offload-direction engine API already exists; the missing work is router policy and propagation.
4. The router owns the decision. vLLM owns enforcement, scheduling correctness, and data movement.

## Terminology

- **Load/reuse:** retrieve KV already present outside the executing request’s GPU prefix cache.
- **Offload/store:** copy newly computed KV from GPU into the configured offload hierarchy for future requests.
- **Source tier:** the tier whose existing copy authorizes reuse. A storage load may traverse CPU as a staging medium without CPU being an allowed original source.
- **Static calibration:** deploy a configured threshold measured for a model, topology, and tier configuration.
- **Dynamic adaptation:** change the threshold from runtime latency, bandwidth, pressure, or queue state. This is a later phase.

## Verified current state


| Capability                                | Current state                                              | Initial action                                |
| ----------------------------------------- | ---------------------------------------------------------- | --------------------------------------------- |
| vLLM secondary-tier load filtering        | Exists through `kv_load_tiers` from PR #48123              | Reuse                                         |
| vLLM primary CPU load filtering           | Missing; CPU lookup is unconditional                       | Add binary empty-filter enforcement           |
| vLLM per-request offload control          | Exists through `max_offload_tokens` from PR #39983         | Reuse; do not add another API                 |
| llm-d-router per-endpoint matches by tier | Exists as `CachedBlocksByTier` from precise-prefix-cache   | Consume                                       |
| llm-d-router selective load policy        | Missing                                                    | Add                                           |
| llm-d-router offload override policy      | Missing                                                    | Add independently                             |
| Direct EPP payload mutation               | Exists through `PreRequest` and `MutatePayloadMap`         | Reuse                                         |
| Sidecar/coordinator preservation          | Not guaranteed; several paths rebuild `kv_transfer_params` | Audit and wire                                |
| Runtime engine capability discovery       | Missing                                                    | Use an operator-configured compatibility gate |


### Existing offload contract

vLLM documents and implements `kv_transfer_params.max_offload_tokens` as a non-negative integer:


| Value                             | Meaning                                                                                      |
| --------------------------------- | -------------------------------------------------------------------------------------------- |
| omitted or `null`                 | no per-request cap                                                                           |
| `0`                               | disable offload/store for the request                                                        |
| positive `N`                      | only the first `N` request tokens are eligible for offload, subject to chunk/alignment rules |
| non-integer, negative, or boolean | warn and treat as uncapped                                                                   |


The scheduler parses the field in `RequestOffloadState.__post_init__` and applies it in `_calc_num_offloadable_tokens`. This control does not select a destination tier; normal CPU/tiered storage policy still determines where eligible blocks go.

Open vLLM PR [#52397](https://github.com/vllm-project/vllm/pull/52397) fixes an assertion failure when the cap is below a partial-tail boundary and fixes a path where falsy `0` can be treated as “no cap.” Until that lands or an equivalent fix is present, broad router emission of `max_offload_tokens` must be gated and tested on partial-tail-capable models.

## End-to-end request contract

The router writes two sibling fields under `kv_transfer_params`:

```json
{
  "kv_transfer_params": {
    "kv_load_tiers": [],
    "max_offload_tokens": 0
  }
}
```

Represent absence separately from an explicit decision:

- omitted `kv_load_tiers`: preserve vLLM’s default loading behavior;
- empty `kv_load_tiers`: authoritative “do not load from offload tiers”;
- non-empty `kv_load_tiers`: authorize the listed original sources after full source-selective vLLM support exists;
- omitted `max_offload_tokens`: preserve uncapped store behavior;
- `max_offload_tokens: 0`: authoritative “do not offload this request”;
- positive `max_offload_tokens`: cap stored prefix length.

### Independent-decision matrix


| Load decision | Offload decision | Payload                                                       | Example intent                               |
| ------------- | ---------------- | ------------------------------------------------------------- | -------------------------------------------- |
| deny          | deny             | `kv_load_tiers: []`, `max_offload_tokens: 0`                  | recompute and do not retain                  |
| deny          | allow            | `kv_load_tiers: []`, offload field omitted or positive        | recompute now, retain useful computed prefix |
| allow         | deny             | load field omitted/allowed, `max_offload_tokens: 0`           | reuse existing KV, do not add new stored KV  |
| allow         | allow            | load field omitted/allowed, offload field omitted or positive | normal reuse and retention                   |


Never derive `max_offload_tokens` from the load threshold. A low-value load does not imply that the recomputed KV has low future value.

## Initial policy

### Static load threshold

For the selected endpoint:

```text
offloaded_matched_tokens = max(cpu_matched_tokens, storage_matched_tokens)

if evidence is missing, stale, or incompatible:
    omit kv_load_tiers
else if offloaded_matched_tokens < min_reusable_prefix_tokens:
    kv_load_tiers = []
else:
    omit kv_load_tiers
```

The MVP emits only “skip” or “default.” It does not emit CPU-only or STORAGE-only lists. Equality with the threshold qualifies for loading. Use `max`, not a sum, because the same reusable prefix may appear in multiple tiers.

Do not hard-code a threshold in code. Make `min_reusable_prefix_tokens` explicit configuration, document its units, and calibrate it per meaningful model/topology/tier class. Missing or stale evidence must omit the field rather than force recomputation.

### Independent offload policy

Give the policy result an independent optional offload cap. The first production implementation can expose operator-controlled modes:


| Mode       | Output                             |
| ---------- | ---------------------------------- |
| `preserve` | omit `max_offload_tokens`          |
| `disable`  | set `max_offload_tokens: 0`        |
| `cap`      | set a validated positive token cap |


Keep the evaluator interface request-scoped so a later policy can choose the cap from request characteristics or predicted reuse. The static load threshold must not change this output.

## Architecture

```text
vLLM KV events
      |
      v
router KV index -> precise-prefix-cache -> endpoint candidates
                                             |
                                             v
                                     endpoint selection
                                             |
                                             v
                                selective policy for selected endpoint
                                  |                         |
                                  v                         v
                         kv_load_tiers             max_offload_tokens
                                  \                         /
                                   v                       v
                              trusted PreRequest payload mutation
                                             |
                           direct request / sidecar / coordinator
                                             |
                                             v
                              compatible vLLM enforcement
```

The decision must bind to the endpoint that actually executes the request. Retry or reselection must obtain a new decision. Prefill and decode legs need separate decisions unless they demonstrably share the same endpoint and cache state.

## Work package A — vLLM binary load opt-out

Detailed design: [[02 - vLLM primary-tier selective loading]].

### Scope

- Add an explicit empty-filter predicate to `TierFilter`.
- Make tiered lookup return `MISS` before primary or secondary lookup when the filter is explicitly empty.
- Enforce the same rule in standalone `CPUOffloadingManager`.
- Preserve omitted, `null`, malformed, and non-empty behavior.
- Add interaction tests proving `kv_load_tiers` does not alter `max_offload_tokens` store behavior.

### Files

- `vllm/v1/kv_offload/base.py`
- `vllm/v1/kv_offload/tiering/manager.py`
- `vllm/v1/kv_offload/cpu/manager.py`
- `tests/v1/kv_offload/tiering/test_tiering_offloading.py`
- `tests/v1/kv_offload/cpu/test_manager.py`
- connector scheduler tests and user documentation

### Acceptance

- `[]` prevents CPU and secondary loading.
- The opted-out request starts no lookup, promotion, or load work and is not deferred for offload retrieval.
- The request recomputes normally.
- Store behavior remains independently controlled by `max_offload_tokens`.
- Existing non-empty filters continue to behave as before.

## Work package B — llm-d-router policy and wiring

Detailed design: [[03 - llm-d-router selective load and offload policy and wiring]].

### Scope

- Add a request-control data producer plus `PreRequest` hook analogous to `p2psource`.
- Depend explicitly on precise-prefix-cache output.
- Compute a load decision for the selected endpoint using the configured threshold.
- Produce an independent optional offload cap.
- Overwrite client-provided policy fields with validated router decisions while preserving trusted connector-owned fields.
- Carry both fields through direct EPP, sidecar, P/D, coordinator, native generate, retry, and continuation paths as applicable.
- Gate emission by known vLLM capability.

### Conceptual decision type

```go
type SelectiveKVDecision struct {
    LoadState       LoadDecision
    LoadTiers       []TierMatcher
    MaxOffloadTokens *int
    LoadReason      LoadDecisionReason
    OffloadReason   OffloadDecisionReason
}
```

Use repository naming conventions during implementation. The important property is that load and offload have separate state and reason fields.

### Acceptance

- The selected endpoint’s evidence drives the decision.
- Explicit empty list and omitted load field remain distinct.
- `max_offload_tokens` survives every supported request path.
- All four combinations in the independent-decision matrix are tested.
- Conflicting client values cannot override authoritative router policy.
- Disabled, unknown, stale, and incompatible states preserve existing behavior.

## Work package C — calibration

The initial code should be configurable before a default threshold is chosen.

### Research question

For each relevant source tier, at what reusable prefix size does load latency become lower than recomputation latency under representative concurrency and tier pressure?

### Minimum experiment matrix

- baseline: recompute/no offload load;
- CPU load;
- storage load through CPU staging;
- multiple reusable prefix lengths around the expected break-even region;
- representative prompt/output shapes;
- low and target production concurrency;
- warm and pressured CPU/storage conditions;
- at least enough repetitions to report variability.

### Metrics

- TTFT and end-to-end latency;
- request throughput and output-token rate;
- external/reused versus locally recomputed prompt tokens;
- CPU-to-GPU and storage-to-CPU transfer bytes, bandwidth, and latency;
- scheduler running/waiting requests;
- CPU tier occupancy/pressure and storage latency/queue depth where available;
- request errors, cancellations, scheduler restarts, and timeouts.

Record model, revision/image, block size, offload chunk size, tensor/data parallelism, replicas, CPU capacity, secondary tier configuration, concurrency, seed, duration, and cache-cleaning state. Do not publish a universal threshold from one topology.

## Compatibility and rollout

1. Land or otherwise include the vLLM binary empty-list behavior.
2. Resolve or account for vLLM PR #52397 before enabling non-null offload caps on affected models.
3. Add router plugin and direct request tests with the feature disabled by default.
4. Add propagation for the actual deployed request path.
5. Run observe-only mode: compute/log decisions without mutating payloads.
6. Calibrate a static threshold.
7. Enable load enforcement for a compatible homogeneous pool.
8. Enable offload override separately.
9. Consider per-tier and dynamic policy only after metrics explain the static rollout.

Use explicit capability levels:


| Capability               | Allowed router output                               |
| ------------------------ | --------------------------------------------------- |
| unknown                  | omit both fields                                    |
| offload-cap-v1           | `max_offload_tokens` only, subject to #52397 safety |
| binary-load-opt-out-v1   | `kv_load_tiers=[]` and safe offload cap             |
| source-selective-load-v1 | empty, CPU, STORAGE, combined, and safe offload cap |


## Cross-repository tests

1. Default/default: both fields omitted; existing behavior preserved.
2. Load denied/offload denied: recompute; no offload store.
3. Load denied/offload allowed: recompute; computed prefix can be stored.
4. Load allowed/offload denied: existing KV can load; no new KV store.
5. Load allowed/offload allowed: normal load/store.
6. Missing/stale evidence: fields omitted.
7. Older/incompatible engine: fields omitted.
8. Retry to another endpoint: decision recomputed.
9. P/D: each leg receives only its own decision.
10. Native generate: fields appear under `sampling_params.extra_args.kv_transfer_params`.
11. Partial-tail model with cap below boundary and zero cap: no assertion or scheduler failure once the #52397 fix is present.
12. Malformed/client-conflicting values: router emits canonical validated values.

## Suggested validation commands

vLLM:

```bash
.venv/bin/python -m pytest tests/v1/kv_offload/cpu/test_manager.py tests/v1/kv_offload/tiering/test_tiering_offloading.py tests/v1/kv_connector/unit/offloading_connector/test_scheduler.py -v
pre-commit run ruff-check --all-files
```

llm-d-router:

```bash
make test-filter PATTERN=Selective TYPE=epp
make test-filter PATTERN=Selective TYPE=sidecar
make test-unit
make presubmit
```

## PR sequence

### PR 1 — vLLM binary load opt-out

Implement only explicit-empty semantics for primary CPU and secondary tiers plus interaction tests with the existing store cap. Reference the appropriate vLLM KV-offload RFC/issue and explain how this differs from open PR #52397.

### PR 2 — router policy and direct wiring

Open a llm-d-router tracking issue first. Add static load-threshold policy, independent offload modes, typed decision state, direct `PreRequest` mutation, metrics, configuration, and tests.

### PR 3 — deployed-path propagation

Wire both fields through the required sidecar/coordinator/P-D paths, with full-body backend tests. Combine with PR 2 only if the production path cannot exercise PR 2 independently.

### PR 4 — calibration and default recommendation

Run the experiment matrix, record accepted/rejected runs, and publish a topology-specific threshold recommendation. This may be a documentation/configuration contribution rather than code.

### Later — full source-selective and dynamic policy

Add source-aware promotion state in vLLM, then allow the router to emit CPU/STORAGE matchers. Add dynamic pressure/cost inputs only after the static policy is measured.

## Immediate implementation checklist

- [ ] Choose or open the vLLM tracking issue and inspect its comments.
- [ ] Open a llm-d-router tracking issue.
- [ ] Re-run duplicate searches immediately before each PR.
- [ ] Confirm whether PR #52397 has landed or needs to be included/rebased.
- [ ] Implement and test vLLM `kv_load_tiers=[]` semantics.
- [ ] Agree on the router plugin/config names.
- [ ] Implement a pure, table-tested policy function.
- [ ] Add selected-endpoint `PreRequest` mutation.
- [ ] Wire the deployed sidecar/coordinator path.
- [ ] Run observe-only calibration before choosing a default threshold.
- [ ] Record test commands, results, compatibility, and AI assistance in each PR.

## Risks

- **Primary CPU bypass:** current vLLM can violate an empty load list by loading CPU.
- **Promotion correctness:** source-selective CPU filtering can strand or leak a storage promotion; defer it beyond the binary MVP.
- **Partial-tail cap bug:** current `max_offload_tokens` can hit the open #52397 failure on affected models.
- **Stale router evidence:** false recompute decisions can waste reusable KV; omit on uncertainty.
- **Wrong-endpoint policy:** pre-scheduling or reused retry state can describe a different pod.
- **Sidecar overwrite:** reconstructed transfer maps can silently discard either field.
- **Coupled decisions:** using one threshold for both directions can suppress valuable future reuse.
- **Mixed engine versions:** older engines can accept the request but enforce different semantics.
- **Untrusted transfer metadata:** blindly preserving client maps can expose connector addresses or directives.

## Open questions

1. Which model/topology/tier combinations need independent static thresholds?
2. What exact freshness signal is available for precise-prefix evidence?
3. Which production request path must be supported in the first router PR?
4. Should static offload mode be global, route-profile-specific, or supplied by another trusted policy plugin?
5. How is compatible vLLM capability represented in deployment configuration?
6. Should the router override client policy in every case, or is there a separate trusted internal caller class?
7. When should source-selective CPU/STORAGE behavior graduate from binary opt-out?
8. Which runtime signals are stable enough for a later dynamic threshold?

## Related

- [[00 - Index]]
- [[02 - vLLM primary-tier selective loading]]
- [[03 - llm-d-router selective load and offload policy and wiring]]
- [[vLLM and llm-d-router KV cache responsibility split]]
- [[vLLM KV offload retrieval path - lookup, promotion, and load]]
- [[01 - Calibration Protocol]]

## Provenance

This plan combines direct inspection of both repositories, stakeholder clarification from Maroon, vLLM PRs #39983, #48123, and open #52397, llm-d-router issue #1952, and duplicate-work checks performed on 2026-09-03. It is an implementation proposal, not deployed behavior.