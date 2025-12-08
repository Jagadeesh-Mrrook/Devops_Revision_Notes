# Helm Fundamentals — Notes

---

## **1. Concept / What: What is Helm?**

Helm is the **package manager for Kubernetes**, similar to how apt/yum manage packages in Linux. Helm bundles Kubernetes manifests into reusable, versioned **charts** and manages application lifecycle using **releases**. It provides **templating**, **configuration management**, and **version-controlled deployments** across multiple environments.

---

## **2. Why / Purpose / Real Use Case**

### **Why Helm Exists**

Managing applications directly with Kubernetes manifests leads to:

* Multiple YAML files for the same application
* Environment-wise duplication (dev/stage/prod)
* Manual edits prone to errors
* No unified version control for all Kubernetes objects
* Complex rollback procedures involving multiple YAML corrections

### **Helm Solves This By:**

* Packaging all manifests into a **single chart**
* Using **values.yaml** to override configuration per environment
* Managing deployments as **versioned releases**
* Rolling back the entire application (Deployments, Services, Ingress, ConfigMaps, Secrets, etc.) in **one command**
* Making environment drift detection easy for GitOps platforms like ArgoCD and Flux

### **Real-World Use Cases**

* Deploying microservices on EKS with templated Deployments, Services, Ingress
* Blue-green/canary deployments with versioned configuration
* Reusable internal charts that any team can install
* CI/CD pipelines using Helm upgrade to ensure consistent delivery
* GitOps workflows where Helm charts are the source of truth

---

## **3. How It Works / Steps / Syntax**

### **Core Helm Commands**

```bash
helm install my-app ./mychart
helm upgrade my-app ./mychart -f prod-values.yaml
helm rollback my-app 2
helm uninstall my-app
helm template mychart    # Render templates without installing
helm lint mychart        # Validate chart
```

### **Why Helm Rollback Is More Powerful Than kubectl**

With kubectl:

* Only **Deployments/StatefulSets/DaemonSets** can be rolled back
* **Services, Ingress, ConfigMaps, Secrets** cannot be undone
* Manual YAML re-apply is required for each object

With Helm:

* Every release version stores **all rendered manifests**
* `helm rollback` restores **entire application state**
* No manual YAML edits required

### **Examples From Blue-Green Discussion**

* If Service selector changes from `app=blue` → `app=green` using kubectl, rollback requires manually editing the Service YAML and reapplying.
* Kubernetes **cannot rollback Services, Ingress, ConfigMaps, or Secrets**.
* Helm rollback restores the old selector, old deployments, old config—all in a single command.

### **High-Level Workflow**

1. Write Helm chart with templated YAML (Deployment, Service, Ingress, ConfigMap, etc.)
2. Define configurations in `values.yaml`
3. Use environment-specific values files
4. Install → Upgrade → Rollback using release versions
5. GitOps tools auto-sync these charts to cluster

---

## **4. Common Issues / Errors**

| Issue                            | Reason                                | Fix                                           |
| -------------------------------- | ------------------------------------- | --------------------------------------------- |
| Values not applied               | Wrong key path or indentation         | Check with `helm get values`, fix values.yaml |
| Template syntax errors           | Incorrect `{{ }}` expressions         | Use `helm lint` and `helm template` to debug  |
| Resource already exists          | Helm didn’t manage previous resources | Use `helm uninstall` or `--replace`           |
| CI/CD failure after chart update | Breaking changes in chart structure   | Follow semantic versioning for chart updates  |

---

## **5. Troubleshooting / Fixes**

### **Issue: Incorrect selector or config after upgrade**

* Use `helm rollback <release> <version>`
* Validate rendered YAML using `helm template`

### **Issue: Environment drift**

* Run `helm diff upgrade` to compare cluster vs. chart (plugin)
* Ensure values files are correct for each environment

### **Issue: Templating errors**

* Check helpers.tpl for missing or mis-indented templates
* Always run `helm lint` before pushing changes

### **Issue: Rollback doesn’t behave as expected**

