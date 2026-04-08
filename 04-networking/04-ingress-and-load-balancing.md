# Ingress and Load Balancing

## OpenShift Ingress Operator

The Ingress Operator manages **IngressController** resources, which provision and configure HAProxy instances to handle external traffic.

```bash
# View the default IngressController
oc get ingresscontroller default -n openshift-ingress-operator -o yaml

# View IngressController status
oc describe ingresscontroller default -n openshift-ingress-operator

# View HAProxy pods managed by the IngressController
oc get pods -n openshift-ingress
```

---

## Default IngressController

ARO provisions one default IngressController that:
- Runs 2 HAProxy pods (for HA) on infrastructure nodes
- Handles all Routes in all namespaces
- Is exposed via an Azure Load Balancer (public or internal based on cluster visibility)
- Uses wildcard DNS: `*.apps.<cluster-domain>`

---

## Custom IngressController (Route Sharding)

Create a second IngressController to handle a subset of Routes (e.g., internal routes vs external routes):

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: internal
  namespace: openshift-ingress-operator
spec:
  domain: internal.apps.cluster.example.com
  endpointPublishingStrategy:
    type: LoadBalancerService
    loadBalancer:
      scope: Internal    # Azure Internal Load Balancer
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/infra: ""
  routeSelector:
    matchLabels:
      route-type: internal   # Only handle routes with this label
  replicas: 2
  defaultCertificate:
    name: internal-router-cert   # Secret with TLS cert for *.internal.apps...
```

```bash
# Apply the custom IngressController
oc apply -f internal-ingress-controller.yaml

# Watch it provision
oc get ingresscontroller internal -n openshift-ingress-operator -w

# Label a route to be handled by this controller
oc label route my-internal-route route-type=internal
```

---

## Configuring Custom TLS Certificates for the Router

Replace the self-signed wildcard certificate with a trusted certificate:

```bash
# Create a TLS secret with your wildcard certificate
oc create secret tls custom-router-cert \
  --cert=wildcard.crt \
  --key=wildcard.key \
  -n openshift-ingress

# Patch the IngressController to use it
oc patch ingresscontroller default \
  -n openshift-ingress-operator \
  --type merge \
  --patch '{"spec":{"defaultCertificate":{"name":"custom-router-cert"}}}'

# Verify
oc get ingresscontroller default -n openshift-ingress-operator \
  -o jsonpath='{.spec.defaultCertificate}'
```

---

## HAProxy Tuning

Configure HAProxy parameters via IngressController annotations or spec:

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: default
  namespace: openshift-ingress-operator
spec:
  tuningOptions:
    headerBufferBytes: 32768          # Increase for large headers
    headerBufferMaxRewriteBytes: 8192
    maxConnections: 50000             # Max simultaneous connections
    threadCount: 4                    # HAProxy threads per pod
  httpErrorCodePages:                 # Custom error pages
    name: custom-error-pages
```

---

## Kubernetes Ingress Support

ARO also supports standard Kubernetes `Ingress` resources (OpenShift 4.10+). The Ingress resource is handled by the OpenShift router:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  namespace: my-project
  annotations:
    route.openshift.io/termination: edge
spec:
  rules:
    - host: myapp.apps.cluster.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app
                port:
                  number: 80
```

> **Prefer OpenShift Routes** over Kubernetes Ingress for full access to OpenShift router features (A/B routing, passthrough TLS, sticky sessions).

---

## Path-Based vs Host-Based Routing

| Routing Type | Route Example | Use Case |
|-------------|---------------|---------|
| Host-based | `myapp.apps.cluster.com` | Separate subdomains per service |
| Path-based | `api.apps.cluster.com/v1`, `api.apps.cluster.com/v2` | API versioning, microservices |

```yaml
# Path-based routing with two services
# Route 1: /api → api-service
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: api-route
spec:
  host: api.apps.cluster.example.com
  path: /api
  to:
    kind: Service
    name: api-service
---
# Route 2: / → frontend-service
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: frontend-route
spec:
  host: api.apps.cluster.example.com
  path: /
  to:
    kind: Service
    name: frontend-service
```
