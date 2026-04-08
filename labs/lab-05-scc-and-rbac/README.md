# Lab 05: SCC and RBAC

## Overview

Explore OpenShift Security Context Constraints and RBAC by deploying pods that require different privilege levels and configuring fine-grained access control.

**Estimated Time:** 45 minutes  
**Prerequisites:** Lab 01, Module 06 (Security)

---

## Objectives

- Create a ServiceAccount with minimal permissions
- Observe SCC enforcement (pod rejected without proper SCC)
- Grant an SCC to a ServiceAccount
- Create a custom Role and RoleBinding
- Verify permissions with `oc auth can-i`

---

## Step 1: Create Project

```bash
oc new-project lab-05 --display-name="Lab 05 - SCC and RBAC"
```

---

## Step 2: Create ServiceAccount

```bash
oc apply -f manifests/serviceaccount.yaml

# Check the default SCC for new service accounts
oc get sa lab-sa -n lab-05 -o yaml
```

---

## Step 3: Deploy Pod with Default SCC (restricted-v2)

```bash
oc apply -f manifests/pod-with-sa.yaml

# Check if pod is running
oc get pod test-pod

# If it fails, check why:
oc describe pod test-pod | grep -A 10 "Events:"
```

---

## Step 4: Create Custom Role and RoleBinding

```bash
oc apply -f manifests/role.yaml
oc apply -f manifests/rolebinding.yaml

# Verify permissions
oc auth can-i get pods --as system:serviceaccount:lab-05:lab-sa -n lab-05
# Expected: yes

oc auth can-i delete pods --as system:serviceaccount:lab-05:lab-sa -n lab-05
# Expected: no

oc auth can-i create deployments --as system:serviceaccount:lab-05:lab-sa -n lab-05
# Expected: no
```

---

## Step 5: Test SCC Behavior

```bash
# Try to deploy a pod that needs root (will fail with restricted-v2)
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
  namespace: lab-05
spec:
  serviceAccountName: lab-sa
  containers:
    - name: root-container
      image: nginx:1.25
      securityContext:
        runAsUser: 0    # Request root - will be rejected
EOF

# Check the error
oc describe pod root-pod | grep -A 5 "Events:"
# Expected: Error creating: pods "root-pod" is forbidden: unable to validate against any security context constraint

# Grant anyuid SCC to allow running as root
oc adm policy add-scc-to-user anyuid -z lab-sa -n lab-05

# Now the pod should be accepted
oc delete pod root-pod
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
  namespace: lab-05
spec:
  serviceAccountName: lab-sa
  containers:
    - name: root-container
      image: nginx:1.25
      securityContext:
        runAsUser: 0
EOF

oc get pod root-pod
```

---

## Step 6: Check SCC Assignment

```bash
# Which SCC is being used?
oc get pod root-pod -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
# Expected: anyuid

# Who can use anyuid SCC?
oc adm policy who-can use scc anyuid | grep lab-05
```

---

## Cleanup

```bash
oc delete project lab-05
```
