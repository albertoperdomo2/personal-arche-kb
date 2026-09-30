---
title: CurveBender functional-test strategy review
date: 2026-09-30
type: design-review
topic: CurveBender
status: active
issue: https://github.com/project-cb-lab/curvebender-tools/issues/150
repositories:
  - /Users/aperdomo/workspace/redhat/vllm
  - /Users/aperdomo/workspace/redhat/llm-d-router
---

# CurveBender functional-test strategy review

## Decision

Issue 150 has the right top-level architecture: separate **static**, **function**, and **system** tests according to the evidence each test can access and the claim it can prove. The proposed capability profile is also the correct way to prevent one model family's behavior from becoming an accidental requirement for every fleet.

The plan should proceed after the following corrections:

1. Replace the universal saturation rule with a router-version-aware contract. Current llm-d-router deliberately injects a configured saturation detector into scheduling profiles when that detector also implements the scheduling filter interface.
2. Treat “layer” as an **ingress path**. A normal HTTP client can call vLLM, the Envoy/router gateway, or LiteLLM. It does not call the EPP ext-proc service as though it were another OpenAI endpoint.
3. Make endpoint and option coverage conditional by surface and deployment. vLLM exposes more endpoints and fields than every CurveBender fleet necessarily enables.
4. Generate applicable cases from profiles instead of taking the full Cartesian product of all axes.
5. Validate declared model-name mappings rather than requiring all names to be identical.
6. Add parser selection and routing semantics to protocol coverage. A request that reaches vLLM through the passthrough parser can return 200 while silently losing prefix-aware routing and usage-derived behavior.
7. Split client disconnect coverage between function and system tests: the client-visible stream behavior belongs in function tests; resource reclamation belongs in system tests.

This review is based on read-only inspection of:

- vLLM commit `741edeebeec3cedbe938d831b6d87641ed6191ef` on `main`, inspected 2026-09-30;
- llm-d-router commit `c46a1b9800144b34dcd5bcc91c386e1301d440ba`, inspected 2026-09-30;
- CurveBender issue 150 as displayed on 2026-09-30.

The llm-d-router checkout was 17 commits ahead of its configured `origin/main` and matched its `upstream/main`. It also contained unrelated untracked work. No files were changed during this review.

## What “protocol coverage” should mean

Protocol coverage verifies the externally observable API contract required by real clients. It includes:

- which HTTP paths and methods exist;
- request and response schemas;
- streaming event framing and termination;
- usage accounting and finish reasons;
- tool-call delta assembly;
- supported extension fields;
- required, consumed, stripped, rewritten, and returned headers;
- errors for malformed or unsupported requests;
- cancellation behavior;
- the same contract through each supported ingress path.

Protocol coverage is different from model quality. A model can return a semantically weak answer while the protocol is correct. It is also different from cluster readiness: a single successful response does not prove that prefill/decode connectivity, monitoring, or rollout behavior is healthy.

## Repository evidence

### vLLM surfaces

The inspected vLLM checkout implements:

| Surface | Source |
| --- | --- |
| `POST /v1/chat/completions` | `vllm/entrypoints/openai/chat_completion/api_router.py` |
| `POST /v1/completions` | `vllm/entrypoints/openai/completion/api_router.py` |
| `POST /v1/responses` | `vllm/entrypoints/openai/responses/api_router.py` |
| `GET /v1/responses/{response_id}` | `vllm/entrypoints/openai/responses/api_router.py` |
| `POST /v1/responses/{response_id}/cancel` | `vllm/entrypoints/openai/responses/api_router.py` |
| `GET /v1/models` | `vllm/entrypoints/openai/models/api_router.py` |
| `POST /v1/messages` | `vllm/entrypoints/anthropic/api_router.py` |
| `POST /v1/messages/count_tokens` | `vllm/entrypoints/anthropic/api_router.py` |
| Four render endpoints | `vllm/entrypoints/scale_out/render/api_router.py` |

The render endpoints cover chat completions, completions, Messages, and Responses. They are conditional: scale-out factories enable them in dedicated modes or when vLLM starts with `--enable-scale-out`. A profile must therefore declare render support rather than assuming it from the source tree.

The OpenAI protocol models expose many of the fields named in issue 150, including tools, reasoning controls, structured output, log probabilities, multiple choices, token limits, priority, `cache_salt`, and `return_token_ids`. The exact field set differs among Chat Completions, Completions, Responses, and Messages. “Option group” must be represented per surface instead of as one shared list.

