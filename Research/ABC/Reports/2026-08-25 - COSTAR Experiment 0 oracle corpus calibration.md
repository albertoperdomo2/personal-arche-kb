---
title: "COSTAR Experiment 0 — AgentX oracle-corpus calibration"
date: "2026-08-25"
type: "research-experiment-report"
experiment: "ABC / COSTAR-KV Experiment 0"
status: "conditionally-valid"
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_format: "FP8"
vllm_version: "0.27.0"
image: "quay.io/rh-ee-aperdomo/vllm@sha256:0e79705305e63b50ac80454641c4ca277014ae0c47a664df5784a46eb079e17f"
tensor_parallelism: 8
replicas: 1
accelerator: "H100"
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: "vLLM default"
cpu_bytes: 274877906944
cpu_blocks: 131072
kv_bytes_per_chunk: 2097152
offload_spec: "TieringOffloadingSpec"
secondary_tier: "filesystem-backed node-local NVMe"
secondary_tier_threads: "64 read / 64 write"
workload: "AgentX Weka; semianalysisai/cc-traces-weka-062126"
benchmark_profile: "aiperf-agentx-inference"
random_seed: 20260707
duration_seconds: 1800
concurrency:
  f0ea8db6be2044d9a3affbaffbbb87a0: 32
  f306ab08fb1045c3af877439b778d62e: 64
cache_cleaning_state: "not explicitly recorded in MLflow"
oracle_trace_schema: 1
---

# COSTAR Experiment 0 — AgentX oracle-corpus calibration

## Executive summary

This batch asks whether stock-reactive vLLM can emit a sufficiently complete trace for the offline COSTAR-KV oracle without materially changing the workload. It does **not** test whether a prefetch policy improves performance.

Two 30-minute AgentX/Weka cells ran with identical model, immutable image digest, TP8, 256 GiB CPU KV tier, and filesystem-backed NVMe. The only intended workload difference was concurrency 32 versus 64. Both benchmark clients completed without reported request errors or cancellation.

Both traces are valid oracle inputs. C32 contains 2,241,218 events. The manually recovered 6.959 GB C64 trace contains 13,629,779 events. Both passed sequence, lifecycle, transfer-join, source-readiness, CPU-generation, occupancy, and terminal-checkpoint checks. After correcting the offline target boundary to the first resolved lookup, native movement reconstruction has zero mismatches in both corpora.

Most importantly, C32 shows that HTTP admission normally offers only milliseconds of lead time while request working sets are several GiB. It also confirms that the CPU tier reaches full physical capacity. Those observations motivate a physically constrained offline oracle and weaken any expectation that a simple admission-time prefetch can stage complete requests.

## Validity verdict — Conditionally valid

- **C32 corpus: valid and accepted for normalization/replay work.**
- **C64 benchmark: valid as a pressure-regime workload result.**
- **C64 corpus: valid and accepted.** Exact size 6,959,277,072 bytes; SHA-256 `1167b512741bb97d2b76744cb238ede58fa0c6c2ef35ad7b3e9892c05b4ece3d`; 13,629,779 normalized events; zero native replay inconsistencies.
- **C32 versus C64 is not a prefetch A/B.** Their latency difference must not be interpreted as an instrumentation treatment effect.
- This is an initial corpus batch, not the complete Experiment 0 freeze. Replication, native-reactive replay acceptance, normalized tables, and a controlled source-prepopulation variant remain outstanding.

## Main takeaways

- **Measured:** both runs used exact image digest `sha256:0e7970…e17f`, trace schema 1, stock reactive behavior, default `max-num-seqs`, and the intended CPU/NVMe hierarchy.
- **Measured:** C32 passed with 2,241,218 events and C64 passed with 13,629,779 events; both have zero lifecycle, transfer, capacity, or native-movement replay errors.
- **Measured:** C32 admission-to-first-connector-lookup lead time was 7.48 ms median and 25.15 ms p95; its exact first-lookup working-set snapshot averaged 4,012.81 chunks, approximately 7.84 GiB.
- **Measured:** reconstructed CPU occupancy reached exactly 131,072/131,072 blocks and never exceeded capacity.
- **Measured:** C64 created much greater pressure: mean effective concurrency was 29.59 versus 9.87, mean TTFT was 8.89 s versus 1.12 s, and mean ITL was 57.73 ms versus 27.66 ms.
- **Inference:** admission is ordinarily far too late to stage a complete multi-GiB request from NVMe; useful future information probably needs to arrive before HTTP admission or the oracle must select only deadline-feasible request completions.
- **Inference:** speculative CPU placement cannot be modeled as consuming free space after warm-up. It must price eviction consequences.
- **Operational finding:** single multi-GB JSONL artifacts are unsuitable for the current MLflow proxy. Future traces should be compressed and/or rotated into independently validated chunks.

