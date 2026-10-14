# terraform state mv

## Introduction

`terraform state mv` moves a resource from one address to another within the state file. This is how you rename resources, move resources into or out of modules, or reorganize your Terraform code — all without destroying and recreating the actual infrastructure.

Without `terraform state mv`, renaming a resource in your `.tf` file would cause Terraform to destroy the old resource and create a new one. For many resources (like databases, EC2 instances, S3 buckets with data), this is unacceptable. `terraform state mv` solves this by telling Terraform "this resource is the same one, just with a new name."

---

## When You Need terraform state mv

### Scenario 1: Renaming a Resource

You have a resource named `web` and want to rename it to `web_server` for clarity:

**Old code:**
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

**New code:**
```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

Without `state mv`, `terraform plan` would show:
```
- aws_instance.web will be destroyed
+ aws_instance.web_server will be created
```

This would terminate and recreate your EC2 instance — losing data, causing downtime, and getting a new IP address.

**With `terraform state mv`:**

```bash
# Move the resource in state before running plan
terraform state mv aws_instance.web aws_instance.web_server

# Now terraform plan shows no changes
```

---

### Scenario 2: Moving a Resource into a Module

You started with resources at the root level and later refactored them into a module:

**Before refactor:**
```hcl
# main.tf — resources at root level
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}
```

**After refactor:**
```hcl
# main.tf — resources moved into a module
module "networking" {
  source     = "./modules/networking"
  cidr_block = "10.0.0.0/16"
}
```

Without `state mv`, Terraform would destroy the root-level resources and recreate them inside the module.

**With `terraform state mv`:**

```bash
# Move VPC from root to module
terraform state mv aws_vpc.main module.networking.aws_vpc.main

# Move subnet from root to module
terraform state mv aws_subnet.public module.networking.aws_subnet.public
```

---

### Scenario 3: Moving a Resource out of a Module

The reverse — moving resources from a module back to root level:

```bash
terraform state mv module.networking.aws_vpc.main aws_vpc.main
terraform state mv module.networking.aws_subnet.public aws_subnet.public
```

---

### Scenario 4: Renaming a Module

When you rename a module in your code:

**Before:**
```hcl
module "vpc" {
  source = "./modules/networking"
}
```

**After:**
```hcl
module "networking" {
  source = "./modules/networking"
}
```

Without `state mv`, all resources inside `module.vpc` would be destroyed and recreated as `module.networking`.

**With `terraform state mv`:**

```bash
# Move all resources from old module name to new module name
terraform state mv module.vpc module.networking
```

This single command moves the entire module and all its resources to the new address.

---

## Basic Syntax

```bash
terraform state mv <source_address> <destination_address>
```

```bash
# Rename a resource
terraform state mv aws_instance.web aws_instance.web_server

# Move into a module
terraform state mv aws_vpc.main module.vpc.aws_vpc.main

# Move out of a module
terraform state mv module.vpc.aws_vpc.main aws_vpc.main

# Rename a module (moves all resources inside)
terraform state mv module.old_name module.new_name

# Move a count resource to a different index
terraform state mv 'aws_instance.web[2]' 'aws_instance.web[0]'
```

---

## The Safe Workflow for Renaming Resources

Always follow this workflow to safely rename a resource:

**Step 1: Update your `.tf` code first**

Change the resource name in your configuration file:

```hcl
# Changed from "web" to "web_server"
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

**Step 2: Run terraform state mv**

```bash
terraform state mv aws_instance.web aws_instance.web_server
```

Output:
```
Move "aws_instance.web" to "aws_instance.web_server"
Successfully moved 1 object(s).
```

**Step 3: Verify with terraform plan**

```bash
terraform plan
```

Expected output:
```
No changes. Your infrastructure matches the configuration.
```

If the plan shows no changes, the move was successful. The resource was renamed in state to match your new code — Terraform recognizes them as the same resource.

---

## Moving Resources Between State Files

`terraform state mv` can also move resources from one state file to another. This is useful when splitting a large configuration into smaller ones.

```bash
# Move a resource to a different state file
terraform state mv \
  -state=source.tfstate \
  -state-out=destination.tfstate \
  aws_instance.web \
  aws_instance.web
```

Or when moving to a remote state:

