# Helm Installation — Notes

## 1. Concept / What

Helm installation refers to setting up the Helm CLI (`helm`) on your machine. The CLI is required to install charts, render templates, manage releases, and deploy applications to Kubernetes/EKS.

---

## 2. Why / Purpose / Real Use Case

* Helm acts as a **package manager for Kubernetes**.
* Installing Helm enables:

  * Deploying applications using versioned charts
  * Upgrading or rolling back releases
  * Rendering Kubernetes YAML before deployment (`helm template`)
  * Automating EKS deployments in CI/CD
* In real-world teams, Helm is installed on developer laptops, build servers, and CI/CD runners to deploy microservices into EKS clusters.

---

## 3. How it Works / Steps / Syntax

### Step 1 — Download Helm (Linux example)

```bash
curl -LO https://get.helm.sh/helm-v3.13.2-linux-amd64.tar.gz
tar -zxvf helm-v3.13.2-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
```

### Step 2 — Verify Installation

```bash
helm version
```

### Step 3 — Ensure Kubernetes Access

Helm relies on your kubeconfig; it does not bypass permissions.

For EKS:

```bash
aws eks update-kubeconfig --name <cluster-name> --region <region>
kubectl get nodes
```

### Step 4 — Windows Installation

```powershell
choco install kubernetes-helm
```

### Step 5 — Note About Helm v3

* No Tiller component required (unlike Helm 2).
* Only Helm CLI + kubeconfig is needed.

---

## 4. Common Issues / Errors

| Issue                                           | Description                                           |
| ----------------------------------------------- | ----------------------------------------------------- |
| `helm: command not found`                       | Binary not moved into PATH                            |
| Wrong Helm version                              | Downloaded incompatible architecture (arm64 vs amd64) |
| Cannot connect to Kubernetes                    | kubeconfig not configured or missing permissions      |
| `x509: certificate signed by unknown authority` | Cluster CA or local certificate trust issue           |
| Helm commands fail in CI                        | Missing kubeconfig or AWS auth not configured         |

---

## 5. Troubleshooting / Fixes

### Fix PATH issues

```bash
sudo mv linux-amd64/helm /usr/local/bin/
which helm
```

### Fix kubeconfig issues (EKS)

```bash
aws eks update-kubeconfig --name <cluster> --region <region>
kubectl auth can-i create deployments
```

### Fix version or architecture mismatch

Download correct binary:

* `linux-amd64`
* `linux-arm64`
* `windows-amd64`

### Validate Helm functionality

```bash
helm version --debug
helm help
```

### Simulate installation without deploying

```bash
helm install --dry-run --debug test ./chart
```

---

## 6. Best Practices

* Always verify Helm version during installation (`helm version`).
* Use the latest stable Helm release unless restricted by organization policy.
* Configure EKS kubeconfig immediately after installation.
* Store the Helm installation steps in onboarding/internal documentation.
* In CI/CD, install Helm using scripts pinned to a specific version for reproducibility.
* Avoid using system package managers that install outdated versions.

---
---

# Adding Helm Repositories — Notes

## 1. Concept / What

A **Helm repository** is a location (URL) that stores packaged Helm charts. When you run `helm repo add`, you register that repository with Helm so it knows where to look for charts. However, adding a repo alone does not download chart metadata.

`helm repo update` is used to fetch the latest metadata (index.yaml) from all added repositories.

---

## 2. Why / Purpose / Real Use Case

* Helm repositories provide ready-made charts for applications like NGINX, Redis, Prometheus, etc.
* Teams use public repos (Bitnami, Grafana) or private internal repos.
* Artifact Hub is used to **discover** repository URLs, while the actual charts live in the repos.
* Updating repositories ensures Helm has the **latest chart versions** and metadata.

Real-world example: Before installing Bitnami's NGINX chart into EKS, you add the Bitnami repo, update metadata, and then install the chart.

---

## 3. How it Works / Steps / Syntax

### Step 1 — Add a repository

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

This stores the repo URL in Helm's local config (`~/.config/helm/repositories.yaml`).

### Step 2 — Update repositories

```bash
helm repo update
```

This downloads each repository's `index.yaml`, which lists:

* Available chart names
* Versions
* Descriptions
* Chart metadata

### Step 3 — List repositories

```bash
helm repo list
```

### Step 4 — Search for charts

```bash
helm search repo nginx
```

### Step 5 — Install a chart

```bash
helm install my-nginx bitnami/nginx
```

---

## 4. Common Issues / Errors

| Issue                       | Description                                |
| --------------------------- | ------------------------------------------ |
| Chart not found             | `helm repo update` not run, stale metadata |
| URL not a valid repo        | Wrong or deprecated URL                    |
| No such host                | Network or DNS issue                       |
| Old chart version installed | Repo metadata outdated                     |
| Trying old "stable" repo    | Deprecated repository                      |

