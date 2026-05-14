# Conditional Expressions

## Introduction

Conditional expressions let your Terraform configuration make decisions based on values. Instead of creating separate configurations for dev and production, or separate files for different regions, you can write one configuration that adapts based on variable values.

Terraform supports the ternary conditional expression — the same pattern used in many programming languages. This file covers how it works, all the ways it is used, and important limitations to understand.

---

## The Ternary Conditional Expression

The conditional expression has this format:

```
condition ? value_if_true : value_if_false
```

Reading it in plain English: "If `condition` is true, use `value_if_true`. Otherwise, use `value_if_false`."

```hcl
# Simple example
locals {
  instance_type = var.environment == "production" ? "t3.large" : "t2.micro"
}
```

Breaking this down:
- `var.environment == "production"` — the condition (evaluates to true or false)
- `"t3.large"` — the value if the condition is true
- `"t2.micro"` — the value if the condition is false

---

## Conditions You Can Use

The condition can be any expression that evaluates to a boolean (`true` or `false`).

### Equality and Inequality

```hcl
locals {
  # Equality check
  is_prod = var.environment == "production"

  # Inequality check
  not_prod = var.environment != "production"

  # Use in ternary
  instance_type  = var.environment == "production" ? "t3.large" : "t2.micro"
  backup_days    = var.environment != "production" ? 7 : 30
}
```

### Comparison Operators

```hcl
locals {
  # Greater than
  needs_large = var.instance_count > 5 ? "t3.xlarge" : "t3.small"

  # Less than or equal
  small_team  = var.team_size <= 3 ? "t2.micro" : "t3.medium"

  # Greater than or equal
  high_volume = var.requests_per_second >= 1000 ? true : false
}
```

### Boolean Variables

```hcl
variable "enable_https" {
  type    = bool
  default = true
}

locals {
  # Use a bool variable directly as the condition
  lb_protocol   = var.enable_https ? "HTTPS" : "HTTP"
  lb_port       = var.enable_https ? 443 : 80

  # Or negate it
  allow_http    = !var.enable_https ? true : false
}
```

### Logical Operators — AND (`&&`) and OR (`||`)

```hcl
locals {
  # Both conditions must be true
  needs_ha = var.environment == "production" && var.critical_service ? true : false

  # At least one condition must be true
  use_large = var.environment == "production" || var.high_traffic ? "t3.large" : "t2.micro"

  # NOT operator
  allow_destroy = !var.protect_resources

  # Complex condition
  enable_advanced_logging = (var.environment == "production" || var.environment == "staging") && var.debug_mode
}
```

### Function Results as Conditions

```hcl
locals {
  # contains() returns bool — use as condition
  is_aws_region = contains(["us-east-1", "us-west-2", "eu-west-1"], var.region)
  region_label  = contains(["us-east-1", "us-west-2"], var.region) ? "US" : "International"

  # length() > 0 to check if a list is non-empty
  has_extra_tags = length(var.extra_tags) > 0
  tag_note       = length(var.extra_tags) > 0 ? "Custom tags applied" : "Default tags only"

  # startswith() as condition
  is_t_series = startswith(var.instance_type, "t") ? "burstable" : "dedicated"
}
```

---

## What Values Can Be Returned

Both the true and false values can be any type — as long as they are the same type.

### Returning Strings

```hcl
locals {
  env_class      = var.environment == "production" ? "prod" : "non-prod"
  db_engine      = var.use_postgres ? "postgres" : "mysql"
  protocol       = var.enable_https ? "https" : "http"
  log_level      = var.debug ? "DEBUG" : "INFO"
}
```

### Returning Numbers

```hcl
locals {
  backup_retention    = var.environment == "production" ? 30 : 7
  instance_count      = var.high_availability ? 3 : 1
  volume_size         = var.large_storage ? 500 : 100
  max_connections     = var.environment == "production" ? 1000 : 100
}
```

### Returning Booleans

```hcl
locals {
  enable_monitoring    = var.environment == "production" ? true : false
  multi_az             = var.environment == "production"    # same as above, simpler
  deletion_protection  = var.environment == "production"
  force_destroy        = var.environment != "production"
}
```

### Returning Lists

```hcl
locals {
  # Return different lists based on a condition
  open_ports = var.enable_https ? [80, 443] : [80]

  # Different availability zones for different environments
  azs = var.high_availability ? ["us-east-1a", "us-east-1b", "us-east-1c"] : ["us-east-1a"]

  # Different subnet selections
  deploy_subnets = var.environment == "production" ? local.private_subnet_ids : local.public_subnet_ids
}
```

### Returning Entire Maps or Objects

```hcl
locals {
  db_config = var.environment == "production" ? {
    instance_class   = "db.t3.medium"
    allocated_storage = 100
    multi_az         = true
    backup_retention  = 30
  } : {
    instance_class    = "db.t3.micro"
    allocated_storage = 20
    multi_az          = false
    backup_retention  = 7
  }
}

resource "aws_db_instance" "app" {
  instance_class          = local.db_config.instance_class
  allocated_storage       = local.db_config.allocated_storage
  multi_az                = local.db_config.multi_az
  backup_retention_period = local.db_config.backup_retention
}
```

---

## Real-World Usage Patterns

### Pattern 1: Environment-Based Configuration

The most common use — configure things differently based on the environment:

```hcl
locals {
  # Instance sizing
  instance_type    = var.environment == "production" ? "t3.medium" : "t2.micro"
  db_instance_type = var.environment == "production" ? "db.t3.medium" : "db.t3.micro"

  # High availability settings
  multi_az            = var.environment == "production"
  deletion_protection = var.environment == "production"

  # Retention and backup
  log_retention_days  = var.environment == "production" ? 90 : 14
  backup_retention    = var.environment == "production" ? 30 : 7

  # Scaling
  min_capacity = var.environment == "production" ? 2 : 1
  max_capacity = var.environment == "production" ? 20 : 3
}
```

