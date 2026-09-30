---
title: CurveBender functional API compatibility matrix
date: 2026-09-30
type: implementation-specification
topic: CurveBender
status: proposed
parent: "[[2026-09-30 - Functional test strategy review]]"
repositories:
  curvebender_tools: /Users/aperdomo/workspace/redhat/curvebender-tools
  vllm: /Users/aperdomo/workspace/redhat/vllm
  llm_d_router: /Users/aperdomo/workspace/redhat/llm-d-router
---

# CurveBender functional API compatibility matrix

## Purpose

This specification defines the automated function tests that answer:

> Can each real CurveBender client complete the API exchanges it depends on through every supported ingress?

These tests use only a reachable endpoint and client credentials. They validate client-visible HTTP and streaming behavior. Assertions that require pod, router, metric, or cluster access belong in system tests, even when they are triggered by the same request.

The matrix is driven by declared client requirements. It does not equate every endpoint implemented by vLLM with a CurveBender release requirement.


## Repository implementation status

As inspected at functional-test repository commit `b0b02cacaeb8c3cd243f7e0d4080e0f0e061d9fa` (2026-09-28), the repository contains only:

- the root README;
- `src/kustomize-validation/README.md`;
- the self-contained `validate-kustomize.py` renderer.

The protocol matrix described here is **planned, not implemented**. GitHub issue 3 proposes a Python standard-library runner under `src/protocol-matrix/`, JSONL plus self-contained HTML reporting, a `core` CI subset, profiles, layer comparison, alerts, and a prober deployment.

Statements below using “should” or “must” define the target contract. They do not describe behavior already present in the repository.

## Observed client contracts

Read-only inspection of CurveBender tooling found two concrete production-shaped contracts and two evaluation contracts.

### Forge/OpenClaw-style OpenAI traffic

The replay and capture harnesses under `client-harnesses/forge-breifing/` send:

- `POST /v1/chat/completions`;
- `stream: true`;
- `stream_options.include_usage: true`;
- long, multi-turn `messages`;
- large `tools` arrays;
- reasoning, content, tool-name, and tool-argument deltas;
- bearer authentication;
- `x-llm-d-inference-objective`;
- optional session/fairness headers.

This is a release-blocking P0 contract.

### Claude Code-shaped Forge traffic

`agent-console/app/forge.py` sends:

- `POST /v1/messages`;
- `stream: true`;
- `system`, `messages`, `model`, and `max_tokens`;
- `anthropic-version: 2023-06-01`;
- `x-api-key`;
- `x-request-id`;
- `x-llm-d-inference-objective`;
- `x-llm-d-inference-fairness-id`;
- `Accept-Encoding: identity`.

It consumes `message_start` usage and text, thinking, and partial-JSON deltas. This is also a release-blocking P0 contract.

### tau2 agent evaluation

`endpoint-validation/tau2/run_tau2_eval.py` discovers the model through `GET /v1/models` and runs multi-turn OpenAI-style tool calling. Discovery, Chat Completions, tool calls, and the tool-result continuation loop are required to start a tau2 evaluation.

### Kimi vendor-verifier-derived tests

`endpoint-validation/kvv/vendor/` exercises Chat Completions with:

- streaming and non-streaming;
- terminal stream usage;
- forced tool calls;
- complex JSON Schemas for tool arguments;
- `response_format`;
- `tool_choice`;
- thinking controls;
- prompt-token accounting;
- malformed inputs that must return 400-class errors.

These tests are useful sources of fixtures and invariants. They should be incorporated or invoked rather than independently reimplemented without need.

No production application client implementation for `/v1/responses` or raw `/v1/completions` was found in the named Forge, tau2, or KVV client harnesses. However, `/v1/completions` is an active repository dependency in `rollouts-disagg-set/loadgen.py`, where it drives rollout and KV-transfer canaries. `/v1/responses` is referenced by the LiteLLM flow-control path and now has router prefix-scoring work in production. Treat Completions as required by the rollout-test contract and Responses as an infrastructure compatibility surface even before an external application client is registered.

## Matrix execution model

A generated case has this identity:

```text
client contract × surface × scenario × transport × ingress
```

The supported ingresses are:

- `vllm_direct`;
- `router_gateway`, meaning the Envoy HTTP path that invokes EPP;
- `litellm`.

The same logical case runs against every ingress declared by the fleet profile. Tests compare normalized invariants, not byte-for-byte responses.

### Result states

Every generated cell must end in one of these states:

