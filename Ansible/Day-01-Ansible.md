# Ansible Fundamentals – What is Ansible and Why It Is Used

## Concept / What

Ansible is an **open-source automation and configuration management tool** used to manage systems, configure servers, and deploy applications in a **consistent and repeatable** way. It follows a **declarative approach**, where we define the desired state of the system, and Ansible ensures the system reaches and maintains that state.

Ansible uses **YAML-based playbooks** to describe tasks and configurations in a human-readable format. It is **agentless**, meaning no software needs to be installed on managed servers.

---

## Why / Purpose / Real Use Case

### Why Ansible Is Needed

In real-world environments, managing servers manually leads to:

* Configuration drift between environments (Dev / QA / Prod)
* Human errors due to repetitive manual work
* Inconsistent setups across multiple servers
* Slow deployments and rollbacks

Ansible solves these problems by automating system configuration and application deployment.

### Real-World Usage

Ansible is commonly used for:

* **Configuration Management**: Installing packages, managing services, users, permissions, and system configurations
* **Application Deployment**: Deploying Java, Node.js, or Python applications consistently
* **Post-Provisioning Automation**: Configuring servers after they are created using tools like Terraform or CloudFormation
* **CI/CD Pipelines**: Used in Jenkins or GitHub Actions to automate deployments

> Note: While Ansible *can* provision infrastructure, in real-world production setups it is mainly used **after infrastructure creation**, not as a replacement for Terraform.

---

## How it Works / Steps / Syntax

### High-Level Working Flow

1. Write automation logic in **YAML playbooks**
2. Define the desired state (what should be installed, running, configured)
3. Run the playbook from the Ansible control node
4. Ansible executes tasks on target servers
5. Systems reach the desired state

### Example Playbook (Real-World Context)

Ensuring Nginx is installed and running on servers:

```yaml
- name: Install and start nginx
  hosts: webservers
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Start nginx service
      service:
        name: nginx
        state: started
        enabled: true
```

#### Explanation

* **hosts**: Target server group
* **become**: Run tasks with sudo
* **tasks**: List of actions to perform
* **state: present**: Declarative instruction ensuring nginx is installed
* **state: started**: Ensures the service is running

Running this playbook multiple times will not break anything due to **idempotency**.

---

## Common Issues / Errors

* Assuming Ansible is only for application deployment
* Treating Ansible like a shell script tool
* Using Ansible to fully replace infrastructure provisioning tools
* Writing non-idempotent tasks

---

## Troubleshooting / Fixes

* If tasks run every time, verify module usage supports idempotency
* Separate infrastructure provisioning and configuration responsibilities
* Use `--check` mode to preview changes
* Use `-v`, `-vvv` for detailed execution logs

---

## Best Practices

* Use Ansible mainly for **automation and configuration management**
* Keep playbooks declarative, not imperative
* Store playbooks in version control (Git)
* Avoid hardcoding values; use variables
* Test playbooks in lower environments first

---
---

# Ansible Fundamentals – Agentless Architecture (SSH-based)

## Concept / What

Ansible follows an **agentless architecture**, meaning there is **no agent or daemon installed on managed (worker) nodes**. Ansible runs only from a **control node**, and when automation is executed, it temporarily connects to managed nodes to perform tasks.

The control node communicates with managed nodes using **standard remote protocols**, primarily **SSH for Linux/Unix systems** and **WinRM for Windows systems**. No persistent software remains on the managed nodes after execution.

---

## Why / Purpose / Real Use Case

### Why Agentless Architecture Is Important

In real-world production environments:

* Security teams restrict installing additional agents on servers
* Maintaining agents across thousands of servers increases operational overhead
* Agent upgrades and compatibility issues create risk

Ansible avoids these problems by using existing system capabilities instead of requiring a resident agent.

### Real-World Usage

* **Secure environments** where only SSH access is permitted
* **Auto-scaling environments** (e.g., EC2) where servers are short-lived
* **Hybrid and multi-cloud setups** where OS versions differ

Agentless design makes Ansible lightweight, easy to adopt, and suitable for large-scale infrastructure automation.

---

## How it Works / Steps / Execution Flow

