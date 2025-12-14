# Helm Notes — Template Syntax

## **Concept / What**

Template syntax is the mechanism Helm uses (based on Go templating) to embed dynamic expressions inside Kubernetes YAML files. Anything inside `{{ ... }}` is evaluated at render time to generate final manifests.

---

## **Why / Purpose / Real Use Case**

* Avoids hardcoding values in templates.
* Allows dynamic generation of YAML.
* Makes charts reusable across dev, stage, and production.
* Enables CI/CD pipelines to inject values such as image tags, replicas, and configurations.
* Used heavily in EKS deployments where environment‑specific configurations change often.

---

## **How It Works / Steps / Syntax**

### **1. Basic placeholder syntax**

```yaml
env: {{ .Values.environment }}
```

### **2. Using expressions**

```yaml
name: {{ "nginx" | upper }}
```

### **3. Template comments**

```yaml
{{/* This is a comment */}}
```

### **4. Template inclusion**

```yaml
{{ include "mychart.fullname" . }}
```

### **Rendered YAML Example**

**templates/deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.name }}
spec:
  replicas: {{ .Values.replicas }}
```

**values.yaml:**

```yaml
name: app
replicas: 2
```

**Rendered:**

```yaml
metadata:
  name: app
spec:
  replicas: 2
```

---

## **Common Issues / Errors**

* Invalid YAML due to missing quotes.
* Indentation breaks due to incorrect spacing around `{{ }}`.
* Using variables without the correct dot context.
* Missing values in `values.yaml` causing template failures.

---

## **Troubleshooting / Fixes**

* Use `helm template` or `helm install --debug --dry-run` to inspect output.
* Quote string values: `"{{ .Values.value }}"`.
* Use functions like `nindent` for proper formatting.
* Add `default` function to avoid missing value errors.

---

## **Best Practices**

* Always quote rendered strings.
* Use helper templates for repeated patterns.
* Keep business logic minimal inside templates.
* Validate rendered YAML before applying to EKS.

---
---

# Helm Notes — Dot Context (`.`)

## **Concept / What**

The dot (`.`), also called the **root context**, represents the current data object Helm is working with. All template objects—such as `.Values`, `.Chart`, `.Release`—are accessed through this context.

---

## **Why / Purpose / Real Use Case**

* Required to access chart data inside templates.
* Necessary when passing context to helper templates.
* Prevents ambiguity in nested templates.
* Ensures dynamic values can be accessed during deployment.

**Real Case:** Without the leading `.` in:

```yaml
{{ include "mychart.fullname" . }}
```

Helm will not know what `.Values` or `.Release` refers to inside the helper.

---

## **How It Works / Steps / Syntax**

### **1. Accessing chart data**

```yaml
replicas: {{ .Values.replicaCount }}
```

### **2. Passing full context to helpers**

```yaml
metadata:
  name: {{ include "chart.fullname" . }}
```

### **3. Context inside loops (context changes)**

When using `range`, `.` becomes the current loop item.

```yaml
{{ range .Values.ports }}
- containerPort: {{ . }}
{{ end }}
```

### **4. Restoring the root context**

```yaml
{{ $root := . }}
```

### **Rendered YAML Example**

**values.yaml:**

```yaml
name: app
ports:
  - 80
  - 443
```

**template:**

```yaml
metadata:
  name: {{ .Values.name }}
spec:
  ports:
  {{ range .Values.ports }}
    - containerPort: {{ . }}
  {{ end }}
```

---

## **Common Issues / Errors**

* Losing context inside loops.
* Forgetting to pass `.` to `include`.
* Using `.Values` inside `range` without restoring context.

---

## **Troubleshooting / Fixes**

* Store the root context: `{{ $root := . }}`.
* Always pass `.` when using `include`.
* Debug context using `{{ . | toYaml }}` during dry runs.

---

## **Best Practices**

* Always start expressions with a leading dot.
* Pass context explicitly when calling helpers.
* Avoid deep nested context changes—keep templates readable.

---
---

# Helm Notes — `.Values`

## **Concept / What**

`.Values` is the Helm object that stores all configuration values provided to a chart. These values come from `values.yaml`, additional `-f` files, CLI overrides using `--set`, subcharts, or default values inside templates. Templates read `.Values` to dynamically generate Kubernetes manifests.

---

## **Why / Purpose / Real Use Case**

* Prevents hardcoding replicas, ports, image tags, resources, etc.
* Makes charts reusable across **dev / stage / prod** simply by changing values files.
* Allows CI/CD pipelines to inject dynamic values like image tags or environment identifiers.
* Enables EKS workloads to adapt settings such as ALB rules, node-group resources, autoscaling behavior, and environment-specific annotations.
* Provides clear separation of **template logic** and **environment configuration**.

---

## **How It Works / Steps / Syntax**

### **1. Access values**

```yaml
replicas: {{ .Values.replicaCount }}
```

### **2. Access nested values**

```yaml
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### **3. Provide default fallback values**

```yaml
imagePullPolicy: {{ default "IfNotPresent" .Values.image.pullPolicy }}
```

### **4. Use conditionals based on values**

```yaml
{{- if .Values.autoscaling.enabled }}
# autoscaling YAML
{{- end }}
```

### **5. Loop through list values**

```yaml
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}
```

### **6. Passing `.Values` context to helpers**

```yaml
metadata:
  name: {{ include "app.fullname" . }}
```

### **7. Value Precedence (highest → lowest)**

1. CLI: `--set`
2. Extra values file: `-f prod.yaml`
3. Default `values.yaml`
4. Template-level fallback via `default` function

---

## **How It Works With Templates (Example)**

### **values.yaml**

```yaml
replicaCount: 2

image:
  repository: nginx
  tag: "1.24"

service:
  port: 80

resources:
  limits:
    cpu: "200m"
    memory: "256Mi"
```

