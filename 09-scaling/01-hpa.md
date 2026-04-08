# Horizontal Pod Autoscaler (HPA)

## Overview

The Horizontal Pod Autoscaler (HPA) automatically scales the number of pod replicas based on observed metrics: CPU utilization, memory utilization, or custom metrics from Prometheus.

---

## HPA v2 (Multi-Metric)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
  namespace: my-project
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # CPU metric
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # Scale up when avg CPU > 70%
    # Memory metric
    - type: Resource
      resource:
        name: memory
        target:
          type: AverageValue
          averageValue: 512Mi       # Scale up when avg memory > 512Mi
    # Custom metric from Prometheus
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"       # 100 requests/sec per pod
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30   # Wait 30s before scaling up
      policies:
        - type: Percent
          value: 100                   # Double replicas at most
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
        - type: Pods
          value: 2                     # Remove at most 2 pods
          periodSeconds: 60
```

---

## Quick HPA Creation

```bash
# Create HPA from CLI
oc autoscale deployment/my-app \
  --min=2 \
  --max=10 \
  --cpu-percent=70

# View HPA status
oc get hpa
oc describe hpa my-app-hpa

# Watch HPA in real-time
oc get hpa my-app-hpa -w
```

---

## External Metrics HPA (Azure Service Bus)

Scale based on Azure Service Bus queue depth:

```yaml
# Requires: prometheus-adapter or keda operator
- type: External
  external:
    metric:
      name: azure_servicebus_active_messages
      selector:
        matchLabels:
          queue: my-queue
    target:
      type: Value
      value: "10"     # Scale up when queue depth > 10
```

> For Azure-specific metrics, consider **KEDA** (Kubernetes Event-Driven Autoscaling) which has native Azure Service Bus, Storage Queue, and Event Hub scalers.

---

## Vertical Pod Autoscaler (VPA)

VPA automatically adjusts CPU/memory **requests** for containers based on historical usage. Unlike HPA, VPA changes resource requests rather than replica count.

### VPA Modes

| Mode | Behavior |
|------|---------|
| `Off` | Recommendations only, no changes |
| `Initial` | Set resources at pod creation, no updates |
| `Recreate` | Evict and recreate pods with new resources |
| `Auto` | Like Recreate but may in-place update (K8s 1.27+) |

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
  namespace: my-project
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"          # Start with Off to review recommendations
  resourcePolicy:
    containerPolicies:
      - containerName: my-app
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4
          memory: 8Gi
        controlledResources:
          - cpu
          - memory
```

```bash
# Install VPA Operator from OperatorHub
# View VPA recommendations
oc get vpa my-app-vpa -o jsonpath='{.status.recommendation}' | python3 -m json.tool
```

---

## HPA vs VPA

| Aspect | HPA | VPA |
|--------|-----|-----|
| What it scales | Replicas (horizontal) | Resource requests (vertical) |
| Use case | Stateless apps with variable load | Apps with unpredictable resource needs |
| Metrics | CPU, memory, custom, external | CPU, memory (historical) |
| Downtime | No (rolling) | Yes in Recreate mode |
| Can be combined | Yes (use different metrics) | Caution: HPA CPU + VPA CPU = conflict |

> **Don't combine HPA (CPU-based) with VPA** on the same deployment — they fight each other. If using both, have HPA scale on custom metrics and VPA handle resources.

---

## Troubleshooting HPA

```bash
# Check if metrics are available
oc describe hpa my-app-hpa
# Look for: "unable to fetch metrics" or "metric not available"

# Check metrics API
oc get --raw /apis/metrics.k8s.io/v1beta1/pods | python3 -m json.tool | head -50

# Check if custom metrics are available (via prometheus-adapter)
oc get --raw /apis/custom.metrics.k8s.io/v1beta1 | python3 -m json.tool

# HPA not scaling down (stuck at high replica count)
# Check stabilizationWindowSeconds — default is 5 min for scale-down
# Check PodDisruptionBudgets — may prevent pod eviction
```
