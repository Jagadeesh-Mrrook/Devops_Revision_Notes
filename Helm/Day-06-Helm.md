## Concept / What

**Helm Template (`helm template`)** is a Helm command used to **render Helm charts into plain Kubernetes YAML manifests locally**, without installing anything into a Kubernetes cluster.

In simple terms, it shows **what Kubernetes YAML Helm will generate** after combining:

* Templates (`templates/` directory)
* Values (`values.yaml` and environment-specific values files)
* Built-in objects (`.Values`, `.Release`, `.Chart`, etc.)

---

## Why / Purpose / Real Use Case

### Why `helm template` is important

* To **verify final Kubernetes manifests before deployment**
* To catch **template errors, indentation issues, missing values** early
* To ensure environment-specific values (dev/qa/prod) are applied correctly
* To avoid debugging failures directly in a live EKS cluster

### Real-world use cases

* Used in **CD pipelines** before deploying to EKS
* Used by DevOps teams to review rendered YAML during code reviews
* Used in CI/CD validation stages along with `helm lint`
* Used to share rendered YAML with security or platform teams

`helm template` is a **safe, non-destructive validation step**.

---

## How it Works / Steps / Syntax

### Basic Command

```bash
helm template myapp .
```

What Helm does internally:

1. Reads `Chart.yaml`
2. Loads `values.yaml`
3. Merges any override values files
4. Processes files in `templates/`
5. Outputs fully rendered Kubernetes YAML

⚠️ No Kubernetes cluster access is required.

---

### Using Environment-Specific Values

```bash
helm template myapp . -f values-dev.yaml
```

Helm merges values in this order:

1. `values.yaml`
2. `values-dev.yaml` (overrides defaults)

---

### Example Helm Template

**templates/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: app
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

**values.yaml**

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.25"
```

---

### Rendered Output (`helm template` result)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: app
          image: "nginx:1.25"
```

This is **exactly what Kubernetes will receive** during deployment.

---

## Usage in CI/CD (Jenkins – Real World)

### Where it is used

* **Only in the CD pipeline**, not CI
* Executed on Jenkins agents (not developer laptops)
* Run **before selecting Kubernetes context**

### Typical CD Stage Order

1. Checkout Helm repository
2. `helm lint`
3. `helm template`
4. Select EKS context (`kubectl config use-context`)
5. Dry run
6. Deploy

### Jenkins CD Example (Conceptual)

```bash
helm template myapp . -f values-${ENV}.yaml
```

Behavior:

* If rendering fails → CD pipeline stops
* If rendering succeeds → pipeline proceeds to dry-run and deployment

This ensures **only valid manifests reach the cluster**.

---

## Common Issues / Errors

* Invalid YAML due to incorrect indentation
* Missing values referenced in templates
* Helper template not found (`include` errors)
* Conditional blocks rendering empty objects
* Incorrect use of `indent` vs `nindent`

---

## Troubleshooting / Fixes

* Redirect output to a file for inspection:

  ```bash
  helm template myapp . > rendered.yaml
  ```
* Validate rendered YAML:

  ```bash
  kubectl apply --dry-run=client -f rendered.yaml
  ```
* Always provide default values in `values.yaml`
* Use `nindent` for proper YAML alignment

---

## Best Practices

* Always run `helm template` before deployment
* Use environment-specific values files
* Keep Helm charts separate from application code
* Fail CD pipeline immediately on template errors
* Never debug Helm templates directly in production clusters

---
---

## Concept / What

**Helm Lint (`helm lint`)** is a Helm command used to **statically validate a Helm chart**. It checks the **chart structure, syntax, and best practices** without rendering templates or connecting to a Kubernetes cluster.

In simple terms, it answers:

> “Is this Helm chart written correctly as a Helm chart?”

---

## Why / Purpose / Real Use Case

### Why `helm lint` is important

* To catch **errors early** before rendering or deployment
* To validate chart structure and required files
* To prevent broken or invalid charts from entering the CD pipeline
* To enforce Helm-recommended best practices

### Real-world use cases

* Used as the **first validation gate** in CD pipelines
* Prevents unnecessary execution of later stages like rendering or dry-run
* Helps DevOps teams maintain consistent and clean Helm charts

`helm lint` is a **fast and low-cost validation step**.

---

## How it Works / Steps / Syntax

### Basic Command

```bash
helm lint .
```

What Helm validates:

* `Chart.yaml` (required fields, format)
* `values.yaml` (YAML correctness)
* Template syntax (Go templating)
* Chart directory structure
* Deprecated or discouraged patterns

⚠️ No Kubernetes cluster access is required.

---

### Linting with Environment-Specific Values

```bash
helm lint . -f values-dev.yaml
```

Why this matters:

* Some template errors appear **only after values are applied**
* Ensures environment-specific overrides do not break templates

---

## Usage in CI/CD (Jenkins – Real World)

### Where it is used

* **Only in the CD pipeline**
* Immediately after checking out the Helm repository
* Before `helm template`

### Typical CD Stage Order

1. Checkout Helm repository
2. `helm lint`
3. `helm template`
4. Select Kubernetes context
5. Dry-run
6. Deploy

### Jenkins CD Example (Conceptual)

```bash
helm lint .
```

Behavior:

* ❌ Lint failure → CD pipeline stops
* ✅ Lint success → pipeline continues

This ensures **only valid Helm charts move forward**.

---

## Common Issues / Errors

