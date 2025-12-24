# Ansible Playbooks – Playbook Structure

---

## Concept / What

An **Ansible playbook** is a YAML file that defines *what configuration or automation tasks should be executed*, *on which hosts*, and *how they should be applied*.
The **playbook structure** refers to the **mandatory and optional YAML sections (keys)** that make up a play and determine how Ansible interprets and runs it.

A playbook consists of **one or more plays**, and each play targets a group of hosts and executes tasks on them.

---

## Why / Purpose / Real Use Case

Playbook structure exists to:

* Provide a **clear, declarative layout** for automation
* Separate **target selection**, **privilege control**, and **execution logic**
* Ensure **repeatable and consistent configuration** across environments
* Make automation **readable, auditable, and maintainable**

### Real-world use cases

* Configuring OS packages on 100+ servers
* Deploying applications consistently across dev, qa, and prod
* Enforcing standard server baselines (users, permissions, services)

Without a clear structure:

* Playbooks become hard to debug
* Scaling automation becomes risky
* Team collaboration breaks down

---

## How it Works / Steps / Syntax

### 1. Minimum Valid Playbook Structure (Mandatory Sections)

Only **two sections are mandatory at the play level**:

```yaml
---
- hosts: web
  tasks:
    - name: Example task
      debug:
        msg: "Hello from Ansible"
```

### Mandatory sections explained

* **hosts**
  Defines *where* the play runs. It refers to a host or group from the inventory.

* **tasks**
  A list of actions executed sequentially using Ansible modules.

Without these two, a playbook is invalid.

---

### 2. Optional Playbook Sections (Structure-Level)

These sections are **not required**, but commonly used in real-world playbooks:

```yaml
---
- name: Web server setup
  hosts: web
  become: yes
  vars:
    http_port: 80

  tasks:
    - name: Install nginx
      yum:
        name: nginx
        state: present

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

### Optional sections explained

* **name** – Human-readable description of the play
* **become** – Enables privilege escalation (sudo)
* **vars** – Defines play-level variables
* **handlers** – Tasks triggered by notifications
* **pre_tasks / post_tasks** – Tasks executed before or after main tasks
* **roles** – Reusable automation units

These sections improve clarity, reuse, and control but are not mandatory.

---

### Important Clarification (Common Confusion)

* **Mandatory vs Optional applies to PLAYBOOK SECTIONS**, not to tasks
* Tasks themselves are **not mandatory or optional by structure**
* Task execution is controlled by logic such as:

  * `when` (conditions)
  * `ignore_errors`
  * `failed_when`

---

## Common Issues / Errors

* YAML indentation errors
* Missing `hosts` or `tasks`
* Defining modules outside `tasks`
* Inventory group name mismatch
* Assuming tasks can define their own hosts

---

## Troubleshooting / Fixes

* Syntax validation:

  ```bash
  ansible-playbook playbook.yml --syntax-check
  ```

* Verbose execution:

  ```bash
  ansible-playbook playbook.yml -vvv
  ```

* Inventory verification:

  ```bash
  ansible-inventory --list
  ```

---

## Best Practices

* Always use descriptive `name` fields for plays and tasks
* Keep one logical purpose per play
* Define hosts at the **play level only**
* Use optional sections only when needed
* Avoid mixing too many responsibilities in one play

---
---

# Ansible Playbooks – Tasks and Modules

---

## Concept / What

An **Ansible module** is a reusable unit of work that performs a specific action on a managed host, such as installing a package, managing a file, or starting a service.

A **task** is a wrapper around a module inside a playbook. It defines *how* and *when* a module is executed and provides human-readable context that appears in logs and output.

In short:

* **Modules do the actual work**
* **Tasks control execution, readability, and flow**

---

## Why / Purpose / Real Use Case

### Why modules exist

* Abstract low-level OS commands
* Provide **state-based execution** (desired state)
* Enable **idempotency**
* Handle OS differences automatically

### Why tasks exist

* Make automation **readable and traceable**
* Enable conditions, loops, tags, handlers, retries
* Provide logging and debugging visibility

### Real-world use cases

* Installing packages across hundreds of servers
* Managing users and permissions
* Configuring services safely and repeatedly
* Applying changes without breaking existing state

---

## How it Works / Steps / Syntax

### Basic task using a module

```yaml
- name: Install nginx
  yum:
    name: nginx
    state: present
