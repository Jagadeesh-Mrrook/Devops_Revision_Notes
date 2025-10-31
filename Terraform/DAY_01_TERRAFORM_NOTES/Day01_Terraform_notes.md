# Terraform Fundamentals: What is Terraform & Why It's Used

## Concept
- Terraform is an **open-source Infrastructure as Code (IaC) tool** to define, provision, and manage infrastructure across cloud and on-prem environments using declarative configuration files.

## Purpose / Use Case in Real-World
- Ensures **consistent and reproducible infrastructure** across dev, staging, and prod.
- Enables **version control** for infrastructure changes.
- Supports **multi-cloud and hybrid environments**.
- Automates infrastructure provisioning and facilitates **team collaboration**.
- Plans changes before applying to reduce human error.

## How it Works / Steps / Syntax
- Write `.tf` configuration files defining resources and providers.
- Use `terraform init` to initialize the project.
- Use `terraform plan` to preview changes.
- Use `terraform apply` to provision infrastructure.

## Common Issues / Errors
- Misconfigured provider credentials → authentication failures.
- Applying resources in unsupported regions.
- Dependency errors between resources.

## Troubleshooting / Fixes
- Verify provider credentials and access permissions.
- Check resource availability in the selected region.
- Use `terraform plan` to inspect changes before applying.

## Best Practices / Tips
- Version control all `.tf` files in Git.
- Use workspaces for managing multiple environments.
- Avoid hardcoding sensitive data; use variables and secret management.
- Run `terraform fmt` for consistent formatting.

---

# Terraform Fundamentals: Architecture & Workflow

## Concept
- Terraform Architecture consists of:
  1. **Terraform Core** – reads config files, builds dependency graph, generates execution plan.
  2. **Providers** – plugins for interacting with cloud platforms (AWS, Azure, GCP) or services (GitHub, Cloudflare).
  3. **State** – `terraform.tfstate` file tracks current infrastructure and changes.

## Purpose / Use Case
- Manage infrastructure collaboratively with accurate state tracking.
- Detect and reconcile drift between desired configuration and actual resources.
- Integrate with CI/CD pipelines and remote state backends for teams.

## How it Works / Steps / Syntax
1. Write `.tf` configuration files (resources, variables, providers).
2. Initialize project: `terraform init` → installs providers and sets environment.
3. Plan changes: `terraform plan` → previews changes.
4. Apply changes: `terraform apply` → provisions or modifies resources.
5. Manage state: Terraform updates `.tfstate` to reflect current infra.
6. Destroy resources: `terraform destroy` → deletes all defined resources.

## Common Issues / Errors
- State file conflicts → lock errors in team environments.
- Missing provider plugins → errors during `terraform init`.
- Configuration drift → resources modified outside Terraform.

## Troubleshooting / Fixes
- Use remote state with locking (e.g., AWS S3 + DynamoDB).
- Run `terraform init -upgrade` to update provider plugins.
- Use `terraform refresh` to sync state with actual infrastructure.

## Best Practices / Tips
- Keep state files secure and version-controlled (or use remote state).
- Modularize code for reuse across environments.
- Always review `terraform plan` before applying.
- Use workspaces or separate state files for different environments.

---

# Terraform Installation (Linux)

## Concept
- Installing Terraform on Linux enables CLI usage to provision and manage infrastructure.

## Purpose / Use Case
- Most DevOps and cloud automation runs on Linux servers.
- Allows integration with CI/CD tools like Jenkins, GitLab CI, etc.

## How it Works / Steps / Syntax
1. Download Terraform: use `wget https://releases.hashicorp.com/terraform/<version>/terraform_<version>_linux_amd64.zip`
2. Unzip the package: use `unzip terraform_<version>_linux_amd64.zip`
3. Move binary to `/usr/local/bin`: use `sudo mv terraform /usr/local/bin/`
4. Verify installation: use `terraform version`

