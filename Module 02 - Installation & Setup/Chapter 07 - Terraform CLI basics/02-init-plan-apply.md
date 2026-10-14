# Terraform Init, Plan, and Apply — Deep Dive

## Introduction

In Module 1, we briefly introduced the Terraform workflow. Now that you have Terraform installed and your AWS provider configured, it is time to go much deeper into the three most important commands you will use every single day: `terraform init`, `terraform plan`, and `terraform apply`.

This file goes beyond the basics and covers flags, options, real-world scenarios, and important details that make the difference between a beginner and a confident Terraform user.

---

## Setting Up a Practice Project

Before we explore these commands in depth, let's create a small practice project to work with throughout this file.

Create a new directory and set up these files:

**`versions.tf`:**
```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

**`providers.tf`:**
```hcl
provider "aws" {
  region = var.aws_region
}
```

**`variables.tf`:**
```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}
```

**`main.tf`:**
```hcl
resource "aws_s3_bucket" "app_bucket" {
  bucket = "my-terraform-practice-${var.environment}-${random_id.suffix.hex}"
}

resource "random_id" "suffix" {
  byte_length = 4
}
```

---

## `terraform init` — Deep Dive

### What Init Does Step by Step

When you run `terraform init`, Terraform performs several tasks in order:

**1. Backend Initialization**
Terraform sets up the backend — the location where state will be stored. For local state, this is your current directory. For remote state (S3, Terraform Cloud), it configures the connection.

**2. Provider Plugin Installation**
Terraform reads your `required_providers` block, goes to the Terraform Registry, downloads the appropriate provider versions, and stores them in `.terraform/providers/`.

**3. Module Installation**
If your configuration references any modules (local or remote), Terraform downloads them and stores them in `.terraform/modules/`.

**4. Lock File Update**
Terraform records the exact provider versions selected in `.terraform.lock.hcl`.

---

### Running Init for the First Time

```bash
cd my-practice-project
terraform init
```

Full output:
```
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Finding hashicorp/random versions matching "~> 3.0"...
- Installing hashicorp/aws v5.31.0...
- Installed hashicorp/aws v5.31.0 (signed by HashiCorp)
- Installing hashicorp/random v3.6.0...
- Installed hashicorp/random v3.6.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!

You may now begin working with Terraform. Try running "terraform plan" to see
any changes that are required for your infrastructure. All Terraform commands
should now work.
```

---

### `terraform init` Flags You Should Know

**`-upgrade`** — Upgrade providers to the newest version within your constraints

```bash
terraform init -upgrade
```

Use this when you want to pick up a new provider version. Without `-upgrade`, Terraform respects the lock file and uses the exact versions recorded there.

**`-reconfigure`** — Reconfigure the backend, ignoring any existing configuration

```bash
terraform init -reconfigure
```

Useful when you change your backend configuration (for example, moving from local to S3 state) and want Terraform to adopt the new settings.

**`-migrate-state`** — Migrate state from the old backend to the new one

```bash
terraform init -migrate-state
```

When you change backends (for example from local to S3), this flag copies your existing state to the new location.

**`-backend=false`** — Skip backend initialization

```bash
terraform init -backend=false
```

Useful in CI/CD when you only want to validate or format code without setting up the state backend.

**`-var-file`** — Specify variable files during init (rare)

```bash
terraform init -backend-config="bucket=my-state-bucket"
```

Used to pass backend configuration values without hardcoding them.

---

### When to Re-run Init

You need to run `terraform init` again whenever:
- You add or change a provider in your configuration
- You add or change a module reference
- You change the backend configuration
- A team member clones the repository (they need to run it on their machine)
- You delete the `.terraform` directory

You do NOT need to run init again when:
- You edit resource configurations (add arguments, change values)
- You add new resource blocks using already-initialized providers
- You run any other Terraform command

---

## `terraform plan` — Deep Dive

### What Plan Does Step by Step

When you run `terraform plan`, Terraform:

1. **Reads all `.tf` files** in the current directory
2. **Reads the state file** (local or remote) to know what currently exists
3. **Calls the cloud API** (via the provider) to verify the actual current state of each resource
4. **Calculates the diff** between desired state (your `.tf` files) and actual state
5. **Displays the plan** showing exactly what will change

### Reading a Plan Output

Here is a detailed plan and how to read each part:

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create
  ~ update in-place
  - destroy
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # aws_s3_bucket.app_bucket will be created
  + resource "aws_s3_bucket" "app_bucket" {
      + acceleration_status         = (known after apply)
      + acl                         = (known after apply)
      + arn                         = (known after apply)
      + bucket                      = "my-terraform-practice-dev-a1b2c3d4"
      + bucket_domain_name          = (known after apply)
      + force_destroy               = false
      + id                          = (known after apply)
      + object_lock_enabled         = (known after apply)
      + region                      = (known after apply)
      + request_payer               = (known after apply)
      + tags_all                    = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**Understanding `(known after apply)`:**
Some attributes are only assigned by AWS after the resource is created — like the ARN, the region-specific domain name, and the resource ID. Terraform shows `(known after apply)` for these because it cannot know them until the resource actually exists.

**Understanding the action symbols:**

```
+ create         → New resource will be created
~ update         → Existing resource will be updated (no recreation)
- destroy        → Resource will be deleted
-/+ replace      → Resource will be destroyed and recreated
  (forces replacement) appears next to the argument that caused this