---

## 5. Troubleshooting / Fixes

### Fix 1 — Update repositories

```bash
helm repo update
```

### Fix 2 — Remove and re-add repo

```bash
helm repo remove bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami
```

### Fix 3 — Verify connectivity

Check DNS and internet access when URL fails.

### Fix 4 — Validate repo entry

```bash
helm repo list
```

### Fix 5 — Inspect repo metadata

```bash
helm search repo <chart-name>
```

---

## 6. Best Practices

* Always run `helm repo update` after adding any repository.
* Update repos before every install/upgrade in CI/CD.
* Use trusted repos (Bitnami, Grafana, Prometheus).
* Discover repo URLs using Artifact Hub, not deprecated stable repos.
* Document repository URLs inside your project.
* Re-add repos if metadata gets corrupted.

---
---

# Helm Repo Update & Real‑World Helm Repository Usage — Notes

## 1. Concept / What

A **Helm repository** is a storage location that contains **packaged Helm charts (.tgz files)** along with an **index.yaml** file. Helm uses this `index.yaml` to understand what charts and chart versions are available in that repository.

The command:

```bash
helm repo update
```

fetches the latest `index.yaml` from all added repositories so Helm always has fresh metadata. Without updating, Helm cannot reliably find or install charts.

Organizations can choose between:

* Using **raw Helm charts** from source control (no repo needed)
* Storing **.tgz files only** in storage (not a real repo)
* Hosting a **proper Helm repository** (S3, GitHub Pages, Nexus, Artifactory, Harbor)

---

## 2. Why / Purpose / Real Use Case

### Why do we use `helm repo update`?

When we add a repository:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Helm **only stores the URL** locally. It does **not** download information about charts.

`helm repo update` downloads the latest `index.yaml` for the repo, which tells Helm:

* Which charts this repo has
* What versions exist
* What metadata the charts contain
* Where to download chart packages from

If this index file is not updated, Helm:

* Cannot find charts
* Cannot see new versions
* May install outdated versions
* May throw chart‑not‑found errors

### Real‑world usage patterns

**Most companies DO NOT maintain a Helm repository.** Instead, they:

* Keep raw Helm charts in their Git repo
* Deploy using `helm install ./chart`

Some companies package charts using:

```bash
helm package ./chart
```

and store `.tgz` files in places like S3 or GitHub Releases **without** creating `index.yaml`. This is simply file storage, **not a Helm repository**.

Only mature DevOps/GitOps setups maintain full Helm repositories. These include `.tgz` files **plus** `index.yaml`, generated with:

```bash
helm repo index .
```

Then teams use:

```bash
helm repo add
helm repo update
helm install repo/chart
```

Example real scenario:

* Application Docker images stored in ECR
* Helm charts stored as raw folder inside repo OR packaged and stored in S3
* Deploy to EKS using `helm install` or `helm upgrade`
* No need for repo add/update unless using a proper Helm repository

---

## 3. How it Works / Steps / Syntax

### A. When using **raw Helm charts** (MOST COMMON)

```
helm install app ./chart
helm upgrade app ./chart
```

✔ No `helm package`
✔ No `helm repo add`
✔ No `helm repo update`
✔ No `index.yaml`

### B. When packaging charts (NOT a repo by default)

```
helm package ./chart
```

Produces:

```
chart-1.0.0.tgz
```

You can store this file anywhere (S3, GitHub, etc.). Installing:

```
helm install app ./chart-1.0.0.tgz
```

✔ Still no repo add/update required
✔ Still not a Helm repository

### C. Creating a REAL Helm repository

A location becomes a Helm repository **only if it has:**

* Chart `.tgz` files
* An automatically generated `index.yaml`

Generate it:

```bash
helm repo index /path/to/folder
```

Upload both `.tgz` and `index.yaml` to a storage endpoint.

Client usage:

```bash
helm repo add myrepo https://mycompany.com/helm
helm repo update
helm install app myrepo/chart
```

### D. What happens during `helm repo update`?

Helm downloads all repository `index.yaml` files to:

```
~/.cache/helm/repository/
```

This lets Helm know all available charts and versions.

---

## 4. Common Issues / Errors

| Issue                                      | Root Cause                                         |
| ------------------------------------------ | -------------------------------------------------- |
| Chart not found                            | Repo metadata outdated; `helm repo update` not run |
| Unable to locate chart version             | Stale or missing index.yaml                        |
| Repo URL invalid                           | Mistyped or deprecated repo URL                    |
| Installing outdated chart versions         | Old metadata cache                                 |
| Confusion between S3 storage and real repo | No index.yaml → not a repo                         |

---

## 5. Troubleshooting / Fixes

