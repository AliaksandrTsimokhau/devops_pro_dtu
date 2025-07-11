# Advanced Kubernetes Hands-On Labs

## Prerequisites
- Kubernetes cluster (v1.25+) with admin access
- kubectl configured
- Docker installed
- Basic understanding of Kubernetes concepts
- Helm 3.x installed

---

## Lab 1: Custom Resource Definitions (CRDs) and Custom Controllers

### Objective
Create a custom resource definition and implement a simple controller to manage custom resources.

### Steps

1. **Create a Custom Resource Definition**
```yaml
# crd-website.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.example.com
spec:
  group: example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              url:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 10
          status:
            type: object
            properties:
              availableReplicas:
                type: integer
  scope: Namespaced
  names:
    plural: websites
    singular: website
    kind: Website
```

2. **Apply the CRD**
```bash
kubectl apply -f crd-website.yaml
kubectl get crd websites.example.com
```

3. **Create a Custom Resource**
```yaml
# website-example.yaml
apiVersion: example.com/v1
kind: Website
metadata:
  name: my-website
  namespace: default
spec:
  url: "https://example.com"
  replicas: 3
```

4. **Apply and verify**
```bash
kubectl apply -f website-example.yaml
kubectl get websites
kubectl describe website my-website
```

5. **Create a simple controller script**
```bash
# Install required tools
pip3 install kubernetes

# Create controller.py
cat > controller.py << 'EOF'
import time
from kubernetes import client, config, watch

def main():
    config.load_incluster_config() if config.load_incluster_config() else config.load_kube_config()

    v1 = client.CustomObjectsApi()

    while True:
        try:
            websites = v1.list_namespaced_custom_object(
                group="example.com",
                version="v1",
                namespace="default",
                plural="websites"
            )

            for website in websites['items']:
                name = website['metadata']['name']
                spec = website.get('spec', {})
                print(f"Processing website: {name}, URL: {spec.get('url')}, Replicas: {spec.get('replicas')}")

        except Exception as e:
            print(f"Error: {e}")

        time.sleep(30)

if __name__ == "__main__":
    main()
EOF
```

### Verification
- CRD is created and accessible
- Custom resources can be created and retrieved
- Controller monitors custom resources

---

## Lab 2: Advanced RBAC and Security Contexts

### Objective
Implement fine-grained RBAC policies and security contexts for workload isolation.

### Steps

1. **Create a dedicated namespace**
```bash
kubectl create namespace secure-app
```

2. **Create a service account with limited permissions**
```yaml
# rbac-setup.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-app-sa
  namespace: secure-app

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: secure-app
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "create", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: secure-app
subjects:
- kind: ServiceAccount
  name: secure-app-sa
  namespace: secure-app
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

3. **Deploy a secure application**
```yaml
# secure-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
  namespace: secure-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      serviceAccountName: secure-app-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: app
        image: nginx:alpine
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 50m
            memory: 64Mi
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: cache-volume
          mountPath: /var/cache/nginx
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: cache-volume
        emptyDir: {}
```

4. **Apply and test**
```bash
kubectl apply -f rbac-setup.yaml
kubectl apply -f secure-app.yaml

# Test RBAC permissions
kubectl auth can-i get pods --as=system:serviceaccount:secure-app:secure-app-sa -n secure-app
kubectl auth can-i create deployments --as=system:serviceaccount:secure-app:secure-app-sa -n secure-app
```

### Verification
- Service account has limited permissions
- Pods run with non-root user
- Security contexts are properly applied

---

## Lab 3: Network Policies for Micro-segmentation

### Objective
Implement network policies to control traffic flow between pods and namespaces.

### Steps

1. **Create test namespaces and applications**
```bash
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database

# Label namespaces
kubectl label namespace frontend tier=frontend
kubectl label namespace backend tier=backend
kubectl label namespace database tier=database
```

2. **Deploy test applications**
```yaml
# frontend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: frontend
        image: nginx:alpine
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

