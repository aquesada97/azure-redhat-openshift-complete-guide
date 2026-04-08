# Disaster Recovery for ARO

## ARO SLA and What's Covered

ARO provides a **99.95% SLA** for the cluster control plane API. The SLA covers:
- API server availability
- Control plane health
- Automatic master node replacement
- etcd backup management

**Not covered by ARO SLA:**
- Application data (your responsibility)
- Worker node data (back up with Velero/OADP)
- Persistent volume data (back up with VolumeSnapshots or Velero)

---

## etcd Backup

ARO automatically backs up etcd. In a cluster failure requiring full restore, contact Microsoft/Red Hat support.

```bash
# View etcd backup status (ARO manages this automatically)
oc get etcdbackup -n openshift-etcd

# Check etcd health
oc get clusteroperator etcd
```

---

## Application Backup with OADP (Velero)

**OADP (OpenShift API for Data Protection)** is the Red Hat-supported Velero operator for ARO.

### Install OADP Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: redhat-oadp-operator
  namespace: openshift-adp
spec:
  channel: stable
  name: redhat-oadp-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Create Azure Storage for Backups

```bash
# Create a storage account and container
BACKUP_SA="arobackups$(date +%s)"
az storage account create \
  --name $BACKUP_SA \
  --resource-group $RESOURCEGROUP \
  --location $LOCATION \
  --sku Standard_GRS   # Geo-redundant for DR

az storage container create \
  --name velero \
  --account-name $BACKUP_SA

# Create service principal for Velero
SP_CREDS=$(az ad sp create-for-rbac \
  --name "sp-velero" \
  --role Contributor \
  --scopes /subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP \
  --output json)

# Create credentials secret
cat > /tmp/velero-credentials <<EOF
AZURE_SUBSCRIPTION_ID=${AZURE_SUBSCRIPTION_ID}
AZURE_TENANT_ID=$(echo $SP_CREDS | jq -r .tenant)
AZURE_CLIENT_ID=$(echo $SP_CREDS | jq -r .appId)
AZURE_CLIENT_SECRET=$(echo $SP_CREDS | jq -r .password)
AZURE_RESOURCE_GROUP=${RESOURCEGROUP}
AZURE_CLOUD_NAME=AzurePublicCloud
EOF

oc create secret generic cloud-credentials \
  --from-file=cloud=/tmp/velero-credentials \
  -n openshift-adp
```

### DataProtectionApplication CR

```yaml
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: aro-dpa
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - azure
        - openshift      # Required for OpenShift resources (routes, SCCs, etc.)
        - csi            # For CSI volume snapshots
      resourceTimeout: 10m
    restic:
      enable: false      # Use CSI snapshots instead
  backupLocations:
    - name: default
      velero:
        provider: azure
        default: true
        objectStorage:
          bucket: velero
          prefix: aro-cluster
        config:
          resourceGroup: $RESOURCEGROUP
          storageAccount: $BACKUP_SA
          subscriptionId: $AZURE_SUBSCRIPTION_ID
  snapshotLocations:
    - name: default
      velero:
        provider: azure
        config:
          resourceGroup: $RESOURCEGROUP
          subscriptionId: $AZURE_SUBSCRIPTION_ID
          incremental: "true"
```

---

## Creating a Backup

```yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: my-project-backup-20240101
  namespace: openshift-adp
spec:
  includedNamespaces:
    - my-project
  storageLocation: default
  snapshotVolumes: true
  ttl: 720h    # Keep for 30 days
  labelSelector:
    matchLabels:
      backup: "true"    # Optional: only backup labeled resources
```

```bash
# Apply the backup
oc apply -f backup.yaml -n openshift-adp

# Check backup status
oc get backup my-project-backup-20240101 -n openshift-adp
oc describe backup my-project-backup-20240101 -n openshift-adp
```

---

## Backup Schedule

```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: daily-backup
  namespace: openshift-adp
spec:
  schedule: "0 2 * * *"    # 2 AM UTC daily
  template:
    includedNamespaces:
      - my-project
      - database-project
    storageLocation: default
    snapshotVolumes: true
    ttl: 720h
```

---

## Restoring from Backup

```yaml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: my-project-restore
  namespace: openshift-adp
spec:
  backupName: my-project-backup-20240101
  includedNamespaces:
    - my-project
  restorePVs: true
  # Restore to a different namespace:
  # namespaceMapping:
  #   my-project: my-project-restored
```

```bash
# Apply restore
oc apply -f restore.yaml -n openshift-adp

# Monitor restore progress
oc get restore my-project-restore -n openshift-adp -w
oc describe restore my-project-restore -n openshift-adp
```

---

## RTO/RPO Considerations

| Scenario | RPO | RTO | Strategy |
|----------|-----|-----|---------|
| Pod failure | N/A | Seconds | K8s self-healing |
| Node failure | N/A | ~5 min | MachineHealthCheck |
| Namespace data loss | Last backup | ~30 min | Velero restore |
| Full cluster failure | Last backup | ~60 min | New cluster + Velero restore |
| Region failure | Last geo-backup | Hours | Cross-region cluster + Azure GRS |

---

## Active-Passive DR

```
Region East US (Primary)
├── ARO Cluster (active)
├── Azure Database for PostgreSQL
└── Azure Blob Storage (GRS — geo-replicated)

Region West US 2 (DR)
├── ARO Cluster (standby, can be scaled from 0)
└── Azure Database for PostgreSQL (read replica)

Azure Traffic Manager (DNS-level failover)
├── Primary: East US ARO ingress
└── Failover: West US 2 ARO ingress
```
