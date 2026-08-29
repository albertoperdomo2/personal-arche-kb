# 2026-08-29 — A30 prompt-size threshold sweep

## Aim

Find a reproducible request size that exercises native tiering without the timeout collapse of the earlier concurrency-eight stress run. This is a calibration experiment, not a treatment comparison.

## Method

After restarting the vLLM pod, send one direct `curl` request for each unique prompt size through an isolated port-forward. Each request uses one output token, thinking disabled, and the stock deployment:

- Qwen3.6-27B-AWQ, one A30, vLLM 0.27.0
- 4 GiB CPU KV tier and 1 TiB NVMe-backed filesystem secondary tier
- native reactive `OffloadingConnector`

Prompt sizes were approximately 983, 1,943, 3,863, and 5,783 tokens. Direct subprocess `curl` was used after two Python/urllib clients stalled despite an idle server.

## Results

All four requests returned HTTP 200:

| Prompt tokens | Wall latency |
|---:|---:|
| 983 | 16.0 s |
| 1,943 | 32.2 s |
| 3,863 | 62.2 s |
| 5,783 | 93.9 s |

The run produced approximately 5.24 GB cumulative GPU→CPU stores and 10 lookup events in the post-restart metric snapshot. Aggregate asynchronous lookup delay was approximately 57.6 ms and synchronous lookup delay 2.5 ms. These counters are cumulative for the post-restart server and should be interpreted together with the request log; they do not by themselves prove a secondary read for every size.

## Conclusion

This establishes a usable calibration regime: sequential requests up to roughly 5.8k prompt tokens complete reliably within 94 seconds, unlike the earlier high-concurrency stress run. The 3.9k–5.8k range is a good candidate for the next controlled placement/retention comparison. To isolate policy value, repeat the same direct-curl sequence twice with identical prompts and order, collecting before/after metric deltas and request-level latencies. Keep concurrency at one or two initially; add concurrency only after a complete baseline exists.

Artifacts:

```
/home/crcuser/costar-overnight-20260829-curl-sweep/requests.json
/home/crcuser/costar-overnight-20260829-curl-sweep/run.log
/home/crcuser/costar-overnight-20260829-curl-sweep/metrics-final.prom
/home/crcuser/costar-overnight-20260829-curl-sweep/port-forward.log
```