# 09 - Scaling

## Module Overview

ARO supports multiple layers of scaling: pod-level with HPA/VPA, node-level with Cluster Autoscaler (via Machine API), and resource governance with quotas and PodDisruptionBudgets.

---

## Learning Objectives

1. Configure Horizontal Pod Autoscaler (HPA) with CPU, memory, and custom metrics
2. Deploy Cluster Autoscaler with MachineAutoscaler
3. Set ResourceQuotas and LimitRanges for multi-tenant environments
4. Apply performance tuning: affinity, taints, topology spread

---

## Topics

| File | Topic |
|------|-------|
| [01-hpa.md](./01-hpa.md) | HPA v1/v2, VPA, custom and external metrics |
| [02-cluster-autoscaler.md](./02-cluster-autoscaler.md) | Machine API, MachineAutoscaler, ClusterAutoscaler |
| [03-resource-quotas.md](./03-resource-quotas.md) | ResourceQuota, LimitRange, PodDisruptionBudgets |
| [04-performance-tuning.md](./04-performance-tuning.md) | Affinity, taints, topology spread, Tuned Operator |

---

## Estimated Time

~75 minutes

---

## Next Module

[10 - Advanced](../10-advanced/README.md)
