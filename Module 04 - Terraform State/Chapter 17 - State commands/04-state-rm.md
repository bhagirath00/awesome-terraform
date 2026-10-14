# terraform state rm

## Introduction

`terraform state rm` removes a resource from Terraform's state file without destroying the actual infrastructure. After running this command, Terraform no longer knows about the resource — it is as if Terraform never created it. The real resource in AWS remains completely untouched.

This is a powerful but dangerous command. Used correctly, it solves real problems. Used incorrectly, it creates drift between state and real infrastructure that is painful to reconcile.

---

## When to Use terraform state rm

### Use Case 1: Stop Managing a Resource

You want Terraform to stop managing a specific resource — perhaps because another team or tool will take it over — but you do not want to delete the actual resource.

```bash
# Terraform stops tracking this bucket — the bucket itself is untouched in AWS
terraform state rm aws_s3_bucket.legacy_data
```

After this command:
- The bucket still exists in AWS with all its data
- Terraform's state no longer has any record of it
- Running `terraform plan` will show the bucket as "to be created" if it is still in your config
- Running `terraform destroy` will NOT delete the bucket (since it is not in state)

---

### Use Case 2: Splitting a Configuration

When splitting a large Terraform configuration into smaller ones, you move some resources to a new configuration. After importing them into the new config's state, you remove them from the old config's state.

```bash
# Remove from old config's state (after importing to new config)
terraform state rm aws_rds_cluster.main
terraform state rm aws_db_subnet_group.main
```

---

### Use Case 3: Removing a Deleted Resource from State

If a resource was deleted outside of Terraform (manually in AWS Console) and you do not want Terraform to try recreating it, remove the resource from both state and configuration:

1. Remove the resource block from your `.tf` file
2. Remove it from state

```bash
terraform state rm aws_instance.old_server
```

Now both state and config agree that the resource does not exist. `terraform plan` shows no changes.

---

### Use Case 4: Recovering from Import Errors

If you run `terraform import` incorrectly and import the wrong resource, use `state rm` to undo the import:

```bash
# Oops, imported the wrong instance
terraform import aws_instance.web i-wrong-instance-id

# Remove it from state so you can try again
terraform state rm aws_instance.web

# Now import the correct one
terraform import aws_instance.web i-correct-instance-id
```

---

### Use Case 5: Removing a Module from Management

Remove all resources managed by a module from state at once:

```bash
# Remove the entire module from state tracking
terraform state rm module.old_vpc
```

This removes every resource inside `module.old_vpc` from state in one command.

---

## Basic Syntax

```bash
terraform state rm <resource_address>
```

```bash
# Remove a single resource
terraform state rm aws_instance.web_server

# Remove a module and all its resources
terraform state rm module.vpc

# Remove a specific count instance
terraform state rm 'aws_instance.web[0]'

# Remove a specific for_each instance
terraform state rm 'aws_subnet.public["us-east-1a"]'

# Remove multiple resources
terraform state rm aws_instance.old aws_security_group.old aws_eip.old
```

---

## Output After Running

```bash
terraform state rm aws_s3_bucket.legacy_data
```

```
Removed aws_s3_bucket.legacy_data
Successfully removed 1 resource instance(s).
```

For modules:
```bash
terraform state rm module.vpc
```

```
Removed module.vpc.aws_internet_gateway.main
Removed module.vpc.aws_route_table.public
Removed module.vpc.aws_route_table_association.public[0]
Removed module.vpc.aws_route_table_association.public[1]
Removed module.vpc.aws_subnet.private[0]
Removed module.vpc.aws_subnet.private[1]
Removed module.vpc.aws_subnet.public[0]
Removed module.vpc.aws_subnet.public[1]
Removed module.vpc.aws_vpc.main
Successfully removed 9 resource instance(s).
```

---

## Dry Run Before Removing

Always verify what will be removed before running the actual command. Use `-dry-run`:

```bash
terraform state rm -dry-run module.vpc
```

```
Would remove module.vpc.aws_internet_gateway.main
Would remove module.vpc.aws_route_table.public
Would remove module.vpc.aws_subnet.public[0]
...
```

Nothing is changed. Review the list carefully, then run without `-dry-run` to execute.

---

## What Happens After terraform state rm

Let's trace through exactly what happens after removing a resource from state.

**Before removal:**
- State knows about `aws_s3_bucket.logs`
- Your `.tf` file has `resource "aws_s3_bucket" "logs"`
- The bucket exists in AWS

**Run: `terraform state rm aws_s3_bucket.logs`**

**After removal:**
- State no longer has `aws_s3_bucket.logs`
- Your `.tf` file still has `resource "aws_s3_bucket" "logs"`
- The bucket still exists in AWS

**Run: `terraform plan`**

```
+ aws_s3_bucket.logs will be created
  + bucket = "myapp-logs-bucket"
  ...

Plan: 1 to add, 0 to change, 0 to destroy.
```

Terraform sees the bucket in your config but not in state, so it plans to create it. Since the bucket already exists in AWS, `terraform apply` would actually fail with a "bucket already exists" error.

**The correct follow-up actions depend on your intention:**

| Intention | Follow-up Action |
|---|---|
| Stop managing this resource entirely | Also remove the resource block from your `.tf` file |
| Transfer to another Terraform config | Import the resource into the new config's state |
| Temporarily remove then re-import | Use `terraform import` to bring it back |

---

## The Complete "Stop Managing" Workflow

If you want to completely remove a resource from Terraform management:

**Step 1:** Remove the resource block from your `.tf` file

```hcl
# Delete or comment out this block
# resource "aws_s3_bucket" "legacy_data" {
#   bucket = "company-legacy-data-2019"
# }
```

**Step 2:** Remove the resource from state

```bash
terraform state rm aws_s3_bucket.legacy_data
```

**Step 3:** Verify with plan

```bash
terraform plan
# Should show: No changes. Your infrastructure matches the configuration.
```

The bucket still exists in AWS, Terraform has no record of it, and your config does not mention it. Everything is clean.

---

## Removing Resources That Reference Each Other

When removing resources that have dependencies, remove in the correct order (dependents first):

```bash
# Wrong order — trying to remove VPC before its subnets
terraform state rm aws_vpc.main           # might cause confusion
terraform state rm aws_subnet.public[0]
terraform state rm aws_subnet.public[1]

# Correct order — remove dependents first
terraform state rm aws_subnet.public[0]
terraform state rm aws_subnet.public[1]
terraform state rm aws_vpc.main
```

For modules, removing the entire module at once handles ordering automatically:

```bash
# Terraform handles the removal order automatically for modules
terraform state rm module.networking
```

---

## Terraform state rm vs terraform destroy

These two commands are often confused. The difference is critical:

| Command | Effect on State | Effect on Real Infrastructure |
|---|---|---|
| `terraform state rm` | Removes resource from state | Does NOT touch real infrastructure |
| `terraform destroy -target` | Removes resource from state | DELETES the real infrastructure |

Use `terraform state rm` when you want to stop tracking something.
Use `terraform destroy -target` when you want to actually delete something.

---

## Backup Before Removing

Before removing important resources from state, create a backup:

```bash
# Create a manual backup of current state
terraform state pull > state-backup-$(date +%Y%m%d-%H%M%S).tfstate

# Then perform the removal
terraform state rm aws_s3_bucket.legacy_data
```

If something goes wrong, you can restore from the backup:

```bash
# Restore from backup
terraform state push state-backup-20240115-103045.tfstate
```

---

## Summary

- `terraform state rm` removes a resource from Terraform state without deleting the actual infrastructure
- Use it when: stopping management of a resource, splitting configurations, recovering from bad imports, removing deleted resources
- Always use `-dry-run` first to preview what will be removed
- After `state rm`, the resource still exists in AWS but Terraform no longer tracks it
- If the resource is still in your config, `terraform plan` will want to recreate it — remove it from config too if you want to fully stop managing it
- For a complete "stop managing" workflow: remove from config AND run `state rm`
- Back up your state with `terraform state pull` before any significant `state rm` operations
- `state rm` is NOT the same as `terraform destroy` — `state rm` does not delete infrastructure

In the next file, we cover `terraform state pull` and `terraform state push` — the commands for manually managing the state file content.