## Common Issues / Errors
- `terraform: command not found` → binary not in PATH.
- Permission errors when moving binary.
- Version mismatch if scripts require a different Terraform version.

## Troubleshooting / Fixes
- Ensure `/usr/local/bin` is in your PATH.
- Use `sudo` to move the binary if permission denied.
- Use `tfenv` to manage multiple Terraform versions.

## Best Practices / Tips
- Use `tfenv` for version management.
- Check release notes before installing.
- Verify installation with `terraform version` before starting work.

---

# tfenv - Terraform Version Management

## Concept
- `tfenv` is a Terraform version manager that allows installing, switching, and managing multiple Terraform versions on the same system.

## Purpose / Use Case
- Avoid version conflicts across projects.
- Ensure consistent Terraform version in team projects.
- Easily test infrastructure with different Terraform versions.
- Integrate with CI/CD pipelines requiring specific versions.

## How it Works / Steps
1. Install `tfenv`:
   git clone https://github.com/tfutils/tfenv.git ~/.tfenv
   echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
   source ~/.bashrc
2. List available Terraform versions: `tfenv list-remote`
3. Install a specific version: `tfenv install 1.6.0`
4. Set a global default version: `tfenv use 1.6.0`
5. Set a project-specific version: create `.terraform-version` in project directory with desired version.

## Common Issues / Errors
- PATH not set correctly → `tfenv: command not found`.
- Installed version not recognized → need to re-source shell.
- Conflicts with manually installed Terraform binaries.

## Troubleshooting / Fixes
- Ensure `~/.tfenv/bin` is in PATH.
- Use `tfenv list` to verify installed and active versions.
- Remove old manual Terraform binaries from `/usr/local/bin` if conflicting.

## Best Practices / Tips
- Use `.terraform-version` per project to enforce version consistency.
- Keep `.terraform-version` in Git for team alignment.
- Regularly check for new Terraform releases via `tfenv list-remote`.

---

# Terraform CLI Commands - Detailed Notes

## 1. terraform init

### **Concept**

Initializes a Terraform project by:

* Downloading required provider plugins
* Fetching module configuration files into `.terraform/modules`
* Configuring the backend (local or remote like S3 + DynamoDB)

---

### **Purpose / Use Case**

* Must be run **before** `plan` or `apply`
* Sets up **backend**, **providers**, and **modules** for the workspace
* Used when switching backend configurations or upgrading providers

---

### **Syntax**

```bash
terraform init [options]
```

---

## Terraform Init - Real-World Options

### 1. `-backend-config="key=value"`

* **Description:** Pass backend configuration values directly via CLI.
* **Use Case:** Override or supply specific backend parameters (e.g., bucket name, region) dynamically.
* **Example:**

  ```bash
  terraform init -backend-config="bucket=my-tf-state" -backend-config="region=ap-south-1"
  ```

### 2. `-backend-config=<FILE>`

* **Description:** Load backend settings from a separate configuration file.
* **Use Case:** Keep backend details (like S3 bucket, key, region) organized and versioned separately.
* **Example:**

  ```bash
  terraform init -backend-config=backend.hcl
  ```

### 3. `-reconfigure`

* **Description:** Reinitialize the backend, ignoring any saved configuration.
* **Use Case:** Used when switching environments or changing backend type (e.g., from local to S3).
* **Example:**

  ```bash
  terraform init -backend-config=env/qa/backend.hcl -reconfigure
  ```

### 4. `-migrate-state`

* **Description:** Moves (migrates) the current Terraform state to a new backend.
* **Use Case:** When changing backend configuration (e.g., migrating state from one S3 bucket to another).
* **Example:**

  ```bash
  terraform init -backend-config=new-backend.hcl -migrate-state
  ```

### 5. `-upgrade`

* **Description:** Upgrade all Terraform provider and module versions to the latest allowed by configuration.
* **Use Case:** Ensure provider plugins are up-to-date, especially after version changes in configuration.
* **Example:**

  ```bash
  terraform init -upgrade
  ```