## Run registry and headline metrics

| Cell | Run | Concurrency | Measured completions | Request throughput | Mean / p95 TTFT | Mean request latency | Mean ITL | Trace disposition |
|---|---|---:|---:|---:|---:|---:|---:|---|
| Lower pressure | [f0ea8db6be2044d9a3affbaffbbb87a0](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f0ea8db6be2044d9a3affbaffbbb87a0?workspace=benchflow) | 32 | 860 | 0.4674 req/s | 1.121 / 4.310 s | 21.112 s | 27.658 ms | **Valid:** 1.635 GB, 2,241,218 events, validator passed |
| Higher pressure | [f306ab08fb1045c3af877439b778d62e](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/f306ab08fb1045c3af877439b778d62e?workspace=benchflow) | 64 | 1,196 | 0.6500 req/s | 8.885 / 26.207 s | 45.522 s | 57.727 ms | **Valid:** 6.959 GB, 13,629,779 events, validator and native replay passed |

AIPerf reported no errors and `was_cancelled=false` in both measured profiles. The C32 server trace contains 901 complete request lifecycles rather than 860 because it also covers readiness or warm-up traffic outside AIPerf's measured set.

## Workload pressure and performance context

Figure 1 compares the two workload regimes relative to C32. These ratios describe greater workload pressure, not a prefetch treatment.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1 — C64 outcome ratios relative to C32","width":680,"height":300,"data":{"values":[{"measure":"Request throughput","ratio":1.391,"configuration":"C64 / C32"},{"measure":"Mean TTFT","ratio":7.927,"configuration":"C64 / C32"},{"measure":"p95 TTFT","ratio":6.081,"configuration":"C64 / C32"},{"measure":"Mean request latency","ratio":2.156,"configuration":"C64 / C32"},{"measure":"Mean ITL","ratio":2.087,"configuration":"C64 / C32"}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"measure","type":"nominal","title":"Outcome metric","sort":["Request throughput","Mean TTFT","p95 TTFT","Mean request latency","Mean ITL"],"axis":{"labelAngle":-25}},"y":{"field":"ratio","type":"quantitative","title":"C64 / C32 ratio","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Comparison","scale":{"scheme":"category10"}},"tooltip":[{"field":"measure","type":"nominal","title":"Metric"},{"field":"ratio","type":"quantitative","title":"Ratio","format":".3f"}]}}
~~~

Source: AIPerf profile exports stored with the two MLflow runs. C64 processed 39.1% more requests per second, but mean TTFT increased 7.93× and p95 TTFT 6.08×. The higher-throughput cell is therefore a useful saturated/queued oracle corpus, not a better serving configuration.

Figure 2 makes the scheduler-pressure difference explicit.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 2 — Mean effective concurrency","width":500,"height":280,"data":{"values":[{"configuration":"Configured C32","effective_concurrency":9.868},{"configuration":"Configured C64","effective_concurrency":29.590}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Workload cell"},"y":{"field":"effective_concurrency","type":"quantitative","title":"Mean effective concurrency (requests)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Workload cell","scale":{"scheme":"category10"}},"tooltip":[{"field":"configuration","type":"nominal","title":"Cell"},{"field":"effective_concurrency","type":"quantitative","title":"Mean requests","format":".3f"}]}}
~~~

Source: native AIPerf effective-concurrency statistics. The scenario's pacing and idle gaps mean configured concurrency is a ceiling rather than the continuously active request count. Nevertheless, C64 produced roughly 3× the effective concurrency of C32.

## C32 trace-integrity evidence

The validator returned:

~~~json
{"valid":true,"event_count":2241218,"errors":[],"warnings":["validated an open trace through its final idle checkpoint"]}
~~~

The warning is expected. The pod remained alive, and the collector copied a trace ending at `TRACE_CHECKPOINT(reason="tier_manager_idle")` rather than process-shutdown `TRACE_END`.

