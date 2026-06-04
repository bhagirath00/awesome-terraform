# Hardened Storage: Mastering AWS S3 Security

S3 is the most common target for cloud data leaks, often due to misconfigurations rather than platform vulnerabilities. Securing an S3 bucket requires a defense-in-depth approach that spans encryption, identity management, network restrictions, and proactive auditing. In a professional DevOps environment, S3 security is not an afterthought; it is enforced through systemic controls and "Deny-by-Default" policies. This chapter provides a definitive exploration of S3 security mechanisms, focusing on server-side encryption with KMS, Public Access Blocks, and the concept of a "Data Perimeter" managed through Terraform.

## 1. The Global Safety Switch: Public Access Block

The most important security feature for S3 is the "Public Access Block" (PAB). This provides a centralized control mechanism to prevent the accidental exposure of private data.

```hcl
resource "aws_s3_bucket_public_access_block" "hardened" {
  bucket = aws_s3_bucket.main.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### 1.1 Understanding the Four Toggles
1.  **`block_public_acls`**: Rejects any PUT request that carries a public ACL.
2.  **`ignore_public_acls`**: Causes S3 to ignore any existing public ACLs on the bucket or objects.
3.  **`block_public_policy`**: Rejects calls to `PutBucketPolicy` if the policy grants public access.
4.  **`restrict_public_buckets`**: Limits access to buckets with public policies to ONLY AWS services and authorized users in the bucket owner's account.

**Pro Tip:** Always set all four to `true` unless you are explicitly building a public website.

## 2. Encryption at Rest: SSE-S3 vs. SSE-KMS

AWS S3 supports several server-side encryption (SSE) methods for protecting data at rest.

### 2.1 SSE-S3 (Server-Managed Keys)
The simplest method where AWS manages the root keys. It is free and requires no manual key management.

### 2.2 SSE-KMS (Key Management Service)
The enterprise standard. You use a customer-managed key (CMK) to encrypt the data.
- **Benefits:** Provides an audit trail in CloudTrail showing exactly who accessed the data and when. It allow for key rotation and cross-account access through KMS Key Policies.

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "kms_encryption" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.s3_key.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true # Reduces KMS costs by 99%
  }
}
```

**Technical Note:** Setting `bucket_key_enabled = true` is a critical cost-optimization feature. It creates a temporary bucket-level key, significantly reducing the number of API calls made to KMS for object-level operations.

## 3. Defining a Network "Data Perimeter"

A data perimeter ensures that even a user with valid credentials cannot access your S3 data unless they are coming from a trusted network—typically your VPC.

```hcl
data "aws_iam_policy_document" "vpc_only_access" {
  statement {
    effect = "Deny"
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    actions = ["s3:*"]
    resources = [
      aws_s3_bucket.main.arn,
      "${aws_s3_bucket.main.arn}/*"
    ]
    condition {
      test     = "StringNotEquals"
      variable = "aws:SourceVpce"
      values   = [aws_vpc_endpoint.s3.id]
    }
  }
}
```
This "Explicit Deny" pattern is the strongest security control in AWS. It essentially locks the bucket so that its contents are only visible to traffic originating from a specific VPC Endpoint.

## 4. Access Control Lists (ACLs): The Legacy Problem

Historically, S3 used ACLs to manage permissions. Today, they are considered a security risk because they are fragmented (applied to individual objects) and hard to audit.
- **Modern Best Practice:** Disable ACLs entirely using `aws_s3_bucket_ownership_controls` and setting the rule to `BucketOwnerEnforced`. This moves all security control to IAM and Bucket Policies.

## 5. Auditing and Monitoring

You cannot secure what you do not monitor. S3 provides three primary layers of auditing.

### 5.1 S3 Server Access Logs
Captures the request-level details (source IP, requester identity, timestamp) for every bucket operation. 
- **Terraform Integration:** Use the `aws_s3_bucket_logging` resource to point logs to a dedicated centralized "Logging Bucket."

### 5.2 AWS CloudTrail Data Events
While standard CloudTrail logs "Management Events" (like creating a bucket), S3 Data Events log "Object-level Events" (like `GetObject` or `PutObject`). 
- **Cost Warning:** Data Events are expensive for high-traffic buckets. Only enable them for sensitive data buckets.

### 5.3 Amazon Macie
An automated security service that uses machine learning to discover and protect sensitive data (like PII, credit card numbers, or secret keys) stored in your buckets.

## 6. S3 Object Lock and WORM

For compliance-heavy industries, protecting against data **modification** or **deletion** is mandatory.
- **Object Lock:** Implements a Write-Once-Read-Many (WORM) model. 
- **Use Case:** Preventing a ransomware attacker from encrypting or deleting your backups.

```hcl
resource "aws_s3_bucket" "backups" {
  bucket = "company-secure-backups"
  object_lock_enabled = true
}
```

## 7. Security Best Practice Checklist

1.  **Block Public Access:** Set all PAB settings to `true`.
2.  **Encryption:** Enforce SSE-KMS for all sensitive buckets.
3.  **No ACLs:** Use Bucket Owner Enforced to disable ACLs.
4.  **Least Privilege:** Use IAM roles for EC2 instances instead of long-lived access keys.
5.  **Secure Transport:** Use a bucket policy to deny all non-HTTPS traffic.
6.  **MFA Delete:** Require MFA for permanent object deletion (especially for backups).

## 8. Summary

S3 security is a layered defense model. By combining account-level Public Access Blocks, bucket-level KMS encryption, and network-level VPC restrictions, you create a storage environment that is resilient to both human error and malicious intent. Terraform's declarative management of these security resources ensures that your organization's "Security Guardrails" are consistently applied and automatically audited, providing the necessary peace of mind for managing petabyte-scale data in the public cloud.
