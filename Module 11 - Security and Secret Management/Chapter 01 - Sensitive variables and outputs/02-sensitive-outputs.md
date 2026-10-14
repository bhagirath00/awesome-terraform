# Protecting the Finish Line: Sensitive Outputs

IAM policies, S3 buckets, and VPCs are often used as inputs for other processes, and their metadata is exposed via Terraform **Outputs**. While most outputs are public (like a VPC ID), some contain high-value data such as database connection strings, generated passwords, or private key material. If these are not managed correctly, they will be printed in clear text at the end of every `terraform apply`, polluting CI/CD logs and exposing secrets to anyone with read access to the console. This chapter provide a definitive deep dive into the `sensitive` attribute for outputs, the technical mechanics of sensitivity propagation, and the patterns for securely consuming sensitive data downstream.

## 1. Technical Mechanics: The `sensitive` Flag for Outputs

An output can be marked as sensitive to prevent its value from being displayed in the terminal.

```hcl
output "database_url" {
  value     = "postgres://${aws_db_instance.main.username}:${var.db_password}@${aws_db_instance.main.endpoint}/app_db"
  sensitive = true
}
```

### 1.1 The Terminal Experience
When you run `terraform apply`, standard outputs are printed clearly:
`vpc_id = "vpc-012345"`
`public_ip = "1.2.3.4"`

However, the `database_url` will be displayed as:
`database_url = <sensitive>`

## 2. The "Infection" Rule: Mandatory Sensitivity

Terraform implements a safety check to prevent accidental disclosure. If an output's value is derived from an attribute or variable that is **already** marked as sensitive, Terraform **requires** that the output also be marked as sensitive.

- **Scenario:** You have a `variable "token"` which is `sensitive = true`.
- **Result:** If you try to create an output `output "token" { value = var.token }` without setting `sensitive = true`, Terraform will fail during the `plan` phase with a "Sensitive Output" error.

## 3. Consuming Sensitive Outputs

While the value is hidden in the terminal, you still need to use it in other scripts or pipelines.

### 3.1 The JSON Extraction Pattern
The most common way to get a sensitive value is to request it in JSON format.
```bash
terraform output -json database_url
```
This will return the raw value. Because this is a deliberate programmatic request, Terraform assumes the user knows what they are doing. 

**DevOps Warning:** In CI/CD, you should immediately pipe this output into a variable rather than printing it to the screen to prevent it from entering the logs.

## 4. Professional Use Cases

- **DB Connection Strings:** Building the full URI with the password embedded.
- **TLS/SSL Private Keys:** If Terraform generates a private key via the `tls_private_key` resource, the resulting PEM must be marked sensitive.
- **SSM Parameters:** Exporting values that will be stored in AWS Systems Manager Parameter Store of the SecureString type.

## 5. Summary

Sensitive outputs are a critical boundary in the Terraform security model. By ensuring that private information is masked during the final report of an "Apply" cycle, you protect your environment from accidental disclosure and log-based credential harvesting. While they do not provide cryptographic security (the values are still in the state!), they are a vital operational "Guardrail" that ensures your secrets are only visible through deliberate, audited requests. Mastering the logic of sensitivity propagation is a hallmark of a professional DevOps engineer who values both automation and strict data privacy.