### **templates/deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "app.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "app.name" . }}
    spec:
      containers:
        - name: {{ include "app.name" . }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - containerPort: {{ .Values.service.port }}
          resources:
            limits:
              cpu: {{ .Values.resources.limits.cpu }}
              memory: {{ .Values.resources.limits.memory }}
```

---

## **Rendered YAML Context**

```yaml
spec:
  replicas: 2
  template:
    spec:
      containers:
        - image: "nginx:1.24"
          ports:
            - containerPort: 80
          resources:
            limits:
              cpu: 200m
              memory: 256Mi
```

---

## **Common Issues / Errors**

* ❌ **Nil pointer errors** when values are missing.
* ❌ **Broken YAML indentation** inside loops.
* ❌ **Incorrect CLI override path**, e.g. `--set image=nginx` instead of `--set image.repository=nginx`.
* ❌ **Type mismatch**, like strings used where integers are expected.

---

## **Troubleshooting / Fixes**

### ✔ Debug rendered YAML

```bash
helm install --debug --dry-run .
```

### ✔ Print full `.Values` during debugging

```yaml
{{- toYaml .Values | indent 2 }}
```

### ✔ Avoid nil errors

```yaml
{{ default "none" .Values.env }}
```

### ✔ Fix YAML indentation

```yaml
resources:
{{- toYaml .Values.resources | nindent 2 }}
```

---

## **Best Practices**

* Keep `values.yaml` clean, structured, and well-documented.
* Use separate files for dev, stage, and prod environments.
* Always quote string values.
* Use `default` for optional values.
* Avoid deeply nested values to improve readability.
* Move repeated structures into helpers instead of duplicating them.
* Never hardcode environment-specific settings inside templates.

---
---

# Helm Notes — `.Release`

## **Concept / What**

`.Release` is a Helm built-in object that provides metadata about the current Helm release during installation, upgrade, or rollback. It contains the release name, namespace, revision number, and boolean indicators for whether the chart is being installed or upgraded.

Key fields include:

* `.Release.Name` — name of the release provided during `helm install`
* `.Release.Namespace` — namespace provided during installation
* `.Release.Revision` — revision number of the release
* `.Release.IsInstall` — true during a fresh install
* `.Release.IsUpgrade` — true during an upgrade operation

---

## **Why / Purpose / Real Use Case**

* Ensures Kubernetes resources dynamically use the correct **release name** and **namespace**.
* Helps maintain unique resource names in shared clusters (e.g., EKS multi-environment setup).
* Determines install vs upgrade logic to run initialization jobs only once.
* Helpful for NOTES.txt output to show user information about the deployed release.
* Useful for CI/CD pipelines to track revision numbers or conditional behaviors.

**Real EKS Examples:**

* Use `.Release.Name` to differentiate deployments for dev/stage/prod.
* Use `.Release.Namespace` to deploy into correct namespaces managed by EKS.
* Use `.Release.IsUpgrade` to skip expensive initialization tasks during upgrades.

---

## **How It Works / Steps / Syntax**

### **1. Accessing Release Name**

```yaml
metadata:
  name: {{ .Release.Name }}-config
```

### **2. Using Release Namespace**

```yaml
namespace: {{ .Release.Namespace }}
```

### **3. Accessing Revision Number**

```yaml
data:
  revision: "{{ .Release.Revision }}"
```

### **4. Using Install/Upgrade Conditionals**

```yaml
{{- if .Release.IsInstall }}
# initialization logic
{{- end }}

{{- if .Release.IsUpgrade }}
# upgrade-specific logic
{{- end }}
```

### **5. Printing Release Metadata in NOTES.txt**

```yaml
Your application has been deployed!
Release Name: {{ .Release.Name }}
Namespace: {{ .Release.Namespace }}
Revision: {{ .Release.Revision }}
```

---

## **Template Example Using `.Release`**

### **templates/configmap.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-settings
  namespace: {{ .Release.Namespace }}
data:
  releaseRevision: "{{ .Release.Revision }}"
  isInstall: "{{ .Release.IsInstall }}"
  isUpgrade: "{{ .Release.IsUpgrade }}"
```

### **Rendered YAML Example (Install)**

```yaml
metadata:
  name: webapp-settings
  namespace: default
data:
  releaseRevision: "1"
  isInstall: "true"
  isUpgrade: "false"
```

### **Rendered YAML Example (Upgrade)**

```yaml
data:
  releaseRevision: "2"
  isInstall: "false"
  isUpgrade: "true"
```

---

## **Common Issues / Errors**

* **Namespace mismatch** when templates hardcode a namespace instead of using `.Release.Namespace`.
* **Invalid resource names** when `.Release.Name` includes uppercase characters or numbers that don't meet K8s naming rules.
* **Misusing `.Release.Revision`** for logic—revision changes frequently and shouldn’t drive runtime behavior.
* **Missing install/upgrade conditions**, causing initialization jobs to run repeatedly.

---

## **Troubleshooting / Fixes**

### ✔ Validate namespace during installation

```bash
helm install myapp . --namespace dev
```

Ensure templates use:

```yaml
namespace: {{ .Release.Namespace }}
```

### ✔ Print release metadata while debugging

```yaml
{{- toYaml .Release | indent 2 }}
```

### ✔ Normalize names using helper templates

```yaml
metadata:
  name: {{ .Release.Name | lower }}-svc
```

### ✔ Use `.Release.IsInstall` and `.Release.IsUpgrade` for conditional behavior

Prevents running initialization jobs on every deployment.

---

## **Best Practices**

* Never hardcode namespace—always use `.Release.Namespace`.
* Use `.Release.Name` to generate unique resource names across environments.
* Avoid using `.Release.Revision` for logic; use it for debugging only.
* Prefer helper templates to sanitize and format names derived from release metadata.
* Use install/upgrade boolean flags for conditional tasks.
* Display release information in NOTES.txt for better visibility.

---
---

# Helm Notes — `.Chart`

## **Concept / What**

`.Chart` is a Helm built-in object that provides access to the metadata defined inside the chart's `Chart.yaml` file. It includes information such as the chart name, chart version, app version, description, maintainers, and more. Templates use this metadata for labeling, annotations, informational messages, and version tracking.

Key fields inside `.Chart`:

* `.Chart.Name` — chart name
* `.Chart.Version` — chart version (package version)
* `.Chart.AppVersion` — version of the application being deployed
* `.Chart.Description` — chart description
* `.Chart.Type` — application or library chart

---

## **Why / Purpose / Real Use Case**

* Helps include consistent chart metadata in labels and annotations.
* Useful for tracking chart version across environments and CI/CD pipelines.
* Frequently used inside NOTES.txt to show chart information after installation.
* Allows applications to automatically reference their app version (e.g., as an image tag) when chart metadata and app version match.
* Supports debugging and traceability in EKS deployments: teams can easily identify which chart version deployed a given workload.

**Real EKS Examples:**

* Add chart version to labels for debugging and tracking rollouts.
* Display chart name and version in NOTES.txt output for developers.
* Use `.Chart.AppVersion` as the container image tag when chart versioning aligns with app release version.

---

## **How It Works / Steps / Syntax**

### **1. Access chart name**

```yaml
metadata:
  labels:
    chart: {{ .Chart.Name }}
```

### **2. Access chart version**

```yaml
metadata:
  labels:
    chartVersion: {{ .Chart.Version }}
```

### **3. Use AppVersion**

```yaml
image: "nginx:{{ .Chart.AppVersion }}"
```

### **4. Display chart metadata in NOTES.txt**

```yaml
Chart: {{ .Chart.Name }}
Version: {{ .Chart.Version }}
App Version: {{ .Chart.AppVersion }}
```

### **5. Using chart metadata in labels**

```yaml
metadata:
  labels:
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
```

---

## **Template Example Using `.Chart`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "app.fullname" . }}
  labels:
    chart: "{{ .Chart.Name }}-{{ .Chart.Version }}"
spec:
  replicas: 2
  template:
    metadata:
      labels:
        appVersion: "{{ .Chart.AppVersion }}"
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "nginx:{{ .Chart.AppVersion }}"
```

---

## **Rendered YAML Context**

```yaml
metadata:
  labels:
    chart: "myapp-1.2.0"
template:
  metadata:
    labels:
      appVersion: "1.20"
spec:
  containers:
    - name: myapp
      image: "nginx:1.20"
```

---

## **Common Issues / Errors**

* **Confusing chart version with application version:** `.Chart.Version` ≠ `.Chart.AppVersion`.
* **Using chart version as image tag** when chart versioning and application versioning do not align.
* **Missing required fields in Chart.yaml** causing Helm lint or install failures.
* **Hardcoding version information** instead of using `.Chart` metadata.

---

## **Troubleshooting / Fixes**

### ✔ Validate chart metadata

```bash
helm lint
```

### ✔ Print chart metadata while debugging

```yaml
{{- toYaml .Chart | indent 2 }}
```

### ✔ Use helpers to sanitize names derived from chart metadata

```yaml
metadata:
  name: {{ .Chart.Name | lower }}-svc
```

### ✔ Ensure chart version is updated for real releases

Chart version must increment for packaging and registry publishing.

---

## **Best Practices**

* Use `.Chart.Name` and `.Chart.Version` in labels to help track deployments.
* Use `.Chart.AppVersion` only when your workflow aligns chart versioning with app release versioning.
* Keep `Chart.yaml` metadata accurate for CI/CD and debugging.
* Never hardcode chart metadata inside templates—always reference `.Chart`.
* Show chart details in NOTES.txt for transparency to users.

---
---

# Helm Notes — Conditionals (if/else)

## **Concept / What**

Conditionals in Helm templates allow you to dynamically include or exclude parts of Kubernetes YAML based on values, environment settings, or flags defined by the user. Helm uses Go template `if/else` logic to determine whether certain YAML blocks should be rendered.

Basic structure:

```yaml
{{ if CONDITION }}
  # YAML included when condition is true
{{ else }}
  # YAML included when condition is false
{{ end }}
```

---

## **Why / Purpose / Real Use Case**

* Enable or disable resources dynamically (e.g., Ingress, autoscaling, metrics).
* Make Helm charts reusable across dev/stage/prod with environment-specific behavior.
* Used in EKS deployments to conditionally add:

  * ALB annotations
  * HPA autoscaling configuration
  * nodeSelector / tolerations
  * TLS settings
* CI/CD pipelines use conditionals to toggle features using `--set` flags.
* Helps reduce template duplication by applying logic inside the same file.

---

## **How It Works / Steps / Syntax**

### **1. Basic conditional**

```yaml
{{- if .Values.ingress.enabled }}
# Ingress YAML
{{- end }}
```

### **2. if/else block**

```yaml
{{- if .Values.autoscaling.enabled }}
minReplicas: 2
{{- else }}
replicas: {{ .Values.replicaCount }}
{{- end }}
```

### **3. Comparison conditions**

```yaml
{{- if eq .Values.env "prod" }}
# production settings
{{- end }}
```

### **4. Boolean operators**

```yaml
{{- if and .Values.debug .Values.verbose }}
# render only if both values are true
{{- end }}
```

### **5. Nested conditionals**

```yaml
{{- if eq .Values.env "prod" }}
  {{- if .Values.autoscaling.enabled }}
  # autoscaling block
  {{- end }}
{{- end }}
```

---

## **Template Example Using Conditionals**

### **values.yaml**

```yaml
ingress:
  enabled: true
  host: example.com
```

### **templates/ingress.yaml**

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "app.fullname" . }}
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "app.fullname" . }}
                port:
                  number: 80
{{- end }}
```

---

## **Rendered YAML Context**

### **When enabled = true**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
    - host: example.com
```

### **When enabled = false**

*No YAML is rendered. The entire Ingress file is skipped.*

---

## **Common Issues / Errors**

* **Incorrect indentation** breaks YAML when using if/else blocks.
* **Using strings instead of booleans**, e.g., `enabled: "true"` instead of `enabled: true`.
* **Nil pointer errors** when checking values that do not exist.
* **Invalid placement inside lists**, causing bad indentation and rendering errors.
* **Overly nested logic**, making templates hard to read.

---

## **Troubleshooting / Fixes**

### ✔ Validate rendered output

```bash
helm template .
helm install --debug --dry-run .
```

### ✔ Use `default` to avoid nil errors

```yaml
{{ if default false .Values.autoscaling.enabled }}
```

### ✔ Fix YAML indentation with `nindent`

```yaml
{{- if .Values.extra }}
{{ toYaml .Values.extra | nindent 2 }}
{{- end }}
```

### ✔ Print values while debugging

```yaml
{{- toYaml .Values | indent 2 }}
```

---

## **Best Practices**

* Keep conditions simple and avoid deep nesting.
* Use boolean flags in `values.yaml` for clarity.
* Clearly document all conditional fields.
* Maintain proper YAML indentation with `nindent`.
* Split complex conditional logic into helper templates.
* Always test rendering with dry-run before deploying to EKS.

---
---

# Helm Notes — Loops (`range`)

## **Concept / What**

Loops in Helm templates use the Go templating `range` keyword to iterate over lists or maps defined in `values.yaml`. During a loop, the dot context (`.`) changes to the current item, allowing templates to dynamically generate repeated YAML sections such as ports, environment variables, annotations, or ConfigMap entries.

Helm supports:

* Looping through **lists**
* Looping through **lists of maps**
* Looping through **maps (key/value pairs)**
* Using **index variables**
* Saving **root context** to access `.Values`, `.Release`, `.Chart` inside loops

---

## **Why / Purpose / Real Use Case**

* Avoids writing duplicate YAML blocks.
* Supports dynamic configuration driven entirely by values files.
* Allows EKS workloads to define multiple:

  * container ports
  * environment variables
  * ALB ingress paths
  * annotations
  * volumes or volumeMounts
* Keeps charts reusable and environment‑agnostic.
* Allows CI/CD pipelines to modify lists without touching templates.

**Real EKS Examples:**

* Multiple service ports based on microservice needs.
* Dynamic ALB ingress rules using a list of paths.
* ConfigMaps built from key/value structures.

---

## **How It Works / Steps / Syntax**

### **1. Looping through a list**

```yaml
{{- range .Values.ports }}
- containerPort: {{ . }}
{{- end }}
```

### **2. Looping through a list of maps**

**values.yaml**

```yaml
env:
  - name: APP_ENV
    value: production
  - name: LOG_LEVEL
    value: debug
```

**template**

```yaml
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value }}
{{- end }}
```

### **3. Looping with index**

```yaml
{{- range $index, $value := .Values.ports }}
- name: port{{ $index }}
  value: {{ $value }}
{{- end }}
```

### **4. Looping through maps (key/value pairs)**

**values.yaml**

```yaml
config:
  mode: prod
  timeout: "30s"
  region: us-east-1
