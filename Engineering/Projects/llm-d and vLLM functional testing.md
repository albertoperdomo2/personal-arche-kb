---
title: "llm-d and vLLM functional testing program"
date: 2026-09-28
type: project
topic: functional-testing
status: proposed
---

# llm-d and vLLM functional testing program

## Intent and agreed scope

Alberto has joined a group working on functional testing of llm-d and vLLM and wants to lead the effort. He brings senior performance-engineering experience and wants functional-testing terminology and implementation guidance. He selected Kubernetes/OpenShift with NVIDIA GPUs as the initial platform. The scope is the main supported serving workflows and paths, with a program and deployments that can operate at scale. No release, team size or deadline was specified. Performance benchmarking is a later workstream.

Recommended program: reuse existing upstream component suites and build a small shared user-acceptance layer. Run applicable client contracts against direct vLLM and the actual llm-d entry point. Extend from API basics to distributed feature correctness, composed user journeys, lifecycle changes and scale. Domain SMEs approve specialized contracts and coverage. The proposal is not yet a group decision or an executed qualification result.

## Research checkpoint

Nine public repositories were cloned and statically inspected on 2026-09-28: llm-d, vLLM, llm-d-router, llm-d-infra, llm-d-batch-gateway, llm-d-inference-sim, llm-d-kv-cache, the deprecated standalone routing-sidecar repository, and the autoscaling repository reached through the WVA URL.

Primary snapshots:

- llm-d: `80680722ee9c7153db92c0b4ddc3ca57d70b8c8b`
- vLLM: `6d3ea3c2c9037deadb9fb70d40bc69b7947af463`
- llm-d-router: `8a2f37d3e2060994683ff61a1fb27723775e6d12`
- llm-d-infra: `a8864e03deddcb7a78d4dc8d364bc1ca27138811`
- llm-d-batch-gateway: `2592db064156a929143e2127d95fae906baa52dd`
- llm-d-autoscaling: `3c06c8814bcde4f7da0c7b4fadeed015e135a9de`

No GPU inference, cluster deployment, upstream test execution, CI run-history audit or branch-protection audit was performed. All proposed scenarios are unexecuted for Alberto's target environment. Local validation checks only the planning artifacts, source paths, pins and traceability.

## Findings that change the implementation plan

