# Terraform Expressions & Functions - Quick Revision

## Expressions

* Combine variables, resource attributes, operators, and functions.
* Make configs dynamic and reusable.
* Example: `var.env == "prod" ? "t3.large" : "t3.micro"`
* Used in resource arguments, locals, outputs.

## Must-Know Functions

### String

* `format()`, `join()`, `split()`, `replace()`, `lower()`, `upper()`

### Numeric

* `max()`, `min()`, `ceil()`, `floor()`, `length()`

### Collection

* `merge()`, `lookup()`, `toset()`, `tolist()`, `concat()`, `contains()`, `element()`, `zipmap()`

### Date/Time

* `timestamp()`

### Other

* `coalesce()`, `file()`, `jsonencode()`, `jsondecode()`

## Conditional Expressions

* Syntax: `condition ? true_value : false_value`
* Example: `var.env == "prod" ? "t3.large" : "t3.micro"`

## Operators

* Arithmetic: `+`, `-`, `*`, `/`, `%`
* Comparison: `==`, `!=`, `>`, `<`, `>=`, `<=`
* Logical: `&&`, `||`, `!`
* String: `~=` (regex match)

## Common Issues

* Wrong placeholder count in `format()`
* Type mismatches (list vs number, set vs list)
* Unknown variable names
* Indexing sets (unordered)

## Tips

* Focus first on: `format()`, `join()`, `replace()`, `length()`, `merge()`, `lookup()`, `toset()`, `contains()`, `element()`, `concat()`
* Test expressions in `terraform console`
* Use `locals` for complex logic

---
---

# Terraform Expressions (Conditionals & Operators) - Quick Revision

## Concept

* Expressions = combination of values, variables, operators, and functions.
* Conditionals & operators = arithmetic, comparison, logical, ternary expressions.

## Why

* Dynamic configuration.
* Conditional resource creation.
* Compute counts, sizes, boolean values.

## Syntax / Examples

```hcl
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
total_size = var.base_size + var.extra_size
create_resource = length(var.subnets) > 0 ? true : false
```

## Operators

* Arithmetic: `+ - * / %`
* Comparison: `== != > < >= <=`
* Logical: `&& || !`
* String/Regex: `~=`

## Common Issues

* Type mismatch, unknown variable, syntax errors, invalid index.

## Troubleshooting

* Use `terraform console`.
* Validate types and variables.
* Use `terraform validate`.

## Tips

* Keep expressions readable.
* Use locals for computed values.
* Test in console before applying.

---
---

# Terraform Built-in Functions - Quick Revision

## Concept

* Functions = predefined helpers that take inputs and return outputs.
* Used inside expressions for dynamic value computation.
* Categories: string, numeric, collection, date/time, type conversion, JSON, file.

## Why

* Avoid manual calculations or concatenation.
* Improve readability and maintainability.
* Enable dynamic resource naming and iteration.

## Must-Know Functions

* **String:** `format()`, `join()`, `split()`, `replace()`, `lower()`, `upper()`, `trimspace()`
* **Numeric:** `min()`, `max()`, `ceil()`, `floor()`, `abs()`
* **Collection:** `length()`, `contains()`, `element()`, `lookup()`, `merge()`, `concat()`, `toset()`, `tolist()`, `zipmap()`
* **Date/Time:** `timestamp()`
* **Type Conversion:** `tostring()`, `tonumber()`, `tolist()`, `toset()`, `tomap()`
* **Other:** `file()`, `jsonencode()`, `jsondecode()`, `coalesce()`, `cidrsubnet()`

## Examples

```hcl
bucket_name = format("app-%s-%s", var.env, random_id.bucket.hex)
tags = merge(var.default_tags, var.extra_tags)
instance_count = length(var.instances)
```

## Common Issues

* Invalid argument types, misspelled function name, type mismatch, index out of range.

## Troubleshooting

* Test in `terraform console`, use type conversion if needed, validate with `terraform validate`.

## Tips

* Focus on must-know functions: `format()`, `join()`, `split()`, `replace()`, `length()`, `contains()`, `element()`, `lookup()`, `merge()`, `toset()`.
* Use locals for complex expressions.
* Combine with expressions for dynamic modules.

---
---

# Terraform - Using Expressions in Resource Arguments (Quick Revision)

## Concept