```

### Field-level explanation

* `name` – Description shown in Ansible output
* `yum` – Module name
* `name: nginx` – Resource to manage
* `state: present` – Desired end state

Each task:

* Uses **one module only**
* Runs **once per host** in the play
* Executes **top to bottom**

---

## Idempotency (Critical Concept)

### What idempotency means

Idempotency ensures that **running the same playbook multiple times does not change the system if the desired state is already achieved**.

### Who provides idempotency?

* **Modules are idempotent**
* **Tasks inherit idempotency** because they use modules
* **Playbooks themselves are aware of idempotency only through modules**

---

## Non-Idempotent or Partially Idempotent Modules

Not all modules are truly idempotent.

### 1. `shell` module ❌

Executes shell commands directly. Ansible cannot determine system state.

```yaml
- name: Create file
  shell: touch /tmp/test.txt
```

* Runs every time
* No state awareness

---

### 2. `command` module ❌

Similar to `shell` but without shell features.

```yaml
- name: Create file
  command: touch /tmp/test.txt
```

* Always executes
* Not idempotent by default

---

### 3. `raw` module ❌

Runs raw SSH commands without Python on the host.

* No idempotency
* No state tracking
* Used mainly for bootstrapping

---

### 4. `script` module ❌

Copies and runs a script on the target host.

* Ansible has no visibility into script logic
* Idempotency depends entirely on script implementation

---

### 5. `expect` module ⚠️ (conditionally idempotent)

Used for interactive commands.

* Execution-based, not state-based
* Can cause repeated changes if not controlled

---

### Making non-idempotent modules safer (not true idempotency)

```yaml
- name: Create file only if not exists
  command: touch /tmp/test.txt
  args:
    creates: /tmp/test.txt
```

This *simulates* idempotency but Ansible still does not manage the resource.

---

## Common Issues / Errors

* Overusing `shell` instead of native modules
* Assuming all modules are idempotent
* Writing complex logic inside scripts
* Losing visibility into what changed and why

---

## Troubleshooting / Fixes

* Prefer native modules (`package`, `service`, `file`, `user`)
* Use verbose output to inspect changes:

  ```bash
  ansible-playbook playbook.yml -vvv
  ```
* Check `changed` vs `ok` status carefully

---

## Best Practices

* Always prefer **native state-based modules**
* Use `shell` / `command` only when no module exists
* Keep one action per task
* Give meaningful task names
* Avoid embedding long scripts inside tasks

---
---

# Ansible Playbooks – Handlers

---

## Concept / What

An **Ansible handler** is a special type of task that is executed **only when it is explicitly notified by another task**.

Handlers:

* Do **not run by default**
* Run **only when a notifying task reports `changed`**
* Are executed **at the end of a play by default**

Handlers follow an **event-driven model**.

---

## Why / Purpose / Real Use Case

In real systems, certain actions should happen **only when something actually changes**.

### Why handlers are needed

* Restarting services unnecessarily causes downtime
* Reloading services repeatedly affects performance
* Blind restarts break stable systems

Handlers ensure:

* Services restart **only when required**
* Configuration changes are applied safely
* Automation is efficient and predictable

### Real-world use cases

* Restart nginx only if its configuration file changes
* Reload systemd only if a unit file is updated
* Restart an application only after deployment changes

---

## How it Works / Steps / Syntax

### Basic handler example

```yaml
---
- name: Web server configuration
  hosts: web
  become: yes

  tasks:
    - name: Update nginx config
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: restart nginx

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted
```

### Execution flow

1. Task runs
2. If task status is `changed`
3. Handler is **queued**
4. All remaining tasks continue
5. Handler runs **once at the end of the play**

If the task reports `ok`, the handler is never executed.

---

## Important Handler Rules

### Rule 1: Name matching

The value in `notify` must **exactly match** the handler name.

```yaml
notify: restart nginx
```

Matches:

```yaml
- name: restart nginx
```

---

### Rule 2: Handlers run only once

If multiple tasks notify the same handler, it still runs **only once**.

---

### Rule 3: Handlers run at play end

Handlers always execute **after all tasks in the play finish**, unless explicitly flushed.

---

## Immediate Handler Execution (`meta: flush_handlers`)

By default, handlers wait until the end of the play. To run them immediately:

```yaml
- meta: flush_handlers
```

### Example

```yaml
- name: Update config
  copy:
    src: nginx.conf
    dest: /etc/nginx/nginx.conf
  notify: restart nginx