3. **Deploy backend and database similarly**
```yaml
# backend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
      - name: backend
        image: httpd:alpine
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80

---
# database-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
        tier: database
    spec:
      containers:
      - name: database
        image: postgres:alpine
        env:
        - name: POSTGRES_PASSWORD
          value: "password"
        ports:
        - containerPort: 5432

---
apiVersion: v1
kind: Service
metadata:
  name: database-svc
  namespace: database
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

4. **Create network policies**
```yaml
# network-policies.yaml
# Deny all ingress traffic to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-deny-all
  namespace: database
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# Allow backend to access database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-allow-backend
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432

---
# Allow frontend to access backend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80

---
# Frontend egress policy
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
  - to: {}
    ports:
    - protocol: UDP
      port: 53
```

5. **Test network connectivity**
```bash
# Deploy all resources
kubectl apply -f frontend-app.yaml
kubectl apply -f backend-app.yaml
kubectl apply -f database-app.yaml
kubectl apply -f network-policies.yaml

# Test connectivity
kubectl exec -n frontend deployment/frontend -- wget -qO- http://backend-svc.backend.svc.cluster.local
kubectl exec -n backend deployment/backend -- nc -zv database-svc.database.svc.cluster.local 5432
kubectl exec -n frontend deployment/frontend -- nc -zv database-svc.database.svc.cluster.local 5432 # Should fail
```

### Verification
- Frontend can access backend
- Backend can access database
- Frontend cannot directly access database

---

## Lab 4: Operators and Operator Framework

### Objective
Deploy and manage applications using Kubernetes operators.

### Steps

1. **Install Operator Lifecycle Manager (OLM)**
```bash
curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.25.0/install.sh | bash -s v0.25.0
```

2. **Install Prometheus Operator**
```bash
kubectl create -f https://operatorhub.io/install/prometheus.yaml
```

3. **Create a Prometheus instance**
```yaml
# prometheus-instance.yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-example
  namespace: default
