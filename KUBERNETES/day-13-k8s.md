# Helm – What & Why (Detailed Explanation Version)

## **Concept / What**

Helm is the **package manager for Kubernetes**, similar to how apt/yum manage packages on Linux. It bundles multiple Kubernetes manifests into a single versioned package called a **Helm chart**. A chart contains templates for Kubernetes objects such as Deployments, Services, ConfigMaps, and more.

Helm uses a templating system where manifest files are written once inside the `templates/` directory, and their values are injected dynamically using `values.yaml` files. This allows a single chart to be reused across multiple environments (dev, stage, prod) without duplicating YAML files.

---

## **Why / Purpose / Use Case in Real-World**

* **Avoid YAML duplication:** One template per resource type is reused across all environments.
* **Consistency across environments:** Same chart, different values → predictable deployments.
* **Deploy complex apps easily:** Bundles multiple resources (Deployment, Service, Ingress, etc.).
* **Versioning & Rollbacks:** Helm releases are versioned; upgrades and rollbacks are simple.
* **Dependency Management:** Charts can depend on other charts; Helm installs them in a safe order.
* **Better CI/CD & GitOps:** Works well with ArgoCD/Flux for automated deployments.
* **Standardization:** Teams can enforce internal standards through centralized charts.

---

## **How It Works / Steps / Syntax**

1. **Write templates** inside the `templates/` directory:

   * Example: `deployment.yaml`, `service.yaml`, etc.
   * Templates contain placeholders like `{{ .Values.replicas }}`.

2. **Define values** in `values.yaml` or environment-specific files:

   * `values-dev.yaml`
   * `values-stage.yaml`
   * `values-prod.yaml`

3. **Install the chart:**

   ```sh
   helm install myapp ./mychart -f values-dev.yaml
   ```

4. **Upgrade the chart:**

   ```sh
   helm upgrade myapp ./mychart -f values-prod.yaml
   ```

5. **Rendered YAML:**
   Helm combines **templates + values** to generate final Kubernetes manifests.

---

## **Common Issues / Errors**

* **Incorrect indentation in templates** leading to rendering failures.
* **Missing values** causing incomplete manifests.
* **Over-complex templating** making charts hard to debug.
* **Values overridden incorrectly** in multi-environment setups.
* **Dependency version mismatch** between parent & child charts.

---

## **Troubleshooting / Fixes**

* Use `helm template` to preview rendered YAML locally.
* Use `helm lint` to validate chart structure.
* Use `helm get manifest <release>` to inspect applied resources.
* Validate expected values using `helm show values <chart>`.
* Simplify templates if they become too nested or complex.

---

## **Best Practices / Tips**

* Keep templates readable and avoid excessive logic.
* Use separate values files per environment.
* Version charts properly for controlled upgrades.
* Store charts in private registries (S3, CodeArtifact, etc.).
* Test rendered YAML before applying it.
* Keep default values minimal and environment overrides explicit.

---
---

# Helm Chart Structure + Installing & Upgrading Charts

## **Detailed Explanation Version (Updated with Real-World Behavior)**

---

# 🟦 **1. Concept / What**

A **Helm chart** is a packaged structure that contains everything required to deploy an application onto Kubernetes. It includes metadata (Chart.yaml), configuration values (values.yaml), and Kubernetes manifest templates (templates/).

The command:

```
helm create <chart-name>
```

creates a *sample chart structure*, but these sample files are **not production-ready**. Real-world teams **delete** most of the auto-generated templates and write their own templates manually.

Once the custom templates are ready, Helm reuses them across environments using different values files (dev, stage, prod). This prevents YAML duplication and maintains consistency.

---

# 🟦 **2. Why / Purpose / Real-World Use Cases**

