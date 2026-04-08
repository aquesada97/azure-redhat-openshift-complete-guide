# Lab 01: Deploy Your First App

## Overview

Deploy a containerized NGINX web application to ARO, expose it via an OpenShift Route, and scale it.

**Estimated Time:** 30 minutes  
**Prerequisites:** Modules 01-02 complete, `oc` logged in as project admin

---

## Objectives

- Create a project
- Deploy a containerized app with resource limits
- Expose the app via a ClusterIP Service
- Create an edge TLS Route
- Verify external connectivity
- Scale the deployment
- Observe load balancing across replicas

---

## Step 1: Create Project

```bash
oc new-project lab-01 \
  --display-name="Lab 01 - First App" \
  --description="Deploy, expose, and scale NGINX"

oc project lab-01
```

---

## Step 2: Deploy the Application

```bash
# Apply the Deployment manifest
oc apply -f manifests/deployment.yaml

# Watch pods start up
oc get pods -w
# Wait until: my-app-xxxxx-xxxxx   1/1   Running

# Check the deployment
oc get deployment my-app
oc describe deployment my-app
```

---

## Step 3: Create the Service

```bash
oc apply -f manifests/service.yaml

# Verify service and endpoints
oc get service my-app
oc get endpoints my-app
# Endpoints should show pod IP:80
```

---

## Step 4: Create the Route

```bash
oc apply -f manifests/route.yaml

# Get the route URL
ROUTE_URL=$(oc get route my-app -o jsonpath='{.spec.host}')
echo "Route URL: https://$ROUTE_URL"

# Test connectivity
curl -k https://$ROUTE_URL
# Should return the NGINX default page HTML
```

---

## Step 5: Scale Up

```bash
# Scale to 3 replicas
oc scale deployment/my-app --replicas=3

# Watch new pods start
oc get pods -w

# Verify 3 pods running
oc get pods -l app=my-app
```

---

## Step 6: Verify Load Balancing

```bash
# Run multiple requests — HAProxy distributes across pods
for i in {1..6}; do
  curl -sk https://$ROUTE_URL | grep -i "server name" || echo "Request $i: OK"
done
```

---

## Step 7: View Logs

```bash
# View logs from all pods matching the label
oc logs -l app=my-app --tail=10

# Follow logs from a specific pod
POD=$(oc get pods -l app=my-app -o name | head -1)
oc logs -f $POD
```

---

## Bonus: Trigger a Rolling Update

```bash
# Update the image tag to trigger a rolling update
oc set image deployment/my-app my-app=nginx:1.26

# Watch rolling update
oc rollout status deployment/my-app

# View history
oc rollout history deployment/my-app

# Rollback if needed
oc rollout undo deployment/my-app
```

---

## Cleanup

```bash
oc delete project lab-01
```

---

## Expected Outcomes

- [ ] Pod running with `1/1 Running` status
- [ ] Service has endpoints matching pod IPs
- [ ] Route accessible via `curl` returning HTTP 200
- [ ] Scaling to 3 replicas succeeds
- [ ] Load balancing verified across replicas
