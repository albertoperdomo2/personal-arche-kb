---
title: "CephFS performance tuning for KV-cache offloading"
date: 2026-07-28
type: learning
cluster: diadochos
topic: CephFS tuning for vLLM KV-cache workloads
---

# CephFS performance tuning for KV-cache offloading

CephFS can provide a shared, read-write filesystem for vLLM KV-cache offloading, but a default Rook-Ceph deployment is usually optimized for durable general-purpose storage rather than high-throughput, regenerable cache data.

On the `diadochos` cluster, the untuned path suffered from replicated writes, insufficient OSD and MDS resources, a slow overlay-network data path, small client queues, and too few vLLM filesystem workers. Those problems formed a feedback loop:

```text
slow CephFS writes
  → the CPU cache fills
  → vLLM refuses additional block stores
  → prompt tokens are recomputed
  → GPU work and cache churn increase
  → more blocks are evicted
```

This guide shows the final configuration and the commands used to implement it. It is intended for Ceph and OpenShift administrators deploying CephFS as a secondary tier behind a CPU KV cache.

## What the tuning achieved

The complete storage tuning sequence increased synthetic throughput from about 2 GB/s to:

- **23.5 GB/s reads**
- **22.6 GB/s writes**

Those peak figures were measured at I/O depth 8. At depth 1, the tuned path delivered 13.0 GB/s reads and 12.7 GB/s writes with lower tail latency.

In a clean Qwen3.6-35B-A3B vLLM run at concurrency 32, local NVMe and CephFS both delivered **1.518 requests/s**. The CephFS data pool averaged 291.9 MiB/s read and 227.5 MiB/s write while the PVC grew by 372.6 GiB, confirming that the tier was exercised.

These results apply to the tested cluster and workload. They do not establish universal CephFS/NVMe equivalence.

## Final architecture

| Component | Tuned configuration |
|---|---|
| Ceph | 19.2.4 Squid |
| Rook | 1.20.2 |
| Storage | 21 NVMe OSDs across three nodes |
| Filesystem | `kvcache-fs` |
| Data pool | `kvcache-fs-data0`, NVMe, 128 PGs |
| Data replicas | One copy for regenerable KV-cache data |
| Metadata replicas | Three copies |
| MDS | Two active daemons with standby-replay |
| Network | Host networking over 200 Gbit/s NICs, MTU 9000 |
| Volume | 3 TiB RWX PVC using `rook-cephfs-fast` |
| vLLM | 64 GiB CPU tier plus CephFS secondary tier |
| Filesystem workers | 64 readers and 32 writers |

> **Important:** `size=1` provides no data redundancy. Use it only for a dedicated cache pool whose contents can be recomputed. Do not use it for model weights, checkpoints, user data, or other durable content. Keep the CephFS metadata pool replicated.

## 1. Record the baseline

Run Ceph commands through the Rook toolbox. Save the output before changing anything so every override can be reviewed or reverted later.

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph -s
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph versions
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph osd tree
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph fs status kvcache-fs
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get kvcache-fs-data0 all
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph config dump

oc get cephcluster -n rook-ceph -o yaml
oc get cephfilesystem -n rook-ceph kvcache-fs -o yaml
oc get storageclass rook-cephfs-fast -o yaml
```

Do not tune a cluster with unexplained degraded or inactive PGs, down OSDs, or an unhealthy filesystem.

## 2. Configure the KV-cache filesystem and pools

The filesystem uses a replicated metadata pool and a non-replicated NVMe data pool:

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

Apply the manifest:

```bash
oc apply -f kvcache-fs.yaml

oc wait -n rook-ceph cephfilesystem/kvcache-fs \
  --for=condition=Ready \
  --timeout=10m
```

For an existing filesystem, reconcile the data pool explicitly:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set kvcache-fs-data0 size 1 \
  --yes-i-really-mean-it

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set kvcache-fs-data0 min_size 1

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set kvcache-fs-data0 pg_num 128

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool set kvcache-fs-data0 pgp_num 128
```

The original pool had one placement group, which directed the workload to one primary OSD. Increasing both `pg_num` and `pgp_num` to 128 distributed objects across all 21 OSDs.

Wait for clean PGs before continuing:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph pg stat

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get kvcache-fs-data0 size

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get kvcache-fs-data0 pg_num

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph osd pool get kvcache-fs-data0 pgp_num
```

The cluster used `ceph health mute POOL_NO_REDUNDANCY` for this intentional cache-pool exception. Do not use that command to conceal an accidental loss of redundancy elsewhere.

## 3. Increase OSD and MDS resources

The OSD containers were raised to a 12 GiB memory limit, with an 8 GiB Ceph memory target:

```yaml
spec:
  resources:
    osd:
      requests:
        memory: 8Gi
      limits:
        memory: 12Gi
