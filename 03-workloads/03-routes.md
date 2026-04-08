# OpenShift Routes

## What is a Route?

An OpenShift Route is a first-class API object that exposes a Service externally via the cluster's built-in HAProxy router. Routes are the primary mechanism for HTTP/HTTPS ingress in OpenShift.

**Key features:**
- Automatic hostname: `<route-name>-<namespace>.apps.<cluster-domain>`
- TLS termination: edge, passthrough, or re-encrypt
- Load balancing: roundrobin, leastconn, source (sticky sessions)
- A/B testing via route weights
- No additional ingress controller needed

---

## Route Types

### 1. Unsecured Route (HTTP only)

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app-http
  namespace: my-project
spec:
  host: myapp.apps.cluster.example.com  # Omit to auto-generate
  to:
    kind: Service
    name: my-app
    weight: 100
  port:
    targetPort: http  # Named port from the Service, or a number
```

### 2. Edge TLS (TLS terminated at the router, HTTP to pod)

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app-edge
  namespace: my-project
spec:
  to:
    kind: Service
    name: my-app
  port:
    targetPort: 8080
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect  # HTTP → HTTPS redirect
    # Optional: provide custom certificate
    # certificate: |-
    #   -----BEGIN CERTIFICATE-----
    #   ...
    # key: |-
    #   -----BEGIN RSA PRIVATE KEY-----
    #   ...
    # caCertificate: |-
    #   ...
```

### 3. Passthrough TLS (TLS all the way to the pod)

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app-passthrough
  namespace: my-project
spec:
  to:
    kind: Service
    name: my-app-tls
  port:
    targetPort: 8443
  tls:
    termination: passthrough
    # insecureEdgeTerminationPolicy not applicable for passthrough
```

> The pod must handle TLS itself. The router forwards raw TCP to the pod.

### 4. Re-encrypt TLS

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app-reencrypt
  namespace: my-project
spec:
  to:
    kind: Service
    name: my-app
  port:
    targetPort: 8443
  tls:
    termination: reencrypt
    # Certificate for the external-facing side (router → client)
    certificate: |-
      -----BEGIN CERTIFICATE-----
      ...
    key: |-
      -----BEGIN RSA PRIVATE KEY-----
      ...
    # CA cert to validate the backend pod's certificate
    destinationCACertificate: |-
      -----BEGIN CERTIFICATE-----
      ...
```

---

## Creating Routes with oc CLI

```bash
# Create an unsecured route (auto-hostname)
oc expose svc/my-app

# Create an edge TLS route
oc create route edge my-app-edge \
  --service=my-app \
  --insecure-policy=Redirect

# Create a passthrough route
oc create route passthrough my-app-pass \
  --service=my-app-tls \
  --port=8443

# Create a route with custom hostname
oc expose svc/my-app \
  --hostname=myapp.example.com

# Get route URL
oc get route my-app -o jsonpath='{.spec.host}'
```

---

## A/B Routing with Weights

Split traffic between two services (e.g., 90% v1, 10% v2):

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: my-app-ab
  namespace: my-project
spec:
  to:
    kind: Service
    name: my-app-v1
    weight: 90
  alternateBackends:
    - kind: Service
      name: my-app-v2
      weight: 10
  port:
    targetPort: 8080
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

---

## Load Balancing Strategies

Configure via route annotation:

```yaml
metadata:
  annotations:
    haproxy.router.openshift.io/balance: roundrobin
    # Options: roundrobin (default), leastconn, source (sticky sessions by IP)
    
    # Sticky sessions via cookie
    haproxy.router.openshift.io/disable_cookies: "false"
    
    # Connection timeouts
    haproxy.router.openshift.io/timeout: 2m
    
    # Max connections
    haproxy.router.openshift.io/pod-concurrent-connections: "200"
    
    # Rate limiting (requests per second per IP)
    haproxy.router.openshift.io/rate-limit-connections: "true"
    haproxy.router.openshift.io/rate-limit-connections.rate-http: "100"
```

---

## Inspecting Routes

```bash
# List all routes in current project
oc get routes

# Get route with URL
oc get route my-app -o jsonpath='{.spec.host}'

# Full route details
oc describe route my-app

# Test the route
curl https://$(oc get route my-app -o jsonpath='{.spec.host}')
curl -k https://$(oc get route my-app -o jsonpath='{.spec.host}')  # Skip TLS verify
```

---

## TLS Certificate Management

By default, ARO routes use the wildcard certificate for `*.apps.<cluster-domain>`. For custom domains, provide your own certificate.

```bash
# Create route with custom TLS certificate from files
oc create route edge my-custom-route \
  --service=my-app \
  --cert=tls.crt \
  --key=tls.key \
  --ca-cert=ca.crt \
  --hostname=myapp.example.com

# Update certificate on existing route
oc patch route my-app \
  --type merge \
  --patch '{"spec":{"tls":{"certificate":"<base64-cert>","key":"<base64-key>"}}}'
```
