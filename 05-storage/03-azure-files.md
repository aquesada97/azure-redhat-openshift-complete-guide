# Azure Files Storage

## Overview

Azure Files provides **shared file storage (NFS/SMB)** accessible by multiple pods simultaneously (ReadWriteMany). Backed by the **Azure Files CSI driver** (`file.csi.azure.com`).

---

## When to Use Azure Files vs Azure Disk

| Requirement | Azure Disk | Azure Files |
|-------------|-----------|-------------|
| Multiple pods reading/writing simultaneously | No (RWO) | Yes (RWX) |
| Database (single writer) | Yes | No |
| Shared config files | No | Yes |
| CMS media uploads (multiple web pods) | No | Yes |
| Log aggregation (multiple writers) | No | Yes (with care) |
| Performance requirements (high IOPS) | Better | Lower |

---

## Protocol Options

| Protocol | Port | Performance | Use Case |
|----------|------|-------------|---------|
| SMB 3.0 | 445 | Standard | Windows workloads, general use |
| NFS 4.1 | 2049 | Better (no SMB overhead) | Linux workloads, performance-sensitive |

> **NFS is recommended for Linux pods** as it avoids SMB protocol overhead and supports Linux file permissions correctly.

---

## StorageClass for Azure Files with NFS

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-csi-nfs
provisioner: file.csi.azure.com
parameters:
  protocol: nfs           # Use NFS 4.1
  skuName: Premium_LRS    # Standard_LRS, Premium_LRS
  networkEndpointType: privateEndpoint  # Use private endpoint (recommended)
  storeAccountKey: "false"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: Immediate
mountOptions:
  - nconnect=8            # Multiple TCP connections (performance)
  - noatime              # Reduce metadata ops
```

---

## PVC for Azure Files

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-media
  namespace: my-project
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 100Gi
```

---

## Deployment with ReadWriteMany PVC

Multiple replicas can all mount the same Azure Files PVC:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-server
  namespace: my-project
spec:
  replicas: 3              # All 3 pods mount the same PVC
  selector:
    matchLabels:
      app: web-server
  template:
    metadata:
      labels:
        app: web-server
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          volumeMounts:
            - name: media
              mountPath: /usr/share/nginx/html/uploads
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
      volumes:
        - name: media
          persistentVolumeClaim:
            claimName: shared-media
```

---

## Azure Files Premium Performance

| Tier | Min Size | Max IOPS | Max Throughput |
|------|----------|----------|----------------|
| Standard (Transaction Optimized) | 1 GiB | 10,000 | 300 MB/s |
| Standard (Hot) | 1 GiB | 10,000 | 300 MB/s |
| Premium | 100 GiB | 100,000 | 10 GB/s |

> **Premium Azure Files requires a minimum of 100 GiB** allocation.

---

## Troubleshooting Azure Files Mount Issues

```bash
# Check PVC status
oc describe pvc shared-media

# Check CSI driver pods
oc get pods -n openshift-cluster-csi-drivers | grep file

# Check pod events for mount errors
oc describe pod <pod-name>

# Common errors:
# "mount.nfs: Connection refused" — NSG blocking NFS port 2049
# "Permission denied" — SELinux context or UID/GID mismatch
# "Transport endpoint not connected" — SMB session dropped (use NFS instead)

# Fix NFS port in NSG (if using private endpoint, add rule to allow port 2049)
az network nsg rule create \
  --resource-group $RESOURCEGROUP \
  --nsg-name worker-nsg \
  --name allow-nfs \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 2049
```