1. Ansible is installed on a machine designated as the **control node**
2. A playbook is executed from the control node
3. The control node initiates an **SSH connection** to the managed nodes
4. Required Ansible modules (written in Python) are **copied temporarily** to the managed nodes
5. The modules execute the tasks
6. Modules are removed after execution

Managed nodes do not initiate communication and do not maintain any persistent connection with the control node.

---

## Requirements on Managed Nodes

For Linux/Unix systems:

* SSH service enabled
* Python installed (python or python3)
* Valid SSH credentials and network access

For Windows systems:

* WinRM enabled
* Proper authentication configured

---

## Common Issues / Errors

* SSH connection failures due to key or permission issues
* Python not installed on managed nodes
* Incorrect user privileges during task execution
* Assuming agentless means no dependencies at all

---

## Troubleshooting / Fixes

* Verify SSH access manually before running playbooks
* Install Python or configure `ansible_python_interpreter`
* Use `become: true` for privilege escalation
* Run playbooks with `-v`, `-vvv` for detailed logs

---

## Best Practices

* Use SSH key-based authentication
* Avoid password-based SSH access
* Ensure Python is installed on all managed nodes
* Clearly separate control node and managed node responsibilities
* Keep automation push-based and stateless

---
---

# Ansible Fundamentals – Control Node vs Managed Nodes

## Concept / What

In Ansible architecture, systems are logically divided into **Control Node** and **Managed Nodes**.

* The **Control Node** is the machine where **Ansible is installed and executed**. It is responsible for running playbooks, initiating connections, and pushing automation tasks.
* **Managed Nodes** are the target servers that Ansible configures and manages. They do not have Ansible installed and do not run any Ansible-related service.

This separation is fundamental to Ansible’s **agentless, push-based design**.

---

## Why / Purpose / Real Use Case

### Why This Separation Exists

In real-world environments, teams want:

* Centralized control over automation
* Better security by limiting where credentials live
* Clear separation between automation logic and application runtime

By keeping Ansible execution centralized:

* SSH keys and inventories are managed in one place
* Changes are auditable and traceable
* Automation access can be tightly controlled

### Real-World Usage

* A **dedicated EC2 instance** is often used as an Ansible control node in mature or security-sensitive setups
* **Jenkins** or other CI/CD tools act as orchestrators that trigger Ansible, not replace it
* Managed nodes can be EC2 instances, VMs, on-prem servers, or cloud hosts

---

## How it Works / Steps / Interaction Flow

1. Ansible is installed on the **control node**
2. Playbooks and inventories are maintained on the control node
3. Jenkins or a user triggers playbook execution
4. The control node initiates **SSH connections** to managed nodes
5. Tasks are executed on managed nodes using temporary modules
6. Results are sent back to the control node

Managed nodes never initiate communication and do not maintain any persistent connection to the control node.

---

## Key Differences

| Aspect            | Control Node | Managed Nodes |
| ----------------- | ------------ | ------------- |
| Ansible installed | Yes          | No            |
| Initiates SSH     | Yes          | No            |
| Runs playbooks    | Yes          | No            |
| Holds inventories | Yes          | No            |
| Persistent agent  | No           | No            |

---

## Common Issues / Errors

* Assuming managed nodes need Ansible installed
* Confusing Jenkins with the Ansible control node
* Allowing Jenkins to directly hold infrastructure SSH credentials
* Misunderstanding that managed nodes communicate back independently

---

## Troubleshooting / Fixes

* Verify SSH access from control node to managed nodes
* Ensure correct user and permissions are configured
* Use `become: true` when elevated privileges are required
* Enable verbose logging (`-v`, `-vvv`) for execution visibility

---

## Best Practices

* Treat the control node as the single source of automation truth
* Keep configuration management (Ansible) separate from CI/CD orchestration
* Store playbooks and inventories in version control
* Restrict direct access to the control node
* Automate Ansible execution via Jenkins instead of manual runs

---
---

# Ansible Fundamentals – Push Model

## Concept / What

Ansible follows a **push-based execution model**, where the **control node initiates all actions** and explicitly pushes tasks to managed nodes. Managed nodes do not request configuration changes and do not run any background agent or service.

