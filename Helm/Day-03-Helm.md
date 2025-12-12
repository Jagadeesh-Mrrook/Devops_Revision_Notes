# Helm Chart Structure — Detailed Notes

---

## **Concept / What**

A Helm chart is a packaged, reusable collection of Kubernetes manifests. The **chart structure** defines how files are organized inside a Helm chart, including metadata, templates, default configurations, and optional dependencies.

---

## **Why / Purpose / Real Use Case**

* Provides a **standardized layout** for Kubernetes applications.
* Enables **parameterization** via `values.yaml` so the same chart works for DEV, QA, STAGE, PROD.
* Supports **CI/CD automation**, versioning, and rollbacks.
* Helps teams enforce **consistency** across deployments on EKS.
* Makes it easy to package, publish, version, and reuse Kubernetes application deployments.

---

## **How it Works / Steps / Syntax**

A typical Helm chart contains:

```
mychart/
├── Chart.yaml          # Metadata for the chart
├── values.yaml         # Default configuration
├── templates/          # All Kubernetes templates rendered by Helm
│   ├── _helpers.tpl    # Reusable template blocks
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── NOTES.txt
├── charts/             # Dependencies (optional)
├── Chart.lock          # Dependency lock file
└── .helmignore         # Files to exclude when packaging
```

### **Chart.yaml (metadata)**

```yaml
apiVersion: v2
name: mychart
version: 0.1.0
appVersion: "1.2.3"
description: My sample chart
```

### **values.yaml (default values)**

```yaml
replicaCount: 2
image:
  repository: myrepo/myapp
  tag: "1.2.3"
service:
  type: ClusterIP
  port: 80
```