* Check previous release history using `helm history <release>`
* Ensure no manual kubectl edits have modified resources unexpectedly

---

## **6. Best Practices**

* Store all Kubernetes manifests in Helm templates
* Never hardcode values—move them to values.yaml
* Maintain separate values files for dev/stage/prod
* Use `helpers.tpl` for labels, annotations, name templates
* Keep chart versions updated with semantic versioning
* Use `helm lint` and `helm template` before deploying
* Avoid manual kubectl edits for Helm-managed resources
* Use GitOps (ArgoCD/Flux) with Helm charts for complete lifecycle management

---
---

# Difference Between Helm and kubectl — Notes

---

## **1. Concept / What**

### **kubectl**

`kubectl` is the Kubernetes API client used to apply, update, delete, and view Kubernetes resources using raw manifest files. It works at the **resource level**, applying YAML exactly as written.

### **Helm**

Helm is the **package manager for Kubernetes**, which bundles multiple Kubernetes manifests into reusable, versioned **charts** and manages them as **application releases**.

---

## **2. Why / Purpose / Real Use Case**

### **Why Kubectl Alone Becomes Difficult**

* Each manifest must be applied manually.
* For multi-environment deployments, separate YAMLs must be maintained.
* No packaging, no versioning of YAML.
* Rollbacks are limited only to Deployments/StatefulSets.
* No history for Services, Ingress, ConfigMaps, Secrets, etc.

### **Why Helm Is Used**

* All manifests packaged into one chart.
* **Templating** allows dynamic configuration instead of multiple YAML copies.
* **values.yaml** enables environment-specific overrides.
* **Release history** enables one-click rollback.
* Charts can be stored/distributed via **repositories**.
* Perfect fit for CI/CD and GitOps workflows.

### **Real-World Use Cases**

* Deploying microservices with shared templates across environments.
* Blue-green rollouts where Service selectors must be versioned.
* Reverting full applications without manual YAML edits.
* CI/CD pipelines using `helm upgrade` for safe deployments.
* Storing internal application charts for reuse by other teams.

---

## **3. How It Works / Steps / Syntax**

### **kubectl Workflow**