In this model, Ansible connects to managed nodes only when a playbook is executed, performs the required tasks, and then exits. There is no continuous communication or enforcement after execution completes.

---

## Why / Purpose / Real Use Case

### Why Push Model Is Used

The push model is designed to provide:

* **Centralized control** over when and how changes are applied
* **Improved security**, since managed nodes do not need outbound access or agents
* **Operational simplicity**, with no agent installation or maintenance
* **Predictable change management**, as tasks run only when triggered

This fits well with environments where changes must be controlled and auditable.

### Real-World Usage

* **Production deployments** where Jenkins triggers Ansible during approved windows
* **Configuration updates** such as installing packages or updating system files
* **On-demand operations** like patching servers or restarting services
* **Auto-scaling setups**, where new servers are configured once after creation

---

## How it Works / Execution Flow

1. A playbook execution is triggered (manually or via Jenkins)
2. The Ansible **control node** starts execution
3. The control node opens **SSH (or WinRM) connections** to managed nodes
4. Tasks are pushed and executed using temporary modules
5. Results are collected and reported
6. Execution ends with no persistent process left behind

Managed nodes never initiate communication and do not continuously enforce state.

---

## Push Model vs Pull Model (Context)

* In a **push model**, changes happen only when explicitly triggered
* In a **pull model**, managed nodes periodically check a central server for changes

Ansible intentionally avoids the pull model to remain lightweight, secure, and easy to operate.

---

## Common Issues / Errors

* Assuming push model means manual execution only
* Expecting automatic self-healing without re-running playbooks
* Confusing Ansible’s push model with Kubernetes reconciliation

---

## Troubleshooting / Fixes

* Use CI/CD tools like Jenkins to automate playbook execution
* Rerun playbooks to re-apply desired state when needed
* Increase verbosity (`-v`, `-vvv`) to debug execution issues

---

## Best Practices

* Trigger Ansible via CI/CD instead of manual runs
* Keep playbooks idempotent to safely re-run them
* Use inventories and variables to target environments cleanly
* Schedule or pipeline-trigger executions for consistency

---
---

# Ansible Fundamentals – YAML Basics

## Concept / What

YAML (YAML Ain’t Markup Language) is a **human-readable data serialization format** used by Ansible to define playbooks, tasks, variables, inventories, and configuration data. YAML is **not a programming language**; it is a structured way to represent data that Ansible interprets to perform automation.

In Ansible, YAML is used to describe **what the desired state should be**, not how to execute logic step by step.

---

## Why / Purpose / Real Use Case

### Why Ansible Uses YAML

YAML is chosen because it is:

* Easy to read and understand by humans
* Less verbose than JSON or XML
* Friendly for DevOps and operations teams
* Suitable for version control and code reviews

In real-world organizations:

* Playbooks are reviewed by multiple teams
* YAML makes automation **auditable and maintainable**
* Clean structure reduces misconfiguration risks

---

## How it Works / Rules and Structure

### Indentation (Critical Rule)

YAML uses **spaces for indentation**, not tab characters. Indentation defines hierarchy and structure.

* Tabs are not allowed by the YAML specification
* Editors like VS Code often convert the Tab key into spaces for YAML files
* Plain text files do not enforce this, which can cause YAML parsing errors

Example:

```yaml
tasks:
  - name: Install nginx
    apt:
      name: nginx
      state: present
```

Consistent indentation is mandatory for correct parsing.

---

### Key–Value Pairs

YAML represents data using `key: value` syntax.

Example:

```yaml
become: true
```

Here:

* `become` is the key
* `true` is the value

---

### Lists

Lists are defined using hyphens (`-`). Lists are heavily used in Ansible, especially for tasks.

Example:

```yaml
tasks:
  - name: Start nginx
    service:
      name: nginx
      state: started
```

Each hyphen represents one list item.

---

### Dictionaries (Maps)

Dictionaries are nested key–value structures used to pass parameters to Ansible modules.

Example:

```yaml
apt:
  name: nginx
  state: present
```

---

### Data Types

YAML supports basic data types:

```yaml
name: nginx
enabled: true
replicas: 3
```