spec:
  serviceAccountName: prometheus-example
  serviceMonitorSelector:
    matchLabels:
      app: example-app
  resources:
    requests:
      memory: 400Mi
  retention: 7d
  storage:
    volumeClaimTemplate:
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus-example
  namespace: default

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-example
rules:
- apiGroups: [""]
  resources: ["nodes", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["extensions"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-example
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus-example
subjects:
- kind: ServiceAccount
  name: prometheus-example
  namespace: default
```

4. **Create a ServiceMonitor**
```yaml
# service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: example-app-monitor
  labels:
    app: example-app
spec:
  selector:
    matchLabels:
      app: example-app
  endpoints:
  - port: metrics
    path: /metrics
    interval: 30s
```

5. **Deploy a sample application with metrics**
```yaml
# example-app-with-metrics.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: example-app
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: app
        image: prom/node-exporter
        ports:
        - containerPort: 9100
          name: metrics

---
apiVersion: v1
kind: Service
metadata:
  name: example-app-svc
  labels:
    app: example-app
spec:
  selector:
    app: example-app
  ports:
  - name: metrics
    port: 9100
    targetPort: 9100
```

6. **Apply and verify**
```bash
kubectl apply -f prometheus-instance.yaml
kubectl apply -f service-monitor.yaml
kubectl apply -f example-app-with-metrics.yaml

# Check Prometheus pod
kubectl get pods -l app.kubernetes.io/name=prometheus

# Port forward to access Prometheus UI
kubectl port-forward svc/prometheus-operated 9090:9090
```

### Verification
- Prometheus operator is running
- Prometheus instance is created and collecting metrics
- ServiceMonitor is discovering targets

---

## Lab 5: Horizontal Pod Autoscaler (HPA) with Custom Metrics

### Objective
Configure HPA with custom metrics using Prometheus adapter.

### Steps

1. **Install Metrics Server (if not already installed)**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

2. **Deploy Prometheus Adapter**
```yaml
# prometheus-adapter.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: adapter-config
  namespace: monitoring
data:
  config.yaml: |
    rules:
    - seriesQuery: 'http_requests_total{namespace!="",pod!=""}'
      resources:
        overrides:
          namespace: {resource: "namespace"}
          pod: {resource: "pod"}
      name:
        matches: "^(.*)_total"
        as: "${1}_per_second"
      metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[2m])'

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-adapter
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-adapter
  template:
    metadata:
      labels:
        app: prometheus-adapter
    spec:
      containers:
      - name: prometheus-adapter
        image: k8s.gcr.io/prometheus-adapter/prometheus-adapter:v0.10.0
        args:
        - --cert-dir=/tmp/cert
        - --secure-port=6443
        - --prometheus-url=http://prometheus-operated.default.svc:9090
        - --config=/etc/adapter/config.yaml
        volumeMounts:
        - name: config
          mountPath: /etc/adapter
        ports:
        - containerPort: 6443
      volumes:
      - name: config
        configMap:
          name: adapter-config
```

3. **Create a test application that generates metrics**
```yaml
# test-app-with-metrics.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: test-app
  template:
    metadata:
      labels:
        app: test-app
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi

---
apiVersion: v1
kind: Service
metadata:
  name: test-app-svc
spec:
  selector:
    app: test-app
  ports:
  - port: 80
    targetPort: 80
```

4. **Create HPA with multiple metrics**
```yaml
# hpa-custom-metrics.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: test-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: test-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
```

5. **Load test the application**
```bash
# Apply configurations
kubectl apply -f test-app-with-metrics.yaml
kubectl apply -f hpa-custom-metrics.yaml

# Generate load
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh
# Inside the pod:
while true; do wget -q -O- http://test-app-svc.default.svc.cluster.local; done

# Monitor HPA in another terminal
kubectl get hpa test-app-hpa --watch
```

### Verification
- HPA scales based on CPU and memory metrics
- Custom scaling behavior is applied
- Metrics are being collected and used for scaling decisions

---

## Lab 6: StatefulSets with Persistent Storage

### Objective
Deploy a StatefulSet with persistent storage and understand ordered deployment.

### Steps

1. **Create a StorageClass**
```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd  # Change based on your cloud provider
parameters:
  type: pd-ssd
  replication-type: none
allowVolumeExpansion: true
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

2. **Deploy a StatefulSet (MongoDB replica set)**
```yaml
# mongodb-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-headless
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
  - port: 27017
    targetPort: 27017

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: mongodb-headless
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: mongodb
        image: mongo:5.0
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          value: "admin"
        - name: MONGO_INITDB_ROOT_PASSWORD
          value: "password123"
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        - name: mongodb-config
          mountPath: /data/configdb
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
          limits:
            cpu: 1000m
            memory: 2Gi
        livenessProbe:
          exec:
            command:
            - mongo
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - mongo
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: mongodb-config
        emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: mongodb-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
```

3. **Create initialization job**
```yaml
# mongodb-init-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: mongodb-init
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: mongodb-init
        image: mongo:5.0
        command:
        - /bin/bash
        - -c
        - |
          sleep 60
          mongo --host mongodb-0.mongodb-headless:27017 -u admin -p password123 --authenticationDatabase admin <<EOF
          rs.initiate({
            _id: "rs0",
            members: [
              {_id: 0, host: "mongodb-0.mongodb-headless:27017"},
              {_id: 1, host: "mongodb-1.mongodb-headless:27017"},
              {_id: 2, host: "mongodb-2.mongodb-headless:27017"}
            ]
          })
          EOF
```

4. **Create a PodDisruptionBudget**
```yaml
# pdb-mongodb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: mongodb-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: mongodb
```

5. **Deploy and test**
```bash
kubectl apply -f storage-class.yaml
kubectl apply -f mongodb-statefulset.yaml
kubectl apply -f pdb-mongodb.yaml

# Wait for pods to be ready
kubectl get pods -l app=mongodb --watch

# Initialize replica set
kubectl apply -f mongodb-init-job.yaml

# Test ordered scaling
kubectl scale statefulset mongodb --replicas=2
kubectl get pods -l app=mongodb --watch

kubectl scale statefulset mongodb --replicas=3
kubectl get pods -l app=mongodb --watch
```

6. **Test data persistence**
```bash
# Connect to primary and insert data
kubectl exec -it mongodb-0 -- mongo -u admin -p password123 --authenticationDatabase admin
# Inside mongo shell:
use testdb
db.testcol.insert({name: "test", value: 123})
db.testcol.find()
exit

# Delete pod and verify data persistence
kubectl delete pod mongodb-0
kubectl get pods -l app=mongodb --watch

# Reconnect and verify data
kubectl exec -it mongodb-0 -- mongo -u admin -p password123 --authenticationDatabase admin
use testdb
db.testcol.find()
```

### Verification
- StatefulSet pods are created in order
- Each pod gets its own persistent volume
- Data persists across pod restarts
- Replica set is properly configured

---

## Lab 7: Jobs and CronJobs for Batch Processing

### Objective
Create and manage batch workloads using Jobs and CronJobs with various completion patterns.

### Steps

1. **Create a simple Job**
```yaml
# simple-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
spec:
  parallelism: 3
  completions: 6
  backoffLimit: 4
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

2. **Create a Job with work queue pattern**
```yaml
# work-queue-job.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: job-config
data:
  tasks.txt: |
    task-1
    task-2
    task-3
    task-4
    task-5
    task-6
    task-7
    task-8
    task-9
    task-10

---
apiVersion: batch/v1
kind: Job
metadata:
  name: work-queue-job
spec:
  parallelism: 3
  completions: 10
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: worker
        image: busybox:1.35
        command:
        - /bin/sh
        - -c
        - |
          TASK_ID=$RANDOM
          echo "Worker $HOSTNAME processing task $TASK_ID"
          # Simulate work
          sleep $((10 + $RANDOM % 20))
          echo "Worker $HOSTNAME completed task $TASK_ID"
        volumeMounts:
        - name: task-config
          mountPath: /tasks
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
      volumes:
      - name: task-config
        configMap:
          name: job-config
```

3. **Create a CronJob for regular maintenance**
```yaml
# maintenance-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  timeZone: "UTC"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 300
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 3600
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:15-alpine
            env:
            - name: PGPASSWORD
              value: "password123"
            command:
            - /bin/sh
            - -c
            - |
              echo "Starting backup at $(date)"
              # Simulate backup process
              pg_dump -h mongodb-headless -U admin -d testdb > /backup/backup-$(date +%Y%m%d-%H%M%S).sql || true
              echo "Backup completed at $(date)"
              # Cleanup old backups
              find /backup -name "backup-*.sql" -mtime +7 -delete
            volumeMounts:
            - name: backup-storage
              mountPath: /backup
            resources:
              requests:
                cpu: 100m
                memory: 256Mi
              limits:
                cpu: 500m
                memory: 512Mi
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

4. **Create a CronJob with success/failure notifications**
```yaml
# notification-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: health-check
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
  concurrencyPolicy: Replace
  successfulJobsHistoryLimit: 2
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: Never
          containers:
          - name: health-checker
            image: curlimages/curl:8.0.0
            command:
            - /bin/sh
            - -c
            - |
              echo "Health check started at $(date)"

              # Check service health
              if curl -f -s http://test-app-svc.default.svc.cluster.local/health; then
                echo "✅ Service is healthy"
                exit 0
              else
                echo "❌ Service health check failed"
                # Send notification (webhook, slack, etc.)
                curl -X POST -H 'Content-type: application/json' \
                  --data '{"text":"Service health check failed"}' \
                  $WEBHOOK_URL || true
                exit 1
              fi
            env:
            - name: WEBHOOK_URL
              value: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
            resources:
              requests:
                cpu: 50m
                memory: 64Mi
```

5. **Deploy and test**
```bash
# Apply all jobs
kubectl apply -f simple-job.yaml
kubectl apply -f work-queue-job.yaml
kubectl apply -f maintenance-cronjob.yaml
kubectl apply -f notification-cronjob.yaml

# Monitor jobs
kubectl get jobs --watch
kubectl get cronjobs

# Check job logs
kubectl logs -l job-name=pi-calculation
kubectl logs -l job-name=work-queue-job

# Test CronJob manually
kubectl create job --from=cronjob/database-backup manual-backup-test

# Check CronJob history
kubectl get jobs -l cronjob=database-backup
```

### Verification
- Jobs complete with specified parallelism and completions
- CronJobs run on schedule
- Failed jobs are retried according to backoffLimit
- Job history is maintained according to configured limits

---

## Lab 8: Advanced Ingress with SSL/TLS and Rate Limiting

### Objective
Configure advanced ingress features including SSL termination, rate limiting, and path-based routing.

### Steps

1. **Install NGINX Ingress Controller**
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
```

2. **Deploy sample applications**
```yaml
# sample-apps.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
      version: v1
  template:
    metadata:
      labels:
        app: sample-app
        version: v1
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: content
        configMap:
          name: app-v1-content

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-v2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
      version: v2
  template:
    metadata:
      labels:
        app: sample-app
        version: v2
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: content
        configMap:
          name: app-v2-content

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-v1-content
data:
  index.html: |
    <html>
    <body>
      <h1>Application Version 1</h1>
      <p>This is version 1 of the application</p>
    </body>
    </html>

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-v2-content
data:
  index.html: |
    <html>
    <body>
      <h1>Application Version 2</h1>
      <p>This is version 2 of the application</p>
    </body>
    </html>

---
apiVersion: v1
kind: Service
metadata:
  name: app-v1-svc
spec:
  selector:
    app: sample-app
    version: v1
  ports:
  - port: 80
    targetPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: app-v2-svc
spec:
  selector:
    app: sample-app
    version: v2
  ports:
  - port: 80
    targetPort: 80
```

3. **Create SSL certificate**
```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.local/O=myapp"

# Create TLS secret
kubectl create secret tls myapp-tls --key tls.key --cert tls.crt
```

4. **Create advanced Ingress with multiple features**
```yaml
# advanced-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "10"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/rate-limit-connections: "5"
    nginx.ingress.kubernetes.io/canary: "false"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range"
    nginx.ingress.kubernetes.io/enable-cors: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.local
    secretName: myapp-tls
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: app-v1-svc
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: app-v2-svc
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v1-svc
            port:
              number: 80
```

5. **Create canary deployment ingress**
```yaml
# canary-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: canary-ingress
  annotations:
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "20"
    nginx.ingress.kubernetes.io/canary-by-header: "X-Canary"
    nginx.ingress.kubernetes.io/canary-by-header-value: "always"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.local
    secretName: myapp-tls
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-v2-svc
            port:
              number: 80
```

6. **Create rate limiting with custom responses**
```yaml
# rate-limit-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
data:
  rate-limit-status-code: "429"
  rate-limit-exceed-action: "drop"
  custom-http-errors: "429,503"
```

7. **Test the ingress**
```bash
# Apply all configurations
kubectl apply -f sample-apps.yaml
kubectl apply -f advanced-ingress.yaml
kubectl apply -f canary-ingress.yaml

# Get ingress IP
kubectl get ingress advanced-ingress

# Add to /etc/hosts (replace with actual ingress IP)
echo "$(kubectl get ingress advanced-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}') myapp.local" | sudo tee -a /etc/hosts

# Test different paths
curl -k https://myapp.local/v1
curl -k https://myapp.local/v2
curl -k https://myapp.local/

# Test canary deployment
curl -k https://myapp.local/ -H "X-Canary: always"

# Test rate limiting
for i in {1..15}; do curl -k https://myapp.local/; sleep 1; done

# Test CORS
curl -k -X OPTIONS https://myapp.local/ -H "Origin: http://example.com" -v
```

### Verification
- SSL/TLS termination works correctly
- Path-based routing directs traffic appropriately
- Rate limiting blocks excessive requests
- Canary deployment routes traffic based on headers and weight
- CORS headers are properly set

---

## Lab 9: Pod Security Standards and Admission Controllers

### Objective
Implement Pod Security Standards and configure admission controllers for policy enforcement.

### Steps

1. **Enable Pod Security Standards**
```yaml
# pod-security-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted

---
apiVersion: v1
kind: Namespace
metadata:
  name: baseline-ns
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline

---
apiVersion: v1
kind: Namespace
metadata:
  name: privileged-ns
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
```

2. **Install Open Policy Agent (OPA) Gatekeeper**
```bash
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml

# Wait for gatekeeper to be ready
kubectl wait --for=condition=Ready --timeout=180s pod -l gatekeeper.sh/operation=webhook -n gatekeeper-system
```

3. **Create constraint templates**
```yaml
# constraint-templates.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        type: object
        properties:
          labels:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          required := input.parameters.labels
          provided := input.review.object.metadata.labels
          missing := required[_]
          not provided[missing]
          msg := sprintf("Missing required label: %v", [missing])
        }