### 6. `-input=false`

* **Description:** Disables interactive input prompts (used in automation pipelines).
* **Use Case:** Prevent manual input during automated Terraform initialization.
* **Example:**

  ```bash
  terraform init -input=false
  ```

### 7. `-lockfile=readonly`

* **Description:** Prevents modification of the `.terraform.lock.hcl` file.
* **Use Case:** Used in production pipelines where dependency versions must remain consistent.
* **Example:**

  ```bash
  terraform init -lockfile=readonly
  ```

---

### **Common Issues**

* Provider plugin missing or outdated
* Backend misconfiguration (incorrect bucket/key/region)
* State migration conflicts

---

### **Troubleshooting**

* Verify backend settings in `.tf` or `.hcl` files
* Use `terraform init -reconfigure` when backend changes
* Run `terraform init -upgrade` to refresh plugins and modules

---

## 2. terraform plan
### Concept
- Shows execution plan: what Terraform will create, modify, or destroy.
### Purpose / Use Case
- Preview changes before applying to avoid unintended modifications.
### Syntax
- terraform plan
- Optional: `-out=planfile` → save plan for later apply
## Terraform Plan - Commonly Used Real-world Options
## Terraform Plan - Real-World Options

### 1. `-out=<FILE>`

* **Description:** Saves the generated execution plan to a file.
* **Use Case:** To review or apply the exact same plan later (ensures no drift between plan and apply).
* **Example:**

  ```bash
  terraform plan -out=plan.tfplan
  ```

### 2. `-var` and `-var-file`

* **Description:** Pass variables directly or from a `.tfvars` file.
* **Use Case:** Useful for environment-based variable management (dev, qa, prod).
* **Examples:**

  ```bash
  terraform plan -var="instance_type=t3.micro"
  terraform plan -var-file=qa.tfvars
  ```

### 3. `-lock` and `-lock-timeout`

* **Description:** Controls Terraform's state locking behavior.
* **Use Case:** Prevents concurrent modifications to the state file during planning.
* **Examples:**

  ```bash
  terraform plan -lock=false
  terraform plan -lock-timeout=5m
  ```

  > If Terraform can’t acquire the state lock within 5 minutes, the plan fails.

### 4. `-input=false`

* **Description:** Disables interactive variable prompts.
* **Use Case:** Used in automation pipelines to avoid Terraform asking for input.
* **Example:**

  ```bash
  terraform plan -input=false
  ```

### 5. `-parallelism=N`

* **Description:** Defines how many resources Terraform can plan in parallel.
* **Default:** 10
* **Use Case:** Control concurrency for large-scale infrastructure planning.
* **Example:**

  ```bash
  terraform plan -parallelism=5
  ```

### 6. `-refresh=false`

* **Description:** Skips refreshing the real infrastructure state before generating a plan.
* **Use Case:** Speeds up planning when you know the infrastructure hasn’t changed.
* **Example:**

  ```bash
  terraform plan -refresh=false
  ```

### 7. `-destroy`

* **Description:** Creates a plan to destroy all managed resources.
* **Use Case:** Used before decommissioning environments or testing teardown.
* **Example:**

  ```bash
  terraform plan -destroy -out=destroy.tfplan
  ```


### Common Issues
- Invalid configuration
- Missing variables
### Troubleshooting
- Use `terraform validate`
- Provide variables via `-var` or `.tfvars`

---

## 3. terraform apply
### Concept
- Applies changes to achieve the desired infrastructure state.
### Purpose / Use Case
- Provision or update resources; can use saved plan for safety.
* **Syntax:**

  * `terraform apply`
  * Optional: `terraform apply planfile`
  * Optional: `terraform apply -target=RESOURCE` (to apply specific resources only)
## Terraform Apply - Real-World Options

### 1. `-auto-approve`

