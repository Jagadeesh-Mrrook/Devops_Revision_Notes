# Detailed Explanation Version

## Concept / What:

**CI/CD & Automation** refers to automating the Continuous Integration (merging/testing code) and Continuous Deployment/Delivery (releasing to environments) processes. In Terraform, it means automating commands like `terraform init`, `plan`, and `apply` through CI/CD tools such as Jenkins, GitHub Actions, or GitLab CI.

---

## Why / Purpose / Use Case in Real-World:

* **Consistency:** Ensures every environment is deployed identically.
* **Speed:** Eliminates manual Terraform runs.
* **Error Reduction:** Automated testing/validation reduces mistakes.
* **Collaboration:** Multiple engineers can safely manage infra via Git.
* **Compliance:** Pipelines enforce reviews and approvals.

**Example:** A Git commit triggers Jenkins → runs `terraform fmt`, `terraform validate`, `terraform plan`, waits for approval → executes `terraform apply`.

---

## How it Works / Steps / Syntax:

Typical CI/CD flow for Terraform:

1. **Developer commits code** → pipeline triggers.
2. **Stages in pipeline:**

   * `terraform fmt -check` → ensures formatting.
   * `terraform validate` → validates syntax.
   * `terraform plan -out=tfplan` → generates execution plan.
   * Manual approval (optional).
   * `terraform apply tfplan` → applies infrastructure.
3. **Remote backend:** S3 + DynamoDB or Terraform Cloud used for shared state.
4. **Notifications:** Send build status (Slack, email, etc.).

**Example Jenkinsfile:**

```groovy
pipeline {
  agent any
  stages {
    stage('Init') { steps { sh 'terraform init' } }
    stage('Validate') { steps { sh 'terraform validate' } }
    stage('Plan') { steps { sh 'terraform plan -out=tfplan' } }
    stage('Apply') {
      when { branch 'main' }
      steps { sh 'terraform apply -auto-approve tfplan' }
    }
  }
}
```

---

## Common Issues / Errors:

| Error                           | Cause                                                                     |
| ------------------------------- | ------------------------------------------------------------------------- |
| `Error locking state`           | Another Terraform process or leftover lock preventing state modification. |
| `Backend initialization failed` | Incorrect backend configuration or missing credentials.                   |
| `Insufficient permissions`      | Jenkins/pipeline user lacks IAM permissions.                              |
| `Invalid Terraform version`     | Agent Terraform version differs from required version.                    |

---

## Troubleshooting / Fixes:

* **State Lock Issues:** Check for active runs; if safe, unlock manually:

  ```bash
  terraform force-unlock <lock-id>
  ```
* **Backend Errors:** Verify backend configuration or credentials.
* **Permission Issues:** Ensure CI/CD service account has proper IAM roles.
* **Version Mismatch:** Define required version in code:

  ```hcl
  terraform {
    required_version = ">= 1.5.0"
  }
  ```

---

## Terraform Locking (In-Depth Explanation):

Terraform creates a **state lock** during operations like `plan`, `apply`, or `destroy` to avoid concurrent modifications to the same state file.

### Causes of Locking:

1. Another `terraform apply` or pipeline running concurrently.
2. Previous run crashed or timed out.
3. Manual interruption (Ctrl + C) left lock file.
4. Backend provider issue (S3, DynamoDB, or Terraform Cloud outage).
5. Misconfigured IAM permissions (e.g., no `dynamodb:DeleteItem`).
6. Network issues causing Terraform to think the lock still exists.

### How to Resolve:

* Identify who/what created the lock from the error log.
* Confirm no active Terraform run.
* Use `terraform force-unlock <lock-id>` cautiously.
* Validate DynamoDB lock table, IAM roles, and backend configuration.

---

## Best Practices / Tips:

* Always run `terraform fmt` and `terraform validate` before `plan`/`apply`.
* Store Terraform state in remote backend (not local).
* Include approval gates before production applies.
* Use environment variables and workspaces.
* Save `terraform plan` outputs as review artifacts.
* Manage credentials securely via secret managers.
* Keep Terraform versions consistent across agents.

---

# Quick Revision Version

### Concept / What:

CI/CD automates Terraform workflows to eliminate manual execution and enforce consistency.