---
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sresourcelimits
spec:
  crd:
    spec:
      names:
        kind: K8sResourceLimits
      validation:
        type: object
        properties:
          cpuLimit:
            type: string
          memoryLimit:
            type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sresourcelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := "CPU limits are required"
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := "Memory limits are required"
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          cpu_limit := container.resources.limits.cpu
          cpu_limit > input.parameters.cpuLimit
          msg := sprintf("CPU limit %v exceeds maximum allowed %v", [cpu_limit, input.parameters.cpuLimit])
        }
```

4. **Create constraints**
```yaml
# constraints.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: must-have-labels
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces: ["restricted-ns", "baseline-ns"]
  parameters:
    labels: ["app", "version", "environment"]

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sResourceLimits
metadata:
  name: resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["restricted-ns"]
  parameters:
    cpuLimit: "1"
    memoryLimit: "1Gi"
```

5. **Create ValidatingAdmissionWebhook**
```yaml
# admission-webhook.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionWebhook
metadata:
  name: security-policy-webhook
webhooks:
- name: security.example.com
  clientConfig:
    service:
      name: security-webhook-svc
      namespace: default
      path: "/validate"
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  failurePolicy: Fail

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: security-webhook
spec:
  replicas: 1
  selector:
    matchLabels:
      app: security-webhook
  template:
    metadata:
      labels:
        app: security-webhook
    spec:
      containers:
      - name: webhook
        image: nginx:alpine
        ports:
        - containerPort: 8443
        env:
        - name: TLS_CERT_FILE
          value: "/etc/certs/tls.crt"
        - name: TLS_PRIVATE_KEY_FILE
          value: "/etc/certs/tls.key"
        volumeMounts:
        - name: certs
          mountPath: /etc/certs
          readOnly: true
      volumes:
      - name: certs
        secret:
          secretName: webhook-certs

