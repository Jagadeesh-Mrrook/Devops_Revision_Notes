# Installing and Upgrading Releases

## Helm Install

### What

`helm install` is used to **create a new Helm release** in a Kubernetes cluster from a Helm chart. It installs the application **for the first time** under a specific **release name** in a given **namespace**.

Each successful install creates **revision 1** for that release.

---

### Why / Purpose / Real Use Case

* Used when deploying an application **for the first time** to an environment
* Used while **bootstrapping new environments** (dev / qa / prod)
* Helps Helm start tracking release history and revisions

In real-world CI/CD, `helm install` is mainly used for **initial setup**, not for day-to-day deployments.

---

### How it Works / Steps / Syntax

#### Basic Command

```bash
helm install myapp-dev . -f values-dev.yaml
```

What happens internally:

1. Helm renders all templates
2. Helm sends manifests to the Kubernetes API server
3. Kubernetes creates the resources
4. Helm stores the release with **revision = 1**

---

#### Installing into a Namespace

```bash
helm install myapp-dev . -n dev --create-namespace
```

* `myapp-dev` → Helm **release name**
* `dev` → Kubernetes **namespace**

---

### Usage in CI/CD (Jenkins)

* Rarely used in regular CD pipelines
* Mostly used for:

  * First-time deployments
  * Environment bootstrapping scripts

Most production pipelines prefer:

```bash
helm upgrade --install
```

to avoid handling install vs upgrade separately.

---

### Common Issues / Errors

* Release name already exists
* Namespace does not exist (without `--create-namespace`)
* Missing required values
* RBAC permission issues

---

### Best Practices

* Use `helm install` only for first-time deployments
* Do not use `helm install` for upgrades
* Prefer `helm upgrade --install` in automation
* Use meaningful release names per environment

---
---

## Helm Upgrade

### What

`helm upgrade` is used to **update an existing Helm release** by applying new values, updated templates, or a new application version. Each successful upgrade creates a **new revision** of the same release.

`helm upgrade` works only when the release **already exists**.

---

### Why / Purpose / Real Use Case

* Deploy a **new application version** (new Docker image tag)
* Update configuration values
* Modify Kubernetes manifests via Helm templates
* Perform deployment strategies like **rolling update** or **blue–green**

In real-world CI/CD pipelines, `helm upgrade` is the **primary deployment command**.

---

### How it Works / Steps / Syntax

#### Basic Command

```bash
helm upgrade myapp-dev . -f values-dev.yaml
```

What happens internally:

1. Helm renders templates with updated values
2. Helm compares with the current release state
3. Helm sends updated manifests to the Kubernetes API server
4. Kubernetes updates resources (rolling update by default)
5. Helm stores a new revision (2, 3, …)

---

### Namespace Usage

```bash
helm upgrade myapp-dev . -n dev
```

* Release name remains the same
* Namespace must match the original install

---

### Blue–Green Deployment with Helm Upgrade

> Helm does **not** provide a special blue–green command. Blue–green is achieved through **chart design and values**.

#### Core Idea

* One Helm release per environment
* Two Kubernetes Deployments (blue & green)
* Service or Ingress controls traffic using labels

#### Example Values Override

```bash
helm upgrade myapp-dev . --set color=green,image.tag=v2
```

How traffic switches:

* Helm applies overridden values (`--set` has highest precedence)
* Service selector or Ingress rules are re-rendered
* Traffic moves from blue to green

Helm charts remain unchanged; **only values change**.

---

### Usage in CI/CD (Jenkins)

Typical CD flow:

1. Checkout Helm repo
2. `helm lint`
3. `helm template`
4. `helm upgrade --install --dry-run`
5. `helm upgrade --install`

`helm upgrade` is executed only after validations pass.

---

### Common Issues / Errors

* Release does not exist
* Wrong namespace specified
* Invalid or missing values
* Failed rolling updates

---

### Best Practices

* Use `helm upgrade --install` in automation
* Avoid changing chart templates for traffic switching
* Use values or `--set` for environment-specific behavior
* Keep one release per environment
* Never use `helm install` repeatedly for updates

