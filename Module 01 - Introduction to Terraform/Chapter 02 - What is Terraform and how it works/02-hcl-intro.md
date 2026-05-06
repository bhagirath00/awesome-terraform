# Introduction to HCL — HashiCorp Configuration Language

## What is HCL?

**HCL** stands for **HashiCorp Configuration Language**. It is the language you use to write Terraform configuration files.

HCL was designed with one main goal — to be easy for humans to read and write, while also being easy for machines to parse. It sits in the middle ground between a full programming language (like Python) and a data format (like JSON or YAML).

You do not need to be a programmer to write HCL. If you understand basic concepts like key-value pairs and blocks, you can start writing Terraform code right away.

---

## Why Not JSON or YAML?

Terraform actually supports JSON as an alternative to HCL — but almost nobody uses it. HCL is much easier to read and write.

Here is the same Terraform resource written in both formats:

**HCL version:**
```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

**JSON version:**
```json
{
  "resource": {
    "aws_instance": {
      "web_server": {
        "ami": "ami-0c55b159cbfafe1f0",
        "instance_type": "t2.micro",
        "tags": {
          "Name": "web-server",
          "Environment": "production"
        }
      }
    }
  }
}
```

Both do the same thing. But HCL is clearly easier to write and understand — especially when configurations get large. This is why HCL is the standard for Terraform.

---

## Basic HCL Syntax

### Arguments (Key-Value Pairs)

The most basic building block of HCL is an **argument**. An argument assigns a value to a name:

```hcl
name  = "value"
count = 3
enabled = true
```

The format is always: `name = value`

- The name (left side) is called the **argument name** or **key**
- The `=` sign separates the name from the value
- The value (right side) can be a string, number, boolean, list, or map

---

### Strings

Strings in HCL are written inside double quotes:

```hcl
name   = "my-server"
region = "us-east-1"
ami    = "ami-0c55b159cbfafe1f0"
```

HCL also supports **string interpolation** — embedding expressions inside strings using `${}`:

```hcl
environment = "production"
name        = "web-server-${environment}"
# Result: "web-server-production"
```

You can also write multi-line strings using a heredoc syntax:

```hcl
user_data = <<-EOF
  #!/bin/bash
  echo "Hello from Terraform"
  apt-get update
  apt-get install -y nginx
EOF
```

---

### Numbers

Numbers are written without quotes:

```hcl
port     = 80
count    = 3
max_size = 10
timeout  = 30
```

---

### Booleans

Booleans represent true or false values:

```hcl
enable_monitoring = true
multi_az          = false
deletion_protection = true
```

---

### Lists (Arrays)

A list is an ordered collection of values, written inside square brackets:

```hcl
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

ingress_ports = [80, 443, 8080]

allowed_ips = [
  "10.0.0.0/8",
  "172.16.0.0/12",
  "192.168.0.0/16"
]
```

---

### Maps (Objects)

A map is a collection of key-value pairs, written inside curly braces:

```hcl
tags = {
  Name        = "web-server"
  Environment = "production"
  Team        = "platform"
  CostCenter  = "engineering"
}
```

Maps can be nested inside each other:

```hcl
settings = {
  database = {
    host = "db.example.com"
    port = 5432
    name = "myapp"
  }
  cache = {
    host = "cache.example.com"
    port = 6379
  }
}
```

---

## Blocks

A **block** is the main structural unit in HCL. Blocks are used to group related arguments together.

A block has this structure:

```
block_type "label_1" "label_2" {
  argument1 = "value1"
  argument2 = "value2"
}
```

Let's break this down:

- **block_type** — the type of block (e.g., `resource`, `variable`, `output`, `provider`)
- **labels** — identify the specific block (not all block types have labels)
- **body** — the content inside `{}`, containing arguments and sometimes nested blocks

---

### The `resource` Block

The most common block in Terraform. It defines a piece of infrastructure you want to create:

```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name-2024"

  tags = {
    Name = "My bucket"
  }
}
```

Here:
- `resource` — the block type
- `"aws_s3_bucket"` — the resource type (which AWS service and which kind of resource)
- `"my_bucket"` — the local name (used to reference this resource elsewhere in your code)
- Everything inside `{}` — the arguments that configure the resource

---

### The `provider` Block

Configures which cloud or service Terraform will connect to:

```hcl
provider "aws" {
  region  = "us-east-1"
  profile = "default"
}
```

---

### The `variable` Block

Defines an input variable so you can pass values into your configuration:

```hcl
variable "instance_type" {
  description = "The type of EC2 instance to create"
  type        = string
  default     = "t2.micro"
}
```

---

### The `output` Block

Defines a value that Terraform will display after applying, and that can be used by other configurations:

```hcl
output "server_ip" {
  description = "The public IP address of the web server"
  value       = aws_instance.web_server.public_ip
}
```

---

### The `terraform` Block

Configures Terraform itself — like which version is required and which providers to use:

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

---

### The `locals` Block

Defines local values — computed values that you use within your configuration to avoid repeating yourself:

```hcl
locals {
  environment = "production"
  app_name    = "myapp"

  common_tags = {
    Environment = local.environment
    Application = local.app_name
    ManagedBy   = "Terraform"
  }
}
```

---

## Comments in HCL

HCL supports three types of comments:

```hcl
# This is a single-line comment (most common)

