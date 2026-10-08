---
title: "2026-10-08 — diadochos API unresponsive: undersized masters overloaded by ~1,200 experiment pods, etcd quorum lost"
date: 2026-10-08
type: incident
status: open
cluster: diadochos
platform: IBM Cloud VPC (eu-de)
ocp_version: 4.22.2
---

# 2026-10-08 — diadochos API unresponsive, etcd quorum repeatedly lost (control plane overload + master /var full)

> **Status: OPEN at 10:23 UTC — API flapping. Disk-full condition fixed; experiment workload mostly removed; the two reachable masters still take turns stalling under load and master-1 has been down since 2026-10-07 17:35 UTC. No H100 node was lost or touched.**

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
- **10:07–10:14** — In the windows: all 500 Jobs deleted, 223+ of 300 agent Deployments deleted.
- **10:22** — master-2 stalls again; master-0 recovered. API down. Deletion loop still retrying.

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

## Root Cause
**Measured chain:**
1. The control plane is three 4 vCPU / 20 GB masters. On 2026-10-07 an experiment added ~1,200 pods, 500 Jobs, 316 Deployments and ~2,000 ConfigMaps on two H100 nodes. kubelet opens a watch per mounted ConfigMap/Secret per pod, so those two kubelets became the heaviest API clients.
2. The masters ran out of headroom: kube-apiserver grows to 7–9.5 GB, load average goes to 50–90, the node stops answering (no swap, page-cache thrash). When one master's apiserver drops, all traffic lands on the other, which then stalls — the masters alternate.
3. Side effect: on a slow control plane the `downloads` pod (liveness `timeoutSeconds: 1`) restart-loops, and each start leaks ~3.1 GB into its emptyDir. That filled master-2's `/var` and killed its etcd — the first, hard outage.
4. With master-1 already down, any single stall or failure of master-0 or master-2 costs etcd quorum.

**Inference (not verified):** master-1 was lost on 10-07 17:35 by the same overload stall and never recovered.

**Not known:** why the 10-04 master reboots and the 10-05 H100 machine replacement happened; whether `openshell-tracesim` alone is still enough to overload the masters.

## Resolution
Applied so far (all user-approved):

**1. Free master disks** (`ssh -J core@149.81.35.27 core@<ip>`; find the pod UID with `sudo du -xs /var/lib/kubelet/pods/* | sort -rn | head`):
```bash
T='/var/lib/kubelet/pods/<downloads-pod-uid>/volumes/kubernetes.io~empty-dir/tmp'
sudo cp -a /var/lib/etcd /var/home/core/etcd-backup-$(date +%Y%m%d)   # if etcd is stopped; free one dir first if the disk is 100% full
sudo find "$T" -mindepth 1 -maxdepth 1 -type d -name 'tmp*' ! -name <newest-dir> -exec rm -rf {} +
```
kubelet restarted etcd on master-2 by itself ~90 s later.

> **Mistake made on master-2:** the command was run without `-mindepth 1`. The emptyDir is literally named `tmp`, so `-name 'tmp*'` matched the starting directory and the whole emptyDir was deleted, including `serve.py`. `downloads-55cff565b6-p7gj9` then crash-looped with `/tmp/serve.py: Permission denied` (fix: delete that pod). Impact limited to the console CLI-downloads page.

**2. Remove the experiment load** (in the short windows when the API answers; batches of 100, retried):
```bash
oc get jobs -n trace-replay -l app=aharush-experiment -o name | head -100 | xargs oc delete -n trace-replay --wait=false
oc get deploy -n trace-replay -o name | grep -E -- '-openclaw-[0-9]+$' | head -100 | xargs oc delete -n trace-replay --wait=false
```
Backends (`aharush-jaeger`, `aharush-mock-llm`, `aharush-*-backend`, `aharush-openclaw-shell`) left in place. `openshell-tracesim` not touched yet.

Useful checks:
```bash
oc get --raw '/livez/etcd'
oc exec -n openshift-etcd etcd-diadochos-hqxzk-master-0 -c etcdctl -- etcdctl endpoint health --cluster
oc get machinehealthcheck -A
# on a master:
uptime; free -m; ps -eo rss,pcpu,comm --sort=-rss | head; sudo crictl ps -a | grep -E ' (etcd|kube-apiserver) '
```

### Still to do
1. Finish deleting the remaining `trace-replay` agent Deployments; confirm pods are garbage-collected.
2. Reboot master-1 from IBM Cloud (`ibmcloud is instance-reboot diadochos-hqxzk-master-1` — reboot, never stop/start) to get a third apiserver and etcd member.
3. If the masters still stall: remove `openshell-tracesim` sandboxes.
4. Delete the broken `downloads-55cff565b6-p7gj9` pod.
5. Remove `/var/home/core/etcd-backup-20261008` on master-2 when no longer needed.
6. Review degraded cluster operators once stable.

## Safety constraints used
- Never stop/start/reboot an H100 instance from IBM Cloud — no reservation, may not restart (see [[2026-07-17 - IBM Cloud H100 node failed cannot_start_capacity on diadochos]]).
- Masters: reboot only (not stop/start), one at a time, never the last healthy etcd member.
- Copy `/var/lib/etcd` before a stopped member restarts.
- Check `MachineHealthCheck` as soon as the API recovers.
- No `oc delete node`, no machine deletion, no MachineSet scaling.
- A master that is unreachable but `running` in IBM Cloud may be thrashing, not dead — master-2 recovered without a reboot twice.

## Prevention / Runbook
_To be completed after full resolution._ Candidates:
- **Resize the masters.** 4 vCPU / 20 GB cannot carry ~1,700 pods; this is the underlying cause. Needs a planned, one-at-a-time stop/resize/start.
- Cap experiment scale: namespace `ResourceQuota` on pods/configmaps for large agent experiments; keep per-node pod counts far below 600–750.
- Alert on master `/var` usage, master memory and etcd fsync latency.
- Bound the downloads pod's disk use (ephemeral-storage limit / emptyDir `sizeLimit`), if console-operator allows it.
- Faster boot-volume profile for masters (etcd reports slow fdatasync on 3000 IOPS general-purpose).
- Keep a documented SSH path to the masters; the leftover bootstrap VM was the only way in.

## Related
- [[2026-07-17 - IBM Cloud H100 node failed cannot_start_capacity on diadochos]]
- [[2026-07-22 - Kueue webhook no endpoints rejected post-benchmark TaskRuns]] — earlier API timeout episode on this cluster
- [[2026-07-20 - Ceph HEALTH_WARN orphaned OSDs crash-looping on diadochos]]