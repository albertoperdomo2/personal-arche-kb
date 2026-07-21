# llm-d

## What / Why
Distributed inference / serving. Performance and scalability work as part of the PSAP team at Red Hat.

## Current Focus
- Forge orchestration for repeatable comparisons of default, precise-prefix-cache, and approximate-prefix-cache routing.
- Isolated deployment × workload benchmark matrices launched from Fournos PR comments.

## Key Context
- Repo: `openshift-psap/forge`, project `projects/llm_d`
- Areas I touch: LLMInferenceService rendering, deployment profiles, GuideLLM workloads, Fournos presets.

## Recent Progress
- 2026-07-21: Added the `llama-33-70b-rhoai-release` preset for a 1 model × 3 deployment × 3 workload matrix (9 isolated runs).
- Standardized these release deployments on 4 replicas with tensor parallelism 2.
- Translated working approximate- and precise-prefix-cache LLMInferenceService references into reusable Forge manifests. Precise routing dynamically resolves the model name, EPP service, and cached-model mount path.
- Converted vLLM arguments to underscore-keyed dictionaries so precise-prefix settings deep-merge with shared defaults.
- Configured the Llama release preset to install RHOAI from the default `redhat-operators` catalog's `beta` channel (3.5 EA2 at this checkpoint), without the RC custom-catalog path.
- Diagnosed the EPP failure as a `FailedAttachVolume Multi-Attach` error: the RWO model-cache PVC was attached to the four serving pods' GPU node while the scheduler's tokenizer sidecar was scheduled on a different node.
- Enabled required pod affinity on both LLMInferenceService serving and EPP scheduler templates. The shared `app.kubernetes.io/name` selector and `kubernetes.io/hostname` topology keep all pods of that service together; the scheduler also has the GPU-node selector, so it cannot establish the group on a CPU node. No hostname is supplied in the PR comment.
- Validated rendering and affinity shape with the llm_d test suite. On-cluster validation of the former Multi-Attach case remains pending.

## Launch from GitHub
Comment on a Forge pull request:

```text
/test fournos llm_d llama-33-70b-rhoai-release
/cluster <registered-cluster-name>
```

The cluster name must match the Fournos cluster registration. EPP and serving pods co-locate automatically. Because the release profile uses 4 replicas × TP=2, the selected GPU node needs at least 8 allocatable GPUs or the serving replicas will remain Pending.

Add `/fournos wip` or `/fournos staging` only when targeting the corresponding Fournos control namespace; otherwise the default is `psap-automation`. An optional `/var runtime.model_name: organization/model` line overrides the preset's model.

The preset selects the moving `beta` channel head; it does not pin an immutable operator bundle or image digest. RHOAI EA deployments require a fresh installation rather than an upgrade from an existing EA/GA installation.

## Open Threads
- Re-run the prior RWO model-cache scenario and confirm the EPP scheduler and all serving pods share one node with no `FailedAttachVolume` events.
- Run the matrix on the target RHOAI cluster and record accepted/rejected benchmark runs and MLflow links under the relevant research experiment.
- Confirm model-cache storage/reuse strategy for nine isolated namespaces before repeated 70B runs.

## Related
- [[vLLM]]
- Incidents: _link resolved llm-d issues here as they land in `Incidents/`._