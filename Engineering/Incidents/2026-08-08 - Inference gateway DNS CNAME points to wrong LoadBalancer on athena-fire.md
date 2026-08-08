---
title: Inference gateway DNS CNAME points to wrong LoadBalancer on athena-fire
date: 2026-08-08
type: incident
cluster: psap-fire-athena
---

# 2026-08-08 — Inference gateway DNS CNAME points to wrong LoadBalancer on athena-fire

## Symptom

Smoke request curl to `https://inference-gateway.apps.psap-fire-athena.ibm.rhperfscale.org/...` times out with `curl: (28) Connection timed out after 60002 milliseconds`. All vLLM pods and the router-scheduler are Running/Ready, LLMInferenceService shows `Ready: True`, HTTPRoute shows `Accepted: True`.

DNS resolution for `inference-gateway.apps.psap-fire-athena.ibm.rhperfscale.org` returns a CNAME to `a6d99e97-eu-de.lb.appdomain.cloud`, which is an HTTP-only (port 80) LoadBalancer. The correct HTTPS (port 443) LoadBalancer is `fede6214-eu-de.lb.appdomain.cloud`.

## Environment

- **Cluster:** psap-fire-athena (IBM Cloud VPC, eu-de)
- **Component & version:** RHOAI 3.5.0, OpenShift Gateway API, Istio/OSSM (istiod v1.28.5)
- **When it started:** 2026-08-05, when the `openshift-ai-inference` Gateway was created before its GatewayClass existed
- **Kubeconfig:** `/Users/aperdomo/workspace/redhat/clusters/psap-h200-fire-athena/auth/kubeconfig`

## Timeline

1. **Aug 5, 20:25 UTC** — `openshift-ai-inference` Gateway created with `gatewayClassName: openshift-ai-inference`. That GatewayClass did **not exist yet**. The Istio gateway controller (`openshift.io/gateway-controller/v1`) reconciled using the only available class it manages: `data-science-gateway-class`. Created deployment `openshift-ai-inference-data-science-gateway-class`, LB service (hostname `a6d99e97`, ports 15021 + **80**), and envoy proxy pod.
2. **Aug 7, 14:17 UTC** — `openshift-ai-inference` GatewayClass created via `kubectl apply`. Controller reconciled again and created the **correct** deployment `openshift-ai-inference-openshift-ai-inference`, LB service (hostname `fede6214`, ports 15021 + **443**), and envoy pod. But the orphaned resources from step 1 were **never cleaned up**. The ingress operator's `service_dns_controller` created a `DNSRecord` owned by the **old** service, pointing `inference-gateway.apps.*` → `a6d99e97` (the HTTP-only LB).
3. **Aug 8** — Forge smoke request curls HTTPS to `inference-gateway.apps.*`, DNS resolves to `a6d99e97` (port 80 only), HTTPS on port 443 times out.

## Root Cause

The `openshift-ai-inference` Gateway was created ~2 days before its matching GatewayClass. During that window, the Istio gateway controller created resources tagged with the wrong gateway class (`data-science-gateway-class`). When the correct class was later created, the controller provisioned correct resources but did not garbage-collect the orphaned set. The ingress operator's `service_dns_controller` created the `DNSRecord` based on the **first** LB service it saw (the orphaned HTTP/80 one), pointing the hostname to a LoadBalancer that doesn't serve port 443.

Two LB services existed for the same gateway:

| Service | LB hostname | Ports | Gateway class label | DNS record? |
|---|---|---|---|---|
| `openshift-ai-inference-data-science-gateway-class` | `a6d99e97` | 15021, **80** | `data-science-gateway-class` | Yes → wrong target |
| `openshift-ai-inference-openshift-ai-inference` | `fede6214` | 15021, **443** | `openshift-ai-inference` | No |

The `data-science-gateway-class` GatewayClass is legitimately used by RHOAI for the internal `data-science-gateway` Gateway (ClusterIP). The orphaned resources were a side effect of creating the inference gateway before its class — not a problem with the `data-science-gateway-class` itself.

## Resolution

1. Deleted the orphaned deployment:
   ```bash
   oc delete deployment openshift-ai-inference-data-science-gateway-class -n openshift-ingress
   ```
2. Deleted the orphaned LB service (the `DNSRecord` was garbage-collected via ownerReference):
   ```bash
   oc delete svc openshift-ai-inference-data-science-gateway-class -n openshift-ingress
   ```
3. The `service_dns_controller` did **not** auto-create a new `DNSRecord` for the remaining service (it only watches services labeled `gateway-class-name: data-science-gateway-class`). Created one manually:
   ```bash
   cat <<'EOF' | oc apply -f -
   apiVersion: ingress.operator.openshift.io/v1
   kind: DNSRecord
   metadata:
     name: openshift-ai-inference-wildcard
     namespace: openshift-ingress
     labels:
       gateway.istio.io/managed: openshift.io-gateway-controller-v1
       gateway.networking.k8s.io/gateway-class-name: openshift-ai-inference
       gateway.networking.k8s.io/gateway-name: openshift-ai-inference
     ownerReferences:
     - apiVersion: v1
       kind: Service
       name: openshift-ai-inference-openshift-ai-inference
       uid: <service-uid>
   spec:
     dnsManagementPolicy: Managed
     dnsName: inference-gateway.apps.psap-fire-athena.ibm.rhperfscale.org.
     recordTTL: 30
     recordType: CNAME
     targets:
     - fede6214-eu-de.lb.appdomain.cloud
   EOF
   ```
4. Verified DNS resolves to `fede6214`, gateway status `DNSReady: True`, and `curl -k https://inference-gateway.apps.psap-fire-athena.ibm.rhperfscale.org/` returns HTTP 404 (envoy responding, no route matched for `/`).

## Prevention / Runbook

- **Always create the GatewayClass before the Gateway.** The Istio controller will use any available class with a matching controller name as a fallback, creating orphaned resources that are never cleaned up.
- When deploying a new inference gateway on an RHOAI cluster, verify that only one set of resources (deployment, service, pod) exists per gateway:
  ```bash
  oc get deploy,svc -n openshift-ingress -l gateway.networking.k8s.io/gateway-name=openshift-ai-inference
  ```
  There should be exactly one deployment and one LB service with matching `gateway-class-name` labels.
- If the `service_dns_controller` doesn't pick up a new gateway class's service for DNS, a manual `DNSRecord` is required (see Resolution step 3).

## Related

- [[2026-08-06 - RHOAI DSC NotReady dual controller ownerRef on odh-model-controller SA]] — same cluster, same RHOAI 3.5 deployment
- CIS DNS zone: `71f0a89ed7064ab78f7f5dc69357fc3d` (domain `ibm.rhperfscale.org`), managed via CIS instance `4b377dba-7732-43f6-a17a-d67636818f7f` (`Internet Services-vd`)