# Resource Blocks — Syntax and Structure

## What is a Resource Block?

Everything in Terraform starts with a resource. A resource block tells Terraform: "I want this thing to exist in the cloud." It could be an EC2 server, an S3 bucket, a security group, a database — anything that your cloud provider can create.

Think of a resource block like a **blueprint**. When a builder gets a blueprint, they know exactly what to build — how many floors, what materials, what dimensions. When Terraform gets a resource block, it knows exactly what cloud infrastructure to create — which region, which size, which settings.

Your job as a DevOps engineer is to write these blueprints. Terraform's job is to make them real.

---

## The Basic Structure of a Resource Block

Every resource block follows the same pattern:

```hcl
resource "<RESOURCE_TYPE>" "<LOCAL_NAME>" {
  argument_1 = value_1
  argument_2 = value_2
}
```

Here is a real example — creating an EC2 instance on AWS:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name        = "web-server"
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}
```

This tells Terraform: "Create one AWS EC2 instance, using this AMI image, with this size, and tag it with these labels."

Let's break down each part.

---

## Part 1 — The Resource Type

The first label after the `resource` keyword is the **resource type**. In the example above, that is `aws_instance`.

The resource type always follows the pattern: `<provider>_<service>`.

- `aws_instance` → AWS provider, EC2 instance
- `aws_s3_bucket` → AWS provider, S3 bucket
- `aws_security_group` → AWS provider, security group
- `aws_db_instance` → AWS provider, RDS database instance

The resource type tells Terraform two things:
1. **Which provider** is responsible for this resource (AWS, Azure, GCP, etc.)
2. **What kind of object** to create and which API calls to make

Every resource type comes from the provider's documentation. You cannot invent your own resource types — they must exist in the provider.

---

## Part 2 — The Local Name

The second label is the **local name**. In the example above, that is `web_server`.

The local name is **your name** for this resource inside your Terraform code. It does not become the name of the resource in AWS — it is only used inside your `.tf` files to reference this resource.

Think of it as a variable name. You use `web_server` to talk about this EC2 instance inside your code:

```hcl
# Reference this EC2 instance's public IP in another part of your code
output "server_ip" {
  value = aws_instance.web_server.public_ip
}
```

Rules for local names:
- Must start with a letter or underscore
- Can contain letters, numbers, underscores, and hyphens
- Must be unique within the same resource type in a module

**Best practice:** Use descriptive, lowercase names with underscores.

```hcl
# Good
resource "aws_instance" "web_server" {}
resource "aws_instance" "bastion_host" {}

# Avoid
resource "aws_instance" "instance1" {}
resource "aws_instance" "MyWebServer" {}
```

---

## Part 3 — The Resource Address

The combination of resource type and local name forms the **resource address**. This is how Terraform identifies a resource in its state file and in commands.

```
aws_instance.web_server
```

You will see this address in:
- The output of `terraform plan` and `terraform apply`
- The `terraform.tfstate` file
- Terraform CLI commands like `terraform state show aws_instance.web_server`

When resources are inside modules, the address becomes longer:

```
module.networking.aws_vpc.main
module.compute.aws_instance.web_server
```

Understanding addresses helps you when you need to target specific resources with commands like:

```bash
# Only plan changes for this one resource
terraform plan -target=aws_instance.web_server

