# HCL Data Types

## Introduction

Every value in HCL has a type. Understanding types is important because:
- Terraform validates that you are passing the right type of value to each argument
- Variables have types that determine what values they accept
- Some functions only work with specific types
- Type mismatches cause clear error messages during `terraform validate` or `terraform plan`

HCL has two categories of types: **primitive types** (simple single values) and **complex types** (collections of values).

---

## Primitive Types

Primitive types hold a single value. There are three primitive types in HCL.

### `string`

A string is a sequence of characters — text. Strings are always written inside double quotes.

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

resource "aws_s3_bucket" "app" {
  bucket = "my-application-bucket"   # string value
}
```

**String interpolation** — embed expressions inside strings using `${}`:

```hcl
locals {
  environment = "production"
  bucket_name = "myapp-${local.environment}-data"  # "myapp-production-data"
  server_name = "web-server-${count.index + 1}"    # "web-server-1", "web-server-2", etc.
}
```

**Multi-line strings** — use heredoc syntax for strings that span multiple lines:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  user_data = <<-EOF
    #!/bin/bash
    apt-get update -y
    apt-get install -y nginx
    systemctl start nginx
    systemctl enable nginx
    echo "Server ready" > /var/www/html/index.html
  EOF
}
```

The `<<-EOF` syntax allows indentation inside the heredoc — the leading whitespace is stripped. The `EOF` marker must appear on its own line and marks the end of the string.

---

### `number`

A number represents a numeric value. It can be an integer or a decimal.

```hcl
variable "instance_count" {
  type    = number
  default = 3
}

variable "disk_size_gb" {
  type    = number
  default = 20
}

resource "aws_autoscaling_group" "web" {
  min_size         = 1          # number
  max_size         = 10         # number
  desired_capacity = 3          # number
}
```

Numbers in HCL do not use quotes:
```hcl
# Correct
port = 80

# Wrong — this is a string "80", not the number 80
port = "80"
```

Some arguments expect a number. If you pass a string, Terraform may convert it automatically (implicit conversion), but it is best practice to use the correct type.

---

### `bool`

A boolean represents a true or false value. In HCL, booleans are written as `true` or `false` without quotes.

```hcl
variable "enable_monitoring" {
  type    = bool
  default = false
}

resource "aws_instance" "web" {
  ami                     = "ami-0c55b159cbfafe1f0"
  instance_type           = "t2.micro"
  monitoring              = true            # bool
  ebs_optimized           = false           # bool
  disable_api_termination = var.enable_monitoring
}
```

Common mistakes with booleans:

```hcl
# Correct
monitoring = true

# Wrong — "true" is a string, not a boolean
monitoring = "true"

# Also wrong
monitoring = 1  # This is a number, not a boolean (though Terraform may convert it)
```

---

## Complex Types

Complex types hold multiple values. There are two main categories: **collection types** and **structural types**.

---

## Collection Types

Collection types group multiple values of the **same type** together.

### `list`

