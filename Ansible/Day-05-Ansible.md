# Package Management (Ansible)

---

## Concept / What

Package Management in Ansible refers to managing **OS-level software packages** (installing, updating, or removing) on managed nodes using **Ansible modules inside playbooks**.
Ansible works in a **declarative and idempotent** way, where we define the *desired state* of a package rather than running imperative install commands.

---

## Why / Purpose / Real Use Case

Package management is a core part of configuration management because:

* Servers must have consistent software versions
* Manual installation does not scale
* Shell-based installs break idempotency
* Configuration drift must be avoided

**Real-world use cases:**

* Install `nginx` on all web servers
* Ensure Java is installed on application servers
* Remove insecure packages (e.g., `telnet`) for compliance
* Perform OS patching during maintenance windows

In production, package modules are used in **base roles**, **application roles**, and **OS hardening playbooks**.

---

## How it Works / Steps / Syntax

Ansible provides **OS-specific package modules** and a **generic wrapper module**.

### 1. Generic `package` Module (Preferred)

The `package` module acts as a **wrapper** over OS package managers such as `apt`, `yum`, and `dnf`.
It automatically selects the correct backend based on the target system.

```yaml
- name: Install nginx
  package:
    name: nginx
    state: present
```

**Explanation:**

* `package:` → Generic package management module
* `name` → Package to manage
* `state: present` → Ensures the package is installed (idempotent)

**Why preferred:**

* Works across multiple OS types
* Ideal for roles and reusable playbooks

---

### 2. Installing Multiple Packages

```yaml
- name: Install required packages
  package:
    name:
      - git
      - curl
      - vim
    state: present
```

Used commonly during:

* Server bootstrap
* Base OS configuration

---

### 3. Removing Packages

```yaml
- name: Remove insecure package
  package:
    name: telnet
    state: absent
```

Used in:

* Security hardening
* Compliance remediation

---

### 4. Updating Packages (Controlled Usage)

```yaml
- name: Update nginx to latest version
  package:
    name: nginx
    state: latest
```

Used only during:

* Planned patching
* Maintenance windows

---

### 5. OS-Specific Package Modules

#### `apt` (Ubuntu / Debian)

Used when apt-specific features are required (e.g., updating cache).

```yaml
- name: Install nginx on Ubuntu
  apt:
    name: nginx
    state: present
    update_cache: yes
```

* `update_cache` refreshes package metadata before installation

---

#### `yum` (RHEL 7, CentOS 7, Amazon Linux 2)

```yaml
- name: Install httpd on RHEL 7
  yum:
    name: httpd
    state: present
```

Used in legacy RHEL-based environments.

---

#### `dnf` (RHEL 8+, CentOS Stream)

```yaml
- name: Install httpd on RHEL 8
  dnf:
    name: httpd
    state: present
```

Used in modern RHEL-based systems.

---

## Common Issues / Errors

* Package not found due to outdated repository cache
* Accidental upgrades caused by misuse of `state: latest`
* Using shell commands instead of package modules
* Mixing package and service concepts

---

## Troubleshooting / Fixes

* Use `package` module by default
* Use OS-specific modules only when required
* Avoid `state: latest` unless patching is intentional
* Ensure repositories are properly configured

---

## Best Practices

* Prefer `package` module for portability
* Keep install and upgrade logic separate
* Never use shell commands for package installation
* Use roles to manage packages consistently
* Always think in terms of **desired state**

---
---

# Service Management (Ansible)

---

## Concept / What

Service Management in Ansible refers to **controlling the runtime state of services** on managed nodes using Ansible modules inside playbooks.
This includes starting, stopping, restarting, reloading services and enabling or disabling them at system boot.

Service management is **different from package management**:

* Package modules install software
* Service modules manage already-installed services

---

## Why / Purpose / Real Use Case

After installing a package, its service must be properly managed to make the application usable and reliable.

Service management is required to:

* Ensure services are running
* Enable services to start automatically after reboot
* Restart or reload services after configuration changes
* Stop services during maintenance or patching

**Real-world use cases:**

* Start `nginx` after installation
* Restart `nginx` when configuration changes
* Ensure `docker` starts on reboot
* Reload services to apply config changes without downtime

---

## How it Works / Steps / Syntax

Ansible provides two main modules for service management:

