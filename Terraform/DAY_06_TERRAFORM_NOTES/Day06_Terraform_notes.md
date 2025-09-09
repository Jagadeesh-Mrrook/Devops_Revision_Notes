
### Terraform Provisioners Notes - Detailed Version

### Provisioners (General)

* **Concept / What:** Terraform provisioners allow execution of scripts or commands on local or remote machines after resource creation.
* **Why / Purpose / Use Case:**

  * Automate post-deployment configuration not handled by Terraform resources.
  * Examples: Installing software (Apache/Nginx), database initialization, copying configuration files.
  * Used when tasks cannot be handled declaratively in Terraform.
* **How / Steps / Syntax:**

  ```hcl
  resource "aws_instance" "example" {
    ami           = "ami-123456"
    instance_type = "t2.micro"

    provisioner "local-exec" {
      command = "echo 'Hello from Terraform'"
    }
  }
  ```
* **Common Issues / Errors:**

  * SSH/WinRM connection failures.
  * Command/script failures due to dependencies or wrong paths.
  * Overuse can break Terraform's declarative model.
* **Troubleshooting / Fixes:**

  * Ensure network access and correct credentials.
  * Test scripts manually before use.
  * Use proper `connection` block settings.
* **Best Practices / Tips:**

  * Avoid using provisioners for tasks Terraform can handle.
  * Prefer configuration management tools for complex setups.


---

### Terraform Local-Exec Provisioner Notes - Detailed Version

### local-exec Provisioner

* **Concept / What:** The `local-exec` provisioner in Terraform executes commands or scripts on the local machine where Terraform is run, immediately after a resource is created.
* **Why / Purpose / Use Case:**

  * Automates tasks that need to run locally after resource creation.
  * Ensures scripts are executed without manual intervention.
  * Real-world scenarios:

    * Triggering a local script to configure DNS entries.
    * Calling an external API to notify other systems about resource creation.
    * Running scripts that orchestrate other tools outside Terraform.
* **How / Steps / Syntax:**

```hcl
resource "aws_instance" "example" {
  ami           = "ami-123456"
  instance_type = "t2.micro"

  provisioner "local-exec" {
    command = "echo 'Terraform created this EC2 instance'"
  }
}
```

* Use `${}` interpolation to reference resource attributes.
* Ensure the command is executable in the local environment.
* **Common Issues / Errors:**

  * Command fails if the local environment is missing required software or permissions.
  * Incorrect paths to scripts or files.
  * OS differences (Windows vs Linux) causing command errors.
* **Troubleshooting / Fixes:**

  * Verify scripts exist and are executable.
  * Use absolute paths if needed.
  * Test commands manually before applying Terraform.
* **Best Practices / Tips:**

  * Avoid running tasks that should be executed on the remote resource (use `remote-exec` instead).
  * Keep commands/scripts idempotent.
  * Prefer well-defined scripts over complex inline commands.

---

# Terraform `remote-exec` Provisioner – Detailed Notes

## 1. What is `remote-exec`?

* `remote-exec` is a **Terraform provisioner** used to run commands or scripts **on a remote resource** (e.g., an EC2 instance) **after it is created**.
* Unlike `local-exec` (which runs on the local machine where Terraform is executed), `remote-exec` runs inside the **newly created remote server**.

---

## 2. How it Works

1. A resource (like an EC2 instance) is created.
2. Terraform uses the **connection block** to connect to that resource via **SSH** (Linux/Unix) or **WinRM** (Windows).
3. Once the connection is established, the commands/scripts defined in `remote-exec` are executed on that server.

---

## 3. Syntax

```hcl
resource "aws_instance" "example" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update -y",
      "sudo apt-get install -y nginx"
    ]
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }
}
```

---

## 4. Key Components

* **`inline`**: A list of commands to execute sequentially.
* **`script`**: Run a single script file on the remote server.
* **`scripts`**: Run multiple script files on the remote server.
* **`connection` block**: Defines how Terraform connects to the resource.

  * `type` = `ssh` or `winrm`
  * `user` = username for login
  * `private_key` or `password`
  * `host` = resource IP or DNS

---

## 5. Use Cases

* Installing required software (e.g., web servers, monitoring agents).
* Running startup/configuration scripts.
* Bootstrapping applications after VM creation.
* Dynamic configurations based on resource attributes.

---

## 6. Important Points

* `remote-exec` requires **network access** (e.g., security group rules for SSH/WinRM).
* If the connection fails, Terraform will mark the resource creation as failed.
* It should be used carefully because provisioners are considered a **last resort** in Terraform (infrastructure should ideally be configured via images or configuration management tools).

---

## 7. Example with Script

```hcl
provisioner "remote-exec" {
  script = "./setup.sh"
}
```