The main generative routes use vLLM's cancellation wrapper. That supports a client-visible cancellation test, but resource cleanup requires server or cluster observations and therefore belongs in system coverage.

`X-Request-Id` response behavior is conditional on vLLM's request-ID-header setting. A request body ID and an HTTP response header are separate contracts. The fleet profile must say whether header echo/generation is enabled.

### llm-d-router parsing and routing

The router's default parser chain contains `openai-parser`, `anthropic-parser`, `vllmhttp-parser`, and a trailing `passthrough-parser`. It resolves parsers by the longest matching path suffix. The fallback forwards unmatched traffic but cannot extract payload signals for features such as prefix-cache-aware scheduling.

This creates a critical distinction:

- **endpoint reachable** means a request obtained an HTTP response;
- **endpoint parsed** means EPP recognized the protocol and extracted the attributes that routing, accounting, or flow control depends on.

Both need tests.

The inspected OpenAI parser recognizes standard Responses, Chat Completions, Completions, embeddings, selected media endpoints, and the Chat Completions and Completions render paths. It does **not** recognize `/v1/responses/render`, although the inspected vLLM exposes that endpoint. With the default fallback, such a request may still succeed while bypassing payload-derived routing. This should become an explicit compatibility test and an upstream or local disposition decision.

The router also treats system control headers differently from ordinary client headers. Objective, fairness, TTL, subset, model-rewrite, and related routing headers can be consumed or stripped before the backend, while other headers can be forwarded, replaced, or returned. Tests must declare the expected disposition at each ingress instead of asserting that every header is echoed end to end.

`GET /v1/models` has no generation body. It is useful for discovery and model identity, but it does not prove parser selection, prefix extraction, or scheduling behavior.

## Corrections to the static-test plan

### Use the router's own configuration semantics

Plugin-reference coherence is a valid static gate. The current router loader already validates plugin types, duplicate declarations, profile references, parser references, saturation detector references, feature gates, and dependency structure. CurveBender should invoke an exact-version, component-native validator where possible instead of recreating these rules independently.

There is no obvious standalone `validate-config` command in the inspected checkout. The preferred long-term addition is an offline validation entry point in llm-d-router. Until that exists, the functional-test repository can wrap the exact loader version or run a short-lived validation process from the exact deployed image.

The test result should record:

- config source and rendered hash;
- router version or image digest;
- whether defaulting was applied;
- the normalized plugin/profile graph;
- all validation errors.

### Correct the saturation rule

Issue 150 currently proposes that the same detector must never be both `flowControl.saturationDetector` and a scheduling filter. That is not a universal invariant.

Commit `1a6cafd29923be134b4bb036a3dd1b75c0541fea`, “fix(config): auto-inject saturation detector as scheduling filter,” was added to llm-d-router on 2026-09-08. In the inspected version, `ensureSaturationDetector` injects a detector into every explicit scheduling profile when the detector implements `Filter`. The commit explains that excluding the filter can leave saturated endpoints eligible for high-weight cache scorers and cause hotspotting.

The CurveBender observation of 16,635 responses with 429 status, a 77.9% throughput loss, and TTFT p50 increasing from 48.5 to 360.1 seconds remains valuable evidence. It shows that a particular version, topology, configuration, and workload produced a bad outcome. It does not establish a timeless schema rule.

Replace the proposed assertion with:

> Every fleet declares its router version/image digest and intended saturation semantics. Static validation checks that the configuration is valid under that exact version and records the normalized profile after defaulting. A load/system test verifies the intended queue-versus-reject behavior and endpoint redistribution under saturation.

This also means comments in older values files should be treated as versioned operational evidence, not the specification for current router main.

### Strengthen implicit dependency validation

The `concurrency-detector` case should be expressed as a general rule:

> Every named producer dependency must resolve after the exact router version applies defaults and auto-creation.

Checking merely for `token-load-scorer` is too indirect. That scorer may currently trigger creation of the needed producer, but the real contract is that the configured detector receives live input. Static validation should inspect the resolved dependency graph. A system test should assert that the producer is active and yields data under traffic.

### Model identity is a mapping

Exact string equality across vLLM, LiteLLM, EPP, clients, and benchmarks is too restrictive because LiteLLM can intentionally expose a public alias while targeting a provider-qualified upstream model. Each profile should declare:

- public client model ID;
- LiteLLM `model_name`;
- LiteLLM upstream/provider-qualified model ID;
- EPP or InferenceModel target;
- vLLM `--served-model-name`;
- benchmark request model.

