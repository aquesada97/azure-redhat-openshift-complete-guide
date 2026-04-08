# Security Context Constraints (SCCs)

## What are SCCs?

Security Context Constraints (SCCs) are OpenShift's pod security enforcement mechanism. They define what a pod is allowed to do: what user IDs it can run as, what capabilities it can have, what volumes it can mount, and more.

SCCs are enforced by an **admission controller** — when a pod is created, OpenShift selects the most restrictive SCC that the pod's service account is authorized to use.

---

## Built-In SCCs

```bash
oc get scc
```

| SCC | runAsUser | Purpose |
|-----|-----------|---------|
| `restricted-v2` | MustRunAsNonRoot | Default for all pods — most secure |
| `restricted` | MustRunAsRange (project UID range) | Legacy default, less strict than v2 |
| `anyuid` | RunAsAny | Run as any UID including root |
| `nonroot` | MustRunAsNonRoot | Explicit non-root with more volume access |
| `nonroot-v2` | MustRunAsNonRoot | Updated nonroot SCC |
| `hostnetwork` | MustRunAsRange | Access host network namespace |
| `hostnetwork-v2` | MustRunAsRange | Updated hostnetwork |
| `hostaccess` | MustRunAsRange | Access host filesystem |
| `hostmount-anyuid` | RunAsAny | Host mounts, any UID |
| `privileged` | RunAsAny | Unrestricted — root, all capabilities |
| `node-exporter` | RunAsAny | For node-exporter DaemonSet |
| `pipelines-scc` | MustRunAsNonRoot | For Tekton pipeline pods |

---

## Key SCC Fields

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: my-custom-scc
# User ID strategy
runAsUser:
  type: MustRunAsRange      # MustRunAsRange, MustRunAsNonRoot, MustRunAs, RunAsAny
  uidRangeMin: 1000
  uidRangeMax: 65535
# SELinux context
seLinuxContext:
  type: MustRunAs           # MustRunAs, RunAsAny
# Filesystem group
fsGroup:
  type: MustRunAs
  ranges:
    - min: 1000
      max: 65535
# Supplemental groups
supplementalGroups:
  type: RunAsAny
# Capabilities
allowedCapabilities: []               # No extra caps
defaultAddCapabilities: []
requiredDropCapabilities:
  - ALL                               # Drop all Linux capabilities
# Privilege escalation
allowPrivilegeEscalation: false
# Host access
allowHostPID: false
allowHostIPC: false
allowHostNetwork: false
allowHostPorts: false
# Volumes allowed
volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
# Sysctls
forbiddenSysctls:
  - '*'
```

---

## Granting an SCC to a ServiceAccount

```bash
# Grant anyuid SCC to a service account
oc adm policy add-scc-to-user anyuid \
  -z my-service-account \
  -n my-namespace

# Grant to a specific user
oc adm policy add-scc-to-user privileged user1

# Grant to all service accounts in a namespace
oc adm policy add-scc-to-group anyuid \
  system:serviceaccounts:my-namespace

# Remove an SCC grant
oc adm policy remove-scc-from-user anyuid \
  -z my-service-account \
  -n my-namespace
```

---

## Creating a Custom SCC

```yaml
apiVersion: security.openshift.io/v1
kind: SecurityContextConstraints
metadata:
  name: custom-nginx
allowPrivilegedContainer: false
allowPrivilegeEscalation: false
runAsUser:
  type: MustRunAsNonRoot
seLinuxContext:
  type: MustRunAs
fsGroup:
  type: RunAsAny
supplementalGroups:
  type: RunAsAny
volumes:
  - configMap
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
requiredDropCapabilities:
  - ALL
allowedCapabilities:
  - NET_BIND_SERVICE   # Allow binding to port 80/443
allowHostNetwork: false
allowHostPID: false
allowHostIPC: false
```

```bash
oc apply -f custom-nginx-scc.yaml
oc adm policy add-scc-to-user custom-nginx -z nginx-sa -n my-namespace
```

---

## Checking SCC on Running Pods

```bash
# Check which SCC was assigned to a pod
oc get pod <pod-name> -o yaml | grep 'openshift.io/scc'

# Check with jsonpath
oc get pod <pod-name> \
  -o jsonpath='{.metadata.annotations.openshift\.io/scc}'

# Find all pods using a specific SCC
oc get pods --all-namespaces \
  -o jsonpath='{range .items[?(@.metadata.annotations.openshift\.io/scc=="privileged")]}{.metadata.namespace}/{.metadata.name}{"\n"}{end}'
```

---

## SCC Inspection Commands

```bash
# Who can use a specific SCC?
oc adm policy who-can use scc privileged

# Check all SCCs and their priorities
oc get scc -o custom-columns=NAME:.metadata.name,PRIORITY:.priority,RUNASUSER:.runAsUser.type

# Describe an SCC
oc describe scc restricted-v2
```

---

## SCC Troubleshooting

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `CrashLoopBackOff` with permission error | Pod runs as wrong UID | Grant `anyuid` or `nonroot` SCC |
| Pod can't write to volume | fsGroup mismatch | Use `anyuid` or set `fsGroup` in pod spec |
| Container can't bind port 80 | Missing `NET_BIND_SERVICE` capability | Custom SCC with `NET_BIND_SERVICE` |
| Pod stuck `Pending` with SCC error | No matching SCC for service account | Grant appropriate SCC |
| SELinux `Permission denied` | SELinux context mismatch | Add `seLinuxOptions` or use `MustRunAs: RunAsAny` |

---

## SCC Best Practices

1. **Use `restricted-v2` by default** — the most secure built-in SCC
2. **Never use `privileged`** unless the workload absolutely requires it (e.g., node-level system tools)
3. **Create minimal custom SCCs** — grant only what the workload needs
4. **Bind SCCs to ServiceAccounts, not users** — least-privilege principle
5. **Audit SCC usage regularly** — `oc adm policy who-can use scc privileged`
6. **Document why each SCC grant was needed** — add to runbook/IaC comments