- name: Run handlers immediately
  meta: flush_handlers
```

### Key points

* Flushes **all currently notified handlers**
* Does **not** flush handlers that are not notified yet
* Cannot selectively flush a single handler

---

## Controlling Handler Execution Order (Best Practice)

### Limitation

It is **not possible** to flush only one handler selectively.

### Correct design pattern

Use **multiple plays in the same playbook file**.

Each play:

* Has its **own handler queue**
* Executes handlers independently

### Example

```yaml
---
- name: Critical config and restart
  hosts: web
  become: yes

  tasks:
    - name: Update nginx config
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: restart nginx

  handlers:
    - name: restart nginx
      service:
        name: nginx
        state: restarted

---
- name: Remaining setup
  hosts: web
  become: yes

  tasks:
    - name: Install curl
      yum:
        name: curl
        state: present

  handlers:
    - name: reload firewall
      service:
        name: firewalld
        state: reloaded
```

This ensures one handler executes earlier and others later.

---

## Common Issues / Errors

* Handler never runs because task did not change
* Typo in handler name
* Expecting handler to run immediately
* Restarting services inside tasks instead of handlers
* Overusing `meta: flush_handlers`

---

## Troubleshooting / Fixes

* Check task result (`changed` vs `ok`)
* Run with verbosity:

  ```bash
  ansible-playbook playbook.yml -vvv
  ```
* Validate handler names
* Prefer multiple plays instead of forced flush

---

## Best Practices

* Always use handlers for service restarts
* Keep handlers simple and reactive
* Avoid selective flush logic (not supported)
* Use multiple plays to control execution order
* Do not abuse `flush_handlers`

---
---

# Ansible Playbooks – Tags

---

## Concept / What

An **Ansible tag** is a label attached to a **task, play, or role** that allows selective execution of parts of a playbook.

Tags do **not change task logic** or execution order. They only control **which tasks are executed or skipped at runtime**.

Think of tags as **execution filters**, not conditions.

---

## Why / Purpose / Real Use Case

In real-world environments:

* Playbooks are large
* Running everything every time is slow and risky
* Production changes must be controlled

Tags help to:

* Run only specific sections of a playbook
* Skip risky operations like restarts or deletes
* Speed up troubleshooting and testing
* Reuse the same playbook for multiple purposes

### Real-world use cases

* Run only installation steps
* Apply only configuration changes
* Skip service restarts in production
* Debug a single task without executing everything

---

## How it Works / Steps / Syntax

### Tagging a task

```yaml
- name: Install nginx
  yum:
    name: nginx
    state: present
  tags:
    - install
```

Run only this task:

```bash
ansible-playbook site.yml --tags install
```

---

### Multiple tags on a task

```yaml
- name: Start nginx
  service:
    name: nginx
    state: started
  tags:
    - service
    - start
```

Running with either tag will execute this task.

---

### Tagging an entire play

```yaml
- name: Web server setup
  hosts: web
  tags:
    - web
```

All tasks inside the play inherit this tag.

---

### Tagging roles (conceptual awareness)

```yaml
roles:
  - role: nginx
    tags:
      - web
```

All tasks inside the role inherit the tag.
(Roles will be covered in later concepts.)

---

## Special Tags (Built-in)

### `always` (special)

* Runs regardless of `--tags` or `--skip-tags`

```yaml
tags:
  - always
```

---

### `never` (special)

* Does not run by default
* Runs only when explicitly tagged

```yaml
tags:
  - never
```

Run explicitly:

```bash
ansible-playbook site.yml --tags never
```

---

### Custom tags (normal)

Tags like `dangerous`, `install`, `config`:

* Have no special behavior
* Are meaningful by convention only

Common safety pattern:

```yaml
tags:
  - never
  - dangerous
```

This blocks execution by default and allows controlled execution.

---

## Skipping Tasks Using Tags

```bash
ansible-playbook site.yml --skip-tags restart
```

Example task:

```yaml
- name: Restart nginx
  service:
    name: nginx
    state: restarted
  tags:
    - restart
