# GitOps and CI/CD for ARO

## GitOps Principles

1. **Declarative**: All configuration is declared in Git as YAML
2. **Versioned**: Every change is a Git commit with history
3. **Automated**: A controller continuously syncs the cluster to Git state
4. **Observable**: Any drift from Git is detected and can be auto-corrected

---

## OpenShift GitOps (ArgoCD)

### Install OpenShift GitOps Operator

```bash
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
  installPlanApproval: Automatic
EOF

# Wait for ArgoCD to deploy
oc wait deployment/openshift-gitops-server \
  -n openshift-gitops \
  --for=condition=Available \
  --timeout=120s

# Get ArgoCD URL
echo "ArgoCD URL: https://$(oc get route openshift-gitops-server -n openshift-gitops -o jsonpath='{.spec.host}')"

# Get initial admin password
oc extract secret/openshift-gitops-cluster \
  -n openshift-gitops \
  --to=-
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
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### ApplicationSet (Multiple Environments)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: my-app-environments
  namespace: openshift-gitops
spec:
  generators:
    - list:
        elements:
          - env: dev
            namespace: my-app-dev
            branch: develop
            server: https://kubernetes.default.svc
          - env: staging
            namespace: my-app-staging
            branch: staging
            server: https://kubernetes.default.svc
          - env: prod
            namespace: my-app-prod
            branch: main
            server: https://kubernetes.default.svc
  template:
    metadata:
      name: "my-app-{{env}}"
      namespace: openshift-gitops
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/my-app.git
        targetRevision: "{{branch}}"
        path: k8s/overlays/{{env}}
      destination:
        server: "{{server}}"
        namespace: "{{namespace}}"
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## OpenShift Pipelines (Tekton)

### Install OpenShift Pipelines Operator

```bash
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator-rh
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### Simple Build and Deploy Pipeline

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-deploy
  namespace: my-project
spec:
  params:
    - name: git-url
      type: string
    - name: git-revision
      type: string
      default: main
    - name: image
      type: string
  workspaces:
    - name: source
    - name: docker-credentials
  tasks:
    - name: clone
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: Task
          - name: name
            value: git-clone
          - name: namespace
            value: openshift-pipelines
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)
      workspaces:
        - name: output
          workspace: source

    - name: build-and-push
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: Task
          - name: name
            value: buildah
          - name: namespace
            value: openshift-pipelines
      runAfter:
        - clone
      params:
        - name: IMAGE
          value: $(params.image)
      workspaces:
        - name: source
          workspace: source
        - name: dockerconfig
          workspace: docker-credentials

    - name: deploy
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: Task
          - name: name
            value: openshift-client
          - name: namespace
            value: openshift-pipelines
      runAfter:
        - build-and-push
      params:
        - name: SCRIPT
          value: |
            oc set image deployment/my-app \
              my-app=$(params.image) \
              -n my-project
            oc rollout status deployment/my-app -n my-project
```

---

## Integrating Azure Container Registry (ACR)

```bash
# Create ACR pull secret
oc create secret docker-registry acr-pull-secret \
  --docker-server=myacr.azurecr.io \
  --docker-username=$(az acr credential show -n myacr --query username -o tsv) \
  --docker-password=$(az acr credential show -n myacr --query "passwords[0].value" -o tsv) \
  -n my-project

# Link to default service account (for automatic pulls)
oc secrets link default acr-pull-secret --for=pull -n my-project

# Or add to global cluster pull secret (for all namespaces)
oc set data secret/pull-secret \
  --from-file=.dockerconfigjson=merged-config.json \
  -n openshift-config
```

---

## GitHub Actions to ARO Pipeline

```yaml
# .github/workflows/deploy-to-aro.yaml
name: Deploy to ARO
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Azure
        uses: azure/login@v2
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get ARO kubeconfig
        run: |
          az aro get-admin-kubeconfig \
            --resource-group ${{ vars.RESOURCEGROUP }} \
            --name ${{ vars.CLUSTER_NAME }} \
            --file ./kubeconfig
          echo "KUBECONFIG=$PWD/kubeconfig" >> $GITHUB_ENV

      - name: Deploy to ARO
        run: |
          oc apply -f k8s/deployment.yaml
          oc set image deployment/my-app \
            my-app=${{ vars.ACR_LOGIN_SERVER }}/my-app:${{ github.sha }}
          oc rollout status deployment/my-app --timeout=5m
```
