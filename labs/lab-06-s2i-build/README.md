# Lab 06: S2I Build and Deploy

## Overview

Build a Node.js application from source code using Source-to-Image (S2I), configure a GitHub webhook for automatic builds, use ImageStreams, and demonstrate automatic deployment on new builds.

**Estimated Time:** 45 minutes  
**Prerequisites:** Lab 01, Module 07 (OpenShift Features)

---

## Objectives

- Create an ImageStream for the application
- Create a BuildConfig using S2I
- Trigger a build from source code
- Watch the build process
- Configure automatic deployment on new image
- Demonstrate rollback with `oc rollout undo`

---

## Step 1: Create Project

```bash
oc new-project lab-06 --display-name="Lab 06 - S2I Build"
```

---

## Step 2: Apply Manifests

```bash
# Create ImageStream (tracks built images)
oc apply -f manifests/imagestream.yaml

# Create BuildConfig (defines how to build)
oc apply -f manifests/buildconfig.yaml

# Create Deployment (references ImageStream)
oc apply -f manifests/deployment.yaml

# Check ImageStream
oc get imagestream nodejs-app

# Check BuildConfig
oc get buildconfig nodejs-app
```

---

## Step 3: Trigger a Build

```bash
# Start a build
oc start-build nodejs-app --follow

# Watch the build logs in real time
oc logs -f bc/nodejs-app

# Check build status
oc get builds
# Expected: nodejs-app-1   Running → Complete
```

---

## Step 4: Verify Deployment

```bash
# After build completes, check if deployment updated
oc get pods

# Create a route to test the app
oc expose svc/nodejs-app

ROUTE=$(oc get route nodejs-app -o jsonpath='{.spec.host}')
curl http://$ROUTE
```

---

## Step 5: Trigger Another Build (Simulate Code Change)

```bash
# Start a second build
oc start-build nodejs-app --follow

# Watch the deployment update automatically via ImageStream trigger
oc rollout status deployment/nodejs-app

# View rollout history
oc rollout history deployment/nodejs-app
```

---

## Step 6: Rollback

```bash
# Rollback to previous build/deployment
oc rollout undo deployment/nodejs-app

# Verify rollback
oc rollout status deployment/nodejs-app
oc rollout history deployment/nodejs-app
```

---

## Step 7: Configure GitHub Webhook

```bash
# Get the GitHub webhook URL
oc describe bc/nodejs-app | grep -A 3 "GitHub"

# Copy the webhook URL and add it in your GitHub repository:
# Settings → Webhooks → Add webhook
# Payload URL: <webhook-url>
# Content type: application/json
# Secret: <secret-from-buildconfig>
# Events: Just the push event

# After adding webhook, push a commit to GitHub
# BuildConfig will automatically start a new build
```

---

## Step 8: Binary Build (Deploy from Local File)

```bash
# Build from local directory
oc start-build nodejs-app --from-dir=. --follow

# Or build from a specific file
# oc start-build nodejs-app --from-file=app.tar.gz --follow
```

---

## Cleanup

```bash
oc delete project lab-06
```
