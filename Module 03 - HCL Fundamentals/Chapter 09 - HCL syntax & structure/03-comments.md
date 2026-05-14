# Comments in HCL

## Introduction

Comments are lines in your Terraform code that are ignored by Terraform — they exist only for humans reading the code. Writing good comments is a professional habit that makes your Terraform configurations much easier to understand, maintain, and hand off to other engineers.

This file covers the three types of comments HCL supports, when to use each one, and practical guidelines for writing comments that actually add value.

---

## Three Types of Comments

HCL supports three different comment syntaxes.

### Type 1: Single-Line Comment with `#`

The `#` character starts a single-line comment. Everything after `#` on that line is ignored by Terraform.

```hcl
# This is a single-line comment

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2023 AMI for us-east-1
  instance_type = "t2.micro"               # Free tier eligible
}
```

This is the **most commonly used** comment style in Terraform. You will see it everywhere.

---

### Type 2: Single-Line Comment with `//`

The `//` syntax also creates a single-line comment. It works exactly the same as `#`.

```hcl
// This is also a valid single-line comment

resource "aws_s3_bucket" "logs" {
  bucket = "my-app-logs"  // Bucket name must be globally unique
}
```

While `//` is valid HCL, **the `#` style is strongly preferred** in the Terraform community. Most Terraform code you encounter, including HashiCorp's own examples, uses `#`. The `terraform fmt` command does not convert `//` to `#`, but `#` is the convention. Stick with `#` for consistency.

---

### Type 3: Multi-Line Comment with `/* */`

For comments that span multiple lines, wrap them in `/*` and `*/`:

```hcl
/*
  This is a multi-line comment.
  It can span as many lines as you need.
  Everything between the opening and closing markers is ignored.
*/

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

Multi-line comments are useful for:
- Long explanations that do not fit on one line
- Temporarily disabling a block of configuration
- Writing file-level documentation at the top of a file

---

## Inline Comments

You can place a comment at the end of a line after an argument:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"   # us-east-1 Amazon Linux 2023
  instance_type = "t2.micro"                # free tier — change for production
  monitoring    = false                      # enable for production environments
}
```

Inline comments work well for quick notes directly next to the value they explain. Keep them short — if the explanation is long, use a full-line comment above the argument instead.

---

## Commenting Out Code

One of the most practical uses of comments is temporarily disabling configuration without deleting it.

### Commenting Out a Single Argument

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  # user_data = file("startup.sh")  # disabled while debugging startup issues
}
```

### Commenting Out an Entire Block

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  /*
  root_block_device {
    volume_size = 50
    volume_type = "gp3"
    encrypted   = true
  }
  */
}
```

Using `/* */` is convenient for temporarily disabling a nested block. You just wrap it and unwrap when you are ready to enable it again.

---

## When to Write Comments

Good comments explain **why** something is done, not **what** is done. The code itself already shows what is happening — a good comment adds context that the code cannot.

### Good Comments — Explain Why

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.small"  # t3.small required — app needs >1GB RAM (t2.micro only has 1GB)

  # Disable termination protection during testing
  # Enable this before go-live: disable_api_termination = true
  disable_api_termination = false
}

resource "aws_s3_bucket" "logs" {
  bucket        = "myapp-logs-${var.environment}"
  force_destroy = true  # Allow bucket deletion even if not empty — needed for dev teardown

  # Note: Set force_destroy = false in production to prevent accidental data loss
}
```

### Bad Comments — Repeat What the Code Says

```hcl
# Create an EC2 instance
resource "aws_instance" "web" {
  # Set the AMI
  ami = "ami-0c55b159cbfafe1f0"

  # Set the instance type
  instance_type = "t2.micro"
}
```

These comments add no value — anyone reading HCL already knows that `ami` sets the AMI and `instance_type` sets the instance type. Delete comments like these.

---

## Practical Comment Patterns for Terraform

### File Header Comment

Add a comment at the top of each file explaining what it contains and any important notes:

```hcl
# networking.tf
# 
# This file defines the VPC, subnets, route tables, and internet gateway
# for the application's network infrastructure.
#
# Architecture:
#   - 1 VPC with CIDR 10.0.0.0/16
#   - 2 public subnets (for load balancers and NAT gateways)
#   - 2 private subnets (for application servers and databases)
#
# Last reviewed: 2024-01
# Owner: platform-team@company.com
```

---

### Explaining Non-Obvious Values

```hcl
resource "aws_security_group" "rds" {
  name = "rds-security-group"

  ingress {
    from_port   = 5432       # PostgreSQL default port
    to_port     = 5432
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]  # Allow entire private network range
  }
}

