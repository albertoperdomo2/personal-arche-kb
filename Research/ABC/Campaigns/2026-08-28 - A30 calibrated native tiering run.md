# 2026-08-28 — A30 calibrated native tiering run

## Aim

After the long-context stress campaign produced client timeouts, this run tests whether a smaller request shape gives a complete, interpretable native-vLLM baseline.

## Configuration

Same deployment as the stress run: stock `vllm/vllm-openai:v0.27.0`, Qwen3.6-27B-AWQ, one NVIDIA A30, 4 GiB CPU KV tier, 1 TiB NVMe-backed filesystem secondary tier, native reactive `OffloadingConnector`. The client used approximately 512–1,024-token prompts, one output token, thinking disabled, concurrency two, and four requests per phase:

- four unique warm-up requests;
- reuse of the same four contexts;
- four additional unique requests;
- reuse after pressure.

## Result

All 16 requests completed successfully. Phase means were approximately:

| Phase | Completed | Mean latency |
|---|---:|---:|
| warm | 4/4 | 17.4 s |
| reuse | 4/4 | 18.5 s |
| pressure | 4/4 | 18.5 s |
| reuse-after | 4/4 | 18.5 s |

The Prometheus snapshots show no increase in cumulative secondary-load or store counters during this run; the prior stress campaign's counters remained unchanged. Thus this calibrated shape did not create external-tier movement and cannot test retention or placement value. It is useful as a completed-request control and a sanity check for the client harness.

## Conclusion

The smaller workload is suitable for validating request accounting and end-to-end client behavior, but it is below the threshold needed to exercise the 4 GiB CPU tier. The long-context stress workload does exercise tiering but is too slow for a fair A/B comparison. The next experiment should sweep prompt size at low concurrency until it produces a small, repeatable number of secondary reads while retaining near-100% completion. Then run the same request sequence against a single treatment policy.

Artifacts:

```
/home/crcuser/costar-overnight-20260828-calibrated/requests.json
/home/crcuser/costar-overnight-20260828-calibrated/metrics-before.prom
/home/crcuser/costar-overnight-20260828-calibrated/metrics-after.prom
/home/crcuser/costar-overnight-20260828-calibrated/metrics-final.prom
/home/crcuser/costar-overnight-20260828-calibrated/vllm-logs-final.txt
```