| Module    | Purpose                                        |
| --------- | ---------------------------------------------- |
| `service` | Generic, OS-agnostic service management        |
| `systemd` | Systemd-specific, recommended for modern Linux |

---

### 1. `service` Module (Generic)

The `service` module provides a common interface for managing services across different init systems.

```yaml
- name: Start and enable nginx service
  service:
    name: nginx
    state: started
    enabled: yes
```

**Explanation:**

* `name` → Service name
* `state: started` → Ensures service is running
* `enabled: yes` → Ensures service starts on boot

---

### 2. `systemd` Module (Preferred)

Modern Linux distributions use **systemd** as the init system.
The `systemd` module provides more reliable and advanced control.

```yaml
- name: Start and enable nginx using systemd
  systemd:
    name: nginx
    state: started
    enabled: yes
```

**Key point:**

* Ansible does **not** run `systemctl` directly
* The `systemd` module internally interacts with systemd in an idempotent way

---

### 3. Restarting Services

```yaml
- name: Restart nginx service
  systemd:
    name: nginx
    state: restarted
```

Used when:

* Configuration files change
* Application upgrades are deployed

---

### 4. Reload vs Restart

#### Reload (Preferred when supported)

```yaml
- name: Reload nginx service
  systemd:
    name: nginx
    state: reloaded
```

* Reloads configuration
* Does not stop the service
* Avoids downtime

#### Restart

* Stops and starts the service
* May cause brief downtime

---

## Common Issues / Errors

* Service fails to start because package is not installed
* Service name mismatch between package and service
* Restarting services unnecessarily
* Using shell commands instead of modules

---

## Troubleshooting / Fixes

* Ensure package installation task runs before service management
* Validate configuration files before restarting services
* Prefer `systemd` module on modern Linux
* Use handlers for service restarts (best practice)

---

## Best Practices

* Always install packages before managing services
* Enable services that must survive reboots
* Prefer reload over restart when supported
* Use handlers to restart services only when required
* Avoid using `shell` or `command` for service control

---
---

# File & Config Management — `file` Module (Ansible)

---

## Concept / What

The `file` module in Ansible is used to **manage the existence and metadata of files and directories** on managed nodes.
It can create or delete files and directories, change ownership, group, permissions, and manage symbolic links.

The `file` module **does not manage file content**. It only controls *whether something exists* and *how it is owned and permitted*.

---

## Why / Purpose / Real Use Case

In real-world systems, applications depend on:

* Pre-existing directories
* Correct ownership and permissions
* Predictable filesystem structure

Manual commands like `mkdir`, `chmod`, and `chown` do not scale across many servers and are error-prone.

**Real-world use cases:**

* Creating application directories before deployment
* Preparing log directories with correct permissions
* Ensuring config paths exist before copying files
* Cleaning up old directories during decommissioning

The `file` module ensures **consistency, idempotency, and scalability**.

---

## How it Works / Steps / Syntax

The behavior of the `file` module is controlled mainly by the `state` parameter.

### Common `state` Values

| State       | Meaning                                    |
| ----------- | ------------------------------------------ |
| `directory` | Ensure a directory exists                  |
| `file`      | Ensure an empty file exists                |
| `touch`     | Create file if missing or update timestamp |
| `link`      | Create a symbolic link                     |
| `absent`    | Remove file or directory                   |

---

### 1. Creating a Directory

```yaml
- name: Create application directory
  file:
    path: /opt/myapp
    state: directory
```

Ensures the directory exists. If it already exists, no change is made.

---

### 2. Directory with Ownership and Permissions

```yaml
- name: Create app directory with permissions
  file:
    path: /opt/myapp
    state: directory
    owner: appuser
    group: appgroup
    mode: '0755'
```

* `owner` → File owner
* `group` → Group ownership
* `mode` → Permissions (must be quoted)

---

### 3. Creating an Empty File

```yaml
- name: Create empty config file
  file:
    path: /etc/myapp.conf
    state: file
```

Used when an application expects a file to exist, but content will be managed later.

---

### 4. Touching a File

```yaml
- name: Touch log file
  file:
    path: /var/log/myapp.log
    state: touch
```

Creates the file if missing or updates the modification time.

---

### 5. Removing Files or Directories

```yaml
- name: Remove old application directory
  file:
    path: /opt/oldapp
    state: absent
```