Validation should ensure every transition is declared and resolvable. It may enforce equality for a fleet that chooses a single canonical name, but equality is a profile policy rather than a universal rule.

### Monitor reachability needs static and live halves

The static check should evaluate rendered objects and account for:

- `matchLabels` and `matchExpressions`;
- labels on workload pod templates;
- Service selectors and named ports;
- namespace selectors;
- labels injected by controllers;
- explicitly external targets.

A static label match proves only that discovery is plausible. The system counterpart must assert that Prometheus actually has an active target and recent samples for the expected pods.

### Build-trigger coverage needs Docker semantics

The build-trigger test should account for multi-stage `COPY --from`, directory copies, globs, `.dockerignore`, build context roots, and GitHub Actions `paths` semantics. Only source paths copied from the build context need workflow coverage.

### Secret scanning should use a mature scanner

Use an established secret scanner with a reviewed baseline and allowlist. A simple regular expression will either miss encoded credentials or flag harmless examples such as placeholder API keys. Fail on new findings and retain evidence without copying secret values into reports.

### Add general manifest graph checks

The static suite should also verify:

- duplicate Kubernetes identities after rendering;
- references to ConfigMaps, Secrets, Services, service accounts, PVCs, and InferenceObjectives;
- container ports versus Service target ports and monitor port names;
- selector/template-label agreement for workloads and Services;
- referenced files, mounted keys, and command-line paths;
- image tags against the fleet's mutability policy;
- profile schema plus semantic checks;
- explicit allowlists for resources supplied outside the rendered unit.

## Function-test matrix

The executable case catalog, observed client contracts, assertion rules, and generated matrix are specified in [[2026-09-30 - Functional API compatibility matrix]]. That companion document is the source of truth for function-test case IDs and release-gate scope.

Issue 150 says “five axes” but lists six entries. More significantly, a full Cartesian product will create invalid and redundant cases. Generate cases from a capability profile and use targeted pairwise combinations, plus hand-written tests for high-risk interactions.

A useful case identity is:

```text
surface × transport × feature × ingress × expectation
```

Model family and deployed component versions are properties of the fleet profile and appear in the report.

### Ingress paths

Use these logical ingress names:

- `vllm_direct`;
- `router_gateway` for the Envoy path that invokes EPP;
- `litellm`.

Do not name a normal HTTP ingress `epp` unless the deployment actually exposes a client-facing HTTP service with that contract. EPP itself participates through Envoy's external-processing protocol.

Comparisons across ingresses should assert normalized invariants, not byte equality. IDs, timestamps, chunking, headers, error wrappers, and finish-event order can legitimately differ.

### P0 protocol coverage

The first useful suite should cover the contracts production clients are most likely to depend on:

1. **Discovery**
   - `GET /v1/models` returns the declared public model at each exposed ingress.
   - Aliases resolve according to the profile.

2. **Chat Completions**
   - non-streaming success;
   - streaming SSE framing, terminal marker, and finish reason;
   - `stream_options.include_usage` with nonnegative arithmetic and one terminal usage record;
   - a deterministic tool call whose streamed argument fragments reassemble into valid JSON;
   - family-specific reasoning smoke test with the declared configuration;
   - malformed JSON, missing required fields, unknown model, and context-limit errors.

3. **Cancellation**
   - client terminates a stream and can immediately make a fresh healthy request;
   - system test separately confirms request counters, scheduler state, and KV/resources return to baseline.

4. **Headers**
   - request ID behavior only when the relevant component setting is enabled;
   - objective and fairness headers have the declared consume/strip/forward behavior;
   - unknown benign headers follow the declared policy;
   - rate-limit headers are asserted only where a component promises them.

5. **Anthropic Messages**
   - `POST /v1/messages` non-streaming and streaming for fleets serving Anthropic clients;
   - `POST /v1/messages/count_tokens`;
   - surface-specific error shape;
   - replace the issue's corrupted `anthroding: identity` text with the actual header and expected behavior. If it was intended to mean `anthropic-version`, name that explicitly.

6. **Responses**
   - `POST /v1/responses` non-streaming and streaming when required by clients;
   - stateful fetch and cancel only when the deployment enables that behavior;
   - Responses-specific event and error assertions rather than Chat Completions assumptions.

7. **Render**
   - run only when the profile declares scale-out/render enabled;
   - cover chat, completions, Messages, and Responses render endpoints that the fleet exposes;
   - assert both reachability and the selected router parser;
   - make `/v1/responses/render` an expected finding until router parsing support or an intentional passthrough policy is established.

