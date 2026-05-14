# For Expressions

## Introduction

`for` expressions allow you to transform, filter, and build new collections (lists and maps) from existing ones. They are one of the most powerful features in HCL and are heavily used in real-world Terraform configurations.

If you have ever written a list comprehension in Python or a `map()` function in JavaScript, `for` expressions work in a similar way — they process each element of a collection and produce a new collection from the results.

---

## Basic `for` Expression Syntax

A `for` expression can produce either a **list** or a **map**, depending on the brackets you use.

### Producing a List — Square Brackets `[]`

```
[for item in collection : transform(item)]
```

```hcl
locals {
  names     = ["web", "api", "db"]
  # Transform each name to uppercase
  upper_names = [for name in local.names : upper(name)]
  # Result: ["WEB", "API", "DB"]
}
```

### Producing a Map — Curly Braces `{}`

```
{for item in collection : key_expression => value_expression}
```

```hcl
locals {
  names = ["web", "api", "db"]
  # Build a map of name to length
  name_lengths = {for name in local.names : name => length(name)}
  # Result: {"web" = 3, "api" = 3, "db" = 2}
}
```

---

## Iterating Over Lists

### Transform Every Element

```hcl
locals {
  environments  = ["dev", "staging", "production"]

  # Add a prefix to every item
  env_buckets = [for env in local.environments : "myapp-${env}-data"]
  # Result: ["myapp-dev-data", "myapp-staging-data", "myapp-production-data"]

  # Convert every item to uppercase
  upper_envs = [for env in local.environments : upper(env)]
  # Result: ["DEV", "STAGING", "PRODUCTION"]

  # Check a condition on every item — produce booleans
  is_prod_list = [for env in local.environments : env == "production"]
  # Result: [false, false, true]
}
```

### Filter Elements

Add an `if` clause to include only elements that match a condition:

```hcl
locals {
  all_instances = [
    { name = "web-01", environment = "production" },
    { name = "web-02", environment = "staging" },
    { name = "api-01", environment = "production" },
    { name = "db-01",  environment = "production" }
  ]

  # Keep only production instances
  prod_instances = [
    for inst in local.all_instances : inst
    if inst.environment == "production"
  ]
  # Result: [
  #   {name = "web-01", environment = "production"},
  #   {name = "api-01", environment = "production"},
  #   {name = "db-01", environment = "production"}
  # ]

  # Get just the names of production instances
  prod_instance_names = [
    for inst in local.all_instances : inst.name
    if inst.environment == "production"
  ]
  # Result: ["web-01", "api-01", "db-01"]
}
```

### Filter and Transform Together

```hcl
locals {
  server_configs = [
    { name = "web-01",  type = "t3.medium", active = true },
    { name = "web-02",  type = "t3.medium", active = false },
    { name = "batch-01", type = "t3.large", active = true }
  ]

  # Get the uppercase names of active servers only
  active_server_names = [
    for s in local.server_configs : upper(s.name)
    if s.active
  ]
  # Result: ["WEB-01", "BATCH-01"]
}
```

---

## Iterating Over Maps

When iterating over a map, you get access to both the key and the value:

```hcl
variable "instance_types" {
  default = {
    web   = "t3.medium"
    api   = "t3.small"
    batch = "t3.large"
  }
}

locals {
  # Get just the values (instance types)
  all_types = [for role, type in var.instance_types : type]
  # Result: ["t3.medium", "t3.small", "t3.large"] (order not guaranteed)

  # Build a new map with transformed keys
  prefixed_types = {for role, type in var.instance_types : "app-${role}" => type}
  # Result: {"app-web" = "t3.medium", "app-api" = "t3.small", "app-batch" = "t3.large"}

  # Filter the map — keep only certain entries
  large_types = {
    for role, type in var.instance_types : role => type
    if type == "t3.large"
  }
  # Result: {"batch" = "t3.large"}

  # Build a list of formatted strings from map entries
  type_descriptions = [
    for role, type in var.instance_types : "${role}: ${type}"
  ]
  # Result: ["web: t3.medium", "api: t3.small", "batch: t3.large"]
}
```

