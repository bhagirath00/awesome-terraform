# Quality Assurance: Requirements for Publishing Modules

Publishing a module, whether to the Public Terraform Registry or an internal Private Registry, requires a level of engineering rigor beyond simple local development. A published module is a "Platform Product" that other engineers will depend on. To ensure stability, security, and usability, HashiCorp enforces specific technical requirements, and the DevOps community has established additional standards for high-quality modules. This chapter details the mandatory and recommended requirements for publishing professional Terraform modules.

## 1. Mandatory File Structure

The Terraform Registry will not index a module unless it follows the "Standard Module Structure." This isn't just a suggestion; it is a technical requirement for the registry's parsing engine.

### 1.1 The Primary Triad
- **`main.tf`**: Must contain the primary resource logic.
- **`variables.tf`**: Must define all inputs. Every variable **should** have a description, as this is rendered in the registry UI.
- **`outputs.tf`**: Must define all exported data.

### 1.2 The Metadata Layer
- **`versions.tf`**: This file is critical for publication. It must specify the `required_version` of Terraform and the `required_providers` with their version constraints. This prevents your module from breaking when a user has a different environment.
- **`README.md`**: The landing page for your module. It must explain what the module does and show a minimal usage example.

## 2. Resource Naming Conventions

For the Public Registry, the repository name in your Version Control System (VCS) must follow a very specific pattern: `terraform-<PROVIDER>-<NAME>`.

**Examples:**
- `terraform-aws-vpc`
- `terraform-azurerm-compute`
- `terraform-google-network`

If your repository is named anything else (e.g., `my-cool-module`), the public registry will refuse to index it. For private registries, this convention is often recommended but not always strictly enforced.

## 3. Documentation Excellence

A published module lives and dies by its documentation. Because users cannot easily see your internal code, they rely on the documentation to understand the "Contract."

### 3.1 Descriptions are Mandatory
Every variable and output in a published module must have the `description` attribute.
```hcl
variable "vpc_id" {
  type        = string
  description = "The ID of the VPC where the application will be deployed."
}
```
The registry uses these descriptions to build the "Inputs" and "Outputs" tables. Without them, your module will appear unprofessional and will be difficult to use.

### 3.2 Licensing
Every published module should include a `LICENSE` file. For public modules, the **Apache 2.0 License** or **MIT License** are the most common choices. This provides legal clarity to organizations on how they can use and modify your code.

## 4. Constraint Management in `versions.tf`

When publishing, you must be careful not to over-constrain your module.

- **Too Strict:** Requiring `version = "1.5.0"` means your module won't work for anyone on 1.4.x or 1.6.x.
- **Too Loose:** Requiring `>= 0.12` might cause your module to fail when a user tries to use a feature only available in 1.0.

**Best Practice:** Use the "Pessimistic Constraint" operator (`~>`).
```hcl
terraform {
  required_version = "~> 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## 5. Security and Secret Auditing

Before publishing, you must perform a "Scrub" of your codebase.

- **No Hardcoded Secrets:** Check for API keys, passwords, or tokens used during development.
- **No Hardcoded IAM ARNs:** Ensure that IAM policies use dynamic variables for Account IDs and regions rather than hardcoded strings from your own account.
- **Sensitive Flags:** Ensure that any sensitive outputs are marked with `sensitive = true`.

## 6. Testing before Publication

You should never publish a module that hasn't been tested in a real cloud environment.

### 6.1 `terraform validate`
A static analysis tool that checks your HCL for syntax errors and internal logic consistency. Your CI/CD pipeline should run this on every commit.

### 6.2 `terraform test`
Introduced in Terraform 1.6, the native testing framework allows you to write test files that provision the module, verify the outputs, and then destroy the resources. This ensures that the combined resources in your module actually work as expected.

## 7. The Examples Directory

Every high-quality module includes an `examples/` directory. Each subfolder in `examples/` should be a self-contained, runnable configuration that demonstrates a specific use case (e.g., `examples/basic-usage` and `examples/advanced-networking`). The registry uses these examples to show "Usage" snippets on the module's homepage.

## 8. Summary

Publishing a module is the transition from "Scripting" to "Software Engineering." By following the standard file structure, naming conventions, and documentation requirements, you create an infrastructure component that is reliable, professional, and easy for the community (or your colleagues) to adopt. A module that meets all these requirements is a robust foundation upon which complex, enterprise-scale infrastructure can be built with confidence.
