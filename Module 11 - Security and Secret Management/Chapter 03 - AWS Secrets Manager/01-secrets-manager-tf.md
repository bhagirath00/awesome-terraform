# Native Security: Provisioning AWS Secrets Manager with Terraform

AWS Secrets Manager is a dedicated service for managing, rotating, and retrieving database credentials, API keys, and other secrets throughout their lifecycle. While HashiCorp Vault is ideal for multi-cloud environments, Secrets Manager is the professional choice for organizations running exclusively on AWS due to its native integration with RDS, Lambda, and IAM. In a professional DevOps environment, Secrets Manager eliminates the need for hardcoded `TF_VAR_` values in your CI/CD pipelines. This chapter provides a definitive deep dive into the `aws_secretsmanager_secret` and `aws_secretsmanager_secret_version` resources, and the technical patterns for managing and consuming secrets as code.

## 1. Technical Anatomy of Secrets Manager

Unlike a standard key-value store, Secrets Manager treats a "Secret" as a versioned object.

### 1.1 The Metadata (`aws_secretsmanager_secret`)
This resource defines the properties of the secret: its name, description, and the KMS key used for encryption. It does NOT contain the actual sensitive data.

### 1.2 The Value (`aws_secretsmanager_secret_version`)
This resource contains the actual sensitive payload (usually a JSON string). Terraform manages this as a separate resource to allow for version tracking and recovery.

```hcl
resource "aws_secretsmanager_secret" "db_password" {
  name        = "production/database/master-password"
  description = "Managed by Terraform"
}

resource "aws_secretsmanager_secret_version" "current" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = jsonencode({
    username = "admin"
    password = random_password.master.result
  })
}
```

## 2. Consuming Secrets with Data Sources

In a large organization, one Terraform project might **Create** a secret, while another project (e.g., the Application deployment) needs to **Read** it.

```hcl
data "aws_secretsmanager_secret" "db_secret" {
  name = "production/database/master-password"
}

data "aws_secretsmanager_secret_version" "db_creds" {
  secret_id = data.aws_secretsmanager_secret.db_secret.id
}

locals {
  creds = jsondecode(data.aws_secretsmanager_secret_version.db_creds.secret_string)
}

# The password is now available as local.creds.password
```

## 3. Security Hardening: Resource Policies

One of the greatest advantages of Secrets Manager is the ability to attach **Resource-based Policies**. This allows you to implement a data perimeter around your secrets.

- **Scenario:** You want to ensure that a secret can ONLY be read if the request originates from a specific VPC Endpoint.
- **Pattern:** Use the `aws_secretsmanager_secret_policy` resource to deny all access unless the `aws:SourceVpce` condition is met.

```hcl
resource "aws_secretsmanager_secret_policy" "restrict_to_vpc" {
  secret_arn = aws_secretsmanager_secret.db_password.arn
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid       = "DenyExternalAccess"
        Effect    = "Deny"
        Principal = "*"
        Action    = "secretsmanager:GetSecretValue"
        Resource  = "*"
        Condition = {
          StringNotEquals = {
            "aws:SourceVpce" = "vpce-12345678"
          }
        }
      }
    ]
  })
}
```

## 4. Why Use Secrets Manager Over Parameter Store?

DevOps engineers often ask why they should pay $0.40/month per secret for Secrets Manager when Parameter Store (SecureString) is free.
1.  **Automated Rotation:** Secrets Manager can automatically change passwords in RDS/Redshift/DocumentDB without human intervention.
2.  **Secret Generation:** It can generate random, complex passwords that comply with AWS best practices automatically.
3.  **Cross-Account Sharing:** Much easier to share secrets between different AWS accounts compared to Parameter Store.

## 5. Summary

AWS Secrets Manager is the essential "Safe" for your cloud-native foundations. By using Terraform to manage the secret metadata, the versioned values, and the restrictive resource policies, you build a security posture that is both automated and highly resilient to unauthorized access. Terraform's ability to seamlessly "Push" generated secrets during an apply and "Pull" them during a data fetch ensures that your organizational foundations are built on a private, protected, and fully audited credential management system. Mastering these native security patterns is a mandatory requirement for any engineer managing professional AWS environments at scale.
