---

# Terraform Modules Notes.

## Quick Revision Version

* **Modules:** Group related resources together to make code modular, reusable, and easier to maintain.
* **Root vs Child:** Root module is where Terraform commands run; child modules are reusable blocks that can be called from the root to organize resources efficiently.
* **Inputs (Variables):** Define configurable values in `variables.tf`; pass them via root module, `.tfvars`, CLI, or environment variables to make modules flexible.
* **Outputs:** Expose resource values from a module; access them in root module using `module.<name>.<output>`, optionally define root outputs for CLI display.
* **Calling Modules:** Can be called from local paths, Git repositories, or Terraform Registry using the `source` argument.
* **Versioning:** For Registry modules, specify a version to avoid breaking changes; Git modules can use branch, tag, or commit references.
## ⚡ Quick Revision Version

* **Why**: Consistency, stability, rollback, audit.
* **How**: `source = "git::repo-url//module?ref=tag"`
* **Refs**: Tag (`v1.0.0`), Branch (`main`), Commit (`abc123`).
* **Best practice**: Use **tags**, semantic versioning, avoid direct `main`.
* **Real-world**: VPC module v1.0.0 → later updated v1.1.0.
* **Pitfalls**: Don’t forget tags; branches drift; typos break modules.

---

# Terraform Modules – Quick Revision Notes (Expanded)

1. **Modules with count**

   * Create multiple instances of a module/resource.
   * `count = N` → N copies.
   * `count.index` → access list values or numeric index.
   * Use in tags: `Name = "webserver-${count.index}"`.
   * Access outputs: `module.<name>[0].output`.

2. **Modules with for\_each**

   * Loop over maps/sets; create multiple instances with meaningful keys.
   * `each.key` → identifier, `each.value` → input.
   * Access outputs: `module.<name}["key"].output`.
   * Preferred over count for descriptive identifiers.

3. **Count index vs String Interpolation vs List Indexing**

   * `${count.index}` → inside strings (e.g., tags).
   * `[count.index]` → access element in list.
   * `each.key` / `each.value` → for\_each loops.
   * Function calls → `func()`.
   * Rule: string → `${}`, list/map → `[index/key]`, for\_each → `each`.

---

---

# Terraform Modules – Quick Revision Notes for `depends_on` & Module Composition

1. **depends\_on**

   * Explicit module/resource creation order.
   * Syntax: `depends_on = [module.<name>]`.
   * Use only when implicit dependencies insufficient.
   * Avoid circular dependencies.

2. **Module Composition**

   * Nest child modules inside a parent module.
   * Parent exposes outputs; root calls parent module.
   * Root module usually calls both independent child modules + composed parent modules.
   * Keep child modules small; parent for complex grouping.
   * Document module composition clearly.