* **Description:** Skips the interactive approval step (no need to type 'yes').
* **Use Case:** Used in automation pipelines (CI/CD) for non-interactive applies.
* **Example:**

  ```bash
  terraform apply -auto-approve
  ```

### 2. `-var` and `-var-file`

* **Description:** Pass variables directly or load from a variable file.
* **Use Case:** Provide environment-specific configurations.
* **Examples:**

  ```bash
  terraform apply -var="instance_type=t2.micro"
  terraform apply -var-file=dev.tfvars
  ```

### 3. `-lock` and `-lock-timeout`

* **Description:** Prevent multiple Terraform runs from changing the state at the same time.
* **Use Case:** Ensures state file consistency in team environments.
* **Examples:**

  ```bash
  terraform apply -lock=false
  terraform apply -lock-timeout=5m
  ```

### 4. `-parallelism=N`

* **Description:** Limits how many resources Terraform applies in parallel.
* **Default:** 10
* **Use Case:** Control speed and avoid provider (e.g., AWS) API rate limits.
* **Example:**

  ```bash
  terraform apply -parallelism=5
  ```

  > Example: If you have 20 resources and use `-parallelism=5`, Terraform creates **5 at a time**, in **4 batches total**, but the apply command runs **once**.

### 5. `-input=false`

* **Description:** Disables interactive variable prompts. Terraform fails if required variables are missing.
* **Use Case:** Used in CI/CD pipelines where manual input isn’t possible.
* **Example:**

  ```bash
  terraform apply -input=false
  ```


### Common Issues
- Authentication failures
- Resource conflicts due to manual changes
### Troubleshooting
- Verify credentials
- Use `terraform refresh` to sync state

---

## 4. terraform destroy
### Concept
- Deletes all resources defined in the configuration.
### Purpose / Use Case
- Clean up dev/test environments to save cost.
### Syntax
- terraform destroy
- Optional flags:
  - `-target=resource_name` → destroy specific resource
- `-auto-approve` → skip confirmation

## Terraform Destroy - Real-World Options

### 1. `-auto-approve`

* **Description:** Skips the interactive approval step (no need to type 'yes').
* **Use Case:** Commonly used in automation or CI/CD pipelines where manual confirmation isn't possible.
* **Example:**

  ```bash
  terraform destroy -auto-approve
  ```

### 2. `-var` and `-var-file`

* **Description:** Pass variable values directly or from a variable file.
* **Use Case:** Specify environment-specific configurations for destruction (e.g., destroy only a dev environment).
* **Examples:**

  ```bash
  terraform destroy -var="env=dev"
  terraform destroy -var-file=dev.tfvars
  ```

### 3. `-lock` and `-lock-timeout`

* **Description:** Controls Terraform's state file locking behavior during destroy.
* **Use Case:** Prevent concurrent destroy operations or state corruption in shared backends.
* **Examples:**

  ```bash
  terraform destroy -lock=false
  terraform destroy -lock-timeout=5m
  ```

### 4. `-target=resource_type.name`

* **Description:** Destroys only the specified resource and its dependencies.
* **Use Case:** Useful when you want to delete a specific resource instead of the entire infrastructure.
* **Example:**

  ```bash
  terraform destroy -target=aws_instance.app_server
  ```

### 5. `-input=false`

* **Description:** Disables interactive input prompts.
* **Use Case:** Used in CI/CD pipelines where user input is not available.
* **Example:**

  ```bash
  terraform destroy -input=false
  ```

### 6. `-refresh=false`

* **Description:** Skips refreshing remote objects before destruction.
* **Use Case:** Speeds up destroy in automation when you are sure resources are already in the known state.
* **Example:**

  ```bash
  terraform destroy -refresh=false
  ```

### 7. `-parallelism=N`

* **Description:** Limits how many resources Terraform destroys in parallel.
* **Default:** 10
* **Use Case:** Controls API rate limits or ensures graceful teardown in large infrastructures.
* **Example:**

  ```bash
  terraform destroy -parallelism=3
  ```

