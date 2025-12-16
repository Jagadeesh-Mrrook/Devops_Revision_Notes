# Helm Release Lifecycle — What is a Release

## Concept / What

A **Helm release** is a **named instance of a Helm chart deployed to a Kubernetes cluster**.
When you run `helm install`, Helm creates a release that represents **one concrete deployment** of an application with a specific chart, values, namespace, and revision history.

In simple terms:

> A chart is a template, a **release is a live deployment created from that template**.

---

## Why / Purpose / Real Use Case

Helm releases exist to manage **multiple deployments of the same application** independently.

### Real-world EKS scenarios

* Same application deployed to **dev, qa, prod** using the same chart
* Each environment needs:

  * Different replica counts
  * Different image tags
  * Different resource limits
* Each environment must support:

  * Independent upgrades
  * Safe rollbacks
  * Isolated failures

Without releases:

* Helm cannot track deployment history
* Rollbacks are impossible
* Upgrades become risky

Example:

* `myapp-dev` → Dev namespace
* `myapp-prod` → Prod namespace
  Same chart, **two separate releases**, fully isolated.

---

## How it Works / Steps / Syntax

### 1. Creating a Release

```bash
helm install myapp-dev ./mychart -n dev
```

This command:

* Renders templates using `values.yaml`
* Applies Kubernetes manifests
* Creates a **release record** in the cluster

Key parts:

* `myapp-dev` → Release name
* `./mychart` → Helm chart
* `dev` → Kubernetes namespace

---

### 2. What Helm Stores for a Release

For every release, Helm stores:

* Release name
* Chart name and version
* Values used during installation
* Fully rendered Kubernetes manifests
* Revision number (v1, v2, v3…)

This data is stored as Kubernetes **Secrets (or ConfigMaps in older setups)**.

This storage enables:

* `helm upgrade`
* `helm rollback`
* `helm status`

---

### 3. One Chart → Multiple Releases

```bash
helm install myapp-dev  ./mychart -n dev
helm install myapp-qa   ./mychart -n qa
helm install myapp-prod ./mychart -n prod
```

All deployments:

* Use the same chart
* Use the same templates
* Are **different releases** with separate lifecycles

---

### 4. Viewing and Managing Releases

```bash
helm list -n dev
helm status myapp-dev
```

These commands work only because Helm tracks deployments as **releases**.

---

## Common Issues / Errors

* Confusing **chart** with **release**
* Assuming Helm deploys charts directly (Helm deploys releases)
* Forgetting the namespace while upgrading or checking status
* Trying to install again instead of upgrading an existing release

---

## Troubleshooting / Fixes

* Release not found error:

  ```bash
  helm list --all-namespaces
  ```
* Release already exists error:

  * Use `helm upgrade` instead of `helm install`
* Stuck or broken release:

  ```bash
  helm uninstall <release-name>
  ```

---

## Best Practices

* Use meaningful release names (`app-env`)
* One release per environment
* Always specify namespace explicitly
* Treat releases as **deployable and rollback-safe units**

---
---

# Helm Release Lifecycle — Release Naming Conventions

## Concept / What

A **Helm release name** is the **unique identifier** Helm uses to manage a deployed instance of a chart.
All Helm lifecycle operations such as **install, upgrade, rollback, status, and uninstall** are executed using the release name.

In simple terms:

> The release name represents the **identity of a deployment** in Helm.

---

## Why / Purpose / Real Use Case

Release naming is critical in **real-world EKS and Kubernetes environments** because:

* The same chart is deployed across multiple environments (dev, qa, prod)
* CI/CD pipelines rely on release names to upgrade the **correct deployment**
* Rollbacks depend on identifying the exact release and its history
* Poor naming can cause accidental upgrades or rollbacks in production

### Real-world scenario

An organization deploying the same application:

* `orders-dev` → Development
* `orders-qa` → QA
* `orders-prod` → Production

All use the same chart but are **separate Helm releases**.

---

## How it Works / Steps / Syntax

### 1. Manually Defining a Release Name (Recommended)

```bash
helm install orders-dev ./orders-chart -n dev
```

* `orders-dev` → Explicit release name
* Provides clarity and control
* Easy to manage in CI/CD pipelines

---

### 2. Auto-Generated Release Names (Not Recommended for Production)

```bash
helm install ./orders-chart -n dev --generate-name
```

Helm generates random names like:

```text
happy-tiger
brave-lion
```

Limitations:

* Difficult to track
* Confusing during upgrades and rollbacks
* Not suitable for CI/CD automation

Used mainly for:

* Local testing
* Temporary experiments

---

### 3. Common Naming Conventions (Industry Practices)

Organizations usually standardize release names using combinations of:

* Application name
* Environment
* Namespace
* Region

Examples:

```text
orders-dev
orders-prod
orders-prod-ap-south-1
orders-dev-platform
```

The exact format depends on organizational standards.

---

### 4. CI/CD Usage Example (EKS)

