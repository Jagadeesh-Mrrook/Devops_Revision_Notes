# Terraform Fundamentals: What is Terraform & Why It's Used

## Concept
- Open-source IaC tool to define, provision, and manage infrastructure.

## Purpose / Use Case
- Consistent infra across dev/staging/prod  
- Version control for infra changes  
- Multi-cloud/hybrid management  
- Automates provisioning, reduces human error  

## Steps / Syntax
- Write `.tf` files → `terraform init` → `terraform plan` → `terraform apply`  

## Common Issues
- Wrong provider credentials  
- Unsupported regions  
- Resource dependency errors  

## Fixes
- Verify credentials & permissions  
- Check resource availability  
- Use `terraform plan` before apply  

## Best Practices
- Git version control  
- Use workspaces for environments  
- Avoid hardcoding secrets  
- Run `terraform fmt` for consistent formatting


---
# Terraform Architecture & Workflow - Quick Revision

## Concept
- Core: execution plan
- Providers: cloud/service interaction
- State: tracks current infra

## Purpose / Use Case
- Collaborative infra management
- Detect & fix drift
- CI/CD & remote state integration

## Steps / Syntax
- Write `.tf` → `terraform init` → `plan` → `apply` → manage state → `destroy`

## Common Issues
- State conflicts / lock errors
- Missing providers
- Configuration drift

## Fixes
- Remote state with locking
- `terraform init -upgrade`
- `terraform refresh`

## Best Practices
- Secure & version-controlled state
- Modularize code
- Always check `terraform plan`
- Use workspaces for environments
---
# Terraform Installation (Linux) - Quick Revision

## Concept
- Install Terraform CLI on Linux to manage infrastructure.

## Purpose / Use Case
- Linux servers for DevOps & automation.
- Integrates with Jenkins, GitLab CI, etc.

## Steps
1. `wget https://releases.hashicorp.com/terraform/<version>/terraform_<version>_linux_amd64.zip`
2. `unzip terraform_<version>_linux_amd64.zip`
3. `sudo mv terraform /usr/local/bin/`
4. `terraform version`

## Common Issues
- Command not found
- Permission errors
- Version mismatch

## Fixes
- Check PATH
- Use sudo
- Use `tfenv` for version management

## Best Practices
- Use `tfenv`
- Check release notes
- Verify with `terraform version`

---

# tfenv - Quick Revision

## Concept
- Terraform version manager for installing and switching versions.

## Purpose / Use Case
- Avoid version conflicts across projects.
- Ensure consistent Terraform version for team projects.
- Test multiple Terraform versions easily.

## Steps
1. Install `tfenv` via Git and update PATH
2. `tfenv list-remote` → see available versions
3. `tfenv install <version>` → install version
4. `tfenv use <version>` → set global version
5. `.terraform-version` → set project-specific version

## Common Issues
- PATH not set → command not found
- Version not recognized → re-source shell
- Conflicts with manual Terraform binaries

## Fixes
- Check PATH
- Use `tfenv list`
- Remove conflicting binaries

## Best Practices
- Use `.terraform-version` per project
- Keep `.terraform-version` in Git
- Check for updates via `tfenv list-remote`

---

# Terraform CLI Commands - Quick Revision

## init
- Initializes project, downloads providers/modules
- terraform init
- Issues: missing plugin, backend misconfig
- Fix: check provider/backend, use -upgrade

## plan
- Shows execution plan
- terraform plan [-out=planfile]
- Issues: invalid config, missing vars
- Fix: terraform validate, provide vars

## apply
- Applies desired infra state
- terraform apply [planfile]
- Issues: auth failures, manual conflicts
- Fix: check credentials, terraform refresh

## destroy
- Deletes resources
- terraform destroy [-target=resource, -auto-approve]
- Issues: dependency errors, auth failures
- Fix: destroy in order, verify permissions

## validate
- Checks config syntax
- terraform validate
- Issues: syntax/type errors
- Fix: correct syntax, verify provider

## fmt
- Formats Terraform files
- terraform fmt [-recursive]
- Issues: style only
- Fix: fmt -recursive

## taint
- Marks resource for recreation
- terraform taint <resource_type.resource_name>
- Issues: wrong name/type, state mismatch
- Fix: terraform state list, terraform state rm if deleted
- Tips: use sparingly, plan before apply, avoid critical prod

## import
- Imports existing infra into state
- terraform import <resource_type.resource_name> <resource_id>
- Issues: wrong type/ID, mismatch with .tf
- Fix: verify type/ID, run plan, update .tf
- Tips: backup state, start non-critical, plan to confirm

