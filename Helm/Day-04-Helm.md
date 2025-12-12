# Helm Notes — Global Values (Optional Concept)

## Concept / What

**Global values** are values defined under the `global:` key inside `values.yaml`. They are special values intended to be shared across a **parent Helm chart** and its **subcharts**. In a single microservice chart, global values behave the same as normal values and are optional.

## Why / Purpose / Real Use Case

### When global values matter

* Used when deploying **multiple subcharts** under one parent chart.
* Ensures shared configuration (image tags, domain, app name) is inherited across all subcharts.
* Reduces duplication for umbrella/platform-level charts.

### When global values do **not** matter

* In single microservice charts (most product/microservice architectures), `.Values` is enough.
* Normal values already work across all templates inside one chart.

Companies typically use global values only when:

* Managing **platform bundles** (Istio, monitoring stack, ingress controllers).
* Deploying **many components together** with shared settings.

## How it Works / Steps / Syntax

### 1. Define global values in `values.yaml`:

```yaml
global:
  appName: jagga-app
  image:
    repository: myrepo
    tag: "1.0"
```

### 2. Reference them in templates:

```yaml
{{ .Values.global.appName }}
{{ .Values.global.image.repository }}
```

### 3. Subcharts automatically inherit globals

If a chart has subcharts inside the `charts/` directory, the subcharts can also access:

```yaml
.Values.global.*
```

### 4. In a single-chart microservice

You can still use:

```yaml
.Values.global.*
```

But it provides *no additional functionality* over normal `.Values`.

## Common Issues / Errors

### ❌ Defining `globals:` instead of `global:`

Helm will not recognize the key.

### ❌ Expecting global values to work across separate Helm charts

Global values only work **within one parent chart + subcharts**, not across different applications.

### ❌ Using global values unnecessarily in single charts

Adds complexity without any benefit.

## Troubleshooting / Fixes

* Run `helm template --debug` to inspect merged values.
* Confirm that global keys are correctly nested under `global:`.
* Ensure subcharts do not redefine values that should come from global.

## Best Practices

### 👍 Recommended

* Use global values **only** for umbrella charts with subcharts.
* Store shared constants (image tags, domain, labels) under `global:` in multi-chart deployments.

### 👎 Not Recommended

* Do not use global values for single microservice charts unless your team prefers organizational grouping.
* Do not expect global values to work across independent Helm charts.

---
---

# Helm Notes — Environment-Specific Values (dev / stage / prod)

## Concept / What

Environment-specific values allow you to use **the same Helm templates** across multiple environments (dev, stage, prod) while customizing only the values that differ. Helm merges:

```
values.yaml  +  environment-specific file
```

The environment file overrides matching keys from `values.yaml` and adds new keys as needed.

## Why / Purpose / Real Use Case

* Configure different replicas, resources, domains, and logging levels per environment.
* Avoid maintaining separate templates or manifest files.
* Enable flexible GitOps and CI/CD flows (each environment selects its own values file).
* Keep common/default values in one place (`values.yaml`), and override only what changes.

This allows consistent deployments across EKS environments while minimizing duplication.

## How it Works / Steps / Syntax

### 1. Create separate values files:

```
values.yaml          # Common defaults
values-dev.yaml      # Dev overrides
values-stage.yaml    # Stage overrides
values-prod.yaml     # Prod overrides
```

### 2. Define base values (example `values.yaml`):

```yaml
replicaCount: 1
image:
  repository: myapp
  tag: "latest"
service:
  port: 80
```

### 3. Define environment-specific overrides:

#### values-dev.yaml

```yaml
replicaCount: 1
image:
  tag: "dev-123"
logging: DEBUG
```

#### values-prod.yaml

```yaml
replicaCount: 5
image:
  tag: "prod-789"
logging: INFO
ingress:
  host: myapp.prod.company.com
```

### 4. Deploy using the appropriate file:

```bash
helm upgrade --install myapp ./chart -f values-dev.yaml
helm upgrade --install myapp ./chart -f values-prod.yaml
```

### 5. How Helm merges values

* Helm **first loads** `values.yaml`.
* Then it **applies overrides** from the environment file.
* Matching keys are replaced; new keys are added.

Examples:

* `values.yaml: 1,2,3` + `env.yaml: 1,2,3` → env values overwrite.
* `values.yaml: 1` + `env.yaml: 2,3` → merged into `1,2,3`.
* `values.yaml: 1,2` + `env.yaml: 2,3` → final: `1` (base), `2` (env override), `3` (new).

## Common Issues / Errors

* Forgetting to use `-f` leads to default-only deployments.
* Wrong file passed in CI/CD results in incorrect configuration.
* YAML indentation mistakes cause unexpected merges.
* Assuming values override works across unrelated charts (it doesn’t).

## Troubleshooting / Fixes

* Validate merged output:

```bash
helm template myapp ./chart -f values-prod.yaml
```

* Double-check environment values files for indentation and structure.
* Hardcode correct environment-specific values files in CI/CD or GitOps.

