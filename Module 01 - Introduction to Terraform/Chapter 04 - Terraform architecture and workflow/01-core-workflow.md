# Terraform Core Workflow

## Introduction

The **Terraform core workflow** is the set of steps you follow every time you use Terraform to create or change infrastructure. Understanding this workflow deeply is fundamental — everything else you learn about Terraform builds on top of it.

The workflow has four main commands:

1. `terraform init` — Initialize the working directory
2. `terraform plan` — Preview what changes will be made
3. `terraform apply` — Apply the changes to real infrastructure
4. `terraform destroy` — Remove all managed infrastructure

Every Terraform user, from beginner to expert, uses these same four commands daily. Let's explore each one in detail.

---

## The Workflow Diagram

```
Write .tf files
      |
      v
terraform init
(Download providers, set up backend)
      |
      v
terraform plan
(Compare desired state with current state, show changes)
      |
      v
Review the plan
(Is this what you expected? Yes or No?)
      |
      v
terraform apply
(Create, update, or destroy resources as planned)
      |
      v
Infrastructure is live
      |
      v
(When done)
      |
      v
terraform destroy
(Remove all resources)
```

---

## Step 1: Write Your Configuration

Before running any commands, you write your infrastructure configuration in `.tf` files. This is where you describe what you want to exist.

Create a directory for your project and add a `main.tf` file:

```bash
mkdir my-terraform-project
cd my-terraform-project
```

Create `main.tf`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-12345"

  tags = {
    Name        = "My Bucket"
    Environment = "dev"
  }
}
```

Now you are ready to run Terraform commands.

---

## Step 2: `terraform init`

### What it does

`terraform init` is always the first command you run in a new Terraform project or after making changes to provider requirements. It:

1. **Downloads providers** — reads your `required_providers` block and downloads the necessary provider plugins (e.g., the AWS provider) from the Terraform Registry
2. **Sets up the backend** — configures where Terraform will store its state file
3. **Downloads modules** — if you reference any modules in your configuration, it downloads them

### Running it

```bash
terraform init
```

### What you see

```
Initializing the backend...

Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.31.0...
- Installed hashicorp/aws v5.31.0 (signed by HashiCorp)

Terraform has created a lock file .terraform.lock.hcl to record the provider
selections it made above. Include this file in your version control repository
so that Terraform can guarantee to make the same selections by default when
you run "terraform init" in the future.

