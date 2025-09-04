---

# Terraform Modules Notes

## Detailed Explanation Version

### Concept / What

* A module in Terraform is a container for multiple resources that are used together.
* Modules help organize code into reusable, logical components.
* Root modules = main working directory, child modules = reusable pieces called by the root module.

### Why / Purpose / Use Case

* Reusability: Write once, use multiple times across environments.
* Maintainability: Changes propagate wherever module is used.
* Abstraction: Hide complex resource definitions behind simple inputs and outputs.
* Real-world examples:

  * VPC module provisioning subnets, route tables, security groups.
  * EC2 module reused for dev, QA, prod with different instance types or counts.

### How it Works / Steps / Syntax

1. Create module folder:

```
modules/
  vpc/
    main.tf
    variables.tf
    outputs.tf
  ec2/
    main.tf
    variables.tf
    outputs.tf
```

2. **Module Inputs (Variables)**

```hcl
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
}
```

* Pass values from root module:

```hcl
module "my_vpc" {
  source   = "./modules/vpc"
  vpc_cidr = "10.0.0.0/16"
}
```

3. **Module Outputs**

```hcl
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}
```

* Access from root: `module.my_vpc.vpc_id`

4. **Calling Modules from Different Sources**

* Local path: `source = "./modules/vpc"`
* Git: `source = "git::https://github.com/org/terraform-modules.git//vpc"`
* Registry: `source = "terraform-aws-modules/vpc/aws"`

5. **Module Versioning**

* For Registry modules: specify version to lock module:

```hcl
source  = "terraform-aws-modules/vpc/aws"
version = "3.15.0"
```

### Common Issues / Errors

* `No module call found` → source path incorrect.
* `Variable required but not provided` → missing module inputs.
* `Module version conflict` → incompatible module versions.
* Circular dependency errors → outputs used as inputs in a loop.

### Troubleshooting / Fixes

* Check source path and syntax.
* Provide all required variables.
* Lock module versions.
* Break circular dependencies using `depends_on` carefully.

### Best Practices / Tips

* Keep modules small and focused.
* Use inputs and outputs for all configurable values.
* Version control modules (Git tags, registry versions).
* Avoid hardcoding environment-specific values.
* Document modules with a README.
* Run `terraform fmt` and `terraform validate`.

## 📘 Terraform Modules with Git Versioning

### Concept / What

* Modules can be sourced directly from Git repositories.
* Instead of Terraform Registry, Git allows using internal or external repos.
* Versioning is done using **tags, branches, or commit hashes** in the `source` URL.

---

### Why / Purpose / Use Case in Real-World

* **Purpose**: Maintain full control over custom modules and their versions.
* **Use Cases**:

  * Internal org modules (VPC, EC2, IAM, Networking).
  * No dependency on external Terraform Registry.
  * Teams lock to tested/stable versions via Git tags.
* **Benefit**: Reproducibility, rollback, and consistency across environments.

---

### How it Works / Steps / Syntax

1. **Reference Git Repository**:

   ```hcl
   module "vpc" {
     source = "git::https://github.com/org/terraform-vpc.git"
   }
   ```

2. **Reference Specific Tag (Recommended)**:

   ```hcl
   module "vpc" {
     source = "git::https://github.com/org/terraform-vpc.git?ref=v1.0.0"
   }
   ```

3. **Reference Branch**:

   ```hcl
   module "vpc" {
     source = "git::https://github.com/org/terraform-vpc.git?ref=feature-branch"
   }
   ```

4. **Reference Commit Hash**:

   ```hcl
   module "vpc" {
     source = "git::https://github.com/org/terraform-vpc.git?ref=abcd123"
   }
   ```

5. **Important Note**:

   * `version =` argument is only valid for **Terraform Registry modules**.
   * For Git-based modules → always use `?ref=` inside the `source`.

---

### Quick Revision Notes

* Git modules use `source` with `?ref=` (tag/branch/commit).
* Tags (`v1.0.0`) → stable, preferred for prod.
* Branches (`dev`, `feature-x`) → ongoing dev.
* Commits (`abcd123`) → pin exact code.
* `version` keyword ❌ not valid here, only for registry.
* Real-world → Git versioning is common for org-specific modules.


---
---

# Terraform Modules Advanced Concepts Notes

## 1️⃣ Using Modules with count

### Concept / What

* Use `count` to create multiple instances of a module or resource dynamically.

### Why / Purpose / Use Case

* Avoid repetition, easily create multiple similar resources (VPCs, EC2 instances).

### How it Works / Steps / Syntax