### Fix 1 — Update repository metadata

```bash
helm repo update
```

### Fix 2 — Remove and re‑add repo

```bash
helm repo remove myrepo
helm repo add myrepo <url>
helm repo update
```

### Fix 3 — Verify a real Helm repository

Check for presence of:

* `.tgz` chart files
* `index.yaml`

### Fix 4 — View local metadata

```
ls ~/.cache/helm/repository
```

### Fix 5 — Test connectivity

```
curl <repo-url>/index.yaml
```

---

## 6. Best Practices

* Always run `helm repo update` before installing or upgrading from a repo.
* Understand that **storage ≠ Helm repository** unless it contains a valid `index.yaml`.
* Use raw charts for simple projects; use Helm repos for shared or versioned chart distribution.
* Generate `index.yaml` using `helm repo index`; never create it manually.
* Use Artifact Hub only as a discovery/catalog tool — it does **not** store charts.
* Public repos: Bitnami, Grafana, Prometheus; private repos: S3, Nexus, Artifactory.
* Keep chart versions consistent across environments via versioned repositories.

---
---

# Helm Chart Structure — Notes

## 1. Concept / What

A Helm chart has a standardized folder structure that contains everything required to deploy an application to Kubernetes. This includes:

* **Chart.yaml** → chart metadata
* **values.yaml** → default configuration values
* **templates/** → Kubernetes manifest templates
* **charts/** → subchart dependencies
* **.helmignore** → files to exclude during packaging

Helm uses this structure to render final Kubernetes YAML, package charts, or publish them to a Helm repository.

---

## 2. Why / Purpose / Real Use Case

* Ensures consistency across environments
* Allows separation of configuration (`values.yaml`) from logic (`templates/`)
* Supports reusable and maintainable Kubernetes deployments
* Enables versioning and packaging through `helm package`
* Helps CI/CD pipelines deploy applications cleanly to EKS

Real-world examples:

* Deploying your application using Helm directly from Git (`helm install ./chart`)
* Packaging charts for internal distribution (`helm package` → `.tgz`)
* Hosting charts in S3, GitHub Pages, or Nexus for team-wide usage

---

## 3. How it Works / Steps / Syntax

### Folder Structure

```
mychart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl
│   └── NOTES.txt
├── charts/
└── .helmignore
```

### A. Chart.yaml — Metadata

```yaml
apiVersion: v2
name: myapp
description: Example Helm app
type: application
version: 1.0.0
appVersion: "1.2.3"
```

Defines chart identity, version, dependencies, and app version.

### B. values.yaml — Default Values

```yaml
replicaCount: 2
image:
  repository: myorg/myapp
  tag: "1.2.3"
service:
  type: ClusterIP
  port: 80
```

Users or CI/CD override these values for each environment.

### C. templates/ — Kubernetes Manifest Templates

Contains templated YAML files using Go templating syntax:

```gotemplate
{{ .Values.image.repository }}:{{ .Values.image.tag }}
{{ include "myapp.fullname" . }}
```

Rendered into final Kubernetes YAML when running:

```
helm template ./mychart
```

### D. _helpers.tpl — Reusable Helper Functions

```gotemplate
{{- define "myapp.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end }}
```

Used to avoid repetition across templates.

### E. charts/ — Subcharts

Stores packaged or local dependent charts. Managed via:

```
helm dependency update
```

### F. .helmignore — Ignored Files During Packaging

Works like `.gitignore` to keep charts clean.

### Packaging

```
helm package ./mychart
```

Produces:

```
mychart-1.0.0.tgz
```

This `.tgz` contains your entire chart (templates, values, helpers), **but NOT** an `index.yaml`.

---

## 4. Common Issues / Errors

| Issue                         | Description                                     |
| ----------------------------- | ----------------------------------------------- |
| Hardcoded values in templates | Should be placed in `values.yaml`               |
| Inconsistent naming           | Missing helper templates                        |
| Template syntax errors        | Incorrect indenting or missing braces           |
| charts/ folder empty          | Dependencies not downloaded                     |
| Confusion about index.yaml    | Belongs to Helm repositories, not chart folders |

---

## 5. Troubleshooting / Fixes

* Use `helm lint` to validate chart structure
* Render charts locally using `helm template` to inspect final YAML
* Add helper templates for consistent naming/labels
* Use `.helmignore` to avoid packaging unnecessary files
* Ensure `Chart.yaml` version is updated when packaging new releases

---

## 6. Best Practices

* Keep templates clean and reusable
* Store environment configuration in separate value files
* Do not write index.yaml manually; charts do not include it
* Use meaningful naming through `_helpers.tpl`
* Keep chart version (`version`) and app version (`appVersion`) consistent
* Use raw charts from Git for simple deployments; package charts only when needed

---
---
---