Removes files or directories recursively. Use carefully.

---

### 6. Creating a Symbolic Link

```yaml
- name: Create symlink for current version
  file:
    src: /opt/myapp/releases/v2
    dest: /opt/myapp/current
    state: link
```

Used in versioned or blue-green style deployments.

---

## Common Issues / Errors

* Forgetting to create directories before copying files
* Incorrect permissions causing application failures
* Not quoting permission modes
* Using shell commands instead of the `file` module

---

## Troubleshooting / Fixes

* Always create directories before using `copy` or `template`
* Explicitly define ownership and permissions
* Quote file modes to avoid YAML parsing issues
* Use Ansible modules instead of shell commands

---

## Best Practices

* Use the `file` module to prepare filesystem structure
* Separate file existence management from content management
* Always define owner, group, and mode explicitly
* Avoid using shell commands like `mkdir` and `chmod`
* Think in terms of desired state, not commands

---
---

# File & Config Management — `copy` Module (Ansible)

---

## Concept / What

The `copy` module in Ansible is used to **copy files or file content from the control node to managed nodes**.
Unlike the `file` module, `copy` is responsible for **managing file content**.

It can copy static files, directories, or inline content, and can also set ownership and permissions during the copy operation.

---

## Why / Purpose / Real Use Case

Applications require:

* Configuration files
* Certificates and keys
* Static assets
* Scripts

Manually copying files to multiple servers:

* Does not scale
* Causes inconsistencies
* Has no change tracking

**Real-world use cases:**

* Copy application configuration files
* Deploy static website assets
* Distribute certificates and scripts
* Push environment-specific config files

The `copy` module ensures **consistency, idempotency, and safe re-runs**.

---

## How it Works / Steps / Syntax

The `copy` module compares checksums of source and destination files.
If the content is unchanged, no action is taken. If content differs, the file is updated.

---

### 1. Copying a File

```yaml
- name: Copy application config file
  copy:
    src: myapp.conf
    dest: /etc/myapp.conf
```

* `src` → File on control node
* `dest` → Destination path on managed node

---

### 2. Copy with Ownership and Permissions

```yaml
- name: Copy config with permissions
  copy:
    src: myapp.conf
    dest: /etc/myapp.conf
    owner: root
    group: root
    mode: '0644'
```

Ownership and permissions are applied **during the copy operation**.

---

### 3. Copying Inline Content

```yaml
- name: Create config using inline content
  copy:
    content: |
      server_name example.com;
      listen 80;
    dest: /etc/nginx/conf.d/example.conf
```

* Inline content writes **full file content**
* Destination file may be empty or already exist
* Existing content is **overwritten**, not appended

---

### 4. Copying Directories

```yaml
- name: Copy static files directory
  copy:
    src: static/
    dest: /var/www/html/
```

Copies directory contents recursively.

---

### 5. Backup Before Overwrite

```yaml
- name: Copy config with backup
  copy:
    src: myapp.conf
    dest: /etc/myapp.conf
    backup: yes
```

* Creates a backup of the existing file
* Backup is stored **in the same directory** as the destination
* Backup filename includes timestamp and suffix

---

## Common Issues / Errors

* Destination directory does not exist
* Incorrect permissions causing application failures
* Using inline content for large or dynamic files
* Expecting inline content to append data

---

## Troubleshooting / Fixes

* Create directories first using the `file` module
* Explicitly define ownership and permissions
* Use `template` for dynamic or variable-based configs
* Enable `backup: yes` for critical files

---

## Best Practices

* Use `copy` only for **static content**
* Use `template` for dynamic configuration files
* Prepare directories using `file` module before copying
* Quote file modes to avoid YAML issues
* Avoid using shell commands like `cp`

---
---

# File & Config Management — `template` Module (Ansible)

---

## Concept / What

The `template` module in Ansible is used to **generate files dynamically on managed nodes** using templates written with **Jinja2 syntax**.
Unlike the `copy` module, `template` allows the file content to change based on **variables, conditions, and per-host values**.

The template file is created on the control node (usually with a `.j2` extension) and rendered separately for **each managed host**.

---

## Why / Purpose / Real Use Case

In real-world environments, configuration files are rarely identical across all servers.

Reasons include:

* Different IP addresses per server
* Different ports or hostnames per environment
* Environment-specific settings (dev / qa / prod)
* Conditional configuration blocks

