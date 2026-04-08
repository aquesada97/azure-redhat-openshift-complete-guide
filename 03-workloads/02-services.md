# Services in OpenShift

## Service Types

| Type | Description | Use Case |
|------|-------------|---------|
| `ClusterIP` | Internal-only virtual IP | Pod-to-pod communication within cluster |
| `NodePort` | Exposes service on each node's IP at a static port | Dev/test direct access |
| `LoadBalancer` | Provisions Azure Load Balancer | External access to a service |
| `ExternalName` | DNS alias to external service | Integrate external services by DNS name |

---

## ClusterIP Service

The default and most common type — accessible only within the cluster:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-project
  labels:
    app: my-app
spec:
  type: ClusterIP
  selector:
    app: my-app          # Matches pod labels
  ports:
    - name: http
      port: 80           # Port the Service listens on
      targetPort: 8080   # Port on the pod
      protocol: TCP
```

---

## LoadBalancer Service (Azure)

Creates an Azure Load Balancer and assigns an external IP:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-external
  namespace: my-project
  annotations:
    # Use internal load balancer (within VNet only)
    service.beta.kubernetes.io/azure-load-balancer-internal: "true"
    # Specify internal LB subnet
    service.beta.kubernetes.io/azure-load-balancer-internal-subnet: "subnet-worker"
    # Use a specific static public IP (must exist in ARO managed RG)
    # service.beta.kubernetes.io/azure-pip-name: "my-static-ip"
    # Idle timeout (in minutes, default: 4)
    service.beta.kubernetes.io/azure-load-balancer-tcp-idle-timeout: "30"
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - port: 443
      targetPort: 8443
```

> **Prefer OpenShift Routes over LoadBalancer services** for HTTP/HTTPS traffic. Routes use the shared ARO ingress router and don't require additional Azure LB resources per service.

---

## Multi-Port Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-multiport-app
spec:
  selector:
    app: my-multiport-app
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: https
      port: 443
      targetPort: 8443
    - name: metrics
      port: 9090
      targetPort: 9090
```

---

## Headless Service (for StatefulSets)

A headless service has no ClusterIP — DNS returns individual pod IPs instead:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-db-headless
spec:
  clusterIP: None     # This makes it headless
  selector:
    app: my-db
  ports:
    - port: 5432
      targetPort: 5432
```

DNS for headless services returns A records for each pod:
- `my-db-headless.my-project.svc.cluster.local` → all pod IPs
- `my-db-0.my-db-headless.my-project.svc.cluster.local` → specific pod IP

---

## Service Discovery

### DNS-Based Discovery

Every Service gets a DNS entry automatically:

```
<service-name>.<namespace>.svc.cluster.local
```

From within the same namespace, you can use just the service name:
```bash
curl http://my-app        # Same namespace
curl http://my-app.other-ns  # Cross-namespace
curl http://my-app.other-ns.svc.cluster.local  # Full FQDN
```

### Environment Variables

Kubernetes injects service environment variables into pods (for services that existed before the pod started):
```bash
MY_APP_SERVICE_HOST=10.96.0.100
MY_APP_SERVICE_PORT=80
MY_APP_PORT=tcp://10.96.0.100:80
```

> **Prefer DNS over environment variables** — env vars require service creation order and don't update if the ClusterIP changes.

---

## Debugging Services

```bash
# Check service exists and has the right selector
oc get svc my-app -o yaml

# Check if service has endpoints (pods matched by selector)
oc get endpoints my-app
oc describe endpoints my-app

# If endpoints is empty, the selector doesn't match any pods
# Check pod labels match service selector:
oc get pods --show-labels
oc get svc my-app -o jsonpath='{.spec.selector}'

# Test service connectivity from another pod
oc run -it --rm test-pod --image=curlimages/curl --restart=Never -- \
  curl http://my-app.my-project.svc.cluster.local
```

---

## ExternalName Service

Maps a service to an external DNS name (no proxying, just DNS CNAME):

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
  namespace: my-project
spec:
  type: ExternalName
  externalName: my-db.postgres.database.azure.com
```

Pods in `my-project` can now reach the database at `external-db` (DNS resolves to `my-db.postgres.database.azure.com`).
