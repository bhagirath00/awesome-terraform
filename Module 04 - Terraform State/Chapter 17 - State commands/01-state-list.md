# terraform state list

## Introduction

`terraform state list` is the command you use to see all the resources that Terraform is currently tracking in its state file. It gives you a clean, readable list of every resource address — the unique identifier for each resource in Terraform's world.

This is typically the first command you reach for when you need to understand what Terraform is managing, debug an issue, or prepare to run other state commands.

---

## Basic Usage

```bash
terraform state list
```

Sample output for a typical web application infrastructure:

```
aws_db_instance.app
aws_db_subnet_group.main
aws_iam_instance_profile.app
aws_iam_role.app
aws_iam_role_policy.app
aws_instance.web[0]
aws_instance.web[1]
aws_instance.web[2]
aws_internet_gateway.main
aws_lb.main
aws_lb_listener.http
aws_lb_listener.https
aws_lb_target_group.app
aws_lb_target_group_attachment.app[0]
aws_lb_target_group_attachment.app[1]
aws_lb_target_group_attachment.app[2]
aws_route_table.public
aws_route_table_association.public[0]
aws_route_table_association.public[1]
aws_s3_bucket.app_data
aws_s3_bucket.logs
aws_security_group.alb
aws_security_group.app
aws_security_group.rds
aws_subnet.private[0]
aws_subnet.private[1]
aws_subnet.public[0]
aws_subnet.public[1]
aws_vpc.main
```

Each line is a **resource address** — the unique name Terraform uses to identify that resource in state.

---

## Understanding Resource Addresses

A resource address is made up of the resource type and the resource name from your `.tf` file:

```
resource_type.resource_name
```

For example:
- `aws_vpc.main` — the VPC resource named `main`
- `aws_instance.web_server` — the EC2 instance named `web_server`
- `aws_s3_bucket.app_data` — the S3 bucket named `app_data`

### Addresses for `count` Resources

When a resource uses `count`, each instance gets an index in square brackets:

```
aws_instance.web[0]
aws_instance.web[1]
aws_instance.web[2]
```

These are three separate EC2 instances, all defined by the same `resource "aws_instance" "web"` block with `count = 3`.

### Addresses for `for_each` Resources

When a resource uses `for_each`, each instance uses the map key in square brackets:

```
aws_subnet.public["us-east-1a"]
aws_subnet.public["us-east-1b"]
aws_subnet.private["us-east-1a"]
aws_subnet.private["us-east-1b"]
```

### Addresses for Module Resources

Resources inside modules use the module path as a prefix:

```
module.vpc.aws_vpc.main
module.vpc.aws_subnet.public[0]
module.vpc.aws_subnet.private[0]
module.eks.aws_eks_cluster.main
module.eks.aws_eks_node_group.workers
```

The format is `module.module_name.resource_type.resource_name`.

### Addresses for Nested Modules

If modules are nested (a module calls another module):

```
module.infrastructure.module.vpc.aws_vpc.main
module.infrastructure.module.vpc.aws_subnet.public[0]
```

---

## Filtering the List

### Filter by Resource Type

Pass a partial address to filter results:

```bash
# Show only EC2 instances
terraform state list aws_instance

# Show only S3 buckets
terraform state list aws_s3_bucket

# Show only security groups
terraform state list aws_security_group
```

Output when filtering for `aws_instance`:
```
aws_instance.web[0]
aws_instance.web[1]
aws_instance.web[2]
```

---

### Filter with Wildcards

Use glob-style patterns to filter:

```bash
# Show all resources in a module
terraform state list 'module.vpc.*'

# Show all subnet resources
terraform state list 'aws_subnet.*'

# Show all IAM resources
terraform state list 'aws_iam_*'
```

Output for `module.vpc.*`:
```
module.vpc.aws_internet_gateway.main
module.vpc.aws_route_table.public
module.vpc.aws_route_table_association.public[0]
module.vpc.aws_route_table_association.public[1]
module.vpc.aws_subnet.private[0]
module.vpc.aws_subnet.private[1]
module.vpc.aws_subnet.public[0]
module.vpc.aws_subnet.public[1]
module.vpc.aws_vpc.main
```

---

### Read from a Specific State File

By default, `terraform state list` reads from the configured backend. To read from a specific state file:

```bash
terraform state list -state=path/to/terraform.tfstate
```

Useful when inspecting a backup or a state file from a different environment.

---

## Practical Use Cases

### Use Case 1: Verify Resources Were Created

After `terraform apply`, confirm the expected resources exist:

```bash
terraform state list | grep aws_instance
# aws_instance.web[0]
# aws_instance.web[1]
# aws_instance.web[2]
```

If you expected 3 instances and see 3, the apply worked correctly.

---

### Use Case 2: Count Resources

```bash
# Count total resources in state
terraform state list | wc -l

# Count specific resource types
terraform state list aws_subnet | wc -l
```

---

### Use Case 3: Prepare for State Commands

Before running `terraform state show`, `terraform state mv`, or `terraform state rm`, use `terraform state list` to get the exact resource address you need:

```bash
# Step 1: Find the resource address
terraform state list | grep security_group

# Output:
# aws_security_group.alb
# aws_security_group.app
# aws_security_group.rds

# Step 2: Use the exact address in another command
terraform state show aws_security_group.app
```

---

### Use Case 4: Debugging Missing Resources

If a resource exists in AWS but `terraform plan` shows it will be created (suggesting Terraform does not know about it), check if it is in state:

```bash
# Is this bucket in state?
terraform state list aws_s3_bucket
```

If the bucket does not appear, it is either not managed by Terraform or was removed from state. You may need to `terraform import` it.

---

### Use Case 5: Verify a Module's Resources

After adding a module to your configuration and applying:

```bash
terraform state list 'module.vpc.*'
```

This confirms all the resources the module should have created are actually in state.

---

### Use Case 6: Scripting with State List

`terraform state list` output is clean text — one address per line, making it easy to use in scripts:

```bash
#!/bin/bash
# Destroy all EC2 instances one at a time for debugging

for instance in $(terraform state list aws_instance); do
  echo "Destroying: $instance"
  terraform destroy -target="$instance" -auto-approve
done
```

```bash
#!/bin/bash
# Export all resource addresses for documentation

echo "Resources managed by Terraform:" > resources.txt
terraform state list >> resources.txt
echo "Total: $(terraform state list | wc -l) resources" >> resources.txt
```

---

## Comparing State List to Other Inspection Commands

| Command | What it shows |
|---|---|
| `terraform state list` | Just the resource addresses — clean list |
| `terraform state show <address>` | Full details of one specific resource |
| `terraform show` | Full details of ALL resources (very verbose) |
| `terraform show -json` | Full state as JSON (for automation) |

Start with `terraform state list` to find what you are looking for, then use `terraform state show` to dig into the details of a specific resource.

---

## Summary

- `terraform state list` shows all resource addresses currently tracked in Terraform state
- Resource addresses follow the format `resource_type.resource_name`
- `count` resources use numeric indexes: `aws_instance.web[0]`
- `for_each` resources use map keys: `aws_subnet.public["us-east-1a"]`
- Module resources use the module path prefix: `module.vpc.aws_vpc.main`
- Filter results by passing a partial address or glob pattern: `terraform state list aws_instance`
- Use `terraform state list` as the first step before running other state commands
- The clean text output makes it easy to use in shell scripts

In the next file, we cover `terraform state show` — how to inspect the full details of a specific resource in state.
