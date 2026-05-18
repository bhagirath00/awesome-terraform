# What is Terraform State?

## Introduction

Terraform state is the mechanism Terraform uses to keep track of the infrastructure it manages. It is essentially Terraform's memory — without it, Terraform would have no idea what resources it has already created, what their current configuration is, or how they relate to each other.

Every time you run `terraform apply`, Terraform reads the state, compares it with your configuration, figures out what needs to change, and then updates the state after making those changes.

Understanding state deeply is one of the most important skills for any Terraform user. Most Terraform problems that engineers face in real-world projects — corruption, drift, conflicts, accidental deletions — are state-related. This chapter gives you that deep understanding.

---

## Why Terraform Needs State

Let's think about what Terraform needs to do its job:

1. **Know what already exists** — If you run `terraform apply` twice, Terraform should not create the same resource twice. It needs to know the resource already exists.

2. **Track resource IDs** — When AWS creates an EC2 instance, it assigns it an ID like `i-0a1b2c3d4e5f67890`. Your Terraform code refers to the resource as `aws_instance.web_server`. State is the bridge between your name and AWS's ID.

3. **Track attribute values** — Some attributes are only known after creation (public IP, ARN, DNS name). State stores these so Terraform can use them in subsequent plans.

4. **Understand dependencies** — When you destroy infrastructure, Terraform needs to know the order. State helps Terraform understand the full picture of what was created and in what relationship.

5. **Detect drift** — If someone changes a resource outside of Terraform (via the AWS console), state helps Terraform detect that the real infrastructure no longer matches the desired configuration.

Without state, Terraform would be blind. It would have to query every single resource in your entire AWS account every time — which would be extremely slow and require very broad permissions.

---

## State as a Database

Think of Terraform state as a **database** that Terraform maintains about your infrastructure. Like any database, it:

- Stores records (one per managed resource)
- Gets updated when records change (after every apply)
- Gets read before making decisions (during plan)
- Must be kept consistent (never manually edit it)
- Must be backed up (losing it is a serious problem)

The "database" is stored in a JSON file called `terraform.tfstate`. But the important thing is not the file format — it is what the state represents: a complete snapshot of all the infrastructure Terraform manages.

---

## What State Tracks

For every resource Terraform manages, state records:

### Resource Identity

```json
{
  "type": "aws_s3_bucket",
  "name": "app_bucket",
  "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]"
}
```

This tells Terraform: "There is a resource of type `aws_s3_bucket` known locally as `app_bucket`, managed by the AWS provider."

### Resource Attributes

```json
{
  "attributes": {
    "id": "myapp-production-data",
    "bucket": "myapp-production-data",
    "arn": "arn:aws:s3:::myapp-production-data",
    "region": "us-east-1",
    "hosted_zone_id": "Z3AQBSTGFYJSTF",
    "bucket_domain_name": "myapp-production-data.s3.amazonaws.com",
    "tags": {
      "Environment": "production",
      "ManagedBy": "Terraform"
    }
  }
}
```

Every attribute of the resource is stored — both the ones you set (like `bucket` and `tags`) and the ones AWS assigned after creation (like `arn`, `region`, `hosted_zone_id`).

### Dependencies

State also tracks which resources depend on which. This is used to determine the correct order for creating and destroying resources.

---

## The State Lifecycle

Here is exactly what happens with state during each Terraform operation:

### During `terraform plan`

1. Terraform reads your `.tf` configuration files (desired state)
2. Terraform reads the state file (last known state)
3. Terraform calls the cloud API to verify current real state of each resource
4. Terraform compares desired state vs real state
5. Terraform outputs the diff — what needs to be created, changed, or destroyed

### During `terraform apply`

1. Terraform runs the plan again
2. Terraform executes the changes one by one
3. After each resource is successfully created/updated/destroyed, Terraform immediately writes that change to the state file
4. If apply fails halfway, the state file reflects what was completed before the failure

### During `terraform destroy`

1. Terraform reads state to know what resources exist
2. Terraform destroys each resource in reverse dependency order
3. After each destruction, Terraform removes that resource from state
4. When complete, state is empty (but the file still exists)

