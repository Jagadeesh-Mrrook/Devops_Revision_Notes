# Terraform Expressions & Functions Notes

## 1. Terraform Expressions

Expressions allow logic, computation, and dynamic values inside Terraform configurations.

**Examples:**

* Combine strings → `"app-${var.env}-server"`
* Arithmetic → `${var.count + 2}`
* Conditionals → `${var.env == "prod" ? 3 : 1}`

**Use Cases:**

* Dynamically naming resources.
* Deciding instance counts based on environment.
* Referencing other resource attributes.

---

## 2. Terraform Functions

Functions transform or compute values inside expressions. Used to manipulate strings, numbers, collections, etc.

**Example:**

```hcl
bucket_name = format("app-%s-%s", var.env, random_id.bucket.hex)
```

Here, `format()` is a built-in Terraform function.

---

## 3. Must-Know Terraform Functions for DevOps Engineers

### 🔹 String Functions

* `format()`: Format strings — `format("app-%s", var.env)`
* `join()`: Join list items — `join(",", ["a", "b"])`
* `split()`: Split strings — `split(",", "a,b,c")`
* `replace()`: Replace text — `replace(var.name, "_", "-")`
* `lower()` / `upper()`: Convert case
* `trimspace()`: Remove spaces

### 🔹 Numeric Functions

* `min()` / `max()` — Compare numbers.
* `ceil()` / `floor()` — Round values.

### 🔹 Collection Functions

* `length()`: Count items — `length(var.list)`
* `contains()`: Check presence — `contains(var.list, "dev")`
* `element()`: Get element by index.
* `lookup()`: Safe map lookup — `lookup(var.map, "key", "default")`
* `merge()`: Combine maps.
* `toset()` / `tolist()`: Convert list ↔ set.
* `zipmap(keys, values)`: Combine two lists into a map.

### 🔹 Date & Time

* `timestamp()`: Current UTC timestamp.

### 🔹 Type Conversion

* `tostring()`, `tonumber()`, `tolist()`, `toset()`, `tomap()`

### 🔹 Others

* `file()`: Read local file.
* `jsonencode()` / `jsondecode()`: JSON conversions.
* `coalesce()`: Return first non-null value.
* `cidrsubnet()`: Network calculations.

---

## 4. Commonly Used Conditional Expressions

```hcl
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
```

**Syntax:** `condition ? true_value : false_value`

**Example Use Case:** Different instance types for prod and dev.

---

## 5. Operators

| Type       | Examples                         | Description                  |        |               |
| ---------- | -------------------------------- | ---------------------------- | ------ | ------------- |
| Arithmetic | `+`, `-`, `*`, `/`, `%`          | Basic math operations        |        |               |
| Comparison | `==`, `!=`, `>`, `<`, `>=`, `<=` | Value comparison             |        |               |
| Logical    | `&&`, `                          |                              | `, `!` | Boolean logic |
| String     | `~=`                             | Regex match (Terraform 1.3+) |        |               |

---

✅ **Tip for Practice:**
Start by learning only these 10 functions well:
`format()`, `join()`, `split()`, `replace()`, `lookup()`, `merge()`, `toset()`, `length()`, `contains()`, `element()`.

Once you’re comfortable, expand to others gradually.

---
---

# Terraform Expressions (Conditionals & Operators) - Detailed Notes

## Concept / What

* **Terraform expressions** are combinations of **values, variables, operators, and functions** that Terraform evaluates to produce a value.
* **Conditionals and operators** are part of expressions, enabling arithmetic, comparison, logical decisions, and dynamic selections.

## Why / Purpose / Use Case in Real-World

* Allows **dynamic and reusable configurations**.
* Enables conditional resource configuration depending on environment or other variables.
* Supports arithmetic and logical computations for counts, sizes, or boolean checks.

**Practical Examples:**

* Choosing EC2 instance type based on environment.
* Calculating total disk size dynamically.
* Conditional creation of resources based on subnet availability.

## How It Works / Steps / Syntax

### Operators

