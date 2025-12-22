# Inventory Management – Static Inventory (INI & YAML)

---

## Concept / What

In Ansible, an **inventory** defines the **managed nodes** (servers, VMs, instances) that Ansible will control. A **static inventory** is one where hosts are **manually defined** and remain mostly fixed.

Static inventory can be written in two formats:

* **INI** (traditional, simple)
* **YAML** (structured, scalable, preferred in real projects)

The inventory does not just store IPs or DNS names; it also defines **logical hostnames, groups, and connection variables**.

---

## Why / Purpose / Real Use Case

Static inventory is used when:

* Servers are long-running and do not change frequently
* Environments are small to medium in size
* You want explicit control over hosts and grouping
* Learning or bootstrapping Ansible

### Real-world scenarios

* Fixed EC2 instances in dev/qa/prod
* On-premise servers
* Bastion hosts and database servers
* Legacy environments without auto-scaling

Static inventory is predictable, transparent, and easy to debug.

---

## How it Works / Steps / Syntax

### 1. Static Inventory Using INI Format

```ini
[web]
web1 ansible_host=10.0.0.10
web2 ansible_host=10.0.0.11

[db]
db1 ansible_host=10.0.0.20

[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=/home/ec2-user/key.pem
```

**Explanation:**

* `web1`, `db1` → Logical hostnames used by Ansible
* `ansible_host` → Actual IP/DNS of the server
* `[web]`, `[db]` → Host groups
* `[all:vars]` → Variables applied to all hosts

INI format is simple but becomes hard to manage as complexity grows.

---

### 2. Static Inventory Using YAML Format

```yaml
all:
  children:
    web:
      hosts:
        web1:
          ansible_host: 10.0.0.10
        web2:
          ansible_host: 10.0.0.11
    db:
      hosts:
        db1:
          ansible_host: 10.0.0.20
  vars:
    ansible_user: ec2-user
    ansible_ssh_private_key_file: /home/ec2-user/key.pem
```

**Explanation:**

* `all` → Parent group (exists implicitly even if not defined)
* `children` → Defines sub-groups
* `hosts` → Defines actual managed nodes
* `vars` → Variables applied at group level

YAML inventory explicitly shows hierarchy and inheritance.

---

## Are `all`, `children`, and `hosts` Mandatory?

**No, they are not always mandatory.**

### Minimal valid YAML inventory (no `all`, no `children`):

```yaml
web:
  hosts:
    web1:
      ansible_host: 10.0.0.10
```

Ansible automatically places this under the `all` group internally.

### When they are required:

* `all` → Needed when defining global variables or parent groups
* `children` → Needed when creating group hierarchy
* `hosts` → Needed when a group directly contains servers

They are used for **clarity, hierarchy, and variable inheritance**, not because Ansible forces them.

---

## Common Issues / Errors

* Incorrect YAML indentation causing inventory parsing errors
* Confusion between logical hostname and IP address
* Forgetting to define `ansible_host`
* Typos in group names leading to "no hosts matched"

---

## Troubleshooting / Fixes

Validate inventory structure:

```bash
ansible-inventory -i inventory.yml --list
```

Test connectivity:

```bash
ansible all -i inventory.yml -m ping
```

Increase verbosity:

```bash
ansible-playbook playbook.yml -i inventory.yml -vvv
```

---

## Best Practices

* Prefer **YAML inventory** for real projects
* Use **logical hostnames**, not raw IPs
* Use groups instead of targeting individual hosts
* Keep inventory separate from playbooks
* Move variables to `group_vars/` and `host_vars/` for scalability
* Do not commit SSH private keys to Git

---

## Key Takeaway

Static inventory is a manually maintained definition of hosts, groups, and variables. INI is simple, YAML is structured and scalable. Keywords like `all`, `children`, and `hosts` are not mandatory but are used to clearly define hierarchy and inheritance in real-world Ansible projects.

---
---

# Inventory Management – Host Groups

---

## Concept / What

In Ansible, **host groups** are **logical collections of managed nodes** defined inside the inventory. A group represents servers that share a **common role, tier, or environment**.

Ansible identifies hosts using **logical hostnames**, and groups are simply labels that tell Ansible **which hosts should be targeted together**.

A single host can belong to **multiple groups** at the same time.

---

## Why / Purpose / Real Use Case

Host groups are used to:

* Run the same configuration on multiple servers
* Avoid hardcoding IP addresses in playbooks
* Model real-world infrastructure cleanly
* Share configuration and variables across servers

### Real-world grouping examples

* `web`, `db`, `app` → application tiers
* `frontend`, `backend` → architecture layers
* `dev`, `qa`, `prod` → environments

Groups reflect **how teams think about infrastructure**, not how servers are physically created.

---

## How it Works / Steps / Syntax

