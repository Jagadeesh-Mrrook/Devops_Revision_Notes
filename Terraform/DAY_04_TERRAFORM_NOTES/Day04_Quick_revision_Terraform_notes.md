## Terraform State Management Quick Revision Notes (Expanded)

#### 1. Terraform State

* Snapshot of infrastructure; maps config to real resources.
* Local: `terraform.tfstate` + `.backup`.
* Remote: S3/GCS/Azure; enable versioning for backups.
* `terraform plan` shows planned changes; `apply` updates resources.
* Acts as **source of truth** for Terraform to track infrastructure.

#### 2. Backend Configuration

* Defines where state is stored (local/remote).
* S3 example: bucket, key, region, dynamodb\_table, encrypt.
* `terraform init` initializes backend.
* Remote backend supports **team collaboration**.
* Enables **locking, versioning, and high availability**.

#### 3. State Locking & Versioning

* Prevent simultaneous writes; ensures atomic operations.
* DynamoDB table with S3 backend locks the state during `apply`.
* `terraform force-unlock` can manually release lock if needed.
* Versioning ensures **rollback if state file is corrupted**.
* Critical for **multi-user team setups**.

#### 4. Workspaces

* Manage multiple environments using the same config file.
* Commands: `workspace list`, `new <name>`, `select <name>`, `show`.
* Local backend: separate state + backup; Remote backend: workspace-specific keys + versioning.
* Helps **isolate dev, staging, prod environments** safely.
* Always check current workspace before applying changes to avoid affecting wrong env.

#### 5. Importing Existing Infrastructure

* `terraform import` brings existing resources into Terraform state.
* Requires **resource name + ID**; updates **state only**.
* Terraformer can **automate generation of `.tf` and state files** for multiple resources.
* Verify configuration matches existing infrastructure.
* Use `terraform plan` after import to ensure no unwanted changes.
* Best: import **one resource at a time**, document all imported resources, combine with remote backend for team collaboration.