Every unsupported capability should have an explicit profile disposition: `unsupported`, `expected_rejection`, or `not_exposed`. It must not disappear as an unexplained skip.

### P1 coverage

Add after P0 is stable:

- structured output with schema validation;
- log probabilities and multiple choices where supported;
- `cache_salt`, priority, `ignore_eos`, `min_tokens`, prompt truncation, and returned token IDs per applicable surface;
- parallel tool calls;
- warmed prefix-cache pairs;
- long-context boundary cases;
- retry and idempotency behavior;
- expected rate-limit behavior;
- gateway-specific error transformations.

A warmed-cache assertion should avoid requiring `cached_tokens > 0` unless the profile guarantees affinity and reuse. Otherwise assert the field's schema and nonnegative value, then measure cache reuse in a controlled routing/system test.

### P2 coverage

Add broad combinations, multimodal inputs, less-used extensions, and fault injection only after production-critical client paths are gated.

## Capability profile

A profile needs enough information to generate valid tests and explain failures. One possible shape is:

```yaml
schema_version: 1
fleet: glm-5.3-scc-ib

components:
  vllm:
    version: "<commit-or-version>"
    image_digest: "sha256:<digest>"
    flags:
      enable_scale_out: true
      enable_request_id_headers: true
  router:
    version: "<commit-or-version>"
    image_digest: "sha256:<digest>"
  litellm:
    version: "<version>"
    image_digest: "sha256:<digest>"

identity:
  public_model: "glm-5.3"
  litellm_model_name: "glm-5.3"
  litellm_upstream_model: "openai/glm-5.3"
  epp_target: "glm-5.3"
  vllm_served_model_name: "glm-5.3"
  benchmark_model: "glm-5.3"

ingresses:
  vllm_direct:
    url_env: "CB_VLLM_BASE_URL"
  router_gateway:
    url_env: "CB_ROUTER_BASE_URL"
  litellm:
    url_env: "CB_LITELLM_BASE_URL"

surfaces:
  chat_completions:
    ingresses: [vllm_direct, router_gateway, litellm]
    stream: true
    include_usage: true
    tools: true
    reasoning:
      mode: chat_template_kwargs
      key: enable_thinking
    structured_output: true
    extensions: [cache_salt, priority, return_token_ids]
  messages:
    ingresses: [vllm_direct, router_gateway]
    stream: true
    count_tokens: true
  responses:
    ingresses: [vllm_direct, router_gateway]
    stream: true
    stateful: false
  render:
    enabled: true
    paths:
      chat_completions: parsed
      completions: parsed
      messages: parsed
      responses: expected_passthrough

headers:
  x-request-id:
    vllm_direct: returned
    router_gateway: returned
    litellm: rewritten
  x-llm-d-inference-objective:
    router_gateway: consumed
    backend: stripped
  x-llm-d-inference-fairness-id:
    router_gateway: consumed
    backend: stripped

cluster:
  namespace: "<namespace>"
  prefill_replicas: 4
  decode_replicas: 8
  decode_selector: "llm-d.ai/role=decode"
  saturation_contract: "<named-versioned-contract>"

safety:
  destructive_system_tests: false
  production_safe_only: true
```

URLs come from environment variables. Credentials never belong in the profile or test report.

## System-test additions

The issue's initial system tests should be expanded to verify observed state, not only declared state:

- exact image digests and component versions match the profile;
- rendered config hash matches the config loaded by EPP;
- EPP readiness and the expected ready-endpoint count;
- InferencePool, LWS, and DisaggregatedSet status;
- prefill/decode counts, labels, topology, and readiness;
- a live request proves KV/NIXL transfer across prefill and decode;
- configured parser is selected for each critical surface;
- prefix index and producer metrics become nonempty under suitable traffic;
- every expected Prometheus target is up and has recent samples;
- objective and fairness mapping reaches the expected profile;
- control headers do not leak to the model backend;
- network and authentication boundaries behave as declared;
- rollout, restart, scale-up/down, and one-replica-unavailable scenarios preserve the agreed service level;
- saturation produces the declared queue/reject behavior and redistributes work as intended;
- cancellation releases live request, scheduler, and KV state.

The router exposes plugin/debug state in current code, including `/debug/plugins/state` on its administrative surface. Use this when safely exposed to the test runner; startup logs alone are weaker evidence.

