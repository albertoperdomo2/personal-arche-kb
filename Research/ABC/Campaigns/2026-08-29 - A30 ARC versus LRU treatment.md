# 2026-08-29 — A30 ARC versus LRU treatment

## Aim

Test whether a standard adaptive replacement policy closes the native pressure/reuse readiness gap without introducing a custom COSTAR image. Only `eviction_policy` changed from `lru` to `arc`; model, tier capacities, I/O threads, request order, and prompts were unchanged.

## Method

The ARC pod was restarted and the exact native control sequence was replayed:

1. seed reusable ~1,943-token prefix;
2. eight unique competing contexts;
3. request the seeded prefix again.

All requests used one output token and disabled Qwen thinking.

## Results

| Request | LRU control | ARC treatment |
|---|---:|---:|
| seed prefix | 45.56 s | 45.94 s |
| unique contexts | 58.66–58.68 s | 58.81–58.89 s |
| reuse after pressure | 8.02 s | 8.03 s |

All requests returned HTTP 200. ARC produced no measurable latency improvement and matched LRU within measurement noise. The treatment logs show normal reactive tiering and no correctness errors.

## Conclusion

ARC is a negative baseline for this workload: standard recency/frequency adaptation does not recover the remaining readiness gap. This is consistent with the COSTAR offline evidence that generic history-based replacement policies leave valuable future continuations unidentified. The next treatment should use explicit continuation/session value (soft priority or TTL), not another generic cache policy.

Artifacts:

```
/home/crcuser/costar-overnight-20260829-arc/requests.json
/home/crcuser/costar-overnight-20260829-arc/run.log
/home/crcuser/costar-overnight-20260829-arc/metrics-final.prom
/home/crcuser/costar-overnight-20260829-arc/vllm-logs-final.txt
```