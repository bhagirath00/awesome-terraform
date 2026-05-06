# Terraform vs Ansible

## Overview

Terraform and Ansible are both very popular tools in the DevOps world, and they are often mentioned together. Many engineers use both in the same project. But they serve different purposes and are designed for different problems.

Understanding the difference between Terraform and Ansible is important because:
- You will frequently see both tools used together in real-world environments
- Interview questions often ask about this comparison
- Choosing the right tool for the right job makes your workflow much smoother

---

## What is Ansible?

**Ansible** is an open-source automation tool created by Red Hat (now owned by IBM). It is designed primarily for:

- **Configuration management** — installing software, editing config files, setting up services on servers
- **Application deployment** — pushing new application code to servers
- **Task automation** — running commands across many servers at once
- **Orchestration** — coordinating complex multi-step processes

Ansible uses **YAML** files called **playbooks** to define what tasks should be performed on which servers.

Here is a simple Ansible playbook that installs Nginx on a server:

```yaml
---
- name: Install and start Nginx
  hosts: web_servers
  become: yes

  tasks:
    - name: Install Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Start and enable Nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Copy configuration file
      copy:
        src: nginx.conf
        dest: /etc/nginx/nginx.conf
      notify: Restart Nginx

  handlers:
    - name: Restart Nginx
      service:
        name: nginx
        state: restarted
```

Ansible connects to your existing servers over SSH and runs these tasks. It does not need any agent software installed on the servers — just SSH access.

---

## What is Terraform?

Terraform is designed primarily for:

- **Infrastructure provisioning** — creating and managing cloud resources (servers, networks, databases, storage, DNS, etc.)
- **Infrastructure lifecycle management** — creating, updating, and destroying infrastructure
- **Multi-cloud management** — managing resources across different cloud providers from one tool

Terraform does NOT install software on your servers. It just creates the servers (and other infrastructure). Once the infrastructure exists, a tool like Ansible can take over to configure and manage what's running on those servers.

---

## The Core Difference: Provisioning vs Configuration Management

This is the most important thing to understand:

| | Terraform | Ansible |
|---|---|---|
| **Primary Purpose** | Create and manage infrastructure | Configure existing infrastructure |
| **What it does** | Creates servers, networks, databases | Installs software, edits files, manages services |
| **When it runs** | When infrastructure needs to change | When software/config needs to change |
| **What it talks to** | Cloud APIs (AWS, Azure, GCP) | SSH on existing servers |

Think of it this way:
- **Terraform** builds the house (creates the servers and network)
- **Ansible** furnishes the house (installs software and configures the servers)

---

## Declarative vs Imperative Approach

### Terraform — Declarative

Terraform is **declarative**. You describe the desired end state, and Terraform figures out how to get there.

```hcl
# I want 3 EC2 instances
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
}
```

If you already have 1 instance and run this, Terraform creates 2 more to reach the desired state of 3.
If you already have 5 instances and run this, Terraform destroys 2 to reach the desired state of 3.

Terraform always moves toward the desired state, regardless of where you currently are.

---

### Ansible — Imperative (mostly)

Ansible is mostly **imperative**. You describe the steps to execute, in order.

```yaml
- name: Create directory
  file:
    path: /var/www/html
    state: directory

- name: Copy application files
  copy:
    src: ./app/
    dest: /var/www/html/

- name: Restart web server
  service:
    name: nginx
    state: restarted
```

