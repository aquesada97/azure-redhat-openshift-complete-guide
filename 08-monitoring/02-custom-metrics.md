# Custom Metrics and Alerting

## Exposing Application Metrics

Your application must expose metrics in **Prometheus format** on an HTTP endpoint (typically `/metrics`).

```python
# Python Flask with prometheus_client
from flask import Flask
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)
requests_total = Counter('http_requests_total', 'Total HTTP requests', ['method', 'status'])
request_duration = Histogram('request_duration_seconds', 'Request duration')

@app.route('/metrics')
def metrics():
    return generate_latest(), 200, {'Content-Type': CONTENT_TYPE_LATEST}
```

---

## ServiceMonitor

A `ServiceMonitor` tells Prometheus which services to scrape:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: my-project
  labels:
    app: my-app
spec:
  selector:
    matchLabels:
      app: my-app         # Select services with this label
  endpoints:
    - port: metrics        # Named port from the Service
      path: /metrics
      interval: 30s        # Scrape every 30 seconds
      scrapeTimeout: 10s
      scheme: http
      # For TLS:
      # scheme: https
      # tlsConfig:
      #   caFile: /etc/prometheus/configmaps/serving-certs-ca-bundle/service-ca.crt
      #   serverName: my-app.my-project.svc
```

### Service with Metrics Port

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-project
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
    - name: http
      port: 8080
      targetPort: 8080
    - name: metrics      # ServiceMonitor references this named port
      port: 9090
      targetPort: 9090
```

---

## PodMonitor

Scrape pods directly (without a Service):

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: my-app-pod-monitor
  namespace: my-project
spec:
  selector:
    matchLabels:
      app: my-app
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

---

## PrometheusRule for Custom Alerts

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: my-project
  labels:
    openshift.io/prometheus-rule-evaluation-scope: leaf-prometheus
spec:
  groups:
    - name: my-app.rules
      interval: 1m
      rules:
        # Recording rule: pre-compute expensive queries
        - record: job:http_requests_total:rate5m
          expr: rate(http_requests_total[5m])

        # Alert: high error rate
        - alert: HighErrorRate
          expr: |
            rate(http_errors_total[5m])
            /
            rate(http_requests_total[5m]) > 0.05
          for: 5m
          labels:
            severity: warning
            namespace: "{{ $labels.namespace }}"
          annotations:
            summary: "High error rate on {{ $labels.job }}"
            description: "Error rate is {{ printf \"%.2f\" $value }} (>5%)"
            runbook_url: https://wiki.example.com/runbooks/high-error-rate

        # Alert: high latency p99
        - alert: HighP99Latency
          expr: |
            histogram_quantile(0.99,
              rate(request_duration_seconds_bucket[5m])
            ) > 2.0
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "P99 latency above 2 seconds"
            description: "P99 latency: {{ printf \"%.3f\" $value }}s"
```

---

## PromQL Query Examples

Access at: **Observe → Metrics** in the OpenShift console, or via Thanos Querier.

```promql
# HTTP request rate (last 5 min)
rate(http_requests_total[5m])

# Error rate percentage
100 * rate(http_errors_total[5m]) / rate(http_requests_total[5m])

# P99 request latency
histogram_quantile(0.99, rate(request_duration_seconds_bucket[5m]))

# Pod memory usage
container_memory_working_set_bytes{namespace="my-project", container="my-app"}

# Pod CPU throttling percentage
100 * rate(container_cpu_cfs_throttled_seconds_total[5m])
/ rate(container_cpu_cfs_periods_total[5m])

# PVC utilization
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100

# Pods in each phase
kube_pod_status_phase{namespace="my-project"}

# Restarting pods
rate(kube_pod_container_status_restarts_total[1h]) * 3600

# Available replicas vs desired
kube_deployment_status_replicas_available / kube_deployment_spec_replicas
```
