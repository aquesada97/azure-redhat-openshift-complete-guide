# oc CLI Basics

## oc vs kubectl

`oc` is a superset of `kubectl`. Every `kubectl` command works with `oc`. Additionally, `oc` provides:
- OpenShift-specific resource management (Routes, BuildConfigs, ImageStreams)
- `oc new-app` — deploy apps from images, source code, or templates
- `oc new-project` — create projects with RBAC
- `oc rsh` — remote shell into a pod
- `oc adm` — cluster administration commands
- `oc get all` — shows OpenShift resources in addition to K8s resources

---

## Authentication

```bash
# Log in with username and password
oc login <api-url> -u kubeadmin -p <password>

# Log in with token (from web console: top right → Copy Login Command)
oc login --token=sha256~<token> --server=https://api.cluster.example.com:6443

# Log out
oc logout

# Who am I?
oc whoami

# What server am I connected to?
oc whoami --show-server

# What is my current token?
oc whoami --show-token
```

---

## Projects

```bash
# Create a new project
oc new-project my-app \
  --display-name="My Application" \
  --description="Production deployment of my-app"

# Switch to a project
oc project my-app

# List all projects you have access to
oc projects

# Show current project resources (deployments, services, routes)
oc status
```

---

## Resource Management

### Viewing Resources

```bash
# List resources (standard K8s resources)
oc get pods
oc get deployments
oc get services
oc get pvc
oc get configmaps
oc get secrets

# List OpenShift-specific resources
oc get routes
oc get buildconfigs
oc get builds
oc get imagestreams

# Get ALL resources including OpenShift types (oc-specific command)
oc get all

# Get resources in all namespaces
oc get pods --all-namespaces
oc get pods -A   # Shorthand

# Get resources with labels
oc get pods -l app=my-app
oc get pods -l app=my-app,tier=backend

# Get resource details in YAML/JSON
oc get pod my-pod -o yaml
oc get deployment my-app -o json

# Watch for changes
oc get pods -w
```

### Describing Resources

```bash
# Full description with events (essential for debugging)
oc describe pod <pod-name>
oc describe deployment <name>
oc describe node <node-name>
oc describe pvc <pvc-name>

# Check pod resource usage
oc adm top pods
oc adm top pods --containers   # Per-container breakdown
oc adm top nodes
```

### Pod Logs

```bash
# View pod logs
oc logs <pod-name>

# Follow (stream) logs
oc logs <pod-name> -f

# Last N lines
oc logs <pod-name> --tail=100

# Specific container in a multi-container pod
oc logs <pod-name> -c <container-name>

# Previous container (if pod restarted)
oc logs <pod-name> --previous
```

### Pod Interaction

```bash
# Remote shell (OpenShift shortcut — tries bash, then sh)
oc rsh <pod-name>

# Exec with specific command
oc exec -it <pod-name> -- /bin/bash
oc exec <pod-name> -- env | grep DATABASE

# Port forwarding (local:pod)
oc port-forward pod/<pod-name> 8080:8080
oc port-forward svc/<service-name> 8080:80

# Copy files to/from pod
oc cp <pod-name>:/path/to/file ./local-file
oc cp ./local-file <pod-name>:/path/to/file
```

### Applying and Deleting Resources

```bash
# Apply a manifest (create or update)
oc apply -f manifest.yaml
oc apply -f ./directory/    # Apply all files in directory
oc apply -k ./kustomize/    # Apply Kustomize overlay

# Delete resources
oc delete -f manifest.yaml
oc delete pod <name>
oc delete pod <name> --force --grace-period=0   # Force delete
oc delete deployment <name>

# Edit a resource in-place
oc edit deployment <name>

# Patch a resource
oc patch deployment <name> \
  --type merge \
  --patch '{"spec":{"replicas":3}}'
```

---

## Application Deployment

```bash
# Deploy from a container image
oc new-app nginx:latest --name=my-nginx

# Deploy from source code (S2I auto-detects language)
oc new-app https://github.com/myorg/myapp.git

# Deploy with a specific builder image
oc new-app nodejs:18-ubi8~https://github.com/myorg/node-app.git

# Deploy from a template
oc new-app --template=postgresql-persistent \
  -p POSTGRESQL_USER=myuser \
  -p POSTGRESQL_PASSWORD=mypass \
  -p POSTGRESQL_DATABASE=mydb

# Expose a service as a route
oc expose svc/my-nginx

# Expose with specific hostname
oc expose svc/my-nginx --hostname=myapp.apps.cluster.example.com

# Expose with TLS (edge termination)
oc create route edge my-route \
  --service=my-nginx \
  --insecure-policy=Redirect
```

---

## Rollout Management

```bash
# Check rollout status
oc rollout status deployment/<name>

# View rollout history
oc rollout history deployment/<name>

# Rollback to previous version
oc rollout undo deployment/<name>

# Rollback to specific revision
oc rollout undo deployment/<name> --to-revision=2

# Pause a rollout (stop auto-updates mid-deploy)
oc rollout pause deployment/<name>

# Resume a paused rollout
oc rollout resume deployment/<name>

# Trigger a restart (re-pulls config/secrets)
oc rollout restart deployment/<name>
```

---

## Cluster Administration

```bash
# View nodes
oc get nodes
oc get nodes -o wide    # Shows IPs, OS, kernel version

# Node resource usage
oc adm top nodes

# Access node filesystem (creates debug pod)
oc debug node/<node-name>

# Cordon a node (prevent new pods from scheduling)
oc adm cordon <node-name>

# Drain a node (evict all pods, for maintenance)
oc adm drain <node-name> \
  --ignore-daemonsets \
  --delete-emptydir-data

# Uncordon a node
oc adm uncordon <node-name>

# Collect diagnostic data (requires cluster-admin)
oc adm must-gather
oc adm must-gather --image=registry.redhat.io/openshift4/ose-must-gather

# Check cluster operators
oc get clusteroperators
oc get co   # Shorthand

# Check cluster version
oc get clusterversion
```

---

## Events and Troubleshooting

```bash
# View recent events (sorted by time)
oc get events --sort-by='.lastTimestamp'

# View only warning events
oc get events --field-selector type=Warning

# View events for a specific namespace
oc get events -n my-namespace --sort-by='.lastTimestamp'

# Search events for a specific object
oc get events --field-selector involvedObject.name=<pod-name>
```

---

## Useful Shortcuts Reference

| Task | Command |
|------|---------|
| Get pods in current project | `oc get pods` |
| Get all resources | `oc get all` |
| Stream pod logs | `oc logs -f <pod>` |
| Shell into pod | `oc rsh <pod>` |
| Port forward | `oc port-forward <pod> 8080:8080` |
| Apply YAML | `oc apply -f file.yaml` |
| Describe resource | `oc describe <type> <name>` |
| Restart deployment | `oc rollout restart deployment/<name>` |
| Scale deployment | `oc scale deployment/<name> --replicas=3` |
| Watch pods | `oc get pods -w` |
| Get node SSH access | `oc debug node/<node>` |
| Collect diagnostics | `oc adm must-gather` |
