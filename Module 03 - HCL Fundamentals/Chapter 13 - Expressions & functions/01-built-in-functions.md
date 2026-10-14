# Built-in Functions

## Introduction

Terraform has a rich library of built-in functions that you can use in any expression — inside locals, variable defaults, resource arguments, and output values. These functions help you transform strings, work with numbers, manipulate collections, handle dates, encode data, and much more.

You do not import or install functions. They are always available, built directly into HCL.

This file covers the most important and commonly used functions, organized by category, with practical Terraform examples for each.

---

## Testing Functions in the Console

Before we dive in — whenever you want to try a function, use the Terraform console:

```bash
terraform console
```

Then type any expression:

```hcl
> upper("hello")
"HELLO"

> length(["a", "b", "c"])
3

> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"

> exit
```

The console is the fastest way to understand what a function does.

---

## String Functions

### `upper(string)` and `lower(string)`

Convert strings to uppercase or lowercase:

```hcl
locals {
  env_upper = upper(var.environment)    # "PRODUCTION"
  env_lower = lower(var.environment)    # "production"
  project   = lower(var.project_name)  # normalize to lowercase
}
```

---

### `trimspace(string)`

Remove leading and trailing whitespace:

```hcl
locals {
  clean_name = trimspace(var.project_name)  # removes spaces at edges
}
```

---

### `trim(string, chars)`, `trimprefix(string, prefix)`, `trimsuffix(string, suffix)`

Remove specific characters or prefixes/suffixes:

```hcl
locals {
  # Remove specific characters from both ends
  no_slashes  = trim("/var/log/", "/")     # "var/log"

  # Remove a known prefix
  short_name  = trimprefix("aws_s3_bucket", "aws_")     # "s3_bucket"

  # Remove a known suffix
  base_name   = trimsuffix("production-env", "-env")     # "production"
}
```

---

### `replace(string, substring, replacement)`

Replace all occurrences of a substring:

```hcl
locals {
  # Replace spaces with hyphens (common for resource names)
  safe_name    = replace("My Web App", " ", "-")    # "My-Web-App"

  # Replace underscores with hyphens
  bucket_name  = replace(lower(var.project), "_", "-")

  # Remove characters entirely
  no_dots      = replace("my.app.name", ".", "")    # "myappname"
}
```

---

### `format(format_string, values...)`

Format a string using printf-style format specifiers:

```hcl
locals {
  # %s = string, %d = integer, %f = float
  message      = format("Deploying %s to %s", var.app_name, var.environment)
  padded_num   = format("%03d", 5)     # "005" (zero-padded to 3 digits)
  percent_val  = format("%.1f%%", 99.9)  # "99.9%"
}
```

---

### `formatlist(format_string, list)`

Apply a format string to each item in a list:

```hcl
locals {
  server_names = ["web", "api", "db"]
  # Add prefix to each item
  prefixed = formatlist("${var.project}-%s", local.server_names)
  # Result: ["myapp-web", "myapp-api", "myapp-db"]

  # Format each port as a string
  ports = [80, 443, 8080]
  port_strings = formatlist("port-%d", local.ports)
  # Result: ["port-80", "port-443", "port-8080"]
}
```

---

### `join(separator, list)`

Join list items into a single string:

```hcl
locals {
  azs = ["us-east-1a", "us-east-1b", "us-east-1c"]
  az_string = join(", ", local.azs)
  # Result: "us-east-1a, us-east-1b, us-east-1c"

  # Join for use in a bash script
  packages   = ["nginx", "curl", "jq", "htop"]
  install_cmd = "apt-get install -y ${join(" ", local.packages)}"
  # Result: "apt-get install -y nginx curl jq htop"
}
```

---

### `split(separator, string)`

Split a string into a list:

```hcl
locals {
  # Split a comma-separated string into a list
  tag_list = split(",", "web,api,database")
  # Result: ["web", "api", "database"]

  # Split an endpoint to get just the hostname
  db_endpoint = "mydb.abc123.us-east-1.rds.amazonaws.com:5432"
  db_host     = split(":", local.db_endpoint)[0]
  # Result: "mydb.abc123.us-east-1.rds.amazonaws.com"

  db_port     = split(":", local.db_endpoint)[1]
  # Result: "5432"
}
```

---

### `startswith(string, prefix)` and `endswith(string, suffix)`

Check if a string starts or ends with a specific value (returns boolean):

