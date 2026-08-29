# 2026-08-29 — A30 COSTAR campaign summary

## Scope

Initial controlled experiments on the supplied A30 deployment, using stock vLLM 0.27.0 with native reactive tiering. No COSTAR treatment image was deployed in this campaign; the purpose was to establish a valid pressure regime and verify whether reusable continuation state exists.

## Deployment

Qwen3.6-27B-AWQ, one NVIDIA A30, 4 GiB CPU KV tier, 1 TiB NVMe-backed filesystem secondary tier, native `OffloadingConnector`, prefix caching enabled. The exact deployment YAML is preserved with the stress artifacts.

## Evidence

| Experiment | Design | Result | Interpretation |
|---|---|---|---|
| Long-context stress | 40 requests, concurrency up to 8, long contexts | 15 completed; 25 hit 300 s client timeout; 5,188 s aggregate deferred delay; 26.64 GB stores | Valid stress envelope, not a fair A/B; movement/scheduling dominates metadata lookup |
| Calibrated baseline | 16 requests, short contexts, concurrency 2 | 16/16 completed in 17–19 s; no new secondary movement | Harness sanity check; insufficient pressure for placement conclusions |
| Prompt-size threshold | Sequential direct-`curl`, ~983–5,783 prompt tokens | 16.0, 32.2, 62.2, 93.9 s; all HTTP 200 | Usable test range is ~2–4k tokens for complete runs; larger contexts become slow |
| Exact-prefix reuse | Two unique contexts then four exact repeats | New: 32.18 s; first repeat: 54.25 s; later repeats: 2.31 s | Strong locality signal; ready reusable prefixes are dramatically cheaper |
| Native pressure/reuse control | Seed prefix, eight unique competitors, request prefix again | Seed: 45.56 s; unique: 58.66–58.68 s; reuse after pressure: 8.02 s | Native LRU preserves some value but leaves a measurable readiness/placement gap |

The stress run also showed the key systems split: ~5,188 s aggregate asynchronous lookup delay versus ~15 ms synchronous lookup delay. The bottleneck is queued/serialized movement and residency, not key-existence probing.

## Current conclusion

The supplied deployment is suitable for COSTAR experiments, and the workload exposes genuine continuation locality. A retention/placement policy has a plausible target: reduce the 8.02 s post-pressure restoration toward the 2.31 s immediate-repeat baseline without harming competing requests. The native baseline does not establish that a new policy will help; it defines the comparison.

## Next experiment

Deploy one retention-aware treatment (soft priority/TTL for the seeded continuation, no hard reserve and no blanket proactive load) and replay the exact pressure sequence. Compare:

- post-pressure reuse latency and TTFT;
- secondary-read bytes/time and deferred lookup delay;
- completion rate and timeout count;
- CPU occupancy and evictions of competing contexts;
- fraction of immediate-repeat latency recovered.

Do not sweep policy parameters until this paired baseline/treatment result is complete. The custom treatment image must be verified for this Qwen deployment before rollout.