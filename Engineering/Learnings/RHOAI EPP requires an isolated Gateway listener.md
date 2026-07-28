# RHOAI EPP requires an isolated Gateway listener

## Rule

For RHOAI distributed inference using the EndpointPicker, do not attach unrelated HTTPRoutes to the same wildcard Gateway listener as an `LLMInferenceService` route.

## Why

RHOAI known issue `INFERENG-6962` documents that Istio aggregates multiple HTTPRoutes on one wildcard listener into one generated VirtualService. The generated configuration omits the per-route `ExtProcPerRoute` override, so EndpointPicker is bypassed and requests fall back to round-robin routing.

The issue applies to any additional HTTPRoute, including a token endpoint, echo service, test route, or another LLMInferenceService route.

## BenchFlow impact

BenchFlow's current RHOAI template leaves `router.gateway: {}` and `router.route: {}`, which makes RHOAI use the shared `openshift-ai-inference` Gateway. This is correct only while that listener has one LLMInferenceService route and no unrelated routes.

## Recommended design

- Keep one dedicated inference Gateway/listener for an isolated EPP route.
- Move non-LLM routes to another Gateway.
- For concurrent EPP-backed LLMInferenceServices, use explicit Gateway references with distinct listener hostnames rather than multiple Gateways sharing the same wildcard hostname.
- Avoid a generic one-wildcard-Gateway-per-route scheme: it creates conflicting exposure/ownership and is not Red Hat's prescribed workaround.
- Verify the exact RHOAI/Istio version before deciding whether an upstream Istio 1.29+ fix removes the limitation.

## Sources

- [RHOAI 3.3 known issue INFERENG-6962](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.3/html/release_notes/known-issues_relnotes)
- [RHOAI gateway discovery supports explicit Gateway selection](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/deploy_models_using_distributed_inference_with_llm-d/gateway-discovery-for-llmd_gateway-discovery)