## Best Practices

* Keep shared configuration in `values.yaml`.
* Only put environment-specific changes in dev/stage/prod files.
* Use clear naming (`dev.yaml`, `prod.yaml`, `values-prod.yaml`).
* Avoid duplicating common values in multiple files.
* Keep templates environment-agnostic (all differences handled through values files).

---
---

# Overriding Values via CLI (`--set`, `--set-string`, `--set-file`)

## Concept / What

CLI overrides allow you to change Helm values **directly from the command line**, without modifying `values.yaml` or environment-specific values files. Helm provides three options:

* `--set` — override simple values
* `--set-string` — force the value to be treated as a string
* `--set-file` — load the value from a file

CLI overrides always take **highest priority** during Helm's value-merging process.

## Why / Purpose / Real Use Case

* **CI/CD pipelines** pass dynamic values such as image tags, commit IDs, or release versions.
* Allows teams to test or modify configurations **without editing files**.
* Useful for **sensitive values** (certificates, keys) using `--set-file`.
* Enables temporary overrides without committing changes to Git.

## How it Works / Steps / Syntax

### 1. Basic override with `--set`:

```bash
helm upgrade --install myapp ./chart --set replicaCount=3
```

Nested values:

```bash
helm upgrade --install myapp ./chart --set image.tag=2.0
```

Multiple overrides:

```bash
helm upgrade --install myapp ./chart --set replicaCount=3,image.tag=2.0
```

### 2. Force a string value using `--set-string`:

```bash
helm upgrade --install myapp ./chart --set-string image.tag=00123
```

Use this when Helm may convert strings into numbers or booleans.

### 3. Load values from a file using `--set-file`:

```bash
helm upgrade --install myapp ./chart --set-file tlsCert=./cert.pem
```

Helm inserts the **file contents** into the value.

## Values Precedence (Highest → Lowest)

1. CLI overrides (`--set`, `--set-string`, `--set-file`) — **highest priority**
2. Environment-specific values (`-f dev.yaml`, `-f prod.yaml`)
3. `values.yaml` defaults
4. Chart's built-in default values — **lowest priority**

Templates do **not** define values; they only consume them.

## Common Issues / Errors

* **Quoting issues**: special characters in CLI may break the shell.
* **Accidentally overwriting entire objects** (e.g., `--set image=nginx` replaces the full image object).
* **Incorrect assumptions about precedence** (CLI always wins).
* **Missing or wrong file paths** when using `--set-file`.

## Troubleshooting / Fixes

* View final merged output:

```bash
helm template myapp ./chart -f prod.yaml --set image.tag=prod-123
```

* Inspect deployed values:

```bash
helm get values <release> -a
```

* Explicitly quote values with special characters.

## Best Practices

* Use CLI overrides **only when needed**, mainly in automation.
* Keep stable/default values in `values.yaml`.
* Store environment-specific differences in separate files.
* Use `--set-file` for sensitive or large content.
* Avoid using CLI for complex or deeply nested objects — use a values file instead.

---
---

# Multiple Values Files

## Concept / What

Multiple values files allow Helm to merge more than one values file during deployment. Helm always loads `values.yaml` first by default, and any additional files provided with the `-f` flag override or extend those defaults.

## Why / Purpose / Real Use Case

* Separate configuration cleanly (common values, environment values, secret values, feature flags).
* Avoid large, cluttered `values.yaml` files.
* Allow teams to manage different parts of the configuration independently.
* Combine multiple layers of overrides depending on the environment and use case.

## How it Works / Steps / Syntax

### 1. Helm loads `values.yaml` automatically

No need to specify it explicitly.

### 2. Provide additional values files as needed:

```bash
helm upgrade --install myapp ./chart -f dev.yaml -f secrets.yaml
```

### 3. Merge order (left → right):

* Load `values.yaml`
* Apply overrides from `dev.yaml`
* Apply overrides from `secrets.yaml` (last file wins)

### Example

#### values.yaml

```yaml
replicaCount: 1
image:
  tag: "base"
```

#### dev.yaml

```yaml
replicaCount: 2
```

#### feature.yaml

```yaml
image:
  tag: "feature-99"
```

#### Command

```bash
helm upgrade --install myapp ./chart -f dev.yaml -f feature.yaml
```

#### Final merged output

```yaml
replicaCount: 2
image:
  tag: "feature-99"
```

## Common Issues / Errors

* Wrong ordering of files results in incorrect final values.
* Duplicate or conflicting keys across files.
* Assuming environment file overrides default values without using `-f`.

## Troubleshooting / Fixes

* Verify merged output:

```bash
helm template myapp ./chart -f dev.yaml -f feature.yaml
```

* Check applied values in the cluster:

```bash
helm get values myapp -a
```

* Keep file structure consistent and avoid unnecessary nesting.

## Best Practices

* Keep shared values in `values.yaml`.
* Keep environment-specific overrides in dedicated files (dev.yaml, prod.yaml).
* Use multiple files only when necessary for clarity or separation.
* Remember: **Helm automatically loads `values.yaml`; no need to specify it**.

