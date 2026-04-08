# ARO Architecture

## High-Level Architecture

```
+----------------------------------------------------------+
|                    Azure Subscription                     |
|                                                           |
|  +-------------------+    +---------------------------+  |
|  | Customer RG        |    | ARO-Managed RG            |  |
|  | (rg-aro-training)  |    | (aro-<uuid>-<cluster>)    |  |
|  |                   |    |                           |  |
|  | - VNet            |    | - Master VMs (3x)         |  |
|  | - Worker VMs      |    | - Infra VMs (3x)          |  |
|  | - Worker Disks    |    | - Internal LB             |  |
|  | - Public LB       |    | - Private DNS zones       |  |
|  | - NSGs            |    | - Storage accounts        |  |
|  +-------------------+    +---------------------------+  |
|           |                         |                    |
|           +------VNet Peering/-------+                    |
|                   Subnets                                 |
+----------------------------------------------------------+

                    +-----------------+
                    |  ARO Control    |
                    |     Plane       |
                    | (Managed by     |
                    |  MS + Red Hat)  |
                    +-----------------+
                           |
              +------------+------------+
              |            |            |
         [Master-0]   [Master-1]   [Master-2]
              |
    +---------+----------+
    |                    |
[Worker-0..N]    [Infra-0..2]
(Customer        (Router, Registry,
 workloads)       Monitoring)
```

---

## Managed Control Plane

ARO's control plane is fully managed — you cannot SSH into master nodes or modify them directly.

### Who Manages What

| Component | Managed By | Customer Control |
|-----------|-----------|-----------------|
| Master node VMs | Microsoft + Red Hat | None |
| etcd | Microsoft + Red Hat | None |
| kube-apiserver | Microsoft + Red Hat | None |
| OpenShift upgrades | Customer-initiated, MS+RH executed | Version selection |
| Worker node VMs | Customer (Machine API) | Full control |
| Worker node count | Customer (MachineSet) | Full control |
| Worker node size | Customer (MachineSet) | Full control |
| Application workloads | Customer | Full control |
| Network policies | Customer | Full control |

### Master Nodes

ARO deploys **3 master nodes** in an availability zone spread across 3 Azure zones (in regions that support zones).

Default master VM size: **Standard_D8s_v3** (8 vCPU, 32 GB RAM)

What runs on master nodes:
- etcd (distributed key-value store, quorum requires 3 nodes)
- kube-apiserver (Kubernetes API server)
- kube-controller-manager
- kube-scheduler
- openshift-apiserver (OpenShift-specific API extensions)
- oauth-server
- cluster-version-operator (manages OCP version)

### Infrastructure Nodes

ARO also deploys **3 infrastructure nodes** that run OpenShift platform services (not your workloads):

| Service | Namespace |
|---------|-----------|
| OpenShift Router (HAProxy) | openshift-ingress |
| Internal Image Registry | openshift-image-registry |
| Monitoring (Prometheus) | openshift-monitoring |
| Logging | openshift-logging |
| OLM (Operator Lifecycle Manager) | openshift-operator-lifecycle-manager |

Default infra VM size: **Standard_D4s_v3** (4 vCPU, 16 GB RAM)

---

## Worker Node Pools

Worker nodes run your application workloads. They are defined and managed via **MachineSets**.

### Default Worker Configuration

| Parameter | Default Value |
|-----------|--------------|
| VM Size | Standard_D4s_v3 |
| Node Count | 3 |
| OS Disk | 128 GB Premium SSD |
| Availability Zones | Zone spread (1, 2, 3) |
| OS | RHCOS (Red Hat CoreOS) |

### Scale Workers via Azure CLI
```bash
# Scale the worker MachineSet to 5 nodes
az aro update \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --worker-count 5
```

### Scale Workers via Machine API
```bash
# List MachineSets
oc get machinesets -n openshift-machine-api

# Scale a specific MachineSet
oc scale machineset <machineset-name> \
  --replicas=5 \
  -n openshift-machine-api
```

---

## Azure Resource Group Structure

When you create an ARO cluster, two resource groups are used:

### 1. Customer Resource Group (`rg-aro-training`)
Resources you create and control:
- Virtual Network (VNet)
- Master and Worker subnets
- Worker VMs (owned by you, managed via Machine API)
- Network Security Groups (NSGs)
- Public Load Balancer (for ingress)

### 2. ARO-Managed Resource Group (`aro-<uuid>-<cluster>-<region>`)
Resources created and managed by ARO service:
- Master VM instances
- Infrastructure node VMs
- Private Load Balancers (for internal API server)
- Private DNS zones (for cluster-internal resolution)
- Storage accounts (for image registry, etcd backups)
- Managed Disks for control plane

> **Important:** Never manually modify resources in the ARO-managed resource group. The ARO service will reconcile any manual changes back, potentially causing cluster instability.

```bash
# View the managed resource group name
az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --query clusterProfile.resourceGroupId -o tsv
```

---

## Virtual Network Requirements

| Subnet | Minimum CIDR | Purpose | Constraints |
|--------|-------------|---------|-------------|
| Master subnet | /23 | Master nodes + infra nodes | Must have `privateLinkServiceNetworkPolicies: Disabled` |
| Worker subnet | /23 | Worker nodes | No special constraints |

### Pod and Service CIDRs

| Network | Default CIDR | Notes |
|---------|-------------|-------|
| Pod network | 10.128.0.0/14 | OVN-Kubernetes pod overlay |
| Service network | 172.30.0.0/16 | ClusterIP service range |

### Custom Pod/Service CIDRs (at cluster creation time only)
```bash
az aro create \
  --pod-cidr 10.200.0.0/14 \
  --service-cidr 172.31.0.0/16 \
  # ... other flags
```

> These CIDRs cannot be changed after cluster creation.

---

## Private vs Public Cluster

| Aspect | Public Cluster | Private Cluster |
|--------|---------------|----------------|
| API server | Internet-accessible | VNet-only |
| Ingress (apps) | Internet-accessible | VNet-only |
| Use case | Dev/test, public apps | Enterprise, regulated |
| DNS resolution | Public DNS | Azure Private DNS |
| Access from on-prem | Direct | Requires VPN/ExpressRoute |

### Create a Private Cluster
```bash
az aro create \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --vnet $VNET_NAME \
  --master-subnet $MASTER_SUBNET \
  --worker-subnet $WORKER_SUBNET \
  --apiserver-visibility Private \
  --ingress-visibility Private \
  --pull-secret @$PULL_SECRET_FILE
```

---

## Cluster Version and Lifecycle

### Check Current Version
```bash
oc get clusterversion
oc get clusterversion -o jsonpath='{.items[0].status.desired.version}'
```

### Update Channels

| Channel | Description | Recommended For |
|---------|-------------|----------------|
| `stable-4.x` | Stable, well-tested releases | Production |
| `fast-4.x` | Newer releases, less soak time | Staging |
| `candidate-4.x` | Release candidates | Testing only |
| `eus-4.x` | Extended Update Support (even versions) | Long-term stability |

### Check Available Updates
```bash
oc adm upgrade
```

### Trigger Upgrade
```bash
# Via Azure CLI (recommended for ARO)
az aro update \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --version 4.15.0

# Via oc (also works)
oc adm upgrade --to=4.15.0
```