```hcl
locals {
  is_https_url    = startswith(var.app_url, "https://")
  is_prod_bucket  = startswith(var.bucket_name, "prod-")
  is_zip_file     = endswith(var.filename, ".zip")
  is_tf_file      = endswith(var.filename, ".tf")
}
```

---

### `substr(string, offset, length)`

Extract a portion of a string:

```hcl
locals {
  # Get first 8 characters of a commit hash (for resource naming)
  short_hash = substr(var.git_commit_hash, 0, 8)
  # "a1b2c3d4" from "a1b2c3d4e5f6789..."
}
```

---

### `regex(pattern, string)` and `regexall(pattern, string)`

Match a string against a regex pattern:

```hcl
locals {
  # Extract the region from an ARN
  arn      = "arn:aws:s3:::my-bucket"
  region   = regex("arn:aws:[^:]*:([^:]*)", var.role_arn)[0]

  # Find all matches (regexall returns a list of all matches)
  all_ports = regexall("[0-9]+", "port-80-to-443")
  # Result: [["80"], ["443"]]
}
```

---

## Collection Functions

### `length(collection)`

Return the number of items in a list, set, or map, or characters in a string:

```hcl
locals {
  subnet_count  = length(var.subnet_ids)     # number of subnets
  tag_count     = length(var.tags)           # number of tags
  name_length   = length(var.project_name)   # characters in name

  # Use in conditionals
  has_subnets   = length(var.subnet_ids) > 0
}
```

---

### `concat(list1, list2, ...)`

Combine multiple lists into one:

```hcl
locals {
  public_subnet_ids  = [for s in aws_subnet.public : s.id]
  private_subnet_ids = [for s in aws_subnet.private : s.id]

  # Combine public and private subnet IDs
  all_subnet_ids = concat(local.public_subnet_ids, local.private_subnet_ids)
}
```

---

### `flatten(list_of_lists)`

Convert a list of lists into a single flat list:

```hcl
locals {
  # Nested lists from multiple resources
  all_ips = flatten([
    aws_instance.web[*].public_ip,
    aws_instance.api[*].public_ip,
    aws_instance.batch[*].public_ip
  ])
  # Result: one flat list of all IP addresses
}
```

---

### `distinct(list)`

Remove duplicate values from a list:

```hcl
locals {
  regions_with_dupes = ["us-east-1", "eu-west-1", "us-east-1", "ap-south-1"]
  unique_regions     = distinct(local.regions_with_dupes)
  # Result: ["us-east-1", "eu-west-1", "ap-south-1"]
}
```

---

### `sort(list)`

Sort a list of strings alphabetically:

```hcl
locals {
  names     = ["charlie", "alice", "bob"]
  sorted    = sort(local.names)
  # Result: ["alice", "bob", "charlie"]

  # Sort and join for consistent output
  sorted_azs = join(", ", sort(var.availability_zones))
}
```

---

### `reverse(list)`

Reverse the order of a list:

```hcl
locals {
  original  = [1, 2, 3, 4, 5]
  reversed  = reverse(local.original)
  # Result: [5, 4, 3, 2, 1]
}
```

---

### `contains(list, value)`

Check if a value exists in a list:

```hcl
locals {
  valid_envs    = ["dev", "staging", "production"]
  is_valid_env  = contains(local.valid_envs, var.environment)

  # Use in conditionals
  allowed_regions = ["us-east-1", "us-west-2", "eu-west-1"]
  is_allowed      = contains(local.allowed_regions, var.region)
}
```

---

### `index(list, value)`

Find the position of a value in a list (zero-based):

```hcl
locals {
  azs     = ["us-east-1a", "us-east-1b", "us-east-1c"]
  pos     = index(local.azs, "us-east-1b")
  # Result: 1
}
```

---

### `element(list, index)`

Get an element from a list by index, with wrapping:

```hcl
locals {
  azs = ["us-east-1a", "us-east-1b"]

  # element() wraps around — useful for distributing resources across AZs
  # If count.index = 3 and there are 2 AZs: element(azs, 3) = azs[3 % 2] = azs[1]
}

resource "aws_subnet" "app" {
  count             = 6  # 6 subnets across 2 AZs
  availability_zone = element(local.azs, count.index)  # distributes evenly
}
```

---

### `slice(list, start, end)`

Extract a portion of a list:

