# Helm Security & Secrets

## Why `values.yaml` Should Not Contain Secrets

---

## Concept / What

In Helm, `values.yaml` is a **plain-text configuration file** used to supply values to templates during chart rendering. It is designed for **non-sensitive configuration data** such as replica counts, image names, ports, and feature flags. Helm does **not provide encryption or secret protection** for values stored in `values.yaml`.

---

## Why / Purpose / Real Use Case

In real-world **EKS + Helm + CI/CD** environments:

* `values.yaml` is usually:

  * Committed to **Git repositories**
  * Reviewed in **pull requests**
  * Rendered during **CI/CD pipelines**
  * Logged via `helm template`, `helm install --debug`, or pipeline logs

Storing secrets (passwords, API keys, tokens) in `values.yaml` means:

* Secrets are stored in **plain text**
* Anyone with Git access can read them
* Secrets can leak via:

  * Git history
  * CI/CD logs
  * Helm release metadata
  * Debug commands

This violates **security best practices** and compliance standards such as **SOC2, ISO, PCI**.

**Golden rule:**

> Helm is for configuration management, not secret storage.

---

## How it Works / Steps / Syntax

### ❌ Incorrect Pattern (Do NOT do this)

**values.yaml**

```yaml
db:
  username: admin
  password: MySuperSecret123
```

**deployment.yaml (Helm template)**

```yaml
env:
  - name: DB_PASSWORD
    value: {{ .Values.db.password }}
```

### Why this is dangerous

* Secret is committed to Git
* Secret is visible using:

  ```bash
  helm template
  helm get values <release-name>
  ```
* Secret may appear in CI/CD logs
* Helm does not encrypt values

---

## Correct Helm Security Model

| Component    | Responsibility                      |
| ------------ | ----------------------------------- |
| values.yaml  | Non-sensitive configuration only    |
| Secret Store | Store and manage secrets securely   |
| Helm         | Reference secrets, never store them |

Helm should **only reference existing secrets**, not contain them.

---

## Recommended Secure Pattern

### Reference an existing Kubernetes Secret

**deployment.yaml**

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

In this pattern:

* Helm does not know the secret value
* Secret lifecycle is managed externally
* Helm charts remain safe to store in Git

---

## Common Issues / Errors

* Secrets committed accidentally to Git
* Secrets exposed in `helm template` output
* Secrets visible via `helm get values`
* Assuming Kubernetes Secrets are encrypted (they are base64 only by default)

---

## Troubleshooting / Fixes

* Immediately rotate leaked secrets
* Remove secrets from Git history
* Audit Helm charts for `.Values.*password*` usage
* Use external secret management systems
* Restrict access to Helm debug commands

---

## Best Practices

* Never store secrets in `values.yaml`
* Treat Helm charts as **public artifacts**
* Use AWS-managed secret services
* Reference secrets using `secretKeyRef`
* Enforce secret scanning in CI/CD pipelines
* Separate configuration from secrets strictly

---
---

# Helm Security & Secrets

## Using AWS Secrets Manager

---

## Concept / What

**AWS Secrets Manager** is used in Kubernetes (EKS) to securely store sensitive information such as passwords, tokens, and keys outside of Git and Helm charts. In Kubernetes, AWS Secrets Manager is **not accessed directly by Helm**. Instead, it is integrated using a Kubernetes controller (commonly **External Secrets Operator**) which syncs secrets from AWS into **Kubernetes Secrets**.

Key clarification blended from discussion:

* Installing External Secrets Operator only installs the **controller and CRDs**
* No secrets are created automatically just by installation
* Kubernetes Secrets are created **only after we explicitly define ExternalSecret manifests**
* The creation is automatic, but the **intent and naming are user-defined**

---

## Why / Purpose / Real Use Case

In real-world **EKS + Helm + CI/CD** setups:

* Secrets must never be committed to Git or stored in `values.yaml`
* Secrets should be encrypted and centrally managed
* Applications should not carry AWS credentials
* Helm charts must remain reusable and environment-independent

Using AWS Secrets Manager allows:

* Centralized secret storage across environments (dev/qa/prod)
* IAM-based access control
* Easy rotation of secrets
* Compliance with enterprise security standards

Helm benefits because it only references Kubernetes Secrets and never handles secret values directly.

---

## How it Works / Steps / Syntax

### Step 1: Store the Secret in AWS Secrets Manager

A secret is created in AWS Secrets Manager, for example:

```text
prod/db/password
```

This secret is fully managed by AWS and encrypted using KMS.

---

### Step 2: Install External Secrets Operator (One-Time)

External Secrets Operator is installed once per cluster:

```bash
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

Important clarification:

* This step only installs the controller and CRDs
* No application secrets are created
* There are no pre-existing secret manifests to edit

---

### Step 3: Configure Authentication (IRSA)

External Secrets Operator authenticates to AWS Secrets Manager using **IAM Roles for Service Accounts (IRSA)**:

* An IAM role with `secretsmanager:GetSecretValue` permission is created
* The role is attached to ESO’s ServiceAccount
* ESO uses this role to fetch secrets securely

Helm and application pods do not access AWS directly.

---

### Step 4: Define How to Access AWS (SecretStore / ClusterSecretStore)

This manifest defines **how Kubernetes connects to AWS Secrets Manager**:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

This does not create secrets — it only defines connectivity.

---

### Step 5: Define ExternalSecret (This Is Mandatory)

This is where we explicitly declare:

* Which AWS secret to fetch
* What Kubernetes Secret name should be created

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-db-external
  namespace: app
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: app-db-secret
  data:
    - secretKey: password
      remoteRef:
        key: prod/db/password
```

Blended clarification from discussion:

* Kubernetes does NOT auto-generate the secret name
* The Kubernetes Secret name is explicitly defined using `target.name`
* External Secrets Operator automatically creates the secret **with that exact name**

---

### Step 6: Reference the Kubernetes Secret in Helm

Helm only references the Kubernetes Secret created by ESO:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-db-secret
        key: password
```

Helm never knows about AWS Secrets Manager or secret values.

---

## Common Issues / Errors

* Assuming secrets are created automatically after installing ESO
* Confusing CRDs with actual resources
* Expecting Kubernetes to auto-generate secret names
* Trying to edit ESO Helm chart files instead of creating new manifests
* Missing IRSA permissions

---

## Troubleshooting / Fixes

* Check ESO pods are running
* Verify ExternalSecret status using `kubectl describe externalsecret`
* Confirm Kubernetes Secret creation
* Inspect ESO controller logs
* Ensure ExternalSecret is applied before Helm deployment

---

## Best Practices

* Always use AWS Secrets Manager for sensitive data
* Never store secrets in Helm `values.yaml`
* Treat ExternalSecret manifests like Deployment manifests
* Use deterministic Kubernetes Secret names
* Apply ExternalSecret manifests before Helm releases

---
---

# Helm Security & Secrets

## Using SSM Parameter Store

---

## Concept / What

**AWS SSM Parameter Store** is an AWS service used to store configuration values and sensitive data (such as passwords, tokens, and keys) as **parameters**. In Kubernetes (EKS), SSM Parameter Store is **not accessed directly by Helm or applications**. Instead, it is integrated using a Kubernetes controller (commonly **External Secrets Operator**) which fetches parameters from SSM and creates **Kubernetes Secrets**.

Important clarifications blended from discussion:

* SSM Parameter Store stores **parameters**, not “SSM secrets”
* Installing External Secrets Operator does not create any secrets by itself
* Kubernetes Secrets are created automatically **only after we define manifests**
* The integration pattern is the same as AWS Secrets Manager

---

## Why / Purpose / Real Use Case

In real-world **EKS + Helm + CI/CD** environments, SSM Parameter Store is used when:

* Cost matters (SSM is cheaper than AWS Secrets Manager)
* Automatic rotation is not required
* Secrets are simple application-level values
* Centralized and IAM-controlled storage is needed

Typical usage pattern:

* **SSM Parameter Store** → application secrets and configs
* **AWS Secrets Manager** → database credentials with rotation

Helm benefits because it never handles secret values and only references Kubernetes Secrets.

---

## How it Works / Steps / Syntax

### Step 1: Store the Parameter in SSM Parameter Store

A parameter is stored in AWS:

```text
Name: /prod/app/db/password
Type: SecureString
```

* Encrypted using KMS
* Managed fully by AWS

---

### Step 2: Install External Secrets Operator (One-Time)

External Secrets Operator is installed once per cluster:

```bash
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

Clarification:

* This installs only the controller and CRDs
* No parameters or secrets are created
* There are no files to edit after installation

---

### Step 3: Configure Authentication (IRSA)

External Secrets Operator authenticates to AWS using **IAM Roles for Service Accounts (IRSA)**:

* IAM role permissions:

  * `ssm:GetParameter`
  * `ssm:GetParameters`
* Role attached to ESO ServiceAccount

Helm and application pods do not access AWS directly.

---

### Step 4: Define SecretStore / ClusterSecretStore for SSM

This manifest defines how Kubernetes connects to **SSM Parameter Store**:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-parameter-store
spec:
  provider:
    aws:
      service: ParameterStore
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets
```

This resource only defines connectivity; it does not create secrets.

---

### Step 5: Define ExternalSecret (Mandatory)

This manifest declares:

* Which SSM parameter to fetch
* What Kubernetes Secret name to create

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-db-external
  namespace: app
spec:
  secretStoreRef:
    name: aws-parameter-store
    kind: ClusterSecretStore
  target:
    name: app-db-secret
  data:
    - secretKey: password
      remoteRef:
        key: /prod/app/db/password
```

Blended clarification:

* Kubernetes does not auto-generate secret names
* The Kubernetes Secret name is explicitly defined by `target.name`
* External Secrets Operator automatically creates the Kubernetes Secret

---

### Step 6: Reference the Kubernetes Secret in Helm