| Invariant evidence | C32 observation |
|---|---:|
| Request arrivals / first lookups / ready / terminal | 901 / 901 / 901 / 901 |
| Terminal status | 898 length-capped; 3 aborted |
| Transfer submit / I/O start / I/O finish / result observed | 1,435 / 1,435 / 1,435 / 1,435 |
| Ordinary persistence jobs | 1,423 |
| Native-demand jobs | 12 |
| CPU resident enter / ready / exit | 416,709 / 416,709 / 285,637 |
| Maximum / final reconstructed CPU occupancy | 131,072 / 131,072 blocks |
| Sequence gaps or hard validator errors | 0 |

## Timing and physical opportunity

Figure 3 compares observable C32 timing boundaries. Values are native request quantiles; no smoothing or downsampling was used.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 3 — C32 request-boundary timing","width":680,"height":300,"data":{"values":[{"boundary":"Admission→first lookup p50","latency_ms":7.483},{"boundary":"Admission→first lookup p95","latency_ms":25.145},{"boundary":"First lookup→ready p50","latency_ms":16.593},{"boundary":"First lookup→ready p95","latency_ms":976.513}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"boundary","type":"nominal","title":"Request lifecycle boundary","sort":["Admission→first lookup p50","Admission→first lookup p95","First lookup→ready p50","First lookup→ready p95"],"axis":{"labelAngle":-20}},"y":{"field":"latency_ms","type":"quantitative","title":"Elapsed time (ms)","scale":{"zero":true}},"color":{"field":"boundary","type":"nominal","title":"Boundary / quantile","scale":{"scheme":"category10"}},"tooltip":[{"field":"boundary","type":"nominal","title":"Boundary"},{"field":"latency_ms","type":"quantitative","title":"Latency (ms)","format":".3f"}]}}
~~~

Source: direct reconstruction from the validated C32 JSONL. The admission window is normally only a few milliseconds. This is physically incompatible with moving an average 7–8 GiB working set from NVMe before first lookup. The next question is not “can we select the right blocks at admission?” but “which native CPU arrivals should be retained so the relatively rare, large external-reuse requests avoid movement?”

C32 working-set and lookup facts:

- Complete ordered first-lookup working-set snapshot: mean 4,012.81 chunks / 7.84 GiB; median 3,900 chunks / 7.62 GiB; p95 6,819 chunks / 13.32 GiB.
- Authoritative external contribution: mean 174.56 chunks across all requests; median/p95 0; maximum 7,311; only 42/901 requests matched any external tokens. The earlier 3,604 figure incorrectly summed total cached group counts, which include GPU-local chunks.
- First lookup attempt: 891 deferred, 10 resolved immediately.
- All 901 requests eventually resolved and emitted `REQUEST_READY`.
- CPU reached its 131,072-block ceiling, proving a finite-CPU oracle needs explicit victim accounting.

## Transfer composition

Figure 4 shows observed C32 submitted transfer volume by provenance.

~~~vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 4 — C32 transfer volume by provenance","width":500,"height":280,"data":{"values":[{"provenance":"Ordinary persistence","volume_gib":719.268},{"provenance":"Native demand","volume_gib":94.617}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"provenance","type":"nominal","title":"Transfer provenance"},"y":{"field":"volume_gib","type":"quantitative","title":"Submitted transfer volume (GiB)","scale":{"zero":true}},"color":{"field":"provenance","type":"nominal","title":"Provenance","scale":{"scheme":"category10"}},"tooltip":[{"field":"provenance","type":"nominal","title":"Provenance"},{"field":"volume_gib","type":"quantitative","title":"Volume (GiB)","format":".3f"}]}}
~~~

Source: C32 `TRANSFER_SUBMIT` events. Ordinary GPU→CPU→secondary persistence dominated submitted bytes. Native-demand secondary reads comprised only 12 large jobs but 94.62 GiB, so the oracle must model batching and link occupancy rather than independent per-block I/O.

Observed tier-I/O service time was 148.14 ms mean, 88.96 ms median, 301.23 ms p95, and 4.85 s maximum per batch. These are job-level measurements, not per-block latency.

## Instrumentation overhead check

The traced C32 aggregate is close to two earlier RHOAI 3.5 C32 controls using the same AgentX seed and workload family: request throughput approximately 0.35% lower, mean TTFT 0.54% higher, p95 TTFT 0.28% lower, and mean ITL 2.1% higher. This supports, but does not fully prove, low perturbation because the controls were separate deployments rather than a same-node crossover. C64 needs a contemporary trace-disabled sibling before its timing perturbation can be bounded confidently.

