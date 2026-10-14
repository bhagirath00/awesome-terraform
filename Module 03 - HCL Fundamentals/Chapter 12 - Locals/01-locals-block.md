# The Locals Block

## Introduction

**Locals** (short for local values) are named values that you compute inside your Terraform configuration and reuse throughout it. They are like variables you define yourself — not passed in from outside, but calculated internally based on your inputs and resource data.

The `locals` block is one of those features that seems optional at first, but once you start using it properly, your Terraform code becomes dramatically cleaner, more readable, and easier to maintain.

---

## Why Locals Exist

Consider this configuration without locals:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Name        = "${var.project}-${var.environment}-web-server"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
    CostCenter  = "engineering-${var.environment}"
  }
}

resource "aws_s3_bucket" "app" {
  bucket = "${var.project}-${var.environment}-app-data"
  tags = {
    Name        = "${var.project}-${var.environment}-app-data"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
    CostCenter  = "engineering-${var.environment}"
  }
}

resource "aws_db_instance" "main" {
  identifier = "${var.project}-${var.environment}-database"
  tags = {
    Name        = "${var.project}-${var.environment}-database"
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
    CostCenter  = "engineering-${var.environment}"
  }
}
```

Problems with this approach:
- The tags block is copied three times
- The prefix `"${var.project}-${var.environment}"` is repeated everywhere
- If you need to change a tag (like adding a new tag or changing `ManagedBy`), you must update every resource
- It is verbose and hard to scan

Now with locals:

```hcl
locals {
  prefix = "${var.project}-${var.environment}"

  common_tags = {
    Environment = var.environment
    Project     = var.project
    ManagedBy   = "Terraform"
    CostCenter  = "engineering-${var.environment}"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = merge(local.common_tags, { Name = "${local.prefix}-web-server" })
}

resource "aws_s3_bucket" "app" {
  bucket = "${local.prefix}-app-data"
  tags   = merge(local.common_tags, { Name = "${local.prefix}-app-data" })
}

resource "aws_db_instance" "main" {
  identifier = "${local.prefix}-database"
  tags       = merge(local.common_tags, { Name = "${local.prefix}-database" })
}
```

Much cleaner. The tags are defined once. The prefix is defined once. If you need to change anything, you change it in one place.

---

## Declaring Locals

Locals are declared using the `locals` block (note: always plural, even for a single value):

```hcl
locals {
  name1 = value1
  name2 = value2
  name3 = value3
}
```

You can have multiple `locals` blocks in the same configuration — Terraform merges them all. This is useful for organizing locals by category:

```hcl
# Naming locals
locals {
  prefix      = "${var.project}-${var.environment}"
  name_suffix = "${var.environment}-${var.aws_region}"
}

# Tag locals
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Computed configuration locals
locals {
  instance_type     = var.environment == "production" ? "t3.large" : "t2.micro"
  enable_monitoring = var.environment == "production"
  db_multi_az       = var.environment == "production"
}
```

---

## Referencing Locals

Reference locals using the `local.` prefix (note: singular, unlike the block name):

```hcl
locals {
  prefix = "${var.project}-${var.environment}"
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Name = "${local.prefix}-vpc"   # reference with local.
  }
}
```

Locals can also reference other locals:

```hcl
locals {
  environment = var.environment
  project     = var.project

  # This local references other locals
  prefix = "${local.project}-${local.environment}"

  # This references the prefix local
  bucket_name = "${local.prefix}-data-${random_id.suffix.hex}"
}
```

**Important:** Terraform handles the ordering automatically. You do not have to declare locals in dependency order — Terraform figures out which locals depend on which and evaluates them in the correct order.

---

## What You Can Put in a Local Value

Locals can hold any valid HCL expression — not just simple strings.

### Simple Values

```hcl
locals {
  region          = "us-east-1"
  account_id      = "123456789012"
  max_retries     = 3
  is_production   = var.environment == "production"
}
```

### Computed Strings

```hcl
locals {
  prefix           = "${var.project}-${var.environment}"
  bucket_name      = "${local.prefix}-${data.aws_caller_identity.current.account_id}"
  log_group_name   = "/aws/${var.project}/${var.environment}/app"
  ssm_prefix       = "/${var.project}/${var.environment}"
}
```

### Maps

```hcl
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
    Repository  = "github.com/myorg/infrastructure"
  }

  instance_type_map = {
    dev        = "t2.micro"
    staging    = "t3.small"
    production = "t3.large"
  }
}
```

### Lists

```hcl
locals {
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

  # Build a list from a range
  port_list = [80, 443, 8080, 8443]

  # Build a list using a for expression
  subnet_names = [for i in range(3) : "${local.prefix}-private-${i + 1}"]
}
```

### Conditional Values

```hcl
locals {
  # Choose instance type based on environment
  instance_type = var.environment == "production" ? "t3.large" : "t2.micro"

  # Enable features only in production
  enable_detailed_monitoring = var.environment == "production"
  multi_az                   = var.environment == "production"
  deletion_protection        = var.environment == "production"

  # Set backup retention based on environment
  backup_retention_days = var.environment == "production" ? 30 : 7
}
```

### Expressions Using Resource Attributes

```hcl
locals {
  # Build a connection string from resource attributes
  db_connection_string = "postgres://${var.db_username}@${aws_db_instance.app.endpoint}/${var.db_name}"

  # Extract the hostname from an endpoint (removes the port)
  db_host = split(":", aws_db_instance.app.endpoint)[0]

  # Build a map of subnet name to ID from a resource
  subnet_id_map = {
    for subnet in aws_subnet.all : subnet.tags["Name"] => subnet.id
  }
}
```

### Function Calls

```hcl
locals {
  # Use functions to transform values
  upper_env      = upper(var.environment)         # "PRODUCTION"
  lower_project  = lower(var.project)             # "myapp"
  trimmed_name   = trimspace(var.project_name)    # removes leading/trailing spaces

  # Merge maps
  all_tags = merge(
    local.common_tags,
    var.extra_tags,
    { Name = local.prefix }
  )

  # Flatten nested lists
  all_subnet_ids = flatten([
    aws_subnet.public[*].id,
    aws_subnet.private[*].id
  ])

  # Filter a list
  prod_instances = [
    for instance in var.instance_configs :
    instance if instance.environment == "production"
  ]
}
```

---

## Common Patterns

### Pattern 1: Naming Prefix

Define a prefix once and use it everywhere for consistent naming:

```hcl
locals {
  prefix = "${var.project}-${var.environment}"
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = { Name = "${local.prefix}-vpc" }
}

