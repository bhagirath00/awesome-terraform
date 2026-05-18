# Backend Configuration

## Introduction

The backend configuration tells Terraform where to store its state file and how to connect to that storage. While the previous file focused specifically on the S3 backend setup, this file covers backend configuration in depth — including partial configurations, backend initialization, switching backends, and best practices for managing backend config across teams and environments.

---

## Where Backend Configuration Lives

Backend configuration goes inside the `terraform` block in your configuration files. It is typically placed in a dedicated file called `versions.tf` or `backend.tf`:

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

---

## An Important Limitation — No Variables in Backend Config

One of the most common surprises for new Terraform users: **you cannot use variables inside the backend configuration block**.

This does NOT work:

```hcl
# WRONG — variables are not allowed in backend config
variable "environment" {
  default = "production"
}

terraform {
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "${var.environment}/terraform.tfstate"  # ERROR
    region = var.aws_region                           # ERROR
  }
}
```

Terraform evaluates the backend configuration before processing variables. At the time the backend is initialized, variables have not been loaded yet.

**Why this limitation exists:**
The backend needs to be initialized before Terraform can do anything else — including read variable files. So the backend config must be fully static.

---

## Partial Backend Configuration — The Solution

The solution to the variable limitation is **partial backend configuration**. You put only a minimal (or empty) backend block in your code, and provide the remaining values at `terraform init` time.

### Method 1: Empty Backend Block + `-backend-config` Flag

Put an empty backend block in your code:

```hcl
# versions.tf
terraform {
  backend "s3" {}  # empty — values provided at init time
}
```

Then provide values when running init:

```bash
# Dev environment
terraform init \
  -backend-config="bucket=mycompany-terraform-state" \
  -backend-config="key=dev/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=terraform-state-lock" \
  -backend-config="encrypt=true"

# Production environment
terraform init \
  -backend-config="bucket=mycompany-terraform-state" \
  -backend-config="key=production/terraform.tfstate" \
  -backend-config="region=us-east-1" \
  -backend-config="dynamodb_table=terraform-state-lock" \
  -backend-config="encrypt=true"
```

---

### Method 2: Partial Config in Code + `-backend-config` Flag

Put the common values in code, provide the environment-specific `key` at init time:

```hcl
# versions.tf — common values hardcoded
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
    # key is NOT specified here — provided at init time
  }
}
```

```bash
# Provide only the key at init time
terraform init -backend-config="key=production/terraform.tfstate"
terraform init -backend-config="key=staging/terraform.tfstate"
terraform init -backend-config="key=dev/terraform.tfstate"
```

This approach keeps the common backend settings in version control and only the environment-specific key is provided dynamically.

---

### Method 3: Backend Config File

Create separate backend config files per environment:

```hcl
# backends/dev.hcl
bucket         = "mycompany-terraform-state"
key            = "dev/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-state-lock"
encrypt        = true
```

```hcl
# backends/staging.hcl
bucket         = "mycompany-terraform-state"
key            = "staging/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-state-lock"
encrypt        = true
```

```hcl
# backends/production.hcl
bucket         = "mycompany-terraform-state"
key            = "production/terraform.tfstate"
region         = "us-east-1"
dynamodb_table = "terraform-state-lock"
encrypt        = true
```

Use them with `-backend-config`:

```bash
terraform init -backend-config="backends/production.hcl"
terraform init -backend-config="backends/staging.hcl"
terraform init -backend-config="backends/dev.hcl"
```

This is clean, readable, and easy to manage. The backend config files can be committed to Git (they contain no secrets).

**Recommended directory structure:**

```
my-terraform-project/
├── backends/
│   ├── dev.hcl
│   ├── staging.hcl
│   └── production.hcl
├── main.tf
├── variables.tf
├── outputs.tf
└── versions.tf
```

---

## Backend Initialization

Running `terraform init` initializes the backend. During initialization:

1. Terraform reads the backend configuration (from code and any `-backend-config` flags)
2. Terraform connects to the backend (verifies S3 bucket exists and is accessible)
3. Terraform downloads or reads the existing state from the backend
4. Terraform writes a `.terraform/terraform.tfstate` file locally — this is NOT the main state file, it is Terraform's record of which backend is configured

The `.terraform/terraform.tfstate` file looks like:

```json
{
  "version": 1,
  "serial": 1,
  "lineage": "abc123",
  "backend": {
    "type": "s3",
    "config": {
      "bucket": "mycompany-terraform-state",
      "dynamodb_table": "terraform-state-lock",
      "encrypt": true,
      "key": "production/terraform.tfstate",
      "region": "us-east-1"
    },
    "hash": 12345678
  }
}
```

This file records which backend Terraform is using. It should be in `.gitignore` (it is inside the `.terraform/` directory which should already be ignored).

---

## Switching Backends

### Switching from Local to S3

If you start with local state and want to move to S3:

1. Add the S3 backend configuration to your `versions.tf`
2. Run `terraform init -migrate-state`

```bash
terraform init -migrate-state
```

Terraform detects you have existing local state and asks if you want to migrate it:

```
Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend to the
  newly configured "s3" backend. No existing state was found in the newly configured
  "s3" backend. Do you want to copy this state to the new "s3" backend?

  Enter a value: yes

Successfully configured the backend "s3"! Terraform will automatically
use this backend unless the backend configuration changes.
```

After migration, your local `terraform.tfstate` file is no longer used. All state operations go to S3.

---

### Switching from S3 to a Different Bucket

If you need to move state to a different S3 bucket (for example, when reorganizing):

1. Update the bucket name in your backend configuration
2. Run `terraform init -migrate-state`
3. Terraform copies the state from the old bucket to the new one

```bash
# After updating the bucket name in versions.tf
terraform init -migrate-state
```

---

### Switching from S3 to Terraform Cloud

```hcl
# Change backend from s3 to cloud
terraform {
  cloud {
    organization = "my-organization"
    workspaces {
      name = "production"
    }
  }
}
```

```bash
terraform init -migrate-state
```

Terraform copies state from S3 to Terraform Cloud automatically.

---

### Going Back to Local State (Removing Remote Backend)

Sometimes you need to switch back to local state (for debugging, testing, etc.):

```hcl
# Remove or comment out the backend block
terraform {
  # backend "s3" { ... }  # commented out
}
```

```bash
terraform init -migrate-state
```

Terraform downloads the state from S3 and saves it locally.

---

## Reconfiguring the Backend

If you change backend configuration but want to keep the same backend type, use `-reconfigure`:

```bash
terraform init -reconfigure
```

This reinitializes the backend without migrating state. Use this when:
- You changed a backend configuration option that doesn't affect where state is stored
- You want to force Terraform to forget the current backend and set up fresh

---

## Backend Configuration in CI/CD

In CI/CD pipelines, you typically use environment variables or pipeline secrets for credentials and `-backend-config` for the key:

```yaml
# GitHub Actions example
- name: Terraform Init
  run: |
    terraform init \
      -backend-config="key=${{ github.event.inputs.environment }}/terraform.tfstate" \
      -backend-config="bucket=mycompany-terraform-state" \
      -backend-config="region=us-east-1" \
      -backend-config="dynamodb_table=terraform-state-lock" \
      -backend-config="encrypt=true"
  env:
    AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
    AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

Or using a backend config file that is generated dynamically:

```yaml
- name: Generate Backend Config
  run: |
    cat > backend.hcl << EOF
    bucket         = "mycompany-terraform-state"
    key            = "${{ github.event.inputs.environment }}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
    EOF

- name: Terraform Init
  run: terraform init -backend-config=backend.hcl
```

---

## Backend Configuration Best Practices

### 1. Keep Backend Config in `versions.tf`

Put all version and backend configuration in a single file called `versions.tf`:

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "mycompany-terraform-state"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
    # key provided via -backend-config or backends/*.hcl files
  }
}
```

---

### 2. Use Backend Config Files for Environment-Specific Keys

Keep environment-specific backend values in `backends/` directory files. Commit these to Git — they contain no secrets.

---

### 3. Never Put Credentials in Backend Config

Backend credentials (AWS access keys) should come from environment variables or IAM roles — never from the backend config block.

```hcl
# WRONG — never do this
backend "s3" {
  bucket     = "mycompany-terraform-state"
  access_key = "AKIAIOSFODNN7EXAMPLE"  # never put keys here
  secret_key = "wJalrXUtnFEMI..."      # never put keys here
}

# CORRECT — use environment variables
# export AWS_ACCESS_KEY_ID=...
# export AWS_SECRET_ACCESS_KEY=...
# Or use an IAM role
```

---

### 4. Use `prevent_destroy` on the State Bucket

Protect the state bucket from accidental deletion:

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"

  lifecycle {
    prevent_destroy = true  # Cannot be destroyed via terraform destroy
  }
}
```

---

### 5. Document the Backend Setup

Add a comment in your `versions.tf` explaining how to initialize with the correct backend:

```hcl
# versions.tf
#
# Backend Setup:
# Initialize for production:
#   terraform init -backend-config="backends/production.hcl"
#
# Initialize for staging:
#   terraform init -backend-config="backends/staging.hcl"
#
# The state bucket and lock table are managed by:
#   ../bootstrap/
#

terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

---

## Summary

- Backend configuration goes inside the `terraform` block — variables cannot be used inside it
- Use **partial backend configuration** to handle environment-specific values: empty block + `-backend-config` flag, partial block + `-backend-config` flag, or separate `.hcl` files per environment
- `terraform init -migrate-state` migrates state when switching backends
- `terraform init -reconfigure` reinitializes without migrating state
- In CI/CD, pass environment-specific keys via `-backend-config` and credentials via environment variables
- Never put AWS credentials in the backend config block
- Use backend config files (`backends/dev.hcl`, `backends/production.hcl`) for clean organization
- Document the init commands in your `versions.tf` so teammates know how to initialize

In the next file, we cover state locking with DynamoDB — how it works, how to set it up, and how to handle lock issues.
