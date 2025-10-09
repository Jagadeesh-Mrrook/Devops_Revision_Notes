# Terraform CLI Workflows: init, plan, apply, destroy

## Detailed Explanation Version

### Concept / What

`terraform init` is the first command in any Terraform project. It sets up the working directory by downloading the required provider plugins (like AWS) and initializing the backend where Terraform stores the state file. Without running init, no other Terraform commands will work.

`terraform plan` is used to preview what Terraform will do before making changes. It shows which resources will be created, modified, or destroyed. It doesn’t make any actual changes.

`terraform apply` actually creates or updates the resources as per the plan. It executes the changes in the real infrastructure.

`terraform destroy` removes all the resources managed by Terraform from the infrastructure. It’s used to clean up environments that are no longer needed.

---

### Why / Purpose / Use Case in Real-World

`init` prepares the Terraform environment by downloading providers and connecting to the backend. `plan` helps review infrastructure changes to prevent mistakes and unexpected deletions. `apply` performs the actual infrastructure creation or update. `destroy` is used to remove resources, mostly in test or temporary environments. In CI/CD pipelines, the typical flow is `init` → `plan -out=tfplan` → review → `apply tfplan` → optional `destroy` for cleanup.

---

### How it Works / Steps / Syntax

**Init**

```bash
terraform init
terraform init -backend-config="bucket=my-terraform-state" -reconfigure
terraform init -upgrade
```

* Downloads provider plugins.
* Initializes backend for state storage.
* `-backend-config` passes backend details dynamically.
* `-reconfigure` resets backend if config changed.
* `-upgrade` updates providers if needed.

**Plan**

```bash
terraform plan -out=tfplan
terraform plan -var-file=dev.tfvars
terraform plan -target=aws_instance.web
terraform plan -refresh=false
```

* Shows changes without applying.
* `-out=tfplan` saves plan for later apply.
* `-var-file` loads environment-specific variables.
* `-target` plans only specific resources.
* `-refresh=false` skips updating state from cloud.

**Apply**

```bash
terraform apply
terraform apply tfplan
terraform apply -auto-approve
terraform apply -refresh=false
```

* Applies infrastructure changes.
* `tfplan` applies a saved plan exactly.
* `-auto-approve` skips confirmation in automation.
* `-refresh=false` skips real-time state check.

**Destroy**

```bash
terraform destroy -auto-approve
terraform destroy -target=aws_instance.web
```

* Deletes Terraform-managed resources.
* `-auto-approve` skips confirmation.
* `-target` deletes only a specific resource.

---

### Common Issues / Errors

* `init`: provider download fails, backend not found, incompatible Terraform version.
* `plan`: missing variable values, state drift, reference to undeclared resource.
* `apply`: API permission errors, rate limits, partial failures.
* `destroy`: `prevent_destroy` set, external dependencies blocking deletion.

---

### Troubleshooting / Fixes

* `init`: check network, provider versions, backend config; use `-reconfigure` or `-upgrade`.
* `plan`: provide missing variable files, refresh state, or import existing resources.
* `apply`: fix IAM permissions, check logs with `TF_LOG=DEBUG`, clean partial state if needed.
* `destroy`: remove lifecycle `prevent_destroy`, check dependencies, use `-target` if necessary.

---

### Best Practices / Tips

* Always run `terraform init` before any other command.
* Run `plan` before `apply` to prevent mistakes.
* Use `plan -out=tfplan` in CI/CD for review and consistency.
* Prefer remote backends with locking for teams (S3 + DynamoDB).
* Avoid `-target` unless emergency.
* Use `-auto-approve` only in automation.
* Pin provider and Terraform versions using `required_providers` and `required_version`.

---

## Quick Revision Version

**Init:** first command, downloads providers, sets backend. Flags: `-backend-config`, `-reconfigure`, `-upgrade`.

**Plan:** previews changes. Flags: `-out=tfplan`, `-var-file`, `-target`, `-refresh=false`.

**Apply:** executes changes. Flags: `tfplan`, `-auto-approve`, `-refresh=false`.

**Destroy:** removes resources. Flags: `-auto-approve`, `-target`.