**Reuse current component ownership.** Sidecar development and KV indexing now belong in llm-d-router. Current autoscaling main contains KEDA blueprints/evaluation tooling; the WVA controller is frozen under legacy. Sources: [KV migration](https://github.com/llm-d/llm-d-kv-cache/blob/8cf43067afb7fc9fefafc1b64de063c769f2c90f/README.md), [sidecar migration](https://github.com/llm-d/llm-d-routing-sidecar/blob/78051b86ff13c4511fb1c8c7def56e04b0850b67/README.md), [autoscaling scope](https://github.com/llm-d/llm-d-autoscaling/blob/3c06c8814bcde4f7da0c7b4fadeed015e135a9de/README.md).

**Existing suites are substantial.** vLLM has explicit API, model, engine, connector, speculative and quantization coverage. Router tests include chart-based simulator E2E, hermetic integration, streaming and disruption cases. Batch gateway has live API lifecycle, tenant isolation and composed Async recovery tests. These are reuse opportunities; current source presence does not prove current CI success. Sources: [vLLM entrypoint jobs](https://github.com/vllm-project/vllm/blob/6d3ea3c2c9037deadb9fb70d40bc69b7947af463/.buildkite/test_areas/entrypoints.yaml), [router E2E](https://github.com/llm-d/llm-d-router/blob/8a2f37d3e2060994683ff61a1fb27723775e6d12/test/e2e/README.md), [batch E2E](https://github.com/llm-d/llm-d-batch-gateway/blob/2592db064156a929143e2127d95fae906baa52dd/test/e2e/README.md).

**The common smoke assertion is limited.** `e2e-validate.sh` checks curl success and an opening brace; it does not parse the full completion contract or verify streaming. Add raw HTTP/SSE and schema assertions through the real user entry point. [Source](https://github.com/llm-d/llm-d/blob/80680722ee9c7153db92c0b4ddc3ca57d70b8c8b/.github/scripts/e2e/e2e-validate.sh#L107).

**Tiered-cache restoration needs explicit proof.** The sampled verifier's load-byte assertion is commented out because its workload does not produce GPU eviction. A successful write or cumulative metric maximum cannot establish a specific replay restored KV correctly. Force and prove external-only state, then validate attributable reads and the output. [Source](https://github.com/llm-d/llm-d/blob/80680722ee9c7153db92c0b4ddc3ca57d70b8c8b/.github/scripts/nightly-e2e-verification/tiered-prefix-cache/verify.py#L197).

**CI invocation is a first-class requirement.** The documented `verify_script` interface was absent from the inspected shared benchmark workflow and callers. Some named OpenShift nightly schedules were commented out. Resolve invocation, scheduling and selected-case counts before counting these as required evidence. Sources: [documented hook](https://github.com/llm-d/llm-d/blob/80680722ee9c7153db92c0b4ddc3ca57d70b8c8b/.github/scripts/nightly-e2e-verification/README.md), [shared workflow](https://github.com/llm-d/llm-d-infra/blob/a8864e03deddcb7a78d4dc8d364bc1ca27138811/.github/workflows/reusable-ci-nightly-benchmark.yaml), [OpenShift baseline caller](https://github.com/llm-d/llm-d/blob/80680722ee9c7153db92c0b4ddc3ca57d70b8c8b/.github/workflows/nightly-e2e-optimized-baseline-ibm-acc-gpu-vllm-x.yaml).

**Resolve the P/D error contract.** Architecture prose describes fallback after prefiller server errors; the sampled NIXLv2 implementation retries selected transient statuses and then propagates the error. Have the SME approve the policy for the pinned image before making an acceptance gate. This is a source/document discrepancy, not a newly reproduced runtime defect. Sources: [architecture](https://github.com/llm-d/llm-d/blob/80680722ee9c7153db92c0b4ddc3ca57d70b8c8b/docs/architecture/advanced/disaggregation/README.md), [implementation](https://github.com/llm-d/llm-d-router/blob/8a2f37d3e2060994683ff61a1fb27723775e6d12/pkg/sidecar/proxy/connector_nixlv2.go#L173).

**Exact generation comparisons need qualification.** vLLM's reproducibility guarantees are conditional; a seed and temperature zero do not justify universal equality across scheduling, hardware or versions. Use structural contracts broadly and approved per-model exact/numerical oracles where valid. [Source](https://github.com/vllm-project/vllm/blob/6d3ea3c2c9037deadb9fb70d40bc69b7947af463/docs/usage/reproducibility.md).

## Proposed delivery and ownership

| Phase | Deliverable | Exit evidence |
|---|---|---|
| M0 | Scope, immutable tuple, reference fixture, ownership and CI audit | Another engineer reproduces deployment; required behaviors have approved oracles |
| M1 | Strict discovery/chat/completion/stream/error/cancellation suite against both targets | Real inference; negative validator checks; complete selected-case/artifact accounting |
| M2 | Routing, prefix/index behavior, real P/D, CPU restoration, flow control and disruption | Correct outputs plus proof that the intended feature and failure phase were exercised |
| M3 | Tools/multi-turn, multimodal, multi-model, batch/async, KEDA, updates/rollback | User journey has success, failure and recovery/cleanup checks |
| M4 | Multi-node/session scale, state cycles, qualification report and release policy | Every logical request reconciled; actual scale tuple declared; domain-reviewed gaps |
| M5 | Wide EP, additional engine profiles/APIs, encoder/coordinator/P2P and other specialist paths | Separate SME-approved capability packs and real hardware evidence |

An indicative 8–12 weeks for the first core qualification milestone assumes a lead, three implementation engineers, shared CI support, SME availability and GPU capacity. This is a planning estimate. Specialist packs continue afterward; advertised features must be promoted to required release scope explicitly.

Alberto should own priorities, the roadmap, evidence quality, coverage decisions and review cadence. Implementation owners maintain tests and fixtures; domain SMEs approve contracts, numerical tolerances and supported combinations; a release authority owns exceptions. Start with a small real-GPU baseline, then a P/D pair across nodes, then a reserved larger distributed profile. Simulator results never substitute for real numerical or transfer qualification.

## Test quality rules

- Each case has requirement ID, applicability, source contract, controlled setup, oracle, owner, timeout rationale, diagnostics and cleanup.
- Distinguish `PASS`, `FAIL`, `BLOCKED`, `INCONCLUSIVE`, `NOT_APPLICABLE` and tracked expected failures. Required skips, missing shards, missing mechanism evidence and dry runs cannot yield a green qualification report.
- Track logical requests separately from internal attempts. Induced faults get explicit allowed outcomes; do not claim exactly-once engine execution or transparent stream migration without a contract.
- Capture artifacts before teardown. Keep first failures and diagnostic reruns; do not erase flakes with retries.
- Version prompts, media, model/tokenizer/template, adapters, numeric backend, connector, topology, images and test code. Record configured references and runtime image IDs.
- Functional scale checks use concurrency to expose state/identity errors. They do not make performance or capacity claims.

## Deliverables and next action

The workspace package is `/private/tmp/llm-d/functional-testing-roadmap/`: leadership brief, roadmap, 31-record evidence ledger, suite design, 66 proposed scenario families across 23 boundaries/paths, 22 work packages, glossary, seven worked specifications, templates, CSV catalogs and exact repository lock. This path is session-local; the public source links above remain the durable evidence. No upstream code, workflow or issue was changed.

Next action: use the charter and coverage catalog in a kickoff; agree one reference tuple, the actual client entry point, initial owners, recurring GPU capacity and the first streaming/error demonstration. Resolve NIXLv2 error behavior and verifier/CI scheduling as explicit M0 decisions.

Related projects: [[llm-d]], [[vLLM]].