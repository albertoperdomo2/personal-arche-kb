---
title: Diadochos router threshold live validation - 2026-09-08
date: 2026-09-08
type: validation-report
topic: Selective KV loading
experiment: Router static-threshold live validation
project: Selective KV Loading and Offloading
status: conditionally-valid
model: nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
model_revision: unknown
vllm_version: v0.27.0-based selective-load build
vllm_image: quay.io/rh-ee-aperdomo/vllm:v0.27.0-selective-load-v2
router_image: quay.io/rh-ee-aperdomo/llm-d-router-endpoint-picker:dev-a808637f
router_source_commit_from_tag: a808637f963e43ba9a6f1e0ebaed3182e918ca6c
tensor_parallelism: 8
replicas: 1
gpu: 8x H100
gpu_memory_utilization: 0.8
max_model_len: 131072
max_num_seqs: default
cpu_bytes: 274877906944
offload_spec: CPU primary tier via OffloadingConnector
secondary_tier: none
secondary_tier_threads: not-applicable
dev_shm: 300Gi
workload: Direct prime, HBM churn, and 512/2048-token probes
random_seed: not-recorded
duration_seconds: 34.636
cache_cleaning_state: Targets primed after EPP readiness, then evicted from a temporary 131072-token HBM cache with two unique approximately 70000-token prompts
configuration:
  load_policy: threshold
  min_external_reusable_tokens: 1024
  temporary_gpu_blocks: 8192
  temporary_gpu_cache_tokens: 131072
  restored_gpu_cache_tokens: 1824960
  kv_event_topic: kv@$(POD_IP)@nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8
  self_describing_kv_events: true
---

# Diadochos router threshold live validation - 2026-09-08

## Executive summary

This live Diadochos validation tested the first llm-d-router image that makes a per-request selective-loading decision from precise cache evidence and a static threshold. The deployed EPP image was `quay.io/rh-ee-aperdomo/llm-d-router-endpoint-picker:dev-a808637f` with `loadPolicy: threshold` and `minExternalReusableTokens: 1024`. The model server used the v0.27.0-based selective-load vLLM build, one TP=8 Nemotron 253B replica on eight H100 GPUs, a 256 GiB CPU offload tier, and no secondary storage tier.

After correcting the EPP-to-model TLS name and the KV-event identity contract, the final reduced-cache run exercised both branches deterministically. The 512-token target was below threshold and recomputed all 518 prompt tokens with zero external tokens. The 2,048-token target was above threshold and restored 2,032 tokens externally, retained 16 local HBM tokens, computed seven tokens, and transferred 266,338,304 bytes from CPU to GPU. This validates the router decision and vLLM enforcement path for the two boundary cases.

The checkpoint does not validate the 1,024-token threshold as a performance optimum. It contains one functional sample per branch, uses a temporary small GPU cache to force eviction, and has no request replay or event-index warmup. Earlier full-cache attempts were invalid or inconclusive and are preserved below.

## Validity verdict: Conditionally valid

The final reduced-cache result is valid as a deterministic functional validation of the static router gate on this single CPU-offload deployment: below-threshold evidence caused recomputation, while above-threshold evidence preserved restoration. It is not valid for comparing performance, recalibrating the threshold, generalizing to storage or P2P tiers, or claiming production readiness. The two earlier full-cache attempts are invalid or inconclusive because the evidence pipeline was initially misconfigured and later did not deterministically create the required external-only cache state.

## Main takeaways

- Measured: with a 1,024 external-token threshold, the 512-token probe produced 518 local-compute tokens and zero external-transfer tokens.
- Measured: the 2,048-token probe produced 2,032 external-transfer tokens, 16 local-cache-hit tokens, seven local-compute tokens, and 266,338,304 load bytes.
- Measured: two approximately 70,000-token unique churn prompts displaced the primed targets from the temporary 131,072-token HBM cache in 17.499 seconds, making the two policy branches reproducible.
- Resolved: the token producer's short service hostname failed TLS verification; using the service FQDN `llm-d-selective-loading-28d3b96d7e-kserve-workload-svc.benchflow.svc` matched the certificate and restored tokenization.
- Resolved: precise cache evidence required endpoint- and model-qualified KV-event topics using `POD_IP` plus `self_describing_kv_events=true`.
- Limitation: the successful run proves functional routing and enforcement only; the 512 and 2,048 wall times are single observations under different prompt and cache paths and must not be ranked.
- Operational: the model deployment was restored to its original 1,824,960-token GPU KV cache, and temporary debug and authorization overrides were removed after validation.

