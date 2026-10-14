# Meta-Arguments — Changing How Resources Behave

## What Are Meta-Arguments?

Every resource block has two kinds of settings inside it:

1. **Regular arguments** — settings specific to that resource type. For example, `ami` and `instance_type` are arguments that only make sense for an `aws_instance`. They are defined by the AWS provider.

2. **Meta-arguments** — special settings that work on **any** resource, regardless of type. They are not defined by the provider — they are built into Terraform Core itself. They change how Terraform **handles** the resource, not what the resource looks like.

Think of it this way: regular arguments are like the specs of a car (engine size, color, fuel type). Meta-arguments are like the rules about how you drive it (drive it solo, drive it in a convoy, wait for another car to go first).

The four meta-arguments you need to master are:

| Meta-Argument | What it does |
|---------------|-------------|
| `depends_on` | Explicitly tells Terraform to wait for another resource |
| `count` | Creates multiple copies of the same resource |
| `for_each` | Creates one resource per item in a map or set |
| `provider` | Tells Terraform which provider configuration to use |

Let's go deep on each one.

---

## 1. depends_on — Explicit Ordering

Terraform is smart about figuring out the order to create resources. When you write `vpc_id = aws_vpc.main.id`, Terraform sees the reference and automatically knows: "create the VPC first, then the subnet."

This is called an **implicit dependency** — Terraform detects it automatically from your references.

But sometimes you have a dependency that Terraform **cannot see** in the code. The most common case is IAM permissions. Imagine you create an EC2 instance that will call an AWS API. You also create an IAM role with the right permissions. Terraform does not know the instance needs the role to be ready before it boots up — because there is no direct attribute reference between them.

Without `depends_on`, Terraform might create the EC2 instance first, and the startup script fails because the IAM role is not attached yet.

With `depends_on`, you tell Terraform explicitly: "wait for this before creating that."

```hcl
resource "aws_iam_role" "ec2_role" {
  name = "ec2-s3-access-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "s3_access" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}

resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.small"

  # Terraform cannot see this dependency from the code alone
  # We must declare it explicitly
  depends_on = [
    aws_iam_role_policy_attachment.s3_access
  ]

  tags = {
    Name = "app-server"
  }
}
```

Now Terraform will wait until the IAM policy attachment is complete before creating the EC2 instance.

### When to use depends_on

Use it when:
- A resource depends on a side effect (like a permission being active) rather than an attribute
- A Lambda function needs an SQS queue to be fully configured before it subscribes
- A startup script on an instance needs a secret to exist in Secrets Manager first

Do not overuse it. If you find yourself writing `depends_on` everywhere, it usually means you can improve the code by referencing attributes directly instead.

---

## 2. count — Create Multiple Copies

The `count` meta-argument lets you create multiple instances of the same resource using one block. You give it a number, and Terraform creates that many copies.

### Basic Usage

```hcl
resource "aws_instance" "web_server" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server-${count.index}"
  }
}
```

This creates three EC2 instances: `web-server-0`, `web-server-1`, and `web-server-2`.

Inside a block that uses `count`, Terraform gives you a special object called `count.index`. It starts at `0` and goes up to `count - 1`. You use it to give each resource a unique name or configuration.

### Referencing count Resources

Because `count` creates multiple instances, the resource becomes a **list** in state and in your code:

```hcl
# Get all instance IDs as a list
output "all_server_ids" {
  value = aws_instance.web_server[*].id
}

# Get just the first server's IP
output "first_server_ip" {
  value = aws_instance.web_server[0].public_ip
}
```

### Conditional Creation with count

One of the most useful patterns with `count` is using it as an on/off switch:

```hcl
variable "create_bastion" {
  type    = bool
  default = false
}

resource "aws_instance" "bastion" {
  count         = var.create_bastion ? 1 : 0
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.small"

  tags = {
    Name = "bastion-host"
  }
}
```

If `create_bastion` is `false`, `count` becomes `0` and Terraform creates nothing. If it is `true`, `count` becomes `1` and one bastion host is created. This is the standard way to make infrastructure optional per environment.

### The Problem with count and Lists

`count` works great for identical resources or simple toggles. It has one weakness: when you use it with a list and remove an item from the middle of the list, Terraform reassigns the indices. This causes Terraform to recreate resources it should not touch.

This is why `for_each` exists — covered next.

---

## 3. for_each — Create One Per Map Item

The `for_each` meta-argument is the more powerful way to create multiple resources. Instead of a number, you give it a **map** or a **set of strings**. Terraform creates one resource instance for each item in the collection.

### Using for_each with a Map

