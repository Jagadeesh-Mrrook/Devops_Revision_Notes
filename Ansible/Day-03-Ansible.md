# Ansible Ad-hoc Commands — Ping All Servers

## Concept / What

Ansible **ping** in ad-hoc commands is a **readiness and connectivity check**, not a traditional network (ICMP) ping. It validates that Ansible can connect to managed nodes via **SSH**, authenticate successfully, execute **Python**, and run an Ansible module on the target hosts. A successful response (`pong`) confirms the node is ready for Ansible operations.

In ad-hoc mode, Ansible executes **exactly one module** at a time. The `ping` module is commonly used as the first validation step before running any real configuration tasks or playbooks.

---

## Why / Purpose / Real Use Case

This check is used as a **first-line verification** in real-world environments:

* After adding new servers to the inventory
* After rotating SSH keys or changing users
* After modifying sudo or privilege escalation settings
* Before running large or risky playbooks
* To quickly identify unreachable or misconfigured hosts

If `ansible -m ping` fails, **no playbook will work reliably**, so this step prevents wasted debugging time later.

---

## How it Works / Steps / Syntax

### Basic ad-hoc ping command

```bash
ansible all -m ping
```

**Explanation:**

* `ansible` → Ansible CLI tool
* `all` → host pattern (targets all inventory hosts)
* `-m ping` → specifies the module to execute

Here, `-m` is used to **select the module**. In ad-hoc commands, modules are never passed as arguments; they are explicitly chosen using `-m`.

A successful output:

