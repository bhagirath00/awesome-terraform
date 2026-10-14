# State Isolation

## Introduction

State isolation means keeping the state for different environments, teams, or components in separate state files rather than one giant shared state file. It is one of the most important architectural decisions you make when designing your Terraform infrastructure.

Without proper isolation, all your infrastructure — development, staging, production, networking, databases, applications — is tracked in a single state file. Any change anywhere requires locking the entire state. A mistake in one area can affect everything. The blast radius of an error is the entire infrastructure.

With proper isolation, each component has its own state file. Teams work independently. A problem in development does not affect production. A bad apply on the application layer does not risk the networking layer.

---

## Why State Isolation Matters

### Problem 1: The Blast Radius Problem

Imagine all infrastructure in one state file. Someone runs `terraform destroy` by mistake — every resource across all environments is destroyed at once.

With state isolation:
- `terraform destroy` on the dev state only affects dev
- Production is in a completely separate state file — safe

---

### Problem 2: The Lock Contention Problem

With one state file, only one person can run `terraform apply` at a time across all infrastructure. The networking team, the database team, and the application team all compete for the same lock.

With isolation, each team manages their own state independently. All three teams can apply simultaneously because they use different state files and different DynamoDB lock entries.

---

### Problem 3: The Scale Problem

A single state file tracking thousands of resources becomes slow. Every plan has to refresh hundreds or thousands of resource attributes from AWS. Plans that should take seconds take minutes.

Smaller, isolated state files are faster to plan and apply.

---

### Problem 4: The Risk Problem

When everything is in one state, a bad resource declaration anywhere can block plans everywhere. A syntax error in the application config blocks the networking team from making changes.

With isolation, problems are contained to their own state file.

---

## The Three Dimensions of State Isolation

### Dimension 1: Isolate by Environment

Never mix dev, staging, and production in the same state file.

```
mycompany-terraform-state/
├── dev/
│   └── terraform.tfstate
├── staging/
│   └── terraform.tfstate
└── production/
    └── terraform.tfstate
```

Each environment is completely independent. You can destroy dev without any risk to production. You can test changes in dev and staging before applying to production.

---

### Dimension 2: Isolate by Component/Layer

Split infrastructure by functional component. A common layered architecture:

```
mycompany-terraform-state/
├── networking/
│   ├── production/terraform.tfstate    ← VPC, subnets, route tables
│   └── staging/terraform.tfstate
├── platform/
│   ├── production/terraform.tfstate    ← EKS cluster, RDS, ElastiCache
│   └── staging/terraform.tfstate
└── applications/
    ├── myapp/
    │   ├── production/terraform.tfstate ← app deployments, S3, CloudFront
    │   └── staging/terraform.tfstate
    └── otherapp/
        ├── production/terraform.tfstate
        └── staging/terraform.tfstate
```

Each layer is managed independently:
- The networking team manages VPCs and subnets
- The platform team manages EKS and databases
- Application teams manage their own deployments

---

### Dimension 3: Isolate by Team Ownership

When multiple teams manage infrastructure, give each team their own state files to avoid conflicts:

```
mycompany-terraform-state/
├── team-platform/
│   ├── shared-networking/production/terraform.tfstate
│   └── shared-eks/production/terraform.tfstate
├── team-backend/
│   ├── api-service/production/terraform.tfstate
│   └── worker-service/production/terraform.tfstate
└── team-frontend/
    ├── web-app/production/terraform.tfstate
    └── cdn/production/terraform.tfstate
```

Each team has full ownership and autonomy over their state files.

---

## Implementing State Isolation

### Directory-Based Isolation

The simplest approach — one directory per component/environment combination, each with its own backend configuration:

```
infrastructure/
├── networking/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf    ← backend key: "networking/production/terraform.tfstate"
├── platform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── versions.tf    ← backend key: "platform/production/terraform.tfstate"
└── applications/
    └── myapp/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── versions.tf  ← backend key: "applications/myapp/production/terraform.tfstate"
```

Each directory is an independent Terraform configuration with its own state.

---

### Workspace-Based Isolation

Terraform workspaces allow multiple states from a single configuration directory. Each workspace gets its own state file.

```bash
# Create workspaces for each environment
terraform workspace new dev
terraform workspace new staging
terraform workspace new production

# Switch between workspaces
terraform workspace select production
terraform apply -var-file="production.tfvars"

terraform workspace select staging
terraform apply -var-file="staging.tfvars"
```

State files in S3 with workspaces:
```
mycompany-terraform-state/
└── env:/
    ├── dev/terraform.tfstate
    ├── staging/terraform.tfstate
    └── production/terraform.tfstate
```

Workspaces are convenient but have limitations:
- All workspaces share the same configuration code
- Easy to accidentally apply to the wrong workspace
- Better for simple multi-environment scenarios

For complex setups with very different configurations per environment, directory-based isolation is usually clearer.

---

### Using the Current Workspace in Configuration

When using workspaces, reference `terraform.workspace` to customize behavior:

```hcl
locals {
  env    = terraform.workspace
  prefix = "myapp-${local.env}"

  instance_type = {
    dev        = "t2.micro"
    staging    = "t3.small"
    production = "t3.medium"
  }[terraform.workspace]
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = local.instance_type
  tags = {
    Name        = "${local.prefix}-web"
    Environment = local.env
  }
}
```

---

## Connecting Isolated States with Remote State Data Source

State isolation creates a challenge: how do components reference each other's resources? The answer is the `terraform_remote_state` data source.

**Networking configuration outputs its VPC details:**

```hcl
# networking/outputs.tf
output "vpc_id" {
  value = aws_vpc.main.id
}

output "private_subnet_ids" {
  value = [for s in aws_subnet.private : s.id]
}

output "public_subnet_ids" {
  value = [for s in aws_subnet.public : s.id]
}
```

**Application configuration reads the networking outputs:**

```hcl
# applications/myapp/main.tf
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "networking/production/terraform.tfstate"
    region = "us-east-1"
  }
}

resource "aws_instance" "app" {
  ami       = "ami-0c55b159cbfafe1f0"
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
}
```

The application layer reads the VPC and subnet information from the networking layer's state — no direct resource references needed, and the two configurations remain completely independent.

---

## What to Put in Each State

### State File: Networking Layer

Resources that change rarely and are shared by everything:
- VPC and subnets
- Route tables and internet gateways
- NAT gateways
- VPC peering connections
- Shared security groups

### State File: Platform Layer

Resources that are shared across applications but change more often:
- EKS cluster and node groups
- RDS databases and clusters
- ElastiCache clusters
- Shared load balancers

### State File: Application Layer

Resources owned by a specific application:
- Application-specific EC2 instances or ECS services
- Application S3 buckets
- Application-specific security groups
- CloudFront distributions
- Route53 records

### State File: Identity/IAM Layer (Optional)

IAM resources are sensitive and benefit from their own isolated state:
- IAM roles and policies
- IAM users and groups
- Service control policies

---

## State Isolation Anti-Patterns to Avoid

### Anti-Pattern 1: One Giant State File

```
# WRONG — everything in one state file
state-file contains:
  - VPC, subnets, networking
  - EKS cluster
  - RDS databases
  - All application resources
  - All environments (dev, staging, prod)
```

Problems: massive blast radius, lock contention, slow plans, cannot isolate team access.

---

### Anti-Pattern 2: State Per Resource

```
# WRONG — too granular
vpc.tfstate
subnet1.tfstate
subnet2.tfstate
security-group1.tfstate
instance1.tfstate
```

Problems: too many states to manage, breaks resource dependencies within a logical unit, harder to reason about.

---

### Anti-Pattern 3: Mixing Environments

```
# WRONG — dev and production in the same state file
state-file contains:
  - aws_instance.dev_web
  - aws_instance.prod_web
  - aws_db_instance.dev_database
  - aws_db_instance.prod_database
```

Problems: destroying dev could affect production, shared state locks affect both environments.

---

## Summary

- State isolation means keeping different environments, components, and team ownership in separate state files
- Isolate by three dimensions: environment (dev/staging/prod), component/layer (networking/platform/app), and team ownership
- Benefits: reduced blast radius, eliminated lock contention, faster plans, cleaner team boundaries
- Implement isolation using directories (one per component/environment) or Terraform workspaces
- Connect isolated states using the `terraform_remote_state` data source to read outputs from one state in another configuration
- Typical layers: networking, platform, applications, and optionally IAM
- Avoid: one giant state file, over-granular per-resource states, and mixing environments

In the next file, we cover sensitive data in state — understanding what ends up in state and how to protect it.
