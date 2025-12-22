# Helm – Packaging & Sharing Charts

## Sub-Concept: Packaging Helm Charts (`helm package`)

---

## Concept / What

Packaging a Helm chart means converting a Helm chart directory into a versioned, compressed `.tgz` archive using the `helm package` command. The packaged chart becomes a single immutable artifact (for example, `myapp-1.0.0.tgz`) that can be stored, shared, and installed by Helm.

A packaged chart is **not required** to deploy applications with Helm, but it is the standard distribution format when Helm repositories are used.

---

## Why / Purpose / Real Use Case

In **real-world CI/CD-only setups**, packaging Helm charts is **optional** and **not commonly used**.

### Why most organizations skip packaging

* Helm can install directly from a chart directory in Git
* Git already provides versioning (commits, tags, branches)
* Packaging adds an extra pipeline step with no functional deployment benefit
* No need to manage chart repositories (`index.yaml`, permissions, storage)
* Faster and simpler CI/CD pipelines

This is why **most organizations (≈90–95%) install Helm charts directly from Git** in CI/CD pipelines.

### Why some organizations still package charts

Packaging is used by a **small percentage of organizations (≈5–10%)**, typically when:

* Charts are treated as **products/artifacts**, not source code
* Strong governance, audit, or compliance is required
* Central platform teams manage charts for multiple teams
* Immutable promotion is required (same artifact → dev → qa → prod)

Common in:

* Banks and regulated enterprises
* Large platform engineering teams

---

## How it Works / Steps / Syntax

### Chart structure prerequisite

A Helm chart must contain at minimum:

```
myapp/
├── Chart.yaml        # Mandatory
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
```

`Chart.yaml` is mandatory for packaging.

---

### Chart.yaml example

```yaml
apiVersion: v2
name: myapp
description: Sample application chart
type: application
version: 1.0.0
appVersion: "1.0"
```

Key fields:

* `apiVersion: v2` → required for modern Helm
* `name` → chart name
* `version` → used in the packaged file name

---

### Packaging the chart

Run from the parent directory:

```bash
helm package myapp
```

Output:

```
myapp-1.0.0.tgz
```

Helm actions during packaging:

* Validates chart structure
* Reads metadata from `Chart.yaml`
* Compresses chart into `.tgz`
* Does **not** render templates
* Does **not** connect to Kubernetes

---

### CI/CD-style packaging

```bash
helm lint myapp
helm package myapp --destination artifacts/
```

Result:

```
artifacts/myapp-1.0.0.tgz
```

The `.tgz` file can then be uploaded to S3, CodeArtifact, or another Helm repository.

---

## Common Issues / Errors

### Missing Chart.yaml

```
Error: Chart.yaml file is missing
```

**Fix:** Create a valid `Chart.yaml`

---

### Invalid version format

```
Error: chart version is invalid
```

**Fix:** Use semantic versioning

```yaml
version: 1.0.1
```

---

### Using apiVersion v1

```
Error: chart apiVersion not supported
```

**Fix:**

```yaml
apiVersion: v2
```

---

### Reusing the same chart version

```
Error: chart already exists
```

**Fix:** Increment the chart version before packaging

---

## Troubleshooting / Fixes

* Always validate with `helm lint` before packaging
* Ensure `Chart.yaml` version is incremented
* Do not reuse packaged chart versions
* Confirm correct directory path when packaging

---

## Best Practices

* Use `helm package` **only when a chart repository is required**
* Prefer direct Git-based installs for simple CI/CD pipelines
* Treat packaged charts as **immutable artifacts**
* Never overwrite an existing chart version
* Use semantic versioning consistently

---

## Real-World Summary

* **Most CI/CD pipelines install Helm charts directly from Git**
* **Packaging is optional, not mandatory**
* **Only ~5–10% of organizations use packaged charts in repositories**
* Packaging is mainly used for governance, compliance, and platform use cases

---
---

# Helm – Packaging & Sharing Charts

## Sub-Concept: Hosting Helm Charts in S3

---

## Concept / What

