# DNS in OpenShift

## CoreDNS in OpenShift

OpenShift uses **CoreDNS** as its cluster DNS resolver, running in the `openshift-dns` namespace. Every pod automatically has its DNS configured to use CoreDNS.

```bash
# Check CoreDNS pods
oc get pods -n openshift-dns

# View DNS operator configuration
oc get dns.operator/default -o yaml

# View CoreDNS ConfigMap
oc get configmap dns-default -n openshift-dns -o yaml
```

---

## DNS Resolution Order

When a pod resolves a hostname, CoreDNS processes in this order:

1. `<name>.<namespace>.svc.cluster.local` — service in specific namespace
2. `<name>.svc.cluster.local` — service in search path
3. `<name>.cluster.local`
4. External DNS (forwarded to upstream resolvers)

---

## Service DNS Patterns

```bash
# Full FQDN (always works from any namespace)
my-service.my-namespace.svc.cluster.local

# Short form (works from same namespace)
my-service

# Cross-namespace (short form)
my-service.other-namespace

# Headless service — returns all pod IPs
my-db-headless.my-namespace.svc.cluster.local

# Specific StatefulSet pod via headless service
my-db-0.my-db-headless.my-namespace.svc.cluster.local
my-db-1.my-db-headless.my-namespace.svc.cluster.local
```

---

## Pod DNS Configuration

View a pod's DNS configuration:

```bash
oc exec my-pod -- cat /etc/resolv.conf
# Output:
# search my-namespace.svc.cluster.local svc.cluster.local cluster.local
# nameserver 172.30.0.10    (CoreDNS ClusterIP)
# options ndots:5
```

The `ndots:5` setting means any hostname with fewer than 5 dots is tried with search suffixes first.

---

## Custom DNS Entries

Add custom DNS records to CoreDNS via the DNS Operator:

```yaml
# Forward all *.internal.example.com queries to a custom resolver
apiVersion: operator.openshift.io/v1
kind: DNS
metadata:
  name: default
spec:
  servers:
    - name: internal-dns
      zones:
        - internal.example.com
      forwardPlugin:
        policy: Random
        upstreams:
          - 10.0.0.4    # Your custom DNS server IP
          - 10.0.0.5
```

```bash
# Apply DNS operator config
oc apply -f dns-custom.yaml

# Watch DNS operator reconcile
oc get clusteroperator dns
```

---

## External DNS with Azure DNS

The ExternalDNS operator automatically creates DNS records in Azure DNS when Routes or LoadBalancer services are created.

```bash
# Install ExternalDNS operator from OperatorHub
# Then create a source for it:

apiVersion: externaldns.olm.openshift.io/v1beta1
kind: ExternalDNS
metadata:
  name: sample-azure
spec:
  provider:
    type: Azure
    azure:
      configFile:
        name: azure-config-file
  source:
    type: OpenShiftRoute
    openshiftRouteOptions:
      routerName: default
  domains:
    - matchType: Exact
      name: myapp.example.com
```

---

## Troubleshooting DNS

```bash
# Test DNS from inside a pod
oc exec -it my-pod -- nslookup my-service.my-namespace.svc.cluster.local
oc exec -it my-pod -- dig +short my-service.my-namespace.svc.cluster.local

# Test external DNS from inside a pod
oc exec -it my-pod -- nslookup google.com

# Check if CoreDNS is healthy
oc get pods -n openshift-dns
oc logs -n openshift-dns -l dns.operator.openshift.io/daemonset-dns=default --tail=50

# If pods can't resolve: check CoreDNS running, check NetworkPolicy allows port 53
```
