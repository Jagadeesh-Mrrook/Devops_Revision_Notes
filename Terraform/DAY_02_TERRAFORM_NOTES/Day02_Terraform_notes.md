# Terraform Today’s Concepts Notes: Providers, Resources, Multi-Profile Usage

---

## Detailed Version

### Providers
**What / Concept:**
- Providers are plugins in Terraform that allow it to interact with cloud platforms (AWS, Azure, GCP) or other services (GitHub, Cloudflare). They are the entry point in a Terraform configuration and define which platform Terraform will manage.

**Why / Purpose / Use Case in Real-World:**
- Specifies the cloud or service Terraform will provision resources on.
- Ensures Terraform downloads and uses the correct plugin version for the provider.
- Supports multi-cloud and hybrid environments by simply switching providers.
- Allows managing multiple accounts, regions, or environments via configuration.

**How it Works / Steps / Syntax:**
- Declare provider in `.tf` file:
```hcl
provider "aws" {
  region  = "us-east-1"
  profile = "default"
}
```
- Optionally use `required_providers` block:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```
- Initialize project with `terraform init` → downloads provider plugin.
- Providers can include credentials, region, or profile information.

**Common Issues / Errors:**
- Missing or misconfigured credentials → authentication failure.
- Provider plugin version mismatch → errors in `terraform plan` or `apply`.
- Region not supported by provider → resource creation fails.

**Troubleshooting / Fixes:**
- Verify credentials in AWS CLI or provider block.
- Use correct provider version in `required_providers`.
- Ensure region is valid for selected provider.

**Best Practices / Tips:**
- Always use `required_providers` in production environments.
- Version-lock provider plugins to avoid unexpected behavior.
- Use profiles for multiple accounts instead of hardcoding credentials.
- Keep provider configuration consistent across environments.

---

### Resources
**What / Concept:**
- Resource blocks in Terraform define the infrastructure objects to be created, updated, or destroyed. Examples include EC2 instances, S3 buckets, VPCs, and subnets.

**Why / Purpose / Use Case in Real-World:**
- Represents actual cloud or service resources Terraform manages.
- Allows declarative management: Terraform ensures resources match desired state.
- Facilitates infrastructure automation without manual intervention.

**How it Works / Steps / Syntax:**
- Basic resource declaration:
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"
}
```
- Supports attributes, arguments, and meta-arguments like `depends_on`, `count`, `for_each`.
- Lifecycle rules can be added to control creation/destruction:
```hcl
lifecycle {
  create_before_destroy = true
  prevent_destroy       = false
}
```
- Terraform manages these resources via `terraform plan` and `terraform apply`.

**Common Issues / Errors:**
- Missing required attributes → resource creation fails.
- Dependency conflicts between resources.
- Misconfigured lifecycle rules → unintended destruction.

**Troubleshooting / Fixes:**
- Check Terraform documentation for required resource arguments.
- Use `terraform plan` to preview changes and dependencies.
- Adjust lifecycle rules carefully to avoid data loss.

**Best Practices / Tips:**
- Use lifecycle rules in production to protect critical resources.
- Modularize resource blocks for reuse.
- Always review `terraform plan` before applying.
- Keep resource names and variables clear for maintainability.

---

### Multi-Profile Usage
**What / Concept:**
- Multi-profile usage allows Terraform to dynamically select different cloud accounts or environments via profiles configured in the CLI or provider block.

**Why / Purpose / Use Case in Real-World:**
- Enables managing multiple environments (Dev, QA, UAT, Prod) using the same Terraform codebase.
- Avoids hardcoding credentials for different accounts.
- Simplifies CI/CD pipelines when working with multiple accounts.

**How it Works / Steps / Syntax:**
- Configure multiple profiles in AWS CLI (`~/.aws/credentials`):
```
[default]
aws_access_key_id = XXX
aws_secret_access_key = XXX

[qa]
aws_access_key_id = YYY
aws_secret_access_key = YYY
```
- Reference profiles in Terraform provider block:
```hcl
provider "aws" {
  region  = "us-east-1"
  profile = var.aws_profile  # variable can be default, qa, uat, etc.
}
```
- Dynamically switch profiles by passing variable:
```bash
terraform apply -var='aws_profile=qa'
```

