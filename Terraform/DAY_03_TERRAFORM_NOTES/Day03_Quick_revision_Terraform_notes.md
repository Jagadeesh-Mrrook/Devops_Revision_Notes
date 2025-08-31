# Terraform Input Variables – Quick Revision Notes

* Input variables let you **dynamically supply values** instead of hardcoding.
* **Data types:** string, list, map, bool.
* **Purpose:** reuse code for dev/staging/prod, share modules, dynamic values.
* **Syntax:**

  ```hcl
  variable "name" {
    type    = string
    default = "value"
  }
  ```
* **Provide values:** CLI (`-var`), `.tfvars` file, default.
* **Use in resources:** `instance_type = var.instance_type`
* **Errors:** undefined variable, type mismatch, missing value.
* **Fixes:** match names, correct types, provide defaults.
* **Best Practices:** add description, use tfvars for environments, avoid hardcoding, meaningful names.


---

# Terraform Default Values & Type Constraints – Quick Revision Notes

* **Default values:** used if no variable value is supplied.
* **Type constraints:** enforce variable type: string, number, bool, list, map.
* **Purpose:** prevent apply failures, ensure correct types, make modules robust.
* **Syntax example:**

  ```hcl
  variable "instance_type" {
    type    = string
    default = "t2.micro"
  }
  ```
* **List example:**

  ```hcl
  variable "subnet_ids" {
    type = list(string)
  }
  ```
* **Variable value precedence:**

  1. CLI `-var`
  2. Environment variables
  3. `.tfvars` file
  4. Default value
* **Errors:** type mismatch, default incompatible with type, wrong override via CLI or tfvars.
* **Fixes:** match types, validate inputs, check Terraform error messages.
* **Tips:** always use type constraints, defaults, descriptions, and tfvars for environment flexibility.

---

# Terraform Output Values & Sensitive Outputs – Quick Revision Notes

* **Output values:** fetch info from resources after apply (IP, DB endpoint, VPC IDs).
* **Sensitive outputs:** hide secrets like passwords or API keys (`sensitive = true`).
* **Syntax example:**

  ```hcl
  output "instance_ip" {
    value = aws_instance.web.private_ip
  }
  output "db_password" {
    value     = aws_db_instance.prod.password
    sensitive = true
  }
  ```
* **Fetch output via CLI:** `terraform output <name>`
* **Common errors:** referencing before apply, typos, exposing secrets.
* **Fixes:** apply resources first, check names, use `sensitive = true`.
* **Tips:** always include descriptions, keep names clear, use outputs for module communication.


---

# Terraform: Using Variables in Modules and Resources – Quick Revision Notes

## Concept / What

* Variables can be used **inside resource blocks** and **passed into modules**.
* Allows dynamic and reusable Terraform configurations.

## Why / Purpose / Use Case

* Makes code **flexible, modular, maintainable**.
* Supports **multi-environment deployments** (dev, QA, prod).
* Reduces **duplicate code**.

## How it Works / Steps / Syntax

* **Resource Block:**

```hcl
resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

* **Module Input:**

```hcl
module "webserver" {
  source        = "./modules/web"
  instance_type = "t2.small"  // overrides default
}
```

* **Multiple variables:** lists, maps, booleans can be passed.
* **Precedence for values:** CLI > .tfvars file > Environment variable > Default value.

## Common Issues / Errors

* Variable undefined or misnamed.
* Wrong data type.
* Missing variable with no default → prompts input.

## Troubleshooting / Fixes

* Ensure variables **exist in module or root**.
* Match **types and names** correctly.
* Use `.tfvars` or CLI for missing values.
* Set **default values** to prevent apply failure.

## Best Practices / Tips

* Define **types and defaults**.
* Keep names **consistent and descriptive**.
* Use **modules for reusable resources**.
* Parameterize for **different environments**.
* Document **descriptions for all variables**.

