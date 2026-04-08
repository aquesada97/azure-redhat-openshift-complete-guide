# Performance Tuning

## Node Sizing Recommendations

| Workload Type | Recommended VM Size | Notes |
|--------------|--------------------|----|
| General purpose | Standard_D4s_v3 (4 vCPU, 16 GB) | Default ARO worker |
| Memory-intensive | Standard_E8s_v3 (8 vCPU, 64 GB) | Databases, JVM apps |
| CPU-intensive | Standard_F8s_v2 (8 vCPU, 16 GB) | Compute, analytics |
| GPU workloads | Standard_NC6s_v3 | ML/AI inference |
| High-storage | Standard_L8s_v2 (NVMe local disk) | Data processing |

---

## Topology Spread Constraints

Spread pods across zones and nodes for high availability:

```yaml
spec:
  topologySpreadConstraints:
    # Spread across availability zones
    - maxSkew: 1
      topologyKey: topology.kubernetes.io/zone
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app: my-app
    # Spread across nodes
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway   # Soft constraint
      labelSelector:
        matchLabels:
          app: my-app
```

---

## Pod Affinity and Anti-Affinity

```yaml
spec:
  affinity:
    # Schedule near pods with same app (for low-latency communication)
    podAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app: cache
            topologyKey: kubernetes.io/hostname

    # Schedule AWAY from other replicas (for HA)
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app: my-app
          topologyKey: kubernetes.io/hostname

    # Schedule on nodes with specific labels
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: node-role.kubernetes.io/worker
                operator: Exists
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 50
          preference:
            matchExpressions:
              - key: topology.kubernetes.io/zone
                operator: In
                values:
                  - eastus-1
                  - eastus-2
```

---

## Taints and Tolerations

Reserve nodes for specific workloads:

```bash
# Taint a node (only pods with matching toleration can schedule here)
oc adm taint nodes worker-gpu-01 \
  nvidia.com/gpu=true:NoSchedule

# Taint an entire MachineSet (apply to future nodes too)
# Edit MachineSet spec.template.spec.taints
```

```yaml
# Pod toleration to schedule on tainted node
spec:
  tolerations:
    - key: "nvidia.com/gpu"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  nodeSelector:
    nvidia.com/gpu: "true"   # Combined with toleration
```

---

## Dedicated Infrastructure Nodes

Move platform workloads off worker nodes by creating infra MachineSets:

```bash
# Create an infra MachineSet (copy worker MachineSet, change role label)
# Add taint to infra nodes:
oc adm taint nodes infra-node-01 node-role.kubernetes.io/infra=:NoSchedule

# Label infra nodes
oc label node infra-node-01 node-role.kubernetes.io/infra=""

# Move router to infra nodes
oc patch ingresscontroller default \
  -n openshift-ingress-operator \
  --type merge \
  --patch '{
    "spec": {
      "nodePlacement": {
        "nodeSelector": {
          "matchLabels": {
            "node-role.kubernetes.io/infra": ""
          }
        },
        "tolerations": [
          {
            "key": "node-role.kubernetes.io/infra",
            "operator": "Exists",
            "effect": "NoSchedule"
          }
        ]
      }
    }
  }'
```

---

## Tuned Operator (OS-Level Performance)

```yaml
# Apply custom kernel parameters
apiVersion: tuned.openshift.io/v1
kind: Tuned
metadata:
  name: high-performance
  namespace: openshift-cluster-node-tuning-operator
spec:
  profile:
    - name: high-performance
      data: |
        [main]
        summary=High performance profile
        [sysctl]
        vm.swappiness=10
        vm.max_map_count=262144
        net.core.somaxconn=65535
        net.ipv4.tcp_max_syn_backlog=65535
        [bootloader]
        cmdline=skew_tick=1 nohz=on nohz_full=1-3 rcu_nocbs=1-3
  recommend:
    - machineConfigLabels:
        machineconfiguration.openshift.io/role: worker-highperf
      match:
        - label:
            type: node
            value: worker-highperf
      priority: 20
      profile: high-performance
```
