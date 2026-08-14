---
title: IBM Cloud VPC LB idle timeout on OCP IPI clusters
date: 2026-08-14
type: learning
platform: IBM Cloud VPC
cluster: OCP IPI (installer-provisioned)
---

# IBM Cloud VPC LB idle timeout on OCP IPI clusters

## The problem

IBM Cloud VPC load balancers default to a 50-second idle connection timeout on all listeners. For inference workloads behind `openshift-ai-inference` Gateways, this is too short — long-running requests (large prompts, streaming) get dropped.

## What does NOT work

### 1. Cloud-conf ConfigMap — no cluster-wide default parameter

There is no `g2IdleConnectionTimeout` (or equivalent) in the CCM's `cloud-conf` ConfigMap (`openshift-config/cloud-provider-config` → synced to `openshift-cloud-controller-manager/cloud-conf`). The IBM VPC CCM does not support a cluster-wide default for LB idle timeout.

### 2. Service annotation — ignored by the OCP-bundled CCM

The documented annotation:

```
service.kubernetes.io/ibm-load-balancer-cloud-provider-vpc-idle-connection-timeout: "600"
```

is valid on **managed ROKS/IKS clusters** but the OCP-bundled CCM (`quay.io/openshift-release-dev/ocp-v4.0-art-dev`) on IPI clusters does not process it. The CCM logs show only TLS cert rotation — no load balancer reconciliation activity. Verified on OCP 4.22.

### 3. MutatingAdmissionWebhook (annotation injection)

A webhook that injects the annotation on new `LoadBalancer` services is ineffective because the CCM ignores it (see above).

## What works

### Direct VPC API listener updates via a controller

A lightweight Python controller deployed as a Deployment that:

1. **Watches** Kubernetes Service events via the watch API
2. **Filters** by label `gateway.networking.k8s.io/gateway-class-name=openshift-ai-inference`
3. **Waits** for the VPC LB to reach `active` provisioning status
4. **Updates every listener** on the LB via the IBM Cloud VPC API (`PATCH /v1/load_balancers/{id}/listeners/{id}` with `idle_connection_timeout: 600`)

### Key implementation details

- **Must use private VPC API endpoint** (`{region}.private.iaas.cloud.ibm.com`) — the public endpoint (`{region}.iaas.cloud.ibm.com`) returns Cloudflare 1010 (Access Denied) from inside the VPC.
- **Private IAM endpoint** (`private.iam.cloud.ibm.com`) also available but public IAM works from inside VPC too.
- **Sequential listener updates required** — the VPC LB goes into `update_pending` after each listener update. Must wait for `active` before updating the next listener, otherwise the API returns `load_balancer_update_conflict`.
- **IBM Cloud API key** stored as a Kubernetes Secret, mounted into the controller pod.
- **OpenShift RBAC** — the controller's ServiceAccount needs only `get/list/watch` on Services (cluster-scoped).
- **Resource group filtering** — filter LBs by cluster ID in the LB name (e.g. `diadochos-hqxzk`) to avoid touching LBs from other clusters sharing the same account.

### Deployed location

Namespace `lb-timeout-controller` on the `diadochos` cluster (`eu-de`).

## Timeline

- Annotation approach tested and confirmed ineffective on OCP 4.22 IPI.
- MutatingWebhook deployed, tested, removed — annotation injected correctly but CCM ignored it.
- Controller approach deployed and verified end-to-end: new Gateway → LB provisioned → all listeners updated to 600s → confirmed via `ibmcloud is lb-ls`.