| State | Meaning |
| --- | --- |
| `PASS` | The declared contract was satisfied. |
| `FAIL` | The endpoint was expected to support the case and violated an assertion. |
| `EXPECTED_REJECTION` | The profile declares the feature unsupported and the endpoint rejected it with the expected status and error schema. |
| `NOT_EXPOSED` | The ingress intentionally does not expose the surface. |
| `NOT_APPLICABLE` | The combination has no protocol meaning, such as OpenAI `stream_options` on Anthropic Messages. |
| `BLOCKED` | A prerequisite outside the API contract prevented execution, such as unavailable credentials or endpoint DNS failure. |

There should be no unexplained `skip`. A missing profile declaration is a profile validation failure.

### Failure attribution

Run every ingress even if an earlier ingress fails. The comparison suggests the likely fault domain:

| Observation | Likely fault domain |
| --- | --- |
| Direct vLLM fails | model server, model template, or request fixture |
| Direct passes; router fails | gateway/EPP parsing, routing, mutation, or timeout |
| Direct and router pass; LiteLLM fails | LiteLLM mapping, transformation, authentication, or timeout |
| All pass individually but the real SDK fails | SDK compatibility, headers, event framing, or endpoint configuration |

This is attribution guidance, not proof. Preserve response evidence for diagnosis.

## Common assertions

Every successful JSON response must satisfy:

- HTTP status is the declared success status;
- media type is compatible with `application/json`;
- body is valid UTF-8 JSON;
- response validates against the surface-specific schema;
- returned model resolves to the profile's declared public/upstream mapping;
- generated identifiers are nonempty where required;
- all token counts are integers greater than or equal to zero;
- no credential, internal endpoint, or routing-only header appears in the body;
- latency stays within a generous functional timeout declared by the profile.

Every SSE response must satisfy:

- HTTP status is successful before stream parsing begins;
- media type is `text/event-stream`;
- each `data:` payload required to be JSON parses independently;
- events form a valid state machine for that surface;
- stream identifiers and model identity remain consistent where the protocol promises it;
- every choice or content-block index is internally consistent;
- UTF-8 characters survive chunk boundaries;
- the stream terminates according to the surface contract;
- accumulated output is structurally equivalent to the corresponding non-streaming response for the same fixture, allowing content variation;
- no silent truncation occurs before a finish/stop reason.

Do not assert exact natural-language output. Use structural invariants. Where a model decision would make a test nondeterministic, force the behavior through `tool_choice`, a constrained output schema, or a profile-specific fixture.

## Release-blocking P0 cases

### Discovery

#### DISC-001 — model list is usable

**Request:** `GET /v1/models`.

**Run against:** every ingress used by an OpenAI-compatible client.

**Assert:**

- status 200;
- JSON object is `list`;
- `data` is a nonempty array;
- every entry has a nonempty `id` and object `model`;
- the profile's public model is present or its alias mapping resolves unambiguously;
- duplicate IDs are rejected;
- the result can be parsed by the pinned OpenAI SDK.

Do not assert volatile `created`, ownership, or permission values unless a client depends on them.

#### DISC-002 — unknown model does not resolve accidentally

**Request:** a minimal generation request using a guaranteed-absent model name.

**Assert:**

- the request fails rather than falling through to the first available model;
- status and error envelope match the ingress-specific profile;
- no generation content is returned;
- the error identifies the model parameter or conveys “model not found” without requiring exact prose.

This case protects against aliases and gateway fallbacks masking a bad client configuration.

### OpenAI Chat Completions

#### CHAT-NS-001 — minimal non-streaming completion

**Request:** one short user message, `stream: false`, small output limit.

**Assert:**

- status 200 and object `chat.completion`;
- nonempty `id`, integer `created`, and declared model identity;
- exactly one choice for the default request;
- choice index 0;
- assistant message has a valid role and a content value allowed by the profile;
- finish reason is one of the profile's allowed OpenAI values;
- usage exists and satisfies:
  `total_tokens = prompt_tokens + completion_tokens`;
- output contains no streaming-only `delta`.

This is the smallest backend health and schema case.

#### CHAT-NS-002 — production message shapes

**Request:** profile fixtures covering:

- system plus user messages;
- multi-turn user/assistant history;
- Unicode text;
- list-form content blocks when the client uses them;
- an assistant tool call followed by a matching tool result when the model family supports it.

**Assert:**

- valid histories are accepted;
- order and role information are preserved sufficiently for a successful generation;
- Unicode is not corrupted;
- no valid client message type is rejected by an intermediate ingress.

Use this to catch chat-template and gateway body-transformation regressions that the minimal case misses.

#### CHAT-ST-001 — basic SSE stream

**Request:** the CHAT-NS-001 fixture with `stream: true`.

**Assert:**

- `text/event-stream`;
- every JSON payload has object `chat.completion.chunk`;
- response ID and model are stable across chunks when present;
- choice index is stable;
- deltas contain only valid incremental fields;
- an assistant role is observable according to the implementation's stream contract;
- content fragments concatenate without loss or duplication;
- exactly one terminal finish reason is observed per choice;
- the stream ends with `data: [DONE]`;
- no JSON payload appears after `[DONE]`.