**Common issues:** init backend errors, plan missing variables, apply permission/API errors, destroy blocked by `prevent_destroy`.

**Best practices:** init first, always plan before apply, use plan files in CI/CD, remote backend with locking, avoid `-target`, use `-auto-approve` only in automation, pin versions.

---
---

# Terraform CLI Workflows: validate, fmt, taint, import

## Detailed Explanation Version

### Concept / What

* **terraform validate** checks your Terraform configuration for syntax errors, missing variables, or internal inconsistencies. It doesn’t interact with the cloud or change infrastructure. It only verifies that the code is correct.

* **terraform fmt** automatically formats Terraform configuration files, fixing indentation and spacing to follow a standard style. It helps keep code consistent and readable.

* **terraform taint** marks a resource as "tainted" in the state file. This means Terraform will destroy and recreate that resource on the next apply. It’s useful when a resource is broken or needs a fresh creation.

* **terraform import** brings an existing resource from the cloud into Terraform’s state so that it can be managed by Terraform moving forward.

---

### Why / Purpose / Use Case in Real-World

* **Validate**: Ensures configurations are correct before applying changes. Often used in CI/CD pipelines to catch errors early.
* **Fmt**: Keeps code clean and consistent across team projects, avoids formatting conflicts.
* **Taint**: Useful when a resource is misbehaving (e.g., stuck EC2 instance). Marks it for destruction and recreation automatically on next apply.
* **Import**: Essential for managing existing resources not yet tracked by Terraform. For example, an existing VPC can be imported so Terraform can manage it.

---

### How it Works / Steps / Syntax

1. **Validate**

```bash
terraform validate
```

* Checks syntax, variable references, and module consistency.

2. **Fmt**

```bash
terraform fmt
terraform fmt -recursive
```

* Formats current folder or recursively through modules.

3. **Taint**

```bash
terraform taint aws_instance.my_ec2
terraform untaint aws_instance.my_ec2
```

* Marks resource for destruction/recreation.
* `untaint` cancels this mark before apply, restoring the resource to its previous state.

4. **Import**

```bash
terraform import aws_instance.my_ec2 i-0abcd1234
```

* First argument: resource address in Terraform.
* Second argument: existing resource ID in the cloud.
* Run `terraform plan` after import to sync state.

---

### Common Issues / Errors

* **Validate**: Missing variables, syntax errors, missing modules.
* **Fmt**: Rare issues; read-only files.
* **Taint**: Resource not in state, already tainted.
* **Import**: Incorrect resource ID, resource type mismatch, resource not existing.

---

### Troubleshooting / Fixes

* **Validate**: Fix syntax errors, provide missing variables, ensure modules exist.
* **Fmt**: Ensure files are writable, use `-recursive` for nested modules.
* **Taint**: Check state using `terraform state list`, import resource if missing.
* **Import**: Verify resource ID and type, ensure Terraform block matches.

---

### Best Practices / Tips

OBOBOB* Run `terraform validate` before applying changes or in CI/CD.
* Always use `terraform fmt` for consistent code; integrate into pre-commit hooks.
OBOBOB* Use `taint` only when needed, always review with `terraform plan` after tainting.
* Use `untaint` to cancel accidental or unnecessary taints before apply.
* After `import`, always run `terraform plan` to confirm no unintended changes.
* Keep resource names consistent to simplify taint/untaint and import operations.

---

## Quick Revision Version

**Validate:** checks configuration syntax and internal consistency. CLI only. Catch errors before apply.

**Fmt:** formats Terraform files for standard style. CLI only. Use `-recursive` for modules.

**Taint:** marks resource for destroy/recreate on next apply. CLI only. Undo with `untaint` before apply. Example: `terraform taint aws_instance.my_ec2` → `terraform untaint aws_instance.my_ec2`.

**Import:** brings existing cloud resource into Terraform state. Example: `terraform import aws_instance.my_ec2 i-0abcd1234`. Always run `terraform plan` after import.

**Common Issues:** validate missing variables or syntax errors, fmt read-only files, taint resource missing or already tainted, import incorrect resource ID/type.