### Common Issues
- Dependency errors
- Authentication failures
### Troubleshooting
- Destroy resources in dependency order
- Verify cloud credentials and permissions

---

## 5. terraform validate
### Concept
- Checks syntax and configuration validity without applying changes.
### Purpose / Use Case
- Ensures configuration is valid before plan/apply.
### Syntax
- terraform validate
### Common Issues
- Syntax errors
- Unsupported resource or attribute names
### Troubleshooting
- Correct syntax errors
- Verify provider version compatibility

---

## 6. terraform fmt
### Concept
- Formats Terraform files to standard style.
### Purpose / Use Case
- Keep code consistent and readable across team.
### Syntax
- terraform fmt
- Optional: `-recursive` → format all subdirectories
## Terraform Fmt - Real-World Options

### 1. `-recursive`

* **Description:** Formats all `.tf` and `.tfvars` files in the current directory **and all subdirectories**.
* **Use Case:** Commonly used in repositories containing multiple modules or environments.
* **Example:**

  ```bash
  terraform fmt -recursive
  ```

---

### 2. `-check`

* **Description:** Checks if Terraform files are properly formatted without changing them.
* **Use Case:** Widely used in **CI/CD pipelines** to enforce Terraform formatting standards.
* **Example:**

  ```bash
  terraform fmt -check
  ```

---

### 3. `-diff`

* **Description:** Displays the differences between formatted and unformatted files.
* **Use Case:** Useful during reviews to see what changes `terraform fmt` would make before applying them.
* **Example:**

  ```bash
  terraform fmt -diff
  ```

---

### 4. `-list=false`

* **Description:** Disables printing the list of files that were formatted.
* **Use Case:** Used in automation scripts where only success/failure status matters, not filenames.
* **Example:**

  ```bash
  terraform fmt -list=false
  ```

---

### 5. `-write=false`

* **Description:** Checks formatting but doesn’t modify any files (only reports differences).
* **Use Case:** Used in pre-commit hooks or validation pipelines to verify format compliance.
* **Example:**

  ```bash
  terraform fmt -write=false
  ```

---

### 6. `-no-color`

* **Description:** Disables colored output in terminal.
* **Use Case:** Helpful when redirecting output to log files or external tools.
* **Example:**

  ```bash
  terraform fmt -no-color
  ```

### Common Issues
- Minimal; mainly style-related
### Troubleshooting
- Use `terraform fmt -recursive` to fix all files

---

## 7. terraform taint
### Concept
- Marks a resource as "tainted," forcing Terraform to destroy and recreate it on next apply.
### Purpose / Use Case
- Fix broken or misconfigured resources without manual deletion.
- Useful in CI/CD pipelines for automated resource recovery.
- Example: Replace failing VM, database, or load balancer.
# Terraform Taint and State File Behavior

* `terraform taint` marks a resource for replacement in the state; it is not immediately removed.
* On `terraform apply`, the old resource is destroyed and a new one is created using the current config.
* The state file is updated after apply with the new resource ID and attributes.
* The state always reflects the current infrastructure; it is never deleted.

### Syntax
- terraform taint <resource_type.resource_name>
- Example: terraform taint aws_instance.web_server

### Terraform Untaint

* **Concept:** Marks a previously tainted resource as untainted, so Terraform will no longer destroy and recreate it on the next apply.
* **Syntax:** terraform untaint <resource_type.resource_name>
* **Example:** terraform untaint aws_instance.web_server
* **Use Case:** If a resource was tainted by mistake, you can untaint it to avoid unnecessary replacement.


### Common Issues / Errors
- Resource not found → wrong name/type
- State mismatch → resource deleted manually
### Troubleshooting / Fixes
- Use `terraform state list` to verify resource name
- If deleted manually, remove from state: terraform state rm <resource_type.resource_name>
### Best Practices / Tips
- Use sparingly; only taint resources that need recreation
- Run `terraform plan` before apply
- Avoid critical production resources without backups
---