Helm only references the Kubernetes Secret created by ESO:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-db-secret
        key: password
```

Helm never interacts with SSM Parameter Store directly.

---

## Common Issues / Errors

* Calling parameters “SSM secrets” instead of parameters
* Assuming secrets are created automatically after installing ESO
* Confusing CRDs with actual resources
* Expecting Kubernetes to auto-create secret names
* Missing IAM permissions for SSM access

---

## Troubleshooting / Fixes

* Verify ESO pods are running
* Check ExternalSecret status:

  ```bash
  kubectl describe externalsecret <name>
  ```
* Confirm Kubernetes Secret creation:

  ```bash
  kubectl get secret <secret-name>
  ```
* Inspect ESO controller logs
* Validate IAM role permissions

---

## Best Practices

* Use SSM Parameter Store for cost-sensitive secrets
* Use Secrets Manager when rotation is required
* Never store secrets in Helm `values.yaml`
* Treat ExternalSecret manifests like Deployment manifests
* Apply ExternalSecret manifests before Helm releases

---
---

# Helm Security & Secrets

## Using External Secrets Operator

---

## Concept / What

**External Secrets Operator (ESO)** is a Kubernetes **controller** that integrates external secret management systems (such as **AWS Secrets Manager** and **AWS SSM Parameter Store**) with Kubernetes by synchronizing external secrets into **Kubernetes Secret** objects.

ESO follows the standard Kubernetes **CRD + controller** pattern:

* Installing ESO adds only **CRDs** and a controller
* CRDs define new resource types (`ExternalSecret`, `SecretStore`, `ClusterSecretStore`)
* No Kubernetes Secrets are created automatically at install time
* Kubernetes Secrets are created **only after we define ExternalSecret manifests**

A key clarification blended from our discussion:

* Secret creation is automatic
* **Secret naming is not automatic** and must be explicitly defined in the manifest

---

## Why / Purpose / Real Use Case

In real-world **EKS + Helm + CI/CD** environments:

* Secrets must not be stored in Git or Helm `values.yaml`
* Applications should not directly access AWS services
* Secret handling should be centralized and standardized
* Helm charts should remain reusable and environment-agnostic

ESO is used because it:

* Acts as a single integration layer for multiple secret backends
* Eliminates secret handling from application code
* Uses Kubernetes-native Secret objects
* Works uniformly for AWS Secrets Manager and SSM Parameter Store

This allows Helm to focus only on deployment logic while ESO handles secret synchronization.

---

## How it Works / Steps / Syntax

### Step 1: Install External Secrets Operator (One-Time)

ESO is installed once per cluster, typically via Helm:

```bash
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace
```

Important clarifications:

* This installs only the controller and CRDs
* No ExternalSecret resources are created
* There are no application-specific manifests to edit

Installing ESO provides capability, not usage.

---

### Step 2: Configure Authentication using SecretStore / ClusterSecretStore

`SecretStore` or `ClusterSecretStore` defines **how ESO authenticates to AWS**.

* Uses **IAM Roles for Service Accounts (IRSA)**
* Specifies backend type (Secrets Manager or Parameter Store)
* Typically created once per cluster

Example (conceptual):

* Secrets Manager → `service: SecretsManager`
* Parameter Store → `service: ParameterStore`

This resource does not create secrets.

---

### Step 3: Define ExternalSecret (Mandatory)

`ExternalSecret` defines:

* Which external secret to fetch
* Which Kubernetes Secret name to create

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: app-db-external
spec:
  target:
    name: app-db-secret
```

Blended clarification from discussion:

* Kubernetes does not auto-generate secret names
* The secret name must be explicitly defined
* ESO automatically creates the Kubernetes Secret with this name

---

### Step 4: Kubernetes Secret Creation (Automatic)

Once the `ExternalSecret` is applied:

* ESO fetches the value from AWS
* ESO creates or updates the Kubernetes Secret
* The secret stays in sync based on refresh configuration

No manual secret creation is required.

---

### Step 5: Reference the Secret in Helm

Helm only references the Kubernetes Secret created by ESO:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: app-db-secret
        key: password
```

Helm does not interact with AWS or ESO directly.

---

## Common Issues / Errors

* Assuming secrets are created automatically after installing ESO
* Confusing CRDs with actual resources
* Expecting Kubernetes to auto-generate secret names
* Trying to edit ESO Helm chart files
* Missing or incorrect IRSA permissions

---

## Troubleshooting / Fixes

* Verify ESO pods are running
* Check ExternalSecret status using:

  ```bash
  kubectl describe externalsecret <name>
  ```
* Confirm Kubernetes Secret creation
* Inspect ESO controller logs
* Validate IAM role permissions

---

## Best Practices

* Install ESO once per cluster
* Use `ClusterSecretStore` for shared AWS access
* Treat ExternalSecret manifests like Deployment manifests
* Define deterministic Kubernetes Secret names
* Apply ExternalSecret manifests before Helm releases
* Keep Helm charts free of secrets

---
---