**Best Practices:** validate before apply, fmt consistently, use taint/untaint carefully, review plan after taint/import, maintain consistent resource naming.

---
---

# Terraform CLI Workflows: State Management, Refresh, Targeting, and Corruption Handling

## Detailed Explanation Version

### **Refresh**

* **Concept / What:** Updates Terraform’s state file to match the actual infrastructure. Detects drift between Terraform state and real resources.
* **Why / Purpose:** Real infrastructure may be changed manually or outside Terraform. Refresh ensures Terraform knows the current state to prevent unnecessary changes.
* **How:**

```bash
terraform refresh
```

* Automatically triggered during `plan` and `apply`.
* **Common Issues:** API rate limits, invalid credentials.
* **Fixes:** Retry with correct credentials and sufficient API limits.
* **Best Practices:** Regularly refresh state, especially in multi-team environments.

### **State Manipulation**

* **Concept / What:** Directly modifying, inspecting, or moving resources in the Terraform state file.
* **Commands:**

```bash
terraform state list
terraform state show <resource>
terraform state mv <old> <new>
terraform state rm <resource>
```

* **Use Cases:**

  * Rename resources (`state mv`) without recreating them.
  * Remove a resource from Terraform management (`state rm`).
  * Correct mistakes or reconcile resources.
* **Notes:**

  * `state mv` by default **renames resources in the same state file**. To move between different state files, use `-state` and `-state-out` flags.
  * `state rm` **only removes the resource from state**; resource remains in the cloud.
* **Common Issues:** Misuse can corrupt state; always backup state before manipulation.
* **Fixes:** Verify using `state list` and `plan` after changes.
* **Best Practices:** Backup state, avoid manual edits unless necessary, run `plan` after changes.

### **Resource Targeting**

* **Concept / What:** Apply changes to specific resources using `-target` flag.
* **Why:** Useful when you need to update only one resource without affecting others.
* **How:**

```bash
terraform plan -target=<resource>
terraform apply -target=<resource>
```

* **Common Issues:** Dependency issues if overused; not recommended for routine operations.
* **Best Practices:** Use sparingly, mainly for emergency fixes.

### **State File Corruption & Locking**

* **Concept / What:** State file may get corrupted due to failed operations, manual edits, or concurrency conflicts. Locking prevents concurrent modifications.
* **Handling Corruption:**

  * For **local state**, use `terraform.tfstate.backup` to restore previous state.
  * For **remote state (S3)**, restore **previous versions** via S3 versioning.
* **Commands:**

```bash
terraform force-unlock <LOCK_ID>
```

* **Best Practices:**

  * Always backup state before manipulation.
  * Use remote state with locking in team environments.
  * Verify recovery using `terraform plan`.

### **S3 Versioning & Remote State Backup**

* **Concept / What:** S3 versioning preserves previous state versions automatically.
* **Limitation:** If the current version is corrupted, new changes in that version are lost. Manual or automated backups are required for true disaster recovery.
* **How:**

  * Enable S3 versioning on the state bucket.
  * Recover previous version if current version is corrupted.
  * Maintain external backups (scripts, CI/CD pipelines, or Lambda) for full recovery.
* **Best Practices:** Combine versioning, state locking, and external backups to ensure safety and disaster recovery.

### **Key Interview Points**

* S3 versioning preserves previous versions, but current version corruption leads to data loss.
* Manual or automated backups are necessary for disaster recovery.
* State locking prevents concurrency issues.
* `state mv` and `state rm` manipulate Terraform state safely if backed up.

---
---

# Terraform CLI Workflows: Diff and Plan Output Interpretation

## Detailed Explanation Version

### **Diff and Plan Output Interpretation**

**Concept / What:**

* `terraform plan` compares your Terraform configuration with the current state and shows a preview of changes.
* Output indicates which resources will be added (`+`), modified (`~`), destroyed (`-`), or destroyed/recreated (`-/+`).

**Why / Purpose / Use Case in Real-World:**