```bash
# Pull remote state to local
terraform state pull > current.tfstate

# Move resource to another local state file
terraform state mv \
  -state=current.tfstate \
  -state-out=new_project.tfstate \
  aws_s3_bucket.shared \
  aws_s3_bucket.shared

# Push the updated state back
terraform state push current.tfstate
```

---

## Dry Run — Preview the Move

Before actually moving, use `-dry-run` to see what would happen:

```bash
terraform state mv -dry-run aws_instance.web aws_instance.web_server
```

Output:
```
~ aws_instance.web would be moved to aws_instance.web_server
```

Nothing is changed — this is just a preview. Remove `-dry-run` to execute the actual move.

---

## Moving count Resources

When renaming a resource that uses `count`, you may need to move each instance:

**Before:**
```hcl
resource "aws_instance" "web" {
  count = 3
  ...
}
```

**After:**
```hcl
resource "aws_instance" "app_server" {
  count = 3
  ...
}
```

You need to move each instance:

```bash
terraform state mv 'aws_instance.web[0]' 'aws_instance.app_server[0]'
terraform state mv 'aws_instance.web[1]' 'aws_instance.app_server[1]'
terraform state mv 'aws_instance.web[2]' 'aws_instance.app_server[2]'
```

Or use a script for many instances:

```bash
COUNT=3
for i in $(seq 0 $((COUNT-1))); do
  terraform state mv "aws_instance.web[$i]" "aws_instance.app_server[$i]"
done
```

---

## Moving for_each Resources

```bash
# Move individual for_each instances
terraform state mv \
  'aws_subnet.old["us-east-1a"]' \
  'aws_subnet.new["us-east-1a"]'

# If all keys are the same, moving the whole resource block works too
# (when renaming from "old" to "new")
```

---

## The `moved` Block — A Modern Alternative

Terraform 1.1 introduced the `moved` block as a declarative alternative to `terraform state mv`. Instead of running a CLI command, you declare the rename in your code:

```hcl
# Add this to your main.tf or a dedicated moves.tf file
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}
```

When you run `terraform plan`, Terraform automatically handles the state move. This approach is better for teams because:

- The rename is documented in code — visible in code review
- It runs automatically on `terraform plan` — no manual CLI commands
- It can be committed to Git so all team members apply the same moves

After the move has been applied everywhere, you can remove the `moved` block (or keep it for documentation).

The `moved` block supports the same types of moves:

```hcl
# Rename a resource
moved {
  from = aws_instance.web
  to   = aws_instance.web_server
}

# Move into a module
moved {
  from = aws_vpc.main
  to   = module.networking.aws_vpc.main
}

# Rename a module
moved {
  from = module.vpc
  to   = module.networking
}
```

---

## Common Mistakes and How to Avoid Them

### Mistake 1: Running plan before state mv

If you rename in code first and then run `terraform plan` without doing `state mv`:

```
- aws_instance.web will be destroyed
+ aws_instance.web_server will be created
```

Terraform does not know they are the same resource. It will destroy and recreate. Always do `state mv` before running plan when renaming.

### Mistake 2: Wrong address syntax

```bash
# WRONG — missing quotes around for_each address
terraform state mv aws_subnet.public[us-east-1a] aws_subnet.private[us-east-1a]

# CORRECT — quotes required for addresses with special characters
terraform state mv 'aws_subnet.public["us-east-1a"]' 'aws_subnet.private["us-east-1a"]'
```

### Mistake 3: Moving to an address that already exists

If the destination address already has a resource in state, the move will fail:

```
Error: Cannot move to aws_instance.web_server: that address is already in state
```

Remove the destination resource from state first, or choose a different destination address.

---

## Summary

- `terraform state mv` renames or moves a resource in state without destroying and recreating it
- Use it when renaming resources, moving resources into/out of modules, or renaming modules
- Always update your `.tf` code first, then run `state mv`, then verify with `terraform plan`
- After a successful move, `terraform plan` should show no changes
- Use `-dry-run` to preview moves before executing them
- For Terraform 1.1+, the `moved` block is the preferred declarative alternative to `terraform state mv`
- Always wrap addresses containing special characters (brackets, quotes) in single quotes

In the next file, we cover `terraform state rm` — removing resources from state without destroying them.