This will copy `setup.sh` from the local machine and execute it on the remote server.

---

## 8. Best Practices

* Use `remote-exec` only when absolutely required.
* Prefer **cloud-init, Ansible, or pre-baked AMIs** for complex provisioning.
* Keep inline commands minimal.
* Ensure SSH keys or WinRM credentials are securely managed.


---

## Detailed Notes – `file` Provisioner

**Concept / What:**

* The `file` provisioner in Terraform copies files or directories from the local machine (where Terraform runs) to a remote resource (e.g., EC2 instance).
* Works over **SSH (Linux/Unix)** or **WinRM (Windows)** connections.

---

**Why / Purpose / Use Case in Real-World:**

* Automates file transfer right after resource creation.
* Example scenarios:

  * Copy configuration files (e.g., `nginx.conf`) to a server.
  * Upload SSL certificates for secure communication.
  * Deploy application binaries or scripts for further execution.
* Preferred when manual `scp` or `rsync` would be repetitive or error-prone.

---

**How it Works / Steps / Syntax:**

1. Add a `provisioner "file" {}` block inside the resource definition.
2. Define:

   * `source` → Path to local file or directory.
   * `destination` → Path on remote machine.
3. Include a `connection {}` block to authenticate with the remote host.

**Example:**

```hcl
resource "aws_instance" "example" {
  ami           = "ami-123456"
  instance_type = "t2.micro"

  provisioner "file" {
    source      = "./app.conf"
    destination = "/etc/app/app.conf"
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }
}
```

* `source` → Local file path (`./app.conf`).
* `destination` → Remote path (`/etc/app/app.conf`).
* `connection` → Provides SSH details (or WinRM for Windows).

---

**Common Issues / Errors:**

1. **Permission denied** – Remote user doesn’t have rights to write to the destination directory.
2. **Connection timeout** – Security group/firewall blocks SSH/WinRM.
3. **Wrong path** – Incorrect local `source` or remote `destination` path.

---

**Troubleshooting / Fixes:**

* Ensure remote user has write permissions.
* Use correct username (`ubuntu`, `ec2-user`, `admin`).
* Validate SSH key or WinRM credentials.
* Confirm firewall/security group allows access (port 22 for SSH, 5985/5986 for WinRM).

---

**Best Practices / Tips:**

* Use `file` provisioner for **small to medium files/configs**.
* Avoid transferring large binaries – use S3, artifact repositories, or image baking.
* Combine with `remote-exec` to execute uploaded scripts.
* Don’t hardcode sensitive files (use Vault, SSM, or secure storage).
* Keep paths explicit and tested locally before applying.


---

Keep provisioner logic minimal; prefer user data or config management for complex setups.## Provisioners Must Be Inside a Resource

**Key Rule:**

* In Terraform, **provisioners cannot be used independently**.
* They must always be defined **inside a resource block**.

---

**Why:**

* Provisioners are tied to the lifecycle of resources (create, update, destroy).
* Terraform needs a resource event (like creation) to know *when* to run the provisioner.
* Without a resource, provisioners have no trigger point.

---

**Correct Example:**

```hcl
resource "aws_instance" "example" {
  ami           = "ami-123456"
  instance_type = "t2.micro"

  provisioner "local-exec" {
    command = "echo Instance created: ${self.public_ip}"
  }
}
```

**Invalid Example (❌ Not Allowed):**

```hcl
provisioner "local-exec" {
  command = "echo hello"
}
```

This will throw an error because provisioners cannot exist outside a resource.

---

**Best Practice Reminder:**

* Always attach provisioners to the **specific resource** they relate to.
* Keep provisioner logic minimal; prefer user data or config management for complex setups.

---

# Null Resource with Triggers (Detailed Notes)

## What is a Null Resource?

* A special Terraform resource that **does not create any infrastructure**.
* Useful for running **provisioners** (local-exec, remote-exec, file) without tying them to a real infrastructure resource.
* Often used for **one-time tasks, scripts, or automation steps**.

---

## Why Use Null Resource?

* Provisioners must be attached to a resource block.
* When you don’t want to create real infrastructure but still need to:

  * Run commands/scripts after `terraform apply`.
  * Copy configuration files or binaries.
  * Automate small tasks without linking them to real infra.
* Example use cases:

  * Bootstrapping applications after infra creation.
  * Registering DNS entries.
  * Running shell scripts for deployment/testing.

---

## Triggers Argument

* `triggers` is a **map of values** inside `null_resource`.
* Purpose: Helps Terraform decide **when to recreate the null\_resource**.
* Without `triggers`, a `null_resource` + provisioner runs **only once** (at creation).
* With `triggers`, if any value in the map changes, Terraform detects it during the next `terraform apply` and **reruns the provisioners**.