### **Template Example (deployment.yaml)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "mychart.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "mychart.name" . }}
    spec:
      containers:
        - name: {{ include "mychart.name" . }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: 80
```

### **Rendered YAML (after `helm template`)**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mychart
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mychart
  template:
    metadata:
      labels:
        app: mychart
    spec:
      containers:
        - name: mychart
          image: "myrepo/myapp:1.2.3"
          ports:
            - containerPort: 80
```

This shows how **chart structure + templates + values** combine to produce the final manifest.

---

## **Common Issues / Errors**

* Incorrect `apiVersion` in Chart.yaml.
* Missing required fields in templates (e.g., metadata.name).
* Extra empty YAML documents caused by template whitespace issues.
* Forgetting to update dependencies → broken `charts/` directory.
* Misusing global values leading to value collisions.

---

## **Troubleshooting / Fixes**

* Use **`helm lint`** to detect structural issues.
* Use **`helm template`** to inspect rendered YAML.
* Use **`--debug --dry-run`** during install/upgrade for detailed output.
* Run **`helm dependency update`** when using subcharts.
* Validate `Chart.yaml` for semantic versioning and correctness.

---

## **Best Practices**

* Keep `values.yaml` environment-agnostic; use separate values files per env.
* Use `_helpers.tpl` to centralize naming and labeling logic.
* Follow semantic versioning for `version` in Chart.yaml.
* Commit `Chart.lock` for consistent dependency builds.
* Always run lint + template steps in CI before deployment.

---
---
# Helm Chart.yaml Fields — Detailed Notes (With Integrated Clarifications)

---

## **Concept / What**

`Chart.yaml` is the metadata file of a Helm chart. It defines all the essential information about the chart itself—such as chart name, chart version, application version, maintainers, dependencies, and supported Kubernetes versions. Helm uses this file to understand *what the chart is* and *how it should be handled* during installation, upgrading, packaging, and publishing.

---

## **Why / Purpose / Real Use Case**

* Identifies and describes the Helm chart being deployed.
* Ensures correct chart version tracking in CI/CD pipelines.
* Controls dependency management for application-level components (like Redis or databases).
* Defines Kubernetes version compatibility to prevent deploying charts on unsupported clusters.
* Offers metadata used by chart repositories and documentation generators.

Real organizations rely on this metadata to ensure consistent deployments across DEV → QA → STAGE → PROD, and to maintain clear version history for rollback and auditing.

---

## **How It Works / Fields Explained With Integrated Discussion**

Below is a realistic `Chart.yaml` example:

````yaml
apiVersion: v2
name: myapp
version: 1.0.0
appVersion: "3.2.1"
description: A Helm chart for the MyApp service
type: application
kubeVersion: "```
">=1.24 <1.30"
````

maintainers:

* name: Jagga
  email: [jagga@example.com](mailto:jagga@example.com)
  keywords:
* ecommerce
* backend
  sources:
* [https://github.com/myorg/myapp](https://github.com/myorg/myapp)

dependencies:

* name: redis
  version: 17.0.1
  repository: "[https://charts.bitnami.com/bitnami](https://charts.bitnami.com/bitnami)"
  condition: redis.enabled

### **apiVersion (Chart Schema Version)**
Must be `v2` for Helm 3.

### **name**
Represents the chart name (not Kubernetes resource name).

### **version (Chart Version)**
- Uses **semantic versioning**.
- **Must be bumped** whenever *any* part of the chart changes—templates, values, helpers, manifests, or dependencies.
- This is how Helm tracks upgrades.

> Integrated clarification: In real-world CI/CD, pipelines will fail a build if the `version:` field is not bumped after a chart change. This ensures reliable rollout and rollback in EKS.

### **appVersion (Application Version)**
- Represents the application’s real version (usually Docker image tag).
- Updated when the actual application version changes.

> For example: image tag moves from `3.2.1` → `3.2.2`, update appVersion accordingly.

### **kubeVersion**
Defines which Kubernetes/EKS versions are allowed.
Supports ranges such as:
">=1.24 <1.30"

> This prevents accidental deployments on unsupported cluster versions during EKS upgrades.

### **description / maintainers / keywords / sources**
Descriptive metadata used by repositories and documentation tools.

### **dependencies**
This section lists **application-level** dependencies that your microservice needs.

Realistic examples used in production:
- Redis
- PostgreSQL
- MySQL
- MongoDB
- Kafka
- RabbitMQ
- Elasticsearch / OpenSearch
- MinIO
- Internal microservice charts (auth-service, notification-service)

> Integrated clarification: These are **application-level** dependencies. They are installed alongside your app because your service directly depends on them.

### ❌ **What is NOT added as dependencies** (Very Important)
Cluster-level tools such as:
- Prometheus Operator
- Grafana Operator
- External Secrets Operator
- NGINX Ingress Controller
- Cert-Manager
- EBS CSI Driver
- Service Mesh (Istio/Linkerd)

These are installed **once per cluster** by SRE/Platform teams—not inside your application’s Chart.yaml.

> Integrated clarification: Operators provide cluster-wide CRDs and controllers. Even though they can be installed via Helm, they should *not* be added as dependencies inside your application's chart.

---

## **Common Issues / Errors**
- Chart version not bumped → CI/CD rejects deployment.
- Incorrect semantic versioning → `helm lint` fails.
- Wrong kubeVersion range → deployment rejected by Helm.
- Dependencies not downloaded → forgot to run `helm dependency update`.
- Confusion between `version` and `appVersion`.
- Trying to include cluster-level operators as dependencies → bad practice and causes cluster conflicts.

---

## **Troubleshooting / Fixes**
- Always run `helm lint` before packaging or deploying.
- Use `helm template` or `helm install --dry-run --debug` to validate metadata and dependencies.
- Update dependency versions manually inside Chart.yaml, then run:

helm dependency update

- Keep appVersion in sync with Docker image versions.
- Validate kubeVersion constraints during Kubernetes/EKS version upgrades.

---

## **Best Practices**
- **Bump chart version for every chart change**, even small template updates.
- Separate **application-level dependencies** from **cluster-level operators**.
- Use semantic versioning for clarity and rollback.
- Keep chart metadata descriptive for maintainability.
- Define kubeVersion to avoid incompatible deployments.
- Commit `Chart.lock` when using subcharts for reproducibility.

---
---

# Helm values.yaml Usage — Detailed Notes (With Integrated Clarifications)

---

## **Concept / What**

`values.yaml` is the default configuration file in a Helm chart. It stores **all variable values** that templates use during rendering. Instead of hardcoding replicas, ports, image tags, or ingress settings into Kubernetes manifests, Helm templates dynamically reference values from this file.

It acts very similarly to **Terraform variable files** but is specifically built for Helm’s templating engine.

---

## **Why / Purpose / Real Use Case**

* Allows a **single chart** to be reused across many environments (DEV, QA, STAGE, PROD).
* Centralizes configuration so templates remain clean and generic.
* Supports CI/CD pipelines where environment-specific files override defaults.
* Ensures application consistency by grouping image settings, resources, service ports, ingress rules, and autoscaling settings cleanly.
* Prevents developers from modifying templates every time a small change (like image tag or replicas) is needed.

Real organizations maintain multiple values files such as:

```
values-dev.yaml
values-qa.yaml
values-prod.yaml
```

Helm selects the right one during deployment using `-f` flags.

---

## **How It Works / Steps / Syntax**

Below is a realistic `values.yaml`:

```yaml
replicaCount: 3

image:
  repository: myrepo/myapp
  tag: "1.0.0"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
  requests:
    cpu: "250m"
    memory: "256Mi"

ingress:
  enabled: true
  className: nginx
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
```

### **Using values in templates**

Example from `templates/deployment.yaml`:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ include "mychart.name" . }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
{{- toYaml .Values.resources | nindent 12 }}
```

### **Rendered YAML (after helm template)**

```yaml
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: myapp
          image: "myrepo/myapp:1.0.0"
          resources:
            limits:
              cpu: "500m"
              memory: "512Mi"
            requests:
              cpu: "250m"
              memory: "256Mi"
```

Values flow → templates → final manifest.

### **Selecting environment values in CI/CD**

```
helm upgrade --install myapp . -f values-prod.yaml
```

---

## **Common Issues / Errors**

* Missing keys cause rendering failures (undefined .Values paths).
* Indentation errors when using `toYaml` or nested fields.
* Wrong data types (e.g., using strings for booleans).
* Overriding values incorrectly with `--set` in CLI.
* Accidentally placing environment-specific values in the default values.yaml.

---

## **Troubleshooting / Fixes**

* Use dry-run to view final rendered manifests:

  ```
  helm install myapp . --dry-run --debug
  ```

  This prints the exact YAML Helm would install.

* Inspect release values:

  ```
  helm get values myapp
  ```

* Render templates locally without installing:

  ```
  helm template myapp .
  ```

* Check combined values used:

  ```
  helm get manifest myapp
  ```

* Ensure indentation is correct when embedding structured values.

---

## **Best Practices**

* Keep `values.yaml` **environment-agnostic**. Use separate files for prod, stage, dev.
* Group related settings (image, service, ingress, resources) into logical sections.
* Use booleans like `enabled: true` to toggle optional features.
* Never store credentials or secrets inside `values.yaml`—use External Secrets, Sealed Secrets, AWS Secrets Manager, or SSM.
* Validate rendered YAML using dry-run before deploying to EKS.

---
---

# Helm Templates (templates/*.yaml) — Detailed Notes

---

## **Concept / What**

The `templates/` directory in a Helm chart contains all the Kubernetes manifest files written using **Helm templating syntax**. These files are not static YAML; instead, they include dynamic placeholders, functions, and logic that Helm processes to generate final Kubernetes manifests. Templates pull values from `values.yaml`, metadata from `Chart.yaml`, and release information from Helm.

---

## **Why / Purpose / Real Use Case**

* Enables **dynamic configuration** without modifying YAML files directly.
* Allows a single chart to work across multiple environments by pulling from different values files.
* Avoids duplication by using helper templates.
* Supports conditional resources (e.g., only create Ingress if enabled).
* Lets organizations configure replicas, ports, image tags, hostnames, resources, and environment-specific settings via values.

Real-world EKS deployments rely on templates to consistently render Deployments, Services, Ingresses, ConfigMaps, Secrets, HPAs, etc., while keeping charts reusable and maintainable.

---

## **How It Works / Steps / Syntax**

Templates are written using **Go templating** and Helm functions. A typical template includes:

### **1. Static YAML**

```yaml
apiVersion: apps/v1
kind: Deployment
```

### **2. Dynamic fields using values**

```yaml
replicas: {{ .Values.replicaCount }}
```

### **3. Helper functions (from _helpers.tpl)**

```yaml
name: {{ include "mychart.fullname" . }}
```

### **4. Conditional logic**

```yaml
{{- if .Values.ingress.enabled }}
# ingress yaml
{{- end }}
```

### **5. Loops**

```yaml
{{- range .Values.ingress.hosts }}
- host: {{ .host }}
{{- end }}
```

### **6. YAML formatting helpers**

```yaml
resources:
{{- toYaml .Values.resources | nindent 10 }}
```

### **Where values come from**

* `.Values` → values.yaml or overridden files
* `.Chart` → Chart.yaml metadata
* `.Release` → release name, namespace, revision
* `.Capabilities` → cluster API versions & supported features

> Integrated clarification: `.Capabilities` is used to determine which Kubernetes API versions exist in the cluster. It helps charts adapt to different Kubernetes/EKS versions.

### **Understanding `include`**

* `include` executes a helper template defined inside `_helpers.tpl`.
* Example:

  ```yaml
  {{ include "mychart.name" . }}
  ```

  This calls the helper function `mychart.name` (defined in helpers.tpl) and inserts its output.

> Integrated clarification: These helper functions will be fully explained in the helpers.tpl concept. They centralize naming, labels, and repeated snippets.

---

## **Template Examples**

### **Deployment Template**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "mychart.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "mychart.name" . }}
    spec:
      containers:
        - name: {{ include "mychart.name" . }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.containerPort }}
```

### **Service Template**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
  selector:
    app: {{ include "mychart.name" . }}
```

### **Ingress Template (conditional)**

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
spec:
  rules:
  {{- range .Values.ingress.hosts }}
    - host: {{ .host }}
      http:
        paths:
        {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "mychart.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
        {{- end }}
  {{- end }}
{{- end }}
```

### **Rendered YAML Example**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mychart
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: mychart
          image: "myrepo/myapp:1.0.0"
```

---

## **Common Issues / Errors**

* Incorrect indentation when embedding YAML blocks.
* Missing values in values.yaml → undefined value errors.
* Using deprecated Kubernetes API versions.
* Wrong scope (`$` vs `.`) when using include inside loops.
* Producing empty YAML documents due to incorrect conditional blocks.

---

## **Troubleshooting / Fixes**

* Render templates locally:

  ```
  helm template myapp .
  ```
* Debug installation:

  ```
  helm install myapp . --dry-run --debug
  ```
* Validate values used by release:

  ```
  helm get values myapp
  ```
* Use `helm lint` to detect missing values, bad indentation, or syntax issues.

---

## **Best Practices**

* Keep templates minimal—use helpers for repeated logic.
* Avoid hardcoding environment-specific values.
* Use conditions (`enabled: true`) to toggle optional components.
* Use `nindent` and `toYaml` for clean, correct YAML structure.
* Verify rendered YAML before deploying to EKS.

---
---

# Helm `_helpers.tpl` — Complete, Clean & Production‑Ready Notes

This document rewrites your previous notes **from scratch**, removing all confusion and adding the complete, correct, interview‑level explanations. It includes:

* Simple explanation of helpers
* How to define & include functions
* Production‑ready naming helpers
* Production‑ready labels helpers
* Selector labels vs metadata labels
* Rendered output examples
* Best practices for 5+ YOE interviews

---

## ## Concept / What

`_helpers.tpl` is a special file inside the `templates/` directory used to create **reusable functions** in Helm. These functions return **text blocks** (names, labels, annotations, etc.) that are inserted into other template files using `include`.

In simple terms:

* **`define` = create a function**
* **`include` = call that function**
* Helpers prevent duplication across Deployment, Service, Ingress, ConfigMap, Secret, etc.
* Helpers can return **multiple lines**, not just one value.

---

## ## Why / Purpose / Real Use Case

Real Kubernetes applications require:

* Consistent naming
* Repeated labels
* Complex naming rules (environment + app + component)
* Truncation, lowercasing, prefix/suffix logic
* Identical selector labels between Deployment and Service
* Metadata labels for monitoring & governance

Without helpers:

* You repeat the same labels/names in 10+ files
* Naming changes require editing every file
* High risk of selector mismatch

With helpers:

* Update one function → all manifests update automatically
* No duplication
* No selector mismatches
* Professional & maintainable charts

---

## ## How Helpers Work

### ### 1. Define a helper function

```gotmpl
{{- define "myapp.name" -}}
{{ .Chart.Name }}
{{- end }}
```

This function returns the chart name.

### ### 2. Include (call) the helper

```yaml
name: {{ include "myapp.name" . }}
```

`include` inserts the returned text.

### ### 3. Indent the output properly

```yaml
labels:
  {{ include "myapp.labels" . | nindent 2 }}
```

`nindent` fixes YAML formatting.

---

# -----------------------------------------

# 🔥 PRODUCTION‑READY HELPERS (REAL COMPANIES USE THIS)

# -----------------------------------------

These helpers follow Kubernetes recommended labels and proper selector rules.

---

## ## Name Helper (Base Name)

```gotmpl
{{- define "myapp.name" -}}
{{ .Chart.Name }}
{{- end }}
```

Returns a simple, reusable application name.

---

## ## Full Name Helper (Environment + App + Component)

```gotmpl
{{- define "myapp.fullname" -}}
{{ printf "%s-%s-%s" .Values.environment .Chart.Name .Values.component | lower | trunc 63 | trimSuffix "-" }}
{{- end }}
```

### Uses:

* **Built‑in Object:** `.Values`
* **Functions:** `printf`, `lower`, `trunc`, `trimSuffix`

### Purpose:

* Creates consistent resource names
* Ensures names never exceed Kubernetes limits (63 chars)

---

# -----------------------------------------

# 🚦 LABEL HELPERS

# -----------------------------------------

There are **three different types of labels** in real‑world charts.

## ## 1. Selector Labels (MUST match Deployment selector + Pod labels)

```gotmpl
{{- define "myapp.selectorLabels" -}}
app: {{ include "myapp.name" . }}
component: {{ .Values.component }}
{{- end }}
```

Used in:

* `spec.selector.matchLabels`
* `spec.template.metadata.labels`
* `Service.spec.selector`

These labels **MUST match exactly** or Pods will not be selected.

---

## ## 2. Metadata Labels (Recommended Kubernetes Standard Labels)

```gotmpl
{{- define "myapp.labels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/version: {{ .Chart.AppVersion }}
helm.sh/chart: {{ .Chart.Name }}-{{ .Chart.Version }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}
```

These labels do **NOT** need to match selectors.
They are used by monitoring, tracing, dashboards, etc.

---

## ## 3. Extra Business Labels (Optional)

```gotmpl
{{- define "myapp.extraLabels" -}}
team: {{ default "devops" .Values.team }}
environment: {{ .Values.environment }}
{{- end }}
```

Used for:

* Governance
* Cost allocation
* Team ownership

---

# -----------------------------------------

# 📌 How Labels Are Used Inside Deployment.yaml

# -----------------------------------------

A typical Deployment will use helpers like this:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{ include "myapp.labels" . | nindent 4 }}

spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{ include "myapp.selectorLabels" . | nindent 6 }}

  template:
    metadata:
      labels:
        {{ include "myapp.selectorLabels" . | nindent 8 }}
        {{ include "myapp.labels" . | nindent 8 }}
        {{ include "myapp.extraLabels" . | nindent 8 }}

    spec:
      containers:
      - name: {{ include "myapp.name" . }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.containerPort }}
```

---

# -----------------------------------------

# 🎯 WHAT MUST MATCH AND WHY (VERY IMPORTANT)

# -----------------------------------------

### ✔ MUST MATCH:

```
selector.matchLabels
=
pod template labels (selector part only)
```

These labels **must be identical**.

### ✔ DO NOT NEED TO MATCH:

* Deployment metadata labels
* Pod extra metadata labels
* app.kubernetes.io/* labels

These EXTRA labels are safe and normal.
They do not affect pod selection.

---

# -----------------------------------------

# 📝 Example Rendered Output (After `helm template`)

# -----------------------------------------

```yaml
metadata:
  labels:
    app.kubernetes.io/name: myapp
    app.kubernetes.io/instance: prod
    app.kubernetes.io/version: "1.0.0"

spec:
  selector:
    matchLabels:
      app: myapp
      component: api

  template:
    metadata:
      labels:
        app: myapp
        component: api
        app.kubernetes.io/name: myapp
        app.kubernetes.io/instance: prod
        team: devops
        environment: prod
```

This confirms:

* Selector labels match ✔
* Metadata labels added ✔
* Extra labels included ✔
* No confusion ✔

---

# ## Best Practices (5+ YOE Interview Level)

* Use helpers for **all** repeated logic.
* Keep selector labels simple and stable.
* Use Kubernetes recommended labels in production.
* Use `fullname` helper for consistent naming.
* Do not hardcode labels in templates.
* Always pass `.` when using `include`.
* Use `nindent` for clean YAML.
* Never place version labels inside selectors.
---
---

# Helm Naming Templates — Detailed Notes (With Built-In Object Labels)

---

## **Concept / What**

Naming templates in Helm are reusable helper functions—typically defined inside `_helpers.tpl`—that generate **consistent, predictable, and organization-compliant names** for all Kubernetes objects created by a chart.

They rely on **Helm Built-In Objects** like `.Chart`, `.Release`, and `.Values` to build names dynamically.

Naming templates ensure:

* Uniform naming across all resources (Deployment, Service, Ingress, ConfigMap, Secret, HPA, etc.)
* Compliance with Kubernetes naming rules (DNS-compatible, lowercase, ≤63 chars)
* Centralized control of naming logic
* Easy updates when naming conventions change

---

## **Why / Purpose / Real Use Case**

Real-world organizations follow strict naming conventions that combine multiple attributes such as:

* environment (`prod`, `dev`)
* team (`payments`)
* application or chart name
* component name (`api`, `worker`)
* version info

Instead of repeating this naming logic across 8–10 templates, Helm uses **naming templates** to define it once and reuse everywhere.

This prevents inconsistencies and makes maintenance far easier.

---

## **How It Works / Steps / Syntax**

Naming templates rely on two mechanisms:

* `define` → to create the naming function inside `_helpers.tpl`
* `include` → to call the function from other templates

### **Built-In Objects Used in Naming Templates**

* **Built-in Object: `.Chart.Name`** → Base chart name
* **Built-in Object: `.Release.Name`** → Release name set during install/upgrade
* **Built-in Object: `.Values`** → User-defined config (environment, component, team)
* **Built-in Object: `.Chart.Version`** → Chart version (optional for naming)

---

## **Common Naming Helpers**

### **1. Base Name Helper (`name`)**

```gotmpl
{{- define "mychart.name" -}}
{{ .Chart.Name | lower }}
{{- end }}
```

Uses the built-in object `.Chart.Name`.

---

### **2. Full Resource Name Helper (`fullname`)**

This is the most important naming template.

```gotmpl
{{- define "mychart.fullname" -}}
{{ printf "%s-%s-%s" .Values.environment .Chart.Name .Values.component | lower | trunc 63 | trimSuffix "-" }}
{{- end }}
```

Uses **multiple built-in objects**:

* `.Values.environment`
* `.Chart.Name`
* `.Values.component`

This helper:

* Combines multiple values
* Converts to lowercase
* Truncates to prevent Kubernetes errors
* Removes trailing hyphens

---

## **How Templates Use Naming Helpers**

### Deployment

```yaml
metadata:
  name: {{ include "mychart.fullname" . }}
```

### Service

```yaml
metadata:
  name: {{ include "mychart.fullname" . }}
```

### Ingress

```yaml
backend:
  service:
    name: {{ include "mychart.fullname" . }}
```

This ensures all resources share the same consistent name.

---

## **Rendered YAML Example**

If `values.yaml` contains:

```yaml
environment: prod
component: api
```

And chart name = `orders`

Rendered name becomes:

```
prod-orders-api
```

Used across Deployment, Service, and Ingress.

---

## **Why Naming Templates Are Critical in Real Organizations**

* Company naming rules may change → update only the helper, not every template.
* All resources referencing each other must share the same name.
* Avoid Kubernetes errors from long or invalid names.
* Prevent mismatches between Service and Ingress backend names.
* Keep charts clean, readable, and DRY (Don’t Repeat Yourself).

---

## **Common Issues / Errors**

| Issue                                | Cause                                    |
| ------------------------------------ | ---------------------------------------- |
| Resource name too long               | Helper missing `trunc`                   |
| Wrong resource name generated        | Wrong function name in `include`         |
| Missing context (`.`)                | Forgot to pass dot to `include`          |
| Inconsistent naming across templates | Hardcoded names instead of using helpers |

---

## **Troubleshooting / Fixes**

* Render templates to inspect final names:

  ```bash
  helm template myapp .
  ```
* Use debug mode during install:

  ```bash
  helm install myapp . --dry-run --debug
  ```
* Confirm `.Values`, `.Chart`, and `.Release` are passed correctly to helpers.
* Validate DNS-compliant names using Kubernetes error messages.

---

## **Best Practices**

* Always use `fullname` helper for resource names.
* Avoid hardcoding names in any template.
* Combine multiple built-in objects to follow organizational naming standards.
* Use `trunc` and `trimSuffix` to maintain compatibility.
* Use `lower` to ensure DNS-safe names.
* Maintain all naming logic inside `_helpers.tpl` only.

---
---
---




