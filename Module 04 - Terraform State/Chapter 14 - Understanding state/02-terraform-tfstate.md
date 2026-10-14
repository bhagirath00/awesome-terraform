# The terraform.tfstate File

## Introduction

The `terraform.tfstate` file is where Terraform stores all the information about the infrastructure it manages. Even though you never need to edit it manually, understanding its structure helps you debug problems, understand what Terraform is tracking, and use state commands more effectively.

This file walks through the structure of a real state file, explains what every section means, and shows you how to inspect state safely.

---

## What the State File Looks Like

Let's create a simple Terraform configuration and look at the state it produces.

**Configuration (`main.tf`):**

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "app_data" {
  bucket = "myapp-production-data-2024"

  tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}

resource "aws_s3_bucket_versioning" "app_data" {
  bucket = aws_s3_bucket.app_data.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

After running `terraform apply`, the `terraform.tfstate` file looks like this:

```json
{
  "version": 4,
  "terraform_version": "1.6.6",
  "serial": 3,
  "lineage": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_s3_bucket",
      "name": "app_data",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "acceleration_status": "",
            "acl": null,
            "arn": "arn:aws:s3:::myapp-production-data-2024",
            "bucket": "myapp-production-data-2024",
            "bucket_domain_name": "myapp-production-data-2024.s3.amazonaws.com",
            "bucket_prefix": null,
            "bucket_regional_domain_name": "myapp-production-data-2024.s3.us-east-1.amazonaws.com",
            "cors_rule": [],
            "force_destroy": false,
            "grant": [],
            "hosted_zone_id": "Z3AQBSTGFYJSTF",
            "id": "myapp-production-data-2024",
            "lifecycle_rule": [],
            "logging": [],
            "object_lock_configuration": [],
            "object_lock_enabled": false,
            "policy": null,
            "region": "us-east-1",
            "replication_configuration": [],
            "request_payer": "BucketOwner",
            "server_side_encryption_configuration": [],
            "tags": {
              "Environment": "production",
              "ManagedBy": "Terraform"
            },
            "tags_all": {
              "Environment": "production",
              "ManagedBy": "Terraform"
            },
            "versioning": [],
            "website": []
          },
          "sensitive_attributes": [],
          "private": "eyJlMmJmYjMwZS1lY2...",
          "dependencies": []
        }
      ]
    },
    {
      "mode": "managed",
      "type": "aws_s3_bucket_versioning",
      "name": "app_data",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "bucket": "myapp-production-data-2024",
            "expected_bucket_owner": null,
            "id": "myapp-production-data-2024",
            "mfa": null,
            "versioning_configuration": [
              {
                "mfa_delete": "",
                "status": "Enabled"
              }
            ]
          },
          "sensitive_attributes": [],
          "private": "eyJlMmJmYjMwZS1lY2...",
          "dependencies": [
            "aws_s3_bucket.app_data"
          ]
        }
      ]
    }
  ],
  "check_results": null
}
```

Now let's go through every part of this file.

---

## Top-Level Fields

### `version`

```json
"version": 4
```

The version of the state file format. This is Terraform's internal format version — not the version of your Terraform code. Version 4 is the current format used by Terraform 0.13 and later. Terraform uses this to know how to read the file.

---

### `terraform_version`

```json
"terraform_version": "1.6.6"
```

The exact version of Terraform that last wrote this state file. This is useful for debugging — if you upgrade Terraform and something breaks, you can check which version created the state.

---

### `serial`

```json
"serial": 3
```

A counter that increases by 1 every time the state file is modified. Think of it as a revision number.

- When Terraform writes state, it increments the serial
- When merging state (for example, after a conflict), Terraform checks serials to determine which version is newer
- If two state files have the same lineage but different serials, the one with the higher serial is newer

In our example, serial `3` means this state file has been written 3 times (initial creation + 2 subsequent applies).

---

### `lineage`

```json
"lineage": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
```

A unique identifier (UUID) that is generated when the state file is first created. It never changes for the lifetime of the state file.

The lineage serves as an identity for this specific state file. If you try to push a state file that has a different lineage than the remote state, Terraform will refuse — this prevents accidentally overwriting state from a completely different project.

---

### `outputs`

```json
"outputs": {}
```

All output values from your configuration are stored here. If you had an output block, it would appear here:

```json
"outputs": {
  "bucket_name": {
    "value": "myapp-production-data-2024",
    "type": "string",
    "sensitive": false
  }
}
```

This is how `terraform output` reads values without needing to query AWS — it just reads from the state file.

---

## The `resources` Array

This is the heart of the state file. Every resource Terraform manages has an entry here.

### `mode`

```json
"mode": "managed"
```

Two possible values:
- `"managed"` — a resource you created with a `resource` block (most resources)
- `"data"` — a data source you queried with a `data` block

Data sources are also tracked in state so Terraform knows their values for subsequent plans.

---

### `type`

```json
"type": "aws_s3_bucket"
```

The resource type — exactly as written in your `.tf` file. This tells Terraform which provider resource this is and how to interact with it.

---

### `name`

```json
"name": "app_data"
```

The local name you gave the resource in your `.tf` file — the second label in the resource block:

```hcl
resource "aws_s3_bucket" "app_data" {   # "app_data" is the name
```

Together, `type` + `name` form the resource address: `aws_s3_bucket.app_data`.

---

### `provider`

```json
"provider": "provider[\"registry.terraform.io/hashicorp/aws\"]"
```

Which provider manages this resource. This is the full registry address of the provider. Terraform uses this to know which provider plugin to call when reading or updating this resource.

---

### `instances`

This is an array because some resources can have multiple instances (when using `count` or `for_each`). For a single resource, there is one item in this array.

#### `schema_version`

```json
"schema_version": 0
```

The schema version of the resource within the provider. Providers can upgrade their resource schemas over time. This number helps Terraform migrate state when providers release new schema versions.

---

#### `attributes`

```json
"attributes": {
  "id": "myapp-production-data-2024",
  "bucket": "myapp-production-data-2024",
  "arn": "arn:aws:s3:::myapp-production-data-2024",
  "region": "us-east-1",
  ...
}
```

This is the most important part of each resource entry. It stores **every attribute** of the resource — both the ones you set and the ones AWS assigned.

Key things stored here:
- `id` — the unique identifier of this resource in AWS (used to call the AWS API for read/update/delete)
- Every argument you set in your `.tf` file
- Every computed attribute that AWS assigned after creation

This is why `terraform plan` is fast — it reads attribute values from state rather than querying every resource from AWS every time.

---

#### `sensitive_attributes`

```json
"sensitive_attributes": []
```

A list of attribute paths that contain sensitive values. When an attribute is marked sensitive (like a password), its value is still stored in state (in plain text), but this field tells Terraform to hide it in plan and apply output.

**Important:** Even though `sensitive_attributes` marks values for hiding in output, the actual values are stored in plain text in the state file. This is why encrypting and securing your state file is critical.

---

#### `private`

```json
"private": "eyJlMmJmYjMwZS1lY2..."
```

Base64-encoded internal data used by the provider. This is opaque — it is for the provider's internal use and you should never try to decode or modify it. Providers use this to store metadata they need between operations.

---

#### `dependencies`

```json
"dependencies": [
  "aws_s3_bucket.app_data"
]
```

A list of other resources that this resource depends on. In our example, `aws_s3_bucket_versioning.app_data` depends on `aws_s3_bucket.app_data` because it uses the bucket's ID.

Terraform uses this dependency information to:
- Create resources in the correct order (dependencies first)
- Destroy resources in the correct reverse order (dependents first)

---

## State with `count` Resources

When you use `count`, each instance gets its own entry in the `instances` array with an `index_key`:

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

State entry:
```json
{
  "type": "aws_instance",
  "name": "web",
  "instances": [
    {
      "index_key": 0,
      "attributes": { "id": "i-0001", ... }
    },
    {
      "index_key": 1,
      "attributes": { "id": "i-0002", ... }
    },
    {
      "index_key": 2,
      "attributes": { "id": "i-0003", ... }
    }
  ]
}
```

---

## State with `for_each` Resources

When you use `for_each`, each instance uses the map key as its `index_key`:

```hcl
resource "aws_s3_bucket" "envs" {
  for_each = toset(["dev", "staging", "prod"])
  bucket   = "myapp-${each.value}-data"
}
```

State entry:
```json
{
  "type": "aws_s3_bucket",
  "name": "envs",
  "instances": [
    {
      "index_key": "dev",
      "attributes": { "id": "myapp-dev-data", ... }
    },
    {
      "index_key": "prod",
      "attributes": { "id": "myapp-prod-data", ... }
    },
    {
      "index_key": "staging",
      "attributes": { "id": "myapp-staging-data", ... }
    }
  ]
}
```

---

## The Backup File

Every time Terraform successfully writes a new state, it saves the previous state to `terraform.tfstate.backup`:

```
my-project/
├── terraform.tfstate         ← current state
└── terraform.tfstate.backup  ← previous state (one version back)
```

Only one backup is kept — each apply overwrites the previous backup with the state before that apply.

If something goes seriously wrong right after an apply, you can restore the previous state:

```bash
# Restore the backup (use with caution)
cp terraform.tfstate.backup terraform.tfstate
```

---

## Inspecting State Safely

Instead of opening the JSON file directly, use Terraform commands to inspect state:

```bash
# See all resources in state (cleaner than reading the JSON)
terraform state list

# See full details of one resource (formatted, readable)
terraform state show aws_s3_bucket.app_data

# Get state as JSON (for automation/scripting)
terraform show -json | jq .
```

Output of `terraform state show aws_s3_bucket.app_data`:
```hcl
# aws_s3_bucket.app_data:
resource "aws_s3_bucket" "app_data" {
    arn                         = "arn:aws:s3:::myapp-production-data-2024"
    bucket                      = "myapp-production-data-2024"
    bucket_domain_name          = "myapp-production-data-2024.s3.amazonaws.com"
    force_destroy               = false
    hosted_zone_id              = "Z3AQBSTGFYJSTF"
    id                          = "myapp-production-data-2024"
    object_lock_enabled         = false
    region                      = "us-east-1"
    request_payer               = "BucketOwner"
    tags                        = {
        "Environment" = "production"
        "ManagedBy"   = "Terraform"
    }
}
```

This is much more readable than the raw JSON file.

---

## `.gitignore` for State Files

Since state files often contain sensitive data (passwords, keys, tokens stored in resource attributes), you must add them to `.gitignore`:

```gitignore
# Terraform state files — never commit these
*.tfstate
*.tfstate.backup

# Terraform working directory
.terraform/

# Crash logs
crash.log
crash.*.log

# Lock file should be committed (do NOT add this)
# .terraform.lock.hcl
```

The lock file (`.terraform.lock.hcl`) is NOT sensitive and should be committed to Git. State files are sensitive and should never be committed.

---

## Summary

- The `terraform.tfstate` file is a JSON file that stores complete information about all resources Terraform manages
- Top-level fields include `version`, `terraform_version`, `serial` (revision counter), `lineage` (unique identity), and `outputs`
- Each resource entry contains `mode`, `type`, `name`, `provider`, and an `instances` array
- Each instance stores all `attributes` (both what you set and what AWS assigned), `dependencies`, `sensitive_attributes`, and internal `private` data
- The `serial` increments on every write — Terraform uses this to detect conflicts
- The `lineage` never changes — Terraform uses this to prevent accidentally mixing state from different projects
- Terraform keeps one backup in `terraform.tfstate.backup`
- Always use `terraform state list` and `terraform state show` to inspect state — never edit the JSON directly
- Add `*.tfstate` and `*.tfstate.backup` to `.gitignore` — state files often contain sensitive data

In the next file, we explore the difference between Terraform state and real infrastructure — and what happens when they get out of sync.
