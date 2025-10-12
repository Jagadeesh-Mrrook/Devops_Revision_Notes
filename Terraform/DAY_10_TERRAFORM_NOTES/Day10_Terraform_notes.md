# Terraform Security & Best Practices – Introduction (Detailed Version)

---

## 🧩 **Detailed Explanation Version**

### **Concept / What:**

Security and Best Practices in Terraform are a set of guidelines and strategies designed to ensure your Infrastructure as Code (IaC) is secure, consistent, and compliant. Terraform interacts directly with your cloud infrastructure, so following these practices prevents exposure of secrets, configuration drift, and non-compliance issues.

### **Why / Purpose / Use Case in Real-World:**

Terraform configurations often contain credentials, resource definitions, and state files that reveal details about your infrastructure. Mismanaging these can lead to data breaches or outages.

Following security best practices helps:

* Protect sensitive data (e.g., API keys, passwords, private keys)
* Prevent accidental exposure of credentials to version control
* Maintain consistent coding standards across teams
* Reduce the risk of manual errors or configuration drift
* Ensure compliance with organizational and cloud security policies

**Example Real-World Scenarios:**

* A developer commits AWS credentials into GitHub → credentials get exposed publicly.
* Terraform state file contains secrets (e.g., RDS passwords) and is stored locally → leaked during a system backup.
* A misconfigured IAM role grants Terraform full admin access instead of least privilege.

### **How it Works / Steps / Syntax:**

Security & Best Practices in Terraform are implemented through multiple methods:

1. **Storing Secrets Securely**
   Use secret management tools (e.g., HashiCorp Vault, AWS Secrets Manager) or environment variables instead of hardcoding credentials.

2. **Sensitive Variables & Outputs**
   Use the `sensitive = true` argument in variables to mask them in logs or Terraform output.

3. **Version Control Discipline**
   Exclude state files, credential files, and temporary data using `.gitignore` and `.terraformignore`.

4. **Code Validation & Formatting**
   Use `terraform fmt`, `terraform validate`, and linting tools to ensure code is clean and consistent.

5. **Minimal Use of Provisioners**
   Avoid using `local-exec` or `remote-exec` unless absolutely necessary — these break idempotency.

6. **Policy Enforcement**
   Use Sentinel (Terraform Cloud) or OPA (Open Policy Agent) to enforce organizational policies.

7. **Least Privilege Principle**
   Terraform service accounts or IAM roles should have only the minimum required permissions.

### **Common Issues / Errors:**

* Accidental exposure of credentials in Git.
* Terraform apply fails due to missing environment variables.
* Inconsistent formatting causes merge conflicts.
* Overuse of provisioners leading to unpredictable deployments.
* Policy enforcement blocks due to noncompliance.

### **Troubleshooting / Fixes:**

* Rotate credentials immediately if leaked.
* Add sensitive files to `.gitignore` and `.terraformignore`.
* Validate Terraform code using `terraform validate` before apply.
* Fix policy violations as per Sentinel or OPA feedback.
* Limit Terraform state access to authorized users only.

### **Best Practices / Tips:**

* Store Terraform state remotely (e.g., AWS S3 with DynamoDB lock and encryption).
* Use CI/CD pipelines to automate validation and formatting.
* Encrypt sensitive outputs and backend data.
* Review Terraform plan outputs before apply.
* Audit IAM policies and access regularly.
* Rotate secrets and credentials periodically.

---
---

# Terraform: Storing Secrets Securely (Detailed Version)

---

## 🧩 **Detailed Explanation Version**

### **Concept / What:**

Storing secrets securely in Terraform means keeping sensitive information like passwords, API keys, and tokens protected while provisioning infrastructure. Secrets should **never be hardcoded** in `.tf` or `.tfvars` files, and should be managed using secure systems like **AWS Secrets Manager** or **environment variables**.

### **Why / Purpose / Use Case in Real-World:**

Secrets are required for authenticating with cloud providers, databases, and APIs. Mismanagement can lead to:

* **Data leaks** if secrets are committed to Git
* **Unauthorized access** to cloud resources
* **Regulatory non-compliance** (GDPR, SOC 2, ISO 27001)

**Examples:**

* Database password for RDS
* AWS access keys
* API tokens for third-party services

