# Output Use Cases

## Introduction

We covered how to declare and read outputs in the previous file. This file focuses on **when and why** you use outputs — the real-world situations where outputs make your Terraform work practical and professional.

Understanding use cases helps you decide what to expose as outputs in your own configurations and modules.

---

## Use Case 1: Displaying Important Information After Apply

The most basic use of outputs is simply showing important information after infrastructure is created — information you need right away.

```hcl
output "web_server_public_ip" {
  description = "Public IP — SSH into the server with: ssh ec2-user@<ip>"
  value       = aws_instance.web.public_ip
}

output "application_url" {
  description = "Open this URL to access the application"
  value       = "http://${aws_lb.main.dns_name}"
}

output "rds_endpoint" {
  description = "Database endpoint — set this as DATABASE_HOST in your app config"
  value       = aws_db_instance.app.endpoint
}
```

After `terraform apply`:
```
Outputs:

web_server_public_ip = "54.23.45.67"
application_url      = "http://myapp-alb-1234567890.us-east-1.elb.amazonaws.com"
rds_endpoint         = "myapp.abc123.us-east-1.rds.amazonaws.com:5432"
```

The team immediately knows the server IP, where to access the app, and what database endpoint to use — without digging through the AWS console.

---

## Use Case 2: Connecting Multiple Terraform Configurations

Large infrastructure is often split into multiple Terraform configurations — one for networking, one for compute, one for databases. Outputs are the bridge between them.

**Pattern: Networking layer exposes outputs**

```hcl
# networking/outputs.tf
output "vpc_id" {
  description = "VPC ID for use by other Terraform configurations"
  value       = aws_vpc.main.id
}

output "private_subnet_ids" {
  description = "Private subnet IDs for application servers"
  value       = [for s in aws_subnet.private : s.id]
}

output "public_subnet_ids" {
  description = "Public subnet IDs for load balancers"
  value       = [for s in aws_subnet.public : s.id]
}

output "db_security_group_id" {
  description = "Security group ID that allows database access"
  value       = aws_security_group.database.id
}
```

**Pattern: Application layer reads those outputs**

```hcl
# application/main.tf

# Read outputs from the networking configuration
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "my-company-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use the networking outputs to create app resources
resource "aws_instance" "app" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.small"

  # Reference the networking layer's output
  subnet_id              = data.terraform_remote_state.networking.outputs.private_subnet_ids[count.index % 2]
  vpc_security_group_ids = [aws_security_group.app.id]
}

resource "aws_lb" "main" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"

  # Reference the networking layer's output
  subnets = data.terraform_remote_state.networking.outputs.public_subnet_ids
}
```

This pattern allows teams to manage different layers of infrastructure independently while still connecting them correctly. The networking team owns `networking/`, the app team owns `application/`, and outputs are the interface between them.

---

## Use Case 3: Module Outputs

When you write a reusable module, outputs are how the module exposes information to the calling code. Without outputs, the caller cannot reference anything the module created.

**VPC module outputs:**
```hcl
# modules/vpc/outputs.tf

output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.this.id
}

output "vpc_cidr_block" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ip" {
  description = "Public IP of the NAT gateway"
  value       = aws_eip.nat.public_ip
}
```

**Calling the module and using its outputs:**
```hcl
# main.tf

module "vpc" {
  source = "./modules/vpc"

  cidr_block   = "10.0.0.0/16"
  environment  = var.environment
}

# Use module outputs to create other resources
resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks.arn

  vpc_config {
    # Access the VPC module's outputs
    subnet_ids = module.vpc.private_subnet_ids
  }
}

resource "aws_db_subnet_group" "main" {
  name       = "db-subnet-group"
  # Access the VPC module's outputs
  subnet_ids = module.vpc.private_subnet_ids
}

# Output the module's values at the root level too
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

Module outputs are accessed using `module.module_name.output_name`. This is the standard way modules communicate with the code that uses them.

---

## Use Case 4: Scripting and Automation

Outputs are the clean way to pass infrastructure information to scripts, deployment tools, and other automation.

**Getting values in shell scripts:**
```bash
#!/bin/bash

