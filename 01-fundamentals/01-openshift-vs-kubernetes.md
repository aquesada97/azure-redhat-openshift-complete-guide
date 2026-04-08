# OpenShift vs Kubernetes

## What is Kubernetes?

Kubernetes (K8s) is an open-source container orchestration platform originally developed by Google. It provides primitives for deploying, scaling, and managing containerized applications. Kubernetes is the foundation upon which OpenShift is built.

## What is OpenShift Container Platform (OCP)?

Red Hat OpenShift Container Platform is an enterprise Kubernetes distribution that adds:
- **Security hardening** (Security Context Constraints, SELinux enforcement)
- **Developer tooling** (S2I builds, ImageStreams, Developer Console)
- **Day-2 operations** (Cluster Operators for self-managing components)
- **Integrated monitoring** (Prometheus, Alertmanager, Grafana pre-configured)
- **Integrated logging** (Cluster Logging Operator)
- **Operator Lifecycle Manager** (OLM) for curated operator catalog
- **OpenShift Router** (HAProxy-based ingress)
- **Built-in image registry**

## What is ARO?

**Azure Red Hat OpenShift (ARO)** is a fully managed deployment of OpenShift Container Platform on Azure, jointly supported by Microsoft and Red Hat. ARO removes the burden of managing the control plane, infrastructure nodes, and cluster upgrades.

---

## Feature Comparison: Vanilla Kubernetes vs OpenShift/ARO

| Feature | Vanilla Kubernetes | OpenShift / ARO |
|---------|--------------------|-----------------|
| Core orchestration | Yes | Yes (K8s-compatible) |
| Pod Security | PodSecurityAdmission (PSA) | Security Context Constraints (SCC) |
| Ingress | Ingress resource (nginx, traefik, etc.) | OpenShift Route (HAProxy built-in) |
| Internal image registry | No (external required) | Yes (built-in) |
| Build system | No | Source-to-Image (S2I), Docker, Pipeline |
| ImageStreams | No | Yes (abstract registry references) |
| Web console | No (3rd party: Lens, k9s) | Full OpenShift Console |
| Operator Lifecycle Manager | No | Yes (OperatorHub) |
| Monitoring | No (install Prometheus separately) | Yes (Prometheus + Grafana pre-configured) |
| Log aggregation | No (install separately) | Cluster Logging Operator |
| OAuth/OIDC | No (configure manually) | Built-in OAuth server (HTPasswd, LDAP, GitHub, AAD) |
| Projects vs Namespaces | Namespaces | Projects (namespaces + RBAC + annotations) |
| Container runtime | containerd (most distros) | CRI-O (Red Hat maintained) |
| Node OS | Any Linux | RHCOS (Red Hat CoreOS, immutable) |
| Multi-tenancy | Manual configuration | Projects, SCCs, quotas, network isolation |
| CLI | kubectl | oc (superset of kubectl) |

---

## Security Context Constraints (SCCs) vs PodSecurityAdmission (PSA)

### Kubernetes PodSecurityAdmission (PSA)

Kubernetes 1.25+ replaced PodSecurityPolicy with PodSecurityAdmission. PSA enforces three policy levels at the namespace level:
- **privileged**: No restrictions
- **baseline**: Prevents known privilege escalations
- **restricted**: Heavily restricted, follows best practices

```yaml
# PSA namespace label
apiVersion: v1
kind: Namespace
metadata:
  name: my-app
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/warn: restricted
```

### OpenShift Security Context Constraints (SCCs)

SCCs are OpenShift's more granular security model. Unlike PSA (namespace-scoped), SCCs are assigned to **ServiceAccounts** via RBAC and apply per-pod based on which service account the pod uses.

```yaml
# SCC assignment via RBAC
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: allow-anyuid
  namespace: my-app
subjects:
  - kind: ServiceAccount
    name: my-sa
    namespace: my-app
roleRef:
  kind: ClusterRole
  name: system:openshift:scc:anyuid
  apiGroup: rbac.authorization.k8s.io
```

