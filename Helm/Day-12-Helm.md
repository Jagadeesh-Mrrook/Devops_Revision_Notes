# Helm Best Practices – Avoid Hardcoding Values

---

## Concept / What

Avoid hardcoding values means **not embedding environment-specific or frequently changing configuration directly inside Helm templates** (`deployment.yaml`, `service.yaml`, `ingress.yaml`). Instead, templates should reference values supplied via `values.yaml`, environment-specific values files, or CI/CD overrides.

In Helm, templates define **structure**, while values define **configuration**.

---

## Why / Purpose / Real Use Case

In real-world Kubernetes and EKS environments:

* The **same Helm chart** is deployed to dev, stage, and prod
* Only configurations such as image tags, replica counts, resource limits, and hostnames change

Hardcoding values causes:

* Poor reusability of charts
* Manual template changes for every environment
* High risk of configuration drift
* CI/CD pipelines that cannot safely promote builds

Helm’s core purpose is **environment portability**, which is only possible when values are externalized.

---

## How it Works / Steps / Syntax

### ❌ Incorrect Approach (Hardcoded Values)

```yaml
# templates/deployment.yaml
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: app
          image: myrepo/app:1.0.0
```

Problems:

* Replica count is fixed
* Image tag cannot be overridden
* Same chart cannot be reused across environments

---

### ✅ Correct Helm Approach (Values-driven Templates)

#### templates/deployment.yaml

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

#### values.yaml

```yaml
replicaCount: 2

image:
  repository: myrepo/app
  tag: latest
```

Now:

* Templates remain unchanged
* Configuration changes are isolated to values files
* Same chart works for all environments

---

### Environment-Specific Values

```text
values-dev.yaml
values-stage.yaml
values-prod.yaml
```

Example override:

```bash
helm upgrade --install app \
  -f values-prod.yaml \
  --set image.tag=build-123
```

This enables **safe image promotion in CI/CD pipelines** without modifying templates.

---

## Common Issues / Errors

* Hardcoding image tags inside templates
* Embedding environment logic inside templates (if prod / dev conditions)
* Copying charts per environment instead of using values files
* Editing templates during deployments

---

## Troubleshooting / Fixes

**Issue:** Cannot reuse chart across environments
**Fix:** Move all environment-specific values to `values.yaml` or env-specific files

**Issue:** CI/CD cannot override configurations
**Fix:** Replace hardcoded values with `.Values` references

**Issue:** Accidental prod-level scaling in dev
**Fix:** Separate replica counts per environment values file

---

## Best Practices

* Keep templates **environment-agnostic**
* Externalize all change-prone values
* Override values using CI/CD pipelines
* Use defaults only for safe, non-critical settings
* Never modify templates during deployments

**Golden Rule:**

> Helm templates define structure; values define behavior.

---
---

# Helm Best Practices – Keep Defaults Minimal

---

## Concept / What

Keeping defaults minimal means designing the **default `values.yaml` as a safe baseline**, not as a shared configuration or production-ready file. It should contain only the **minimum values required for Helm templates to render correctly** and to ensure that an accidental `helm install` does not cause risk, cost, or incorrect behavior.

The default `values.yaml` exists to provide **guardrails, structure, and safety**, not real environment behavior.

---

## Why / Purpose / Real Use Case

In real Helm + EKS usage:

* The same chart is reused across dev, QA, stage, and prod
* Engineers or pipelines may run `helm install` without overrides
* Helm always applies `values.yaml` to **every environment**

If production-like values are placed in the default file:

* Dev/QA may accidentally scale like prod
* Unintended EKS cost increases can occur
* Environment behavior becomes implicit and risky

Minimal defaults ensure:

* Accidental installs are safe
* Production behavior is **explicitly defined**
* Charts remain reusable and predictable

---

## How it Works / Steps / Syntax

### Author Responsibility (Key Point)

As the Helm chart author, the DevOps engineer decides what is “minimal”. This decision is based on **safety**, not on whether a value is shared across environments.

For every value, the author evaluates:

* Will the template fail if this key is missing?
* Is this value safe if applied to all environments?
* Does this value describe structure rather than real behavior?

Only values that satisfy these conditions belong in the default `values.yaml`.

---

### Example: Safe Numeric Defaults

```yaml
replicaCount: 1
```

Why this is minimal:

* One replica is safe in all environments
* Higher numbers represent environment behavior
* Prevents accidental over-scaling

---

### Example: Preventing Template Rendering Failures