---
apiVersion: v1
kind: Service
metadata:
  name: security-webhook-svc
spec:
  selector:
    app: security-webhook
  ports:
  - port: 443
    targetPort: 8443
```

6. **Test pod security standards**
```yaml
# test-pods.yaml
# This should be allowed in privileged namespace
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: privileged-ns
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true

---
# This should be rejected in restricted namespace
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod-restricted
  namespace: restricted-ns
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true

---
# This should be allowed in restricted namespace
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  namespace: restricted-ns
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
      requests:
        cpu: 100m
        memory: 128Mi
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

7. **Test constraints**
```bash
# Apply namespaces and policies
kubectl apply -f pod-security-namespace.yaml
kubectl apply -f constraint-templates.yaml
kubectl apply -f constraints.yaml

# Test deployments without required labels (should fail)
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bad-deployment
  namespace: restricted-ns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bad-app
  template:
    metadata:
      labels:
        app: bad-app
    spec:
      containers:
      - name: app
        image: nginx
EOF

# Test deployment with required labels (should succeed)
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: good-deployment
  namespace: restricted-ns
  labels:
    app: good-app
    version: v1
    environment: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: good-app
  template:
    metadata:
      labels:
        app: good-app
        version: v1
        environment: test
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 100m
            memory: 128Mi
EOF

# Test pod security standards
kubectl apply -f test-pods.yaml

# Check constraint violations
kubectl get K8sRequiredLabels must-have-labels -o yaml
kubectl get K8sResourceLimits resource-limits -o yaml
```