* **Arithmetic:** `+`, `-`, `*`, `/`, `%` (e.g., `total = var.a + var.b`)
* **Comparison:** `==`, `!=`, `>`, `<`, `>=`, `<=` (e.g., `is_prod = var.env == "prod"`)
* **Logical:** `&&`, `||`, `!` (e.g., `create = var.enabled && var.has_subnet`)
* **String/Regex (Terraform >=1.3):** `~=` (e.g., `valid_name = var.name =~ "^app-.*$"`)

### Conditional Expression (Ternary)

* Syntax: `condition ? true_value : false_value`
* Example:

```hcl
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
```

* Reads: *if environment is prod, use t3.large; otherwise t3.micro.*

## Common Issues / Errors

| Error                     | Cause                                          |
| ------------------------- | ---------------------------------------------- |
| Invalid function argument | Mismatch in operator or value type             |
| Type mismatch             | String used in numeric operation or vice versa |
| Unknown variable          | Misspelled or undefined variable               |
| Syntax error              | Missing `? :` in conditional expression        |
| Invalid index             | Indexing a set (unordered collection)          |

## Troubleshooting / Fixes

* Use `terraform console` to test expressions live:

```bash
terraform console
> var.env == "prod" ? "t3.large" : "t3.micro"
"t3.large"
```

* Check variable types explicitly (`string`, `number`, `list`, `map`, `set`).
* Use type conversion functions if needed: `tostring()`, `tolist()`, `toset()`.
* Validate with `terraform validate` before applying changes.

## Best Practices / Tips

* Keep expressions readable; avoid overly complex one-liners.
* Use **locals** for computed values:

```hcl
locals {
  instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
}
```

* Combine expressions with functions carefully and test in `terraform console`.
* Use meaningful variable names to make expressions self-explanatory.

---
---

# Terraform Built-in Functions - Detailed Notes

## Concept / What

* **Functions** are predefined helpers in Terraform that take inputs, process them, and return outputs.
* They are used **inside expressions** to compute values dynamically.
* Categories include **string, numeric, collection, date/time, type conversion, JSON, and file functions**.

## Why / Purpose / Use Case in Real-World

* Avoid manual calculations or concatenation.
* Improve readability and maintainability of code.
* Enable dynamic resource naming, iteration, and value transformations.

**Practical Examples:**

1. Dynamic bucket name:

```hcl
bucket_name = format("app-%s-%s", var.env, random_id.bucket.hex)
```

2. Combine multiple values into a list:

```hcl
tags = concat(["env:${var.env}"], var.additional_tags)
```

3. Safe map lookup:

```hcl
subnet_id = lookup(var.subnets, var.env, "subnet-0000")
```

## How It Works / Steps / Syntax

### String Functions

* `format()`, `join()`, `split()`, `replace()`, `lower()`, `upper()`, `trimspace()`

### Numeric Functions

* `min()`, `max()`, `ceil()`, `floor()`, `abs()`

### Collection Functions

* `length()`, `contains()`, `element()`, `lookup()`, `merge()`, `concat()`, `toset()`, `tolist()`, `zipmap()`

### Date & Time

* `timestamp()`

### Type Conversion

* `tostring()`, `tonumber()`, `tolist()`, `toset()`, `tomap()`

### Other Functions

* `file()`, `jsonencode()`, `jsondecode()`, `coalesce()`, `cidrsubnet()`

**Example Usage:**

```hcl
bucket_name = format("app-%s-%s", var.env, random_id.bucket.hex)
instance_count = length(var.instances)
tags = merge(var.default_tags, var.extra_tags)
```

## Common Issues / Errors

| Error                   | Cause                            |
| ----------------------- | -------------------------------- |
| Invalid argument        | Wrong type passed to function    |
| Function not recognized | Misspelled function name         |
| Type mismatch           | Using list where string expected |
| Index out of range      | Using `element()` incorrectly    |

## Troubleshooting / Fixes

* Check function documentation for input/output types.
* Test in `terraform console`:

```bash
> format("app-%s", "dev")
"app-dev"
```

* Use type conversion functions if needed.
* Validate configuration with `terraform validate`.

## Best Practices / Tips

