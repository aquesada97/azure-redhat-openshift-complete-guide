# 06 - Security

## Module Overview

Security in ARO is multi-layered: Security Context Constraints (SCCs) control pod privileges, RBAC controls API access, Workload Identity enables passwordless Azure service access, and Key Vault CSI provides secret management.

---

## Learning Objectives

1. Understand and configure Security Context Constraints (SCCs)
2. Implement fine-grained RBAC with Roles and RoleBindings
3. Configure Workload Identity for Azure service access
4. Manage secrets securely with Key Vault CSI Driver
5. Implement network-level security controls

---

## Topics

| File | Topic |
|------|-------|
| [01-scc.md](./01-scc.md) | SCCs, built-in policies, custom SCCs |
| [02-rbac.md](./02-rbac.md) | Roles, ClusterRoles, bindings, oc adm policy |
| [03-workload-identity.md](./03-workload-identity.md) | OIDC federation, Azure Workload Identity |
| [04-secrets-management.md](./04-secrets-management.md) | Key Vault CSI, ExternalSecrets, etcd encryption |
| [05-network-security.md](./05-network-security.md) | NSGs, Egress firewall, private cluster security |

---

## Estimated Time

~90 minutes

---

## Next Module

[07 - OpenShift Features](../07-openshift-features/README.md)
