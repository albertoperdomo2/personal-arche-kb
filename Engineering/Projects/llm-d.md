# llm-d

## What / Why
Distributed inference / serving. Performance and scalability work as part of the PSAP team at Red Hat.

## Current Focus
- Forge orchestration for repeatable comparisons of default, precise-prefix-cache, and approximate-prefix-cache routing.
- Isolated deployment × workload benchmark matrices launched from Fournos PR comments.

## Key Context
- Repo: `openshift-psap/forge`, project `projects/llm_d`
- Areas I touch: LLMInferenceService rendering, deployment profiles, model-cache PVCs, GuideLLM workloads, Fournos presets.

## Recent Progress
- 2026-07-21: Added the `llama-33-70b-rhoai-release` preset for a 1 model × 3 deployment × 3 workload matrix (9 isolated runs).
- Standardized these release deployments on 4 replicas with tensor parallelism 2.
- Translated working approximate- and precise-prefix-cache LLMInferenceService references into reusable Forge manifests. Precise routing dynamically resolves the model name, EPP service, and cached-model mount path.
- Converted vLLM arguments to underscore-keyed dictionaries so precise-prefix settings deep-merge with shared defaults.
- Configured the Llama release preset to install RHOAI from the default `redhat-operators` catalog's `beta` channel (3.5 EA2 at this checkpoint), without the RC custom-catalog path.
- Diagnosed two scheduler failures caused by a ReadWriteOnce model-cache PVC. First, EPP scheduled on a different node and hit `FailedAttachVolume Multi-Attach`. The attempted affinity workaround then added a partial `router.scheduler.template`, suppressing KServe's default EPP-container injection and causing `SchedulerReconcileError: spec.template.spec.containers: Required value`.
- Removed the affinity workaround completely, preserving KServe's default EPP injection. The cache configuration documents the supported GitHub overrides for an RWX-capable class: `model_cache.pvc.access_mode` and `model_cache.pvc.storage_class_name`.
- Set every defined GuideLLM benchmark profile (`short`, `concurrent-1k-1k`, `heavy-heterogeneous`, and `multi-turn`) to an explicit `timeout_seconds: 3600`.
- Fixed the GuideLLM timeout path: the workload timeout now sets the Kubernetes Job `activeDeadlineSeconds` and the Forge poll budget (`timeout + 60s` status-observation grace). On wait expiry, Forge captures the GuideLLM Job, pod state, and logs before raising. This fixes the prior 30m34s hard-coded waiter expiration during the 5 × 600s `concurrent-1k-1k` sweep.
- 2026-07-22: Two post-fix athena-fire staging runs using RWX `nfs-rwx` confirmed that `concurrent-1k-1k` completes in about 51m41s–51m46s. The second run, `forge-llm-d-20260722-101720`, completed the benchmark at 11:17:06 but the Fournos PipelineRun's one-hour window expired during capture/cleanup. No later workload or deployment began. This is an outer pipeline-budget constraint, not a GuideLLM Job timeout or benchmark failure.

- 2026-07-22: The precise-prefix-cache deployment in run `20dff480377b47ab98485f59ae105ca5` failed before benchmarking. The workload command is launched through an image-provided `eval`; its rendered `--kv-events-config={"..."}` loses JSON double quotes during that second shell parse, so vLLM receives `{enable_kv_cache_events: ...}` and crashes with `Invalid JSON: key must be a string`. The same run also shows the EPP tokenizer configured with `/mnt/models/base/model`, which lacks `config.json`; after correcting KV JSON escaping, the EPP mount path must be corrected too.

- 2026-07-23: Implemented the precise-prefix-cache rendering corrections. The KV-events value now retains literal single-quote shell grouping around the JSON so the serving image's `eval` passes valid JSON to vLLM, and cached-model EPP tokenization now maps `base` to `/mnt/models/base` rather than the invalid nested `/mnt/models/base/model`. Focused profile tests (53), Ruff, and diff checks pass; a cluster rerun remains required.

## Launch from GitHub
For a cluster with an RWX-capable storage class:

```text
/test fournos llm_d llama-33-70b-rhoai-release
/cluster <registered-cluster-name>
/var model_cache.pvc.access_mode: ReadWriteMany
/var model_cache.pvc.storage_class_name: <rwx-storage-class>
```

The cluster name must match the Fournos cluster registration. Both PVC overrides are needed: setting only an RWX storage class while keeping `ReadWriteOnce` does not permit cross-node mounts. A newly created PVC is required; an existing RWO PVC with the same cache name cannot change access mode or storage class in place.

Add `/fournos wip` or `/fournos staging` only when targeting the corresponding Fournos control namespace; otherwise the default is `psap-automation`. An optional `/var runtime.model_name: organization/model` line overrides the preset's model.

The preset selects the moving `beta` channel head; it does not pin an immutable operator bundle or image digest. RHOAI EA deployments require a fresh installation rather than an upgrade from an existing EA/GA installation.

## Open Threads
- Raise/configure the outer Fournos PipelineRun timeout before launching this matrix. A 1-hour ceiling is insufficient even for the first `concurrent-1k-1k` run plus result collection, and cannot accommodate nine sequential runs.
- Identify the target cluster's RWX-capable storage class.
- Re-run the prior model-cache scenario using a new RWX PVC and confirm EPP starts with KServe's default container injection and no `FailedAttachVolume` events.
- Run the matrix on the target RHOAI cluster and record accepted/rejected benchmark runs and MLflow links under the relevant research experiment.
- Correct precise-prefix-cache KV-events JSON escaping for the serving image's `eval` entrypoint, then correct its EPP tokenizer mount path (`/mnt/models/base/model` is invalid).\n- Confirm model-cache storage/reuse strategy for nine isolated namespaces before repeated 70B runs.

## Related
- [[vLLM]]
- Incidents: _link resolved llm-d issues here as they land in `Incidents/`._