* **Avoid YAML duplication:** One set of templates reused across all environments.
* **Standardization:** Teams follow a uniform Deployment/Service/ConfigMap pattern.
* **Versioning:** Every installation becomes a versioned release.
* **Environment flexibility:** Different values files without changing templates.
* **Dependency management:** Charts can include other charts (Redis, MongoDB, etc.).
* **Easy upgrades:** Changes applied through Helm upgrade instead of editing YAML manually.
* **Repeatability:** Same chart deploys dev/stage/prod consistently.

---

# 🟦 **3. How It Works / Steps / Syntax**

## **3.1 Creating a Chart**

Command:

```
helm create mychart
```

This generates the following structure:

```
mychart/
 ├── Chart.yaml
 ├── values.yaml
 ├── charts/            (empty)
 ├── templates/
 │     ├── deployment.yaml      (sample)
 │     ├── service.yaml         (sample)
 │     ├── hpa.yaml             (sample)
 │     ├── ingress.yaml         (sample)
 │     ├── serviceaccount.yaml  (sample)
 │     ├── tests/               (sample)
 │     └── _helpers.tpl
 └── .helmignore
```

### **Real-World Behavior**

* These sample files **are not used** in organizations.
* Teams usually **delete** all sample templates except the folder structure.
* They then **create their own Deployment/Service/ConfigMap/Secret/HPA templates** manually.

---

## **3.2 Chart.yaml (Metadata)**

Example:

```yaml
apiVersion: v2
name: myapp
description: My application chart
type: application
version: 1.0.0
appVersion: "2.5.1"
```

**Meaning:**

* `version`: chart version for upgrades/rollbacks.
* `appVersion`: real application version (image tag).

---

## **3.3 values.yaml (Configuration Values)**

Contains default values used by templates.
Example:

```yaml
replicas: 2
image:
  repository: nginx
  tag: "1.25"
service:
  port: 80
```

You override these using:

```
helm install myapp . -f values-prod.yaml
```

Real-world setups have:

* values-dev.yaml
* values-stage.yaml
* values-prod.yaml

---

## **3.4 templates/ (Your Real Kubernetes Manifests)**

The **team writes these manually**, not Helm.

