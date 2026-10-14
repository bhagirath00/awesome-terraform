# State vs Real Infrastructure

## Introduction

Terraform state is a snapshot of what Terraform believes exists in your cloud account. Real infrastructure is what actually exists in AWS right now. Most of the time, these two things match perfectly. But sometimes they get out of sync — and when they do, you need to understand why it happened and how to fix it.

This file explains the relationship between state and real infrastructure, what happens when they diverge, how Terraform detects and handles this, and the practical steps to bring them back into sync.

---

## The Three States of Truth

When working with Terraform, there are always three sources of truth you need to think about:

```
1. Desired State        →  Your .tf configuration files
                            "What I want to exist"

2. Recorded State       →  terraform.tfstate file
                            "What Terraform thinks exists"

3. Actual State         →  Real AWS infrastructure
                            "What actually exists right now"
```

Terraform's job is to make **Actual State** match **Desired State**. It uses **Recorded State** as a cache and reference point to do this efficiently.

In a healthy Terraform workflow, all three are in sync:
- Your config says "I want an EC2 instance of type t2.micro"
- State says "I created an EC2 instance i-0a1b2c3 of type t2.micro"
- AWS has an EC2 instance i-0a1b2c3 of type t2.micro running

When any of these diverge, you have a problem.

---

## What is Drift?

**Drift** is when the real infrastructure (Actual State) no longer matches what Terraform's state file (Recorded State) says it should be.

Drift happens when someone makes changes to infrastructure **outside of Terraform** — directly in the AWS Console, using the AWS CLI, or through another tool — without updating the Terraform configuration and state.

### Common Causes of Drift

**1. Manual changes via AWS Console**

An engineer logs into the AWS Console and:
- Changes an EC2 instance type from `t2.micro` to `t3.small` (scaling up urgently)
- Adds a new security group rule to allow a port
- Changes an S3 bucket policy
- Modifies an RDS parameter

Terraform's state still says the old configuration. The real infrastructure now differs.

**2. AWS Auto-Scaling or Auto-Healing**

AWS services sometimes change resource attributes automatically:
- Auto Scaling changes the number of EC2 instances
- RDS applies a pending modification after a maintenance window
- AWS rotates credentials or updates certificates

**3. Direct AWS CLI changes**

```bash
# Someone runs this outside of Terraform
aws ec2 modify-instance-attribute \
  --instance-id i-0a1b2c3d4e5f67890 \
  --instance-type '{"Value": "t3.medium"}'
```

Terraform state still records `t2.micro`. Real AWS has `t3.medium`.

**4. Resource deleted outside Terraform**

- An EC2 instance is terminated via the console
- A colleague accidentally deletes an S3 bucket
- AWS terminates a spot instance

Terraform state says the resource exists. AWS says it does not.

**5. Another tool manages the same resource**

If both Terraform and a separate deployment script try to manage the same resource, they can step on each other.

---

## How Terraform Detects Drift

When you run `terraform plan`, Terraform does not just compare your config to the state file. It also calls the real AWS API to check the current state of each resource.

The process:

```
Step 1: Read .tf files → desired state
Step 2: Read state file → last known state  
Step 3: Call AWS API for each resource → actual state
Step 4: Compare desired state vs actual state
Step 5: Output the plan showing what needs to change
```

This is why `terraform plan` sometimes shows changes even when you haven't changed your `.tf` files — the real infrastructure has drifted.

### Example: Detecting a Manual Change

Your state says:
```json
"instance_type": "t2.micro"
```

Someone manually changed it to `t3.small` in the console.