**Common Issues / Errors:**
- Profile name mismatch → authentication failure.
- Credentials missing or misconfigured.
- Conflicts between environment variables and profile credentials.

**Troubleshooting / Fixes:**
- Verify profile names in `~/.aws/credentials`.
- Use correct variable when switching profiles.
- Ensure no conflicting AWS environment variables are set.

**Best Practices / Tips:**
- Use variables to switch profiles dynamically in CI/CD pipelines.
- Keep profile credentials secure and out of version control.
- Test each profile before production deployment.
- Document all available profiles for team clarity.

---

# 🌩️ Terraform — Multi‑Account (AWS Profiles) Notes

**Scope:** Practical guide + 3 common cases for using AWS CLI *profiles* with Terraform to manage resources across multiple AWS accounts (Account A, Account B). Includes provider configuration, IAM requirements, example code, module pattern, and best practices.

---

## Summary (one-line)

Use **multiple provider blocks with `alias` + AWS CLI profiles** (in `~/.aws/credentials`) and call resources/modules with `provider = aws.<alias>` (or `providers = { aws = aws.<alias> }` for modules) to create resources in different accounts from a single Terraform run.

---

## Prerequisites

* Terraform installed on one execution host (local machine, CI agent, or Jenkins). You DO NOT install Terraform "in" AWS accounts.
* AWS CLI profiles configured in `~/.aws/credentials` (or environment variables). Example profiles: `accountA`, `accountB`.
* IAM users (or IAM roles) in each account with appropriate permissions for the resources Terraform will create.
* (Optional) Remote state backend (S3 + DynamoDB) per account or workspace to avoid conflicts.

`~/.aws/credentials` example:

```ini
[accountA]
aws_access_key_id = AKIA...A
aws_secret_access_key = s3cr3tA

[accountB]
aws_access_key_id = AKIA...B
aws_secret_access_key = s3cr3tB
```

---

## Provider configuration pattern

```hcl
provider "aws" {
  alias   = "accA"
  region  = "ap-south-1"
  profile = "accountA"
}

provider "aws" {
  alias   = "accB"
  region  = "ap-south-1"
  profile = "accountB"
}
```

* Each `provider` block maps to an AWS account/profile. Use `alias` to reference it from resources or modules.

---

# Case 1 — Create resources only in **Account A** (single-account)

**Use case:** Manage dev account resources only.

**Provider:** single provider or `provider "aws" { profile = "accountA" }` (no alias needed).

**Example:** create a VPC and an EC2 in Account A

```hcl
provider "aws" {
  region  = "ap-south-1"
  profile = "accountA"
}

resource "aws_vpc" "a_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "accA-vpc" }
}

resource "aws_instance" "a_ec2" {
  ami           = "ami-..."
  instance_type = "t3.micro"
  subnet_id     = aws_subnet.a_subnet.id
}
```

**IAM/Permissions:** the IAM user for `accountA` profile must have `ec2:Create*`, `ec2:Describe*`, `ec2:Delete*`, `vpc:*` as needed.

**When to use:** simple single-account workflows and dev environments.

---

# Case 2 — Create different resources in **Account A** and **Account B** from one Terraform run

**Use case:** Centralized infra repo that provisions an account-specific VPC in Account A and an EC2 instance in Account B.

**Provider blocks (aliases) — required**

```hcl
provider "aws" {
  alias   = "accA"
  region  = "ap-south-1"
  profile = "accountA"
}

provider "aws" {
  alias   = "accB"
  region  = "ap-south-1"
  profile = "accountB"
}
```

**Resource examples**

```hcl
resource "aws_vpc" "accA_vpc" {
  provider   = aws.accA
  cidr_block = "10.0.0.0/16"
  tags = { Name = "accA-vpc" }
}

resource "aws_instance" "accB_ec2" {
  provider      = aws.accB
  ami           = "ami-..."
  instance_type = "t3.micro"
  tags = { Name = "accB-ec2" }
}
```

**Notes:**

* Each resource explicitly references the provider alias. Terraform will call the AWS API for **Account A** when a resource uses `aws.accA` and **Account B** when `aws.accB` is used.
* Use separate remote state backends per account or a state organization that avoids collisions (recommended).

**IAM/Permissions:** ensure the IAM user tied to `accountA` profile has permissions for VPC resources; the IAM user for `accountB` profile has EC2 permissions.

