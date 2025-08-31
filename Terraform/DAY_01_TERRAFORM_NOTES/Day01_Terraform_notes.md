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
### Concept
- Initializes a Terraform project and downloads required providers and modules.
### Purpose / Use Case
- Required before running plan or apply; sets up backend and providers.
### Syntax
- terraform init
- Optional flags:
  - `-backend-config="key=value"` → customize backend
  - `-upgrade` → update provider plugins
### Common Issues
- Missing provider plugin
- Backend misconfiguration
### Troubleshooting
- Check provider block and backend config
- Use `terraform init -upgrade` to refresh plugins

---

## 2. terraform plan
### Concept
- Shows execution plan: what Terraform will create, modify, or destroy.
### Purpose / Use Case
- Preview changes before applying to avoid unintended modifications.
### Syntax
- terraform plan
- Optional: `-out=planfile` → save plan for later apply
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
### Syntax
- terraform apply
- Optional: terraform apply planfile
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
### Syntax
- terraform taint <resource_type.resource_name>
- Example: terraform taint aws_instance.web_server
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