---

## State is Written Immediately

An important detail: Terraform writes state **immediately after each resource operation**, not at the end of the entire apply.

This means:

- If `terraform apply` creates 5 resources and fails on the 6th, the state file reflects the 5 successfully created resources
- Running `terraform apply` again will pick up from where it failed
- You never lose track of what was created, even on partial failures

This is one of the safety features that makes Terraform reliable for production use.

---

## State vs Configuration

It is important to understand that **state and configuration are different things**:

| | Configuration (`.tf` files) | State (`terraform.tfstate`) |
|---|---|---|
| **What it represents** | What you WANT to exist | What Terraform HAS created |
| **Written by** | You (the engineer) | Terraform (automatically) |
| **When it changes** | When you edit files | After every `terraform apply` |
| **Contains** | Desired resource definitions | Actual resource IDs and attributes |
| **Should be in Git?** | Yes | No (contains sensitive data) |

Your `.tf` files describe the future. The state file describes the past and present.

---

## What Happens Without State

To really appreciate state, consider what would happen without it:

**Scenario:** You create an EC2 instance with Terraform, then run `terraform apply` again without changing any configuration.

**Without state:**
- Terraform has no memory of creating the instance
- It sees your configuration says "I want an EC2 instance"
- It checks AWS and finds... an EC2 instance. But which one is it?
- There might be dozens of EC2 instances in your account from other projects
- Terraform cannot tell which one belongs to this configuration
- It might try to create another one, leading to duplicate resources

**With state:**
- Terraform reads the state file: "I previously created instance `i-0a1b2c3d4e5f67890`"
- It checks AWS: "Instance `i-0a1b2c3d4e5f67890` exists and matches the configuration"
- It outputs: "No changes needed"

State is what makes Terraform intelligent rather than just a resource creator.

---

## State File Location

By default, Terraform stores state in a file called `terraform.tfstate` in your working directory:

```
my-terraform-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfstate          ← state file (created after first apply)
├── terraform.tfstate.backup   ← backup of previous state
└── .terraform/
```

Terraform also maintains a `terraform.tfstate.backup` file — this is automatically created before every apply and contains the previous state. If something goes wrong, you can restore from the backup.

---

## Never Manually Edit State

This is one of the most important rules in Terraform:

**Never manually edit the `terraform.tfstate` file.**

The state file is a structured JSON file and it might look editable — but it contains checksums, dependency graphs, and internal references that are easy to break if you edit manually.

If you need to make changes to state (move a resource, remove a resource from tracking, etc.), use the proper Terraform state commands:

```bash
terraform state list    # see what's in state
terraform state show    # inspect a resource in state
terraform state mv      # move/rename a resource in state
terraform state rm      # remove a resource from state
terraform import        # add an existing resource to state
```

We cover all these commands in detail in Chapter 17.

---

## State and Teams

State becomes especially important — and complicated — when working in a team.

**Problem:** If two engineers both have the state file locally on their laptops and both run `terraform apply` at the same time:

1. Both read the same state file at the start
2. Both make different changes to the infrastructure
3. Both write their updated state file
4. One engineer's changes overwrite the other's
5. The state file is now inconsistent with real infrastructure

**Solution:** Remote state (covered in Chapter 16). Store the state file in a shared location (like AWS S3) with locking (using DynamoDB) so only one person can apply at a time.

---

## Summary

- Terraform state is Terraform's memory — it tracks every resource Terraform manages
- State records resource types, local names, real cloud IDs, and all attribute values
- State is stored in `terraform.tfstate` by default in your working directory
- Terraform reads state during `plan` and updates it immediately after each resource operation during `apply`
- State is different from configuration — config is what you want, state is what exists
- Without state, Terraform cannot track resources, detect drift, or manage dependencies correctly
- Never manually edit the state file — use Terraform state commands instead
- Local state is fine for solo projects but remote state is required for team environments

In the next file, we go deeper into the `terraform.tfstate` file itself — what its structure looks like and what every field means.
