# Workload Identity for ARO

## What is Workload Identity?

Azure Workload Identity allows pods to authenticate to Azure services (Key Vault, Storage, SQL, etc.) **without client secrets or certificates**. It uses Kubernetes ServiceAccount token projection combined with Azure AD OIDC federation.

**How it works:**
1. Pod requests a Kubernetes ServiceAccount token (projected volume, expires in ~1 hour)
2. Azure AD validates the token against the cluster's OIDC issuer (federated credential)
3. Azure AD returns an Azure access token
4. Pod uses the access token to call Azure APIs

---

## Setup Steps

### Step 1: Verify OIDC Issuer on ARO

ARO clusters have a built-in OIDC issuer:

```bash
# Get the OIDC issuer URL
OIDC_ISSUER=$(az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --query "clusterProfile.oidcIssuer" -o tsv)

echo "OIDC Issuer: $OIDC_ISSUER"
```

### Step 2: Create Azure AD Application and Managed Identity

```bash
# Create a managed identity (recommended over app registration for workloads)
az identity create \
  --name "id-my-app" \
  --resource-group $RESOURCEGROUP

CLIENT_ID=$(az identity show \
  --name "id-my-app" \
  --resource-group $RESOURCEGROUP \
  --query clientId -o tsv)

PRINCIPAL_ID=$(az identity show \
  --name "id-my-app" \
  --resource-group $RESOURCEGROUP \
  --query principalId -o tsv)
```

### Step 3: Create Federated Credential

```bash
az identity federated-credential create \
  --name "fc-my-app" \
  --identity-name "id-my-app" \
  --resource-group $RESOURCEGROUP \
  --issuer "$OIDC_ISSUER" \
  --subject "system:serviceaccount:my-project:my-app-sa" \
  --audiences "api://AzureADTokenExchange"
```

### Step 4: Grant Azure RBAC to the Managed Identity

```bash
# Example: grant Key Vault Secrets User role
KEYVAULT_ID=$(az keyvault show \
  --name my-keyvault \
  --resource-group $RESOURCEGROUP \
  --query id -o tsv)

az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee-object-id $PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --scope $KEYVAULT_ID
```

### Step 5: Create Kubernetes ServiceAccount with Annotation

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-app-sa
  namespace: my-project
  annotations:
    azure.workload.identity/client-id: "<CLIENT_ID>"
    azure.workload.identity/tenant-id: "<TENANT_ID>"
```

### Step 6: Label the Pod

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-project
spec:
  template:
    metadata:
      labels:
        app: my-app
        azure.workload.identity/use: "true"   # Enable workload identity
    spec:
      serviceAccountName: my-app-sa
      containers:
        - name: my-app
          image: my-app:latest
          # Azure SDK auto-picks up these env vars (injected by the mutating webhook):
          # AZURE_CLIENT_ID
          # AZURE_TENANT_ID
          # AZURE_FEDERATED_TOKEN_FILE
          # AZURE_AUTHORITY_HOST
```

---

## Using the Identity in Code

The Azure SDK for most languages automatically handles token acquisition when these env vars are present:

```python
# Python example
from azure.identity import WorkloadIdentityCredential
from azure.keyvault.secrets import SecretClient

credential = WorkloadIdentityCredential()
client = SecretClient(
    vault_url="https://my-keyvault.vault.azure.net/",
    credential=credential
)
secret = client.get_secret("my-secret")
print(secret.value)
```

```javascript
// Node.js example
const { WorkloadIdentityCredential } = require("@azure/identity");
const { SecretClient } = require("@azure/keyvault-secrets");

const credential = new WorkloadIdentityCredential();
const client = new SecretClient("https://my-keyvault.vault.azure.net/", credential);
const secret = await client.getSecret("my-secret");
```

---

## Troubleshooting

```bash
# Check that token is projected into pod
oc exec my-app-pod -- ls /var/run/secrets/azure/tokens/
# Should show: azure-identity-token

# Check env vars are injected
oc exec my-app-pod -- env | grep AZURE

# Verify federated credential
az identity federated-credential list \
  --identity-name "id-my-app" \
  --resource-group $RESOURCEGROUP

# Check if OIDC discovery endpoint is accessible from pod
oc exec my-app-pod -- curl "$OIDC_ISSUER/.well-known/openid-configuration"
```
