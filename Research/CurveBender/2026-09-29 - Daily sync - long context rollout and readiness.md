---
title: CurveBender daily sync — long-context rollout and production readiness
date: 2026-09-29
type: meeting-note
topic: CurveBender
source: daily-sync transcript
status: captured
---

# CurveBender daily sync — long-context rollout and production readiness

## Headline

The daily focused on enabling one-million-token context safely, beginning deployment automation, reproducing an API compatibility issue, expanding evaluation and model-performance coverage, and giving rollout reliability and SRE readiness explicit ownership.

The main safety decision was clear: any feature that can crash a node must be fully validated on a representative non-production environment before it is tested on the production RITS Rocky cluster.

## One-million-token context

### PCP rollout

The team reported that prefill context parallelism (PCP) had been validated with KV events and was rolling out. The local repository corroborates this with commit `6a45fdf`, “GLM53 PCP Canary,” dated 2026-09-29.

The repository's canary is an eight-node GLM 5.3 prefill/decode DisaggregatedSet:

- prefill: four H100 nodes, DP4 × PCP8, TP1, EP32, CPU KV offload, and per-rank KV-event publication;
- decode: four H100 nodes, DP16 × TP2, EP32, CUDA graphs, and NIXL KV consumption;
- routing: precise prefix-cache routing driven by KV events.

Treat these as canary-repository facts, not proof that the identical configuration is currently live in production.

### Prefill versus decode capacity

The prefill workers were reported to have roughly 2.5 million tokens of capacity per rank, enough for the target context on that side. Decode capacity had not been calculated or logged clearly enough to make the same claim.

**Working decision:** retain the current approximately 256k context limit until effective decode KV capacity is understood and validated.

**Action:** add or improve logging that prints effective decode KV-cache capacity, then use it to derive a safe configuration.

### HiCache reproduction

A tiered-cache feature transcribed as “Hisparce,” likely HiCache, was reproduced on an SCC cluster using Rob's configuration.

Observed on SCC:

- no kernel panic or node crash;
- no material performance increase;
- NIXL registration took approximately 50 seconds with the feature enabled, compared with about one second or less without it.

Rob clarified that the expected benefit is capacity rather than like-for-like speed. Unlike a conventional offloading connector that retains KV for possible future requests, this mechanism makes GPU and CPU KV memory available to active requests and pages buffers during forward passes. The intended result is longer supported context or more active KV capacity.

The SCC result was not considered sufficient because its network differs from the production-like Rocky environments.

### Safe validation environments

Candidate environments mentioned:

- llm-d Rocky in Frankfurt, which is important but does not carry the production workload;
- another private Rocky OpenShift cluster with fewer GPUs, if the deployment can be scaled down;
- a POC processor cluster where another contributor was attempting the setup;
- CKS, where deployment work was still in progress.

**Guardrail:** do not test the kernel-panic-producing configuration on RITS Rocky until the team understands the failure and validates it elsewhere.

## Deployment automation

The operations team planned an internal requirements meeting on 2026-09-29. The facilitator asked for an initial direction by the next daily and encouraged starting with a small useful step while the broader strategy is defined.

The local repository already contains useful inputs:

- `argocd-poc/` for an ArgoCD app-of-apps;
- declarative deployment snapshots under `deployments/`;
- automatic kustomization discovery through `scripts/kustomize-units.sh`;
- CI validation that uses the sibling `curvebender-functional-tests` repository.

The meeting did not select a tool or architecture.

## API compatibility

Rachel was reproducing a Nemotron-specific compatibility issue in the llm-d Rocky LightLLM setup. The proposed repair was:

1. reproduce the observed failure;
2. use an asynchronous pre-call hook;
3. inject the missing data into the request's `extra_body`;
4. scope the rewrite to Nemotron;
5. verify the behavior against the failing client path.

The precise missing fields and failing API request were not included in the transcript. Logs and prior discussion were said to be available in the relevant channel.

## Evaluation and regression

### CyberGym and Lightwell

CyberGym support had been added to XGENTIC to represent Lightwell's security/CVE workload. The benchmark contains about 1,500 tasks, and the team expected to select a representative subset rather than run all tasks by default.

A meeting with Marcio was scheduled for early the following week to determine how to add the workload to regression.

Open items:

- obtain the exact Lightwell production settings;
- choose the task categories that represent expected use;
- define regression frequency and pass criteria;
- identify which clusters can supply evaluation capacity.

### Model performance

**Granite 5**

- An initial golden configuration existed for the SFT checkpoint.
- New preview RL checkpoints were available but had not yet received the same treatment.
- Final Granite 5 checkpoints were expected in roughly one or two weeks.
- A possible Granite 5 hackathon would require accelerator capacity and prioritization.
- Assets for the accepted SFT configuration were to be kept in the repository.

**DeepSeek**

- An initial configuration existed.
- P/D disaggregation still needed testing.
- Work was expected to continue later in the week.