# Deploy infrastructure
terraform apply -auto-approve -var-file="production.tfvars"

# Get outputs for use in the rest of the script
APP_URL=$(terraform output -raw application_url)
DB_HOST=$(terraform output -raw db_host)
BUCKET=$(terraform output -raw assets_bucket_name)

echo "Infrastructure ready!"
echo "App URL: $APP_URL"

# Run database migrations using the output
PGHOST=$DB_HOST ./scripts/run-migrations.sh

# Upload initial assets to S3 using the output
aws s3 sync ./assets/ s3://$BUCKET/

# Run smoke tests against the new infrastructure
curl -f "$APP_URL/health" || (echo "Health check failed!" && exit 1)

echo "Deployment complete!"
```

**Passing to Ansible for configuration management:**
```bash
# Get the server IPs as a JSON list
SERVER_IPS=$(terraform output -json web_server_ips)

# Use jq to format as Ansible inventory
echo $SERVER_IPS | jq -r '.[]' > /tmp/ansible-inventory

# Run Ansible against the Terraform-provisioned servers
ansible-playbook -i /tmp/ansible-inventory playbooks/configure-app.yml
```

**Using outputs in Python:**
```python
import subprocess
import json

# Get all Terraform outputs as JSON
result = subprocess.run(
    ["terraform", "output", "-json"],
    capture_output=True,
    text=True
)
outputs = json.loads(result.stdout)

# Access specific outputs
db_endpoint = outputs["db_endpoint"]["value"]
bucket_name = outputs["assets_bucket_name"]["value"]
app_url = outputs["application_url"]["value"]

print(f"Configuring app to use database: {db_endpoint}")
print(f"Assets bucket: {bucket_name}")
print(f"Application available at: {app_url}")
```

---

## Use Case 5: CI/CD Pipeline Integration

In CI/CD pipelines, outputs allow different pipeline stages to share infrastructure information.

**GitHub Actions example — deploy infrastructure, then deploy app:**

```yaml
name: Deploy

jobs:
  infrastructure:
    runs-on: ubuntu-latest
    outputs:
      app_url: ${{ steps.tf_outputs.outputs.app_url }}
      s3_bucket: ${{ steps.tf_outputs.outputs.s3_bucket }}

    steps:
      - uses: actions/checkout@v4

      - name: Terraform Apply
        run: terraform apply -auto-approve

      - name: Capture Terraform Outputs
        id: tf_outputs
        run: |
          echo "app_url=$(terraform output -raw application_url)" >> $GITHUB_OUTPUT
          echo "s3_bucket=$(terraform output -raw assets_bucket_name)" >> $GITHUB_OUTPUT

  deploy_app:
    needs: infrastructure
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to S3
        run: |
          aws s3 sync ./dist/ s3://${{ needs.infrastructure.outputs.s3_bucket }}/

      - name: Smoke Test
        run: |
          curl -f "${{ needs.infrastructure.outputs.app_url }}/health"
