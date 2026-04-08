# Creating an ARO Cluster

## Environment Variables

Set these before running any commands. See [00-prerequisites](../00-prerequisites/README.md) for full setup.

### Bash
```bash
export RESOURCEGROUP="rg-aro-training"
export LOCATION="eastus"
export CLUSTER="aro-training-cluster"
export VNET_NAME="vnet-aro"
export MASTER_SUBNET="subnet-master"
export WORKER_SUBNET="subnet-worker"
export PULL_SECRET_FILE="$HOME/.azure/pull-secret.txt"
```

### PowerShell
```powershell
$RESOURCEGROUP = "rg-aro-training"
$LOCATION = "eastus"
$CLUSTER = "aro-training-cluster"
$VNET_NAME = "vnet-aro"
$MASTER_SUBNET = "subnet-master"
$WORKER_SUBNET = "subnet-worker"
$PULL_SECRET_FILE = "$HOME\.azure\pull-secret.txt"
```

---

## The az aro create Command

```bash
az aro create \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --vnet $VNET_NAME \
  --master-subnet $MASTER_SUBNET \
  --worker-subnet $WORKER_SUBNET \
  --pull-secret @$PULL_SECRET_FILE \
  --worker-vm-size Standard_D4s_v3 \
  --worker-count 3 \
  --master-vm-size Standard_D8s_v3 \
  --apiserver-visibility Public \
  --ingress-visibility Public
```

### Flag Reference

| Flag | Required | Description | Default |
|------|----------|-------------|---------|
| `--resource-group` | Yes | Resource group for customer resources | - |
| `--name` | Yes | Cluster name (1-30 chars, alphanumeric + hyphen) | - |
| `--vnet` | Yes | VNet name or resource ID | - |
| `--master-subnet` | Yes | Subnet name or resource ID for master nodes | - |
| `--worker-subnet` | Yes | Subnet name or resource ID for worker nodes | - |
| `--pull-secret` | Recommended | Path to Red Hat pull secret file (prefix with @) | None |
| `--domain` | No | Custom domain prefix for cluster hostname | Cluster name |
| `--worker-vm-size` | No | VM size for worker nodes | Standard_D4s_v3 |
| `--worker-count` | No | Number of initial worker nodes | 3 |
| `--master-vm-size` | No | VM size for master nodes | Standard_D8s_v3 |
| `--apiserver-visibility` | No | API server visibility: Public or Private | Public |
| `--ingress-visibility` | No | App ingress visibility: Public or Private | Public |
| `--version` | No | Specific OCP version to deploy | Latest stable |
| `--client-id` | No | Service principal client ID | Auto-created |
| `--client-secret` | No | Service principal client secret | Auto-created |
| `--pod-cidr` | No | CIDR for pod network | 10.128.0.0/14 |
| `--service-cidr` | No | CIDR for service network | 172.30.0.0/16 |
| `--disk-encryption-set` | No | Customer-managed key disk encryption set ID | None |
| `--fips-validated-modules` | No | Enable FIPS validated cryptographic modules | false |
| `--tags` | No | Space-separated key=value tags | None |

---

## Monitoring Cluster Provisioning

Cluster creation takes approximately **35-45 minutes**.

```bash
# Check provisioning state (run in a separate terminal)
watch -n 30 "az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --query provisioningState -o tsv"

# Possible values: Creating → Succeeded (or Failed)
```

---

## Getting Cluster Endpoints

After provisioning completes:

```bash
# Console URL
az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --query consoleProfile.url -o tsv

# API server URL
az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --query apiserverProfile.url -o tsv

# kubeadmin credentials
az aro list-credentials \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER

# Full cluster details
az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --output table
```

---

## Troubleshooting Common Errors

| Error | Cause | Resolution |
|-------|-------|------------|
| `QuotaExceeded` | Insufficient vCPU quota | Request quota increase in Azure Portal |
| `InvalidCIDRNotation` | Subnet CIDR overlap or invalid | Check VNet address space, no overlaps |
| `AuthorizationFailed` | Missing SP permissions | Ensure Contributor + UAA roles on subscription |
| `ResourceProviderNotRegistered` | Resource provider not registered | Run `az provider register` commands |
| `SubnetNotFound` | Subnet doesn't exist in VNet | Verify VNet and subnet names |
| `PullSecretInvalid` | Invalid or expired pull secret | Re-download from cloud.redhat.com |

---

## Deleting the Cluster

```bash
# Delete cluster (takes ~10 min, deletes all worker VMs and ARO resources)
az aro delete \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --yes

# Delete the resource group (cleans up VNet and all remaining resources)
az group delete \
  --name $RESOURCEGROUP \
  --yes --no-wait
```

> **Warning:** Cluster deletion is irreversible. Ensure you have backups of any persistent data before deleting.