### 1. Host Groups in INI Inventory

```ini
[web]
web1 ansible_host=10.0.0.10
web2 ansible_host=10.0.0.11

[db]
db1 ansible_host=10.0.0.20
```

**Explanation:**

* `web`, `db` are group names
* `web1`, `db1` are logical hostnames
* `ansible_host` maps the hostname to IP/DNS

---

### 2. Reusing a Host in Multiple Groups

```ini
[web]
web1 ansible_host=10.0.0.10

[prod]
web1
```

**Key concept:**

* Host connection details are defined **once**
* The hostname can be reused in multiple groups
* Groups do not redefine the host, they only label it

Ansible treats the hostname as the **unique identity**, not the IP address.

---

### 3. Host Groups in YAML Inventory

```yaml
web:
  hosts:
    web1:
      ansible_host: 10.0.0.10

prod:
  hosts:
    web1: {}
```

This is functionally identical to the INI example.

---

### 4. Using Host Groups in Playbooks

```yaml
- name: Install nginx on web servers
  hosts: web
  become: true
  tasks:
    - name: Install nginx
      yum:
        name: nginx
        state: present
```

Here, Ansible runs the play on **all hosts in the `web` group**.

---

## Common Issues / Errors

* Group name mismatch causing `no hosts matched`
* Redefining the same host with different IPs in multiple groups
* Hardcoding IP addresses in playbooks
* Assuming group order affects execution

---

## Troubleshooting / Fixes

Check resolved inventory:

```bash
ansible-inventory -i inventory.yml --list
```

List hosts in a group:

```bash
ansible web -i inventory.yml --list-hosts
```

Increase verbosity for debugging:

```bash
ansible-playbook playbook.yml -i inventory.yml -vvv
```

---

## Best Practices

* Define host connection details only once
* Reuse hostnames across multiple groups
* Group servers by role or responsibility
* Use groups in playbooks instead of IPs
* Keep group names meaningful and simple

---

## Key Takeaway

Host groups are logical labels that allow Ansible to target and manage servers efficiently. Hosts are identified by their logical names, and groups simply describe how those hosts are related.

---
---

# Inventory Management – Host & Group Variables

---

## Concept / What

In Ansible, **variables** are key–value pairs used to customize how playbooks behave for different servers. When variables are defined at the **inventory level**, they are called **inventory variables**.

There are two core inventory variable types:

* **Host variables** – apply to a single specific host
* **Group variables** – apply to all hosts inside a group

Ansible identifies hosts by their **logical hostnames**, and variables are resolved based on where they are defined.

---

## Why / Purpose / Real Use Case

Host and group variables allow:

* One playbook to work across multiple environments
* Environment-specific configuration without duplicating playbooks
* Clean separation between logic (playbooks) and data (values)

### Real-world examples

* Different ports per host
* Same application user across all web servers
* Environment-specific settings (dev vs prod)
* Different package versions or paths

Variables are the foundation of **reusability and scalability** in Ansible.

---

## How it Works / Steps / Syntax

### 1. Host Variables (INI Inventory)

**Rule:** If a variable is written on the same line as a host, it applies only to that host.

```ini
[web]
web1 ansible_host=10.0.0.10 http_port=80
web2 ansible_host=10.0.0.11 http_port=8080
```

Here:

* `http_port=80` applies only to `web1`
* `http_port=8080` applies only to `web2`

---

### 2. Group Variables (INI Inventory)

Group variables are defined using a **special `group:vars` section**.

```ini
[web]
web1 ansible_host=10.0.0.10
web2 ansible_host=10.0.0.11

[web:vars]
nginx_user=nginx
app_env=production
```

**Important points:**

* `[web]` defines the group membership
* `[web:vars]` defines variables for the group
* Variables under `[web:vars]` apply to **all hosts in `web`**

Hosts are **not repeated** inside the vars section.

---

### 3. Host & Group Variables in YAML Inventory

YAML makes the structure more explicit and readable.

```yaml
web:
  vars:
    nginx_user: nginx
    app_env: production
  hosts:
    web1:
      ansible_host: 10.0.0.10
    web2:
      ansible_host: 10.0.0.11
```

**Plain English meaning:**

* Define variables for the `web` group
* Apply them to every host inside the group

---

### 4. Using Variables in Playbooks

```yaml
- name: Example using group and host variables
  hosts: web
  tasks:
    - name: Show HTTP port
      debug:
        msg: "Port is {{ http_port }}"
```

Ansible automatically resolves:

* Host variables first
* Then group variables

---

## Variable Precedence (High-Level)

When the same variable is defined in multiple places:

```
host variables > group variables > all variables
```

This means host-specific values override group values.

---

## Common Issues / Errors

* Defining group variables under the wrong section
* Expecting group variables to override host variables
* Variable name mismatches (`httpPort` vs `http_port`)
* Overloading inventory files with too many variables