* Focus on **must-know functions** for DevOps scripts:

  * `format()`, `join()`, `split()`, `replace()`, `length()`, `contains()`, `element()`, `lookup()`, `merge()`, `toset()`
* Use **locals** for complex expressions.
* Combine functions with expressions for dynamic, reusable modules.


---
---

# Terraform - Using Expressions in Resource Arguments (Detailed Notes)

## Concept / What

* Terraform allows using **expressions** inside resource arguments to dynamically calculate or assign values.
* These expressions can reference **variables**, **resource attributes**, or **functions**.
* Instead of hardcoding, Terraform evaluates these expressions during the **plan** and **apply** stages.

**Example:**

```hcl
resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
  tags = {
    Name = format("web-%s", var.env)
  }
}
```

Here, expressions decide instance type and tag name dynamically.

---

## Why / Purpose / Use Case in Real-World

* **Avoid repetition:** One resource definition works for multiple environments (prod/dev/staging).
* **Dynamic configuration:** Change attributes like size, tags, and counts using variable-based logic.
* **Automation:** Automatically compute names, IDs, and dependencies.
* **Modularity & reusability:** Reuse the same Terraform code with minimal edits.

**Example:**

```hcl
resource "aws_ebs_volume" "data" {
  size              = var.env == "prod" ? 100 : 50
  availability_zone = data.aws_availability_zones.available.names[0]
}
```

The EBS volume size changes automatically based on the environment.

---

## How It Works / Syntax

Expressions can include:

* **Variable references** → `var.env`
* **Function calls** → `format()`, `lookup()`, `merge()`
* **Arithmetic or logical operators** → `+`, `==`, `&&`
* **Conditionals** → `condition ? true_value : false_value`
* **Resource references** → `aws_instance.example.id`
* **For-expressions / dynamic lists** → `[for s in var.subnets : s.id]`

**Example combining all:**

```hcl
resource "aws_s3_bucket" "app" {
  bucket = format("app-%s-%s", var.env, random_id.bucket.hex)
  acl    = var.env == "prod" ? "private" : "public-read"
  tags = merge(
    var.default_tags,
    { Environment = var.env }
  )
}
```

---

## Common Issues / Errors

| Error                         | Cause                                              |
| ----------------------------- | -------------------------------------------------- |
| **Unknown variable**          | Using undeclared or misspelled variable name       |
| **Invalid function argument** | Incorrect data type or value passed to function    |
| **Circular dependency**       | Referring to a resource before it’s created        |
| **Type mismatch**             | Mixing incompatible types (e.g., string with list) |
| **Invalid reference**         | Accessing a resource output that doesn’t exist yet |

---

## Troubleshooting / Fixes

* Use `terraform console` to test expressions:

  ```bash
  > format("app-%s", var.env)
  "app-dev"
  ```
* Run `terraform validate` before applying.
* Use **`depends_on`** when Terraform cannot infer dependencies.
* Add **outputs** to debug computed values:

  ```hcl
  output "bucket_name" {
    value = aws_s3_bucket.app.bucket
  }
  ```
* Use **locals** to simplify and debug complex logic.

---

## Best Practices / Tips

* Avoid complex inline expressions; move them to **locals**.
* Use descriptive variable names for clarity.
* Keep logic deterministic—avoid functions that change values per apply.
* Use expressions for environment-specific customization instead of separate `.tf` files.
* Combine expressions with built-in functions for maximum flexibility.

---

**Summary:**
Using expressions in resource arguments helps make Terraform configurations **modular, reusable, and environment-aware**, reducing manual edits and improving automation consistency.


---
---

# Terraform - Dynamic Blocks (`dynamic` keyword) (Updated Detailed Notes)

## Concept / What

* **Dynamic blocks** allow Terraform to **generate nested blocks dynamically** inside a resource or module.
* Used when the **number of nested blocks varies** based on input variables (lists or maps).
* Works together with **`for_each`** to loop over elements and generate multiple blocks programmatically.
* Keys in the nested block (like `from_port`, `to_port`, `protocol`) are defined **once** in the block, and values are dynamically applied from the variable list/map.

**Example (Optimized):**

