# Cluster Autoscaler and Machine API

## Machine API Overview

The Machine API is OpenShift's Kubernetes-native approach to managing infrastructure (worker nodes). Instead of managing VMs through the Azure portal, you create Kubernetes objects.

```bash
# View all MachineSets (worker node pools)
oc get machinesets -n openshift-machine-api

# View individual Machines (VM instances)
oc get machines -n openshift-machine-api

# Check Machine status
oc get machines -n openshift-machine-api \
  -o custom-columns="NAME:.metadata.name,PHASE:.status.phase,NODE:.status.nodeRef.name,AGE:.metadata.creationTimestamp"
```

---

## MachineSet YAML

```yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
  name: aro-cluster-workers-eastus1
  namespace: openshift-machine-api
  labels:
    machine.openshift.io/cluster-api-cluster: aro-cluster
spec:
  replicas: 3
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-cluster: aro-cluster
      machine.openshift.io/cluster-api-machineset: aro-cluster-workers-eastus1
  template:
    metadata:
      labels:
        machine.openshift.io/cluster-api-cluster: aro-cluster
        machine.openshift.io/cluster-api-machine-role: worker
        machine.openshift.io/cluster-api-machineset: aro-cluster-workers-eastus1
    spec:
      providerSpec:
        value:
          apiVersion: azureproviderconfig.openshift.io/v1beta1
          kind: AzureMachineProviderSpec
          location: eastus
          vmSize: Standard_D4s_v3
          osDisk:
            diskSizeGB: 128
            managedDisk:
              storageAccountType: Premium_LRS
            osType: Linux
          image:
            offer: aro4
            publisher: azureopenshift
            resourceID: ""
            sku: aro_412
            version: latest
          networkResourceGroup: rg-aro-training
          vnet: vnet-aro
          subnet: subnet-worker
          zone: "1"                   # Azure availability zone
          publicIP: false
          publicLoadBalancer: true
```

---

## MachineAutoscaler

The MachineAutoscaler links a MachineSet to the Cluster Autoscaler, providing min/max bounds:

```yaml
apiVersion: autoscaling.openshift.io/v1beta1
kind: MachineAutoscaler
metadata:
  name: worker-autoscaler-eastus1
  namespace: openshift-machine-api
spec:
  minReplicas: 2          # Never scale below 2 nodes
  maxReplicas: 10         # Never scale above 10 nodes
  scaleTargetRef:
    apiVersion: machine.openshift.io/v1beta1
    kind: MachineSet
    name: aro-cluster-workers-eastus1
```

---

## ClusterAutoscaler

The ClusterAutoscaler makes cluster-wide scaling decisions:

```yaml
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  resourceLimits:
    maxNodesTotal: 24               # Cluster-wide max nodes
    cores:
      min: 8
      max: 96
    memory:
      min: 4
      max: 384                     # In GiB
  scaleDown:
    enabled: true
    delayAfterAdd: 10m             # Wait 10 min after scale-up before checking scale-down
    delayAfterDelete: 0s
    delayAfterFailure: 3m
    unneededTime: 10m              # Node must be unneeded for 10 min before removal
    utilizationThreshold: "0.5"   # Scale down nodes with <50% utilization
  balanceSimilarNodeGroups: false
  ignoreDaemonSetsUtilization: false
  skipNodesWithLocalStorage: true
  skipNodesWithSystemPods: true
  logVerbosity: 1
```

---

## Triggering Scale-Up

The Cluster Autoscaler adds nodes when pods cannot be scheduled due to insufficient resources:

```bash
# Deploy a resource-intensive app to trigger scale-up
cat <<EOF | oc apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-hog
  namespace: my-project
spec:
  replicas: 20
  selector:
    matchLabels:
      app: resource-hog
  template:
    metadata:
      labels:
        app: resource-hog
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
EOF

# Watch autoscaler decisions
oc get events -n openshift-machine-api --sort-by='.lastTimestamp' -w

# Watch new nodes being added
oc get machines -n openshift-machine-api -w
oc get nodes -w
```

---

## MachineHealthCheck

Automatically replace unhealthy nodes:

```yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineHealthCheck
metadata:
  name: worker-health-check
  namespace: openshift-machine-api
spec:
  selector:
    matchLabels:
      machine.openshift.io/cluster-api-machine-role: worker
  unhealthyConditions:
    - type: Ready
      status: Unknown
      timeout: 300s       # Replace node if Unknown for 5 min
    - type: Ready
      status: "False"
      timeout: 300s       # Replace node if NotReady for 5 min
  maxUnhealthy: "40%"     # Stop replacing if >40% of nodes are unhealthy (cluster issue)
  nodeStartupTimeout: 10m  # Time for new node to become Ready
```

```bash
# View MachineHealthChecks
oc get machinehealthchecks -n openshift-machine-api

# Check remediation history
oc get machineremediations -n openshift-machine-api
```
