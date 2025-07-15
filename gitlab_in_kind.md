# GitLab Kubernetes with KinD

> Source: [GitLab Docs - KinD](https://docs.gitlab.com/charts/development/kind/)

This guide helps you set up a local Kubernetes development environment using [KinD](https://kind.sigs.k8s.io/). KinD runs Kubernetes clusters in Docker containers, making it easy to test different versions and multi-node setups.

We’ll use [nip.io](https://nip.io/) for DNS, which maps IP addresses to hostnames (e.g., `192.168.1.250.nip.io` → `192.168.1.250`). No installation is needed.

> **Note:** With SSL-enabled installs, use HTTPS for Git operations. SSH support via NodePorts is planned.

---

## Apple Silicon (M1/M2) Support

KinD works with [Colima](https://github.com/abiosoft/colima) for local Kubernetes on macOS (including M1/M2).

### Prerequisites

- macOS 13+ (Ventura)
- [Colima](https://github.com/abiosoft/colima)
- Rosetta (for x86_64 emulation):

```sh
softwareupdate --install-rosetta
```

### Start the Colima VM

```sh
colima start --cpu 6 --memory 16 --disk 40 --profile docker --arch aarch64 --vm-type=vz --vz-rosetta
```

### Manage the VM

- **Stop:**  
    `colima stop --profile docker`
- **Start:**  
    `colima start --profile docker`
- **Delete:**  
    `colima delete --profile docker`

---

## Preparation

### Find Your Host IP

- **Linux:**  
    `hostname -i`
- **macOS:**  
    `ipconfig getifaddr en0`  
    *(Replace `en0` if your primary interface differs.)*

### Use Namespaces

Create a namespace before installing GitLab:

```sh
kubectl create namespace YOUR_NAMESPACE
```

Add `--namespace YOUR_NAMESPACE` to future `kubectl` commands, or use [`kubens`](https://github.com/ahmetb/kubectx) to switch context.

### Install Dependencies

Use [asdf](https://asdf-vm.com/) or your preferred method to install:

- `kubectl`
- `helm`
- `kind`
- [Docker](https://www.docker.com/)

---

## Get Example Configurations

Clone the GitLab charts repository:

```sh
git clone https://gitlab.com/gitlab-org/charts/gitlab.git
```

---

## Spin Up the KinD Cluster

Review and adjust example configs in `examples/kind/`. Create a cluster:

```sh
kind create cluster --config examples/kind/kind-ssl.yaml
```

---

## Add GitLab Helm Chart

```sh
helm repo add gitlab https://charts.gitlab.io/
helm repo update
```

---

## Deployment Options

### 1. NGINX Ingress NodePort with SSL

Expose NGINX NodePorts with SSL:

```sh
kind create cluster --config examples/kind/kind-ssl.yaml
helm upgrade --install gitlab gitlab/gitlab \
    --set global.hosts.domain=(your host IP).nip.io \
    -f examples/kind/values-base.yaml \
    -f examples/kind/values-ssl.yaml
```

Access GitLab at:  
`https://gitlab.(your host IP).nip.io`

#### (Optional) Trust the Root CA

Download and trust the self-signed root CA:

```sh
kubectl get secret gitlab-wildcard-tls-ca -ojsonpath='{.data.cfssl_ca}' | base64 --decode > gitlab.(your host IP).nip.io.ca.pem
```

Follow your OS instructions to add the CA to your trusted chain.

> For Docker registry login with self-signed certs, see:  
> - [Run an externally-accessible registry](https://docs.gitlab.com/ee/administration/packages/container_registry.html#run-an-externally-accessible-registry)
> - [Add self-signed registry certificates to Docker](https://docs.gitlab.com/ee/administration/packages/container_registry.html#add-self-signed-registry-certificates-to-docker)

---

### 2. NGINX Ingress NodePort without SSL

Expose NGINX NodePorts without SSL:

```sh
kind create cluster --config examples/kind/kind-no-ssl.yaml
helm upgrade --install gitlab gitlab/gitlab \
    --set global.hosts.domain=(your host IP).nip.io \
    -f examples/kind/values-base.yaml \
    -f examples/kind/values-no-ssl.yaml
```

Access GitLab at:  
`http://gitlab.(your host IP).nip.io`

> For Docker registry login, configure Docker to trust your insecure registry.

---

## Handling DNS

If you can't access `nip.io`, see the [minikube DNS docs](https://minikube.sigs.k8s.io/docs/handbook/accessing/#using-nipio-or-xipio) for alternatives.  
When editing `/etc/hosts`, use your host's IP (not `$(minikube ip)`).

---

## Cleaning Up

Delete your KinD cluster:

```sh
kind delete cluster
```

For named or multiple clusters, use `--name`.

---

## Installing GitLab Runner with Helm

See [GitLab Runner on Kubernetes](https://docs.gitlab.com/runner/install/kubernetes/) for full details.
## Create a GitLab Project Runner

To create a project runner in your GitLab project:

1. In the left sidebar, select **Search or go to** and find your project.
2. Go to **Settings > CI/CD**.
3. Expand the **Runners** section.
4. Click **New project runner**.
5. Choose your operating system.
6. In the **Tags** section, select the **Run untagged** checkbox if you want the runner to pick up untagged jobs (optional).
7. Click **Create runner**.

This will generate a registration token to use when configuring your GitLab Runner.

1. **Prepare your `values.yaml`**  
   - Set `gitlabUrl` to your GitLab instance URL (e.g., `https://gitlab.<minikube-ip>.nip.io`).
   - Set `rbac.create: true` to allow the runner to create pods.
   - `values.yaml` content:
     ```yaml
     rbac:
       create: true
     runners:
        privileged: true
     gitlabUrl: https://gitlab.<minikube-ip>.nip.io
     runnerRegistrationToken: "glrt--xxxxx"
     ```
   - Set `runnerToken` to the registration token from your GitLab UI.

2. **Add the GitLab Helm repository:**
    ```sh
    helm repo add gitlab https://charts.gitlab.io
    helm repo update gitlab
    ```

3. **Check available GitLab Runner versions:**
    ```sh
    helm search repo -l gitlab/gitlab-runner
    ```

4. **Install GitLab Runner:**
    - For Helm 3:
      ```sh
      helm install --namespace gitlabrunner --create-namespace gitlab-runner -f <CONFIG_VALUES_FILE> gitlab/gitlab-runner
      ```
    - Replace `<NAMESPACE>` with your Kubernetes namespace and `<CONFIG_VALUES_FILE>` with your `values.yaml` path.

    ### Verifying Your GitLab Runner Installation

    After installing the GitLab Runner, verify that everything is working as expected:

    1. **Check the Helm release:**
        ```sh
        helm --namespace gitlabrunner get all gitlab-runner
        ```

    2. **List all resources in the namespace:**
        ```sh
        kubectl -n gitlabrunner get all
        ```

    3. **View logs for the GitLab Runner Pod:**
        ```sh
        kubectl -n gitlabrunner logs -f pod/<gitlab-runner-pod-name>
        ```
        Replace `<gitlab-runner-pod-name>` with the actual pod name from the previous command.

    4. **Verify the runner is registered in GitLab:**
        - Go to your GitLab project.
        - Navigate to **Settings > CI/CD > Runners**.
        - Confirm that your new runner appears in the list.

    ---

5. **Upgrade GitLab Runner:**
    ```sh
    helm upgrade --namespace gitlabrunner -f <CONFIG_VALUES_FILE> <RELEASE-NAME> gitlab/gitlab-runner
    ```
    - Add `--version <RUNNER_HELM_CHART_VERSION>` to specify a version.

6. **Uninstall GitLab Runner:**
    ```sh
    helm delete --namespace <NAMESPACE> <RELEASE-NAME>
    ```

Refer to the [GitLab Runner Helm Chart documentation](https://docs.gitlab.com/runner/install/kubernetes.html) for advanced configuration and troubleshooting.
