# Lab 02: Routes and Ingress

## Overview

Deploy two versions of an application and configure advanced OpenShift Route features: A/B routing with weighted traffic splitting, different TLS termination modes, and sticky sessions.

**Estimated Time:** 45 minutes  
**Prerequisites:** Lab 01 complete, Module 03 (Routes)

---

## Objectives

- Deploy two versions of an application (v1 and v2)
- Create separate Services for each version
- Configure A/B routing (90% v1, 10% v2)
- Observe traffic distribution
- Configure sticky sessions
- Test passthrough TLS route

---

## Step 1: Create Project and Deploy Both Versions

```bash
oc new-project lab-02 --display-name="Lab 02 - Routes"

# Deploy v1 and v2
oc apply -f manifests/deployment-v1.yaml
oc apply -f manifests/deployment-v2.yaml
oc apply -f manifests/service-v1.yaml
oc apply -f manifests/service-v2.yaml

# Verify both deployments
oc get pods -l 'app in (my-app-v1, my-app-v2)'
```

---

## Step 2: Create A/B Route with Weights

```bash
oc apply -f manifests/route-ab.yaml

ROUTE_URL=$(oc get route my-app-ab -o jsonpath='{.spec.host}')
echo "A/B Route URL: https://$ROUTE_URL"
```

---

## Step 3: Observe Traffic Distribution

Run 20 requests and count v1 vs v2 responses:

```bash
V1=0; V2=0
for i in {1..20}; do
  RESPONSE=$(curl -sk https://$ROUTE_URL | grep -o "v[12]" | head -1)
  if [ "$RESPONSE" = "v1" ]; then
    ((V1++))
  elif [ "$RESPONSE" = "v2" ]; then
    ((V2++))
  fi
done
echo "v1: $V1 requests, v2: $V2 requests"
# Expect approximately: v1: 18, v2: 2 (90/10 split)
```

---

## Step 4: Shift 50% of Traffic to v2

```bash
oc patch route my-app-ab \
  --type json \
  --patch '[
    {"op": "replace", "path": "/spec/to/weight", "value": 50},
    {"op": "replace", "path": "/spec/alternateBackends/0/weight", "value": 50}
  ]'

# Re-run traffic test — should be ~50/50 split now
```

---

## Step 5: Configure Sticky Sessions

```bash
# Annotate route for sticky sessions (source IP-based)
oc annotate route my-app-ab \
  haproxy.router.openshift.io/balance=source \
  --overwrite

# Now all requests from your IP go to the same pod
for i in {1..5}; do curl -sk https://$ROUTE_URL | grep -o "v[12]"; done
# Should always return same version
```

---

## Step 6: Promote v2 to 100% Traffic

```bash
# Full cutover to v2
oc patch route my-app-ab \
  --type json \
  --patch '[
    {"op": "replace", "path": "/spec/to/name", "value": "my-app-v2"},
    {"op": "replace", "path": "/spec/to/weight", "value": 100},
    {"op": "replace", "path": "/spec/alternateBackends", "value": []}
  ]'
```

---

## Cleanup

```bash
oc delete project lab-02
```