Hosting Helm charts in **Amazon S3** means using an S3 bucket as a **Helm chart repository**. The bucket stores:

* Packaged Helm charts (`.tgz` files)
* An `index.yaml` file that acts as the repository catalog

Helm does not manage S3 itself; it only **reads charts over HTTP/HTTPS**. S3 simply acts as static storage for Helm artifacts.

---

## Why / Purpose / Real Use Case

In **CI/CD-only environments**, hosting Helm charts in S3 is used by a **small percentage of organizations (≈5–10%)**.

### Why most organizations avoid S3 Helm repos

* Helm can install charts directly from Git
* No functional deployment benefit compared to Git-based installs
* Requires extra steps: packaging, indexing, uploading
* Requires IAM, bucket policies, and access management
* Slows down pipelines and increases operational overhead

Because of this, **most CI/CD pipelines do not use S3-hosted Helm repositories**.

### Why some organizations choose S3

S3 is used when organizations require:

* Centralized chart distribution
* Immutable, versioned artifacts
* Clear separation of build and deploy stages
* IAM-based access control
* Compliance and auditability

This pattern is common in:

* Banks and regulated enterprises
* Platform engineering teams
* Large AWS-centric organizations

---

## How it Works / Steps / Syntax

### Step 1: Package the Helm chart (mandatory)

```bash
helm package myapp
```

Output:

```
myapp-1.0.0.tgz
```

Only packaged charts can be hosted in S3 Helm repositories.

---

### Step 2: Upload the chart to S3

```bash
aws s3 cp myapp-1.0.0.tgz s3://my-helm-repo/
```

At this stage, Helm still cannot use the chart because the repository index is missing.

---

### Step 3: Generate `index.yaml`

```bash
helm repo index . --url https://my-helm-repo.s3.amazonaws.com
```

Example `index.yaml`:

```yaml
apiVersion: v1
entries:
  myapp:
    - version: 1.0.0
      urls:
        - https://my-helm-repo.s3.amazonaws.com/myapp-1.0.0.tgz
```

`index.yaml` maps chart names and versions to downloadable URLs.

---

### Step 4: Upload `index.yaml`

```bash
aws s3 cp index.yaml s3://my-helm-repo/
```

Now the S3 bucket is a valid Helm chart repository.

---

### Step 5: Add the S3 repo in Helm

```bash
helm repo add myrepo https://my-helm-repo.s3.amazonaws.com
helm repo update
```

---

### Step 6: Install the chart from S3

```bash
helm install myapp myrepo/myapp
```

---

## CI/CD Pipeline Flow (Realistic)

```
Code commit
↓
CI pipeline
↓
helm lint
helm package
helm repo index
aws s3 upload
↓
CD pipeline
↓
helm repo add
helm install
```

This flow is heavier than Git-based installs, which is why it is less commonly adopted.

---

## Common Issues / Errors

### Chart not found

```
Error: chart not found
```

**Cause:** `index.yaml` not updated or uploaded

**Fix:** Regenerate and upload `index.yaml`

---

### Access denied (403)

```
403 Forbidden
```

**Cause:** Incorrect IAM or bucket policy

**Fix:**

* Attach correct IAM permissions
* Validate bucket policy
* Prefer IAM roles over access keys

---

### Overwriting chart versions

```
Error: chart already exists
```

**Cause:** Same chart version reused

**Fix:** Increment chart version in `Chart.yaml`

---

## Troubleshooting / Fixes

* Verify S3 bucket is publicly accessible or properly authenticated
* Ensure `index.yaml` URLs are correct
* Never overwrite existing chart versions
* Validate repository with `helm search repo`

---

## Best Practices

* Use S3 only when a chart repository is required
* Treat packaged charts as immutable artifacts
* Always bump chart versions
* Automate `index.yaml` generation in CI
* Use IAM roles instead of static credentials

---

## Real-World Summary

* Hosting Helm charts in S3 is **supported but not common**
* Most CI/CD pipelines install charts directly from Git
* S3 Helm repos are mainly used in regulated or platform-driven environments
* Choosing S3 is an architectural decision, not a Helm requirement

