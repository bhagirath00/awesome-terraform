# The Architectural Choice: count vs. for_each

Choosing between the `count` and `for_each` meta-arguments is one of the most important architectural decisions in Terraform development. While both achieve the goal of provisioning multiple resources, they do so with fundamentally different technical philosophies: numeric scaling versus identity-based mapping. In a professional DevOps environment, making the wrong choice can lead to significant state instability and production risks during refactoring. This chapter provide a definitive comparison of these two patterns, outlining when to use each and why `for_each` has become the industry standard for complex infrastructure.

## 1. Feature Comparison: At a Glance

| Feature | count | for_each |
|---------|-------|----------|
| **Data Type** | Number (integer) or `length()` | Map or Set of strings |
| **Addressing** | Numeric Index (`[0]`, `[1]`) | Map Key (`["web"]`, `["db"]`) |
| **Ordering** | Strict. Shifting items triggers recreation | Neutral. Ordering does not affect identity |
| **Use Case** | Scaling identical resources / Toggles | Collections of unique resources |
| **Complexity** | Low (Beginner friendly) | Moderate (Requires HCL experience) |

## 2. When to Use count

Despite the rise of `for_each`, `count` remains the correct tool for specific scenarios.

### 2.1 Identical "Pet" Scaling
If you need exactly three identical web servers and you don't care about their individual names or identities beyond their group, `count` is the simplest path.

### 2.2 The Zero-to-One Toggle (Conditional Resources)
For optional infrastructure (e.g., "Only create a Bastion host in Dev"), `count` is the cleaner solution. 
```hcl
count = var.enable_feature ? 1 : 0
```
Using `for_each` for a single optional resource often adds unnecessary complexity (managing an empty map).

## 3. When to Use for_each

### 3.1 Heterogeneous Collections
If you are creating instances with different names, sizes, or tags, `for_each` is mandatory. It allows you to pass unique configuration objects for each instance safely.

### 3.2 Environments where Items Change
If you expect to add or remove items from the collection over time (e.g., a list of authorized IAM users or a list of Security Group rules), **always use `for_each`**. 
- **The Stability Factor:** Because `for_each` uses keys as the unique identifier in the state, adding or removing a key has zero impact on the other resources in the group. This eliminates the "Cascading Recreate" problem inherent to `count`.

## 4. State Management and Refactoring

The choice between `count` and `for_each` determines how easy it will be to refactor your code in the future.

- **Refactoring count:** If you need to move a resource from `aws_instance.web[2]` to `aws_instance.web_new`, you often have to use `terraform state mv`, which is manual and risky.
- **Refactoring for_each:** Because the identity is baked into the key (`["prod"]`), you can safely reorganize the underlying map logic without changing the resource's actual address in the state file.

## 5. Technical Constraints Summary

- **Plan-time Knowledge:** Both `count` and `for_each` (keys only) must be known at plan-time.
- **Null Values:** You cannot use `for_each` on a collection that might resolve to `null`. You must use `coalesce()` or defensive `if` logic.
- **Set Conversion:** Remember that `for_each` does not accept a raw list. You must use `toset(list)` or provide a map.

## 6. Summary

The choice between `count` and `for_each` is a choice between simplicity and stability. For the majority of professional, enterprise-scale modules, `for_each` is the superior choice because of its identity-based state addressing. However, for simple numeric scaling of homogeneous nodes or binary "Exist/Don't Exist" toggles, `count` remains a valid and efficient tool. By understanding the technical implications of numeric indexing versus map-key addressing, you can build infrastructure that is not only functional but also resilient to the long-term demands of refactoring and growth.
