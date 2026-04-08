# Azure Red Hat OpenShift (ARO) Training

![ARO](https://img.shields.io/badge/Azure-ARO-blue?logo=microsoft-azure)
![OpenShift](https://img.shields.io/badge/Red%20Hat-OpenShift-red?logo=red-hat-open-shift)
![License](https://img.shields.io/badge/license-MIT-green)

Welcome to the **Azure Red Hat OpenShift (ARO) Training** repository. This is a hands-on, comprehensive guide for engineers and developers who want to learn ARO — the fully managed OpenShift service on Azure, jointly operated by Microsoft and Red Hat.

---

## Table of Contents

| Module | Description |
|--------|-------------|
| [00 - Prerequisites](./00-prerequisites/README.md) | Tools, accounts, Azure quotas, environment setup |
| [01 - Fundamentals](./01-fundamentals/README.md) | OpenShift vs K8s, ARO architecture, components |
| [02 - Getting Started](./02-getting-started/README.md) | Create cluster, oc CLI, connect to cluster |
| [03 - Workloads](./03-workloads/README.md) | Deployments, Services, Routes, StatefulSets |
| [04 - Networking](./04-networking/README.md) | OVN-K, Routes, Network Policies |
| [05 - Storage](./05-storage/README.md) | PVCs, Azure Disk, Azure Files |
| [06 - Security](./06-security/README.md) | SCCs, RBAC, Workload Identity, Key Vault CSI |
| [07 - OpenShift Features](./07-openshift-features/README.md) | Console, S2I, Operators, ImageStreams |
| [08 - Monitoring](./08-monitoring/README.md) | Prometheus stack, custom metrics, log forwarding |
| [09 - Scaling](./09-scaling/README.md) | HPA, VPA, Cluster Autoscaler (Machine API) |
| [10 - Advanced](./10-advanced/README.md) | Upgrades, multi-tenancy, disaster recovery |
| [Labs](./labs/README.md) | Hands-on lab exercises |

---

## Description

This training covers ARO from the ground up, starting with fundamental OpenShift concepts and progressing through real-world production scenarios.

### Key Focus Areas
- How ARO differs from AKS and self-managed OpenShift
- OpenShift-specific constructs: SCCs, Routes, DeploymentConfigs, ImageStreams, S2I
- Azure integrations: AAD OAuth, Key Vault CSI, Azure Monitor, Workload Identity
- Day-2 operations: upgrades, scaling, monitoring, backup/recovery

---

## Prerequisites Summary

Before starting, ensure you have:
- An Azure subscription with sufficient quota (40+ vCPUs in target region)
- A Red Hat account with a pull secret (cloud.redhat.com)
- Azure CLI (`az`) with `az aro` extension
- OpenShift CLI (`oc`)
- kubectl (optional, `oc` is a superset)

---

## ARO vs AKS vs Vanilla Kubernetes

| Feature | AKS | ARO | Vanilla K8s |
|---------|-----|-----|-------------|
| Managed control plane | Yes | Yes | No |
| Built-in monitoring | Optional | Yes (Prometheus) | No |
| Built-in image registry | No | Yes | No |
| Security Context Constraints | No | Yes | No (PSA) |
| OpenShift Routes | No | Yes | No |
| Source-to-Image (S2I) | No | Yes | No |
| Operator Lifecycle Manager | No | Yes | No |
| Web console (Developer UX) | Basic | Full | No |
| Red Hat support | No | Joint MS+RH | No |
| Azure AD integration | Native | via OAuth | Manual |
| Upgrade management | MS-managed | Joint MS+RH | Self |
