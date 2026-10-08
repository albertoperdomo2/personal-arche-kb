---
title: "2026-10-08 — diadochos API unresponsive: undersized masters overloaded by ~1,200 experiment pods, etcd quorum lost"
date: 2026-10-08
type: incident
status: resolved
cluster: diadochos
platform: IBM Cloud VPC (eu-de)
ocp_version: 4.22.2
---

# 2026-10-08 — diadochos API unresponsive, etcd quorum repeatedly lost (control plane overload + master /var full)

> **Status: RESOLVED at ~14:30 UTC — all 7 nodes Ready, etcd 3/3, zero degraded cluster operators, 450 pods (down from 1,709). No H100 node was lost or touched.**

## Symptom
Every `oc` command against `https://api.diadochos.ibm.rhperfscale.org:6443` hangs or fails. Seen in this order of frequency:

```
Unable to connect to the server: context deadline exceeded (Client.Timeout exceeded while awaiting headers)
Unable to connect to the server: net/http: TLS handshake timeout
Unable to connect to the server: EOF
Error from server (InternalError): ... internal server error: error getting data from etcd: context deadline exceeded
Error from server (TooManyRequests): storage is (re)initializing
```

- `/readyz` can return `200 ok` and discovery (`/api`, `/apis`) can answer while every real read hangs — check `oc get --raw '/livez/etcd'` instead.
- On the masters: `no space left on device` in the kubelet journal, `/var` at 100% (first outage).
- A master stops answering ping/SSH (`Connection timed out during banner exchange`) while IBM Cloud still shows it `running` / `ok`; it may come back on its own tens of minutes later with the same uptime (it was thrashing, not dead).

## Environment
- Cluster: **diadochos** (`diadochos-hqxzk`), OCP 4.22.2 on IBM Cloud VPC, region `eu-de`, Kubernetes `v1.35.5`
- Control plane: `master-0` 10.243.0.6 (eu-de-1), `master-1` 10.243.64.4 (eu-de-2), `master-2` 10.243.128.6 (eu-de-3) — all **`bx3d-4x20` (4 vCPU / 20 GB, no swap)**, boot volume 100 GB `general-purpose` 3000 IOPS, `/var` on `/dev/vda4`
- Workers: `worker-1-8p9vb` (bx3d-4x20), H100 nodes `gpu-h100-gjfjh`, `gpu-h100-lljhv`, `gpu-h100-mt46x` (`gx3d-160x1792x8h100`, eu-de-2)
- Leftover install bootstrap VM `diadochos-hqxzk-bootstrap` (floating IP 149.81.35.27) — the only way in when the API is down: `ssh -J core@149.81.35.27 core@<master-ip>`
- etcd members: master-0 `20287613eecee58f`, master-1 `193dfb599da5651f`, master-2 `3af535b375996a3b`; data dir 1.7 GB
- MachineHealthChecks: only `machine-api-termination-handler` (0 machines). No MHC covers the H100 MachineSet.

