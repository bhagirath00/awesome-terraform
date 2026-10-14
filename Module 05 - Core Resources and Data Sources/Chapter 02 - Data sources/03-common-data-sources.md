# Practical Reference: Common Data Sources in DevOps Workflows

Data sources are the connective tissue of a robust Terraform architecture. They allow independent modules to discover and interact with each other without hardcoded dependencies. This chapter catalogs the most frequently used data sources in the Amazon Web Services (AWS) ecosystem and general-purpose utility data sources that every DevOps engineer should master.

## 1. Networking Lookup Data Sources

Networking is the most common layer where data sources are used, as application teams often deploy into pre-configured VPCs managed by a networking or cloud platform team.

### 1.1 `aws_vpc`
Used to find an existing VPC by its ID or by its tags.
```hcl
data "aws_vpc" "selected" {
  filter {
    name   = "tag:Environment"
    values = ["Production"]
  }
}

# Usage
resource "aws_security_group" "app_sg" {
  vpc_id = data.aws_vpc.selected.id
}
```

### 1.2 `aws_subnet` and `aws_subnets`
While `aws_subnet` fetches a single subnet, `aws_subnets` (plural) is used to find a list of subnet IDs matching a pattern, such as all subnets in a specific Tier.
```hcl
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.selected.id]
  }
  tags = {
    Tier = "Private"
  }
}
```

## 2. Infrastructure Inventory Analysis

### 2.1 `aws_ami`
The most reliable way to ensure your EC2 instances use the latest patched OS images without manually tracking AMI IDs across regions.
- **Filtering by owner:** Prevents using unsafe community AMIs.
- **Using Wildcards:** Allows matching specific patch versions.
```hcl
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

### 2.2 `aws_availability_zones`
Instead of hardcoding "us-east-1a", "us-east-1b", etc., use this data source to fetch the list of active zones in the current region. This makes your code region-agnostic.
```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

# Distribute subnets across all available zones
resource "aws_subnet" "primary" {
  count             = length(data.aws_availability_zones.available.names)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  # ...
}
```

## 3. Security and Identity Lookups

### 3.1 `aws_caller_identity` and `aws_region`
These sources provide metadata about the current Terraform execution environment. They are essential for building ARNs dynamically and ensuring resources are tagged with the correct account info.
```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

# Dynamically construct an ARN
locals {
  s3_arn = "arn:aws:s3:::my-bucket-${data.aws_caller_identity.current.account_id}"
}
```

### 3.2 `aws_iam_policy_document`
This is a "logical" data source. It doesn't make an API call but provides a structured HCL way to write IAM policies, which Terraform then converts to a clean JSON string. This avoids the errors common when using `heredoc` JSON strings.
```hcl
data "aws_iam_policy_document" "s3_read_only" {
  statement {
    actions   = ["s3:GetObject"]
    resources = ["arn:aws:s3:::my-secure-bucket/*"]
  }
}
```

## 4. Secrets and External Configuration

### 4.1 `aws_secretsmanager_secret` and `version`
To avoid storing secrets in plain text variables, you can store them in AWS Secrets Manager and fetch them at run-time.
```hcl
data "aws_secretsmanager_secret" "db_password" {
  name = "prod/db/password"
}

data "aws_secretsmanager_secret_version" "current" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}

# The actual password value
# password = data.aws_secretsmanager_secret_version.current.secret_string
```

### 4.2 `terraform_remote_state` (Critical)
This data source allows one Terraform project to read the **output values** from the state file of another Terraform project. This is the primary method for "Layered Infrastructure" where the Network team exposes VPC IDs and the App team consumes them.
```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "company-tf-state"
    key    = "network/terraform.tfstate"
    region = "us-east-1"
  }
}

# Accessing an output from the network layer
# vpc_id = data.terraform_remote_state.network.outputs.vpc_id
```

## 5. The "Swiss Army Knife": `external` and `http`

### 5.1 `http`
Fetches a web page or API result via a GET request. Commonly used to find your current public IP address to whitelist it in a security group.
```hcl
data "http" "my_ip" {
  url = "https://checkip.amazonaws.com"
}
```

### 5.2 `external`
Allows you to run an arbitrary local script (Python, Bash, PowerShell) that returns a JSON object. This is a powerful fallback when no native Terraform data source supports the specific lookup you need.
```hcl
data "external" "git_info" {
  program = ["sh", "get_git_branch.sh"]
}
```

## 6. Summary and Best Practices

1. **Specific Filters:** Always use specific filters (like `owners` for AMIs) to prevent accidentally fetching resources you don't intend to manage.
2. **Handle Plurals Carefully:** When using `aws_subnets` or similar plural sources, ensure your code can handle an empty list or an unexpectedly long list.
3. **Prefer Remote State for Inner-Org Dependencies:** For dependencies between your own components, use `terraform_remote_state`. Use cloud provider data sources (like `aws_vpc`) for dependencies on infrastructure you don't control.
4. **Avoid Sensitive Data Exposure:** Remember that data source results are stored in Terraform state. Use `sensitive = true` on outputs that reference secrets fetched via data sources.

Data sources transform Terraform from a static template engine into an intelligent infrastructure orchestrator capable of reacting to and integrating with its environment in real-time.
