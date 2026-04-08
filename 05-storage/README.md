# 05 - Storage

## Module Overview

ARO integrates with Azure storage services through the Azure Disk CSI driver and Azure Files CSI driver. This module covers persistent storage patterns, StorageClasses, and advanced storage operations.

---

## Learning Objectives

1. Understand the PV/PVC/StorageClass model in OpenShift
2. Provision Azure Disk storage for ReadWriteOnce workloads
3. Provision Azure Files storage for ReadWriteMany workloads
4. Take VolumeSnapshots and restore from them
5. Resize PVCs dynamically

---

## Topics

| File | Topic |
|------|-------|
| [01-persistent-volumes.md](./01-persistent-volumes.md) | PV, PVC, StorageClass model, access modes |
| [02-azure-disk.md](./02-azure-disk.md) | Azure Disk CSI, snapshots, performance tuning |
| [03-azure-files.md](./03-azure-files.md) | Azure Files CSI, RWX, NFS vs SMB |

---

## Estimated Time

~60 minutes

---

## Next Module

[06 - Security](../06-security/README.md)