## Timeline (UTC)
- **2026-10-04 ~22:30** — All masters rebooted (reason not investigated).
- **2026-10-05 00:27** — `gpu-h100-lljhv` instance created (H100 machine replaced; reason not investigated).
- **2026-10-07 09:05–12:37** — `aharush` experiments launched on the H100 nodes: 500 Jobs, 316 Deployments, 316 Services, 1,984 ConfigMaps, 925 pods in `trace-replay`; 292 Sandboxes / 294 pods in `openshell-tracesim`. Cluster total reaches ~1,700 pods (759 on `gjfjh`, 604 on `lljhv`).
- **~10:00–13:00** — Pods on H100 nodes start sticking in `Init`.
- **14:56** — `openshift-console/downloads` on master-2 starts restart-looping, leaking ~3.1 GB per restart.
- **17:35** — master-1 kubelet stops posting status; unreachable since. etcd on 2/3.
- **2026-10-08 05:08** — master-2 `/var` 100% full, its etcd exits. **Quorum lost (outage 1).**
- **06:55** — Investigation begins.
- **07:34** — master-2 leaked dirs removed (`/var` 100% → 27%); etcd restarted by kubelet. **07:35:56 API back.**
- **07:41** — master-0 leaked dirs removed (`/var` 85% → 35%).
- **07:45–08:05** — Reconnect storm. Load average >50 on both masters; kube-apiserver 7.2 GB RSS on master-2; kube-apiserver on master-0 restarts repeatedly.
- **~08:10** — master-2 stops answering on the network. **Quorum lost (outage 2).** master-0 healthy but alone.
- **~09:40** — master-2 answers again without any reboot (uptime unchanged). API back by 09:47; kube-apiserver on master-2 at 9.5 GB RSS, 385 MB free.
- **09:50** — Deletion of `trace-replay` Jobs and agent Deployments starts (user-approved).
- **~10:00** — master-0 stalls (load average 94). API down again; returns in short windows.
- **10:07–10:32** — In the windows: all 500 Jobs deleted, all 300 agent Deployments deleted.
- **~10:22–10:31** — master-2 stalls again; master-0 recovered. API down. Deletion loop still retrying.
- **10:32** — All trace-replay Jobs and agent Deployments confirmed deleted.
- **~11:06** — User reboots master-1 via IBM Cloud (`ibmcloud is instance-reboot diadochos-hqxzk-master-1 -f`).
- **11:20** — All 7 nodes show Ready; master-1 back. 335 trace-replay pods still Terminating.
- **12:40** — Kueue mutating and validating webhooks patched from `failurePolicy=Fail` to `Ignore` (39 webhooks total). Kueue controller had been crash-looping (173 restarts) with no ready endpoints, blocking all Deployment/Pod/StatefulSet/Job mutations cluster-wide — preventing ~12 cluster operators from reconciling.
- **12:50** — Remaining 16 trace-replay backend deployments deleted. 292 openshell-tracesim Sandbox CRs deleted (`agents.x-k8s.io/v1beta1`). Remaining configmaps cleaned up.
- **13:10** — master-2 rebooted via IBM Cloud (`ibmcloud is instance-reboot diadochos-hqxzk-master-2 -f`) — it had gone `Unknown` (same overload-stall pattern as master-1).
- **~13:25** — master-2 back. All 7 nodes Ready, etcd 3/3 healthy.
- **~13:35** — Degraded operators down to authentication, etcd, storage — all failing on DNS resolution ("server misbehaving" from 172.30.0.10:53).
- **13:40** — Root-caused DNS failure: `br-ex` OVS bridge on master-0 had MTU 1500 (should be 9000 to match `ens3` jumbo frames). Pod-to-upstream-DNS traffic was silently dropped. Fixed via `sudo ovs-vsctl set interface br-ex mtu_request=9000`. OVN node pod restarted and reached 8/8 Ready.
- **~14:00** — DNS working from all pods. All cluster operators recovered. Zero degraded.
- **~14:05** — **All clear: 7/7 nodes Ready, etcd 3/3, 0 degraded COs, 450 pods.** Masters at load 0.75–3.63, disk 32–69%.
- **~14:20** — etcd backup removed from master-2 (1.6 GB freed, disk 29%).
- **~14:25** — Kueue controller confirmed stable (1/1 Ready, webhook endpoints ready). Webhooks restored to `failurePolicy=Fail`.
- **~14:30** — br-ex MTU drift root-caused: NetworkManager profile `ovs-if-br-ex` on master-0 had `802-3-ethernet.mtu: 1500`. Fixed with `nmcli connection modify ovs-if-br-ex 802-3-ethernet.mtu 9000`. All 7 nodes verified at MTU 9000. **Incident closed.**

## Evidence

First outage, master state at 07:24:

| Master | `/var` | etcd | Notes |
| --- | --- | --- | --- |
| master-0 | 84% | Running, no quorum | `waiting for ReadIndex response took too long` |
| master-1 | unknown | unknown | 100% ping loss, all ports time out; IBM `running` / `ok` |
| master-2 | **100% (20 K free)** | **Exited** | kubelet/CRI-O `no space left on device` |

Disk consumers: `/var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~empty-dir/tmp/tmp*` of the `openshift-console/downloads` pods — 27 dirs / 72 GB on master-2 (pod UID `ee550be7-ba1e-40ac-a16e-b3c8510c253f`), 18 dirs / 52 GB on master-0 (UID `0d7d90a6-a3d3-4426-b3d0-6282fd6c6ace`). Each dir holds the `oc` CLI archives built at container start.

Overload evidence (08:00–10:22):

| Signal | Value |
| --- | --- |
| Pods cluster-wide | 1,709 (925 `trace-replay`, 294 `openshell-tracesim`) |
| ConfigMaps | 3,463 total, 1,984 in `trace-replay` |
| Master load average (4 vCPU) | 53 on master-2 at 08:05; 94 on master-0 at 10:07 |
| kube-apiserver RSS | 7.2 GB at 08:05, 9.5 GB at 09:47 (node has 20 GB) |
| Open watches | 519 configmaps, 454 secrets |
| Requests since apiserver start | 272,202 × `configmaps WATCH 429`, 105,068 × `secrets WATCH 429` |
| Busiest API priority flows | `system-nodes` (kubelets) and `service-accounts` |
| etcd on master-0 | `slow fdatasync` up to 1m24s; `leader is overloaded likely from slow disk` |
| `oc adm top nodes` at 07:57 | master-2 memory 91%; H100 nodes 1–3% CPU, 3–6% memory |

