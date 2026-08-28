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

Each request disables Qwen thinking and generates one token, so the measured cost is dominated by prompt processing, scheduling, and KV movement. The pressure contexts are long enough to exercise the finite 4 GiB CPU tier. Request records, before/after Prometheus snapshots, and a 15-second metric time series are persisted on the host.

## Observed progress

The first calibration request (292 prompt tokens) completed in approximately 4.5 seconds. During the campaign, vLLM logs reported deferred requests and active CPU read/write pinning. The KV metrics reported approximately 410 MiB of GPU→CPU stores early in the run; a later snapshot reported 16 store operations totalling approximately 3.54 GiB. This confirms that the workload is exercising native offload and queue pressure rather than merely hitting GPU-resident prefixes.

The campaign was still running at the time of this note; final request-level summaries and post-run metrics will be appended after completion. No treatment or proactive prefetch claim is made here.

## Reproducibility artifacts

```
/home/crcuser/costar-overnight-20260828-native/deployment.yaml
/home/crcuser/costar-overnight-20260828-native/pod.yaml
/home/crcuser/costar-overnight-20260828-native/metrics-before.prom
/home/crcuser/costar-overnight-20260828-native/metrics-timeseries.prom
/home/crcuser/costar-overnight-20260828-native/metrics-after.prom   (written at completion)
/home/crcuser/costar-overnight-20260828-native/requests.json         (written at completion)
/home/crcuser/costar-overnight-20260828-native/run.log
```

## Next decision

Use the completed run to determine whether the pressure/reuse phases create measurable secondary reads and exposed deferred time. If they do, the next controlled comparison should keep this workload and compare a native reactive baseline against a retention/placement treatment, changing one policy variable at a time. If they do not, increase pressure only enough to produce repeatable secondary reads; do not jump directly to a large overnight run.