```hcl
data "aws_availability_zones" "available" {}

locals {
  # Take only the first 2 available AZs (even if there are more)
  azs = slice(data.aws_availability_zones.available.names, 0, 2)
}
```

---

### Map Functions: `keys()`, `values()`, `lookup()`, `merge()`

```hcl
locals {
  config = {
    env    = "production"
    region = "us-east-1"
    team   = "platform"
  }

  # Get all keys as a list
  config_keys   = keys(local.config)
  # Result: ["env", "region", "team"]

  # Get all values as a list
  config_values = values(local.config)
  # Result: ["production", "us-east-1", "platform"]

  # Look up a value with a default if not found
  team          = lookup(local.config, "team", "unknown")        # "platform"
  owner         = lookup(local.config, "owner", "unassigned")    # "unassigned" (not in map)

  # Merge multiple maps (later maps override earlier ones for same keys)
  base_tags    = { ManagedBy = "Terraform", Project = var.project }
  extra_tags   = { Environment = var.environment, Owner = var.team }
  override_tags = { Project = "overridden-project" }

  all_tags = merge(local.base_tags, local.extra_tags, local.override_tags)
  # Result: {
  #   ManagedBy = "Terraform"
  #   Project = "overridden-project"  ← overridden by last map
  #   Environment = "production"
  #   Owner = "platform"
  # }
}
```

---

## Numeric Functions

### `min(numbers...)` and `max(numbers...)`

```hcl
locals {
  smallest = min(5, 3, 8, 1, 9)   # 1
  largest  = max(5, 3, 8, 1, 9)   # 9

  # Ensure instance count stays within bounds
  safe_count = min(var.desired_count, var.max_allowed)
}
```

### `abs(number)`

```hcl
locals {
  diff = abs(var.actual_count - var.desired_count)  # always positive
}
```

### `ceil(number)` and `floor(number)`

```hcl
locals {
  # Round up — ceil(2.3) = 3
  servers_needed = ceil(var.total_requests / var.requests_per_server)

  # Round down — floor(2.9) = 2
  full_batches   = floor(var.total_items / var.batch_size)
}
```

### `pow(base, exponent)`

```hcl
locals {
  storage_bytes = pow(2, 30)  # 1 GB in bytes (2^30)
}
```

---

## Type Conversion Functions

```hcl
locals {
  # Convert to string
  port_str   = tostring(8080)          # "8080"
  flag_str   = tostring(true)          # "true"

  # Convert to number
  port_num   = tonumber("8080")        # 8080

  # Convert to bool
  flag_bool  = tobool("true")          # true

  # Convert list to set (removes duplicates, loses order)
  unique_set = toset(["a", "b", "a"]) # {"a", "b"}

  # Convert set to list (ordered)
  back_list  = tolist(toset(["c", "a", "b"]))  # ["a", "b", "c"]

  # Convert between map and object
  as_map     = tomap({key = "value"})
}
```

---

## Encoding Functions

### `jsonencode(value)` and `jsondecode(string)`

Convert values to and from JSON:

```hcl
locals {
  # Encode a Terraform value as a JSON string
  policy_json = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "${aws_s3_bucket.app.arn}/*"
    }]
  })

  # Decode a JSON string back to a Terraform value
  config_object = jsondecode(data.aws_ssm_parameter.config.value)
}

# Use jsonencode directly in resources
resource "aws_iam_role_policy" "app" {
  name   = "app-policy"
  role   = aws_iam_role.app.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:*"]
      Resource = aws_s3_bucket.app.arn
    }]
  })
}
```

### `base64encode(string)` and `base64decode(string)`

```hcl
locals {
  # Encode a startup script as base64 (required by some services)
  encoded_script = base64encode(local.startup_script)

  # Decode base64 data from an API response
  decoded_config = base64decode(data.aws_secretsmanager_secret_version.app.secret_string)
}
```

---

## Filesystem Functions

### `file(path)`

Read the contents of a file:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  # Read startup script from a file
  user_data = file("${path.module}/scripts/startup.sh")
}

resource "aws_iam_role_policy" "app" {
  name   = "app-policy"
  role   = aws_iam_role.app.id

  # Read policy from a JSON file
  policy = file("${path.module}/policies/app-policy.json")
}
```

### `filebase64(path)`

Read a file and encode it as base64:

```hcl
resource "aws_instance" "web" {
  user_data_base64 = filebase64("${path.module}/scripts/startup.sh")
}
```

### `templatefile(path, vars)`

Read a file and substitute variables into it:

```hcl
# scripts/startup.sh.tftpl
# #!/bin/bash
# export ENVIRONMENT="${environment}"
# export DB_HOST="${db_host}"
# export APP_NAME="${app_name}"
# apt-get install -y ${join(" ", packages)}

