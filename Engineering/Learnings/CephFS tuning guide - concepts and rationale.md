---
title: "CephFS tuning guide for KV-cache workloads"
date: 2026-07-28
type: learning
cluster: diadochos
topic: CephFS performance tuning for KV-cache offloading
---

# CephFS tuning guide for KV-cache workloads

This is the implementation-oriented version of the CephFS tuning work on the `diadochos` cluster. It covers the settings that mattered for a vLLM `TieringOffloadingSpec` filesystem tier, the order in which to apply them, and the checks needed to prove that the workload is using the intended storage and network paths.

## Executive summary

The successful configuration combined four layers:

1. **A KV-cache-specific CephFS pool**: NVMe-backed, 128 PGs, two active MDS daemons, and a single data copy because the cached tensors are regenerable.
2. **A fast Ceph data path**: Rook host networking over 200 Gbit/s NICs, followed by per-OSD distribution across six NICs per storage node.
3. **OSD and client concurrency tuning**: more OSD shards and workers, a client-I/O-oriented mClock profile, larger in-flight limits and receive buffers, and sufficient daemon memory.
4. **Application concurrency**: a 64 GiB CPU tier in front of CephFS, with 64 filesystem read threads and 32 write threads.

On the synthetic storage path, the complete tuning sequence raised throughput from about 2 GB/s to 23.5 GB/s read and 22.6 GB/s write at I/O depth 8. In the clean Qwen3.6-35B-A3B application run, CephFS and local NVMe both reached 1.518 requests/s; CephFS averaged 291.9 MiB/s read and 227.5 MiB/s write. This is evidence for this configuration and workload, not a claim that CephFS is always equivalent to local NVMe.

> [!danger] Ephemeral-data boundary
> `replicated.size: 1` leaves the data pool with no redundancy. Use it only for a KV cache that the model can recompute. Keep the CephFS metadata pool replicated, and never reuse this setting for model weights, checkpoints, user data, or any other durable content.

## Validated target

| Layer | Validated setting |
|---|---|
| Ceph / Rook | Ceph 19.2.4 (Squid), Rook 1.20.2 |
| Storage | 21 NVMe OSDs, seven per GPU node |
| Filesystem | `kvcache-fs`; data pool `kvcache-fs-data0` |
| Data durability | Data pool size/min-size `1/1`; metadata pool size `3` |
| Metadata | Two active MDS daemons plus standby-replay |
| Network | Host networking; 200 Gbit/s ConnectX-7 data NICs; MTU 9000 |
| Kubernetes | RWX PVC from `rook-cephfs-fast`, mounted at `/mnt/kv_cache` |
| vLLM | `TieringOffloadingSpec`, 64 GiB CPU tier, CephFS secondary tier |
| FS workers | 64 read threads, 32 write threads |

## Apply the tuning in this order

The order matters. Establish a baseline first, then fix pool distribution, storage resources, mount behavior, the network path, and finally application concurrency. Re-benchmark after each phase so a regression has a small rollback surface.

### 1. Capture the current state

Run all Ceph commands through the Rook toolbox:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph -s
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph versions
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph fs status kvcache-fs
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph osd tree
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get kvcache-fs-data0 all
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph config dump
oc get cephcluster,cephfilesystem -n rook-ceph -o yaml
oc get storageclass rook-cephfs-fast -o yaml
```

Also record:

- CephFS pool read/write bytes and IOPS.
- OSD apply/commit latency and slow operations.
- MDS request latency and cache pressure.
- NIC byte counters on the management and storage interfaces.
- vLLM request throughput, TTFT, end-to-end latency, prompt-token source, KV-cache occupancy, and offload store-refusal warnings.

Do not continue while PGs are degraded, misplaced, or inactive, or while the cluster has an unexplained health warning.

### 2. Create or reconcile the KV-cache filesystem

The important CephFilesystem fields are shown below. Preserve site-specific placement, priority classes, and annotations from the deployed manifest.

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: kvcache-fs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      size: 3
  dataPools:
    - name: data0
      deviceClass: nvme
      replicated:
        size: 1
  metadataServer:
    activeCount: 2
    activeStandby: true
    resources:
      requests:
        cpu: "2"
        memory: 4Gi
      limits:
        cpu: "4"
        memory: 8Gi
```