```

Apply that change through the managed `CephCluster` manifest:

```bash
oc apply -f ceph-cluster.yaml
```

Configure the OSD memory target and the 4 GiB MDS metadata cache:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd osd_memory_target 8589934592

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set mds mds_cache_memory_limit 4294967296
```

Two active MDS daemons improve metadata concurrency. The standby-replay daemons provide faster failover; they do not add active request capacity.

## 4. Tune OSD scheduling and parallelism

Apply the OSD settings:

```bash
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
```

The `high_client_ops` mClock profile favors serving reads and writes over recovery and scrub work. This is appropriate for the tested ephemeral cache pool, but it can extend recovery time.

`bdev_async_discard_threads=1` is the workaround used for the Ceph Squid v19 discard issue. Re-evaluate it after upgrading Ceph.

Set the mClock capacity for each OSD from that NVMe device's measured capability:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd.<OSD_ID> \
  osd_mclock_max_capacity_iops_ssd <MEASURED_IOPS>
```

The measured drives ranged from approximately 44,000 to 79,000 IOPS. Do not assign the maximum value to every OSD.

Several queue and thread settings are startup-sensitive. Restart one OSD at a time and check cluster health after each restart:

```bash
oc rollout restart -n rook-ceph \
  deploy/rook-ceph-osd-<OSD_ID>

oc rollout status -n rook-ceph \
  deploy/rook-ceph-osd-<OSD_ID> \
  --timeout=10m

oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph -s
```

Sixteen shards with four workers each expose substantial OSD concurrency. Watch OSD CPU utilization and context switching before applying these values to smaller nodes.

## 5. Increase client and messenger concurrency

Apply the Ceph client and networking limits:

```bash
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

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set global ms_tcp_rcvbuf 4194304
```

These values provide:

- 32 MiB client read-ahead
- a larger metadata entry cache
- a 1 GiB client object cache
- up to 4,096 in-flight operations
- up to 500 MB of in-flight operation data
- a 4 MiB Ceph messenger receive buffer

Verify the effective configuration:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config get client client_readahead_max_bytes

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config get client objecter_inflight_ops

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config get global ms_tcp_rcvbuf
```

## 6. Move Ceph traffic to the high-speed network

CephFS data flows directly between the kernel client and OSDs. Moving only MON traffic does not improve file throughput; the OSDs must advertise addresses that the clients can reach over the storage fabric.

The persistent Rook configuration was:

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

Configure the matching Ceph network list:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set global public_network \
  "10.243.65.0/24,10.0.0.0/16,10.1.0.0/16,10.2.0.0/16,10.3.0.0/16,10.4.0.0/16,10.5.0.0/16"
```

> **Disruptive migration:** Do not apply `network.provider: host` to a live overlay-networked cluster as a routine rolling change. The `diadochos` migration required a coordinated MON cutover, updates to `rook-ceph-mon-endpoints` and `rook-ceph-config`, and about 15 minutes of CephFS downtime. Applying only part of the change broke MON quorum. Follow [[Routing Ceph over 200G NICs on OCP with PCI passthrough VFs]] for the exact migration and recovery procedure.

Pin the exact Ceph image version during the cutover. A floating major tag caused an operator version-reconciliation deadlock.

### Spread OSDs across the available NICs

The final layout used six 200 Gbit/s interfaces per storage node:

| Interface | Network | OSDs per node |
|---|---|---:|
| `enp163s0` | `10.0.0.0/16` | 2 |
| `enp173s0` | `10.1.0.0/16` | 1 |
| `enp183s0` | `10.2.0.0/16` | 1 |
| `enp193s0` | `10.3.0.0/16` | 1 |
| `enp203s0` | `10.4.0.0/16` | 1 |
| `enp213s0` | `10.5.0.0/16` | 1 |

Build an explicit OSD-to-node-to-IP inventory. Do not infer node placement from OSD IDs because historical removals left gaps.

Bind and restart one OSD at a time:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config set osd.<OSD_ID> \
  public_addr "v2:<OSD_IP>:0/0"

oc rollout restart -n rook-ceph \
  deploy/rook-ceph-osd-<OSD_ID>

oc rollout status -n rook-ceph \
  deploy/rook-ceph-osd-<OSD_ID> \
  --timeout=10m

oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph -s
```

Verify the advertised addresses:

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph osd dump

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config dump | grep -E 'public_network|public_addr'
```

During a controlled CephFS write, use `ip -s link` on the node to prove that bytes increase on the storage NICs rather than the management interface.

## 7. Configure the CephFS StorageClass

The final mount options were:

```yaml
mountOptions:
  - noatime
  - nodiratime
  - wsize=67108864
  - rsize=67108864
```

The early tuning note used 16 MiB read/write sizes. The final StorageClass uses 64 MiB, shown above.

Patch the performance StorageClass:

```bash
oc patch storageclass rook-cephfs-fast \
  --type merge \
  -p '{"mountOptions":["noatime","nodiratime","wsize=67108864","rsize=67108864"]}'
