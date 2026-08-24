---
title: IBM VPC Load Balancer Timeout Controller
date: 2026-08-24
type: runbook
tags: [ocp, ibm-cloud, vpc, load-balancer, gateway, timeout, controller]
---

# IBM VPC Load Balancer Timeout Controller

## Problem

IBM Cloud VPC load balancers provisioned by the OpenShift Gateway API default to a 50-second idle connection timeout. For long-running inference workloads (vLLM, llm-d), this causes premature connection drops during generation. The `lb-timeout-controller` watches for new Gateway-created LB services and automatically updates the IBM VPC LB listener idle timeout via the IBM Cloud API.

## How It Works

1. The controller watches the Kubernetes API for `Service` resources of type `LoadBalancer` matching a specific `gateway-class-name` label.
2. When a new LB service appears (e.g., from a Gateway being created), it resolves the IBM VPC LB by matching the `.status.loadBalancer.ingress[].hostname` against the VPC API.
3. It updates every listener on that LB to the desired `idle_connection_timeout` (default: 2000 seconds).
4. It refreshes the IAM token every 30 minutes and handles watch reconnection on 410 Gone errors.

## Prerequisites

- An IBM Cloud API key with permissions to read and update VPC load balancers.
- `oc` or `kubectl` access with cluster-admin privileges.
- The target Gateway class must already exist (e.g., `openshift-ai-inference`).

## Deployment

### 1. Create the Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: lb-timeout-controller
```

```bash
oc apply -f namespace.yaml
```

### 2. Create the IBM Cloud API Key Secret

```bash
oc create secret generic ibmcloud-api-key \
  -n lb-timeout-controller \
  --from-literal=api-key='<YOUR_IBM_CLOUD_API_KEY>'
```

### 3. Create the ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: lb-timeout-controller
  namespace: lb-timeout-controller
```

### 4. Create RBAC (ClusterRole + ClusterRoleBinding)

The controller only needs read access to Services cluster-wide.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: lb-timeout-controller
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: lb-timeout-controller
subjects:
- kind: ServiceAccount
  name: lb-timeout-controller
  namespace: lb-timeout-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: lb-timeout-controller
```

### 5. Create the Controller Script ConfigMap

```bash
oc create configmap controller-script \
  -n lb-timeout-controller \
  --from-file=controller.py=controller.py
```

The controller script (`controller.py`):

```python
import json
import os
import ssl
import sys
import time
import urllib.request
import urllib.error

DESIRED_TIMEOUT = int(os.environ.get("LB_IDLE_TIMEOUT", "2000"))
IBM_API_KEY = open("/etc/ibmcloud/api-key").read().strip()
REGION = os.environ.get("IBM_REGION", "eu-de")
CLUSTER_ID = os.environ.get("CLUSTER_ID", "")
GATEWAY_CLASS = os.environ.get("GATEWAY_CLASS", "openshift-ai-inference")
VPC_API = f"https://{REGION}.private.iaas.cloud.ibm.com/v1"
IAM_URL = "https://private.iam.cloud.ibm.com/identity/token"
API_VERSION = "2024-06-01"

seen_hostnames = set()


def log(msg):
    print(f"[controller] {msg}", flush=True)


def get_iam_token():
    data = urllib.parse.urlencode({
        "grant_type": "urn:ibm:params:oauth:grant-type:apikey",
        "apikey": IBM_API_KEY,
    }).encode()
    req = urllib.request.Request(
        IAM_URL,
        data=data,
        headers={"Content-Type": "application/x-www-form-urlencoded"},
    )
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())["access_token"]


def vpc_request(method, path, token, body=None):
    sep = "&" if "?" in path else "?"
    url = f"{VPC_API}{path}{sep}version={API_VERSION}&generation=2"
    data = json.dumps(body).encode() if body else None
    req = urllib.request.Request(url, data=data, method=method, headers={
        "Authorization": f"Bearer {token}",
        "Content-Type": "application/json",
    })
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read())