---

# Case 3 — Reuse code: Create the *same* resource type in multiple accounts (DRY) using modules

**Goal:** avoid copy/paste resource blocks; call the same module for each account and pass a provider mapping.

**Directory layout (recommended)**

```
terraform/
├── main.tf
├── variables.tf
├── providers.tf
└── modules/
    └── ec2/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

**modules/ec2/main.tf**

```hcl
resource "aws_instance" "this" {
  ami           = var.ami
  instance_type = var.instance_type
  # subnet_id or other inputs come from variables
  tags = { Name = var.name }
}
```

**Root main.tf using module multiple times**

```hcl
provider "aws" { alias = "accA"  region = "ap-south-1" profile = "accountA" }
provider "aws" { alias = "accB"  region = "ap-south-1" profile = "accountB" }

module "ec2_in_accA" {
  source    = "./modules/ec2"
  providers = { aws = aws.accA }
  ami       = "ami-..."
  instance_type = "t3.micro"
  name      = "ec2-in-accA"
}

module "ec2_in_accB" {
  source    = "./modules/ec2"
  providers = { aws = aws.accB }
  ami       = "ami-..."
  instance_type = "t3.micro"
  name      = "ec2-in-accB"
}
```

**Advanced:** you can call the module in a loop to create the same resource in N accounts via a map input, but you must still declare provider aliases upfront and map them to module calls.

Example loop (pseudo):

```hcl
locals { accounts = { accA = aws.accA, accB = aws.accB } }
# iterate keys to create modules: you still must map providers explicitly per module instance
```

---

## Important operational details

### 1. Terraform execution location

* TF CLI runs on one machine (developer workstation or CI). It uses AWS APIs over the network. **You do not install Terraform inside each AWS account.**

### 2. IAM permissions

* Each AWS profile corresponds to an IAM user (or role) in that account; attach fine‑grained policies. Example minimum for EC2/VPC: `ec2:Describe*`, `ec2:Create*`, `ec2:Delete*`, `ec2:RunInstances`.

### 3. Remote state

* Use separate remote state per account or use separate workspaces to avoid accidental cross-account collisions. Common pattern: one state bucket per account and environment (e.g., `tf-state-app-accountA`, `tf-state-app-accountB`).

### 4. Secrets & credentials

* Prefer environment-based profiles for CI/CD (configure credentials in Jenkins credentials store or use instance profile for Jenkins agents). Never hardcode keys in Terraform files.

### 5. Provider limitations

* Providers are static at plan-time—you cannot make `provider` dynamic inside a single resource block. Use modules + provider mapping to achieve multi-account DRY deployments.

---

## Example `providers.tf` + `main.tf` (complete minimal example)

```hcl
# providers.tf
provider "aws" {
  alias   = "accA"
  region  = "ap-south-1"
  profile = "accountA"
}

provider "aws" {
  alias   = "accB"
  region  = "ap-south-1"
  profile = "accountB"
}

# main.tf (uses both)
resource "aws_vpc" "accA_vpc" {
  provider   = aws.accA
  cidr_block = "10.0.0.0/16"
  tags = { Name = "accA-vpc" }
}