```

---

## Common Issues / Errors

* Forgetting to tag dependent tasks
* Assuming tags control execution order
* Over-tagging everything
* Forgetting to protect dangerous operations

---

## Troubleshooting / Fixes

* List tags in a playbook:

  ```bash
  ansible-playbook site.yml --list-tags
  ```

* Dry-run with tags:

  ```bash
  ansible-playbook site.yml --tags config --check
  ```

---

## Best Practices

* Use tags for logical grouping
* Protect risky tasks using `never` + custom tag
* Keep tag names simple and meaningful
* Do not use tags as a replacement for conditions

---
---

# Ansible Playbooks – Registers (Capturing Output)

---

## Concept / What

In Ansible, **`register`** is used to **capture the result/output of a task execution** and store it in a variable.

That variable can then be:

* Reused by later tasks
* Evaluated in conditions (`when`)
* Printed for visibility
* Used in loops and decision-making logic

`register` stores the **entire task result as a dictionary**, not just plain output.

---

## Why / Purpose / Real Use Case

By default, when Ansible runs a task:

* The output is displayed on the terminal
* The execution continues
* The output is **not reusable for logic**

Registers exist to:

* Make decisions based on command results
* Check system state before acting
* Avoid unnecessary or risky operations
* Build conditional and intelligent playbooks

### Real-world use cases

* Restart a service only if a config file exists
* Run a task only if a command succeeds
* Validate application health before deployment
* Check service or process status

---

## How it Works / Steps / Syntax

### Basic register example

```yaml
- name: Check disk usage
  command: df -h
  register: disk_output
```

* The task runs normally
* Its execution result is stored in `disk_output`

---

### Viewing registered output (using debug)

```yaml
- name: Print disk output
  debug:
    var: disk_output
```

This is commonly used during learning and troubleshooting.

---

## Structure of a Registered Variable (Very Important)

A registered variable is a **dictionary** containing multiple fields.

Common fields (module-dependent):

* `changed` – Whether the task made changes
* `failed` – Whether the task failed
* `msg` – Informational message (if provided)
* `rc` – Return code (mainly for command/shell)
* `stdout` – Standard output (command/shell)
* `stderr` – Error output (command/shell)

⚠️ Not all modules return all fields.

---

## Register with State-Based Modules

Example using `stat`:

```yaml
- name: Check if nginx config exists
  stat:
    path: /etc/nginx/nginx.conf
  register: nginx_conf
```

Accessing data:

```yaml
nginx_conf.stat.exists
```

This is a **very common production pattern**.

---

## Using Register with Conditions (`when`)

```yaml
- name: Restart nginx only if config exists
  service:
    name: nginx
    state: restarted
  when: nginx_conf.stat.exists
```

Here:

* `register` provides data
* `when` consumes that data

---

## Register with Loops

```yaml
- name: Check services
  command: systemctl is-active {{ item }}
  loop:
    - nginx
    - docker
  register: service_status
```

Important:

* Loop results are stored under:

```yaml
service_status.results
```

Each item contains its own `stdout`, `rc`, etc.

---

## Common Issues / Errors

* Using `register` but never consuming the variable
* Assuming all fields exist for every module
* Forgetting `.results` when using loops
* Accessing `stdout` for non-command modules
* Overusing register when not required

---

## Troubleshooting / Fixes

* Inspect structure first:

  ```yaml
  - debug:
      var: my_register
  ```

* Use verbosity to understand execution:

  ```bash
  ansible-playbook playbook.yml -vvv
  ```

* Guard conditions safely:

  ```yaml
  when: my_var is defined
  ```

---

## Best Practices

* Use `register` only when output is required later
* Prefer state-based modules over command output checks
* Always inspect register structure before using fields
* Keep logic readable and minimal

---
---

# Ansible Playbooks – Conditions (`when`)

---

## Concept / What

In Ansible, **`when`** is a conditional keyword used to **control whether a task is executed or skipped**.

The condition provided to `when` is evaluated as a **boolean expression** at runtime:

* If the condition evaluates to **true**, the task runs
* If the condition evaluates to **false**, the task is skipped

`when` is applied **at the task level** and is a core part of Ansible’s control flow.

---

## Why / Purpose / Real Use Case

Real-world environments are rarely identical:

* Different operating systems
* Different server roles
* Services may or may not exist
* Files or configurations may already be present

Using `when` allows playbooks to be:

* Safe and adaptive
* Environment-aware
* Idempotent-friendly
* Resistant to unnecessary failures

### Real-world use cases

* Restart a service only if a configuration file exists
* Install packages based on OS family
* Run cleanup tasks only in non-production environments
* Execute tasks based on disk, memory, or CPU conditions

---

## How it Works / Steps / Syntax

### Basic `when` usage

```yaml
- name: Restart nginx if config exists
  service:
    name: nginx
    state: restarted
  when: nginx_conf.stat.exists
