# The Handshake of Identity: IAM Trust Relationships

An IAM Trust Relationship (also known as a Trust Policy) is a specific type of resource-based policy attached to an IAM Role. While a standard IAM Policy defines what a role can DO, the Trust Policy defines who is allowed to BECOME that role. In a professional DevOps environment, mastering Trust Relationships is essential for secure service-to-service communication, cross-account management, and federated identity integration. This chapter provides a definitive deep dive into the technical structure of trust policies and the patterns for authorizing AWS services and external identities.

## 1. Technical Purpose of the Trust Policy

Normally, AWS identities (Users/Roles) are strictly isolated. A Role cannot be assumed by another entity unless that Role explicitly "Trusts" the entity.

- **Analogy:** If an IAM Policy is a "Key" that opens a door (the resource), a Trust Policy is the "ID Check" performed at the door to ensure the person holding the key is who they say they are.

## 2. Provisioning Trust Policies with Terraform

In Terraform, the trust relationship is defined via the `assume_role_policy` attribute of the `aws_iam_role` resource.

```hcl
resource "aws_iam_role" "standard_role" {
  name = "cross-account-audit-role"

  # The Trust Relationship Handshake
  assume_role_policy = data.aws_iam_policy_document.trust_policy.json
}

data "aws_iam_policy_document" "trust_policy" {
  statement {
    actions = ["sts:AssumeRole"]
    effect  = "Allow"

    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::123456789012:root"] # Trust another AWS Account
    }
  }
}
```

## 3. Common Trust Scenarios

### 3.1 Service-Linked Trust
Allowing an AWS service (like EC2 or Lambda) to act on your behalf.
- Principal Type: `Service`
- Principal Identifiers: `ec2.amazonaws.com`, `lambda.amazonaws.com`, `rds.amazonaws.com`.

### 3.2 Cross-Account Trust
Allowing a user or role in Account A to assume a role in Account B. This is the foundation of the "Security Hub" or "Shared Services" architecture.
- Principal Type: `AWS`
- Principal Identifiers: `arn:aws:iam::<Account-A-ID>:root`.

### 3.3 Federation Trust (SAML/OIDC)
Allowing users from an external identity provider (like Okta, Azure AD, or Google) to log into AWS.
- Principal Type: `Federated`
- Principal Identifiers: The ARN of the SAML provider or the OIDC issuer.

## 4. The "External ID" for Third-Party Security

When you grant a third-party (like a SaaS monitoring tool) access to your account via a role, you face a security risk known as the **"Confused Deputy" problem**.
- **The Solution:** Use an `ExternalId` condition in your trust policy. The third-party must provide this unique secret string when calling `AssumeRole`.

```hcl
condition {
  test     = "StringEquals"
  variable = "sts:ExternalId"
  values   = [var.third_party_external_id]
}
```

## 5. Troubleshooting Trust Relationship Failures

- **"is not authorized to perform: sts:AssumeRole":** This is the most common error. It means either the trust policy Principal is incorrect, or the user/role attempting to assume the role does not have the `sts:AssumeRole` permission in their own identity policy.
- **"MalformedPolicyDocument":** Usually caused by trying to include regular Service permissions (like `s3:Get*`) inside a Trust Policy. Trust policies ONLY support the `sts:AssumeRole` action.

## 6. Summary

Trust Relationships are the gatekeepers of your cloud identities. By explicitly defining who is allowed to assume your roles and implementing security hardening features like External IDs for third-party access, you create a robust perimeter around your AWS account. Terraform's declarative management of the `assume_role_policy` ensures that your identity handshakes are transparent, versioned, and consistently enforced across your entire multi-account organizations.
