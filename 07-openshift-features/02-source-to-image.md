# Source-to-Image (S2I) Builds

## What is S2I?

Source-to-Image (S2I) is an OpenShift build system that creates container images directly from source code **without writing a Dockerfile**. S2I uses a **builder image** that knows how to compile and package your language/framework.

**S2I Workflow:**
```
Your Source Code (Git repo)
        +
Builder Image (nodejs, python, java, etc.)
        ↓
   S2I Process (assemble script)
        ↓
Application Image (stored in OpenShift image registry)
        ↓
Running Pod
```

---

## Supported Builder Images

| Language/Platform | Builder Image | Sample |
|------------------|---------------|--------|
| Node.js | nodejs:18-ubi8, nodejs:20-ubi9 | Express, React |
| Python | python:3.11-ubi9 | Flask, Django |
| Java | openjdk-17-ubi8 | Spring Boot |
| .NET | dotnet-60 | ASP.NET Core |
| Ruby | ruby-30 | Rails |
| PHP | php-81 | Laravel |
| Go | golang | Standard Go apps |
| Nginx | nginx-122 | Static sites |
| Apache HTTPD | httpd-24 | Static sites |

---

## Quick Deploy with oc new-app

```bash
# Deploy Node.js app from Git (S2I auto-detects language)
oc new-app https://github.com/openshift/nodejs-ex.git \
  --name=my-node-app

# Specify builder image explicitly
oc new-app nodejs:20-ubi9~https://github.com/myorg/myapp.git \
  --name=my-node-app

# From a specific branch/tag
oc new-app nodejs:18-ubi8~https://github.com/myorg/myapp.git#main \
  --name=my-node-app

# With environment variables
oc new-app python:3.11-ubi9~https://github.com/myorg/flask-app.git \
  --name=my-flask-app \
  -e PORT=8080 \
  -e DEBUG=false

# Watch the build
oc logs -f bc/my-node-app
```

---

## BuildConfig YAML

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: my-node-app
  namespace: my-project
spec:
  output:
    to:
      kind: ImageStreamTag
      name: my-node-app:latest
  source:
    type: Git
    git:
      uri: https://github.com/myorg/myapp.git
      ref: main
    contextDir: /               # Build from a subdirectory
    # For private repos:
    # sourceSecret:
    #   name: git-secret
  strategy:
    type: Source                # Source strategy = S2I
    sourceStrategy:
      from:
        kind: ImageStreamTag
        name: nodejs:18-ubi8
        namespace: openshift    # Builder images are in openshift namespace
      env:
        - name: NODE_ENV
          value: production
        - name: NPM_MIRROR
          value: https://registry.npmjs.org
  triggers:
    - type: ConfigChange        # Rebuild when BuildConfig is updated
    - type: ImageChange         # Rebuild when builder image updates
      imageChange: {}
    - type: GitHub              # Rebuild on GitHub webhook
      github:
        secret: my-webhook-secret
    - type: Generic             # Generic webhook
      generic:
        secret: my-generic-secret
  successfulBuildsHistoryLimit: 5
  failedBuildsHistoryLimit: 3
```

---

## Build Triggers

### Webhook Trigger Setup

```bash
# Get GitHub webhook URL
oc describe bc/my-node-app | grep -A 2 "GitHub"
# URL format: https://api.<cluster>/apis/build.openshift.io/v1/namespaces/<ns>/buildconfigs/my-node-app/webhooks/<secret>/github

# Get the webhook secret
oc get bc my-node-app -o jsonpath='{.spec.triggers[?(@.type=="GitHub")].github.secret}'
```

### Manual Trigger

```bash
# Start a build manually
oc start-build my-node-app

# Start from a specific commit
oc start-build my-node-app --commit=abc1234

# Build from a local directory (binary build)
oc start-build my-node-app --from-dir=./src

# Build from a JAR/WAR file
oc start-build my-java-app --from-file=target/app.jar
```

---

## Monitoring Builds

```bash
# List all builds
oc get builds

# Watch a build in progress
oc logs -f build/my-node-app-1

# View build history
oc get builds -l buildconfig=my-node-app

# Describe a build
oc describe build my-node-app-2
```

---

## Build Secrets (Private Git Repositories)

```bash
# Create a secret for SSH key access
oc create secret generic git-ssh-key \
  --from-file=ssh-privatekey=$HOME/.ssh/id_rsa \
  --type=kubernetes.io/ssh-auth

# Create a secret for username/password
oc create secret generic git-credentials \
  --from-literal=username=myuser \
  --from-literal=password=mytoken \
  --type=kubernetes.io/basic-auth

# Link the secret to the builder service account
oc secrets link builder git-ssh-key

# Reference in BuildConfig
# spec.source.sourceSecret.name: git-credentials
```

---

## Docker Strategy (Custom Dockerfile)

```yaml
spec:
  strategy:
    type: Docker        # Use Dockerfile instead of S2I
    dockerStrategy:
      dockerfilePath: Dockerfile   # Path within the repo
      from:
        kind: DockerImage
        name: registry.access.redhat.com/ubi9/ubi:latest
      noCache: false
      forcePull: false
  source:
    type: Git
    git:
      uri: https://github.com/myorg/myapp.git
```
