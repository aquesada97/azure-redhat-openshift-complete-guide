# 00 - Prerequisites

## Overview

Before creating your first ARO cluster, ensure all tools, accounts, and Azure resources are properly configured. This module walks through every prerequisite step.

---

## 1. Azure CLI Installation

### Linux (Ubuntu/Debian)
```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### Windows (PowerShell)
```powershell
winget install --exact --id Microsoft.AzureCLI
# Or download the MSI from: https://aka.ms/installazurecliwindows
```

### Verify
```bash
az version
```

### Install the ARO extension
```bash
az extension add --name aro
az extension update --name aro
az extension show --name aro
```

---

## 2. OpenShift CLI (oc) Installation

The `oc` CLI is a superset of `kubectl`. Download it from the OpenShift mirror or your cluster's console.

### Linux
```bash
curl -LO https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
tar -xvf openshift-client-linux.tar.gz
sudo mv oc kubectl /usr/local/bin/
oc version --client
```

### Windows (PowerShell)
```powershell
$ocUrl = "https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-windows.zip"
Invoke-WebRequest -Uri $ocUrl -OutFile oc-windows.zip
Expand-Archive oc-windows.zip -DestinationPath C:\tools\oc
$env:PATH += ";C:\tools\oc"
oc version --client
```

### From the ARO web console
1. Log into the console
2. Click **?** (Help) → **Command Line Tools**
3. Download the binary for your OS

---

## 3. kubectl Installation

`kubectl` is optional since `oc` is a full superset. To install standalone:

```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# Windows via winget
winget install --exact --id Kubernetes.kubectl
```

---

## 4. Red Hat Pull Secret

The pull secret grants access to Red Hat container images.

1. Go to [https://cloud.redhat.com/openshift/install/azure/aro-provisioned](https://cloud.redhat.com/openshift/install/azure/aro-provisioned)
2. Log in with your Red Hat account (create one if needed — it's free)
3. Click **Download pull secret** and save to `~/.azure/pull-secret.txt`

> **Note:** The pull secret is required for `az aro create`. Without it, your cluster will not have access to Red Hat-certified operator content in OperatorHub.

---

## 5. VS Code Extensions (Recommended)

| Extension | Publisher | Purpose |
|-----------|-----------|---------|
| OpenShift Connector | Red Hat | Manage OpenShift resources from VS Code |
| YAML | Red Hat | Schema validation for Kubernetes/OpenShift YAML |
| Kubernetes | Microsoft | Cluster explorer, port-forward, log streaming |
| Azure CLI Tools | Microsoft | Inline az command help |

---

## 6. Azure Subscription Requirements

### Required Roles
| Role | Scope | Purpose |
|------|-------|---------|
| Contributor | Subscription | Create VNets, resource groups, clusters |
| User Access Administrator | Subscription | Assign roles to ARO service principal |

### Check your role assignment
```bash
az role assignment list --assignee $(az account show --query user.name -o tsv) \
  --query "[].{Role:roleDefinitionName, Scope:scope}" -o table
```

---

## 7. Register Resource Providers

ARO requires several Azure resource providers to be registered:

```bash
az provider register --namespace 'Microsoft.RedHatOpenShift' --wait
az provider register --namespace 'Microsoft.Compute' --wait
az provider register --namespace 'Microsoft.Storage' --wait
az provider register --namespace 'Microsoft.Network' --wait
az provider register --namespace 'Microsoft.Authorization' --wait

# Verify
az provider show --namespace Microsoft.RedHatOpenShift --query registrationState -o tsv
```

---

## 8. Azure Quota Requirements

ARO requires a minimum of **40 vCPUs** in your target region. Default Azure subscriptions often start with lower limits.

| VM Family | Minimum Required | Purpose |
|-----------|-----------------|---------|
| Standard DSv3 Family | 40 vCPUs | Default master + worker SKUs |
| Standard DSv4 Family | 40 vCPUs | (if using D4s_v4 workers) |
| Standard DSv5 Family | 40 vCPUs | (if using D4s_v5 workers) |

### Check current quota
```bash
REGION="eastus"
az vm list-usage --location $REGION \
  --query "[?contains(name.value,'standardDSv3Family')]" \
  --output table
