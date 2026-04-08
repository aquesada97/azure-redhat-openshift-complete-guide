# OVN-Kubernetes Networking

## What is OVN-Kubernetes?

OVN-Kubernetes (OVN-K) is the default Container Network Interface (CNI) for OpenShift 4.12+. It replaced the older OpenShift SDN plugin and is based on **Open Virtual Network (OVN)** — a distributed, software-defined networking system.

OVN-K provides:
- Layer 2 and Layer 3 virtual networking
- Distributed load balancing
- Network Policies (enforced in the kernel via eBPF/OVS)
- Egress IP (stable outbound IPs for namespaces)
- Egress Firewall (control outbound traffic from namespaces)
- Hybrid overlay networking (Windows node support)

---

## Key Concepts

### Logical Components

| Component | Description |
|-----------|-------------|
| OVN North DB | Desired state — logical network topology |
| OVN South DB | Physical state — how to implement the logical topology |
| ovn-northd | Translates North DB to South DB |
| ovn-controller | Runs on each node, programs OVS based on South DB |
| OVS (Open vSwitch) | Kernel-level network switch on each node |

### Network Topology in ARO

```
Pod (node-1) → OVS bridge → Geneve tunnel → OVS bridge → Pod (node-2)
                    |                              |
              OVN controller                 OVN controller
                    |                              |
                OVN South DB ←── ovn-northd ←── OVN North DB
```

---

## Pod-to-Pod Communication

### Same Node
Pods on the same node communicate through the local OVS bridge without leaving the node. No tunneling overhead.

### Cross-Node
Pods on different nodes communicate via **Geneve tunnels** (UDP port 6081) established between nodes.

```bash
# Check pod IPs
oc get pods -o wide

# Test pod-to-pod connectivity
oc exec -it pod-a -- ping <pod-b-ip>
oc exec -it pod-a -- curl http://<pod-b-ip>:<port>
```

---

## Egress IP

Assign a stable outbound IP to a namespace. All pods in the namespace will appear to come from that IP when accessing external services. Useful for firewall rules.

```yaml
# Step 1: Label the node that will host the egress IP
# oc label node <node-name> k8s.ovn.org/egress-assignable=""

# Step 2: Create an EgressIP object
apiVersion: k8s.ovn.org/v1
kind: EgressIP
metadata:
  name: egressip-production
spec:
  egressIPs:
    - 10.0.2.100         # Must be within worker subnet CIDR
  namespaceSelector:
    matchLabels:
      environment: production
```

```bash
# Label namespace to use egress IP
oc label namespace my-prod-app environment=production

# Verify egress IP assignment
oc get egressip
oc describe egressip egressip-production

# Test from a pod in the namespace
oc exec -n my-prod-app -it mypod -- curl https://ifconfig.me
# Should return the egress IP (10.0.2.100)
```

---

## Egress Firewall

Control outbound traffic from a namespace using OVN Egress Firewall:

```yaml
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
  name: default
  namespace: my-project
spec:
  egress:
    # Allow traffic to Azure metadata service
    - type: Allow
      to:
        cidrSelector: 169.254.169.254/32
    # Allow DNS
    - type: Allow
      to:
        cidrSelector: 0.0.0.0/0
      ports:
        - protocol: UDP
          port: 53
    # Allow HTTPS to specific external IP range
    - type: Allow
      to:
        cidrSelector: 20.0.0.0/8
      ports:
        - protocol: TCP
          port: 443
    # Block all other external traffic
    - type: Deny
      to:
        cidrSelector: 0.0.0.0/0
```

---

## Checking OVN-K Health

```bash
# Check OVN-K pods
oc get pods -n openshift-ovn-kubernetes

# OVN-K has pods on every node:
# - ovnkube-master (runs on master nodes): ovnkube-master, ovn-northd, ovndb
# - ovnkube-node (runs on every node): ovnkube-controller, ovn-controller, ovs-vswitchd

# Check ovnkube-master status
oc get pods -n openshift-ovn-kubernetes -l app=ovnkube-master

# Check OVN databases
oc exec -n openshift-ovn-kubernetes \
  $(oc get pods -n openshift-ovn-kubernetes -l app=ovnkube-master -o name | head -1) \
  -c ovn-northd -- ovn-nbctl show | head -50

# Check cluster operator
oc get clusteroperator network
```

---

## Troubleshooting Network Connectivity

```bash
# Step 1: Check if pod has an IP
oc get pods -o wide

# Step 2: Test pod-to-pod (within namespace)
oc exec -it pod-a -- curl http://pod-b-ip:port

# Step 3: Test service DNS resolution
oc exec -it pod-a -- nslookup my-service.my-namespace.svc.cluster.local

# Step 4: Test service connectivity
oc exec -it pod-a -- curl http://my-service:80

# Step 5: Check NetworkPolicy (see 02-network-policies.md)
oc get networkpolicy

# Step 6: Debug at node level
oc debug node/<node-name>
chroot /host
ovs-vsctl show    # View OVS bridge config
ovn-nbctl show    # View OVN logical topology (master nodes only)
```
