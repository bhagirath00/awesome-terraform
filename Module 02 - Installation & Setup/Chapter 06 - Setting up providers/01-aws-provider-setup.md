# AWS Provider Setup

## Introduction

The AWS provider is the plugin that allows Terraform to create and manage AWS resources. Before you can use Terraform with AWS, you need to:

1. Configure the AWS provider in your Terraform code
2. Set up authentication so Terraform can talk to your AWS account
3. Understand how the provider connects Terraform to specific AWS regions and services

This file covers the AWS provider configuration in detail. Authentication is covered in the next file (`provider-authentication.md`).

---

## What the AWS Provider Does

When you write a Terraform configuration that creates an EC2 instance or an S3 bucket, you are not writing AWS API calls yourself. The AWS provider does that for you.

The AWS provider:
- Translates your HCL configuration into AWS API calls
- Handles authentication with AWS on your behalf
- Understands all AWS resource types (EC2, S3, RDS, VPC, IAM, and hundreds more)
- Manages the lifecycle of AWS resources (create, read, update, delete)
- Is maintained by HashiCorp in collaboration with AWS

The AWS provider is one of the most actively developed providers in the Terraform ecosystem. It covers virtually every AWS service.

---

## Declaring the AWS Provider

Every Terraform configuration that uses AWS resources must declare the AWS provider. You do this in the `terraform` block:

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

Let's understand each part:

- `required_providers` — tells Terraform which providers this configuration needs
- `aws` — the local name for this provider (used throughout your configuration as `provider "aws"`)
- `source = "hashicorp/aws"` — the full address on the Terraform Registry (namespace/provider-name)
- `version = "~> 5.0"` — use any version in the 5.x range (5.0, 5.1, 5.31, etc.) but not 6.x

---

## Configuring the AWS Provider

After declaring the provider, you configure it using the `provider` block. The minimum configuration is just the region:

```hcl
provider "aws" {
  region = "us-east-1"
}
```

This is the simplest valid provider configuration. It tells Terraform to create all AWS resources in the `us-east-1` region (US East - North Virginia).

---

## AWS Regions

AWS has data centers in many locations around the world, organized into **regions**. When you configure the AWS provider, you specify which region you want to work in.

Common AWS regions you will use:

| Region Code | Location |
|---|---|
| `us-east-1` | US East (N. Virginia) — most popular, all services available |
| `us-east-2` | US East (Ohio) |
| `us-west-1` | US West (N. California) |
| `us-west-2` | US West (Oregon) |
| `eu-west-1` | Europe (Ireland) |
| `eu-central-1` | Europe (Frankfurt) |
| `ap-south-1` | Asia Pacific (Mumbai) |
| `ap-southeast-1` | Asia Pacific (Singapore) |
| `ap-northeast-1` | Asia Pacific (Tokyo) |

**Tip:** `us-east-1` is the default region for many AWS services and is the most widely supported. If you are not sure which region to use, start with `us-east-1`.

### Using a Variable for Region

Rather than hardcoding the region, it is a better practice to use a variable:

```hcl
variable "aws_region" {
  description = "The AWS region to deploy resources in"
  type        = string
  default     = "us-east-1"
}

provider "aws" {
  region = var.aws_region
}
```

This allows you to easily switch regions without editing the provider block directly.

---

## Provider Configuration Options

The AWS provider has many configuration options beyond just the region. Here are the most important ones:

### `region` (Required)

The AWS region where resources will be created. Can also be set via the `AWS_DEFAULT_REGION` environment variable.

```hcl
provider "aws" {
  region = "us-east-1"
}
```

### `profile`

The named AWS CLI profile to use for authentication. Useful when you have multiple AWS accounts:

```hcl
provider "aws" {
  region  = "us-east-1"
  profile = "my-work-account"
}
```

### `access_key` and `secret_key`

Directly specify AWS credentials in the configuration. **This is NOT recommended** because it means storing secrets in your code files.

