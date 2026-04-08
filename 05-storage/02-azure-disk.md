# Azure Disk Storage

## Overview

Azure Disk provides **block storage** for pods requiring ReadWriteOnce (single-node) access. It uses Azure Managed Disks and is backed by the **Azure Disk CSI driver** (`disk.csi.azure.com`).

---

## Azure Disk SKUs

| SKU | Type | IOPS (per disk) | Throughput | Use Case |
|-----|------|-----------------|------------|---------|
| Standard HDD | HDD | Up to 500 | Up to 60 MB/s | Dev/test, non-critical |
| Standard SSD | SSD | Up to 6000 | Up to 750 MB/s | Web servers, light DBs |
| Premium SSD | SSD | Up to 20000 | Up to 900 MB/s | Production databases |
| Ultra Disk | NVMe SSD | Up to 160000 | Up to 2000 MB/s | Latency-sensitive |

---

## Custom StorageClass for Azure Disk

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-premium-retain
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS        # Standard_LRS, StandardSSD_LRS, Premium_LRS, UltraSSD_LRS
  cachingmode: ReadOnly       # None, ReadOnly, ReadWrite
  kind: managed               # managed (required for CSI)
  networkAccessPolicy: DenyAll  # Restrict disk network access
  diskEncryptionSetID: /subscriptions/.../diskEncryptionSets/my-des  # CMK encryption
reclaimPolicy: Retain         # Retain data after PVC deletion
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

---

## PVC and Deployment Example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: my-project
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 50Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: my-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
      volumes:
        - name: postgres-data
          persistentVolumeClaim:
            claimName: postgres-data
```

---

## Volume Snapshots

Azure Disk CSI supports the VolumeSnapshot API for point-in-time snapshots:

### Create VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-azure-disk-vsc
driver: disk.csi.azure.com
deletionPolicy: Delete
parameters:
  incremental: "true"       # Azure incremental snapshots (cost-efficient)
  resourceGroup: rg-snapshots  # Optional: store snapshots in specific RG
```

### Take a Snapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-snap-20240101
  namespace: my-project
spec:
  volumeSnapshotClassName: csi-azure-disk-vsc
  source:
    persistentVolumeClaimName: postgres-data
```

```bash
# Check snapshot status
oc get volumesnapshot postgres-snap-20240101
# Wait for readyToUse: true
```

### Restore from Snapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-restored
  namespace: my-project
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: postgres-snap-20240101
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## Common Azure Disk Issues

| Issue | Cause | Fix |
|-------|-------|-----|
| PVC stuck `Pending` | `WaitForFirstConsumer` — no pod yet | Create a pod that references the PVC |
| Disk attach fails | VM disk attachment limit reached | Use larger VM size or fewer disks per node |
| Zone mismatch | Disk in zone 1, pod scheduled to zone 2 | Use `WaitForFirstConsumer` volumeBindingMode |
| Slow provisioning | Ultra Disk has longer provisioning time | Use Premium SSD unless Ultra is required |
