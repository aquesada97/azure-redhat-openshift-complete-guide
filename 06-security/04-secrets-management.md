# Secrets Management in ARO

## Kubernetes Secrets (Base64, not encrypted)

By default, Kubernetes Secrets are base64-encoded and stored in etcd **without encryption**. They are accessible to anyone with API access to the Secret resource.

```bash
# Base64 is NOT encryption — anyone can decode it
echo "bXktcGFzc3dvcmQ=" | base64 -d
# Output: my-password
```

### Enabling etcd Encryption at Rest

ARO supports etcd encryption using Azure-managed encryption keys:

```bash
# Check if etcd encryption is enabled
oc get apiserver cluster -o jsonpath='{.spec.encryption}'

# Enable etcd encryption (requires cluster-admin)
oc patch apiserver cluster \
  --type merge \
  --patch '{"spec":{"encryption":{"type":"aescbc"}}}'

# Watch encryption progress
oc get apiserver cluster -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

---

## Azure Key Vault CSI Driver

The **Secrets Store CSI Driver** with the Azure Key Vault provider mounts secrets from Azure Key Vault directly into pods as files or Kubernetes Secrets.

### Installing the CSI Driver

```bash
# Install via Helm
helm repo add csi-secrets-store-provider-azure \
  https://azure.github.io/secrets-store-csi-driver-provider-azure/charts

helm install csi-secrets-store-provider-azure \
  csi-secrets-store-provider-azure/csi-secrets-store-provider-azure \
  --namespace kube-system \
  --set secrets-store-csi-driver.syncSecret.enabled=true
```

### SecretProviderClass

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-kv-secrets
  namespace: my-project
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "false"
    clientID: "<workload-identity-client-id>"
    keyvaultName: "my-keyvault"
    cloudName: ""
    objects: |
      array:
        - objectName: db-password
          objectType: secret
          objectVersion: ""
        - objectName: api-key
          objectType: secret
          objectAlias: API_KEY
        - objectName: tls-cert
          objectType: cert
  # Optional: sync secrets to K8s Secrets
  secretObjects:
    - secretName: app-secrets
      type: Opaque
      data:
        - objectName: db-password
          key: password
        - objectName: api-key
          key: apiKey
  tenantID: "<tenant-id>"
```

### Mount Key Vault Secrets in a Pod

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
        azure.workload.identity/use: "true"
    spec:
      serviceAccountName: my-app-sa   # Must have workload identity annotation
      containers:
        - name: my-app
          image: my-app:latest
          volumeMounts:
            - name: secrets-store
              mountPath: /mnt/secrets
              readOnly: true
          env:
            # Reference synced K8s Secret
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: password
      volumes:
        - name: secrets-store
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: azure-kv-secrets
```

---

## External Secrets Operator

An alternative to CSI driver — creates Kubernetes Secrets automatically from Key Vault:

```bash
# Install External Secrets Operator from OperatorHub
# Then configure a SecretStore:
```

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: azure-keyvault
  namespace: my-project
spec:
  provider:
    azurekv:
      authType: WorkloadIdentity
      vaultUrl: "https://my-keyvault.vault.azure.net"
      serviceAccountRef:
        name: my-app-sa
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
  namespace: my-project
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: azure-keyvault
    kind: SecretStore
  target:
    name: db-credentials   # Created K8s Secret name
    creationPolicy: Owner
  data:
    - secretKey: password
      remoteRef:
        key: db-password   # Key Vault secret name
```

---

## ConfigMap vs Secret

| Aspect | ConfigMap | Secret |
|--------|-----------|--------|
| Data type | Plain text | Base64-encoded |
| Stored in etcd | Yes (plaintext) | Yes (base64, optionally encrypted) |
| Use case | App config, env vars | Passwords, API keys, certs |
| RBAC | Standard RBAC | Standard RBAC (tighter controls recommended) |
| Size limit | 1 MiB | 1 MiB |
| Auto-reload in pod | No (restart needed) | No (restart needed) |

---

## Bitnami Sealed Secrets

Seal secrets for safe GitOps storage:

```bash
# Install kubeseal CLI
# Install Sealed Secrets controller from OperatorHub or Helm

# Seal a secret
oc create secret generic my-secret \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml | \
  kubeseal --format=yaml > sealed-secret.yaml

# Commit sealed-secret.yaml to Git safely
# The controller decrypts and creates the real Secret in-cluster
```