Destructive tests must be opt-in and run in an isolated namespace or designated non-production fleet. A production-safe marker should exclude node failure, forced restarts, malformed live configuration replacement, and other disruptive cases.

## Suggested implementation sequence

### Phase 1: profile and P0 smoke gate

1. Define and validate the profile schema.
2. Implement logical ingress clients.
3. Add discovery, Chat Completions non-streaming/SSE, usage, core unhappy paths, request IDs, and the deployed model's reasoning mode.
4. Emit one machine-readable result per case with fleet, ingress, component versions, duration, status, and normalized failure class.
5. Run the same cases directly and through the router/LiteLLM to localize failures.

**Exit criterion:** the primary production-like fleet has a reproducible report that distinguishes backend, router, and LiteLLM failures.

### Phase 2: routing semantics and static gates

1. Run exact-version router config validation.
2. Validate the resolved producer/plugin dependency graph.
3. Add model identity, objective, monitor, reference graph, build-trigger, and secret checks.
4. Add parser-selection assertions and render coverage.
5. Record normalized/defaulted EPP config.

**Exit criterion:** a 200 response through an unrecognized passthrough path is reported separately from a correctly parsed request.

### Phase 3: system and degraded-stack paths

1. Add live stack conformance and observability checks.
2. Add cancellation cleanup, saturation, rollout, and degraded-replica tests.
3. Run disruptive cases only in an isolated approved environment.

**Exit criterion:** releases can be gated on declared behavior under failure and load, not only happy-path reachability.

### Phase 4: breadth

Add structured output, the remaining extensions, broader surfaces, more model families, and pairwise combinations driven by production clients.

## Proposed acceptance criteria for issue 150

The first deliverable is complete when:

- every fleet profile passes schema and semantic validation;
- reports include exact deployed component versions or image digests;
- the primary client surface passes direct vLLM, router gateway, and LiteLLM P0 cases;
- unsupported features have an explicit expected disposition;
- router parsing is distinguished from passthrough reachability;
- error assertions are surface- and ingress-specific;
- the saturation rule is versioned and verified under load;
- static monitor checks have a live target counterpart;
- cancellation has both client-visible and resource-cleanup coverage;
- no credentials are stored in profiles, fixtures, logs, or reports;
- production-safe and destructive system tests are separately selectable.

## Immediate findings to track

1. **Router/vLLM render mismatch:** vLLM serves `/v1/responses/render`; the inspected router OpenAI parser does not claim it. Decide whether to add parser support, reject it, or explicitly permit passthrough.
2. **Saturation assertion is stale as a universal rule:** current router code auto-injects filter-capable saturation detectors. The CurveBender test must key off the deployed router version.
3. **Version drift is a test input:** CurveBender manifests have used released versions, custom digests, and mutable `:main` images. A profile without an exact version cannot interpret config behavior reliably.
4. **No obvious offline router validator:** using the component's loader is preferable to duplicating its semantics. A supported validation command would simplify the static suite.
5. **The function matrix needs client priorities:** the repositories establish what can be served, but actual Forge, Lightwell, coding-assistant, and other client requirements determine which capabilities are release-blocking.

## Source record

- CurveBender issue 150, “Establish static, function and system test coverage for deployed assets,” read through the GitHub CLI on 2026-09-30. No GitHub mutation was performed.
- Local vLLM repository at `/Users/aperdomo/workspace/redhat/vllm`, commit `741edeebeec3cedbe938d831b6d87641ed6191ef`.
- Local llm-d-router repository at `/Users/aperdomo/workspace/redhat/llm-d-router`, commit `c46a1b9800144b34dcd5bcc91c386e1301d440ba`.
- llm-d-router commit `1a6cafd29923be134b4bb036a3dd1b75c0541fea`, “fix(config): auto-inject saturation detector as scheduling filter.”
- Relevant router sources:
  - `pkg/epp/config/loader/defaults.go`
  - `pkg/epp/config/loader/validation.go`
  - `pkg/epp/framework/plugins/requesthandling/parsers/README.md`
  - `pkg/epp/framework/plugins/requesthandling/parsers/openai/openai.go`
- Relevant vLLM sources:
  - `vllm/entrypoints/openai/*/api_router.py`
  - `vllm/entrypoints/anthropic/api_router.py`
  - `vllm/entrypoints/scale_out/render/api_router.py`
  - `vllm/entrypoints/scale_out/factories.py`
  - protocol models and tests under `vllm/entrypoints/openai/` and `tests/entrypoints/`.