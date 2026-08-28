# 2026-08-28 — A30 native tiering campaign (initial run)

## Purpose

This is the first controlled campaign on the supplied A30 deployment after the COSTAR research reset. It establishes a clean native-vLLM reference before testing any retention or placement policy. The run is intentionally workload-local and does not claim a COSTAR treatment effect.

## Deployment snapshot

- Host: `bb37-h13-000-r750.rdu3.labs.perfscale.redhat.com`
- Namespace/service: `vllm/qwen3-27b-awq-vllm:8000`
- GPU: one NVIDIA A30 (visible inside the pod; host-level NVML is not exposed)
- Image: `vllm/vllm-openai:v0.27.0` (stock)
- Model: `QuantTrio/Qwen3.6-27B-AWQ`
- max model length: 32,768; tensor parallel: 1; prefix caching enabled
- CPU offload: 32 GiB model offload; KV CPU tier: 4 GiB
- Secondary tier: filesystem on a 1 TiB NVMe PVC at `/cache/kv`
- Secondary I/O threads: 32 read, 16 write
- KV connector: native `OffloadingConnector`, reactive tiering only

The exact deployment and pod YAML are saved with the artifacts under `/home/crcuser/costar-overnight-20260828-native/`.

## Workload

A reproducible Python client sends four phases through a temporary service port-forward:

1. eight unique contexts (warm-up);
2. reuse of those contexts;
3. sixteen additional unique contexts at concurrency eight (pressure);
4. reuse after pressure.

Each request disables Qwen thinking and generates one token. The pressure contexts are long enough to exercise the finite 4 GiB CPU tier. Request records, before/after Prometheus snapshots, and a 15-second metric time series are persisted on the host.

## Results

The campaign completed at `2026-08-28T22:10:49Z`. Out of 40 client requests, 15 returned HTTP 200 before the 300-second client timeout; 25 timed out. Successful request latencies ranged from 116–296 seconds (phase means: warm-up 192.1 s, reuse 192.0 s, pressure 192.0 s, reuse-after-pressure 249.4 s). The timeouts are client-side observations, not proof that vLLM abandoned the requests.

Final KV metrics:

| Metric | Value |
|---|---:|
| GPU→CPU store operations | 120 |
| GPU→CPU store bytes | 26.64 GB |
| deferred lookup events | 40 |
| aggregate asynchronous lookup delay | 5,188.46 s |
| aggregate synchronous lookup delay | 0.0149 s |
| final CPU cache usage | 22.9% |
| final in-flight write/read usage | 12.0% / 10.8% |

The key observation is the separation between metadata and movement: synchronous lookup time was only 14.9 ms aggregate, while deferred asynchronous delay accumulated to 5,188 s. The native tier path is therefore dominated by queued/serialized KV movement and request scheduling under this pressure, not by existence probes. The workload is intentionally harsher than a normal short-request benchmark and should be treated as a stress baseline.

## Interpretation and limits

This is strong evidence that the deployment can enter a severe tier-movement bottleneck, but it is not yet evidence that a retention policy helps. The long-context pressure workload produced too many timeouts to support a fair throughput comparison, and the stock image has no COSTAR treatment. We should not infer that reuse failed or that proactive retention is ineffective from these results.

The run does establish a useful failure envelope: with a 4 GiB CPU tier and concurrency eight, native reactive tiering can accumulate multi-thousand-second aggregate deferred delay even while synchronous lookup remains negligible.

## Reproducibility artifacts

```
/home/crcuser/costar-overnight-20260828-native/deployment.yaml
/home/crcuser/costar-overnight-20260828-native/pod.yaml
/home/crcuser/costar-overnight-20260828-native/metrics-before.prom
/home/crcuser/costar-overnight-20260828-native/metrics-timeseries.prom
/home/crcuser/costar-overnight-20260828-native/metrics-after.prom
/home/crcuser/costar-overnight-20260828-native/metrics-final.prom
/home/crcuser/costar-overnight-20260828-native/requests.json
/home/crcuser/costar-overnight-20260828-native/vllm-logs-final.txt
```

## Next experiment

Run a smaller calibrated baseline (512–1,024 prompt tokens, concurrency 2–4, no more than 8 requests per phase) so all requests complete. Then compare the same workload with one treatment variable—preferably retention/placement or the COSTAR image—while keeping model, CPU capacity, tier, and request order identical. Use completed-request rate, TTFT/total latency, deferred lookup delay, secondary read bytes/time, CPU occupancy, and timeout count as gates.