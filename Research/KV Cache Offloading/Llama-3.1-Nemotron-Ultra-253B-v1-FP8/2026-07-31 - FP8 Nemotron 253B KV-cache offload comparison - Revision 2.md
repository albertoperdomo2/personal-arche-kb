---
title: "2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2"
date: "2026-07-31"
type: "experiment-report"
topic: "KV Cache Offloading"
experiment: "MLflow experiment 328"
report_revision: 2
model: "nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8"
model_revision: "not recorded"
runtime_image: "vllm/vllm-openai:v0.23.0"
runtime_image_digest: "sha256:3a1e7f5904e1a1192a02aa0086ceaffc33985d7044c7bb25b3a43d61bdbe3ac0"
vllm_version: "0.23.0"
gpu_type: "H100"
gpu_count: 8
tensor_parallelism: 8
pipeline_parallelism: 1
replicas: 1
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: "not explicitly set"
concurrency: 32
cpu_bytes: 274877906944
offload_spec: "TieringOffloadingSpec for tiered configurations"
secondary_tier: "none, NVMe, or CephFS"
shared_memory: "300Gi"
workload: "AIPerf AgentX inferencex-agentx-mvp"
random_seed: 42
duration_seconds: 1800
configuration_count: 4
baseline: "No offload"
status: "directionally valid single-seed comparison; CephFS conditionally accepted"
---

# 2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2

## Benchmark overview

This experiment compares `nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8` on one eight-H100, TP8 replica under the AgentX MVP workload. Stable parameters are vLLM 0.23.0, 131,072-token context, FP8 KV cache, prefix caching, seed 42, a 1,800-second send window, and 300 GiB shared memory. The four configurations are no offload, a 256 GiB CPU tier, CPU plus local NVMe, and CPU plus CephFS.

This is **Revision 2** ofthe July 30 report. It is not a same-condition repetition: concurrency changes from 16 to 32 and `gpu-memory-utilization` changes from 0.90 to 0.80. Cross-revision comparisons are therefore directional. Within Revision 2, all four cells use the same U0.80/C32 condition.

## AIPerf benchmark invocation

The MLflow artifacts record the following complete AIPerf command. It is the same benchmark invocation for all four cells; BenchFlow substitutes only the deployment-specific service URL and artifact directory.

### Recorded command

```bash
aiperf profile --model 'nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8' \
  --url 'https://cpu-offloading-m1-ae1ba83977-kserve-workload-svc.benchflow.svc.cluster.local:8000' \
  --artifact-dir '/benchmark-results/remote-jobs/benchflow-benchmark-cpu-offloading-d3ca/benchmark' \
  --ui None --benchmark-duration 1800 --concurrency 32 \
  --endpoint '/v1/chat/completions' --endpoint-type 'chat' \
  --hf-weka-repo 'semianalysisai/cc-traces-weka-with-subagents-060826' \
  --max-context-length 131072 --public-dataset 'weka_hf' --random-seed 42 \
  --scenario 'inferencex-agentx-mvp' --streaming --tokenizer-trust-remote-code \
  --use-server-token-count \
  --tokenizer 'nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8'
```

## Executive summary

**Offload is now decisively active and beneficial.** The no-offload baseline serves only 2.8% of prompt tokens from local HBM and recomputes 97.2%. CPU256 supplies 60.9% externally and raises request throughput from 0.1323 to 0.4372 req/s (**3.31×**, +230.6%). NVMe and CephFS supply 65.7% and 64.9% externally and reach 0.7591 and 0.7645 req/s (**5.74×** and **5.78×** baseline).

The secondary tiers are the winners. NVMe and CephFS are at practical parity: CephFS is only +0.71% higher in a single seed. Both are about 74% faster than CPU-only because their longer retention cuts local recomputation to 10.7%, versus 22.3% for the finite 256 GiB CPU tier.