Key SCC fields:
- `runAsUser`: MustRunAs, MustRunAsNonRoot, RunAsAny
- `seLinuxContext`: MustRunAs, RunAsAny
- `fsGroup`: MustRunAs, RunAsAny
- `allowPrivilegeEscalation`: true/false
- `allowHostPID`, `allowHostIPC`, `allowHostNetwork`: true/false
- `volumes`: allowed volume types

### SCCs vs PSA Comparison

| Aspect | PSA | SCC |
|--------|-----|-----|
| Scope | Namespace-level | ServiceAccount-level |
| Granularity | 3 policy levels | Highly customizable |
| Assignment | Namespace label | RBAC to ServiceAccount |
| Built-in policies | 3 | 8+ |
| Custom policies | No | Yes |

---

## OpenShift Routes vs Kubernetes Ingress

### Kubernetes Ingress

The Kubernetes Ingress resource requires a separate ingress controller (nginx, traefik, HAProxy, etc.) to be installed:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-svc
                port:
                  number: 80
```

### OpenShift Route

Routes are OpenShift's first-class ingress object, backed by HAProxy. No ingress controller installation needed:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-route
spec:
  host: myapp.apps.cluster.example.com   # Auto-generated if omitted
  to:
    kind: Service
    name: my-svc
  port:
    targetPort: 8080
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

### Routes vs Ingress Comparison

| Feature | Ingress | Route |
|---------|---------|-------|
| Controller required | Yes (install separately) | No (built-in HAProxy) |
| TLS termination | Via annotations | Native (edge/passthrough/reencrypt) |
| Wildcard hostname | Depends on controller | Yes (*.apps.cluster.domain) |
| A/B routing (weights) | Limited | Yes (native) |
| Sticky sessions | Via annotations | Yes (native) |
| Path-based routing | Yes | Limited |
| API version | networking.k8s.io/v1 | route.openshift.io/v1 |

---

## ARO vs AKS vs Self-Managed OCP

| Dimension | AKS | ARO | Self-Managed OCP on Azure |
|-----------|-----|-----|--------------------------|
| Control plane management | Microsoft | Microsoft + Red Hat | Customer |
| Worker node management | Microsoft | Customer (Machine API) | Customer |
| OS | Ubuntu / Azure Linux | RHCOS (immutable) | RHCOS |
| Kubernetes version | Upstream K8s | OCP (K8s + OpenShift) | OCP |
| Upgrade | Customer-initiated | Customer-initiated | Customer-managed |
| Support | Microsoft | Microsoft + Red Hat | Red Hat (subscription) |
| SLA | 99.95% (uptime) | 99.95% (uptime) | Customer-defined |
| Pricing | Worker VM cost only | Worker VM + surcharge | VM + OCP subscription |
| OpenShift features | No | Full | Full |
| Azure AD integration | Native (AAD pod identity / Workload Identity) | via OAuth OIDC | Manual |
| Time to first cluster | ~5 min | ~35 min | ~45 min |

---

## OpenShift Projects vs Kubernetes Namespaces

An OpenShift **Project** is a Kubernetes namespace with additional annotations and automatic RBAC:

```bash
# Create a project (OpenShift-specific)
oc new-project my-project \
  --display-name="My Application" \
  --description="Production environment for my-app"

# Behind the scenes, this creates:
# - A namespace: my-project
# - RoleBindings for the creator (admin role on the project)
# - Annotations: openshift.io/display-name, openshift.io/description
```

Projects also integrate with:
- **ProjectRequest template**: Auto-apply ResourceQuotas and LimitRanges on every new project
- **Self-provisioner role**: Controls who can create projects
- **Network isolation**: Default OVN-K network policy per project

```bash
# List all projects (shows all namespaces you have access to)
oc projects

# Switch to a project
oc project my-project

# Check project status (shows deployments, services, routes)
oc status
```