Blank lines and legal SSE keepalives may occur and should not fail the test.

#### CHAT-ST-002 — streaming with terminal usage

**Request:** `stream: true` and `stream_options.include_usage: true`.

**Assert everything in CHAT-ST-001, plus:**

- one authoritative terminal usage record is emitted;
- its `choices` array is empty when following the vLLM/OpenAI stream form;
- `total_tokens = prompt_tokens + completion_tokens`;
- prompt and completion counts are nonnegative;
- no later event changes the final usage;
- `prompt_tokens_details.cached_tokens`, when present, is a nonnegative integer no greater than prompt tokens;
- the pinned OpenAI SDK exposes the usage record without a parsing error.

Do not require usage on every ordinary delta. vLLM intentionally places authoritative cache details on the terminal usage chunk.

#### CHAT-ST-003 — client disconnect

**Request:** start a stream that will produce enough output, read through the first meaningful delta, then close the connection.

**Assert using endpoint-only evidence:**

- the client can close without hanging;
- the connection closes within the case timeout;
- a new minimal request succeeds immediately afterward;
- the endpoint does not return a malformed partial error to the disconnected client.

Whether the original scheduler/KV state was reclaimed is a system assertion and must be checked by the companion system test.

#### CHAT-RSN-001 — declared reasoning mode

**Request:** a profile-specific prompt with the fleet's declared reasoning control, for example `chat_template_kwargs.enable_thinking`, default template kwargs, or `reasoning_effort`.

**Assert:**

- the request is accepted on every declared ingress;
- reasoning is exposed in the profile-declared field or intentionally hidden;
- content and reasoning fields have valid types;
- usage accounting includes the declared reasoning-token details when the server promises them;
- disabling reasoning uses the model-family-specific control and does not accidentally leave reasoning enabled.

This is configuration compatibility, not a reasoning-quality score. Do not assert the reasoning text.

#### CHAT-LIMIT-001 — output limit and finish reason

**Request:** force a very small output-token limit with a prompt that needs more output.

**Assert:**

- success rather than a server error;
- completion count does not exceed the applicable limit except for a documented tokenizer boundary;
- finish reason is the profile's length-limit value;
- usage arithmetic remains valid.

### OpenAI tool calls

#### TOOL-NS-001 — forced single tool call

**Request:** one simple function tool with a small JSON Schema and a specific forced `tool_choice`.

**Assert:**

- status 200;
- exactly one tool call;
- nonempty tool-call ID;
- type `function`;
- function name exactly matches the forced tool;
- arguments are a JSON string that parses into an object;
- parsed arguments validate against the supplied JSON Schema;
- finish reason is `tool_calls` or the explicitly declared equivalent;
- empty tool calls are not serialized as misleading empty arrays when no call exists.

Force the tool to separate protocol support from model willingness.

#### TOOL-ST-001 — streamed tool-call reassembly

**Request:** the TOOL-NS-001 fixture with streaming enabled.

**Assert:**

- tool-call index remains stable;
- ID and function name are not changed after first appearance;
- every argument fragment is a string;
- fragments concatenate in wire order into valid JSON;
- the resulting object validates against the tool schema;
- the terminal finish reason indicates tool use;
- any content emitted in the same chunk as the final argument fragment is retained;
- terminal usage and `[DONE]` follow the normal stream contract.

This protects the exact regression where the final tool-argument fragment can share a chunk with trailing content.

#### TOOL-LOOP-001 — tool result continuation

**Exchange:**

1. Force a tool call.
2. Append the assistant tool call to history.
3. Append the tool result with the exact returned tool-call ID.
4. Send a second Chat Completions request.

**Assert:**

- both requests succeed;
- the returned ID is accepted in the continuation;
- the second response produces a normal assistant result;
- the gateway does not remove, rename, or reorder tool-call fields;
- usage is valid for both calls.

This is the minimum agent-loop compatibility test required by tau2-style clients.

#### TOOL-SCHEMA-001 — production JSON Schema subset

Use representative schemas from the existing KVV/walle-valid fixtures, including:

- required properties;
- nested objects and arrays;
- enums;
- nullable or union forms actually sent by clients;
- property names containing non-ASCII or punctuation when present in real fixtures.

Run each selected schema in non-streaming and streaming modes. Validate returned arguments locally with a JSON Schema validator.

The smoke set should be small and deterministic. The full schema corpus belongs in a slower compatibility job.

#### TOOL-UNH-001 — invalid tool declarations

Send separate requests with:

