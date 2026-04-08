# 08 - Monitoring

## Module Overview

ARO includes a fully pre-configured monitoring stack based on Prometheus, Alertmanager, Thanos, and Grafana. This module covers using the built-in monitoring stack, enabling user workload monitoring, writing custom metrics, and log management.

---

## Learning Objectives

1. Understand the ARO built-in monitoring stack components
2. Enable and use user workload monitoring
3. Create ServiceMonitors for custom application metrics
4. Write PrometheusRule alerts
5. Configure log forwarding to Azure Monitor

---

## Topics

| File | Topic |
|------|-------|
| [01-built-in-monitoring.md](./01-built-in-monitoring.md) | Prometheus stack, Grafana, AlertManager |
| [02-custom-metrics.md](./02-custom-metrics.md) | ServiceMonitor, PodMonitor, PrometheusRule, PromQL |
| [03-logging.md](./03-logging.md) | Cluster Logging, Loki, log forwarding to Azure Monitor |

---

## Estimated Time

~75 minutes

---

## Next Module

[09 - Scaling](../09-scaling/README.md)