resource "aws_instance" "web" {
  user_data = templatefile("${path.module}/scripts/startup.sh.tftpl", {
    environment = var.environment
    db_host     = aws_db_instance.app.address
    app_name    = var.project
    packages    = ["nginx", "curl", "jq"]
  })
}
```

---

## Network Functions

### `cidrsubnet(prefix, newbits, netnum)`

Calculate subnet CIDRs from a parent CIDR:

```hcl
locals {
  vpc_cidr = "10.0.0.0/16"

  # cidrsubnet(prefix, newbits, netnum)
  # newbits: how many bits to add to the prefix
  # netnum: which subnet number to generate

  public_subnet_1  = cidrsubnet(local.vpc_cidr, 8, 0)   # "10.0.0.0/24"
  public_subnet_2  = cidrsubnet(local.vpc_cidr, 8, 1)   # "10.0.1.0/24"
  private_subnet_1 = cidrsubnet(local.vpc_cidr, 8, 10)  # "10.0.10.0/24"
  private_subnet_2 = cidrsubnet(local.vpc_cidr, 8, 11)  # "10.0.11.0/24"

  # Build a list of subnets programmatically
  public_cidrs = [for i in range(3) : cidrsubnet(local.vpc_cidr, 8, i)]
  # Result: ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24"]
}
```

### `cidrhost(prefix, hostnum)`

Calculate a specific host IP within a CIDR block:

```hcl
locals {
  vpc_cidr    = "10.0.0.0/16"
  first_host  = cidrhost(local.vpc_cidr, 1)   # "10.0.0.1"
  last_host   = cidrhost(local.vpc_cidr, -1)  # "10.0.255.254"
}
```

---

## The `path` Variable

Terraform provides special path variables for referencing files relative to your module:

```hcl
locals {
  # path.module — directory of the current module
  script_path = "${path.module}/scripts/startup.sh"

  # path.root — directory of the root module (the one you're calling terraform from)
  config_path = "${path.root}/configs/${var.environment}.json"

  # path.cwd — current working directory
}

resource "aws_instance" "web" {
  user_data = file("${path.module}/scripts/startup.sh")
}
```

---

## Quick Reference Summary

| Category | Functions |
|---|---|
| **Strings** | `upper`, `lower`, `trimspace`, `trim`, `replace`, `format`, `formatlist`, `join`, `split`, `startswith`, `endswith`, `substr`, `regex` |
| **Collections** | `length`, `concat`, `flatten`, `distinct`, `sort`, `reverse`, `contains`, `index`, `element`, `slice`, `keys`, `values`, `lookup`, `merge` |
| **Numbers** | `min`, `max`, `abs`, `ceil`, `floor`, `pow` |
| **Type conversion** | `tostring`, `tonumber`, `tobool`, `toset`, `tolist`, `tomap` |
| **Encoding** | `jsonencode`, `jsondecode`, `base64encode`, `base64decode` |
| **Files** | `file`, `filebase64`, `templatefile` |
| **Network** | `cidrsubnet`, `cidrhost` |
| **Validation** | `can`, `try`, `alltrue`, `anytrue` |

---

## Summary

- Terraform's built-in functions are always available — no imports needed
- Use the `terraform console` to test functions interactively
- String functions handle transformation, trimming, splitting, joining, and formatting
- Collection functions handle list and map operations — length, merge, flatten, sort, filter
- Network functions like `cidrsubnet` are essential for building VPC and subnet CIDRs programmatically
- `jsonencode` is the standard way to write IAM policies and other JSON-structured configs in Terraform
- `file()` and `templatefile()` let you keep large scripts and templates in separate files
- `path.module` and `path.root` help you reference files relative to your module directory

This completes Module 3 — HCL Fundamentals. You now have a deep understanding of every major HCL concept — blocks, arguments, attributes, comments, types, variables, outputs, locals, expressions, and functions. These are the building blocks of everything you will write in Terraform going forward.

In Module 4, we dive into Terraform State — understanding the state file deeply, working with remote backends, and mastering state commands.
