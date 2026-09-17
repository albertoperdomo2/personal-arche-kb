# Podman machine disk image never shrinks without fstrim

## Finding

A `podman machine` VM on macOS (applehv provider) backs its root filesystem with a raw disk image:

```
~/.local/share/containers/podman/machine/applehv/podman-machine-default-arm64.raw
```

The file is created sparse at the machine's full declared disk size (100 GiB by default). It grows as blocks are written inside the guest, but **it never shrinks when data is deleted inside the guest**. Pruning images, removing containers, or deleting volumes frees space in the guest filesystem while the host file keeps every block it has ever touched.

Observed on 2026-09-17:

| Measurement | Value |
|---|---|
| Host file size on disk (`du -sh`) | 99 GB |
| Guest filesystem used (`df -h /` inside VM) | 16 GB |
| `podman system df` reclaimable | 7.2 GB |

The host was at 91% capacity with 42 GiB free while the guest reported 84 GiB free. `podman system prune` would have recovered at most 7 GB and would not have touched the host file at all.

## Fix

Issue a filesystem trim inside the guest. The virtio block driver forwards the discard to the host, which punches holes back into the sparse raw file:

```bash
podman machine ssh 'sudo fstrim -av'
```

Output:

```
/: 99.5 GiB (106836242432 bytes) trimmed on /dev/vda4
```

The host file dropped from 99 GB to 15 GB immediately. Verify with:

```bash
du -sh ~/.local/share/containers/podman/machine/applehv/*.raw
```

This is **non-destructive**. The machine keeps running, images and volumes survive, and no rebuild is needed.

## Rule

When a Mac is low on disk and `~/.local/share/containers` is large, run `fstrim` inside the machine *before* considering `podman machine reset` or `podman system prune`. The container data is usually not the problem; the un-trimmed sparse image is.

Worth running periodically on any long-lived machine, since the file only ratchets upward. The same reasoning applies to other VM-backed container runtimes on macOS that use raw or qcow2 images, including Colima and Lima.

Note that `du --apparent-size` and `ls -l` both report the full declared size and will not reveal the problem. Compare plain `du -sh` on the host against `df -h /` inside the guest to spot the gap.