```text
web1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

This means:

* SSH connection succeeded
* Authentication worked
* Python is available
* The Ansible module executed correctly

---

### Ping a specific group

```bash
ansible web -m ping
```

Used when you want to validate only a subset of servers (for example, web nodes only).

---

### Ping with a specific SSH user

```bash
ansible all -m ping -u ec2-user
```

Used when:

* `ansible_user` is not defined in inventory
* Different environments use different SSH users

---

### Ping with privilege escalation

```bash
ansible all -m ping -b
```

Used to confirm that **sudo / become** works correctly on the target nodes.

---

### Important clarification about `-m` and `-a`

* `-m` → selects **which module** Ansible runs (e.g., `ping`, `command`, `yum`)
* `-a` → passes **arguments to the selected module**, not other modules

Example:

```bash
ansible all -m command -a "uptime"
```

Here:

* `command` is the module
* `uptime` is the argument passed to that module using `-a`

If `-m` is omitted, Ansible defaults to the `command` module, but for all other modules, `-m` is mandatory.

---

## Common Issues / Errors

### UNREACHABLE error

```text
UNREACHABLE! => Failed to connect to the host via ssh
```

**Causes:**

* Wrong IP or hostname in inventory
* SSH key mismatch
* Firewall or security group blocking SSH

---

### Python not found

```text
/bin/sh: python: command not found
```

**Causes:**

* Minimal OS installation
* Python not installed or incorrect interpreter path

---

### Permission denied

```text
Permission denied (publickey)
```

**Causes:**

* Wrong SSH user
* Incorrect or missing private key

---

## Troubleshooting / Fixes

* Verify SSH manually:

```bash
ssh user@host
```

* Explicitly define Python interpreter:

```bash
ansible all -m ping -e ansible_python_interpreter=/usr/bin/python3
```

* Run with verbose output to find the root cause:

```bash
ansible all -m ping -vvv
```

Verbose mode shows SSH commands, authentication attempts, and exact failure points, which is critical in production troubleshooting.

---

## Best Practices

* Always run `ansible -m ping` before executing any playbook
* Fix ping failures first; do not debug playbooks until connectivity is verified
* Define `ansible_user` and `ansible_python_interpreter` in inventory
* Prefer group-level pings instead of `all` in large environments
* Treat Ansible ping as a **node readiness check**, not a network test

---
---

# Ansible Ad-hoc Commands — Running Shell Commands

## Concept / What

Running shell commands using Ansible **ad-hoc commands** means executing one-time Linux commands on remote hosts **without creating a playbook**. Ansible performs this by running a **single module** per execution, most commonly the `command` module (default) or the `shell` module when shell features are required.

In ad-hoc mode, Ansible is focused on **immediate execution**, not long-term configuration or state management.

---

## Why / Purpose / Real Use Case

Ad-hoc command execution is used for **quick operational tasks** and validations.

Real-world use cases:

* Check uptime across all servers
* Verify disk usage or memory usage
* Inspect running processes
* Validate OS version or kernel
* Perform quick checks during incidents
* Run pre-checks or post-checks around deployments

This approach avoids writing full playbooks for **temporary or diagnostic actions**.

---

## How it Works / Steps / Syntax

### Default behavior — `command` module

```bash
ansible all -a "uptime"
```

Explanation:

* `-m` is not specified
* Ansible automatically uses the **`command` module**
* `-a` passes the actual Linux command as arguments to the module

This is functionally equivalent to:

```bash
ansible all -m command -a "uptime"
```

The `command` module executes the command **directly**, without invoking a shell.

---

### Explicit use of `command`

```bash
ansible web -m command -a "df -h"
```

Key characteristics of `command`:

* Executes a **single explicit action**
* Does not support pipes (`|`), redirects (`>`), wildcards (`*`), or command chaining
* More predictable and safer for production usage

---

### When `command` is not enough

Commands requiring shell features will fail with `command`:

```bash
ansible all -m command -a "ps aux | grep nginx"
```

Reason:

* Pipes and shell operators require a shell interpreter
* `command` does not parse shell syntax

---

### Using the `shell` module

```bash
ansible all -m shell -a "ps aux | grep nginx"
```

The `shell` module runs commands through `/bin/sh`, allowing:

* Pipes and redirects
* Wildcards
* Environment variables
* Multiple commands (`&&`, `;`)
* Inline shell logic or scripts

---

### Important conceptual clarification

* `-m` is used to **select the module**
* `-a` is used to pass **arguments to that module**
* Modules are never passed as arguments
* If `-m` is omitted, **only `command` is assumed by default**
* `shell` is **never** the default and must always be explicitly specified

---

## Common Issues / Errors

### Commands work in SSH but fail in Ansible

Cause:

* Shell features were used while running via `command`

---

### Permission denied

Cause:

* Command requires elevated privileges
* `-b` (become) not used

---

### Partial or misleading success

Cause:

* Using `shell` with multiple chained commands
* One command fails silently while others succeed

---

## Troubleshooting / Fixes

* Use verbose output to identify failures:

```bash
ansible all -a "uptime" -vvv
```

* Test the command manually via SSH
* Switch from `command` to `shell` **only when shell features are required**
* Use full binary paths when PATH issues occur

---

## Best Practices

* Prefer `command` whenever possible for safety and predictability
* Use `shell` only when shell-specific features are truly required
* Avoid complex scripts inside `shell` ad-hoc commands
* Test ad-hoc commands on limited host groups before running on all
* Convert repeated ad-hoc commands into playbooks for maintainability

---
---

# Ansible Ad-hoc Commands — Installing Packages

## Concept / What

Installing packages with Ansible **ad-hoc commands** means using Ansible’s **package management modules** to install, remove, or update software on remote hosts **without writing a playbook**. Instead of executing raw OS commands via `shell` or `command`, Ansible uses **state-aware modules** that understand the system’s package manager and the desired end state.

Commonly used modules:

* `package` (generic, OS-agnostic)
* `apt` (Debian / Ubuntu)
* `yum` or `dnf` (RHEL, Amazon Linux)

---

## Why / Purpose / Real Use Case

Ad-hoc package installation is used for **quick, controlled changes** across multiple servers.

Real-world scenarios:

* Installing `nginx` on all web servers during setup
* Adding a missing dependency during an incident
* Verifying package availability before a deployment
* Temporarily installing troubleshooting tools

Using package modules is preferred because they are **safer, idempotent, and predictable** compared to shell-based installs.

---

## How it Works / Steps / Syntax

### Generic and recommended approach — `package` module

```bash
ansible web -m package -a "name=nginx state=present" -b
```

Explanation:

* `-m package` → selects the generic package module
* `name=nginx` → package to manage
* `state=present` → ensure the package is installed
* `-b` → run with sudo (required for package management)

Ansible internally:

* Gathers facts
* Detects the OS family
* Automatically chooses the correct backend (`apt`, `yum`, or `dnf`)

This removes the need for OS-based conditionals.

---

### OS-specific modules (when OS is known or control is required)

#### Using `yum` (RHEL / Amazon Linux)

```bash
ansible web -m yum -a "name=nginx state=present" -b
```

#### Using `apt` (Debian / Ubuntu)

```bash
ansible web -m apt -a "name=nginx state=present update_cache=yes" -b
```

OS-specific modules expose **advanced package manager features** that are not available in the generic `package` module.

---

### Removing a package

```bash
ansible web -m package -a "name=nginx state=absent" -b
```

---

### Installing multiple packages

```bash
ansible all -m package -a "name=vim,git,curl state=present" -b
```

---

## Important Conceptual Clarifications (Blended)

* The `package` module is a **wrapper**, not a replacement for `apt` or `yum`.
* It supports only **common operations** (install, remove, basic update).
* Advanced package management (repo enable/disable, cache tuning, excludes, version pinning) requires OS-specific modules.
* This design avoids complex OS detection logic while preserving full control when needed.

Earlier approaches required explicit OS checks:

```yaml
# Old approach
- apt:
    name: nginx
    state: present
  when: ansible_os_family == "Debian"