Example Deployment template:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
spec:
  replicas: {{ .Values.replicas }}
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
```

Templates commonly include:

* deployment.yaml
* service.yaml
* ingress.yaml
* configmap.yaml
* secret.yaml
* hpa.yaml

---

# 🟦 **4. Installing Helm Charts**

Install a release:

```
helm install <release-name> <chart-path>
```

Example:

```
helm install myapp ./mychart -f values-prod.yaml
```

**What happens:**

1. Helm reads Chart.yaml
2. Loads values.yaml
3. Applies overrides
4. Renders templates into final YAML
5. Applies them to Kubernetes
6. Stores a release history

Dry-run before applying:

```
helm install myapp ./mychart --dry-run --debug
```

---

# 🟦 **5. Upgrading Helm Charts**

Upgrade an existing release:

```
helm upgrade myapp ./mychart -f values-prod.yaml
```

Helm:

* compares old vs new rendered manifests
* applies changes
* stores a new revision number

Dry-run upgrade:

```
helm upgrade myapp ./mychart --dry-run --debug
```

Useful commands:

```
helm list
helm status myapp
helm get manifest myapp
helm get values myapp
```

---

# 🟦 **6. Common Issues / Errors**

* Incorrect indentation breaks template rendering.
* Missing values cause invalid manifests.
* Over-complex templates become unreadable.
* Upgrades fail due to immutable fields (selector, clusterIP, etc.).
* Release stuck in `pending-upgrade`.

---

# 🟦 **7. Troubleshooting / Fixes**

* `helm lint` – validate chart
* `helm template` – preview generated YAML
* `helm install --dry-run --debug` – detailed error output
* `helm rollback <release> <revision>` – recover from bad upgrade
* Delete stuck release using `helm uninstall`

---

# 🟩 **8. Best Practices / Tips**

* Delete sample templates created by Helm.
* Write your own minimal and clean templates.
* Use environment-specific values files.
* Keep default values simple.
* Use helpers for naming and labels.
* Always `--dry-run` before install/upgrade.
* Version charts properly.
* Test rendered YAML before applying.

---
---

# Installing and Upgrading Helm Charts – Detailed Explanation Version

## **Concept / What**

Installing and upgrading Helm charts is the process of deploying and modifying applications in Kubernetes using Helm releases. A *release* is a versioned instance of a Helm chart running in the cluster. The commands `helm install` and `helm upgrade` manage the lifecycle of this release.

During installation, Helm renders chart templates with values, applies them to the cluster, and stores release history for rollbacks. During upgrades, Helm compares old and new manifests, applies changes, and increments the release revision.

---

## **Why / Purpose / Use Case in Real-World**

* **Versioned deployments:** Each install/upgrade becomes a new revision.
* **Easy configuration updates:** Change values and upgrade the chart instead of editing YAML.
* **Environment-specific deployments:** Different values files for dev/stage/prod.
* **Consistency:** Same chart reused across all environments.
* **Rollback support:** Restore previous versions instantly.
* **CI/CD automation:** Helm install/upgrade integrated into pipelines.
* **Predictable changes:** Templates + values rendering ensures consistent YAML.

---

## **How It Works / Steps / Syntax**

### **1. Installing a Helm Chart**

Command:

```
helm install <release-name> <chart-path> -f <values-file>
```

Example:

```
helm install myapp ./mychart -f values-prod.yaml
```

**What happens internally:**

1. Loads Chart.yaml metadata
2. Loads default values.yaml
3. Applies override values files
4. Renders templates into final Kubernetes YAML
5. Applies manifests to the cluster
6. Creates a release (revision 1)

### **Dry-run Installation** (recommended before real deployment)

```
helm install myapp ./mychart --dry-run --debug
```

Shows rendered YAML + potential template errors.

---

### **2. Upgrading a Helm Chart**

Command:

```
helm upgrade <release-name> <chart-path> -f <values-file>
```

Example:

```
helm upgrade myapp ./mychart -f values-prod.yaml
```

**What happens internally:**

1. Loads previous release (revision N)
2. Renders chart with new values
3. Compares old vs new manifests
4. Applies differences to the cluster
5. Creates a new revision (N+1)

### **Dry-run Upgrade**

```
helm upgrade myapp ./mychart --dry-run --debug
```

Used to verify changes before applying.

---

### **3. Useful Operational Commands**

List releases:

```
helm list
```

Get release status:

```
helm status myapp
```

View applied manifests:

```
helm get manifest myapp
```

View values used:

```
helm get values myapp
```

---

## **Common Issues / Errors**

* **Template rendering errors** due to indentation or syntax issues.
* **Missing values** leading to invalid manifests.
* **Immutable field updates** causing upgrade failures (e.g., selector, clusterIP).
* **Release stuck in `pending-upgrade`** due to failed apply.
* **Bad values override** causing unexpected configuration.

---

## **Troubleshooting / Fixes**

* Use `--dry-run --debug` to diagnose install/upgrade failures.
* Use `helm template` to preview rendered YAML locally.
* Check applied resources with `kubectl describe`.
* Rollback on failure:

  ```
  helm rollback <release-name> <revision>
  ```
* Remove stuck or broken releases:

  ```
  helm uninstall <release-name>
  ```

---

## **Best Practices / Tips**

* Always dry-run before installing/upgrading.
* Keep chart and app versions updated in Chart.yaml.
* Use separate values files for each environment.
* Maintain minimal default values and override only required fields.
* Test rendered YAML using `helm template`.
* Keep templates simple and avoid complex logic.
* Ensure CI/CD pipelines package consistent values and templates.

---
---

# Overriding values.yaml – Detailed Explanation Version

## **Concept / What**

Overriding `values.yaml` is the mechanism Helm uses to customize deployments for different environments (dev, stage, prod) without changing the templates. Helm merges values from multiple sources in a defined priority order, and the final merged values are used to render the templates.

Templates reference values using `.Values.<key>`, and overriding changes the final rendered Kubernetes manifests.

---

## **Why / Purpose / Use Case in Real-World**

* **Avoid duplication:** Same templates reused across environments.
* **Environment-specific configs:** Different replicas, ports, hosts, image tags.
* **Separation of concerns:** Templates define structure, values define environment data.
* **Safe upgrades:** Change configuration without touching chart logic.
* **Supports CI/CD:** Pipelines pass values for dynamic deployments.
* **Exact control:** `--set` and `--set-string` allow precise overrides.

Real-world usage examples:

* Dev uses 1 replica; prod uses 5 replicas.
* Dev uses `v1-dev` image; prod uses `v1-prod`.
* Different ingress hosts for each environment.

---

## **How It Works / Steps / Syntax**

### **1. Default values (values.yaml)**

This file contains base values:

```yaml
replicas: 2
image:
  repository: nginx
  tag: "1.25"