### Why:

* Speed up infra deployment
* Reduce human error
* Enable approvals and compliance
* Support collaboration

### How:

1. Code commit → triggers pipeline.
2. Pipeline runs `fmt`, `validate`, `plan`, and `apply`.
3. Remote backend manages shared state.
4. Optionally add approval step.

### Common Errors:

* State lock conflict
* Backend misconfiguration
* Permission issues
* Version mismatch

### Terraform Locking:

Happens due to concurrent runs, crashes, or backend/network issues. Use `terraform force-unlock` only after confirming safety.

### Troubleshooting:

* Verify active runs.
* Check backend health and IAM roles.
* Define Terraform version in code.

### Best Practices:

* Always validate before apply.
* Use remote backend.
* Store plans as artifacts.
* Implement approvals.
* Maintain consistent versions.

---
---

# Detailed Explanation Version

## Concept / What:

**Terraform Integration in Jenkins Pipelines** refers to automating Terraform commands (`init`, `validate`, `plan`, `apply`) using Jenkins pipelines. Jenkins acts as an orchestrator to execute Terraform in a repeatable, controlled, and automated way whenever infrastructure code is pushed to a Git repository.

---

## Why / Purpose / Use Case in Real-World:

* Automates infrastructure deployment and removes manual steps.
* Brings Terraform workflows under CI/CD governance with approvals and audits.
* Ensures consistency across environments (dev, staging, prod).
* Provides visibility and audit trail through Jenkins logs.
* Enables collaboration among DevOps teams.

**Example:** Developers push Terraform code to GitHub → Jenkins triggers pipeline → runs `terraform fmt`, `validate`, generates `plan` → optional approval → applies infrastructure updates.

---

## How it Works / Steps / Syntax:

1. **Install Terraform on Jenkins agent** (binary installation or Docker agent).
2. **Store Terraform code in Git repository**.
3. **Configure Jenkins credentials** for cloud provider access (AWS keys, GCP service accounts, etc.).
4. **Create Jenkinsfile** with stages:

   ```groovy
   pipeline {
     agent any
     environment {
       AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
       AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
     }
     stages {
       stage('Init') { steps { sh 'terraform init -backend-config=backend.hcl' } }
       stage('Validate') { steps { sh 'terraform validate' } }
       stage('Plan') { steps { sh 'terraform plan -out=tfplan' } }
       stage('Apply') {
         when { branch 'main' }
         steps { sh 'terraform apply -auto-approve tfplan' }
       }
     }
   }
   ```
5. **Trigger pipeline** via Git push/merge or manually.
6. **Store Terraform state remotely** (S3, Terraform Cloud, etc.).
7. **Archive plan files and logs** for audit and rollback.

---

## Common Issues / Errors:

| Issue                           | Cause                                                |
| ------------------------------- | ---------------------------------------------------- |
| `Backend initialization failed` | Missing or misconfigured backend credentials.        |
| `No space left on device`       | Jenkins workspace full, provider plugins cached.     |
| `Error locking state`           | Parallel builds or leftover lock files.              |
| `Insufficient IAM permissions`  | Jenkins user lacks rights to manage resources.       |
| `terraform not found`           | Terraform binary missing or PATH incorrect.          |
| Pipeline stuck at approval      | Manual gate not cleared or misconfigured input step. |

---

## Troubleshooting / Fixes:

* **Backend errors:** verify credentials and backend config.
* **State lock:** ensure no concurrent run → `terraform force-unlock <lock-id>` if safe.
* **Permission issues:** assign correct IAM roles to Jenkins service account.
* **Terraform missing:** install Terraform or use Docker agent with Terraform pre-installed.
* **Disk full / plugin issues:** clean workspace (`rm -rf .terraform`).
* **Stuck approvals:** check `input` block or timeout logic.

---

## Best Practices / Tips:

* Pin Terraform version using `required_version`.
* Use environment-based workspaces or folders (dev, stage, prod).
* Never store secrets directly in code; use Jenkins credentials binding.
* Always run `plan` first → require approval → then `apply`.
* Enable notifications (Slack, email) for pipeline status.
* Archive `tfplan` and logs for auditing.
* Keep backend remote, never local.
* Use Docker agents for clean and repeatable builds.