* Write YAML files.
* Apply them directly:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
```

* Rollback only works for workloads:

```bash
kubectl rollout undo deployment/my-app
```

* For Services, Ingress, ConfigMaps, Secrets → manual reapply is required.

### **Helm Workflow**

* Put all templates inside `templates/` folder.
* Define environment configs in `values.yaml`.
* Install/upgrade:

```bash
helm install my-app ./mychart
helm upgrade my-app ./mychart -f prod-values.yaml
```

* Rollback full application:

```bash
helm rollback my-app 2
```

* List release history:

```bash
helm history my-app
```

### **Template Reuse vs YAML Duplication**

* kubectl → duplicate deployment-dev.yaml, deployment-prod.yaml
* Helm → one `deployment.yaml` template, multiple values files

---

## **4. Common Issues / Errors**

| Area              | kubectl                       | Helm                           |
| ----------------- | ----------------------------- | ------------------------------ |
| Rollback          | Only Deployments/StatefulSets | Full app rollback              |
| Multi-environment | Duplicate YAML files          | values.yaml overrides          |
| Versioning        | None                          | Release history stored         |
| Drift             | Hard to detect                | Easy to detect via `helm diff` |
| Packaging         | Not possible                  | Charts packaged + stored       |

---

## **5. Troubleshooting / Fixes**

### **kubectl Rollback Limitations**

* If Service selector changes (blue → green), rollback must be done manually by editing YAML.
* For ConfigMaps/Secrets rollback → manually reapply old YAML.

**Fix:** Maintain Git history and reapply the older manifests manually.

### **Helm Rollback Unexpected Behavior**

* If resources were edited manually using kubectl, rollback may not behave consistently.

**Fix:** Avoid mixing kubectl edits with Helm-managed objects.

### **Template Rendering Issues**

* Use `helm lint` for template validation.
* Use `helm template` to preview actual YAML before deployment.

---

## **6. Best Practices**

### kubectl Best Practices

* Use kubectl only for debugging, quick checks, or simple resources.
* Keep manifests in Git for manual rollback.

### Helm Best Practices

* Treat Helm charts as the single source of truth.
* Never manually edit Helm-managed resources with kubectl.
* Use separate values files for dev/stage/prod.
* Store charts in repositories for CI/CD.
* Use semantic versioning for chart updates.
* Validate charts using `helm lint` before deployment.

---
---

# Helm Charts, Releases, and Repositories — Notes

---

## **1. Concept / What**

### **Helm Chart**

A **Helm Chart** is a packaged, structured bundle that contains everything needed to deploy an application on Kubernetes. It includes:

* **Templated manifests** (Deployment, Service, Ingress, ConfigMap, Secret, etc.) under `templates/`
* **values.yaml** for default configuration
* **Chart.yaml** for metadata (name, version, description)
* **helpers.tpl** for reusable template functions
* Additional environment-specific values files if needed

When packaged using `helm package`, it is compressed into a **.tgz** archive.

---

### **Release**

A **Release** is an installed, running instance of a chart inside a Kubernetes cluster.

* Created using `helm install`
* Each upgrade creates a **new revision** stored by Helm
* Supports **rollback** to any previous version
* Multiple releases of the same chart can exist (e.g., `myapp`, `myapp-dev`, `myapp-test`)

---

### **Repository**

A **Helm Repository** is a storage location where charts are stored, versioned, and fetched from.
Repositories can be:

* **Public** (Bitnami, Artifact Hub)
* **Private** (Nexus, Artifactory, S3, GCS, internal company repos)

Repositories enable chart distribution for CI/CD and GitOps workflows.

---

## **2. Why / Purpose / Real Use Case**

### **Why Charts?**

* Package all Kubernetes manifests into a single reusable unit
* Avoid duplicated YAML across environments
* Enable templating through values.yaml
* Provide a standard structure to share applications across teams

### **Why Releases?**

* Version-controlled deployments with full history
* Easy rollbacks using a single command
* Track which values and templates were used at each revision

### **Why Repositories?**

* Store and distribute chart versions
* Teams pull charts instead of copying YAML files
* Essential for automated CI/CD & GitOps (ArgoCD/Flux)
* Prevents configuration drift

**Real Example:**
Package a microservice as a chart → upload to S3 repository → deploy same chart to dev/stage/prod with different values → rollback safely when needed.

---

## **3. How It Works / Steps / Syntax**

### **Packaging a Chart**

```bash
helm package ./mychart
```

Creates:

```
mychart-1.0.0.tgz
```

### **Adding a Repository**

```bash
helm repo add myrepo s3://company-charts
helm repo update
```

### **Installing a Release**

```bash
helm install orderservice myrepo/orderservice --version 1.0.0
```

### **Upgrading a Release**

```bash
helm upgrade orderservice myrepo/orderservice --version 1.0.1
```

### **Rollback**

```bash
helm rollback orderservice 1
```

Restores *every* object deployed by the chart.

### **Release History**

```bash
helm history orderservice
```

---

## **4. Common Issues / Errors**

| Area            | Issue                                         | Fix                                                                |
| --------------- | --------------------------------------------- | ------------------------------------------------------------------ |
| Chart packaging | Missing metadata in `Chart.yaml`              | Add `name`, `version`, and `apiVersion` to `Chart.yaml`            |
| Repository      | Authentication errors when pushing/pulling    | Configure repository credentials or S3 IAM roles                   |
| Release upgrade | Breaking template changes cause failures      | Use semantic versioning; test with `helm template` and `helm lint` |
| Rollback        | Manual `kubectl` edits break release tracking | Avoid editing Helm-managed resources; update chart/values instead  |

---

## **5. Troubleshooting / Fixes**

Troubleshooting / Fixes**

### **Chart Not Rendering Correctly**

* Use `helm template` to inspect rendered manifests
* Validate with `helm lint`

### **Release Fails During Upgrade**

* Run `helm get values <release>` to compare applied configs
* Check diffs using Helm Diff plugin

### **Repository Index Issues**

* Run:

```bash
helm repo update
```

* Ensure charts are versioned uniquely

---

## **6. Best Practices**

* Keep chart templates generic; move all environment settings to values files
* Follow semantic versioning for chart updates
* Never manually modify Helm-managed resources using kubectl
* Use private repositories for internal organizational charts
* Validate charts with `helm lint` before pushing
* Store charts and values in Git for audit and GitOps
* Keep chart dependencies clear and avoid unnecessary complexity

---
---

# 4 — How Helm Improves Configuration Management

---

## Concept / What

Helm improves Kubernetes configuration management by separating **templates** (the reusable structure of Kubernetes objects) from **configuration** (environment-specific data). Templates live under `templates/` inside the chart and use Go-style templating; configuration lives in `values.yaml` and environment-specific override files (e.g., `values-dev.yaml`, `values-prod.yaml`). This makes manifests dynamic, reusable, and consistent across environments.

---

## Why / Purpose / Real Use Case

### Problems with raw Kubernetes manifests

* Manifests are static YAML files and must be duplicated per environment (dev/stage/prod).
* Manual edits across multiple files cause inconsistencies and configuration drift.
* Rollbacks using `kubectl` are limited (workloads only) and do not restore services, configmaps, ingress, secrets, etc.
* CI/CD becomes error-prone when multiple manifests must be kept in sync.

### How Helm solves this

* **Single template, multiple configs:** One template file can be rendered for multiple environments using different `values.yaml` files.
* **Versioned releases:** Each `helm upgrade` creates a new release revision that records rendered manifests and values used, enabling full-application rollbacks.
* **CI/CD / GitOps friendly:** Pipelines and GitOps controllers (ArgoCD/Flux) can deploy the same chart with different values files, ensuring reproducible deployments.

**Real-world example:** a microservice uses one `templates/deployment.yaml` and different `values-*.yaml` files to change `replicas`, `image.tag`, and environment variables between dev and prod.

---

## How it Works / Steps / Syntax

### 1) Template placeholders (example snippet)

`templates/deployment.yaml`

```yaml
replicas: {{ .Values.replicas }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
env:
  - name: LOG_LEVEL
    value: "{{ .Values.logLevel }}"
```

### 2) Default values (`values.yaml`)

```yaml
replicas: 2
image:
  repository: myapp
  tag: "1.0.0"
logLevel: "info"
```

### 3) Environment overrides

`values-dev.yaml`

```yaml
replicas: 1
logLevel: "debug"
```

`values-prod.yaml`

```yaml
replicas: 5
image:
  tag: "1.0.3"
logLevel: "info"
```

### 4) Deploy / upgrade commands

```bash
helm upgrade --install myapp ./chart -f values-prod.yaml
```

### 5) Validate rendered manifests locally

```bash
helm template myapp ./chart -f values-prod.yaml
```

---

## Common Issues / Errors

* **Missing keys** in `values.yaml` cause template renders to fail or produce empty fields.
* **Indentation errors** inside templates or values files break YAML structure.
* **Hardcoded values** in templates reduce reusability and cause maintenance issues.
* **Wrong values file** supplied in CI/CD results in incorrect deployments.
* **Manual kubectl edits** to Helm-managed resources cause release drift and unexpected rollback results.

---

## Troubleshooting / Fixes

* Use `helm lint` to detect chart problems early.
* Use `helm template` to inspect the exact rendered YAML before applying.
* Use the Helm Diff plugin (`helm plugin install https://github.com/databus23/helm-diff`) to preview changes during upgrades.
* Keep `values.yaml` flat and well-documented; avoid unnecessary deep nesting.
* Never manually edit resources that are managed by Helm; make changes through charts/values and perform `helm upgrade`.

---

## Best Practices

* Keep templates generic; move all environment-specific settings to `values` files.
* Maintain clearly named `values-<env>.yaml` files (dev/stage/prod) and store them in Git.
* Validate templates locally with `helm lint` and `helm template` as part of CI checks.
* Use semantic versioning for chart changes and document breaking changes in a changelog.
* Use `helpers.tpl` for common label/name patterns and centralize naming conventions.
* Limit the amount of logic inside templates; prefer simple templates and precomputed values where possible.

---
---

# 5 — Helm in GitOps Workflows (ArgoCD / FluxCD)

---

## **Concept / What**

GitOps is a model where **Git is the single source of truth for Kubernetes deployments**. Tools like **ArgoCD** or **FluxCD** continuously monitor Git repositories, detect changes, render Helm charts, and ensure the cluster always matches Git.

Helm provides the **templating and configuration engine**, while ArgoCD provides the **synchronization and drift correction**.

---

## **Why / Purpose / Real Use Case**

### Problems solved by GitOps + Helm

* CI/CD no longer needs cluster credentials or kubeconfig.
* Deployments become fully auditable and traceable through Git history.
* Any manual change in the cluster is detected and reverted automatically.
* Multi-environment deployments are consistent (dev/stage/prod).
* Rollbacks are simple Git operations.
* Infrastructure and application configuration become declarative.

### Real-world example

You store:

* **One common Helm chart** (templates)
* **Multiple environment-specific values files** (dev/stage/prod)

ArgoCD deploys the correct values file to the correct cluster.

---

## **How It Works / Steps / Syntax**

### **1. Git stores the desired state**

Git holds:

* Helm chart templates
* values-dev.yaml
* values-prod.yaml
* ArgoCD Application manifests

Git change → ArgoCD detects it → deploys automatically.

---

### **2. ArgoCD pulls and renders Helm charts**

ArgoCD runs an internal rendering engine equivalent to:

```bash
helm template myapp . -f values-prod.yaml
```

Then it compares rendered YAML with the currently running cluster state.

---

### **3. ArgoCD syncs changes to the correct EKS cluster**

ArgoCD determines which cluster to deploy to using the **Application manifest**, not kubeconfig.

Example:

```yaml
destination:
  server: https://prod-eks-api-url
  namespace: myapp
```

This tells ArgoCD to deploy to the prod EKS.

---

### **4. Environment-specific deployment using valueFiles**

```yaml
source:
  helm:
    valueFiles:
      - values-prod.yaml
```

* Changing **values-dev.yaml** → Only Dev environment updates.
* Changing **values-prod.yaml** → Only Prod updates.

---

### **5. Template changes apply to ALL environments**

Because all ArgoCD applications use the same Helm chart:

* Changing anything inside `templates/` affects Dev, Stage, Prod.
* This is intentional because templates define the **base structure** of the application.

---

### **6. Rollbacks in GitOps**

Rollback is done by reverting Git commits:

```bash
git revert <commit-sha>
git push
```

ArgoCD detects the revert and restores the old Kubernetes state.

This reverts **Deployments, Services, Ingress, ConfigMaps, Secrets**, everything.

---

## **Common Issues / Errors**

* Wrong values file applied to the wrong environment.
* Template change unintentionally impacts all environments.
* Incorrect `destination` cluster in Application manifest.
* Drift occurs when someone applies `kubectl` manually.
* ArgoCD Application manifest not in sync with repo structure.

---

## **Troubleshooting / Fixes**

* Use ArgoCD UI logs to inspect render and sync steps.
* Validate rendering with:

```bash
helm template . -f values-prod.yaml
```

* Ensure `valueFiles` mapping is correct for each environment.
* Use `argocd app diff` to detect expected changes.
* Revert incorrect Git commits to trigger rollback.
* Avoid manual `kubectl apply` — rely only on Git.

---

## **Best Practices**

* Use **one Helm chart** for all environments.
* Keep environment differences in **values-<env>.yaml** files.
* NEVER manually edit Kubernetes resources managed by ArgoCD.
* Enable drift detection and auto-sync for Dev; manual sync for Prod.
* Use separate Git branches for environment promotion (dev → stage → prod).
* Keep ArgoCD Application manifests in a dedicated GitOps repo.
* Use semantic versioning when referencing packaged charts (if used).
* Limit template logic; keep Helm charts simple and maintainable.

---
---
---