Kueue webhook blocker (discovered at 12:40):

| Metric | Value |
| --- | --- |
| Kueue controller restarts | 173 in 15 hours (readiness probe 404, liveness probe connection refused) |
| Webhooks with `failurePolicy=Fail` | 19 mutating + 20 validating = 39 |
| Resources intercepted | pods, deployments, statefulsets, jobs, jobsets, rayjobs, pytorchjobs, etc. |
| Operators blocked | monitoring, machine-api, openshift-controller-manager, console, csi-snapshot-controller, olm, openshift-apiserver, authentication, kube-apiserver, kube-controller-manager, kube-scheduler, machine-config |

Master-0 br-ex MTU mismatch (discovered at 13:40):

| Interface | master-0 (broken) | master-1 (correct) |
| --- | --- | --- |
| `ens3` | 9000 | 9000 |
| `br-ex` (Linux) | **1500** | 9000 |
| `br-ex` OVS `mtu_request` | **1500** | 9000 |
| NM profile `ovs-if-br-ex` `802-3-ethernet.mtu` | **1500** | 9000 |
| `br-ex` interface index | **211** (recreated many times) | 8 (clean boot) |

Root cause: master-0 was the only node not rebooted during the incident. Repeated OVS bridge recreation during the instability (index 211 vs 8) caused the NM profile to lose the correct MTU. The OVN `configure-ovs` init script re-created the profile with default MTU 1500 instead of inheriting 9000 from `ens3`. OVN overlay MTU (8958) exceeds 1500, so `ovnkube-controller` refused to start, breaking pod-to-upstream-DNS on that node.

## Root Cause
**Measured chain:**
1. The control plane is three 4 vCPU / 20 GB masters. On 2026-10-07 an experiment added ~1,200 pods, 500 Jobs, 316 Deployments and ~2,000 ConfigMaps on two H100 nodes. kubelet opens a watch per mounted ConfigMap/Secret per pod, so those two kubelets became the heaviest API clients.
2. The masters ran out of headroom: kube-apiserver grows to 7–9.5 GB, load average goes to 50–90, the node stops answering (no swap, page-cache thrash). When one master's apiserver drops, all traffic lands on the other, which then stalls — the masters alternate.
3. Side effect: on a slow control plane the `downloads` pod (liveness `timeoutSeconds: 1`) restart-loops, and each start leaks ~3.1 GB into its emptyDir. That filled master-2's `/var` and killed its etcd — the first, hard outage.
4. With master-1 already down, any single stall or failure of master-0 or master-2 costs etcd quorum.

**Secondary blockers discovered during recovery:**
5. Kueue's mutating and validating webhooks (39 total) were set to `failurePolicy=Fail` and intercepted pods, deployments, statefulsets, and jobs. With the kueue controller crash-looping, every Deployment create/update was rejected — blocking ~12 operators from reconciling their workloads.
6. Master-0's `br-ex` OVS bridge had `mtu_request=1500` while `ens3` (jumbo frames) was 9000. OVN overlay MTU (8958) exceeds 1500, so the ovnkube-controller exited. Pods on master-0 couldn't reach upstream DNS (161.26.0.10/11), causing authentication, storage, and etcd operators to report degraded.

**Not known:** why the 10-04 master reboots and the 10-05 H100 machine replacement happened.

## Resolution

All steps user-approved.

### 1. Free master disks
(`ssh -J core@149.81.35.27 core@<ip>`; find the pod UID with `sudo du -xs /var/lib/kubelet/pods/* | sort -rn | head`):
```bash
T='/var/lib/kubelet/pods/<downloads-pod-uid>/volumes/kubernetes.io~empty-dir/tmp'
sudo cp -a /var/lib/etcd /var/home/core/etcd-backup-$(date +%Y%m%d)   # if etcd is stopped; free one dir first if the disk is 100% full
sudo find "$T" -mindepth 1 -maxdepth 1 -type d -name 'tmp*' ! -name <newest-dir> -exec rm -rf {} +
```
kubelet restarted etcd on master-2 by itself ~90 s later.

> **Mistake made on master-2:** the command was run without `-mindepth 1`. The emptyDir is literally named `tmp`, so `-name 'tmp*'` matched the starting directory and the whole emptyDir was deleted, including `serve.py`. `downloads-55cff565b6-p7gj9` then crash-looped with `/tmp/serve.py: Permission denied`. Impact limited to the console CLI-downloads page. Pod was later GC'd and replaced. **Always use `-mindepth 1` when the base directory itself might match the pattern.**

