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
- Diagnosed the next run failure as a GuideLLM wait-budget defect, not a failed benchmark. `concurrent-1k-1k` uses five 600-second rate levels. At 30m34s the GuideLLM Job was still active and serving logs showed ongoing requests, but Forge's fixed `@retry(attempts=180, delay=10)` wait expired. Although the workload supplies `timeout_seconds: 3600`, the GuideLLM toolbox does not consume it for either the waiter or Kubernetes Job deadline; timeout handling and failure-state capture need correction.

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
- Fix GuideLLM timeout plumbing: derive its wait budget and Job `activeDeadlineSeconds` from `workloads.*.timeout_seconds`, and capture job state/logs when the wait expires.
- Identify the target cluster's RWX-capable storage class.
- Re-run the prior model-cache scenario using a new RWX PVC and confirm EPP starts with KServe's default container injection and no `FailedAttachVolume` events.
- Run the matrix on the target RHOAI cluster and record accepted/rejected benchmark runs and MLflow links under the relevant research experiment.
- Confirm model-cache storage/reuse strategy for nine isolated namespaces before repeated 70B runs.

## Related
- [[vLLM]]
- Incidents: _link resolved llm-d issues here as they land in `Incidents/`._