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

## Rule

Do not use a local-NVMe hostPath profile without one of these explicit contracts:

- Provision and validate the mount and writable cache directory on every node that can receive the workload.
- Label only prepared nodes, for example `benchflow.io/nvme-ready=true`, and require that label through node affinity or node selection.

Use `hostPath.type: Directory` rather than `DirectoryOrCreate` for managed local NVMe paths. It fails safely if the mount is absent; `DirectoryOrCreate` can create a root-owned directory on the operating-system disk and silently invalidate the benchmark.

## Immediate Diadochos options

- Prepare one unused NVMe and the cache directory on `gjfjh`, then allow scheduling there.
- Or constrain the NVMe experiment to the already prepared nodes and rerun the failed child.

The profile must not rely on generic GPU scheduling when its storage contract is node-specific.