Using static files would require maintaining multiple copies of the same config.
The `template` module solves this by allowing **one template** to adapt itself dynamically for each server.

**Real-world use cases:**

* Nginx / Apache configuration files
* Application configuration files
* Environment-specific service configs
* Per-host configuration generation

---

## How it Works / Steps / Syntax

1. A template file (`.j2`) is created on the control node.
2. The template contains variables and optional logic.
3. During playbook execution:

   * Ansible gathers variables and facts for each host.
   * Jinja2 renders the template separately for every host.
   * The final rendered file is copied to the destination path.
4. If rendered content changes, Ansible marks the task as `changed`.

---

## Basic Template Example

### Template file (`nginx.conf.j2`)

```jinja2
server {
  listen {{ nginx_port }};
  server_name {{ server_name }};
}
```

### Playbook task

```yaml
- name: Deploy nginx configuration
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
```

The same template produces **different output on different servers** based on variable values.

---

## Where Template Variables Come From

Template variables can come from multiple sources:

### 1. Defined Variables

* Playbook `vars`
* Inventory variables
* `group_vars` / `host_vars`

### 2. Server-Derived Variables (Facts)

* Automatically collected from each managed host
* Different for every server
* Commonly used for:

  * IP addresses
  * Hostnames
  * OS details

Facts allow templates to render **per-host values without manual variable definition**.

---

## Template vs Copy (Correct Decision Rule)

| Scenario                         | Module to Use              |
| -------------------------------- | -------------------------- |
| Static file (any size)           | `copy`                     |
| Small ad-hoc content             | `copy` with inline content |
| Dynamic / variable-based content | `template`                 |

The choice depends on **content dynamism**, not file size.

---

## Ownership and Permissions

```yaml
- name: Deploy config with permissions
  template:
    src: app.conf.j2
    dest: /etc/myapp/app.conf
    owner: root
    group: root
    mode: '0644'
```

Ownership and permissions are applied to the rendered file, similar to the `copy` module.

---

## Common Issues / Errors

* Using `copy` for variable-based configuration files
* Hardcoding environment-specific values
* Forgetting that templates render separately per host
* Overloading templates with complex logic

---

## Troubleshooting / Fixes

* Use facts for per-host values instead of hardcoding
* Keep template logic minimal and readable
* Validate rendered configs when services fail to start
* Use handlers to restart services after template changes

---

## Best Practices

* Use `template` for all configuration files
* Keep one template per service
* Use variables and facts instead of hardcoded values
* Avoid complex logic inside templates
* Combine templates with handlers for safe restarts

---
---

# File & Config Management — `lineinfile` Module (Ansible)

---

## Concept / What

The `lineinfile` module in Ansible is used to **manage a single line in an existing file**.
It ensures that a specific line is **present, modified, or absent**, based on a matching condition.

The module is **line-oriented**, not file-oriented. It does not manage the entire file content.

---

## Why / Purpose / Real Use Case

In real systems, sometimes only **one configuration line** needs to be changed without touching the rest of the file.

Using `copy` or `template` in such cases may overwrite other settings.

**Real-world use cases:**

* Change SSH port
* Disable root login
* Modify SELinux mode
* Remove insecure configuration entries

The `lineinfile` module allows **safe, minimal, and idempotent changes**.

---

## How it Works / Steps / Syntax

The module works by:

1. Reading the target file
2. Searching for a line using an optional regular expression (`regexp`)
3. Taking action based on whether a match is found

---

### 1. Ensure a Line Exists

```yaml
- name: Ensure SELinux is disabled
  lineinfile:
    path: /etc/selinux/config
    line: SELINUX=disabled
```

* If the exact line exists → no change
* If it does not exist → the line is added

---

### 2. Replace an Existing Line Using `regexp`

```yaml
- name: Set SELinux to disabled
  lineinfile:
    path: /etc/selinux/config
    regexp: '^SELINUX='
    line: SELINUX=disabled
```

* `regexp` finds the matching line
* The matched line is replaced with the value in `line`

---

### 3. Remove a Line

```yaml
- name: Remove root login configuration
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    state: absent
```

* Any matching line is removed from the file

---

### 4. Insert Line at a Specific Location

