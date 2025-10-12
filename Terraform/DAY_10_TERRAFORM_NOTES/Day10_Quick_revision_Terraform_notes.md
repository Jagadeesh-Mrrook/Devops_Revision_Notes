# Terraform Security & Best Practices – Introduction (Quick Revision Version)

---

### **Concept / What:**

Guidelines to keep Terraform infrastructure secure, consistent, and compliant.

### **Why / Purpose:**

* Prevent exposure of secrets and credentials.
* Maintain compliance and consistent standards.
* Avoid misconfigurations and human error.

### **How / Steps:**

* Use Vault or environment variables for secrets.
* Mark sensitive data with `sensitive = true`.
* Ignore secrets and state files in `.gitignore` and `.terraformignore`.
* Run `terraform fmt` and `validate`.
* Use minimal provisioners.
* Apply Sentinel/OPA for compliance.
* Follow least privilege access.

### **Common Issues / Fixes:**

| Issue                 | Fix                                    |
| --------------------- | -------------------------------------- |
| Secrets leaked to Git | Rotate credentials & update .gitignore |
| Terraform plan fails  | Check env vars & backend config        |
| Formatting issues     | Run `terraform fmt`                    |
| Policy failure        | Adjust configuration or Sentinel rules |

### **Best Practices:**

* Store state remotely (S3 + DynamoDB lock).
* Validate and format automatically in CI/CD.
* Restrict access to sensitive files.
* Rotate credentials regularly.
* Always review plan before apply.

---
---

# Terraform: Storing Secrets Securely (Quick Revision Version)

---

### **Concept / What:**

Keep sensitive information (passwords, API keys, tokens) secure in Terraform by avoiding hardcoding and using environment variables or AWS Secrets Manager.

### **Why / Purpose:**

* Prevent secrets from being committed to Git.
* Avoid unauthorized access.
* Ensure compliance with security policies.
* Allow safe automation in CI/CD pipelines.

### **How / Steps:**

#### 1️⃣ Using Environment Variables

* Export secret:

```bash
export TF_VAR_db_password="SuperSecret123"
```

* Terraform variable:

```hcl
variable "db_password" {
  type = string
  sensitive = true
}
```

* Terraform reads variable during plan/apply automatically.

#### 2️⃣ Using AWS Secrets Manager

* Store secret:

```bash
aws secretsmanager create-secret --name prod/db_password --secret-string '{"username":"admin","password":"SuperSecret123"}'
```

* Access in Terraform:

```hcl
data "aws_secretsmanager_secret_version" "db_secret_version" {
  secret_id = "prod/db_password"
}
locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_secret_version.secret_string)
}
```

* Use `local.db_creds.password` and `local.db_creds.username` in resources.

* Optionally, export as env var for CI/CD:

```bash
export TF_VAR_db_password=$(aws secretsmanager get-secret-value --secret-id prod/db_password --query 'SecretString' --output text | jq -r '.password')
```

### **Common Issues / Fixes:**

| Issue                 | Fix                          |
| --------------------- | ---------------------------- |
| Terraform fails       | Check env var or secret name |
| AccessDeniedException | Update IAM permissions       |
| Secret in logs        | Use `sensitive = true`       |
| JSON parse error      | Ensure valid JSON secret     |

### **Best Practices / Tips:**

* Never hardcode secrets.
* Mark variables/outputs as `sensitive = true`.
* Encrypt Terraform state files.
* Rotate secrets regularly.
* Fetch secrets dynamically in CI/CD.
* Use ephemeral secrets where possible.

---
---

# Terraform Sensitive Variables & Outputs — Quick Revision Notes

## 🔍 Concept / What

Marking variables or outputs as **sensitive** hides secret values from being printed in the terminal, logs, or CI/CD output.

---

## 💡 Why / Purpose

* Prevent secrets (passwords, tokens, API keys) from leaking in logs.
* Maintain compliance and security.
* Avoid accidental exposure in shared Terraform runs or pipelines.

---

## ⚙️ How / Syntax

