# Identity and Access: Provisioning S3 Bucket Policies

S3 Bucket Policies are resource-based IAM policies that grant or deny access to S3 buckets and the objects within them. While IAM User Policies (Identity-based) specify what a user can do, Bucket Policies specify who can access the bucket itself. In a professional DevOps environment, Bucket Policies are the primary mechanism for cross-account access, VPC-only restricted entry, and enforcing data perimeters. This chapter provides a definitive exploration of the `aws_s3_bucket_policy` resource, the technical structure of the policy JSON, and the advanced conditions for securing your storage assets.

## 1. IAM User Policies vs. Bucket Policies

Understanding when to use which policy is a fundamental skill for cloud security.
- **Identity-based Policies (IAM User/Role):** Best for managing permissions for users within your own account. It answers: "What can this user do?"
- **Resource-based Policies (Bucket Policy):** Best for managing access from external accounts, unauthenticated users (for the public internet), or enforcing environment-wide security rules. It answers: "Who can access this specific bucket?"

**The Security Intersection:** A user is granted access only if BOTH the Identity policy and the Resource policy (if present) permit it. If an explicit **Deny** exists in either, the user is blocked.

## 2. Technical Structure of a Bucket Policy

A bucket policy is a JSON document defined in Terraform using the `aws_s3_bucket_policy` resource.

```hcl
resource "aws_s3_bucket_policy" "secure_access" {
  bucket = aws_s3_bucket.data_lake.id
  policy = data.aws_iam_policy_document.allow_access_from_another_account.json
}

data "aws_iam_policy_document" "allow_access_from_another_account" {
  statement {
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::123456789012:root"]
    }

    actions = [
      "s3:GetObject",
      "s3:ListBucket",
    ]

    resources = [
      aws_s3_bucket.data_lake.arn,
      "${aws_s3_bucket.data_lake.arn}/*",
    ]
  }
}
```

### 2.1 The Principal
The `Principal` specifies who is granted (or denied) access. For cross-account access, you provide the ARN of the external account. For public access, you use `*`.

## 3. Advanced Security Conditions

Conditions are the "Logic" of your firewall. They allow you to grant access only when specific environment variables are met.

### 3.1 Enforcing Secure Transport (HTTPS)
You should never allow data to be transferred over unencrypted HTTP.
```hcl
statement {
  sid    = "AllowSSLRequestsOnly"
  effect = "Deny"
  principals {
    type        = "*"
    identifiers = ["*"]
  }
  actions = ["s3:*"]
  resources = [
    aws_s3_bucket.data_lake.arn,
    "${aws_s3_bucket.data_lake.arn}/*",
  ]
  condition {
    test     = "Bool"
    variable = "aws:SecureTransport"
    values   = ["false"]
  }
}
```

### 3.2 Restricting Access to a Specific VPC
To ensure your production data is never accessed from the public internet—even from an authorized user's laptop—you can restrict access to a specific **VPC Endpoint ID**.
```hcl
condition {
  test     = "StringNotEquals"
  variable = "aws:SourceVpc"
  values   = [aws_vpc_endpoint.s3.id]
}
```

## 4. Enforcing Server-Side Encryption

You can use a bucket policy to automatically reject any upload (PUT) request that does not include encryption headers. This enforces your compliance policy at the storage layer.

```hcl
statement {
  effect = "Deny"
  actions = ["s3:PutObject"]
  resources = ["${aws_s3_bucket.main.arn}/*"]
  condition {
    test     = "StringNotEquals"
    variable = "s3:x-amz-server-side-encryption"
    values   = ["AES256"] # Enforce SSE-S3
  }
}
```

## 5. Public Access Blocks: The Safety Switch

Before a Bucket Policy can grant public access, you must first disable the "S3 Public Access Block" settings. These act as a global "Off Switch" to prevent accidental data leaks.

```hcl
resource "aws_s3_bucket_public_access_block" "block" {
  bucket = aws_s3_bucket.data_lake.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```
**Best Practice:** In an enterprise environment, always set these four booleans to `true` unless the bucket is specifically intended to host a public website.

## 6. Managing Cross-Account Access

S3 is the primary way to share data between different AWS accounts.
1. **The Owner's Task:** Create a Bucket Policy in Account A that grants access to Account B's Principal.
2. **The Requester's Task:** Create an IAM Role in Account B that grants its users permission to access the bucket in Account A.
3. **The Result:** Users in Account B can "Reach over" into Account A. Using `object_ownership = "BucketOwnerPreferred"` is recommended here so that Account A can still manage the files uploaded by Account B.

## 7. Troubleshooting Policy Errors

- **"MalformedPolicy":** Usually a JSON syntax error or a bucket ARN that is incorrectly formatted. Use `data.aws_iam_policy_document` to prevent these errors.
- **"Access Denied" during Terraform Apply:** The IAM user running Terraform must have the `s3:PutBucketPolicy` permission.
- **Policies not applying:** Check if there is an Organization Service Control Policy (SCP) at the AWS account level that restricts S3 policies.

## 8. Summary

Bucket Policies are the powerful, JSON-based guardrails that define who and how your data is accessed. By mastering principals, actions, and advanced condition keys like `aws:SourceVpc` and `aws:SecureTransport`, you transform S3 from a simple storage box into an intelligent, securely guarded data vault. Terraform's integration with IAM policy documents ensures that your security posture is declarative, versioned, and easily audited across your entire cloud landscape.