```

**template**

```yaml
{{- range $key, $value := .Values.config }}
{{ $key }}: {{ $value }}
{{- end }}
```

### **5. Saving root context (`$root := .`)**

Useful because inside loops `.` becomes the current item.

```yaml
{{- $root := . }}
{{- range .Values.ports }}
- containerPort: {{ . }}
  release: {{ $root.Release.Name }}
  imageTag: {{ $root.Values.image.tag }}
{{- end }}
```

---

## **Template Example: Multiple Ports in Deployment**

**values.yaml**

```yaml
ports:
  - 80
  - 443
```

**deployment.yaml**

```yaml
containers:
  - name: app
    image: nginx
    ports:
      {{- range .Values.ports }}
      - containerPort: {{ . }}
      {{- end }}
```

**Rendered YAML**

```yaml
ports:
  - containerPort: 80
  - containerPort: 443
```

---

## **Common Issues / Errors**

* **Losing context**: inside `range`, `.` refers only to the loop item, so `.Release` or `.Values` will fail.
* **Indentation problems**: misaligned YAML when loops are not correctly indented.
* **Nil pointer errors**: looping over a missing list or map.
* **Wrong loop pattern**: using list looping logic on maps or vice‑versa.
* **Assuming order**: maps are unordered in Go; output order is not guaranteed.

---

## **Troubleshooting / Fixes**

### ✔ Save the original root context

```yaml
{{- $root := . }}
```

### ✔ Validate YAML using dry‑run

```bash
helm install --debug --dry-run .
```

### ✔ Use defaults to prevent nil list errors

```yaml
{{- range default (list) .Values.ports }}
```

### ✔ Use `nindent` for reliable indentation

```yaml
{{ toYaml .Values.env | nindent 6 }}
```

### ✔ Debug values during rendering

```yaml
{{- toYaml .Values | indent 2 }}
```

---

## **Best Practices**

* Use lists when order matters; use maps only for unordered key/value pairs.
* Use `$root` whenever accessing `.Values`, `.Release`, `.Chart` inside loops.
* Keep loops simple and avoid deep nested looping.
* Document list and map structures clearly in `values.yaml`.
* Use helper templates to avoid repeating loop logic.
* Always verify rendered YAML formatting.

---
---

# Helm Notes — With Blocks (`with`)

## **Concept / What**

A `with` block in Helm templates is used to **temporarily change the dot context (`.`)** to a specific nested value. Inside the `with` block, the dot no longer refers to the full Helm context; instead, it refers only to the value passed to `with`.

`with` does **not** define new data and does **not** create values. It only redirects the dot (`.`) to an existing object for cleaner and more readable templates.

---

## **Why / Purpose / Real Use Case**

* Avoids repeating long value paths like `.Values.image.repository` multiple times.
* Makes templates cleaner and easier to read.
* Groups related configuration logically (image, resources, ingress, annotations).
* Automatically skips the block if the value is empty or not defined (safe for optional configs).
* Commonly used in EKS Helm charts for image config, resources, ingress, annotations, and labels.

**Real-world examples:**

* Access multiple fields under `.Values.image` without repetition.
* Apply optional ingress or resource blocks only when values exist.
* Simplify complex nested values in large Helm charts.

---

## **How It Works / Steps / Syntax**

### **Basic Syntax**

```yaml
{{- with VALUE }}
  # dot (.) now refers to VALUE
{{- end }}
```

### **Example: Using `with` for image values**

**values.yaml**

```yaml
image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent
```

**template**

```yaml
{{- with .Values.image }}
image: "{{ .repository }}:{{ .tag }}"
imagePullPolicy: {{ .pullPolicy }}
{{- end }}
```

Here, inside the block:

* `.` = `.Values.image`
* `.repository`, `.tag`, `.pullPolicy` are directly accessible

---

## **Using Root Context Inside `with`**

Because `with` changes the dot context, access to `.Values`, `.Release`, or `.Chart` is lost inside the block. To access the original context, use `$` (root context).

```yaml
{{- with .Values.image }}
containerName: {{ $.Release.Name }}
image: "{{ .repository }}:{{ .tag }}"
{{- end }}
```

* `.` → current `with` object (`.Values.image`)
* `$` → full original Helm context

---

## **Nested `with` Blocks**

`with` blocks can be nested to access deeper structures.

```yaml
{{- with .Values.ingress }}
  {{- with .annotations }}