A list is an ordered sequence of values. Lists are written inside square brackets `[]`. The values must all be the same type.

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "allowed_ports" {
  type    = list(number)
  default = [80, 443, 8080]
}
```

**Accessing list elements** — use index (starting from 0):

```hcl
locals {
  azs         = ["us-east-1a", "us-east-1b", "us-east-1c"]
  first_az    = local.azs[0]   # "us-east-1a"
  second_az   = local.azs[1]   # "us-east-1b"
  last_az     = local.azs[2]   # "us-east-1c"
}
```

**Using lists in resources:**

```hcl
resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8", "172.16.0.0/12"]  # list of strings
  }
}
```

**Common list operations:**

```hcl
locals {
  items  = ["a", "b", "c", "d"]
  count  = length(local.items)        # 4
  first  = local.items[0]             # "a"
  merged = concat(["x"], local.items) # ["x", "a", "b", "c", "d"]
}
```

---

### `set`

A set is similar to a list but:
- Values are **unordered** — you cannot guarantee what order they appear in
- Values must be **unique** — no duplicates allowed

```hcl
variable "allowed_regions" {
  type    = set(string)
  default = ["us-east-1", "us-west-2", "eu-west-1"]
}
```

Sets are useful when the order does not matter and you want uniqueness enforced. Many AWS resource arguments that appear to be lists are actually sets internally. Use `toset()` to convert a list to a set:

```hcl
locals {
  regions_list = ["us-east-1", "us-west-2", "us-east-1"]  # duplicate!
  regions_set  = toset(local.regions_list)  # {"us-east-1", "us-west-2"} — duplicate removed
}
```

---

### `map`

A map is a collection of key-value pairs where all values are the same type. Maps are written inside curly braces `{}`.

```hcl
variable "instance_types" {
  type = map(string)
  default = {
    dev     = "t2.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }
}
```

**Accessing map values** — use the key:

```hcl
locals {
  instance_types = {
    dev  = "t2.micro"
    prod = "t3.large"
  }

  dev_type  = local.instance_types["dev"]   # "t2.micro"
  prod_type = local.instance_types["prod"]  # "t3.large"

  # Using lookup with a default value
  staging_type = lookup(local.instance_types, "staging", "t3.small")  # "t3.small" (default)
}
```

**Common map uses — tags:**

```hcl
locals {
  common_tags = {
    ManagedBy   = "Terraform"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags       = local.common_tags  # passing the entire map
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = merge(local.common_tags, {  # merge with additional tags
    Name = "web-server"
    Role = "frontend"
  })
}
```

---

## Structural Types

Structural types hold values of **different types** in a defined structure.

### `object`

An object is a collection of named attributes, each with its own type. Unlike a map (where all values must be the same type), an object can have different types for each attribute.

```hcl
variable "database_config" {
  type = object({
    engine            = string
    engine_version    = string
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
  })

  default = {
    engine            = "mysql"
    engine_version    = "8.0"
    instance_class    = "db.t3.micro"
    allocated_storage = 20
    multi_az          = false
  }
}

# Use the object variable
resource "aws_db_instance" "app" {
  engine            = var.database_config.engine
  engine_version    = var.database_config.engine_version
  instance_class    = var.database_config.instance_class
  allocated_storage = var.database_config.allocated_storage
  multi_az          = var.database_config.multi_az
}
```

Objects are useful for grouping related configuration together into a single variable rather than having many separate variables.

---

### `tuple`

A tuple is an ordered sequence of values where each position has a specific type. Tuples are less commonly used in Terraform than the other types.

```hcl
variable "server_config" {
  type    = tuple([string, number, bool])
  default = ["t2.micro", 20, true]
  # position 0 = instance type (string)
  # position 1 = disk size in GB (number)
  # position 2 = enable monitoring (bool)
}
```

In practice, it is usually clearer to use an `object` instead of a `tuple` because object attributes have names.

---

## The `any` Type

The `any` type is a special type that tells Terraform to accept any type of value. It skips type checking.

```hcl
variable "extra_tags" {
  type    = any
  default = {}
}
```

Use `any` sparingly. It removes the protection that type checking provides. It is occasionally useful when writing flexible modules that need to accept values of different shapes, but for most variables, specify the exact type.

---

## Type Conversion

Terraform can automatically convert between compatible types in some cases. This is called **implicit type conversion**.

```hcl
variable "port" {
  type = number
}

# If you pass "80" (string) where a number is expected,
# Terraform will convert it to 80 (number) automatically in some cases
```

You can also explicitly convert types using built-in functions:

```hcl
locals {
  # Convert number to string
  port_string = tostring(80)           # "80"

  # Convert string to number
  size_number = tonumber("20")         # 20

  # Convert value to bool
  flag = tobool("true")               # true

  # Convert list to set (removes duplicates)
  unique_items = toset(["a", "b", "a"])  # {"a", "b"}

  # Convert set to list (makes it ordered)
  ordered = tolist(toset(["c", "a", "b"]))  # ["a", "b", "c"] (sorted)

  # Convert map to object
  config = tomap({name = "app", env = "prod"})
}
```

---

## Type Constraints in Variables

When you declare a variable, the `type` argument sets a type constraint. Terraform will validate that any value passed for that variable matches the constraint.

```hcl
# Only accepts strings
variable "region" {
  type = string
}

# Only accepts numbers
variable "count" {
  type = number
}

# Only accepts a list of strings
variable "subnets" {
  type = list(string)
}

# Only accepts a map where all values are strings
variable "tags" {
  type = map(string)
}

# Only accepts an object with this exact structure
variable "app_config" {
  type = object({
    name     = string
    replicas = number
    debug    = bool
  })
}

# Accepts a list of objects with this structure
variable "servers" {
  type = list(object({
    name          = string
    instance_type = string
    region        = string
  }))
}
```

---

## Real-World Example Using Multiple Types

Here is a realistic Terraform configuration that uses several different types together:

```hcl
variable "project" {
  type        = string
  description = "Project name"
  default     = "myapp"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
  default     = "dev"
}

variable "instance_count" {
  type        = number
  description = "Number of EC2 instances"
  default     = 2
}

variable "enable_https" {
  type        = bool
  description = "Enable HTTPS (port 443)"
  default     = true
}

variable "allowed_cidrs" {
  type        = list(string)
  description = "CIDR blocks allowed to access the servers"
  default     = ["0.0.0.0/0"]
}

variable "instance_type_map" {
  type        = map(string)
  description = "Instance type per environment"
  default = {
    dev     = "t2.micro"
    staging = "t3.small"
    prod    = "t3.medium"
  }
}

locals {
  # Use the map to look up the correct instance type for the current environment
  instance_type = var.instance_type_map[var.environment]

  # Build common tags map
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }

  # Build the list of ports to open
  open_ports = var.enable_https ? [80, 443] : [80]
}

resource "aws_security_group" "app" {
  name = "${var.project}-${var.environment}-sg"

  dynamic "ingress" {
    for_each = local.open_ports  # iterate over the list
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = var.allowed_cidrs  # list of strings
    }
  }

  tags = local.common_tags  # map
}
```

---

## Summary

- **Primitive types**: `string` (text), `number` (numeric), `bool` (true/false)
- **Collection types**: `list` (ordered, same type), `set` (unordered, unique, same type), `map` (key-value pairs, same value type)
- **Structural types**: `object` (named attributes with different types), `tuple` (ordered with positional types)
- **`any`** type skips type checking — use sparingly
- Type constraints in variables help catch errors early and make modules self-documenting
- Use explicit conversion functions (`tostring`, `tonumber`, `tobool`, `toset`, `tolist`, `tomap`) when you need to convert between types
- Complex types like `list(object({...}))` are very powerful for configuring groups of similar resources

In the next chapter, we cover variables in depth — how to declare them, set values, use them in configurations, and validate them.
