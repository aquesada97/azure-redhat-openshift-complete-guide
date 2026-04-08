# Network Security in ARO

## Defense in Depth

ARO network security uses multiple layers:

```
Internet
    ↓
Azure Front Door / WAF (optional)
    ↓
Azure Load Balancer
    ↓
Network Security Groups (NSGs)
    ↓
OVN-Kubernetes Network Policies
    ↓
OpenShift Router (HAProxy) / Service
    ↓
Pod (SCC enforces pod security)
```

---

## Azure Network Security Groups (NSGs)

NSGs are attached to subnets and control traffic at the Azure network layer:

```bash
# View NSGs attached to subnets
az network vnet subnet show \
  --resource-group $RESOURCEGROUP \
  --vnet-name $VNET_NAME \
  --name $WORKER_SUBNET \
  --query networkSecurityGroup.id -o tsv

# List NSG rules
az network nsg rule list \
  --resource-group $RESOURCEGROUP \
  --nsg-name worker-nsg \
  --output table
```

> **Warning:** ARO manages some NSG rules automatically. Do not delete rules with `aro_` prefix.

---

## Private Cluster Security

Private ARO clusters (`--apiserver-visibility Private`, `--ingress-visibility Private`) offer stronger isolation:

```
On-Prem Network
      |
   VPN/ExpressRoute
      |
Azure VNet
      |
ARO Private API (no public IP)
ARO Private Ingress (no public IP)
```

Benefits:
- API server not exposed to internet
- Application traffic stays within Azure network
- Easier to meet compliance requirements (PCI-DSS, HIPAA)

```bash
# Create a private ARO cluster
az aro create \
  --resource-group $RESOURCEGROUP \
  --name $CLUSTER \
  --vnet $VNET_NAME \
  --master-subnet $MASTER_SUBNET \
  --worker-subnet $WORKER_SUBNET \
  --apiserver-visibility Private \
  --ingress-visibility Private \
  --pull-secret @$PULL_SECRET_FILE
```

---

## OVN Egress Firewall

Control outbound traffic from namespaces using EgressFirewall objects:

```yaml
apiVersion: k8s.ovn.org/v1
kind: EgressFirewall
metadata:
  name: restrict-egress
  namespace: my-project
spec:
  egress:
    - type: Allow
      to:
        cidrSelector: 10.0.0.0/8       # Allow internal Azure traffic
    - type: Allow
      to:
        cidrSelector: 0.0.0.0/0
      ports:
        - protocol: UDP
          port: 53                       # Allow DNS
    - type: Allow
      to:
        cidrSelector: 0.0.0.0/0
      ports:
        - protocol: TCP
          port: 443                      # Allow HTTPS to any
    - type: Deny
      to:
        cidrSelector: 0.0.0.0/0         # Block everything else
```

---

## Image Security

### Restrict Allowed Registries

```yaml
apiVersion: config.openshift.io/v1
kind: Image
metadata:
  name: cluster
spec:
  allowedRegistriesForImport:
    - domainName: registry.access.redhat.com
      insecure: false
    - domainName: registry.redhat.io
      insecure: false
    - domainName: myacr.azurecr.io
      insecure: false
  registrySources:
    allowedRegistries:
      - registry.access.redhat.com
      - registry.redhat.io
      - quay.io
      - myacr.azurecr.io
```

---

## Certificate Management with cert-manager

```bash
# Install cert-manager from OperatorHub
# Then create a ClusterIssuer for Let Encrypt:
```

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    email: admin@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - dns01:
          azureDNS:
            subscriptionID: <subscription-id>
            resourceGroupName: rg-dns
            hostedZoneName: example.com
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-apps-cert
  namespace: openshift-ingress
spec:
  secretName: wildcard-apps-cert
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - "*.apps.cluster.example.com"
```

```bash
# After cert is issued, configure router to use it
oc patch ingresscontroller default \
  -n openshift-ingress-operator \
  --type merge \
  --patch '{"spec":{"defaultCertificate":{"name":"wildcard-apps-cert"}}}'
```
