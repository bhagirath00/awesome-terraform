# Blocks, Arguments, and Attributes

## Introduction

HCL (HashiCorp Configuration Language) is built on three fundamental building blocks: **blocks**, **arguments**, and **attributes**. Understanding these three concepts clearly is the foundation of everything else you will write in Terraform.

Many beginners mix up arguments and attributes, or confuse blocks with resources. This file explains each one clearly with real examples so there is no confusion going forward.

---

## What is a Block?

A **block** is a container that groups related configuration together. Every piece of Terraform configuration lives inside a block of some kind.

A block has this structure:

```
block_type "label_one" "label_two" {
  # content goes here
}
```

- **block_type** — identifies what kind of block this is (e.g., `resource`, `variable`, `provider`)
- **labels** — strings in quotes that further identify the block (not all block types use labels)
- **body** — everything inside the curly braces `{}`

---

### The Most Common Block Types

**`terraform` block** — configures Terraform itself (version, providers, backend)

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

This block has no labels. It appears once per configuration.

---

**`provider` block** — configures a provider

```hcl
provider "aws" {
  region = "us-east-1"
}
```

This block has one label: `"aws"` — the name of the provider.

---

**`resource` block** — defines a piece of infrastructure to create

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

This block has two labels:
- `"aws_instance"` — the resource type (tells Terraform which AWS service and resource)
- `"web_server"` — the local name (used to reference this resource in other parts of your code)

---

**`variable` block** — declares an input variable

```hcl
variable "environment" {
  description = "The deployment environment"
  type        = string
  default     = "dev"
}
```

This block has one label: `"environment"` — the variable name.

---

**`output` block** — declares a value to display after apply

```hcl
output "server_ip" {
  description = "The public IP of the web server"
  value       = aws_instance.web_server.public_ip
}
```

This block has one label: `"server_ip"` — the output name.

---

**`locals` block** — defines local computed values

```hcl
locals {
  environment = "production"
  app_name    = "myapp"
  full_name   = "${local.app_name}-${local.environment}"
}
```

This block has no labels. You can have multiple `locals` blocks in a configuration — they are all merged together.

---

**`data` block** — reads data from an existing resource (does not create anything)

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}
```

This block has two labels: `"aws_ami"` (data source type) and `"ubuntu"` (local name).

---

**`module` block** — calls a reusable module

```hcl
module "vpc" {
  source = "./modules/vpc"

