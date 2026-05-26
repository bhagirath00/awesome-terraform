# Scaling with Meta-arguments: Module `for_each` and `count`

In early versions of Terraform, modules could only be instantiated as singular units. To create multiple instances of a module, you had to duplicate the `module` block multiple times. Modern Terraform (0.13+) introduced support for the `for_each` and `count` meta-arguments on module blocks, transforming them into powerful tools for bulk infrastructure deployment. This chapter explores how to scale your modular architecture using these iteration patterns.

## 1. The Power of `for_each` on Modules

The `for_each` meta-argument allows you to create multiple instances of a module based on a map or a set of strings. This is the preferred method for scaling because it uses a key-based addressing system that is stable during refactoring.

```hcl
module "s3_buckets" {
  source = "./modules/s3-secure-bucket"
  for_each = {
    logs    = "private"
    assets  = "public-read"
    billing = "private"
  }

  bucket_name = "corp-${each.key}-storage"
  acl         = each.value
}
```

### 1.1 The `each` object
Inside the module block, the `each` object refers to the current iteration:
- `each.key`: The map key (e.g., "logs").
- `each.value`: The map value (e.g., "private").

### 1.2 State Addressing
Terraform addresses each instance using the key:
`module.s3_buckets["logs"].aws_s3_bucket.this`
`module.s3_buckets["assets"].aws_s3_bucket.this`

## 2. Using `count` for Conditional Scaling

While `for_each` is best for collections, `count` is often used for simple toggle logic (creating 0 or 1 instance) or creating a numbered fleet of identical resources.

```hcl
module "bastion_host" {
  source = "./modules/ec2-bastion"
  count  = var.enable_bastion ? 1 : 0

  subnet_id = var.public_subnet_id
}
```

### 2.1 The Shifting Problem
Avoid using `count` for lists of items that might change order. If you have `count = length(var.names)`, and you remove the first name in the list, Terraform will destroy and recreate every subsequent module instance because their indices have shifted. Always prefer `for_each` for dynamic collections.

## 3. Passing Complex Data to Looped Modules

When using `for_each`, you often need to pass more than just a string to the module. You can use maps of objects to provide a full configuration for each instance.

```hcl
variable "environments" {
  type = map(object({
    cidr        = string
    instance_count = number
  }))
}

module "environment_stack" {
  source   = "./modules/stack"
  for_each = var.environments

  network_cidr   = each.value.cidr
  capacity       = each.value.instance_count
}
```

## 4. Provider Inheritance in Looped Modules

By default, all instances created by a loop use the default provider configuration. If you need to deploy modules across different regions using a loop, you must explicitly pass shadowed provider aliases.

**Technical Constraint:** You cannot dynamically change the provider configuration inside a `for_each` loop using the `provider` meta-argument in a way that maps different regions to different keys automatically without a complex provider aliasing strategy in the root module.

## 5. Aggregating Outputs from Looped Modules

A common challenge is collecting the data produced by multiple module instances. Since the module is now a map or a list, you must use `for` expressions to extract the results.

```hcl
output "all_bucket_arns" {
  description = "A map of bucket names to their ARNs"
  value       = { for k, v in module.s3_buckets : k => v.bucket_arn }
}
```

This pattern allows you to pass a consolidated list of IDs or ARNs to another part of your infrastructure (e.g., passing 10 subnet IDs to a Load Balancer).

## 6. Planning and Performance

### 6.1 Dependency Resolution
In a looped module, every instance is a distinct node in the dependency graph. However, because they are defined in the same block, they all share the same upstream dependencies. If the variable provided to `for_each` is not known until apply time, the entire module block will be deferred, which can significantly slow down the deployment of large stacks.

### 6.2 Parallelism
Terraform attempts to create all instances of a looped module in parallel (up to the default `-parallelism` limit). This makes `for_each` on modules much faster than running multiple sequential `apply` commands.

## 7. Refactoring: From Single to Loop

If you decide to move a single `module "web"` to a `for_each` loop, be aware that Terraform will see this as a "Delete and Create" operation because the address has changed from `module.web` to `module.web["key"]`.
- **Strategy:** Use the `moved` block (introduced in TF 1.1) to inform Terraform of the refactor without destroying the underlying resources.
```hcl
moved {
  from = module.web
  to   = module.web["primary"]
}
```

## 8. Summary

The introduction of `for_each` and `count` for modules marked a major milestone in Terraform's evolution. It allows for the creation of "Programmable Infrastructure" where a small amount of HCL can generate vast and complex environments. By mastering the use of maps of objects and understanding the implications of state addressing and provider inheritance, you can build modular architectures that scale effortlessly from a single server to global deployments across hundreds of environments.