resource "aws_instance" "accB_ec2" {
  provider      = aws.accB
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = { Name = "accB-ec2" }
}
```

---

## Troubleshooting & tips

* If Terraform fails with `NoCredentialProviders` or `InvalidClientTokenId` → check `~/.aws/credentials` profile names and that the CLI can `aws sts get-caller-identity --profile accountA`.
* When planning for multiple accounts, prefix resource names with account id/alias to avoid confusion.
* Use `terraform workspace` or separate directories per environment to avoid accidental cross-deletes.

---

## Interview-ready summary

1. Define AWS CLI profiles for each account.
2. Declare multiple `provider "aws"` blocks with `alias` and `profile`.
3. Reference the correct provider on each resource (`provider = aws.accA`) or call modules with `providers = { aws = aws.accA }`.
4. Ensure IAM users for each profile have proper policies.
5. Use separate remote state for each account or workspace.

---

If you want, I can also:

* Generate a ready-to-run **example repo** with modules for VPC & EC2 and provider setup; or
* Add a short **cheat-sheet** that you can paste into interviews (2–3 bullet points). Which would you like?

---


# Terraform Cross-Account Resource Creation using IAM Role Assumption

This guide explains step-by-step how a user in Account A can create resources in Account B using IAM Role Assumption. This approach does not require creating multiple AWS profiles.

---

## 1. Prerequisites

* Two AWS accounts: Account A (Developer account) and Account B (Target account for resource creation)
* Terraform installed on the system where execution will happen
* User in Account A with IAM permissions to assume role on Account B

---

## 2. Account B Setup (Target Account)

### 2.1 Create IAM Role

* Go to IAM in Account B
* Create a new role with `Role for Cross-Account Access`
* Select **Another AWS account** and provide Account A ID
* Attach necessary IAM policies (e.g., `AmazonEC2FullAccess`, `AmazonVPCFullAccess`) for resources that need to be created
* Note down the Role ARN (example: `arn:aws:iam::ACCOUNT_B_ID:role/CrossAccountTerraformRole`)

### 2.2 Configure Trust Policy

* Trust policy should allow the IAM user from Account A to assume this role

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_A_ID:user/DevUser"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

---

## 3. Account A Setup (Developer Account)

### 3.1 Configure IAM User

* Ensure IAM user has permission to assume the role in Account B
* Policy example for IAM user:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::ACCOUNT_B_ID:role/CrossAccountTerraformRole"
    }
  ]
}
```

### 3.2 Terraform Configuration in Account A

* Provider configuration for Account A (default account):

```hcl
provider "aws" {
  region = "us-east-1"
}
```

* Provider configuration for Account B using `assume_role`:

```hcl
provider "aws" {
  alias  = "account_b"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::ACCOUNT_B_ID:role/CrossAccountTerraformRole"
  }
}
```

---

## 4. Creating Resources

### 4.1 Resource in Account A

```hcl
resource "aws_vpc" "dev_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "DevVPC"
  }
}
```

### 4.2 Resource in Account B

```hcl
resource "aws_instance" "web_instance" {
  provider = aws.account_b
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"

  tags = {
    Name = "CrossAccountWebInstance"
  }
}
```

* Notice the `provider = aws.account_b` in the resource block to indicate cross-account creation.

---

## 5. Terraform Execution Steps

1. `terraform init` → Initialize Terraform
2. `terraform plan` → Validate resources and providers
3. `terraform apply` → Apply changes, creating resources in Account A and Account B

---

## 6. Notes

* Only one AWS profile is needed for Account A (developer executing Terraform)
* Cross-account resources in Account B are created by assuming the IAM role
* No need to manage multiple AWS profiles or multiple providers unless required for other scenarios
* This approach ensures secure access without sharing static credentials across accounts

---

## 7. Summary

* Account B: Create IAM role and trust policy for cross-account access
* Account A: IAM user with `sts:AssumeRole` permissions
* Terraform: Configure provider with `assume_role` for cross-account resource creation
* Resources: Specify `provider` alias in Terraform resource blocks for Account B
* Execution: Run Terraform from Account A with single profile

This method simplifies cross-account management and aligns with security best practices by using IAM roles instead of static credentials.

---
---



# Terraform Resource Creation & Management

## Detailed Explanation Version

### Concept / What:
- Resource creation and management in Terraform is the process of defining infrastructure resources declaratively, provisioning them in the cloud, updating configurations when needed, and eventually deleting them.
- Resources can include cloud components like EC2 instances, S3 buckets, VPCs, subnets, databases, or even external services like GitHub repositories.
- Terraform ensures that the actual infrastructure matches the desired state described in your configuration files.

### Why / Purpose / Use Case in Real-World:
- Automates infrastructure provisioning across multiple environments (Dev, QA, UAT, Prod).
- Ensures consistent and reproducible setups so that all team members work with identical infrastructure.
- Facilitates version control of infrastructure and tracks changes over time.
- Reduces human errors by previewing planned changes before applying (`terraform plan`).
- Supports collaboration within teams with safe infrastructure modifications.