## Headline results

The source counts and transfer bytes are request-scoped metric deltas captured immediately around each final probe. An omitted zero-valued delta in the raw JSON is reported as zero only where the prompt-token source counters directly establish no external transfer.

| Requested target | Total prompt tokens | Threshold branch | External KV tokens | Local HBM tokens | Local compute tokens | CPU-to-GPU load bytes | Probe wall time (ms) | Verdict |
|---:|---:|---|---:|---:|---:|---:|---:|---|
| 512 | 518 | Recompute: below 1,024 | 0 | 0 | 518 | 0 | 528.441 | Valid functional branch |
| 2,048 | 2,055 | Restore: at or above 1,024 | 2,032 | 16 | 7 | 266,338,304 | 474.435 | Valid functional branch |

Figure 1 plots every prompt-token source value from the two final probes at the finest available request-level categorical grain. The data comes from `.calibration-artifacts/2026-09-08-router-threshold-validation-reduced-cache.json`.

```vega-lite
{"$schema":"https://vega.github.io/schema/vega-lite/v5.json","background":"white","title":"Figure 1. Prompt-token source by static-threshold branch","width":560,"height":300,"data":{"values":[{"probe":"512: recompute","source":"Local compute","tokens":518},{"probe":"512: recompute","source":"Local HBM hit","tokens":0},{"probe":"512: recompute","source":"External KV transfer","tokens":0},{"probe":"2048: restore","source":"Local compute","tokens":7},{"probe":"2048: restore","source":"Local HBM hit","tokens":16},{"probe":"2048: restore","source":"External KV transfer","tokens":2032}]},"mark":{"type":"bar"},"encoding":{"x":{"field":"probe","type":"nominal","title":"Requested-prefix branch"},"y":{"field":"tokens","type":"quantitative","aggregate":"sum","title":"Prompt tokens (tokens)","scale":{"zero":true}},"color":{"field":"source","type":"nominal","title":"Prompt-token source","scale":{"scheme":"category10","domain":["Local compute","Local HBM hit","External KV transfer"]}},"tooltip":[{"field":"probe","type":"nominal","title":"Probe"},{"field":"source","type":"nominal","title":"Source"},{"field":"tokens","type":"quantitative","title":"Tokens"}]}}
```

The composition is the functional result: the below-threshold request was entirely recomputed, whereas nearly the entire above-threshold request came from external KV with the expected small local and computed tails.

## Deployment and control-plane configuration

The successful path used:

- EPP image: `quay.io/rh-ee-aperdomo/llm-d-router-endpoint-picker:dev-a808637f`.
- Static policy: `loadPolicy: threshold`.
- Threshold: `minExternalReusableTokens: 1024`.
- Token producer model: `nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8`.
- Token producer URL: `https://llm-d-selective-loading-28d3b96d7e-kserve-workload-svc.benchflow.svc:8000`.
- KV event topic: `kv@$(POD_IP)@nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8`.
- `POD_IP` populated from `status.podIP`.
- vLLM connector option: `self_describing_kv_events=true`.
- EPP topic filter: `kv@`, allowing the consumer to accept the qualified topic while extracting endpoint and model identity from the event envelope/topic.
- Offload hierarchy: CPU primary only; no filesystem, object-store, or P2P secondary tier.

The `dev-a808637f` tag corresponds to local source commit `a808637f963e43ba9a6f1e0ebaed3182e918ca6c` (`feat: Threshold based policy`). Embedded or exported image build-commit metadata observed during this work was stale, so the tag, source commit, deployed configuration, and behavioral result are recorded separately; the embedded metadata must not be used as sole provenance.

## Wiring failures and corrections

### TLS short-host failure

The token producer initially addressed the model service with the short hostname `https://llm-d-selective-loading-28d3b96d7e-kserve-workload-svc:8000`. TLS verification failed because that host form did not match the serving certificate identity. Replacing it with `https://llm-d-selective-loading-28d3b96d7e-kserve-workload-svc.benchflow.svc:8000` fixed the connection without disabling certificate verification.

This is durable deployment guidance: when EPP tokenization calls a TLS-enabled KServe workload service, use a DNS name present in the certificate SAN rather than assuming Kubernetes short-name expansion is equivalent for TLS verification.