```yaml
resources: {}
```

Why this is minimal:

* Templates referencing `.Values.resources` will not fail
* Empty values do not enforce CPU or memory limits
* Real resource settings are supplied via env-specific files

---

### Example: Documenting Required Structure

```yaml
image:
  repository: ""
  tag: ""
```

Why this belongs in defaults:

* Documents required keys for the chart
* Forces environments or CI/CD to explicitly define real values
* Prevents silent misconfiguration

---

### What Should NOT Be in Default values.yaml

* Production replica counts
* CPU and memory limits
* Ingress hostnames
* Feature flags
* Autoscaling thresholds
* Secrets

Even if such values are common across environments, they still define **behavior** and must be explicitly supplied via environment-specific values files.

---

## Common Issues / Errors

* Treating `values.yaml` as a shared configuration file
* Including production-grade numbers in defaults
* Assuming defaults represent recommended settings
* Forgetting that defaults apply to all environments

---

## Troubleshooting / Fixes

**Issue:** Dev or QA scales like production
**Fix:** Reduce defaults to safe minimums and move real values to env-specific files

**Issue:** Helm template fails due to missing keys
**Fix:** Define empty or placeholder values in default `values.yaml`

**Issue:** CI/CD behavior is unpredictable
**Fix:** Ensure pipelines always supply environment-specific values files

---

## Best Practices

* Treat default `values.yaml` as a **guardrail**, not a configuration
* Choose numeric defaults based on safety, not performance
* Use empty objects (`{}`) to stabilize template rendering
* Force real environment behavior through env-specific values files
* Assume accidental installs will happen and design safely

**Golden Rule:**

> Minimal defaults are chosen to make the worst-case deployment safe, not optimal.

---
---

# Helm Best Practices – Use helpers.tpl

---

## Concept / What

`helpers.tpl` is a special Helm template file used to define **reusable named templates (helper functions)**. These helpers encapsulate common template logic such as naming conventions, labels, annotations, and formatting so that the same logic can be reused across multiple Kubernetes manifests.

The file name starts with an underscore (`_helpers.tpl`) to indicate that it **does not render Kubernetes resources by itself**.

---

## Why / Purpose / Real Use Case

In real Helm + EKS deployments:

* The same labels are required in Deployments, Services, Ingress, HPAs, and NetworkPolicies
* Naming conventions must stay consistent across all resources
* CI/CD pipelines frequently change release names, chart versions, and app versions

Without `helpers.tpl`:

* Template logic is duplicated across multiple files
* Inconsistent labels break Service selectors and monitoring
* Small changes require editing many templates

Using `helpers.tpl` provides:

* A single source of truth
* Consistent naming and labeling
* Easier maintenance and safer refactoring

This is considered a **production-quality Helm best practice**.

---

## How it Works / Steps / Syntax

### File Location

```text
templates/_helpers.tpl
```

Helm loads this file during rendering but does not create any Kubernetes objects from it.

---

### Defining a Helper (Name Generation)

```yaml
{{- define "mychart.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end -}}
```

This helper centralizes how resource names are constructed.

---

### Using the Helper in Templates

```yaml
metadata:
  name: {{ include "mychart.fullname" . }}
```

Any change to the naming logic is now applied everywhere automatically.

---

### Real-World Example: Common Labels

```yaml
{{- define "mychart.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
{{- end -}}
```

Usage in a Deployment, Service, or Ingress:

```yaml
metadata:
  labels:
{{ include "mychart.labels" . | nindent 4 }}
```

This ensures:

* Label consistency across all resources
* Correct Service selectors
* Reliable monitoring and cost attribution

---

## Common Issues / Errors

* Duplicating label and name logic across templates
* Forgetting to update all manifests after a naming change
* Misaligned labels causing Services or HPAs to fail
* Hardcoding labels instead of centralizing them

---

## Troubleshooting / Fixes

**Issue:** Service not selecting pods
**Fix:** Move selector and pod labels into a shared helper and reuse it consistently

**Issue:** Renaming release breaks some resources
**Fix:** Centralize naming logic inside `helpers.tpl`

**Issue:** Chart becomes hard to maintain
**Fix:** Refactor repeated template logic into helpers

---

## Best Practices

* Use `helpers.tpl` for names, labels, and annotations
* Keep helpers generic and reusable
* Avoid environment-specific logic in helpers
* Use `include` with `nindent` for clean YAML formatting
* Treat helpers as part of the chart’s public contract

