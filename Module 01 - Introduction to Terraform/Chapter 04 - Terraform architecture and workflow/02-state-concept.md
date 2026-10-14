# Terraform State — The Concept

## What is Terraform State?

**Terraform state** is how Terraform keeps track of the infrastructure it has created and manages. It is Terraform's memory.

When you run `terraform apply` and create an AWS EC2 instance, Terraform needs to remember:
- That this EC2 instance exists
- What its ID is (e.g., `i-0a1b2c3d4e5f67890`)
- What its current configuration is
- How it relates to other resources in your configuration

Without state, every time you run Terraform, it would have no idea what resources already exist. It couldn't tell the difference between "create this new resource" and "this resource already exists, don't touch it."

Terraform stores all this information in a **state file** — by default a file called `terraform.tfstate` in your project directory.

---

## What Does the State File Look Like?

The state file is a JSON file. You will never need to edit it manually, but understanding its structure helps you understand what Terraform is tracking.

Here is a simplified example of what a state file looks like after creating an S3 bucket:

```json
{
  "version": 4,
  "terraform_version": "1.6.0",
  "serial": 3,
  "lineage": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_s3_bucket",
      "name": "my_bucket",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "id": "my-unique-bucket-12345",
            "bucket": "my-unique-bucket-12345",
            "arn": "arn:aws:s3:::my-unique-bucket-12345",
            "region": "us-east-1",
            "tags": {
              "Environment": "dev",
              "Name": "My Bucket"
            }
          }
        }
      ]
    }
  ]
}
```

The state file records:
- The resource type (`aws_s3_bucket`)
- The resource name in your config (`my_bucket`)
- All the attributes of the resource, including the ones that were assigned by AWS (like `arn` and `region`)

---

## Why State is Essential

### 1. Mapping Configuration to Real Resources

Your Terraform configuration refers to resources by the names you give them:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

The name `web_server` is just the local name you use in your Terraform code. In AWS, this instance has an actual ID like `i-0a1b2c3d4e5f67890`.

The state file is the bridge that maps:
- `aws_instance.web_server` (your Terraform name) → `i-0a1b2c3d4e5f67890` (the real AWS resource ID)

Without this mapping, Terraform would have no way to know which real resource corresponds to which resource in your code.

---

### 2. Tracking Metadata

Some resource attributes are only known after the resource is created. For example:
- An EC2 instance's public IP address is assigned by AWS after creation
- An RDS database's endpoint is only known after the database is created
- An S3 bucket's ARN follows a pattern but is confirmed after creation

The state file stores all these attributes after they are known, so Terraform can reference them in other resources without making API calls every time.

---

### 3. Calculating the Diff (What Needs to Change)

When you run `terraform plan`, Terraform needs to calculate what changes need to be made. It does this by:

1. Reading your current `.tf` configuration (your **desired state**)
2. Reading the state file (the **last known state**)
3. Optionally, calling the cloud API to verify the real state (the **actual state**)
4. Calculating the difference between desired and actual

```
Desired State (your .tf files)
          vs
Actual State (cloud API + state file)
          =
The Plan (what needs to change)
```

Without the state file, step 2 would be missing, and Terraform would have to query every possible resource in your cloud account to figure out what exists. This would be extremely slow and would also require much broader API permissions.

---

### 4. Performance

The state file also acts as a cache. Instead of querying the AWS API for the current state of every resource every time you run a plan, Terraform can read the state file directly. This makes plans much faster.

When Terraform needs to verify real state (for example, to detect drift), it can query just the specific resources it knows about from the state file, rather than scanning your entire AWS account.

---

## Local State vs Remote State

### Local State (Default)

By default, Terraform stores the state file locally in your project directory:

```
my-terraform-project/
├── main.tf
├── variables.tf
├── terraform.tfstate          ← state file
├── terraform.tfstate.backup   ← backup of previous state
└── .terraform/
```

**Advantages of local state:**
- Simple — no setup required
- Works for individual projects and learning

**Disadvantages of local state:**
- If two people run Terraform at the same time, they might corrupt the state (race condition)
- The state file must be shared somehow for team collaboration (e.g., checking it into Git — which is a bad practice due to sensitive data)
- If your laptop crashes or the file is deleted, your state is lost

**Local state is fine for:**
- Learning Terraform
- Personal projects
- Single-developer setups

---

### Remote State

For team environments, you should always use **remote state** — storing the state file in a shared location that everyone on the team can access.

The most common remote state backend for AWS is **S3 + DynamoDB**:
- The state file is stored in an S3 bucket
- DynamoDB is used for state locking (prevents two people from running Terraform at the same time)

Configuring remote state in S3:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

After adding this backend configuration and running `terraform init`, Terraform will:
1. Store the state file in the S3 bucket instead of locally
2. Use the DynamoDB table to lock state during operations

**Advantages of remote state:**
- Multiple team members can safely collaborate
- State is backed up automatically (S3 versioning)
- State locking prevents corruption
- Sensitive data in state is encrypted
- State can be shared between different Terraform configurations

We cover remote state configuration in detail in Module 4.

---

## State Locking

**State locking** is a mechanism that prevents multiple people from running Terraform at the same time against the same state.

Imagine what would happen without locking:
1. Developer A runs `terraform apply` to add a new server
2. At the same time, Developer B runs `terraform apply` to remove an old server
3. Both operations read the same state file at the same time
4. Both write different changes back to the state file
5. The state file is now corrupted — neither change is recorded correctly