resource "aws_lb" "main" {
  name = "${local.prefix}-alb"
}

resource "aws_db_instance" "app" {
  identifier = "${local.prefix}-db"
}

resource "aws_s3_bucket" "app" {
  bucket = "${local.prefix}-app-data"
}
```

If you ever need to change the naming convention, you change one line — the `prefix` local — and all resources are updated.

---

### Pattern 2: Common Tags

Define shared tags once and merge extras per resource:

```hcl
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = var.team_email
  }
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = merge(local.common_tags, {
    Name = "${local.prefix}-web"
    Role = "web-server"
  })
}

resource "aws_db_instance" "app" {
  identifier = "${local.prefix}-db"
  tags = merge(local.common_tags, {
    Name    = "${local.prefix}-db"
    Backup  = "required"
  })
}
```

---

### Pattern 3: Environment-Based Configuration

Use locals to centralize environment-specific decisions:

```hcl
locals {
  # All environment-specific configuration in one place
  env_config = {
    dev = {
      instance_type       = "t2.micro"
      instance_count      = 1
      db_instance_class   = "db.t3.micro"
      multi_az            = false
      deletion_protection = false
      backup_retention    = 1
    }
    staging = {
      instance_type       = "t3.small"
      instance_count      = 2
      db_instance_class   = "db.t3.small"
      multi_az            = false
      deletion_protection = false
      backup_retention    = 7
    }
    production = {
      instance_type       = "t3.medium"
      instance_count      = 5
      db_instance_class   = "db.t3.medium"
      multi_az            = true
      deletion_protection = true
      backup_retention    = 30
    }
  }

  # Get the config for the current environment
  config = local.env_config[var.environment]
}

# Use the config values throughout
resource "aws_instance" "app" {
  count         = local.config.instance_count
  instance_type = local.config.instance_type
}

resource "aws_db_instance" "app" {
  instance_class      = local.config.db_instance_class
  multi_az            = local.config.multi_az
  deletion_protection = local.config.deletion_protection
  backup_retention_period = local.config.backup_retention
}
```

This pattern means all environment-specific decisions are in one map. Adding a new environment is just adding an entry to the map.

---

### Pattern 4: Building Derived Data

Use locals to derive values that resources will use:

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

locals {
  # Take only the first N availability zones
  azs = slice(data.aws_availability_zones.available.names, 0, var.az_count)

  # Build subnet CIDRs programmatically
  public_subnet_cidrs = [
    for i in range(var.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]
  private_subnet_cidrs = [
    for i in range(var.az_count) :
    cidrsubnet(var.vpc_cidr, 8, i + 10)
  ]
}

resource "aws_subnet" "public" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnet_cidrs[count.index]
  availability_zone = local.azs[count.index]
}
```

---

## Organizing Locals

For larger configurations, use multiple `locals` blocks grouped by category:

```hcl
# locals.tf

# Naming
locals {
  prefix = "${var.project}-${var.environment}"
}

# Tags
locals {
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# Network configuration
locals {
  azs                  = slice(data.aws_availability_zones.available.names, 0, 2)
  public_subnet_cidrs  = [for i in range(2) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnet_cidrs = [for i in range(2) : cidrsubnet(var.vpc_cidr, 8, i + 10)]
}

# Environment-based compute settings
locals {
  instance_type       = var.environment == "production" ? "t3.large" : "t2.micro"
  enable_monitoring   = var.environment == "production"
}
```

---

## Summary

- Locals are named values defined inside your Terraform configuration — computed, not passed in from outside
- Declare them with the `locals` block (plural) and reference them with `local.name` (singular)
- Use locals to eliminate repetition — define once, use many times
- Locals can hold any expression: strings, numbers, booleans, lists, maps, conditionals, function calls, or expressions using resource attributes
- Multiple `locals` blocks in the same configuration are merged automatically
- Locals can reference other locals — Terraform handles ordering automatically
- Core patterns: naming prefix, common tags, environment-based configuration, and derived data
- Locals make your configurations DRY (Don't Repeat Yourself) and easier to maintain

In the next file, we compare locals with variables — understanding the difference helps you choose the right one for each situation.