* Expressions = dynamic logic used inside resource arguments.
* Help compute values using variables, functions, and operators.
* Terraform evaluates them during plan/apply.

## Why

* Avoids hardcoding.
* Enables dynamic, reusable, and modular configurations.
* Supports environment-based customization.

## Syntax Examples

```hcl
ami           = var.ami_id
instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
bucket        = format("app-%s-%s", var.env, random_id.bucket.hex)
```

## Expression Types

* Variable references → `var.env`
* Function calls → `format()`, `lookup()`, `merge()`
* Operators → `+`, `==`, `&&`
* Conditionals → `condition ? true : false`
* Resource references → `aws_instance.example.id`

## Common Issues

* Unknown variable or wrong reference name.
* Invalid function argument or type mismatch.
* Circular dependency between resources.

## Troubleshooting

* Use `terraform console` to test expressions.
* Run `terraform validate` before apply.
* Add outputs to debug computed values.
* Use `depends_on` for explicit dependencies.

## Tips

* Keep expressions short and readable.
* Use **locals** for complex calculations.
* Prefer dynamic expressions over multiple .tf files.
* Use expressions + functions to build environment-aware infrastructure.

---
---

# Terraform - Dynamic Blocks (`dynamic` keyword) (Quick Revision)

* **Dynamic blocks** → generate nested blocks dynamically inside a resource or module.
* Used with **for_each** → iterates over list/map values to create multiple nested blocks.
* Keys (like from_port, to_port, protocol) defined **once**; values taken from variable list/map.

**Example:**

```hcl
variable "ingress_ports" { default = [80, 443, 8080, 9090] }

resource "aws_security_group" "web_sg" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

* **Purpose:** avoid repetition, support modular infra, environment-based blocks.
* **Common Issues:** invalid for_each, unknown variable, attribute not found, unsupported dynamic block.
* **Tips:** keep content simple, reuse keys, use locals for complex logic, test in console, avoid mixing count & for_each.

---
---

# Terraform - for_each vs count (Quick Revision)

* **count** → numeric repetition; access via index `[0]`, `[1]`.
* **for_each** → iterate over set/map; access via `each.key` / `each.value`.
* **Lists for for_each** → convert to set using `toset()`.
* **Maps for for_each** → use directly.
* **Deletion:**

  * count → reduces number; last instances deleted.
  * for_each → remove key/value; specific resource deleted.
* **Best practice:**

  * count for identical resources.
  * for_each for named/distinct resources.
* Always run `terraform plan` before apply to check changes.

---
---

# Terraform - For Expressions (Quick Revision)

* **For expressions** → iterate over lists, sets, or maps to produce new collections.
* **Not traditional loops**; used inside expressions.
* Can be used in **resource arguments, locals, modules, variables, outputs**.
* **Syntax (list):** `[for item in var.list : transformation]`
* **Syntax (map):** `{ for k,v in var.map : k => v }`
* Can include **filtering**: `[for n in var.list : n if condition]`
* **Examples:**

  * Transform list: `[for n in var.names : upper(n)]`
  * Create map: `{ for p in var.ports : "port-${p}" => p }`
  * Filter: `[for n in var.numbers : n if n % 2 == 0]`
* Use meaningful iteration variable names (`item`, `name`, `cidr`).
* Combine with functions (`upper()`, `lower()`, `replace()`, `format()`).
* Test expressions with `terraform console`.
* Ensure types match and map keys are unique.

---
---

# Terraform - depends_on (Quick Revision)

* `depends_on` → explicitly declare resource/module dependencies.
* Ensures resources are **created, updated, or destroyed in the correct order**.
* Terraform auto-detects most dependencies using **Dependency Graph**.
* Use when Terraform **cannot infer dependency** or for **explicit ordering**.
* **Syntax (resource):**

```hcl
depends_on = [aws_vpc.main, aws_security_group.sg]
```

* **Syntax (module):**

```hcl
depends_on = [aws_db_instance.main]
```

* **Common issues:** unnecessary use slows apply, cannot depend on dynamic resources directly, circular dependencies.
* **Fixes:** reference specific instances for count/for_each, check plan, remove unnecessary dependencies.
* **Best practices:** use only when needed, combine carefully with count/for_each, visualize with `terraform graph`, prefer implicit dependencies.

---
---
---

