# Protecting the Perimeter: Sensitive Input Variables

Infrastructure as Code often requires the handling of high-value metadata such as database passwords, TLS private keys, and API tokens. If these values are treated as standard strings, they risk being exposed in the terminal output, build logs, and the Terraform console. The `sensitive` flag for variables is the primary defensive mechanism for ensuring that private data is "Masked" by the Terraform CLI. This chapter provide a definitive deep dive into the technical mechanics of the sensitive flag, the limitations of its protection, and the professional patterns for passing secret data into your modules securely.

## 1. Technical Mechanics: The `sensitive` Flag

Starting from Terraform 0.14, you can mark any input variable as `sensitive`. This tells Terraform to obfuscate the value during the `plan` and `apply` phases.

```hcl
variable "db_password" {
  type      = string
  sensitive = true
  description = "The master password for the RDS instance"
}
```

### 1.1 The Obfuscation Logic
When a variable is marked as sensitive:
- **CLI Output:** Instead of showing the actual value (e.g., `P@ssw0rd123`), the Terraform terminal will display `(sensitive value)`.
- **JSON Output:** If you run `terraform show -json`, the value will be replaced with a placeholder.

## 2. Passing Sensitive Values to Terraform

Since sensitive values should never be committed to your HCL code (in `terraform.tfvars`), you must use one of the following "Out-of-Band" methods for passing them in.

### 2.1 The Environment Variable Pattern (`TF_VAR_`)
This is the standard for CI/CD systems like GitHub Actions or Jenkins.
```bash
export TF_VAR_db_password="actual-secure-password"
terraform plan
```
Terraform automatically maps environment variables prefixed with `TF_VAR_` to the corresponding variable in your code.

### 2.2 The Manual Interaction Pattern
If you run Terraform locally and do not provide a value for a sensitive variable, Terraform will prompt you for it in the terminal. Your keystrokes will be hidden (like a standard password prompt), providing a secure manual entry method.

## 3. Propagation and Cascading Sensitivity

Terraform implements a "Infection" model for sensitive data. 
- If a sensitive variable is used as an input for a resource attribute, or as an input to a child module, that attribute or module output is **automatically** marked as sensitive.
- This prevents a scenario where you pass a sensitive password to a resource and then accidentally print that password back out through a standard output.

```hcl
output "database_details" {
  value = {
    host     = aws_db_instance.main.address
    password = var.db_password # This output is now infected and MUST be marked sensitive
  }
  sensitive = true
}
```

## 4. The Critical Limitation: Plain Text State

It is a fundamental technical requirement for every DevOps engineer to understand that the `sensitive` flag is **only for the UI**.
- **The State File:** The `terraform.tfstate` file **ALWAYS** contains the plain-text value of every sensitive variable and resource attribute. 
- **The Risk:** If an attacker gets access to your state file, the `sensitive` flag provides ZERO protection. This is why remote state storage in S3 with KMS encryption and limited IAM access is a non-negotiable security requirement.

## 5. Validation and Constraints

You can combine the `sensitive` flag with `validation` blocks to ensure that even though the value is hidden, it still meets your organizational security standards (e.g., minimum character length).

```hcl
variable "admin_key" {
  type      = string
  sensitive = true
  
  validation {
    condition     = length(var.admin_key) >= 16
    error_message = "The administrative key must be at least 16 characters long."
  }
}
```

## 6. Summary

The `sensitive` flag is the first line of defense in Terraform security. By masking high-value data from developers' eyes and CI/CD logs, it significantly reduces the surface area for accidental data leakage. While it does not secure the underlying state file, its role in the "Infection" model ensures that sensitivity is preserved as data flows through your module graph. Mastering the use of the `sensitive` attribute, combined with secure environment variable injection and state encryption, is a mandatory skill for any engineer responsible for the integrity and privacy of enterprise cloud foundations.
