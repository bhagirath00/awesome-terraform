# Variable Types in Depth

## Introduction

We covered HCL data types in Chapter 9. This file focuses specifically on how types work in the context of **variable declarations** — including complex type constraints, nested types, optional object attributes, and real-world patterns you will use frequently.

Understanding variable types thoroughly helps you write modules that are safe, self-documenting, and easy to use correctly.

---

## Why Type Constraints Matter

When you specify a type in a variable declaration, Terraform enforces it. If someone passes the wrong type, they get a clear error immediately — not a mysterious failure deep in an apply run.

**Without a type constraint:**
```hcl
variable "instance_count" {
  default = 3
}
```

Someone could accidentally pass `"three"` (a string) instead of `3` (a number). Terraform might accept it and then fail in a confusing way when trying to use it.

**With a type constraint:**
```hcl
variable "instance_count" {
  type    = number
  default = 3
}
```

If someone passes `"three"`, Terraform immediately says:
```
Error: Invalid value for input variable
  The given value is not valid for var.instance_count: a number is required.
```

Clear, early, actionable error. That is why types matter.

---

## Primitive Type Variables

### `string`

```hcl
variable "environment" {
  description = "Deployment environment name"
  type        = string
  default     = "development"
}

variable "aws_region" {
  description = "AWS region to deploy resources in"
  type        = string
  default     = "us-east-1"
}

variable "ami_id" {
  description = "AMI ID for EC2 instances"
  type        = string
  # No default — must be provided (AMI IDs are region-specific)
}
```

### `number`

```hcl
variable "instance_count" {
  description = "Number of application server instances"
  type        = number
  default     = 2
}

variable "root_volume_size" {
  description = "Size of the root EBS volume in GB"
  type        = number
  default     = 20
}

variable "max_session_duration" {
  description = "Maximum IAM role session duration in seconds"
  type        = number
  default     = 3600  # 1 hour
}
```

### `bool`

```hcl
variable "enable_deletion_protection" {
  description = "Prevent accidental deletion of the RDS database"
  type        = bool
  default     = false  # set to true for production
}

variable "multi_az" {
  description = "Enable Multi-AZ for RDS high availability"
  type        = bool
  default     = false
}

variable "enable_https" {
  description = "Enable HTTPS listener on the load balancer"
  type        = bool
  default     = true
}
```

---

## Collection Type Variables

### `list(string)`

A list where every item must be a string. The most common list type you will use.

```hcl
variable "availability_zones" {
  description = "List of availability zones to deploy subnets into"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "allowed_cidr_blocks" {
  description = "CIDR blocks allowed to access the application"
  type        = list(string)
  default     = ["10.0.0.0/8"]
}

variable "subnet_ids" {
  description = "List of subnet IDs for the load balancer"
  type        = list(string)
  # No default — subnet IDs depend on the VPC, must be provided
}
```

### `list(number)`

```hcl
variable "allowed_ports" {
  description = "List of ports to allow inbound traffic on"
  type        = list(number)
  default     = [80, 443]
}

variable "ebs_volume_sizes" {
  description = "Size in GB for each additional EBS volume to attach"
  type        = list(number)
  default     = [50, 100]
}
```

### `set(string)`

A set where every item must be a string, with no duplicates and no guaranteed order.

```hcl
variable "iam_policy_arns" {
  description = "Set of IAM policy ARNs to attach to the role"
  type        = set(string)
  default = [
    "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess",
    "arn:aws:iam::aws:policy/CloudWatchReadOnlyAccess"
  ]
}
```

### `map(string)`

A map where all values must be strings.

```hcl
variable "common_tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default = {
    ManagedBy = "Terraform"
    Team      = "platform"
  }
}

variable "instance_type_per_environment" {
  description = "EC2 instance type to use per environment"
  type        = map(string)
  default = {
    dev        = "t2.micro"
    staging    = "t3.small"
    production = "t3.large"
  }
}
```

Using a `map(string)` variable:
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type_per_environment[var.environment]

  tags = merge(var.common_tags, {
    Name        = "web-server"
    Environment = var.environment
  })
}
```

### `map(number)`

```hcl
variable "instance_counts" {
  description = "Number of instances per environment"
  type        = map(number)
  default = {
    dev        = 1
    staging    = 2
    production = 5
  }
}
```

---

## Structural Type Variables

### `object({...})`

An object groups multiple related values of potentially different types into one variable. This is very useful for configuring a "thing" that has multiple properties.

```hcl
variable "database" {
  description = "RDS database configuration"
  type = object({
    engine            = string
    engine_version    = string
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
    name              = string
    username          = string
  })
  default = {
    engine            = "postgres"
    engine_version    = "15.3"
    instance_class    = "db.t3.micro"
    allocated_storage = 20
    multi_az          = false
    name              = "appdb"
    username          = "dbadmin"
  }
}

# Using the object variable
resource "aws_db_instance" "app" {
  engine            = var.database.engine
  engine_version    = var.database.engine_version
  instance_class    = var.database.instance_class
  allocated_storage = var.database.allocated_storage
  multi_az          = var.database.multi_az
  db_name           = var.database.name
  username          = var.database.username
  password          = var.db_password
}
```

Instead of 7 separate variables (`var.db_engine`, `var.db_version`, etc.), you have one organized variable.

---

### `list(object({...}))`

This is one of the most powerful type combinations. It lets you define a list of structured items.

**Example — defining multiple subnets:**

```hcl
variable "subnets" {
  description = "List of subnets to create"
  type = list(object({
    name              = string
    cidr_block        = string
    availability_zone = string
    public            = bool
  }))
  default = [
    {
      name              = "public-1"
      cidr_block        = "10.0.1.0/24"
      availability_zone = "us-east-1a"
      public            = true
    },
    {
      name              = "public-2"
      cidr_block        = "10.0.2.0/24"
      availability_zone = "us-east-1b"
      public            = true
    },
    {
      name              = "private-1"
      cidr_block        = "10.0.10.0/24"
      availability_zone = "us-east-1a"
      public            = false
    },
    {
      name              = "private-2"
      cidr_block        = "10.0.11.0/24"
      availability_zone = "us-east-1b"
      public            = false
    }
  ]
}

