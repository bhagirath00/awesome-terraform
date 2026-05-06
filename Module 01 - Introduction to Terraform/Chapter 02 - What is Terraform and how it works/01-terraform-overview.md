# What is Terraform?

## Introduction

Terraform is an open-source Infrastructure as Code tool created by **HashiCorp** in 2014. It allows you to define, provision, and manage cloud infrastructure using a simple configuration language called **HCL (HashiCorp Configuration Language)**.

With Terraform, you can manage infrastructure across all major cloud providers — AWS, Azure, Google Cloud, and hundreds of others — using the same tool and the same workflow.

As of today, Terraform is the most widely used IaC tool in the industry. It is used by companies of all sizes — from small startups to large enterprises like Airbnb, Uber, Stripe, and many others.

---

## Why Terraform?

Before Terraform existed, each cloud provider had its own tool for managing infrastructure:

- AWS had **CloudFormation**
- Azure had **ARM Templates**
- Google Cloud had **Deployment Manager**

The problem was that most real-world companies use more than one cloud provider or more than one service. If you needed to manage AWS AND an external DNS provider AND a GitHub repository, you needed three different tools.

Terraform solved this by providing **one single tool** that works with everything through a system called **providers**.

---

## Key Features of Terraform

### 1. Multi-Cloud and Multi-Provider Support

Terraform works with:
- **Cloud providers** — AWS, Azure, GCP, DigitalOcean, Oracle Cloud
- **Infrastructure services** — Kubernetes, Docker, Helm
- **Monitoring tools** — Datadog, PagerDuty, New Relic
- **Version control** — GitHub, GitLab, Bitbucket
- **Databases** — MySQL, PostgreSQL, MongoDB Atlas
- **DNS providers** — Cloudflare, Route53
- **And 3000+ more** through the Terraform Registry

All of these are managed using the same language and the same commands.

---

### 2. Declarative Language

You tell Terraform *what* you want, not *how* to create it. Terraform figures out the steps on its own.

For example, if you want an AWS S3 bucket:

```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name"
}
```

You don't write the API calls. You don't write loops or conditionals to check if the bucket exists. You just declare what you want, and Terraform handles the rest.

---

### 3. Execution Plans

Before Terraform makes any changes to your real infrastructure, it shows you exactly what it is going to do. This is called a **plan**.

When you run `terraform plan`, you see output like:

```
+ aws_instance.web_server will be created
  + ami           = "ami-0c55b159cbfafe1f0"
  + instance_type = "t2.micro"
  + tags          = {
      + "Name" = "web-server"
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

You review the plan, confirm it looks correct, and then apply it. This safety step prevents accidental changes.

---

### 4. State Management

Terraform keeps track of the infrastructure it has created in a **state file**. This file is Terraform's memory — it knows what resources exist, what their properties are, and how they relate to each other.

When you run Terraform again, it compares the current state with your configuration and only makes the changes needed to reach the desired state.

We will cover state in detail in Module 4. For now, just understand that state is how Terraform knows what it has already created.

---

### 5. Resource Graph

Terraform understands the relationships and dependencies between your resources. It builds a dependency graph and creates resources in the correct order.

For example, to create an EC2 instance inside a VPC, Terraform knows it must:
1. First create the VPC
2. Then create the subnet
3. Then create the security group
4. Then create the EC2 instance

It figures this out automatically from your configuration — you don't have to specify the order manually.

---

### 6. Modules

Terraform allows you to package and reuse infrastructure configurations as **modules**. A module is like a function in programming — you write it once and use it many times.

For example, you could write a module that creates a complete web application stack (VPC + EC2 + RDS + Load Balancer). Then use that same module to create dev, staging, and production environments with just a few lines of code.

---

## How Terraform is Structured

### The Terraform Binary

Terraform is a single binary (executable file) that you download and install on your computer. When you run terraform commands, this binary does all the work.

### Configuration Files

Your infrastructure is described in `.tf` files. These are plain text files written in HCL. You can have one big file or many smaller files — Terraform reads them all together.

Typical files in a Terraform project:

```
my-terraform-project/
├── main.tf          # Main resources
├── variables.tf     # Input variables
├── outputs.tf       # Output values
├── providers.tf     # Provider configuration
└── terraform.tfvars # Variable values
```

### Providers

A provider is a plugin that allows Terraform to interact with a specific service. For example, the AWS provider knows how to talk to the AWS API to create and manage AWS resources.

You declare which providers you need in your configuration:

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
```

