# Deployments in OpenShift

## Deployment vs DeploymentConfig

OpenShift historically used `DeploymentConfig` (DC) as its deployment primitive. Since OCP 4.x, standard Kubernetes `Deployment` is the recommended approach.

| Feature | Deployment (K8s) | DeploymentConfig (OpenShift) |
|---------|-----------------|------------------------------|
| API group | apps/v1 | apps.openshift.io/v1 |
| Status in OCP 4.14+ | Recommended | Deprecated |
| Rolling updates | Yes | Yes |
| Recreate strategy | Yes | Yes |
| ImageStream triggers | Via annotation | Native |
| Lifecycle hooks | No | Yes (pre/mid/post) |
| Custom strategies | No | Yes |
| `oc rollout` | Yes | Yes |

> **Use standard `Deployment` for all new workloads.** DeploymentConfigs are deprecated as of OCP 4.14.

---

## Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-project
  labels:
    app: my-app
    version: "1.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max extra pods during update
      maxUnavailable: 0  # No downtime (zero-downtime rollout)
  template:
    metadata:
      labels:
        app: my-app
        version: "1.0"
    spec:
      containers:
        - name: my-app
          image: nginx:1.25
          ports:
            - containerPort: 8080
              protocol: TCP
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "256Mi"
          env:
            - name: APP_ENV
              value: "production"
            - name: DB_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
            - name: CONFIG_VALUE
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: value
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
            failureThreshold: 3
          imagePullPolicy: IfNotPresent
          securityContext:
            allowPrivilegeEscalation: false
            runAsNonRoot: true
            seccompProfile:
              type: RuntimeDefault
            capabilities:
              drop:
                - ALL
```

---

## Resource Requests and Limits

**Requests** are what the scheduler uses to find a node with enough capacity. **Limits** are enforced by the container runtime.

| Field | Purpose | Scheduler Uses | Runtime Enforces |
|-------|---------|----------------|-----------------|
| `requests.cpu` | Guaranteed CPU | Yes (node selection) | No (burstable) |
| `requests.memory` | Guaranteed memory | Yes (node selection) | No (but OOM kill if exceeded) |
| `limits.cpu` | Max CPU (throttle) | No | Yes (CPU throttling) |
| `limits.memory` | Max memory | No | Yes (OOM kill) |

> **In ARO, resource requests are required** for most SCCs. Without requests, pods may be rejected by the `restricted-v2` SCC.

```bash
# Check resource usage vs requests/limits
oc adm top pods --containers
oc describe node <node> | grep -A 20 "Allocated resources"
```

---

## Liveness vs Readiness vs Startup Probes

| Probe | Purpose | Failure Action |
|-------|---------|----------------|
| `livenessProbe` | Is the container healthy? | Restart container |
| `readinessProbe` | Is the container ready to serve traffic? | Remove from Service endpoints |
| `startupProbe` | Has the app finished starting? | Prevents liveness probe during slow startup |

### Probe Types

```yaml
# HTTP GET (most common for web apps)
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
    httpHeaders:
      - name: Authorization
        value: Bearer <token>
  initialDelaySeconds: 10
  periodSeconds: 10

# TCP Socket (for non-HTTP services like databases)
livenessProbe:
  tcpSocket:
    port: 5432
  initialDelaySeconds: 30
  periodSeconds: 15

# Exec (run a command inside the container)
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - "pg_isready -U postgres"
  initialDelaySeconds: 30
  periodSeconds: 10
```

---

## Rolling Update vs Recreate Strategy

### Rolling Update (default — zero downtime)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%       # Or absolute number: 1
    maxUnavailable: 25% # Or absolute number: 0 for zero-downtime
```

### Recreate (for apps that cannot run multiple versions)

```yaml
strategy:
  type: Recreate
  # All old pods deleted before new pods start
  # Results in brief downtime — use for stateful apps, DB migrations
```

---

## Rollout Commands

```bash
# Check rollout status
oc rollout status deployment/my-app

# View rollout history (shows revision numbers)
oc rollout history deployment/my-app

# Rollback to previous version
oc rollout undo deployment/my-app

# Rollback to specific revision
oc rollout undo deployment/my-app --to-revision=2

# Trigger new rollout (useful after secret/configmap changes)
oc rollout restart deployment/my-app

# Scale a deployment
oc scale deployment/my-app --replicas=5
```

---

## Common Deployment Pitfalls

| Problem | Symptom | Root Cause | Fix |
|---------|---------|-----------|-----|
| Pod stuck in `Pending` | Never starts | SCC rejection or no node capacity | Check `oc describe pod`, add resources, fix SCC |
| `CrashLoopBackOff` | Restarts repeatedly | App crashes on start, often wrong UID | Check logs, check SCC, check startup probe |
| `ImagePullBackOff` | Can't pull image | Wrong image name, missing pull secret | Check image name, add imagePullSecret |
| `OOMKilled` | Container exits with 137 | Memory limit exceeded | Increase memory limit |
| Pods not receiving traffic | Service shows endpoints | readinessProbe failing | Fix probe path/port |