# Show the state of this resource
terraform state show aws_instance.web_server
```

---

## Part 4 — Arguments Inside the Block

Inside the curly braces, you write **arguments** — the settings that configure the resource.

Arguments come in two forms:

### Simple Arguments (key = value)

```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "my-app-logs-bucket"
  force_destroy = true
}
```

These are straightforward key-value pairs. `bucket` is the argument name, `"my-app-logs-bucket"` is the value.

### Nested Block Arguments

Some arguments are complex objects that require their own block:

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-security-group"
  description = "Allow HTTP and HTTPS traffic"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Here, `ingress` and `egress` are nested blocks. You can have multiple `ingress` blocks in one security group. The provider's documentation tells you which nested blocks are allowed and how many.

---

## Required vs Optional Arguments

Every resource type has a list of arguments, and each one is either required or optional.

| Type | What it means | Example |
|------|--------------|---------|
| **Required** | You must provide this, or Terraform will error | `ami` in `aws_instance` |
| **Optional** | Has a default value — you can skip it | `instance_type` defaults to nothing, but it is required for EC2 |
| **Computed** | Set by the provider after creation — you cannot set it | `public_ip`, `arn`, `id` |

Here is a practical example showing all three:

```hcl
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"  # Required — must provide
  instance_type = "t3.small"               # Required — must provide

  # Optional — if you skip this, Terraform uses the default VPC
  subnet_id = aws_subnet.private.id

  # Computed — you cannot set this, AWS assigns it after creation
  # public_ip = ???  ← this would cause an error

  tags = {
    Name = "app-server"
  }
}
```

After Terraform creates this instance, you can read the computed values:

```hcl
output "instance_ip" {
  value = aws_instance.app_server.public_ip  # AWS fills this in
}

output "instance_arn" {
  value = aws_instance.app_server.arn        # AWS fills this in too
}
```

---

## How Terraform Uses the Resource Block

When you run `terraform apply`, here is exactly what Terraform does with your resource blocks:

1. **Reads** all your `.tf` files and builds an internal picture of what you want
2. **Reads** the state file to see what already exists
3. **Calls** the provider API to check the real current state in AWS
4. **Compares** desired state vs real state
5. **Decides** what action is needed:
   - **Create** — resource is in your config but not in AWS
   - **Update** — resource exists but some arguments changed
   - **Destroy and recreate** — some arguments cannot be changed in-place (like an EC2 AMI)
   - **No-op** — everything matches, nothing to do

This is the **declarative** power of Terraform. You write what you want, not the steps to get there. Terraform figures out the steps.

---

## Naming Conventions — Best Practices

Good naming makes your code easy to read and maintain. Here are the rules senior DevOps engineers follow:

```hcl
# Use singular nouns for single resources
resource "aws_vpc" "main" {}

# Use descriptive names that explain the purpose
resource "aws_security_group" "alb_sg" {}       # ALB security group
resource "aws_security_group" "app_sg" {}       # App tier security group
resource "aws_security_group" "db_sg" {}        # Database security group

# Avoid repeating the resource type in the name
resource "aws_s3_bucket" "bucket" {}            # Redundant — "bucket" tells you nothing
resource "aws_s3_bucket" "terraform_state" {}   # Clear — tells you the purpose
```

---

## A Complete Real-World Example

Here is a full example of creating an EC2 instance with all the pieces working together — showing how resources reference each other:

```hcl
# Create a security group
resource "aws_security_group" "web_sg" {
  name   = "web-server-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Create the EC2 instance, referencing the security group above
resource "aws_instance" "web_server" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  tags = {
    Name        = "web-server"
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}
```

Notice how `aws_instance.web_server` references `aws_security_group.web_sg.id`. Terraform sees this and automatically knows to create the security group **before** the EC2 instance. You do not need to specify the order — Terraform figures it out from the references.

---

## Summary

- A resource block defines a piece of infrastructure you want Terraform to create and manage
- Every resource block has two labels: the **resource type** (what to create) and the **local name** (how to refer to it in your code)
- The combination of type and name forms the **resource address** used in state, plans, and CLI commands
- Arguments inside the block can be simple key-value pairs or nested blocks
- Arguments are either required, optional, or computed (set by the provider after creation)
- Terraform compares your resource blocks against the current state to decide what actions to take
- Use lowercase, underscore-separated, descriptive names for clarity

In the next file, we explore **meta-arguments** — special arguments like `count`, `for_each`, `depends_on`, and `provider` that change how Terraform handles a resource block at a fundamental level.