```hcl
resource "aws_iam_user" "devops_users" {
  for_each = {
    alice = "alice@company.com"
    bob   = "bob@company.com"
    carol = "carol@company.com"
  }

  name = each.key

  tags = {
    Email = each.value
    Team  = "DevOps"
  }
}
```

Inside a `for_each` block, Terraform gives you the `each` object:
- `each.key` — the map key (in this case: `alice`, `bob`, `carol`)
- `each.value` — the map value (in this case: the email address)

Terraform creates three IAM users. In state, they are tracked as:
- `aws_iam_user.devops_users["alice"]`
- `aws_iam_user.devops_users["bob"]`
- `aws_iam_user.devops_users["carol"]`

### Using for_each with a Variable

In real projects, you define the map in a variable file so it is easy to change:

```hcl
variable "environments" {
  type = map(string)
  default = {
    dev  = "10.0.0.0/16"
    staging = "10.1.0.0/16"
    prod = "10.2.0.0/16"
  }
}

resource "aws_vpc" "env_vpcs" {
  for_each   = var.environments
  cidr_block = each.value

  tags = {
    Name = "${each.key}-vpc"
  }
}
```

This creates three VPCs — one per environment — each with a different CIDR block and name.

### Using for_each with a Set

If you have a list of values (not a map), convert it to a set using `toset()`:

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "public" {
  for_each = toset(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  cidr_block        = "10.0.${index(var.availability_zones, each.key)}.0/24"

  tags = {
    Name = "public-subnet-${each.key}"
  }
}
```

### Why for_each is Safer Than count

The key difference is in how resources are tracked in state:

| | count | for_each |
|--|-------|---------|
| State key | Numeric index (`[0]`, `[1]`) | Map key (`["alice"]`) |
| Remove middle item | Causes cascade of recreations | Only removes that one item |
| Add new item | May shift other items | No impact on existing items |
| Best for | Identical resources, toggles | Distinct named resources |

If you have a list of servers where each one has a unique name, use `for_each`. If you need 5 identical nodes in an Auto Scaling group, use `count`.

---

## 4. provider — Using Multiple Provider Configurations

By default, every resource uses the default provider configuration. But sometimes you need to create resources in multiple AWS regions, or across multiple AWS accounts, in the same Terraform project.

The `provider` meta-argument lets you tell a specific resource to use a specific provider configuration instead of the default.

### Setting Up Aliased Providers

First, define the extra provider configuration with an alias:

```hcl
# Default AWS provider — all resources use this unless told otherwise
provider "aws" {
  region = "us-east-1"
}

# Second AWS provider for disaster recovery region
provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}
```

### Using the Aliased Provider in a Resource

```hcl
# This S3 bucket goes in us-east-1 (default)
resource "aws_s3_bucket" "primary_backup" {
  bucket = "myapp-backup-primary"
}

# This S3 bucket goes in us-west-2 (alias)
resource "aws_s3_bucket" "dr_backup" {
  provider = aws.us_west
  bucket   = "myapp-backup-dr"
}
```

The `provider` meta-argument takes the format `<provider_name>.<alias>`. Terraform routes all API calls for that resource through the aliased configuration.

### Multi-Account Setup

The same pattern works for multiple AWS accounts:

```hcl
provider "aws" {
  alias  = "dev_account"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
  }
}

provider "aws" {
  alias  = "prod_account"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
  }
}

resource "aws_s3_bucket" "dev_logs" {
  provider = aws.dev_account
  bucket   = "dev-application-logs"
}

resource "aws_s3_bucket" "prod_logs" {
  provider = aws.prod_account
  bucket   = "prod-application-logs"
}
```

This is a common pattern in enterprise AWS setups where different environments live in separate accounts for security and billing isolation.

---

## Rules and Restrictions

There are a few important rules to keep in mind:

- You **cannot** use both `count` and `for_each` in the same resource block — pick one
- `depends_on` takes a **list** of resource references, not strings
- When using `for_each`, the keys must be known at plan time — they cannot depend on values that are only determined during apply
- The `provider` meta-argument must reference a provider that is already defined in your configuration

---

## Summary

- Meta-arguments are special Terraform Core settings that work on any resource, unlike regular provider-specific arguments
- `depends_on` creates explicit ordering when Terraform cannot detect the dependency automatically — use it for IAM permission timing and similar scenarios
- `count` creates multiple identical copies of a resource using a number — great for clusters and optional resources
- `for_each` creates one resource per item in a map or set — safer than `count` because resources are tracked by key, not index
- `provider` routes a specific resource through an aliased provider — essential for multi-region and multi-account setups
- Never use `count` and `for_each` together in the same block

In the next file, we look at the **lifecycle block** — a set of rules you can add to any resource to control exactly how Terraform creates, updates, and destroys it in production environments.