The transcript did not provide run IDs, exact configurations, or measured results. Do not treat these status statements as benchmark evidence.

## LightLLM and cluster maintenance

The team was waiting for confirmation from Forge before severing old connections and relying only on the newer LightLLM instance in IBM Cloud.

Separately, returning bad nodes to service had exposed complications around a paused MachineConfig associated with Magma logging, described as a CISO requirement. The owner was determining whether applying it would require machine restarts and a planned shutdown. The logging itself was working, but filtering might not match the desired configuration.

No maintenance decision was made on the call.

## SRE and production readiness

The daily exposed an ownership and access gap:

- at least one contributor expected to help with SRE work did not have cluster access;
- no formal operational-excellence plan was visible to the group;
- the board contained an SRE item, but active ownership was unclear.

The agreed direction was for the new contributors to coordinate with Chris and Michael and produce a plan covering automation and SRE support.

The plan still needs concrete acceptance criteria such as access, alerts, runbooks, ownership, maintenance, failure handling, and escalation.

## Rollout reliability

Rob raised a separate high-priority concern: rolling out changes to the disaggregated serving setup under traffic was degrading KV-cache hit rates because of known rollout behavior.

The facilitator agreed to create a top-level milestone for rollout reliability and to sync with Rob on the exact work.

This issue is operationally distinct from model quality or steady-state performance. It needs tests that observe cache continuity, revision behavior, capacity during transition, latency, errors, and recovery while traffic remains active.

The repository includes `rollouts-disagg-set/`, a manual zero-downtime P/D rollout procedure and results. It should be reviewed as the starting point for the new milestone rather than assumed to solve the production case.

## Other status

- A previously tracked “auto mode” item was considered done enough for the moment, with a workable temporary solution. The transcript does not define the feature clearly enough to document further.
- The project board still contained duplicates and stale placement. Contributors were asked to update status before the daily so discussion could focus on the most important items.
- Daily meetings were intended to remain short, with Slack used for detailed follow-up.

## Decisions

1. Keep the production context limit near 256k until decode-side KV capacity is known.
2. Add visibility into effective decode KV capacity.
3. Continue HiCache reproduction on non-production Rocky environments.
4. Do not expose production RITS Rocky to a node-crash risk without representative validation.
5. Produce an initial deployment-automation direction quickly.
6. Reproduce the Nemotron API issue before implementing the scoped hook.
7. Treat the Granite SFT configuration as the current initial golden config; separately evaluate RL checkpoints.
8. Establish an SRE/production-readiness plan with explicit owners.
9. Create a top-level milestone for rollout reliability and cache continuity.

## Action register

| Action | Owner mentioned | State |
| --- | --- | --- |
| Determine and log effective decode KV capacity | Rob/Maroon with a volunteer to be confirmed | Open |
| Reproduce HiCache on Rocky without touching production | Praveen and contributors using llm-d Rocky or another Rocky cluster | In progress |
| Define deployment-automation requirements and initial direction | Operations/RITS team | In progress |
| Reproduce Nemotron API issue and validate scoped async hook | Rachel | In progress |
| Select CyberGym tasks representing Lightwell | Evaluation team with Lightwell input | Open |
| Add evaluation workload to regression | Evaluation team and Marcio | Planned |
| Preserve Granite SFT golden-config assets in repository | Praveen/evaluation-performance team | Open |
| Test Granite RL checkpoints as capacity permits | Model-performance team | Open |
| Test DeepSeek P/D disaggregation | Model-performance team | Open |
| Decide whether Magma MachineConfig requires maintenance | Michael/operations | In progress |
| Draft SRE and production-readiness plan | New SRE contributors with Chris and Michael | Open |
| Define rollout-reliability milestone | Carlos and Rob | Open |

Owner names reflect transcript attribution and should be checked against the project board.

## Open questions

1. What is the exact decode KV capacity with the proposed one-million-token configuration?
2. Is “HiCache” the correct feature name, and which version or branch is under test?
3. What caused the earlier kernel panic, and is the failure network-, driver-, kernel-, memory-registration-, or configuration-dependent?
4. Which Rocky environment most closely matches production network and host-memory behavior?
5. Which one-million-context tests exercise both capacity and realistic multi-user behavior?
6. What fields are missing in the Nemotron API path?
7. Which CyberGym subset represents Lightwell, and what pass threshold gates a release?
8. Which Granite checkpoint will be used for a hackathon and eventual production?
9. What cache-hit degradation is acceptable during a rollout?
10. Who owns operational approval for production changes?

## Source

- User-provided daily-sync transcript, supplied 2026-09-29. Speaker names are incomplete and diarization is unreliable.
- Local repository `/Users/aperdomo/workspace/redhat/curvebender-tools`, branch `main`, inspected at `c382f57`.
- Repository commit `6a45fdf`, “GLM53 PCP Canary,” dated 2026-09-29.