* Strings usually do not require quotes
* Booleans are `true` or `false`
* Numbers are unquoted

---

### YAML Document Start

The `---` at the beginning of a file indicates the start of a YAML document. It is optional but considered good practice in Ansible playbooks.

---

## Real-World Ansible Example

```yaml
---
- name: Configure web server
  hosts: webservers
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present

    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: true
```

This example shows how YAML structure defines plays, tasks, and module parameters.

---

## Common Issues / Errors

* Using real tab characters instead of spaces
* Incorrect indentation levels
* Mixing list and dictionary indentation
* Treating YAML like a scripting language

---

## Troubleshooting / Fixes

* Use consistent spacing (commonly 2 spaces)
* Enable whitespace rendering in editors
* Run `ansible-playbook --syntax-check`
* Use YAML linters in editors or CI pipelines

---

## Best Practices

* Always use spaces, never real tab characters
* Keep indentation consistent across files
* Use meaningful key names
* Validate YAML before execution
* Store YAML files in version control

---
---

# Ansible Fundamentals – Installation and ansible.cfg Basics

## Concept / What

Before using Ansible, it must be **installed on the control node**. Managed nodes do not require Ansible to be installed. Along with installation, Ansible behavior is controlled using a configuration file called **`ansible.cfg`**, which defines default settings for how Ansible operates.

`ansible.cfg` does not store secrets or credentials. It only defines **behavioral defaults** such as inventory location, SSH user, privilege escalation, and execution settings.

---

## Why / Purpose / Real Use Case

### Why Installation Matters

* Ansible runs **only from the control node**
* Centralizes automation logic
* Keeps managed nodes lightweight and agentless

In real-world setups, the control node may be:

* A dedicated EC2 instance
* A bastion/admin server
* A Jenkins agent (in smaller setups)

### Why `ansible.cfg` Is Needed

Without `ansible.cfg`, operators must repeatedly pass command-line flags, increasing the risk of mistakes. Using `ansible.cfg`:

* Standardizes execution behavior
* Reduces human error
* Keeps pipelines and commands clean
* Ensures consistent behavior across environments

---

## How it Works / Steps / Syntax

### Installing Ansible (Control Node Only)

On Amazon Linux / RHEL / CentOS:

```bash
sudo yum install ansible -y
```

On Ubuntu / Debian:

```bash
sudo apt update
sudo apt install ansible -y
```

Verify installation:

```bash
ansible --version
```

This command also shows which `ansible.cfg` file is being used.

---

## What is `ansible.cfg`

`ansible.cfg` is Ansible’s main configuration file. It defines default values so they don’t need to be passed on every run.

Typical settings include:

* Inventory file path
* Default SSH user
* SSH behavior
* Privilege escalation settings
* Parallel execution limits

---

## `ansible.cfg` Precedence Order

Ansible loads configuration files in the following order:

1. File specified by `ANSIBLE_CONFIG` environment variable
2. `ansible.cfg` in the current project directory
3. `~/.ansible.cfg` (user-level)
4. `/etc/ansible/ansible.cfg` (system-level)

The **first file found is used**, which allows project-specific configuration.

---

## Common `ansible.cfg` Example

```ini
[defaults]
inventory = ./inventory
remote_user = ec2-user
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = true
become_method = sudo
become_user = root
```

### Explanation

* Sets default inventory location
* Defines SSH user
* Disables host key checking for automation
* Enables privilege escalation by default

---

## Common Issues / Errors

* `ansible.cfg` not being picked up
* Conflicting multiple configuration files
* Incorrect inventory path
* Permission issues due to missing `become`

---

## Troubleshooting / Fixes

* Run `ansible --version` to confirm active config file
* Use project-level `ansible.cfg`
* Avoid maintaining multiple conflicting config files
* Keep configuration minimal and explicit

---

## Best Practices

* Install Ansible only on control nodes
* Use project-level `ansible.cfg` for consistency
* Do not store secrets or SSH keys inside `ansible.cfg`
* Version-control `ansible.cfg` (excluding sensitive data)
* Let Jenkins trigger Ansible, not replace it

---
---
---
