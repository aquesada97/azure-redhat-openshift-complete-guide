# Operators and OLM

## What is an Operator?

An Operator is a Kubernetes controller that extends the Kubernetes API to manage complex, stateful applications. It follows the **Operator Pattern**:

1. Define custom resources (CRDs) to represent the application state
2. A controller watches these resources and reconciles the cluster to the desired state
3. The operator "knows" how to install, upgrade, back up, and recover the application

**Example:** A PostgreSQL Operator knows how to:
- Create a PostgreSQL cluster from a `PostgreSQLCluster` CR
- Add read replicas
- Take backups on a schedule
- Handle failover automatically

---

## Operator Lifecycle Manager (OLM)

OLM is built into OpenShift and manages the lifecycle of operators:

```
OperatorHub (catalog of operators)
      ↓
Subscription (install an operator)
      ↓
InstallPlan (approved installation steps)
      ↓
ClusterServiceVersion / CSV (installed operator metadata)
      ↓
Running Operator Pod(s)
      ↓
Custom Resources (operator-specific CRDs)
```

---

## Installing an Operator via CLI

### Step 1: Create OperatorGroup (namespace scope)

```yaml
apiVersion: operators.coreos.com/v1
kind: OperatorGroup
metadata:
  name: my-project-og
  namespace: my-project
spec:
  targetNamespaces:
    - my-project    # Operator manages resources in this namespace
    # Use [] for all namespaces (cluster-wide operators)
```

### Step 2: Create Subscription

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cert-manager
  namespace: my-project
spec:
  channel: stable            # Release channel
  name: cert-manager         # Operator package name
  source: community-operators  # OperatorHub catalog source
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic  # Automatic or Manual
  # Pin to specific version:
  # startingCSV: cert-manager.v1.13.0
```

### Step 3: Monitor Installation

```bash
# Check subscription
oc get subscription cert-manager -n my-project

# Check install plan
oc get installplan -n my-project

# For Manual approval: approve the install plan
oc patch installplan <plan-name> \
  --type merge \
  --patch '{"spec":{"approved":true}}'

# Check CSV (ClusterServiceVersion)
oc get csv -n my-project

# Check operator pod
oc get pods -n my-project | grep cert-manager
```

---

## OperatorHub Catalog Sources

```bash
# View available catalogs
oc get catalogsource -n openshift-marketplace

# List available operators in a catalog
oc get packagemanifest -n openshift-marketplace | head -30

# Search for an operator
oc get packagemanifest -n openshift-marketplace | grep -i vault

# Get details about an operator (channels, versions)
oc describe packagemanifest vault -n openshift-marketplace
```

---

## Common Operators in ARO

| Operator | Package Name | Source | Purpose |
|----------|-------------|--------|---------|
| Cert Manager | cert-manager | community-operators | TLS certificate automation |
| External Secrets | external-secrets | community-operators | Sync secrets from Azure Key Vault |
| OpenShift GitOps (ArgoCD) | openshift-gitops-operator | redhat-operators | GitOps with ArgoCD |
| OpenShift Pipelines (Tekton) | openshift-pipelines-operator-rh | redhat-operators | CI/CD pipelines |
| OADP (Velero) | oadp-operator | redhat-operators | Backup and restore |
| Cluster Logging | cluster-logging | redhat-operators | Log aggregation |
| Azure Service Operator | azure-service-operator | community-operators | Provision Azure resources from K8s |
| Jaeger | jaeger-product | redhat-operators | Distributed tracing |
| OpenTelemetry Collector | opentelemetry-operator | community-operators | Telemetry collection |

---

## Cluster Operators vs OLM Operators

| Aspect | Cluster Operators | OLM Operators |
|--------|------------------|---------------|
| Installed by | ARO at cluster creation | Customer (via Subscription) |
| Purpose | OpenShift platform components | Additional workload capabilities |
| Managed by | `clusteroperators` API | OLM (csv, subscription) |
| Examples | kube-apiserver, ingress, monitoring | cert-manager, ArgoCD, Velero |
| Updates | Via cluster upgrade | Via Subscription channel |

---

## Troubleshooting Operators

```bash
# Operator installed but not working
oc describe csv cert-manager.v1.13.0 -n my-project

# Check operator pod logs
oc logs -l app.kubernetes.io/name=cert-manager \
  -n my-project --tail=100

# Check if CRDs were created
oc get crd | grep cert-manager

# Operator upgrade stuck
oc get subscription cert-manager -n my-project \
  -o jsonpath='{.status}'

# Check for version conflicts
oc get csv -n my-project
```