def wait_lb_active(lb_id, token, max_wait=300):
    for _ in range(max_wait // 5):
        lb = vpc_request("GET", f"/load_balancers/{lb_id}", token)
        if lb["provisioning_status"] == "active":
            return True
        time.sleep(5)
    return False


def enforce_timeout_on_lb(hostname, token):
    lbs = vpc_request("GET", "/load_balancers", token)
    lb = next((lb for lb in lbs["load_balancers"]
               if lb.get("hostname") == hostname
               and (not CLUSTER_ID or CLUSTER_ID in lb.get("name", ""))), None)
    if not lb:
        log(f"  LB not found for hostname {hostname}")
        return

    lb_id = lb["id"]
    lb_name = lb["name"]

    if not wait_lb_active(lb_id, token):
        log(f"  LB {lb_name} did not become active, skipping")
        return

    listeners = vpc_request("GET", f"/load_balancers/{lb_id}/listeners", token)
    for listener in listeners["listeners"]:
        lid = listener["id"]
        port = listener["port"]
        current = listener.get("idle_connection_timeout", 50)
        if current != DESIRED_TIMEOUT:
            log(f"  Updating {lb_name} listener port={port} ({lid}): {current}s -> {DESIRED_TIMEOUT}s")
            wait_lb_active(lb_id, token)
            vpc_request("PATCH", f"/load_balancers/{lb_id}/listeners/{lid}", token,
                        {"idle_connection_timeout": DESIRED_TIMEOUT})
        else:
            log(f"  {lb_name} listener port={port} already {DESIRED_TIMEOUT}s")


def watch_services():
    log(f"Starting LB timeout controller (target={DESIRED_TIMEOUT}s, region={REGION}, class={GATEWAY_CLASS})")
    token = get_iam_token()
    token_time = time.time()

    ctx = ssl.create_default_context()
    ca_path = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
    if os.path.exists(ca_path):
        ctx.load_verify_locations(ca_path)

    sa_token = open("/var/run/secrets/kubernetes.io/serviceaccount/token").read().strip()
    api_host = os.environ.get("KUBERNETES_SERVICE_HOST", "kubernetes.default.svc")
    api_port = os.environ.get("KUBERNETES_SERVICE_PORT", "443")
    base = f"https://{api_host}:{api_port}"

    resource_version = ""
    while True:
        if time.time() - token_time > 1800:
            token = get_iam_token()
            token_time = time.time()
            log("Refreshed IAM token")

        url = f"{base}/api/v1/services?watch=true&timeoutSeconds=300"
        if resource_version:
            url += f"&resourceVersion={resource_version}"

        req = urllib.request.Request(url, headers={
            "Authorization": f"Bearer {sa_token}",
        })
        try:
            with urllib.request.urlopen(req, context=ctx) as resp:
                for line in resp:
                    if not line.strip():
                        continue
                    event = json.loads(line)
                    event_type = event.get("type")

                    if event_type == "ERROR":
                        status = event.get("object", {})
                        if status.get("code") == 410:
                            log("Watch expired (410 Gone), restarting")
                            resource_version = ""
                            break
                        continue

                    obj = event.get("object", {})
                    meta = obj.get("metadata", {})
                    resource_version = meta.get("resourceVersion", resource_version)
                    spec = obj.get("spec", {})
                    status = obj.get("status", {})

                    if spec.get("type") != "LoadBalancer":
                        continue
                    if event_type not in ("ADDED", "MODIFIED"):
                        continue

                    labels = meta.get("labels", {})
                    if labels.get("gateway.networking.k8s.io/gateway-class-name") != GATEWAY_CLASS:
                        continue

                    ingress = status.get("loadBalancer", {}).get("ingress", [])
                    if not ingress:
                        continue

                    hostname = ingress[0].get("hostname", "")
                    if not hostname or not hostname.endswith(".lb.appdomain.cloud"):
                        continue

                    svc_name = f"{meta.get('namespace')}/{meta.get('name')}"
                    cache_key = f"{svc_name}:{hostname}"
                    if cache_key in seen_hostnames:
                        continue

                    seen_hostnames.add(cache_key)
                    log(f"New LB detected: {svc_name} -> {hostname}")
                    try:
                        enforce_timeout_on_lb(hostname, token)
                    except Exception as e:
                        log(f"  Error enforcing timeout: {e}")
                        seen_hostnames.discard(cache_key)

        except urllib.error.HTTPError as e:
            if e.code == 410:
                log("Watch expired (410), restarting")
                resource_version = ""
            else:
                log(f"HTTP error {e.code}: {e.reason}, retrying in 10s")
                time.sleep(10)
        except Exception as e:
            log(f"Watch error: {e}, retrying in 10s")
            time.sleep(10)


if __name__ == "__main__":
    watch_services()
```

### 6. Create the Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: lb-timeout-controller
  namespace: lb-timeout-controller
  labels:
    app: lb-timeout-controller
spec:
  replicas: 1
  selector:
    matchLabels:
      app: lb-timeout-controller
  template:
    metadata:
      labels:
        app: lb-timeout-controller
    spec:
      serviceAccountName: lb-timeout-controller
      containers:
      - name: controller
        image: registry.access.redhat.com/ubi9/python-311:latest
        command: ["python3", "/app/controller.py"]
        env:
        - name: LB_IDLE_TIMEOUT
          value: "2000"
        - name: IBM_REGION
          value: "eu-de"
        - name: CLUSTER_ID
          value: "<YOUR_CLUSTER_ID>"
        - name: GATEWAY_CLASS
          value: "openshift-ai-inference"
        resources:
          requests:
            cpu: 10m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 64Mi
        volumeMounts:
        - name: script
          mountPath: /app
          readOnly: true
        - name: ibmcloud-api-key
          mountPath: /etc/ibmcloud
          readOnly: true
      volumes:
      - name: script
        configMap:
          name: controller-script
      - name: ibmcloud-api-key
        secret:
          secretName: ibmcloud-api-key
```

### Quick Apply (All-in-One)

```bash
# 1. Namespace + SA + RBAC
oc apply -f namespace.yaml
oc apply -f sa.yaml
oc apply -f rbac.yaml

# 2. Secret (replace with your actual API key)
oc create secret generic ibmcloud-api-key \
  -n lb-timeout-controller \
  --from-literal=api-key='<YOUR_IBM_CLOUD_API_KEY>'

# 3. Controller script
oc create configmap controller-script \
  -n lb-timeout-controller \
  --from-file=controller.py=controller.py

# 4. Deployment (edit env vars for your cluster)
oc apply -f deployment.yaml
```

## Configuration

| Environment Variable | Default | Description |
|---|---|---|
| `LB_IDLE_TIMEOUT` | `2000` | Desired idle connection timeout in seconds |
| `IBM_REGION` | `eu-de` | IBM Cloud VPC region |
| `CLUSTER_ID` | `""` | Cluster ID substring to match LB names (prevents cross-cluster collisions) |
| `GATEWAY_CLASS` | `openshift-ai-inference` | Gateway class name to filter services by |

The `CLUSTER_ID` is used to scope LB matching — the controller only updates LBs whose VPC name contains the cluster ID. Find it with:

```bash
oc get infrastructure cluster -o jsonpath='{.status.infrastructureName}'
```

## Verification

Check that the controller is running and processing gateways:

```bash
# Pod status
oc get pods -n lb-timeout-controller

# Logs — look for "New LB detected" and "Updating ... listener" messages
oc logs deployment/lb-timeout-controller -n lb-timeout-controller --tail=30

# Verify a specific gateway's LB timeout (from the IBM Cloud CLI)
ibmcloud is load-balancers --output json | \
  jq '.[] | select(.name | contains("<CLUSTER_ID>")) | {name, id}'
ibmcloud is load-balancer-listeners <LB_ID> --output json | \
  jq '.[].idle_connection_timeout'
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `LB not found for hostname` | LB not yet provisioned in VPC or `CLUSTER_ID` mismatch | Verify the hostname resolves in `ibmcloud is lbs`; check `CLUSTER_ID` env var |
| `LB did not become active, skipping` | LB stuck in provisioning (another update in flight) | Wait for LB to settle; check IBM Cloud console for provisioning state |
| `HTTP error 401` on IAM refresh | Expired or invalid API key | Rotate the key in the `ibmcloud-api-key` secret and restart the pod |
| `Watch expired (410 Gone)` | Normal — Kubernetes watch bookmark expired | No action needed; the controller reconnects automatically |
| Controller not picking up gateways | Service label `gateway.networking.k8s.io/gateway-class-name` doesn't match `GATEWAY_CLASS` | Verify the gateway class name matches the env var |

## Limitations

- The controller uses an in-memory `seen_hostnames` set, so it only processes each LB once per pod lifetime. If a timeout is manually changed back, restart the pod to re-enforce.
- Only targets LBs with hostnames ending in `.lb.appdomain.cloud` (IBM Cloud VPC).
- Single-replica; no leader election. Running multiple replicas may cause duplicate VPC API calls (harmless but wasteful).