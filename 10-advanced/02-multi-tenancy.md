# Multi-Tenancy in ARO

## Multi-Tenancy Patterns

| Pattern | Isolation Level | Management Overhead | Cost |
|---------|----------------|---------------------|------|
| Namespace-per-team | Medium (shared cluster) | Low | Low (shared) |
| Namespace-per-app | High (strict isolation) | Medium | Low (shared) |
| Cluster-per-tenant | Highest | High | High (separate clusters) |

Most organizations use **namespace-per-team** with strong network policies and RBAC as the default, reserving cluster-per-tenant for compliance-critical workloads.

---

## Namespace Isolation Components

For effective multi-tenancy, apply all of these per namespace:

1. **RBAC** — teams can only access their namespaces
2. **ResourceQuota** — prevent noisy neighbors
3. **LimitRange** — ensure fair resource usage
4. **NetworkPolicy** — prevent cross-namespace traffic
5. **SCC** — restrict pod privileges

---

## Disabling Self-Provisioning

By default, any authenticated user can create new projects. For controlled environments:

```bash
# Remove self-provisioner from all authenticated OAuth users
oc adm policy remove-cluster-role-from-group \
  self-provisioner \
  system:authenticated:oauth

# Verify
oc describe clusterrolebinding self-provisioners

# Now only cluster-admins can create projects
# Users must request projects through an approval process
```

---

## HierarchicalNamespaces Operator (HNC)

HNC provides namespace hierarchy with automatic RBAC and policy inheritance:

```bash
# Install HNC from OperatorHub

# Create a hierarchy: parent → child
oc annotate namespace team-alpha \
  hnc.x-k8s.io/self-provisioner=""

# Create subnamespace
cat <<EOF | oc apply -f -
apiVersion: hnc.x-k8s.io/v1alpha2
kind: SubnamespaceAnchor
metadata:
  name: team-alpha-dev
  namespace: team-alpha
EOF
```

---

## OpenShift Sandbox (Kata Containers)

For highest isolation (untrusted workloads), use Kata Containers which run each pod in a lightweight VM:

```bash
# Install OpenShift Sandboxed Containers Operator from OperatorHub
# Then enable for a namespace:
oc annotate namespace my-isolated-ns \
  io.openshift.workload-profile=high-confidence

# Use RuntimeClass in pod spec:
```

```yaml
spec:
  runtimeClassName: kata-containers
  containers:
    - name: my-app
      image: my-app:latest
```

---

## Cost Management with Metrics Operator

Track resource usage per team/namespace:

```bash
# Install Cost Management Metrics Operator from OperatorHub
# Then view cost data in console.redhat.com/openshift/cost-management
```

---

## Compliance Operator

Automated compliance scanning for regulatory frameworks:

```bash
# Install Compliance Operator from OperatorHub
# Create a ScanSettingBinding:
cat <<EOF | oc apply -f -
apiVersion: compliance.openshift.io/v1alpha1
kind: ScanSettingBinding
metadata:
  name: cis-compliance
  namespace: openshift-compliance
profiles:
  - name: ocp4-cis
    kind: Profile
    apiGroup: compliance.openshift.io/v1alpha1
  - name: ocp4-cis-node
    kind: Profile
    apiGroup: compliance.openshift.io/v1alpha1
settingsRef:
  name: default
  kind: ScanSetting
  apiGroup: compliance.openshift.io/v1alpha1
EOF

# View scan results
oc get compliancecheckresults -n openshift-compliance | grep FAIL | head -20
```

---

## ProjectRequest Template for Multi-Tenancy

Automatically apply isolation configuration to every new project:

```bash
# Create the template (see 09-scaling/03-resource-quotas.md for full template)
oc apply -f project-request-template.yaml -n openshift-config

# Configure cluster to use it
oc patch project.config.openshift.io/cluster \
  --type merge \
  --patch '{"spec":{"projectRequestTemplate":{"name":"project-request"}}}'

# Test by creating a new project
oc new-project test-isolation
oc get quota -n test-isolation   # Should have quota auto-applied
oc get limitrange -n test-isolation   # Should have limits auto-applied
```
