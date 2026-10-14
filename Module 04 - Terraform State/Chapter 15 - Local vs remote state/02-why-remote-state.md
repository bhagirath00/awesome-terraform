# Why Remote State

## Introduction

Remote state means storing your Terraform state file in a shared, external location rather than on your local machine. Instead of `terraform.tfstate` sitting in your project directory, it lives in a place like an AWS S3 bucket, Terraform Cloud, or Azure Blob Storage — somewhere that every team member and every CI/CD pipeline can access.

Remote state is not optional for professional Terraform work. The moment you have more than one person working on infrastructure, or the moment you start managing real production environments, remote state becomes a necessity.

This file explains exactly what problems remote state solves, how it works, what backends are available, and why the S3 + DynamoDB combination is the most popular choice for AWS teams.

---

## The Four Problems Remote State Solves

### Problem 1: State Cannot Be Shared (Collaboration)

**The problem with local state:**

Your teammate Sarah joins the project. She clones the repository, runs `terraform init`, and then runs `terraform plan`. Terraform has no state file on her machine, so it thinks nothing exists. It plans to create every single resource from scratch.

If she runs `terraform apply`, Terraform will try to create duplicate resources — a second VPC, a second database, a second load balancer — alongside the ones you already created. This is a serious problem.

**How remote state solves it:**

Both you and Sarah point Terraform at the same state file in S3. When either of you runs `terraform plan`, Terraform reads the shared state. It knows what exists. Plans are accurate. No duplicates.

```hcl
# Both you and Sarah use this same backend configuration
terraform {
  backend "s3" {
    bucket = "mycompany-terraform-state"
    key    = "production/terraform.tfstate"
    region = "us-east-1"
  }
}
```

When Sarah runs `terraform init`, Terraform downloads the provider plugins and connects to the S3 backend. Her local machine never stores the state — it always reads from and writes to S3.

---

### Problem 2: Concurrent Applies Corrupt State (Locking)

**The problem with local state:**

You run `terraform apply` to add a new subnet. At the same moment, Sarah runs `terraform apply` to update a security group. Both are reading state serial 42.

You finish first and write state serial 43 (with your subnet added).
Sarah finishes and writes state serial 43 (with her security group updated, but without your subnet).

Sarah's write overwrites yours. Your subnet changes are lost from state. The real infrastructure has the subnet, but state does not. You now have drift and an inconsistent state file.

**How remote state solves it — State Locking:**

When remote state is combined with a locking mechanism (DynamoDB for S3, or built-in locking for Terraform Cloud), only one `terraform apply` can run at a time.

```
You run terraform apply:
  → Terraform acquires a lock on the state
  → Lock entry created in DynamoDB: {project: "myapp", locked_by: "you"}

Sarah runs terraform apply at the same time:
  → Terraform tries to acquire a lock
  → Sees existing lock entry
  → Outputs: "Error: state is locked by another process"
  → Sarah must wait or you must release the lock

You finish:
  → Terraform writes the new state
  → Terraform releases the lock

Sarah runs terraform apply again:
  → Lock is available
  → Sarah acquires the lock
  → Sarah's apply runs successfully
```

This guarantees that state is always updated atomically — no two applies can interfere with each other.

---

### Problem 3: State is Lost When Local Machine Fails (Durability)

**The problem with local state:**

Your laptop is stolen. Or the hard drive fails. Or you accidentally delete the project directory. Your `terraform.tfstate` file is gone.

Without state, Terraform has no idea what infrastructure exists. You cannot run `terraform plan` accurately. You cannot run `terraform destroy`. You have to manually track down every resource and either import it back into a new state file or manually delete it from AWS.

For a large infrastructure with hundreds of resources, this recovery can take days.

**How remote state solves it:**

S3 is designed for durability — Amazon guarantees 99.999999999% (11 nines) durability for objects stored in S3. Your state file is safer in S3 than on any laptop.

Additionally, with S3 versioning enabled, every version of the state file is preserved. If you accidentally corrupt the state file, you can retrieve any previous version from the S3 version history.

```bash
# List all versions of the state file
aws s3api list-object-versions \
  --bucket mycompany-terraform-state \
  --prefix production/terraform.tfstate

# Restore a specific version
aws s3api get-object \
  --bucket mycompany-terraform-state \
  --key production/terraform.tfstate \
  --version-id "abcdef123456" \
  terraform.tfstate.restored
```

---

### Problem 4: No Audit Trail (History and Accountability)

**The problem with local state:**

When something goes wrong in production, you need to answer: "What changed? When? Who made the change?" Local state files have no history — you only have the current state and one backup.

**How remote state solves it:**

With S3 versioning enabled, every apply creates a new version of the state file. Combined with S3 server access logging, you have a complete audit trail:

- Every version of the state file is preserved
- Timestamps show when each change was made
- CloudTrail logs show who made the API calls (who ran the apply)

For compliance, security audits, and post-incident reviews, this history is invaluable.

---

## How Remote State Works

The flow with remote state is nearly identical to local state from a user perspective — the difference is invisible in your daily workflow.

