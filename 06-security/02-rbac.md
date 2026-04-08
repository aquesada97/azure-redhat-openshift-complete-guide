# RBAC in OpenShift

## Overview

Role-Based Access Control (RBAC) controls who can do what to which resources in Kubernetes/OpenShift. OpenShift extends standard Kubernetes RBAC with additional built-in roles and `oc adm policy` shortcuts.

---

## RBAC Objects

| Object | Scope | Description |
|--------|-------|-------------|
| `Role` | Namespace | Permissions for resources in one namespace |
| `ClusterRole` | Cluster-wide | Permissions for resources cluster-wide or across namespaces |
| `RoleBinding` | Namespace | Binds a Role (or ClusterRole) to users/groups/SA in a namespace |
| `ClusterRoleBinding` | Cluster-wide | Binds a ClusterRole to users/groups/SA cluster-wide |

---

## OpenShift Built-In Cluster Roles

| Role | Description |
|------|-------------|
| `cluster-admin` | Full cluster access — superuser |
| `admin` | Full project/namespace access (can manage RBAC within project) |
| `edit` | Read/write workloads and services, cannot manage RBAC |
| `view` | Read-only access to all resources in namespace |
| `self-provisioner` | Can create new projects (assigned to all authenticated users by default) |
| `system:node` | Node permissions (assigned to node service accounts) |
| `system:image-puller` | Pull images from namespace's image registry |
| `system:image-builder` | Push images to namespace's image registry |
| `system:deployer` | Manage deployments (assigned to deployer SA) |

---

## Role YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: my-project
rules:
  - apiGroups: [""]               # "" = core API group (pods, services, etc.)
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["route.openshift.io"]
    resources: ["routes"]
    verbs: ["get", "list"]
```

### Available Verbs

| Verb | Description |
|------|-------------|
| `get` | Read a specific resource by name |
| `list` | List all resources of a type |
| `watch` | Stream resource changes |
| `create` | Create a new resource |
| `update` | Update an existing resource |
| `patch` | Partially update a resource |
| `delete` | Delete a resource |
| `deletecollection` | Delete multiple resources |
| `exec` | Execute commands in a pod |
| `portforward` | Port-forward to a pod |

---

## RoleBinding YAML

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pod-reader-binding
  namespace: my-project
subjects:
  - kind: User                    # User, Group, or ServiceAccount
    name: alice@example.com
    apiGroup: rbac.authorization.k8s.io
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: my-sa
    namespace: my-project         # Required for ServiceAccount
roleRef:
  kind: Role                      # Role or ClusterRole
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

---

## ClusterRole and ClusterRoleBinding

```yaml
# ClusterRole: accessible cluster-wide
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-viewer
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["metrics.k8s.io"]
    resources: ["nodes", "pods"]
    verbs: ["get", "list"]
---
# ClusterRoleBinding: grant across all namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-viewer-binding
subjects:
  - kind: Group
    name: ops-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-viewer
  apiGroup: rbac.authorization.k8s.io
```

---

## oc adm policy Shortcuts

```bash
# Grant cluster-admin to a user
oc adm policy add-cluster-role-to-user cluster-admin alice

# Grant a cluster role scoped to a namespace (creates RoleBinding, not ClusterRoleBinding)
oc adm policy add-role-to-user admin alice -n my-project
oc adm policy add-role-to-user edit alice -n my-project
oc adm policy add-role-to-user view alice -n my-project

# Grant to a group
oc adm policy add-role-to-group edit dev-team -n my-project

# Grant to a service account
oc adm policy add-role-to-user edit -z my-sa -n my-project

# Remove role
oc adm policy remove-role-from-user edit alice -n my-project
oc adm policy remove-cluster-role-from-user cluster-admin alice

# Disable self-provisioning (prevent users from creating new projects)
oc adm policy remove-cluster-role-from-group \
  self-provisioner \
  system:authenticated:oauth
```

---

## ServiceAccount RBAC

Pods use ServiceAccounts to authenticate to the Kubernetes API:

```bash
# Create a service account
oc create serviceaccount my-app-sa -n my-project

# Grant view permissions to the SA
oc adm policy add-role-to-user view -z my-app-sa -n my-project

# Reference the SA in a Deployment
# spec.template.spec.serviceAccountName: my-app-sa
```

---

## Checking Permissions

```bash
# Can the current user do this?
oc auth can-i get pods
oc auth can-i create deployments -n my-project
oc auth can-i delete secrets -n my-project

# Can a specific user do this?
oc auth can-i get pods --as alice
oc auth can-i create deployments --as alice -n my-project

# Can a service account do this?
oc auth can-i get secrets \
  --as system:serviceaccount:my-project:my-app-sa

# Who can perform a specific action?
oc adm policy who-can get pods -n my-project
oc adm policy who-can create deployments

# View all RoleBindings in a namespace
oc get rolebindings -n my-project -o wide

# View all ClusterRoleBindings
oc get clusterrolebindings -o wide | grep alice
```

---

## RBAC Best Practices

1. **Principle of least privilege** — grant only the minimum verbs and resources needed
2. **Use Groups, not individual users** — manage access by team/role, not per-person
3. **Prefer namespace-scoped Roles over ClusterRoles** when possible
4. **Never grant `cluster-admin` to CI/CD** — use a scoped role per namespace
5. **Review ClusterRoleBindings regularly** — they grant cluster-wide access
6. **Use `oc auth can-i` to verify** — test permissions before deploying
7. **Document all RBAC decisions** — add comments to YAML explaining why
