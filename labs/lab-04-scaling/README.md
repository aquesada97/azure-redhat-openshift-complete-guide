# Lab 04: Scaling and Autoscaling

## Overview

Configure HPA for automatic pod scaling based on CPU, generate load to trigger scale-out, and configure the Cluster Autoscaler to add nodes.

**Estimated Time:** 60 minutes  
**Prerequisites:** Lab 01, Modules 08-09 (Monitoring and Scaling)

---

## Objectives

- Deploy an app with HPA configured
- Generate load to trigger HPA scale-out
- Observe HPA decisions in real time
- Configure MachineAutoscaler for node-level scaling
- Observe node provisioning

---

## Step 1: Create Project and Deploy App

```bash
oc new-project lab-04 --display-name="Lab 04 - Scaling"
oc apply -f manifests/deployment.yaml
oc apply -f manifests/hpa.yaml
```

---

## Step 2: Verify Initial State

```bash
# Check deployment (1 replica initially)
oc get deployment load-test-app

# Check HPA status
oc get hpa load-test-app-hpa
# TARGETS should show current CPU usage

# Note: HPA requires metrics server and CPU requests to be set
```

---

## Step 3: Generate Load

Deploy a load generator that sends continuous HTTP requests:

```bash
# Deploy load generator
oc run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "
    while true; do
      wget -q -O- http://load-test-app.lab-04.svc.cluster.local
    done
  "

# In a separate terminal, watch HPA
oc get hpa load-test-app-hpa -w

# Also watch pods being added
oc get pods -w
```

---

## Step 4: Observe HPA Scaling

```bash
# Watch CPU climb and HPA react
watch -n 5 "oc get hpa load-test-app-hpa && echo && oc get pods -l app=load-test-app"

# Check HPA events
oc describe hpa load-test-app-hpa | grep -A 20 Events

# Typical flow:
# CPU: 5% → 80% (load generator running)
# HPA: REPLICAS 1 → 3 → 5 → 10 (scaling up)
```

---

## Step 5: Stop Load and Observe Scale-Down

```bash
# Stop the load generator
oc delete pod load-generator

# Watch HPA scale back down (takes 5 min by default due to stabilizationWindow)
watch -n 30 "oc get hpa load-test-app-hpa && oc get pods -l app=load-test-app"
```

---

## Step 6: Configure Cluster Autoscaler (Optional — requires cluster-admin)

```bash
# Apply MachineAutoscaler (requires MachineSet name from your cluster)
MACHINESET=$(oc get machinesets -n openshift-machine-api -o name | head -1 | cut -d/ -f2)

# Update the MachineAutoscaler with your MachineSet name
sed "s/MACHINESET_NAME/$MACHINESET/" manifests/machineautoscaler.yaml | oc apply -f -

# Apply ClusterAutoscaler
cat <<EOF | oc apply -f -
apiVersion: autoscaling.openshift.io/v1
kind: ClusterAutoscaler
metadata:
  name: default
spec:
  resourceLimits:
    maxNodesTotal: 10
  scaleDown:
    enabled: true
    delayAfterAdd: 10m
    unneededTime: 10m
EOF
```

---

## Cleanup

```bash
oc delete pod load-generator --ignore-not-found
oc delete project lab-04
```
