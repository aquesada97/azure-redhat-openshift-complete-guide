# ARO Training Labs

## Overview

These hands-on labs reinforce the concepts covered in the training modules. Each lab has a real cluster exercise with all required YAML manifests.

---

## Labs

| Lab | Title | Estimated Time | Prerequisites |
|-----|-------|----------------|---------------|
| [Lab 01](./lab-01-deploy-first-app/README.md) | Deploy Your First App | 30 min | Modules 01-02 |
| [Lab 02](./lab-02-routes-and-ingress/README.md) | Routes and Ingress | 45 min | Lab 01, Module 03 |
| [Lab 03](./lab-03-storage/README.md) | Persistent Storage | 45 min | Lab 01, Module 05 |
| [Lab 04](./lab-04-scaling/README.md) | Scaling and Autoscaling | 60 min | Lab 01, Modules 08-09 |
| [Lab 05](./lab-05-scc-and-rbac/README.md) | SCC and RBAC | 45 min | Lab 01, Module 06 |
| [Lab 06](./lab-06-s2i-build/README.md) | S2I Build and Deploy | 45 min | Lab 01, Module 07 |

---

## Lab Environment Requirements

- ARO cluster (4.x) up and running
- `oc` CLI logged in as cluster-admin
- `kubectl` (optional)
- `curl` for testing routes
- Internet access for pulling container images

---

## How to Use These Labs

1. Follow the module content before attempting the lab
2. Read the full lab README before starting
3. Apply manifests using `oc apply -f manifests/`
4. Complete each step and verify before proceeding
5. Clean up when done to preserve cluster resources

---

## Cleanup After Labs

```bash
# Delete a specific lab namespace
oc delete project lab-01
oc delete project lab-02
# etc.

# Or delete all lab namespaces at once
for i in 01 02 03 04 05 06; do
  oc delete project lab-$i --ignore-not-found
done
```