**Golden Rule:**

> If the same template logic appears more than once, it belongs in `helpers.tpl`.

---
---

# Helm Best Practices – Separate dev / stage / prod values

---

## Concept / What

Separating dev, stage, and prod values means using **one single Helm chart** while maintaining **multiple environment-specific values files**. The chart remains unchanged, and only configuration values vary per environment.

This approach cleanly separates **application structure (templates)** from **environment behavior (values)**.

---

## Why / Purpose / Real Use Case

In real Helm + EKS environments:

* Dev, stage, and prod have different scale, performance, and exposure requirements
* The same application artifact must be promoted safely across environments
* CI/CD pipelines should deploy the same chart without modification

Without separating values:

* Charts are duplicated per environment
* Manual edits are required before deployments
* High risk of deploying incorrect configuration to production
* Rollbacks and promotions become unreliable

Separating values enables **safe promotion, reuse, and consistency**.

---

## How it Works / Steps / Syntax

### Recommended Values File Structure

```text
values.yaml          # minimal, safe defaults
values-dev.yaml      # dev-specific behavior
values-stage.yaml    # stage-specific behavior
values-prod.yaml     # prod-specific behavior
```

---

### Default values.yaml (Baseline Only)

```yaml
replicaCount: 1
resources: {}
```

Purpose:

* Provides safe defaults
* Prevents template rendering failures
* Applies to all environments

---

### Environment-Specific Values

#### Dev

```yaml
replicaCount: 1
```

#### Stage

```yaml
replicaCount: 2
```

#### Prod

```yaml
replicaCount: 5

resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
```

These files define **real environment behavior**.

---

### Helm Deployment Commands

```bash
helm upgrade --install app -f values-dev.yaml
helm upgrade --install app -f values-stage.yaml
helm upgrade --install app -f values-prod.yaml
```

Helm merges values in this order:

```text
values.yaml → env-specific values file → --set (highest priority)
```

---

## CI/CD Integration

In CI/CD pipelines:

* The environment determines which values file is used
* The same Helm chart artifact is deployed everywhere
* Only configuration changes between environments

This ensures:

* Predictable deployments
* Safe promotions
* Easy rollbacks

---

## Common Issues / Errors

* Copying Helm charts per environment
* Hardcoding environment logic inside templates
* Mixing dev and prod values in the same file
* Forgetting which values file is used in CI/CD

---

## Troubleshooting / Fixes

**Issue:** Wrong configuration applied to prod
**Fix:** Verify the correct environment-specific values file is passed

**Issue:** Chart behaves differently across environments unexpectedly
**Fix:** Audit values files for unintended overrides

**Issue:** CI/CD deployments are inconsistent
**Fix:** Enforce explicit values file usage per environment

---

## Best Practices

* Maintain a single Helm chart per application
* Keep `values.yaml` minimal and safe
* Use one values file per environment
* Avoid conditional environment logic inside templates
* Make environment selection explicit in CI/CD pipelines

**Golden Rule:**

> One chart, many environments — behavior is controlled only through values.

---
---

# Helm Best Practices – CI/CD Integration

---

## Concept / What

CI/CD integration with Helm means using Helm as the **automated deployment mechanism inside pipelines**, where builds, configuration selection, deployments, promotions, and rollbacks are executed without manual intervention.

In this model, Helm charts are treated as **immutable deployment artifacts**, and CI/CD pipelines supply only environment-specific values and runtime inputs.

---

## Why / Purpose / Real Use Case

In real Helm + EKS environments:

* Manual `helm upgrade` in production is unsafe and error‑prone
* The same application artifact must be promoted across environments
* Deployments must be repeatable, auditable, and reversible

Without CI/CD integration:

* Configuration drift occurs
* Human error increases
* Rollbacks are unreliable
* Promotions require rebuilds

CI/CD integration ensures **safe, consistent, and traceable deployments**.

---

## How it Works / Steps / Syntax

### Typical CI/CD Flow

#### 1. CI Phase

* Build application code
* Create Docker image
* Push image to container registry (e.g., ECR)
* Tag image uniquely (commit SHA / build ID)

#### 2. CD Phase

* Select target environment (dev / stage / prod)
* Choose the correct values file
* Deploy using Helm

---

### Helm Deployment Command in Pipeline

```bash
helm upgrade --install app \
  ./chart \
  -f values-prod.yaml \
  --set image.tag=build-123
```

This approach ensures:

* Same chart is reused
* Same image is promoted
* Only configuration changes per environment