```

**The summary line:**
```
Plan: 2 to add, 1 to change, 0 to destroy.
```

Always check this line. It tells you at a glance how many resources will be created, changed, or destroyed.

---

### `terraform plan` Flags You Should Know

**`-out=filename`** — Save the plan to a file

```bash
terraform plan -out=myplan.tfplan
```

This saves the exact plan to a binary file. You can then apply this exact plan later:

```bash
terraform apply myplan.tfplan
```

This is important in CI/CD pipelines where you want to apply exactly the plan that was reviewed and approved, not a new plan that might be different.

**`-var`** — Pass a variable value directly

```bash
terraform plan -var="environment=production"
terraform plan -var="aws_region=eu-west-1" -var="environment=staging"
```

**`-var-file`** — Use a specific variable file

```bash
terraform plan -var-file="production.tfvars"
```

**`-target`** — Plan only specific resources

```bash
terraform plan -target=aws_s3_bucket.app_bucket
```

Useful during debugging when you only want to see the plan for one specific resource. However, avoid using `-target` in normal workflows as it can leave your infrastructure in a partially-configured state.

**`-refresh=false`** — Skip refreshing real state

```bash
terraform plan -refresh=false
```

Skips the step where Terraform queries the cloud API to verify current state. Faster but may miss drift. Rarely needed in practice.

**`-detailed-exitcode`** — Return different exit codes based on plan results

```bash
terraform plan -detailed-exitcode
echo "Exit code: $?"
```

Exit codes:
- `0` — Plan succeeded, no changes needed
- `1` — Plan errored
- `2` — Plan succeeded, changes exist

Very useful in CI/CD scripts where you want to behave differently based on whether changes exist.

---

### Three Key Plan Scenarios

**Scenario 1: First apply — everything is new**
```
Plan: 3 to add, 0 to change, 0 to destroy.
```

**Scenario 2: Running plan with no changes**
```
No changes. Your infrastructure matches the configuration.

