# Terraform Destroy

## Introduction

`terraform destroy` is the command you use to remove all infrastructure that Terraform is managing. It is essentially the reverse of `terraform apply` — instead of creating resources, it deletes them.

Understanding `terraform destroy` properly is important because:
- Used correctly, it is a safe and valuable tool for cleaning up environments
- Used carelessly, it can permanently delete production infrastructure

This file covers destroy in detail — how it works, when to use it, how to use it safely, and alternatives when you only want to remove some resources.

---

## What Terraform Destroy Does

When you run `terraform destroy`:

1. Terraform reads your configuration and the state file
2. It generates a plan showing everything that will be deleted
3. It asks for confirmation
4. It deletes resources in the correct order (respecting dependencies)
5. It updates the state file as each resource is successfully deleted

The dependency order during destroy is the reverse of creation. For example:
- During `apply`: VPC → Subnet → Security Group → EC2 instance
- During `destroy`: EC2 instance → Security Group → Subnet → VPC

Terraform knows this automatically from the dependency graph it built when creating the resources.

---

## Basic Destroy Command

```bash
terraform destroy
```

Output:

```
Terraform will perform the following actions:

  # aws_instance.web_server will be destroyed
  - resource "aws_instance" "web_server" {
      - ami                          = "ami-0c55b159cbfafe1f0" -> null
      - arn                          = "arn:aws:ec2:us-east-1:123456789:instance/i-0a1b2c3d4e5f67890" -> null
      - id                           = "i-0a1b2c3d4e5f67890" -> null
      - instance_type                = "t2.micro" -> null
      - public_ip                    = "54.23.45.67" -> null
        [... more attributes ...]
    }

  # aws_security_group.web_sg will be destroyed
  - resource "aws_security_group" "web_sg" {
      - id   = "sg-0a1b2c3d4e5f67890" -> null
      - name = "web-security-group" -> null
        [...]
    }

Plan: 0 to add, 0 to change, 2 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value:
```

The confirmation message is intentionally worded with weight: **"There is no undo."**

Type `yes` and press Enter to proceed.

---

## Destroy Output While Running

As resources are deleted:

```
aws_instance.web_server: Destroying... [id=i-0a1b2c3d4e5f67890]
aws_instance.web_server: Still destroying... [id=i-0a1b2c3d4e5f67890, 10s elapsed]
aws_instance.web_server: Still destroying... [id=i-0a1b2c3d4e5f67890, 20s elapsed]
aws_instance.web_server: Destruction complete after 32s
aws_security_group.web_sg: Destroying... [id=sg-0a1b2c3d4e5f67890]
aws_security_group.web_sg: Destruction complete after 1s

Destroy complete! Resources: 2 destroyed.
```

Notice that the EC2 instance was destroyed first, then the security group. Terraform respected the dependency order — you cannot delete a security group that is still attached to a running instance.

---

## Auto-Approve Destroy (For Automation)

To skip the confirmation prompt — useful in automated scripts and CI/CD pipelines:

```bash
terraform destroy -auto-approve
```

**Be very careful with this flag in production.** It destroys everything without asking. Always double-check which workspace and environment you are in before using it.

---

## Previewing a Destroy Without Executing It

Before running destroy, you can preview exactly what will be deleted using `terraform plan -destroy`:

```bash
terraform plan -destroy
```

This shows you the full destroy plan without actually making any changes. It is the same output as `terraform destroy` would show, minus the confirmation prompt.

Use this when:
- You want to review what will be destroyed before committing to it
- You want to save the destroy plan and apply it later

```bash
# Save a destroy plan
terraform plan -destroy -out=destroy-plan.tfplan

# Review it carefully...

# Apply the destroy plan
terraform apply destroy-plan.tfplan
```

---

## Destroying Only Specific Resources

Sometimes you want to remove just one or a few resources, not everything. Use the `-target` flag:

```bash
# Destroy only one specific resource
terraform destroy -target=aws_instance.web_server

# Destroy multiple specific resources
terraform destroy -target=aws_instance.web_server -target=aws_security_group.web_sg
```

Terraform will show you exactly what will be deleted (just the targeted resources) and ask for confirmation.

**Important:** After using `-target` for destroy, always run `terraform plan` afterward to check the state of your remaining infrastructure. Destroying some resources can sometimes affect others in ways that need cleanup.

---

## Real-World Use Cases for Destroy

### Use Case 1: Cleaning Up Development and Staging Environments

Development and staging environments should not run 24/7 if no one is using them. Destroying them when not needed saves significant cloud costs.

```bash
# Switch to the dev workspace
terraform workspace select dev

# Destroy all dev infrastructure
terraform destroy -auto-approve

# Later, recreate it when needed
terraform apply -auto-approve
```

### Use Case 2: Cleaning Up After Learning or Experiments

When you are learning Terraform, you create resources as practice. Always clean up afterward to avoid unexpected AWS charges:

```bash
# After you are done experimenting
terraform destroy
```

This is especially important because AWS charges for things like:
- EC2 instances (even if stopped)
- RDS databases
- NAT Gateways
- Elastic IPs (when not attached to a running instance)
- Load Balancers

