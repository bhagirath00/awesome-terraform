# Input Variables

## Introduction

**Input variables** are the way you make your Terraform configurations flexible and reusable. Instead of hardcoding values like region names, instance types, or environment names directly in your resources, you define variables and pass values in at runtime.

Think of input variables like function parameters in programming. A function with no parameters can only do one thing. A function with parameters can do different things based on what you pass in. Variables make your Terraform configurations work the same way.

Without variables:
```hcl
# Hardcoded — works only for one specific setup
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Environment = "dev"
  }
}
```

With variables:
```hcl
# Flexible — works for any environment, instance type, or AMI
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  tags = {
    Environment = var.environment
  }
}
```

---

## Declaring a Variable

Variables are declared using the `variable` block. The block has one required label — the variable name — and several optional arguments inside.

### Basic Variable Declaration

```hcl
variable "region" {
  description = "The AWS region to deploy resources in"
  type        = string
  default     = "us-east-1"
}
```

### Variable Block Arguments

**`description`** — A human-readable explanation of what this variable is for. Shown when someone runs `terraform plan` or when the module is documented. Always write one.

**`type`** — The data type this variable accepts (string, number, bool, list, map, object, etc.). If omitted, the variable accepts any type.

**`default`** — The value to use if no value is provided when running Terraform. If there is no default, the variable is required — Terraform will prompt for a value or error if it cannot find one.

**`sensitive`** — When set to `true`, Terraform hides the variable's value in plan output and logs. Use for passwords, API keys, and other secrets.

**`validation`** — A block that defines rules the value must satisfy. Covered in detail in `variable-validation.md`.

---

## A Complete Variable with All Arguments

```hcl
variable "instance_type" {
  description = "EC2 instance type for the web servers"
  type        = string
  default     = "t2.micro"

  validation {
    condition     = contains(["t2.micro", "t2.small", "t3.micro", "t3.small"], var.instance_type)
    error_message = "Instance type must be one of: t2.micro, t2.small, t3.micro, t3.small."
  }
}
```

---

## Referencing Variables

Once declared, you reference a variable using the `var.` prefix:

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type      # reference

  tags = {
    Name        = "web-server-${var.environment}"  # reference in string
    Environment = var.environment                   # reference
  }
}
```

---

## Required vs Optional Variables

### Optional Variables (with default)

A variable with a `default` is optional — you do not have to provide a value when running Terraform:

```hcl
variable "region" {
  type    = string
  default = "us-east-1"  # optional — uses this value if nothing is provided
}
```

### Required Variables (no default)

A variable without a `default` is required — Terraform needs a value from you before it can proceed:

```hcl
variable "project_name" {
  description = "The name of the project (used in resource names and tags)"
  type        = string
  # No default — must be provided every time
}
```

If you run `terraform plan` without providing a value for a required variable, Terraform will either:
1. Prompt you to enter a value interactively
2. Error if it cannot prompt (in CI/CD environments)

This is why it is important to always provide required variables through one of the methods covered later.

---

## Sensitive Variables

Mark variables as sensitive when they contain secrets:

```hcl
variable "db_password" {
  description = "Master password for the RDS database"
  type        = string
  sensitive   = true
}

variable "api_key" {
  description = "Third-party API key"
  type        = string
  sensitive   = true
}
```

When a variable is marked sensitive:
- Its value is hidden in `terraform plan` output (shown as `(sensitive value)`)
- Its value is hidden in `terraform apply` output
- Terraform logs will not show the value

```
+ resource "aws_db_instance" "app" {
    + password = (sensitive value)
  }
```

**Important:** Sensitive variables are still stored in the state file in plain text. Always encrypt your state file and restrict access to it.

---

## Ways to Provide Variable Values

When you run Terraform and it needs a variable value, it looks for values in this order (later methods override earlier ones):

### 1. Default Value in the Variable Declaration

```hcl
variable "region" {
  default = "us-east-1"  # used if nothing else provides a value
}
```

### 2. `terraform.tfvars` File

Terraform automatically loads a file named `terraform.tfvars` if it exists in the current directory:

```hcl
# terraform.tfvars
region        = "us-west-2"
environment   = "staging"
instance_type = "t3.small"
```

### 3. `*.auto.tfvars` Files

Any file ending in `.auto.tfvars` is automatically loaded:

```hcl
# production.auto.tfvars (loaded automatically)
environment   = "production"
instance_type = "t3.large"
```

### 4. `-var-file` Flag

Specify a variable file explicitly when running Terraform:

```bash
terraform plan -var-file="production.tfvars"
terraform apply -var-file="staging.tfvars"
```

The file can have any name when using `-var-file`.

### 5. `-var` Flag

Pass individual variable values on the command line:

```bash
terraform plan -var="environment=production"
terraform apply -var="instance_type=t3.medium" -var="region=eu-west-1"
```

### 6. Environment Variables with `TF_VAR_` Prefix

Set environment variables with the `TF_VAR_` prefix — Terraform reads them automatically:

```bash
export TF_VAR_environment="production"
export TF_VAR_instance_type="t3.large"
export TF_VAR_db_password="SuperSecret123!"

