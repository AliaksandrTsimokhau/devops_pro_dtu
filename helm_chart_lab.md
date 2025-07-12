# LAB: Create Your First Helm Chart

This lab will guide you through creating, deploying, and sharing your first Helm chart for Kubernetes.

---

## Overview

A typical cloud-native application with a 3-tier architecture consists of:

- **Frontend** (e.g., web UI) - a presentation tier
- **Backend** (e.g., API server) - an application tier (business logic)
- **Database** (e.g., MySQL, PostgreSQL) - a data tier (backend)

This architecture promotes scalability, flexibility, and maintainability, which are key characteristics of cloud-native applications.

Below is a diagram illustrating how these tiers interact in Kubernetes, managed by Helm:

```mermaid
flowchart LR
    Client[Client (Frontend)]
    Server[Server (Backend)]
    Database[Database]

    Client --> Server --> Database
```

Helm charts package these Kubernetes objects (Deployments, Services, ConfigMaps, Secrets) together, enabling easy deployment, configuration, and management of the entire application stack.

---

## Use Cases for Helm

- Find and use popular software packaged as Kubernetes charts
- Share your own applications as Kubernetes charts
- Create reproducible builds of your Kubernetes applications
- Intelligently manage your Kubernetes object definitions
- Manage releases of Helm packages

---

# Basic approach. The easiest one.

## Using Existing Kubernetes YAML Files with Helm

If you already have working Kubernetes YAML files, you can use them as the foundation for your first Helm chart. The simplest Helm chart can be created by copying your existing `deployment.yaml` and `service.yaml` files into the `templates/` directory of your chart—no templating required.

> **Tip:** For Helm installation instructions, see the [Helm GitHub repository](https://github.com/helm/helm#install).

### Basic Chart: Copy and Use Your YAML

1. **Copy your YAML files:**  
    Place your `deployment.yaml` and `service.yaml` into the `templates/` directory of your Helm chart.

2. **Install the chart:**  
    You can now install your chart as-is, and Helm will deploy your original resources.

This approach works, but the real power of Helm comes from using template variables.

---

## Customizing with Template Variables

Helm templates allow you to inject values and customize your deployments. For example, a generated `deployment.yaml` might look like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ template "chart.fullname" . }}
  labels:
     app: {{ template "chart.name" . }}
     chart: {{ template "chart.chart" . }}
     release: {{ .Release.Name }}
     heritage: {{ .Release.Service }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
     matchLabels:
        app: {{ template "chart.name" . }}
```

### Types of Template Variables

- `{{ template "chart.fullname" . }}`: Values from `Chart.yaml` or helper templates.
- `{{ .Release.Name }}`: Built-in release information (set by Helm).
- `{{ .Values.replicaCount }}`: Values from `values.yaml` (user-configurable).

You can edit `Chart.yaml` for chart metadata, add or change values in `values.yaml`, and use built-in objects like `.Release` for dynamic information.

For more advanced templating features, see the [Helm Template Guide](https://helm.sh/docs/chart_template_guide/).

---
Now that you understand the basics, let’s walk through creating a demo Helm chart and customizing it step by step.
## Step 1: Generate Your First Chart

Use the `helm create` command to scaffold a new chart named `mychart`:

```sh
helm create mychart
```

This creates a directory structure:

```
mychart/
├── Chart.yaml
├── charts/
├── templates/
│   ├── NOTES.txt
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── ingress.yaml
│   └── service.yaml
└── values.yaml
```

### Templates

The `templates/` directory contains YAML definitions for your Kubernetes objects, rendered using Go templating. For example, `service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
    name: {{ template "fullname" . }}
    labels:
        chart: "{{ .Chart.Name }}-{{ .Chart.Version | replace "+" "_" }}"
spec:
    type: {{ .Values.service.type }}
    ports:
        - port: {{ .Values.service.externalPort }}
            targetPort: {{ .Values.service.internalPort }}
            protocol: TCP
            name: {{ .Values.service.name }}
    selector:
        app: {{ template "fullname" . }}
```

To inspect the generated YAML, run:

```sh
helm install mychart --dry-run --debug ./mychart
```

### Values

The `.Values` object exposes configuration set at deploy-time. Defaults are in `values.yaml`. Change `service.internalPort` and run a dry-run to see the effect:

```sh
helm install mychart --dry-run --debug ./mychart --set service.internalPort=8080
```

### Helpers and Functions

Templates can use partials from `_helpers.tpl` and functions like `replace`. See the [Helm documentation](https://helm.sh/docs/chart_template_guide/) for more.

### Documentation

`NOTES.txt` in `templates/` is printed after deployment and can include templated instructions.

### Metadata

`Chart.yaml` contains chart metadata, constraints, and versioning.

---

## Step 2: Deploy Your First Chart

Deploy the chart, exposing NGINX via a NodePort:

```sh
helm install example ./mychart --set service.type=NodePort
```

After deployment, follow the NOTES to access the application:

```sh
export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services example-mychart)
export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT/
```

---

## Step 3: Modify Chart to Deploy a Custom Service

Update `values.yaml` to use a todo list application:

```yaml
image:
    repository: prydonius/todo
    tag: 1.0.0
    pullPolicy: IfNotPresent
```

Lint your chart:

```sh
helm lint ./mychart
```

Deploy the updated chart:

```sh
helm install example2 ./mychart --set service.type=NodePort
```

Access the application using the NOTES instructions.

---

## Step 4: Package and Share Your Chart

Package your chart:

```sh
helm package ./mychart
```

Install from the package:

```sh
helm install example3 mychart-0.1.0.tgz --set service.type=NodePort
```

### Repositories

Run a local repository:

```sh
helm serve
```

Search and install from the local repo:

```sh
helm search local
helm install example4 local/mychart --set service.type=NodePort
```

---

## Step 5: Add Dependencies

Add a dependency (e.g., MariaDB) in `requirements.yaml`:

```yaml
dependencies:
    - name: mariadb
        version: 0.6.0
        repository: https://charts.helm.sh/stable
```

Update dependencies:

```sh
helm dep update ./mychart
```

Install the chart with dependencies:

```sh
helm install example5 ./mychart --set service.type=NodePort
```

---

Congratulations! You have created, deployed, and shared your first Helm chart, including adding dependencies.