---

## Troubleshooting / Fixes

Inspect resolved inventory and variables:

```bash
ansible-inventory -i inventory.yml --list
```

Debug variable values:

```yaml
- debug:
    var: http_port
```

Increase verbosity for deeper inspection:

```bash
ansible-playbook playbook.yml -vvv
```

---

## Best Practices

* Keep playbooks free of environment-specific values
* Use group variables for shared configuration
* Use host variables only when truly required
* Prefer `group_vars/` and `host_vars/` directories in real projects
* Use clear and consistent variable naming

---

## Key Takeaway

Host variables customize individual servers, while group variables apply shared configuration across multiple servers. Groups define **scope**, and variables define **behavior**.

---
---

# Inventory Management – Inventory Best Practices

---

## Concept / What

Inventory best practices define **how Ansible inventory should be structured and maintained** so that it remains **clean, scalable, reusable, and safe** as environments grow.

They are not enforced rules, but **production-proven conventions** used by experienced teams working with Ansible.

Inventory should clearly describe:

* What servers exist
* How they are logically grouped
* How Ansible connects to them

---

## Why / Purpose / Real Use Case

Without best practices:

* Inventory files become messy and hard to read
* IPs and variables get hardcoded everywhere
* Debugging becomes difficult
* Scaling to new environments is painful

With best practices:

* One inventory can support many environments
* Playbooks remain reusable and clean
* Changes are isolated to inventory only
* Teams can collaborate safely

Inventory best practices are what differentiate **learning Ansible** from **using Ansible in production**.

---

## How it Works / Steps / Guidelines

### 1. Use Logical Hostnames Instead of Raw IPs

❌ Avoid using IPs directly as inventory hostnames:

```ini
[web]
10.0.0.10
```

✅ Prefer logical hostnames with connection mapping:

```ini
[web]
web1 ansible_host=10.0.0.10
```

**Why:**

* Logical hostnames act as stable identities
* IP addresses can change
* Only inventory needs to be updated when IPs change

---

### 2. Understand When `ansible_host` Is Required

`ansible_host` is an **Ansible inventory variable**, not something that exists in EC2 or on the server.

**Rule:**

* If the inventory hostname is **not** the real IP or DNS, you must define `ansible_host`
* If the inventory hostname **is** the IP or DNS, `ansible_host` is not required

Examples:

No `ansible_host` needed:

```ini
[web]
10.0.0.10
```

```ini
[web]
web1.prod.company.com
```

`ansible_host` required:

```ini
[web]
web1 ansible_host=10.0.0.10
```

---

### 3. Always Use Groups in Playbooks

❌ Bad practice:

```yaml
hosts: web1
```

✅ Best practice:

```yaml
hosts: web
```

**Why:**

* Improves scalability
* Avoids hardcoding
* Makes playbooks reusable

---

### 4. Keep Inventory and Playbooks Separate

Inventory defines **WHERE** tasks run.
Playbooks define **WHAT** tasks run.

Never mix IPs or hostnames directly inside playbooks.

---

### 5. Avoid Duplicating Variables

❌ Bad:

```ini
web1 http_port=80
web2 http_port=80
```

✅ Good:

```ini
[web:vars]
http_port=80
```

**Why:**

* Reduces duplication
* Minimizes mistakes
* Easier maintenance

---

### 6. Use `group_vars/` and `host_vars/` in Real Projects

Recommended structure:

```text
inventory/
  hosts.yml
group_vars/
  web.yml
  prod.yml
host_vars/
  web1.yml
```

**Why:**

* Cleaner inventory files
* Better Git diffs
* Easier scaling

---

### 7. Never Store Secrets in Inventory

❌ Do not store passwords or keys:

```ini
db_password=MySecret123
```

✅ Use:

* Ansible Vault
* External secret managers

Inventory should be safe to read and share.

---

## Common Issues / Errors

* Using raw IPs everywhere
* Redefining the same host multiple times
* Mixing environment variables into inventory
* Storing secrets in plain text
* Inconsistent naming conventions

---

## Troubleshooting / Fixes

Validate inventory:

```bash
ansible-inventory -i inventory.yml --list
```

List hosts in a group:

```bash
ansible web -i inventory.yml --list-hosts
```

Debug variable sources:

```bash
ansible-playbook playbook.yml -vvv
```

---

## Best Practices Summary

* Use logical hostnames
* Map IP/DNS using `ansible_host` when required
* Use groups instead of single hosts
* Keep inventory minimal and readable
* Separate variables from inventory
* Never store secrets in inventory

---

## Key Takeaway

Inventory should describe infrastructure identity and grouping, while connection details and variables are cleanly separated. This ensures minimal change, maximum reuse, and production safety.

---
---
---
