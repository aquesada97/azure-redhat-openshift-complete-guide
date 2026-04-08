# OpenShift Web Console

## Accessing the Console

```bash
# Get the console URL
az aro show \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --query consoleProfile.url -o tsv
# Opens: https://console-openshift-console.apps.<cluster-domain>
```

---

## Administrator Perspective

Switch to **Administrator** view using the dropdown at the top left.

### Home Section
- **Overview**: Cluster health, resource usage, alerts, events at a glance
- **Projects**: List and manage all namespaces/projects
- **Search**: Universal search across all resource types
- **Explore**: API browser — discover all available resource types and CRDs

### Workloads Section
- **Pods**: View all pods, stream logs, exec into containers
- **Deployments**: Manage Deployments, trigger rollouts
- **StatefulSets**: Manage StatefulSets, scale replicas
- **DaemonSets**: View DaemonSets and per-node pod status
- **Jobs / CronJobs**: Manage batch workloads, trigger manual CronJob runs
- **ReplicaSets**: View auto-created ReplicaSets from Deployments
- **Horizontal Pod Autoscalers**: View and manage HPA objects

### Networking Section
- **Services**: View all Services and their endpoints
- **Routes**: Manage Routes, view URLs, certificate status
- **Ingresses**: Manage Kubernetes Ingress objects
- **Network Policies**: View and create NetworkPolicy objects

### Storage Section
- **PersistentVolumeClaims**: Request, resize, and delete PVCs
- **PersistentVolumes**: View cluster-level PV objects
- **StorageClasses**: View available StorageClasses

### Builds Section
- **BuildConfigs**: Manage S2I and Docker builds
- **Builds**: View build history and logs
- **ImageStreams**: View internal image registry contents

### User Management Section
- **Users**: List authenticated users
- **Groups**: Manage groups and group membership
- **ServiceAccounts**: Create and manage ServiceAccounts
- **Roles / RoleBindings**: View and create RBAC policies
- **ClusterRoles / ClusterRoleBindings**: Cluster-wide RBAC

### Administration Section
- **Cluster Settings**: View cluster version, update channel, manage upgrades
- **Namespaces**: Create and configure namespaces with labels
- **ResourceQuotas**: View per-namespace quotas and usage
- **LimitRanges**: Configure default resource limits
- **Custom Resource Definitions**: View all CRDs installed in the cluster
- **Operator Hub**: Browse and install operators
- **Installed Operators**: Manage operator lifecycle

---

## Developer Perspective

Switch to **Developer** view using the dropdown at the top left. Designed for application developers who don't need full cluster access.

### Topology View
- Visual graph showing pods, services, routes, and their connections
- Color-coded status (green = healthy, yellow = degraded, red = error)
- Click any resource to see details, logs, and metrics in a side panel
- Drag to create connections between services

### +Add (Add Resources)
- **From Git**: Deploy directly from a Git repo using S2I
- **Container Image**: Deploy any container image
- **From Catalog**: Browse service catalog templates and operators
- **From Dockerfile**: Build and deploy from a Dockerfile
- **From YAML**: Paste raw YAML to create resources
- **Pipeline**: Create Tekton pipelines

### Builds Section
- View BuildConfigs and active builds
- Stream build logs in real time
- Re-trigger builds manually

### Observe Section (Per-Project)
- **Metrics**: PromQL query interface with charts
- **Alerts**: Active alerts relevant to your project
- **Events**: Recent events for your project

---

## In-Browser Terminal (Web Terminal Operator)

The Web Terminal Operator provides a `bash` terminal in the browser with `oc` and `kubectl` pre-installed:

```bash
# Install from OperatorHub: "Web Terminal Operator" by Red Hat
# After installation, a terminal icon (>_) appears in the top right of the console
```

---

## Useful Console Shortcuts

| Task | Navigation |
|------|-----------|
| Stream pod logs | Workloads → Pods → click pod → Logs tab |
| Copy login command | Your username (top right) → Copy Login Command |
| Execute in pod | Workloads → Pods → click pod → Terminal tab |
| View metrics | Observe → Metrics → enter PromQL |
| View cluster events | Home → Events |
| Upgrade cluster | Administration → Cluster Settings → Update |
| Install operator | Operators → OperatorHub → search → Install |
