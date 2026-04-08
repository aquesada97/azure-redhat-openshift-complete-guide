# ImageStreams

## What are ImageStreams?

An ImageStream is an OpenShift abstraction over container image references. Instead of hardcoding registry URLs in your Deployments, you reference an ImageStream tag. OpenShift monitors the ImageStream and can trigger deployments automatically when a new image is pushed.

**Benefits:**
- Decouple applications from specific registry URLs
- Trigger builds/deployments automatically when images change
- Manage image tags and history in one place
- Enable promotion workflows (dev → staging → prod)

---

## ImageStream YAML

```yaml
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: my-app
  namespace: my-project
spec:
  lookupPolicy:
    local: true    # Allow Deployments to reference this IS by short name
  tags:
    - name: latest
      from:
        kind: DockerImage
        name: nginx:latest
      importPolicy:
        scheduled: true          # Periodically re-import to check for updates
        importMode: Legacy
      referencePolicy:
        type: Local              # Rewrite references to use internal registry
```

---

## Importing External Images

```bash
# Import an image from a public registry
oc import-image nginx:latest \
  --from=docker.io/library/nginx:latest \
  --confirm

# Import and create the ImageStream in one step
oc import-image my-app:v1.2 \
  --from=myacr.azurecr.io/my-app:v1.2 \
  --confirm

# Schedule periodic import (check for updates every 15 min)
oc import-image nginx:latest \
  --from=nginx:latest \
  --scheduled=true \
  --confirm

# View the ImageStream
oc get imagestream nginx
oc describe imagestream nginx
```

---

## ImageStream Tags and Promotion

```bash
# Tag: copy a tag from one ImageStream to another (promotion workflow)
oc tag my-app:latest my-app:staging
oc tag my-app:staging my-app:production

# Tag from external registry
oc tag docker.io/library/nginx:1.25 my-nginx:1.25 --scheduled

# Tag from another namespace (copy image to this namespace)
oc tag other-project/my-app:latest my-project/my-app:latest

# Remove a tag
oc tag -d my-app:old-tag

# View all tags and their history
oc get imagestream my-app -o yaml
oc describe imagestream my-app
```

---

## Using ImageStreams in Deployments

### With Annotation Trigger

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-project
  annotations:
    image.openshift.io/triggers: |
      [
        {
          "from": {
            "kind": "ImageStreamTag",
            "name": "my-app:latest",
            "namespace": "my-project"
          },
          "fieldPath": "spec.template.spec.containers[?(@.name==\"my-app\")].image",
          "pause": "false"
        }
      ]
spec:
  template:
    spec:
      containers:
        - name: my-app
          image: image-registry.openshift-image-registry.svc:5000/my-project/my-app:latest
```

---

## ImageStreamImport

Trigger a one-time import/update:

```yaml
apiVersion: image.openshift.io/v1
kind: ImageStreamImport
metadata:
  name: my-app
  namespace: my-project
spec:
  import: true
  images:
    - from:
        kind: DockerImage
        name: myacr.azurecr.io/my-app:v2.0
      to:
        name: v2.0
```

---

## Internal Registry Integration

S2I builds automatically push to the internal registry and update ImageStreams:

```bash
# After oc new-app with S2I, the build pushes to:
# image-registry.openshift-image-registry.svc:5000/my-project/my-app:latest

# This is referenced by the ImageStreamTag:
oc get istag my-app:latest -o jsonpath='{.image.dockerImageReference}'

# Allow other namespaces to pull from this ImageStream
oc policy add-role-to-user system:image-puller \
  system:serviceaccount:other-project:default \
  -n my-project
```
