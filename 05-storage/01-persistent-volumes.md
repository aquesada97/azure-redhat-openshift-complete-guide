# Persistent Volumes in ARO

## The PV/PVC/StorageClass Model

Kubernetes uses a three-layer abstraction for persistent storage:

```
StorageClass → (dynamic provisioning) → PersistentVolume (PV)
                                              ↑ bound
                                    PersistentVolumeClaim (PVC)
                                              ↑ mounted in
                                              Pod
```

| Object | Description | Who Manages |
|--------|-------------|-------------|
| `StorageClass` | Template for provisioning storage | Cluster admin |
| `PersistentVolume (PV)` | Actual storage resource | Cluster admin (static) or provisioner (dynamic) |
| `PersistentVolumeClaim (PVC)` | Request for storage by an app | Developer |

---

## ARO Built-In StorageClasses

```bash
oc get storageclass
```

| StorageClass | Provisioner | Access Mode | Use Case |
|-------------|-------------|-------------|---------|
| `managed-csi` | disk.csi.azure.com | RWO | General purpose (Standard SSD) |
| `managed-premium` | disk.csi.azure.com | RWO | High-performance (Premium SSD) |
| `azurefile-csi` | file.csi.azure.com | RWO/RWX/ROX | Shared storage (Standard) |
| `azurefile-csi-premium` | file.csi.azure.com | RWO/RWX/ROX | Shared storage (Premium) |
| `managed` (legacy) | kubernetes.io/azure-disk | RWO | Legacy, prefer managed-csi |
| `azurefile` (legacy) | kubernetes.io/azure-file | RWX | Legacy, prefer azurefile-csi |

---

## Access Modes

| Mode | Short | Description | Azure Storage |
|------|-------|-------------|---------------|
| `ReadWriteOnce` | RWO | One node read/write | Azure Disk |
| `ReadWriteMany` | RWX | Multiple nodes read/write | Azure Files |
| `ReadOnlyMany` | ROX | Multiple nodes read-only | Azure Files |
| `ReadWriteOncePod` | RWOP | One pod read/write (K8s 1.22+) | Azure Disk CSI |

---

## PVC for Azure Disk (ReadWriteOnce)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data
  namespace: my-project
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-premium    # Premium SSD
  resources:
    requests:
      storage: 20Gi
```

---

## PVC for Azure Files (ReadWriteMany)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-files
  namespace: my-project
spec:
  accessModes:
    - ReadWriteMany             # Multiple pods can mount simultaneously
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 50Gi
```

---

## Using a PVC in a Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-with-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app-with-storage
  template:
    metadata:
      labels:
        app: my-app-with-storage
    spec:
      containers:
        - name: my-app
          image: my-app:latest
          volumeMounts:
            - name: app-data
              mountPath: /app/data
      volumes:
        - name: app-data
          persistentVolumeClaim:
            claimName: app-data    # References the PVC
```

---

## VolumeBindingMode

| Mode | Behavior |
|------|---------|
| `Immediate` | PV provisioned as soon as PVC is created (default for most classes) |
| `WaitForFirstConsumer` | PV provisioned when a pod is scheduled (ensures zone alignment) |

> **Azure Disk with WaitForFirstConsumer** is recommended for zonal clusters. This prevents a disk from being provisioned in zone 1 when the pod schedules to zone 2.

```bash
# Check if a StorageClass uses WaitForFirstConsumer
oc get storageclass managed-csi -o jsonpath='{.volumeBindingMode}'
```

---

## PVC Operations

```bash
# List PVCs
oc get pvc

# Describe a PVC (check status, events)
oc describe pvc my-pvc

# Check PV bound to a PVC
oc get pvc my-pvc -o jsonpath='{.spec.volumeName}'

# Resize a PVC (storage class must support expansion)
oc patch pvc my-pvc \
  --type merge \
  --patch '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# Check if StorageClass allows expansion
oc get storageclass managed-csi -o jsonpath='{.allowVolumeExpansion}'
```

---

## PV Lifecycle

```
Provisioning → Binding → Using → Released → Reclaiming
```

### Reclaim Policies

| Policy | Behavior After PVC Deletion |
|--------|----------------------------|
| `Delete` | PV and underlying Azure resource deleted | 
| `Retain` | PV stays, data preserved, must be manually reclaimed |

```bash
# Change reclaim policy on an existing PV
oc patch pv <pv-name> \
  --type merge \
  --patch '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```