### How it Works / Steps / Syntax:
1. **Define Resource:** Specify resource type, name, and required attributes in a `.tf` file.
```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"
  tags = {
    Name = "WebServer"
  }
}
```
2. **Initialize Terraform:** Run `terraform init` to download required provider plugins and prepare the working directory.
3. **Plan Changes:** Run `terraform plan` to preview what Terraform will create, modify, or destroy.
4. **Apply Changes:** Run `terraform apply` to provision or update resources to match your configuration.
5. **Update Resources:** Modify resource attributes in `.tf` files; rerun `terraform plan` and `apply`.
6. **Destroy Resources:** Run `terraform destroy` to safely remove all defined resources.
7. **Optional Management:** Use `terraform refresh` to synchronize state with actual infrastructure or `terraform import` to bring existing resources under Terraform management.

### Common Issues / Errors:
- Missing or invalid required attributes in resource blocks.
- Resource dependencies not defined properly, leading to creation order issues.
- Manual changes outside Terraform causing drift between desired and actual infrastructure.
- Conflicts during updates if resources are live or in use.

### Troubleshooting / Fixes:
- Always run `terraform plan` to inspect changes before applying.
- Use `depends_on` to define dependencies explicitly.
- Use `terraform refresh` to sync state with actual infrastructure.
- Use `terraform import` to bring manually created resources into Terraform state.
- Review changes carefully before applying in production environments.

### Best Practices / Tips:
- Modularize resources for reuse across projects.
- Name resources clearly and consistently.
- Keep `.tf` files in version control (Git) for collaboration.
- Avoid making manual changes outside Terraform; if done, sync with state.
- Test resource creation in Dev/QA before deploying in Prod.
- Use lifecycle rules carefully to prevent accidental destruction of critical resources.

---

# Terraform Resource Attributes, Arguments, and Meta-Arguments

## Detailed Explanation Version

### Concept / What:
- **Arguments:** User-provided inputs in the resource block to define how a resource should be created (e.g., `ami`, `instance_type` for EC2).
- **Attributes:** Output values generated by Terraform after resource creation, which can be referenced in other resources (e.g., `id`, `arn`).
- **Meta-Arguments:** Special arguments that control Terraform’s behavior with the resource, not the resource’s properties (e.g., `count`, `for_each`, `depends_on`, `lifecycle`).

### Why / Purpose / Use Case in Real-World:
- **Arguments:** Define exact configuration needed for infrastructure creation.
- **Attributes:** Allow referencing resource outputs dynamically, enabling dependencies and integration between resources.
- **Meta-Arguments:** Handle advanced resource behavior, e.g., creating multiple instances efficiently, managing dependencies, or preventing accidental destruction.
- Ensures predictable, maintainable, and scalable infrastructure management.

### How it Works / Steps / Syntax:

**Example: AWS EC2 instance with arguments, attributes, and meta-arguments**
```hcl
resource "aws_instance" "web_server" {
  # Arguments
  ami           = "ami-0abcdef1234567890"
  instance_type = "t2.micro"
  tags = {
    Name = "WebServer"
  }

  # Meta-arguments
  count = 2  # creates 2 instances
  depends_on = [aws_security_group.web_sg]

  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
  }
}

# Attributes usage example
output "web_instance_id" {
  value = aws_instance.web_server[0].id
}
```
- **Arguments:** `ami`, `instance_type` provided by the user.
- **Attributes:** `aws_instance.web_server[0].id` produced after resource creation.
- **Meta-Arguments:** `count`, `depends_on`, `lifecycle` control Terraform’s execution.

### Common Issues / Errors:
- Referencing attributes before resource creation → plan fails.
- Misusing `count` or `for_each` → index/key errors.
- Circular dependencies → incorrect `depends_on` usage.
- Provider override misconfiguration → resource creation fails.

### Troubleshooting / Fixes:
- Use `terraform plan` to inspect attributes and dependencies.
- Double-check `count` and `for_each` syntax.
- Validate dependencies; use `terraform graph` to visualize.
- Ensure provider block matches resource if overridden.

### Best Practices / Tips:
- Reference attributes dynamically instead of hardcoding IDs.
- Parameterize resources using arguments for multi-environment setups.
- Use meta-arguments judiciously for advanced control.
- Avoid circular dependencies; rely on implicit ones when possible.
- Document meta-argument usage clearly for team collaboration.

---

# Terraform Lifecycle Rules / Meta-Arguments - Detailed Version


## Concept / What
Lifecycle arguments are special **meta-arguments** in Terraform that control how Terraform manages the **creation, modification, and deletion of resources**. They do not define resources themselves but manage Terraform's behavior while handling the resources.