- missing function name;
- non-object parameters;
- invalid JSON Schema;
- a forced tool name not present in `tools`;
- malformed prior tool result or unmatched tool-call ID when the surface validates it.

Assert a client-class error with the declared OpenAI error envelope. The request must not silently downgrade to unconstrained text generation unless the profile explicitly declares that behavior.

#### TOOL-PAR-001 — parallel tool calls

Run only when the model and profile declare parallel tools.

Assert unique tool-call IDs and indices, independently parseable argument streams, and no interleaving that assigns fragments to the wrong call. When parallel tools are unsupported, send the request once and assert the declared rejection or single-call behavior.

### Anthropic Messages

#### MSG-NS-001 — basic non-streaming message

**Request:** `POST /v1/messages` with the real client headers, a system prompt, one user message, and positive `max_tokens`.

**Assert:**

- status 200;
- response `type` is `message`;
- role is `assistant`;
- ID and model are nonempty and valid;
- `content` is an ordered array of valid Anthropic blocks;
- stop reason is one of `end_turn`, `max_tokens`, `stop_sequence`, or `tool_use`;
- usage contains nonnegative input and output tokens;
- the pinned Anthropic SDK parses the response.

#### MSG-ST-001 — Claude-compatible text stream

**Request:** MSG-NS-001 with `stream: true`.

**Assert the event state machine:**

1. first logical event is `message_start`;
2. `message_start.message.type` is `message`;
3. `message_start.message.role` is `assistant`;
4. each content block starts before receiving deltas;
5. text arrives through `content_block_delta` with `text_delta`;
6. every started block is stopped exactly once;
7. `message_delta` carries a valid stop reason and final usage;
8. final event is `message_stop`;
9. block indices are nonnegative, ordered, and internally consistent;
10. the pinned Anthropic SDK consumes the complete stream without schema errors.

This explicitly protects strict Claude Code clients that reject a `message_start` missing nested type or role.

#### MSG-ST-002 — thinking stream

Run only when the fleet declares Anthropic thinking support.

**Assert:**

- thinking content uses the declared thinking block and `thinking_delta`;
- signature deltas, when promised, belong to the correct block;
- ordinary text blocks still have valid ordering;
- display controls accepted but ignored by vLLM are documented as such in the profile;
- disabling thinking removes thinking blocks while leaving the request valid;
- a fixed thinking budget smaller than `max_tokens` is accepted.

Do not assert the thought text or a minimum reasoning length.

#### MSG-TOOL-001 — Anthropic tool use, non-streaming

**Request:** one forced Anthropic tool using `input_schema`.

**Assert:**

- one `tool_use` content block;
- nonempty block ID;
- exact tool name;
- `input` is a JSON object rather than a JSON-encoded string;
- input validates against the supplied schema;
- stop reason is `tool_use`;
- usage is valid.

#### MSG-TOOL-002 — Anthropic streamed tool use

**Assert:**

- a `tool_use` block starts before its argument deltas;
- `input_json_delta.partial_json` fragments concatenate into valid JSON;
- the final object validates against `input_schema`;
- a final argument fragment and text in the same upstream chunk are both preserved;
- block stops, `message_delta`, and `message_stop` are all emitted;
- the stop reason is `tool_use`.

#### MSG-TOOL-003 — Anthropic tool result continuation

Perform a two-request loop using a `tool_result` block whose `tool_use_id` matches MSG-TOOL-001/002. Assert that the continuation succeeds and yields a normal assistant message.

#### MSG-COUNT-001 — token counting

**Request:** `POST /v1/messages/count_tokens` with the same system, messages, and tools used by a generation fixture.

**Assert:**

- status 200;
- `input_tokens` is a nonnegative integer;
- adding a nonempty message does not reduce the count;
- the pinned Anthropic SDK parses the response if it exposes this method;
- an unknown model and malformed message receive the surface-specific error shape.

Exact token equality should be tested only against a versioned tokenizer fixture. A general compatibility smoke test should avoid assuming that different model revisions tokenize identically.

### Real header behavior

Header assertions require two views:

1. the client-visible response;
2. for headers expected to be consumed or stripped, an instrumented echo backend or equivalent test deployment.

An ordinary production vLLM response cannot prove that a control header was removed before reaching the backend.

#### HDR-REQID-001 — request correlation

Send a unique, valid `x-request-id`.

**Assert:**

- the request succeeds;
- the same value is returned only on ingresses configured to echo it;
- ingresses configured to rewrite it return a valid replacement and expose the original-to-replacement relationship if promised;
- two requests do not share IDs;
- an absent ID is generated only where declared;
- invalid or oversized IDs are rejected or replaced according to the profile.

The vLLM response header requires its request-ID-header feature to be enabled; this is not universal.

#### HDR-OBJ-001 — inference objective

Send a known objective used by the client, such as Forge's interactive objective.