# 7.1 terraform apply -replace

## Concept

Forces Terraform to destroy and recreate a specific resource **in the same apply run**, without marking it as tainted.

## Purpose / Use Case

Replace broken or misconfigured resources in a single step.
Safer and cleaner alternative to `terraform taint` in automation and CI/CD pipelines.
Useful when you want to avoid a separate taint step and ensure state consistency.

## Terraform Apply -Replace and State File Behavior

* Does **not mark** the resource as tainted.
* Terraform generates a plan showing the resource will be **destroyed and recreated**.
* During apply:

  1. Old resource is destroyed in the cloud.
  2. New resource is created according to the configuration.
* The state file is updated **atomically**: old resource is removed, new resource ID and attributes are added.
* After apply, the state fully reflects the current infrastructure; no tainted flags remain.

## Syntax

```bash
terraform apply -replace="<resource_type.resource_name>"
```

## Example

```bash
terraform apply -replace="aws_instance.web_server"
```

## Key Differences from `terraform taint`

* Single-step operation; no separate `taint` + `apply`.
* State file is updated automatically during apply.
* Does not leave a “tainted” mark in the state.
* Recommended approach in Terraform ≥0.15.

## Common Issues / Errors

* Resource not found → wrong name/type
* Dependencies may require additional replacements

## Troubleshooting / Fixes

* Use `terraform state list` to verify resource names
* Check plan output to see what will be replaced
* Ensure configuration matches desired resource attributes

## Best Practices / Tips

* Use for replacing resources safely in automation
* Run `terraform plan -replace="..."` first to preview changes
* Avoid replacing critical production resources without backups

---
---



## 8. terraform import
### Concept
- Imports existing infrastructure into Terraform state.
### Purpose / Use Case
- Onboard existing resources (VMs, S3, databases) into Terraform management.
- Avoids manual recreation of production infrastructure.
- Helps migrate manual infra into Infrastructure-as-Code.
### Syntax
- terraform import <resource_type.resource_name> <resource_id>
- Example: terraform import aws_instance.web_server i-1234567890abcdef0
## Terraform Import - Real-World Options

### 1. `-config=PATH`

* **Description:** Specifies the path to the Terraform configuration directory (defaults to current directory).
* **Use Case:** Used when importing resources into configurations stored in a different folder.
* **Example:**

  ```bash
  terraform import -config=./vpc aws_vpc.main vpc-1234abcd
  ```

---

### 2. `-var`

* **Description:** Passes variable values directly from the command line.
* **Use Case:** Helpful when importing resources that depend on dynamic variable values.
* **Example:**

  ```bash
  terraform import -var="env=dev" aws_s3_bucket.my_bucket my-app-dev
  ```

---

### 3. `-var-file`

* **Description:** Loads variables from a `.tfvars` file.
* **Use Case:** Used in real-world setups to import resources into environment-specific configurations.
* **Example:**

  ```bash
  terraform import -var-file=dev.tfvars aws_vpc.main vpc-1234abcd
  ```

---

### 4. `-input=false`

* **Description:** Disables interactive variable prompts.
* **Use Case:** Used in CI/CD pipelines or automation where no user input is available.
* **Example:**

  ```bash
  terraform import -input=false aws_s3_bucket.logs_bucket my-logs
  ```

---

### 5. `-lock` and `-lock-timeout`

* **Description:** Controls Terraform state file locking during import.
* **Use Case:** Prevents simultaneous import operations that could corrupt the state file.
* **Examples:**

  ```bash
  terraform import -lock=false aws_instance.web i-0123abcd
  terraform import -lock-timeout=5m aws_vpc.main vpc-1234abcd
  ```

---

### 6. `-no-color`