### Cache-evidence identity

A bare `kv@` publisher topic and non-self-describing events were insufficient for the precise-prefix path to associate events reliably with the discovered backend and model. The working configuration added `POD_IP` from the downward API, published on `kv@$(POD_IP)@nvidia/Llama-3_1-Nemotron-Ultra-253B-v1-FP8`, and enabled `self_describing_kv_events` in the OffloadingConnector configuration.

The EPP continued filtering on the `kv@` prefix. This preserved broad subscription while supplying the endpoint and model identity required by the indexer.

### Redundant tokenizer workload

A KServe tokenizer workload created during the initial wiring remained redundant and crashing. The successful EPP token producer called the model workload service directly, so the crashing tokenizer was not on the validated request path and did not invalidate the final two probes. It remains an operational caveat: remove or repair the redundant tokenizer before treating the deployment as clean, and do not infer tokenizer redundancy is generally safe for other configurations.

### No replay or index warmup

The KV-event index had no replay mechanism or explicit warmup in this validation. Events emitted before the corrected EPP became ready were not assumed to exist in its index. The final run therefore primed the target prefixes only after the corrected EPP and event subscription were live.

This matters for rollout and restart tests: a newly started EPP may fail open until it observes fresh store events, even though vLLM's CPU tier already contains reusable data. A production validation needs an explicit index-warmup criterion or a documented fail-open window.

## Failed and inconclusive full-cache attempts

### Attempt 1: evidence path misconfigured — invalid

The first full-cache run used 15 requested 122,000-token churn prompts, observed 1,830,220 prompt tokens, and spent 243.842 seconds in churn. Both probes restored externally: the 512 request used 512 external tokens and loaded 67,108,864 bytes; the 2,048 request used 2,032 external tokens and loaded 266,338,304 bytes.

The 512 result contradicted the 1,024 threshold, but it did not falsify the policy. The EPP tokenization and event-identity path was not healthy, so the policy lacked reliable compatible evidence and preserved default vLLM loading. This attempt is invalid for threshold enforcement.

Raw artifact: `.calibration-artifacts/2026-09-08-router-threshold-validation-pre-fix.json`. The duplicate filename `.calibration-artifacts/2026-09-08-router-threshold-validation.json` contains the same bytes.

### Attempt 2: corrected wiring, non-deterministic cache state — inconclusive

After the TLS and event-identity fixes, a full-cache run issued 16 requested 122,000-token churn prompts, observed 1,952,246 prompt tokens, and spent 1,143.677 seconds in churn. The 512 probe recomputed all 518 tokens, but the 2,048 probe also failed to restore: it recorded 16 local HBM tokens, 2,039 local-compute tokens, and zero external tokens.

The below-threshold branch was consistent with policy, but the absence of an above-threshold restore meant the run could not validate the second branch. With the normal 1,824,960-token HBM cache, very large churn, no index replay, and no direct proof that the expected external evidence remained usable at decision time, the intended cache state was not deterministic. This attempt is inconclusive rather than a performance result.

Raw artifact: `.calibration-artifacts/2026-09-08-router-threshold-validation-post-fix.json`.

## Deterministic reduced-cache validation

The model deployment was temporarily restarted with 8,192 GPU blocks, equivalent to 131,072 tokens at the configured 16-token block size. The 512- and 2,048-token targets were then primed after EPP readiness. Two distinct requested 70,000-token churn prompts produced 140,029 observed prompt tokens, exceeding the temporary GPU cache and deterministically displacing the target prefixes while leaving their CPU-offloaded copies available.

The two prime requests took 441.641 ms for the 512 target and 585.148 ms for the 2,048 target. Churn took 17,498.742 ms in total; the slower churn request took 17,137.086 ms. The measured probes then took 528.441 ms and 474.435 ms respectively. The complete runner interval was 34.636 seconds, from 2026-09-08T01:44:57Z through 2026-09-08T01:45:32Z.

The 512 probe crossed the router as a below-threshold external match and reached vLLM with loading disabled. Its metric delta was 518 local-compute tokens, zero local HBM hits, zero external tokens, and no CPU-to-GPU load. The target's 512 cacheable tokens were newly created again, as expected for recomputation.

