# Granular Authorization: Provisioning AWS IAM Policies

IAM Policies are the JSON documents that define the permissions for an identity (User, Group, or Role) or a resource (S3 Bucket, KMS Key). Policies determine exactly what "Actions" can be performed on which "Resources" and under what "Conditions." In a professional DevOps environment, the management of IAM policies is an exercise in the "Principle of Least Privilege," ensuring that no entity has more access than is strictly necessary for its function. This chapter provides a definitive deep dive into the technical structure of IAM policies, the different types of policy attachments, and the advanced logic of conditions and boundaries managed through Terraform.

## 1. Technical Anatomy of an IAM Policy

A policy is composed of one or more "Statements." Each statement is a rule.

```hcl
data "aws_iam_policy_document" "s3_read_only" {
  statement {
    sid    = "AllowListedBucketAccess"
    effect = "Allow"
    actions = [
      "s3:ListBucket",
      "s3:GetBucketLocation"
    ]
    resources = [
      "arn:aws:s3:::company-data-bucket"
    ]
  }

  statement {
    sid    = "AllowObjectRetrieval"
    effect = "Allow"
    actions = [
      "s3:GetObject",
      "s3:GetObjectVersion"
    ]
    resources = [
      "arn:aws:s3:::company-data-bucket/*"
    ]
  }
}

resource "aws_iam_policy" "read_only" {
  name        = "S3-Read-Only-Policy"
  description = "Grants read access to the production data lake"
  policy      = data.aws_iam_policy_document.s3_read_only.json
}
```

## 2. Policy Types: Inline vs. Managed

### 2.1 Managed Policies (`aws_iam_policy`)
These are standalone resources with their own ARN.
- **Advantage:** Reusable across multiple users and roles. Updates to a managed policy are immediately inherited by everyone attached to it. Version control is handled by AWS (up to 5 versions).

### 2.2 Inline Policies
These are embedded directly within a user or role.
- **Disadvantage:** Not reusable. Deleted when the user/role is deleted. Harder to audit at scale. In professional environments, avoid inline policies.

## 3. The Power of Conditions

Conditions allow you to add "If-Then" logic to your security rules.

### 3.1 MFA Enforcement
You can deny access to sensitive actions (like deleting an RDS instance) if the user has not recently authenticated with MFA.
```hcl
condition {
  test     = "Bool"
  variable = "aws:MultiFactorAuthPresent"
  values   = ["true"]
}
```

### 3.2 Network Restriction (IP Whitelisting)
Grant access only if the request originates from a trusted corporate IP range.
```hcl
condition {
  test     = "IpAddress"
  variable = "aws:SourceIp"
  values   = ["192.168.1.0/24"]
}
```

## 4. Policy Evaluation Logic: The "Decision Tree"

To troubleshoot IAM effectively, you must understand the order in which AWS processes policies:
1.  **Default:** All requests are denied by default.
2.  **Explicit Deny:** If any policy anywhere contains a `Deny`, the request is immediately rejected.
3.  **Allow:** If the request is not denied, and there is an `Allow`, the request is permitted.
4.  **Final Deny:** If there is no `Allow`, the request is denied.

## 5. IAM Policy Boundaries

A Permissions Boundary is an advanced feature used to limit the maximum permissions that an identity-based policy can grant to an IAM entity.
- **Use Case:** You want to give a developer permission to create their own IAM Roles for Lambda, but you want to ensure they cannot create a Role with `AdministratorAccess`.
- **Implementation:** You attach a boundary to the developer's user. Even if the developer creates a "Root" role, that role can only do what the boundary allows.

## 6. Dynamic Policies with Variables

Terraform allows you to use IAM variables that are evaluated by AWS at runtime.
- `${aws:username}`: Resolves to the name of the user making the request.
- **Scenario:** Create a single policy that allows every user to manage their OWN S3 folder.

```hcl
resources = [
  "arn:aws:s3:::user-uploads/$${aws:username}/*"
]
```
*(Note the double dollar sign `$$` used in HCL to escape the variable so it is passed to AWS instead of being interpolated by Terraform).*

## 7. Attaching Policies: The "Glues" of IAM

In Terraform, you link a policy to an identity using specific attachment resources.
- `aws_iam_user_policy_attachment`
- `aws_iam_role_policy_attachment`
- `aws_iam_group_policy_attachment`

**Professional Tip:** Avoid the `aws_iam_policy_attachment` resource (without the specific word "user" or "role"). This resource is "Global" and will remove any manual attachments made in the console, often leading to unintended lockouts.

## 8. Summary

IAM Policies are the fundamental language of cloud security. By using managed policies over inline ones, leveraging the safety of `data.aws_iam_policy_document` to prevent JSON syntax errors, and implementing complex logic through condition keys, you create a granular and highly secure authorization layer. Terraform's ability to manage these policies as versioned code ensures that your organizational access rules are transparent, auditable, and easily reproducible across every AWS account in your environment.