### Verification
- Pod Security Standards enforce appropriate security policies
- OPA Gatekeeper constraints block non-compliant resources
- Admission webhooks validate resources according to custom policies
- Different namespaces enforce different security levels

---

## Lab 10: Multi-Cluster Management with Cluster API

### Objective
Set up and manage multiple Kubernetes clusters using Cluster API for infrastructure automation.

### Steps

1. **Install clusterctl**
```bash
# Download and install clusterctl
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.5.1/clusterctl-linux-amd64 -o clusterctl
chmod +x clusterctl
sudo mv clusterctl /usr/local/bin/

# Verify installation
clusterctl version
```

2. **Initialize Cluster API management cluster**
```bash
# Set environment variables for your cloud provider
export AWS_REGION=us-west-2
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_B64ENCODED_CREDENTIALS=$(clusterctl generate credentials | base64 -w 0)

# Initialize the management cluster
clusterctl init --infrastructure aws
```

3. **Create cluster configuration**
```yaml
# cluster-config.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: dev-cluster
  namespace: default
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
    kind: AWSCluster
    name: dev-cluster
  controlPlaneRef:
    kind: KubeadmControlPlane
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    name: dev-cluster-control-plane

---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSCluster
metadata:
  name: dev-cluster
  namespace: default
spec:
  region: us-west-2
  sshKeyName: my-key-pair
  networkSpec:
    vpc:
      availabilityZoneUsageLimit: 3
      availabilityZoneSelection: Ordered
    subnets:
    - availabilityZone: us-west-2a
      cidrBlock: "10.0.0.0/24"
      isPublic: true
    - availabilityZone: us-west-2b
      cidrBlock: "10.0.1.0/24"
      isPublic: true
    - availabilityZone: us-west-2c
      cidrBlock: "10.0.2.0/24"
      isPublic: true

---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: dev-cluster-control-plane
  namespace: default
spec:
  version: v1.28.0
  replicas: 3
  machineTemplate:
    infrastructureRef:
      kind: AWSMachineTemplate
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
      name: dev-cluster-control-plane
  kubeadmConfigSpec:
    initConfiguration:
      nodeRegistration:
        kubeletExtraArgs:
          cloud-provider: aws
    clusterConfiguration:
      apiServer:
        extraArgs:
          cloud-provider: aws
      controllerManager:
        extraArgs:
          cloud-provider: aws
    joinConfiguration:
      nodeRegistration:
        kubeletExtraArgs:
          cloud-provider: aws

---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSMachineTemplate
metadata:
  name: dev-cluster-control-plane
  namespace: default
spec:
  template:
    spec:
      instanceType: t3.medium
      ami:
        id: ami-0c02fb55956c7d316  # Ubuntu 20.04 LTS
      iamInstanceProfile: control-plane.cluster-api-provider-aws.sigs.k8s.io
      sshKeyName: my-key-pair

---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: dev-cluster-workers
  namespace: default
spec:
  clusterName: dev-cluster
  replicas: 3
  selector:
    matchLabels: {}
  template:
    spec:
      clusterName: dev-cluster
      version: v1.28.0
      bootstrap:
        configRef:
          name: dev-cluster-workers
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
      infrastructureRef:
        name: dev-cluster-workers
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
        kind: AWSMachineTemplate

---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta2
kind: AWSMachineTemplate
metadata:
  name: dev-cluster-workers
  namespace: default
spec:
  template:
    spec:
      instanceType: t3.medium
      ami:
        id: ami-0c02fb55956c7d316
      iamInstanceProfile: nodes.cluster-api-provider-aws.sigs.k8s.io
      sshKeyName: my-key-pair

---
apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
kind: KubeadmConfigTemplate
metadata:
  name: dev-cluster-workers
  namespace: default
spec:
  template:
    spec:
      joinConfiguration:
        nodeRegistration:
          kubeletExtraArgs:
            cloud-provider: aws
```