State locking prevents this by:
1. Developer A starts `terraform apply` — acquires a lock on the state
2. Developer B tries to run `terraform apply` — Terraform sees the lock and waits (or fails with a message)
3. Developer A finishes — releases the lock
4. Developer B can now run their operation safely

When using the S3 backend, DynamoDB handles the locking. When using Terraform Cloud, locking is built in.

---

## Sensitive Data in State

One important thing to be aware of: **the state file can contain sensitive information**.

When you create resources like:
- RDS database (the master password might be stored)
- AWS IAM access keys
- Any resource where you pass a password or secret

Those values might appear in the state file in plain text.

This is why:
- **Never commit state files to Git** — always add `*.tfstate` and `*.tfstate.backup` to your `.gitignore`
- **Always encrypt remote state** — use `encrypt = true` in the S3 backend configuration
- **Use AWS Secrets Manager or HashiCorp Vault** for actual secrets instead of passing them directly to resources

---

## The State File is the Source of Truth

An important thing to understand about Terraform state:

**The state file is Terraform's source of truth about what infrastructure it manages — NOT your `.tf` files.**

Your `.tf` files describe what you WANT. The state file records what Terraform HAS CREATED and is currently managing.

This distinction matters in several situations:

**Situation 1: Resource created outside Terraform**
If someone creates an S3 bucket manually in the AWS Console, Terraform doesn't know about it. It's not in the state file. Terraform will not manage or touch that bucket.

**Situation 2: Resource deleted outside Terraform**
If someone deletes an EC2 instance directly in the AWS Console, the state file still says it exists. Running `terraform plan` will detect the discrepancy and show that it needs to recreate the instance.

**Situation 3: Importing existing resources**
If you want Terraform to start managing a resource that was created manually, you use `terraform import` to add it to the state file. After that, Terraform knows about it and can manage it.

---

## Common State Operations

### `terraform state list`

Shows all resources currently tracked in state:

```bash
terraform state list
```

Output:
```
aws_instance.web_server
aws_s3_bucket.app_storage
aws_security_group.web_sg
aws_vpc.main
```

### `terraform state show`

Shows the full details of a specific resource in state:

```bash
terraform state show aws_instance.web_server
```

Output:
```
# aws_instance.web_server:
resource "aws_instance" "web_server" {
    ami                          = "ami-0c55b159cbfafe1f0"
    arn                          = "arn:aws:ec2:us-east-1:123456789:instance/i-0a1b2c3d4e5f67890"
    id                           = "i-0a1b2c3d4e5f67890"
    instance_type                = "t2.micro"
    public_ip                    = "54.23.45.67"
    [...]
}
```

### `terraform state mv`

Moves a resource from one state address to another. Useful when refactoring:

```bash
# Rename a resource
terraform state mv aws_instance.web_server aws_instance.app_server
```

### `terraform state rm`

Removes a resource from state without destroying it. Useful when you want Terraform to "forget" about a resource without deleting it:

```bash
terraform state rm aws_instance.web_server
```

After this, Terraform no longer manages that instance — the instance still exists in AWS, but Terraform won't touch it.

---

## What Happens When State is Lost?

If your state file is lost or corrupted:

1. Terraform no longer knows which real resources it created
2. Running `terraform apply` might try to create duplicate resources
3. Running `terraform destroy` will fail because it doesn't know what to destroy

**This is why remote state with backups is so important in production.**

If you do lose your state, you have two options:
1. **Manually import** each resource back into state using `terraform import`
2. **Recreate** the state from scratch by importing everything (time-consuming)

This is a painful experience that emphasizes why proper state management is critical.

---

## State Backends Overview

Here is a summary of common state backends:

| Backend | Best For | Cost |
|---|---|---|
| **Local** | Learning, personal projects | Free |
| **S3 + DynamoDB** | AWS teams, most common | Minimal AWS costs |
| **Terraform Cloud** | Teams wanting managed experience | Free tier available |
| **Azure Blob Storage** | Azure teams | Minimal Azure costs |
| **Google Cloud Storage** | GCP teams | Minimal GCP costs |

We will configure S3 + DynamoDB remote state in Module 4 with a hands-on example.

---

## Summary

- Terraform state is how Terraform tracks what infrastructure it manages — stored in a `terraform.tfstate` file
- State maps your Terraform resource names to real cloud resource IDs and attributes
- State is used to calculate what changes need to be made during `terraform plan`
- **Local state** is simple but not suitable for teams
- **Remote state** (S3, Terraform Cloud, etc.) should always be used for team environments
- State locking prevents multiple people from corrupting state by running Terraform simultaneously
- The state file can contain sensitive data — never commit it to Git and always encrypt remote state
- Common state commands: `state list`, `state show`, `state mv`, `state rm`
- Losing your state file is painful — always set up proper remote state with backups

This completes Module 1! You now have a solid understanding of:
- What Infrastructure as Code is and why it matters
- What Terraform is and how it works
- How HCL is written and structured
- How Terraform compares to Ansible, CloudFormation, and Pulumi
- The core Terraform workflow (init → plan → apply → destroy)
- How providers work and how to use the Terraform Registry
- What Terraform state is and why it is critical

In Module 2, we move from concepts to hands-on — installing Terraform, setting up the AWS provider, and running your first real Terraform configuration.