```

Here:

* `nginx_conf` is a registered variable
* The task runs only if the condition evaluates to `true`

---

### Using `when` with facts

```yaml
- name: Install nginx on Ubuntu
  apt:
    name: nginx
    state: present
  when: ansible_facts['os_family'] == 'Debian'
```

---

### Multiple conditions

```yaml
- name: Restart nginx only on RedHat and if config exists
  service:
    name: nginx
    state: restarted
  when:
    - nginx_conf.stat.exists
    - ansible_facts['os_family'] == 'RedHat'
```

All listed conditions must evaluate to `true`.

---

### Using `when` with registered command output

```yaml
- name: Check if nginx is installed
  command: nginx -v
  register: nginx_check
  ignore_errors: yes

- name: Install nginx if not present
  yum:
    name: nginx
    state: present
  when: nginx_check.rc != 0
```

---

## Common Condition Patterns

### Check if a variable exists

```yaml
when: my_var is defined
```

---

### Boolean variables

```yaml
when: enable_nginx
```

---

### String comparison

```yaml
when: env == 'prod'
```

---

### Numeric comparison

```yaml
when: disk_usage > 80
```

---

## Common Issues / Errors

* Using `{{ }}` inside `when` conditions
* Comparing strings with booleans incorrectly
* Using shell conditionals instead of `when`
* Forgetting `ignore_errors` when checking failing commands

---

## Troubleshooting / Fixes

* Inspect variables using debug:

  ```yaml
  - debug:
      var: my_var
  ```

* Run playbooks with verbosity:

  ```bash
  ansible-playbook playbook.yml -vvv
  ```

* Validate condition logic carefully

---

## Best Practices

* Use `when` instead of shell-based condition logic
* Keep conditions simple and readable
* Prefer facts and module output over raw command checks
* Avoid complex Jinja expressions inside `when`

---
---

# Ansible Playbooks – Loops (`loop`)

---

## Concept / What

In Ansible, **`loop`** is used to **execute the same task multiple times with different input values**.

Instead of repeating identical tasks, a loop allows you to:

* Define the task once
* Iterate over a list of items
* Run the task once per item

Each iteration uses a special variable called **`item`**.

---

## Why / Purpose / Real Use Case

In real-world automation, repetition is common:

* Installing multiple packages
* Creating multiple users
* Managing multiple services
* Checking status of several components

Without loops:

* Playbooks become long and repetitive
* Maintenance becomes difficult

Loops make playbooks:

* Short and readable
* Scalable
* Less error-prone

---

## How it Works / Steps / Syntax

### Basic loop example

```yaml
- name: Install multiple packages
  yum:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl
```

Execution:

* Task runs once for each item
* `item` takes values sequentially (`nginx`, `git`, `curl`)

---

## Understanding `item` (Important)

* `item` is an **implicit variable** provided by Ansible
* It represents the **current value** in the loop

---

## Loop with Variables

```yaml
vars:
  packages:
    - nginx
    - git
    - curl

tasks:
  - name: Install packages
    yum:
      name: "{{ item }}"
      state: present
    loop: "{{ packages }}"
```

---

## Loop with Dictionaries (Key–Value)

```yaml
- name: Create users
  user:
    name: "{{ item.name }}"
    uid: "{{ item.uid }}"
  loop:
    - { name: dev1, uid: 1001 }
    - { name: dev2, uid: 1002 }
```

---

## Loop with Register (Capturing Output)

```yaml
- name: Check services
  command: systemctl is-active {{ item }}
  loop:
    - nginx
    - docker
  register: service_status
