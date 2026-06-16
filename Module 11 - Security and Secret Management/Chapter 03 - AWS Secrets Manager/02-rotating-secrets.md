# Continuous Security: Automating Secret Rotation

The greatest risk of a secret is not its creation, but its persistence. If a password is never changed, a single leak can compromise a system permanently. AWS Secrets Manager's "killer feature" is its ability to automatically rotate secrets without any human intervention. By using a Lambda function to update both the secret store and the target service (like a database) simultaneously, you virtually eliminate the risk of long-term credential leakage. This chapter provide a definitive deep dive into the `aws_secretsmanager_secret_rotation` resource, the versioning label system (`AWSCURRENT`, `AWSPENDING`), and the technical patterns for managed and custom rotation.

## 1. Technical Mechanics: The Rotation Handshake

When rotation is triggered (either manually or on a schedule), AWS facilitates a complex "versioning dance."

1.  **Create Pending:** Secrets Manager creates a new version of the secret and labels it as `AWSPENDING`.
2.  **Invoke Lambda:** AWS calls a specialized Lambda function and passes the secret ID.
3.  **Update Target:** The Lambda logs into the target service (e.g., RDS) and changes the password to the new `AWSPENDING` value.
4.  **Test:** The Lambda verifies it can log in with the new password.
5.  **Finalize:** If successful, AWS moves the `AWSCURRENT` label to the new version and removes it from the old one.

## 2. Implementing Managed RDS Rotation

AWS provides pre-written Lambda functions for the most common database engines (MySQL, PostgreSQL, etc.).

```hcl
resource "aws_secretsmanager_secret_rotation" "db_rotation" {
  secret_id           = aws_secretsmanager_secret.db_password.id
  rotation_lambda_arn = "arn:aws:lambda:us-east-1:123456789012:function:aws-managed-rds-rotation"

  rotation_rules {
    automatically_after_days = 30
  }
}
```

## 3. The Custom Rotation Pattern

For non-standard services (e.g., an external SaaS API key), you must author your own Lambda function in Terraform and give Secrets Manager permission to invoke it.

```hcl
resource "aws_lambda_permission" "allow_secrets_manager" {
  statement_id  = "AllowExecutionFromSecretsManager"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.custom_rotator.function_name
  principal     = "secretsmanager.amazonaws.com"
}
```

## 4. Operational Best Practices: Versioning and Recovery

- **The Previous Version:** Secrets manager always keeps the `AWSPREVIOUS` version. If a rotation fails or an application crashes, you can manually revert to the previous working password in seconds.
- **Immediate Rotation:** In the event of a known leak, you can trigger an immediate rotation by calling the `RotateSecret` API. Terraform doesn't naturally "trigger" a one-time rotation; this is an operational task performed via CLI or Console, but the **Capability** for that rotation must be provisioned via Terraform.

## 5. Summary

Automated secret rotation is the foundation of a modern, zero-trust security posture. By ensuring that your high-value credentials are short-lived and automatically updated through secure Lambda handshakes, you remove the "Human Element" from the most sensitive part of your infrastructure. Terraform's ability to orchestrate the secret store, the rotation rules, and the necessary IAM permissions ensures that your organization's commitment to continuous security is perfectly enforced and easily audited. Mastering these rotation patterns is an essential skill for any engineer managing professional, high-compliance cloud foundations at scale.
