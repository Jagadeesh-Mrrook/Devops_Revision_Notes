# Terraform CLI Workflows: Quick Revision Version

**Init:** first command, downloads providers, sets backend. Common flags: `-backend-config`, `-reconfigure`, `-upgrade`.

**Plan:** previews what will change before applying. Flags: `-out=tfplan` (save plan), `-var-file` (load env variables), `-target` (specific resource), `-refresh=false` (skip state refresh).

**Apply:** executes the changes. Flags: `tfplan` (apply saved plan), `-auto-approve` (skip confirmation), `-refresh=false` (skip state refresh).

**Destroy:** removes Terraform-managed resources. Flags: `-auto-approve` (skip confirmation), `-target` (destroy specific resource).

**Common Issues:** init backend/provider errors, plan missing variables, apply permission/API errors, destroy blocked by `prevent_destroy` or dependencies.

**Troubleshooting:** init → `-reconfigure`, `-upgrade`; plan → provide missing vars, refresh/import state; apply → fix permissions, check logs; destroy → remove `prevent_destroy`, use `-target`.

**Best Practices:** always init first, plan before apply, use plan files in CI/CD, remote backend with locking, avoid `-target` unless emergency, `-auto-approve` only in automation, pin versions.

---
---

# Terraform CLI Workflows: Quick Revision Version (Validate, Fmt, Taint, Import)

**Validate:** checks configuration syntax and consistency. CLI only. Catch errors before apply.

**Fmt:** formats Terraform files for standard style. CLI only. Use `-recursive` for modules.

**Taint:** marks resource for destroy/recreate on next apply. CLI only. Undo with `untaint` before apply. Example: `terraform taint aws_instance.my_ec2` → `terraform untaint aws_instance.my_ec2`.

**Import:** brings existing cloud resource into Terraform state. Example: `terraform import aws_instance.my_ec2 i-0abcd1234`. Always run `terraform plan` after import.

**Common Issues:** validate missing variables/syntax, fmt read-only files, taint missing/already tainted, import incorrect resource ID/type.

**Best Practices:** validate before apply, fmt consistently, use taint/untaint carefully, review plan after taint/import, maintain consistent resource naming.

---
---

# Terraform CLI Workflows: State Management - Quick Revision Version

* **Refresh:** Updates state to match real infra; run manually or auto during plan/apply.
* **State Manipulation:** `state list`, `show`, `mv`, `rm`; `mv` renames, `rm` removes from state only.
* **Resource Targeting:** `-target=<resource>`; use sparingly, for emergency fixes.
* **Corruption & Locking:** Restore from `terraform.tfstate.backup` (local) or previous S3 version; use `terraform force-unlock` for stuck locks.
* **S3 Versioning:** Stores previous versions; current version loss requires backup recovery.
* **Disaster Recovery:** Always maintain external backups; versioning + locking is not enough alone.
* **Best Practices:** Backup state, validate, use plan, avoid manual edits, use locking for team environments.

---
---

# Terraform CLI Workflows: Diff and Plan Output Interpretation - Quick Revision Version

* **Purpose:** `terraform plan` shows a preview of changes before applying them. Helps prevent accidental deletion or misconfiguration of resources.

* **Symbols in Plan Output:**

  * `+` → resource will be created
  * `~` → resource will be modified in-place
  * `-` → resource will be destroyed
  * `-/+` → resource will be destroyed and recreated

* **Usage:**

  * Run `terraform plan` to see all changes.
  * Use `-target=<resource>` to focus on specific resources.
  * Save the plan to a file using `-out=tfplan` and apply later with `terraform apply tfplan`.

* **Common Issues:**

  * Unexpected destroy due to renaming or state mismatch.
  * Plan may fail if dependencies are missing.
  * Sensitive data may appear in the output.

* **Tips:**

  * Always review plan output carefully before applying.
  * Use `terraform refresh` to sync state if needed.
  * Combine with version control to track configuration changes.
  * Target and out-file options allow controlled and safe application of resources.

---
---

# Terraform CLI Workflows: Advanced Plan/Apply Options & Workflow Best Practices

## Quick Revision Version

* **Purpose:** Automate Terraform in pipelines, control execution, detect drift, manage state.

* **Common Flags:**

  * `-auto-approve` → skip manual approval.
  * `-refresh=false` → skip automatic refresh (may miss drifts).
  * `-lock=false` → disable remote state locking (use only in isolated/test environments).
  * `-input=false` → disable prompts, provide variables via `.tfvars` or CLI.

* **Drift & State:**

  * Terraform compares **configuration vs state** (memory or on-disk).
  * `plan` → refreshes in-memory state, shows drift, does not update state file.
  * `apply` → refreshes state, applies changes, updates state file.
  * `refresh` → updates state file from cloud without applying changes.
  * Manual changes in cloud → captured in refresh; update config to retain them.

* **Tips:**

  * Always review plan before applying.
  * Use `.tfvars` to avoid prompts.
  * Maintain version control to track configuration changes.
  * Enable locks in team environments to avoid state corruption.

---
---
---
