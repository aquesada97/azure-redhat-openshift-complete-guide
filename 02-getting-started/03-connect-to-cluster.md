# Connecting to Your ARO Cluster

## Getting kubeadmin Credentials

The `kubeadmin` user is the initial cluster administrator. Use it to set up identity providers, then disable it.

```bash
# Get kubeadmin password
az aro list-credentials \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER

# Output example:
# {
#   "kubeadminPassword": "XXXXX-XXXXX-XXXXX-XXXXX",
#   "kubeadminUsername": "kubeadmin"
# }

# Get API server URL
APISERVER=$(az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --query apiserverProfile.url -o tsv)

# Log in
oc login $APISERVER \
  -u kubeadmin \
  -p $(az aro list-credentials \
        --resource-group $RESOURCEGROUP \
        --name $CLUSTER \
        --query kubeadminPassword -o tsv)
```

---

## Accessing the Web Console

```bash
# Get console URL
CONSOLE=$(az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --query consoleProfile.url -o tsv)

echo "Console URL: $CONSOLE"
# Open in browser: https://console-openshift-console.apps.<cluster>.<domain>
```

### Web Console Perspectives

**Administrator Perspective** (for cluster admins and DevOps):
- Home → Overview, Projects, Search, Explore (API browser)
- Workloads → Pods, Deployments, StatefulSets, DaemonSets, Jobs, CronJobs
- Networking → Services, Routes, Ingresses, NetworkPolicies
- Storage → PersistentVolumeClaims, StorageClasses, VolumeSnapshots
- Builds → BuildConfigs, Builds, ImageStreams
- User Management → Users, Groups, ServiceAccounts, Roles, RoleBindings
- Administration → Cluster Settings, Namespaces, ResourceQuotas, LimitRanges
- Operators → OperatorHub, Installed Operators

**Developer Perspective** (for application developers):
- Topology → Visual graph of your app's pods, routes, services
- Add → Deploy image, from git, from catalog, from YAML
- Builds → Monitor builds in progress
- Observe → Metrics, logs, events for your project

---

## Setting Up Azure AD as Identity Provider

Replace `kubeadmin` with your organization's Azure AD accounts.

### Step 1: Create Azure AD App Registration

```bash
# Register an app in Azure AD
APP_NAME="openshift-auth-$CLUSTER"

APP_ID=$(az ad app create \
  --display-name $APP_NAME \
  --web-redirect-uris "$(az aro show --resource-group $RESOURCEGROUP --name $CLUSTER --query consoleProfile.url -o tsv)/auth/callback" \
  --query appId -o tsv)

# Create client secret
CLIENT_SECRET=$(az ad app credential reset \
  --id $APP_ID \
  --append \
  --query password -o tsv)

TENANT_ID=$(az account show --query tenantId -o tsv)

echo "APP_ID: $APP_ID"
echo "CLIENT_SECRET: $CLIENT_SECRET"
echo "TENANT_ID: $TENANT_ID"
```

### Step 2: Create the Secret in OpenShift

```bash
oc create secret generic aad-secret \
  --from-literal=clientSecret=$CLIENT_SECRET \
  -n openshift-config
```

### Step 3: Configure OAuth

```yaml
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
    - name: AAD
      mappingMethod: claim
      type: OpenID
      openID:
        clientID: <APP_ID>
        clientSecret:
          name: aad-secret
        claims:
          preferredUsername:
            - email
          name:
            - name
          email:
            - email
          groups:
            - groups
        issuer: https://login.microsoftonline.com/<TENANT_ID>/v2.0
        extraScopes:
          - email
          - profile
```

```bash
# Apply (replace placeholders first)
oc apply -f oauth-aad.yaml

# Watch authentication operator reconcile
oc get clusteroperator authentication -w
```

---

## Kubeconfig Management

### Get Admin Kubeconfig

```bash
# Download kubeconfig for kubeadmin
az aro get-admin-kubeconfig \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --file ./kubeconfig-$CLUSTER

# Use it
export KUBECONFIG=./kubeconfig-$CLUSTER
oc get nodes
```

### Multiple Contexts

```bash
# View all contexts
oc config get-contexts

# Switch context
oc config use-context <context-name>

# View current context
oc config current-context

# Merge multiple kubeconfigs
export KUBECONFIG=~/.kube/config:./kubeconfig-cluster1:./kubeconfig-cluster2
kubectl config view --flatten > ~/.kube/config-merged
```

### Copy Login Command from Console

1. Log into the web console
2. Click your username (top right) → **Copy Login Command**
3. Click **Display Token**
4. Copy the `oc login` command and run it in your terminal

---

## Disabling kubeadmin

After setting up an identity provider and confirming you have another cluster-admin:

```bash
# First: verify you have cluster-admin via the new IdP
oc login <api-url> -u <your-aad-user>
oc get nodes   # Should work

# Grant cluster-admin to your AAD user
oc adm policy add-cluster-role-to-user cluster-admin <your-aad-username>

# Then disable kubeadmin
oc delete secret kubeadmin -n kube-system
```

> **Warning:** This is irreversible. Ensure at least one other cluster-admin account exists before deleting kubeadmin.

---

## Service Accounts for CI/CD

```bash
# Create a service account for CI/CD pipelines
oc create serviceaccount cicd-sa -n my-app

# Grant cluster-admin (for admin pipelines — use minimal permissions for apps)
oc adm policy add-cluster-role-to-user cluster-admin \
  -z cicd-sa \
  -n my-app

# For application deployments, use namespace-scoped edit role instead
oc adm policy add-role-to-user edit \
  -z cicd-sa \
  -n my-app

# Get service account token (for use in CI/CD systems)
oc serviceaccounts get-token cicd-sa -n my-app

# Create a long-lived token secret
oc create secret generic cicd-token \
  --from-literal=token=$(oc serviceaccounts get-token cicd-sa -n my-app) \
  -n my-app
```
