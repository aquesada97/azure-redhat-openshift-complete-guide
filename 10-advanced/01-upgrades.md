# ARO Cluster Upgrades

## ARO Upgrade Model

ARO uses a **customer-initiated, service-executed** upgrade model:
- **You decide** when to upgrade and to which version
- **ARO service** (Microsoft + Red Hat) executes the upgrade safely
- Upgrades are rolling: control plane first, then workers
- Estimated time: 60-90 minutes for a standard upgrade

---

## Update Channels

| Channel | Description | Recommended For |
|---------|-------------|----------------|
| `stable-4.x` | Stable releases, well-tested | Production |
| `fast-4.x` | Newer releases, less soak time | Staging/Pre-prod |
| `candidate-4.x` | Release candidates | Testing only, not production |
| `eus-4.x` | Extended Update Support (even minor versions) | Long-term stability |

```bash
# View available versions and channels
oc adm upgrade

# Change update channel
oc patch clusterversion version \
  --type merge \
  --patch '{"spec":{"channel":"stable-4.15"}}'
```

---

## Checking Current Version

```bash
# Current cluster version
oc get clusterversion

# Detailed version info
oc describe clusterversion

# Available updates
oc adm upgrade

# All Cluster Operator versions
oc get clusteroperators -o custom-columns="NAME:.metadata.name,VERSION:.status.versions[0].version"
```

---

## Triggering an Upgrade

```bash
# Method 1: Azure CLI (recommended for ARO)
az aro update \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --version 4.15.0

# Method 2: oc CLI
oc adm upgrade --to=4.15.0

# Force upgrade to a specific image (for hotfixes)
oc adm upgrade --to-image=quay.io/openshift-release-dev/ocp-release@sha256:<digest> --force

# View available minor versions
az aro get-upgrade-versions \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER
```

---

## Pre-Upgrade Checklist

```bash
# 1. Check all Cluster Operators are healthy
oc get clusteroperators
# All should be: AVAILABLE=True, PROGRESSING=False, DEGRADED=False

# 2. Check all nodes are Ready
oc get nodes
# All should be in Ready state

# 3. Check for PodDisruptionBudgets that might block node drain
oc get pdb --all-namespaces

# 4. Check for pending pods that might block upgrades
oc get pods --all-namespaces --field-selector=status.phase!=Running,status.phase!=Succeeded | grep -v Completed

# 5. Verify etcd is healthy
oc get clusteroperator etcd
# Should be: AVAILABLE=True, DEGRADED=False

# 6. Review release notes for breaking changes
# https://docs.openshift.com/container-platform/latest/release_notes/ocp-4-15-release-notes.html
```

---

## Monitoring Upgrade Progress

```bash
# Watch Cluster Operators (main indicator of upgrade progress)
watch -n 30 "oc get clusteroperators"

# Watch nodes being updated
oc get nodes -w

# Check upgrade progress
oc get clusterversion -o jsonpath='{.items[0].status.conditions}' | python3 -m json.tool

# View upgrade history
oc get clusterversion -o jsonpath='{.items[0].status.history}' | python3 -m json.tool

# Watch MachineConfigPool for worker node updates
oc get machineconfigpool -w
```

---

## Post-Upgrade Validation

```bash
# 1. All nodes Ready
oc get nodes

# 2. All Cluster Operators healthy
oc get clusteroperators

# 3. Check new version
oc get clusterversion

# 4. Verify key workloads still running
oc get pods --all-namespaces | grep -v Running | grep -v Completed | grep -v Succeeded

# 5. Test application routes
curl https://$(oc get route my-app -n my-project -o jsonpath='{.spec.host}')

# 6. Check no deprecated API usage
oc get apirequestcount --sort-by=.status.requestCount | tail -20
```

---

## EUS-to-EUS Upgrades

Extended Update Support allows upgrading across two minor versions (e.g., 4.12 → 4.14):

```bash
# Check EUS compatibility
oc adm upgrade --include-not-recommended

# EUS upgrade path: 4.12 → 4.13 → 4.14 (or use eus-4.14 channel directly)
# Switch to EUS channel
oc patch clusterversion version \
  --type merge \
  --patch '{"spec":{"channel":"eus-4.14"}}'

# Then upgrade
oc adm upgrade --to=4.14.0
```

---

## Rollback

> **Important: ARO upgrades cannot be rolled back** after the upgrade completes. The cluster is committed to the new version.

This is why pre-upgrade validation is critical. If an issue is found post-upgrade:
1. Check if it's a known bug with a fix in a newer patch release
2. Apply the patch upgrade to the newer version
3. Contact Red Hat support if the issue is blocking

---

## Maintenance Windows

```bash
# ARO supports upgrade windows (Azure CLI)
az aro update \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --version 4.15.0

# The upgrade starts immediately when triggered
# Schedule it during your maintenance window by triggering at the right time
```