### Example:

```hcl
resource "null_resource" "example" {
  triggers = {
    version = "1.0"
  }

  provisioner "local-exec" {
    command = "echo Running version ${self.triggers.version}"
  }
}
```

* If `version` changes to `1.1`, Terraform will destroy & recreate the `null_resource`, re-running the provisioner.

---

## Special Case: Using Timestamps

```hcl
triggers = {
  always_run = timestamp()
}
```

* Since `timestamp()` changes every second, the provisioner will **rerun on every terraform apply**.
* ⚠️ Risk: Can cause unnecessary re-runs. Use carefully.

---

## Lifecycle of Provisioners in Null Resource

1. First `terraform apply` → provisioner runs.
2. If nothing changes (no triggers updated) → provisioner does not run again.
3. If trigger value changes → null resource is recreated, provisioner runs again.

---

## Real-World Scenarios

* Running DB schema migrations.
* Copying app binaries before deployment.
* Registering apps with monitoring systems.
* Bootstrapping test data into environments.

---

## Best Practices

* Use null\_resource sparingly (avoid overusing).
* Use triggers with meaningful values (e.g., `filemd5()`, `app_version`).
* Avoid timestamp-based triggers unless absolutely needed.
* For complex workflows, prefer external tools (e.g., Ansible, Jenkins) instead of stuffing too much logic in null\_resource.

---

## Troubleshooting

* **Problem**: Provisioner not rerunning.

  * **Fix**: Check triggers, update value to force rerun.
* **Problem**: Null resource runs too often.

  * **Fix**: Avoid `timestamp()` unless intentional.
* **Problem**: SSH/connection errors.

  * **Fix**: Verify connection block when using remote-exec/file provisioners.


---

# Terraform Provisioners – When & Why to Use (Detailed Notes)

## Concept / What

* Provisioners are Terraform blocks used to **run scripts/commands** on local machine (`local-exec`) or remote resource (`remote-exec` / `file`) **after a resource is created**.
* They are used when Terraform **cannot declaratively manage a task**.

## Why / Purpose / Real-World Use

* **Post-creation configuration**: Run scripts that depend on outputs from other resources.

  * Example: Run database migration on app server after DB and server creation.
* **One-time operational tasks**: Bootstrap scripts, notify external systems via API, register services in CMDB.
* **Existing infrastructure configuration**: After importing resources (`terraform import`), provisioners can configure them.
* **Copy files / small configs**: Push TLS certs, app binaries, or config files immediately after resource creation.
* **Destroy-time tasks**: Deregister from load balancer, cleanup files or external systems using `when = "destroy"`.

**When NOT to use:**

* Standard OS or package setup (prefer **user-data**, **pre-baked images**, or **config management**).

## How it Works / Steps / Syntax

1. Attach provisioner inside a resource (cannot be standalone).

```hcl
resource "aws_instance" "app" {
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }
}
```

2. Use `file` provisioner to copy files before running scripts.
3. Use `null_resource` + `triggers` for tasks not tied to actual resources.
4. Destroy-time actions with `when = "destroy"`.
5. Control failure behavior: `on_failure = continue`.
6. Use `depends_on` to manage execution order if required.

## Common Issues / Errors

* **Connection failures**: SSH/WinRM blocked, wrong user/key, network not ready.
* **Timing issues**: Resource not fully available (cloud-init still running).
* **Non-idempotent scripts**: Causes repeated failures or inconsistent state.
* **Secrets leakage**: Hardcoded secrets in scripts.
* **Failed provisioners taint resource**: Triggers recreation on next `terraform apply`.
* **Overuse**: Makes code procedural and brittle.

## Troubleshooting / Fixes

* Verify SSH/connection details (security group, user, host).
* Add retries/waits in scripts.
* Make scripts idempotent.
* Use `on_failure = continue` only when safe.
* Avoid volatile inputs (like timestamp) in triggers.
* Consider pre-baked images, user-data, or config management tools for complex tasks.

## Best Practices / Tips

* Use provisioners as **last resort**.
* Keep scripts small, idempotent, and tested locally.
* Use `file` for small config files; artifact repositories for large binaries.
* Avoid embedding secrets; fetch securely from Vault/SSM.
* Document why each provisioner exists.
* For tasks not tied to a resource, use `null_resource` + `triggers`.
---

# Terraform Provisioners – Avoiding Anti-Patterns (Detailed Notes)

## Concept / What

* Anti-patterns are **bad practices** with provisioners that make Terraform code procedural, brittle, or hard to maintain.
* Terraform is declarative, but provisioners introduce **imperative behavior**; improper use can break idempotency and create hidden dependencies.

