# Helm Dependency Management

## charts/ directory

The **`charts/` directory** is used by Helm to **store dependency charts (subcharts)** that are required by a parent chart.

Important real-world clarification:

* You **do NOT manually place charts** in this directory
* This directory is **automatically populated by Helm**
* It contains packaged charts (`.tgz`) downloaded based on `Chart.yaml`

Example structure:

```text
myapp/
├── Chart.yaml
├── values.yaml
├── templates/
└── charts/
    └── redis-17.3.2.tgz
```

In real-world microservice architectures:

* This directory is often **empty**
* Because dependencies are usually deployed as **separate Helm charts**

---

## Declaring dependencies

Dependencies are declared **only in `Chart.yaml`**, not in templates, helpers, or the `charts/` directory.

Example:

```yaml
apiVersion: v2
name: myapp
version: 0.1.0

dependencies:
  - name: redis
    version: 17.3.2
    repository: https://charts.bitnami.com/bitnami
```

Key points:

* This is only a **declaration**, nothing is downloaded yet
* Dependencies represent **separate long-running services**
* Init containers and sidecars are **NOT dependencies**

Real-world practice:

* Most microservices **do not declare dependencies**
* Each microservice has its **own Helm chart and CI/CD pipeline**
* Shared services (Redis, DB, Kafka) are deployed independently

---

## Updating dependencies

Updating dependencies means **downloading or refreshing the dependency charts** declared in `Chart.yaml`.

This is required when:

* A dependency version is added or changed
* `Chart.yaml` is modified
* The `charts/` directory is empty or outdated

In real-world CI/CD pipelines:

* This step is usually automated
* Dependencies are fetched during build or deploy stages

---

## helm dependency update

The `helm dependency update` command:

```bash
helm dependency update
```

What it does:

* Reads dependencies from `Chart.yaml`
* Downloads the required charts
* Stores them in the `charts/` directory

Example result:

```text
charts/
├── redis-17.3.2.tgz
└── mysql-9.4.1.tgz
```

Important real-world notes:

* This command is **not mandatory** if you are not using dependencies
* In most production microservice setups, this command is **rarely used**
* It is common in:

  * Monoliths
  * Dev / PoC setups
  * App-specific bundled components

---

## Best Practices

* Treat Helm dependencies as **optional**, not default
* Do not bundle shared infrastructure into app charts
* Use separate Helm charts per microservice
* Keep CI/CD pipelines independent
* Never manually edit the `charts/` directory

---
---

## Declaring dependencies

### Concept / What

Declaring dependencies in Helm means **specifying which other Helm charts your chart depends on**. These dependencies are defined **only in `Chart.yaml`** using the `dependencies:` section.

This step is a **declaration of intent**. At this stage, Helm only knows *what* your chart depends on; it does not download or install anything yet.

---

### Why / Purpose / Real Use Case

Dependencies are declared when:

* Your application requires another **long-running service** to function
* The dependency lifecycle is **tightly coupled** with the application
* You want a **single Helm workflow** to manage both

Typical use cases:

* Application + app-specific Redis
* Application + bundled database for dev / PoC
* Small or monolithic deployments

Real-world microservices context:

* Declaring dependencies is **optional**
* Most production microservices use **separate Helm charts and separate CI/CD pipelines**
* Shared infrastructure (Redis, DB, Kafka) is usually deployed independently

---

### How it Works / Steps / Syntax

#### Step 1: Declare dependency in `Chart.yaml`

```yaml
apiVersion: v2
name: myapp
version: 0.1.0

dependencies:
  - name: redis
    version: 17.3.2
    repository: https://charts.bitnami.com/bitnami
```

Meaning of fields:

* `name` → Dependency chart name
* `version` → Exact chart version to use
* `repository` → Helm repository URL to download from

At this point:

* Nothing is downloaded
* Helm only records the dependency definition

---

#### Step 2: Fetch dependencies

```bash
helm dependency update
```

Helm actions:

* Reads dependencies from `Chart.yaml`
* Uses the repository URL
* Downloads the dependency charts
* Stores them in the **`charts/` directory**

Example result:

```text
charts/
└── redis-17.3.2.tgz
```

---

#### Step 3: Install parent chart

```bash
helm install myapp ./myapp-chart
```

Helm installs:

* Parent chart
* All declared dependency charts automatically

---

### What Is NOT a Dependency (Important)

The following should **not** be declared as dependencies:

* Init containers
* Sidecar containers
* One-time setup or migration jobs

Reason:

* They run inside the **same Pod**
* Helm dependencies are meant for **separate, long-running services**

---

### Common Issues / Errors

* Assuming dependencies are mandatory
* Forgetting to run `helm dependency update`
* Declaring shared infrastructure as dependencies
* Editing files inside the `charts/` directory manually

---

### Troubleshooting / Fixes

* Dependency not found during install:

  ```bash
  helm dependency update
  ```
* Values not applied to dependency:

  * Check correct dependency name in `values.yaml`
* Deployment coupling issues:

  * Remove dependency and use separate charts

---

### Best Practices

* Declare dependencies **only when lifecycles are tightly coupled**
* Avoid dependencies in production microservice setups
* Pin exact dependency versions
* Treat `Chart.yaml` as the **single source of truth**
* Prefer separate Helm charts + CI/CD for shared services