Latency follows the same mechanism. Relative to baseline, mean TTFT falls by 82.4% with CPU-only and 83.9% with NVMe; mean end-to-end latency falls by 74.0% and 80.0%. Completed sessions rise from 4 to 21 with CPU-only and 28 with either secondary tier.

**Validity verdict:** the mechanism and the large baseline/offload separation are directionally valid, but the matrix is one seed and all cells end with grace-period cancellations. CephFS is conditionally accepted: 1,399 successful requests are included, but two requests fail with `ServerDisconnectedError`, and the captured Ceph cluster is `HEALTH_WARN` for low MON disk space and two recent daemon crashes. CephFS should not be called strictly production-clean until a repeat has zero request errors and healthy infrastructure.

**Tuning decision:** keep TP8. U0.80 at C32 already forces spill and should not be lowered. Peak HBM KV utilization is 99.5–100.0%, with a small non-zero preemption signal. For the next clean comparison, **U0.82 at C32** is the recommended center: the run-local projection raises capacity from 1.795M to 1.900M tokens and lowers the projected peak to about 94.5%, while retaining substantial offload pressure.

## Headline results


| Configuration       | MLflow                                                                                                                                                                  | Req/s  | Output tok/s | Mean TTFT (s) | TTFT P95 (s) | Mean E2E (s) | Sessions | External prompt share | KV mean / peak | Errors |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------ | ------------ | ------------- | ------------ | ------------ | -------- | --------------------- | -------------- | ------ |
| No offload          | [5f9a464d28e84284943c8454f0bd5db3](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5f9a464d28e84284943c8454f0bd5db3?workspace=benchflow) | 0.1323 | 64.9         | 59.74         | 99.12        | 228.71       | 4 / 34   | 0.0%                  | 77.0% / 99.5%  | 0      |
| CPU offload 256 GiB | [17704f73746a417eba8cc6677570b5fc](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/17704f73746a417eba8cc6677570b5fc?workspace=benchflow) | 0.4372 | 280.4        | 10.52         | 32.60        | 59.40        | 21 / 51  | 60.9%                 | 76.3% / 99.9%  | 0      |
| CPU + NVMe          | [7d85703182174927aca7dcf490b9a7b8](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/7d85703182174927aca7dcf490b9a7b8?workspace=benchflow) | 0.7591 | 511.6        | 9.60          | 35.74        | 45.80        | 28 / 58  | 65.7%                 | 76.3% / 100.0% | 0      |
| CPU + CephFS        | [77246ae40d8b4a27a73dbdeb92cfa0a1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/77246ae40d8b4a27a73dbdeb92cfa0a1?workspace=benchflow) | 0.7645 | 515.6        | 9.63          | 35.92        | 45.76        | 28 / 57  | 64.9%                 | 74.6% / 99.7%  | 2      |


Request throughput is useful here but not perfectly workload-invariant: AgentX branching diverges as configurations complete different amounts of work. Output-token throughput, prompt-source shares, latency, and completed session/branch counts all corroborate the same ranking.

## Performance comparison