## Why / Purpose / Use Case in Real-World
- Prevent accidental deletion or downtime of critical or high-availability resources.
- Handle attributes that may change outside Terraform without triggering unwanted updates.
- Ensure safe and predictable resource management in production environments.
- Examples:
- `PreventDestroy` → critical production databases.
- `CreateBeforeDestroy` → load balancers, auto-scaling groups where downtime is unacceptable.
- `IgnoreChanges` → attributes managed by external processes or auto-scaling policies.


## How it Works / Steps / Syntax
Lifecycle arguments are defined **inside a resource block**:


```hcl
resource "aws_instance" "web" {
ami = "ami-12345678"
instance_type = "t2.micro"


lifecycle {
prevent_destroy = true
create_before_destroy = true
ignore_changes = [instance_type]
}
}
```


- **PreventDestroy:** Terraform will **fail** if a destroy is attempted on this resource.
- **CreateBeforeDestroy:** Terraform **creates a new resource first** before destroying the old one to avoid downtime.
- **IgnoreChanges:** Terraform **ignores changes** to specified attributes, even if they differ from the configuration.


### Workflow with Drift
- If manual changes happen outside Terraform:
- `terraform refresh` → updates state to reflect reality.
- `ignore_changes` → allows specific attributes to drift without Terraform overwriting them.


## Common Issues / Errors
- Misunderstanding `ignore_changes` → expecting Terraform to revert manual changes.
- Using `create_before_destroy` on resources without proper dependencies → temporary conflicts.
- Applying `prevent_destroy` on resources that must be replaced → plan fails unexpectedly.


## Troubleshooting / Fixes
- Use `terraform plan` to preview lifecycle behavior before applying.
- For ignored attributes, verify state with `terraform show`.
- Adjust resource dependencies when using `create_before_destroy` to avoid conflicts.


## Best Practices / Tips
- Use lifecycle rules **only when necessary**; don’t overuse.
- Combine `prevent_destroy` with proper backup strategies for critical resources.
- Document `ignore_changes` for team awareness.
- Always review `terraform plan` output carefully when lifecycle rules are applied.

---

# Terraform Data Sources Notes

---
## 📘 Detailed Explanation Version

### Concept / What:
Terraform **Data Sources** allow us to fetch or reference existing resources or information that are **not directly created in the current Terraform configuration**.  
They act as **read-only** objects which expose attributes from infrastructure for use in other resources or modules.

---

### Why / Purpose / Use Case in Real-World:
- To **reuse existing infrastructure** instead of duplicating configuration.  
- Example: Use an existing **VPC, AMI, Security Group, IAM Role** instead of creating new ones.  
- Enables **multi-team/multi-environment collaboration** → one team manages networking, another consumes those networks using data sources.  
- Provides **dynamic and consistent lookups** (e.g., latest AMI, available zones).  
- Helps avoid **hardcoding values** that may change across environments (dev, qa, prod).

---

### How it Works / Steps / Syntax:
1. Use the `data` block in Terraform.  
2. Specify the provider, resource type, and filters.  
3. Reference attributes in other resources.  

**Example:** Fetching latest Ubuntu AMI in AWS  
```hcl
data "aws_ami" "ubuntu_latest" {
  most_recent = true
  owners      = ["099720109477"] # Canonical
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu_latest.id
  instance_type = "t2.micro"
}
```

---

### Common Issues / Errors:
- **Data not found** → wrong filters or incorrect region.  
- **Permissions error** → IAM role/user doesn’t have permission to describe resources.  
- **Hardcoding values** when a data source should have been used.  
- **Cross-account reference issue** when trying to fetch resources from another AWS account without permissions.  

---

### Troubleshooting / Fixes:
- Validate filters using AWS CLI before using in Terraform.  
- Add correct IAM policies (e.g., `ec2:DescribeImages`).  
- Use `terraform plan` to verify data fetch.  
- Use **outputs** to debug and print data source values.  

---

### Best Practices / Tips:
- Prefer **data sources** instead of hardcoding values (like AMI IDs).  
- Use **variables + data sources** for flexibility across environments.  
- Avoid over-fetching or complex filters → keep queries optimized.  
- Clearly **separate modules** → resources in one module, data sources in consumers.  

---