---

# Quick Revision Version

### Concept / What:

Automating Terraform commands in Jenkins pipelines for controlled, repeatable infrastructure provisioning.

### Why:

* Automates deployment
* Ensures environment consistency
* Provides audit trails and approvals
* Enables DevOps team collaboration

### How:

1. Terraform code in Git.
2. Jenkins pipeline: `init`, `validate`, `plan`, `apply`.
3. Triggered by Git push or manually.
4. Remote backend for state.
5. Archive plans/logs.

### Common Issues:

* Backend init failure
* Disk/space errors
* State locking due to concurrent runs
* Permission issues
* Terraform binary missing
* Approval step stuck

### Troubleshooting:

* Verify backend config & credentials
* Force unlock state if safe
* Assign correct IAM roles
* Install Terraform / use Docker agent
* Clean workspace if disk issues

### Best Practices:

* Pin Terraform version
* Use environment-specific workspaces
* Store secrets in Jenkins credentials
* Plan → approval → apply
* Enable notifications & archive plans/logs
* Remote backend mandatory
* Use Docker agents for repeatability

---
---

# 🧩 Concept: Terraform Cloud & Remote Operations

## Concept / What:

**Terraform Cloud & Remote Operations** refers to using HashiCorp's SaaS platform (Terraform Cloud) to store Terraform state remotely, run Terraform commands in the cloud, enforce collaboration, and manage policy governance.

---

## Why / Purpose / Use Case in Real-World:

* **Centralized State Management:** avoids local state files and reduces corruption risk.
* **Collaboration:** multiple team members can safely queue runs, review plans, and approve changes.
* **Audit & Governance:** Terraform Cloud logs all runs, approvals, and changes.
* **Remote Execution:** Terraform commands execute in Terraform Cloud, not locally.
* **Policy Enforcement:** Optional Sentinel policies ensure compliance.

**Example:** A developer pushes Terraform code → Terraform Cloud runs `plan` → team reviews → after approval, `apply` executes infrastructure changes in AWS/GCP/Azure.

---

## How it Works / Steps / Syntax

### Step 1: Configure Remote Backend

```hcl
terraform {
  backend "remote" {
    organization = "my-org"
    workspaces {
      name = "dev-environment"
    }
  }
}
```

### Step 2: Initialize Terraform

```bash
terraform init
```

### Step 3: Execute Remote Operations

* Plan remotely: `terraform plan`
* Apply remotely: `terraform apply`

### Step 4: Workspaces

* Use separate workspaces for dev, staging, prod.

### Step 5: Collaboration

* Multiple users can review plans and approve before applying.

---

## Common Issues / Errors

| Error                          | Cause                                               |
| ------------------------------ | --------------------------------------------------- |
| Unable to authenticate         | Invalid Terraform Cloud token                       |
| State lock                     | Another operation is running                        |
| Backend configuration mismatch | Local config differs from Terraform Cloud workspace |
| Insufficient permissions       | User token lacks access                             |
| Workspace not found            | Incorrect workspace name in config                  |

---

## Troubleshooting / Fixes

* Verify Terraform Cloud API token (`export TF_CLOUD_TOKEN="..."`).
* Wait/cancel running operations; force unlock if safe.
* Run `terraform init -reconfigure` for backend mismatch.
* Update workspace permissions.
* Create missing workspace via UI/CLI.

---

## Best Practices / Tips

* Always use **remote backend** for team projects.
* Use **workspaces** for each environment (dev, staging, prod).
* Enable **VCS integration** for auto-triggered runs.
* Store secrets using Terraform Cloud variables or secret storage.
* Require plan review before apply in production.
* Pin Terraform version using `required_version`.
* Enable notifications (Slack/email) for runs.
* Audit runs for compliance and troubleshooting.
* Consider free vs paid tier for team size and policy needs.

---
---

# 🧩 Detailed Notes – Automating Multi-Environment Deployments with Modules & AWS Accounts

## Concept / What:

**Automating Multi-Environment Deployments** is the process of using Terraform to deploy infrastructure across multiple environments (dev, QA, UAT, prod) and multiple AWS accounts automatically, with modules, environment-specific variables, and remote state management.

