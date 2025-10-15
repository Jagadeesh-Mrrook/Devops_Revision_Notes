## Quick Revision Notes: Terraform Troubleshooting & Common Errors (Introduction)

### 🧩 Concept / What

Terraform troubleshooting focuses on identifying and fixing errors during various stages like `init`, `plan`, `apply`, or `destroy`.

---

### 💡 Why / Purpose

* Ensures stable infrastructure deployments.
* Helps detect and resolve issues faster.
* Prevents downtime and state corruption.

---

### ⚙️ How It Works

Errors commonly occur in:

* **Init** → Provider or backend issues.
* **Plan/Apply** → Syntax, dependency, or drift problems.
* **State Management** → Conflicts or lock errors.

---

### 🔍 Common Error Categories

1. Provider authentication failures.
2. State conflicts or lock errors.
3. Plan vs. Apply discrepancies.
4. Resource dependency cycles.
5. Infrastructure drift.

---

### 🧰 Common Issues & Fixes

| Issue Type   | Troubleshooting Step                              |
| ------------ | ------------------------------------------------- |
| Auth Failure | Check credentials, re-run `terraform init`.       |
| Lock Error   | Release state lock (DynamoDB/S3).                 |
| Plan Drift   | Run `terraform refresh` or `plan -refresh-only`.  |
| Cycle Error  | Break dependency using variables or data sources. |
| Drift        | Avoid manual infra changes; re-sync state.        |

---

### 🧠 Best Practices

* Always use **remote state with locking** (e.g., S3 + DynamoDB).
* Run Terraform sequentially (avoid parallel executions).
* Don’t manually modify resources managed by Terraform.
* Keep provider plugins and credentials updated.
* Refresh state regularly before major changes.

---
---

# Quick Revision Notes: Terraform Provider Authentication Failures

## Concept / What

Terraform fails to authenticate with the cloud provider due to invalid, missing, expired, or misconfigured credentials.

---

## Why / Purpose

* Ensures secure access to cloud resources
* Critical for CI/CD automation reliability
* Prevents failed deployments and drift

---

## How / Steps / Syntax

* Credentials via environment variables, shared files, provider block, or IAM roles
* Terraform reads credentials → tries API calls → fails if invalid

---

## Common Errors

* `Error: error configuring Terraform AWS Provider`
* `InvalidClientTokenId`
* `ExpiredToken`
* `AccessDenied`

---

## Real-Time Scenario

* Local laptop: only for isolated dev accounts
* CI/CD Jenkins agent: preferred for dev/QA/prod
* Manual runs: log into Jenkins agent, not laptop

---

## Troubleshooting / Fixes

* Validate credentials (`aws sts get-caller-identity`)
* Export environment variables correctly
* Re-run `terraform init`
* Use IAM roles or Jenkins credentials store
* Rotate expired keys

---

## Best Practices / Tips

* No hardcoding credentials
* Run Terraform on CI/CD agents or controlled runners
* Maintain audit logs
* Local runs only on isolated dev accounts

---
---

# Quick Revision Notes: Terraform State Conflicts & Lock Errors

## Concept / What

State conflicts happen when multiple Terraform processes try to read/write the same remote state at the same time. Terraform uses locking to prevent this.

## Why

* Prevents state corruption and resource drift
* Ensures safe, consistent infrastructure updates
* Critical in multi-user and CI/CD environments

## How

* Terraform acquires lock on remote state during `apply`
* Lock errors occur if another process holds the lock
* Example command to manually release: `terraform force-unlock <LOCK_ID>`
* Backend example: AWS S3 + DynamoDB with locking table

## Common Errors

* `Error acquiring the state lock`
* `Lock already held by ...`
* Network/connectivity issues
* Concurrent applies on same state

## Troubleshooting / Fixes

* Wait for other Terraform process to finish
* Use `terraform force-unlock <LOCK_ID>` safely
* Check backend config and network
* Avoid concurrent pipeline executions

## Best Practices / Tips