```

### Request quota increase
1. In the Azure Portal, go to **Subscriptions → Your Subscription → Usage + quotas**
2. Filter by compute and your region
3. Click on the VM family and request an increase
4. Approval typically takes a few minutes to a few hours

---

## 9. Environment Variables Setup

### Bash/Linux/macOS
```bash
export AZURE_SUBSCRIPTION_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export AZURE_TENANT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export RESOURCEGROUP="rg-aro-training"
export LOCATION="eastus"
export CLUSTER="aro-training-cluster"
export VNET_NAME="vnet-aro"
export MASTER_SUBNET="subnet-master"
export WORKER_SUBNET="subnet-worker"
export PULL_SECRET_FILE="$HOME/.azure/pull-secret.txt"

# Set active subscription
az account set --subscription $AZURE_SUBSCRIPTION_ID
```

### PowerShell/Windows
```powershell
$AZURE_SUBSCRIPTION_ID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$AZURE_TENANT_ID = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$RESOURCEGROUP = "rg-aro-training"
$LOCATION = "eastus"
$CLUSTER = "aro-training-cluster"
$VNET_NAME = "vnet-aro"
$MASTER_SUBNET = "subnet-master"
$WORKER_SUBNET = "subnet-worker"
$PULL_SECRET_FILE = "$HOME\.azure\pull-secret.txt"

az account set --subscription $AZURE_SUBSCRIPTION_ID
```

---

## 10. VNet and Subnet Setup

ARO requires two dedicated subnets — one for master nodes and one for workers. The subnets must have `privateLinkServiceNetworkPolicies` and `serviceEndpoints` configured correctly.

```bash
# Create Resource Group
az group create --name $RESOURCEGROUP --location $LOCATION

# Create VNet (address space must accommodate both subnets)
az network vnet create \
  --resource-group $RESOURCEGROUP \
  --name $VNET_NAME \
  --address-prefixes 10.0.0.0/22

# Create master subnet (minimum /27 — recommend /23 for production)
az network vnet subnet create \
  --resource-group $RESOURCEGROUP \
  --vnet-name $VNET_NAME \
  --name $MASTER_SUBNET \
  --address-prefixes 10.0.0.0/23 \
  --service-endpoints Microsoft.ContainerRegistry

# Create worker subnet
az network vnet subnet create \
  --resource-group $RESOURCEGROUP \
  --vnet-name $VNET_NAME \
  --name $WORKER_SUBNET \
  --address-prefixes 10.0.2.0/23 \
  --service-endpoints Microsoft.ContainerRegistry

# Disable private link network policies on master subnet (required by ARO)
az network vnet subnet update \
  --name $MASTER_SUBNET \
  --resource-group $RESOURCEGROUP \
  --vnet-name $VNET_NAME \
  --disable-private-link-service-network-policies true
```

---

## 11. Service Principal (Optional — ARO can create one automatically)

For fine-grained control, create a service principal manually:

```bash
# Create service principal
az ad sp create-for-rbac \
  --name "sp-aro-$CLUSTER" \
  --role Contributor \
  --scopes /subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP \
  --output json

# Capture the output
SP_CLIENT_ID=$(az ad sp list --display-name "sp-aro-$CLUSTER" --query '[0].appId' -o tsv)
SP_CLIENT_SECRET="<secret-from-output>"

# Add the Contributor role on the VNet resource group too
az role assignment create \
  --role "Network Contributor" \
  --assignee $SP_CLIENT_ID \
  --scope /subscriptions/$AZURE_SUBSCRIPTION_ID/resourceGroups/$RESOURCEGROUP
```

> If you omit `--client-id` and `--client-secret` from `az aro create`, ARO will create a service principal automatically.

---

## 12. Pre-Flight Checklist

Before running `az aro create`, confirm:

- [ ] Azure CLI installed and logged in (`az account show`)
- [ ] ARO extension installed (`az extension show --name aro`)
- [ ] `oc` CLI installed and in PATH (`oc version --client`)
- [ ] Pull secret downloaded to `~/.azure/pull-secret.txt`
- [ ] Resource providers registered (Microsoft.RedHatOpenShift: Registered)
- [ ] Sufficient vCPU quota in target region (≥40 DSv3 vCPUs)
- [ ] Resource group created
- [ ] VNet created with correct address space
- [ ] Master and worker subnets created with correct CIDRs
- [ ] Private link service network policies disabled on master subnet
- [ ] Subscription Owner or Contributor + UAA roles confirmed