metadata:
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

Each nested `with` updates the dot context step-by-step.

---

## **Rendered YAML Example**

```yaml
image:
  name: nginx
  tag: 1.25
```

---

## **Common Issues / Errors**

* **Forgetting that dot changes** inside the `with` block.
* **Trying to access `.Values` or `.Release` directly** without using `$`.
* **Using fields that don’t exist** inside the selected object.
* **Indentation errors** when rendering YAML inside `with` blocks.
* **Silent skipping** of the block when the value is empty or undefined.

---

## **Troubleshooting / Fixes**

* Use `$` to access the original Helm context:

```yaml
{{ $.Values }}
{{ $.Release.Name }}
```

* Debug context by printing it:

```yaml
{{- toYaml . | indent 2 }}
```

* Validate rendered templates:

```bash
helm install --debug --dry-run .
```

* Use `default` to ensure values exist when needed.

---

## **Best Practices**

* Use `with` for grouped, nested values (image, resources, ingress).
* Keep `with` blocks short and readable.
* Always remember that `.` changes inside the block.
* Use `$` when accessing `.Values`, `.Release`, `.Chart` inside `with`.
* Avoid deep nesting of `with` blocks.
* Prefer `with` for optional configuration sections.

---
---

# Helm Notes — Context Switching inside `range` / `with`

## **Concept / What**

Context switching in Helm templates refers to the automatic change of the dot context (`.`) when using `range` or `with`. Inside these blocks, the dot no longer refers to the full Helm context (which contains `.Values`, `.Release`, `.Chart`, etc.). Instead, it points to the current item (`range`) or the selected object (`with`).

This is not a new Helm feature or syntax, but an important behavioral rule of the Go templating engine that Helm uses.

---

## **Why / Purpose / Real Use Case**

* Prevents accidental misuse of `.Values`, `.Release`, and `.Chart` inside loops or scoped blocks.
* Explains why templates fail with errors like "can't evaluate field Release".
* Essential for writing correct Helm charts with nested `range`, `with`, and `include` blocks.
* Helps avoid subtle bugs in production Helm charts, especially in EKS deployments.
* Required knowledge when working with helpers, `include`, `tpl`, and complex templates.

**Real-world scenarios:**

* Using `.Release.Name` while looping over ports or environment variables.
* Accessing global values inside nested `with` blocks.
* Passing correct context to helper templates.

---

## **How It Works / Steps / Syntax**

### **1. Default context (before switching)**

At the start of template rendering:

```text
. = full Helm context
```

This allows access to:

```gotemplate
.Values
.Release
.Chart
```

---

### **2. Context switching with `range`**

```gotemplate
{{- range .Values.ports }}
{{ . }}
{{- end }}
```

Inside the loop:

* `.` = current list item (e.g., `80`, `443`)
* `.Values`, `.Release`, `.Chart` are no longer accessible

---

### **3. Context switching with `with`**

```gotemplate
{{- with .Values.image }}
{{ .repository }}
{{- end }}
```

Inside the block:

* `.` = `.Values.image`
* Global objects are out of scope

---

### **4. Using root context (`$`) to access global objects**

To retain access to global objects inside `range` or `with`, save or use the root context:

```gotemplate
{{- $root := . }}
{{- range .Values.ports }}
port: {{ . }}
release: {{ $root.Release.Name }}
{{- end }}
```

Here:

* `.` → local context (loop item)
* `$root` or `$` → full Helm context

---

### **5. Nested context switching example**

```gotemplate
{{- with .Values.ingress }}
  {{- range .hosts }}
host: {{ . }}
release: {{ $.Release.Name }}
  {{- end }}
{{- end }}
```

Context flow:

1. Start → full context
2. `with` → `.Values.ingress`
3. `range` → current host
4. `$` → always full context

---

## **Common Issues / Errors**

* **Trying to access `.Values`, `.Release`, or `.Chart` inside a loop without root context**.
* **Misunderstanding what `.` refers to** inside nested blocks.
* **Unexpected nil or evaluation errors** due to context loss.
* **Incorrect helper outputs** because wrong context was passed.

---

## **Troubleshooting / Fixes**

* Always assume `.` has changed inside `range` or `with`.
* Use `$` or `$root` to access global objects.
* Debug the current context:

```gotemplate
{{- toYaml . | indent 2 }}
```

* Render templates locally before deploying:

```bash
helm template .
helm install --debug --dry-run .
```

---

## **Best Practices**

* Treat context switching as a rule, not an exception.
* Use `.` for local data and `$` for global data.
* Save root context explicitly in complex templates.
* Keep nested `range` / `with` blocks readable.
* Always verify context before accessing fields.
* Remember: context switching is behavioral, not syntactic.

---
---

# Helm Notes — Passing Context to `include`

## **Concept / What**

In Helm, `include` is used to render and reuse helper templates (usually defined in `_helpers.tpl`). The `include` function works like a function call: it renders the named template using **only the context that is explicitly passed as the second argument**.

Helpers do not automatically have access to `.Values`, `.Release`, or `.Chart`. Whatever object is passed to `include` becomes the dot context (`.`) inside the helper template.

---

## **Why / Purpose / Real Use Case**

* Enables reuse of common logic such as naming, labels, annotations, and image strings.
* Keeps Helm templates clean, DRY, and consistent across resources.
* Required for large production Helm charts where naming and labels must be centralized.
* Critical when using helpers inside `range` or `with` blocks, where context is already switched.

**Real-world scenarios:**

* Generating consistent resource names using a helper.
* Reusing label and selector logic across Deployments, Services, and Ingress.
* Avoiding template duplication in EKS Helm charts.

---

## **How It Works / Steps / Syntax**

### **1. Defining a helper template**

Helpers are usually defined in `_helpers.tpl`:

```gotemplate
{{- define "mychart.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end -}}
```

---

### **2. Calling a helper with full context**

```gotemplate
{{ include "mychart.fullname" . }}
```

Here:

* `.` = full Helm context
* Helper can access `.Values`, `.Release`, `.Chart`

---

### **3. Problem: Calling include inside `range` or `with`**

```gotemplate
{{- range .Values.ports }}
{{ include "mychart.fullname" . }}
{{- end }}
```

Inside `range`, `.` becomes the current item (e.g., `80`, `443`). The helper now receives an invalid context and cannot access `.Release` or `.Chart`.

---

### **4. Correct solution: Pass root context (`$`)**

```gotemplate
{{- range .Values.ports }}
{{ include "mychart.fullname" $ }}
{{- end }}
```

Here:

* `$` = root Helm context
* Helper regains access to all built-in objects

---

### **5. Passing partial context intentionally**

Sometimes helpers are designed to work with a specific object:

```gotemplate
{{- define "mychart.image" -}}
{{ .repository }}:{{ .tag }}
{{- end -}}
```

Called as:

```gotemplate
{{ include "mychart.image" .Values.image }}
```

Inside the helper:

* `.` = `.Values.image`

---

### **6. Passing multiple contexts using `dict` (advanced)**

```gotemplate
{{ include "mychart.container" (dict "root" $ "image" .Values.image) }}
```

Inside helper:

```gotemplate
{{ .image.repository }}
{{ .root.Release.Name }}
```

---

## **Common Issues / Errors**

* Passing `.` from inside a `range` or `with` unintentionally.
* Helpers failing with errors like `can't evaluate field Release`.
* Assuming helpers automatically know the parent context.
* Incorrect helper output due to wrong context being passed.

---

## **Troubleshooting / Fixes**

* Always verify what `.` refers to before calling `include`.
* Use `$` when helpers need access to global objects.
* Debug helper context:

```gotemplate
{{- toYaml . | indent 2 }}
```

* Render templates locally:

```bash
helm template .
helm install --debug --dry-run .
```

---

## **Best Practices**

* Always pass `.` or `$` intentionally to `include`.
* Prefer passing `$` when calling helpers inside `range` or `with`.
* Design helpers to clearly document expected context.
* Use `dict` when helpers need both local and global data.
* Keep helper templates small and focused.
* Treat context passing as mandatory, not optional.

---
---

# Helm Notes — Built-in Template Functions

---

## **Concept / What**

Built-in template functions in Helm are predefined functions (from Go templates + Sprig + Helm additions) used to **transform data into valid, safe, and production-ready YAML during template rendering**.