---
---

## Helm Rollback

### What

`helm rollback` is used to **revert a Helm release to a previous revision**. It restores the application, configuration, and Kubernetes manifests exactly as they existed in the selected revision.

A rollback is treated as a **new revision** in Helm history.

---

### Why / Purpose / Real Use Case

* Recover quickly from a failed deployment
* Revert application to a previously known stable version
* Handle production incidents safely
* Avoid manual reconfiguration during failures

In real-world CI/CD and production operations, `helm rollback` is the **recommended and safest rollback mechanism**.

---

### How it Works / Steps / Syntax

#### View Release History

```bash
helm history myapp-dev
```

This shows all revisions with their status.

---

#### Roll Back to a Previous Revision

```bash
helm rollback myapp-dev 3
```

What happens internally:

1. Helm re-applies manifests from revision `3`
2. Helm sends updated specs to the Kubernetes API server
3. Kubernetes creates pods for the old version
4. Kubernetes automatically terminates pods from the failed/current version
5. Helm records the rollback as a **new revision**

⚠️ No manual pod deletion is required.

---

### Rollback Timing Behavior

* Rollback is **not instantaneous**
* Pods are recreated and must pass readiness checks
* Time depends on image pull and startup duration

---

### Usage in CI/CD (Jenkins)

* Typically executed manually during incidents
* Can also be automated as a rollback stage
* Used when post-deployment checks or monitoring detect failures

Example:

```bash
helm rollback myapp-dev <previous-revision>
```

---

### Rollback vs Upgrade with Old Values

| Method                         | Behavior                                |
| ------------------------------ | --------------------------------------- |
| `helm rollback`                | Reverts to an existing revision         |
| `helm upgrade` with old values | Creates a new revision using old config |

`helm rollback` preserves release history and intent.

---

### Common Issues / Errors

* Rolling back to a non-existent revision
* Insufficient permissions to update resources
* Rollback blocked by failing readiness probes

---

### Best Practices

* Prefer `helm rollback` over manual fixes
* Do not delete pods manually before rollback
* Keep multiple previous revisions for safety
* Monitor application health after rollback

---
---

## Helm History

### What

`helm history` displays the **revision history of a Helm release**. Each revision represents a state of the release created by an **install, upgrade, or rollback** operation.

Helm assigns an incremental **revision number** to every change it makes to a release.

---

### Why / Purpose / Real Use Case

* To track **what versions of an application were deployed**
* To identify **failed or unstable deployments**
* To decide **which revision to roll back to**
* To audit deployment changes in production environments

In real-world operations, `helm history` is the **first command run during an incident**.

---

### How it Works / Steps / Syntax

#### Basic Command

```bash
helm history myapp-dev
```

---

### Example Output (Simplified)

```text
REVISION  UPDATED                   STATUS       CHART        DESCRIPTION
1         2025-01-01 10:00:00       superseded   myapp-1.0    Install complete
2         2025-01-02 11:00:00       superseded   myapp-1.1    Upgrade complete
3         2025-01-03 12:00:00       failed       myapp-1.2    Upgrade failed
4         2025-01-03 12:10:00       deployed     myapp-1.1    Rollback to 2
```

---

### Important Fields Explained

* **REVISION** – Sequential number used by Helm
* **STATUS** – State of the revision (`deployed`, `superseded`, `failed`)
* **DESCRIPTION** – Action that created the revision

---

### Relationship with Rollback

* Rollback requires a **revision number** from `helm history`
* Rollback itself creates a **new revision**

Example:

```bash
helm rollback myapp-dev 2
```

---

### Usage in CI/CD and Operations

Typical production recovery flow:

1. Deployment fails or application misbehaves
2. Run `helm history` to inspect revisions
3. Identify last known stable revision
4. Execute `helm rollback`

---

### Key Rules

* History is maintained **per release**
* Revisions are **not deleted automatically**
* Multiple rollbacks and upgrades increase revision count

---

### Best Practices