- yum:
    name: nginx
    state: present
  when: ansible_os_family == "RedHat"
```

Using `package` removes this complexity.

---

## Idempotency and Safety

* Package modules are **idempotent by design**.
* Ansible checks the current state before making changes.
* Re-running the same command does not reinstall packages unnecessarily.

This is why **package modules should never be replaced with `shell` or `command`** for installations.

---

## Common Issues / Errors

### Permission denied

Cause:

* Missing `-b` (sudo not enabled)

---

### Package not found

Cause:

* Incorrect package name
* Required repository not enabled

---

### Package manager lock

Cause:

* Another install process running
* Interrupted previous installation

---

## Troubleshooting / Fixes

* Verify package availability manually via SSH
* Use OS-specific modules if repository control is needed
* Run with verbosity to inspect failures:

```bash
ansible web -m package -a "name=nginx state=present" -b -vvv
```

---

## Best Practices

* Prefer `package` when OS is unknown or mixed
* Use `apt` / `yum` only when OS-specific control is required
* Always declare desired state (`present` or `absent`)
* Avoid using `shell` or `command` for package management
* Convert repeated ad-hoc installs into playbooks for long-term configuration

---
---

# Ansible Ad-hoc Commands — Managing Services

## Concept / What

Managing services with Ansible **ad-hoc commands** means controlling system services (start, stop, restart, reload, enable, disable) on remote hosts **without creating a playbook**. Ansible uses **service management modules** that understand the service state instead of executing raw system commands.

Common modules:

* `service` → generic service manager (works across many init systems)
* `systemd` → systemd-specific module (recommended on modern Linux)

---

## Why / Purpose / Real Use Case

Service control is a frequent operational need in day-to-day administration.

Real-world use cases:

* Start a service after package installation
* Restart a service after configuration changes
* Stop a faulty service during an incident
* Enable services to start on boot
* Validate that required services are running before deployments

Using Ansible modules ensures **idempotent and predictable behavior**, unlike shell-based service commands.

---

## How it Works / Steps / Syntax

### Start a service

```bash
ansible web -m service -a "name=nginx state=started" -b
```

Explanation:

* `-m service` → selects the service module
* `name=nginx` → service to manage
* `state=started` → ensures the service is running
* `-b` → privilege escalation (required for service control)

If the service is already running, Ansible performs no action.

---

### Stop a service

```bash
ansible web -m service -a "name=nginx state=stopped" -b
```

---

### Restart a service

```bash
ansible web -m service -a "name=nginx state=restarted" -b
```

Used when configuration changes require a service restart.

---

### Enable a service at boot

```bash
ansible web -m service -a "name=nginx enabled=yes" -b
```

Ensures the service starts automatically after system reboot.

---

### Using the `systemd` module (recommended)

```bash
ansible web -m systemd -a "name=nginx state=started enabled=yes" -b
```

Why `systemd`:

* Explicit support for modern Linux systems
* Better handling of reloads, masks, and daemon reloads
* Clearer behavior compared to generic service abstraction

---

## Common Issues / Errors

### Service not found

Cause:

* Incorrect service name
* Package not installed

---

### Permission denied

Cause:

* Missing `-b` (sudo privileges not enabled)

---

### Service fails to start

Cause:

* Invalid configuration
* Port conflicts
* Missing dependencies

---

## Troubleshooting / Fixes

* Verify service state with verbose output:

```bash
ansible web -m service -a "name=nginx state=started" -b -vvv
```

* Check logs directly on the host:

```bash
journalctl -u nginx
```

* Ensure the service package is installed before managing it

---

## Best Practices

* Use `service` or `systemd` modules instead of shell-based `systemctl` commands
* Prefer `systemd` on modern Linux distributions
* Always install the package before managing its service
* Use ad-hoc service control for quick operational actions
* Move repeated or permanent service logic into playbooks

---
---

# Ansible Ad-hoc Commands — Copying Files

## Concept / What

Copying files with Ansible **ad-hoc commands** means transferring files between the control node and managed nodes **without writing a playbook**. Ansible uses **state-aware file modules** instead of raw tools like `scp` or `rsync`, allowing it to manage not only file content but also ownership, permissions, and existence in an idempotent way.

Primary modules involved:

* `copy` → copies file content from control node to managed nodes
* `fetch` → copies files from managed nodes back to the control node
* `file` → manages filesystem objects (permissions, ownership, directories, links), not content

---

## Why / Purpose / Real Use Case

File management is a frequent operational requirement.

Real-world use cases:

* Distributing configuration files
* Pushing scripts or small binaries
* Setting correct permissions on application files
* Creating required directories
* Collecting logs during incidents

Using Ansible modules ensures **consistency, safety, and idempotency**, which is difficult to achieve with shell-based copying.

---

## How it Works / Steps / Syntax

### Copy a file from control node to managed nodes

```bash
ansible web -m copy -a "src=/tmp/nginx.conf dest=/etc/nginx/nginx.conf" -b
```

Explanation:

* `src` → file path on the control node
* `dest` → destination path on the managed node
* `-b` → required for system paths

Ansible checks the file checksum and copies only if the content differs.

---

### Copy a file with ownership and permissions

```bash
ansible web -m copy -a "src=app.sh dest=/usr/local/bin/app.sh owner=root group=root mode=0755" -b
```

Here, permissions and ownership are **declared as desired state**. If the file already exists with correct owner, group, and mode, Ansible makes no changes. If any attribute differs, only that attribute is corrected.

---

### Understanding `mode` syntax (permissions)

Permission format used by Ansible:

```
[special][owner][group][others]
```

Examples:

* `0755` → rwx r-x r-x
* `0644` → rw- r-- r--

The first digit (`0`) represents special permissions. In most real-world DevOps usage, this remains `0`. Knowing that this digit exists is sufficient; learning special permissions is not required for daily Ansible usage.

Best practice:

* Always use **four-digit modes**
* Quote modes in playbooks to avoid YAML parsing issues

---

### Create a directory using the `file` module

```bash
ansible web -m file -a "path=/opt/myapp state=directory owner=root group=root mode=0755" -b
```

Ensures the directory exists with correct permissions. If it already exists and matches the desired state, no action is taken.

---

### Change permissions or ownership of an existing file

```bash
ansible web -m file -a "path=/usr/local/bin/app.sh owner=root group=root mode=0755" -b
```

The `file` module modifies metadata only and does not touch file content.

---

### Create an empty file

```bash
ansible web -m file -a "path=/var/log/myapp.log state=touch owner=root group=root mode=0644" -b
```

Creates the file if missing and enforces ownership and permissions.

---

### Fetch a file from managed node to control node

```bash
ansible web -m fetch -a "src=/var/log/nginx/error.log dest=/tmp/logs/"
```

Files are copied into host-specific directories on the control node, making this useful for log collection and debugging.

---

## Common Issues / Errors

### File not found

Cause:

* Incorrect `src` path on the control node

---

### Permission denied

Cause:

* Missing `-b`
* Insufficient privileges on destination path

---

### Inefficient large file transfer

Cause:

* `copy` is not optimized for very large files

Fix:

* Use `synchronize` (rsync-based) for large data transfers

---

## Troubleshooting / Fixes

* Verify file paths on control node
* Confirm permissions on destination directories
* Run with verbose output for debugging:

```bash
ansible web -m copy -a "src=test.txt dest=/tmp/test.txt" -b -vvv
```

---

## Best Practices

* Use `copy` when file content matters
* Use `file` when only existence, ownership, or permissions matter
* Use `fetch` for collecting logs or artifacts
* Avoid shell-based `scp` or `rsync` for small files
* Move repeated file operations into playbooks

---
---

# Ansible Ad-hoc Commands — Managing Users

## Concept / What

Managing users with Ansible **ad-hoc commands** means creating, modifying, locking, or deleting Linux user accounts and groups on remote hosts **without writing a playbook**. Ansible uses **state-aware user management modules** instead of raw system commands, allowing safe and repeatable user operations.

Primary modules:

* `user` → manages user accounts and their attributes
* `group` → manages system groups

---

## Why / Purpose / Real Use Case

User and group management is a routine operational task in real environments.

Common scenarios:

* Creating application or service users
* Adding users during onboarding
* Removing users during offboarding
* Managing group membership for access control
* Temporarily locking user accounts for security reasons

Using Ansible ensures **consistency, idempotency, and safety** across all servers.

---

## How it Works / Steps / Syntax

### Create a user

```bash
ansible all -m user -a "name=appuser state=present" -b
```

Ensures the user exists. If the user is already present, Ansible makes no change.

---

### Create a user with a home directory

```bash
ansible all -m user -a "name=appuser state=present create_home=yes" -b
```

Creates the home directory if it does not already exist.

---

### Set a specific login shell

```bash
ansible all -m user -a "name=appuser shell=/bin/bash" -b
```

Manages the user’s login shell. This does not affect file permissions.

---

### Create a group

```bash
ansible all -m group -a "name=developers state=present" -b
```

Ensures the group exists on the target hosts.

---

### Add a user to a group

```bash
ansible all -m user -a "name=appuser groups=developers append=yes" -b
```

Important behavior:

* `groups=developers` → secondary group assignment
* `append=yes` → preserves existing group memberships

Omitting `append=yes` may remove the user from other groups, which is risky in production.

---

### Delete a user

```bash
ansible all -m user -a "name=appuser state=absent" -b
```

Removes the user account but leaves the home directory by default.

---

### Delete a user and remove home directory

```bash
ansible all -m user -a "name=appuser state=absent remove=yes" -b
```

Ensures both the user and their home directory are removed.

---

### Lock a user account

```bash
ansible all -m user -a "name=appuser password_lock=yes" -b
```

Used to temporarily disable user access without deleting the account.

---

## Common Issues / Errors

### Permission denied

Cause:

* Missing `-b` (sudo privileges required)

---

### Group membership overwritten

Cause:

* Using `groups=` without `append=yes`

---

### User already exists error (manual commands)

Cause:

* Using shell-based user management instead of Ansible modules

---

## Troubleshooting / Fixes

* Verify user existence:

```bash
id appuser
```

* Check group membership:

```bash
groups appuser
```

* Run ad-hoc commands with verbosity for debugging:

```bash
ansible all -m user -a "name=appuser state=present" -b -vvv
```

---

## Best Practices

* Always use `user` and `group` modules instead of shell commands
* Use `append=yes` when modifying group memberships
* Avoid managing file permissions with the `user` module
* Use ad-hoc commands for one-time user operations
* Move standard user definitions into playbooks for long-term consistency

---
---

# Ansible Ad-hoc Commands — Why Playbooks Are Preferred Over Ad-hoc

## Concept / What

In Ansible, **ad-hoc commands** and **playbooks** serve different purposes even though they use the same underlying modules. Ad-hoc commands are designed for **quick, one-time actions**, while playbooks are designed for **structured, repeatable, and maintainable automation**.

Ad-hoc commands are executed directly from the CLI and perform a single task at a time. Playbooks are YAML files that define **multiple tasks**, their order, and the desired end state of systems.

---

## Why / Purpose / Real Use Case

### Ad-hoc commands are used when:

* A task is one-time or temporary
* Quick checks are needed (uptime, disk usage, service status)
* Emergency or incident-response actions are required
* You want to test a module or validate connectivity

### Playbooks are used when:

* Tasks must be repeatable and consistent
* Systems need to be configured the same way every time
* Applications need structured deployment steps
* Automation must be shared across teams
* Changes must be reviewed, audited, and version-controlled

In real organizations, ad-hoc commands are useful for **immediate actions**, but playbooks are essential for **long-term automation**.

---

## How it Works / Steps / Syntax

### Ad-hoc command example

```bash
ansible web -m service -a "name=nginx state=started" -b
```

Characteristics:

* Executes a single action
* Must be manually re-run
* Logic is not stored anywhere
* Limited control flow

---

### Playbook example

```yaml
- name: Configure web servers
  hosts: web
  become: yes
  tasks:
    - name: Install nginx
      package:
        name: nginx
        state: present

    - name: Start nginx service
      service:
        name: nginx
        state: started
