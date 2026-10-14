# Clean Infrastructure: Separation of Concerns in Terraform

The Principle of Separation of Concerns (SoC) is a design pattern that dictates each part of a system should address a separate, distinct concern. In Terraform, failing to separate your code (logic) from your configuration (data) results in brittle, difficult-to-maintain modules that cannot be easily shared across teams. A professional DevOps architecture ensures that building blocks are generic, environment-specific data is externalized, and state management is isolated from execution logic. This chapter provide a definitive deep dive into the three pillars of SoC in Infrastructure as Code and the technical patterns for building "Stateless" modules.

## 1. The Three Pillars of SoC

To achieve a scalable architecture, you must strictly decouple the following three elements:

### 1.1 The Code (The "What")
This consists of your reusable **Modules**. These should be generic templates that represent organizational standards (e.g., "The Standard Company S3 Bucket"). A well-designed module contains NO hardcoded names, CIDRs, or environment-specific tags.

### 1.2 The Configuration (The "How")
This consists of your **Variables**, **.tfvars files**, or **Terragrunt configs**. This is the data that tells the generic code *how* to behave in a specific context (e.g., "Deploy as 'Prod' size in 'us-west-2'").

### 1.3 The State (The "Where")
This is the **Persistence Layer** managed by your backend. It tracks the mapping between your code and the real cloud resources. 

## 2. The Dangers of "Intermingling"

When these concerns are mixed, the following architectural debt is incurred:
- **Hardcoding:** Putting `region = "us-east-1"` directly inside a VPC resource means that module can never be used for a Disaster Recovery site in `us-west-2` without a complete rewrite.
- **Provider Bloat:** Including an `aws` provider block inside a reusable module makes it impossible to use that module in a multi-region environment without causing initialization errors.

## 3. Professional Patterns for Separation

### 3.1 The Wrapper Module Pattern
Advanced teams create "Wrapper" (or "Composition") modules. 
1.  **Lower Level:** A generic "EC2" module in a registry.
2.  **Middle Level:** A "Company-Standard-EC2" wrapper that calls the generic module but enforces specific security group logic and tagging mandates.
3.  **Top Level:** The Environment folder that provides the specific AMI and Instance Type via variables.

### 3.2 Dynamic Data Fetching
Instead of passing IDs between modules via variables, use **Data Sources** to dynamically find existing infrastructure. This separates the "Concern" of networking from the "Concern" of computing.
```hcl
data "aws_vpc" "selected" {
  filter {
    name   = "tag:Environment"
    values = [var.environment]
  }
}
```

## 4. Ownership and Responsibility

Separation of Concerns also aligns with organizational boundaries.
- **Security Concerns:** (IAM, KMS, WAF) are managed in a restricted repo by the Security Team.
- **Platform Concerns:** (VPC, EKS, DNS) are managed by the Platform Team.
- **Application Concerns:** (Lambdas, S3 buckets for app data) are managed by the App Developers.

## 5. Summary

Separation of Concerns is the difference between "Spaghetti Code" and "Infrastructure as Code." By strictly decoupling generic logic from environment-specific data and isolating state management from the execution tier, you create a modular, testable, and highly resilient cloud foundation. Terraform's input/output variable system and its support for third-party wrappers like Terragrunt provide the technical framework necessary to implement these clean-code principles at scale. Mastering the separation of these three pillars is a mandatory requirement for any senior DevOps professional responsible for the long-term agility and governance of enterprise infrastructure.
