# ARO Components

## Overview

ARO is a self-managing platform. Its internal components are managed by **Cluster Operators (COs)** — OpenShift controllers that continuously reconcile component state. This document covers each major component, its purpose, and how to inspect it.

---

## 1. etcd

**What it is:** The distributed key-value store that serves as Kubernetes' backing store for all cluster state. All resources (pods, services, deployments, secrets, etc.) are persisted in etcd.

**Why it matters:** etcd uses Raft consensus — 3 members provide quorum. If 2+ members fail, the cluster loses write capability and most operations halt.

**In ARO:** etcd is managed by the **etcd Cluster Operator**. Microsoft and Red Hat manage etcd health, backups, and defragmentation.

```bash
# Check etcd pods
oc get pods -n openshift-etcd

# Check etcd operator
oc get clusteroperator etcd

# Check etcd member health (requires etcd pod exec)
oc rsh -n openshift-etcd etcd-<master-node-name> \
  etcdctl endpoint health \
  --cacert=/etc/kubernetes/static-pod-resources/configmaps/etcd-serving-ca/ca-bundle.crt \
  --cert=/etc/kubernetes/static-pod-resources/secrets/etcd-all-certs/etcd-serving-<node>.crt \
  --key=/etc/kubernetes/static-pod-resources/secrets/etcd-all-certs/etcd-serving-<node>.key
```

---

## 2. kube-apiserver

**What it is:** The Kubernetes API server — the front door to the cluster. All `kubectl`/`oc` commands, controllers, and operators communicate through the API server.

**Key features:**
- REST API for all Kubernetes resources
- Admission controllers (validation, mutation webhooks)
- Authentication and authorization
- Audit logging

```bash
# Check kube-apiserver pods on master nodes
oc get pods -n openshift-kube-apiserver

# Check kube-apiserver operator
oc get clusteroperator kube-apiserver

# View audit logs (stored on master nodes)
oc adm node-logs --role=master --path=kube-apiserver/audit.log | tail -20
```

---

## 3. OpenShift API Server

**What it is:** The OpenShift-specific API server that extends Kubernetes with OpenShift resources: Routes, DeploymentConfigs, ImageStreams, BuildConfigs, Projects, etc.

```bash
# Check openshift-apiserver pods
oc get pods -n openshift-apiserver

# Check operator
oc get clusteroperator openshift-apiserver

# View OpenShift-specific API groups
oc api-resources --api-group=route.openshift.io
oc api-resources --api-group=build.openshift.io
oc api-resources --api-group=image.openshift.io
```

---

## 4. OAuth Server

**What it is:** The built-in OAuth 2.0 provider for OpenShift. Handles user authentication and issues tokens. Supports multiple identity providers simultaneously.

**Supported Identity Providers:**
| Provider | Type | Use Case |
|----------|------|---------|
| HTPasswd | Local file | Dev/test, small teams |
| LDAP | Directory service | Active Directory, FreeIPA |
| GitHub | OAuth | Open source teams |
| GitLab | OAuth | Enterprise GitLab |
| OpenID Connect | OIDC | Azure AD, Okta, Keycloak |
| Google | OAuth | G Suite organizations |

```bash
# Check OAuth pods
oc get pods -n openshift-authentication

# Check OAuth operator
oc get clusteroperator authentication

# View current OAuth configuration
oc get oauth cluster -o yaml

# View identity providers configured
oc get oauth cluster -o jsonpath='{.spec.identityProviders[*].name}'
```

---

## 5. Integrated Image Registry

**What it is:** A built-in container image registry hosted at `image-registry.openshift-image-registry.svc:5000`. Stores images built by S2I, Docker, or pipeline builds. Referenced via ImageStreams.

**Storage:** Backed by a PVC. Default uses Azure Blob Storage in ARO.

```bash
# Check registry pods
oc get pods -n openshift-image-registry

# Check registry configuration
oc get configs.imageregistry.operator.openshift.io cluster -o yaml

# Check registry storage (PVC or S3/Blob)
oc get configs.imageregistry.operator.openshift.io cluster \
  -o jsonpath='{.spec.storage}'

# Expose the registry externally (default is internal only)
oc patch configs.imageregistry.operator.openshift.io cluster \
  --type merge \
  --patch '{"spec":{"defaultRoute":true}}'

# Get the external registry URL
oc get route default-route -n openshift-image-registry \
  -o jsonpath='{.spec.host}'

# Log in to the internal registry
oc registry login
docker login -u $(oc whoami) -p $(oc whoami --show-token) \
  $(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
```

---

## 6. OpenShift Router (HAProxy)

**What it is:** The ingress router for OpenShift, backed by HAProxy. Handles all external traffic coming into the cluster on ports 80/443 and routes it to the appropriate pods based on Route objects.

**Key features:**
- Wildcard DNS: `*.apps.<cluster-domain>` points to router IPs
- TLS termination: edge, passthrough, re-encrypt
- Load balancing: roundrobin, leastconn, source (sticky)
- A/B traffic splitting via route weights

```bash
# Check router pods
oc get pods -n openshift-ingress

# Check IngressController
oc get ingresscontroller default -n openshift-ingress-operator -o yaml

# Check router service (external IPs)
oc get svc -n openshift-ingress

# View HAProxy stats (via router pod)
oc exec -n openshift-ingress \
  $(oc get pods -n openshift-ingress -l ingresscontroller.operator.openshift.io/deployment-ingresscontroller=default -o name | head -1) \
  -- cat /var/lib/haproxy/conf/haproxy.config | head -50
```

---

