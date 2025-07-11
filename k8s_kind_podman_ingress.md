# Learning Environment Preparation

Follow these steps to set up your Kubernetes learning environment with all required tools.

## 1. Install Podman

- [Podman Desktop Downloads](https://podman-desktop.io/downloads)
- [Podman Installation Guide](https://podman.io/docs/installation)

## 2. Install Kind (Kubernetes IN Docker/Podman)

- [Kind Quick Start Guide](https://kind.sigs.k8s.io/docs/user/quick-start)
- Note: Kind can run with Docker or Podman as the container runtime.

## 3. Install kubectl (Kubernetes CLI)

- [kubectl Installation Instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

## 4. Install Helm (Kubernetes Package Manager)

- [Helm Installation Guide](https://helm.sh/docs/intro/install/)

## 5. Install freelens (Kubernetes IDE)

- [freelens GitHub Repository](https://github.com/freelensapp/freelens?tab=readme-ov-file#windows)

## Prerequisites

- [Install kind](https://kind.sigs.k8s.io/)
- [Ingress documentation for kind](https://kind.sigs.k8s.io/docs/user/ingress/)
- [Install Metrics Server on kind](https://maggnus.com/install-metrics-server-on-the-kind-kubernetes-cluster-12b0a5faf94a)
- [Example Gist: Metrics Server on kind](https://gist.github.com/sanketsudake/a089e691286bf2189bfedf295222bd43)


# Create Cluster

```bash
export KIND_EXPERIMENTAL_PROVIDER=podman
```

Create a kind cluster with extraPortMappings and node-labels.

    extraPortMappings allow the local host to make requests to the Ingress controller over ports 80/443
    node-labels only allow the ingress controller to run on a specific node(s) matching the label selector

```bash
cat <<EOF | kind create cluster --name k8s-playground --config=-
# three node (two workers) cluster config
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
EOF
```

# Create Ingress NGINX

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
```

```bash
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

Using Ingress

Note, this example uses an nginx-specific Ingress annotation which may not be supported by all Ingress implementations.

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: foo-app
  labels:
    app: foo
spec:
  containers:
  - command:
    - /agnhost
    - netexec
    - --http-port
    - "8080"
    image: registry.k8s.io/e2e-test-images/agnhost:2.39
    name: foo-app
---
kind: Service
apiVersion: v1
metadata:
  name: foo-service
spec:
  selector:
    app: foo
  ports:
  # Default port used by the image
  - port: 8080
---
kind: Pod
apiVersion: v1
metadata:
  name: bar-app
  labels:
    app: bar
spec:
  containers:
  - command:
    - /agnhost
    - netexec
    - --http-port
    - "8080"
    image: registry.k8s.io/e2e-test-images/agnhost:2.39
    name: bar-app
---
kind: Service
apiVersion: v1
metadata:
  name: bar-service
spec:
  selector:
    app: bar
  ports:
  # Default port used by the image
  - port: 8080
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - http:
      paths:
      - pathType: ImplementationSpecific
        path: /foo(/|$)(.*)
        backend:
          service:
            name: foo-service
            port:
              number: 8080
      - pathType: ImplementationSpecific
        path: /bar(/|$)(.*)
        backend:
          service:
            name: bar-service
            port:
              number: 8080
---
```

```bash
kubectl apply -f https://kind.sigs.k8s.io/examples/ingress/usage.yaml
```
Now verify that the ingress works

```bash
# should output "foo-app"
curl localhost/foo/hostname
# should output "bar-app"
curl localhost/bar/hostname
```


# Install Metrics Server

# Install Prometheus

https://medium.com/@giorgiodevops/kind-install-prometheus-operator-and-fix-missing-targets-b4e57bcbcb1f

## Kind configuration
```yaml
# File name cluster.yaml
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
name: prometheus-cluster
kubeadmConfigPatches:
- |-
  kind: ClusterConfiguration
  # configure controller-manager bind address
  controllerManager:
    extraArgs:
      bind-address: 0.0.0.0 #Disable localhost binding
      secure-port: "0"      #Disable the https 
      port: "10257"         #Enable http on port 10257
  # configure etcd metrics listen address
  etcd:
    local:
      extraArgs:
        listen-metrics-urls: http://0.0.0.0:2381
  # configure scheduler bind address
  scheduler:
    extraArgs:
      bind-address: 0.0.0.0  #Disable localhost binding
      secure-port: "0"       #Disable the https
      port: "10259"          #Enable http on port 10259
- |-
  kind: KubeProxyConfiguration
  # configure proxy metrics bind address
  metricsBindAddress: 0.0.0.0
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
```
## install Prometheus operator using the helm chart repository https://github.com/prometheus-operator/prometheus-operator 

```bash
helm install --wait --timeout 15m \
   --namespace monitoring --create-namespace \
   --repo https://prometheus-community.github.io/helm-charts \
   kube-prometheus-stack kube-prometheus-stack --values - <<EOF 
 kubeEtcd:
   service:
     targetPort: 2381
 kubeControllerManager:  
   service:
     targetPort: 10257
 kubeScheduler:
   service:
     targetPort: 10259
EOF
```

## When the operator has been deployed just proxy the port `9090`
```bash
$ kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```
and check the url http://localhost:9090/target