```

These options eliminate cache-irrelevant access-time writes and request larger I/O operations. They apply when the volume is mounted, so restart consumers after changing the StorageClass.

Verify the effective mount from the workload container:

```bash
oc exec -n benchflow <VLLM_POD> -- \
  findmnt -T /mnt/kv_cache \
  -o TARGET,FSTYPE,OPTIONS
```

The effective mount output is authoritative; confirm that the client kernel accepted the requested `rsize` and `wsize`.

## 8. Create the RWX KV-cache volume

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

Apply and wait for the claim:

```bash
oc apply -f vllm-kv-cache-pvc.yaml

oc wait -n benchflow pvc/vllm-kv-cache \
  --for=jsonpath='{.status.phase}'=Bound \
  --timeout=10m
```

## 9. Configure the vLLM filesystem tier

The validated deployment used a 64 GiB CPU cache in front of CephFS, with 64 filesystem read threads and 32 write threads:

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

The filesystem worker counts are vLLM settings, not Ceph settings. Too few workers leave a higher-latency shared backend idle while requests wait.

Confirm the deployed command and mount:

```bash
oc get pod -n benchflow <VLLM_POD> \
  -o jsonpath='{.spec.containers[0].args}' | tr ' ' '\n'

oc exec -n benchflow <VLLM_POD> -- \
  test -w /mnt/kv_cache

oc exec -n benchflow <VLLM_POD> -- \
  df -h /mnt/kv_cache
```

## 10. Validate the complete path

### Ceph health

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph -s
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph health detail
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph pg stat
oc exec -n rook-ceph deploy/rook-ceph-tools -- ceph fs status kvcache-fs
```

Expected state:

- All 21 OSDs are `up` and `in`.
- Data-pool PGs are `active+clean`.
- `kvcache-fs` has two active MDS daemons and standby coverage.
- Only the dedicated KV-cache data pool has one replica.

### Configuration

```bash
oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config get osd osd_memory_target

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config get osd osd_op_num_shards_ssd

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config get osd osd_op_num_threads_per_shard_ssd

oc exec -n rook-ceph deploy/rook-ceph-tools -- \
  ceph config get osd osd_mclock_profile

oc get storageclass rook-cephfs-fast -o yaml
oc get pvc -n benchflow vllm-kv-cache
```

### Workload evidence

Archive at least:

- Request throughput, TTFT, and end-to-end latency
- Completed, cancelled, and errored sessions
- Prompt-token rate by source
- GPU KV-cache occupancy
- CPU-to-GPU and GPU-to-CPU KV-transfer bandwidth
- CephFS pool read/write throughput and IOPS
- PVC used bytes
- OSD and MDS operation latency
- Slow operations, stalls, and vLLM block-store refusals

The workload must show both storage activity and PVC growth. A healthy Ceph cluster alone does not prove that vLLM used the secondary tier.

## Choosing I/O depth

Synthetic tests showed:

| I/O depth | Read | Write | Trade-off |
|---:|---:|---:|---|
| 1 | 13.0 GB/s | 12.7 GB/s | Lower latency |
| 4 | 21.0 GB/s | 20.4 GB/s | Large throughput gain |
| 8 | 23.5 GB/s | 22.6 GB/s | Maximum measured throughput; doubled P99 latency |

I/O depth is a benchmark or application concurrency setting. It is not a Ceph cluster parameter. Select it from the serving workload's latency objective rather than aggregate bandwidth alone.

## Rollback

Rollback from the workload inward:

1. Remove the filesystem secondary tier from vLLM and verify operation with the CPU tier.
2. Revert the StorageClass options and remount the PVC.
3. Restore the Ceph values recorded before tuning, or remove an override:

   ```bash
   oc exec -n rook-ceph deploy/rook-ceph-tools -- \
     ceph config rm <SCOPE> <SETTING>
   ```

4. Restore `osd_mclock_profile=balanced` if recovery work must take priority.
5. Remove per-OSD `public_addr` overrides one OSD at a time, checking health after every restart.
6. Treat host-network and MON-address rollback as a coordinated outage. Use the tested procedure in [[Routing Ceph over 200G NICs on OCP with PCI passthrough VFs]].
7. Plan pool-replication changes for the resulting backfill and capacity demand; they are not instantaneous.

## Related

- [[CephFS tuning guide - concepts and rationale]] — setting-by-setting reference and rationale
- [[Routing Ceph over 200G NICs on OCP with PCI passthrough VFs]] — exact disruptive network migration and monmap recovery
- [[2026-07-27 - CephFS mount DeadlineExceeded on worker node due to stale monitor IP]] — stale CephFS kernel mounts after MON migration
- [[2026-07-28 - Clean U0.85 offload matrix]] — clean application-level validation