```

Characteristics:

* Defines multiple ordered tasks
* Safe to re-run
* Declarative and idempotent
* Acts as documentation

---

## Key Differences Explained

### Repeatability

* Ad-hoc: Manual execution every time
* Playbooks: Can be re-run safely without unintended changes

---

### Idempotency

* Ad-hoc often relies on `shell` or `command`
* Playbooks prefer state-aware modules that enforce desired state

---

### Readability and Documentation

* Ad-hoc commands live in terminal history
* Playbooks are readable YAML files that document system configuration

---

### Control Flow and Logic

Playbooks support:

* Multiple tasks
* Conditionals
* Loops
* Variables
* Handlers
* Tags

Ad-hoc commands do not support these features.

---

### Version Control and Collaboration

* Ad-hoc commands are not tracked
* Playbooks are stored in Git, reviewed via pull requests, and auditable

---

## Conceptual Clarification

Ad-hoc commands are **not wrong** and should not be avoided entirely. They are simply **not scalable**. As automation grows, repeating ad-hoc commands becomes error-prone and hard to manage.

Ansible’s design philosophy:

> Use ad-hoc commands for *now*, and playbooks for *always*.

---

## Best Practices

* Use ad-hoc commands for one-time actions and troubleshooting
* Use playbooks for consistent, repeatable automation
* Convert successful ad-hoc workflows into playbooks
* Avoid running complex shell logic repeatedly via ad-hoc
* Treat playbooks as production-grade automation artifacts

---
---
---
