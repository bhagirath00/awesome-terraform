# Managing a Cloud Galaxy: Multi-Account Patterns with Terragrunt

In a modern enterprise, managing everything in a single AWS account is considered a significant security risk and an operational bottleneck. Instead, organizations adopt a Multi-Account Strategy—using separate accounts for Development, Staging, Production, Security, and Shared Services. Terragrunt is the ultimate tool for orchestrating this multi-account world, allowing you to manage hundreds of isolated "Cloud Silos" through a single, unified codebase. This chapter provide a definitive deep dive into IAM role assumption, account-level isolation strategies, and the technical mechanics of the `run-all` command for global infrastructure management.

## 1. The Necessity of Multi-Account Isolation

### 1.1 Blast Radius Reduction
If a developer accidentally initiates a destructive command in a shared account, they could take down Production. By placing Production in a completely separate AWS account with separate credentials, the "Blast Radius" of any single error is limited to that specific account boundary.

### 1.2 Resource Quotas and Limits
AWS imposes limits (e.g., number of VPCs, size of Lambda bundles) per account. A large organization will quickly hit these limits in a single account. Spreading resources across multiple accounts provides a higher aggregate ceiling for growth.

## 2. Technical Mechanics: IAM Role Assumption

Terragrunt can automatically assume an IAM role before running any Terraform commands. This means your CI/CD pipeline only needs one set of credentials (the "Identity Account") to manage many "Workload Accounts."

```hcl
# In the environment-level terragrunt.hcl
iam_role = "arn:aws:iam::${local.account_id}:role/terraform-executor"
```

### 2.1 The Seamless Authentication
When you run `terragrunt apply`, Terragrunt calls `sts:AssumeRole` on the provided ARN, fetches temporary credentials, and injects them into the environmental variables of the subsequent Terraform process. This ensures that the state and the resources are always handled in the context of the correct account.

## 3. Account-Level Configuration Inheritance

By using the "Folder-as-Data" pattern discussed in the previous chapter, you can manage account-specific metadata globally.

```hcl
# project/prod/account.hcl
locals {
  account_name   = "production-workloads"
  aws_account_id = "123456789012"
  aws_profile    = "prod"
}
```

Every module within the `prod/` folder will automatically inherit these settings, ensuring that they are provisioned in the correct account without any manual configuration changes.

## 4. Global Orchestration: The `run-all` Command

The "Killer Feature" of Terragrunt for multi-account management is the `run-all` command.

```bash
terragrunt run-all plan
```
- **Concurrency:** Terragrunt will traverse your entire folder structure, identify every module, and run the plan in parallel.
- **Dependency Awareness:** It identifies that the "Production Redis" in Account B depends on the "Production VPC" in Account A, and it will orchestrate the execution order across the different accounts automatically.

## 5. Security Guardrails

- **Allowed Account IDs:** In your root `terragrunt.hcl`, always use the `generate` block to inject `allowed_account_ids` into every AWS provider. 
- **The "Safety Switch":** This ensures that even if you accidentally run a production command from your local laptop with developer credentials, AWS will reject the request if the account ID doesn't match your code.

## 6. Summary

Multi-account management is the hallmark of enterprise-grade cloud foundations. By leveraging Terragrunt's native support for IAM role assumption, hierarchical account metadata, and global `run-all` orchestration, you transform the complexity of managing a "Cloud Galaxy" into a streamlined and highly secure operation. Terraform's integration with Terragrunt ensures that your account-level boundaries are versioned, peer-reviewed, and perfectly enforced, providing the ultimate isolation needed to protect your organization's most critical assets at scale. Mastering these multi-account patterns is an essential skill for any senior DevOps professional working in diverse, high-scale enterprise environments.
