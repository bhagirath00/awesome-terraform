# Terraform tfvars Files

## Introduction

`.tfvars` files are how you provide values for your Terraform variables in a clean, organized way. Instead of passing values one by one on the command line every time you run Terraform, you put them in a file and Terraform reads them automatically.

This file covers everything about `.tfvars` files — the file format, automatic loading, manual loading, managing multiple environments, and best practices used in professional teams.

---

## What is a tfvars File?

A `.tfvars` file is a plain text file that contains variable assignments in HCL format. It is separate from your main configuration files — it only contains values, not declarations.

**Variable declarations** live in `variables.tf`:
```hcl
variable "environment" {
  type = string
}

variable "instance_type" {
  type = string
}

variable "instance_count" {
  type = number
}
```

**Variable values** live in `terraform.tfvars`:
```hcl
environment    = "dev"
instance_type  = "t2.micro"
instance_count = 2
```

The two files work together — declarations define what variables exist and their types, while the `.tfvars` file provides the actual values.

---

## The `terraform.tfvars` File

Terraform automatically loads a file named exactly `terraform.tfvars` if it exists in your working directory. No flags needed — just having the file there is enough.

```
my-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
└── terraform.tfvars    ← loaded automatically
```

When you run `terraform plan` or `terraform apply`, Terraform reads `terraform.tfvars` and uses those values for the matching variables.

### Example `terraform.tfvars`

```hcl
# General
project     = "my-web-application"
environment = "dev"
aws_region  = "us-east-1"

# Networking
vpc_cidr             = "10.0.0.0/16"
availability_zones   = ["us-east-1a", "us-east-1b"]
public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]

# Compute
instance_type  = "t2.micro"
instance_count = 1

# Database
db_instance_class = "db.t3.micro"
db_name           = "appdb"
db_username       = "dbadmin"
multi_az          = false

# Tags
common_tags = {
  ManagedBy = "Terraform"
  Team      = "platform"
}
```

---

## The `*.auto.tfvars` Files

Any file ending in `.auto.tfvars` is also automatically loaded. You can have multiple files, and Terraform loads all of them alphabetically.

```
my-project/
├── main.tf
├── variables.tf
├── terraform.tfvars      ← loaded automatically
├── networking.auto.tfvars ← also loaded automatically
└── compute.auto.tfvars   ← also loaded automatically
```

This is useful for splitting a large set of variables into organized categories:

**`networking.auto.tfvars`:**
```hcl
vpc_cidr             = "10.0.0.0/16"
availability_zones   = ["us-east-1a", "us-east-1b"]
public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]
```

**`compute.auto.tfvars`:**
```hcl
instance_type  = "t3.small"
instance_count = 3
min_instances  = 1
max_instances  = 10
```

**`database.auto.tfvars`:**
```hcl
db_instance_class = "db.t3.small"
db_name           = "appdb"
db_username       = "dbadmin"
multi_az          = false
```

---

## Manual Variable Files with `-var-file`

Files with any other name (not `terraform.tfvars` and not ending in `.auto.tfvars`) must be specified explicitly with the `-var-file` flag:

```bash
terraform plan -var-file="production.tfvars"
terraform apply -var-file="staging.tfvars"
terraform apply -var-file="environments/production.tfvars"
```

You can specify multiple `-var-file` flags:
```bash
terraform apply \
  -var-file="base.tfvars" \
  -var-file="production.tfvars"
```

When multiple files are specified, later files override earlier ones for the same variable.

---

## Managing Multiple Environments with tfvars

The most important use of `.tfvars` files in real-world DevOps work is **managing different environments** (dev, staging, production) with the same Terraform code but different values.

### Directory-per-environment approach

```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
└── environments/
    ├── dev.tfvars
    ├── staging.tfvars
    └── production.tfvars
```

**`environments/dev.tfvars`:**
```hcl
environment    = "dev"
instance_type  = "t2.micro"
instance_count = 1
multi_az       = false

db_instance_class = "db.t3.micro"
db_name           = "appdb_dev"

common_tags = {
  Environment = "dev"
  ManagedBy   = "Terraform"
  CostCenter  = "engineering-dev"
}
```

**`environments/staging.tfvars`:**
```hcl
environment    = "staging"
instance_type  = "t3.small"
instance_count = 2
multi_az       = false

db_instance_class = "db.t3.small"
db_name           = "appdb_staging"

common_tags = {
  Environment = "staging"
  ManagedBy   = "Terraform"
  CostCenter  = "engineering-staging"
}
```

**`environments/production.tfvars`:**
```hcl
environment    = "production"
instance_type  = "t3.medium"
instance_count = 5
multi_az       = true

db_instance_class = "db.t3.medium"
db_name           = "appdb_prod"

common_tags = {
  Environment = "production"
  ManagedBy   = "Terraform"
  CostCenter  = "product-production"
}
```

