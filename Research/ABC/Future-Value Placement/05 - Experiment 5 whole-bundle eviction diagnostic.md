---
title: "COSTAR-KV Experiment 5 — whole-bundle eviction diagnostic"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "COSTAR-KV future-value placement / Experiment 5"
status: "valid-negative-result"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
vllm_version: "v0.27.0-derived offline tooling"
code_base_revision: "e50f7d36960980c0c89651ffd0ce281a9fb8a466 plus uncommitted tools/costar experiment code"
tensor_parallelism: 8
concurrency: 32
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
secondary_tier: "filesystem-backed node-local NVMe"
workload: "AgentX/Weka C32"
random_seed: "deterministic; no randomized policy"
cache_cleaning_state: "offline replay of recorded native state"
source_run: "f0ea8db6be2044d9a3affbaffbbb87a0"
corpus_sha256: "788adedcbc9850a64443681fbd8acf36ea62cdce950eeac6311f730f34cce687"
---

# COSTAR-KV Experiment 5 — whole-bundle eviction diagnostic

## Executive summary

Experiment 4 preserved source-request identity but still removed victims one block at a time. Experiment 5 tests whether evicting one complete old source bundle whenever capacity binds improves complete future working sets, even if that temporarily leaves CPU slots unused.

It does not. Whole-bundle FIFO completes the same 27/42 external targets as blockwise bundle FIFO and retains fewer target keys. Whole-bundle LRU completes the same 26/42 as blockwise bundle LRU. Both remain worse than recorded placement.

## Validity verdict — Valid negative mechanism result

The comparison uses identical recorded arrivals and nominal 131,072-slot capacity. Whole-bundle policies are allowed to leave temporary slack after coarse eviction, and that utilization loss is measured.

This is valid for rejecting **source-request identity plus FIFO/LRU ordering plus whole-bundle victims** on C32. It does not reject richer prefix subdivisions, future-value scores, session/workflow metadata, or cost-aware bundle packing.

## Results

| Policy | Complete targets | Key-hit rate | Reads avoided | New misses | Net miss delta | Min resident after full | Time-weighted utilization |
|---|---:|---:|---:|---:|---:|---:|---:|
| Blockwise bundle FIFO | 27/42 (64.29%) | 61.88% | 3/12 | 6 | **+3** | 131,072 | 85.18% |
| Blockwise bundle LRU | 26/42 (61.90%) | 61.02% | 2/12 | 6 | **+4** | 131,072 | 85.18% |
| Whole-bundle FIFO | 27/42 (64.29%) | 59.51% | 3/12 | 6 | **+3** | 123,753 | 84.31% |
| Whole-bundle LRU | 26/42 (61.90%) | 60.38% | 2/12 | 6 | **+4** | 123,753 | 84.22% |
| Recorded physical reference | 30/42 (71.43%) | 69.20% | 0/12 | 0 | 0 | N/A | N/A |
| Clairvoyant next-use reference | 42/42 (100%) | 100% | 12/12 | 0 | -12 | N/A | N/A |

The utilization percentage includes the initial cache-fill period. The discriminating statistic is the minimum after first reaching capacity: whole-bundle eviction creates up to 7,319 empty slots, about 5.58% of nominal CPU capacity.

## Main takeaways

- **Measured:** whole-bundle eviction changes 388 FIFO and 421 LRU victim events, but changes zero request-completeness outcomes.
- **Measured:** whole-bundle FIFO loses 3,726 key hits relative to blockwise FIFO.
- **Measured:** coarse victims introduce measurable occupancy slack without reducing net misses.
- **Conclusion:** fragmentation caused by blockwise eviction was not the limiting factor for FIFO/LRU on C32.
- **Conclusion:** request-bundle identity is useful as a representation, but still requires a predictive value/cost signal. Granularity alone is not a policy.

## Evidence

Figure 1 compares the primary outcome. Provenance: the 42 authoritative external targets under exact common-arrival replay.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — Whole-bundle eviction does not improve complete targets","width":650,"height":285,"data":{"values":[{"policy":"Block FIFO","complete_percent":64.29,"granularity":"Blockwise"},{"policy":"Whole FIFO","complete_percent":64.29,"granularity":"Whole bundle"},{"policy":"Block LRU","complete_percent":61.90,"granularity":"Blockwise"},{"policy":"Whole LRU","complete_percent":61.90,"granularity":"Whole bundle"},{"policy":"Recorded","complete_percent":71.43,"granularity":"Reference"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy","type":"nominal","title":"Victim policy","sort":null,"axis":{"labelAngle":-18}},"y":{"field":"complete_percent","type":"quantitative","title":"External targets CPU-complete (%)","scale":{"zero":true}},"color":{"field":"granularity","type":"nominal","title":"Victim granularity","scale":{"scheme":"category10"}},"tooltip":[{"field":"policy","type":"nominal","title":"Policy"},{"field":"complete_percent","type":"quantitative","title":"Complete targets (%)","format":".2f"}]}}
~~~

Whole-bundle victims do not change request readiness.

Figure 2 shows the cost paid for that non-improvement. Provenance: exact replay occupancy integrated over event time; minimum measured after the cache first reaches capacity.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — Minimum occupied CPU slots after first reaching capacity","width":600,"height":280,"data":{"values":[{"policy":"Block FIFO","resident":131072},{"policy":"Whole FIFO","resident":123753},{"policy":"Block LRU","resident":131072},{"policy":"Whole LRU","resident":123753}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"policy","type":"nominal","title":"Victim policy","sort":null},"y":{"field":"resident","type":"quantitative","title":"Occupied CPU KV slots (count)","scale":{"zero":true}},"color":{"field":"policy","type":"nominal","title":"Policy","scale":{"scheme":"category10"}},"tooltip":[{"field":"policy","type":"nominal","title":"Policy"},{"field":"resident","type":"quantitative","title":"Minimum occupied slots","format":","}]}}
~~~

The coarse policy sacrifices up to 7,319 usable slots and receives no complete-request benefit.

## Decision

Reject whole-source-request FIFO/LRU eviction as a live candidate.

Together, Experiments 2–5 establish:

1. exact-key access history is absent when all valuable ordinary arrivals first appear;
2. source-request grouping is real but does not identify which group is valuable;
3. simple source context and post-hoc prompt bands regress net request misses;
4. whole-bundle victim granularity does not repair the ranking.

Do not mine more C32-only thresholds. With only 42 external reuse requests, further post-hoc rules would be increasingly unconstrained.

## Next data experiment

The next useful offline experiment requires an **independent or enriched trace**, not another C32 policy:

- certify/normalize the existing C64 trace if the source artifact can be validated;
- add stable session/conversation/user identity;
- add workload category, agent/tool/DAG event, and destination replica;
- preserve source request → generated prefix bundle identity;
- record the time at which a future continuation becomes knowable;
- record per-request exposed reactive stall, not only device service.

Then rerun Experiment 3 as a train/validation information audit. A feature must show held-out request-completion or cost-weighted ranking lift before another live implementation.

## Reproduction

- `tools/costar/whole_bundle_benchmark.py`
- `tools/costar/run_whole_bundle_benchmark.py`
- `tests/tools/test_costar_whole_bundle_benchmark.py`

~~~text
../vllm/.venv/bin/python -m tools.costar.run_whole_bundle_benchmark   /private/tmp/abc-oracle-validation-20260825/c32-normalized.sqlite
~~~

Focused verification: 3 tests passed; Ruff passed.

Source corpus: [MLflow C32 run f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow).