# Terraform Input Variables – Detailed Explanation Notes

## Concept / What

* Input variables in Terraform are used to **dynamically supply values** instead of hardcoding them.
* We can define variables as different **data types**: string, list, map, bool.
* They make Terraform code **reusable and flexible** for different environments.

## Why / Purpose / Use Case in Real-World

* Avoid writing multiple configuration files for dev, staging, prod.
* Dynamically supply values while running Terraform.
* Share a single module with different teams or projects.
* Example: instance type, region, AMI ID, subnet IDs.

## How it Works / Steps / Syntax

1. **Define a variable in the Terraform file:**

   ```hcl
   variable "instance_type" {
     type        = string
     default     = "t2.micro"
     description = "EC2 instance type"
   }
   ```
2. **Provide values in a `.tfvars` file:**

   ```hcl
   instance_type = "t3.medium"
   ```
3. **Provide values from CLI at runtime:**

   ```bash
   terraform apply -var="instance_type=t3.large"
   ```
4. **Use the variable in a resource block:**

   ```hcl
   resource "aws_instance" "web" {
     ami           = "ami-123456"
     instance_type = var.instance_type
   }
   ```

## Common Issues / Errors

* Terraform fails if a variable is referenced but not defined.
* Type mismatch (e.g., string vs list).
* Missing variable value and no default provided.

## Troubleshooting / Fixes

* Make sure variable names match across files.
* Check the type matches the value you provide.
* Provide missing variables via CLI, `.tfvars`, or default value.

## Best Practices / Tips

* Always add **description** for variables.
* Use `.tfvars` files for environment-specific values.
* Keep variable names **meaningful and consistent**.
* Avoid hardcoding values; always use input variables for flexibility.

---

# Terraform Default Values & Type Constraints – Detailed Explanation Notes

## Concept / What

* When defining input variables in Terraform, we can add **default values**.
* Default values are used **if no value is supplied** during Terraform run.
* **Type constraints** specify the data type the variable should accept: string, number, bool, list, map.
* This ensures **correct values** and avoids failures during apply.

## Why / Purpose / Use Case in Real-World

* Default values prevent Terraform from **stopping if user doesn’t supply a variable**.
* Type constraints prevent **wrong type values** from being used.
* Makes Terraform configuration **safe, reusable, and robust**.
* Example: default instance type for dev environment, enforcing subnet\_ids as a list.

## How it Works / Steps / Syntax

1. **Default value example:**

   ```hcl
   variable "instance_type" {
     type        = string
     default     = "t2.micro"
     description = "EC2 instance type"
   }
   ```

   * If no value is supplied, Terraform uses `t2.micro`.

2. **Type constraint example (list):**

   ```hcl
   variable "subnet_ids" {
     type        = list(string)
     description = "List of subnet IDs"
   }
   ```

   * Terraform enforces the type and fails if wrong type is provided.

3. **Combining default and type constraint:**

   ```hcl
   variable "enable_monitoring" {
     type    = bool
     default = true
     description = "Enable CloudWatch monitoring"
   }
   ```

4. **Variable value precedence:**

   * Terraform applies variable values in this order of precedence:

     1. **Command-line `-var` option** (highest priority)
     2. **Environment variables**
     3. **`.tfvars` file**
     4. **Default value** (lowest priority)
   * Example: If a variable has a default but also exists in `.tfvars` and provided via CLI, **CLI value will be used**.

## Common Issues / Errors

* Type mismatch (e.g., string given for list variable).
* Default value not compatible with type constraint.
* User overrides default with wrong type via CLI or `.tfvars`.

## Troubleshooting / Fixes

* Check variable type and default value match.
* Validate CLI or `.tfvars` inputs follow the defined type.
* Terraform error messages clearly indicate type mismatch.

## Best Practices / Tips

* Always define type constraints for safety.
* Use defaults for common values to reduce manual input.
* Provide descriptions for clarity.
* Combine defaults, type constraints, and `.tfvars` for **environment flexibility**.
* Remember variable precedence when debugging unexpected values.



---

# Terraform Output Values & Sensitive Outputs – Detailed Explanation Notes

## Concept / What

* **Output values** are used to fetch and display information from Terraform-created resources after they are applied.
* **Sensitive outputs** are special output values for secrets or sensitive information that **won't display on the terminal** by default.

## Why / Purpose / Use Case in Real-World

* Fetch important details like instance IPs, database endpoints, VPC IDs after resource creation.
* Supply outputs to **other modules or scripts** for dynamic infrastructure linking.
* Protect sensitive data like passwords, API keys, or tokens from being exposed.

## How it Works / Steps / Syntax

1. **Basic output block:**

   ```hcl
   output "instance_ip" {
     value       = aws_instance.web.private_ip
     description = "Private IP of EC2 instance"
   }
   ```

   * Displays after `terraform apply`.

2. **Fetch specific output via CLI:**

   ```bash
   terraform output instance_ip
   ```

3. **Sensitive output block:**

   ```hcl
   output "db_password" {
     value     = aws_db_instance.prod.password
     sensitive = true
   }
   ```

   * Hides value on terminal; can still fetch using `terraform output db_password`.

## Common Issues / Errors

* Referencing outputs **before resource creation** → Terraform throws error.
* Typos in output variable names.
* Forgetting `sensitive = true` for secrets, exposing sensitive info.

## Troubleshooting / Fixes

* Ensure the resource exists and has been applied before referencing output.
* Check spelling of output names.
* Use `sensitive = true` for secrets and fetch via CLI if needed.

## Best Practices / Tips

* Always include **description** for outputs.
* Use **sensitive** attribute for passwords, API keys, or secrets.
* Use outputs to **pass values between modules** rather than hardcoding.
* Keep output names **clear and consistent**.
---


# Terraform: Using Variables in Modules and Resources – Detailed Explanation Notes

## Concept / What

* Variables in Terraform can be used both **inside resource blocks** and **passed into modules**.
* They allow configuration values to be dynamic, reusable, and environment-specific instead of hardcoding values.

## Why / Purpose / Use Case in Real-World

* Makes Terraform code **flexible, modular, and maintainable**.
* Resource blocks can adapt dynamically to values supplied via variables (e.g., instance type, subnet IDs).
* Modules can accept input variables from parent modules to **reconfigure resources across environments** (dev, QA, prod).
* Reduces duplicate code and allows a single module or configuration to be reused.

## How it Works / Steps / Syntax

### 1. Using Variables inside Resource Blocks

```hcl
variable "instance_type" {
  type    = string
  default = "t2.micro"
}

resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = var.instance_type
}
```

* `var.instance_type` fetches the variable's value dynamically.
* Default is used if no value is supplied externally.

### 2. Passing Variables into Modules

```hcl
// modules/web/variables.tf
variable "instance_type" {
  type    = string
  default = "t2.micro"
}

// modules/web/main.tf
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = var.instance_type
}

// Parent module
module "webserver" {
  source        = "./modules/web"
  instance_type = "t2.small"  // Overrides module default
}
```

* Modules receive values by **matching variable names** from the parent module.
* Lists, maps, and booleans can also be passed.

### 3. Multiple Variables and Dynamic Usage

```hcl
variable "subnet_ids" { type = list(string) }

module "network" {
  source     = "./modules/network"
  subnet_ids = ["subnet-123", "subnet-456"]
}
```

* Enables dynamic deployment across different subnets or environments.

### 4. Precedence for Variable Values

1. **Runtime via CLI arguments** (`terraform apply -var 'instance_type=t2.large'`)
2. **.tfvars files**
3. **Environment variables**
4. **Default values defined in variables.tf**

* Terraform uses the first available value based on this order.

## Common Issues / Errors

* Variable not defined in resource/module → Terraform throws error.
* Passing wrong type → Terraform apply fails.
* Misnamed variables → Module or resource does not receive value.
* Forgetting to supply variable with no default → Terraform prompts for input.

## Troubleshooting / Fixes

* Ensure all variables are **defined** inside modules or root configuration.
* Validate **type matches** (string, list, map, bool).
* Check **variable names** in parent and module.
* Use `.tfvars` or CLI to provide missing variables.
* Add **default values** to avoid apply interruptions.

## Best Practices / Tips

* Always define **types and default values**.
* Keep variable names **consistent and descriptive**.
* Use **modules for reusable infrastructure**, not copying resource blocks.
* Parameterize as much as possible for **multi-environment support**.
* Document each variable with **description**.
* Combine **variables in resources and modules** to maximize flexibility.