service:
  port: 80
```

These values are used by templates unless overridden.

---

### **2. Overriding Using Separate Values Files (-f)**

Most common and recommended method.

Example commands:

```
helm install myapp ./mychart -f values-dev.yaml
helm upgrade myapp ./mychart -f values-prod.yaml
```

Example environment files:

**values-dev.yaml**:

```yaml
replicas: 1
image:
  tag: "v1-dev"
host: dev.myapp.com
```

**values-prod.yaml**:

```yaml
replicas: 5
image:
  tag: "v1-prod"
host: myapp.com
```

Same templates → Different environments → Zero duplication.

---

### **3. Overriding Using --set (Inline Values)**

Used for quick overrides, but auto-converts types.

```
--set replicas=3
--set image.tag=v2
```

* Numbers become integers
* `true/false` become boolean
* Can cause issues when exact string is needed

---

### **4. Overriding Using --set-string (Force String Type)**

For cases where values must remain strings.

```
--set-string version=001
--set-string mode=false
```

Examples where this is required:

* Zero-padded numbers (
  001 should not become 1)
* Boolean strings (
  "false" should not become boolean false)
* Environment variable text (must stay string)

---

### **5. Overriding Using Multiple Values Files**

```
helm install myapp ./mychart -f base.yaml -f prod.yaml -f secrets.yaml
```

Priority:

* `base.yaml` → lowest
* `prod.yaml` → overrides base
* `secrets.yaml` → highest among files

**Last file wins.**

---

## **Priority Order (Lowest → Highest)**

Helm merges values in this exact order:

1️⃣ **Default values.yaml**
2️⃣ **Values files provided with -f** (first file = lower priority, last file = highest)
3️⃣ **--set** (inline overrides)
4️⃣ **--set-string** (stronger than --set)
5️⃣ **CI/CD injected values** (tool-dependent, optional)

**Rule:**
➡️ **The LAST applied value always wins.**

---

## **Common Issues / Errors**

* Wrong indentation or YAML structure in override files.
* Incorrect use of `--set` causing value type issues.
* Accidentally passing the wrong values file (e.g., `prod` in dev cluster).
* Missing required keys leading to templating failures.
* Deeply nested values overridden incorrectly.

---

## **Troubleshooting / Fixes**

* Preview rendered YAML:

  ```
  helm template ./mychart -f values-prod.yaml
  ```
* View final applied values:

  ```
  helm get values myapp --all
  ```
* Use debug mode:

  ```
  helm install myapp ./mychart --dry-run --debug
  ```
* Validate override structure:

  ```
  yamllint values-prod.yaml
  ```
* Use separate secrets file if needed.

---

## **Best Practices / Tips**

* Use dedicated values files: `values-dev.yaml`, `values-stage.yaml`, `values-prod.yaml`.
* Keep `values.yaml` minimal (only safe defaults).
* Use `--set` only for temporary or CI/CD overrides.
* Use `--set-string` when type accuracy is important.
* Keep secrets separate (secrets.yaml or encrypted with SOPS/SealedSecrets).
* Always dry-run before deploying with overrides.
* Document environment-specific differences clearly.

---
---

# Rollback and Versioning – Detailed Explanation Version

## **Concept / What**

Helm uses **revisions** to track every installation, upgrade, and rollback of a release. Each revision contains the full rendered Kubernetes manifests, values used, and chart metadata. When a rollback occurs, Helm restores a previous revision's content but creates a **new revision number**, ensuring history remains linear.

Versioning ensures that every deployment action is traceable, repeatable, and reversible without modifying the chart or Kubernetes resources manually.

---

## **Why / Purpose / Use Case in Real-World**

* **Safe deployments:** Every upgrade creates a new revision for tracking.
* **Rollback capability:** If an upgrade breaks, you return to a known good revision.
* **Audit history:** Teams can see when changes were made and by whom.
* **Immutable history:** Kubernetes resources at every revision are preserved.
* **Consistent environments:** Easy to roll back dev/stage/prod to stable states.
* **Disaster recovery:** When production breaks after deployment, rollback restores service instantly.

---

## **How It Works / Steps / Syntax**

### **1. Install → Revision 1**

```
helm install myapp ./mychart
```

Creates **revision 1**.

### **2. Upgrade → Revision increases**

```
helm upgrade myapp ./mychart -f values-prod.yaml
```

Creates **revision 2**, then revision 3, and so on.

### **3. View History**

```
helm history myapp
```

Shows each revision, its status, chart version, and timestamp.

---

### **4. Rollback**

```
helm rollback myapp <revision-number>
```

Example:

```
helm rollback myapp 1
```

This restores revision 1's content BUT creates a **new revision**.

If the working revision was 3 and you rollback to 1:

* Helm will create revision 4 containing **revision 1's content**.
* Revision numbers never decrease.

**Revision sequence example:**

```
1 → install
2 → upgrade
3 → upgrade
4 → rollback to revision 1
```

Revision **4** now contains the same manifests as **revision 1**, even though the numbers differ.

---

### **5. Revision Behavior (Important)**

* Revision numbers **always increase**.
* Rollbacks **DO NOT** revert revision numbers.
* Rollbacks create a **new revision** with the old revision’s manifests.
* Two revisions can have **identical content**, e.g., revision 1 and revision 4.
* Revision numbers represent **action order**, not uniqueness of content.

---

## **Common Issues / Errors**

* **Immutable field error**: Some fields (like selector, clusterIP) cannot be changed on upgrade.
* **Rollback fails** due to corrupted release history.
* **Too many revisions** accumulating over time.
* **Missing revision secret** if someone manually deleted it.
* **Failed upgrade leaves release stuck in pending-upgrade.**

---

## **Troubleshooting / Fixes**

* Check history:

  ```
  helm history myapp
  ```
* Inspect manifests of any revision:

  ```
  helm get manifest myapp
  ```
* Check cluster resources:

  ```
  kubectl describe deployment myapp
  ```
* Rollback to a stable revision:

  ```
  helm rollback myapp <revision>
  ```
* Uninstall stuck release:

  ```
  helm uninstall myapp
  ```
* Limit revision history (safe way):

  ```
  helm upgrade myapp . --history-max 3
  ```

---

## **Best Practices / Tips**

* Never delete individual revision secrets manually.
* Use `--history-max` to control revision count safely.
* Always run `--dry-run --debug` before upgrades.
* Increase chart version (`version:` in Chart.yaml) for every change.
* Keep stable values files for rollback readiness.
* Regularly review `helm history` in prod clusters.
* Understand that revision number ≠ content version; identical content can exist in different revisions.

---
---

# Hosting Helm Charts in Private Repositories (S3 / CodeArtifact)

## **Concept / What**

Hosting Helm charts in private repositories means storing the packaged chart (`.tgz` file) in a secure, centralized registry such as **Amazon S3** or **AWS CodeArtifact**, rather than deploying Helm charts directly from the source code in GitHub.

A private Helm repository contains:

* Packaged chart files (`mychart-1.0.0.tgz`)
* `index.yaml` (chart metadata and version index)

These repositories allow consistent, versioned, repeatable deployments across environments.

---

## **Why / Purpose / Use Case in Real-World**

* **Versioned chart management** for dev/qa/stage/prod.
* **Separation of source and deployment artifacts**.
* **Rollback support** using chart versions.
* **Environment consistency** – same chart deployed everywhere.
* **Security** – charts stored in private registries.
* **CI/CD friendly** – pipelines fetch charts from a stable registry.
* **Enterprise governance** – CodeArtifact supports fine-grained IAM access.

---

## **How It Works / Steps / Syntax**

### **1. Package the Helm Chart**

Create a `.tgz` artifact:

```
helm package mychart/
```

Produces:

```
mychart-1.0.0.tgz
```

This `.tgz` file is the deployable unit.

---

## **2. Hosting Charts in S3 (Static Helm Repository)**

### **Step 1 — Create an S3 bucket**

Example:

```
s3://company-helm-repo/
```

### **Step 2 — Upload packaged chart**

```
aws s3 cp mychart-1.0.0.tgz s3://company-helm-repo/
```

### **Step 3 — Generate or update index.yaml**

```
helm repo index . --url https://<s3-url>
```

This creates or updates `index.yaml`.

### **Step 4 — Upload index.yaml**

```
aws s3 cp index.yaml s3://company-helm-repo/
```

### **Step 5 — Add repo to Helm client**

```
helm repo add companyrepo https://<s3-url>
helm repo update
```

### **Step 6 — Install from S3**

```
helm install myapp companyrepo/mychart --version 1.0.0
```

---

## **3. Hosting Charts in AWS CodeArtifact (Managed Helm Repository)**

### **Step 1 — Create Domain & Repository**

Example:

```
Domain: mydomain
Repository: my-helm-repo
```

### **Step 2 — Login Helm client to CodeArtifact**

```
aws codeartifact login --tool helm \
    --domain mydomain \
    --repository my-helm-repo \
    --domain-owner <AWS_ACCOUNT_ID>
