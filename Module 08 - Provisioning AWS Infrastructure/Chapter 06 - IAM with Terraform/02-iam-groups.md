# Collective Access Control: Provisioning AWS IAM Groups

An IAM Group is a collective identity that contains multiple IAM users. Groups allow you to specify permissions for multiple users, which can make it easier to manage the permissions for those users. In a professional DevOps environment, IAM Groups are the primary mechanism for implementing Role-Based Access Control (RBAC). Instead of attaching policies to individual users—which is difficult to audit and leads to "Permission Drift"—you attach policies to a Group and then simply add or remove users from that group as their job functions change. This chapter provides a definitive deep dive into the `aws_iam_group`, `aws_iam_group_policy_attachment`, and `aws_iam_user_group_membership` resources.

## 1. The Principle of Group-Based Authorization

The industry standard for managing human identities is to never attach a policy directly to an IAM user.

- **The Problem:** If you have 50 developers and you attach the `S3-Access` policy to each one individually, and your policy needs to change, you have 50 resources to update. This is a massive operational overhead and a security risk.
- **The Solution:** Create a `Developer-Group`, attach the `S3-Access` policy to the group once, and make all 50 developers members of that group.

## 2. Provisioning IAM Groups in Terraform

The implementation of Group-based access involves three distinct resources.

```hcl
# 1. The Group
resource "aws_iam_group" "admins" {
  name = "system-administrators"
  path = "/users/"
}

# 2. The Policy Attachment
resource "aws_iam_group_policy_attachment" "admin_attach" {
  group      = aws_iam_group.admins.name
  policy_arn = "arn:aws:iam::aws:policy/AdministratorAccess"
}

# 3. User Membership
resource "aws_iam_user_group_membership" "admin_members" {
  user = aws_iam_user.john_doe.name

  groups = [
    aws_iam_group.admins.name,
    aws_iam_group.on_call.name,
  ]
}
```

## 3. Best Practices for RBAC (Role-Based Access Control)

Professional Terraform modules for IAM should reflect the organizational hierarchy.

- **Admin Group:** Has `AdministratorAccess` or a highly permissive PowerUser policy.
- **Developer Group:** Has access to develop and debug (EC2, RDS, Lambda) but perhaps no access to billing or IAM management.
- **ReadOnly Group:** For auditors or new employees who need to see the infrastructure but cannot change anything.
- **Security Group:** For security engineers who need to manage IAM, CloudTrail, and GuardDuty.

## 4. Technical Constraints of IAM Groups

- **No Principal Status:** An IAM Group **cannot** be identified as a `Principal` in a resource-based policy (like an S3 Bucket Policy) or a Trust Policy for a Role. Only Users and Roles can be principals.
- **Hierarchy:** IAM Groups cannot be nested. You cannot make a "ReadOnly" group a member of a "Developer" group.
- **Quotas:** An AWS account can have up to 300 groups. Each user can be a member of up to 10 groups.

## 5. Groups vs. Roles for Humans

While Roles are the standard for software, Groups are still the standard for **Human Console Users**.
- If a human logs in with a username and password, they inherit the permissions of the groups they belong to.
- For enterprise environments, the move is toward **AWS Identity Center (SSO)**, which uses "Permission Sets" (essentially managed groups) to handle identity across multiple AWS accounts.

## 6. Summary

IAM Groups are the organizational glue of your identity architecture. By strictly adhering to the "No User-Level Policy" rule and managing human access through job-function-aligned groups, you create an authorization layer that is clean, auditable, and easy to maintain. Terraform's ability to orchestrate group creation, policy attachment, and user membership ensures that your organization's access control policies are transparent and consistently applied to every team member.