resource "aws_iam_role_policy_attachment" "ecr_read" {
  role       = aws_iam_role.app.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  # This policy allows the app to pull Docker images from ECR
  # without needing explicit credentials
}
```

---

### Explaining Dependencies

```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  # The internet gateway must be created before public route tables
  # Terraform handles this automatically via implicit dependency,
  # but noting it here for clarity
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id

  # NAT gateway must be in a PUBLIC subnet to route internet traffic
  # Using public[0] — the first availability zone's public subnet
  depends_on = [aws_internet_gateway.main]
}
```

---

### Explaining TODO Items

```hcl
resource "aws_rds_instance" "app_db" {
  identifier        = "app-database"
  engine            = "mysql"
  instance_class    = "db.t3.micro"
  allocated_storage = 20

  # TODO: Enable multi-AZ before production launch
  multi_az = false

  # TODO: Reduce backup retention from 7 to match company policy
  backup_retention_period = 7

  # TODO: Add enhanced monitoring when budget allows
  # monitoring_interval = 60
  # monitoring_role_arn = aws_iam_role.rds_monitoring.arn
}
```

TODO comments are useful for tracking things that need to be done before a feature is complete or before moving to production.

---

### Explaining Workarounds

Sometimes you have to do something that looks strange because of a bug, limitation, or temporary constraint. Always comment these:

```hcl
resource "aws_s3_bucket_acl" "app_bucket_acl" {
  bucket = aws_s3_bucket.app.id
  acl    = "private"

  # Workaround: AWS requires the bucket ownership controls to be set
  # before the ACL can be applied. The depends_on ensures ordering.
  # See: https://github.com/hashicorp/terraform-provider-aws/issues/28237
  depends_on = [aws_s3_bucket_ownership_controls.app_bucket]
}
```

This kind of comment saves the next engineer hours of debugging. They will not wonder why `depends_on` is there — the comment explains it and even links to the issue.

---

### Section Dividers in Large Files

For large files, use comment dividers to organize sections visually:

```hcl
# ==============================================================================
# VPC and Networking
# ==============================================================================

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# ------------------------------------------------------------------------------
# Subnets
# ------------------------------------------------------------------------------

resource "aws_subnet" "public_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
}

resource "aws_subnet" "public_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
}

# ==============================================================================
# Security Groups
# ==============================================================================

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
}
```

---

## Comments and `terraform fmt`

`terraform fmt` does not modify the content of comments — only code formatting. Your comments will stay exactly as you write them.

However, `terraform fmt` does correctly indent lines that start with `#`:

```hcl
resource "aws_instance" "web" {
# This comment was at the wrong indentation level
  ami = "ami-0c55b159cbfafe1f0"
}
```

After `terraform fmt`:
```hcl
resource "aws_instance" "web" {
  # This comment is now properly indented
  ami = "ami-0c55b159cbfafe1f0"
}
```

---

## Comments in Variable and Output Files

Variables and outputs have built-in `description` arguments that serve as formal documentation. Use descriptions for the "what" and comments for the "why":

```hcl
# This variable controls the size of the EC2 instances.
# We use t3.medium in staging because the profiler showed the app
# needs at least 4GB RAM under realistic load. t3.small was insufficient.
variable "instance_type" {
  description = "EC2 instance type for the application servers"
  type        = string
  default     = "t3.medium"
}
```

The `description` will appear in `terraform output`, `terraform console`, and in module documentation. The `#` comment is for engineers reading the source code and adding extra context.

---

## Summary

- HCL supports three comment styles: `#` (preferred), `//` (valid but less common), and `/* */` (multi-line)
- Always prefer `#` for single-line comments — it is the Terraform community standard
- Use `/* */` for temporarily commenting out blocks of code
- Good comments explain **why** — not what the code already clearly shows
- Common comment patterns: file headers, explaining non-obvious values, documenting dependencies, TODO items, workaround explanations, and section dividers
- `terraform fmt` does not change comment content but does fix comment indentation
- Use `description` arguments for formal documentation and `#` comments for additional context

In the next file, we cover all the data types that HCL supports — strings, numbers, booleans, lists, maps, and more.