4. **Create cluster fleet management**
```yaml
# fleet-management.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: fleet-system

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-manager
  namespace: fleet-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-manager
  template:
    metadata:
      labels:
        app: cluster-manager
    spec:
      containers:
      - name: manager
        image: rancher/fleet:v0.7.0
        args:
        - --leader-elect
        - --namespace=fleet-system
        env:
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        resources:
          limits:
            cpu: 1000m
            memory: 1Gi
          requests:
            cpu: 100m
            memory: 128Mi

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: cluster-manager
  namespace: fleet-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-manager
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-manager
subjects:
- kind: ServiceAccount
  name: cluster-manager
  namespace: fleet-system
```

5. **Create cluster monitoring setup**
```yaml
# cluster-monitoring.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitor-config
  namespace: fleet-system
data:
  monitor.py: |
    import time
    import subprocess
    import json
    from kubernetes import client, config

    def monitor_clusters():
        config.load_incluster_config()
        v1 = client.CoreV1Api()

        while True:
            try:
                # Get all clusters
                result = subprocess.run(['kubectl', 'get', 'clusters', '-o', 'json'],
                                      capture_output=True, text=True)
                clusters = json.loads(result.stdout)

                for cluster in clusters['items']:
                    name = cluster['metadata']['name']
                    status = cluster.get('status', {})
                    phase = status.get('phase', 'Unknown')

                    print(f"Cluster: {name}, Phase: {phase}")

                    if phase == 'Provisioned':
                        # Get kubeconfig and test connectivity
                        kubeconfig_cmd = f"clusterctl get kubeconfig {name}"
                        result = subprocess.run(kubeconfig_cmd.split(),
                                              capture_output=True, text=True)

                        if result.returncode == 0:
                            print(f"✅ Cluster {name} is accessible")
                        else:
                            print(f"❌ Cluster {name} is not accessible")

            except Exception as e:
                print(f"Error monitoring clusters: {e}")

            time.sleep(60)

    if __name__ == "__main__":
        monitor_clusters()

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-monitor
  namespace: fleet-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cluster-monitor
  template:
    metadata:
      labels:
        app: cluster-monitor
    spec:
      containers:
      - name: monitor
        image: python:3.9-slim
        command: ["python", "/app/monitor.py"]
        volumeMounts:
        - name: config
          mountPath: /app
        - name: kubeconfig
          mountPath: /root/.kube
        env:
        - name: PYTHONPATH
          value: "/usr/local/lib/python3.9/site-packages"
      volumes:
      - name: config
        configMap:
          name: cluster-monitor-config
      - name: kubeconfig
        emptyDir: {}
      initContainers:
      - name: install-deps
        image: python:3.9-slim
        command: ["pip", "install", "kubernetes"]
        volumeMounts:
        - name: deps
          mountPath: /usr/local/lib/python3.9/site-packages
      volumes:
      - name: deps
        emptyDir: {}
```