Apply it and wait for the filesystem to become ready:

```bash
oc apply -f kvcache-fs.yaml
oc wait -n rook-ceph cephfilesystem/kvcache-fs \
  --for=condition=Ready --timeout=10m
```

If the filesystem already exists, changing the CR does not necessarily rewrite every existing pool property. Reconcile the data pool explicitly:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set kvcache-fs-data0 size 1 --yes-i-really-mean-it
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set kvcache-fs-data0 min_size 1
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set kvcache-fs-data0 pg_num 128
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set kvcache-fs-data0 pgp_num 128
```

The change from one PG to 128 was essential: one PG sent the pool's work to one primary OSD, while 128 PGs spread objects over all 21 OSDs. Wait for `active+clean` after changing PGs before proceeding.

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get kvcache-fs-data0 size
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get kvcache-fs-data0 min_size
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get kvcache-fs-data0 pg_num
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get kvcache-fs-data0 pgp_num
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph pg stat
```

Do not hide `POOL_NO_REDUNDANCY` globally. If the warning is intentionally muted for this dedicated ephemeral pool, document that exception and continue monitoring all other health checks.

### 3. Apply daemon and client configuration

These are the values applied on `diadochos`:

```bash
# OSD memory, scheduling, internal parallelism, and messenger workers
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd osd_memory_target 8589934592
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd osd_op_num_shards_ssd 16
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd osd_op_num_threads_per_shard_ssd 4
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd osd_mclock_profile high_client_ops
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd ms_async_op_threads 5
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd bdev_async_discard_threads 1

# MDS cache
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mds mds_cache_memory_limit 4294967296

# Ceph client/objecter limits
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set client client_readahead_max_bytes 33554432
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set client client_cache_size 65536
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set client client_oc_size 1073741824
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set client objecter_inflight_ops 4096
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set client objecter_inflight_op_bytes 524288000

# Messenger receive buffer
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set global ms_tcp_rcvbuf 4194304
```

`bdev_async_discard_threads=1` is the workaround used for the Squid v19 discard issue. Re-evaluate it during a Ceph upgrade rather than carrying it forward indefinitely.

The mClock capacity must be set per OSD from that drive's measured capability; the observed values were approximately 44,000–79,000 IOPS:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd.<OSD_ID> \
  osd_mclock_max_capacity_iops_ssd <MEASURED_IOPS>
```

Do not assign the highest observed number to every drive. Keep an OSD-to-device inventory and use the result measured for that device.

The Rook OSD pod needs a memory limit above the 8 GiB Ceph target; the tuned pods used 12 GiB. Configure the CephCluster resource stanza managed by your deployment rather than patching a generated OSD Deployment permanently:

```yaml
spec:
  resources:
    osd:
      requests:
        memory: 8Gi
      limits:
        memory: 12Gi
```

Several OSD queue/thread values are startup-sensitive. Roll OSDs one at a time, keeping the cluster healthy between restarts:

```bash
oc rollout restart -n rook-ceph deploy/rook-ceph-osd-<OSD_ID>
oc rollout status -n rook-ceph deploy/rook-ceph-osd-<OSD_ID> --timeout=10m
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph -s
```

Sixteen shards and four workers per shard expose up to 64 operation workers per OSD. This is a calibrated setting for the tested CPU and NVMe layout, not a default for smaller nodes. Watch OSD CPU saturation, context switching, and latency when transferring it to another cluster.

### 4. Move the data path to the high-speed NICs

The CephFS client sends file data directly to OSDs. A fast backend is ineffective if OSDs still advertise overlay or management addresses.

The persistent CephCluster intent is:

```yaml
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19.2.4
  network:
    provider: host
    addressRanges:
      public:
        - "10.243.65.0/24"
        - "10.0.0.0/16"
        - "10.1.0.0/16"
        - "10.2.0.0/16"
        - "10.3.0.0/16"
        - "10.4.0.0/16"
        - "10.5.0.0/16"
