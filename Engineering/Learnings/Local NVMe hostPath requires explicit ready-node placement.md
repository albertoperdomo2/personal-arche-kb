# Local NVMe hostPath requires an explicit ready-node placement contract

## Finding

A BenchFlow runtime hostPath is node-local. Kubernetes does not know whether the requested host directory is backed by the intended local NVMe filesystem.

On Diadochos, the RHOAI NVMe profile mounts:

```yaml
hostPath:
  path: /var/mnt/benchflow-nvme/benchflow-kv-cache
  type: Directory
```

The run `cpu-offloading-m3` scheduled on `diadochos-hqxzk-gpu-h100-gjfjh`. That node had eight raw 7-TB NVMe disks but no `/var/mnt/benchflow-nvme` mount or `benchflow-kv-cache` directory. Kubelet correctly rejected the pod with:

```
MountVolume.SetUp failed for volume "nvme-kv-cache":
hostPath type check failed:
/var/mnt/benchflow-nvme/benchflow-kv-cache is not a directory
```

## Repair applied on 2026-07-28

All four Diadochos H100 nodes now have one dedicated BenchFlow XFS disk mounted at `/var/mnt/benchflow-nvme` and the cache directory:

| Node | Dedicated device |
|---|---|
| `diadochos-hqxzk-gpu-h100-6kl5z` | `/dev/nvme1n1` |
| `diadochos-hqxzk-gpu-h100-fx7c8` | `/dev/nvme0n1` |
| `diadochos-hqxzk-gpu-h100-gjfjh` | `/dev/nvme0n1` |
| `diadochos-hqxzk-gpu-h100-mt46x` | `/dev/nvme0n1` |

Each filesystem is labeled `bflow-nvme`. The enabled `benchflow-local-nvme.service` resolves that label at boot, mounts XFS with `noatime,nodiratime`, and creates the mount root. The cache directory is mode `1777` and labeled `container_file_t:s0:c14,c27`.

Only devices confirmed to have no filesystem, partition, LVM membership, or Ceph BlueStore signature were formatted. All `ceph_bluestore` devices were left untouched.

## Rule

Do not use a local-NVMe hostPath profile without one of these explicit contracts:

- Provision and validate the mount and writable cache directory on every node that can receive the workload.
- Label only prepared nodes, for example `benchflow.io/nvme-ready=true`, and require that label through node affinity or node selection.

Use `hostPath.type: Directory` rather than `DirectoryOrCreate` for managed local NVMe paths. It fails safely if the mount is absent; `DirectoryOrCreate` can create a root-owned directory on the operating-system disk and silently invalidate the benchmark.

The profile must not rely on generic GPU scheduling when its storage contract is node-specific.