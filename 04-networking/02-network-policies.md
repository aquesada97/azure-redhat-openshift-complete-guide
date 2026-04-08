# Network Policies

## What are Network Policies?

Network Policies are Kubernetes resources that control which pods can communicate with which other pods and external endpoints. In OpenShift with OVN-Kubernetes, Network Policies are enforced at the kernel level via OVS flow rules.

**Default behavior (no policy):** All traffic is allowed between all pods in all namespaces.

---

## Default Deny All Ingress

Start with a deny-all policy and explicitly allow only required traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: my-project
spec:
  podSelector: {}     # Applies to ALL pods in this namespace
  policyTypes:
    - Ingress         # Deny all incoming traffic
```

---

## Allow Same-Namespace Traffic

After default deny, allow pods within the same namespace to talk to each other:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: my-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector: {}   # Any pod in the same namespace
```

---

## Allow Traffic from a Specific Namespace

Allow traffic from the `frontend` namespace to pods in `backend` namespace:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: frontend
```

---

## Allow Specific Pod Labels

Allow only pods with label `role: api-consumer` to reach the database:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-db
  namespace: my-project
spec:
  podSelector:
    matchLabels:
      app: postgres
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: api-consumer
      ports:
        - protocol: TCP
          port: 5432
```

---

## Allow OpenShift Router Ingress

Allow traffic from the OpenShift router to reach your application pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-ingress
  namespace: my-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress
```

---

## Allow Prometheus Monitoring Scraping

Allow the monitoring namespace to scrape metrics:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-openshift-monitoring
  namespace: my-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: monitoring
      ports:
        - protocol: TCP
          port: 9090    # metrics port
```

---

## Egress Network Policy

Deny all outbound traffic from a namespace (except DNS):

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: my-project
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53     # Allow DNS
        - protocol: TCP
          port: 53     # Allow DNS (TCP fallback)
```

---

## Complete Namespace Isolation Policy Set

A production-ready policy set for full namespace isolation:

```bash
# Apply all policies
cat <<EOF | oc apply -f -
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: my-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal
  namespace: my-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - podSelector: {}
  egress:
    - to:
        - podSelector: {}
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: my-project
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-router-ingress
  namespace: my-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring
  namespace: my-project
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: monitoring
EOF
```

---

## Testing Network Policies

```bash
# Deploy a test pod in a different namespace
oc run test-pod -n other-namespace --image=curlimages/curl -it --rm -- \
  curl http://my-service.my-project.svc.cluster.local
# If network policy blocks: "Connection refused" or timeout

# Deploy a test pod in the SAME namespace (should succeed if allow-internal applied)
oc run test-pod -n my-project --image=curlimages/curl -it --rm -- \
  curl http://my-service
```

> **Common mistake:** Applying default-deny without adding the router/monitoring allow rules. This breaks Route access and Prometheus scraping silently.