Ansible executes these tasks in order, top to bottom. Some Ansible modules are idempotent (running them again doesn't cause problems), but the overall approach is procedural — you define the steps.

---

## State Management

### Terraform

Terraform maintains a **state file** that tracks everything it has created. When you run Terraform again, it compares the current state with your configuration and calculates the minimum changes needed.

This means Terraform is very efficient — it only changes what needs to change.

### Ansible

Ansible is **stateless** — it has no memory of previous runs. Every time you run a playbook, Ansible executes all the tasks again.

Some Ansible modules are idempotent (e.g., "install Nginx" won't reinstall if Nginx is already installed), but Ansible itself doesn't track what it has done in the past.

---

## Agent vs Agentless

### Ansible — Agentless

Ansible connects to servers over **SSH** (for Linux) or **WinRM** (for Windows). No software needs to be installed on the target servers. This makes Ansible very easy to get started with.

### Terraform — API-based

Terraform communicates directly with **cloud APIs** over HTTPS. It doesn't connect to individual servers at all (for the provisioning step). It talks to AWS, Azure, or GCP's control plane to create resources.

---

## Mutable vs Immutable Infrastructure

### Mutable Infrastructure (Ansible's typical approach)

With Ansible, you update servers in place. If you need to upgrade Nginx from version 1.18 to 1.24, you run an Ansible playbook that upgrades it on the running server.

**Advantages:** Fast, no downtime needed
**Disadvantages:** Server state can drift over time, harder to reproduce exactly

### Immutable Infrastructure (Terraform's typical approach)

With Terraform, when you need to change a server configuration, you typically:
1. Create a new server with the new configuration
2. Route traffic to the new server
3. Destroy the old server

**Advantages:** Servers are always in a known, clean state — no configuration drift
**Disadvantages:** Requires more infrastructure planning (load balancers, rolling deployments)

---

## When to Use Terraform

Use Terraform when you need to:
- Create cloud resources (EC2, VPC, RDS, S3, etc.)
- Manage DNS records, CDN configurations
- Set up Kubernetes clusters
- Create and manage databases, queues, storage
- Manage GitHub repositories, IAM users, CI/CD pipelines

---

## When to Use Ansible

Use Ansible when you need to:
- Install software on servers (Nginx, Docker, Java, etc.)
- Edit configuration files on servers
- Deploy application code
- Run tasks across many servers simultaneously
- Orchestrate complex multi-step operations
- Manage on-premises servers that aren't in any cloud

---

## Using Terraform and Ansible Together

In real-world DevOps environments, Terraform and Ansible are very commonly used together. They complement each other perfectly:

**Step 1 — Terraform creates the infrastructure:**
```hcl
resource "aws_instance" "app_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.medium"
  key_name      = "my-key"
}

output "server_ip" {
  value = aws_instance.app_server.public_ip
}
```

**Step 2 — Ansible configures the server:**
```yaml
# inventory file: hosts
[app_servers]
54.23.45.67  # IP from Terraform output

# playbook.yml
- hosts: app_servers
  tasks:
    - name: Install Docker
      apt:
        name: docker.io
        state: present

    - name: Deploy application
      docker_container:
        name: myapp
        image: myapp:latest
        state: started
```

Terraform provisions the server. Ansible installs Docker and deploys the application onto it. Each tool does what it's best at.

---

## Quick Comparison Summary

| Feature | Terraform | Ansible |
|---|---|---|
| **Primary use** | Infrastructure provisioning | Configuration management |
| **Language** | HCL (declarative) | YAML (imperative) |
| **State management** | Yes, maintains state file | No state, stateless |
| **Agent required** | No | No (agentless via SSH) |
| **Cloud support** | Excellent, 3000+ providers | Good, but not primary focus |
| **Server configuration** | Not designed for this | Excellent |
| **Learning curve** | Moderate | Easy to moderate |
| **Community** | Large, growing | Very large, mature |
| **Best for** | Cloud infrastructure | Software installation & config |

---

## Summary

- Terraform and Ansible solve different problems — Terraform provisions infrastructure, Ansible configures it
- Terraform is declarative and stateful; Ansible is mostly imperative and stateless
- Both tools are agentless in different ways — Terraform talks to cloud APIs, Ansible talks to servers over SSH
- In practice, many teams use both tools together, with Terraform creating servers and Ansible configuring them
- Knowing when to use each tool is an important skill for any DevOps engineer

In the next file, we compare Terraform with AWS CloudFormation — the native IaC tool for AWS.
