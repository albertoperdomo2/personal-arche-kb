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
- Added optional `runtime.node_name` placement. It renders required `kubernetes.io/hostname` node affinity on both the serving and scheduler templates so all four TP=2 replicas and the EPP scheduler are pinned to one selected node.
- Validated the preset through the actual `llm_d ci` dry-run entrypoint; all nine LLMInferenceService manifests rendered successfully. The node-affinity renderer is covered by the llm_d test suite.

## Launch from GitHub
Comment on a Forge pull request:

```text
/test fournos llm_d llama-33-70b-rhoai-release
/cluster <registered-cluster-name>
/var runtime.node_name: <kubernetes-node-hostname>
```

The cluster name must match the Fournos cluster registration. The node name must exactly match the node's `kubernetes.io/hostname` label; find it with `oc get nodes -L kubernetes.io/hostname`. Because the release profile uses 4 replicas × TP=2, the selected node needs at least 8 allocatable GPUs or the serving pods will remain Pending.

Add `/fournos wip` or `/fournos staging` only when targeting the corresponding Fournos control namespace; otherwise the default is `psap-automation`. An optional `/var runtime.model_name: organization/model` line overrides the preset's model. Omit `runtime.node_name` to retain the scheduler's normal placement behavior.

The preset selects the moving `beta` channel head; it does not pin an immutable operator bundle or image digest. RHOAI EA deployments require a fresh installation rather than an upgrade from an existing EA/GA installation.

## Open Threads
- Run the matrix on the target RHOAI cluster and record accepted/rejected benchmark runs and MLflow links under the relevant research experiment.
- Confirm model-cache storage/reuse strategy for nine isolated namespaces before repeated 70B runs.

## Related
- [[vLLM]]
- Incidents: _link resolved llm-d issues here as they land in `Incidents/`._