---
---

# Helm – Packaging & Sharing Charts

## Sub-Concept: Hosting Helm Charts in GitHub Pages

---

## Concept / What

Hosting Helm charts in **GitHub Pages** means using GitHub’s Pages feature as a **static Helm chart repository**. In this approach:

* Helm charts are **packaged** into `.tgz` files
* An `index.yaml` file is generated
* Both are stored in a special branch (commonly `gh-pages`)
* GitHub Pages serves them over **HTTP/HTTPS** like a static web server

Although the charts are stored inside a GitHub repository, they are **not treated as source code**. Instead, the repository acts as a **static hosting location**, similar to Amazon S3.

---

## Why / Purpose / Real Use Case

In **CI/CD-only environments**, GitHub Pages is an **optional and secondary choice** for hosting Helm charts.

### Why most organizations do not use GitHub Pages

* Helm charts can be installed directly from Git
* No functional deployment advantage over Git-based installs
* Limited access control compared to cloud-native solutions
* Not suitable for regulated or security-sensitive environments

Because of this, **enterprise CI/CD pipelines rarely rely on GitHub Pages** for Helm chart hosting.

---

### Why some teams choose GitHub Pages

GitHub Pages is used mainly when teams want:

* A **free and simple** Helm repository
* No dependency on cloud providers
* Easy automation with GitHub Actions
* Distribution of **open-source or internal utility charts**

Common usage scenarios:

* Open-source Helm chart projects
* Small teams or startups
* Learning, demo, or proof-of-concept environments

---

## How it Works / Steps / Syntax

### Step 1: Package the Helm chart (mandatory)

```bash
helm package myapp
```

Output:

```
myapp-1.0.0.tgz
```

Only packaged charts can be hosted in GitHub Pages.

---

### Step 2: Generate `index.yaml`

```bash
helm repo index . --url https://<org>.github.io/<repo-name>
```

Example `index.yaml`:

```yaml
apiVersion: v1
entries:
  myapp:
    - version: 1.0.0
      urls:
        - https://myorg.github.io/helm-charts/myapp-1.0.0.tgz
```

The `index.yaml` file maps chart names and versions to downloadable URLs.

---

### Step 3: Push to GitHub Pages branch

Typical repository layout:

```
helm-charts-repo/
└── gh-pages/
    ├── index.yaml
    └── myapp-1.0.0.tgz
```

GitHub Pages configuration:

* Branch: `gh-pages`
* Folder: `/`

Once enabled, GitHub serves the files as a static website.

---

### Step 4: Add the repo in Helm

```bash
helm repo add myrepo https://myorg.github.io/helm-charts
helm repo update
```

---

### Step 5: Install the chart

```bash
helm install myapp myrepo/myapp
```

---

## CI/CD Automation Pattern

```
Code push
↓
CI pipeline
↓
helm lint
helm package
helm repo index
git push to gh-pages
↓
CD pipeline
↓
helm repo add
helm install
```

GitHub Actions is commonly used to automate this workflow.

---

## Common Issues / Errors

### GitHub Pages not enabled

```
404 Not Found
```

**Fix:**

* Enable GitHub Pages in repository settings
* Verify branch and directory configuration

---

### Incorrect repository URL

```
Error: chart not found
```

**Fix:**

* Ensure the `--url` value exactly matches the GitHub Pages URL

---

### Overwriting chart versions

```
Error: chart already exists
```

**Fix:**

* Always increment the chart version in `Chart.yaml`
* Never replace existing `.tgz` files

---

## Troubleshooting / Fixes

* Validate GitHub Pages URL in a browser
* Confirm `index.yaml` URLs are correct
* Use `helm search repo <repo>` to verify chart visibility
* Automate version bumps in CI pipelines

---

## Best Practices

* Use GitHub Pages mainly for open-source or internal tooling
* Avoid GitHub Pages for regulated or security-heavy environments
* Keep `gh-pages` branch managed only by CI
* Treat packaged charts as immutable artifacts

---

## Real-World Summary