These functions do **not create or store data**. They only **operate on existing data objects** (strings, maps, lists) at render time.

Key idea learned in this concept:

> **Helm works with data objects internally, not YAML text. Built-in functions help convert and safely render that data into YAML.**

---

## **Why / Purpose / Real Use Case**

Built-in functions exist to solve real Helm problems:

* Convert **maps/lists into valid Kubernetes YAML** (`toYaml`)
* Maintain **correct YAML indentation** (`indent`, `nindent`)
* Enforce **mandatory configuration** (`required`)
* Provide **safe fallbacks** (`default`)
* Avoid **YAML type issues** (`quote`)
* Standardize strings (`upper`, `lower`)

**Real-world usage (EKS / CI-CD):**

* Rendering `resources`, `annotations`, `labels`, `env`, `tolerations`
* Preventing broken deployments due to missing values
* Failing fast in pipelines when mandatory inputs are not provided

---

## **How It Works / Steps / Syntax**

### **1. Data vs YAML (core understanding)**

* `values.yaml` is written in YAML, but once Helm loads it, it becomes **data objects (maps/lists/strings) in memory**.
* Kubernetes expects **YAML text** in manifests.
* Built-in functions help convert **data → YAML text** safely.

---

### **2. `toYaml` — Map/List to YAML text**

**What it does:**

> Converts a **map or list data object** into valid YAML text.

**When to use:**

* ONLY for **maps or lists**
* NOT for strings, numbers, or booleans

**Example:**

```yaml
# values.yaml
labels:
  app: myapp
  tier: backend
```

```gotemplate
labels:
  {{ toYaml .Values.labels | nindent 2 }}
```

**Key realization from discussion:**

> Even though `values.yaml` is YAML, Helm treats it as data internally. `toYaml` converts that data back into YAML text.

---

### **3. `indent` vs `nindent` — YAML structure safety**

#### `indent`

* Adds **spaces only**
* Does **NOT** add a new line
* Requires the value to already start on the next line

#### `nindent`

* Adds a **new line + spaces**
* Safe even if value is written on the same line as the key
* **Preferred in real Helm charts**

**Practical rule discovered:**

> If you don’t want to worry about where the newline comes from, use `nindent`.

---

### **4. `default` — Optional values**

**What it does:**

> Provides a fallback value when the original value is missing or empty.

```gotemplate
replicas: {{ .Values.replicaCount | default 1 }}
```

Use when:

* Value is optional
* You want Helm to continue rendering

---

### **5. `required` — Mandatory values**

**What it does:**

> Forces Helm to stop rendering if a value is missing or empty.

```gotemplate
image: {{ required "image.repository is required" .Values.image.repository }}
```

**Important clarifications from discussion:**

* Fails for **missing AND empty** values
* The message is **just a human-readable error message** (not syntax)
* The message **cannot be omitted**

**Rule learned:**

> `default` continues Helm, `required` stops Helm.

---

### **6. `quote` — YAML safety**

**What it does:**

> Wraps values in double quotes to avoid YAML type confusion.

```gotemplate
value: {{ .Values.env | quote }}
```

Prevents issues with:

* `true`, `false`, `on`, `off`
* Numbers being auto-converted

---

### **7. `upper` / `lower` — String formatting**

```gotemplate
{{ .Values.env | upper }}
{{ .Release.Name | lower }}
```

Used for:

* Labels
* DNS-safe names
* Standardized naming

---

## **Common Issues / Errors**

* Using `toYaml` for strings (no benefit)
* Forgetting `nindent` after `toYaml` → broken YAML
* Using `indent` on same line as key
* Assuming `required` message affects logic
* Forgetting that Helm works with data objects, not YAML text

---

## **Troubleshooting / Fixes**

* If YAML looks wrong → check `toYaml` + `nindent`
* If Helm fails early → check `required`
* Debug rendered output:

```bash
helm template .
helm install --debug --dry-run .
```

* Print current object during debugging:

```gotemplate
{{ toYaml . | indent 2 }}
```

---

## **Best Practices**

* Use `toYaml` ONLY for maps and lists
* Always pair `toYaml` with `nindent`
* Prefer `nindent` over `indent`
* Use `required` for production-critical values
* Use `default` for optional configs
* Don’t memorize all functions — understand patterns

---
---

# Pipeline Operators (`|`) — Helm Templating Engine

---

## Concept / What

In Helm templates, a **pipeline operator (`|`)** is used to **pass the output of one function or expression as the input to another function**, similar to Unix shell pipelines.
It allows you to **chain multiple template functions together** so that data flows step by step through transformations before being rendered into the final Kubernetes YAML.

In simple terms:
**Left side produces data → `|` passes it → right side modifies it**

---

## Why / Purpose / Real Use Case

Pipeline operators are essential in Helm because real-world templates often require **multiple transformations on the same value**.

Typical needs include:

* Applying default values when a key is missing
* Converting maps/lists into valid YAML
* Fixing indentation for Kubernetes manifests
* Safely formatting user-provided or CI/CD-provided values

In production Helm charts (especially with **EKS and CI/CD pipelines**):

* Values come from `values.yaml`, environment-specific files, or pipeline variables
* These values must be rendered correctly to avoid invalid Kubernetes YAML
* Pipelines keep templates **clean, readable, and maintainable** while preventing syntax issues

---

## How it Works / Steps / Syntax

### Basic Syntax

```yaml
{{ VALUE | function1 | function2 | function3 }}
```

### Execution Flow

1. `VALUE` is evaluated first
2. The result is passed to `function1`
3. Output of `function1` is passed to `function2`
4. Final output is rendered into the manifest

---

## Simple Example

```yaml
{{ .Values.env | upper }}
```

Explanation:

* `.Values.env` reads the value from `values.yaml`
* `upper` converts it to uppercase

If `values.yaml` contains:

```yaml
env: dev
```

Rendered output:

```yaml
DEV
```

---

## Real-World Deployment Example

```yaml
labels:
  env: {{ .Values.env | default "dev" | quote }}
```

Step-by-step:

1. `.Values.env` reads the value
2. `default "dev"` supplies a fallback if the value is missing
3. `quote` wraps the value in quotes to make YAML safe

Rendered output:

```yaml
env: "dev"
```

---

## `toYaml` + `indent` — Most Common Helm Pattern

### values.yaml

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
```

### deployment.yaml

```yaml
resources:
{{ .Values.resources | toYaml | indent 2 }}
```

### Pipeline Flow

1. `.Values.resources` → Go map object
2. `toYaml` → converts the map into YAML format
3. `indent 2` → aligns YAML correctly under `resources:`

### Rendered YAML

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Without pipeline operators, this formatting would be complex and error-prone.

---

## Pipeline vs Nested Functions

❌ Hard to read (nested functions):

```yaml
{{ indent 2 (toYaml .Values.resources) }}
```

✅ Clean and recommended (pipeline style):

```yaml
{{ .Values.resources | toYaml | indent 2 }}
```

This readability advantage is why pipelines are the **standard approach in production Helm charts**.

---

## Pipeline with `include`

```yaml
{{ include "mychart.labels" . | nindent 4 }}
```

Explanation:

* `include` renders a helper template
* Output is piped into `nindent 4`
* Ensures correct indentation under YAML keys

This pattern is heavily used with `helpers.tpl`.

---

## Common Issues / Errors

* Incorrect order of functions in the pipeline
* Using `indent` before `toYaml`
* Forgetting indentation after rendering YAML blocks
* Passing the wrong data type (map vs string) into functions

Example of incorrect usage:

```yaml
{{ .Values.resources | indent 2 | toYaml }}
```

---

## Troubleshooting / Fixes

* Render templates locally to inspect output:

```bash
helm template .
```

* Debug installation rendering:

```bash
helm install --dry-run --debug
```

* If YAML breaks:

  * Verify `indent` vs `nindent`
  * Check pipeline function order
  * Confirm expected data type

---

## Best Practices

* Always use pipelines with `toYaml`
* Prefer `nindent` when rendering under YAML keys
* Keep pipelines readable and minimal
* Avoid deeply nested function calls
* Treat pipelines as formatting and safety tools

---
---

# indent / nindent — Helm Templating Engine

---

## Concept / What

In Helm templating, **`indent`** and **`nindent`** are built-in template functions used to **format multi-line rendered content by adding spaces at the beginning of each line**, so that the final output becomes **valid Kubernetes YAML**.

* **`indent`**: Adds a specified number of spaces to the beginning of **each line** of the rendered output.
* **`nindent`**: First adds a **newline**, then adds the specified number of spaces to the beginning of **each line**.

These functions do **not convert data to YAML**. Their only responsibility is **indentation and formatting**. Conversion from data structures (map/list) to YAML text is done by **`toYaml`**.

---

## Why / Purpose / Real Use Case

Kubernetes YAML is **strictly indentation-sensitive**. Even if the values are correct, **wrong indentation makes the manifest invalid**, causing Helm installs or upgrades to fail.

In real-world Helm usage (especially with **EKS and CI/CD pipelines**):

* Values are defined in `values.yaml`
* These values are passed to templates as **data structures**, not YAML text
* Lists and maps must first be converted using `toYaml`
* The resulting YAML must then be placed **at the correct indentation level** inside manifests

Manually adding spaces is error-prone and hard to maintain. `indent` and `nindent` provide a **safe, consistent, and readable way** to format rendered YAML blocks.

---

## How it Works / Steps / Syntax

### Data Rendering Flow (Important Mental Model)

```
values.yaml (map/list data)
→ toYaml (convert data to YAML text)
→ indent / nindent (format YAML indentation)
→ final Kubernetes manifest
```

---

## indent Function

### Syntax

```yaml
{{ VALUE | indent N }}
```

### Behavior

* Adds **N spaces** to the beginning of each line
* **Does NOT add a newline**
* Assumes the YAML key is already on a separate line

---

### Example: indent

#### values.yaml

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
```

