# Locals vs Variables

## Introduction

Both locals and input variables store named values in Terraform, but they serve very different purposes. Confusing them leads to configurations that are either too rigid or unnecessarily complex. This file explains clearly when to use each one, why the distinction matters, and shows real examples of making the right choice.

---

## The Core Difference

The single most important distinction:

- **Input variables** — values that come FROM OUTSIDE your configuration (from the user, CI/CD, tfvars files)
- **Locals** — values that are computed INSIDE your configuration (derived from other values, never set by the user)

Think of it from the perspective of your Terraform configuration as a black box:

```
[User / CI/CD / tfvars]  →  [Input Variables]  →  [Your Terraform Configuration]  →  [Infrastructure]
                                                          ↑
                                               [Locals live here — internal]
```

Input variables are the front door — they receive values from outside. Locals are internal calculations — they do not receive values from outside.

---

## Side-by-Side Comparison

| Aspect | Input Variables | Locals |
|---|---|---|
| **Who sets the value** | The user running Terraform | The configuration itself |
| **Can be passed via command line** | Yes (`-var` flag) | No |
| **Can be in `.tfvars` files** | Yes | No |
| **Can have a `default`** | Yes | N/A (they have a computed value) |
| **Can be `sensitive`** | Yes | No |
| **Can have `validation`** | Yes | No |
| **Can reference resources** | No (only other variables) | Yes |
| **Can reference data sources** | No | Yes |
| **Can reference other locals** | No | Yes |
| **Purpose** | Accept external configuration | Avoid repetition, compute derived values |

---

## When to Use Input Variables

Use a variable when:

### 1. The value changes between environments or deployments

```hcl
# CORRECT — environment name changes per deployment
variable "environment" {
  type = string
}

# CORRECT — region might be different per team or deployment
variable "aws_region" {
  type    = string
  default = "us-east-1"
}

# CORRECT — instance type might differ between dev and production
variable "instance_type" {
  type    = string
  default = "t2.micro"
}
```

### 2. The value needs to be provided by the user or CI/CD

```hcl
# CORRECT — password should never be in code, must come from outside
variable "db_password" {
  type      = string
  sensitive = true
}

# CORRECT — team-specific values passed from outside
variable "team_name" {
  type = string
}

variable "cost_center" {
  type = string
}
```

### 3. The value makes your module reusable

```hcl
# CORRECT — makes the module work for any project name
variable "project" {
  type = string
}

# CORRECT — makes the module work in any VPC
variable "vpc_id" {
  type = string
}
```

### 4. You want to allow the caller to customize behavior

```hcl
# CORRECT — caller can turn this on or off
variable "enable_deletion_protection" {
  type    = bool
  default = false
}

# CORRECT — caller specifies which subnets to use
variable "subnet_ids" {
  type = list(string)
}
```

---

## When to Use Locals

Use a local when:

### 1. The value is always derived from other values

```hcl
# CORRECT — always computed from variables, never set directly
locals {
  prefix = "${var.project}-${var.environment}"
}

# WRONG — there is no reason for this to be a variable
# someone should not pass "myapp-production" directly
variable "prefix" {
  type = string
}
```

### 2. The value is repeated multiple times

```hcl
# CORRECT — define once, use everywhere
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# WRONG — copying the same map to every resource
resource "aws_instance" "web" {
  tags = {
    Project     = var.project      # repeated!
    Environment = var.environment  # repeated!
    ManagedBy   = "Terraform"      # repeated!
  }
}
```

### 3. The value is computed using functions or expressions

```hcl
# CORRECT — locals are great for function results
locals {
  azs                  = slice(data.aws_availability_zones.available.names, 0, 3)
  public_subnet_cidrs  = [for i in range(3) : cidrsubnet(var.vpc_cidr, 8, i)]
  upper_environment    = upper(var.environment)
  db_host              = split(":", aws_db_instance.app.endpoint)[0]
}

# WRONG — you would never ask the user to compute cidrsubnet values
variable "public_subnet_cidrs" {
  # This forces the user to manually calculate CIDRs when they could be computed
  type = list(string)
}
```

### 4. The value references resource attributes

```hcl
# CORRECT — locals can reference resources, variables cannot
locals {
  # Build a connection string using the database resource's attributes
  db_connection_string = "postgres://${var.db_user}@${aws_db_instance.app.endpoint}/${var.db_name}"

  # Build a map of name to ID after subnets are created
  subnet_map = { for s in aws_subnet.all : s.tags["Name"] => s.id }
}

# This would be WRONG as a variable — variables cannot reference resources
variable "db_connection_string" { ... }
```

### 5. The value should not be overridable by the user

```hcl
# CORRECT — this is an internal computation that should never be overridden
locals {
  is_production = var.environment == "production"
  instance_type = local.is_production ? "t3.large" : "t2.micro"
}

# WRONG if you want to enforce this logic — making it a variable
# lets the user bypass the logic and pass any instance type they want
variable "instance_type" {
  default = "t2.micro"
  # User could pass "p3.16xlarge" and bypass your sizing logic
}
```

---

## Common Mistakes and Corrections

### Mistake 1: Using a variable when a local is better