Terraform has been successfully initialized!
```

### What files are created

After `terraform init`, you will see a new directory in your project:

```
my-terraform-project/
├── main.tf
├── .terraform/                   # Created by init
│   └── providers/
│       └── registry.terraform.io/
│           └── hashicorp/
│               └── aws/
│                   └── 5.31.0/  # The downloaded provider
└── .terraform.lock.hcl           # Provider version lock file
```

The `.terraform` directory contains the downloaded provider binaries. The `.terraform.lock.hcl` file records the exact provider versions used — this ensures everyone on your team uses the same versions.

### When to run init

- When you first set up a new Terraform project
- When you add or change a provider in your configuration
- When you add a new module
- When you change the backend configuration
- When someone else checks out your project for the first time

---

## Step 3: `terraform plan`

### What it does

`terraform plan` compares what you have described in your `.tf` files (your **desired state**) with what actually exists in your cloud account (the **current state**), and shows you exactly what changes it would need to make.

It does NOT make any changes. It is purely a preview. Think of it as a "dry run."

### Running it

```bash
terraform plan
```

### What you see

When creating resources for the first time:

```
Terraform used the selected providers to generate the following execution plan.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_s3_bucket.my_bucket will be created
  + resource "aws_s3_bucket" "my_bucket" {
      + acceleration_status         = (known after apply)
      + acl                         = (known after apply)
      + arn                         = (known after apply)
      + bucket                      = "my-unique-bucket-12345"
      + bucket_domain_name          = (known after apply)
      + force_destroy               = false
      + id                          = (known after apply)
      + object_lock_enabled         = (known after apply)
      + region                      = (known after apply)
      + request_payer               = (known after apply)
      + tags                        = {
          + "Environment" = "dev"
          + "Name"        = "My Bucket"
        }
      + tags_all                    = {
          + "Environment" = "dev"
          + "Name"        = "My Bucket"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### Understanding the symbols

When reading a plan, these symbols tell you what will happen to each resource:

| Symbol | Meaning |
|---|---|
| `+` | Resource will be **created** |
| `-` | Resource will be **destroyed** |
| `~` | Resource will be **updated in-place** |
| `-/+` | Resource will be **destroyed and recreated** |

The last line always gives you a summary:
`Plan: 1 to add, 0 to change, 0 to destroy.`

This tells you the total number of changes at a glance.

### The three scenarios

**Scenario 1: First time (no infrastructure exists)**
- Terraform sees that none of the resources in your config exist yet
- The plan shows everything with `+` (to be created)

**Scenario 2: Infrastructure exists, config unchanged**
- Terraform sees that everything matches
- The plan shows: `No changes. Your infrastructure matches the configuration.`
- Nothing will happen when you apply

**Scenario 3: You changed the config**
- Terraform sees differences between your config and what exists
- The plan shows what needs to change (`~` for updates, `-/+` for replacements)

### Saving a plan

You can save a plan to a file and apply it later:

```bash
terraform plan -out=myplan.tfplan
terraform apply myplan.tfplan
```

This is useful in CI/CD pipelines where you want to review the plan before applying it.

---

## Step 4: `terraform apply`

### What it does

`terraform apply` actually makes the changes to your infrastructure. It:

1. Runs a `plan` internally (or uses a saved plan file)
2. Shows you the plan and asks for confirmation
3. Creates, updates, or destroys resources as planned
4. Updates the state file to reflect the new state

### Running it

```bash
terraform apply
```

### What you see

```
Terraform used the selected providers to generate the following execution plan.
[... same plan output as before ...]

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

You must type `yes` (exactly) and press Enter to proceed. Typing anything else cancels the operation.

After confirmation:

```
aws_s3_bucket.my_bucket: Creating...
aws_s3_bucket.my_bucket: Creation complete after 2s [id=my-unique-bucket-12345]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

### Auto-approve (for automation)

In CI/CD pipelines, you don't want Terraform waiting for a human to type "yes". Use the `-auto-approve` flag:

```bash
terraform apply -auto-approve
```

Use this carefully — it skips the confirmation prompt and applies immediately.

### Passing variables at apply time

```bash
terraform apply -var="environment=production"
terraform apply -var-file="production.tfvars"
```

---

## Step 5: `terraform destroy`

### What it does

`terraform destroy` removes all resources that Terraform manages in your current configuration. It asks for confirmation before making any changes.

### Running it

```bash
terraform destroy
```

### What you see

```
Terraform will perform the following actions:

  # aws_s3_bucket.my_bucket will be destroyed
  - resource "aws_s3_bucket" "my_bucket" {
      - bucket      = "my-unique-bucket-12345" -> null
      - id          = "my-unique-bucket-12345" -> null
      [...]
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

aws_s3_bucket.my_bucket: Destroying... [id=my-unique-bucket-12345]
aws_s3_bucket.my_bucket: Destruction complete after 1s

Destroy complete! Resources: 1 destroyed.
```

### When to use destroy

- After finishing experiments or learning exercises
- To tear down a staging or development environment to save costs
- When a project is complete and infrastructure is no longer needed

**Be careful with destroy in production!** Always review what will be destroyed before confirming.

---

## Other Useful Commands

### `terraform fmt`

Formats your `.tf` files to follow the standard Terraform style — consistent indentation, spacing, and layout.

```bash
terraform fmt
```

Run this before committing your code. Many teams enforce formatting in CI/CD.

### `terraform validate`

Checks your configuration files for syntax errors and logical problems — without connecting to any cloud provider.

```bash
terraform validate
```

Output when everything is correct:
```
Success! The configuration is valid.
```

### `terraform show`

Shows the current state in a human-readable format.

```bash
terraform show
```

### `terraform output`

Shows the values of output variables defined in your configuration.

```bash
terraform output
terraform output bucket_name  # Show a specific output
```

### `terraform refresh`

Updates the state file to match the actual current state of real infrastructure. Useful when resources have been changed outside of Terraform.

```bash
terraform refresh
```

Note: In modern Terraform, `terraform apply -refresh-only` is preferred over `terraform refresh`.

---

## A Complete Workflow Example

Here is a full workflow from start to finish, showing all the commands in sequence:

```bash
# 1. Create your project directory
mkdir terraform-demo
cd terraform-demo

# 2. Write your configuration files
# (Create main.tf, variables.tf, etc.)

# 3. Initialize Terraform
terraform init

# 4. Check for syntax errors
terraform validate

# 5. Format your code
terraform fmt

# 6. Preview changes
terraform plan

# 7. Apply changes
terraform apply

# 8. Check what was created
terraform show
terraform output

# 9. Make changes to your configuration
# (Edit main.tf, add more resources, etc.)

# 10. Preview the changes
terraform plan

# 11. Apply the changes
terraform apply

# 12. When done, clean up
terraform destroy
```

---

## The Workflow in a Team Environment

When working with a team, the workflow is slightly extended:

```
Developer A writes/changes configuration
        |
        v
git commit and push to feature branch
        |
        v
Pull Request is opened
        |
        v
CI/CD runs: terraform init + terraform plan
        |
        v
Team reviews the plan output in the PR
        |
        v
PR is approved and merged to main
        |
        v
CI/CD runs: terraform apply on the main branch
        |
        v
Infrastructure is updated
```

This workflow ensures that:
- Every infrastructure change is reviewed by the team
- The plan is visible and verifiable before anything is applied
- Changes are tracked in Git history

---

## Summary

- The Terraform workflow is: **init → plan → apply → (destroy when done)**
- `terraform init` downloads providers and sets up the backend — run it first always
- `terraform plan` shows a preview of changes — always review it before applying
- `terraform apply` makes the actual changes to infrastructure — requires confirmation
- `terraform destroy` removes all managed infrastructure — use carefully
- Additional commands like `fmt`, `validate`, `show`, and `output` support the core workflow
- In team environments, the workflow integrates with Git and CI/CD for safe, reviewed deployments

In the next file, we explore providers and the Terraform Registry — understanding how Terraform connects to different cloud services.