#### deployment.yaml

```yaml
resources:
{{ .Values.resources | toYaml | indent 2 }}
```

### Rendered YAML

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Here, `indent` works correctly because `resources:` is already placed on its own line.

---

## nindent Function

### Syntax

```yaml
{{ VALUE | nindent N }}
```

### Behavior

* Adds a **newline first**
* Then adds **N spaces** to each line
* Safer and more flexible than `indent`

---

### Example: nindent

#### deployment.yaml

```yaml
resources:{{ .Values.resources | toYaml | nindent 2 }}
```

### Rendered YAML

```yaml
resources:
  limits:
    cpu: "500m"
    memory: "512Mi"
```

Because `nindent` adds the newline automatically, it avoids YAML formatting issues.

---

## indent vs nindent (Key Difference)

| Function | Adds Newline | Adds Spaces | Typical Usage                         |
| -------- | ------------ | ----------- | ------------------------------------- |
| indent   | No           | Yes         | When key already starts on a new line |
| nindent  | Yes          | Yes         | **Most real-world Helm charts**       |

**Rule of thumb:** If you are unsure, use **`nindent`**.

---

## Real-World Pattern with Helper Templates

```yaml
metadata:
  labels:
{{ include "mychart.labels" . | nindent 4 }}
```

Explanation:

1. `include` renders the helper template output
2. Output is multi-line YAML
3. `nindent 4` adds newline and aligns it correctly under `labels:`

Rendered output:

```yaml
metadata:
  labels:
    app: myapp
    env: dev
```

---

## Common Issues / Errors

1. **Missing indentation**

```yaml
resources:
{{ .Values.resources | toYaml }}
```

→ Invalid YAML

2. **Using indent when newline is required**

```yaml
resources: {{ .Values.resources | toYaml | indent 2 }}
```

→ YAML breaks due to missing newline

3. **Wrong indentation level**

* Too few spaces → incorrect hierarchy
* Too many spaces → invalid nesting

---

## Troubleshooting / Fixes

* Render templates locally:

```bash
helm template .
```

* Debug Helm install:

```bash
helm install --dry-run --debug
```

* Always inspect the **rendered YAML**, not just the template

---

## Best Practices

* Always combine `toYaml` with `indent` or `nindent`
* Prefer `nindent` in most scenarios
* Never rely on manual spacing
* Think in terms of **final rendered YAML structure**
* Use consistent indentation across all templates

---
---

# tpl Function — Helm Templating Engine

---

## Concept / What

In Helm, the **`tpl`** function is used to **evaluate a string value as a Helm template**.

Normally, Helm renders template syntax (`{{ }}`) **only inside files present in the `templates/` directory**. Any template-like syntax written inside `values.yaml` is treated as **plain text** and is **not evaluated**.

The `tpl` function explicitly tells Helm to:

> "Take this string value and render it again as a template using the given context."

---

## Why / Purpose / Real Use Case

In most Helm charts, all logic is written directly inside the `templates/` directory, and **`tpl` is not required**. This is why many users work with Helm for a long time without ever using `tpl`.

`tpl` becomes necessary only in **advanced or reusable chart scenarios**, where:

* Template logic is intentionally moved into `values.yaml`
* Chart users or CI/CD pipelines must control dynamic behavior
* The chart author does not want users to modify template files

Typical real-world reasons:

* Platform or shared charts used by multiple teams
* CI/CD pipelines injecting dynamic expressions
* User-defined naming conventions or annotations
* Highly configurable Helm libraries

---

## How it Works / Steps / Syntax

### Syntax

```yaml
{{ tpl STRING CONTEXT }}
```

Where:

* `STRING` → the value containing Helm template syntax (usually from `values.yaml`)
* `CONTEXT` → the scope used to render the template (commonly `.`)

The context defines what objects are available inside the rendered string, such as:

* `.Values`
* `.Release`
* `.Chart`

---

## Normal Helm Rendering (Without tpl)

### values.yaml

```yaml
fullname: "{{ .Release.Name }}"
```

### template file

```yaml
metadata:
  name: {{ .Values.fullname }}
```

### Rendered Output

```yaml
name: {{ .Release.Name }}
```

Helm treats the value as plain text and does not evaluate the template syntax.

---

## Rendering with tpl

### template file

```yaml
metadata:
  name: {{ tpl .Values.fullname . }}
```

### Rendered Output

```yaml
name: my-release
```

Here, `tpl` forces Helm to re-evaluate the string as a template.

---

## Context Importance in tpl

The second argument to `tpl` is critical.

Correct usage:

```yaml
{{ tpl .Values.fullname . }}
```

If the wrong context is passed:

```yaml
{{ tpl .Values.fullname .Values }}
```

Then objects like `.Release` or `.Chart` will not be accessible, leading to empty or incorrect values.

---

## Real-World Example: Reusable Naming Logic

### values.yaml

```yaml
env: dev
fullname: "{{ .Release.Name }}-{{ .Values.env }}"
```

### deployment.yaml

```yaml
metadata:
  name: {{ tpl .Values.fullname . }}
```

### Rendered Output

```yaml
name: myapp-dev
```

This allows users to define naming rules without modifying chart templates.

---

## When tpl Is Needed

`tpl` is required only when **all of the following are true**:

* Template syntax exists inside `values.yaml`
* That value must be dynamically evaluated
* The logic is intentionally delegated to values or CI/CD

---

## When tpl Is NOT Needed

* Standard use of `.Values`, `.Release`, `.Chart` inside templates
* Static values in `values.yaml`
* Application-specific charts with fixed logic

In such cases, Helm automatically renders templates without `tpl`.

---

## Common Issues / Errors

1. Forgetting to use `tpl` when values contain template syntax
2. Passing incorrect context to `tpl`
3. Overusing `tpl`, making charts hard to understand
4. Debugging difficulty due to hidden logic in values

---

## Troubleshooting / Fixes

* Render templates locally:

```bash
helm template .
```

* Debug Helm install:

```bash
helm install --dry-run --debug
```

If `{{ }}` appears in the rendered YAML, `tpl` is missing or misused.

---

## Best Practices

* Use `tpl` only when necessary
* Prefer keeping logic inside templates when possible
* Use `tpl` mainly for reusable or platform-level charts
* Document any value that requires `tpl`
* Avoid hiding complex logic inside `values.yaml`

---
---

# Helper Templates (`helpers.tpl` — `define` / `include`)

---

## Concept / What

In Helm, **helper templates** are reusable template snippets that are defined once and reused across multiple YAML template files. They help avoid duplication and keep Helm charts clean and consistent.

Helper templates are usually stored in:

```
templates/_helpers.tpl
```

