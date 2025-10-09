# Quick Revision Notes – CI/CD & Automation with Terraform

### Concept / What:

CI/CD automates Terraform workflows like `init`, `plan`, and `apply` using Jenkins, GitHub Actions, or GitLab CI.

### Why / Purpose:

* Speed and consistency in deployments.
* Reduces manual intervention and human error.
* Enforces reviews, compliance, and approvals.
* Enables collaboration across teams.

### How / Steps:

1. Code commit triggers pipeline.
2. Stages: `terraform fmt`, `terraform validate`, `terraform plan -out=tfplan`, `terraform apply tfplan`.
3. Use remote backend (S3, DynamoDB, Terraform Cloud) for shared state.
4. Add approval step for production.

### Common Issues:

* **State Lock Error:** Another run in progress or leftover lock.
* **Backend Error:** Misconfiguration or missing credentials.
* **Permission Error:** Insufficient IAM roles for Jenkins or CI/CD service account.
* **Version Error:** Terraform version mismatch on pipeline agent.

### Terraform Locking:

Locking prevents multiple Terraform runs on the same state file.

**Causes:** concurrent runs, crashed runs, Ctrl+C interruptions, backend or network issues.
**Fix:** verify no active runs → use `terraform force-unlock <lock-id>` carefully.

### Troubleshooting:

* Check backend connectivity and IAM permissions.
* Ensure consistent Terraform version.
* Confirm only one apply job runs at a time.

### Best Practices:

* Always run `fmt` and `validate` before `plan`.
* Store state remotely.
* Save plan files as artifacts.
* Use approval gates for production.
* Maintain consistent Terraform version across agents.

---
---

# Quick Revision Notes – Terraform Integration in Jenkins Pipelines

### Concept / What:

Automates Terraform commands (`init`, `validate`, `plan`, `apply`) in Jenkins pipelines for repeatable infrastructure provisioning.

### Why:

* Automates deployments
* Ensures consistency across environments
* Provides audit trails and approvals
* Facilitates DevOps collaboration

### How / Steps:

1. Terraform code stored in Git.
2. Jenkins pipeline runs `init`, `validate`, `plan`, `apply`.
3. Trigger via Git push/merge or manually.
4. Use remote backend for Terraform state.
5. Archive plan files and logs.

### Common Issues:

* Backend initialization failure
* Disk or workspace issues
* State lock conflicts (concurrent runs)
* Missing permissions / IAM roles
* Terraform binary not found
* Approval step stuck

### Troubleshooting:

* Verify backend and credentials
* Force unlock state if safe
* Assign correct IAM roles
* Install Terraform or use Docker agent
* Clean workspace if disk issues

### Best Practices:

* Pin Terraform version
* Use environment-specific workspaces
* Store secrets in Jenkins credentials
* Plan → Approval → Apply
* Enable notifications & archive logs/plans
* Always use remote backend
* Prefer Docker agents for clean builds

---
---

# 🧩 Quick Revision Notes – Terraform Cloud & Remote Operations

### Concept / What:

Terraform Cloud is a SaaS platform for remote state storage, Terraform execution, collaboration, and policy enforcement.

### Why / Purpose:

* Centralized state management
* Team collaboration
* Audit trails and governance
* Remote execution of Terraform commands
* Optional policy enforcement (Sentinel)

### How / Steps:

1. Configure backend `remote` in Terraform.
2. Initialize Terraform with `terraform init`.
3. Run `terraform plan` and `terraform apply` remotely.
4. Use workspaces for environment separation (dev, staging, prod).
5. Collaborate via plan review and approval.

### Common Issues / Errors:

* Authentication errors (invalid API token)
* State lock due to concurrent runs
* Backend configuration mismatch
* Insufficient permissions
* Workspace not found

### Troubleshooting / Fixes:

* Verify Terraform Cloud API token
* Wait for or cancel running operations; force unlock if safe
* Reconfigure backend (`terraform init -reconfigure`)
* Update workspace permissions
* Create missing workspace via UI/CLI

### Best Practices / Tips:

* Always use a remote backend
* Separate workspaces per environment
* Enable VCS integration for auto-triggered runs
* Store secrets securely via Terraform Cloud variables
* Follow Plan → Review → Apply workflow
* Pin Terraform version
* Enable notifications and audit logs
* Choose free vs paid tier based on team size and governan


---
---

# 🧩 Quick Revision Notes – Multi-Environment Terraform Deployments

## Concept / What:

Deploy Terraform across multiple environments (dev, QA, UAT, prod) and AWS accounts automatically using modules, environment-specific variables, and remote state.

---

## Key Points:

* **Modules:** Reusable components (vpc, ec2, autoscaling)
* **Environment Directories:**

  * Non-prod: dev/, QA/, UAT/
  * Prod: prod/
* **main.tf:** Calls modules, references environment-specific variables
* **variables.tf:** Defines variables for each environment
* **.tfvars:** Environment-specific values (e.g., instance_type, region, vpc_id)
* **Backend (Remote State):**

  * Non-prod: single S3 bucket, dynamic keys (`dev/terraform.tfstate`, `qa/terraform.tfstate`, `uat/terraform.tfstate`)
  * Prod: separate bucket for isolation
  * DynamoDB table for locking
* **Terraform Behavior:** Creates separate state files automatically, locks per environment

---

## CI/CD Integration:

```groovy
stage('Deploy to Environment') {
  steps {
    dir("${ENVIRONMENT}") {
      sh "terraform init -backend-config=backend.hcl"
      sh "terraform plan -var-file=${ENVIRONMENT}.tfvars -out=tfplan"
      sh "terraform apply -auto-approve tfplan"
    }
  }
}
```

* ENVIRONMENT = dev, QA, UAT, prod
* Use approval gates for prod

---

## Best Practices:

* Workspaces for non-prod environments (optional)
* Separate backend for prod
* Parameterize environment values using `.tfvars`
* Reuse modules to avoid duplication
* Automate with CI/CD pipelines
* Maintain plan artifacts and audit logs
* Use DynamoDB for state locking
* Always isolate prod environment

---

## Common Issues:

* Wrong workspace or AWS account
* Missing `.tfvars` values
* State conflicts from concurrent runs
* Permission errors
* Backend misconfiguration

---

## Troubleshooting:

* `terraform workspace show` to check active workspace
* Verify AWS credentials: `aws sts get-caller-identity`
* Reinitialize backend if needed: `terraform init -reconfigure`
* Ensure correct `.tfvars` file
* Force unlock DynamoDB lock if safe

---
---

# 🧩 Quick Revision Notes – terraform fmt & terraform validate in CI/CD

## Concept / What:

* `terraform fmt`: Formats Terraform files to standard style.
* `terraform validate`: Checks Terraform code for syntax and logical errors.
* Used in CI/CD to ensure code quality before plan/apply.

---

## Key Points:

* **Formatting**: `terraform fmt -recursive`, `terraform fmt -check` for CI/CD
* **Validation**: `terraform validate` after `terraform init` if using modules or backend
* **CI/CD Example**:

```groovy
stage('Lint & Validate Terraform') {
  steps {
    dir("${ENVIRONMENT}") {
      sh 'terraform fmt -check -recursive'
      sh 'terraform init -backend-config=backend.hcl'
      sh 'terraform validate'
    }
  }
}
```

* Fails pipeline if formatting incorrect or validation errors exist
* Always run **before plan/apply**

---

## Best Practices:

* Use pre-commit hooks to auto-format code locally
* Validate all environments in CI/CD
* Combine with plan artifacts for review
* Ensure backend is initialized before validate if using modules

---

## Common Issues & Fixes:

* Formatting fails → run `terraform fmt -recursive`
* Missing variables → provide `.tfvars` or defaults
* Module errors → ensure backend init and module sources

---
---

# 🧩 Quick Revision Notes – Terraform Plan as Code Review Artifact

## Concept / What:

* `terraform plan` generates execution plan showing changes.
* Storing plan files in CI/CD pipeline allows **review before apply**.

---

## Key Points:

* Generate plan with `terraform plan -out=tfplan`
* **Archive plan file** in CI/CD pipeline for review and audit
* **Use separate Jenkins working directory per environment**:

```groovy
dir("non-prod/${ENVIRONMENT}") {
    sh 'terraform init -backend-config=backend.hcl'
    sh 'terraform plan -var-file=${ENVIRONMENT}.tfvars -out=tfplan'
}
```

* Manual approval before apply:

```groovy
input message: "Approve plan for ${ENVIRONMENT}?"
sh 'terraform apply tfplan'
```

* Ensures environment isolation, safe changes, and audit trail

---

## Best Practices:

* Generate plan for every PR/merge
* Store plan files as **pipeline artifacts**
* Manual approval for production
* Short-lived plan files; delete after apply
* Combine with `terraform fmt` and `terraform validate`
* Use **dir() block** to isolate environment in Jenkins

---

## Common Issues & Fixes:

* Plan file missing → ensure `-out=tfplan` and correct path
* Apply fails → check `.tfvars` and backend consistency
* Permission errors → pipeline agent IAM permissions
* Conflicts → keep plan files per environment

---
---

# 🧩 Quick Revision Notes – Terraform Workspace Selection in Pipelines

## Concept / What:

* Terraform workspace allows multiple state files for a single configuration.
* Useful for separating environments (dev, QA, UAT, prod) without duplicating code.
* Each workspace maintains its own isolated state.

---

## Key Points:

* List workspaces: `terraform workspace list`
* Create workspace: `terraform workspace new dev`
* Select workspace: `terraform workspace select dev`
* Pipeline example:

```groovy
dir('terraform') {
  sh 'terraform init -backend-config=backend.hcl'
  sh "terraform workspace select ${ENVIRONMENT} || terraform workspace new ${ENVIRONMENT}"
  sh 'terraform plan -var-file=${ENVIRONMENT}.tfvars -out=tfplan'
  archiveArtifacts artifacts: 'tfplan', fingerprint: true
}
```

* `dir()` isolates working directory per environment

---

## State File Management:

* Each workspace → separate state file automatically
* Same S3 bucket, different keys:

```
key = "workspace/${terraform.workspace}/terraform.tfstate"
```

* DynamoDB table handles locking per workspace
* Optional: separate S3 bucket for prod for full isolation

---

## Best Practices:

* Ideal for non-prod environments with similar infrastructure
* Separate backend for prod environments
* Always select workspace before plan/apply in pipelines
* Combine with `.tfvars` for environment-specific variables
* Use `dir()` in Jenkins for working directory isolation

---

## Common Issues & Fixes:

* Workspace not found → create workspace
* State conflicts → avoid concurrent runs in same workspace
* Apply fails → check workspace selection and `.tfvars`

---
---
---