* Always check `helm history` before rollback
* Keep multiple past revisions for safety
* Do not manually delete Helm release history
* Use meaningful release names for clarity

---
---

## Helm Uninstall

### What

`helm uninstall` is used to **completely remove a Helm release** from a Kubernetes cluster. It deletes **all Kubernetes resources** created by the release and removes the **Helm release metadata and history**.

After uninstalling, the release no longer exists in Helm.

---

### Why / Purpose / Real Use Case

* Decommission an application
* Clean up unused or failed releases
* Tear down temporary environments (dev / preview environments)
* Free the release name for reuse

In real-world operations, `helm uninstall` is used **intentionally and carefully**, especially in production.

---

### How it Works / Steps / Syntax

#### Basic Command

```bash
helm uninstall myapp-dev
```

What happens internally:

1. Helm identifies all Kubernetes resources created by the release
2. Helm sends delete requests to the Kubernetes API server
3. Kubernetes deletes the resources
4. Helm removes the release record and revision history

---

### Uninstall from a Specific Namespace

```bash
helm uninstall myapp-dev -n dev
```

* Namespace must match the namespace where the release was installed

---

### Important Behavior (Very Important)

* Rollback is **NOT possible** after uninstall
* `helm history` will no longer show the release
* All revisions are permanently removed
* Release name becomes available for reuse

`helm uninstall` is a **destructive operation**.

---

### Usage in CI/CD and Operations

Used mainly for:

* Environment cleanup jobs
* Manual operational tasks
* Deleting preview or temporary deployments

Not commonly used in automated production CD pipelines.

---

### Common Issues / Errors

* Release not found
* Wrong namespace specified
* Insufficient RBAC permissions

---

### Best Practices

* Double-check release name and namespace before uninstalling
* Avoid using uninstall in automated production flows
* Prefer rollback over uninstall for recovery
* Use uninstall only when the application is no longer needed

---
---

## How Helm Stores Release History

### What

Helm stores **release state and revision history inside the Kubernetes cluster** so it can manage upgrades, rollbacks, and history tracking.

In **Helm v3**, this information is stored as **Kubernetes Secrets** by default.

---

### Where Helm Stores the Data

For a release:

* **Release name**: `myapp-dev`
* **Namespace**: `dev`

Helm stores release history as **Secrets in the same namespace**:

```text
Namespace: dev
Resource type: Secret
```

Secret naming pattern:

```text
sh.helm.release.v1.<release-name>.v<revision>
```

Example:

```text
sh.helm.release.v1.myapp-dev.v1
sh.helm.release.v1.myapp-dev.v2
sh.helm.release.v1.myapp-dev.v3
```

Each secret represents **one revision** of the Helm release.

---

### What Is Stored Inside the Secret

Each revision secret contains:

* Rendered Kubernetes manifests
* Values used for that revision
* Chart metadata
* Release status (`deployed`, `failed`, `superseded`)
* Revision number

This stored data allows Helm to restore exact previous states.

---

### Why Helm Stores History This Way

* No external database required
* State is kept **cluster-local**
* Namespace isolation per environment
* Works naturally with Kubernetes RBAC
* Enables safe and accurate rollbacks

---

### Relationship with Helm Commands

| Command          | How it uses stored history         |
| ---------------- | ---------------------------------- |
| `helm upgrade`   | Creates a new revision secret      |
| `helm history`   | Reads revision secrets             |
| `helm rollback`  | Re-applies data from an old secret |
| `helm uninstall` | Deletes all revision secrets       |

---

### Important Behaviors

* Every upgrade or rollback increases the revision count
* Rollback does **not** delete old history
* Uninstall removes all stored secrets
* Losing these secrets means rollback is not possible

---

### CI/CD and Operations Relevance

* Jenkins rollbacks depend on these secrets
* RBAC can restrict access to rollback operations
* Backup of cluster state implicitly backs up Helm history

---

### Best Practices

* Avoid manually deleting Helm secrets
* Restrict access to Helm secrets using RBAC
* Keep multiple revisions for safer rollbacks
* Use meaningful release names per environment

---
---
---