---
---

## Updating dependencies

### Concept / What

Updating dependencies in Helm means **downloading or refreshing the dependency charts** that are declared in `Chart.yaml` so they are available locally inside the `charts/` directory.

This step ensures that the **actual dependency charts match the versions declared** in `Chart.yaml`.

---

### Why / Purpose / Real Use Case

Updating dependencies is required when:

* A new dependency is added in `Chart.yaml`
* A dependency version is changed
* The `charts/` directory is empty (fresh Git clone)
* CI/CD pipelines pull only source code without bundled charts

Real-world scenarios:

* Developers update Redis or database chart versions
* CI/CD pipelines need reproducible deployments
* Fresh environment setup (dev / QA / prod)

Without updating dependencies:

* Helm install may fail
* Required subcharts may be missing
* Incorrect dependency versions may be used

---

### How it Works / Steps / Syntax

#### Step 1: Dependencies already declared

Dependencies must exist in `Chart.yaml`:

```yaml
dependencies:
  - name: redis
    version: 17.3.2
    repository: https://charts.bitnami.com/bitnami
```

---

#### Step 2: Run dependency update command

```bash
helm dependency update
```

Helm actions:

* Reads dependency definitions from `Chart.yaml`
* Uses repository URLs
* Downloads required dependency charts
* Stores them in the `charts/` directory

Example result:

```text
charts/
└── redis-17.3.2.tgz
```

---

#### Step 3: Behavior when dependencies already exist

* Helm verifies versions
* Downloads only missing or updated charts
* Reuses existing correct versions

---

### What Updating Dependencies Does NOT Do

* Does NOT install the application
* Does NOT deploy anything to Kubernetes
* Does NOT modify running Helm releases

It only prepares dependency charts locally.

---

### Common Issues / Errors

* Forgetting to run `helm dependency update`
* Invalid or unreachable repository URLs
* Version mismatches in `Chart.yaml`
* Assuming Helm auto-downloads dependencies

---

### Troubleshooting / Fixes

* Dependency missing during install:

  ```bash
  helm dependency update
  ```
* Repository not available:

  ```bash
  helm repo add <name> <url>
  helm repo update
  ```
* Wrong dependency version:

  * Verify version in `Chart.yaml`

---

### Best Practices

* Run dependency update in CI/CD pipelines
* Pin exact dependency versions
* Treat `charts/` directory as generated content
* Avoid committing dependency `.tgz` files unless required
* Keep dependency updates out of live production fixes

---
---

## helm dependency update

### Concept / What

`helm dependency update` is the Helm command used to **download and synchronize dependency charts** that are declared in `Chart.yaml`. It ensures that all required subcharts are present locally inside the `charts/` directory.

In simple terms:

> It syncs what is declared in `Chart.yaml` with what exists in `charts/`.

---

### Why / Purpose / Real Use Case

This command is required because:

* `Chart.yaml` only **declares** dependencies
* Helm does **not automatically download** dependencies during `helm install`
* CI/CD pipelines usually clone only source code, not dependency artifacts

Real-world scenarios:

* Fresh Git clone where `charts/` is empty
* Adding a new dependency
* Changing a dependency version
* Preparing builds in CI/CD pipelines

Without running this command:

* Helm install may fail
* Required dependency charts may be missing
* Incorrect versions may be used

---

### How it Works / Steps / Syntax

#### Step 1: Dependency already declared

```yaml
dependencies:
  - name: redis
    version: 17.3.2
    repository: https://charts.bitnami.com/bitnami
```

At this stage:

* Helm knows the dependency definition
* Nothing is downloaded yet

---

#### Step 2: Run the command

```bash
helm dependency update
```

Helm performs the following actions:

* Reads dependencies from `Chart.yaml`
* Resolves versions
* Downloads charts from the specified repositories
* Stores them as `.tgz` files in the `charts/` directory

Example result:

```text
charts/
└── redis-17.3.2.tgz
```

---

#### Step 3: If dependencies already exist

* Helm checks existing versions
* Downloads only **missing or updated** charts
* Leaves correct versions untouched

---

### What This Command Does NOT Do

* Does NOT install the application
* Does NOT deploy anything to Kubernetes
* Does NOT upgrade an existing release

It only prepares dependency charts locally.

---

### Common Issues / Errors

* Forgetting to run `helm dependency update` after editing `Chart.yaml`
* Invalid or unreachable repository URLs
* Expecting Helm install to auto-download dependencies
* Network or proxy issues in CI/CD environments

---

### Troubleshooting / Fixes

* Dependency missing during install:

  ```bash
  helm dependency update
  ```
* Repository not found:

  ```bash
  helm repo add <name> <url>
  helm repo update
  ```
* Wrong version downloaded:

  * Verify dependency version in `Chart.yaml`

---

### Best Practices

* Always run `helm dependency update` in CI/CD pipelines
* Pin exact dependency versions in `Chart.yaml`
* Treat the `charts/` directory as generated content
* Avoid committing dependency `.tgz` files unless required
* Skip this command entirely if your chart has no dependencies

---
---
---