terraform apply  # picks up TF_VAR_ variables automatically
```

This is very useful in CI/CD pipelines where you store secrets as environment variables (GitHub Secrets, Jenkins credentials, etc.).

---

## Variable Precedence

When multiple sources provide a value for the same variable, Terraform uses this priority (highest wins):

```
1. -var flag (highest priority)
2. -var-file flag
3. *.auto.tfvars files
4. terraform.tfvars file
5. TF_VAR_ environment variables
6. Default value in variable declaration (lowest priority)
```

**Example:**

Variable declaration:
```hcl
variable "environment" {
  default = "dev"  # lowest priority
}
```

`terraform.tfvars`:
```hcl
environment = "staging"  # overrides the default
```

Command line:
```bash
terraform apply -var="environment=production"  # overrides everything
```

Result: `environment = "production"`

---

## Variable Best Practices

### Keep Variables in a Dedicated File

Store all variable declarations in `variables.tf`:

```
project/
├── main.tf           # resources
├── variables.tf      # variable declarations
├── outputs.tf        # output declarations
├── providers.tf      # provider config
└── terraform.tfvars  # variable values
```

### Always Write Descriptions

Descriptions serve as documentation. Anyone reading or using your code should understand what each variable does without needing to look elsewhere:

```hcl
# Good — clear description
variable "max_instance_count" {
  description = "Maximum number of EC2 instances in the auto scaling group. Set higher for peak traffic periods."
  type        = number
  default     = 5
}

# Bad — no description
variable "max_instance_count" {
  type    = number
  default = 5
}
```

### Use Specific Types

Always specify the type. It catches mistakes early:

```hcl
# Good
variable "instance_count" {
  type    = number
  default = 3
}

# Bad — accepts anything including typos like "three"
variable "instance_count" {
  default = 3
}
```

### Never Put Secrets in Default Values

```hcl
# NEVER do this — secrets in code get committed to Git
variable "db_password" {
  default = "SuperSecret123!"  # WRONG
}

# Correct — require the value to be provided externally
variable "db_password" {
  description = "RDS master password — provide via TF_VAR_db_password or secrets manager"
  type        = string
  sensitive   = true
  # No default — must be provided
}
```

### Group Related Variables with Comments

```hcl
# ============================================================
# Networking Variables
# ============================================================

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for the public subnets (one per AZ)"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

# ============================================================
# Compute Variables
# ============================================================

variable "instance_type" {
  description = "EC2 instance type for application servers"
  type        = string
  default     = "t3.micro"
}

variable "instance_count" {
  description = "Number of application server instances"
  type        = number
  default     = 2
}
```

---

## A Complete Real-World Variables Example

Here is a complete `variables.tf` file for a realistic web application project:

```hcl
# variables.tf

# ============================================================
# General
# ============================================================

variable "project" {
  description = "Project name — used in resource names and tags"
  type        = string
}

variable "environment" {
  description = "Deployment environment (dev, staging, production)"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be one of: dev, staging, production."
  }
}

variable "aws_region" {
  description = "AWS region for all resources"
  type        = string
  default     = "us-east-1"
}

# ============================================================
# Networking
# ============================================================

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "List of CIDR blocks for public subnets (one per AZ)"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "List of CIDR blocks for private subnets (one per AZ)"
  type        = list(string)
  default     = ["10.0.10.0/24", "10.0.11.0/24"]
}

# ============================================================
# Compute
# ============================================================

variable "instance_type" {
  description = "EC2 instance type for application servers"
  type        = string
  default     = "t3.micro"
}

variable "min_instances" {
  description = "Minimum number of instances in the auto scaling group"
  type        = number
  default     = 1
}

variable "max_instances" {
  description = "Maximum number of instances in the auto scaling group"
  type        = number
  default     = 5
}

# ============================================================
# Database
# ============================================================

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "db_name" {
  description = "Name of the application database"
  type        = string
  default     = "appdb"
}

variable "db_username" {
  description = "Master username for the RDS database"
  type        = string
  default     = "dbadmin"
}

variable "db_password" {
  description = "Master password for the RDS database — provide via TF_VAR_db_password"
  type        = string
  sensitive   = true
}
```

And the corresponding `terraform.tfvars` for the dev environment:

```hcl
# terraform.tfvars (dev environment defaults)

project     = "my-web-app"
environment = "dev"
aws_region  = "us-east-1"

instance_type = "t2.micro"
min_instances = 1
max_instances = 2

db_instance_class = "db.t3.micro"
db_name           = "appdb"
db_username       = "dbadmin"
# db_password — set via TF_VAR_db_password environment variable
```

---

## Summary

- Input variables make Terraform configurations flexible and reusable by avoiding hardcoded values
- Declare variables using the `variable` block with `description`, `type`, `default`, `sensitive`, and `validation` arguments
- Reference variables using `var.variable_name` syntax
- Variables without a `default` are required — Terraform will error if no value is provided
- Mark sensitive variables with `sensitive = true` to hide them from plan and apply output
- Values can be provided through: defaults, `terraform.tfvars`, `*.auto.tfvars`, `-var-file`, `-var` flags, and `TF_VAR_` environment variables
- Command-line `-var` flags have the highest priority; default values have the lowest
- Always write descriptions, specify types, and never put secrets in default values

In the next file, we go deeper into variable types — covering complex types like `list(object(...))` with detailed examples.