```vega-lite
{"width":740,"height":300,"title":"Figure 1 \u2014 Completed-request throughput","data":{"values":[{"configuration":"No offload","requests_s":0.13225405801708404},{"configuration":"CPU offload 256 GiB","requests_s":0.43722321612285625},{"configuration":"CPU + NVMe","requests_s":0.7591018658636823},{"configuration":"CPU + CephFS","requests_s":0.7645190487332728}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"requests_s","type":"quantitative","title":"Requests per second","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"requests_s","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

Figure 1 shows a large, mechanism-backed separation. CPU-only is 3.31× baseline; the persistent secondary tiers are about 1.74–1.75× CPU-only.

```vega-lite
{"width":740,"height":300,"title":"Figure 2 \u2014 Output-token throughput","data":{"values":[{"configuration":"No offload","tokens_s":64.91753546551442},{"configuration":"CPU offload 256 GiB","tokens_s":280.40983048627294},{"configuration":"CPU + NVMe","tokens_s":511.6264599477821},{"configuration":"CPU + CephFS","tokens_s":515.6443266918077}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"tokens_s","type":"quantitative","title":"Output tokens per second","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"tokens_s","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

Figure 2 confirms the result in token terms: CPU-only produces 280.4 output tok/s, NVMe 511.6, and CephFS 515.6, versus 64.9 for no offload.

```vega-lite
{"width":760,"height":320,"title":"Figure 3 \u2014 Request latency","data":{"values":[{"configuration":"No offload","metric":"TTFT mean","seconds":59.74131100185062},{"configuration":"No offload","metric":"TTFT P95","seconds":99.124215964},{"configuration":"No offload","metric":"E2E mean","seconds":228.71392998856018},{"configuration":"No offload","metric":"E2E P95","seconds":574.723053559},{"configuration":"CPU offload 256 GiB","metric":"TTFT mean","seconds":10.517493055411249},{"configuration":"CPU offload 256 GiB","metric":"TTFT P95","seconds":32.60349589324998},{"configuration":"CPU offload 256 GiB","metric":"E2E mean","seconds":59.40496560353},{"configuration":"CPU offload 256 GiB","metric":"E2E P95","seconds":185.10616478969985},{"configuration":"CPU + NVMe","metric":"TTFT mean","seconds":9.595415945570194},{"configuration":"CPU + NVMe","metric":"TTFT P95","seconds":35.739813188599996},{"configuration":"CPU + NVMe","metric":"E2E mean","seconds":45.801652921855286},{"configuration":"CPU + NVMe","metric":"E2E P95","seconds":143.26430098799997},{"configuration":"CPU + CephFS","metric":"TTFT mean","seconds":9.629986047125088},{"configuration":"CPU + CephFS","metric":"TTFT P95","seconds":35.9218674507},{"configuration":"CPU + CephFS","metric":"E2E mean","seconds":45.7603873834589},{"configuration":"CPU + CephFS","metric":"E2E P95","seconds":145.43464404760002}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["TTFT mean","TTFT P95","E2E mean","E2E P95"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"seconds","type":"quantitative","title":"Seconds","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"seconds","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

The baseline is capacity-bound: mean TTFT is 59.7 seconds and mean E2E latency is 228.7 seconds. All offload tiers sharply reduce both.

```vega-lite
{"width":760,"height":320,"title":"Figure 4 \u2014 Request, session, and branch outcomes","data":{"values":[{"configuration":"No offload","metric":"Requests completed","count":241},{"configuration":"No offload","metric":"Requests cancelled","count":25},{"configuration":"No offload","metric":"Sessions completed","count":4},{"configuration":"No offload","metric":"Branches completed","count":33},{"configuration":"CPU offload 256 GiB","metric":"Requests completed","count":800},{"configuration":"CPU offload 256 GiB","metric":"Requests cancelled","count":18},{"configuration":"CPU offload 256 GiB","metric":"Sessions completed","count":21},{"configuration":"CPU offload 256 GiB","metric":"Branches completed","count":49},{"configuration":"CPU + NVMe","metric":"Requests completed","count":1389},{"configuration":"CPU + NVMe","metric":"Requests cancelled","count":67},{"configuration":"CPU + NVMe","metric":"Sessions completed","count":28},{"configuration":"CPU + NVMe","metric":"Branches completed","count":223},{"configuration":"CPU + CephFS","metric":"Requests completed","count":1401},{"configuration":"CPU + CephFS","metric":"Requests cancelled","count":64},{"configuration":"CPU + CephFS","metric":"Sessions completed","count":28},{"configuration":"CPU + CephFS","metric":"Branches completed","count":224}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Requests completed","Requests cancelled","Sessions completed","Branches completed"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"count","type":"quantitative","title":"Count","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"count","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

The faster cells progress much further through the stateful workload. Secondary tiers complete 28 sessions and 223–224 branches; CPU-only completes 21 sessions and 49 branches; baseline completes 4 sessions and 33 branches.

## Why this revision forces offload

```vega-lite
{"width":760,"height":320,"title":"Figure 5 \u2014 Prompt tokens by source","data":{"values":[{"configuration":"No offload","source":"External KV transfer","share":0.0},{"configuration":"No offload","source":"Local HBM hit","share":2.802174574060601},{"configuration":"No offload","source":"Local compute","share":97.19782542593939},{"configuration":"CPU offload 256 GiB","source":"External KV transfer","share":60.94027988980015},{"configuration":"CPU offload 256 GiB","source":"Local HBM hit","share":16.795046979094092},{"configuration":"CPU offload 256 GiB","source":"Local compute","share":22.26467313110575},{"configuration":"CPU + NVMe","source":"External KV transfer","share":65.6859905486354},{"configuration":"CPU + NVMe","source":"Local HBM hit","share":23.635042870718173},{"configuration":"CPU + NVMe","source":"Local compute","share":10.67896658064642},{"configuration":"CPU + CephFS","source":"External KV transfer","share":64.85164214805714},{"configuration":"CPU + CephFS","source":"Local HBM hit","share":24.456792497239},{"configuration":"CPU + CephFS","source":"Local compute","share":10.69156535470386}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"share","type":"quantitative","title":"Integrated prompt-token rate share (%)","stack":"zero","scale":{"domain":[0,100]}},"color":{"field":"source","type":"nominal","title":"Prompt source","scale":{"domain":["External KV transfer","Local HBM hit","Local compute"],"range":["#9467bd","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"source","type":"nominal"},{"field":"share","type":"quantitative","format":".3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

Revision 1 at U0.90/C16 peaked below 50% of a 2.314M-token HBM shelf, so at least 91.9% of prompt tokens were local HBM hits and external restoration was at most 0.15%. Revision 2 reduces the shelf to 1.795M tokens and doubles concurrency. Every cell reaches approximately 100% peak occupancy.

The result is a workload phase change:

- No offload loses reusable history and recomputes 97.2% of prompt tokens.
- CPU256 restores 60.9% externally, but its circular 256 GiB tier still lets enough history expire that 22.3% is recomputed.
- NVMe and CephFS retain a deeper history. They restore about 65% externally and recompute only 10.7%.

```vega-lite
{"width":760,"height":320,"title":"Figure 6 \u2014 HBM KV-cache utilization","data":{"values":[{"configuration":"No offload","metric":"Mean","percent":77.02685798271871},{"configuration":"No offload","metric":"Peak","percent":99.46619375651663},{"configuration":"CPU offload 256 GiB","metric":"Mean","percent":76.31649157527026},{"configuration":"CPU offload 256 GiB","metric":"Peak","percent":99.88147540837514},{"configuration":"CPU + NVMe","metric":"Mean","percent":76.28697319790747},{"configuration":"CPU + NVMe","metric":"Peak","percent":99.97682977908086},{"configuration":"CPU + CephFS","metric":"Mean","percent":74.57194000957536},{"configuration":"CPU + CephFS","metric":"Peak","percent":99.70591642679547}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Mean","Peak"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"percent","type":"quantitative","title":"KV-cache utilization (%)","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"percent","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

Mean KV occupancy is 74.6–77.0% because startup and drain samples are included, but the peaks show that the working shelf is fully exercised.

```vega-lite
{"width":760,"height":320,"title":"Figure 7 \u2014 Cumulative KV transfer volume","data":{"values":[{"configuration":"CPU offload 256 GiB","direction":"GPU \u2192 CPU","gib":3100.52734375},{"configuration":"CPU offload 256 GiB","direction":"CPU \u2192 GPU","gib":8177.609375},{"configuration":"CPU + NVMe","direction":"GPU \u2192 CPU","gib":2278.87109375},{"configuration":"CPU + NVMe","direction":"CPU \u2192 GPU","gib":12926.1328125},{"configuration":"CPU + CephFS","direction":"GPU \u2192 CPU","gib":2263.53125},{"configuration":"CPU + CephFS","direction":"CPU \u2192 GPU","gib":12825.58203125}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"direction","type":"nominal","title":"Metric","sort":["GPU \u2192 CPU","CPU \u2192 GPU"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"gib","type":"quantitative","title":"GiB","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"direction","type":"nominal"},{"field":"gib","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

CPU-only restores 8.18 TiB and stores 3.10 TiB. NVMe restores 12.93 TiB and stores 2.28 TiB; CephFS restores 12.83 TiB and stores 2.26 TiB. Repeated restoration makes cumulative reads legitimately exceed cumulative writes.

```vega-lite
{"width":740,"height":300,"title":"Figure 8 \u2014 Estimated preemptions during the send window","data":{"values":[{"configuration":"No offload","estimated_count":0.7066172839506173},{"configuration":"CPU offload 256 GiB","estimated_count":3.1292027434842247},{"configuration":"CPU + NVMe","estimated_count":10.951136625514405},{"configuration":"CPU + CephFS","estimated_count":8.677313701431494}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"estimated_count","type":"quantitative","title":"Estimated preemptions","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"estimated_count","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

The rate telemetry implies roughly 0.7, 3.1, 11.0, and 8.7 preemptions over the 1,800-second window. These are low relative to request counts, but they show that U0.80/C32 sits at the pressure boundary rather than a comfortable operating point.

## What changed from Revision 1

```vega-lite
{"width":760,"height":330,"title":"Figure 9 \u2014 Directional cross-revision throughput","data":{"values":[{"revision":"Revision 1 \u00b7 U0.90/C16","configuration":"No offload","requests_s":0.43270876395978086},{"revision":"Revision 1 \u00b7 U0.90/C16","configuration":"CPU offload 256 GiB","requests_s":0.43116858133457087},{"revision":"Revision 1 \u00b7 U0.90/C16","configuration":"CPU + NVMe","requests_s":0.4348582021805327},{"revision":"Revision 1 \u00b7 U0.90/C16","configuration":"CPU + CephFS","requests_s":0.43108445807122675},{"revision":"Revision 2 \u00b7 U0.80/C32","configuration":"No offload","requests_s":0.13225405801708404},{"revision":"Revision 2 \u00b7 U0.80/C32","configuration":"CPU offload 256 GiB","requests_s":0.43722321612285625},{"revision":"Revision 2 \u00b7 U0.80/C32","configuration":"CPU + NVMe","requests_s":0.7591018658636823},{"revision":"Revision 2 \u00b7 U0.80/C32","configuration":"CPU + CephFS","requests_s":0.7645190487332728}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"configuration","type":"nominal","title":"Configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"xOffset":{"field":"revision"},"y":{"field":"requests_s","type":"quantitative","title":"Requests per second","scale":{"zero":true}},"color":{"field":"revision","type":"nominal","title":"Experiment","scale":{"domain":["Revision 1 \u00b7 U0.90/C16","Revision 2 \u00b7 U0.80/C32"],"range":["#9ecae1","#08519c"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"revision","type":"nominal"},{"field":"requests_s","type":"quantitative","format":".4f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```


| Parameter / observation  | Revision 1          | Revision 2                         | Meaning                                     |
| ------------------------ | ------------------- | ---------------------------------- | ------------------------------------------- |
| `gpu-memory-utilization` | 0.90                | 0.80                               | Smaller HBM KV shelf                        |
| Concurrency              | 16                  | 32                                 | More simultaneous live context              |
| GPU KV capacity          | 2.314M tokens       | 1.795M tokens                      | 22.4% less HBM KV capacity                  |
| Peak HBM KV use          | 47.1–50.0%          | 99.5–100.0%                        | Offload pressure is now forced              |
| External prompt share    | 0–0.15%             | 60.9–65.7%                         | External tiers are on the critical path     |
| Result                   | All cells at parity | Offload wins; secondary tiers lead | The original hypothesis is now demonstrated |


Figure 9 must not be read as a controlled U-only or concurrency-only effect because both variables changed. It shows that the experiment moved from an underfilled regime to a saturated regime.

## GPU memory utilization

Startup reports 27.4 GiB of KV cache per GPU and 1,795,424 aggregate GPU KV tokens at U0.80. With the observed 128 KiB per token, each +0.01 of U adds approximately 6.4 GiB aggregate, or 52,429 tokens.

```vega-lite
{"width":760,"height":330,"title":"Figure 10 \u2014 C32 HBM calibration projection","data":{"values":[{"gpu_memory_utilization":0.8,"capacity_mtokens":1.795424,"projected_peak_percent":99.97682977908086},{"gpu_memory_utilization":0.82,"capacity_mtokens":1.9002815999999996,"projected_peak_percent":94.46010508614961},{"gpu_memory_utilization":0.84,"capacity_mtokens":2.0051392,"projected_peak_percent":89.52036827631544},{"gpu_memory_utilization":0.86,"capacity_mtokens":2.1099968,"projected_peak_percent":85.07159803715176},{"gpu_memory_utilization":0.9,"capacity_mtokens":2.319712,"projected_peak_percent":77.38064019554}]},"mark":{"type":"line","point":true,"strokeWidth":2,"tooltip":true},"encoding":{"x":{"field":"gpu_memory_utilization","type":"quantitative","title":"gpu-memory-utilization","scale":{"domain":[0.795,0.905]}},"y":{"field":"projected_peak_percent","type":"quantitative","title":"Projected peak KV utilization (%)","scale":{"domain":[70,102]}},"color":{"value":"#08519c"},"tooltip":[{"field":"gpu_memory_utilization","type":"quantitative","format":".2f"},{"field":"capacity_mtokens","type":"quantitative","title":"KV capacity (M tokens)","format":".3f"},{"field":"projected_peak_percent","type":"quantitative","title":"Projected peak (%)","format":".1f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

The projection holds demand constant at the largest observed Revision 2 peak:


| U at C32 | Projected GPU KV tokens | Projected peak occupancy | Use                                                      |
| -------- | ----------------------- | ------------------------ | -------------------------------------------------------- |
| 0.80     | 1.795M                  | 100.0%                   | Proven forced-spill showcase; slight preemption pressure |
| 0.82     | 1.900M                  | 94.5%                    | **Recommended clean-comparison center**                  |
| 0.84     | 2.005M                  | 89.5%                    | Headroom guardrail                                       |
| 0.90     | 2.320M                  | 77.4%                    | Likely still offloads at C32, but less strongly          |


These values are projections from this exact model/image/TP8 startup profile. Reconcile every rerun against the actual `GPU KV cache size` log line. The correct U depends on concurrency: the earlier U0.68 recommendation applied to C16; it should **not** be carried into C32.

## Tensor parallelism

**Keep TP8.** The new result proves that TP8 can force offload without changing the compute topology. The model loads at 30.06 GiB per GPU, and the cache shape `(112214, 1, 64, 2, 16, 128)` remains cleanly sharded.

- TP4 would approximately double per-GPU weight pressure and halve the compute fabric. It would force memory pressure, but it would confound offload benefit with a different compute regime.
- TP16 would add aggregate HBM and could introduce KV-head replication, moving away from the intended pressure point.
- Change TP only in a separate deployment-efficiency experiment after the TP8 result is replicated.

## NVMe and CephFS

```vega-lite
{"width":760,"height":320,"title":"Figure 11 \u2014 Secondary-tier mean throughput","data":{"values":[{"configuration":"CPU + NVMe","metric":"Read","value":693.6298589767661},{"configuration":"CPU + NVMe","metric":"Write","value":417.3082574231253},{"configuration":"CPU + CephFS","metric":"Read","value":1046.0784891376734},{"configuration":"CPU + CephFS","metric":"Write","value":414.1295297532558}]},"mark":{"type":"bar","tooltip":true},"encoding":{"x":{"field":"metric","type":"nominal","title":"Metric","sort":["Read","Write"]},"xOffset":{"field":"configuration","sort":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"]},"y":{"field":"value","type":"quantitative","title":"MiB/s","scale":{"zero":true}},"color":{"field":"configuration","type":"nominal","title":"Configuration","scale":{"domain":["No offload","CPU offload 256 GiB","CPU + NVMe","CPU + CephFS"],"range":["#1f77b4","#ff7f0e","#2ca02c","#d62728"]}},"tooltip":[{"field":"configuration","type":"nominal"},{"field":"metric","type":"nominal"},{"field":"value","type":"quantitative","format":",.3f"}]},"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white"}

```

NVMe averages 693.6 MiB/s reads and 417.3 MiB/s writes, peaking at 1,536.6 and 1,063.3 MiB/s. It averages 694 read IOPS, 568 write IOPS, and 20.0% busy time on `nvme0n1`; `nvme1n1` is idle. The device is active but does not appear saturated.

CephFS pool telemetry covers the full captured window and averages 1,046.1 MiB/s reads and 414.1 MiB/s writes, peaking at 2,511.5 and 1,007.9 MiB/s. It averages 643 read IOPS and 611 write IOPS. The fresh PVC grows to 989.2 GiB, 32.20% of 3 TiB. Application logs contain zero `cannot store blocks` warnings, unavailable-block storms, CUDA OOMs, or tracebacks.

Ceph health requires a separate caveat. The final cluster condition is `HEALTH_WARN` because MONs `l` and `o` are low on space and two daemons recently crashed; an MDS liveness event also reports an MDS ID temporarily absent from the map. Pool/PVC/health telemetry is present, but client, OSD-latency, and MDS-performance series are absent from this artifact. The two AIPerf server disconnects cannot be proven to originate in CephFS, so the report records correlation rather than causation.

Native-cadence figures:

- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/01 - Prompt-token sources - No offload|Prompt-token sources — no offload]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/02 - Prompt-token sources - CPU offload|Prompt-token sources — CPU offload]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/03 - Prompt-token sources - NVMe|Prompt-token sources — NVMe]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/04 - Prompt-token sources - CephFS|Prompt-token sources — CephFS]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/05 - GPU KV-cache utilization|GPU KV-cache utilization]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/06 - Scheduler pressure - Baseline|Scheduler pressure — baseline]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/07 - Scheduler pressure - CPU offload|Scheduler pressure — CPU offload]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/07b - Scheduler pressure - Secondary tiers|Scheduler pressure — secondary tiers]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/08 - CPU KV transfer bandwidth|CPU KV transfer bandwidth]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/09 - Secondary-tier KV transfer bandwidth|Secondary-tier KV transfer bandwidth]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/10 - Generation throughput|Generation throughput]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/11 - TTFT P90|TTFT P90]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/12 - NVMe throughput|NVMe throughput]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/13 - NVMe IOPS|NVMe IOPS]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/14 - NVMe busy time|NVMe busy time]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/15 - CephFS throughput|CephFS throughput]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/16 - CephFS IOPS|CephFS IOPS]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/17 - CephFS capacity|CephFS capacity]]
- [[2026-07-31 - FP8 Nemotron 253B KV-cache offload comparison - Revision 2/18 - CephFS health|CephFS health]]

## Validity and limitations

- All four cells use the same model identifier, image digest, TP8, U0.80, context, concurrency, seed, workload, and shared-memory size.
- The no-offload, CPU-only, and NVMe profiles contain zero request errors. CephFS has 1,399 successful requests plus two `ServerDisconnectedError` failures.
- Model logs contain zero CUDA OOMs, tracebacks, `cannot store blocks` warnings, and unavailable-block storms.
- All cells reach the fixed send cutoff and the 30-second grace timeout. Cancellations range from 18 to 67; faster cells have more work in flight at the boundary.
- AgentX is stateful and branch-generating, so completed request mixes diverge. Mean input length ranges from 54.2K to 64.0K tokens. No single request-count metric should stand alone.
- This is one seed, not a replicated estimate. The very large effect and matching mechanism telemetry support a directional claim, not a confidence interval.
- Revision 1 and Revision 2 change two independent variables at once. They cannot isolate the causal contribution of U versus concurrency.
- NVMe records `cleanup: false`; explicit clean-state proof is absent. The rerun should use a fresh path.
- The model repository commit is not pinned.

## Conclusions

### Established by the data

- U0.80/C32 successfully forces HBM eviction and external restoration at TP8.
- Offload avoids severe prompt recomputation and materially improves throughput and latency.
- A 256 GiB CPU tier helps substantially, but NVMe and CephFS retain deeper history and reduce compute further.
- NVMe and CephFS are at performance parity in this single seed.
- The current U is slightly aggressive for a clean operating comparison because every cell touches 100% KV occupancy and preemptions are non-zero.

### Decision

Use **TP8, concurrency 32, and U0.82** for the next clean four-cell comparison. Keep U0.80 as the explicit forced-spill stress point. Do not lower U at C32.

## Next experiment

1. Run no-offload and CPU256 at U0.82/C32 to confirm 90–98% peak occupancy and sustained external share.
2. If peak occupancy falls below 90%, use U0.81; if it remains above 98% or preemptions persist, use U0.83–0.84.
3. Lock the selected U and run all four configurations at seeds 42, 123, and 456.
4. Require zero request errors, zero model/router restarts, a fresh NVMe path, a fresh CephFS PVC, and healthy Ceph MON/MDS state.
5. Preserve prompt-source, KV utilization, scheduler, direct KV-transfer, NVMe, CephFS pool/PVC, and Ceph health telemetry.

## Run registry and provenance


| Configuration       | Execution / deployment                                         | Node                             | MLflow                                                                                                                                                                  | Disposition                                                 |
| ------------------- | -------------------------------------------------------------- | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| No offload          | `cpu-offloading-383d35` / `cpu-kv-offload-distributed-default` | `diadochos-hqxzk-gpu-h100-fx7c8` | [5f9a464d28e84284943c8454f0bd5db3](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/5f9a464d28e84284943c8454f0bd5db3?workspace=benchflow) | Directionally accepted                                      |
| CPU offload 256 GiB | `cpu-offloading-fc7d4e` / `rhoai-cpu-kv-offload-256g`          | `diadochos-hqxzk-gpu-h100-6kl5z` | [17704f73746a417eba8cc6677570b5fc](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/17704f73746a417eba8cc6677570b5fc?workspace=benchflow) | Directionally accepted                                      |
| CPU + NVMe          | `cpu-offloading-7fc331` / `multi-tier-offloading-nvme`         | `diadochos-hqxzk-gpu-h100-mt46x` | [7d85703182174927aca7dcf490b9a7b8](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/7d85703182174927aca7dcf490b9a7b8?workspace=benchflow) | Directionally accepted                                      |
| CPU + CephFS        | `cpu-offloading-960d4c` / `multi-tier-offloading-cephfs`       | `diadochos-hqxzk-gpu-h100-6kl5z` | [77246ae40d8b4a27a73dbdeb92cfa0a1](https://mlflow.apps.psap-automation.ibm.rhperfscale.org/#/experiments/328/runs/77246ae40d8b4a27a73dbdeb92cfa0a1?workspace=benchflow) | Conditionally accepted (2 request errors; Ceph HEALTH_WARN) |


Artifacts were downloaded on 2026-07-31. Results use `results/profile_export_aiperf.json`, `benchmark/profile_export.jsonl`, complete AIPerf/model logs, rendered manifests, Kubernetes state, Ceph diagnostics, and native Prometheus JSON. Prompt-source shares integrate uniform-cadence rate samples and normalize across sources. Companion figures preserve the native 15-second source cadence without smoothing or downsampling.