```hcl
# DO NOT DO THIS - credentials in code are a security risk
provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

Never commit credentials in your `.tf` files. Use environment variables or IAM roles instead (covered in `provider-authentication.md`).

### `default_tags`

This is a very useful feature. It automatically applies the specified tags to every AWS resource created by this provider:

```hcl
provider "aws" {
  region = "us-east-1"

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Project     = "my-application"
      Environment = "production"
      Team        = "platform-engineering"
    }
  }
}
```

Without `default_tags`, you would have to add these tags to every single resource block. With `default_tags`, they are applied automatically to everything. This is excellent for cost tracking and resource management.

### `assume_role`

Allows Terraform to assume an IAM role before making API calls. This is commonly used for cross-account access:

```hcl
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/TerraformDeployRole"
  }
}
```

We cover this in more detail in Module 11 (Security & Secret Management).

---

## Working with Multiple AWS Regions

Sometimes you need resources in more than one AWS region — for example, a primary application in `us-east-1` and a disaster recovery setup in `us-west-2`.

Terraform handles this using **provider aliases**:

```hcl
# Primary region (no alias — this is the default)
provider "aws" {
  region = "us-east-1"
}

# Secondary region (with alias)
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

# This resource goes to us-east-1 (default provider)
resource "aws_s3_bucket" "primary" {
  bucket = "my-app-primary-bucket"
}

# This resource goes to us-west-2 (aliased provider)
resource "aws_s3_bucket" "backup" {
  provider = aws.west
  bucket   = "my-app-backup-bucket"
}
```

The resource that uses the aliased provider must specify `provider = aws.west`. Resources without a `provider` argument use the default provider (the one without an alias).

---

## Working with Multiple AWS Accounts

Similar to multiple regions, you can manage resources in different AWS accounts using aliases and the `assume_role` argument:

```hcl
# Default provider - production account
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
  }
}

# Aliased provider - staging account
provider "aws" {
  alias  = "staging"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
  }
}

# Resource in production
resource "aws_vpc" "prod_vpc" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "production-vpc" }
}

# Resource in staging
resource "aws_vpc" "staging_vpc" {
  provider   = aws.staging
  cidr_block = "10.1.0.0/16"
  tags = { Name = "staging-vpc" }
}
```

---

## A Complete, Realistic Provider Configuration

Here is what a real-world production AWS provider configuration looks like:

```hcl
terraform {
  required_version = ">= 1.5"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

variable "aws_region" {
  description = "Primary AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
}

variable "project" {
  description = "Project name for tagging"
  type        = string
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project
      ManagedBy   = "Terraform"
      Repository  = "github.com/myorg/infra"
    }
  }
}
```

This configuration:
- Pins the Terraform version to 1.5 or later
- Pins the AWS provider to any 5.x version
- Configures S3 remote state with DynamoDB locking
- Uses variables for region and environment
- Applies consistent tags to all resources automatically

---

## What Happens After Provider Configuration

Once you have your provider block in place, the next step is to run `terraform init`:

```bash
terraform init
```

This downloads the AWS provider plugin from the Terraform Registry and stores it in the `.terraform/providers/` directory. You will see output like:

```
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.31.0...
- Installed hashicorp/aws v5.31.0 (signed by HashiCorp)

Terraform has been successfully initialized!
```

After initialization, you can start writing and applying your AWS resources.

---

## Finding AWS Provider Documentation

The AWS provider has excellent documentation for every resource it supports. You can find it at:

**registry.terraform.io/providers/hashicorp/aws**

For any AWS resource you want to use, the documentation shows:
- **Example Usage** — copy-paste ready examples
- **Argument Reference** — all available configuration options
- **Attributes Reference** — values available after resource creation
- **Import** — how to bring existing AWS resources under Terraform management

For example, to find documentation for EC2 instances:
- Go to the AWS provider page
- Click "Resources" in the left sidebar
- Search for "aws_instance"

---

## Summary

- The AWS provider is the plugin that lets Terraform communicate with AWS
- Declare it in the `terraform` block using `required_providers`
- Configure it using the `provider "aws"` block — the minimum required argument is `region`
- The `default_tags` block is very useful for automatically tagging all resources
- Use **provider aliases** to manage resources in multiple regions or accounts
- Never hardcode credentials in your `.tf` files — use environment variables or IAM roles
- Always run `terraform init` after adding or changing the provider configuration to download the plugin

In the next file, we cover all the ways to authenticate Terraform with AWS — the correct and secure methods used in real-world environments.