### 2. Remove the experiment load
In the short windows when the API answers; batches of 100, retried:
```bash
oc delete jobs --all -n trace-replay --wait=false
oc delete deployments --all -n trace-replay --wait=false
oc delete sandboxes.agents.x-k8s.io --all -n openshell-tracesim --wait=false
oc delete configmaps --all -n trace-replay --wait=false
oc delete job 5c7e068a-1-workspace-prep -n openshell-tracesim --wait=false
```
All 500 Jobs, 316 Deployments, 292 Sandboxes, and ~1,984 ConfigMaps deleted. Pod count dropped from 1,709 to 450.

### 3. Reboot hung masters
```bash
ibmcloud is instance-reboot diadochos-hqxzk-master-1 -f   # user ran at ~11:06
ibmcloud is instance-reboot diadochos-hqxzk-master-2 -f   # at 13:10
```
Both recovered within 7–15 minutes. **Reboot only, never stop/start** — stop/start competes for on-demand capacity.

### 4. Patch kueue webhooks
```bash
# For both mutating and validating:
oc get mutatingwebhookconfigurations kueue-mutating-webhook-configuration -o json \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [wh.__setitem__('failurePolicy','Ignore') for wh in d['webhooks']]; print(json.dumps(d))" \
  | oc replace -f -
# Same for validatingwebhookconfigurations kueue-validating-webhook-configuration
```
This unblocked 12 operators within minutes. After kueue stabilized (~30 min later, 1/1 Ready with ready endpoints), webhooks were restored to `failurePolicy=Fail`.

### 5. Fix br-ex MTU on master-0
```bash
ssh -J core@149.81.35.27 core@10.243.0.6
# Fix OVS database (immediate effect):
sudo ovs-vsctl set interface br-ex mtu_request=9000
# Fix NetworkManager profile (persistent across bridge recreation):
sudo nmcli connection modify ovs-if-br-ex 802-3-ethernet.mtu 9000
```
Then delete the OVN node pod on master-0 to force re-initialization with the correct MTU. Pod networking and upstream DNS resolution restored.

### 6. Cleanup
- etcd backup `/var/home/core/etcd-backup-20261008` removed from master-2 (1.6 GB freed).
- All 7 nodes verified at br-ex MTU 9000 (both OVS and NM profile).

## Safety constraints used
- Never stop/start/reboot an H100 instance from IBM Cloud — no reservation, may not restart (see [[2026-07-17 - IBM Cloud H100 node failed cannot_start_capacity on diadochos]]).
- Masters: reboot only (not stop/start), one at a time, never the last healthy etcd member.
- Copy `/var/lib/etcd` before a stopped member restarts.
- Check `MachineHealthCheck` as soon as the API recovers.
- No `oc delete node`, no machine deletion, no MachineSet scaling.
- A master that is unreachable but `running` in IBM Cloud may be thrashing, not dead — master-2 recovered without a reboot twice.

## Prevention / Runbook
- **Resize the masters.** 4 vCPU / 20 GB cannot carry ~1,700 pods; this is the underlying cause. Next size up is `bx3d-8x40` (8 vCPU / 40 GB). Needs a planned, one-at-a-time stop/resize/start.
- Cap experiment scale: namespace `ResourceQuota` on pods/configmaps for large agent experiments; keep per-node pod counts far below 600–750.
- Alert on master `/var` usage, master memory and etcd fsync latency.
- Bound the downloads pod's disk use (ephemeral-storage limit / emptyDir `sizeLimit`), if console-operator allows it.
- Faster boot-volume profile for masters (etcd reports slow fdatasync on 3000 IOPS general-purpose).
- Keep a documented SSH path to the masters; the leftover bootstrap VM was the only way in.
- **Set kueue webhooks to `failurePolicy=Ignore`** or add a `namespaceSelector` excluding system namespaces — a crash-looping admission webhook with `Fail` policy blocks the entire cluster.
- **Monitor br-ex MTU on all nodes** — if it drifts from the physical interface MTU (9000 on IBM Cloud VPC jumbo frames), pod networking silently breaks. The NM profile `ovs-if-br-ex` `802-3-ethernet.mtu` is the persistent source of truth; the OVS `mtu_request` is the runtime value.

## Related
- [[2026-07-17 - IBM Cloud H100 node failed cannot_start_capacity on diadochos]]
- [[2026-07-22 - Kueue webhook no endpoints rejected post-benchmark TaskRuns]] — earlier API timeout episode on this cluster
- [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]