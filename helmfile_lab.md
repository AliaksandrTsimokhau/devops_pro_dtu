
# LAB: Create a Repository to Deploy Helm Charts Using Helmfile

## Lab Requirements

- `kubectl`
- `helm`
- Existing or new Helm chart(s)
- Kubernetes cluster (local: kind, microk8s, minikube, or managed: GKE, EKS, AKS, etc.)
- GitLab server

**Estimated time:** 1h 30min

---

## 1. Create a GitLab Repository

Create a new repository in GitLab for your Helmfile-based deployment setup.

### Instructions to Create a GitLab Repository

1. Log in to your GitLab account.
2. Click the **"New project"** button (usually found on the dashboard or under the Projects menu).
3. Choose **"Create blank project"**.
4. Enter a **Project name** (e.g., `release-helmfile`).
5. Optionally, add a description and set the visibility (Private, Internal, or Public).
6. Click **"Create project"**.
7. Clone the repository to your local machine:

    ```sh
    git clone https://<your-gitlab-hostname>/<your-namespace>/<your-repo>.git
    cd <your-repo>
    ```

8. Start adding your files and commit your changes.

---

## 2. Repository Structure

Design your repository with logical abstraction layers for releases and values.

<details>
<summary>Example structure</summary>

```
release-helmfile/
├── ci/                  # GitLab CI YAML files and scripts
├── charts/              # Local Helm charts for development
├── release/             # Values for releases, organized by logical tiers
│   ├── common-tier/
│   │   ├── release-name1/
│   │   ├── release-name2/
│   │   └── helmfile.yaml
│   ├── another-tier/
│   │   ├── release-name1/
│   │   ├── release-name2/
│   │   └── helmfile.yaml
│   └── common.yaml      # Common Helmfile settings and values paths
├── .gitignore
├── .gitlab.yaml         # Main GitLab pipeline file
├── environments.yaml    # Environment-specific settings and feature flags
├── helmfile.yaml        # Main Helmfile referencing logical tiers
└── README.md
```
</details>

**Requirement:** Create at least two abstraction layers and deploy at least two charts (one local, one from an artifact registry).

---

## 3. Add Helm Charts

- Place an existing or new Helm chart in the `charts/` folder. you could resuse chart from lab `gitlab_ci_lab_helm.md`

#! give an example of release description one for local and for external
---

## 4. Create Helmfile for Each Logical Tier

- In each logical tier folder (e.g., `common-tier`, `another-tier`), create a `helmfile.yaml`.
- Add release descriptions for:
    - A local chart (from `charts/`)
    - An external chart (from a remote registry)

**Example:**

```yaml
# release/common-tier/helmfile.yaml
releases:
    - name: my-local-app
        chart: ../../charts/my-local-app
        values:
            - values.yaml

    - name: nginx-ingress
        chart: stable/nginx-ingress
        version: 4.0.6
        values:
            - values-nginx.yaml
```

---

## 5. Reference Logical Tiers in the Main Helmfile

- In the root `helmfile.yaml`, reference all logical tier helmfiles.

**Example:**

```yaml
# helmfile.yaml
helmfiles:
    - path: release/common-tier/helmfile.yaml
    - path: release/another-tier/helmfile.yaml
```

---

## 6. Define Common Settings in `common.yaml`

- Add remote registry aliases and describe all values paths for templating.
- Paths can be templated; missing files will be skipped.

**Example:**

```yaml
# release/common.yaml
repositories:
    - name: stable
        url: https://charts.helm.sh/stable

valuesPaths:
    - "{{ .Release.Name }}/values.yaml"
    - "common-values.yaml"
```

---

## 7. Define Environments in `environments.yaml`

- Describe environments, values, feature flags, etc.

**Example:**

```yaml
# environments.yaml
environments:
    dev:
        kubeContext: kind-dev
        clusterName: dev-cluster
        featureFlags:
            enableFeatureX: true
    prod:
        kubeContext: prod-context
        clusterName: prod-cluster
        featureFlags:
            enableFeatureX: false
```

---

## 8. Create GitLab Pipeline in `.gitlab.yaml`

- Define pipeline stages: scan, validate, plan, deploy.
- Store reusable parts in `ci/` and include them.
- Use CI/CD variables for secrets (e.g., registry, cluster auth).

**Example:**

```yaml
# .gitlab.yaml
stages:
    - scan
    - validate
    - plan
    - deploy

include:
    - local: 'ci/before_script.yaml'

scan:
    stage: scan
    script:
        - trivy config .

validate:
    stage: validate
    script:
        - helm lint charts/*

plan:
    stage: plan
    script:
        - helmfile diff

deploy:
    stage: deploy
    script:
        - helmfile apply
    environment:
        name: $CI_ENVIRONMENT_NAME
```

---

## 9. Add `.gitignore`

- Ignore files and folders as needed (e.g., `.DS_Store`, `*.tgz`, etc.).

---

## 10. Document Your Repository

- In `README.md`, describe the repo's purpose, structure, and usage instructions.
- Include any setup or developer notes.

---