```

### **Step 3 — Publish the chart**

```
aws codeartifact publish-package-version \
  --domain mydomain \
  --repository my-helm-repo \
  --format helm \
  --namespace myteam \
  --package mychart \
  --package-version 1.0.0 \
  --asset-content mychart-1.0.0.tgz
```

### **Step 4 — Install from CodeArtifact**

```
helm install myapp my-helm-repo/mychart --version 1.0.0
```

---

## **Common Issues / Errors**

* Missing or outdated `index.yaml` (S3).
* Incorrect S3 bucket permissions.
* Wrong or missing chart version.
* CodeArtifact authentication failures.
* Helm repo URL misconfiguration.
* Not packaging the chart before upload.

---

## **Troubleshooting / Fixes**

* Rebuild index for S3 repos:

  ```
  helm repo index . --url <repo-url>
  ```
* Re-login to CodeArtifact:

  ```
  aws codeartifact login --tool helm ...
  ```
* Validate chart structure:

  ```
  helm lint mychart/
  ```
* Test installation directly from repo:

  ```
  helm search repo companyrepo
  ```
* Validate S3 permissions in IAM policies.

---

## **Best Practices / Tips**

* Always bump `version:` in Chart.yaml for each release.
* NEVER store `.tgz` chart packages in GitHub.
* Use CI/CD to package & upload charts automatically.
* Prefer **CodeArtifact** for enterprise security and governance.
* Use **S3** for simple, low-cost development repos.
* Keep older chart versions for rollback.
* Validate charts with `helm lint` before packaging.

---
---
---