* Missing mandatory fields in `Chart.yaml` (name, version, apiVersion)
* Invalid YAML indentation in templates or values files
* Syntax errors in Go templating (`{{ }}`)
* Incorrect chart directory structure
* Deprecated fields or patterns

---

## Troubleshooting / Fixes

* Fix errors reported by `helm lint` output
* Validate indentation in `values.yaml`
* Ensure all templates are inside the `templates/` directory
* Provide default values for all referenced variables
* Optionally use stricter checks:

  ```bash
  helm lint --strict
  ```

---

## Best Practices

* Always run `helm lint` before `helm template`
* Treat lint errors as **hard failures** in CD
* Use environment-specific values during linting
* Keep Helm charts separate from application source code
* Never skip linting for production deployments

---
---

## Debugging Helm Templates

**Helm Lint (`helm lint`)** is a Helm command used to **statically validate a Helm chart**. It checks the **chart structure, syntax, and best practices** without rendering templates or connecting to a Kubernetes cluster.

In simple terms, it answers:

> “Is this Helm chart written correctly as a Helm chart?”

---

## Why / Purpose / Real Use Case

### Why `helm lint` is important

* To catch **errors early** before rendering or deployment
* To validate chart structure and required files
* To prevent broken or invalid charts from entering the CD pipeline
* To enforce Helm-recommended best practices

### Real-world use cases

* Used as the **first validation gate** in CD pipelines
* Prevents unnecessary execution of later stages like rendering or dry-run
* Helps DevOps teams maintain consistent and clean Helm charts

`helm lint` is a **fast and low-cost validation step**.

---

## How it Works / Steps / Syntax

### Basic Command

```bash
helm lint .
```

What Helm validates:

* `Chart.yaml` (required fields, format)
* `values.yaml` (YAML correctness)
* Template syntax (Go templating)
* Chart directory structure
* Deprecated or discouraged patterns

⚠️ No Kubernetes cluster access is required.

---

### Linting with Environment-Specific Values

```bash
helm lint . -f values-dev.yaml
```

Why this matters:

* Some template errors appear **only after values are applied**
* Ensures environment-specific overrides do not break templates

---

## Usage in CI/CD (Jenkins – Real World)

### Where it is used

* **Only in the CD pipeline**
* Immediately after checking out the Helm repository
* Before `helm template`

### Typical CD Stage Order

1. Checkout Helm repository
2. `helm lint`
3. `helm template`
4. Select Kubernetes context
5. Dry-run
6. Deploy

### Jenkins CD Example (Conceptual)

```bash
helm lint .
```

Behavior:

* ❌ Lint failure → CD pipeline stops
* ✅ Lint success → pipeline continues

This ensures **only valid Helm charts move forward**.

---

## Common Issues / Errors

* Missing mandatory fields in `Chart.yaml` (name, version, apiVersion)
* Invalid YAML indentation in templates or values files
* Syntax errors in Go templating (`{{ }}`)
* Incorrect chart directory structure
* Deprecated fields or patterns

---

## Troubleshooting / Fixes

* Fix errors reported by `helm lint` output
* Validate indentation in `values.yaml`
* Ensure all templates are inside the `templates/` directory
* Provide default values for all referenced variables
* Optionally use stricter checks:

  ```bash
  helm lint --strict
  ```

---

## Best Practices

* Always run `helm lint` before `helm template`
* Treat lint errors as **hard failures** in CD
* Use environment-specific values during linting
* Keep Helm charts separate from application source code
* Never skip linting for production deployments

---
---

## Helm Dry Run (`--dry-run`)

### What

A **Helm dry run** is used to **simulate a Helm deployment** by sending the rendered Kubernetes manifests to the **Kubernetes API server for validation**, without actually creating or updating any resources.

Helm performs rendering, sends the request to the API server, gets it validated, and stops there — **nothing is persisted in etcd**.

---

### Why / Purpose / Real Use Case

* To validate manifests **against the actual Kubernetes cluster**
* To catch errors that `helm template` cannot detect
* To ensure API versions, CRDs, and permissions are correct
* To act as the **final safety gate** before real deployment

In real-world CI/CD pipelines, dry run is the **last validation step** before production deployment.

---

### How it Works / Steps / Syntax

#### Basic Command

```bash
helm upgrade --install myapp-dev . \
  -f values-dev.yaml \
  --dry-run
```

What happens internally:

1. Helm renders all templates
2. Helm sends the manifests to the Kubernetes API server
3. API server validates schema, permissions, and API compatibility
4. API server does **not** store anything in etcd

---

### Dry Run vs Helm Template

| Aspect                    | `helm template` | `helm --dry-run` |
| ------------------------- | --------------- | ---------------- |
| Renders YAML              | Yes             | Yes              |
| Talks to API server       | No              | Yes              |
| Validates against cluster | No              | Yes              |
| Creates resources         | No              | No               |

---

### Usage in CI/CD (Jenkins)

* Used **only in the CD pipeline**
* Executed **after selecting the correct Kubernetes context**
* Runs after `helm lint` and `helm template`

Typical order:

1. Helm lint
2. Helm template
3. Select EKS context
4. Helm dry run
5. Deploy

If dry run fails, the CD pipeline **stops immediately**.

---

### Common Errors Caught by Dry Run

* Invalid or deprecated API versions
* Missing CRDs
* RBAC or permission issues
* Invalid resource specifications
* Namespace-related issues

---

### Best Practices

* Always run dry run before real deployments
* Never skip dry run for QA, UAT, or PROD
* Use environment-specific values files
* Treat dry run failures as **hard failures**
* Use `helm upgrade --install` consistently

---
---

