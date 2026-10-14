# S3 Backend

## Introduction

The S3 backend is the most widely used remote state backend for AWS teams. It stores your Terraform state file in an Amazon S3 bucket — giving you durability, versioning, encryption, and shared access. Combined with DynamoDB for state locking, it is the complete solution for professional Terraform state management on AWS.

This file walks through everything you need to know about the S3 backend — what it does, how to set up the required AWS resources, how to configure it in Terraform, and all the available configuration options.

---

## What the S3 Backend Provides

Before setting anything up, understand what S3 gives you for state storage:

**Durability:** Amazon S3 is designed for 99.999999999% (11 nines) durability. If you store your state in S3, it is extremely unlikely to ever be lost due to hardware failure.

**Versioning:** With S3 versioning enabled, every version of your state file is preserved. If a bad apply corrupts your state, you can retrieve any previous version.

**Encryption:** S3 supports server-side encryption — your state file is encrypted at rest automatically. This protects sensitive data like passwords and API keys stored in state.

**Access Control:** Access is controlled via IAM policies — the same mechanism used for all AWS permissions. You can restrict who can read or write state using fine-grained IAM rules.

**Audit Trail:** With S3 access logging and AWS CloudTrail enabled, you have a complete record of every time the state file was read or written, by whom, and when.

**Cost:** S3 storage is extremely cheap. A state file is typically a few KB to a few MB. Storing it in S3 costs fractions of a cent per month.

---

## Step 1 — Create the S3 Bucket

The first step is creating an S3 bucket to store your state files. This bucket itself is typically created manually (before Terraform can manage anything) or through a separate bootstrapping Terraform configuration.

**Using the AWS CLI:**

```bash
# Create the bucket (us-east-1 does not need LocationConstraint)
aws s3api create-bucket \
  --bucket mycompany-terraform-state \
  --region us-east-1

# For any other region, you must specify LocationConstraint
aws s3api create-bucket \
  --bucket mycompany-terraform-state \
  --region eu-west-1 \
  --create-bucket-configuration LocationConstraint=eu-west-1
```

**Important:** S3 bucket names are globally unique across all AWS accounts. Choose a name that clearly identifies your organization and purpose — for example `mycompany-terraform-state` or `myorg-infra-tfstate`.

---

## Step 2 — Enable Versioning

Versioning is critical. It preserves every version of the state file so you can recover from accidents.

```bash
aws s3api put-bucket-versioning \
  --bucket mycompany-terraform-state \
  --versioning-configuration Status=Enabled
```

With versioning enabled, every `terraform apply` creates a new version of the state file in S3. You can view all versions:

```bash
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix production/terraform.tfstate
```

To restore a previous version:

```bash
# Copy a specific version to a local file for inspection
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key production/terraform.tfstate \
  --version-id "abc123def456" \
  terraform.tfstate.restored

# After verifying, push it back as the current state
aws s3 cp terraform.tfstate.restored \
  s3://mycompany-terraform-state/production/terraform.tfstate
```

---

## Step 3 — Enable Server-Side Encryption

Encrypt the state file at rest to protect sensitive data:

```bash
aws s3api put-bucket-encryption \
  --bucket mycompany-terraform-state \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "AES256"
        },
        "BucketKeyEnabled": true
      }
    ]
  }'
```

This enables AES-256 encryption for all objects stored in the bucket. For higher security requirements, you can use AWS KMS (Key Management Service) instead:

```bash
aws s3api put-bucket-encryption \
  --bucket mycompany-terraform-state \
  --server-side-encryption-configuration '{
    "Rules": [
      {
        "ApplyServerSideEncryptionByDefault": {
          "SSEAlgorithm": "aws:kms",
          "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789:key/your-key-id"
        }
      }
    ]
  }'
```

---

## Step 4 — Block Public Access

State files must never be publicly accessible. Enable all public access blocks:

```bash
aws s3api put-public-access-block \
  --bucket mycompany-terraform-state \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

---

## Step 5 — Add a Bucket Policy (Optional but Recommended)

Add a bucket policy to enforce encryption and restrict access:

```bash
aws s3api put-bucket-policy \
  --bucket mycompany-terraform-state \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "DenyUnencryptedObjectUploads",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::mycompany-terraform-state/*",
        "Condition": {
          "StringNotEquals": {
            "s3:x-amz-server-side-encryption": "AES256"
          }
        }
      },
      {
        "Sid": "DenyHTTP",
        "Effect": "Deny",
        "Principal": "*",
        "Action": "s3:*",
        "Resource": [
          "arn:aws:s3:::mycompany-terraform-state",
          "arn:aws:s3:::mycompany-terraform-state/*"
        ],
        "Condition": {
          "Bool": {
            "aws:SecureTransport": "false"
          }
        }
      }
    ]
  }'
```

This policy:
- Denies uploads that are not encrypted
- Denies all requests that do not use HTTPS (only encrypted transport allowed)

---

## Step 6 — Configure the Backend in Terraform

Now add the backend configuration to your Terraform code:

```hcl
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

## Step 7 — Initialize with the New Backend

```bash
# If migrating from local state
terraform init -migrate-state

# If starting fresh
terraform init
```

Terraform will connect to S3 and confirm the backend is working. If you are migrating from local state, it will ask if you want to copy the existing state to S3 — type `yes`.

---

## The State Key — Organizing State Files in S3

The `key` in the backend configuration is the path within the S3 bucket where the state file is stored. Think of it like a file path:

```hcl
# Single environment
key = "terraform.tfstate"

# Organized by environment
key = "production/terraform.tfstate"
key = "staging/terraform.tfstate"
key = "dev/terraform.tfstate"

# Organized by project and environment
key = "myapp/production/terraform.tfstate"
key = "myapp/staging/terraform.tfstate"

# Organized by team, project, and environment
key = "platform-team/networking/production/terraform.tfstate"
key = "app-team/backend/production/terraform.tfstate"
key = "app-team/frontend/production/terraform.tfstate"
```

A well-organized key structure in S3 makes it easy to navigate your state files and understand what each one belongs to.

**Recommended structure for organizations:**

```
mycompany-terraform-state/
├── networking/
│   ├── production/terraform.tfstate
│   └── staging/terraform.tfstate
├── platform/
│   ├── eks/
│   │   ├── production/terraform.tfstate
│   │   └── staging/terraform.tfstate
│   └── rds/
│       ├── production/terraform.tfstate
│       └── staging/terraform.tfstate
└── applications/
    ├── myapp/
    │   ├── production/terraform.tfstate
    │   └── staging/terraform.tfstate
    └── otherapp/
        ├── production/terraform.tfstate
        └── staging/terraform.tfstate
```

---

## All S3 Backend Configuration Options

```hcl
backend "s3" {
  # Required
  bucket = "mycompany-terraform-state"     # S3 bucket name
  key    = "production/terraform.tfstate"  # Path within the bucket
  region = "us-east-1"                     # AWS region of the bucket

  # Recommended
  encrypt        = true                    # Encrypt state at rest
  dynamodb_table = "terraform-state-lock" # DynamoDB table for locking

  # Authentication (usually handled by environment variables or IAM role)
  profile        = "terraform-user"        # AWS CLI profile to use
  # access_key   = "AKIA..."              # Avoid — use env vars instead
  # secret_key   = "..."                  # Avoid — use env vars instead

  # Cross-account access
  role_arn       = "arn:aws:iam::123456789:role/TerraformStateRole"
  # Assumes this role before accessing S3

  # KMS encryption (if using KMS instead of AES256)
  kms_key_id     = "arn:aws:kms:us-east-1:123456789:key/your-key-id"

  # Workspace support
  workspace_key_prefix = "env"
  # Creates keys like: env/production/terraform.tfstate
}
```

---

## Using a Terraform Bootstrap Configuration

A common pattern is to use a separate "bootstrap" Terraform configuration that creates the S3 bucket and DynamoDB table. This configuration uses local state (since it creates the remote state infrastructure itself).

```hcl
# bootstrap/main.tf
# This configuration creates the remote state infrastructure
# It uses LOCAL state (no backend block)

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "terraform_state" {
  bucket = "mycompany-terraform-state"

  lifecycle {
    prevent_destroy = true  # Never accidentally delete the state bucket
  }

  tags = {
    Name      = "Terraform State"
    ManagedBy = "Terraform"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket                  = aws_s3_bucket.terraform_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  lifecycle {
    prevent_destroy = true  # Never accidentally delete the lock table
  }

  tags = {
    Name      = "Terraform State Lock"
    ManagedBy = "Terraform"
  }
}

output "state_bucket_name" {
  value = aws_s3_bucket.terraform_state.bucket
}

output "lock_table_name" {
  value = aws_dynamodb_table.terraform_locks.name
}
```

Run this bootstrap configuration once:
```bash
cd bootstrap/
terraform init
terraform apply
```

After this, all your other Terraform configurations can use the S3 backend.

---

## IAM Permissions for the S3 Backend

The IAM user or role running Terraform needs specific permissions to use the S3 backend:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::mycompany-terraform-state",
        "arn:aws:s3:::mycompany-terraform-state/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/terraform-state-lock"
    }
  ]
}
```

---

## Summary

- The S3 backend stores Terraform state in an S3 bucket — the most popular choice for AWS teams
- Setting up requires: creating an S3 bucket, enabling versioning, enabling encryption, blocking public access, and creating a DynamoDB table for locking
- Configure the backend in the `terraform` block with at minimum: `bucket`, `key`, `region`, and `encrypt = true`
- The `key` is the path inside the S3 bucket — organize it by project, environment, or team for clarity
- Use a bootstrap Terraform configuration to create the state bucket and lock table themselves
- Add `prevent_destroy = true` to the state bucket and lock table to prevent accidental deletion
- The running Terraform process needs IAM permissions to read/write S3 and read/write/delete DynamoDB

In the next file, we cover all backend configuration options in detail — including how to handle partial configurations and workspace-based state organization.
