# 2026-08-29 — A30 exact-prefix reuse experiment

## Aim

Test whether the deployment/workload exposes strong locality that a future-value-aware retention policy could exploit. This isolates reuse from speculative prefetch: no COSTAR treatment was enabled.

## Method

Stock vLLM 0.27.0, Qwen3.6-27B-AWQ, one A30, 4 GiB CPU KV tier, and 1 TiB NVMe-backed filesystem secondary tier. Six sequential direct-`curl` requests used approximately 1,943-token prompts:

- two unique contexts;
- four requests repeating the same exact context.

Each request generated one token with Qwen thinking disabled.

## Results

| Request | Context | Wall latency |
|---|---|---:|
| unique-1 | new | 32.18 s |
| unique-2 | new | 32.18 s |
| repeat-1 | first reuse | 54.25 s |
| repeat-2 | exact repeat | 2.31 s |
| repeat-3 | exact repeat | 2.31 s |
| repeat-4 | exact repeat | 2.31 s |

All six requests returned HTTP 200. The first repeated request was slower because the reusable KV was not yet immediately GPU-ready; subsequent exact repeats were ~23.5× faster than the unique requests, consistent with prefix-cache reuse.

## Interpretation

This is strong evidence that the A30/vLLM workload contains a valuable reusable prefix and that retention/readiness can materially affect end-to-end latency. It does not prove a placement-policy gain: the experiment used native prefix caching and did not compare a retention treatment or a cache-pressure eviction condition. The first-repeat penalty is precisely the opportunity to test next: preserve the reusable CPU copy across a controlled pressure interval, then compare its restoration latency with native LRU.

## Next experiment

Run a paired pressure test with the same exact prefix:

1. seed the prefix;
2. issue enough unique contexts to approach the 4 GiB CPU tier;
3. request the prefix again;
4. compare native LRU against a retention-aware treatment.

Keep the request sequence and prompt bytes identical, capture before/after metric deltas, and report request-ready/secondary-read latency rather than block-hit counts alone.

Artifacts:

```
/home/crcuser/costar-overnight-20260829-reuse/requests.json
/home/crcuser/costar-overnight-20260829-reuse/run.log
/home/crcuser/costar-overnight-20260829-reuse/metrics-final.prom
/home/crcuser/costar-overnight-20260829-reuse/port-forward.log
```