<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://cdn.simpleicons.org/terraform/white" />
    <img alt="Terraform Logo" src="https://cdn.simpleicons.org/terraform/black" width="60" />
  </picture>
</p>

<h1 align="center">Awesome Terraform</h1>

<p align="center">
  <a href="https://awesome.re"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=awesomelists" /><img alt="awesome" src="https://shieldcn.dev/badge/awesome.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=awesomelists" /></picture></a>
  <a href="https://www.terraform.io/"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/Terraform-1.7+.svg?variant=outline&amp;mode=dark&amp;font=geist&amp;logo=terraform" /><img alt="Terraform" src="https://shieldcn.dev/badge/Terraform-1.7+.svg?variant=outline&amp;mode=light&amp;font=geist&amp;logo=terraform" /></picture></a>
  <a href="LICENSE"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=dark&amp;font=geist" /><img alt="license" src="https://shieldcn.dev/badge/license-MIT.svg?variant=outline&amp;mode=light&amp;font=geist" /></picture></a>
  <a href="https://github.com/bhagirath00/awesome-terraform"><picture><source media="(prefers-color-scheme: dark)" srcset="https://shieldcn.dev/views/repo/bhagirath00/awesome-terraform.svg?variant=outline&amp;mode=dark&amp;font=geist" /><img alt="views" src="https://shieldcn.dev/views/repo/bhagirath00/awesome-terraform.svg?variant=outline&amp;mode=light&amp;font=geist" /></picture></a>
</p>

## Table of Contents

- [Introduction](#introduction)
- [Why Terraform for DevOps?](#why-terraform-for-devops)
- [Learning Modules](#learning-modules)
  - [Phase 1: Foundations & Language Syntax](#phase-1-foundations--language-syntax)
  - [Phase 2: State, Resources & Modules](#phase-2-state-resources--modules)
  - [Phase 3: Workspaces & Cloud Provisioning](#phase-3-workspaces--cloud-provisioning)
  - [Phase 4: CI/CD, Security & Enterprise Scale](#phase-4-cicd-security--enterprise-scale)
- [Resources](#resources)
- [Contributing](#contributing)
- [Security](#security)
- [License](#license)

---

## Introduction

Welcome to **Awesome Terraform**! This repository serves as an exhaustive, hands-on architectural roadmap and reference manual designed for site reliability engineers (SREs), DevOps practitioners, and cloud architects.

> **"Infrastructure is code. Treat it like software: version it, test it, review it, and automate it."**

---

## Why Terraform for DevOps?

Terraform is the industry-standard declarative Infrastructure as Code engine:

| Capability | Importance in DevOps |
| :--- | :--- |
| **Multi-Cloud Portability** | Single consistent workflow across AWS, Azure, GCP, Kubernetes, and 3,000+ providers |
| **Declarative Desired State** | Define the target topology; Terraform computes the execution plan to reconcile drift |
| **Stateful Management** | Continuous drift detection, resource mapping, and deterministic dependency resolution |
| **Modular & Reusable** | Package repeatable architecture blueprints into versioned, testable modules |

---

## Learning Modules

Organized into 12 comprehensive modules covering core HCL, cloud orchestration, GitOps automation, and security hardening.

### Phase 1: Foundations & Language Syntax

| Module | Description |
| :--- | :--- |
| [**Module 01 - Introduction to Terraform**](./Module%2001%20-%20Introduction%20to%20Terraform/) | IaC paradigms, declarative vs imperative, Terraform core architecture, and execution lifecycle |
| [**Module 02 - Installation & Setup**](./Module%2002%20-%20Installation%20%26%20Setup/) | CLI installation, provider configurations, local workstations, and editor tooling |
| [**Module 03 - HCL Fundamentals**](./Module%2003%20-%20HCL%20Fundamentals/) | HCL syntax, input variables, output values, local values, expressions, and built-in functions |

### Phase 2: State, Resources & Modules

| Module | Description |
| :--- | :--- |
| [**Module 04 - Terraform State**](./Module%2004%20-%20Terraform%20State/) | Understanding `tfstate`, remote backends (S3/DynamoDB), state locking, state inspection, and disaster recovery |
| [**Module 05 - Core Resources and Data Sources**](./Module%2005%20-%20Core%20Resources%20and%20Data%20Sources/) | Resource blocks, meta-arguments, lifecycle rules, data sources, and explicit dependencies (`depends_on`) |
| [**Module 06 - Terraform Modules**](./Module%2006%20-%20Terraform%20Modules/) | Module architecture, root vs child modules, public registry consumption, composition, and publishing |

### Phase 3: Workspaces & Cloud Provisioning

| Module | Description |
| :--- | :--- |
| [**Module 07 - Workspaces and Environments**](./Module%2007%20-%20Workspaces%20and%20Environments/) | CLI workspaces, directory-based environment isolation (dev/stage/prod), and `.tfvars` hierarchies |
| [**Module 08 - Provisioning AWS Infrastructure**](./Module%2008%20-%20Provisioning%20AWS%20Infrastructure/) | Real-world AWS provisioning: VPC, Subnets, Gateways, EC2, ASG, ALB/NLB, S3, RDS, IAM, and EKS clusters |
| [**Module 09 - Advanced HCL and Meta-Arguments**](./Module%2009%20-%20Advanced%20HCL%20and%20Meta-Arguments/) | `count` vs `for_each`, dynamic blocks, `lifecycle` rules, `moved` blocks, and native `import` blocks |

### Phase 4: CI/CD, Security & Enterprise Scale

| Module | Description |
| :--- | :--- |
| [**Module 10 - Terraform with CI-CD**](./Module%2010%20-%20Terraform%20with%20CI-CD/) | Automated CI/CD pipelines, GitHub Actions workflows, Jenkins pipelines, and GitOps with Atlantis |
| [**Module 11 - Security and Secret Management**](./Module%2011%20-%20Security%20and%20Secret%20Management/) | Sensitive variables, HashiCorp Vault integration, AWS Secrets Manager, and IAM least privilege |
| [**Module 12 - Terraform at Scale**](./Module%2012%20-%20Terraform%20at%20Scale/) | Terragrunt DRY architecture, large codebase patterns, Terraform Cloud/Enterprise, and drift detection |

---

## Resources

### Official Documentation

| Resource | Link |
| :--- | :--- |
| Terraform Documentation | [developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform) |
| Terraform Registry | [registry.terraform.io](https://registry.terraform.io/) |
| Terraform Best Practices | [terraform-best-practices.com](https://www.terraform-best-practices.com/) |
| OpenTofu (CNCF Project) | [opentofu.org](https://opentofu.org/) |

### Recommended Tools

- **tflint**: Pluggable Terraform linter detecting syntax and provider errors.
- **checkov / tfsec**: Static analysis security scanners for infrastructure code.
- **infracost**: Cloud cost estimation for Terraform pull requests.
- **terragrunt**: Thin wrapper providing extra tools for keeping configurations DRY.
- **atlantis**: GitOps pull request automation for Terraform.

---

## Contributing

Contributions are welcome! Please read our [Contributing Guidelines](CONTRIBUTING.md) before submitting a Pull Request.

---

## Security

Please review our [Security Policy](SECURITY.md) for state file security, sensitive variable management, and vulnerability reporting.

---

## License

This educational content is open source and available under the **MIT License**. See [LICENSE](LICENSE) for details.

<p align="center">
  <strong>Happy Provisioning!</strong>
</p>