```

The `infrastructure` job creates the infrastructure and captures the outputs. The `deploy_app` job uses those outputs to deploy the application to the correct S3 bucket and verify the correct URL.

---

## Use Case 6: Documentation and Debugging

Outputs serve as self-documenting records of what your Terraform configuration created. When another engineer inherits your infrastructure or needs to debug something, they can run `terraform output` to quickly get a summary of what exists.

**Designing a `deployment_summary` output:**

```hcl
output "deployment_summary" {
  description = "Complete summary of what this configuration manages"
  value = {
    # Identity
    project     = var.project
    environment = var.environment
    region      = var.aws_region
    account_id  = data.aws_caller_identity.current.account_id

    # Networking
    vpc_id     = aws_vpc.main.id
    vpc_cidr   = aws_vpc.main.cidr_block

    # Access
    app_url    = "https://${aws_lb.main.dns_name}"
    alb_dns    = aws_lb.main.dns_name

    # Database
    db_host    = split(":", aws_db_instance.app.endpoint)[0]
    db_port    = aws_db_instance.app.port
    db_name    = aws_db_instance.app.db_name

    # Storage
    assets_bucket = aws_s3_bucket.assets.bucket
    logs_bucket   = aws_s3_bucket.logs.bucket
  }
}
```

Running `terraform output deployment_summary` gives a complete picture of the deployment without having to look at individual outputs one by one.

---

## Use Case 7: Verifying Infrastructure After Apply

Outputs can be designed specifically to help you quickly verify that the infrastructure is configured correctly:

```hcl
output "verification_checklist" {
  description = "Check these values to verify the deployment is correct"
  value = {
    # Count checks
    instance_count     = length(aws_instance.app)
    subnet_count       = length(aws_subnet.private) + length(aws_subnet.public)
    security_group_ids = [aws_security_group.app.id, aws_security_group.alb.id]

    # Configuration checks
    multi_az_enabled        = aws_db_instance.app.multi_az
    deletion_protection     = aws_db_instance.app.deletion_protection
    storage_encrypted       = aws_db_instance.app.storage_encrypted
    https_listener_enabled  = length(aws_lb_listener.https) > 0

    # Endpoint checks
    load_balancer_dns = aws_lb.main.dns_name
    db_endpoint       = aws_db_instance.app.endpoint
  }
}
```

After applying, you can run `terraform output verification_checklist` and quickly confirm that the right number of instances were created, that multi-AZ is enabled, that storage is encrypted, and so on.

---

## Best Practices for Outputs

### 1. Output Everything Someone Will Need

Think about who will use this configuration and what they will need to know. When in doubt, add the output — it is easy to add and costs nothing.

### 2. Write Clear Descriptions

The description should explain what the value is AND what it is used for:

```hcl
# Good description — tells you what and why
output "nat_gateway_ip" {
  description = "Public IP of the NAT gateway — whitelist this in your third-party firewall rules"
  value       = aws_eip.nat.public_ip
}

# Poor description — just restates the output name
output "nat_gateway_ip" {
  description = "NAT gateway IP"
  value       = aws_eip.nat.public_ip
}
```

### 3. Use Consistent Naming

Name outputs consistently so they are predictable:
- `*_id` for resource IDs (e.g., `vpc_id`, `instance_id`)
- `*_arn` for ARNs (e.g., `role_arn`, `bucket_arn`)
- `*_endpoint` for connection endpoints (e.g., `db_endpoint`, `redis_endpoint`)
- `*_ids` (plural) for lists of IDs (e.g., `subnet_ids`, `instance_ids`)
- `*_url` or `*_dns` for web addresses (e.g., `app_url`, `alb_dns`)

### 4. Group Outputs Logically

Use comments to organize outputs in `outputs.tf`:

```hcl
# ==============
# Networking
# ==============
output "vpc_id" { ... }
output "subnet_ids" { ... }

# ==============
# Compute
# ==============
output "instance_ids" { ... }
output "alb_dns" { ... }

# ==============
# Database
# ==============
output "db_endpoint" { ... }
```

---

## Summary

- **Displaying information** — show IP addresses, URLs, and endpoints immediately after apply
- **Connecting configurations** — use `terraform_remote_state` to pass networking outputs to application configurations
- **Module interfaces** — outputs are how modules expose values to calling code
- **Scripting** — use `terraform output -raw` and `terraform output -json` to pass values to scripts and automation tools
- **CI/CD integration** — capture outputs in one pipeline stage and use them in the next
- **Documentation** — design a `deployment_summary` output for easy human inspection
- **Verification** — design outputs that confirm the infrastructure is configured correctly

Write outputs from the perspective of the person who will use your configuration after you — give them everything they need to connect to, access, and verify the infrastructure you created.

In the next chapter, we move to locals — a powerful way to define computed values inside your configuration.
