# Variable Validation

## Introduction

Variable validation lets you define custom rules that a variable's value must satisfy. When someone provides a value that breaks the rule, Terraform immediately shows a clear, helpful error message — before it even tries to plan or create any infrastructure.

Without validation, bad variable values might:
- Cause confusing AWS API errors deep in the apply process
- Create resources with wrong configurations that only fail at runtime
- Lead to security misconfigurations (like allowing 0.0.0.0/0 where it should not be)

With validation, problems are caught immediately with a message that tells the user exactly what is wrong and what is expected.

---

## The `validation` Block

Validation is added inside a `variable` block using one or more `validation` blocks:

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be one of: dev, staging, production."
  }
}
```

The `validation` block has two required arguments:

- **`condition`** — An expression that must evaluate to `true` for the value to be valid. If it evaluates to `false`, the error message is shown.
- **`error_message`** — The message shown to the user when the condition fails. Must be a complete sentence ending with a period.

---

## How Validation Works

When you run `terraform plan` or `terraform apply`:

1. Terraform reads all variable values
2. For each variable with a `validation` block, it evaluates the `condition`
3. If the condition is `false`, it shows the `error_message` and stops
4. If all conditions are `true`, it continues

```bash
$ terraform plan -var="environment=testing"

╷
│ Error: Invalid value for variable
│
│   on variables.tf line 3:
│    3: variable "environment" {
│
│ Environment must be one of: dev, staging, production.
│
│ This was checked by the validation rule at variables.tf line 8,14.
╵
```

The error shows:
- Which variable failed (`environment`)
- The error message you wrote
- Which line the validation rule is on

---

## Common Validation Patterns

### Pattern 1: Restrict to Specific Values

Use `contains()` to limit a variable to a set of allowed values:

```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be one of: dev, staging, production."
  }
}

variable "log_level" {
  type = string
  validation {
    condition     = contains(["DEBUG", "INFO", "WARN", "ERROR"], var.log_level)
    error_message = "Log level must be one of: DEBUG, INFO, WARN, ERROR."
  }
}

variable "db_engine" {
  type = string
  validation {
    condition     = contains(["mysql", "postgres", "mariadb"], var.db_engine)
    error_message = "Database engine must be mysql, postgres, or mariadb."
  }
}
```

---

### Pattern 2: Validate Number Ranges

Use comparison operators to ensure numbers are within acceptable ranges:

```hcl
variable "instance_count" {
  type = number
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 20
    error_message = "Instance count must be between 1 and 20."
  }
}

variable "port" {
  type = number
  validation {
    condition     = var.port >= 1 && var.port <= 65535
    error_message = "Port must be a valid port number between 1 and 65535."
  }
}

variable "root_volume_size" {
  description = "Root EBS volume size in GB"
  type        = number
  validation {
    condition     = var.root_volume_size >= 8 && var.root_volume_size <= 16384
    error_message = "Root volume size must be between 8 GB and 16384 GB."
  }
}

variable "max_session_duration" {
  description = "IAM role maximum session duration in seconds"
  type        = number
  validation {
    condition     = var.max_session_duration >= 3600 && var.max_session_duration <= 43200
    error_message = "Maximum session duration must be between 3600 (1 hour) and 43200 (12 hours) seconds."
  }
}
```

---

### Pattern 3: Validate String Format

Use `regex()` or `can(regex(...))` to validate string patterns:

```hcl
# Validate that a project name uses only lowercase letters, numbers, and hyphens
variable "project_name" {
  type = string
  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.project_name))
    error_message = "Project name must contain only lowercase letters, numbers, and hyphens."
  }
}

# Validate that a CIDR block has the correct format
variable "vpc_cidr" {
  type = string
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid IPv4 CIDR block (e.g., 10.0.0.0/16)."
  }
}

# Validate that a string starts with a specific prefix
variable "bucket_name" {
  type = string
  validation {
    condition     = startswith(var.bucket_name, "mycompany-")
    error_message = "Bucket name must start with 'mycompany-' to follow naming conventions."
  }
}

# Validate string length
variable "resource_name" {
  type = string
  validation {
    condition     = length(var.resource_name) >= 3 && length(var.resource_name) <= 50
    error_message = "Resource name must be between 3 and 50 characters."
  }
}
```

---

### Pattern 4: Validate Lists

Validate the length or content of lists:

```hcl
# Must provide at least 2 availability zones for high availability
variable "availability_zones" {
  type = list(string)
  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones are required for high availability."
  }
}

# List must not be empty
variable "subnet_ids" {
  type = list(string)
  validation {
    condition     = length(var.subnet_ids) > 0
    error_message = "At least one subnet ID must be provided."
  }
}

# Validate that all items in a list match a pattern
variable "allowed_cidr_blocks" {
  type = list(string)
  validation {
    condition     = alltrue([for cidr in var.allowed_cidr_blocks : can(cidrhost(cidr, 0))])
    error_message = "All values in allowed_cidr_blocks must be valid CIDR blocks."
  }
}
```

The `alltrue()` function combined with a `for` expression is a powerful pattern for validating every item in a list.

---

### Pattern 5: Security-Focused Validation

Prevent common security misconfigurations:

```hcl
# Prevent accidentally opening SSH to the entire internet in production
variable "ssh_allowed_cidr" {
  type = string
  validation {
    condition     = var.ssh_allowed_cidr != "0.0.0.0/0"
    error_message = "SSH access must not be open to the entire internet (0.0.0.0/0). Specify a specific IP range."
  }
}

