# Infrastructure as Code (IaC) — Overview

## What is Infrastructure?

Before we understand Infrastructure as Code, let's first understand what "infrastructure" means in the context of software and DevOps.

Infrastructure refers to all the underlying components that are needed to run an application. This includes:

- **Servers** — the machines (physical or virtual) that run your application
- **Networks** — connections between servers, firewalls, load balancers
- **Storage** — disks, databases, file systems where your data lives
- **Operating Systems** — the software layer that manages hardware resources
- **Configuration** — settings, environment variables, installed software

In the early days of computing, all of this was managed manually. Engineers would physically go into a data center, set up machines, plug in cables, and configure everything by hand. This worked when applications were small and changes were rare.

But as applications grew larger and more complex, managing infrastructure manually became a serious problem.

---

## The Problem with Manual Infrastructure Management

Imagine you work at a company that runs an online shopping website. Your website is growing fast, and you need to:

- Set up 10 new servers every time traffic increases
- Configure each server exactly the same way
- Make sure all servers have the same software versions
- Repeat this across 3 different environments — development, staging, and production

If you do all of this manually:

1. **It takes a lot of time** — setting up even one server can take hours
2. **Human errors happen** — you might forget a step or make a typo in a config file
3. **Environments become inconsistent** — server 5 might have a slightly different config than server 6
4. **No one knows what changed** — if something breaks, there's no record of what was done
5. **Knowledge lives in one person's head** — if that person leaves, the team struggles

This is exactly the problem that Infrastructure as Code was created to solve.

---

## What is Infrastructure as Code?

**Infrastructure as Code (IaC)** is the practice of defining and managing your infrastructure using code — just like you write code to build an application.

Instead of clicking through a web console or running commands manually, you write a configuration file that describes exactly what infrastructure you want. A tool then reads that file and creates or updates the infrastructure automatically.

Think of it like a **recipe**. A recipe tells you exactly what ingredients you need and what steps to follow. Anyone who follows the same recipe will produce the same dish. IaC works the same way — your configuration file is the recipe, and the tool creates the exact same infrastructure every time.

### A Simple Example

Without IaC, to create an AWS EC2 server, you might:
1. Log into the AWS Console
2. Click "Launch Instance"
3. Choose an operating system
4. Choose an instance type
5. Configure networking, storage, security groups
6. Click "Launch"

With IaC, you write a file like this:

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server"
  }
}
```

Run one command, and the server is created — exactly as described, every single time.

---

## Core Principles of IaC

### 1. Declarative vs Imperative

There are two ways to write IaC:

**Declarative** — You describe *what* you want, not *how* to get there. The tool figures out the steps.

Example: "I want 3 servers running Ubuntu 22.04"

The tool looks at the current state, figures out what needs to change, and makes it happen.

**Imperative** — You describe *how* to do it, step by step.

Example: "First check if servers exist. If not, create them. Then install Ubuntu. Then configure networking..."

Most modern IaC tools like Terraform use the declarative approach because it is simpler and less error-prone.

---

### 2. Idempotency

This is one of the most important concepts in IaC.

**Idempotency** means that running the same configuration multiple times always produces the same result. It doesn't matter if you run it once or ten times — the outcome is always the same.

For example, if your configuration says "I want one server named web-server", and you already have that server:
- Running the configuration again will NOT create a second server
- It will see that the server already exists and do nothing

This makes IaC safe to run repeatedly without fear of creating duplicate resources.

---

### 3. Version Control

Since your infrastructure is defined in code, you can store it in a version control system like Git. This gives you:

- **History** — you can see every change ever made to your infrastructure
- **Collaboration** — multiple team members can work on infrastructure together
- **Rollback** — if something breaks, you can revert to a previous version
- **Code Review** — infrastructure changes go through the same review process as application code

---

### 4. Automation

IaC enables full automation of infrastructure management. You can:

- Automatically create infrastructure when a new project starts
- Automatically scale up servers when traffic increases
- Automatically destroy environments when they're no longer needed
- Integrate infrastructure changes into your CI/CD pipeline

---

## Benefits of Infrastructure as Code

### Speed and Efficiency

Manual infrastructure setup that takes days can be done in minutes with IaC. You write the configuration once and apply it instantly — whether you're creating 1 server or 100.

### Consistency and Reliability

Every environment is created from the same code, so they are always identical. No more "it works on staging but not on production" problems caused by configuration differences.

### Reduced Risk

Because everything is automated and tested, the chance of human error is much lower. Changes are predictable and repeatable.

### Cost Savings

With IaC, you can easily spin up and tear down environments. Instead of keeping a staging environment running 24/7, you can create it when needed and destroy it when done — saving money on cloud costs.

### Documentation

Your infrastructure configuration files are living documentation. Anyone on the team can read the code and understand exactly what infrastructure exists and how it's configured.

### Disaster Recovery

If your infrastructure is destroyed (hardware failure, accidental deletion), you can recreate everything from your code in minutes instead of days.

---

## How IaC Fits into DevOps

IaC is a foundational practice in DevOps. DevOps is about breaking down the wall between development and operations teams — and IaC is one of the key tools that makes this possible.

With IaC:

- **Developers** can provision their own environments without waiting for the operations team
- **Operations teams** can manage infrastructure with the same tools and workflows as developers
- **Both teams** can collaborate on infrastructure using pull requests and code reviews

IaC also enables the key DevOps practice of **continuous delivery** — where code changes are automatically tested and deployed. Without IaC, you can automate the application deployment but still have manual, slow, error-prone infrastructure setup. IaC completes the automation story.

---

## Common IaC Tools

There are several popular IaC tools used in the industry today:

| Tool | Type | Best For |
|---|---|---|
| **Terraform** | Declarative, multi-cloud | Provisioning any cloud infrastructure |
| **Ansible** | Imperative, agentless | Configuration management, software installation |
| **AWS CloudFormation** | Declarative, AWS-only | AWS infrastructure management |
| **Pulumi** | Declarative, uses real code | Developers who prefer Python/TypeScript/Go |
| **Chef / Puppet** | Imperative, agent-based | Configuration management on servers |

In this course, we focus on **Terraform** — the most widely used IaC tool across the industry, supporting all major cloud providers.

---

## Summary

- Infrastructure as Code is the practice of managing infrastructure using configuration files instead of manual processes
- It solves problems like slow setup, human errors, inconsistent environments, and lack of visibility
- Key principles include declarative syntax, idempotency, version control, and automation
- IaC brings speed, consistency, reliability, and cost savings to infrastructure management
- It is a core DevOps practice that enables teams to move faster and more safely

In the next file, we will look at how traditional infrastructure management works and compare it directly with the IaC approach.
