# Templates, Helm, and GitOps

## OpenShift Templates

A Template is an OpenShift-specific resource that bundles multiple Kubernetes resources together with parameterization.

### Template Structure

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: my-app-template
  annotations:
    description: "Deploy my application with configurable replicas and image tag"
    tags: "nodejs,database"
parameters:
  - name: APP_NAME
    description: Application name
    required: true
  - name: IMAGE_TAG
    description: Container image tag
    value: latest
  - name: REPLICAS
    description: Number of replicas
    value: "2"
  - name: DATABASE_PASSWORD
    description: PostgreSQL password
    generate: expression    # Auto-generate if not provided
    from: '[a-zA-Z0-9]{16}'
objects:
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ${APP_NAME}
    spec:
      replicas: ${{REPLICAS}}   # ${{}} for integers
      template:
        spec:
          containers:
            - name: ${APP_NAME}
              image: my-registry/${APP_NAME}:${IMAGE_TAG}
  - apiVersion: v1
    kind: Service
    metadata:
      name: ${APP_NAME}
    spec:
      selector:
        app: ${APP_NAME}
      ports:
        - port: 80
```

```bash
# Process and apply a template
oc process -f my-template.yaml \
  -p APP_NAME=my-app \
  -p IMAGE_TAG=v1.2.0 \
  | oc apply -f -

# Use a template from the openshift namespace
oc new-app --template=openshift/postgresql-persistent \
  -p POSTGRESQL_USER=myuser \
  -p POSTGRESQL_PASSWORD=mypass \
  -p POSTGRESQL_DATABASE=mydb

# List available templates
oc get templates -n openshift
```

---

## Helm in OpenShift

Helm is fully supported in ARO. The `helm` CLI works exactly as it does with vanilla Kubernetes.

```bash
# Install Helm (if not already installed)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Add a Helm repo
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install a chart
helm install my-postgresql bitnami/postgresql \
  --namespace my-project \
  --set auth.postgresPassword=mysecret \
  --set primary.persistence.storageClass=managed-premium

# List installed releases
helm list -n my-project

# Upgrade a release
helm upgrade my-postgresql bitnami/postgresql \
  --namespace my-project \
  --set primary.resources.requests.memory=2Gi

# Uninstall
helm uninstall my-postgresql -n my-project
```

### HelmChartRepository (OpenShift Console)

```yaml
apiVersion: helm.openshift.io/v1beta1
kind: HelmChartRepository
metadata:
  name: bitnami
spec:
  connectionConfig:
    url: https://charts.bitnami.com/bitnami
  description: Bitnami Helm Charts
```

---

## Helm vs Templates vs Kustomize

| Feature | Helm | Templates | Kustomize |
|---------|------|-----------|-----------|
| Language | Go templates | OpenShift templates | YAML overlays |
| Package management | Yes (charts, repos) | No | No |
| Parameterization | Values files, `--set` | `-p` parameters | Patches, vars |
| Rollback support | Yes (release history) | No | No |
| OpenShift-specific | No | Yes | No |
| GitOps friendly | Yes (Helm CRD in ArgoCD) | Limited | Yes |
| Complexity | Medium | Low | Low-Medium |

---

## GitOps with OpenShift GitOps (ArgoCD)

### Install OpenShift GitOps Operator

```bash
# Install via Subscription
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-gitops-operator
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-gitops-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Wait for ArgoCD to be ready
oc wait --for=condition=Available deployment/openshift-gitops-server \
  -n openshift-gitops --timeout=120s

# Get ArgoCD console URL
oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}'
```

### ArgoCD Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: openshift-gitops
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/my-app.git
    targetRevision: main
    path: k8s/overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: my-app
  syncPolicy:
    automated:
      prune: true        # Delete resources removed from Git
      selfHeal: true     # Revert manual changes
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### AppProject for Multi-Tenancy

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: team-frontend
  namespace: openshift-gitops
spec:
  description: Frontend team project
  sourceRepos:
    - https://github.com/myorg/frontend-apps.git
  destinations:
    - namespace: frontend-*     # Wildcard namespace matching
      server: https://kubernetes.default.svc
  clusterResourceWhitelist: []  # No cluster-level resources
  namespaceResourceWhitelist:
    - group: apps
      kind: Deployment
    - group: ""
      kind: Service
    - group: route.openshift.io
      kind: Route
```