```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

* Output shows as `<sensitive>` during `terraform apply`.
* Still stored in **state file** (not encrypted unless backend encrypts it).

Access manually if needed:

```bash
terraform output db_password
```

---

## ⚠️ Common Issues

* Sensitive values visible in logs → check CI/CD echo settings.
* Believing `sensitive=true` encrypts data → it doesn’t.
* State file unsecured → secrets still in plain text.

---

## 🔧 Troubleshooting

* Use `TF_LOG=ERROR` to suppress debug logs.
* Avoid `terraform output -json` if not required.
* Restrict access to backend (S3, remote state).

---

## 🧠 Best Practices

* Always mark secret outputs as `sensitive = true`.
* Secure Terraform state backend (e.g., S3 + KMS).
* Don’t print sensitive outputs in Jenkins or automation logs.
* Combine with AWS Secrets Manager or environment variables.

---

## 🔄 Summary Table

| Aspect        | Description                        |
| ------------- | ---------------------------------- |
| Option        | `sensitive = true`                 |
| Purpose       | Hide secret data from logs/output  |
| State storage | Yes (plaintext)                    |
| View safely   | `terraform output <name>`          |
| Combine with  | Secrets Manager / IAM restrictions |

---
---

# Quick Revision — Version Control of Terraform Files

## 🔹 Concept / What

* Managing Terraform `.tf`, `.tfvars`, and module files in **Git** (GitHub, GitLab, Bitbucket).
* Tracks changes, enables collaboration, and allows rollback to previous versions.

## 🔹 Why / Purpose

* Collaboration among team members.
* Track and audit changes.
* Rollback broken configurations (similar to S3 versioning).
* Supports PR reviews and approvals.

## 🔹 How / Steps

1. Initialize Git repository:

```bash
git init
git add .
git commit -m "Initial Terraform setup"
```

2. Use `.gitignore` to exclude sensitive/temporary files:

```
*.tfstate
*.tfstate.backup
.terraform/
terraform.tfvars
*.pem
```

3. Follow branching strategy:

* `main` → production
* `dev` → testing
* `feature/*` → feature branches

4. Tag stable versions:

```bash
git tag v1.0.0
git push origin v1.0.0
```

5. Integrate CI/CD checks: `terraform fmt`, `validate`, `tflint`

## ⚠️ Common Issues

* Accidentally committing `.tfstate` (contains secrets).
* Merge conflicts when multiple users edit the same files.
* Hardcoded secrets in code.
* Missing `.gitignore` leading to temporary files in repo.

## 🧩 Troubleshooting / Fixes

* Remove sensitive files from repo:

```bash
git rm --cached terraform.tfstate
git commit -m "Remove sensitive state file"
```

* Rotate exposed credentials immediately.
* Enforce branch protection and pre-commit hooks.

## ✅ Best Practices

* Never commit `.tfstate` or `.tfvars`.
* Use proper `.gitignore` and `.terraformignore`.
* Use PRs for code review.
* Tag stable versions for tracking.
* Store secrets externally (Secrets Manager, Vault).
* Separate folders/branches for dev, QA, prod.
* Integrate `terraform fmt`, `validate`, and `tflint` in CI/CD.
* Document every change in commits/PRs.

---
---

# Quick Revision — Avoid Hardcoding Credentials

## 🧩 Concept / What

* Avoid embedding sensitive data (passwords, API keys) directly in Terraform files.

## 🎯 Why / Purpose

* Security: Prevents exposure and unauthorized access.
* Compliance: Meets SOC2/PCI DSS standards.
* Flexibility: Enables secret rotation without changing code.

## ⚙️ How / Steps

1. Environment variables (`export AWS_ACCESS_KEY_ID=...`).
2. Secrets Manager or Vault using Terraform data sources.
3. Terraform variables (`.tfvars`) kept outside Git.

## ⚠️ Common Mistakes

* Committing secrets to GitHub.
* Hardcoding credentials in provider blocks.

## 🧩 Troubleshooting / Fixes

* Remove sensitive files from repo and history.
* Rotate exposed credentials immediately.

## ✅ Best Practices

* Use env variables or secret managers.
* Mark outputs as `sensitive = true`.
* Regularly rotate credentials.
* Do not commit `.tfvars` containing secrets.
* Secure Terraform state backend (S3 + KMS).

---
---

# Quick Revision — Minimal Use of Provisioners

## 🧩 Concept / What

* Provisioners execute scripts or commands on a resource after creation (`remote-exec`, `local-exec`).
* Use **only when necessary**; prefer configuration management tools.

## 🎯 Why / Purpose

* Reliability: Provisioners can fail if resources aren’t ready.
* Maintainability: Inline scripts reduce module reusability.
* Separation: Terraform = infrastructure, Ansible/Chef/Puppet = configuration.

## ⚙️ How / Steps

1. `remote-exec` for commands on remote resources (SSH).
2. `local-exec` for commands on the local machine.
3. Use `null_resource` and `triggers` for repeatable provisioning.

## ⚠️ Common Mistakes

* Overusing provisioners for config tasks.
* Inline scripts with secrets.
* Not using `depends_on` properly.

## 🧩 Troubleshooting / Fixes

* Check connection/SSH errors.
* Test scripts separately.
* Externalize complex scripts.

## ✅ Best Practices

* Use only when no other option exists.
* Prefer Ansible, Chef, Puppet.
* Keep scripts minimal and simple.
* Avoid hardcoding secrets.
* Use `null_resource` with triggers for repeatable runs.


---
---

# Quick Revision — Using `terraform fmt`, `validate`, and Linting Tools

## 🧩 Concept / What

* Tools and commands to maintain **Terraform code quality, consistency, and correctness**:

  * `terraform fmt` → formats Terraform files.
  * `terraform validate` → checks syntax & structure.
  * Linting tools (`tflint`, `checkov`) → static analysis & best-practice enforcement.

## 🎯 Why / Purpose

* Ensures consistent code style.
* Detects syntax errors before apply.
* Enforces best practices and security compliance.
* Integrates with CI/CD pipelines for early error detection.

## ⚙️ How / Steps

1. `terraform fmt` → formats code (`terraform fmt`, `-recursive`).
2. `terraform validate` → validates configuration.
3. Linting tools → `tflint` or `checkov` for static analysis.

## ⚠️ Common Mistakes

* Skipping formatting → inconsistent code.
* Ignoring validate → deployment errors.
* Not integrating linters → missed security issues.
* Outdated Terraform version → validation/linter failures.

## 🧩 Troubleshooting / Fixes

* Run `terraform fmt` to fix formatting.
* Read validate errors carefully.
* Update linter rules/plugins for version differences.
* Ensure consistent Terraform and tool versions.

## ✅ Best Practices

* Run `terraform fmt` before commits.
* Validate configs locally and in CI/CD.
* Integrate linters (`tflint`, `checkov`) for best practices & security.
* Use pre-commit hooks.
* Keep Terraform and linter versions consistent across team/pipeline.

---
---

# Terraform Policy Enforcement – Sentinel / OPA (Policy as Code) – Quick Revision Notes

---

## Concept / What

* Policy as Code: enforce rules and policies on Terraform before applying changes.
* Tools: **Sentinel** (Enterprise), **OPA** (open-source), **TFLint**, **Checkov**, **TFSEC**.

---

## Purpose / Use Case

* Ensure infrastructure meets org standards, security, best practices.
* Automate checks in CI/CD pipelines.
* Examples: no public S3 buckets, approved instance types, least-privilege IAM.

---

## How / Steps

* **Sentinel:** attach rules to Terraform Enterprise workspaces, written in Sentinel language.
* **OPA:** write Rego policies, evaluate Terraform plan JSON.
* **TFLint / Checkov / TFSEC:** lint and scan Terraform code for best practices and security misconfigurations.
* **DevOps role:** integrate all checks into CI/CD pipelines; security team writes the policies.

---

## Common Issues

* Pipeline misconfiguration → policies not enforced.
* Bypassing checks if tools not executed.
* Misconfigured rules blocking valid deployments.

---

## Fixes / Troubleshooting

* Attach Sentinel policies to correct workspaces.
* Test Rego policies against sample Terraform plan for OPA.
* Integrate TFLint/Checkov/TFSEC properly in Jenkins/CI pipelines.
* Test on feature branches to ensure policies trigger.

---

## Best Practices / Tips

* DevOps focuses on integration, not writing policies.
* Security team defines rules/policies.
* Combine static analysis tools with OPA/Sentinel for full coverage.
* Automate policy checks on PRs in CI/CD.
* Maintain documentation of policies and pipeline integration.

---
---

# Terraform `.terraformignore` – Quick Revision Notes

---

## Concept / What

* `.terraformignore`: Terraform module file to **exclude specific files/folders** when publishing/sharing a module.
* Syntax similar to `.gitignore`, **not related to Git or local plan/apply**.

---

## Purpose / Use Case

* Avoid unnecessary, sensitive, or local-only files in module package.
* Reduce module size and clutter.
* Examples: `tests/`, `README.md`, `scripts/`, `*.DS_Store`.

---

## How / Steps / Syntax

1. Place `.terraformignore` in **module root**.
2. List patterns for files/folders to exclude.

**Example:**

```
tests/
README.md
scripts/
*.DS_Store
*.tmp
```

---

## Common Issues

* Wrong directory placement → ignored.
* Incorrect patterns → files still included.

---

## Fixes / Troubleshooting

* Ensure file in module root.
* Use correct relative paths.
* Test with local module packaging: `terraform package -module-dir=./my-module`.

---

## Best Practices

* Exclude only unnecessary/sensitive files.
* Keep minimal & clear.
* Combine with `.gitignore` for dev workflow.
* `.terraformignore` affects **publishing only**, not plan/apply.

---
---