```

Pin the exact Ceph version during this migration. A floating major tag caused an operator upgrade/quorum deadlock in the original work.

> [!danger] This is a disruptive migration
> Do not apply `network.provider: host` to a live overlay-networked cluster as an ordinary rolling change. On `diadochos`, old overlay MONs and new host-network daemons could not reach one another, quorum failed, and monmap repair was required. Follow [[Routing Ceph over 200G NICs on OCP with PCI passthrough VFs]] for the actual coordinated MON migration, ConfigMap/Secret updates, generated Deployment patches, verification, and rollback procedure.

The completed migration must satisfy all of these conditions:

- The operator, toolbox, CSI controller, MONs, OSDs, MGRs, and MDSs can reach the selected host networks.
- `rook-ceph-mon-endpoints` and `rook-ceph-config` contain the current MON addresses.
- MON/MGR `--public-addr=$(ROOK_POD_IP)` arguments do not override the intended bind address.
- Existing kernel CephFS mounts with stale MON addresses are unmounted from the host mount namespace and remounted.
- The kernel clients can route to every advertised OSD address.
- MTU 9000 works end to end on every storage path; a single smaller hop invalidates the jumbo-frame assumption.

Set the Ceph network list to match the CR:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set global public_network \
  "10.243.65.0/24,10.0.0.0/16,10.1.0.0/16,10.2.0.0/16,10.3.0.0/16,10.4.0.0/16,10.5.0.0/16"
```

#### Distribute OSDs across the six NICs

All seven OSDs on a node initially selected the first 200 Gbit/s NIC. The final layout placed two OSDs on the first NIC and one on each of the other five:

| Interface | Subnet | OSDs per node |
|---|---|---:|
| `enp163s0` | `10.0.0.0/16` | 2 |
| `enp173s0` | `10.1.0.0/16` | 1 |
| `enp183s0` | `10.2.0.0/16` | 1 |
| `enp193s0` | `10.3.0.0/16` | 1 |
| `enp203s0` | `10.4.0.0/16` | 1 |
| `enp213s0` | `10.5.0.0/16` | 1 |

Build an explicit OSD-to-IP inventory from `ceph osd metadata`, the OSD Deployment's node placement, and `ip -br addr` on that node. Then set one address and restart one OSD at a time:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd.<OSD_ID> public_addr "v2:<OSD_IP>:0/0"
oc rollout restart -n rook-ceph deploy/rook-ceph-osd-<OSD_ID>
oc rollout status -n rook-ceph deploy/rook-ceph-osd-<OSD_ID> --timeout=10m
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph -s
```

Do not generate this mapping from OSD IDs alone: historical OSD removal left gaps, so the ID does not reliably imply a node or device.

Verify the advertised addresses and the physical path:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd dump
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config dump | grep -E 'public_network|public_addr'

# From a node while a controlled CephFS write is running:
ip -s link show enp3s0
ip -s link show enp163s0
ip -s link show enp173s0
ip -s link show enp183s0
ip -s link show enp193s0
ip -s link show enp203s0
ip -s link show enp213s0
```

The management interface should remain nearly idle for file data, while the assigned high-speed interfaces increase.

### 5. Apply CephFS mount options

The tuned StorageClass uses:

```yaml
mountOptions:
  - noatime
  - nodiratime
  - wsize=67108864
  - rsize=67108864
```

Patch the existing class, or clone it to `rook-cephfs-fast` while preserving its site-specific CSI parameters and secrets:

```bash
oc patch storageclass rook-cephfs-fast --type merge \
  -p '{"mountOptions":["noatime","nodiratime","wsize=67108864","rsize=67108864"]}'
```

`noatime` and `nodiratime` avoid metadata writes that have no value for cache files. The large read/write sizes reduce RPC overhead for large sequential KV tensors. Confirm that the client kernel accepts the requested values; the effective mount options, not the StorageClass YAML, are the source of truth.

The options apply on mount, so recycle workload pods after the StorageClass change:

```bash
findmnt -T /mnt/kv_cache -o TARGET,FSTYPE,OPTIONS
```

### 6. Provision and mount the RWX cache volume