```

### How register behaves with loops

* The register variable is **not overwritten**
* Output from each iteration is **appended**
* Results are stored under:

```yaml
service_status.results
```

---

## Understanding `item.item` (Very Important)

When looping over registered results:

```yaml
loop: "{{ service_status.results }}"
```

* Outer `item` → one result object
* Inner `item` (`item.item`) → original loop value

Example access:

* `item.item` → original service name
* `item.rc` → return code
* `item.stdout` → command output

---

## Using Loop with Conditions (`when`)

```yaml
- name: Restart active services
  service:
    name: "{{ item.item }}"
    state: restarted
  loop: "{{ service_status.results }}"
  when: item.rc == 0
```

Meaning:

* Loop through results
* Restart service only if command succeeded (`rc == 0`)

---

## Old Loop Syntax (Awareness Only)

```yaml
with_items:
```

* Deprecated
* `loop` is the recommended syntax

---

## Common Issues / Errors

* Forgetting to use `{{ item }}`
* Confusing `item` with `item.item`
* Forgetting `.results` when using register with loops
* Using loops where modules already accept lists

---

## Troubleshooting / Fixes

* Inspect loop variable:

  ```yaml
  - debug:
      var: item
  ```

* Inspect registered results:

  ```yaml
  - debug:
      var: service_status
  ```

---

## Best Practices

* Use loops to eliminate duplication
* Prefer module-supported lists when available
* Keep loop logic simple and readable
* Avoid deeply nested loops

---
---

# Ansible Playbooks – Delegation (`delegate_to`, `run_once`)

---

## Concept / What

By default, Ansible executes tasks **on every host defined in the `hosts` field** of a play.

**Delegation** allows you to change this behavior by controlling:

* **WHERE** a task runs → using `delegate_to`
* **HOW MANY TIMES** a task runs → using `run_once`

Delegation is commonly used when tasks must run on a **specific server**, **only once**, or **outside the target host group**.

---

## Why / Purpose / Real Use Case

In real-world automation:

* Some tasks must run **only once per deployment**
* Some tasks must run on a **central system** (load balancer, DB, control node)
* Some tasks must run on **localhost** (API calls, cloud actions)

Delegation helps avoid:

* Repeated execution of the same task
* Incorrect execution on all hosts
* Manual orchestration outside Ansible

### Real-world use cases

* Updating a load balancer configuration once
* Generating configuration files centrally
* Calling cloud APIs from localhost
* Running database migrations one time

---

## `delegate_to` — How it Works

### Concept

`delegate_to` instructs Ansible to **execute a task on a different host** than the one currently being iterated.

The delegated host:

* Must be **reachable**
* Must be **defined in inventory**, resolvable via DNS, or be `localhost`

---

### Basic example

```yaml
- name: Create file on localhost
  file:
    path: /tmp/test.txt
    state: touch
  delegate_to: localhost
```

Even if the play targets remote hosts, this task runs **locally**.

---

### Delegating to another inventory host

Inventory:

```ini
[loadbalancer]
lb1 ansible_host=10.0.0.10
```

Playbook:

```yaml
- name: Update LB config
  template:
    src: lb.conf.j2
    dest: /etc/nginx/lb.conf
  delegate_to: lb1
```

Ansible resolves `lb1` via inventory and connects using SSH.

---

## `run_once` — How it Works

### Concept

`run_once: true` ensures a task is **executed only one time per play run**, regardless of the number of target hosts.

Important:

* `run_once` does **not remember previous runs**
* The task can run again on the next playbook execution

---

### Example without delegation

```yaml
- name: Run only once
  command: echo "Hello"
  run_once: true
```

If multiple hosts are targeted, the task runs on the **first host in inventory order**.

---

## `delegate_to` + `run_once` (Most Common Pattern)

```yaml
- name: Update load balancer only once
  template:
    src: lb.conf.j2
    dest: /etc/nginx/lb.conf
  delegate_to: lb1
  run_once: true