```hcl
variable "ingress_ports" {
  default = [80, 443, 8080, 9090]  # only values
}

resource "aws_security_group" "web_sg" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"          # same for all
      cidr_blocks = ["0.0.0.0/0"]  # same for all
    }
  }
}
```

Here, Terraform generates **one ingress block per port** dynamically.

---

## Why / Purpose / Use Case in Real-World

* Avoids manual duplication of nested blocks.
* Supports **dynamic and modular infrastructure**.
* Useful for resources with multiple **ingress/egress rules, IAM policy statements, or listener rules**.
* Enables **environment-specific or variable-based block creation**.
* Minimizes repetition by defining keys only once, and changing **values dynamically**.

---

## How It Works / Syntax

```hcl
dynamic "block_name" {
  for_each = <list_or_map>
  content {
    key1 = each.value       # reused key
    key2 = static_value     # same for all elements
  }
}
```

* `block_name` → nested block to generate
* `for_each` → iterates over a list/map variable
* `content {}` → defines the block content
* `each.key` / `each.value` → references current element in iteration

---

## Common Issues / Errors

| Error                       | Cause                                            |
| --------------------------- | ------------------------------------------------ |
| Invalid for_each argument   | Expression not a valid list/map                  |
| Unknown variable            | Variable not declared or misspelled              |
| Attribute not found         | Key in each.value does not exist                 |
| Dynamic block not supported | Certain provider nested blocks cannot be dynamic |

---

## Troubleshooting / Fixes

* Test `for_each` expressions in `terraform console`.
* Use `jsonencode()` for complex objects.
* Validate variable types (e.g., `list(object({ ... }))`).
* Always use `each.key` / `each.value` correctly.
* Reuse keys inside content block, only provide variable values to reduce repetition.

---

## Best Practices / Tips

* Keep `content` blocks simple.
* Use **locals** for complex dynamic logic.
* Don’t overuse dynamic blocks — only when necessary.
* Avoid mixing `count` and `for_each` in the same resource.
* Test logic in `terraform console` before apply.
* Dynamic block → nested block creation, for_each → loop through values/keys.
* Define nested block keys once, reuse them, and pass only variable values to the dynamic block for cleaner code.

---
---

# Terraform - for_each vs count (Detailed Notes)

## Concept / What

* **count** and **for_each** are Terraform meta-arguments to create multiple instances of a resource or module.
* **count** → simple numeric repetition.
* **for_each** → key/value or set-based iteration for named resources.
* **Deletion behavior differs:** `count` deletes last instances only; `for_each` allows selective deletion.

---

## Why / Purpose / Use Case

* Avoid duplication of resource blocks.
* Dynamically scale infrastructure based on variables.
* Useful for EC2 instances, S3 buckets, IAM users, etc.
* `for_each` gives **fine-grained control** over resource creation and deletion.

---

## How It Works / Syntax

**1. count** (simple numeric scaling)

```hcl
variable "num_instances" { default = 3 }

resource "aws_instance" "web" {
  count         = var.num_instances
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  tags = { Name = "web-${count.index}" }
}
```

* Access instances via `aws_instance.web[0]`, `[1]`, `[2]`.
* Reducing count deletes last instances only.

**2. for_each** (key/value-based scaling)

* **For lists:** convert to set using `toset()`

```hcl
variable "ec2_names" { default = ["app1", "app2", "app3"] }

resource "aws_instance" "web" {
  for_each      = toset(var.ec2_names)
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  tags = { Name = each.value }
}
```

* **For maps:** use directly

```hcl
variable "ec2_map" {
  default = {
    app1 = "t2.micro"
    app2 = "t2.small"
  }
}

resource "aws_instance" "web" {
  for_each      = var.ec2_map
  ami           = "ami-12345678"
  instance_type = each.value
  tags = { Name = each.key }
}
```

* Access instances via key: `aws_instance.web["app1"]`.

---

## Deletion Behavior

* **count:** only the last `n` instances are deleted when reducing count.
* **for_each:** remove the key/value from the list/map → Terraform deletes that specific resource.
* `for_each` provides **fine-grained deletion control**.

---

## Common Issues / Errors