```yaml
- name: Add timeout setting after a section
  lineinfile:
    path: /etc/myapp.conf
    line: timeout=30
    insertafter: '^connection_settings'
```

Other option:

* `insertbefore`

---

## Understanding `regexp`

* `regexp` stands for **regular expression**
* It is used to **search for matching lines**, similar to how `sed` works conceptually
* Example:

  ```yaml
  regexp: '^SELINUX='
  ```

  Matches any line starting with `SELINUX=`

---

## Common Issues / Errors

* Forgetting to use `regexp`, leading to duplicate lines
* Writing incorrect regular expressions
* Using `lineinfile` for many lines or complex configs
* Forgetting to restart services after configuration changes

---

## Troubleshooting / Fixes

* Always use `regexp` when replacing existing settings
* Test regular expressions carefully
* Use handlers to restart services after changes
* Switch to `template` for large or structured configurations

---

## Best Practices

* Use `lineinfile` only for single-line changes
* Always pair `regexp` with `line` for replacements
* Avoid using it for complex configuration files
* Prefer declarative modules over shell commands
* Think in terms of **ensuring state**, not executing commands

---
---

# File & Config Management — `blockinfile` Module (Ansible)

---

## Concept / What

The `blockinfile` module in Ansible is used to **manage a continuous block of multiple lines** inside an existing file.
It inserts, updates, or removes a block of text that Ansible clearly marks as **managed**.

Unlike `lineinfile`, which works on a single line, `blockinfile` always manages **a group of lines together at one location**.

---

## Why / Purpose / Real Use Case

In real systems, configuration changes often involve **multiple related lines** that must stay together.

Using `lineinfile` repeatedly for such cases:

* Becomes hard to manage
* Makes intent unclear

**Real-world use cases:**

* Adding multiple application settings
* Appending managed configuration sections
* Injecting environment-specific blocks
* Managing safe, isolated config sections

The `blockinfile` module allows Ansible to **own only a specific block**, without touching the rest of the file.

---

## How it Works / Steps / Syntax

1. Ansible inserts the block wrapped between **begin and end markers**.
2. On subsequent runs, Ansible searches for those markers.
3. If the block exists:

   * It updates the content inside the block
4. If the block does not exist:

   * It inserts the block at the specified location

This behavior makes the module **idempotent and safe**.

---

## Basic Example

```yaml
- name: Add application configuration block
  blockinfile:
    path: /etc/myapp.conf
    block: |
      timeout=30
      retries=5
      log_level=INFO
```

* All lines are inserted **together**
* The block is treated as **one unit**

---

## Block Placement

```yaml
- name: Insert block after application section
  blockinfile:
    path: /etc/myapp.conf
    block: |
      feature_enabled=true
      max_connections=100
    insertafter: '^# Application settings'
```

Options:

* `insertafter`
* `insertbefore`

The block is always placed at **one single location**.

---

## Markers (Important)

By default, Ansible adds markers like:

```text
# BEGIN ANSIBLE MANAGED BLOCK
# END ANSIBLE MANAGED BLOCK
```

These markers allow Ansible to:

* Identify the block
* Update or remove it safely

Markers can be customized if required.

---

## Removing a Block

```yaml
- name: Remove managed block
  blockinfile:
    path: /etc/myapp.conf
    state: absent
```

Only the managed block is removed; the rest of the file remains untouched.

---

## Important Rules to Remember

* One `blockinfile` task manages **one continuous block**
* A single block cannot span multiple unrelated positions
* Multiple blocks require **multiple tasks**
* One task can use **only one module**

---

## Common Issues / Errors

* Trying to manage multiple blocks in one task
* Expecting a block to be split across different positions
* Using `blockinfile` for structured configs (YAML, JSON, nginx)
* Forgetting where the block is inserted

---

## Troubleshooting / Fixes

* Use separate tasks for separate blocks
* Use `template` when you own the full file
* Use `lineinfile` for single-line changes
* Verify insert position with regex carefully

---

## Best Practices

* Use `blockinfile` for grouped, related settings
* Keep blocks small and readable
* Let Ansible manage only what it owns
* Avoid mixing manual edits inside managed blocks
* Follow the rule: **one task = one module = one intent**

---
---

# File & Config Management — `replace` Module (Ansible)

---

## Concept / What

The `replace` module in Ansible is used to **search for text patterns in a file and replace all matching occurrences** with new content.
It operates on the **entire file** and uses **regular expressions (regex)** to identify what needs to be replaced.

