# Ansible Master Concept Plan (Configuration Management Focus)

## 1. Ansible Fundamentals

* [ ] What is Ansible and why it is used
* [ ] Agentless architecture (SSH-based)
* [ ] Control node vs managed nodes
* [ ] Push model
* [ ] YAML basics
* [ ] Installation and ansible.cfg basics

## 2. Inventory Management

* [ ] Static inventory (INI, YAML)
* [ ] Host groups
* [ ] Host and group variables
* [ ] Inventory best practices

## 3. Ad-hoc Commands

* [ ] Ping all servers
* [ ] Running shell commands
* [ ] Installing packages
* [ ] Managing services
* [ ] Copying files
* [ ] Managing users
* [ ] Why playbooks are preferred over ad-hoc

## 4. Playbooks

* [ ] Playbook structure
* [ ] Tasks and modules
* [ ] Handlers
* [ ] Tags
* [ ] Registers (capturing output)
* [ ] Conditions (when)
* [ ] Loops (loop)
* [ ] Delegation (delegate_to, run_once)
* [ ] Idempotency concept

## 5. Core Modules for Configuration Management

### Package Management

* [ ] yum, dnf, apt, package module

### Service Management

* [ ] service, systemd

### File & Config Management

* [ ] copy, template, lineinfile, blockinfile, replace, file

### Command Execution

* [ ] command, shell, raw, script

### User Management

* [ ] user, group, authorized_key

## 6. Templates (Jinja2)

* [ ] Purpose of templates
* [ ] Variables in templates
* [ ] Conditional logic
* [ ] Examples for nginx/application configs

## 7. Variables

* [ ] Variable types
* [ ] Variable precedence
* [ ] vars, vars_files, group_vars, host_vars
* [ ] Using facts
* [ ] Default values

## 8. Handlers

* [ ] How handlers work
* [ ] notify keyword
* [ ] Restarting services only when needed

## 9. Roles

* [ ] Role directory structure
* [ ] tasks/, handlers/, vars/, templates/, files/
* [ ] Reusable roles
* [ ] Organizing large playbooks using roles

## 10. Tags

* [ ] Running specific tasks
* [ ] Skipping tasks
* [ ] Common real-time examples

## 11. Error Handling & Debugging

* [ ] ignore_errors
* [ ] failed_when
* [ ] changed_when
* [ ] debug module
* [ ] Verbosity levels (-vvv)

## 12. Ansible Vault

* [ ] Encrypting files
* [ ] Decrypting and editing
* [ ] Using vault within CI/CD
* [ ] Best practices

## 13. Best Practices (Senior DevOps)

* [ ] Directory layout
* [ ] Use roles everywhere
* [ ] Avoid unnecessary shell commands
* [ ] Template over static files
* [ ] Clean inventory structure
* [ ] Encrypt secrets
* [ ] Git workflow for Ansible projects

## 14. Real-World DevOps Scenarios

* [ ] Install nginx/apache on multiple servers
* [ ] Install Docker
* [ ] Push config files using templates
* [ ] Application deployment on VMs
* [ ] Manage SSH keys/users
* [ ] OS patching
* [ ] Rolling updates (serial)

## 15. Final Project (End-to-End Practical)

* [ ] Inventory with 3–5 EC2 servers
* [ ] Install nginx using playbook
* [ ] Nginx config template with handler
* [ ] Deploy application config templates
* [ ] Create users and SSH keys
* [ ] Use ansible vault
* [ ] Convert everything into roles
* [ ] Build a production-like folder structure

## Optional Advanced Concepts (Nice to Know)

* [ ] Custom facts and fact filtering
* [ ] Strategies (linear, free, serial)
* [ ] Ansible pull mode
* [ ] Collections (community.general, community.aws)
* [ ] Advanced vault usage (env-specific)
* [ ] Testing with ansible-lint and molecule

