# Output Values

## Introduction

**Output values** are a way to expose specific information from your Terraform configuration after `terraform apply` runs. They serve as a structured way to surface important information — like IP addresses, resource IDs, ARNs, and DNS names — that you need to use after your infrastructure is created.

Think of outputs like the return values of a function. Your Terraform configuration does a lot of work (creates infrastructure), and outputs are how it tells you what it created and what you need to know about it.

---

## Declaring an Output

Outputs are declared using the `output` block. Each output block has one label — the output name — and a required `value` argument:

```hcl
output "web_server_ip" {
  description = "The public IP address of the web server"
  value       = aws_instance.web_server.public_ip
}
```

### Output Block Arguments

**`value`** (required) — The value to expose. This can be any expression: a resource attribute, a local value, a variable, or a complex expression.

**`description`** (recommended) — A human-readable explanation of what this output represents. Always write one.

**`sensitive`** (optional) — When `true`, hides the value in terminal output (same as sensitive variables).

**`depends_on`** (rarely needed) — Explicitly declare dependencies for this output.

---

## When Outputs Are Shown

Outputs are displayed in two situations:

### 1. After `terraform apply`

```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

web_server_ip = "54.23.45.67"
db_endpoint   = "myapp-db.abc123def456.us-east-1.rds.amazonaws.com"
bucket_name   = "myapp-production-data-a1b2c3d4"
```

### 2. When Running `terraform output`

```bash
# Show all outputs
terraform output

# Show a specific output
terraform output web_server_ip

# Get raw value (no quotes, useful in scripts)
terraform output -raw web_server_ip

# Get all outputs as JSON (useful for automation)
terraform output -json
```

---

## Output Value Expressions

The `value` argument accepts any valid HCL expression.

### Simple Attribute Reference

```hcl
# Single resource attribute
output "instance_id" {
  value = aws_instance.web.id
}

output "instance_arn" {
  value = aws_instance.web.arn
}

output "public_ip" {
  value = aws_instance.web.public_ip
}

output "private_ip" {
  value = aws_instance.web.private_ip
}
```

### String with Interpolation

```hcl
output "ssh_command" {
  description = "Command to SSH into the web server"
  value       = "ssh -i ~/.ssh/my-key.pem ec2-user@${aws_instance.web.public_ip}"
}

output "app_url" {
  description = "URL of the application"
  value       = "https://${aws_lb.main.dns_name}"
}
```

### List of Values

```hcl
# Output all public IPs when using count
output "web_server_ips" {
  description = "Public IP addresses of all web servers"
  value       = aws_instance.web[*].public_ip
}

# Output IDs from a for_each resource
output "subnet_ids" {
  description = "IDs of all created subnets"
  value       = values(aws_subnet.all)[*].id
}
```

### Map of Values

```hcl
output "subnet_info" {
  description = "Map of subnet name to subnet ID"
  value = {
    for name, subnet in aws_subnet.all : name => subnet.id
  }
}

output "instance_details" {
  description = "Key details about each instance"
  value = {
    for idx, instance in aws_instance.web : idx => {
      id         = instance.id
      public_ip  = instance.public_ip
      private_ip = instance.private_ip
    }
  }
}
```

### Computed Value

```hcl
output "connection_string" {
  description = "PostgreSQL connection string"
  value = "postgresql://${var.db_username}@${aws_db_instance.app.endpoint}/${var.db_name}"
}

output "total_instance_count" {
  description = "Total number of EC2 instances created"
  value       = length(aws_instance.web)
}
```

---

## Sensitive Outputs

Mark outputs as sensitive when they contain secrets or security-sensitive information:

```hcl
output "db_password" {
  description = "Database master password"
  value       = random_password.db.result
  sensitive   = true
}

output "connection_string_with_password" {
  description = "Full database connection string including password"
  value       = "postgresql://${var.db_user}:${random_password.db.result}@${aws_db_instance.app.endpoint}/${var.db_name}"
  sensitive   = true
}
```

When you run `terraform output`, sensitive values appear as:
```
db_password = <sensitive>
```

To retrieve a sensitive output value in a script:
```bash
# This works even for sensitive outputs
terraform output -raw db_password
```

**Note:** Sensitive outputs are still stored in plain text in the state file. Always secure your state file.

---

## Organizing Outputs in `outputs.tf`

All output declarations should live in a dedicated `outputs.tf` file:

