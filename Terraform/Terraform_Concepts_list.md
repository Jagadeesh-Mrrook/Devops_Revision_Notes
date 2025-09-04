# Terraform Master Checklist for 5+ Years Experience

A comprehensive list of Terraform concepts, features, and integrations for interviews and real-world DevOps work.

---

## 1. **Terraform Fundamentals**
- [ ] What is Terraform & why it's used
- [ ] Terraform architecture & workflow
- [ ] Installation (Linux, Windows, MacOS)
- [ ] Terraform CLI overview (`init`, `plan`, `apply`, `destroy`, `validate`, `fmt`, `taint`, `import`)

---

## 2. **Providers & Resources**
- [ ] Providers configuration (AWS, Azure, GCP, etc.)
- [ ] Resource creation & management
- [ ] Resource attributes, arguments, and meta-arguments
- [ ] Lifecycle rules (`create_before_destroy`, `prevent_destroy`)
- [ ] Data sources

---

## 3. **Variables & Outputs**
- [ ] Input variables (string, list, map, bool)
- [ ] Default values & type constraints
- [ ] Output values & sensitive outputs
- [ ] Using variables in modules and resources

---
## 4. **State Management**
- [ ] Terraform state (local & remote)
- [ ] Backend configuration (S3, GCS, Azure Storage, Consul)
- [ ] State locking & versioning
- [ ] Workspaces for multi-environment setups
- [ ] Importing existing infrastructure
- [ ] State file encryption (e.g., KMS when using S3 backend)
- [ ] terraform state subcommands (list, mv, rm, show)

---
## 5. **Modules**
- [ ] Creating reusable modules
- [ ] Module inputs and outputs
- [ ] Calling modules from local paths, Git, registry
- [ ] Module versioning and best practices
- [ ] Publishing modules to Terraform Registry (internal/private)
- [ ] Nested modules & composition

---

## 6. **Provisioners**
- [ ] `local-exec`
- [ ] `remote-exec`
- [ ] `file` provisioner
- [ ] null_resource with triggers
- [ ] When and why to use provisioners (real-world scenarios)
- [ ] Avoiding anti-patterns with provisioners

---
## 7. **Expressions & Functions**
- [ ] Terraform expressions (conditionals, operators)
- [ ] Built-in functions (string, numeric, collection, date)
- [ ] Using expressions in resource arguments
- [ ] Dynamic blocks (`dynamic` keyword)
- [ ] for_each vs count (scaling resources)
- [ ] for loops & for expressions
- [ ] depends_on (explicit dependency handling)


---

## 8. **Terraform CLI Workflows**
- [ ] `terraform init`, `plan`, `apply`, `destroy`
- [ ] `terraform validate`, `fmt`, `taint`, `import`
- [ ] Refresh, state manipulation, and resource targeting
- [ ] Diff and plan output interpretation

---
## 9. **CI/CD & Automation**
- [ ] Terraform integration in Jenkins pipelines
- [ ] Terraform Cloud & remote operations
- [ ] Automating multi-environment deployments
- [ ] Using `terraform fmt` & `validate` in CI/CD
- [ ] Terraform plan as code review artifact (storing plan files in pipeline)
- [ ] Terraform workspace selection in pipelines

---
## 10. **Security & Best Practices**
- [ ] Storing secrets securely (Vault, environment variables)
- [ ] Sensitive variables & outputs
- [ ] Version control of Terraform files
- [ ] Avoid hardcoding credentials
- [ ] Minimal use of provisioners
- [ ] Using `terraform fmt`, `validate`, and `linting tools`
- [ ] Enforce policies with Sentinel / OPA (Policy as Code)
- [ ] .terraformignore usage (when building modules)

---

## 11. **Troubleshooting & Common Errors**
- [ ] Provider authentication failures
- [ ] State conflicts & lock errors
- [ ] Plan/apply discrepancies
- [ ] Resource dependencies & cycle errors
- [ ] Handling drift between code & real infrastructure

---
## 12. **Hands-On Scenarios**
- [ ] Provision a VPC with subnets, route tables, and security groups
- [ ] Deploy EC2 instances (or equivalent compute) using modules
- [ ] Multi-environment setup (dev/qa/prod) with workspaces
- [ ] Integrate Terraform with CI/CD pipeline
- [ ] Manage remote state and collaborate in a team environment


---

**✅ Pro Tip:**  
Check off each item as you learn it. Combine this with **hands-on practice** to be interview-ready and confident in real-world Terraform usage.