When you run `terraform init`, Terraform downloads the required providers automatically.

---

## The Terraform Workflow

The core Terraform workflow has three main steps. You will use these every day:

### Step 1: Write

Write your infrastructure configuration in `.tf` files. Describe the resources you want to create.

### Step 2: Plan

Run `terraform plan` to see what Terraform will do. Review the output carefully. Make sure it matches your expectations.

### Step 3: Apply

Run `terraform apply` to actually create or update the infrastructure. Terraform will show the plan one more time and ask for confirmation before making any changes.

There is also a fourth step used when you want to remove infrastructure:

### Step 4: Destroy

Run `terraform destroy` to tear down all the resources that Terraform created. Useful for cleaning up after testing or removing environments you no longer need.

---

## Terraform vs Manual Cloud Console

Let's see a real example to understand the difference.

**Goal:** Create an AWS EC2 instance with a specific security group

**Manual approach (AWS Console):**
1. Navigate to VPC → Security Groups
2. Click "Create security group"
3. Fill in name, description, VPC
4. Add inbound rules (HTTP port 80, SSH port 22)
5. Click Create
6. Navigate to EC2 → Instances
7. Click "Launch Instance"
8. Choose AMI
9. Choose instance type
10. Configure networking, attach the security group you just created
11. Add storage
12. Review and launch
13. Select or create a key pair
14. Launch

**Terraform approach:**

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-security-group"
  description = "Allow HTTP and SSH"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
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

resource "aws_instance" "web_server" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.web_sg.id]

  tags = {
    Name = "web-server"
  }
}
```

Run `terraform apply` and both resources are created in the correct order, automatically.

Want to do the same thing in a different region? Change `region = "us-east-1"` to `region = "eu-west-1"` and run again. Done.

Want 5 servers instead of 1? Add `count = 5` to the EC2 resource. Done.

---

## Terraform Open Source vs Terraform Cloud vs Terraform Enterprise

There are three versions of Terraform:

**Terraform Open Source (CLI)**
- Free and open source
- Runs on your local machine or CI/CD pipeline
- What we use throughout this course

**Terraform Cloud**
- SaaS product by HashiCorp
- Adds remote state storage, team collaboration, remote plan/apply
- Has a free tier for small teams

**Terraform Enterprise**
- Self-hosted version of Terraform Cloud
- For large organizations with strict security and compliance requirements

We will focus on the open-source CLI throughout this course, with an introduction to Terraform Cloud in Module 13.

---

## Terraform Versions

Terraform has been evolving rapidly. Key milestones:

- **0.x versions** — Early Terraform, many breaking changes
- **0.12** — Major improvement to HCL language (2019)
- **0.13** — Automatic provider installation from registry (2020)
- **1.0** — Stability guarantee, production-ready (2021)
- **1.5** — New `import` block for importing existing resources (2023)
- **1.6+** — Built-in testing framework

Always check the version being used in a project. You can specify required versions in your configuration to avoid compatibility issues.

---

## Summary

- Terraform is an open-source IaC tool by HashiCorp that lets you manage infrastructure using code
- It supports 3000+ providers — AWS, Azure, GCP, Kubernetes, GitHub, and more
- It uses a declarative language (HCL) where you describe what you want, not how to create it
- Key features include execution plans, state management, dependency graphs, and modules
- The core workflow is: Write → Plan → Apply (→ Destroy when done)
- Terraform is the industry standard for multi-cloud infrastructure management

In the next file, we will explore HCL — the language you use to write Terraform configurations.