### Use Case 3: Completely Rebuilding Infrastructure

If your infrastructure is severely misconfigured and it is easier to start fresh:

```bash
# Destroy everything
terraform destroy

# Recreate from scratch
terraform apply
```

This is sometimes called "nuke and pave" and is a valid approach when state has become too complex to fix incrementally.

### Use Case 4: End-of-Project Cleanup

When a project is complete and all its infrastructure is no longer needed:

```bash
terraform destroy
```

All associated AWS resources are cleanly removed, billing stops, and no orphaned resources are left behind.

---

## What Happens to the State File After Destroy

After `terraform destroy` completes successfully, the state file is updated to reflect that no resources exist:

```json
{
  "version": 4,
  "terraform_version": "1.6.6",
  "serial": 8,
  "lineage": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "outputs": {},
  "resources": []
}
```

The `resources` array is empty. The state file still exists — it is not deleted — but it records that there are no managed resources.

If you run `terraform apply` again after this, Terraform will recreate everything from scratch.

---

## Resources That Cannot Be Destroyed

Some resources have protections that prevent accidental deletion.

### The `prevent_destroy` Lifecycle Rule

You can protect individual resources from being destroyed by adding a lifecycle block:

```hcl
resource "aws_rds_cluster" "production_db" {
  cluster_identifier = "production-database"
  engine             = "aurora-mysql"
  master_username    = "admin"
  master_password    = var.db_password

  lifecycle {
    prevent_destroy = true
  }
}
```

If someone runs `terraform destroy` and this resource is in the plan, Terraform will refuse and show an error:

```
Error: Instance cannot be destroyed

  on main.tf line 1, in resource "aws_rds_cluster" "production_db":
   1: resource "aws_rds_cluster" "production_db" {

Resource aws_rds_cluster.production_db has lifecycle.prevent_destroy set, but
the plan calls for this resource to be destroyed. To avoid this error and
continue with the destroy, either remove the lifecycle configuration or
increase the target resource count.
```

This is very useful for protecting critical production resources like databases and state buckets from accidental deletion.

### AWS Deletion Protection

Many AWS resources also have their own deletion protection that you must disable before they can be deleted. When you run `terraform destroy` and a resource has deletion protection enabled, the destroy will fail.

```hcl
resource "aws_rds_instance" "production" {
  identifier        = "production-db"
  instance_class    = "db.t3.medium"
  engine            = "mysql"

  # This must be set to false before the instance can be deleted
  deletion_protection = true
}
```

To destroy this resource, you must first set `deletion_protection = false` and apply, then run destroy.

---

## Destroy and Dependencies

Terraform always respects resource dependencies during destroy. Let's trace through a real example.

**Configuration:**
```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  subnet_id     = aws_subnet.public.id
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

**Destroy order (reverse of creation):**
1. `aws_instance.web` is destroyed first (depends on the subnet)
2. `aws_subnet.public` is destroyed second (depends on the VPC)
3. `aws_vpc.main` is destroyed last (nothing depends on it)

If Terraform tried to destroy the VPC before the subnet or instance, AWS would reject it because you cannot delete a VPC that still has subnets or instances. Terraform prevents this automatically.

---

## Common Destroy Failures and How to Fix Them

### Failure: Resource has dependencies that prevent deletion

```
Error: deleting VPC: DependencyViolation: The vpc 'vpc-0a1b2c3d' has dependencies and cannot be deleted.
```

**Cause:** Something inside the VPC (instance, subnet, NAT gateway, etc.) was not deleted first.

**Fix:**
```bash
# Check what is still in state
terraform state list

# Destroy the dependent resources first
terraform destroy -target=aws_instance.web
terraform destroy -target=aws_subnet.public
# Then destroy the VPC
terraform destroy -target=aws_vpc.main
```

Or: find and remove any manually created resources in the VPC that are not in Terraform state.

### Failure: Resource protected by `prevent_destroy`

As shown earlier — remove the `prevent_destroy` lifecycle rule or use `-target` to destroy only unprotected resources.

### Failure: S3 bucket not empty

```
Error: deleting S3 Bucket: BucketNotEmpty: The bucket you tried to delete is not empty
```

**Fix:** Add `force_destroy = true` to the S3 bucket resource, apply, then destroy:

```hcl
resource "aws_s3_bucket" "app" {
  bucket        = "my-app-bucket"
  force_destroy = true  # Allows deletion even if bucket has objects
}
```

---

## Summary

- `terraform destroy` removes all infrastructure that Terraform manages — use it carefully
- Always review what will be destroyed using `terraform plan -destroy` before running the actual destroy
- Use `-auto-approve` only in automated pipelines — never casually in production
- Use `-target` to destroy specific resources rather than everything
- The `prevent_destroy` lifecycle rule protects critical resources from accidental deletion
- Terraform respects dependency order during destroy — it deletes in the reverse order of creation
- After destroy, the state file still exists but records no managed resources
- Common failures include dependency violations, deletion protection, and non-empty S3 buckets

In the next file, we cover `terraform fmt` and `terraform validate` — essential commands for keeping your code clean and correct.