Running `terraform plan` shows:
```
  # aws_instance.web_server will be updated in-place
  ~ resource "aws_instance" "web_server" {
        id            = "i-0a1b2c3d4e5f67890"
      ~ instance_type = "t3.small" -> "t2.micro"
        tags          = { ... }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

Terraform detected that the real instance type (`t3.small`) does not match what your config says (`t2.micro`) and plans to revert it.

---

## How to Handle Drift

When Terraform detects drift, you have three choices:

### Option 1: Apply to Revert the Drift (Most Common)

Run `terraform apply` to bring the real infrastructure back to match your configuration.

```bash
terraform apply
```

This is the correct choice when:
- The manual change was a mistake
- The change should not have been made
- You want to enforce the configuration as the source of truth

After apply, Terraform reverts the instance type back to `t2.micro` and updates state to match.

---

### Option 2: Update Configuration to Accept the Change

If the manual change was intentional and correct, update your `.tf` files to match what now exists.

```hcl
# Update your config to accept the change
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.small"   # updated to match what's running
}
```

Then run `terraform plan` — it should show no changes.

This is the correct choice when:
- Someone made a valid emergency change that should be kept
- You realize the original configuration was wrong

---

### Option 3: Refresh State Only (Refresh-Only Apply)

Sometimes you want to update the state file to match real infrastructure without making any resource changes. Use `terraform apply -refresh-only`:

```bash
terraform apply -refresh-only
```

This:
1. Queries AWS for the current state of all managed resources
2. Updates the state file to reflect what actually exists
3. Does NOT change any real infrastructure

After a refresh-only apply, your state file matches reality. Now you can decide what to do with any differences between your config and the updated state.

---

## The Resource Deleted Outside Terraform Scenario

One of the most important drift scenarios to understand is when a resource is deleted outside Terraform.

**Scenario:**
1. Terraform creates an S3 bucket and records it in state
2. Someone accidentally deletes the bucket through the AWS Console
3. The bucket no longer exists in AWS
4. Terraform's state still says it exists

**What happens when you run `terraform plan`:**

```
  # aws_s3_bucket.app_data will be created
  + resource "aws_s3_bucket" "app_data" {
      + bucket        = "myapp-production-data-2024"
      + id            = (known after apply)
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

Terraform detects the bucket is gone and plans to recreate it. Running `terraform apply` will recreate the bucket — restoring it as your configuration describes.

This is Terraform's "reconciliation loop" behavior. No matter what happens to the real infrastructure, `terraform apply` will always bring it back to match your desired state.

---

## Resources Created Outside Terraform

The opposite problem: a resource exists in AWS that Terraform knows nothing about.

**Scenario:**
An engineer creates an RDS database manually via the AWS Console. Your Terraform configuration does not have this database. Terraform's state has no record of it.

**What happens:**
- `terraform plan` shows no changes — Terraform doesn't see resources it doesn't manage
- `terraform apply` does not touch the manually created database
- `terraform destroy` does NOT destroy it — Terraform only destroys what it manages

This database is an "unmanaged resource." Terraform is completely blind to it.

**If you want Terraform to manage it:** Use `terraform import` to bring it into state. After importing, Terraform knows about it and can manage its lifecycle.

```bash
# Import an existing RDS instance into Terraform management
terraform import aws_db_instance.app myapp-production-db
```

---

## Drift in the Real World — What DevOps Teams Do

### Prevention: Restrict Console Access

The best way to handle drift is to prevent it. Many mature DevOps teams:

- Restrict IAM permissions so engineers cannot make manual changes to production infrastructure
- Require all infrastructure changes to go through Terraform and pull requests
- Use SCPs (Service Control Policies) in AWS Organizations to enforce this

---

### Detection: Regular Drift Checks

Run `terraform plan` regularly (not just when you want to make changes) to detect drift early:

```bash
# In CI/CD — run a plan daily and alert if drift is detected
terraform plan -detailed-exitcode
# Exit code 2 means changes exist (drift detected)
```

---

### Remediation: Decide and Act Quickly

When drift is detected:
1. Understand what changed and why
2. Decide: revert (apply) or accept (update config)
3. Act quickly — the longer drift persists, the harder it is to reconcile

---

## State Refresh During Plan

By default, every `terraform plan` refreshes state by querying AWS. For large configurations with hundreds of resources, this can be slow.

You can skip the refresh with `-refresh=false`:

```bash
terraform plan -refresh=false
```

This uses the cached state without querying AWS. It is faster but may miss recent drift.

Use `-refresh=false` when:
- You know nothing has changed outside Terraform
- You need a fast plan for CI/CD validation purposes
- You are working with a large number of resources

Use the default (with refresh) when:
- You want to detect drift
- You are about to apply changes
- Something seems wrong and you want the most accurate picture

---

## A Complete Drift Scenario Walkthrough

Here is a full realistic example of drift and resolution:

**Day 1:** Terraform creates a security group with port 80 open.
```
State: aws_security_group.web — port 80 allowed
AWS:   Security group sg-abc123 — port 80 allowed
Config: port 80 open
✓ All three in sync
```

**Day 3:** An engineer adds port 3000 directly in the AWS Console for debugging.
```
State: aws_security_group.web — port 80 allowed
AWS:   Security group sg-abc123 — ports 80 AND 3000 allowed
Config: port 80 open
✗ Drift: AWS has port 3000, state and config do not
```

**Day 4:** Team runs `terraform plan`:
```
~ aws_security_group.web will be updated in-place
  - ingress port 3000 will be removed
Plan: 0 to add, 1 to change, 0 to destroy.
```

**Decision:** The team decides port 3000 was a temporary debug port that should be removed.

**Resolution:**
```bash
terraform apply   # removes port 3000, restores to config
```

**Result:**
```
State: aws_security_group.web — port 80 allowed
AWS:   Security group sg-abc123 — port 80 allowed
Config: port 80 open
✓ All three back in sync
```

---

## Summary

- There are three states of truth: desired state (config), recorded state (state file), and actual state (real AWS)
- **Drift** is when actual infrastructure diverges from recorded state — caused by manual changes, AWS automation, or other tools
- Terraform detects drift by querying the real AWS API during `terraform plan`
- When drift is detected, you have three choices: apply to revert, update config to accept, or refresh-only to sync state
- Resources deleted outside Terraform will be recreated on next apply
- Resources created outside Terraform are completely invisible to Terraform — use `terraform import` to bring them under management
- Prevent drift by restricting console access and requiring all changes to go through Terraform
- Detect drift early by running `terraform plan` regularly in CI/CD
- Use `-refresh=false` for faster plans when you know no drift has occurred

This completes Chapter 14. You now have a thorough understanding of what Terraform state is, how the state file is structured, and the critical relationship between state and real infrastructure.

In Chapter 15, we move to local vs remote state — understanding when local state is fine and when you must switch to remote state for team environments.