* Allows engineers to **preview changes** before applying them.
* Helps prevent accidental deletion or misconfiguration.
* Used in **CI/CD pipelines** to validate planned infrastructure changes.
* Useful for **auditing and troubleshooting**, showing exactly what Terraform will do.

**How it Works / Steps / Syntax:**

1. Run `terraform plan` to see the full diff:

```bash
terraform plan
```

2. Target specific resources if needed:

```bash
terraform plan -target=aws_instance.my_instance
```

3. Save the plan to a file for later apply:

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

4. Interpret plan output:

   * `+` resource will be **created**
   * `-` resource will be **destroyed**
   * `~` resource will be **modified in-place**
   * `-/+` resource will be **destroyed and recreated**
   * Check **attribute-level changes** for modifications.

**Common Issues / Errors:**

* Unexpected destroy due to renaming or lost state tracking.
* Missing dependencies causing plan failure.
* Sensitive data exposed in plan output.

**Troubleshooting / Fixes:**

* Run `terraform refresh` to sync state.
* Verify resource existence with `terraform state list` or `show`.
* Use `terraform state mv` if resources were renamed accidentally.

**Best Practices / Tips:**

* Always **review plan output** before applying, especially in production.
* Use **targeting** and **out file** for controlled apply.
* Combine plan review with version control to track configuration changes.
* Suppress sensitive data from plan output using `sensitive = true` where applicable.

---
---

# Terraform CLI Workflows: Advanced Plan/Apply Options & Workflow Best Practices

## Detailed Explanation Version

### **Advanced Plan/Apply Options & Workflow Best Practices**

**Concept / What:**

* Terraform provides additional options with `plan` and `apply` commands to control execution behavior, automate workflows, and handle special cases.
* Examples include `-auto-approve`, `-refresh=false`, `-lock=false`, and `-input=false`.

**Why / Purpose / Use Case in Real-World:**

* **Automation:** For CI/CD pipelines where Terraform must run non-interactively.
* **Performance:** Skipping unnecessary refresh or locks can speed up operations.
* **Safe Execution:** Proper flags prevent unexpected prompts or blocking due to locks.
* **Drift Detection:** Ensures any manual changes in cloud resources are detected before applying.

**How it Works / Steps / Syntax:**

1. **Auto-approve**

```bash
terraform apply -auto-approve
```

* Skips interactive approval; useful in automated pipelines.

2. **Skip State Refresh**

```bash
terraform plan -refresh=false
```

* Disables automatic refresh; speeds up planning but may miss drifts.

3. **Disable State Locking**

```bash
terraform apply -lock=false
```

* Prevents remote state locking; only use in isolated environments.

4. **Disable Interactive Prompts**

```bash
terraform apply -input=false
```

* Terraform won't prompt for input variables; values must be supplied via `.tfvars` or CLI.

5. **Drift and State Handling:**

* Terraform always **compares configuration files vs state** (on-disk or in-memory).
* **`plan` command:** refreshes state in memory to detect changes in the cloud and compares **configuration vs in-memory state**; state file on disk is not updated.
* **`apply` command:** refreshes state by default, compares **configuration vs refreshed state**, applies changes, and updates the **state file**.
* **`refresh` command:** compares **state file vs actual cloud resources** and updates the state file without changing cloud resources.
* Manual changes in the cloud are captured via refresh. To keep changes, update the **configuration files**; otherwise Terraform will reconcile cloud resources to match configuration.

**Common Issues / Errors:**

* Unintended changes if using `-auto-approve` without reviewing plan.
* Missing drift detection if `-refresh=false` is used.
* Risk of state corruption if locks are disabled in team environments.

**Troubleshooting / Fixes:**

* Always run `terraform plan` first, even when using `-auto-approve`.
* Use `terraform refresh` to sync state if needed.
* Avoid disabling locks in shared environments.
* Update configuration files to reflect desired changes if manual cloud modifications should be retained.

**Best Practices / Tips:**

* Review plan output carefully before applying.
* Use `.tfvars` to avoid interactive prompts.
* Combine plan review with version control to track configuration changes.
* Target and out-file options allow controlled and safe application.
* Understand drift and refresh workflow for correct state management.
i
---
---
---
