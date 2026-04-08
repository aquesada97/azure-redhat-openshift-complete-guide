# Lab 03: Persistent Storage

## Overview

Deploy stateful applications using Azure Disk and Azure Files. Test data persistence across pod restarts and take a VolumeSnapshot.

**Estimated Time:** 45 minutes  
**Prerequisites:** Lab 01, Module 05 (Storage)

---

## Objectives

- Provision an Azure Disk PVC (ReadWriteOnce) for PostgreSQL
- Provision an Azure Files PVC (ReadWriteMany) for shared file access
- Verify data persists across pod restarts
- Take a VolumeSnapshot and restore from it

---

## Step 1: Create Project

```bash
oc new-project lab-03 --display-name="Lab 03 - Storage"
```

---

## Step 2: Deploy PostgreSQL with Azure Disk

```bash
# Create the PVC first
oc apply -f manifests/pvc-disk.yaml

# Check PVC status
oc get pvc postgres-data
# May be Pending until a pod claims it (WaitForFirstConsumer)

# Create PostgreSQL secret
oc create secret generic postgres-secret \
  --from-literal=password=lab03password

# Deploy PostgreSQL
oc apply -f manifests/postgres-deployment.yaml

# Wait for pod to be running
oc get pods -w
```

---

## Step 3: Test Data Persistence

```bash
# Write data to PostgreSQL
POSTGRES_POD=$(oc get pods -l app=postgres -o name | head -1)
oc exec -it $POSTGRES_POD -- psql -U postgres -c "
  CREATE TABLE test_table (id SERIAL PRIMARY KEY, data TEXT);
  INSERT INTO test_table (data) VALUES ('Hello from lab 03!');
  SELECT * FROM test_table;
"

# Delete the pod (simulates pod restart/eviction)
oc delete pod $POSTGRES_POD

# Wait for new pod to start
oc get pods -w

# Reconnect and verify data persists
NEW_POD=$(oc get pods -l app=postgres -o name | head -1)
oc exec -it $NEW_POD -- psql -U postgres -c "SELECT * FROM test_table;"
# Data should still be there!
```

---

## Step 4: Azure Files (ReadWriteMany)

```bash
# Create Azure Files PVC
oc apply -f manifests/pvc-files.yaml

# Check PVC
oc get pvc shared-files

# Deploy two pods that both mount the same PVC
# (Edit postgres-deployment.yaml replicas=2 to simulate multiple writers with RWX)
# For demonstration, use the shared-files PVC in a deployment
```

---

## Step 5: Take a VolumeSnapshot

```bash
oc apply -f manifests/snapshot.yaml

# Wait for snapshot to be ready
oc get volumesnapshot postgres-snap
# readyToUse: true

# Restore: create a new PVC from the snapshot
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data-restored
  namespace: lab-03
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-csi
  resources:
    requests:
      storage: 10Gi
  dataSource:
    name: postgres-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
EOF

oc get pvc postgres-data-restored
```

---

## Cleanup

```bash
oc delete project lab-03
```
