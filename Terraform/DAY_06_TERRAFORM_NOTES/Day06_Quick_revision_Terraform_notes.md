### Terraform Provisioners Notes - Quick Revision Version

### Provisioners

* Run scripts/commands after resource creation.
* Types: `local-exec`, `remote-exec`, `file`.
* Use cases: Post-deployment tasks, dynamic configuration.
* Avoid: Tasks already handled by Terraform or user data.
* Key tips: Test scripts, ensure connectivity, prefer configuration management tools.
* Errors: Connection issues, command failures, breaking idempotency.

---

### Terraform Local-Exec Provisioner Notes - Quick Revision Version

### local-exec Provisioner

* Runs commands or scripts **on the local machine** after resource creation.
* Automates tasks to avoid manual execution.
* Common use cases: Trigger local scripts, call APIs, orchestrate tools outside Terraform.
* Syntax: Use `local-exec` block inside a resource and define `command`.
* Common issues: Missing software, incorrect paths, OS differences.
* Tips: Keep scripts idempotent, test manually, prefer calling scripts over complex inline commands, avoid tasks meant for remote servers.


---


## Quick Revision – `remote-exec` Provisioner

**Concept / What:**

* Runs commands/scripts on a remote resource (like EC2) after creation.
* Uses SSH (Linux/Unix) or WinRM (Windows) for connectivity.

**Why / Purpose / Use Case:**

* Automates post-provision configuration (install packages, start services).
* Useful when cloud-init/user data is insufficient.
* Helps execute dynamic tasks using created resource data.

**How it Works / Steps / Syntax:**

* Add `provisioner "remote-exec" {}` block inside resource.
* Requires `connection {}` block with details (type = ssh/winrm, host, user, private\_key/password).
* Supports `inline` (commands list) or `script` (external file).

**Common Issues / Errors:**

* SSH/WinRM not reachable (security group/firewall issues).
* Wrong credentials (bad key, wrong user).
* Timeout during connection.

**Troubleshooting / Fixes:**

* Ensure security group/firewall allows SSH/WinRM.
* Validate username (e.g., `ubuntu`, `ec2-user`, `admin`).
* Verify private key/credentials.
* Check that the resource is fully up before execution.

**Best Practices / Tips:**

* Use `user_data` for simple boot-time tasks; `remote-exec` only for dynamic post-creation tasks.
* Avoid heavy provisioning (use config management tools like Ansible/Chef for complex setup).
* Keep commands idempotent (safe to re-run).


---

## Quick Revision – `file` Provisioner

**Concept / What:**

* Copies files/directories from local (Terraform host) to remote resource.
* Works over SSH (Linux/Unix) or WinRM (Windows).

**Why / Purpose / Use Case:**

* Automates file transfer after resource creation.
* Example: Upload configs (`nginx.conf`), SSL certs, app binaries, or scripts.
* Saves effort compared to manual `scp`/`rsync`.

**How it Works / Syntax:**

```hcl
provisioner "file" {
  source      = "./localfile"
  destination = "/remote/path/file"
}

connection {
  type        = "ssh"
  user        = "ubuntu"
  private_key = file("~/.ssh/id_rsa")
  host        = self.public_ip
}
```

* `source` = local path
* `destination` = remote path
* Requires `connection` block (SSH/WinRM)

**Common Issues / Errors:**

* Permission denied (wrong user or path).
* Connection timeout (firewall/SG issue).
* Wrong path in source/destination.

**Troubleshooting / Fixes:**

* Use correct remote user (`ubuntu`, `ec2-user`, etc.).
* Ensure destination path has write permissions.
* Open ports (22 for SSH, 5985/5986 for WinRM).
* Validate key/credentials.

**Best Practices / Tips:**

* Use only for small/medium configs; avoid large binaries.
* Combine with `remote-exec` to run uploaded files.
* Secure sensitive files with Vault/SSM.
* Prefer artifact repos (S3, Nexus) for big files.


---

# Null Resource with Triggers – Quick Revision Notes

## Concept / What

* **null\_resource**: Terraform resource that doesn’t create infra, used to run provisioners.
* **triggers**: Map of values that force `null_resource` recreation when changed.

## Why / Purpose / Real-World Use

* Run one-time tasks/commands/scripts without creating infra.
* Automate actions like configuration updates, notifications, script runs.
* Ensure provisioners rerun when specific values change (dynamic conditions).

## How it Works / Steps / Syntax

```hcl
resource "null_resource" "example" {
  provisioner "local-exec" {
    command = "echo Running script..."
  }

  triggers = {
    file_hash = filesha1("script.sh")
  }
}
```

* **Provisioner runs once** at first creation.
* If `triggers` value changes → resource recreated → provisioner re-runs.
* Without triggers → provisioner won’t run again on subsequent applies.

## Common Issues / Errors

* Using `timestamp()` in triggers → forces rerun every `terraform apply`.
* Forgetting to define triggers → provisioners never rerun.
* Unstable trigger values → unnecessary reruns.

## Troubleshooting / Fixes

* Debug with `terraform plan` to see if null\_resource will be replaced.
* Use file hashes (`filesha1`, `filemd5`) instead of `timestamp()` for stable checks.
* Verify triggers are deterministic for predictable behavior.

## Best Practices / Tips

* Use only when necessary (avoid provisioner-heavy workflows).
* Keep triggers minimal and stable.
* Prefer configuration management tools (Ansible, Chef) for heavy tasks.
* Use `filesha1`/`filemd5` for detecting config changes reliably.


---

# Terraform Provisioners – When & Why to Use (Quick Revision)

## Concept / What

* Run scripts/commands on local (`local-exec`) or remote resource (`remote-exec` / `file`) after resource creation.
* Used when Terraform **cannot declaratively manage** a task.

## Why / Use Cases

* Post-creation configuration: DB migration, API calls.
* One-time operational tasks: bootstrap, CMDB registration.
* Existing infra configuration after `terraform import`.
* Copy files/configs: TLS certs, binaries.
* Destroy-time tasks: cleanup, deregistration.

**Avoid for**: standard OS/package setup (use user-data, pre-baked AMIs, or config management).

## Key Points / Syntax

* Provisioner **inside resource block**.
* `file` before `remote-exec` for file copying.
* `null_resource + triggers` for non-resource tasks.
* Destroy-time: `when = "destroy"`.
* Control failures: `on_failure = continue`.
* Manage order: `depends_on`.

## Common Errors

* Connection failure, timing issues, non-idempotent scripts.
* Secrets leakage, failed provisioners taint resource, overuse.

## Fixes / Best Practices

* Make scripts idempotent.
* Use retries/waits.
* `on_failure = continue` cautiously.
* Prefer pre-baked images, user-data, or config management.
* Document purpose; use `null_resource + triggers` for one-time orchestration.

# Terraform Provisioners – Avoiding Anti-Patterns (Quick Revision)

## Concept / What

* Anti-patterns are **bad practices** with provisioners that make Terraform code procedural, brittle, or non-idempotent.
* Provisioners introduce **imperative behavior**.

## Why / Use Cases

* Use provisioners **only as last resort**.
* Avoid installing standard software, chaining provisioners, or relying on non-idempotent scripts.
* Anti-patterns lead to hidden dependencies, taints, and unpredictable infra.

## Key Points / Examples

* OS/package installs → use **user-data** or **pre-baked images**.
* Chaining provisioners across resources → use `depends_on` or external orchestration.
* Provisioners on imported resources → risk of taints; prefer manual/config management.

## Best Practices

* Keep scripts **idempotent, small, documented**.
* Use **null\_resource + triggers** for one-time tasks.
* Avoid chaining; manage dependencies explicitly.
* Prefer declarative methods over provisioners whenever possible.

---