Deploying to each environment:
```bash
# Deploy to dev
terraform apply -var-file="environments/dev.tfvars"

# Deploy to staging
terraform apply -var-file="environments/staging.tfvars"

# Deploy to production
terraform apply -var-file="environments/production.tfvars"
```

---

## Variable Precedence with tfvars Files

When multiple sources provide values for the same variable, this is the order Terraform uses (later sources override earlier ones):

```
1. Default value in variable declaration           (lowest priority)
2. TF_VAR_ environment variables
3. terraform.tfvars (auto-loaded)
4. *.auto.tfvars files (auto-loaded, alphabetical)
5. -var-file flags (in order specified)
6. -var flags                                       (highest priority)
```

**Practical example:**

You have `terraform.tfvars` with `instance_type = "t2.micro"`, and you want to test with a larger instance without editing the file:

```bash
terraform plan -var="instance_type=t3.medium"
```

The `-var` flag overrides the value from `terraform.tfvars`. The file is unchanged — just the current run uses the larger instance type.

---

## Sensitive Values in tfvars Files

You can put sensitive values in `.tfvars` files, but you must be careful about security:

```hcl
# secrets.tfvars — NEVER commit this to Git
db_password     = "SuperSecretPassword123!"
api_secret_key  = "sk_live_abcdef1234567890"
```

**Rules for tfvars files with secrets:**

1. **Add to `.gitignore` immediately** — never commit secret values to Git
2. **Use a separate file** — keep secrets in their own file (e.g., `secrets.tfvars`) so you can gitignore that specific file
3. **Consider using environment variables instead** — `TF_VAR_db_password` is often safer for secrets

**Example `.gitignore` for a Terraform project:**
```gitignore
# Terraform state files
*.tfstate
*.tfstate.backup
.terraform/
.terraform.lock.hcl

# Sensitive variable files
secrets.tfvars
*.secrets.tfvars
*-secrets.tfvars

# Local override files
local.tfvars
*.local.tfvars
```

Your non-secret variable files (`dev.tfvars`, `staging.tfvars`, `production.tfvars`) can be committed to Git because they contain no sensitive information.

---

## tfvars File Format

### Simple String Values

```hcl
project     = "my-application"
environment = "production"
aws_region  = "us-east-1"
```

### Numbers and Booleans

```hcl
instance_count = 5
multi_az       = true
port           = 8080
timeout        = 30
```

### Lists

```hcl
availability_zones  = ["us-east-1a", "us-east-1b", "us-east-1c"]
allowed_cidr_blocks = ["10.0.0.0/8", "172.16.0.0/12"]
allowed_ports       = [80, 443]
```

### Maps

```hcl
common_tags = {
  ManagedBy   = "Terraform"
  Environment = "production"
  Team        = "platform-engineering"
  CostCenter  = "engineering-prod"
}

instance_type_per_env = {
  dev        = "t2.micro"
  staging    = "t3.small"
  production = "t3.medium"
}
```

### Objects

```hcl
database = {
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  multi_az          = true
}
```

### Lists of Objects

```hcl
security_group_rules = [
  {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  },
  {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
]
```

---

## tfvars in CI/CD Pipelines

In CI/CD pipelines (GitHub Actions, Jenkins, GitLab CI), you typically:

1. Store non-secret values in committed `.tfvars` files
2. Store secrets as CI/CD secrets (GitHub Secrets, Jenkins Credentials) and pass them as `-var` flags or `TF_VAR_` environment variables

**GitHub Actions example:**
```yaml
name: Terraform Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/TerraformRole
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan -var-file="environments/production.tfvars"
        env:
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
          TF_VAR_api_key: ${{ secrets.API_KEY }}

      - name: Terraform Apply
        run: terraform apply -var-file="environments/production.tfvars" -auto-approve
        env:
          TF_VAR_db_password: ${{ secrets.DB_PASSWORD }}
          TF_VAR_api_key: ${{ secrets.API_KEY }}
```

The committed `environments/production.tfvars` file contains all the non-secret values. Secrets come from GitHub Secrets via environment variables.

---

## Summary

- `.tfvars` files hold variable values separately from variable declarations
- `terraform.tfvars` is loaded automatically — just having it in your directory is enough
- `*.auto.tfvars` files are also loaded automatically (alphabetically)
- Files with other names require `-var-file="filename.tfvars"` to load them
- Use separate `.tfvars` files per environment (dev.tfvars, staging.tfvars, production.tfvars) for clean environment management
- Variable precedence: default → `TF_VAR_` env vars → auto-loaded files → `-var-file` flags → `-var` flags
- Never commit `.tfvars` files containing secrets to Git — add them to `.gitignore`
- Keep secrets out of `.tfvars` files entirely — use `TF_VAR_` environment variables for secrets in CI/CD
- `.tfvars` files support all HCL value types: strings, numbers, booleans, lists, maps, objects

In the next chapter, we cover output values — how to expose useful information after a Terraform apply.