## Why / Purpose / Real-World Use

* Provisioners should **only be used as a last resort** when declarative Terraform cannot handle a task.
* Common anti-patterns lead to:

  * Installing standard software via provisioners (should use user-data or pre-baked images).
  * Multiple provisioners chained across resources causing hidden dependencies.
  * Relying on provisioners for tasks Terraform cannot track, causing inconsistent state.
* Avoiding anti-patterns ensures:

  * Infrastructure remains predictable and reproducible.
  * Provisioners do not cause resource taints or unintentional recreations.
  * Team collaboration remains safe without procedural surprises.

## How it Works / Examples

### Anti-Pattern Examples

1. **Using provisioners for OS/package installs**

```hcl
resource "aws_instance" "app" {
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]
  }
}
```

* **Problem:** Standard packages should be installed via user-data or pre-baked AMI.
* **Fix:** Move installation to user-data or config management tool (Ansible/Chef/Puppet).

2. **Chaining provisioners across resources**

```hcl
resource "aws_instance" "web" { ... }
resource "aws_instance" "db" {
  provisioner "remote-exec" {
    command = "initialize-db.sh"
  }
}
```

* **Problem:** DB provisioning depends imperatively on web instance; hidden dependency.
* **Fix:** Use explicit `depends_on` or orchestrate outside Terraform.

3. **Relying on provisioners for post-import setup**

* Using provisioners for configuring imported resources repeatedly can cause taints.
* **Fix:** Run configuration manually or via configuration management.

## Common Anti-Patterns to Avoid

* Installing OS packages/software via provisioners.
* Using provisioners as a substitute for declarative resources.
* Hidden dependencies between provisioners and resources.
* Non-idempotent scripts causing failures/recreation.
* Storing sensitive info directly in scripts.
* Overusing provisioners instead of using pre-baked images, user-data, or external tools.

## Best Practices / Tips

* Provisioners should be **last resort**.
* Prefer:

  1. **User-data scripts** for bootstrapping.
  2. **Pre-baked images / AMIs** for standard software.
  3. **Configuration management tools** for complex setups.
* Keep scripts **idempotent**, small, and documented.
* Use **null\_resource + triggers** for one-time tasks not tied to real resources.
* Avoid chaining provisioners across resources; manage dependencies explicitly.

---

# Terraform SSH Key Management for Provisioners

## 1. Overview

* Terraform uses **provisioners** (remote-exec, file) to configure newly created resources.
* Provisioners require **SSH connectivity** to the target EC2 instances.
* Secure management of SSH keys is critical in production environments.

---

## 2. Key Generation

* Security team generates an SSH key pair on the Ansible/Jenkins EC2 instance.

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible_key
```

* Outputs:

  * `ansible_key` → **private key**
  * `ansible_key.pub` → **public key**

---

## 3. Private Key Storage

* Store the **private key** securely in **AWS Secrets Manager**.
* Ansible or Jenkins agent fetches it dynamically when needed.
* No manual copying of private keys to other machines.

---

## 4. Public Key Injection via Terraform

* Use `aws_key_pair` resource to register the public key in AWS.

```hcl
resource "aws_key_pair" "ansible" {
  key_name   = "ansible-key"                # Custom key name
  public_key = file("~/.ssh/ansible_key.pub")  # Path to public key
}
```

* Attach this key to EC2 instances:

```hcl
resource "aws_instance" "app" {
  ami           = "ami-0abcd1234"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.ansible.key_name
}
```

---

## 5. SSH Connectivity for Provisioners

* Provisioners like `remote-exec` or `file` use the private key to connect:

```hcl
provisioner "remote-exec" {
  inline = [
    "sudo apt-get update -y",
    "sudo apt-get install -y nginx"
  ]
}

connection {
  type        = "ssh"
  user        = "ubuntu"
  private_key = file("/path/to/fetched/private/key")
  host        = self.public_ip
}
```

* The private key is fetched securely from Secrets Manager or Jenkins credentials.

---

## 6. Security Considerations

* **Private key never stored in Terraform code or repository.**
* **IAM Role** attached to Jenkins/Ansible EC2 instance provides temporary credentials to access Secrets Manager.
* Temporary credentials are auto-refreshed by AWS.
* Public key is injected into EC2 instances automatically during creation.

---

## 7. Summary Flow

1. Security team generates SSH key pair on Ansible/Jenkins EC2.
2. Private key stored in Secrets Manager.
3. Terraform `aws_key_pair` uses public key to create key pair in AWS.
4. EC2 instances launched with this key.
5. Terraform provisioners or Ansible connect to EC2 using private key.
6. IAM roles ensure secure and temporary access for fetching secrets.