---
---

# Values Precedence

## Concept / What

Values precedence defines **which values win** when the same key is defined in multiple places. Helm merges values from several sources, and when conflicts occur, Helm follows a strict priority order to decide the final value.

## Why / Purpose / Real Use Case

* Values may come from defaults, `values.yaml`, environment files, and CLI overrides.
* CI/CD pipelines rely on precedence to correctly apply environment overrides.
* Prevents unexpected behavior when multiple configurations define the same key.
* Ensures deterministic deployments across different environments.

## How it Works / Steps / Syntax

Helm merges values in the following order (highest → lowest):

### **1. CLI overrides (highest priority)**

Examples:

```bash
--set image.tag=prod-123
--set-string version=001
--set-file tlsCert=./cert.pem
```

These always override all other values.

### **2. Environment-specific values files**

Examples:

```bash
-f dev.yaml
-f prod.yaml
```

These override the common defaults from `values.yaml`.

### **3. values.yaml (default chart values)**

Helm loads this automatically every time. Acts as the base for your chart.

### **4. Chart's built-in default values (lowest priority)**

These are the default values packaged **inside dependency charts**. For your own chart, this simply means the defaults defined in your own `values.yaml`.

## Example

### values.yaml

```yaml
replicaCount: 1
image:
  tag: "latest"
```

### dev.yaml

```yaml
replicaCount: 2
```

### CLI

```bash
--set replicaCount=3
```

### Final applied value

```
replicaCount = 3
```

Because CLI has the highest precedence.

## Special Case: Multiple Values Files

Order matters. Helm merges from left to right:

```bash
-f base.yaml -f dev.yaml -f secrets.yaml
```

Final merge order:

1. base.yaml
2. dev.yaml
3. secrets.yaml (overrides previous values)

## Subcharts and Global Values

Inside subcharts:

* Subchart values override parent chart values.
* `global:` values override both parent and subchart values.

Simplified precedence for subcharts:

```
CLI > env-file > subchart values > parent values > chart defaults
```

## Common Issues / Errors

* Passing files in the wrong order.
* Expecting values.yaml to override CLI (it cannot).
* Duplicate keys causing unexpected overrides.
* Assuming templates define values (they do not; they only reference values).

## Troubleshooting / Fixes

Check the final rendered output before deploying:

```bash
helm template myapp ./chart -f dev.yaml --set replicaCount=10
```

Inspect deployed values in the cluster:

```bash
helm get values myapp -a
```

## Best Practices

* Keep complex or nested values inside values files, not CLI.
* Always maintain a consistent order when passing multiple files.
* Use CLI overrides sparingly (mainly in automation pipelines).
* Store stable defaults in `values.yaml` and environment-specific changes in separate files.

---
---

# How Values Populate Templates

## Concept / What

Values populate templates through the special Helm object **`.Values`**. Anything defined in `values.yaml`, environment-specific values files, or CLI overrides becomes accessible in templates using `.Values.<key>`. The `.Values` object does **not** exist until Helm renders the chart.

## Why / Purpose / Real Use Case

* Keeps templates generic and reusable across environments.
* Allows dynamic configuration without modifying templates.
* Enables CI/CD pipelines to pass versions, tags, and overrides.
* Separates logic (templates) from configuration (values files).

## How it Works / Steps / Syntax

### 1. Helm loads all values sources in precedence order:

* Chart default values (lowest)
* `values.yaml`
* Environment files (`-f dev.yaml`, `-f prod.yaml`)
* CLI overrides (`--set`, `--set-string`, `--set-file`) — highest

### 2. Helm merges all these into one internal object:

```
.Values = merged final values
```

### 3. `.Values` becomes available to templates only during render

This happens when you run:

* `helm install`
* `helm upgrade`
* `helm template`

`.Values` does **not** exist just because files are created. It exists only during Helm rendering.

### 4. Templates read from `.Values`

Example values:

```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.25"
```

Template usage:

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  containers:
    - image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

Rendered output:

```yaml
spec:
  replicas: 3
  containers:
    - image: "nginx:1.25"
```

## Common Issues / Errors

* **Missing values** → template renders empty fields.
* **Wrong key path** (case-sensitive): `.Values.Image.Tag` ❌ vs `.Values.image.tag` ✔️
* **Unquoted values** causing YAML errors.
* **Indentation issues** when inserting values.

## Troubleshooting / Fixes

* Preview final output:

```bash
helm template myapp ./chart -f dev.yaml
```

* View applied values in cluster:

```bash
helm get values myapp -a
```

* Use `required` to enforce mandatory values:

```yaml
{{ required "replicaCount is required" .Values.replicaCount }}
```

## Best Practices

* Keep templates clean; store configuration in values files.
* Always quote strings in YAML-sensitive areas.
* Use nested structures (`.Values.image.tag`) for clarity.
* Validate values before rendering.
* Use helpers (`_helpers.tpl`) for repeated logic.

---
---
---
