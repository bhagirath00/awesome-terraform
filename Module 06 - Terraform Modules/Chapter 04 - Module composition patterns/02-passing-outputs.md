# Building Complex Systems: Passing Outputs between Modules

In a modular architecture, no component exists in isolation. A network module creates subnets that a compute module needs. A database module creates an endpoint that an application module requires. The process of connecting these components is known as **Module Composition**. Effectively passing outputs between modules is the key to building a cohesive, self-aware infrastructure stack. This chapter explores the patterns, dependencies, and technical mechanics of module-to-module data flow.

## 1. The Core Pattern: Source to Sink

The most basic form of composition involves taking an output from one module and using it as a variable input for another in the same Root Module.

```hcl
# The Source Module
module "networking" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

# The Sink Module
module "compute" {
  source    = "./modules/ec2"
  # Passing the output from 'networking' into 'compute'
  subnet_id = module.networking.private_subnets[0]
}
```

### 1.1 Logical Flow
By connecting `module.networking` to `module.compute`, you are creating an **Implicit Dependency**. Terraform's engine automatically determines that the networking module must be finished before it can start provisioning the compute module because the `subnet_id` is an output of the former.

## 2. Managing Data Transformation

Sometimes, the output of one module is not in the exact format required by the next. Instead of cluttering your module call with complex logic, use `locals` in the Root Module to transform the data.

```hcl
locals {
  # Select only the first AZ's public subnet
  frontend_subnet = module.vpc.public_subnets[0]
  
  # Filter a map of security groups to find the "web" group
  web_sg_id = lookup(module.security.group_ids, "web", null)
}

module "web_tier" {
  source    = "./modules/app"
  subnet_id = local.frontend_subnet
  sg_id     = local.web_sg_id
}
```

Using `locals` for transformation makes the code significantly more readable and easier to debug during a `terraform plan`.

## 3. Dependency Chains and "Late Binding"

A "Late Binding" dependency occurs when you pass a value that is only known after a resource is created (e.g., a specific RDS endpoint).

### 3.1 Planning Limitations
If `Module B` uses an output from `Module A` inside a `count` or `for_each` argument, Terraform may fail during the plan phase if `Module A` hasn't been applied yet.
- **Why?** `count` and `for_each` must be known at plan time.
- **Solution:** Pass "Plan-time" known values (like static names or hardcoded counts) rather than "Apply-time" dynamic outputs whenever possible into orchestration logic.

## 4. Multi-Step Pipelines with Remote State

If your modules are managed in separate Terraform workspaces (e.g., a separate state file for Networking and a separate state file for Apps), you cannot link them via simple module calls. Instead, you use the `terraform_remote_state` data source.

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-tf-state"
    key    = "networking.tfstate"
  }
}

module "app" {
  source    = "./modules/app"
  # Pulling from a separate state file
  vpc_id    = data.terraform_remote_state.network.outputs.vpc_id
}
```

## 5. Avoiding the "Wall of Variables"

A common architectural trap is passing 50 outputs from one module into 50 inputs of another. This is hard to maintain and prone to error.

**Refinement Patterns:**
- **Pass the Object:** Instead of passing 10 subnet IDs, pass the entire subnet list or the entire VPC object output.
- **Implicit Lookups:** Instead of passing IDs, have the modules use `data` sources to find resources based on a shared "Project Name" or "Environment" tag.

## 6. The "Dependency Injection" Principle

In professional software engineering, dependency injection is a pattern where an object's dependencies are provided by the caller. Terraform follows this by forcing you to pass IDs rather than having modules "look up" their own dependencies.

**Advantage:** This makes your modules truly reusable. A `compute` module that requires a `vpc_id` can be used in your VPC, a shared corporate VPC, or a peer's VPC just by changing the input variable.

## 7. Handling Sensitive Data between Modules

When passing credentials or tokens between modules, ensure every link in the chain respects the `sensitive` attribute.
- If `Module A` outputs a sensitive value.
- The Root Module must define a variable as `sensitive = true` if it's going to relay that value.
- `Module B` must define its input variable as `sensitive = true`.

Failure to maintain the "Sensitivity Chain" can lead to secrets appearing in the clear text output of the root module's plan.

## 8. Summary

Passing outputs is the "Glue" of the Cloud Operating Model. It allows small, specialized modules to form a complex, integrated system. By mastering the use of `locals` for transformation and respecting the boundaries between plan-time and apply-time data, you can build infrastructure that is decoupled but perfectly synchronized. This patterns-based approach ensures that as your architecture grows from 10 resources to 10,000, your code remains modular, testable, and robust.
