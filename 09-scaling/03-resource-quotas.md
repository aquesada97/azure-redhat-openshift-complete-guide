# Resource Quotas and Governance

## ResourceQuota

ResourceQuota sets total resource limits for a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: my-project-quota
  namespace: my-project
spec:
  hard:
    # Compute
    requests.cpu: "10"              # Total CPU requests across all pods
    limits.cpu: "20"                # Total CPU limits
    requests.memory: 20Gi
    limits.memory: 40Gi
    # Pod count
    pods: "50"
    # Services
    services: "20"
    services.loadbalancers: "2"
    services.nodeports: "0"         # Disallow NodePort services
    # Storage
    persistentvolumeclaims: "10"
    requests.storage: 500Gi
    managed-premium.storageclass.storage.k8s.io/requests.storage: 200Gi
    # Objects
    configmaps: "20"
    secrets: "20"
    replicationcontrollers: "0"     # Disallow legacy RC
```

```bash
# Apply quota
oc apply -f quota.yaml

# Check quota usage
oc describe quota -n my-project
oc get resourcequota -n my-project -o yaml

# Quota status example:
# Resource              Used  Hard
# --------              ----  ----
# limits.cpu            4     20
# limits.memory         8Gi   40Gi
# pods                  12    50
```

---

## LimitRange

LimitRange sets defaults and bounds for individual containers:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: my-project-limits
  namespace: my-project
spec:
  limits:
    - type: Container
      default:              # Default limits if not specified
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:       # Default requests if not specified
        cpu: "100m"
        memory: "128Mi"
      max:                  # Maximum allowed per container
        cpu: "4"
        memory: "8Gi"
      min:                  # Minimum required per container
        cpu: "50m"
        memory: "64Mi"
    - type: Pod
      max:
        cpu: "8"
        memory: "16Gi"
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi
```

```bash
# Check LimitRange
oc describe limitrange -n my-project
```

---

## Pod Disruption Budgets (PDBs)

PDBs ensure minimum availability during voluntary disruptions (node drain, cluster upgrades):

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: my-app-pdb
  namespace: my-project
spec:
  selector:
    matchLabels:
      app: my-app
  minAvailable: 2         # At least 2 pods must be available
  # OR:
  # maxUnavailable: 1     # At most 1 pod can be unavailable
```

**PDB interaction with upgrades:**
- Cluster upgrades drain nodes one at a time
- If a PDB prevents pod eviction, the upgrade will wait (up to 1 hour per node)
- Ensure your PDB allows at least one pod to be evicted

```bash
# View PDBs
oc get pdb -n my-project
oc describe pdb my-app-pdb -n my-project

# Check disruption status
oc get pdb my-app-pdb -o jsonpath='{.status}'
```

---

## Priority Classes

PriorityClasses allow high-priority workloads to preempt lower-priority ones:

```yaml
# High priority (platform-critical)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority for production workloads"
preemptionPolicy: PreemptLowerPriority
---
# Low priority (batch/background jobs)
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Low priority for batch workloads"
preemptionPolicy: Never    # Won't preempt other pods
```

```yaml
# Use in a pod
spec:
  priorityClassName: high-priority
  containers: ...
```

---

## Namespace-Level Resource Isolation for Multi-Tenancy

Complete isolation setup for a team namespace:

```bash
# Create namespace with quotas and limits
oc new-project team-alpha --display-name="Team Alpha"

# Apply ResourceQuota
oc apply -f team-alpha-quota.yaml -n team-alpha

# Apply LimitRange
oc apply -f team-alpha-limitrange.yaml -n team-alpha

# Apply default NetworkPolicy (deny all ingress)
oc apply -f default-deny-ingress.yaml -n team-alpha

# Grant namespace admin to team
oc adm policy add-role-to-group admin team-alpha -n team-alpha

# Restrict to specific SCCs
oc adm policy add-scc-to-group restricted-v2 system:serviceaccounts:team-alpha
```

---

## ProjectRequest Template

Auto-apply ResourceQuota and LimitRange to every new project:

```yaml
apiVersion: template.openshift.io/v1
kind: Template
metadata:
  name: project-request
  namespace: openshift-config
parameters:
  - name: PROJECT_NAME
  - name: PROJECT_DISPLAYNAME
  - name: PROJECT_DESCRIPTION
  - name: PROJECT_ADMIN_USER
  - name: PROJECT_REQUESTING_USER
objects:
  - apiVersion: project.openshift.io/v1
    kind: Project
    metadata:
      name: ${PROJECT_NAME}
      displayName: ${PROJECT_DISPLAYNAME}
      annotations:
        openshift.io/description: ${PROJECT_DESCRIPTION}
        openshift.io/requester: ${PROJECT_REQUESTING_USER}
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: admin
      namespace: ${PROJECT_NAME}
    subjects:
      - kind: User
        name: ${PROJECT_ADMIN_USER}
    roleRef:
      kind: ClusterRole
      name: admin
  - apiVersion: v1
    kind: ResourceQuota
    metadata:
      name: default-quota
      namespace: ${PROJECT_NAME}
    spec:
      hard:
        pods: "20"
        requests.cpu: "4"
        requests.memory: 8Gi
        limits.cpu: "8"
        limits.memory: 16Gi
  - apiVersion: v1
    kind: LimitRange
    metadata:
      name: default-limits
      namespace: ${PROJECT_NAME}
    spec:
      limits:
        - type: Container
          default:
            cpu: "500m"
            memory: "512Mi"
          defaultRequest:
            cpu: "100m"
            memory: "128Mi"
```

```bash
# Configure the template for all new projects
oc patch project.config.openshift.io/cluster \
  --type merge \
  --patch '{"spec":{"projectRequestTemplate":{"name":"project-request"}}}'
```