---

## Promotion Model

Correct CI/CD promotion strategy:

* Build image once
* Promote the same image across environments
* Never rebuild for production

Helm enables this by separating:

* Templates (structure)
* Values (environment behavior)

---

## Rollback & Safety

Helm tracks release history automatically:

```bash
helm history app
helm rollback app 2
```

CI/CD pipelines can:

* Roll back automatically on failure
* Restore last known‑good release safely

---

## Common Issues / Errors

* Running Helm commands manually in production
* Rebuilding images for each environment
* Hardcoding image tags in templates
* Not versioning Helm releases

---

## Troubleshooting / Fixes

**Issue:** Wrong image deployed to prod
**Fix:** Ensure pipelines promote the same image tag across environments

**Issue:** Deployment fails inconsistently
**Fix:** Verify correct values file is selected per environment

**Issue:** Rollback not possible
**Fix:** Avoid manual changes outside Helm and rely on Helm release history

---

## Best Practices

* Treat Helm charts as immutable artifacts
* Control deployments only through CI/CD pipelines
* Inject image tags at runtime using `--set`
* Use environment‑specific values files
* Automate rollback strategies

**Golden Rule:**

> CI/CD pipelines supply values; Helm charts supply structure.

---
---

# Helm Best Practices – Keep Secrets Outside Helm

---

## Concept / What

Keeping secrets outside Helm means **not storing or managing sensitive data inside Helm charts or values files**. Helm should only **reference existing Kubernetes Secrets**, not contain secret values such as passwords, API keys, tokens, or certificates.

Helm is a deployment and templating tool, not a secrets management system.

---

## Why / Purpose / Real Use Case

In real Helm + EKS environments:

* Helm charts and values files are stored in Git repositories
* CI/CD pipelines may log rendered manifests
* Helm stores release data (including values) inside the cluster

If secrets are stored in Helm:

* Secrets may be committed to Git
* Secrets can leak through CI/CD logs
* Secrets are persisted in Helm release history
* Secret rotation becomes difficult and risky

This violates basic security and compliance practices.

---

## How it Works / Steps / Syntax

### Correct Design Principle

Helm templates **reference secret names**, while secrets themselves are created and managed **outside Helm**.

---

### Referencing an Existing Kubernetes Secret

#### values.yaml (non-sensitive)

```yaml
secretName: app-secrets
```

#### templates/deployment.yaml

```yaml
envFrom:
  - secretRef:
      name: {{ .Values.secretName }}
```

In this approach:

* Helm deploys the application
* Kubernetes Secret already exists
* Secret lifecycle is independent of Helm

---

## Where Secrets Should Come From

### 1. External Secrets Operator (Preferred)

* Secrets stored in AWS Secrets Manager or Parameter Store
* Automatically synced into Kubernetes Secrets
* Helm only references the secret name

This is the **recommended production approach**.

---

### 2. CI/CD Secret Injection

* CI/CD pipeline securely fetches secrets
* Creates or updates Kubernetes Secrets before Helm deployment
* Helm never receives raw secret values

---

### 3. Encrypted Files (Limited Use)

* Tools like SOPS encrypt values files
* Decryption happens at deploy time
* Useful for learning or small setups
* Less preferred than external secret managers for production

---

## What NOT to Do

* Do not store secrets in `values.yaml` or env-specific values files
* Do not hardcode secrets in templates
* Do not commit secrets to Git, even temporarily

Example of what NOT to do:

```yaml
password: mypassword
```

---

## Common Issues / Errors

* Treating Helm as a secrets manager
* Accidentally committing secrets to Git
* Exposing secrets through CI/CD logs
* Tying secret lifecycle to Helm releases

---

## Troubleshooting / Fixes

**Issue:** Secrets appear in Helm values or Git history
**Fix:** Remove secrets from Helm, rotate credentials, move to a secret manager

**Issue:** Application cannot access secrets
**Fix:** Verify secret exists and the referenced name is correct

**Issue:** Secrets leak during deployments
**Fix:** Ensure CI/CD logs do not render secret values and Helm never receives them

---

## Best Practices

* Use a dedicated secrets manager (AWS Secrets Manager, Parameter Store)
* Sync secrets to Kubernetes using External Secrets Operator
* Reference secrets by name in Helm templates
* Keep secret lifecycle independent from Helm releases
* Assume Helm values may be logged or stored

**Golden Rule:**

> Helm should reference secrets, never store or manage them.

---
---
---