* **Description:** Disables colored output in the terminal.
* **Use Case:** Used when sending logs to external systems that don’t support color codes.
* **Example:**

  ```bash
  terraform import -no-color aws_vpc.main vpc-0a1b2c3d4e
  ```

---

### 7. `-state=PATH`

* **Description:** Specifies a custom state file location for the import operation.
* **Use Case:** Useful for managing imports in multi-environment setups with different state files.
* **Example:**

  ```bash
  terraform import -state=./dev.tfstate aws_vpc.main vpc-0abc123
  ```

### Key Notes
- Only imports into state; does not generate `.tf` config
- After import, manually write `.tf` block matching imported resource
- Can import into modules or workspaces
### Common Issues / Errors
- Wrong resource type or ID → import fails
- Mismatch between imported resource and `.tf` config → unwanted changes
### Troubleshooting / Fixes
- Verify resource type and ID
- Run `terraform plan` after import
- Update `.tf` files to match imported resource configuration
### Best Practices / Tips
- Backup state before importing
- Start with non-critical resources
- Use import + plan to confirm no unexpected changes

---


## 9. terraform refresh

### Concept

Updates the Terraform state file to match the real-world infrastructure.

### Purpose / Use Case

* Detect and resolve **drift** between Terraform configuration and actual infrastructure.
* Ensures state file reflects current resource properties.
* Useful before running `terraform plan` or `terraform apply` to see real-time changes.

### Syntax

```
terraform refresh
```
## Terraform Refresh - Real-World Options

### 1. `-var`

* **Description:** Passes variable values directly from the command line.
* **Use Case:** Useful when the refresh depends on environment-specific variables that aren't set by default.
* **Example:**

  ```bash
  terraform refresh -var="env=dev"
  ```

---

### 2. `-var-file`

* **Description:** Loads variables from a `.tfvars` file.
* **Use Case:** Commonly used in multi-environment setups to refresh state for specific environments (dev, qa, prod).
* **Example:**

  ```bash
  terraform refresh -var-file=dev.tfvars
  ```

---

### 3. `-lock` and `-lock-timeout`

* **Description:** Controls Terraform state file locking during the refresh process.
* **Use Case:** Prevents multiple concurrent refresh operations that could cause state corruption.
* **Examples:**

  ```bash
  terraform refresh -lock=false
  terraform refresh -lock-timeout=5m
  ```

---

### 4. `-input=false`

* **Description:** Disables interactive variable prompts.
* **Use Case:** Used in automation and CI/CD pipelines where manual input is not possible.
* **Example:**

  ```bash
  terraform refresh -input=false
  ```

---

### 5. `-no-color`

* **Description:** Disables color output in the terminal.
* **Use Case:** Ideal for CI/CD logs or systems that don’t support ANSI color codes.
* **Example:**

  ```bash
  terraform refresh -no-color
  ```

---

### 6. `-target=resource_type.name`

* **Description:** Refreshes only the specified resource instead of the entire state.
* **Use Case:** Useful when you want to update the state of a specific resource after a manual change in AWS or another provider.
* **Example:**

  ```bash
  terraform refresh -target=aws_instance.web_server
  ```


### Example

```
terraform refresh
```

This will read the current state of all resources and update the state file without making any changes to the infrastructure.

### Key Notes

* **Does not create, modify, or destroy resources** — only updates the state.
* Can detect manual changes done outside Terraform.
* Useful in multi-team environments where infra can be modified directly.

### Common Issues / Errors

* Resources deleted outside Terraform → refresh marks them as “tainted” or missing.
* Permission or API issues → unable to read resource state.

### Troubleshooting / Fixes

* Verify credentials and API access.
* Check resource IDs in configuration match actual infrastructure.
* Run `terraform plan` after refresh to review drift.

### Best Practices / Tips

* Always run `terraform refresh` before `plan` in environments with potential manual changes.
* Backup the state file before refreshing, especially in production.
* Combine with `terraform plan` to safely review differences before applying changes.