---

## Building Maps from Lists

One of the most useful patterns is converting a list of objects into a map — required by `for_each`:

```hcl
locals {
  # A list of subnet configurations
  subnets = [
    { name = "public-1",  cidr = "10.0.1.0/24", az = "us-east-1a" },
    { name = "public-2",  cidr = "10.0.2.0/24", az = "us-east-1b" },
    { name = "private-1", cidr = "10.0.10.0/24", az = "us-east-1a" },
    { name = "private-2", cidr = "10.0.11.0/24", az = "us-east-1b" }
  ]

  # Convert to a map keyed by name (required for for_each)
  subnet_map = {for s in local.subnets : s.name => s}
  # Result: {
  #   "public-1"  = {name = "public-1",  cidr = "10.0.1.0/24",  az = "us-east-1a"},
  #   "public-2"  = {name = "public-2",  cidr = "10.0.2.0/24",  az = "us-east-1b"},
  #   "private-1" = {name = "private-1", cidr = "10.0.10.0/24", az = "us-east-1a"},
  #   "private-2" = {name = "private-2", cidr = "10.0.11.0/24", az = "us-east-1b"}
  # }
}

# Use for_each with the map
resource "aws_subnet" "all" {
  for_each = local.subnet_map

  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az

  tags = {
    Name = each.key  # "public-1", "public-2", etc.
  }
}
```

---

## Real-World Terraform Examples

### Example 1: Build Security Group Rules from a Variable

```hcl
variable "ingress_rules" {
  type = list(object({
    description = string
    port        = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  default = [
    { description = "HTTP",  port = 80,  protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { description = "HTTPS", port = 443, protocol = "tcp", cidr_blocks = ["0.0.0.0/0"] },
    { description = "SSH",   port = 22,  protocol = "tcp", cidr_blocks = ["10.0.0.0/8"] }
  ]
}

locals {
  # Build a map for for_each (key = port number as string)
  ingress_rule_map = {
    for rule in var.ingress_rules : tostring(rule.port) => rule
  }
}

resource "aws_security_group" "web" {
  name = "web-sg"

  dynamic "ingress" {
    for_each = local.ingress_rule_map
    content {
      description = ingress.value.description
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

### Example 2: Create IAM Users from a List

```hcl
variable "developers" {
  type    = list(string)
  default = ["alice", "bob", "charlie"]
}

locals {
  # Convert list to set for for_each
  developer_set = toset(var.developers)
}

resource "aws_iam_user" "developers" {
  for_each = local.developer_set
  name     = each.value
  tags     = { Team = "engineering" }
}

# Output a map of username to ARN
output "developer_arns" {
  value = {for name, user in aws_iam_user.developers : name => user.arn}
}
```

### Example 3: Extract Subnet IDs After Creation

```hcl
# After creating subnets with for_each, extract their IDs
resource "aws_subnet" "public" {
  for_each = toset(["us-east-1a", "us-east-1b"])

  vpc_id            = aws_vpc.main.id
  availability_zone = each.value
  cidr_block        = "10.0.${index(tolist(toset(["us-east-1a", "us-east-1b"])), each.value)}.0/24"
}

locals {
  # Get list of all public subnet IDs (for use in load balancer, etc.)
  public_subnet_ids = [for subnet in aws_subnet.public : subnet.id]

  # Get map of AZ to subnet ID
  az_to_subnet_id = {for az, subnet in aws_subnet.public : az => subnet.id}
}