Proper secret management ensures security, compliance, and operational reliability.

### **How it Works / Steps / Syntax:**

#### **1. Using Environment Variables**

* Export secrets in memory rather than storing in files:

```bash
export TF_VAR_db_password="SuperSecret123"
```

* Define Terraform variable:

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

* Terraform automatically uses `TF_VAR_` prefixed environment variables during plan/apply.

**Benefits:**

* Secrets not stored in Git or local files
* Only present in memory during execution
* Works well in CI/CD pipelines (GitHub Actions, Jenkins)

#### **2. Using AWS Secrets Manager**

* **Step 1:** Store secret in AWS Secrets Manager

```bash
aws secretsmanager create-secret \
  --name prod/db_password \
  --secret-string '{"username":"admin","password":"SuperSecret123"}'
```

* **Step 2:** Fetch secret in Terraform using data sources

```hcl
provider "aws" {
  region = "us-east-1"
}

data "aws_secretsmanager_secret" "db_secret" {
  name = "prod/db_password"
}

data "aws_secretsmanager_secret_version" "db_secret_version" {
  secret_id = data.aws_secretsmanager_secret.db_secret.id
}

locals {
  db_creds = jsondecode(data.aws_secretsmanager_secret_version.db_secret_version.secret_string)
}

output "db_username" {
  value     = local.db_creds.username
  sensitive = true
}

output "db_password" {
  value     = local.db_creds.password
  sensitive = true
}
```

* **Step 3:** Optionally, export the secret as an environment variable if needed for CI/CD

```bash
export TF_VAR_db_password=$(aws secretsmanager get-secret-value \
  --secret-id prod/db_password \
  --query 'SecretString' \
  --output text | jq -r '.password')
```

### **Common Issues / Errors:**

| Issue                      | Cause                                                     |
| -------------------------- | --------------------------------------------------------- |
| Terraform plan/apply fails | Missing environment variable or wrong secret name         |
| AccessDeniedException      | IAM policy does not allow `secretsmanager:GetSecretValue` |
| Secret visible in logs     | Variable or output not marked `sensitive = true`          |
| JSON parse error           | Secret not stored in valid JSON format                    |

### **Troubleshooting / Fixes:**

* Ensure environment variables are correctly set (`TF_VAR_` prefix)
* Verify IAM permissions for accessing AWS Secrets Manager
* Use `sensitive = true` for all secret variables and outputs
* Check secret JSON formatting and key names
* Use encrypted remote state backend to prevent leakage

### **Best Practices / Tips:**

* Never hardcode secrets in `.tf` files
* Prefer AWS Secrets Manager or other secret management systems
* Mark all sensitive variables and outputs as `sensitive = true`
* Encrypt Terraform state files and restrict access
* Rotate secrets regularly
* For CI/CD pipelines, fetch secrets dynamically and optionally export as environment variables
* Use ephemeral secrets in CI/CD jobs — they disappear after the job completes

---
---

# Terraform Sensitive Outputs — Detailed Notes

## 🧩 What

Terraform allows marking output values as **sensitive** to prevent them from being displayed in the terminal or logs.
This feature helps protect secrets such as passwords, API keys, or tokens from accidental exposure.

---

## 💡 Why

Without `sensitive = true`, Terraform prints all outputs to:

* CLI after `terraform apply`
* Logs or CI/CD consoles (like Jenkins)
* Remote state files (if not properly secured)

This can lead to **security leaks** or **compliance issues**.

---

## ⚙️ How

You define sensitive outputs in your Terraform configuration like this:

```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

### Behavior:

* Terraform hides the value in the terminal:

  ```
  Outputs:
  db_password = <sensitive>
  ```
* Hidden in plan/apply output logs.
* Still stored in the Terraform state file (which must be secured, e.g., S3 with encryption and limited access).

---

## 🔍 Accessing Sensitive Outputs

You can still view them explicitly if needed:

```bash
terraform output db_password
```

This command prints the actual value, but only for authorized users who have:

* Access to the Terraform working directory
* Permission to read the backend (e.g., S3 bucket)
* Proper IAM permissions (if Terraform runs on EC2)

---

## 🧱 Example with AWS Secrets Manager

If your secrets come from AWS Secrets Manager:

```hcl
data "aws_secretsmanager_secret_version" "db_secret" {
  secret_id = "prod/db/password"
}

