## Getting Started: Core Kubernetes Resources

- **Practice hands-on with essential Kubernetes resources:**
  - **Pods:** Learn to deploy and manage single or multi-container Pods.
  - **Deployments:** Explore rolling updates, scaling, and self-healing features.

- **Try interactive labs for guided exercises:**
  - [Pods Lab](https://studio.kodekloud.com/labs/kubernetes/pods-stable)
  - [Deployments Lab](https://studio.kodekloud.com/labs/kubernetes/deployments-stable)

- **Estimated time:** 30 minutes

# Kubernetes Intermediate Labs - 10 Hands-On Exercises

## Prerequisites
- Kubernetes cluster (minikube, kind, or cloud provider)
- kubectl configured and connected to your cluster
- Basic understanding of Pods, Services, and Deployments
- Docker basics

## Lab 1: Advanced Deployment Strategies with Rolling Updates and Rollbacks

### Objective
Master deployment strategies, rolling updates, and rollback mechanisms.

### Instructions

1. **Create a deployment with version tracking:**
```bash
# Create deployment
kubectl create deployment webapp --image=nginx:1.19 --replicas=3

# Add labels for version tracking
kubectl patch deployment webapp -p '{"metadata":{"labels":{"version":"v1"}}}'
```

2. **Configure rolling update strategy:**
```yaml
# webapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    version: v1
spec:
  replicas: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: v1
    spec:
      containers:
      - name: webapp
        image: nginx:1.19
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

3. **Apply and monitor the deployment:**
```bash
kubectl apply -f webapp-deployment.yaml
kubectl rollout status deployment/webapp
kubectl get pods -l app=webapp --show-labels
```

4. **Perform rolling update:**
```bash
# Update to nginx:1.20
kubectl set image deployment/webapp webapp=nginx:1.20
kubectl rollout status deployment/webapp

# Watch the rolling update in real-time
kubectl get pods -l app=webapp -w
```

5. **Check rollout history:**
```bash
kubectl rollout history deployment/webapp
kubectl rollout history deployment/webapp --revision=2
```

6. **Simulate a bad deployment and rollback:**
```bash
# Deploy a non-existent image
kubectl set image deployment/webapp webapp=nginx:bad-version
kubectl rollout status deployment/webapp --timeout=60s

# Rollback to previous version
kubectl rollout undo deployment/webapp
kubectl rollout status deployment/webapp
```

### Verification
```bash
kubectl get deployment webapp -o wide
kubectl describe deployment webapp
```

---

## Lab 2: ConfigMaps and Secrets Management

### Objective
Learn to manage application configuration and sensitive data using ConfigMaps and Secrets.

### Instructions

1. **Create ConfigMap from literal values:**
```bash
kubectl create configmap app-config \
  --from-literal=database_url=postgresql://db:5432/myapp \
  --from-literal=redis_url=redis://redis:6379 \
  --from-literal=log_level=debug
```

2. **Create ConfigMap from file:**
```bash
# Create a config file
cat > app.properties << EOF
spring.datasource.url=jdbc:postgresql://db:5432/myapp
spring.redis.host=redis
spring.redis.port=6379
logging.level.root=INFO
EOF

kubectl create configmap app-properties --from-file=app.properties
```

3. **Create Secrets:**
```bash
# Create secret from literal
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=supersecret123

# Create TLS secret
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=myapp.local"
kubectl create secret tls tls-secret --key=tls.key --cert=tls.crt
```

4. **Create deployment using ConfigMaps and Secrets:**
```yaml
# configmap-secret-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configmap-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: configmap-app
  template:
    metadata:
      labels:
        app: configmap-app
    spec:
      containers:
      - name: app
        image: busybox
        command: ['sh', '-c', 'while true; do echo "Config: $DATABASE_URL, User: $DB_USER"; sleep 30; done']
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_url
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log_level
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: app-properties
      - name: secret-volume
        secret:
          secretName: db-credentials
```

5. **Apply and test:**
```bash
kubectl apply -f configmap-secret-app.yaml
kubectl logs -l app=configmap-app
kubectl exec -it deployment/configmap-app -- ls -la /etc/config
kubectl exec -it deployment/configmap-app -- ls -la /etc/secrets
```

### Verification
```bash
kubectl get configmaps
kubectl get secrets
kubectl describe configmap app-config
kubectl describe secret db-credentials
```

---

## Lab 3: Persistent Volumes and StatefulSets

### Objective
Understand persistent storage and stateful applications using StatefulSets.

### Instructions

1. **Create StorageClass (for dynamic provisioning):**
```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

2. **Create PersistentVolumes manually:**
```yaml
# pv-example.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  hostPath:
    path: /tmp/k8s-pv-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-database-2
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  hostPath:
    path: /tmp/k8s-pv-2
```

3. **Create StatefulSet with persistent storage:**
```yaml
# postgres-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-service
  replicas: 2
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
        image: postgres:13
        env:
        - name: POSTGRES_DB
          value: myapp
        - name: POSTGRES_USER
          value: admin
        - name: POSTGRES_PASSWORD
          value: password123
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        - name: init-script
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: init-script
        configMap:
          name: postgres-init
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 1Gi
```

4. **Create headless service:**
```yaml
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

5. **Create initialization ConfigMap:**
```yaml
# postgres-init-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-init
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS users (
      id SERIAL PRIMARY KEY,
      name VARCHAR(100),
      email VARCHAR(100)
    );
    INSERT INTO users (name, email) VALUES 
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com');
```

6. **Deploy everything:**
```bash
kubectl apply -f storage-class.yaml
kubectl apply -f pv-example.yaml
kubectl apply -f postgres-init-configmap.yaml
kubectl apply -f postgres-service.yaml
kubectl apply -f postgres-statefulset.yaml
```

7. **Test persistent storage:**
```bash
# Wait for pods to be ready
kubectl get statefulset postgres -w

# Connect to first replica and create data
kubectl exec -it postgres-0 -- psql -U admin -d myapp -c "INSERT INTO users (name, email) VALUES ('Charlie', 'charlie@example.com');"

# Verify data
kubectl exec -it postgres-0 -- psql -U admin -d myapp -c "SELECT * FROM users;"

# Delete pod and verify data persistence
kubectl delete pod postgres-0
kubectl get pods -l app=postgres -w

# Verify data survived pod restart
kubectl exec -it postgres-0 -- psql -U admin -d myapp -c "SELECT * FROM users;"
```

### Verification
```bash
kubectl get pv
kubectl get pvc
kubectl get statefulset
kubectl describe statefulset postgres
```

---

## Lab 4: Service Mesh Basics with Network Policies

### Objective
Implement network security policies and understand service-to-service communication.

### Instructions

1. **Enable NetworkPolicy support (if using minikube):**
```bash
# For minikube with Calico
minikube start --cni=calico
# Or enable after start
minikube addons enable calico
```

2. **Create a multi-tier application:**
```yaml
# frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    tier: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
      tier: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: frontend
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

3. **Create backend service:**
```yaml
# backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    tier: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
      tier: backend
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
  name: backend-service
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
```

4. **Create database service:**
```yaml
# database-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  labels:
    tier: database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: database
      tier: database
  template:
    metadata:
      labels:
        app: database
        tier: database
    spec:
      containers:
      - name: database
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          value: password
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
spec:
  selector:
    app: database
  ports:
  - port: 5432
    targetPort: 5432
```

5. **Create restrictive NetworkPolicies:**
```yaml
# network-policies.yaml
# Deny all ingress traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow frontend to receive traffic from anywhere
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-ingress
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Ingress
  ingress:
  - {} # Allow all ingress
---
# Allow frontend to communicate with backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  podSelector:
    matchLabels:
      tier: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
  - to: {} # Allow DNS
    ports:
    - protocol: UDP
      port: 53
---
# Allow backend to receive from frontend and communicate with database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      tier: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          tier: database
    ports:
    - protocol: TCP
      port: 5432
  - to: {} # Allow DNS
    ports:
    - protocol: UDP
      port: 53
---
# Allow database to receive from backend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-policy
spec:
  podSelector:
    matchLabels:
      tier: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 5432
```

6. **Deploy and test:**
```bash
kubectl apply -f frontend-deployment.yaml
kubectl apply -f backend-deployment.yaml
kubectl apply -f database-deployment.yaml

# Wait for all pods to be ready
kubectl get pods -o wide

# Apply network policies
kubectl apply -f network-policies.yaml

# Test connectivity
kubectl run test-pod --image=busybox -it --rm -- sh
# Inside the pod, try:
# wget -qO- frontend-service  # Should work
# wget -qO- backend-service   # Should be blocked
# wget -qO- database-service  # Should be blocked
```

### Verification
```bash
kubectl get networkpolicies
kubectl describe networkpolicy default-deny-all
kubectl get pods --show-labels
```

---

## Lab 5: Horizontal Pod Autoscaler (HPA) and Vertical Pod Autoscaler (VPA)

### Objective
Implement automatic scaling based on resource utilization.

### Instructions

1. **Enable metrics server (if not already enabled):**
```bash
# For minikube
minikube addons enable metrics-server

# Or install manually
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

2. **Create a resource-intensive application:**
```yaml
# cpu-intensive-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-intensive-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-intensive-app
  template:
    metadata:
      labels:
        app: cpu-intensive-app
    spec:
      containers:
      - name: app
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: cpu-intensive-service
spec:
  selector:
    app: cpu-intensive-app
  ports:
  - port: 80
    targetPort: 80
```

3. **Create Horizontal Pod Autoscaler:**
```yaml
# hpa-config.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cpu-intensive-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-intensive-app
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

4. **Deploy and test HPA:**
```bash
kubectl apply -f cpu-intensive-app.yaml
kubectl apply -f hpa-config.yaml

# Wait for deployment to be ready
kubectl get deployment cpu-intensive-app

# Check HPA status
kubectl get hpa
kubectl describe hpa cpu-intensive-hpa
```

5. **Generate load to trigger scaling:**
```bash
# Run load generator
kubectl run load-generator --image=busybox -it --rm -- sh

# Inside the load generator pod:
while true; do wget -q -O- http://cpu-intensive-service; done
```

6. **Monitor scaling in another terminal:**
```bash
# Watch HPA status
kubectl get hpa cpu-intensive-hpa -w

# Watch pods scaling
kubectl get pods -l app=cpu-intensive-app -w

# Check detailed metrics
kubectl top pods -l app=cpu-intensive-app
```

7. **Create VPA configuration:**
```yaml
# vpa-config.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: cpu-intensive-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cpu-intensive-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      maxAllowed:
        cpu: 1
        memory: 500Mi
      minAllowed:
        cpu: 50m
        memory: 32Mi
```

8. **Install VPA (if not already installed):**
```bash
# Clone VPA repository
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/

# Install VPA
./hack/vpa-up.sh
```

### Verification
```bash
kubectl get hpa
kubectl describe hpa cpu-intensive-hpa
kubectl get vpa
kubectl describe vpa cpu-intensive-vpa
kubectl top pods
```

---

## Lab 6: Jobs, CronJobs, and Init Containers

### Objective
Understand batch processing, scheduled tasks, and pod initialization patterns.

### Instructions

1. **Create a simple Job:**
```yaml
# simple-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
spec:
  parallelism: 2
  completions: 4
  backoffLimit: 3
  activeDeadlineSeconds: 300
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: pi
        image: perl
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
```

2. **Create a CronJob for regular tasks:**
```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "*/5 * * * *"  # Every 5 minutes
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: backup
            image: postgres:13
            command:
            - /bin/bash
            - -c
            - |
              echo "Starting backup at $(date)"
              echo "Simulating database backup..."
              sleep 30
              echo "Backup completed at $(date)"
            env:
            - name: PGPASSWORD
              value: "password"
```

3. **Create application with Init Containers:**
```yaml
# app-with-init.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Initialized App</title></head>
    <body>
      <h1>Application Successfully Initialized!</h1>
      <p>Init containers have prepared the environment.</p>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-with-init
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app-with-init
  template:
    metadata:
      labels:
        app: web-app-with-init
    spec:
      initContainers:
      - name: database-check
        image: postgres:13
        command:
        - sh
        - -c
        - |
          echo "Checking database connectivity..."
          until pg_isready -h database-service -p 5432; do
            echo "Waiting for database..."
            sleep 2
          done
          echo "Database is ready!"
      - name: setup-files
        image: busybox
        command:
        - sh
        - -c
        - |
          echo "Setting up application files..."
          cp /config/index.html /shared/
          echo "Files setup complete!"
        volumeMounts:
        - name: shared-data
          mountPath: /shared
        - name: config-volume
          mountPath: /config
      containers:
      - name: web-server
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: shared-data
        emptyDir: {}
      - name: config-volume
        configMap:
          name: web-content
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app-with-init
  ports:
  - port: 80
    targetPort: 80
```

4. **Create a dependency service (database):**
```yaml
# dependency-database.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dependency-database
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dependency-database
  template:
    metadata:
      labels:
        app: dependency-database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          value: password
        ports:
        - containerPort: 5432
---
apiVersion: v1
kind: Service
metadata:
  name: database-service
spec:
  selector:
    app: dependency-database
  ports:
  - port: 5432
    targetPort: 5432
```

5. **Deploy and test:**
```bash
# Deploy dependency first
kubectl apply -f dependency-database.yaml

# Wait for database to be ready
kubectl get pods -l app=dependency-database

# Deploy jobs
kubectl apply -f simple-job.yaml
kubectl apply -f backup-cronjob.yaml

# Deploy app with init containers
kubectl apply -f app-with-init.yaml

# Monitor job execution
kubectl get jobs
kubectl describe job pi-calculation
kubectl logs -l job-name=pi-calculation

# Monitor CronJob
kubectl get cronjobs
kubectl get jobs -l cronjob=database-backup

# Check init container logs
kubectl describe pod -l app=web-app-with-init
kubectl logs -l app=web-app-with-init -c database-check
kubectl logs -l app=web-app-with-init -c setup-files
```

6. **Test the web application:**
```bash
kubectl port-forward service/web-app-service 8080:80
# Visit http://localhost:8080 in browser
```

### Verification
```bash
kubectl get jobs
kubectl get cronjobs
kubectl describe cronjob database-backup
kubectl get pods -l app=web-app-with-init
```

---

## Lab 7: RBAC (Role-Based Access Control) and Service Accounts

### Objective
Implement security through proper authentication and authorization mechanisms.

### Instructions

1. **Create custom Service Account:**
```yaml
# service-accounts.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: default
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-service-account
  namespace: default
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: readonly-service-account
  namespace: default
```

2. **Create Roles with different permissions:**
```yaml
# roles.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployment-manager
  namespace: default
rules:
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-operator
  namespace: default
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "update", "patch"]
  resourceNames: ["app-config"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["app-secret"]
```

3. **Create ClusterRole for cluster-wide permissions:**
```yaml
# cluster-roles.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["metrics.k8s.io"]
  resources: ["nodes", "pods"]
  verbs: ["get", "list"]
```

4. **Create RoleBindings:**
```yaml
# role-bindings.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: readonly-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: readonly-service-account
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: admin-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: admin-service-account
  namespace: default
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: app-service-account
  namespace: default
roleRef:
  kind: Role
  name: app-operator
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-reader-binding
subjects:
- kind: ServiceAccount
  name: readonly-service-account
  namespace: default
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

5. **Create test resources:**
```yaml
# test-resources.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.yaml: |
    database:
      host: db.example.com
      port: 5432
    cache:
      enabled: true
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # password123
  api-key: YWJjZGVmZ2hpams=   # abcdefghijk
```

6. **Create pods with different service accounts:**
```yaml
# rbac-test-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-pod
spec:
  serviceAccountName: readonly-service-account
  containers:
  - name: kubectl
    image: bitnami/kubectl
    command: ['sleep', '3600']
---
apiVersion: v1
kind: Pod
metadata:
  name: admin-pod
spec:
  serviceAccountName: admin-service-account
  containers:
  - name: kubectl
    image: bitnami/kubectl
    command: ['sleep', '3600']
---
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  serviceAccountName: app-service-account
  containers:
  - name: kubectl
    image: bitnami/kubectl
    command: ['sleep', '3600']
```

7. **Deploy and test RBAC:**
```bash
# Apply all RBAC configurations
kubectl apply -f service-accounts.yaml
kubectl apply -f roles.yaml
kubectl apply -f cluster-roles.yaml
kubectl apply -f role-bindings.yaml
kubectl apply -f test-resources.yaml
kubectl apply -f rbac-test-pods.yaml

# Wait for pods to be ready
kubectl get pods

# Test readonly service account permissions
kubectl exec -it readonly-pod -- kubectl get pods
kubectl exec -it readonly-pod -- kubectl get services
kubectl exec -it readonly-pod -- kubectl get nodes
kubectl exec -it readonly-pod -- kubectl create deployment test --image=nginx  # Should fail

# Test admin service account permissions
kubectl exec -it admin-pod -- kubectl get deployments
kubectl exec -it admin-pod -- kubectl create deployment rbac-test --image=nginx
kubectl exec -it admin-pod -- kubectl get nodes  # Should fail

# Test app service account permissions
kubectl exec -it app-pod -- kubectl get configmap app-config
kubectl exec -it app-pod -- kubectl get secret app-secret
kubectl exec -it app-pod -- kubectl get pods  # Should fail
```

### Verification
```bash
kubectl get serviceaccounts
kubectl get roles
kubectl get rolebindings
kubectl describe role pod-reader
kubectl auth can-i get pods --as=system:serviceaccount:default:readonly-service-account
```

---

## Lab 8: Ingress Controllers and TLS Termination

### Objective
Set up ingress routing with SSL/TLS termination and path-based routing.

### Instructions

1. **Install NGINX Ingress Controller:**
```bash
# For minikube
minikube addons enable ingress

# Or install manually
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml
```

2. **Create multiple backend services:**
```yaml
# backend-services.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: web-content
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-app
  template:
    metadata:
      labels:
        app: api-app
    spec:
      containers:
      - name: api
        image: httpd:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/local/apache2/htdocs
      volumes:
      - name: html
        configMap:
          name: api-content
---
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api-app
  ports:
  - port: 80
    targetPort: 80
```

3. **Create content ConfigMaps:**
```yaml
# content-configmaps.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>Web Application</title></head>
    <body>
      <h1>Welcome to Web App</h1>
      <p>This is the main web application.</p>
    </body>
    </html>
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-content
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head><title>API Service</title></head>
    <body>
      <h1>API Service</h1>
      <p>This is the API backend service.</p>
    </body>
    </html>
```

4. **Generate TLS certificates:**
```bash
# Create self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout myapp.key -out myapp.crt \
  -subj "/CN=myapp.local/O=myapp.local"

# Create TLS secret
kubectl create secret tls myapp-tls --key=myapp.key --cert=myapp.crt
```

5. **Create Ingress with path-based routing and TLS:**
```yaml
# ingress-config.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
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
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

6. **Create additional Ingress for different host:**
```yaml
# api-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE"
spec:
  rules:
  - host: api.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

7. **Deploy and test:**
```bash
# Apply all configurations
kubectl apply -f content-configmaps.yaml
kubectl apply -f backend-services.yaml
kubectl apply -f ingress-config.yaml
kubectl apply -f api-ingress.yaml

# Get ingress IP
kubectl get ingress
INGRESS_IP=$(kubectl get ingress myapp-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# For minikube, get the IP
minikube ip

# Add to /etc/hosts (replace with actual IP)
echo "$(minikube ip) myapp.local api.myapp.local" | sudo tee -a /etc/hosts

# Test HTTP (should redirect to HTTPS)
curl -v http://myapp.local

# Test HTTPS
curl -k https://myapp.local
curl -k https://myapp.local/api
curl -k https://api.myapp.local
```

8. **Advanced Ingress features:**
```yaml
# advanced-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: advanced-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
    nginx.ingress.kubernetes.io/limit-connections: "10"
    nginx.ingress.kubernetes.io/limit-rps: "5"
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "10"
spec:
  rules:
  - host: canary.myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

9. **Create basic auth secret:**
```bash
# Create htpasswd file
htpasswd -c auth admin
# Enter password when prompted

# Create secret
kubectl create secret generic basic-auth --from-file=auth
```

### Verification
```bash
kubectl get ingress
kubectl describe ingress myapp-ingress
kubectl get services -n ingress-nginx
curl -k https://myapp.local
```

---

## Lab 9: Monitoring and Observability with Prometheus and Grafana

### Objective
Set up comprehensive monitoring stack with metrics collection and visualization.

### Instructions

1. **Install Prometheus using Helm:**
```bash
# Add Prometheus Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set grafana.adminPassword=admin123
```

2. **Create sample application with metrics:**
```yaml
# metrics-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-metrics-app
  labels:
    app: sample-metrics-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: sample-metrics-app
  template:
    metadata:
      labels:
        app: sample-metrics-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: quay.io/prometheus/prometheus-example-app:latest
        ports:
        - containerPort: 8080
        env:
        - name: PORT
          value: "8080"
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: sample-metrics-service
  labels:
    app: sample-metrics-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
spec:
  selector:
    app: sample-metrics-app
  ports:
  - port: 8080
    targetPort: 8080
    name: metrics
```

3. **Create ServiceMonitor for Prometheus:**
```yaml
# service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: sample-app-monitor
  namespace: monitoring
  labels:
    app: sample-metrics-app
spec:
  selector:
    matchLabels:
      app: sample-metrics-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
  namespaceSelector:
    matchNames:
    - default
```

4. **Create custom PrometheusRule:**
```yaml
# prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: sample-app-rules
  namespace: monitoring
  labels:
    prometheus: kube-prometheus
    role: alert-rules
spec:
  groups:
  - name: sample-app.rules
    rules:
    - alert: HighRequestRate
      expr: rate(promhttp_metric_handler_requests_total[5m]) > 0.1
      for: 2m
      labels:
        severity: warning
      annotations:
        summary: "High request rate detected"
        description: "Request rate is {{ $value }} requests per second"
    
    - alert: PodCrashLooping
      expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
      for: 5m
      labels:
        severity: critical
      annotations:
        summary: "Pod {{ $labels.pod }} is crash looping"
        description: "Pod {{ $labels.pod }} in namespace {{ $labels.namespace }} is restarting frequently"
    
    - alert: HighMemoryUsage
      expr: (container_memory_working_set_bytes / container_spec_memory_limit_bytes) * 100 > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High memory usage in pod {{ $labels.pod }}"
        description: "Memory usage is {{ $value }}% in pod {{ $labels.pod }}"
```

5. **Create load generator for testing:**
```yaml
# load-generator.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-generator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-generator
  template:
    metadata:
      labels:
        app: load-generator
    spec:
      containers:
      - name: load-gen
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            wget -q -O- http://sample-metrics-service:8080/metrics
            sleep 1
          done
```

6. **Deploy monitoring components:**
```bash
# Deploy the sample application
kubectl apply -f metrics-app.yaml

# Wait for the app to be ready
kubectl get pods -l app=sample-metrics-app

# Deploy ServiceMonitor and PrometheusRule
kubectl apply -f service-monitor.yaml
kubectl apply -f prometheus-rules.yaml

# Deploy load generator
kubectl apply -f load-generator.yaml
```

7. **Access monitoring UIs:**
```bash
# Port forward to Prometheus
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090 &

# Port forward to Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 &

# Port forward to AlertManager
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-alertmanager 9093:9093 &
```

8. **Create custom Grafana dashboard:**
```yaml
# custom-dashboard.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  custom-dashboard.json: |
    {
      "dashboard": {
        "id": null,
        "title": "Sample App Metrics",
        "tags": ["kubernetes", "monitoring"],
        "style": "dark",
        "timezone": "browser",
        "panels": [
          {
            "id": 1,
            "title": "Request Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(promhttp_metric_handler_requests_total[5m])",
                "legendFormat": "Request Rate"
              }
            ],
            "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
          },
          {
            "id": 2,
            "title": "Pod Memory Usage",
            "type": "graph",
            "targets": [
              {
                "expr": "container_memory_working_set_bytes{pod=~\"sample-metrics-app.*\"}",
                "legendFormat": "{{ pod }}"
              }
            ],
            "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
          }
        ],
        "time": {"from": "now-1h", "to": "now"},
        "refresh": "5s"
      }
    }
```

9. **Test monitoring stack:**
```bash
# Check Prometheus targets
curl http://localhost:9090/api/v1/targets

# Query metrics
curl "http://localhost:9090/api/v1/query?query=up"

# Access Grafana (admin/admin123)
# Visit http://localhost:3000

# Check AlertManager
# Visit http://localhost:9093
```

### Verification
```bash
kubectl get pods -n monitoring
kubectl get servicemonitors -n monitoring
kubectl get prometheusrules -n monitoring
curl http://localhost:9090/api/v1/targets
```

---

## Lab 10: Advanced Troubleshooting and Debugging

### Objective
Master advanced troubleshooting techniques and debugging methods for Kubernetes applications.

### Instructions

1. **Create problematic applications for debugging:**
```yaml
# problematic-apps.yaml
# App with resource constraints
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-constrained-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: resource-constrained
  template:
    metadata:
      labels:
        app: resource-constrained
    spec:
      containers:
      - name: app
        image: nginx
        resources:
          requests:
            cpu: 50m
            memory: 32Mi
          limits:
            cpu: 100m
            memory: 64Mi
        env:
        - name: LARGE_VAR
          value: "$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 10000 | head -n 1)"
---
# App with failing health checks
apiVersion: apps/v1
kind: Deployment
metadata:
  name: failing-healthcheck-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: failing-healthcheck
  template:
    metadata:
      labels:
        app: failing-healthcheck
    spec:
      containers:
      - name: app
        image: nginx
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /nonexistent
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
---
# App with DNS issues
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dns-issue-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: dns-issue
  template:
    metadata:
      labels:
        app: dns-issue
    spec:
      containers:
      - name: app
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            nslookup nonexistent-service.default.svc.cluster.local
            sleep 10
          done
---
# App with volume mount issues
apiVersion: apps/v1
kind: Deployment
metadata:
  name: volume-issue-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: volume-issue
  template:
    metadata:
      labels:
        app: volume-issue
    spec:
      containers:
      - name: app
        image: nginx
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/conf.d
        - name: nonexistent-volume
          mountPath: /nonexistent
      volumes:
      - name: config-volume
        configMap:
          name: nonexistent-configmap
```

2. **Create debugging toolbox pod:**
```yaml
# debug-toolbox.yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-toolbox
spec:
  containers:
  - name: toolbox
    image: nicolaka/netshoot
    command: ['sleep', '3600']
    securityContext:
      capabilities:
        add: ["NET_ADMIN"]
    volumeMounts:
    - name: host-proc
      mountPath: /host/proc
      readOnly: true
    - name: host-sys
      mountPath: /host/sys
      readOnly: true
  volumes:
  - name: host-proc
    hostPath:
      path: /proc
  - name: host-sys
    hostPath:
      path: /sys
  hostNetwork: true
  hostPID: true
```

3. **Deploy problematic applications:**
```bash
kubectl apply -f problematic-apps.yaml
kubectl apply -f debug-toolbox.yaml
```

4. **Troubleshooting scenarios and techniques:**

**Scenario 1: Pod Startup Issues**
```bash
# Check pod status and events
kubectl get pods
kubectl describe pod <failing-pod-name>

# Check logs
kubectl logs <failing-pod-name>
kubectl logs <failing-pod-name> --previous

# Debug with ephemeral container (K8s 1.23+)
kubectl debug <failing-pod-name> -it --image=busybox --target=<container-name>

# Check resource usage
kubectl top pods
kubectl describe node
```

**Scenario 2: Network Debugging**
```bash
# Use debug toolbox for network troubleshooting
kubectl exec -it debug-toolbox -- bash

# Inside toolbox:
# Check DNS resolution
nslookup kubernetes.default.svc.cluster.local
nslookup <service-name>.<namespace>.svc.cluster.local

# Test connectivity
nc -zv <service-ip> <port>
curl -v http://<service-name>:<port>

# Check network policies
iptables -L
ss -tulpn

# Trace network packets
tcpdump -i any port 80
```

**Scenario 3: Resource and Performance Issues**
```bash
# Check resource usage
kubectl top pods --sort-by=cpu
kubectl top pods --sort-by=memory
kubectl top nodes

# Detailed resource analysis
kubectl describe pod <pod-name> | grep -A 5 -B 5 "Limits\|Requests"

# Check events for resource issues
kubectl get events --sort-by='.lastTimestamp'
kubectl get events --field-selector involvedObject.name=<pod-name>

# Analyze with metrics
kubectl exec -it debug-toolbox -- htop
kubectl exec -it debug-toolbox -- iotop
```

5. **Advanced debugging commands:**
```bash
# Debug pod creation issues
kubectl get pods -o wide
kubectl get pods -o yaml <pod-name>
kubectl describe replicaset <rs-name>

# Check scheduler issues
kubectl get events --field-selector reason=FailedScheduling

# Debug service discovery
kubectl get endpoints <service-name>
kubectl describe service <service-name>

# Debug persistent volume issues
kubectl get pv,pvc
kubectl describe pv <pv-name>
kubectl describe pvc <pvc-name>

# Check RBAC issues
kubectl auth can-i <verb> <resource> --as=<user>
kubectl auth can-i --list --as=<user>

# Debug ingress issues
kubectl describe ingress <ingress-name>
kubectl get ingress -o yaml <ingress-name>
```

6. **Create debugging scripts:**
```bash
# debug-script.sh
#!/bin/bash

echo "=== Kubernetes Debugging Report ==="
echo "Date: $(date)"
echo

echo "=== Cluster Info ==="
kubectl cluster-info
kubectl version --short
echo

echo "=== Node Status ==="
kubectl get nodes -o wide
kubectl top nodes
echo

echo "=== Pod Status ==="
kubectl get pods --all-namespaces -o wide
echo

echo "=== Recent Events ==="
kubectl get events --sort-by='.lastTimestamp' | tail -20
echo

echo "=== Resource Usage ==="
kubectl top pods --all-namespaces --sort-by=cpu | head -10
echo

echo "=== Failed Pods ==="
kubectl get pods --all-namespaces --field-selector=status.phase=Failed
echo

echo "=== Pending Pods ==="
kubectl get pods --all-namespaces --field-selector=status.phase=Pending
```

7. **Performance testing and analysis:**
```bash
# Create load testing pod
kubectl run load-test --image=busybox -it --rm -- sh

# Inside load test pod:
while true; do
  time wget -qO- http://<service-name>/
done

# Monitor during load test
kubectl top pods -w
kubectl get hpa -w
```

8. **Log analysis techniques:**
```bash
# Aggregate logs from multiple pods
kubectl logs -l app=<app-name> --tail=100

# Follow logs in real-time
kubectl logs -f deployment/<deployment-name>

# Get logs with timestamps
kubectl logs <pod-name> --timestamps

# Search logs for errors
kubectl logs <pod-name> | grep -i error
kubectl logs <pod-name> | grep -E "(error|exception|fail)"

# Export logs for analysis
kubectl logs <pod-name> > pod-logs.txt
```

### Verification and Cleanup
```bash
# Check debugging tools
kubectl get pods debug-toolbox
kubectl exec -it debug-toolbox -- which tcpdump curl nslookup

# Clean up problematic apps
kubectl delete deployment resource-constrained-app failing-healthcheck-app dns-issue-app volume-issue-app
kubectl delete pod debug-toolbox

# Verify cleanup
kubectl get pods
```

---

## Lab Completion Summary

Congratulations! You have completed all 10 intermediate Kubernetes labs. Here's what you've learned:

1. **Advanced Deployments** - Rolling updates, rollbacks, and deployment strategies
2. **Configuration Management** - ConfigMaps, Secrets, and environment variables
3. **Persistent Storage** - StatefulSets, PVs, PVCs, and storage classes
4. **Network Security** - Network policies and service mesh basics
5. **Auto-scaling** - HPA, VPA, and resource-based scaling
6. **Batch Processing** - Jobs, CronJobs, and Init Containers
7. **Security** - RBAC, Service Accounts, and access control
8. **Ingress** - Load balancing, TLS termination, and routing
9. **Monitoring** - Prometheus, Grafana, and observability
10. **Troubleshooting** - Advanced debugging and performance analysis

### Next Steps

- Practice these labs multiple times with different scenarios
- Explore advanced topics like Operators, Custom Resources, and Helm
- Set up production-grade clusters with proper security and monitoring
- Learn about GitOps workflows with ArgoCD or Flux
- Study for Kubernetes certifications (CKA, CKAD, CKS)

### Additional Resources

- Official Kubernetes Documentation: https://kubernetes.io/docs/
- Kubernetes Patterns: https://k8spatterns.io/
- CNCF Landscape: https://landscape.cncf.io/
- Kubernetes Best Practices: https://kubernetes.io/docs/concepts/configuration/overview/