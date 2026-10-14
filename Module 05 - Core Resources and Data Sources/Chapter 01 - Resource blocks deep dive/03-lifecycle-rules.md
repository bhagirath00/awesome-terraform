# The Lifecycle Block — Controlling How Resources Are Created, Updated, and Destroyed

## Why Lifecycle Rules Exist

By default, Terraform has a simple strategy for handling resources: if something needs to change, destroy the old version and create a new one. This approach is clean and predictable.

But in production, "destroy first, then create" can mean **downtime**. Your users cannot access the application while the old server is gone and before the new one is ready. For critical infrastructure like databases, load balancers, and compute fleets, this is not acceptable.

Terraform's `lifecycle` block gives you the controls to override the default behavior and handle changes the way your production system actually needs.

Think of it like a set of rules you attach to a resource:
- "Create the replacement before killing the old one"
- "Never let anyone delete this accidentally"
- "Ignore certain changes — another system manages those"

These rules go inside a `lifecycle` block nested inside your resource block:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  lifecycle {
    # lifecycle rules go here
  }
}
```

There are four lifecycle rules. Let's go through each one in depth.

---

## 1. create_before_destroy — Zero-Downtime Replacements

### The Default Behavior (Causes Downtime)

When you change an argument that requires Terraform to replace a resource — for example, changing the AMI of an EC2 instance — Terraform does this by default:

1. Destroy the existing instance
2. Create a new instance with the new AMI

Between step 1 and step 2, there is a gap. Your web server is gone. Users see errors. This is downtime.

### The Fix

Setting `create_before_destroy = true` reverses the order:

1. Create the new instance with the new AMI
2. Wait for it to be ready
3. Destroy the old instance

At no point is there a gap. Users are always hitting a running server.

```hcl
resource "aws_launch_template" "web" {
  name_prefix   = "web-"
  image_id      = var.web_ami
  instance_type = "t3.small"

  lifecycle {
    create_before_destroy = true
  }
}
```

### Using name_prefix Instead of name

There is an important detail here. Most AWS resources require a **unique name**. If you use a hardcoded `name`, Terraform will try to create the new resource while the old one still exists — and both have the same name. AWS will reject the request.

The solution is to use `name_prefix` instead of `name`. With `name_prefix`, Terraform generates a unique random suffix each time:

```hcl
resource "aws_security_group" "app_sg" {
  # Use name_prefix, not name, when using create_before_destroy
  name_prefix = "app-sg-"
  description = "Application security group"
  vpc_id      = aws_vpc.main.id

  lifecycle {
    create_before_destroy = true
  }
}
```

This generates names like `app-sg-abc123`, ensuring uniqueness during the overlap period.

### Where to Use It

`create_before_destroy` is most important for:
- EC2 instances and Launch Templates behind a load balancer
- Security groups attached to running resources
- Auto Scaling group launch configurations
- Any resource where downtime during replacement is not acceptable

---

## 2. prevent_destroy — Protect Critical Resources

### The Problem

Your production RDS database takes hours to recreate. Someone on your team accidentally runs `terraform apply` after removing the database resource block from the code. Or they change a setting that forces replacement. Without any protection, Terraform happily deletes your production database.

`prevent_destroy = true` prevents this from happening. If any operation would result in the destruction of that resource, Terraform immediately stops and throws an error.

```hcl
resource "aws_db_instance" "production" {
  identifier        = "prod-mysql"
  engine            = "mysql"
  engine_version    = "8.0"
  instance_class    = "db.t3.medium"
  allocated_storage = 100
  username          = var.db_username
  password          = var.db_password
  skip_final_snapshot = false

  lifecycle {
    prevent_destroy = true
  }
}
```

Now if anyone tries to destroy this database through Terraform, they will see:

```
Error: Instance cannot be destroyed
  on main.tf line 1:
  resource "aws_db_instance" "production" {

This resource has lifecycle.prevent_destroy set to true.
To allow this object to be destroyed, you must first set
prevent_destroy to false and apply this updated config.
```

### To Actually Destroy the Resource

If you genuinely need to destroy a protected resource, you have to consciously do one of:

1. Set `prevent_destroy = false` in the code, run `terraform apply` to update the plan, then run `terraform destroy`
2. Remove the resource from state with `terraform state rm` and delete manually in the AWS console

This two-step process forces anyone who wants to delete critical infrastructure to pause, update the code, and think about what they are doing. Accidental deletions become nearly impossible.

### What to Protect

Apply `prevent_destroy` to infrastructure that is expensive or time-consuming to recreate:

```hcl
# Protect production databases
resource "aws_db_instance" "prod_db" {
  lifecycle { prevent_destroy = true }
}

# Protect S3 buckets containing critical data
resource "aws_s3_bucket" "audit_logs" {
  lifecycle { prevent_destroy = true }
}

# Protect KMS encryption keys
resource "aws_kms_key" "data_encryption" {
  lifecycle { prevent_destroy = true }
}

# Protect core networking
resource "aws_vpc" "production" {
  lifecycle { prevent_destroy = true }
}
```

---

## 3. ignore_changes — Let Other Systems Manage Certain Attributes

### The Problem

You provision an Auto Scaling Group with Terraform. The `desired_capacity` is set to 2. During the day, traffic spikes and AWS Auto Scaling scales it up to 8 instances. That night, you run `terraform apply` for an unrelated change. Terraform sees `desired_capacity = 8` in AWS but `desired_capacity = 2` in your code. It "fixes" it by scaling back down to 2.

This is called **drift** — the real infrastructure no longer matches your Terraform code. But in this case, the drift is intentional. You want Auto Scaling to manage the count dynamically.

`ignore_changes` tells Terraform: "I know this attribute might be different in real life — that's fine, don't try to correct it."

```hcl
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  min_size            = 2
  max_size            = 20
  desired_capacity    = 2
  vpc_zone_identifier = [aws_subnet.private_a.id, aws_subnet.private_b.id]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  lifecycle {
    ignore_changes = [
      # AWS Auto Scaling changes this — don't fight it
      desired_capacity
    ]
  }
}
```

Now Terraform will set `desired_capacity = 2` when first creating the ASG, but it will never try to change it back after that. Auto Scaling is free to do its job.

### Ignoring Tags

Another common use case is when a security scanning tool or cost management tool adds its own tags to your resources:

```hcl
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.small"

  tags = {
    Name        = "app-server"
    Environment = "production"
  }

  lifecycle {
    ignore_changes = [
      # Our security tool adds its own tags — ignore them
      tags["SecurityScanDate"],
      tags["CostCenter"]
    ]
  }
}
```

### Ignoring All Changes

In rare cases, you want Terraform to create a resource but never touch it again — let another system manage it entirely:

```hcl
resource "aws_instance" "legacy_app" {
  ami           = "ami-legacy-1234"
  instance_type = "t3.medium"

  lifecycle {
    ignore_changes = all
  }
}
```

Use `ignore_changes = all` carefully. Once you do this, Terraform will never update the resource, even if you change something important in the code.

---

## 4. replace_triggered_by — Force Replacement on Related Changes

### The Problem

Sometimes a resource needs to be replaced when another resource changes — even though there is no direct configuration link between them.

For example: you have an EC2 instance running your application. You update the IAM role attached to it with new permissions. The EC2 instance itself does not change — but it needs to be restarted to pick up the new role properly.

Without `replace_triggered_by`, Terraform would update the IAM role and leave the EC2 instance running with stale permissions (until you restart it manually).

```hcl
resource "aws_iam_role_policy" "app_permissions" {
  name = "app-s3-policy"
  role = aws_iam_role.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject"]
      Resource = "${aws_s3_bucket.app_data.arn}/*"
    }]
  })
}

resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.small"
  iam_instance_profile = aws_iam_instance_profile.app.name

  lifecycle {
    # Replace this instance whenever the IAM policy changes
    replace_triggered_by = [
      aws_iam_role_policy.app_permissions
    ]
  }
}
```

Now every time the IAM policy changes, Terraform will replace the EC2 instance too — ensuring it boots with the latest permissions.

---

## 5. precondition and postcondition — Validation at Apply Time

Terraform also lets you add validation checks inside the lifecycle block. These are different from variable validation — they run against live infrastructure during plan and apply.

### precondition — Check Before Creating

A precondition runs before Terraform creates or updates the resource. If it fails, Terraform stops:

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = "t3.small"
  subnet_id     = var.subnet_id

  lifecycle {
    precondition {
      condition     = data.aws_subnet.selected.available_ip_address_count > 10
      error_message = "The selected subnet does not have enough free IP addresses."
    }
  }
}
```

### postcondition — Check After Creating

A postcondition runs after Terraform creates or updates the resource. Use it to enforce security rules:

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = "t3.small"

  lifecycle {
    postcondition {
      # Enforce that this instance never gets a public IP
      condition     = self.public_ip == null || self.public_ip == ""
      error_message = "App server must not have a public IP address assigned."
    }
  }
}
```

If the instance accidentally gets a public IP (because someone changed the subnet settings), this postcondition catches it and raises an error immediately.

---

## Combining Lifecycle Rules

You can use multiple lifecycle rules together in the same block:

```hcl
resource "aws_db_instance" "production" {
  identifier        = "prod-postgres"
  engine            = "postgres"
  engine_version    = "15.3"
  instance_class    = "db.r6g.large"
  allocated_storage = 500
  username          = var.db_user
  password          = var.db_password

  lifecycle {
    # Don't delete production databases accidentally
    prevent_destroy = true

    # If the instance class changes, create new first to avoid downtime
    create_before_destroy = true

    # AWS manages automated backups — don't fight those changes
    ignore_changes = [
      backup_retention_period,
      maintenance_window
    ]
  }
}
```

---

## Quick Reference

| Rule | When to Use |
|------|------------|
| `create_before_destroy = true` | When downtime during replacement is not acceptable |
| `prevent_destroy = true` | For databases, KMS keys, core VPCs, S3 buckets with important data |
| `ignore_changes = [...]` | When another system (Auto Scaling, security tools) manages certain attributes |
| `replace_triggered_by = [...]` | When a resource needs replacing due to a change in a related resource |
| `precondition {}` | Validate conditions before creating/updating |
| `postcondition {}` | Enforce security or compliance rules after creation |

---

## Summary

- The `lifecycle` block lets you override Terraform's default create/update/destroy behavior for individual resources
- `create_before_destroy = true` eliminates downtime during replacements — use with `name_prefix` instead of `name` to avoid naming conflicts
- `prevent_destroy = true` protects critical resources from accidental deletion — you must consciously remove the protection before destroying
- `ignore_changes` prevents Terraform from reverting attributes managed by external systems like Auto Scaling or security tools
- `replace_triggered_by` forces a resource replacement when a related resource changes, even without a direct dependency link
- `precondition` and `postcondition` add runtime validation to enforce infrastructure rules during apply

In the next chapter, we move on to **Data Sources** — a way to read existing infrastructure that Terraform did not create, so you can use its attributes in your own resources.