resource "aws_lb" "main" {
  name               = "main-alb"
  load_balancer_type = "application"
  subnets            = local.public_subnet_ids  # list of IDs
}
```

### Example 4: Build SSM Parameters from a Map

```hcl
variable "app_config" {
  type = map(string)
  default = {
    "database/host"      = "db.example.com"
    "database/name"      = "appdb"
    "api/base_url"       = "https://api.example.com"
    "feature/dark_mode"  = "enabled"
  }
}

resource "aws_ssm_parameter" "config" {
  for_each = var.app_config

  name  = "/${var.project}/${var.environment}/${each.key}"
  type  = "String"
  value = each.value

  tags = local.common_tags
}

# Output all created parameter names
output "parameter_names" {
  value = [for k, param in aws_ssm_parameter.config : param.name]
}
```

### Example 5: Filter Resources by Tag

```hcl
data "aws_instances" "all" {
  filter {
    name   = "tag:Project"
    values = [var.project]
  }
}

locals {
  # Get all instance IDs
  all_instance_ids = data.aws_instances.all.ids

  # Build a map from the instances data
  instance_map = {
    for id in data.aws_instances.all.ids : id => {
      id = id
    }
  }
}
```

---

## The `alltrue` and `anytrue` Functions with `for`

Combine `for` with `alltrue()` or `anytrue()` to validate collections:

```hcl
variable "cidr_blocks" {
  type    = list(string)
  default = ["10.0.0.0/8", "172.16.0.0/12"]
}

locals {
  # Check if ALL CIDRs are valid
  all_valid_cidrs = alltrue([
    for cidr in var.cidr_blocks : can(cidrhost(cidr, 0))
  ])

  # Check if ANY port is privileged (below 1024)
  any_privileged_port = anytrue([
    for port in var.allowed_ports : port < 1024
  ])

  # Find all non-private CIDRs (for security checking)
  non_private_cidrs = [
    for cidr in var.cidr_blocks : cidr
    if !startswith(cidr, "10.") && !startswith(cidr, "172.") && !startswith(cidr, "192.168.")
  ]
}
```

---

## Common Pitfalls

### Pitfall 1: `for_each` requires a map or set, not a list

```hcl
# WRONG — for_each does not accept a list
resource "aws_iam_user" "devs" {
  for_each = var.developers  # Error if developers is a list
  name     = each.value
}

# CORRECT — convert to set first
resource "aws_iam_user" "devs" {
  for_each = toset(var.developers)
  name     = each.value
}
```

### Pitfall 2: Map keys must be unique

```hcl
locals {
  items = [
    { name = "web", type = "t2.micro" },
    { name = "web", type = "t3.small" }  # duplicate name!
  ]

  # WRONG — duplicate keys will cause an error
  item_map = {for i in local.items : i.name => i.type}
  # Error: Duplicate key: "web"
}
```

Make sure the key you choose in a `for` map expression is unique across all items.

### Pitfall 3: Order of list results from maps is not guaranteed

```hcl
locals {
  my_map = { b = 2, a = 1, c = 3 }

  # The order of results from iterating a map is NOT guaranteed
  values = [for k, v in local.my_map : v]
  # Could be [2, 1, 3] or [1, 2, 3] or any order
}
```

If you need a specific order, sort the results:
```hcl
locals {
  sorted_keys = sort(keys(local.my_map))
  sorted_values = [for k in local.sorted_keys : local.my_map[k]]
}
```

---

## Summary

- `for` expressions transform collections — lists become new lists or maps, maps become new lists or maps
- List output uses `[]`: `[for item in collection : transform(item)]`
- Map output uses `{}`: `{for item in collection : key => value}`
- Add `if condition` to filter elements
- When iterating maps, use `for key, value in map` to access both key and value
- Converting lists to maps with `{for item in list : item.key => item}` enables `for_each`
- Combine with `alltrue()` and `anytrue()` for collection validation
- Map keys must be unique — choose keys carefully
- Map iteration order is not guaranteed — sort if order matters

In the next file, we cover Terraform's built-in functions — the tools that make string manipulation, math, and collection operations easy.