```

Meaning:

* Task runs **only once**
* Task runs **on `lb1`**, not on all hosts

---

## Inventory and Delegation (Very Important)

`delegate_to` does **not bypass inventory**.

For Ansible to connect, the delegated host must:

* Exist in inventory (recommended)
* OR be DNS resolvable
* OR be `localhost`

### Best practice

> Always define delegated hosts in inventory, even if they are not part of the main `hosts` group.

---

## Common Issues / Errors

* Forgetting `run_once`, causing repeated execution
* Delegating to a host not reachable or not in inventory
* Assuming `run_once` prevents future executions
* Confusing execution host with inventory host

---

## Troubleshooting / Fixes

* Check where the task runs:

  ```yaml
  - debug:
      msg: "Running on {{ inventory_hostname }}"
  ```

* Increase verbosity:

  ```bash
  ansible-playbook playbook.yml -vvv
  ```

* Verify inventory resolution:

  ```bash
  ansible-inventory --list
  ```

---

## Best Practices

* Use `delegate_to` for centralized actions
* Use `run_once` for tasks that must execute once per play
* Combine both for infrastructure-wide orchestration
* Do not confuse `run_once` with idempotency

---
---

# Ansible Playbooks – Idempotency Concept

---

## Concept / What

**Idempotency** means that **running the same automation multiple times results in the same final system state**.

In Ansible, a task is considered idempotent if:

* First run → makes the required change
* Subsequent runs → make no changes if the desired state is already achieved

Idempotency is about **state enforcement**, not about how many times a task runs.

---

## Why / Purpose / Real Use Case

In real-world environments:

* Playbooks are re-run frequently
* CI/CD pipelines may retry jobs
* Partial failures require re-execution

Idempotency ensures:

* Safe re-runs
* No unnecessary service restarts
* No repeated configuration changes
* Predictable and stable systems

Without idempotency:

* Systems drift
* Automation becomes unsafe
* Production outages become likely

---

## How Idempotency Works in Ansible

### Core Rule (Very Important)

> **Idempotency is provided by Ansible modules, not by playbooks or tasks.**

State-based modules:

* Check the current system state
* Compare it with the desired state
* Decide whether a change is required

---

### Idempotent Module Examples

#### Package management

```yaml
- name: Ensure nginx is installed
  yum:
    name: nginx
    state: present
```

* First run → installs nginx (`changed`)
* Subsequent runs → no action (`ok`)

---

#### File system state

```yaml
- name: Ensure directory exists
  file:
    path: /opt/app
    state: directory
```

* Directory missing → created
* Directory exists → skipped

---

## Non-Idempotent Modules (Execution-Based)

The following modules are **not idempotent by default**:

* `command`
* `shell`
* `raw`
* `script`

Example:

```yaml
- name: Create file using shell
  shell: touch /tmp/test.txt
```

* Runs every time
* Ansible cannot determine desired state

---

## Reducing Damage with `creates` and `removes`

### Using `creates`

```yaml
- name: Create file only if missing
  command: touch /tmp/test.txt
  args:
    creates: /tmp/test.txt
```

Behavior:

* File does not exist → command runs
* File exists → command is skipped

---

### Using `removes`

```yaml
- name: Remove file only if present
  command: rm /tmp/test.txt
  args:
    removes: /tmp/test.txt
```

Behavior:

* File exists → command runs
* File does not exist → skipped

⚠️ These are **guard conditions**, not true idempotency.

---

## Why `creates` / `removes` Are Not True Idempotency

* They do not enforce desired state
* They do not recover from manual changes
* They only prevent repeated execution

If the file is deleted manually later:

* `command + creates` will not recreate it

---

## Idempotency vs `run_once`

| Concept     | Controls        | Purpose                       |
| ----------- | --------------- | ----------------------------- |
| Idempotency | State change    | Safe re-runs                  |
| `run_once`  | Execution count | Limit task execution per play |

They solve **different problems**.

---

## Idempotency and Handlers

Handlers rely on idempotent tasks:

```yaml
- name: Update nginx config
  copy:
    src: nginx.conf
    dest: /etc/nginx/nginx.conf
  notify: restart nginx
```

* If file unchanged → handler not triggered
* If file changed → handler runs

This works because the module is idempotent.

---

## Common Issues / Errors

* Overusing `shell` instead of native modules
* Assuming playbooks themselves are idempotent
* Confusing idempotency with `run_once`
* Restarting services directly instead of handlers

---

## Best Practices

* Always prefer native, state-based modules
* Assume playbooks will be re-run
* Use handlers for service restarts
* Use `command` / `shell` only when no module exists

---
---
---
