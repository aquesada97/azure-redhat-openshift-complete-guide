# 04 - Networking

## Module Overview

ARO uses OVN-Kubernetes (OVN-K) as its Container Network Interface (CNI). This module covers the networking model, Network Policies, DNS, and ingress/load balancing.

---

## Learning Objectives

1. Understand OVN-Kubernetes architecture and pod networking
2. Write and apply Network Policies for traffic isolation
3. Configure CoreDNS for custom DNS resolution
4. Manage IngressControllers and custom TLS certificates

---

## Topics

| File | Topic |
|------|-------|
| [01-ovn-kubernetes.md](./01-ovn-kubernetes.md) | OVN-K CNI, pod networking, Egress IP |
| [02-network-policies.md](./02-network-policies.md) | Network Policies, default deny, namespace isolation |
| [03-dns.md](./03-dns.md) | CoreDNS, custom resolvers, ExternalDNS |
| [04-ingress-and-load-balancing.md](./04-ingress-and-load-balancing.md) | IngressController, Azure LB, custom TLS |

---

## Estimated Time

~75 minutes

---

## Next Module

[05 - Storage](../05-storage/README.md)