* GitHub Pages acts as a **static Helm chart repository**
* It is **not mandatory** and not widely used in enterprise CI/CD
* AWS-based organizations usually prefer **S3** instead
* GitHub Pages is best viewed as **additional knowledge**, not a core requirement

---
---

# Helm – Packaging & Sharing Charts

## Sub-Concept: Hosting Helm Charts in AWS CodeArtifact

---

## Concept / What

**AWS CodeArtifact** is a fully managed **AWS-native artifact repository service**. When used with Helm, CodeArtifact acts as a **private Helm chart repository** that stores **packaged Helm charts (`.tgz`)** and allows Helm to fetch and install them securely.

In this model:

* Helm charts are packaged using `helm package`
* Packaged charts are pushed to CodeArtifact
* Helm pulls charts from CodeArtifact using authenticated access

CodeArtifact provides a managed alternative to third-party artifact tools such as JFrog Artifactory or Nexus.

---

## Why / Purpose / Real Use Case

Hosting Helm charts in CodeArtifact is **not the default approach** and is used only in **specific, mature AWS environments**.

### Why most organizations do NOT use CodeArtifact for Helm

* Helm can install charts directly from Git
* S3-based Helm repositories are simpler
* CodeArtifact requires authentication tokens
* Additional IAM and operational complexity

Because of this, **most CI/CD pipelines do not rely on CodeArtifact** for Helm charts.

---

### Why some organizations choose CodeArtifact

Organizations use CodeArtifact when they need:

* A **private and secure Helm repository**
* Centralized artifact governance
* IAM-based fine-grained access control
* A single artifact system for:

  * Java (`.jar`)
  * Node (`npm`)
  * Python (`pip`)
  * Helm charts

This approach is common in:

* Large AWS enterprises
* Regulated environments
* Platform engineering teams

---

## How it Works / Steps / Syntax

### Step 1: Create CodeArtifact domain and repository

```bash
aws codeartifact create-domain --domain helm-domain

aws codeartifact create-repository \
  --domain helm-domain \
  --repository helm-repo
```

---

### Step 2: Authenticate Helm to CodeArtifact

CodeArtifact requires authentication using a **temporary authorization token**.

```bash
aws codeartifact get-authorization-token \
  --domain helm-domain \
  --query authorizationToken \
  --output text
```

Add the repository to Helm:

```bash
helm repo add myrepo \
  https://<domain>-<account>.d.codeartifact.<region>.amazonaws.com/helm/helm-repo \
  --username aws \
  --password <TOKEN>
```

⚠️ Tokens expire (default ~12 hours) and must be refreshed automatically in CI/CD.

---

### Step 3: Package the Helm chart

```bash
helm package myapp
```

Output:

```
myapp-1.0.0.tgz
```

---

### Step 4: Push the chart to CodeArtifact

```bash
helm push myapp-1.0.0.tgz myrepo
```

CodeArtifact natively supports Helm repositories.

---

### Step 5: Install the chart from CodeArtifact

```bash
helm install myapp myrepo/myapp
```

---

## CI/CD Pipeline Flow (Typical)

```
Code commit
↓
CI pipeline
↓
helm lint
helm package
helm push (CodeArtifact)
↓
CD pipeline
↓
helm repo add (auth)
helm install
```

This flow is more complex than Git- or S3-based installs, which limits adoption.

---

## Common Issues / Errors

### Authorization token expired

```
401 Unauthorized
```

**Cause:** Token expiry

**Fix:**

* Regenerate token
* Automate authentication in CI/CD pipelines

---

### Access denied

```
AccessDeniedException
```

**Cause:** Missing IAM permissions

**Fix:**

* Attach CodeArtifact permissions to the CI role
* Use IAM roles instead of static credentials

---

### Chart not found

```
Error: chart not found
```

**Cause:** Incorrect repository URL or missing chart

**Fix:** Verify domain, repository name, region, and URL

---

## Troubleshooting / Fixes

* Validate authentication token generation
* Verify IAM role permissions
* Confirm repository URL format
* Use `helm search repo myrepo` to verify chart visibility

