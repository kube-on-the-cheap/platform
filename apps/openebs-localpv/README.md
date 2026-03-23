# OpenEBS LocalPV

Local persistent storage for Kubernetes clusters using [OpenEBS](https://openebs.io/) with the rawfile CSI driver.

## Architecture

The deployment uses the OpenEBS Helm chart (`openebs ~4.4`) managed by FluxCD, with only the **rawfile** engine enabled. LVM, ZFS, and Mayastor engines are disabled.

### Components

- **rawfile-localpv-controller** -- StatefulSet (1 replica) managing volume lifecycle
- **rawfile-localpv-node** -- DaemonSet running on all nodes (including control-plane via tolerations)
- **localpv-provisioner** -- Handles hostPath provisioning (not actively used)

### StorageClass

| Name | Provisioner | Default | Volume Binding | Expansion |
|------|-------------|---------|----------------|-----------|
| `openebs-rawfile` | `rawfile.csi.openebs.io` | Yes | `WaitForFirstConsumer` | Yes |

Parameters: `basePath=/var/mnt/openebs-rawfiles`, `fsType=xfs`.

The `openebs-lvm` StorageClass created by the chart is deleted via a kustomize `$patch: delete` in the overlay.

## Directory Structure

```
apps/openebs-localpv/
  base/
    helmrelease.yaml        # HelmRelease (chart: openebs ~4.4, namespace: openebs)
    helmrepository.yaml     # HelmRepository (openebs.github.io/openebs)
    kustomization.yaml
    namespace.yaml          # Namespace with privileged PodSecurity
  overlays/
    understairs/
      kustomization.yaml
      kustomizeconfig.yaml  # nameReference for ConfigMap -> HelmRelease valuesFrom
      openebs-localpv-values.yaml  # Helm values (rawfile only, Talos tolerations)
      storageclass.yaml     # openebs-rawfile StorageClass definition
    freeloader/
      ...                   # Separate cluster overlay
```

## Talos Prerequisites

The rawfile CSI driver requires a dedicated volume mounted at `/var/mnt/openebs-rawfiles` on each node. This is configured outside this repository in the Talos machine configuration:

**Repository:** `onprem-infra/config/understairs/talos/`

- `minipc1-volumes.yaml` -- `UserVolumeConfig` that provisions an ext4 filesystem on an NVMe disk (1 GiB - 750 GiB) via Talos disk selector (`disk.transport == "nvme"`)

This volume is mounted by Talos before kubelet starts, making it available for the rawfile CSI node plugin to create sparse files backing each PersistentVolume.

## Notes

- The rawfile node DaemonSet tolerates `node-role.kubernetes.io/control-plane` taints, so it runs on all nodes. This is acceptable because `WaitForFirstConsumer` binding ensures PVCs only provision on nodes where workloads actually schedule.
- Monitoring (Alloy, Loki) is disabled in the Helm values; the cluster uses a centralized Grafana Alloy deployment instead.