| Error                     | Cause                                            |
| ------------------------- | ------------------------------------------------ |
| Invalid for_each argument | Expression not a valid list/set/map              |
| Duplicate values in list  | Terraform requires unique keys for for_each      |
| Mixing count & for_each   | Not allowed in the same resource                 |
| Unexpected deletion       | Changing count or for_each without checking plan |

---

## Troubleshooting / Fixes

* Use `terraform plan` to preview changes.
* Convert lists to sets using `toset()` for for_each.
* Check variable types before applying.
* Avoid mid-project changes to count or for_each without careful planning.

---

## Best Practices / Tips

* Use **count** for identical resource repetition.
* Use **for_each** for named/distinct resources.
* Lists → convert to set with `toset()`.
* Maps → use directly.
* For selective deletion and safer scaling, prefer `for_each`.
* Always test in `terraform plan` before `apply`.

---
---

# Terraform - For Expressions (Detailed Notes)

## Concept / What

* **For expressions** allow Terraform to **iterate over lists, sets, or maps** and produce new collections.
* They are **not traditional loops**; they are expressions that return a derived list, set, or map.
* Can include **conditional filtering** with `if` clauses.
* Can be used in **resource arguments, locals, modules, variables, or outputs**.

---

## Why / Purpose / Use Case

* To **transform or derive collections** dynamically.
* Avoids manual duplication of values.
* Useful for:

  * Creating dynamic resource arguments (e.g., multiple ingress rules per subnet)
  * Formatting or filtering variable lists/maps
  * Passing transformed collections to modules or outputs
* Improves **modularity, readability, and reduces errors**.

---

## How It Works / Syntax

**1. List Transformation**

```hcl
variable "names" { default = ["app1", "app2", "app3"] }

output "upper_names" {
  value = [for n in var.names : upper(n)]
}
```

* Loops over `var.names`.
* Applies `upper()` to each element.
* Produces new list: `["APP1","APP2","APP3"]`

**2. Map Transformation**

```hcl
variable "ports" { default = [80, 443, 8080] }

output "port_map" {
  value = { for p in var.ports : "port-${p}" => p }
}
```

* Creates a map with keys `port-80`, `port-443`, etc.

**3. Filtering with `if`**

```hcl
variable "numbers" { default = [1,2,3,4,5,6] }

output "even_numbers" {
  value = [for n in var.numbers : n if n % 2 == 0]
}
```

* Produces `[2,4,6]` by filtering even numbers.

**4. Using in resource arguments**

```hcl
variable "subnets" { default = ["10.0.1.0/24","10.0.2.0/24"] }

resource "aws_security_group" "sg" {
  name = "web-sg"
  ingress = [
    for cidr in var.subnets : {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = [cidr]
    }
  ]
}
```

* Generates one ingress rule per subnet dynamically.

**5. Locals and Modules**

* Locals:

```hcl
locals { upper_names = [for n in var.names : upper(n)] }
```

* Modules:

```hcl
module "web_ports" { source = "./module" ports = [for p in var.ports : p] }
```

* Dynamically transform before passing values.

---

## Common Issues / Errors

* Syntax mistakes: missing colon `:` after iteration variable.
* Applying for expression on non-collection types.
* Type mismatch: assigning list expression to a map variable.
* Duplicate keys in map expressions.

---

## Troubleshooting / Fixes

* Test expressions in `terraform console`.
* Ensure variable type matches output type.
* Use unique keys for map transformations.
* Keep iteration variable consistent inside expression.
* For lists, consider `toset()` if needed in for_each later.

---

## Best Practices / Tips

* Use meaningful iteration variable names (e.g., `name`, `item`, `subnet`).
* Combine with functions (`upper()`, `lower()`, `replace()`, `format()`) for dynamic transformations.
* Use `if` conditions for filtering and conditional logic.
* Can be used in **resource arguments, locals, modules, variables, or outputs**, not just outputs.
* Keep expressions readable; consider breaking complex transformations into **locals**.

---
---

# Terraform - depends_on (Detailed Notes)

## Concept / What