Unlike `lineinfile`, which targets a single logical line, `replace` performs **bulk search-and-replace** operations.

---

## Why / Purpose / Real Use Case

There are scenarios where the same configuration value appears **multiple times** in a file and must be updated consistently.

Using `lineinfile` in such cases:

* Is insufficient
* Requires multiple tasks

**Real-world use cases:**

* Replace all `http://` with `https://`
* Update deprecated configuration values everywhere
* Change ports or paths appearing multiple times
* Perform safe configuration migrations

The `replace` module allows these changes to be made **declaratively and idempotently**.

---

## How it Works / Steps / Syntax

1. Ansible reads the target file.
2. It searches for all matches of the given regular expression (`regexp`).
3. Every match is replaced with the provided replacement value.
4. If no match is found, no changes are made.

---

## Basic Example

```yaml
- name: Replace http with https
  replace:
    path: /etc/myapp.conf
    regexp: 'http://'
    replace: 'https://'
```

All occurrences of `http://` are replaced with `https://`.

---

## Replacing Repeated Configuration Values

```yaml
- name: Change application port everywhere
  replace:
    path: /etc/myapp.conf
    regexp: 'port=8080'
    replace: 'port=9090'
```

Used when the same setting appears multiple times in a file.

---

## Key Difference: `replace` vs `lineinfile`

| Requirement                   | Module       |
| ----------------------------- | ------------ |
| Replace a single logical line | `lineinfile` |
| Replace all matching patterns | `replace`    |

This distinction is crucial for choosing the correct module.

---

## What `replace` Is Good At

* Global replacements
* Updating repeated patterns
* Simple migrations
* Bulk configuration changes

---

## What `replace` Is Not Good At

* Managing structured configuration files (YAML, JSON)
* Ensuring a line exists
* Fine-grained single-line control

In such cases, use `template` or `lineinfile` instead.

---

## Common Issues / Errors

* Writing overly broad regular expressions
* Replacing unintended content
* Using `replace` when only one line should change
* Applying `replace` on strictly ordered or structured configs

---

## Troubleshooting / Fixes

* Test regular expressions carefully
* Narrow regex patterns to avoid accidental replacements
* Prefer `lineinfile` for single-line changes
* Use `template` when full-file control is required

---

## Best Practices

* Use `replace` only for true bulk replacements
* Keep regex patterns specific and safe
* Avoid using `replace` on structured config formats
* Always think in terms of **scope of change** before choosing the module

---
---

# Command Execution Modules (Ansible)

---

## Concept / What

Command execution modules in Ansible are used to **run commands or scripts directly on managed nodes**.
These modules are intended as a **last resort**, to be used only when no suitable Ansible module exists for the required task.

Modules covered:

* `command`
* `shell`
* `raw`
* `script`

---

## Why / Purpose / Real Use Case

Ansible is designed to work using **idempotent modules** rather than imperative commands.

However, command execution modules are required when:

* No native Ansible module exists
* Performing one-time operational tasks
* Bootstrapping minimal systems
* Running legacy scripts

**Real-world scenarios:**

* Checking software versions
* Running temporary diagnostics
* Installing Python on minimal servers
* Executing existing shell scripts

---

## How it Works / Steps / Syntax

Each module executes commands in a different way and has its own scope and limitations.

---

## `command` Module

### Purpose

Executes a command **directly**, without using a shell.

```yaml
- name: Check Java version
  command: java -version
```

### Key Characteristics

* Does **not** support shell features
* No pipes (`|`), redirects (`>`), variables, or conditionals
* Safer and more predictable than `shell`

---

## `shell` Module

### Purpose

Executes a command **through a shell**.

```yaml
- name: Find nginx process
  shell: ps -ef | grep nginx
```

### Key Characteristics

* Supports shell features such as pipes and redirection
* Less safe than `command`
* Harder to keep idempotent

---

## `raw` Module

### Purpose

Executes commands **directly over SSH**, without using Ansible modules or Python.

```yaml
- name: Install python on minimal system
  raw: yum install -y python3
```

### Key Characteristics

* Does **not require Python** on the managed node
* Used mainly for bootstrapping systems
* No idempotency guarantees

---

## `script` Module

### Purpose