  cidr_block  = "10.0.0.0/16"
  environment = var.environment
}
```

This block has one label: `"vpc"` — the module name.

---

## Nested Blocks

Blocks can contain other blocks inside them. These are called **nested blocks**.

```hcl
resource "aws_security_group" "web" {
  name        = "web-security-group"
  description = "Allow web traffic"

  ingress {                          # ← nested block
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {                          # ← another nested block of same type
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {                           # ← nested block of different type
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Here:
- The outer block is `resource "aws_security_group" "web"`
- Inside it are three nested blocks: two `ingress` blocks and one `egress` block
- Nested blocks can be repeated (like `ingress` here) when the resource supports it

---

## What is an Argument?

An **argument** is a key-value pair inside a block that sets a specific configuration value. Arguments are what you write to configure a block.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"   # argument
  instance_type = "t2.micro"                # argument
  key_name      = "my-key-pair"             # argument
}
```

The format is always: `argument_name = value`

### Rules for Arguments

- Argument names are defined by the provider — you cannot make up argument names
- Each argument name can appear only once within a block (unless the provider explicitly allows it)
- Some arguments are **required** — Terraform will error if you don't provide them
- Some arguments are **optional** — they have default values if you don't provide them
- The value type must match what the argument expects (string, number, bool, list, map, etc.)

---

### Required vs Optional Arguments

**Required argument** — must be provided, no default exists

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # required
  instance_type = "t2.micro"               # required
}
```

If you omit a required argument, `terraform validate` or `terraform plan` will fail with an error like:
```
Error: Missing required argument
The argument "ami" is required, but no definition was found.
```

**Optional argument** — has a default value if you don't provide it

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  # These are optional — they have defaults if you leave them out
  monitoring              = false   # default: false
  disable_api_termination = false   # default: false
  ebs_optimized           = false   # default: false
}
```

---

## What is an Attribute?

An **attribute** is a value that belongs to a resource and is available after the resource is created. Attributes are read from a resource — you do not set them yourself.

This is the key difference:
- **Arguments** — you set them (input to the resource)
- **Attributes** — you read them (output from the resource)

When you create an EC2 instance, AWS assigns it:
- An instance ID (`id`)
- A public IP address (`public_ip`)
- A private IP address (`private_ip`)
- An ARN (`arn`)

These are attributes. They exist after the resource is created and you can reference them in other resources.

---

### Referencing Attributes

The syntax for referencing a resource's attribute is:

```
resource_type.resource_name.attribute_name
```

For example:
```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

resource "aws_eip" "web_ip" {
  # Reference the instance attribute
  instance = aws_instance.web_server.id
}

output "server_public_ip" {
  # Reference another attribute
  value = aws_instance.web_server.public_ip
}
```

Here:
- `aws_instance.web_server.id` — the ID of the EC2 instance (assigned by AWS after creation)
- `aws_instance.web_server.public_ip` — the public IP address (assigned by AWS after creation)

---

### `(known after apply)` in Plans

When you run `terraform plan` before a resource exists, attributes that will be assigned by AWS appear as `(known after apply)`:

```
+ resource "aws_instance" "web_server" {
    + ami           = "ami-0c55b159cbfafe1f0"
    + id            = (known after apply)       ← attribute
    + public_ip     = (known after apply)       ← attribute
    + private_ip    = (known after apply)       ← attribute
    + arn           = (known after apply)       ← attribute
  }
```

After `terraform apply`, Terraform knows these values and stores them in the state file. Any subsequent plan shows the real values.

---

## Arguments vs Attributes — Side by Side

```hcl
resource "aws_s3_bucket" "my_bucket" {
  # These are ARGUMENTS — you set them
  bucket        = "my-unique-bucket-name"   # argument (required)
  force_destroy = true                       # argument (optional)

  tags = {                                   # argument (optional)
    Name = "my-bucket"
  }
}

# After the bucket is created, these ATTRIBUTES are available:
# aws_s3_bucket.my_bucket.id          → "my-unique-bucket-name"
# aws_s3_bucket.my_bucket.arn         → "arn:aws:s3:::my-unique-bucket-name"
# aws_s3_bucket.my_bucket.region      → "us-east-1"
# aws_s3_bucket.my_bucket.bucket_domain_name → "my-unique-bucket-name.s3.amazonaws.com"
```

---

## Block Body Rules

### One Argument Per Line

Each argument goes on its own line:

```hcl
# Correct
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}

# Wrong — multiple arguments on one line (valid HCL but bad practice)
resource "aws_instance" "web" { ami = "ami-0c55b159cbfafe1f0" instance_type = "t2.micro" }
```

### Alignment of Equals Signs

Terraform's formatter (`terraform fmt`) aligns the `=` signs in argument groups:

```hcl
# After terraform fmt
resource "aws_instance" "web" {
  ami             = "ami-0c55b159cbfafe1f0"
  instance_type   = "t2.micro"
  key_name        = "my-key"
  security_groups = ["web-sg"]
}
```

The `=` signs line up, making the code easier to scan.

### Blank Lines Between Logical Groups

Use blank lines to separate logical groups of arguments:

```hcl
resource "aws_instance" "web" {
  # Core configuration
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  # Networking
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.web.id]
  associate_public_ip_address = true

  # Storage
  root_block_device {
    volume_size = 20
    volume_type = "gp3"
  }

  # Tags
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

---

## A Complete Example Showing All Concepts

```hcl
# terraform block — no labels
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# provider block — one label
provider "aws" {
  region = "us-east-1"    # argument
}

# variable block — one label
variable "instance_type" {
  description = "EC2 instance type"   # argument
  type        = string                # argument
  default     = "t2.micro"           # argument
}

# resource block — two labels
resource "aws_security_group" "web_sg" {
  name        = "web-sg"              # argument
  description = "Web server SG"       # argument

  ingress {                           # nested block
    from_port   = 80                  # argument in nested block
    to_port     = 80                  # argument in nested block
    protocol    = "tcp"               # argument in nested block
    cidr_blocks = ["0.0.0.0/0"]     # argument in nested block
  }
}

# resource block using attributes from the security group above
resource "aws_instance" "web" {
  ami                    = "ami-0c55b159cbfafe1f0"  # argument (you set this)
  instance_type          = var.instance_type          # argument using a variable
  vpc_security_group_ids = [aws_security_group.web_sg.id]  # argument using an attribute

  tags = {
    Name = "web-server"
  }
}

# output block — reads attributes from the instance
output "web_server_ip" {
  value = aws_instance.web.public_ip   # attribute (set by AWS after creation)
}

output "web_server_id" {
  value = aws_instance.web.id          # attribute (set by AWS after creation)
}
```

---

## Summary

- A **block** is a container that groups related configuration — identified by block type and optional labels
- Common block types: `terraform`, `provider`, `resource`, `variable`, `output`, `locals`, `data`, `module`
- Blocks can contain **nested blocks** for hierarchical configuration
- An **argument** is a key-value pair you write inside a block to configure it — you set arguments
- Arguments can be **required** (must provide) or **optional** (have defaults)
- An **attribute** is a value that belongs to a resource and is available after it is created — you read attributes
- The syntax for reading an attribute is: `resource_type.resource_name.attribute_name`
- Attributes that are not yet known appear as `(known after apply)` in plan output

In the next file, we cover comments in HCL — how to write them and when to use them effectively.