```hcl
# WRONG — the user should not be setting this directly
# It is always derived from project + environment
variable "resource_prefix" {
  type    = string
  default = "myapp-dev"
}

# CORRECT — compute it from the real inputs
locals {
  resource_prefix = "${var.project}-${var.environment}"
}
```

If you make `resource_prefix` a variable, a user could accidentally pass `"prod"` when deploying to dev, creating naming confusion. Making it a local ensures it is always computed correctly from the actual project and environment values.

---

### Mistake 2: Using a local when a variable is needed

```hcl
# WRONG — this hardcodes the region in code
# It cannot be changed without editing the file
locals {
  aws_region = "us-east-1"
}

# CORRECT — allow the region to be configured from outside
variable "aws_region" {
  type    = string
  default = "us-east-1"
}
```

If the region is a local, different team members or environments cannot use different regions without changing the code.

---

### Mistake 3: Putting complex expressions in variables

```hcl
# WRONG — variables cannot compute values; this is confusing
variable "bucket_name" {
  default = "${var.project}-${var.environment}-data"
  # This does not work — variable defaults cannot reference other variables!
}

# CORRECT — use a local for derived values
variable "project"     { type = string }
variable "environment" { type = string }

locals {
  bucket_name = "${var.project}-${var.environment}-data"
}
```

Variable `default` values can only be literal values — they cannot reference other variables or resources. If you need a computed default, use a local.

---

### Mistake 4: Repeating complex expressions instead of using locals

```hcl
# WRONG — the CIDR calculation is repeated in every resource
resource "aws_subnet" "public_1" {
  cidr_block = cidrsubnet(var.vpc_cidr, 8, 0)
}

resource "aws_subnet" "public_2" {
  cidr_block = cidrsubnet(var.vpc_cidr, 8, 1)
}

resource "aws_subnet" "private_1" {
  cidr_block = cidrsubnet(var.vpc_cidr, 8, 10)
}

# CORRECT — compute the CIDRs once as locals
locals {
  public_cidrs  = [for i in range(2) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_cidrs = [for i in range(2) : cidrsubnet(var.vpc_cidr, 8, i + 10)]
}

resource "aws_subnet" "public" {
  count      = 2
  cidr_block = local.public_cidrs[count.index]
}

resource "aws_subnet" "private" {
  count      = 2
  cidr_block = local.private_cidrs[count.index]
}
```

---

## A Decision Framework

When you need to store a value, ask these questions:

```
Does the value come from outside the configuration?
(User, tfvars, CI/CD, command line)
    Yes → Use a VARIABLE

Does the value need to be customizable by whoever runs Terraform?
    Yes → Use a VARIABLE

Is the value always calculated from other variables or resources?
    Yes → Use a LOCAL

Is the value repeated in multiple places?
    Yes → Use a LOCAL

Does the value reference resource attributes?
    Yes → Must be a LOCAL (variables cannot do this)

Is the value a secret or sensitive?
    Yes → Use a VARIABLE with sensitive = true
```

---

## Real-World Example: Getting the Balance Right

Here is a well-designed configuration showing the correct use of both:

```hcl
# variables.tf — values that come from outside
variable "project" {
  description = "Project name"
  type        = string
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "az_count" {
  description = "Number of availability zones to use"
  type        = number
  default     = 2
}

variable "db_password" {
  description = "Database master password"
  type        = string
  sensitive   = true
}

variable "extra_tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}

# locals.tf — values computed internally
locals {
  # Naming — always derived from variables
  prefix = "${var.project}-${var.environment}"

  # Tags — defined once, used everywhere
  common_tags = merge(
    {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "Terraform"
      Region      = var.aws_region
    },
    var.extra_tags
  )

  # Network — computed from VPC CIDR
  public_subnet_cidrs  = [for i in range(var.az_count) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i in range(var.az_count) : cidrsubnet(var.vpc_cidr, 8, i + 10)]

  # Environment-based decisions — computed, not user-settable
  is_production       = var.environment == "production"
  instance_type       = local.is_production ? "t3.medium" : "t2.micro"
  db_multi_az         = local.is_production
  deletion_protection = local.is_production
  backup_retention    = local.is_production ? 30 : 7
}
```

In this example:
- Variables handle everything the user needs to configure (project name, environment, region, password)
- Locals handle everything that should be computed (prefix, tags, CIDRs, environment-based settings)
- The result is a clean separation — external configuration in variables, internal logic in locals

---

## Summary

- **Input variables** receive values from outside (users, CI/CD, tfvars files) — use them for values that are different between deployments or users
- **Locals** are computed internally from other values — use them to avoid repetition and to compute derived values
- Variables can have defaults, types, validation, and sensitive flags — locals just have a computed expression
- Variables cannot reference resources or data sources — locals can reference anything
- Variable default values cannot reference other variables — locals can reference any other local
- A simple rule: if someone running `terraform apply` might want to change the value, it is a variable. If it is always derived from other values, it is a local.
- Correct use of both makes your Terraform configurations clean, maintainable, and easy to understand

In the next chapter, we cover expressions and functions — the tools that make locals and configurations truly powerful.