---

## Best Practices

* Use CodeArtifact only when strong security and governance are required
* Automate token handling in CI/CD pipelines
* Treat packaged charts as immutable artifacts
* Avoid CodeArtifact for simple or small-scale CI/CD setups

---

## Real-World Summary

* AWS CodeArtifact can act as a **private Helm chart repository**
* It replaces tools like JFrog or Nexus in AWS-centric environments
* It adds authentication and operational overhead
* Used mainly by mature, regulated AWS organizations

---
---

# Helm – Packaging & Sharing Charts

## Sub-Concept: `helm repo index`

---

## Concept / What

`helm repo index` is a Helm command used to **generate or update the `index.yaml` file** for a Helm chart repository. The `index.yaml` file is the **catalog** that tells Helm:

* What charts exist in the repository
* What versions are available
* Where each chart package (`.tgz`) can be downloaded from

Without `index.yaml`, a Helm repository **does not exist** from Helm’s perspective.

---

## Why / Purpose / Real Use Case

`helm repo index` is required **only when using a Helm chart repository** (S3, GitHub Pages, CodeArtifact, etc.).

### Why most CI/CD pipelines do not need it

* Most organizations install Helm charts **directly from Git**
* Git-based installs do not use Helm repositories
* Therefore, `index.yaml` is not required in most pipelines

### When `helm repo index` becomes necessary

This command is used when:

* Charts are packaged (`.tgz`)
* Charts are hosted in a **static or managed Helm repository**
* A central chart catalog is required

Typical users:

* Regulated enterprises
* Platform engineering teams
* Organizations treating charts as artifacts

---

## How it Works / Steps / Syntax

### Basic command

```bash
helm repo index .
```

This scans the current directory for `.tgz` files and generates an `index.yaml` file.

---

### Command with repository URL (most common)

```bash
helm repo index . --url https://my-helm-repo.example.com
```

The `--url` flag tells Helm **where the charts will be served from**.

---

### Example directory structure

```
helm-repo/
├── myapp-1.0.0.tgz
├── myapp-1.1.0.tgz
└── index.yaml
```

---

### Example `index.yaml`

```yaml
apiVersion: v1
entries:
  myapp:
    - version: 1.1.0
      urls:
        - https://my-helm-repo.example.com/myapp-1.1.0.tgz
    - version: 1.0.0
      urls:
        - https://my-helm-repo.example.com/myapp-1.0.0.tgz
```

Helm uses this file to resolve chart names and versions.

---

## CI/CD Usage Pattern

```
helm lint myapp
helm package myapp
helm repo index . --url <repo-url>
upload .tgz and index.yaml
```

Whenever a new chart version is added, `index.yaml` **must be regenerated**.

---

## Common Issues / Errors

### Chart not found

```
Error: chart not found
```

**Cause:** `index.yaml` missing or outdated

**Fix:** Re-run `helm repo index` and upload the updated file

---

### Incorrect URLs in index.yaml

```
Error: failed to download chart
```

**Cause:** Wrong `--url` value

**Fix:** Ensure the URL matches the actual hosting location exactly

---

### Overwriting index.yaml incorrectly

**Cause:** Manual edits or partial uploads

**Fix:** Always regenerate `index.yaml` using Helm, not manually

---

## Troubleshooting / Fixes

* Verify `index.yaml` is accessible over HTTP/HTTPS
* Open the repository URL in a browser to confirm availability
* Use `helm search repo <repo>` to validate repository state
* Automate index generation in CI pipelines

---

## Best Practices

* Use `helm repo index` only when chart repositories are required
* Always use the `--url` flag in production
* Treat `index.yaml` as a generated file, not hand-written
* Never delete old chart versions from the repository
* Regenerate index after every new chart upload

---

## Real-World Summary

* `helm repo index` is the **backbone of Helm repositories**
* It is **not needed** for Git-based Helm installs
* Required for S3, GitHub Pages, and CodeArtifact-based repos
* Used mainly by organizations that package and distribute charts

---
---