**Assert at function level:**

- request succeeds through the router ingress;
- client-visible body and streaming framing are unchanged;
- the header is not reflected into the response unless declared.

**Assert with an instrumented backend or system access:**

- the router consumed the value;
- the header did not leak to the model server;
- the expected objective/profile was selected.

Also send an unknown objective. Silent priority-0 fallback should be represented explicitly as current behavior or, preferably, treated as a contract failure once the gateway is expected to reject unknown names.

#### HDR-FAIR-001 — fairness identity

Send two distinct fairness IDs and repeated requests with the same ID.

**Function assertions:**

- valid IDs do not change the API schema;
- values do not appear in response bodies;
- malformed/oversized values follow the declared error or normalization policy.

Fair scheduling itself requires metrics or routing observations and belongs in system tests.

#### HDR-AUTH-001 — authentication

For each exposed ingress, test:

- valid credential;
- missing credential;
- invalid credential;
- wrong credential style, for example bearer token versus `x-api-key`, only where meaningful.

Assert exact status class and surface-specific error envelope. Never store or print the credential. Direct vLLM may intentionally be unauthenticated inside a trusted network; the profile must declare that instead of marking the case skipped.

#### HDR-ANTH-001 — Anthropic client headers

For `/v1/messages`, send the observed client headers:

- `anthropic-version: 2023-06-01`;
- `x-api-key`;
- `Accept-Encoding: identity`;
- `Content-Type: application/json`.

Assert the request works through every declared ingress and the response is not compressed when identity is requested. Test missing or unsupported Anthropic-version behavior according to the ingress contract; the inspected vLLM route itself does not establish a strict version-header requirement.

#### HDR-RATE-001 — rate-limit response headers

Run only where the gateway promises `x-ratelimit-*` headers.

Assert:

- documented headers are present on successful and throttled responses as declared;
- numeric remaining values never become negative;
- reset values parse in the documented format;
- a 429 has the expected retry guidance and error envelope;
- headers do not contradict the status.

Generating a real quota breach may require a dedicated tenant or system fixture. If so, keep schema parsing in function tests and move enforcement to system tests.

#### HDR-TRACE-001 — trace-context acceptance

Send a valid `traceparent` and optional `tracestate`.

Assert the API response remains valid and any client-visible trace header follows the profile. Backend propagation or replacement requires an echo backend or telemetry and is not provable from the inference response alone.

### Response-field contracts

#### FIELD-USAGE-001 — non-streaming usage arithmetic

Run across every non-streaming generative surface that promises usage.

For OpenAI-style responses assert total equals prompt plus completion. For Anthropic assert nonnegative input and output counts; do not impose OpenAI field names.

#### FIELD-CACHE-001 — cache usage fields

Run a controlled pair of identical requests when the profile enables prompt-token details.

Assert:

- cache fields are absent when the feature is disabled;
- when present, values are nonnegative and bounded by input/prompt tokens;
- streaming OpenAI cache detail appears on the terminal usage chunk;
- streaming Anthropic `message_start` may omit cache fields and the final `message_delta` may carry authoritative values, matching the inspected vLLM behavior.

Do not require a cache hit from an arbitrary gateway request. A positive-hit assertion requires deterministic affinity and a warmed-cache system fixture.

#### FIELD-FINISH-001 — finish/stop reason mapping

Exercise at least:

- normal completion;
- forced length limit;
- tool call;
- explicit stop sequence.

Assert OpenAI finish reasons and Anthropic stop reasons use their surface-specific enums. LiteLLM normalization must be declared; it must not invent a tool-call finish reason without a tool call.

#### FIELD-LOGPROBS-001 — log probabilities

Run only when declared.

Assert requested logprobs are present, token entries are ordered with output tokens, numeric values are finite or use documented sentinel behavior, and `top_logprobs` respects the requested bound. For unsupported models, assert a deliberate rejection instead of silently omitting the field.

#### FIELD-N-001 — multiple choices

Run only on surfaces supporting `n > 1`.

Assert the number of choices, unique indices, terminal reason per choice, and aggregate usage semantics. Streaming reconstruction must keep fragments separated by choice index.

#### FIELD-STRUCT-001 — structured output

Send a small JSON Schema through the surface's supported field, such as `response_format` or `structured_outputs`.

Assert output parses as JSON and validates against the schema. Also send an impossible or invalid schema and assert a client-class failure. Removed legacy guided-decoding fields must not count as success merely because vLLM accepts unknown extra fields and ignores them.

### Client SDK smoke tests

Raw HTTP tests remain the source of truth because SDKs can normalize payloads. Add a small SDK lane to catch compatibility that raw JSON schemas miss.

#### SDK-OAI-001

Using a pinned OpenAI Python SDK:

- list models;
- make one non-streaming Chat Completions call;
- consume one Chat Completions stream with terminal usage;
- complete one forced tool call.

Assert no SDK validation or stream-decoding error.

#### SDK-ANTH-001

Using a pinned Anthropic SDK:

- make one non-streaming Messages call;
- consume one streaming Messages call;
- complete one forced tool-use response;
- call token counting when supported by the pinned SDK.

This case protects field presence and event ordering required by strict SDK models.

Record SDK name and version in every result. Upgrade SDK versions intentionally and run old/new versions in parallel before changing the compatibility floor.

## Conditional surfaces

### Legacy Completions

Enable only for a registered client.

- `COMP-NS-001`: non-streaming text completion, choice indices, finish reason, usage.
- `COMP-ST-001`: SSE deltas, terminal reason, `[DONE]`.
- `COMP-ST-002`: terminal usage with empty choices.
- `COMP-ERR-001`: malformed prompt, unknown model, and invalid token limits.

Do not infer Chat Completions behavior from this surface; request and response schemas differ.

### OpenAI Responses

Enable when a client profile requires it.

- `RESP-NS-001`: non-streaming response object, ordered output items, status, model, usage.
- `RESP-ST-001`: valid Responses event lifecycle ending in `response.completed`; every event type and sequence number satisfies the protocol.
- `RESP-TOOL-001`: declared tool-call representation and argument reconstruction.
- `RESP-GET-001`: retrieve a stored response only when stateful Responses is enabled.
- `RESP-CANCEL-001`: cancel an in-progress stored response only when enabled.
- `RESP-ERR-001`: unknown response ID, malformed input, unknown model, and unsupported stateful operation.

Do not reuse Chat Completions SSE assertions. Responses streams use named events such as `response.output_text.delta` and `response.completed`.

### Render endpoints

These are infrastructure compatibility cases, not ordinary external-client cases. Enable only when the deployment uses vLLM scale-out rendering.

- `RENDER-CHAT-001`: Chat Completions render accepts the client request shape and returns the declared rendered/tokens result.
- `RENDER-COMP-001`: Completions render.
- `RENDER-MSG-001`: native Anthropic Messages render.
- `RENDER-RESP-001`: Responses render.

For the router ingress, assert both HTTP success and parser disposition. At the inspected commits, vLLM exposes `/v1/responses/render` while the router OpenAI parser does not claim it. A 200 through passthrough is therefore not equivalent to parsed compatibility.

## Unhappy-path catalog

Each surface must define its own status and error schema. Assert stable fields such as status, error object type, and parameter/code. Avoid exact full error messages.

| Case | Input | Required assertion |
| --- | --- | --- |
| ERR-JSON-001 | truncated or syntactically invalid JSON | 4xx; valid surface error envelope; no generation |
| ERR-BODY-001 | JSON array/string instead of object | 4xx; no 500 |
| ERR-REQ-001 | missing model, messages/input/prompt, or Anthropic max_tokens | 4xx naming the invalid field or request |
| ERR-TYPE-001 | wrong field types | 4xx; no coercion that changes meaning |
| ERR-MODEL-001 | unknown model | declared 4xx; no fallback model |
| ERR-CONTEXT-001 | prompt exceeds declared context | declared 4xx; no partial generation; useful context-limit classification |
| ERR-LIMIT-001 | zero/negative or contradictory token limits | 4xx |
| ERR-OPTION-001 | unsupported option | declared rejection or explicitly documented ignore; never an unexplained 200 |
| ERR-TOOL-001 | invalid tool schema/choice/history | 4xx or declared surface behavior |
| ERR-AUTH-001 | missing/invalid credential | 401/403 as declared; no model details or secret echo |
| ERR-PATH-001 | unknown path and wrong method | 404/405; no passthrough to an unrelated model route |
| ERR-OBJECTIVE-001 | unknown inference objective | explicit declared behavior; preferred outcome is rejection when objective controls priority |
| ERR-HEADER-001 | malformed/oversized request ID or fairness ID | declared rejection/replacement; no 500 |
| ERR-PRESTREAM-001 | validation failure on a request with `stream: true` | error status before a successful SSE stream starts |
| ERR-RATE-001 | quota or admission limit | 429 when promised, valid error schema and retry headers |
| ERR-INTERNAL-001 | injected backend failure in a test deployment | 5xx envelope; no stack trace or internal address |

A failure that occurs after a stream has begun needs a surface-specific error event or clean termination policy. Producing this reliably requires a fault-injection backend and can be a later system-integrated function test.

## Minimum generated matrix

For the currently observed clients, the initial release gate should generate:

| Contract | Surface | Required cases | Required ingress |
| --- | --- | --- | --- |
| Forge/OpenClaw | Chat Completions | CHAT-NS-001, CHAT-NS-002, CHAT-ST-001/002/003, CHAT-RSN-001, TOOL-NS-001, TOOL-ST-001, TOOL-LOOP-001, HDR-REQID/OBJ/FAIR, core errors | router gateway and LiteLLM; vLLM direct for attribution |
| Claude Code-shaped Forge | Anthropic Messages | MSG-NS-001, MSG-ST-001, thinking when enabled, MSG-TOOL-001/002/003, MSG-COUNT-001, HDR-REQID/OBJ/FAIR/ANTH, core errors | router gateway; LiteLLM only if this surface is exposed; vLLM direct for attribution |
| tau2 | Models + Chat Completions | DISC-001, CHAT-NS-001, TOOL-NS-001, TOOL-LOOP-001, SDK-OAI-001 | every ingress offered to tau2 |
| KVV preflight | Chat Completions | streaming/non-streaming tools, TOOL-SCHEMA-001, FIELD-STRUCT-001, FIELD-USAGE-001, reasoning controls, malformed-request cases | candidate ingress used for evaluation |
| Rollout probe | Completions | COMP-NS-001, request ID, usage, exact output-limit behavior, revision response header, corrupted/mismatched-KV canary | router gateway used by `rollouts-disagg-set` |
| Router prefix scoring | Responses + render | RESP-NS/ST as enabled, RENDER-PREFIX-RESP-001, parser disposition, nonzero/shared-prefix scoring | direct vLLM and router gateway |

“Core errors” means ERR-JSON, ERR-REQ, ERR-TYPE, ERR-MODEL, ERR-CONTEXT, ERR-AUTH where authentication applies, and ERR-PRESTREAM.

## Profile additions

The fleet profile should declare client requirements directly:

```yaml
clients:
  forge_openclaw:
    required: true
    surfaces:
      chat_completions:
        ingresses: [router_gateway, litellm]
        transports: [non_stream, sse, sse_usage, disconnect]
        features: [tools, tool_loop, reasoning]
        fixture: forge_smoke
  forge_claude_code:
    required: true
    surfaces:
      messages:
        ingresses: [router_gateway]
        transports: [non_stream, sse, disconnect]
        features: [thinking, tools, tool_loop, count_tokens]
        fixture: claude_code_smoke

sdk_floor:
  openai: "<pinned-version>"
  anthropic: "<pinned-version>"

headers:
  x-request-id:
    router_gateway: echo
    litellm: rewrite
  x-llm-d-inference-objective:
    router_gateway:
      request: consume
      backend: strip
  x-llm-d-inference-fairness-id:
    router_gateway:
      request: consume
      backend: strip
  anthropic-version:
    messages:
      required_value: "2023-06-01"

errors:
  chat_completions:
    router_gateway:
      malformed_json:
        statuses: [400, 422]
        envelope: openai
      unknown_model:
        statuses: [404]
        envelope: openai
  messages:
    router_gateway:
      malformed_request:
        statuses: [400]
        envelope: anthropic
```

Prefer one expected status when the product contract is settled. A temporary set such as `[400, 422]` should carry an owner and removal date so the matrix does not normalize permanent drift.

## Fixtures

Maintain three fixture tiers:

1. **Synthetic minimal fixtures** for fast deterministic schema checks.
2. **Reduced production-shaped fixtures** preserving real role, tool, header, and option shapes without proprietary content.
3. **Recorded replay fixtures** for slower regression jobs, sanitized and versioned.

Each fixture records:

- source client and capture date;
- original request-shape hash;
- transformations and redactions;
- approximate prompt and tool-token sizes;
- model-family applicability;
- expected reasoning/tool behavior;
- whether prefix reuse is intentional.

Do not place captured credentials, webhook URLs, personal data, or proprietary prompt content in the test repository.

## Test isolation and repeatability

- Give every request a unique correlation ID.
- Use low output limits except in explicit long-output and disconnect cases.
- Use forced tool choices for protocol tests.
- Set deterministic sampling where the model supports it, while avoiding exact text assertions.
- Keep functional concurrency at one unless concurrency is part of the case.
- Separate cache-warm tests from ordinary cases.
- Do not retry protocol/schema failures. A transport retry, when enabled, must be reported as an attempt rather than hiding the first failure.
- Set per-case timeouts in profiles because long-context fleets may have multi-minute TTFT.
- Preserve response headers, status, event-type sequence, redacted body excerpts, and timing in failure artifacts.
- Redact authorization and API-key headers at collection time.

## Agreed implementation shape

The initial sync with Tyler established these implementation constraints:

- build a small Python package and CLI;
- run it from a laptop, CI job, or Kubernetes Job/Pod against supplied endpoints;
- keep scheduling outside the core runner;
- do not build a controller or CRDs for the first version;
- keep case definitions, transports, storage, and notifications modular so the suite can migrate toward llm-d-benchmark or llm-d Lens;
- always write a portable local artifact bundle;
- allow later result sinks for PVCs and object storage;
- preserve the exact rendered manifests or an immutable manifest reference with every run;
- provide a notifier interface so Slack or other alerts can be added without coupling them to case execution.

The concrete coordination and operational decisions are recorded in [[2026-09-30 - Initial functional-testing sync with Tyler]].

### Minimum Python package boundaries

```text
curvebender_functional_tests/
  cli.py
  profiles/
  cases/
    discovery.py
    chat_completions.py
    tools.py
    messages.py
    headers.py
    errors.py
  transports/
    http.py
    sse.py
    openai_sdk.py
    anthropic_sdk.py
  assertions/
  results/
    model.py
    local.py
  notifications/
    base.py
  metadata/
```

This is a responsibility map, not a required filename layout. The key constraint is that test semantics must not depend on Kubernetes, a PVC, Slack, or one scheduler.

### First runnable slice

The first CLI milestone should execute:

- DISC-001;
- CHAT-NS-001;
- CHAT-ST-001 and CHAT-ST-002;
- TOOL-NS-001;
- MSG-NS-001 and MSG-ST-001;
- HDR-REQID-001;
- ERR-JSON-001, ERR-REQ-001, ERR-MODEL-001, and ERR-PRESTREAM-001.

It should accept endpoint and credential overrides, select cases by client contract, emit JSONL plus the self-contained HTML matrix specified in issue 3, redact secrets, and exit nonzero when a required case fails. JUnit XML is a useful optional CI adapter, but it is not part of the repository's current issue-3 output contract.

This is the smallest first runnable slice within issue 3's broader first milestone. Issue 3 currently calls for all named surfaces plus a `core`-tagged subset; the core slice should land first without silently narrowing the milestone.

## Machine-readable report

Emit one record per generated cell:

```json
{
  "run_id": "2026-09-30T12:00:00Z-glm53",
  "case_id": "MSG-ST-001",
  "client_contract": "forge_claude_code",
  "fleet": "glm-5.3-scc-ib",
  "ingress": "router_gateway",
  "surface": "messages",
  "transport": "sse",
  "result": "PASS",
  "http_status": 200,
  "request_id": "cbtest-...",
  "event_types": [
    "message_start",
    "content_block_start",
    "content_block_delta",
    "content_block_stop",
    "message_delta",
    "message_stop"
  ],
  "component_versions": {
    "vllm": "<version>",
    "router": "<version>",
    "litellm": "<version>"
  },
  "sdk": {
    "name": "anthropic",
    "version": "<version>"
  },
  "duration_ms": 1432
}
```

Produce the JSONL artifact and self-contained HTML matrix required by issue 3, grouped by client, ingress, surface, and failure class. A JUnit XML exporter may be added for CI systems that consume it.

## Acceptance criteria for the first implementation

The functional compatibility suite is useful as a release gate when:

- the four observed client/evaluation contracts are represented by named profiles;
- Chat Completions and Messages P0 cases run through their real ingress paths and directly against vLLM for attribution;
- SSE parsers validate event state, not merely the presence of lines beginning with `data:`;
- streamed tool arguments are reconstructed and schema-validated;
- OpenAI and Anthropic errors are asserted separately;
- control headers have declared dispositions;
- real OpenAI and Anthropic SDK smoke tests pass at pinned versions;
- every matrix cell has an explicit result state;
- failures preserve enough redacted evidence to assign a likely layer;
- unsupported features fail in the declared way instead of disappearing as skips;
- the suite finishes quickly enough for a deployment gate, with full schema corpora and replay fixtures in separate slower jobs.

## Source record

- `client-harnesses/forge-breifing/loadgen.py`, `capture.py`, and `replay.mjs` in curvebender-tools.
- `agent-console/app/forge.py` and `agent-console/README.md` in curvebender-tools.
- `endpoint-validation/tau2/run_tau2_eval.py` and `endpoint-validation/kvv/vendor/README.md` in curvebender-tools.
- vLLM API routers and protocol models under `vllm/entrypoints/openai/`, `vllm/entrypoints/anthropic/`, and `vllm/entrypoints/scale_out/`.
- vLLM tests under `tests/entrypoints/openai/` and `tests/entrypoints/anthropic/`, including streamed tool-call and strict Anthropic event regressions.
- llm-d-router request handling under `pkg/epp/framework/plugins/requesthandling/parsers/` and header mutation tests under `pkg/epp/handlers/request_test.go`.
- Repository snapshots are recorded in [[2026-09-30 - Functional test strategy review]].