```hcl
# outputs.tf

# ==============================================================================
# Networking Outputs
# ==============================================================================

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = [for subnet in aws_subnet.public : subnet.id]
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = [for subnet in aws_subnet.private : subnet.id]
}

# ==============================================================================
# Compute Outputs
# ==============================================================================

output "web_server_ids" {
  description = "IDs of the web server EC2 instances"
  value       = aws_instance.web[*].id
}

output "load_balancer_dns" {
  description = "DNS name of the application load balancer"
  value       = aws_lb.main.dns_name
}

output "load_balancer_arn" {
  description = "ARN of the application load balancer"
  value       = aws_lb.main.arn
}

# ==============================================================================
# Database Outputs
# ==============================================================================

output "db_endpoint" {
  description = "Endpoint of the RDS database"
  value       = aws_db_instance.app.endpoint
}

output "db_name" {
  description = "Name of the database"
  value       = aws_db_instance.app.db_name
}

# ==============================================================================
# Storage Outputs
# ==============================================================================

output "app_bucket_name" {
  description = "Name of the application S3 bucket"
  value       = aws_s3_bucket.app.bucket
}

output "app_bucket_arn" {
  description = "ARN of the application S3 bucket"
  value       = aws_s3_bucket.app.arn
}
```

---

## Using Outputs in Scripts and Automation

Outputs are especially useful when your infrastructure needs to be used by other tools or scripts.

### Getting a Single Value in Bash

```bash
# Get the load balancer DNS name
LB_DNS=$(terraform output -raw load_balancer_dns)
echo "Application URL: https://$LB_DNS"

# Use it to test the application
curl -f "https://$LB_DNS/health" || echo "Health check failed"
```

### Getting All Outputs as JSON

```bash
# Get all outputs in JSON format
OUTPUTS=$(terraform output -json)

# Parse with jq
DB_ENDPOINT=$(echo $OUTPUTS | jq -r '.db_endpoint.value')
VPC_ID=$(echo $OUTPUTS | jq -r '.vpc_id.value')

echo "Database endpoint: $DB_ENDPOINT"
echo "VPC ID: $VPC_ID"
```

### Passing Outputs to Another Terraform Configuration

When you have multiple Terraform configurations and one needs data from another, you can use the `terraform_remote_state` data source to read outputs from the other configuration's state:

**Networking configuration outputs:**
```hcl
# networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = [for subnet in aws_subnet.private : subnet.id]
}
```

**Application configuration reads those outputs:**
```hcl
# application/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  ami       = "ami-0c55b159cbfafe1f0"
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
}
```

---

## Outputs and `terraform apply -refresh-only`

Running `terraform apply -refresh-only` updates the state without making infrastructure changes. Outputs are recalculated from the refreshed state, so running `terraform output` after a refresh shows the current real values.

---

## A Complete Real-World Example

Here is a complete set of outputs for a three-tier web application:

```hcl
# outputs.tf

# VPC and Networking
output "vpc_id" {
  description = "VPC ID — needed when creating resources in this VPC from other configurations"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "Public subnet IDs — use for load balancers and NAT gateways"
  value       = [for s in aws_subnet.public : s.id]
}

output "private_subnet_ids" {
  description = "Private subnet IDs — use for application servers and databases"
  value       = [for s in aws_subnet.private : s.id]
}

# Load Balancer
output "app_url" {
  description = "Application URL — open this in your browser to access the app"
  value       = "https://${aws_lb.main.dns_name}"
}

output "alb_dns_name" {
  description = "ALB DNS name — use for CNAME records in your DNS provider"
  value       = aws_lb.main.dns_name
}

output "alb_zone_id" {
  description = "ALB hosted zone ID — use for Route53 alias records"
  value       = aws_lb.main.zone_id
}

# EC2 Instances
output "instance_ids" {
  description = "IDs of all application server instances"
  value       = aws_instance.app[*].id
}

# Database
output "db_endpoint" {
  description = "RDS endpoint — use in your application's database configuration"
  value       = aws_db_instance.app.endpoint
}

output "db_port" {
  description = "RDS port number"
  value       = aws_db_instance.app.port
}

# Security Groups
output "app_security_group_id" {
  description = "App server security group ID — use when adding rules for new services"
  value       = aws_security_group.app.id
}

# S3
output "assets_bucket_name" {
  description = "S3 bucket name for static assets — configure your app's ASSETS_BUCKET env var"
  value       = aws_s3_bucket.assets.bucket
}

# Useful combination output
output "deployment_summary" {
  description = "Summary of key deployment information"
  value = {
    environment  = var.environment
    region       = var.aws_region
    app_url      = "https://${aws_lb.main.dns_name}"
    db_endpoint  = aws_db_instance.app.endpoint
    vpc_id       = aws_vpc.main.id
  }
}
```

---

## Summary

- Outputs expose information from your Terraform configuration after apply completes
- Declare outputs using the `output` block with a `value` argument and a `description`
- Outputs appear automatically after `terraform apply` and via `terraform output`
- Use `terraform output -raw name` to get a plain value for use in scripts
- Use `terraform output -json` to get all outputs in JSON format for automation
- Mark outputs `sensitive = true` to hide them in terminal display (but they remain in state)
- Keep all output declarations in an `outputs.tf` file for organization
- Outputs from one Terraform configuration can be consumed by another via `terraform_remote_state`
- Write descriptive output names and descriptions — they serve as documentation for other engineers using your modules

In the next file, we cover the practical use cases for outputs — when and why you use them.