* `depends_on` is a Terraform **meta-argument** to **explicitly declare dependencies** between resources or modules.
* Ensures Terraform **creates, updates, or destroys resources in the correct order**.
* Normally, Terraform auto-detects dependencies via references (Dependency Graph).
* `depends_on` is used when **explicit ordering is required**.

---

## Why / Purpose / Use Case

* Terraform auto-handles most dependencies using the **Dependency Graph**.
* Some resources do not have explicit references, so Terraform may not know the correct order.
* Use `depends_on` to:

  * Ensure a resource waits for another resource to be created or modified.
  * Control module execution order.
  * Avoid race conditions in complex infrastructure setups.

**Examples of use cases:**

* Security group creation **after VPC**.
* IAM policy attachment **after IAM role** creation.
* Module execution **after database** provisioning.

---

## How It Works / Syntax

**Resource example:**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  depends_on = [aws_vpc.main, aws_security_group.sg]
}
```

* Ensures `aws_vpc.main` and `aws_security_group.sg` are created first.

**Module example:**

```hcl
module "app" {
  source = "./app_module"

  depends_on = [aws_db_instance.main]
}
```

* Module execution waits until `aws_db_instance.main` is ready.

---

## Common Issues / Errors

* Unnecessary use can **slow down Terraform apply**.
* Cannot directly depend on **dynamic resources** created with `count` or `for_each` using the base resource name; must reference specific instances.
* Circular dependencies may occur if misused.

---

## Troubleshooting / Fixes

* Remove unnecessary `depends_on` to improve execution speed.
* Reference specific instances for resources with `count` or `for_each`:

```hcl
depends_on = [aws_instance.web[0], aws_instance.web[1]]
```

* Use `terraform plan` to check execution order.
* Avoid circular dependency references.

---

## Best Practices / Tips

* Only use `depends_on` when Terraform cannot auto-detect dependencies.
* Combine carefully with `count`/`for_each` resources.
* Keep dependencies explicit for critical resources (networking, IAM, DBs).
* Use `terraform graph` to visualize and debug dependency chains.
* Prefer implicit dependencies when possible; explicit dependencies are for special cases.

---
---

### Terraform Locals – Detailed Explanation Notes

**Concept / What**
Locals in Terraform are used to define **computed or derived values** inside a module.
They help avoid repeating logic, simplify complex expressions, and keep configuration DRY (Don’t Repeat Yourself).
Locals are **module-scoped** and **cannot be overridden** by CLI, `.tfvars`, or other files.

**Why / Purpose / Use Case in Real-World**
Compute values that are reused across resources in the same module.
Generate dynamic resource names, tags, or merged maps.
Avoid duplication of expressions or long logic blocks.
Use for environment-specific derived values without external overrides.

**How it Works / Steps / Syntax**
Define locals in a Terraform file:

```hcl
locals {
  env_name       = var.env
  instance_type  = var.env == "prod" ? "t3.large" : "t2.micro"
  common_tags    = { App = var.app_name, Environment = var.env }
  full_name      = "${var.app_name}-${var.env}"
}
```

Use the local value in a resource block:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-123456"
  instance_type = local.instance_type
  tags          = local.common_tags
}
```

**Common Issues / Errors**
Cannot override locals from CLI, `.tfvars`, or other modules.
Overly complex locals can make code hard to read.
Referencing undefined locals causes Terraform plan/apply to fail.

**Troubleshooting / Fixes**
Check that the local is properly defined in the same module.
Simplify long expressions or break into multiple locals.
Use `terraform console` to evaluate locals interactively.

**Best Practices / Tips**
Use locals to **centralize reusable logic** like names, tags, or computed defaults.
Keep expressions clear and readable; avoid deeply nested locals.
Combine maps and `merge()` or `lookup()` for flexible configurations.
Use variables for any value that needs to be overridden externally.

---
---

### Terraform Variables vs Locals – Detailed Explanation Notes

**Concept / What**
Variables in Terraform are **inputs** to modules or resources and can be configured externally via `.tfvars`, CLI, or environment variables.
Locals are **computed or derived values** defined inside a module and are used for internal logic, simplification, and reusability.

