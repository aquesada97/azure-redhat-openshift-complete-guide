# StatefulSets, Jobs, and Advanced Workloads

## StatefulSets

StatefulSets manage pods that require:
- **Stable, unique network identity**: `pod-0`, `pod-1`, `pod-2`
- **Stable persistent storage**: each pod gets its own PVC
- **Ordered deployment and scaling**: pods are created/deleted in order

Use StatefulSets for: databases (PostgreSQL, MySQL, MongoDB), distributed systems (Kafka, ZooKeeper, Elasticsearch), or any app requiring stable identity.

### StatefulSet YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: my-project
spec:
  serviceName: postgres-headless   # Must match a headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:15
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: managed-csi
        resources:
          requests:
            storage: 20Gi
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
  namespace: my-project
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

### StatefulSet Pod Identity

Pods get stable DNS names:
```
postgres-0.postgres-headless.my-project.svc.cluster.local
postgres-1.postgres-headless.my-project.svc.cluster.local
postgres-2.postgres-headless.my-project.svc.cluster.local
```

```bash
# Scale StatefulSet (ordered: scale up 0→1→2, scale down 2→1→0)
oc scale statefulset postgres --replicas=5
oc get pods -w   # Watch ordered startup
```

---

## Jobs

A Job runs a pod to completion. Unlike a Deployment, a Job pod is not restarted after success — only on failure.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  namespace: my-project
spec:
  completions: 1          # Number of successful completions needed
  parallelism: 1          # Number of pods running in parallel
  backoffLimit: 3         # Retries on failure before marking Job as failed
  activeDeadlineSeconds: 300  # Fail job if not complete in 5 minutes
  template:
    spec:
      restartPolicy: OnFailure  # Required for Jobs: OnFailure or Never
      containers:
        - name: migration
          image: my-app:latest
          command: ["/bin/sh", "-c", "python manage.py migrate"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: url
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
```

```bash
# View job status
oc get jobs
oc describe job db-migration
oc logs -l job-name=db-migration
```

---

## CronJobs

CronJobs create Jobs on a schedule (standard cron syntax):

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
  namespace: my-project
spec:
  schedule: "0 2 * * *"    # Run at 2 AM daily (UTC)
  concurrencyPolicy: Forbid # Don't run if previous job is still running
  successfulJobsHistoryLimit: 3  # Keep last 3 successful jobs
  failedJobsHistoryLimit: 1      # Keep last 1 failed job
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:15
              command:
                - /bin/sh
                - -c
                - pg_dump $DATABASE_URL | gzip > /backup/db-$(date +%Y%m%d).sql.gz
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: db-secret
                      key: url
              volumeMounts:
                - name: backup-storage
                  mountPath: /backup
          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-pvc
```

```bash
# Trigger a CronJob manually
oc create job manual-backup --from=cronjob/db-backup

# View CronJob history
oc get jobs --selector=cronjob-name=db-backup
```

---

## DaemonSets

A DaemonSet ensures one pod runs on every (or selected) node. Use for log collectors, monitoring agents, node configuration tools, or security agents.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: openshift-logging
spec:
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      tolerations:
        - key: node-role.kubernetes.io/master
          operator: Exists
          effect: NoSchedule
      containers:
        - name: fluentd
          image: fluent/fluentd:v1.16
          resources:
            requests:
              cpu: "100m"
              memory: "200Mi"
            limits:
              cpu: "500m"
              memory: "500Mi"
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

---

## Init Containers

Init containers run to completion before app containers start. Use them for:
- Database migration before app starts
- Waiting for a dependency to become available
- Fetching configuration or secrets
- Setting file permissions

```yaml
spec:
  initContainers:
    - name: wait-for-db
      image: busybox:1.36
      command:
        - sh
        - -c
        - |
          until nc -z postgres-service 5432; do
            echo "Waiting for database...";
            sleep 2;
          done
          echo "Database is ready!"
    - name: run-migrations
      image: my-app:latest
      command: ["python", "manage.py", "migrate"]
      env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
  containers:
    - name: my-app
      image: my-app:latest
      # App container starts after all init containers complete
```

---

## Multi-Container Pods

### Sidecar Pattern

A helper container runs alongside the main container, extending its functionality:

```yaml
spec:
  containers:
    - name: my-app
      image: my-app:latest
      ports:
        - containerPort: 8080
    - name: log-shipper       # Sidecar: reads logs, ships to aggregator
      image: fluentbit:latest
      volumeMounts:
        - name: logs
          mountPath: /logs
  volumes:
    - name: logs
      emptyDir: {}
```

### Ambassador Pattern

A proxy container handles external communication on behalf of the main container:

```yaml
spec:
  containers:
    - name: my-app
      image: my-app:latest
      env:
        - name: DB_HOST
          value: "localhost"   # Talks to proxy on localhost
        - name: DB_PORT
          value: "5432"
    - name: db-proxy          # Ambassador: handles auth/TLS to real DB
      image: cloud-sql-proxy:latest
      command:
        - /cloud-sql-proxy
        - --address=0.0.0.0
        - --port=5432
        - myproject:eastus:mydb
```