* Always use remote backend with locking
* Serialize Terraform runs
* Monitor lock tables for stale locks
* CI/CD pipelines preferred for all environments
* Local backend only for isolated dev experiments

---
---

# Quick Revision Notes: Terraform Plan/Apply Discrepancies

## Concept / What

When Terraform plan output differs from what actually happens during `terraform apply`. Execution plan shows certain changes, but applied changes differ.

## Why

* Ensures execution matches expectations
* Detects unintended changes early
* Prevents drift in multi-team and CI/CD setups

## How

* Run `terraform plan` to see changes
* Run `terraform apply` to apply changes
* Use `-out=tfplan` to lock plan for consistent apply
* Discrepancies occur if state is out of sync, config changes after plan, or provider behavior differs

## Common Errors

* `to be changed` shows in plan but no change applied
* Unexpected destroy/create operations
* Drift from manual changes
* Provider-specific plan/apply differences

## Troubleshooting / Fixes

* Run plan immediately before apply
* Use `terraform refresh` to sync state
* Apply exact stored plan (`-out=tfplan`)
* Avoid manual changes outside Terraform
* Check provider behavior

## Best Practices / Tips

* Always review plan before apply
* Use `terraform apply tfplan` for consistency
* Automate plan checks in pipelines
* Regularly refresh state to prevent drift
* Lock state to avoid conflicts

---
---

# Quick Revision Notes: Terraform Resource Dependencies & Cycle Errors

## Concept / What

* Resource dependencies define creation, update, and destroy order.
* Cycle errors occur when there is a circular dependency.

## Why

* Ensures correct resource creation and deletion order.
* Prevents runtime errors and deployment failures.
* Critical in complex infra like VPC → Subnet → EC2 → Security Group.

## How

* Terraform builds dependency graph automatically from references.
* `depends_on` used for explicit dependencies.
* Both create and destroy operations respect the graph.
* Cycle errors: circular `depends_on` references cannot be resolved.

## Real-World Examples

* AWS infra: VPC → Subnet → Internet Gateway → Route Table → Security Group → EC2 → Elastic IP
* Lambda + S3 circular dependency can cause cycle error.

## Common Issues / Errors

* `Error: Cycle detected`
* Misconfigured `depends_on`
* Missing implicit dependencies
* Large dependency graphs slow plan/apply

## Troubleshooting / Fixes

* Visualize dependency graph: `terraform graph | dot -Tpng > graph.png`
* Remove unnecessary `depends_on`
* Modularize complex resources
* Ensure implicit dependencies are correct

## Best Practices / Tips

* Prefer implicit dependencies
* Use `depends_on` sparingly
* Avoid circular references
* Terraform handles destroy order automatically
* Use real-world AWS examples in interviews

---
---

# Quick Revision Notes: Handling Drift Between Code & Real Infrastructure

## Concept / What

* Drift: actual infrastructure differs from Terraform code and state.
* Causes: manual edits, provider changes, partial applies.

## Why

* Keeps Terraform-managed infrastructure consistent and predictable.
* Prevents failures from manual or external changes.
* Critical in production/shared environments.

## How

* Detect drift: `terraform plan` shows differences.
* Refresh state: `terraform refresh` updates state without applying.
* Correct drift: `terraform apply` to revert changes or update code.
* Prevent drift: use CI/CD, limit manual changes, enable state locking.

## Common Issues / Errors

* Drift detected on plan due to manual/provider changes.
* Partial apply leaves resources out-of-sync.
* Missing/outdated state file.

## Troubleshooting / Fixes

* Check `terraform plan` for unexpected changes.
* Use `terraform refresh` to sync state.
* Apply corrective changes with `terraform apply`.
* Lock state to prevent concurrent manual edits.
* Enforce policies to reduce manual drift.

## Best Practices / Tips

* Avoid manual changes; use Terraform.
* Regularly run `terraform plan`.
* Use version control for configurations.
* Use remote backends with state locking.
* Update code if manual/provider changes are intentional.

---
---
---