```
Without remote state:
terraform apply → reads local .tfstate → makes changes → writes local .tfstate

With remote state:
terraform apply → downloads state from S3 → makes changes → uploads new state to S3
```

Terraform downloads the state at the start of each operation, works with it locally in memory, and then uploads the updated state when done. You never see or touch the state file directly — it all happens automatically.

---

## Available Remote Backends

Terraform supports many remote backends. Here are the most common ones:

### S3 Backend (AWS — Most Popular)

The most widely used backend for AWS teams. Stores state in an S3 bucket with optional DynamoDB locking.

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

**Pros:** Well-supported, very cheap, integrates naturally with AWS, versioning built-in, encryption at rest
**Cons:** Requires initial setup (creating S3 bucket and DynamoDB table)

---

### Terraform Cloud Backend

HashiCorp's managed state service. Free for small teams, paid for larger organizations.

```hcl
terraform {
  cloud {
    organization = "my-organization"
    workspaces {
      name = "production"
    }
  }
}
```

**Pros:** Zero setup, built-in locking, UI for state inspection, run history, team access controls
**Cons:** Data leaves your AWS account (may not meet some compliance requirements), cost for larger teams

---

### Azure Blob Storage Backend

The equivalent of S3 for Azure teams.

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "mycompanytfstate"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
  }
}
```

---

### GCS Backend (Google Cloud Storage)

The equivalent of S3 for GCP teams.

```hcl
terraform {
  backend "gcs" {
    bucket = "mycompany-terraform-state"
    prefix = "production"
  }
}
```

---

## The S3 + DynamoDB Setup — Why It's the Standard

For AWS teams, S3 + DynamoDB is the industry standard for remote state because:

1. **S3** provides durable, versioned, encrypted object storage for the state file
2. **DynamoDB** provides the locking mechanism to prevent concurrent applies
3. Both are AWS-native — no third-party service needed
4. Combined cost is minimal — typically less than $1/month for most teams
5. Access is controlled via IAM — the same way everything else in AWS is secured

The combination gives you everything remote state needs:
- ✅ Shared access for all team members
- ✅ State locking via DynamoDB
- ✅ 11-nines durability from S3
- ✅ Full version history with S3 versioning
- ✅ Encryption at rest with S3 SSE
- ✅ Access control via IAM policies
- ✅ Audit trail via CloudTrail

---

## Setting Up Remote State — Overview

Here is the high-level process for setting up S3 + DynamoDB remote state (detailed configuration is covered in Chapter 16):

**Step 1:** Create the S3 bucket for state storage
```bash
aws s3api create-bucket \
  --bucket mycompany-terraform-state \
  --region us-east-1
```

**Step 2:** Enable versioning on the bucket
```bash
aws s3api put-bucket-versioning \
  --bucket mycompany-terraform-state \
  --versioning-configuration Status=Enabled
```

**Step 3:** Enable server-side encryption
```bash
aws s3api put-bucket-encryption \
  --bucket mycompany-terraform-state \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

**Step 4:** Create the DynamoDB table for locking
```bash
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

**Step 5:** Configure the backend in your Terraform code
```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

**Step 6:** Initialize Terraform (migrates state if you had local state)
```bash
terraform init -migrate-state
```

---

## Migrating from Local to Remote State

If you already have local state and want to move to remote state, Terraform handles the migration automatically:

```bash
# 1. Add the backend configuration to your .tf files

# 2. Run init with migrate-state flag
terraform init -migrate-state
```

Terraform will:
1. Read your existing local state
2. Connect to the new remote backend
3. Copy the state to the remote backend
4. Confirm the migration was successful

You will see a prompt:
```
Initializing the backend...
Do you want to copy existing state to the new backend?
  Pre-existing state was found while migrating the previous "local" backend to the
  newly configured "s3" backend. No existing state was found in the newly configured
  "s3" backend. Do you want to copy this state to the new "s3" backend?

  Enter a value: yes
```

Type `yes` and your local state is safely migrated to S3.

---

## Remote State Data Source

One powerful feature of remote state is the ability to read outputs from one Terraform configuration in another. This is called the **remote state data source**:

```hcl
# In your application Terraform config
# Read outputs from the networking config's state
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "mycompany-terraform-state"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use the networking config's outputs
resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.networking.outputs.private_subnet_ids[0]
}
```

This is how large infrastructure is split into manageable pieces while still allowing different configurations to reference each other's resources.

---

## Summary

- Remote state stores `terraform.tfstate` in a shared external location (S3, Terraform Cloud, Azure Blob, GCS) instead of locally
- Remote state solves four critical problems: collaboration (sharing), locking (preventing concurrent applies), durability (no data loss), and history (audit trail)
- S3 + DynamoDB is the industry standard for AWS teams — durable, cheap, secure, and fully AWS-native
- Remote state works transparently — your daily workflow (init, plan, apply) stays the same
- Migrating from local to remote state is handled automatically by `terraform init -migrate-state`
- The `terraform_remote_state` data source lets you read outputs from one configuration's state in another configuration
- For any team environment or production infrastructure, remote state is not optional — it is a requirement

In Chapter 16, we configure remote backends in detail — setting up S3 + DynamoDB step by step, configuring the backend block, and understanding all available options.