* Files starting with `_` are **not rendered as Kubernetes resources**
* Helpers are defined using **`define`**
* Helpers are rendered using **`include`**

You can think of helper templates as **functions in programming**:

* `define` → function definition
* `include` → function call

---

## Why / Purpose / Real Use Case

In real-world Helm charts:

* The same labels, names, and annotations are repeated in multiple resources
* Copy-pasting this logic leads to inconsistencies and maintenance issues

Helper templates solve this by:

* Eliminating duplication
* Enforcing consistency across resources
* Making charts easier to maintain and extend
* Enabling standardization in platform or organization-level charts

Almost **all production Helm charts** use helper templates for labels, names, and selectors.

---

## How it Works / Steps / Syntax

### Step 1: Define a Helper Template

Helpers are defined using `define` inside `_helpers.tpl`.

```yaml
{{- define "mychart.labels" -}}
app: myapp
env: {{ .Values.env }}
{{- end -}}
```

Key points:

* `define` only **declares** the helper
* It does **not render anything by itself**
* The helper name is a string (commonly prefixed with chart name)

---

### Step 2: Include the Helper in a Template

Helpers are rendered inside YAML files under the `templates/` directory using `include`.

```yaml
metadata:
  labels:
{{ include "mychart.labels" . | nindent 4 }}
```

Execution flow:

1. `include` renders the helper
2. Output is returned as a **string**
3. `nindent` formats the output correctly under `labels:`

---

## Importance of Context (`.`)

When calling a helper, a context must be passed.

```yaml
{{ include "mychart.labels" . }}
```

Passing `.` allows the helper to access:

* `.Values`
* `.Release`
* `.Chart`

If an incorrect context is passed, the helper may fail or render empty values.

---

## Common Helper Example: Full Resource Name

### _helpers.tpl

```yaml
{{- define "mychart.fullname" -}}
{{ .Release.Name }}-{{ .Chart.Name }}
{{- end -}}
```

### deployment.yaml

```yaml
metadata:
  name: {{ include "mychart.fullname" . }}
```

Rendered output:

```yaml
name: myrelease-mychart
```

---

## include vs template (Important Distinction)

Both `include` and `template` can be used to call helper templates, but they behave differently.

### include

* Returns helper output as a **string**
* Can be piped to `indent` or `nindent`
* Safer for YAML formatting
* **Preferred and recommended** in modern Helm charts

### template

* Prints output directly
* Cannot be piped
* Harder to control indentation
* Mainly exists for backward compatibility

**Best practice:** Learn and use `include` only.

---

## Real-World Usage Pattern

```yaml
labels:
{{ include "mychart.labels" . | nindent 4 }}

selector:
  matchLabels:
{{ include "mychart.selectorLabels" . | nindent 6 }}
```

This ensures:

* Consistent labels across Deployments and Services
* No mismatch between selectors and labels

---

## Common Issues / Errors

* Forgetting to use `nindent` after `include`
* Passing the wrong context
* Duplicating logic instead of creating helpers
* Defining helpers in non-underscore files

---

## Troubleshooting / Fixes

* Render templates locally to inspect output:

```bash
helm template .
```

* Debug Helm rendering:

```bash
helm install --dry-run --debug
```

* Always inspect the **final rendered YAML** for indentation issues

---

## Best Practices

* Always store helpers in `_helpers.tpl`
* Prefix helper names with chart name
* Use `include` instead of `template`
* Combine `include` with `nindent`
* Keep helpers small and focused
* Use helpers for labels, names, selectors, and annotations

---
---

# Template Patterns — Helm (Deployment, Service, Ingress, ConfigMap, Secret)

---

## Concept / What

Template patterns in Helm refer to the **standard, repeatable structure** used while writing Kubernetes manifest templates inside the `templates/` directory. These patterns focus on **how templates are organized and structured**, not on Helm syntax itself.

Each Kubernetes resource (Deployment, Service, Ingress, ConfigMap, Secret) follows its own Kubernetes schema, but Helm templates apply **common structural patterns** to make charts reusable, configurable, and environment-agnostic.

---

## Why / Purpose / Real Use Case

In real-world Helm usage:

* The same application is deployed to multiple environments (dev, qa, prod)
* Hardcoding values inside manifests is not scalable
* Configuration must change without modifying templates

Template patterns solve this by:

* Moving all changeable values to `values.yaml`
* Keeping templates generic and reusable
* Enforcing consistency across all manifests
* Making CI/CD-driven deployments possible

---

## How it Works / Steps / Structure

Inside the `templates/` directory, a Helm chart usually contains multiple manifest files such as:

* `deployment.yaml`
* `service.yaml`
* `ingress.yaml`
* `configmap.yaml`
* `secret.yaml`

Each file follows Kubernetes syntax, but Helm patterns are applied consistently.

---

## Deployment Template Pattern

### Common Structure

* `metadata.name` → derived using helper templates
* `metadata.labels` → reused via helpers
* `spec.replicas` → controlled via `values.yaml`
* `containers.image` → configurable image repo and tag
* `resources`, `env`, `ports` → passed from values