---

## Why / Purpose / Real-World Use Case:

* **Consistency:** Environments (dev, QA, UAT, prod) are structurally identical.
* **Reproducibility:** Changes tested in non-prod can be safely applied to prod.
* **Isolation:** Non-prod and prod accounts are isolated for security.
* **Automation:** CI/CD pipelines reduce manual deployments.
* **Collaboration & Governance:** Multiple teams can work safely; plan review and approval gates.

**Example:** Dev team pushes Terraform code → pipeline selects dev environment → plan and apply run → QA/UAT environments follow → production with approval.

---

## How it Works / Steps / Syntax

### 1. Directory Structure

```
terraform/
├── modules/
│   ├── vpc/
│   ├── ec2/
│   └── autoscaling/
├── non-prod/
│   ├── dev/
│   ├── qa/
│   └── uat/
└── prod/
```

* **modules/** contains reusable code (vpc, ec2, autoscaling).
* Each environment directory contains:

  * `main.tf` → calls modules
  * `variables.tf` → environment-specific variables
  * `.tfvars` → environment-specific values
  * `backend.hcl` → S3 + DynamoDB configuration for remote state

### 2. Using Modules

```hcl
module "vpc" {
  source      = "../../modules/vpc"
  cidr_block  = var.vpc_cidr
  environment = "dev"
}
```

* Reusable across environments; pass environment-specific variables

### 3. Environment-Specific Variables

* `dev.tfvars`, `qa.tfvars`, `uat.tfvars`, `prod.tfvars`

```hcl
aws_region = "us-east-1"
instance_type = "t2.micro"
vpc_id = "vpc-xxxxxx"
```

* Overrides default values defined in `variables.tf`

### 4. Backend Configuration (Remote State)

* **Single S3 bucket with dynamic keys** for non-prod environments:

```hcl
bucket         = "my-terraform-nonprod-state"
key            = "${var.env}/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-nonprod-lock"
encrypt        = true
```

* Prod has **separate bucket and backend** for isolation:

```hcl
bucket         = "my-terraform-prod-state"
key            = "prod/terraform.tfstate"
region         = "us-east-2"
dynamodb_table = "terraform-prod-lock"
encrypt        = true
```

* Terraform automatically creates state files per environment
* DynamoDB table ensures locking to prevent concurrent runs

### 5. CI/CD Pipeline Example

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

* `ENVIRONMENT` = dev, qa, uat, prod
* Dynamic backend key ensures separate state files
* Manual approval for prod recommended

---

## Common Issues / Errors

| Issue                    | Cause                                 |
| ------------------------ | ------------------------------------- |
| Wrong workspace selected | Pipeline/developer selected wrong env |
| Wrong AWS account        | Misconfigured provider credentials    |
| State conflicts          | Concurrent runs on same backend key   |
| Missing variables        | `.tfvars` file incomplete or missing  |
| Permission errors        | IAM roles lack access                 |

---

## Troubleshooting / Fixes

* Check active workspace: `terraform workspace show`
* Verify AWS credentials with `aws sts get-caller-identity`
* Reconfigure backend if needed: `terraform init -reconfigure`
* Ensure `.tfvars` file exists and is correct
* Locking handled by DynamoDB table; wait or force-unlock if safe

---

## Best Practices / Tips

* Use **workspaces** for non-prod environments (dev, QA, UAT)
* Use **separate backend** for production environment
* Parameterize all environment-specific values using `.tfvars`
* Keep **modules reusable** across environments
* Automate environment selection and deployment via **CI/CD pipelines**
* Use **approval gates** for production apply
* Maintain **audit logs and plan artifacts** for review
* Always use remote backend (S3 + DynamoDB) for state management and locking

---
---

# 🧩 Detailed Notes – Using terraform fmt & terraform validate in CI/CD

## Concept / What:

**`terraform fmt`**: Automatically formats Terraform configuration files according to standard style.

**`terraform validate`**: Checks Terraform code for syntax errors, configuration issues, and logical consistency without applying changes.

Used in CI/CD pipelines to ensure code quality and prevent errors before `plan` or `apply`.

---

## Why / Purpose / Real-World Use Case:

* Ensures consistent code style across teams.
* Detects syntax and logical errors early.
* Prevents broken infrastructure deployments.
* Integrated in CI/CD pipelines to enforce code quality.

**Example:**

* Developer pushes code → Pipeline runs `terraform fmt -check` and `terraform validate` → fails if code is misformatted or invalid → only valid code proceeds to `plan` and `apply`.

---

## How it Works / Steps / Syntax

### 1. terraform fmt

```bash
terraform fmt            # formats all .tf files in current directory
terraform fmt -recursive # formats files in subdirectories
terraform fmt -check     # checks formatting without modifying files (use in CI/CD)
```

### 2. terraform validate

```bash
terraform validate       # checks configuration for syntax and logical errors
```

* Combine with init if using modules or remote backend:

```bash
terraform init -backend-config=backend.hcl
terraform validate
```

### 3. CI/CD Pipeline Example

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

* `-check` ensures pipeline fails if formatting is incorrect.
* Validation stops pipeline on syntax or configuration errors.

---

## Common Issues / Errors

| Issue                       | Cause                                                   |
| --------------------------- | ------------------------------------------------------- |
| Formatting check fails      | Code not formatted per Terraform style                  |
| Validation fails            | Syntax error, missing variables, misconfigured provider |
| Validation fails on modules | Backend not initialized or module source missing        |

---

## Troubleshooting / Fixes

* Run `terraform fmt -recursive` locally to auto-format
* Ensure all required variables are provided
* Initialize backend before validation if using remote modules
* Check provider versions and module sources

---

## Best Practices / Tips

* Always run `terraform fmt -check` and `terraform validate` in CI/CD pipelines before plan/apply
* Configure pre-commit hooks locally to auto-format code
* Validate all environments in the pipeline
* Combine with plan artifact storage to catch issues before apply

---
---

# 🧩 Detailed Notes – Terraform Plan as Code Review Artifact (Storing Plan Files in Pipeline)

## Concept / What:

* Terraform plan generates an execution plan showing what changes will be made.
* Storing the plan file as a code review artifact allows **reviewing infrastructure changes before applying**.
* Ensures **safety, transparency, and collaboration** in CI/CD pipelines.

---

## Why / Purpose / Real-World Use Case:

* Prevents accidental changes to critical infrastructure.
* Code reviewers can **see exactly what resources will be created, modified, or destroyed**.
* Useful in **CI/CD pipelines** for gated approvals.

**Example:** Developer pushes code → Pipeline generates `terraform plan -out=tfplan` → plan file stored in artifacts → reviewers approve → apply happens.

---

## How it Works / Steps / Syntax

### 1. Generate Plan

```bash
terraform init -backend-config=backend.hcl
terraform plan -var-file=dev.tfvars -out=tfplan
```

### 2. Store Plan in CI/CD Pipeline with Environment Isolation

```groovy
pipeline {
    agent any
    environment {
        ENVIRONMENT = 'dev'  // dev, qa, uat, prod
    }
    stages {
        stage('Terraform Plan') {
            steps {
                dir("non-prod/${ENVIRONMENT}") {  // Jenkins working directory per environment
                    sh 'terraform init -backend-config=backend.hcl'
                    sh 'terraform fmt -check -recursive'
                    sh 'terraform validate'
                    sh 'terraform plan -var-file=${ENVIRONMENT}.tfvars -out=tfplan'
                    archiveArtifacts artifacts: 'tfplan', fingerprint: true
                }
            }
        }
        stage('Terraform Apply') {
            steps {
                input message: "Approve plan for ${ENVIRONMENT}?"
                dir("non-prod/${ENVIRONMENT}") {
                    sh 'terraform apply tfplan'
                }
            }
        }
    }
}
```

* **`dir()` block**: Changes the Jenkins working directory to the correct environment folder. Ensures Terraform commands are executed in the right context with correct `.tfvars` and backend.
* **Archive artifacts**: Stores plan files for review and audit.
* **Manual approval**: Ensures safety before applying changes.

---

## Common Issues / Errors

| Issue               | Cause                                                            |
| ------------------- | ---------------------------------------------------------------- |
| Plan file not found | `-out=tfplan` not used or incorrect path                         |
| Apply fails         | Variables in plan differ from current `.tfvars` or backend state |
| Permission denied   | Pipeline user lacks access to backend or resources               |

---

## Troubleshooting / Fixes

* Generate plan in the **same directory and backend** as apply
* Use **version-controlled `.tfvars`** to match plan and apply
* Ensure **pipeline agent has proper IAM permissions**
* Keep plan files **per environment** to avoid conflicts

---

## Best Practices / Tips

* Generate plan for every **push/merge request**
* Store plan files as **pipeline artifacts** for review and audit
* Use **manual approval gates** for production
* Keep plan files **short-lived** and delete after apply
* Combine with `terraform fmt` and `terraform validate` for full pipeline quality checks
* Use separate Jenkins working directories (`dir()`) for each environment to ensure isolation

---
---

# 🧩 Detailed Notes – Terraform Workspace Selection in Pipelines

## Concept / What:

* Terraform workspace allows multiple state files for a single configuration.
* Useful for separating environments (dev, QA, UAT, prod) without duplicating code.
* Each workspace maintains its own isolated state.

---

## Why / Purpose / Real-World Use Case:

* Avoids duplication of Terraform code for similar infrastructure across environments.
* Allows quick switching between environments.
* Integrated in CI/CD pipelines for dynamic environment selection.

---

## How it Works / Steps / Syntax

### 1. List Workspaces

```bash
terraform workspace list
```

### 2. Create Workspace

```bash
terraform workspace new dev
terraform workspace new qa
```

### 3. Select Workspace

```bash
terraform workspace select dev
terraform plan -out=tfplan
terraform apply tfplan
```

### 4. Workspace in CI/CD Pipeline

```groovy
pipeline {
    agent any
    environment {
        ENVIRONMENT = 'dev'  // dev, qa, uat, prod
    }
    stages {
        stage('Select Workspace & Plan') {
            steps {
                dir('terraform') {
                    sh 'terraform init -backend-config=backend.hcl'
                    sh "terraform workspace select ${ENVIRONMENT} || terraform workspace new ${ENVIRONMENT}"
                    sh 'terraform plan -var-file=${ENVIRONMENT}.tfvars -out=tfplan'
                    archiveArtifacts artifacts: 'tfplan', fingerprint: true
                }
            }
        }
        stage('Apply') {
            steps {
                input message: "Approve plan for ${ENVIRONMENT}?"
                dir('terraform') {
                    sh 'terraform apply tfplan'
                }
            }
        }
    }
}
```

* Pipeline selects or creates workspace dynamically.
* `dir()` isolates working directory per environment.

---

## Terraform State File Management for Workspaces

* Each workspace has **its own state file**, managed automatically.
* Use **same S3 bucket with dynamic key per workspace**:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-nonprod-state"
    key            = "workspace/${terraform.workspace}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-nonprod-lock"
    encrypt        = true
  }
}
```

* S3 structure:

```
my-terraform-nonprod-state/
├── workspace/dev/terraform.tfstate
├── workspace/qa/terraform.tfstate
└── workspace/uat/terraform.tfstate
```

* **DynamoDB** handles locking per workspace key.
* **Prod** environment can optionally use a **separate bucket** for isolation.
* Terraform automatically maps workspace → state file; no manual state file creation needed.

---

## Common Issues / Errors

| Issue               | Cause                                                   |
| ------------------- | ------------------------------------------------------- |
| Workspace not found | Not created or typo in name                             |
| State conflicts     | Multiple environments using same workspace concurrently |
| Apply fails         | Workspace selected incorrectly, wrong `.tfvars`         |

---

## Troubleshooting / Fixes

* Verify active workspace: `terraform workspace show`
* Create workspace if missing in pipeline: `terraform workspace new <name>`
* Ensure `.tfvars` matches environment
* Avoid using workspaces for fully isolated prod environments; consider separate backend

---

## Best Practices / Tips

* Ideal for **non-prod environments** sharing similar infrastructure
* Use **separate backend** for prod environments
* Always select workspace **before plan/apply** in CI/CD
* Combine with `.tfvars` for environment-specific variables
* Use `dir()` in Jenkins to isolate working directory per environment

---
---
---