# Enforce minimum password length
variable "admin_password" {
  type      = string
  sensitive = true
  validation {
    condition     = length(var.admin_password) >= 16
    error_message = "Admin password must be at least 16 characters long."
  }
}

# Prevent creating unencrypted RDS instances
variable "db_storage_encrypted" {
  type = bool
  validation {
    condition     = var.db_storage_encrypted == true
    error_message = "Database storage encryption must be enabled. Set db_storage_encrypted to true."
  }
}

# Require specific tags to be set
variable "required_tags" {
  type = map(string)
  validation {
    condition     = contains(keys(var.required_tags), "Owner") && contains(keys(var.required_tags), "CostCenter")
    error_message = "Tags must include 'Owner' and 'CostCenter' keys for cost allocation."
  }
}
```

---

### Pattern 6: Validate AWS-Specific Values

```hcl
# Validate AWS region format
variable "aws_region" {
  type = string
  validation {
    condition     = can(regex("^[a-z]{2}-[a-z]+-[0-9]$", var.aws_region))
    error_message = "AWS region must be in the format: us-east-1, eu-west-2, ap-southeast-1, etc."
  }
}

# Validate EC2 instance type format
variable "instance_type" {
  type = string
  validation {
    condition     = can(regex("^[a-z][0-9][a-z]?\\.[a-z0-9]+$", var.instance_type))
    error_message = "Instance type must be a valid EC2 instance type (e.g., t2.micro, t3.medium, m5.large)."
  }
}

# Validate that an ARN looks like an ARN
variable "iam_role_arn" {
  type = string
  validation {
    condition     = startswith(var.iam_role_arn, "arn:aws:iam::")
    error_message = "IAM role ARN must start with 'arn:aws:iam::'."
  }
}
```

---

## Multiple Validation Blocks

You can add multiple `validation` blocks to a single variable. Each one tests a different condition with its own error message:

```hcl
variable "project_name" {
  type = string

  validation {
    condition     = length(var.project_name) >= 3
    error_message = "Project name must be at least 3 characters long."
  }

  validation {
    condition     = length(var.project_name) <= 20
    error_message = "Project name must not exceed 20 characters."
  }

  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.project_name))
    error_message = "Project name must contain only lowercase letters, numbers, and hyphens."
  }

  validation {
    condition     = !startswith(var.project_name, "-") && !endswith(var.project_name, "-")
    error_message = "Project name must not start or end with a hyphen."
  }
}
```

Each validation is checked independently. If multiple conditions fail, only the first failure is shown. Fix that, run again, and the next failure appears.

---

## Useful Functions for Validation

These HCL functions are commonly used in validation conditions:

| Function | What it does | Example |
|---|---|---|
| `contains(list, value)` | Returns true if value is in list | `contains(["a","b"], var.x)` |
| `length(value)` | Returns length of string, list, or map | `length(var.name) >= 3` |
| `can(expression)` | Returns true if expression succeeds without error | `can(regex("...", var.x))` |
| `regex(pattern, string)` | Tests string against regex pattern | `can(regex("^[a-z]+$", var.x))` |
| `startswith(string, prefix)` | Returns true if string starts with prefix | `startswith(var.name, "app-")` |
| `endswith(string, suffix)` | Returns true if string ends with suffix | `endswith(var.name, "-prod")` |
| `alltrue(list_of_bools)` | Returns true only if all values are true | `alltrue([for x in var.list : x > 0])` |
| `anytrue(list_of_bools)` | Returns true if any value is true | `anytrue([for x in var.list : x > 0])` |
| `cidrhost(cidr, host)` | Parses a CIDR (useful for CIDR validation) | `can(cidrhost(var.cidr, 0))` |
| `keys(map)` | Returns list of keys from a map | `contains(keys(var.tags), "Env")` |

---

## Writing Good Error Messages

The error message is what a user sees when they make a mistake. Write messages that:

1. **State what is wrong** — not just that it is wrong
2. **State what is expected** — give the user enough information to fix it
3. **End with a period** — Terraform requires this

```hcl
# Poor error message
validation {
  condition     = contains(["dev", "staging", "production"], var.environment)
  error_message = "Invalid environment."
}

# Good error message
validation {
  condition     = contains(["dev", "staging", "production"], var.environment)
  error_message = "Environment must be one of: dev, staging, production. Received: ${var.environment}."
}

# Even better — includes what was received and what was expected
validation {
  condition     = var.instance_count >= 1 && var.instance_count <= 10
  error_message = "Instance count must be between 1 and 10. Got ${var.instance_count}."
}
```

Including `${var.variable_name}` in the error message shows the user what they actually passed, which helps them identify and fix the issue quickly.

---

## Summary

- Variable validation catches bad inputs immediately with a clear, helpful error message
- Add a `validation` block inside `variable` with a `condition` and `error_message`
- The `condition` must evaluate to `true` for the value to be accepted
- You can have multiple `validation` blocks per variable — each tests a different rule
- Common patterns: restrict to allowed values with `contains()`, validate ranges with `>=` and `<=`, validate formats with `can(regex(...))`, validate lists with `length()` and `alltrue()`
- Security validations can prevent dangerous misconfigurations before they reach AWS
- Write error messages that tell the user what was wrong, what is expected, and ideally what value was received

In the next file, we cover `.tfvars` files — how to use them to manage variable values across different environments.
