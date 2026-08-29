# 2026-08-29 — A30 native pressure/reuse control

## Aim

Measure whether a reusable prefix remains fast after competing KV contexts are admitted into the finite CPU tier. This is the native reactive-LRU baseline for a future retention-aware treatment.

## Method

Stock vLLM 0.27.0, Qwen3.6-27B-AWQ, one A30, 4 GiB CPU KV tier, and 1 TiB NVMe-backed filesystem secondary tier. Direct `curl` requests used the same approximately 1,943-token prefix:

1. seed reusable prefix;
2. eight distinct competing contexts;
3. request the original prefix again.

One output token was generated with thinking disabled. All requests returned HTTP 200.

## Results

| Request | Wall latency |
|---|---:|
| seed prefix | 45.56 s |
| unique 0–7 | 58.66–58.68 s each |
| reuse after pressure | 8.02 s |

The final server snapshot reported 10.99 GB cumulative GPU→CPU stores, 22 lookup events, and 0.101 s aggregate asynchronous lookup delay. These counters are cumulative for the current pod and should be interpreted with the request sequence, not as per-request attribution.

## Conclusion

Native tiering preserved enough of the reusable prefix to make the post-pressure request substantially faster than a cold unique request (8.0 s versus 58.7 s), but it was still slower than an immediate exact repeat observed earlier (2.31 s). This is evidence of a remaining readiness/placement gap, not proof that native LRU failed completely. The next decisive experiment is a paired run with identical request bytes and order under a retention-aware policy that protects the seeded continuation softly, then compares post-pressure reuse latency, secondary-read bytes/time, and impact on competing requests.

Artifacts:

```
/home/crcuser/costar-overnight-20260829-pressure-reuse/requests.json
/home/crcuser/costar-overnight-20260829-pressure-reuse/run.log
/home/crcuser/costar-overnight-20260829-pressure-reuse/metrics-final.prom
/home/crcuser/costar-overnight-20260829-pressure-reuse/port-forward.log
```