output "db_password" {
  value     = data.aws_secretsmanager_secret_version.db_secret.secret_string
  sensitive = true
}
```

This ensures the password fetched from Secrets Manager stays masked in logs or apply output.

---

## 🧠 Best Practices

1. Always mark outputs as sensitive if they contain any secret data.
2. Secure your backend (e.g., use encrypted S3 and restricted IAM).
3. Avoid printing sensitive outputs in CI/CD logs.
4. Control who can access Terraform state files.
5. Combine `sensitive = true` with `env var` or Secrets Manager usage for best security.

---

## ⚠️ Common Mistakes

* Assuming `sensitive = true` encrypts data — **it does not**. It only hides it from logs.
* Forgetting to secure state files — sensitive values are still stored in plain text inside them.
* Displaying sensitive outputs in automation scripts.

---

## 🔧 Troubleshooting

If outputs are still visible:

* Check if you accidentally printed them using `terraform output -json`
* Verify logging in Jenkins or pipelines — disable echo for sensitive outputs
* Re-run with `TF_LOG=ERROR` to suppress detailed logs

---

## ✅ Summary

| Aspect                 | Description                                        |
| ---------------------- | -------------------------------------------------- |
| Purpose                | Prevent secrets from appearing in logs or terminal |
| Key Option             | `sensitive = true`                                 |
| Visibility             | Hidden in CLI & logs                               |
| Still stored in state? | Yes (unencrypted unless backend encrypts it)       |
| Safe Access            | Use `terraform output <name>` manually             |
| Combine with           | Secrets Manager, IAM policies, backend encryption  |

---
---

# Terraform Version Control of Files — Detailed Notes

## 🧩 Concept / What

Version control in Terraform means managing your `.tf`, `.tfvars`, and related infrastructure-as-code files using systems like **Git (GitHub, GitLab, Bitbucket)**.
It helps track every change, collaborate across teams, and roll back to previous configurations when needed.

---

## 🎯 Why / Purpose / Real-World Use Case

1. **Collaboration:** Multiple DevOps engineers can safely work on infrastructure updates simultaneously.
2. **Change tracking:** Each commit represents a version (snapshot) of your infrastructure.
3. **Rollback:** If a change breaks infra, revert to a stable commit just like restoring a previous S3 object version.
4. **Code review:** Enables Pull/Merge Requests and approval workflows before applying infrastructure.
5. **Audit & compliance:** Every infrastructure change is traceable through commit history.

**Real Scenario:**
If a Terraform apply in production introduces a faulty configuration, you can revert the Git repo to a stable commit.
This works similarly to S3 versioning — reverting to an older, safe version of configuration files.

---

## ⚙️ How / Steps

### 1️⃣ Initialize Git Repository

```bash
git init
git add .
git commit -m "Initial Terraform setup"
```

### 2️⃣ Use `.gitignore`

Exclude sensitive and temporary Terraform files:

```
*.tfstate
*.tfstate.backup
.terraform/
crash.log
override.tf
override.tf.json
terraform.tfvars
*.pem
```

### 3️⃣ Follow Branching Strategy

* `main` → production infrastructure
* `dev` → test environment
* `feature/*` → feature or module updates

### 4️⃣ Use Tags/Releases

Tag stable infra states like `v1.0.0` to maintain version history.

```bash
git tag v1.0.0
git push origin v1.0.0
```

### 5️⃣ Integrate CI/CD

Run Terraform checks before merging:

```bash
terraform fmt -check
terraform validate
tflint
```

---

## 🧠 Common Issues

| Issue                     | Description                                                                |
| ------------------------- | -------------------------------------------------------------------------- |
| `.tfstate` pushed to repo | Exposes sensitive data like DB passwords or IPs                            |
| Merge conflicts           | Two users editing same `.tf` files simultaneously                          |
| Missing `.gitignore`      | Causes `.terraform` folder or tfvars to be committed                       |
| Hardcoded secrets         | Secrets stored in code instead of Secrets Manager or environment variables |

---

## 🧩 Troubleshooting / Fixes

* **Remove state file from repo**:

  ```bash
  git rm --cached terraform.tfstate
  git commit -m "Remove sensitive state file"
  git push
  ```
* Rotate exposed credentials immediately.
* Enforce **branch protection rules** on `main`.
* Use **pre-commit hooks** to validate code before commit.

---

## 🧠 Comparing with S3 Versioning

| Aspect   | S3 Versioning                   | Git Version Control             |
| -------- | ------------------------------- | ------------------------------- |
| Type     | Object versions                 | Code versions (commits)         |
| Restore  | Replace current with old object | Revert to old commit            |
| Loss     | New data overwritten            | New commits ignored in rollback |
| Recovery | Restore older object manually   | Recover commits via history     |
| Merge    | Not possible                    | Possible via rebase/cherry-pick |

---

## 💡 Best Practices

1. Never commit `.tfstate` or `.tfvars` files.
2. Always use `.gitignore` and `.terraformignore`.
3. Keep separate folders/branches for dev, QA, prod.
4. Use Pull Requests for all infra changes.
5. Protect `main` branch and enable approvals.
6. Integrate `terraform fmt`, `validate`, and `tflint` checks in CI/CD.
7. Tag stable versions (`v1.0.0`, `v2.0.0`, etc.) for tracking.
8. Store secrets externally (e.g., AWS Secrets Manager or Vault).
9. Document every change in commit messages and PRs.

---

## 🧩 Summary

| Aspect        | Description                                                                 |
| ------------- | --------------------------------------------------------------------------- |
| Purpose       | Track, manage, and revert Terraform infrastructure changes                  |
| Tool          | Git (GitHub, GitLab, Bitbucket)                                             |
| Protect Files | `.gitignore` & `.terraformignore`                                           |
| Risk          | Pushing tfstate/secrets to repo                                             |
| Rollback      | Restore previous commit/version                                             |
| Similarity    | Works like S3 versioning — older versions can be restored if current breaks |

---
---

# Terraform — Avoid Hardcoding Credentials (Detailed Notes)

## 🧩 Concept / What

Hardcoding credentials refers to **embedding sensitive data** like passwords, API keys, or AWS access keys directly in Terraform files (`.tf`, `.tfvars`) or scripts. This is a security risk and should be avoided. Instead, credentials should be **externalized** using environment variables, AWS Secrets Manager, Vault, or CI/CD secret management tools.

---

## 🎯 Why / Purpose / Real-World Use Case

1. **Security Risk:** Directly visible in code repositories or logs.
2. **Compliance:** Many organizations require secrets to be externalized (SOC2, PCI DSS).
3. **Flexibility:** Secrets can be rotated or updated without changing code.
4. **Reusability:** Same Terraform module works across multiple environments without embedding new credentials.

**Example Scenario:**

* Storing AWS access keys in `provider.tf` can allow anyone with repository access to misuse them.
* Using environment variables or AWS Secrets Manager mitigates this risk.

---

## ⚙️ How / Steps / Syntax

### 1️⃣ Environment Variables

```bash
export AWS_ACCESS_KEY_ID="your_access_key"
export AWS_SECRET_ACCESS_KEY="your_secret_key"
```

Terraform automatically picks these up with:

```hcl
provider "aws" {
  region = "us-east-1"
}
```

### 2️⃣ AWS Secrets Manager

```hcl
data "aws_secretsmanager_secret_version" "db_secret" {
  secret_id = "prod/db/password"
}

output "db_password" {
  value     = data.aws_secretsmanager_secret_version.db_secret.secret_string
  sensitive = true
}
```

### 3️⃣ Terraform Variables with `.tfvars`

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

* Store actual values in `terraform.tfvars` **not committed to Git**, or pass via `-var` flag or environment variable.

---

## ⚠️ Common Mistakes

* Committing `.tfvars` with secrets.
* Placing credentials directly in provider blocks (`access_key`, `secret_key`).
* Printing secrets in output without `sensitive = true`.
* Using default values for secret variables.
* **Accidentally pushing secrets to GitHub (public repo)**.

---

## 🧩 Troubleshooting / Fixes

* Remove hardcoded credentials from repo:

```bash
git rm --cached provider.tf
git commit -m "Remove hardcoded credentials"
```

* Rotate any secrets accidentally committed.
* Use environment variables or secret stores.
* Always mark sensitive outputs as `sensitive = true`.

### 🔹 Recovering from credentials pushed to GitHub

1. **Remove the file**:

```bash
git rm --cached provider.tf
git commit -m "Remove sensitive file"
git push
```

2. **Rewrite history** (remove from all previous commits):

```bash
bfg --delete-files provider.tf
# or using git filter-branch
```

3. **Force push after history rewrite**:

```bash
git push origin --force --all
```

4. **Rotate the credentials immediately** (access keys, passwords, tokens).
5. **Invalidate old credentials** and use new ones stored securely.

---

## ✅ Best Practices

1. Never hardcode credentials in `.tf` files.
2. Use **environment variables** or **AWS Secrets Manager / Vault**.
3. Mark sensitive variables as `sensitive = true`.
4. Do not commit `terraform.tfvars` containing secrets.
5. Secure Terraform state backend (S3 + KMS).
6. Rotate credentials regularly and automatically if possible.
7. If credentials are accidentally pushed, remove them from repo and history, and rotate immediately.

---

## 🧩 Summary Table

| Aspect                | Recommendation                                                               |
| --------------------- | ---------------------------------------------------------------------------- |
| Hardcoded credentials | Avoid at all costs                                                           |
| Recommended storage   | Env variables, Secrets Manager, Vault                                        |
| Output                | `sensitive = true`                                                           |
| Git                   | Never commit secrets; if accidentally pushed, remove from history and rotate |
| Backend               | Secure with encryption (KMS)                                                 |
| Rotation              | Regularly rotate and update externally                                       |

---
---

# Terraform — Minimal Use of Provisioners (Detailed Notes)

## 🧩 Concept / What

Provisioners in Terraform are used to **execute scripts or commands on a resource after it is created**, such as installing software on a VM. Common types are `remote-exec` and `local-exec`.

> Best practice: Use provisioners **only when absolutely necessary** because they are less reliable and harder to maintain compared to configuration management tools like Ansible, Chef, or Puppet.

---

## 🎯 Why / Purpose / Real-World Use Case

* **Reliability:** Provisioners can fail if the resource isn’t ready, leading to partial deployments.
* **Maintainability:** Inline scripts make Terraform modules harder to reuse.
* **Separation of Concerns:** Terraform should manage infrastructure; configuration management tools should handle software setup.

**Example Scenario:**

* Installing Nginx directly on EC2 via `remote-exec` is possible but not ideal.
* Better: Use Ansible to configure EC2 after Terraform provisions the instance.

---

## ⚙️ How / Steps / Syntax

### 1️⃣ `remote-exec` provisioner

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = "t2.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }
}
```

### 2️⃣ `local-exec` provisioner

```hcl
resource "null_resource" "example" {
  provisioner "local-exec" {
    command = "echo ${self.id} > result.txt"
  }
}
```

---

## ⚠️ Common Mistakes

* Overusing provisioners for configuration tasks instead of infrastructure tasks.
* Not handling failures properly — can block `terraform apply`.
* Mixing secrets in inline scripts — risk of exposure.
* Not using `depends_on` correctly, causing execution before resource is ready.

---

## 🧩 Troubleshooting / Fixes

* Check SSH or connection errors for `remote-exec`.
* Use `ignore_errors` for non-critical steps but prefer configuration management tools.
* Split complex scripts into external files instead of inline.
* Test scripts separately before adding to Terraform.

---

## ✅ Best Practices

1. Use provisioners **only when no other option exists**.
2. Prefer **Ansible, Chef, Puppet** for software/configuration management.
3. Keep inline scripts **simple and minimal**.
4. Avoid hardcoding secrets in provisioners.
5. Use `null_resource` with `triggers` for repeatable provisioning if necessary.

---

## 🧩 Summary Table

| Aspect              | Recommendation                                       |
| ------------------- | ---------------------------------------------------- |
| Use of Provisioners | Only when absolutely necessary                       |
| Preferred Tool      | Ansible, Chef, Puppet                                |
| Script Placement    | Keep minimal; externalize complex scripts            |
| Secrets             | Never hardcode in provisioners                       |
| Reliability         | Use `null_resource` and triggers for repeatable runs |

---
---

# Terraform — Using `terraform fmt`, `validate`, and Linting Tools (Detailed Notes)

## 🧩 Concept / What

Tools and commands that help maintain **Terraform code quality, consistency, and correctness**:

1. **`terraform fmt`** – Automatically formats Terraform files to a canonical style.
2. **`terraform validate`** – Checks the configuration for **syntax errors and structural correctness** without applying changes.
3. **Linting Tools (`tflint`, `checkov`)** – Static analysis tools that enforce **best practices, security checks, and policy compliance**.

---

## 🎯 Why / Purpose / Real-World Use Case

* **Consistency:** Ensures all team members follow the same style.
* **Error prevention:** Detects syntax or logical mistakes before deployment.
* **Compliance & Security:** Linting tools catch security misconfigurations or deprecated features.
* **CI/CD Integration:** These tools can be part of pre-commit hooks or pipeline checks.

**Example Scenario:**

* A developer forgets a required argument in a resource. `terraform validate` catches it before `apply`.
* `tflint` warns about insecure security group rules or outdated resource syntax.
* `terraform fmt` ensures indentation and spacing are consistent across all modules.

---

## ⚙️ How / Steps / Syntax

### 1️⃣ `terraform fmt`

```bash
# Format all Terraform files in the current directory
terraform fmt

# Recursively format all files in subdirectories
terraform fmt -recursive
```

### 2️⃣ `terraform validate`

```bash
# Validate the current configuration
terraform validate
```

### 3️⃣ Linting Tools

* **tflint** (Terraform linter)

```bash
# Install tflint
brew install tflint  # macOS
# Run linter
tflint
```

* **checkov** (Terraform security linter)

```bash
# Scan current directory
checkov -d .
```

---

## ⚠️ Common Mistakes

* Skipping formatting → inconsistent code across team members.
* Ignoring `validate` → leads to failed `apply` runs in CI/CD.
* Not integrating linting → security misconfigurations may go unnoticed.
* Using outdated Terraform version → `validate` or linter may fail.

---

## 🧩 Troubleshooting / Fixes

* Run `terraform fmt` to fix formatting issues.
* Read `validate` errors carefully; missing arguments or invalid blocks are common.
* Update linter rules or plugins if errors appear due to version differences.
* Ensure CI/CD pipeline uses the same Terraform version as local development.

---

## ✅ Best Practices

1. Always run `terraform fmt` before committing code.
2. Validate configuration (`terraform validate`) locally and in CI/CD pipelines.
3. Integrate **tflint** or **checkov** for best-practice and security checks.
4. Enforce pre-commit hooks to catch issues early.
5. Keep Terraform and linter versions consistent across team and pipelines.

---

## 🧩 Summary Table

| Aspect              | Recommendation                                            |
| ------------------- | --------------------------------------------------------- |
| Formatting          | Always use `terraform fmt`                                |
| Syntax / Validation | Always run `terraform validate`                           |
| Linting             | Use `tflint` or `checkov` for best practices and security |
| CI/CD               | Integrate fmt, validate, and linter in pipelines          |
| Version             | Keep Terraform and tool versions consistent               |

---
---


# Terraform Policy Enforcement – Sentinel / OPA (Policy as Code) – Detailed Notes

---

## Concept / What

* Policy as Code allows organizations to define **rules and policies** that Terraform must follow before applying infrastructure changes.
* Tools: **Sentinel** (Terraform Enterprise/Cloud), **OPA (Open Policy Agent)** for open-source Terraform.
* Supporting tools for static checks: **TFLint** (linting), **Checkov**, **TFSEC** (security scanning).

---

## Why / Purpose / Use Case

* Ensures infrastructure complies with organizational standards, security requirements, and best practices.
* Automates policy enforcement in CI/CD pipelines.
* Example use cases:

  * No public S3 buckets.
  * Only approved instance types can be used.
  * IAM policies follow least-privilege rules.

---

## How / Steps / Syntax

* **Sentinel (Terraform Enterprise)**:

  * Policies written in **Sentinel language**.
  * Attached to workspaces; Terraform plan evaluated against rules.
  * Example rule (pseudo):

    ```sentinel
    import "tfplan/v2" as tfplan
    main = rule {
      all tfplan.resources.aws_s3_bucket as _, b {
        b.applied.public == false
      }
    }
    ```

* **OPA (Open Policy Agent)**:

  * Policies written in **Rego**.
  * Evaluates Terraform plan JSON output.
  * Example pseudo-rule:

    ```rego
    package terraform.s3
    deny[msg] {
      resource := input.resource_changes[_]
      resource.type == "aws_s3_bucket"
      resource.change.after.public == true
      msg = sprintf("Public bucket not allowed: %v", [resource.address])
    }
    ```

* **TFLint, Checkov, TFSEC**:

  * TFLint → lints Terraform code for best practices, deprecated resources.
  * Checkov / TFSEC → scan Terraform files for security misconfigurations.
  * Integrated in CI/CD pipeline to check PRs before merging.

* **DevOps role:**

  * Security team defines rules/policies.
  * DevOps engineers integrate these tools in pipelines.
  * Automated checks triggered on every PR to enforce policies.

---

## Common Issues / Errors

* Policies not applied because the pipeline isn’t properly integrated.
* Terraform plan changes bypass policy checks if OPA/Sentinel not executed.
* Misconfigured Rego/Sentinel rules can block valid deployments.

---

## Troubleshooting / Fixes

* Ensure Sentinel policies are attached to the correct Terraform Enterprise workspaces.
* For OPA, validate Rego policies against sample Terraform plan JSON before CI/CD integration.
* Integrate TFLint/Checkov/TFSEC checks properly in Jenkins/CI pipelines.
* Test pipeline on feature branches to ensure policies trigger correctly.

---

## Best Practices / Tips

* DevOps engineers focus on **integration and automation**, not writing policies.
* Security or infrastructure team defines and maintains policy rules.
* Combine static analysis tools (TFLint, Checkov, TFSEC) with policy enforcement (OPA/Sentinel) for full coverage.
* Automate checks in CI/CD pipelines to prevent policy violations before deploying to cloud.
* Maintain clear documentation of policies and pipeline integration for audits.

---
---

# Terraform `.terraformignore` – Detailed Notes

---

## Concept / What

* `.terraformignore` is a Terraform-specific file used in **module directories** to exclude certain files or folders when **publishing or sharing a module**.
* Similar in syntax to `.gitignore`, but its scope is **Terraform module packaging**, not Git tracking or local execution.

---

## Why / Purpose / Use Case

* Prevents unnecessary, sensitive, or local-only files from being included in a module package.
* Reduces module size and clutter for consumers.
* Ensures only the **required Terraform code** is published.
* Examples of files to exclude: tests, documentation, local scripts, temporary/system files.

---

## Terraform Module Directory Structure Example

```
my-module/
│
├── main.tf           # module code
├── variables.tf      # input variables
├── outputs.tf        # output variables
├── README.md         # documentation
├── tests/            # local tests
│   └── test_main.tf
├── scripts/          # helper scripts
│   └── helper.sh
├── .terraformignore  # file to exclude unwanted files
└── .DS_Store         # system file
```

* Only `main.tf`, `variables.tf`, and `outputs.tf` are essential for the module.

---

## How / Steps / Syntax

1. Create a `.terraformignore` file in the **module root directory**.
2. List files and directories to exclude (patterns relative to module root).

**Example `.terraformignore`:**

```
# Ignore test files
tests/

# Ignore documentation
README.md

# Ignore system/temp files
*.DS_Store
*.tmp

# Ignore local scripts
scripts/
```

3. Publish or share the module. Terraform will **skip these files** in the package.

---

## Common Issues / Errors

* Placing `.terraformignore` in a subdirectory → has no effect.
* Forgetting to include sensitive files → accidental exposure in published module.
* Incorrect patterns → files still included in module package.

---

## Troubleshooting / Fixes

* Ensure `.terraformignore` is in the **root of the module**.
* Use correct relative paths for files/folders to exclude.
* Test module packaging locally to verify ignored files:

```bash
terraform package -module-dir=./my-module
```

---

## Best Practices / Tips

* Always use `.terraformignore` for modules that will be shared externally.
* Exclude tests, documentation, temp files, local scripts, and sensitive files.
* Keep the ignore file **minimal and clear**, only excluding what’s necessary.
* Combine with `.gitignore` for development, but remember they serve different purposes.
* `.terraformignore` does **not affect `terraform plan` or `apply`**, only module publishing.

---
---