### Pattern Example

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
{{ include "mychart.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
{{ include "mychart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
{{ include "mychart.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

---

## Service Template Pattern

### Common Structure

* Service name from helpers
* Service type and ports from values
* Selector labels must match Deployment labels

### Pattern Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "mychart.fullname" . }}
  labels:
{{ include "mychart.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
  selector:
{{ include "mychart.selectorLabels" . | nindent 4 }}
```

---

## Ingress Template Pattern

### Common Structure

* Conditionally created using `if`
* Host, annotations, and paths driven by values
* Backend service name reused via helpers

### Pattern Example

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "mychart.fullname" . }}
  annotations:
{{ toYaml .Values.ingress.annotations | nindent 4 }}
spec:
  rules:
    - host: {{ .Values.ingress.host }}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: {{ include "mychart.fullname" . }}
                port:
                  number: {{ .Values.service.port }}
{{- end }}
```

---

## ConfigMap Template Pattern

### Common Structure

* Config data stored as key-value pairs
* Values sourced entirely from `values.yaml`
* Maps rendered using `toYaml` with indentation

### Pattern Example

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "mychart.fullname" . }}
data:
{{ toYaml .Values.config | nindent 2 }}
```

---

## Secret Template Pattern

### Common Structure

* Similar to ConfigMap
* Data provided via values (often base64 encoded)
* Rendered using `toYaml` and indentation

### Pattern Example

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "mychart.fullname" . }}
type: Opaque
data:
{{ toYaml .Values.secrets | nindent 2 }}
```

---

## Common Patterns Across All Templates

* Names and labels come from helpers
* Dynamic values come from `values.yaml`
* Lists and maps use `toYaml` + `nindent`
* Optional resources use conditional blocks
* No environment-specific values are hardcoded

---

## Common Issues / Errors

* Hardcoding values instead of using `values.yaml`
* Mismatched labels between Deployment and Service
* Missing indentation after `toYaml`
* Repeating logic instead of using helpers

---

## Best Practices

* Keep templates generic and reusable
* Push all environment-specific values to `values.yaml`
* Use helpers for names and labels
* Follow consistent structure across all manifests
* Think in terms of patterns, not individual files

---
---

# NOTES.txt — Post-install Message Templating in Helm

---

## Concept / What

`NOTES.txt` is a **special Helm template file** used to display **human‑readable messages** after a Helm release is installed or upgraded.

* Location:

  ```
  templates/NOTES.txt
  ```
* It is **not a Kubernetes manifest**
* It is **not applied to the cluster**
* It is rendered and printed **only to the terminal**

In simple terms:

> **`NOTES.txt` exists only to guide users after deployment, not to create resources.**

---

## Why / Purpose / Real Use Case

In real-world Helm usage, users often need immediate answers after installation, such as:

* How to access the application
* Which URL, hostname, or IP was created
* Which port is exposed
* What command to run next

Without `NOTES.txt`, users would have to:

* Inspect Services or Ingress manually
* Read manifests to understand access details

`NOTES.txt` improves:

* User experience
* Chart usability
* Self-service deployments for other teams

It is especially useful for **shared or platform-level charts**.

---

## When NOTES.txt Is Displayed

Helm renders and prints `NOTES.txt` automatically when running:

```bash
helm install
helm upgrade
```

The output appears **after** a successful deployment.

---

## How NOTES.txt Works

Although it is a text file, `NOTES.txt` is still a **Helm template**. This means:

* You can use `.Values`, `.Release`, `.Chart`
* You can use conditionals (`if / else`)
* You can use loops (`range`)

Helm renders it the same way as other templates, but:

* The output is plain text
* Nothing is sent to Kubernetes

---

## Typical Information Shown in NOTES.txt

Common patterns include:

* Application access URL
* Ingress hostname
* LoadBalancer IP instructions
* Port-forward commands
* Environment-specific notes

The goal is to show **clear next steps** for the user.

---

## Conceptual Example Pattern

```text
1. Your application has been deployed successfully.
2. Access it using the following URL:
3. If ingress is disabled, use port-forwarding.
```

The actual content varies based on chart values.

---

## Conditional Logic Usage (Pattern)

A common real-world pattern:

* If ingress is enabled → show ingress URL
* Else → show service access instructions

This ensures users are **not misled**.

---

## Important Behavior to Remember

* `NOTES.txt` is rendered **after all resources**
* Errors inside `NOTES.txt` can **fail Helm install/upgrade**
* It has **no effect** on Kubernetes resources

---

## Common Issues / Errors

* Syntax errors inside NOTES.txt
* Hardcoded values instead of `.Values`
* Showing incorrect instructions when configuration changes
* Writing overly long or unclear messages

---

## Best Practices

* Keep messages short and actionable
* Use values and conditions instead of hardcoding
* Guide the user on **what to do next**
* Think from the **chart user’s perspective**, not the author’s
* Treat NOTES.txt as documentation rendered at runtime

---
---

# Accessing Files in Helm using `.Files` (Get, GetBytes, Glob)

---

## Concept / What

In Helm, **files** refer to **static, non-template files** that are bundled inside a Helm chart and can be read at template-render time. These files are typically stored outside the `templates/` directory (commonly under a `files/` directory).

The built-in Helm object **`.Files`** allows templates to **read the contents of these files** and embed them into Kubernetes manifests such as ConfigMaps or Secrets.

Important clarifications:

* Files are **not Kubernetes resources**
* Files are **not rendered automatically**
* Files are **read-only inputs** for templates

---

## Why / Purpose / Real Use Case

In real-world Helm charts, `.Files` is used when you need to bundle **entire configuration files or scripts** with the chart instead of placing large content directly in `values.yaml`.

Reasons for using files instead of `values.yaml`:

* Large configuration files make `values.yaml` hard to read and manage
* Multiple config files become confusing when flattened into values
* Keeping configs as real files preserves structure and readability

Typical real-world use cases:

* Injecting full application configuration files into a ConfigMap
* Shipping Nginx / Apache / app config files
* Providing scripts that containers need at runtime
* Bundling certificates or static assets (less common today)

---

## Where Files Live in a Helm Chart

A common chart structure:

```
mychart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   └── service.yaml
├── files/
│   ├── app.conf
│   ├── nginx.conf
│   └── init.sh
```

Key rules:

* Files under `templates/` **cannot** be accessed using `.Files`
* Any non-template file in the chart can be accessed
* The `files/` directory is a convention, not a requirement

---

## How `.Files` Works (Mental Model)

```
Static file in chart
→ accessed using .Files
→ embedded into a template
→ applied as part of a Kubernetes resource
```

`.Files` itself does nothing unless explicitly used in a template.

---

## `.Files.Get` — Read File as Text

### What it Does

* Reads the content of a file
* Returns it as a **string**

### When to Use

* Text-based files such as `.conf`, `.yaml`, `.properties`, `.sh`
* Most common `.Files` function

### Pattern Example

```yaml
data:
  app.conf: |
{{ .Files.Get "files/app.conf" | nindent 4 }}
```

---

## `.Files.GetBytes` — Read File as Bytes

### What it Does

* Reads the content of a file
* Returns **raw bytes** instead of a string

### When to Use

* Binary files
* Certificates or encoded content

### Note

* Rarely used in modern Helm charts
* Mostly seen in legacy or specialized use cases

---

## `.Files.Glob` — Read Multiple Files

### What it Does

* Matches multiple files using a glob pattern
* Returns a collection of files

### When to Use

* Multiple configuration files
* Iterating over a directory of files

### Pattern Example

```yaml
data:
{{- range $path, $file := .Files.Glob "files/*.conf" }}
  {{ base $path }}: |
{{ $file | nindent 4 }}
{{- end }}
```

---

## Common Patterns with `.Files`

* ConfigMap populated from one or more files
* Secrets populated from file content (less common)
* Scripts mounted into Pods via ConfigMaps

---

## Common Issues / Errors

* Expecting files to be applied automatically
* Placing files inside `templates/` directory
* Forgetting indentation when embedding file content
* Treating files as environment-specific instead of static

---

## Troubleshooting / Fixes

* Render templates locally to inspect file embedding:

```bash
helm template .
```

* Debug Helm rendering:

```bash
helm install --dry-run --debug
```

* Always verify final rendered YAML

---

## Real-World Usage Frequency (Reality Check)

* Most application charts **do not use `.Files`**
* Common in platform, middleware, or shared charts
* Often encountered when reading open-source charts

`.Files` is a **situational concept**, not a core day-to-day Helm feature.

---

## Best Practices

* Use `.Files` only when full files are required
* Keep environment-specific values in `values.yaml`
* Prefer simple values over files when possible
* Document file usage clearly for chart users
* Treat `.Files` as a read-only input mechanism

---
---

# Template Debugging in Helm (`helm template`, `--dry-run`, `--debug`)

---

## Concept / What

Helm template debugging refers to the set of commands used to **inspect, validate, and troubleshoot Helm charts before actually applying resources to a Kubernetes cluster**.

Helm debugging focuses on **what YAML Helm generates** and **how Helm behaves during install or upgrade**, not on Kubernetes runtime issues.

---

## Why / Purpose / Real Use Case

In real-world Helm usage, most failures happen due to:

* Incorrect template logic
* Invalid YAML indentation
* Wrong values passed to templates
* Conditional logic behaving unexpectedly
* Errors during install or upgrade lifecycle

Debugging tools help catch these issues **before affecting the cluster**, making deployments safer and easier to troubleshoot.

---

## `helm template`

### What it Does

* Renders all Helm templates locally
* Substitutes values into templates
* Prints the final Kubernetes YAML output
* Does **not** install anything

### What Problems It Catches

* YAML indentation issues
* Template syntax errors
* Incorrect value rendering
* Issues with helpers, `toYaml`, `range`, and `if`

### When to Use

* While developing or modifying templates
* To inspect exactly what YAML Helm generates
* As the first step in debugging

---

## `--dry-run`

### What it Does

* Simulates `helm install` or `helm upgrade`
* Does not create or modify Kubernetes resources
* Executes Helm’s full install or upgrade lifecycle

### What Problems It Catches

* Errors during install or upgrade logic
* Issues related to Helm hooks
* Release lifecycle and validation problems

### When to Use

* Before performing a real install or upgrade
* In CI/CD pipelines
* When `helm template` output looks correct but install still fails

---

## `--debug`

### What it Does

* Prints additional internal Helm details
* Shows rendered manifests, values, and error context

### How It Is Used

* Usually combined with `--dry-run`
* Helps understand **why** Helm failed, not just that it failed

---

## Common Real-World Usage Pattern

```bash
helm install myapp ./chart --dry-run --debug
```

This command:

* Fully renders templates
* Simulates the install process
* Provides maximum visibility
* Applies no changes to the cluster

---

## How These Commands Differ (Mental Model)

* `helm template` → **Rendering validation only**
* `--dry-run` → **Simulated install/upgrade lifecycle**
* `--debug` → **Detailed diagnostic output**

They solve different debugging problems and are often used together.

---

## Typical Debugging Flow

1. Run `helm template` to validate rendered YAML
2. If YAML is correct, run `helm install --dry-run --debug`
3. Fix any install-time or lifecycle errors
4. Perform the real install or upgrade

---

## Common Issues / Errors

* Relying only on `helm install` without rendering first
* Skipping `helm template` during development
* Ignoring `--debug` output
* Debugging Kubernetes before debugging Helm

---

## Best Practices

* Always inspect rendered YAML before installing
* Use `helm template` during chart development
* Use `--dry-run --debug` before real installs
* Treat Helm debugging as a separate step from Kubernetes debugging

---
---
---

