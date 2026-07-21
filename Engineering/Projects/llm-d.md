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
- Enabled dynamic serving-replica co-location by default. Forge renders required pod affinity on the controller's workload-only labels with `kubernetes.io/hostname` topology: the first GPU-serving replica selects a viable GPU node and the other replicas must join it. No hostname is supplied in the PR comment.
- Validated the preset through the actual `llm_d ci` dry-run entrypoint; all nine LLMInferenceService manifests rendered successfully. The pod-affinity renderer is covered by the llm_d test suite.

## Launch from GitHub
Comment on a Forge pull request:

```text
/test fournos llm_d llama-33-70b-rhoai-release
/cluster <registered-cluster-name>
```

The cluster name must match the Fournos cluster registration. The serving replicas co-locate automatically. Because the release profile uses 4 replicas × TP=2, the GPU node selected for the first serving replica needs at least 8 allocatable GPUs or the remaining replicas will remain Pending.

Add `/fournos wip` or `/fournos staging` only when targeting the corresponding Fournos control namespace; otherwise the default is `psap-automation`. An optional `/var runtime.model_name: organization/model` line overrides the preset's model.

The preset selects the moving `beta` channel head; it does not pin an immutable operator bundle or image digest. RHOAI EA deployments require a fresh installation rather than an upgrade from an existing EA/GA installation.

## Open Threads
- Run the matrix on the target RHOAI cluster and record accepted/rejected benchmark runs and MLflow links under the relevant research experiment.
- Confirm model-cache storage/reuse strategy for nine isolated namespaces before repeated 70B runs.

## Related
- [[vLLM]]
- Incidents: _link resolved llm-d issues here as they land in `Incidents/`._