6. **Deploy and manage clusters**
```bash
# Apply cluster configuration
kubectl apply -f cluster-config.yaml

# Monitor cluster creation
kubectl get clusters --watch
kubectl get kubeadmcontrolplane --watch
kubectl get machinedeployments --watch

# Get kubeconfig for the new cluster
clusterctl get kubeconfig dev-cluster > dev-cluster-kubeconfig.yaml

# Switch to the new cluster
export KUBECONFIG=dev-cluster-kubeconfig.yaml
kubectl get nodes

# Install CNI (Calico) on the new cluster
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# Deploy fleet management
kubectl apply -f fleet-management.yaml
kubectl apply -f cluster-monitoring.yaml

# Test cluster scaling
kubectl patch machinedeployment dev-cluster-workers --type='merge' -p='{"spec":{"replicas":5}}'

# Monitor scaling
kubectl get machines --watch
```

7. **Create multi-cluster application deployment**
```yaml
# multi-cluster-app.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  deploy.yaml: |
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: sample-app
      namespace: default
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: sample-app
      template:
        metadata:
          labels:
            app: sample-app
        spec:
          containers:
          - name: app
            image: nginx:alpine
            ports:
            - containerPort: 80
            resources:
              limits:
                cpu: 200m
                memory: 256Mi
              requests:
                cpu: 100m
                memory: 128Mi

    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: sample-app-svc
      namespace: default
    spec:
      selector:
        app: sample-app
      ports:
      - port: 80
        targetPort: 80
      type: LoadBalancer

---
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-cluster-deploy
  namespace: fleet-system
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: deployer
        image: bitnami/kubectl:latest
        command:
        - /bin/bash
        - -c
        - |
          # Get list of all clusters
          for cluster in $(kubectl get clusters -o jsonpath='{.items[*].metadata.name}'); do
            echo "Deploying to cluster: $cluster"

            # Get kubeconfig
            clusterctl get kubeconfig $cluster > /tmp/$cluster-kubeconfig

            # Deploy application
            kubectl --kubeconfig=/tmp/$cluster-kubeconfig apply -f /config/deploy.yaml

            echo "Deployed to $cluster successfully"
          done
        volumeMounts:
        - name: config
          mountPath: /config
      volumes:
      - name: config
        configMap:
          name: app-config
```

### Verification
- Management cluster can provision workload clusters
- Workload clusters are properly configured and accessible
- Multi-cluster deployments work across all clusters
- Cluster scaling operations complete successfully
- Fleet management provides centralized control

---

## Conclusion

These 10 advanced Kubernetes labs cover essential topics for intermediate to advanced practitioners:

1. **Custom Resource Definitions (CRDs)