```hcl
module "vpc" {
  source   = "./modules/vpc"
  count    = 2
  vpc_cidr = ["10.0.0.0/16", "10.1.0.0/16"][count.index]
}
```

* `count.index` picks the correct element from the list.
* For EC2 tags:

```hcl
tags = { Name = "webserver-${count.index}" }
```

### Common Issues / Errors

* Forgetting `count.index` for lists
* Same resource names without interpolation

### Best Practices

* Use lists or maps to provide different inputs per instance
* Access outputs as `module.vpc[0].vpc_id`, `module.vpc[1].vpc_id`

---

## 2️⃣ Using Modules with for\_each

### Concept / What

* `for_each` allows looping over a map or set to create multiple module/resource instances with meaningful keys.

### Why / Purpose / Use Case

* Better than `count` when each instance needs a descriptive key (like dev, qa, prod).

### How it Works / Steps / Syntax

```hcl
module "vpc" {
  source   = "./modules/vpc"
  for_each = { dev = "10.0.0.0/16", qa = "10.1.0.0/16" }
  vpc_cidr = each.value
}
```

* Access outputs: `module.vpc["dev"].vpc_id`

### Best Practices

* Prefer `for_each` over `count` for descriptive keys
* Keep map simple and documented

---

## 3️⃣ Count Index vs String Interpolation vs List Indexing

### Concept / What

* Terraform uses different syntaxes depending on context:

  1. `${count.index}` → inside strings
  2. `[count.index]` → accessing list elements
  3. `each.key` / `each.value` → in `for_each` loops

### Examples

**String interpolation:**

```hcl
tags = { Name = "webserver-${count.index}" }
```

**List indexing:**

```hcl
vpc_cidr = ["10.0.0.0/16", "10.1.0.0/16"][count.index]
```

**for\_each iteration:**

```hcl
for_each = { dev="10.0.0.0/16", qa="10.1.0.0/16" }
vpc_cidr = each.value
```

### Rule of Thumb

* Inside a string → `${}`
* Accessing list/map elements → `[index]` / `[key]`
* Iterating maps/lists → `each.key` / `each.value`
* Function calls → `func()`

---

---

# Terraform Modules – `depends_on` and Module Composition

## Detailed Explanation Version

### 1️⃣ Using `depends_on` Between Modules

**Concept / What**

* `depends_on` explicitly sets the order of module/resource creation.
* Ensures one module finishes before another starts.

**Why / Purpose / Use Case**

* Terraform usually detects dependencies automatically.
* Explicit ordering is needed when outputs are not directly referenced.
* Real-world: EC2 instances should wait for VPC/subnet creation.

**How it Works / Steps / Syntax**

```hcl
module "vpc" {
  source = "./modules/vpc"
}

module "ec2" {
  source     = "./modules/ec2"
  depends_on = [module.vpc]
}
```

* `terraform apply` enforces creation order.

**Common Issues / Errors**

* Circular dependencies if misused.
* Overuse can slow down Terraform.
* Ignoring implicit dependencies can cause unnecessary explicit `depends_on`.

**Troubleshooting / Fixes**

* Check plan for dependency errors.
* Only add `depends_on` when needed.
* Prefer outputs as inputs over explicit `depends_on`.

**Best Practices / Tips**

* Minimize use; rely on implicit dependencies first.
* Use for critical cross-module dependencies.
* Keep dependency graph simple.

---

### 2️⃣ Module Composition (Calling Modules from Within Modules)

**Concept / What**

* Nesting modules inside another module.
* Parent module calls child modules internally.

**Why / Purpose / Use Case**

* Simplifies complex infrastructure.
* Promotes reusability and encapsulation.
* Real-world: `networking` module calls `vpc` + `subnet` internally.
* Root module can call the parent module only.

**How it Works / Steps / Syntax**

```hcl
# Inside networking module
module "vpc" {
  source   = "../vpc"
  vpc_cidr = var.vpc_cidr
}

module "subnet" {
  source      = "../subnet"
  vpc_id      = module.vpc.vpc_id
  subnet_cidr = var.subnet_cidr
}
```

* Root module calls `networking` as a single module.
* Outputs from child modules exposed by parent for root module use.

**Common Issues / Errors**

* Circular dependencies if parent and child reference each other.
* Forgetting to expose outputs in parent module.
* Confusing variable names.

**Troubleshooting / Fixes**

* Validate child modules separately.
* Use descriptive input/output variable names.
* Test parent module before root integration.

**Best Practices / Tips**

* Keep child modules small and focused.
* Parent modules compose complex infrastructure.
* Expose only necessary outputs.
* Document module composition in README.
* Real-world: Root module usually calls both independent child modules and composed parent modules for clarity and maintainability.

















---