## 7. Cluster Operators (COs)

**What they are:** COs are controllers that manage a specific OpenShift component. They continuously watch their component and reconcile it to the desired state. COs are how OpenShift is self-managing.

**Status columns explained:**
- `AVAILABLE`: Component is running and healthy
- `PROGRESSING`: Component is being updated/reconciled
- `DEGRADED`: Component has errors or is not meeting its health checks

```bash
# View all cluster operators and their status
oc get clusteroperators

# Watch COs during an upgrade
watch oc get clusteroperators

# Check a specific CO
oc describe clusteroperator ingress

# Check CO conditions
oc get clusteroperator authentication \
  -o jsonpath='{.status.conditions}' | python3 -m json.tool
```

### Critical Cluster Operators

| Cluster Operator | Manages | Impact if Degraded |
|-----------------|---------|-------------------|
| authentication | OAuth server | Users cannot log in |
| etcd | etcd cluster | Cluster data loss risk |
| kube-apiserver | API server | All operations halt |
| ingress | HAProxy router | External traffic fails |
| image-registry | Internal registry | Builds fail |
| monitoring | Prometheus stack | No metrics/alerts |
| network | OVN-Kubernetes CNI | Pod networking broken |
| storage | CSI drivers | PVC provisioning fails |
| console | Web console | No UI access |
| dns | CoreDNS | Service discovery fails |

---

## 8. CRI-O Container Runtime

**What it is:** An OCI-compliant container runtime maintained by Red Hat. CRI-O replaces Docker as the container runtime in OpenShift 4.x. It implements the Kubernetes Container Runtime Interface (CRI).

**Why CRI-O (not containerd):**
- Designed specifically for Kubernetes (minimal footprint)
- Tight integration with Red Hat security features (SELinux, seccomp)
- No Docker daemon overhead

```bash
# CRI-O is accessible only from node debug sessions
oc debug node/<node-name>

# Inside the node debug pod:
chroot /host
crictl ps          # List running containers
crictl images      # List pulled images
crictl logs <id>   # Get container logs
crictl inspect <id>  # Inspect container
```

---

## 9. Machine API

**What it is:** The Machine API provides Kubernetes-native management of infrastructure (VMs). Instead of managing VMs through the Azure portal, you manage `Machine` objects in OpenShift.

**Key Resources:**

| Resource | Description |
|----------|-------------|
| `MachineSet` | Defines a pool of worker VMs (like a ReplicaSet for nodes) |
| `Machine` | Represents a single VM |
| `MachineHealthCheck` | Automatically replaces unhealthy machines |
| `MachineAutoscaler` | Links a MachineSet to the Cluster Autoscaler |
| `ClusterAutoscaler` | Global autoscaler configuration |

```bash
# View MachineSets
oc get machinesets -n openshift-machine-api

# View individual Machines
oc get machines -n openshift-machine-api

# Describe a MachineSet (shows Azure provider spec)
oc describe machineset <name> -n openshift-machine-api

# Check Machine health
oc get machine -n openshift-machine-api \
  -o custom-columns=NAME:.metadata.name,PHASE:.status.phase,NODE:.status.nodeRef.name

# View MachineHealthChecks
oc get machinehealthchecks -n openshift-machine-api
```

---

## 10. RHCOS (Red Hat CoreOS)

**What it is:** The immutable, container-optimized operating system used on all OpenShift nodes (masters and workers). RHCOS is based on RHEL and uses `rpm-ostree` for transactional system updates.

**Key principles:**
- **Immutable OS:** System files are read-only; changes are applied declaratively via Ignition
- **No SSH by default:** Node access is via `oc debug node/<name>`
- **MachineConfig:** Apply OS-level changes (files, systemd units, kernel args) via the MachineConfig API

```bash
# Access a worker node (creates a privileged debug pod)
oc debug node/<node-name>
chroot /host   # Access the host filesystem

# View node OS details
oc debug node/<node-name> -- chroot /host rpm-ostree status

# Apply OS configuration via MachineConfig
cat <<EOF | oc apply -f -
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-sysctl
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.4.0
    storage:
      files:
        - path: /etc/sysctl.d/99-custom.conf
          mode: 0644
          contents:
            source: data:,vm.max_map_count%3D262144
EOF

# Watch MachineConfig rollout
oc get machineconfigpool worker -w
```

### MachineConfig Pools

| Pool | Applies To | Notes |
|------|-----------|-------|
| `master` | Master nodes | Changes trigger master node reboots (rolling) |
| `worker` | Worker nodes | Changes trigger worker node reboots (rolling) |
| Custom | Specific nodes | Use for infra nodes, GPU nodes, etc. |

---

## Component Health Summary

| Component | Namespace | If Degraded |
|-----------|-----------|-------------|
| etcd | openshift-etcd | Cluster may lose quorum; emergency |
| kube-apiserver | openshift-kube-apiserver | No API access; all operations halt |
| openshift-apiserver | openshift-apiserver | OpenShift resources unavailable |
| authentication (OAuth) | openshift-authentication | Cannot log in; use kubeconfig |
| ingress (HAProxy) | openshift-ingress | External apps unreachable |
| image-registry | openshift-image-registry | S2I builds fail; image push fails |
| monitoring | openshift-monitoring | No metrics, no alerts |
| network (OVN-K) | openshift-ovn-kubernetes | Pod networking broken |
| dns (CoreDNS) | openshift-dns | Service DNS resolution fails |
| console | openshift-console | Web console unavailable |
| storage (CSI) | openshift-cluster-csi-drivers | PVC provisioning fails |