The 2,048 probe crossed the router as an above-threshold external match and retained normal loading. Its metric delta was 2,032 external tokens, 16 local HBM tokens, seven local-compute tokens, 2,032 load tokens, and 266,338,304 CPU-to-GPU load bytes. The response reported 2,048 cached tokens. This is the decisive functional evidence for the restore branch.

Raw artifact: `.calibration-artifacts/2026-09-08-router-threshold-validation-reduced-cache.json`.

## What this checkpoint establishes

Measured observation: the EPP image, qualified evidence path, and vLLM binary opt-out jointly produced the correct behavior on opposite sides of the configured 1,024-token threshold.

Inference: the threshold plugin can be exercised end to end when its cache evidence is fresh, model-compatible, and associated with the selected endpoint.

Not established: the single-request wall times do not show that 1,024 is optimal, that loading 2,048 tokens is always faster than recomputation, or that the policy is safe under representative concurrency and transfer pressure. Those questions remain governed by [[07 - Selective loading calibration test plan]] and the limitations in [[08 - 2026-09-07 selective loading calibration results]].

## Restoration and cleanup

After the validation, the temporary 8,192-block override was removed and the model returned to its normal 1,824,960-token GPU KV-cache capacity. Temporary debug logging and authorization overrides used during diagnosis were removed. The final deployment should therefore not be assumed to retain the deterministic reduced-cache conditions or diagnostic access used for this test.

The threshold EPP image and functional wiring were the subject of the validation. The redundant crashing KServe tokenizer remained a caveat and should be cleaned up separately.

## Artifact registry

| Artifact | Purpose |
|---|---|
| `.calibration-artifacts/2026-09-08-router-threshold-validation-pre-fix.json` | Invalid first full-cache attempt with broken evidence path |
| `.calibration-artifacts/2026-09-08-router-threshold-validation.json` | Byte-identical duplicate of the pre-fix artifact |
| `.calibration-artifacts/2026-09-08-router-threshold-validation-post-fix.json` | Inconclusive corrected-wiring full-cache attempt |
| `.calibration-artifacts/2026-09-08-router-threshold-validation-reduced-cache.json` | Accepted deterministic functional validation |
| `/private/tmp/validate_selective_kv_threshold.py` | Live prime/churn/probe validation runner |
| `/private/tmp/llmisvc-selective-kv-threshold-json-patch.json` | Initial threshold EPP and request-control patch |
| `/private/tmp/llmisvc-fix-selective-kv-evidence.json` | Service FQDN, qualified KV topic, and `POD_IP` corrections |
| `/private/tmp/llmisvc-self-describing-events-json-patch.json` | `self_describing_kv_events` connector correction |
| `/private/tmp/epp-selective-kv-threshold-patch.yaml` | Standalone EPP threshold configuration snapshot |
| `/private/tmp/selective-kv-model-metrics` | Model metric snapshot collected during diagnosis |

The `/private/tmp` artifacts are ephemeral working files; the three JSON records under `.calibration-artifacts` are the durable local evidence for this checkpoint. No MLflow run was created for these direct live probes.

## Next steps

1. Remove or repair the redundant KServe tokenizer so every deployed workload is healthy.
2. Add an explicit EPP cache-index readiness or warmup signal, or document the fail-open interval after EPP restart.
3. Repeat the narrowed threshold sweep with multiple valid repetitions while keeping model placement, cache size, telemetry, and pressure controlled.
4. Validate threshold behavior under representative request and CPU-transfer pressure before production activation.
5. Make the router image report trustworthy build provenance and verify it against the deployed digest rather than relying on a mutable development tag.
6. Extend validation to secondary tiers only after the CPU-primary path remains stable.

## Related

- [[00 - Index]]
- [[06 - 2026-09-05 naive llm-d-router static gating prototype]]
- [[07 - Selective loading calibration test plan]]
- [[08 - 2026-09-07 selective loading calibration results]]
- [[05 - 2026-09-04 vLLM binary opt-out cluster validation]]
- [[03 - llm-d-router selective load and offload policy and wiring]]

## Provenance

Direct live execution against the Diadochos cluster on 2026-09-08, the three request-level JSON artifacts listed above, EPP and LLMInferenceService patch snapshots, vLLM metric deltas, and local source inspection of llm-d-router commit `a808637f963e43ba9a6f1e0ebaed3182e918ca6c`. No benchmark-ranking claim is made, no samples were omitted from Figure 1, and no unmeasured latency was inferred.