## C64 certification update

The complete leaf was manually downloaded and matches MLflow's declared size exactly: 6,959,277,072 bytes. SHA-256 is `1167b512741bb97d2b76744cb238ede58fa0c6c2ef35ad7b3e9892c05b4ece3d`.

Normalization produced 13,629,779 events, 1,280 closed requests, 2,480 closed transfers, and 1,564,329 CPU residency generations. Maximum and final reconstructed occupancy are exactly 131,072 blocks. The native replay is internally consistent.

The operational lesson remains: ordinary single-response MLflow proxy download timed out. MLflow presigned multipart download works but was slow through this client path. Future traces should still be compressed and rotated into independently verifiable chunks.

## What this establishes—and what it does not

### Established

1. The instrumentation can capture a complete, internally consistent C32 request/transfer/cache trace under realistic AgentX load.
2. C32 instrumentation overhead appears small relative to prior comparable controls.
3. The CPU tier becomes physically full.
4. Typical HTTP-admission lead time is milliseconds while complete request working sets are several GiB.
5. C32 and C64 provide meaningfully different lower- and higher-pressure regimes.

### Not established

1. That any live prefetch policy improves TTFT or throughput.
2. That perfect future knowledge has sufficient physical value.
3. That C64's higher pressure or trace validity proves any practical placement policy improves end-to-end performance.
4. That C32 and C64 outcome differences are caused by tracing.
5. That raw events already reproduce native ready/deferred timing accurately enough for a clairvoyant counterfactual.

## Conclusion

Experiment 0 has passed its corpus gate: both C32 and C64 are normalized, capacity-consistent, and trustworthy enough for offline placement reconstruction. The evidence sharpens the research problem. Admission-time block selection is unlikely to hide complete NVMe movement because the horizon is normally tens of milliseconds while the target is several GiB and CPU is full.

This does **not** say prefetching is a dead end. It says the defensible remaining version is harder and earlier: reveal a future request early enough, move the deadline-feasible portion of its complete working set, and jointly decide which CPU state may be displaced. The offline oracle must now determine whether even perfect information creates enough benefit to justify building that mechanism.

## Next steps

1. Preserve the accepted C64 checksum and validator result with the corpus record.
2. Convert future traces into chunked/compressed Parquet or Arrow tables with explicit request, key, source, transfer, and CPU-generation joins.
3. Implement native-reactive replay acceptance: reproduce ready/deferred outcomes exactly and fit transfer/ready timing within a predeclared error envelope.
4. Add compressed/rotated trace output or collection so every segment can be downloaded and validated independently.
5. Collect at least one repeat per pressure cell and one controlled source-prepopulation variant before freezing the corpus.
6. Continue with the physically constrained retention oracle recorded in [[Reports/COSTAR Offline Oracle/00 - Index|the COSTAR offline oracle series]]. Do not implement another online heuristic before the oracle quantifies avoided native movement.
## Post-publication semantic correction — 2026-08-25

Offline target reconstruction found that `matched_key_counts` records the total cached prefix, including GPU-local chunks; it is not the external CPU target. The authoritative external segment is derived from `matched_tokens` and each group's chunk size. Corrected analysis finds 157,283 external references, 116,409 unique external keys, and only 42/901 requests with nonzero external reuse. See [[Reports/COSTAR Offline Oracle/04 - Finite CPU retention oracle|finite-CPU retention oracle]].

## Post-publication C64 certification and target-boundary correction — 2026-08-25

The C64 trace is now accepted. The complete 6,959,277,072-byte object has SHA-256 `1167b512741bb97d2b76744cb238ede58fa0c6c2ef35ad7b3e9892c05b4ece3d`, normalizes to 13,629,779 events, and passes lifecycle, transfer, capacity, and native movement reconstruction.

C64 exposed a target-selection bug in the offline tools: the last resolved connector lookup may belong to a later decode-era working-set version. Initial TTFT/placement analysis must use the first resolved lookup. After applying that correction consistently, C32 remains at 0/898 movement mismatches and C64 reaches 0/1,263. See [[../Future-Value Placement/06 - Experiment 6 C64 independent pressure validation|Experiment 6]].