The validated claim was:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vllm-kv-cache
  namespace: benchflow
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: rook-cephfs-fast
  resources:
    requests:
      storage: 3Ti
```

Apply and verify:

```bash
oc apply -f vllm-kv-cache-pvc.yaml
oc wait -n benchflow pvc/vllm-kv-cache \
  --for=jsonpath='{.status.phase}'=Bound --timeout=10m
```

### 7. Configure vLLM for the CephFS tier

The clean run used a 64 GiB CPU tier as the primary offload cache and CephFS as its secondary filesystem tier. The exact connector configuration was:

```yaml
spec:
  template:
    containers:
      - name: main
        env:
          - name: PYTHONHASHSEED
            value: "0"
        args:
          - >-
            --kv-transfer-config={"kv_connector":"OffloadingConnector","kv_role":"kv_both","kv_connector_extra_config":{"spec_name":"TieringOffloadingSpec","cpu_bytes_to_use":68719476736,"secondary_tiers":[{"type":"fs","root_dir":"/mnt/kv_cache","n_read_threads":64,"n_write_threads":32}]}}
          - --enable-prefix-caching
          - --kv-cache-metrics
          - --kv-cache-metrics-sample=0.01
        volumeMounts:
          - name: cephfs-kv-cache
            mountPath: /mnt/kv_cache
    volumes:
      - name: cephfs-kv-cache
        persistentVolumeClaim:
          claimName: vllm-kv-cache