Copies a script from the control node to the managed node and executes it.

```yaml
- name: Run cleanup script
  script: cleanup.sh
```

### Key Characteristics

* Script is copied temporarily
* Executed on the managed node
* Removed after execution

---

## Module Comparison

| Module    | Uses Shell | Requires Python | Typical Use Case                |
| --------- | ---------- | --------------- | ------------------------------- |
| `command` | No         | Yes             | Simple, safe command execution  |
| `shell`   | Yes        | Yes             | Commands needing shell features |
| `raw`     | No         | No              | Bootstrapping minimal systems   |
| `script`  | Depends    | Yes             | Running existing scripts        |

---

## Common Issues / Errors

* Overusing shell commands instead of modules
* Writing non-idempotent logic
* Using `shell` when `command` is sufficient
* Forgetting Python dependency when not using `raw`

---

## Troubleshooting / Fixes

* Always check if a native Ansible module exists
* Prefer `command` over `shell`
* Use `raw` only for initial setup
* Gradually replace scripts with Ansible modules

---

## Best Practices

* Treat command execution modules as a **last option**
* Avoid complex logic in shell commands
* Keep tasks simple and readable
* Document why a command module is used
* Prefer declarative modules whenever possible

---
---

# User Management Modules (Ansible)

---

## Concept / What

User Management in Ansible refers to **managing Linux users, groups, and SSH access** on managed nodes using dedicated Ansible modules.
These modules allow consistent, secure, and repeatable control over **who can access servers and how**.

Modules covered:

* `group`
* `user`
* `authorized_key`

---

## Why / Purpose / Real Use Case

In real-world environments:

* Users join and leave teams
* Access must be granted and revoked safely
* Manual user and SSH key management does not scale

**Real-world use cases:**

* Creating application users
* Assigning users to groups
* Managing sudo access
* Managing SSH key-based authentication
* Removing users when they leave the organization

Using Ansible ensures **security, auditability, and consistency**.

---

## How it Works / Steps / Syntax

A typical user-management flow follows this order:

1. Create required groups
2. Create users and assign them to groups
3. Configure SSH access using public keys

---

## `group` Module

### Purpose

Manages Linux groups.

```yaml
- name: Create application group
  group:
    name: appgroup
    state: present
```

Used to define shared permissions and access control.

---

## `user` Module

### Purpose

Manages Linux users.

```yaml
- name: Create application user
  user:
    name: appuser
    group: appgroup
    groups: sudo
    shell: /bin/bash
    state: present
```

### Key Capabilities

* Create or remove users
* Assign primary and secondary groups
* Set login shell
* Manage home directory

---

## Removing Users Safely

```yaml
- name: Remove old user
  user:
    name: olduser
    state: absent
    remove: yes
```

* `remove: yes` deletes the user’s home directory

---

## `authorized_key` Module

### Purpose

Manages SSH public keys in a user’s `authorized_keys` file.

```yaml
- name: Add SSH key for appuser
  authorized_key:
    user: appuser
    state: present
    key: "{{ lookup('file', 'appuser.pub') }}"
```

This controls **who can SSH into the user account**.

---

## Understanding `lookup('file', ...)`

* `lookup` is an Ansible function
* It runs **on the control node**, not on managed nodes
* `lookup('file', 'appuser.pub')`:

  * Reads the contents of the public key file on the control node
  * Passes the key content to the module

`lookup` does **not copy files**; it only fetches values for use in tasks.

---

## SSH Key Model (Important Concept)

* **Private key**:

  * Stored with the user or system initiating SSH
  * In AWS, stored in the `.pem` file
* **Public key**:

  * Stored on the target server
  * Located in `~/.ssh/authorized_keys`

During EC2 creation, AWS copies the **public key** into the instance when a key pair is selected.

---

## Common Issues / Errors

* Creating users before groups
* Managing SSH access manually
* Losing private keys (`.pem` files)
* Hardcoding public keys inside playbooks

---

## Troubleshooting / Fixes

* Always create groups before users
* Store public keys on the control node
* Use `authorized_key` instead of manual edits
* Revoke access by removing keys or users

---

## Best Practices

* Always manage users and groups via playbooks
* Use SSH keys instead of passwords
* Never share private keys
* Use `authorized_key` for SSH access control
* Remove users and keys immediately when access is no longer required

---
---
---
