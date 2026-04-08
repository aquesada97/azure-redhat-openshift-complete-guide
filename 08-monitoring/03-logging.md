# Logging in ARO

## Logging Options

| Option | Description | Best For |
|--------|-------------|---------|
| `oc logs` | Direct pod stdout/stderr | Development, quick debugging |
| Cluster Logging Operator (Loki) | Full log aggregation with query UI | Production, log retention |
| Azure Monitor Container Insights | Azure-native log management | Azure-integrated environments |
| External forwarding | Send to Splunk, Elasticsearch | Centralized enterprise logging |

---

## Cluster Logging Operator (Loki Stack)

### Install from OperatorHub

```bash
# Install Loki Operator
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: loki-operator
  namespace: openshift-operators-redhat
spec:
  channel: stable
  name: loki-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF

# Install Cluster Logging Operator
cat <<EOF | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: cluster-logging
  namespace: openshift-logging
spec:
  channel: stable
  name: cluster-logging
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

### LokiStack CR

```yaml
apiVersion: loki.grafana.com/v1
kind: LokiStack
metadata:
  name: logging-loki
  namespace: openshift-logging
spec:
  size: 1x.small          # 1x.extra-small, 1x.small, 1x.medium
  storage:
    schemas:
      - version: v12
        effectiveDate: "2022-06-01"
    secret:
      name: logging-loki-azure   # Azure Blob Storage secret
      type: azure
  storageClassName: managed-csi
  tenancy:
    mode: OpenshiftLogging
```

### ClusterLogging CR

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogging
metadata:
  name: instance
  namespace: openshift-logging
spec:
  collection:
    type: vector          # Log collector: vector (recommended) or fluentd
  logStore:
    type: lokistack
    lokistack:
      name: logging-loki
  visualization:
    type: ocp-console     # Show logs in OpenShift console
```

---

## Log Types

| Log Type | Source | Content |
|----------|--------|---------|
| `application` | Pod stdout/stderr | Your app logs |
| `infrastructure` | System pods, nodes | OpenShift platform logs |
| `audit` | kube-apiserver, oauth | API audit trail |

---

## Forwarding Logs to Azure Monitor

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
    - name: azure-monitor
      type: azureMonitor
      azureMonitor:
        customerId: <log-analytics-workspace-id>
        logType: openshift_logs
        secret:
          name: azure-monitor-secret   # Contains sharedKey
  pipelines:
    - name: app-logs-to-azure
      inputRefs:
        - application
      outputRefs:
        - azure-monitor
    - name: infra-logs-to-azure
      inputRefs:
        - infrastructure
      outputRefs:
        - azure-monitor
```

```bash
# Create Azure Monitor shared key secret
oc create secret generic azure-monitor-secret \
  --from-literal=sharedKey=<workspace-primary-key> \
  -n openshift-logging
```

---

## Forwarding to External Elasticsearch

```yaml
apiVersion: logging.openshift.io/v1
kind: ClusterLogForwarder
metadata:
  name: instance
  namespace: openshift-logging
spec:
  outputs:
    - name: elasticsearch-prod
      type: elasticsearch
      url: https://elasticsearch.example.com:9200
      secret:
        name: elasticsearch-secret
      elasticsearch:
        version: 8
  pipelines:
    - name: all-to-elasticsearch
      inputRefs:
        - application
        - infrastructure
        - audit
      outputRefs:
        - elasticsearch-prod
```

---

## Querying Logs in the Console

With Cluster Logging + Loki:
1. Go to **Observe → Logs** (admin perspective)
2. Or **Observe → Logs** in developer perspective (project-scoped)

LogQL query examples:
```logql
# All logs from a namespace
{kubernetes_namespace_name="my-project"}

# Error logs from a specific app
{kubernetes_namespace_name="my-project", kubernetes_labels_app="my-app"} |= "ERROR"

# JSON log parsing
{kubernetes_namespace_name="my-project"} | json | level="error"

# Log rate per minute
rate({kubernetes_namespace_name="my-project"}[1m])
```

---

## Node-Level Logs

```bash
# View systemd journal from a node
oc adm node-logs <node-name>

# Filter by service
oc adm node-logs <node-name> --unit=kubelet
oc adm node-logs <node-name> --unit=crio

# Follow a node log
oc adm node-logs <node-name> --unit=kubelet -f

# View audit logs
oc adm node-logs --role=master --path=kube-apiserver/audit.log | tail -100
```
