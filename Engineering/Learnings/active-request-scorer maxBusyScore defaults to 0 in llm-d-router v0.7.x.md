---
title: active-request-scorer maxBusyScore defaults to 0 in llm-d-router v0.7.x
date: 2026-10-06
type: learning
topic: llm-d-router EPP scheduling
components: [llm-d-router, llm-d-inference-scheduler, EPP, active-request-scorer]
affected_router_versions: v0.7.0 - v0.8.x
fixed_in_router_version: v0.9.0
affected_llm_d_release: v0.6.0
affected_rhoai_release: "3.4"
verification: upstream source read at tags; not reproduced on a cluster
---

# active-request-scorer maxBusyScore defaults to 0 in llm-d-router v0.7.x

## Symptom

With `active-request-scorer` configured without parameters on llm-d-router v0.7.0 to v0.8.x, load is balanced only while some pod is idle. Once every pod has at least one in-flight request, per-pod load becomes uneven and tail latency grows.

At EPP startup the affected versions log:

```
Active request scorer configured with idle preference  idleThreshold=0 maxBusyScore=0
```

That line appearing with `maxBusyScore` 0 when no parameters were set is the direct indicator.

## Rule

On RHOAI 3.4 (llm-d v0.6.0, router v0.7.1), always set `maxBusyScore` explicitly:

```yaml
plugins:
  - type: active-request-scorer
    parameters:
      maxBusyScore: 1.0
      requestTimeout: 5m
schedulingProfiles:
  - name: default
    plugins:
      - pluginRef: active-request-scorer
        weight: 1
```

Keep the `apiVersion` the 3.4 default EPP config already uses. `llm-d.ai/v1alpha1` exists only from router v0.9.0.

## What maxBusyScore is

The scorer gives each pod a score in $[0, 1]$ from its in-flight request count. Pods at or below `idleThreshold` score 1.0. Busy pods score

$$
\text{score} = \frac{\text{maxCount} - \text{count}}{\text{maxCount}} \times \text{maxBusyScore}
$$

`maxBusyScore` optionally caps busy pods below idle ones (0.5 means the least busy pod scores at most 0.5). The intended default is 1.0, meaning no cap.

## Cause

At tag v0.7.1, `pkg/plugins/scorer/active_request.go`:

```go
MaxBusyScore float64 `json:"maxBusyScore"`          // line 50, zero when omitted

maxBusyScore := 1.0                                  // line 121
if params != nil && params.MaxBusyScore >= 0 && params.MaxBusyScore <= 1.0 {
    maxBusyScore = params.MaxBusyScore               // 0.0 passes the range check
}

scoredEndpointsMap[endpoint] = float64(maxCount-count) / float64(maxCount) * s.maxBusyScore  // line 225
```

The factory always passes a non-nil params struct, so an omitted field arrives as 0.0 and overwrites the default. Every busy pod then scores 0.

| Pod | In-flight requests | Intended score | Score on v0.7.1 with defaults |
|---|---|---|---|
| A | 2 | 0.75 | 0 |
| B | 5 | 0.38 | 0 |
| C | 8 | 0.00 | 0 |

Fixed by commit `2de860ec` (PR #919, "treat unset maxBusyScore as default 1.0 in active-request-scorer"), which made the field a pointer. First shipped in router v0.9.0. The field does not exist in v0.6.1 and earlier.

Setting `maxBusyScore: 1.0` is safe on all versions before v0.9.0: parameters were parsed non-strictly, so the key is ignored where the field does not exist.

## Version mapping

From the llm-d umbrella release notes (`gh release view <tag> --repo llm-d/llm-d`):

| RHOAI | llm-d release | Router version | Component name |
|---|---|---|---|
| 3.4 | v0.6.0 | v0.7.1 | `llm-d-inference-scheduler` |
| 3.5 | v0.8.0 | v0.9.0 | `llm-d-router-endpoint-picker` |

The umbrella number is not the router tag. The RHOAI-to-llm-d pairing was stated by a colleague, not verified against a downstream build.

## Other differences in the scorer before router v0.9.0

The scorer was rewritten in v0.9.0 (commit `df9da71c`, "Migrate active request scorer to inflight load") to read counts from `inflight-load-producer`. Before that:

- **Own TTL cache, 2 minute default.** A request still running after `requestTimeout` is dropped from the count, so the pod looks lighter than it is. Matters when E2E latency exceeds 2 minutes. `requestTimeout` is deprecated and ignored from v0.9.0.
- **Counts are per EPP process.** Each EPP replica sees only the requests it routed. Cross-replica sync arrived with `inflight-load-producer`.
- **`inflight-load-producer` does not exist.** A 3.5 config that declares it will not load on 3.4.

## Context

Raised on 2026-10-06 while reviewing a proposal to use `active-request-scorer` on RHOAI 3.4 as a two-week placeholder until `token-load-scorer` is available in 3.5. The workload was long-prompt summarization: 256-token shared prefix, about 25,000 prompt tokens (15,000 to 30,000), 1,500 output tokens, 16 replicas of `RedHatAI/gemma-4-31B-it-FP8-dynamic`, GuideLLM 0.8.0, concurrency 1 to 512.

The default 3.4 profile (`queue-scorer` 2 + `prefix-cache-scorer` 3) concentrated traffic on a few replicas because the prefix scorer followed a cache hit of about 1.1% of the prompt.

Measured on the RHOAI 3.5 scheduler, single run per profile, completed-request E2E P99 against a 60 s SLA:

| Streams | Active-request | Optimized 3.5 (affinity filter + token-load) | 3.5 defaults | 3.4 defaults (translated) |
|---|---|---|---|---|
| 16 | 22.1 s | 33.7 s | 54.6 s | 60.4 s |
| 64 | 33.5 s | 47.6 s | 109.7 s | 94.7 s |
| 128 | 47.5 s | 80.4 s | 76.0 s | 121.2 s |
| 256 | 74.6 s | 108.0 s | 90.9 s | 125.6 s |

Per-replica attempt spread for active-request was 50 to 52 at 32 streams and 80 to 84 at 64 streams. A 3.4 run that shows a much wider spread at those levels points to this issue.

Limits of that data: all four profiles ran on router v0.9.0 or later, so none exercised the v0.7.1 code path. Each profile had one run with its own seed, and concurrency levels ran in sequence without a cache reset. TTFT for active-request was worse than the token-load profile at mid concurrency (64 streams: 4.9 s P50 against 2.2 s).

## Open items

- Confirm on a 3.4 cluster with the startup log line and the attempt-spread check; this note is from source reading only.
- Whether the RHOAI 3.4 downstream build carries a backport of PR #919.
- Whether the v0.7.1 picker breaks ties randomly. The picker lived in gateway-api-inference-extension at that version and was not read.
- Whether v0.7.1 decrements the count when a client disconnects mid-request.

## Sources

- llm-d-router repository, tags v0.7.0, v0.7.1, v0.8.0, v0.9.0.
- Commits `2de860ec` (PR #919) and `df9da71c` (PR #931).
- llm-d umbrella release notes for v0.6.0 and v0.8.0.
- "Gemma Summarization - EPP Scheduling Report" (GuideLLM summarization latency report shared by a colleague, 2026-10).