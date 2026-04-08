# Built-In Monitoring Stack

## Overview

ARO includes a fully pre-configured monitoring stack based on the **Prometheus Operator**. It runs in the `openshift-monitoring` namespace and is managed by the cluster-monitoring-operator Cluster Operator.

---

## Monitoring Stack Components

| Component | Namespace | Purpose |
|-----------|-----------|---------|
| prometheus-k8s | openshift-monitoring | Scrapes platform metrics (nodes, pods, K8s objects) |
| prometheus-user-workload | openshift-user-workload-monitoring | Scrapes user application metrics (when enabled) |
| alertmanager-main | openshift-monitoring | Handles alert routing and notifications |
| grafana | openshift-monitoring | Dashboards (read-only in OCP 4.11+) |
| thanos-querier | openshift-monitoring | Unified query layer across Prometheus instances |
| thanos-ruler | openshift-user-workload-monitoring | Alerting rules for user workloads |
| node-exporter | openshift-monitoring | Node-level metrics (CPU, mem, disk, network) |
| kube-state-metrics | openshift-monitoring | K8s object state metrics |
| prometheus-adapter | openshift-monitoring | Exposes Prometheus metrics for HPA |

```bash
# Check all monitoring pods
oc get pods -n openshift-monitoring
oc get pods -n openshift-user-workload-monitoring
```

---

## Enabling User Workload Monitoring

By default, only platform metrics are collected. Enable user workload monitoring to scrape your application metrics:

```bash
# Create or edit the monitoring ConfigMap
cat <<EOF | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
    prometheusK8s:
      retention: 15d
      volumeClaimTemplate:
        spec:
          storageClassName: managed-premium
          resources:
            requests:
              storage: 50Gi
    alertmanagerMain:
      volumeClaimTemplate:
        spec:
          storageClassName: managed-premium
          resources:
            requests:
              storage: 10Gi
EOF

# Wait for user workload monitoring pods to start
oc get pods -n openshift-user-workload-monitoring -w
```

---

## Accessing Prometheus and Grafana

```bash
# Get Prometheus UI URL
oc get route prometheus-k8s -n openshift-monitoring -o jsonpath='{.spec.host}'

# Get Thanos Querier URL (unified query)
oc get route thanos-querier -n openshift-monitoring -o jsonpath='{.spec.host}'

# Get Grafana URL
oc get route grafana -n openshift-monitoring -o jsonpath='{.spec.host}'
```

From the OpenShift Console: **Observe → Metrics** (uses Thanos Querier)

---

## AlertManager Configuration

Configure notifications for alerts:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-main
  namespace: openshift-monitoring
stringData:
  alertmanager.yaml: |
    global:
      smtp_smarthost: 'smtp.example.com:587'
      smtp_from: 'alertmanager@example.com'
    route:
      receiver: default
      group_by: ['alertname', 'namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      routes:
        - match:
            severity: critical
          receiver: pagerduty
        - match:
            severity: warning
          receiver: slack
    receivers:
      - name: default
        email_configs:
          - to: ops-team@example.com
      - name: slack
        slack_configs:
          - api_url: https://hooks.slack.com/services/xxx/yyy/zzz
            channel: '#alerts'
            title: '{{ .GroupLabels.alertname }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
      - name: pagerduty
        pagerduty_configs:
          - service_key: <pagerduty-service-key>
```

```bash
# Apply the AlertManager config
oc apply -f alertmanager-config.yaml -n openshift-monitoring
```

---

## Default Alerts

Key platform alerts to know:

| Alert | Severity | Meaning |
|-------|----------|---------|
| `KubePodCrashLooping` | Warning | Pod restarting repeatedly |
| `KubePodNotReady` | Warning | Pod not in Ready state >15 min |
| `KubeNodeNotReady` | Warning | Node not ready |
| `KubeNodeUnreachable` | Critical | Node unreachable |
| `AlertmanagerDown` | Critical | AlertManager unavailable |
| `PrometheusDown` | Critical | Prometheus unavailable |
| `etcdHighCommitDurations` | Warning | etcd slow writes |
| `KubePersistentVolumeFillingUp` | Warning | PVC >85% full |
| `ClusterOperatorDegraded` | Warning | Cluster Operator degraded |

---

## Azure Monitor Integration

```bash
# Enable Container Insights for ARO
az aro update \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --workspace-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.OperationalInsights/workspaces/<workspace>
```

Container Insights provides:
- Node and pod CPU/memory metrics in Azure Monitor
- Log Analytics integration
- Azure dashboards and workbooks
- Integration with Azure Alerts