Terraform has compared your real infrastructure against your configuration
and found no differences, so no changes are needed.
```

This is actually the ideal state — it means your real infrastructure exactly matches your code.

**Scenario 3: Some changes needed**
```
  # aws_s3_bucket.app_bucket will be updated in-place
  ~ resource "aws_s3_bucket" "app_bucket" {
        id     = "my-terraform-practice-dev-a1b2c3d4"
      ~ tags   = {
          + "Environment" = "dev"
            # (1 unchanged element hidden)
        }
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

---

## `terraform apply` — Deep Dive

### What Apply Does Step by Step

1. **Generates an execution plan** (same as `terraform plan`)
2. **Shows the plan and asks for confirmation**
3. **Creates, updates, or destroys resources** in the correct dependency order
4. **Updates the state file** after each resource is successfully managed

### Running Apply

```bash
terraform apply
```

Terraform shows the plan and then prompts:

```
Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

Type `yes` and press Enter to proceed.

### Apply Output While Running

As Terraform creates resources, it shows progress:

```
aws_s3_bucket.app_bucket: Creating...
aws_s3_bucket.app_bucket: Creation complete after 2s [id=my-terraform-practice-dev-a1b2c3d4]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

bucket_name = "my-terraform-practice-dev-a1b2c3d4"
```

### `terraform apply` Flags You Should Know

**`-auto-approve`** — Skip the confirmation prompt

```bash
terraform apply -auto-approve
```

Use this in CI/CD pipelines. Be careful using it manually — it applies changes immediately without asking.

**`-var` and `-var-file`** — Same as plan

```bash
terraform apply -var="environment=production"
terraform apply -var-file="production.tfvars"
```

**Applying a saved plan**

```bash
# First, save the plan
terraform plan -out=prod-plan.tfplan

# Review it, get approval from your team, then apply exactly that plan
terraform apply prod-plan.tfplan
```

When applying a saved plan file, Terraform does NOT ask for confirmation — it applies immediately. The assumption is you already reviewed and approved the plan.

**`-target`** — Apply only specific resources

```bash
terraform apply -target=aws_s3_bucket.app_bucket
```

Like with plan, use this sparingly. It is useful for fixing a specific broken resource but should not be used in normal workflows.

**`-parallelism`** — Control how many resources are created at once

```bash
terraform apply -parallelism=20
```

By default, Terraform creates up to 10 resources simultaneously. Increase this for large configurations, but be aware that some APIs have rate limits.

**`-refresh-only`** — Update state to match real infrastructure without making changes

```bash
terraform apply -refresh-only
```

This updates the state file to reflect the current real state of your infrastructure — useful after someone makes manual changes outside of Terraform.

---

### What Happens When Apply Fails

If a resource fails to create (for example, because of an IAM permission issue or an API error):

1. Terraform stops creating new resources
2. Already-created resources remain in place and are recorded in state
3. Terraform shows an error message explaining what failed

The state file will be partially updated — resources that were successfully created are recorded, failed ones are not.

**Fix the error and run `terraform apply` again.** Terraform will attempt to create the failed resources again without touching the ones that already succeeded (because they are in state).

---

## A Complete Real-World Apply Example

Here is what a full real-world apply session looks like for a new infrastructure deployment:

```bash
# Navigate to your Terraform project
cd ~/projects/my-app-infra

# Initialize (first time or after changes)
terraform init

# Validate configuration
terraform validate

# Format code
terraform fmt

# Preview changes
terraform plan -out=deploy.tfplan

# Review the plan carefully...
# If satisfied, apply
terraform apply deploy.tfplan

# Verify outputs
terraform output

# Check what's in state
terraform state list
```

---

## Summary

- `terraform init` downloads providers and modules, sets up the backend — run it first, and re-run it when providers or modules change
- `terraform plan` previews changes without making them — always review it before applying
- `terraform apply` makes changes to real infrastructure — requires confirmation or `-auto-approve`
- Key plan symbols: `+` create, `~` update, `-` destroy, `-/+` replace
- Save plans with `-out` for use in CI/CD approval workflows
- Use `-var` and `-var-file` to pass variable values at runtime
- When apply fails, fix the issue and run apply again — already-created resources are safe

In the next file, we cover `terraform destroy` — how to safely remove infrastructure that Terraform manages.