### Pattern 2: Feature Flags

Enable or disable features based on boolean variables:

```hcl
variable "enable_waf"         { type = bool; default = false }
variable "enable_cdn"         { type = bool; default = false }
variable "enable_monitoring"  { type = bool; default = true }

locals {
  alb_access_logs_enabled = var.enable_monitoring ? "true" : "false"
  cdn_price_class         = var.enable_cdn ? "PriceClass_All" : "PriceClass_100"
  waf_association_needed  = var.enable_waf
}

# Conditionally create a WAF association
resource "aws_wafv2_web_acl_association" "alb" {
  count        = var.enable_waf ? 1 : 0    # create only if WAF is enabled
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main[0].arn
}
```

### Pattern 3: Count-Based Conditional Resource Creation

The `count = condition ? 1 : 0` pattern is used to conditionally create resources:

```hcl
# Create a NAT gateway only in production
resource "aws_nat_gateway" "main" {
  count         = var.environment == "production" ? 1 : 0
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}

# Create a CloudFront distribution only when CDN is enabled
resource "aws_cloudfront_distribution" "main" {
  count   = var.enable_cdn ? 1 : 0
  # ... configuration
}

# Create a read replica only for production
resource "aws_db_instance" "read_replica" {
  count               = var.environment == "production" ? 1 : 0
  identifier          = "${local.prefix}-db-replica"
  replicate_source_db = aws_db_instance.primary.identifier
  instance_class      = "db.t3.small"
}
```

Accessing a conditionally-created resource:
```hcl
# Must use [0] index because count resources are lists
output "nat_gateway_ip" {
  value = var.environment == "production" ? aws_nat_gateway.main[0].public_ip : "N/A"
}
```

### Pattern 4: Selecting from Multiple Options

When you have more than two choices, nest conditionals or use a lookup map:

**Nested ternary (works but gets hard to read quickly):**
```hcl
locals {
  instance_type = (
    var.environment == "production" ? "t3.large" :
    var.environment == "staging"    ? "t3.small" :
    "t2.micro"  # default for dev and everything else
  )
}
```

**Lookup map (usually cleaner for multiple options):**
```hcl
locals {
  instance_type_map = {
    production = "t3.large"
    staging    = "t3.small"
    dev        = "t2.micro"
  }

  instance_type = lookup(local.instance_type_map, var.environment, "t2.micro")
  # lookup(map, key, default) — returns default if key is not found
}
```

The lookup map approach is generally more readable when you have three or more options.

---

## Using Conditionals in Resource Arguments

Conditionals can be used directly in resource arguments:

```hcl
resource "aws_db_instance" "app" {
  identifier     = "${local.prefix}-database"
  engine         = "postgres"
  instance_class = var.environment == "production" ? "db.t3.medium" : "db.t3.micro"

  # Multi-AZ only in production
  multi_az = var.environment == "production"

  # More backup in production
  backup_retention_period = var.environment == "production" ? 30 : 7

  # Delete protection in production
  deletion_protection = var.environment == "production"

  # Encrypt in production and staging, not dev
  storage_encrypted = var.environment != "dev"

  tags = merge(local.common_tags, {
    Name = "${local.prefix}-database"
    Tier = var.environment == "production" ? "primary" : "secondary"
  })
}
```

---

## Limitations of Conditionals

### Both branches must return the same type

```hcl
# WRONG — returns string or number (different types)
locals {
  value = var.flag ? "enabled" : 1
}

# CORRECT — both return strings
locals {
  value = var.flag ? "enabled" : "disabled"
}
```

### Conditionals cannot create or skip blocks

```hcl
# WRONG — you cannot conditionally include a block like this
resource "aws_instance" "web" {
  ami = "ami-0c55b159cbfafe1f0"

  # This does not work
  if var.enable_monitoring {
    monitoring = true
  }
}

# CORRECT — just use the conditional in the argument value
resource "aws_instance" "web" {
  ami        = "ami-0c55b159cbfafe1f0"
  monitoring = var.enable_monitoring ? true : false  # or just: var.enable_monitoring
}
```

For conditionally creating entire nested blocks, use the `dynamic` block (covered in Module 9).

### Conditionals in `count` require careful resource referencing

When using `count = 0` to skip creating a resource, any outputs or references to that resource must also handle the case where it does not exist:

```hcl
resource "aws_nat_gateway" "main" {
  count = var.environment == "production" ? 1 : 0
  # ...
}

# WRONG — will fail in dev because the resource doesn't exist
output "nat_ip" {
  value = aws_nat_gateway.main.public_ip  # error: no instances
}

# CORRECT — handle the empty case
output "nat_ip" {
  value = var.environment == "production" ? aws_nat_gateway.main[0].public_ip : "N/A"
}
```

---

## Summary

- The ternary conditional expression syntax is: `condition ? value_if_true : value_if_false`
- Conditions can use `==`, `!=`, `>`, `<`, `>=`, `<=`, `&&`, `||`, `!`, and function results
- Both branches must return the same data type
- Common patterns: environment-based configuration, feature flags, conditional resource creation with `count = condition ? 1 : 0`, selecting from multiple options
- For more than two options, use a lookup map instead of nested ternaries for better readability
- Conditionals work in locals, resource arguments, and output values
- Conditionals cannot conditionally include entire blocks — use `dynamic` blocks for that
- When using `count = 0`, always handle the empty case when referencing those resources

In the next file, we cover `for` expressions — how to transform, filter, and build new collections from existing ones.