```bash
helm upgrade --install orders-prod ./orders-chart \
  -n prod \
  -f values-prod.yaml
```

Why this works well:

* Pipeline upgrades a **known release**
* Rollbacks are predictable
* No accidental cross-environment deployments

---

### 5. Release Name Scope (Important Rule)

* Release names must be **unique within a namespace**
* The same release name **can exist in different namespaces**

Valid example:

```bash
helm install orders ./chart -n dev
helm install orders ./chart -n prod
```

---

## Common Issues / Errors

* Using auto-generated release names in production
* Inconsistent naming across environments
* Forgetting the namespace during upgrades
* Reusing release names unintentionally in the same namespace

---

## Troubleshooting / Fixes

* List all releases across namespaces:

  ```bash
  helm list --all-namespaces
  ```
* Check release existence before upgrade
* Standardize release naming in CI/CD variables

---

## Best Practices

* Always use **manual release names** in production
* Follow a consistent, documented naming convention
* Include environment information in the release name
* Avoid `--generate-name` outside local testing
* Treat release names as **stable and immutable identifiers**

---
---

# Helm Release Lifecycle — Release Versioning

## Concept / What

**Helm Release Versioning** refers to the **revision numbers** that Helm assigns to a release every time its state changes through Helm operations.

* These are called **revisions**: `1`, `2`, `3`, …
* This is **NOT** the same as:

  * Chart version
  * Application version

In short:

> **Release versioning tracks the deployment history of a Helm release.**

---

## Why / Purpose / Real Use Case

Release versioning exists to make Helm deployments:

* Safe
* Auditable
* Rollback-friendly

### Real-world EKS scenarios

* CI/CD pipelines deploy new versions frequently
* A bad image, config, or value can break production
* Helm release revisions allow teams to:

  * See what changed
  * Know when it changed
  * Quickly roll back to a known-good state

Without release versioning:

* Rollbacks would require redeploying manually
* Debugging production issues would be difficult

---

## How it Works / Steps / Syntax

### 1️⃣ Initial Install → Revision 1

```bash
helm install ecommerce-prod ./ecommerce-chart -n prod
```

* Helm creates the release
* Revision number starts at **1**
* Release status becomes `deployed`

---

### 2️⃣ Upgrade → New Revision

```bash
helm upgrade ecommerce-prod ./ecommerce-chart \
  -n prod \
  -f values-prod.yaml
```

* Helm re-renders templates
* Applies updated manifests
* Revision increments to **2** (then 3, 4, … on future upgrades)

Every `helm upgrade` always creates a **new revision**.

---

### 3️⃣ Viewing Release History

```bash
helm history ecommerce-prod -n prod
```

Example output:

```text
REVISION  UPDATED                   STATUS      DESCRIPTION
1         Mon Jan 10 10:00:00        superseded  Install complete
2         Tue Jan 11 14:20:00        deployed    Upgrade complete
```

This shows the **complete revision history** of the release.

---

### 4️⃣ Rollback Using a Revision Number

```bash
helm rollback ecommerce-prod 1 -n prod
```

What happens during rollback:

* Helm reapplies manifests from revision `1`
* Rollback itself creates a **new revision** (e.g., revision 3)
* No history is deleted

---

### 5️⃣ What Helm Stores Per Revision

For every revision, Helm stores:

* Chart name and chart version
* Values used for that revision
* Rendered Kubernetes manifest files
* Release metadata (timestamps, status, description)

This data is stored as Kubernetes **Secrets** (or ConfigMaps in older setups).

---

## Common Issues / Errors

* Confusing **release revision** with chart version
* Assuming `helm uninstall` creates a revision (it does not)
* Rolling back without checking history
* Accumulating too many stored revisions

---

## Troubleshooting / Fixes

* Always inspect history before rollback:

  ```bash
  helm history <release-name> -n <namespace>
  ```
* Limit stored revisions to avoid secret bloat:

  ```bash
  helm upgrade --history-max 5 <release-name> ./chart
  ```
* Use consistent CI/CD pipelines to manage upgrades

---

## Blue–Green Context (Relation to Release Versioning)

In Helm-based blue–green deployments:

* **Blue and Green are separate Helm releases** (e.g., `app-blue`, `app-green`)
* Each release has its **own independent revision history**
* Release versioning applies **within a release**, not across releases

Examples:

* `app-blue` → revisions 1, 2, 3
* `app-green` → revisions 1, 2

Important clarifications:

* Helm release versioning **does not control traffic switching**
* Rollback using `helm rollback` reverts a **release revision**, not user traffic
* Traffic routing (Ingress / Load Balancer) is handled separately

This separation allows safe upgrades, independent rollbacks, and clean blue–green deployments.

---

## Best Practices

* Treat revisions as **immutable deployment records**
* Always use `helm history` before rollback
* Do not manually delete Helm secrets
* Limit revision history in high-frequency deployments
* Clearly distinguish:

  * Release revision
  * Chart version
  * Application version

---
---

