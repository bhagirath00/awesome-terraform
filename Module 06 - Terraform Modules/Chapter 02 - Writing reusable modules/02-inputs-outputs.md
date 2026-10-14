# Designing the Interface: Inbound and Outbound Data

The success of a Terraform module depends on its interface. A well-designed module is flexible enough to handle different environments but strict enough to prevent configuration errors. This is achieved through the strategic use of Input Variables and Output Values. This chapter discusses how to design robust, self-validating interfaces for professional-grade modules.

## 1. Input Variables: The Module's Controls

Input variables are the parameters a user provides when calling a module. They allow a single module to adapt to different regions, instance sizes, and environments.

### 1.1 Required vs. Optional Variables
- **Required:** If a variable has no `default` value, Terraform will stop and prompt the user if it's missing. Use these for parameters that must be unique, like a `project_name` or `vpc_id`.
- **Optional:** If a variable has a `default` value, it becomes optional. Use these to provide a "Standard Configuration" while allowing advanced users to override it (e.g., `instance_type = "t3.micro"`).

### 1.2 Type Constraints and Complexity
Always use specific type constraints instead of the generic `any`.
- **Primitive Types:** `string`, `number`, `bool`.
- **Structural Types:** `list()`, `map()`, `object()`.
- **Best Practice:** Use `object()` for complex configurations like a list of subnets with specific CIDRs and tags. This ensures the user provides all necessary keys in the correct format.

## 2. Enforcing Standards: Variable Validation

One of the most powerful features of modern HCL is the `validation` block. This allows a module author to enforce organizational policies directly in the code.

```hcl
variable "instance_type" {
  type        = string
  description = "The EC2 instance type to use for the application."

  validation {
    condition     = can(regex("^t3\\.", var.instance_type))
    error_message = "The instance_type must be part of the t3 family (e.g., t3.micro, t3.small)."
  }
}
```

By adding validation, you build a "Self-Correcting" infrastructure. If a developer tries to use an expensive instance type not allowed by policy, Terraform will fail with a descriptive error before it even attempts to talk to the cloud provider.

## 3. Output Values: The Module's Results

Outputs are the only way to get data back from a module. Without outputs, a module is a dead-end; its resources cannot be referenced by other modules.

### 3.1 What to Output
A professional module should output enough information to enable further automation.
- **Primary Identifiers:** Always output the `id` and `arn` of the main resources.
- **Connection Data:** DNS names, public IPs, or connection strings.
- **Dependency Data:** If your module creates a security group, output its ID so other modules can add rules to it.

### 3.2 Hiding Secrets with `sensitive`
If an output contains potentially sensitive information (like a database password, private key, or API token), set the `sensitive` flag to `true`.
```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```
This prevents the value from being printed to the console during `terraform apply`. Note: The value is still stored in the state file in plain text, but the `sensitive` flag protects it from accidental exposure in logs.

## 4. The "Object vs. Attribute" Debate

When designing outputs, should you output specific strings or the entire resource object?

### 4.1 Outputting Attributes
```hcl
output "vpc_id" {
  value = aws_vpc.main.id
}
```
**Pros:** Clean, simple, and self-documenting.
**Cons:** If the user later needs the VPC CIDR block, you must go back and update the module code to add a new output.

### 4.2 Outputting Objects
```hcl
output "vpc" {
  value = aws_vpc.main
}
```
**Pros:** Future-proof. The user can access any attribute of the VPC (e.g., `module.vpc_layer.vpc.arn`).
**Cons:** It exposes the internal implementation details of the module and makes the dependency graph more complex.

**Recommendation:** Output the specific attributes that are part of the module's core mission and provide resource objects as a secondary "developer" output if needed.

## 5. Parameter Propagation (Bubble Patterns)

In complex architectures, a variable might need to pass through three layers of modules.
- **Root** (gets var from CLI) -> **App Cluster** (receives var) -> **Web Nodes** (uses var).

To maintain sanity:
1. Use identical names for variables across all layers (e.g., `environment` is `environment` everywhere).
2. Use identical defaults to prevent confusion.
3. Don't propagate a variable if a module doesn't actually need it; keep modules as independent as possible.

## 6. Defaulting Strategy: The "Zero Friction" Principle

Design your modules so they work with the minimal possible input.
- If a user just wants a "VPC," and they don't care about the CIDR block, provide a default like `10.0.0.0/16`.
- Only make a variable required if there is no safe or logical default. This reduces the cognitive load on the engineer using your module.

## 7. Referencing Inputs and Outputs in Code

- **Inside the Module:** Use `var.<NAME>`.
- **Outside the Module (Calling the output):** Use `module.<INSTANCE_NAME>.<OUTPUT_NAME>`.

```hcl
resource "aws_alb" "main" {
  # Referencing an output from a network module
  subnets = module.network.public_subnets
}
```

## 8. Summary

Inputs and outputs define the relationship between your infrastructure layers. By using strict type constraints, descriptive names, and custom validation, you create "Intelligent Modules" that guide the user toward correct behavior. High-quality outputs ensure that your modules are composable, allowing different parts of your infrastructure to communicate and collaborate effectively as your architecture grows in complexity.