```

The read/write worker counts are application settings, not Ceph settings. The filesystem backend has higher latency than CPU memory, so the default thread counts did not keep enough work in flight. The tuned values were:

- `n_read_threads: 64`
- `n_write_threads: 32`

`PYTHONHASHSEED=0` was also present in the validated CephFS deployment and should be retained for deterministic comparison. Keep all other workload controls—model, tensor parallelism, GPU-memory utilization, shared memory, concurrency, seed, and replay duration—identical while evaluating storage changes.

### 8. Choose client queue depth deliberately

Synthetic throughput increased with I/O depth:

| I/O depth | Read | Write | Interpretation |
|---:|---:|---:|---|
| 1 | 13.0 GB/s | 12.7 GB/s | Latency-oriented baseline after multi-NIC tuning |
| 4 | 21.0 GB/s | 20.4 GB/s | Best large throughput gain |
| 8 | 23.5 GB/s | 22.6 GB/s | Highest measured throughput; P99 latency doubled |

I/O depth is a benchmark/application concurrency control, not a cluster knob. Use depth 1, 4, and 8 to characterize a deployment. Do not select depth 8 solely from aggregate bandwidth if the serving workload is tail-latency-sensitive.

## Configuration reference

| Scope | Setting | Value | Main purpose |
|---|---|---:|---|
| Pool | `size`, `min_size` | `1`, `1` | Remove replication cost for regenerable cache data |
| Pool | `pg_num`, `pgp_num` | `128`, `128` | Distribute objects across 21 OSDs |
| MDS | active count | `2` | Parallelize CephFS metadata work |
| MDS | `mds_cache_memory_limit` | `4294967296` | 4 GiB metadata cache |
| OSD | pod memory limit | `12Gi` | Leave headroom above Ceph's memory target |
| OSD | `osd_memory_target` | `8589934592` | 8 GiB Ceph memory target |
| OSD | `osd_op_num_shards_ssd` | `16` | More independent OSD queues |
| OSD | `osd_op_num_threads_per_shard_ssd` | `4` | More workers per queue |
| OSD | `osd_mclock_profile` | `high_client_ops` | Favor client I/O over recovery/scrub |
| OSD | `osd_mclock_max_capacity_iops_ssd` | per drive | Calibrate mClock to measured NVMe IOPS |
| OSD | `bdev_async_discard_threads` | `1` | Squid v19 discard workaround |
| OSD | `ms_async_op_threads` | `5` | More messenger workers |
| Client | `client_readahead_max_bytes` | `33554432` | 32 MiB read-ahead |
| Client | `client_cache_size` | `65536` | Larger metadata entry cache |
| Client | `client_oc_size` | `1073741824` | 1 GiB object cache |
| Client | `objecter_inflight_ops` | `4096` | More operations in flight |
| Client | `objecter_inflight_op_bytes` | `524288000` | 500 MB in-flight byte budget |
| Global | `ms_tcp_rcvbuf` | `4194304` | 4 MiB receive buffer |
| Mount | `rsize`, `wsize` | `67108864` | 64 MiB requested RPC size |
| Mount | `noatime`, `nodiratime` | enabled | Remove cache-irrelevant metadata writes |
| vLLM | CPU tier | `68719476736` | 64 GiB primary offload cache |
| vLLM | FS read/write threads | `64` / `32` | Keep CephFS I/O concurrent |

## Acceptance checklist

Do not call the tuning complete until all of the following are true:

- `ceph -s` has no unexplained warning and all expected OSDs are `up` and `in`.
- All data-pool PGs are `active+clean`, with `pg_num=pgp_num=128`.
- `kvcache-fs` is `Ready`, with two active MDS daemons and standby coverage.
- The metadata pool remains replicated; only the dedicated KV-cache data pool is size 1.
- OSDs advertise the intended 200 Gbit/s addresses and traffic counters prove that file data uses those NICs.
- The six-NIC layout is visible in OSD addresses rather than seven OSDs sharing the first interface.
- The workload's effective CephFS mount includes `noatime`, `nodiratime`, and the accepted `rsize`/`wsize`.
- The PVC is RWX, bound, mounted at `/mnt/kv_cache`, and its used bytes grow during the run.
- vLLM logs show the `TieringOffloadingSpec` filesystem tier with 64 read and 32 write workers.
- Store-refusal warnings and token recomputation do not regress.
- CephFS pool throughput/IOPS, MDS/OSD latency, KV-cache occupancy, TTFT, E2E latency, and request throughput are archived for comparison.

## Results and interpretation

The clean U=0.85, concurrency-32 Qwen3.6-35B-A3B run produced:

| Configuration | Requests/s | Mean storage read | Mean storage write |
|---|---:|---:|---:|
| CPU 64 GiB + local NVMe | 1.518 | 295.2 MiB/s | 228.1 MiB/s |
| CPU 64 GiB + CephFS | 1.518 | 291.9 MiB/s | 227.5 MiB/s |

The CephFS PVC grew by 372.6 GiB, so the secondary tier was exercised. Its pool averaged 89.1 read IOPS and 75.0 write IOPS, showing that Ceph achieved similar byte throughput with fewer, larger operations than the local NVMe telemetry. No slow-op, stall, or store-refusal evidence appeared in the retained Ceph logs for the benchmark window.

The result should be read as **parity in one controlled run**. Ceph OSD operation-latency and MDS client-request series were unavailable, and one run per cell does not quantify variance. See [[2026-07-28 - Clean U0.85 offload matrix]] for the request-, session-, prompt-source-, KV-utilization-, NVMe-, and CephFS-level evidence.

## Rollback

Rollback from the outside in:

1. Remove the CephFS secondary tier from vLLM and confirm that the service runs from its CPU tier.
2. Revert the StorageClass mount options and remount/recycle consumers.
3. Restore the previous Ceph config values captured in step 1, or remove an override with `ceph config rm <scope> <setting>`.
4. Restore `osd_mclock_profile=balanced` before prioritizing recovery-heavy operation.
5. Roll back per-OSD `public_addr` values one OSD at a time and verify health between restarts.
6. Treat host-network/MON-address rollback as a coordinated outage. Use the tested rollback and monmap-repair procedure in [[Routing Ceph over 200G NICs on OCP with PCI passthrough VFs]].
7. Change pool replication only after confirming enough free space and understanding the resulting backfill. Do not assume changing `size` is instantaneous.

## Related

- [[CephFS performance tuning for KV cache offloading]] — original root-cause notes
- [[Routing Ceph over 200G NICs on OCP with PCI passthrough VFs]] — disruptive 200G migration and recovery runbook
- [[2026-07-27 - CephFS mount DeadlineExceeded on worker node due to stale monitor IP]] — stale kernel mounts after MON migration
- [[2026-07-28 - Clean U0.85 offload matrix]] — clean application-level validation