**Why / Purpose / Use Case in Real-World**
Variables are used when values need to be **flexible or environment-specific**, such as instance type, region, AMI ID, or counts.
Locals are used when values are **computed, reused multiple times**, or derived from other variables to avoid repetition, simplify complex logic, or generate dynamic resource names/tags.

**How it Works / Steps / Syntax**
Variables example:

```hcl
variable "env" {
  default = "dev"
}
resource "aws_instance" "web" {
  instance_type = var.env == "prod" ? "t3.large" : "t2.micro"
}
```

Locals example:

```hcl
locals {
  instance_type = var.env == "prod" ? "t3.large" : "t2.micro"
  tags          = { App = "myapp", Environment = var.env }
}
resource "aws_instance" "web" {
  instance_type = local.instance_type
  tags          = local.tags
}
```

**Common Issues / Errors**
Variables not provided and no default → Terraform fails.
Type mismatch between variable definition and value.
Locals cannot be overridden externally; trying to pass via CLI or `.tfvars` has no effect.

**Troubleshooting / Fixes**
Provide missing variables using CLI, `.tfvars`, or default values.
Check variable types match the assigned values.
Use `terraform console` to inspect local values.

**Best Practices / Tips**
Use variables for values that **need external overrides**.
Use locals for **derived, reusable, or computed values** inside a module.
Keep locals simple, readable, and avoid excessive nesting.
Document variables and locals clearly for team collaboration.

---
---

# Map of Objects in Terraform

## What is a Map of Objects?

A **map of objects** is a Terraform variable type where:

* You have **multiple keys** (like a map)
* Each key contains **multiple attributes** (like an object)

This is the most powerful and flexible structure in Terraform for creating multiple resources with different configurations.

---

## Why Use Map of Objects?

Use a map of objects when:

* You want to create **multiple resources dynamically**
* Each resource needs **multiple attributes** (not just one)
* You need clean, scalable, production‑level Terraform code

Examples:

* Multiple EC2 instances with different AMIs, instance types, disks, tags
* Multiple NACL rules with different ports, protocols, rule numbers
* Multiple SG rules with dynamic configurations
* Multiple subnets with different CIDRs and AZs

---

## Example: Map of Objects Structure

### variables.tf

```hcl
variable "servers" {
  type = map(object({
    ami           = string
    instance_type = string
    volume_size   = number
  }))
}
```

### terraform.tfvars

```hcl
servers = {
  web = {
    ami           = "ami-111"
    instance_type = "t2.micro"
    volume_size   = 20
  }

  app = {
    ami           = "ami-222"
    instance_type = "t3.micro"
    volume_size   = 30
  }
}
```

### main.tf

```hcl
resource "aws_instance" "servers" {
  for_each      = var.servers

  ami           = each.value.ami
  instance_type = each.value.instance_type

  root_block_device {
    volume_size = each.value.volume_size
  }

  tags = {
    Name = each.key
  }
}
```

---

## How It Works Internally

A map of objects behaves like:

* **map** → you get a key for each resource (web, app)
* **object** → each key contains many attributes

So:

* `each.key` → resource name (web, app)
* `each.value.ami` → AMI value
* `each.value.instance_type` → instance type
* `each.value.volume_size` → disk size

---

## Why It’s Better Than Lists or Simple Maps

| Feature                          | List | Map | Object | Map of Objects |
| -------------------------------- | ---- | --- | ------ | -------------- |
| Multiple resources               | ✔    | ✔   | ❌      | ✔              |
| Multiple attributes per resource | ❌    | ❌   | ✔      | ✔              |
| Best for dynamic infra           | ❌    | ❌   | ❌      | ✔              |

Map of objects gives **maximum flexibility** and is used in real companies.

---

## When to Always Use Map of Objects

* Multiple EC2s with different configs
* Multiple NACL rules
* Multiple SG rules
* Multiple subnets
* Multiple route table associations

If you need different values for different resources → use **map of objects**.

---

## One-Line Memory Trick

**List → many values**
**Map → one value per key**
**Object → many values in one item**
**Map of Objects → many items, each with many values (the most powerful)**

---

Let me know if you want examples for:

* NACL using map of objects
* SG rules using map of objects
* Subnets using map of objects

---
---
---