// This is also a single-line comment

/*
  This is a
  multi-line comment
*/
```

Use comments to explain why a particular value is set, or to temporarily disable a configuration:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"  # Changed from t3.micro to save cost

  # Temporarily disabled while debugging
  # user_data = file("startup.sh")
}
```

---

## References and Expressions

One of the most powerful features of HCL is the ability to reference values from other resources and variables.

### Referencing a Variable

```hcl
variable "region" {
  default = "us-east-1"
}

provider "aws" {
  region = var.region  # Reference the variable
}
```

### Referencing Another Resource

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # Reference the VPC's ID
  cidr_block = "10.0.1.0/24"
}
```

The format for referencing a resource attribute is:
`resource_type.resource_name.attribute`

So `aws_vpc.main.id` means: from the resource of type `aws_vpc` named `main`, get the `id` attribute.

---

### Referencing a Local Value

```hcl
locals {
  app_name = "myapp"
}

resource "aws_s3_bucket" "data" {
  bucket = "${local.app_name}-data-bucket"  # "myapp-data-bucket"
}
```

---

## A Complete Example

Here is a complete, simple Terraform configuration that uses all the concepts we covered:

```hcl
# Configure Terraform and the AWS provider
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# AWS provider configuration
provider "aws" {
  region = var.aws_region
}

# Input variables
variable "aws_region" {
  description = "The AWS region to deploy resources in"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "The deployment environment"
  type        = string
  default     = "development"
}

# Local values
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}

# S3 bucket resource
resource "aws_s3_bucket" "app_storage" {
  bucket = "myapp-storage-${var.environment}"
  tags   = local.common_tags
}

# Output the bucket name
output "bucket_name" {
  description = "The name of the S3 bucket"
  value       = aws_s3_bucket.app_storage.bucket
}
```

This configuration:
- Sets up the AWS provider for a specific region
- Accepts two input variables (region and environment)
- Defines common tags using locals
- Creates an S3 bucket with a name that includes the environment
- Outputs the bucket name after creation

---

## HCL File Naming and Organization

Terraform reads all `.tf` files in a directory as one combined configuration. How you organize your files is up to you, but there are common conventions:

| File | Purpose |
|---|---|
| `main.tf` | Main resources |
| `variables.tf` | All variable declarations |
| `outputs.tf` | All output declarations |
| `providers.tf` | Provider and Terraform block configuration |
| `locals.tf` | Local value definitions |
| `terraform.tfvars` | Actual values for variables |

You can also split resources into multiple files by category:
- `network.tf` — VPC, subnets, routing
- `compute.tf` — EC2 instances, auto scaling
- `database.tf` — RDS, ElastiCache
- `storage.tf` — S3 buckets

---

## Summary

- HCL is the language used to write Terraform configurations — designed to be readable and simple
- The basic building blocks are **arguments** (key-value pairs) and **blocks** (grouped configurations)
- Common data types include strings, numbers, booleans, lists, and maps
- Common block types are `resource`, `provider`, `variable`, `output`, `locals`, and `terraform`
- You can reference values from other resources and variables using the `type.name.attribute` syntax
- Terraform reads all `.tf` files in a directory together
- Comments use `#`, `//`, or `/* */` syntax

In the next chapter, we will compare Terraform with other popular IaC tools to understand where it fits in the ecosystem.