# Create all subnets from the list using for_each
resource "aws_subnet" "all" {
  for_each = { for subnet in var.subnets : subnet.name => subnet }

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.availability_zone

  map_public_ip_on_launch = each.value.public

  tags = {
    Name   = each.value.name
    Public = each.value.public
  }
}
```

**Example — defining security group rules:**

```hcl
variable "security_group_rules" {
  description = "List of inbound security group rules"
  type = list(object({
    description = string
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    {
      description = "HTTP"
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "HTTPS"
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      description = "SSH from internal"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    }
  ]
}
```

---

### `map(object({...}))`

A map where each value is an object. Useful when you need to look up configurations by name.

```hcl
variable "environments" {
  description = "Configuration per environment"
  type = map(object({
    instance_type  = string
    instance_count = number
    multi_az_db    = bool
  }))
  default = {
    dev = {
      instance_type  = "t2.micro"
      instance_count = 1
      multi_az_db    = false
    }
    staging = {
      instance_type  = "t3.small"
      instance_count = 2
      multi_az_db    = false
    }
    production = {
      instance_type  = "t3.medium"
      instance_count = 5
      multi_az_db    = true
    }
  }
}

# Look up the configuration for the current environment
locals {
  env_config = var.environments[var.environment]
}

resource "aws_instance" "app" {
  count         = local.env_config.instance_count
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = local.env_config.instance_type
}
```

---

## Optional Object Attributes

In Terraform 1.3 and later, you can mark specific attributes within an object type as optional, with or without a default:

```hcl
variable "server_config" {
  type = object({
    instance_type   = string
    instance_count  = number
    enable_detailed_monitoring = optional(bool, false)   # optional with default false
    key_name        = optional(string)                   # optional with default null
    user_data_file  = optional(string, "default.sh")    # optional with default
  })
}
```

When you use `optional(type)`, the attribute does not have to be provided. When you use `optional(type, default)`, a default value is used if the attribute is omitted.

This is powerful for creating flexible modules where most config can be omitted when the defaults are fine.

---

## The `any` Type — When to Use It

The `any` type tells Terraform to accept any value without type checking:

```hcl
variable "extra_config" {
  type    = any
  default = {}
}
```

**When `any` is appropriate:**
- When writing a very generic module that accepts deeply nested structures
- When the exact shape of the data is not known at module design time
- When wrapping existing Terraform modules that use `any`

**When NOT to use `any`:**
- When you know what type the variable should be (use the specific type instead)
- In most business configuration — specific types provide safety and documentation

---

## Type Conversion in Practice

Sometimes you receive a value in one type but need it in another:

```hcl
locals {
  # Convert list to set for use with for_each
  # (for_each requires a set or map, not a list)
  az_set = toset(var.availability_zones)

  # Convert number to string for use in a name
  port_label = tostring(var.app_port)

  # Convert string "true"/"false" to bool
  enable_flag = tobool(var.enable_feature_string)

  # Convert map to list of keys
  env_names = keys(var.environments)  # ["dev", "production", "staging"]

  # Convert map to list of values
  env_configs = values(var.environments)
}
```

---

## Real-World Complete Example

Here is a complete `variables.tf` that uses a variety of types for a realistic multi-tier application:

```hcl
# General
variable "project"     { type = string }
variable "environment" { type = string }
variable "aws_region"  { type = string; default = "us-east-1" }

# Networking — using list(object) for subnets
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# Compute — using map(string) for instance types per environment
variable "instance_type_map" {
  type = map(string)
  default = {
    dev        = "t2.micro"
    staging    = "t3.small"
    production = "t3.medium"
  }
}

variable "asg_config" {
  description = "Auto Scaling Group configuration"
  type = object({
    min_size         = number
    max_size         = number
    desired_capacity = number
    health_check_type = string
  })
  default = {
    min_size         = 1
    max_size         = 3
    desired_capacity = 2
    health_check_type = "ELB"
  }
}

# Database
variable "db" {
  description = "RDS database configuration"
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  default = {
    engine         = "postgres"
    engine_version = "15"
    instance_class = "db.t3.micro"
    storage_gb     = 20
    multi_az       = false
  }
}

variable "db_password" {
  type      = string
  sensitive = true
}

# Tags — using map(string)
variable "extra_tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

---

## Summary

- Specify explicit types in all variable declarations — it catches errors early and documents the expected input
- Use `string`, `number`, `bool` for simple single values
- Use `list(string)`, `list(number)` for ordered collections of the same type
- Use `map(string)` for key-value lookups (like instance type per environment, tags)
- Use `object({...})` to group related configuration for a single "thing" (like database config)
- Use `list(object({...}))` for defining multiple items with structured configuration (like subnets or security rules)
- Use `map(object({...}))` for per-environment or per-name configurations
- Use `optional(type, default)` inside object types for flexible module inputs (Terraform 1.3+)
- Use `any` sparingly — only when you genuinely cannot know the type at design time

In the next file, we cover variable validation — how to add custom rules that ensure variable values meet your requirements.
