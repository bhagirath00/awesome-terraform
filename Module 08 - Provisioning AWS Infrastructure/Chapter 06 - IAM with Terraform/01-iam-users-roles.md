# Identity Orchestration: Provisioning AWS IAM Users and Roles

AWS Identity and Access Management (IAM) is the most critical service in the cloud. It is the security foundation that controls who (Authentication) can do what to which resources (Authorization). In a professional DevOps lifecycle, managing IAM through Terraform is mandatory to ensure that the principle of "Least Privilege" is enforced across every project. Modern security standards prioritize the use of Roles over Users to eliminate long-lived credentials. This chapter provides a definitive deep dive into the `aws_iam_user`, `aws_iam_role`, and `aws_iam_access_key` resources, and the technical strategies for implementing a zero-trust identity architecture.

## 1. IAM Users: The Legacy Identity

An IAM user is an identity with long-term credentials used to interact with AWS.

```hcl
resource "aws_iam_user" "developer" {
  name = "john.doe"
  path = "/engineering/"

  tags = {
    Department = "Product-Engineering"
  }
}
```

### 1.1 The Risk of Access Keys
When you create an IAM User, you often generate an Access Key and Secret Key for CLI or application access.

```hcl
resource "aws_iam_access_key" "dev_key" {
  user = aws_iam_user.developer.name
}
```

**Security Warning:** Access keys are "Permanent" credentials. They do not expire unless manually rotated. If an access key is committed to a public repository (e.g., GitHub), an attacker can compromise your entire account in seconds. **Professional Standard:** Avoid IAM Users for automated processes; use IAM Roles instead.

## 2. IAM Roles: The Modern Identity Standard

An IAM Role is an identity that you can "Assume" to obtain temporary security credentials. It does not have any long-term passwords or access keys.

```hcl
resource "aws_iam_role" "app_role" {
  name = "production-application-role"

  # The Trust Policy (Who can assume this role?)
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}
```

### 2.1 Technical Handshake: The Trust Policy
Every role requires a "Trust Relationship" (represented by the `assume_role_policy` attribute). This defines which principal (a service, another AWS account, or a SAML provider) is allowed to take on the role. In the example above, only the EC2 service is allowed to assume the role.

## 3. Comparing Users vs. Roles

| Feature | IAM User | IAM Role |
|---------|----------|----------|
| **Credentials** | Long-lived (Password / Access Keys) | Temporary (STS Token) |
| **Duration** | Permanent until deleted | Session-based (1h to 12h) |
| **Use Case** | Humans accessing the console | Applications, Services, Cross-account access |
| **Security** | High risk of leakage | Inherently secure; credentials rotate automatically |

## 4. Cross-Account Access with Roles

Roles are the primary mechanism for cross-account orchestration.
- **Scenario:** You have a "Security" account and want an automation tool in there to manage resources in a "Production" account.
- **Implementation:**
    1.  Create a Role in the Production account.
    2.  Set its Trust Policy to allow the Security account ID to `sts:AssumeRole`.
    3.  Grant the Role permissions to manage production resources.
    4.  The Security account tool then calls `AssumeRole` to work in Production.

## 5. IAM Virtual MFA and Passwords

For the remaining human users who must have an IAM User account:
- **`aws_iam_user_login_profile`**: Manages the user's password for console login.
- **Best Practice:** Always enforce Multi-Factor Authentication (MFA). Terraform cannot directly manage a physical MFA device, but you can use an IAM Policy to deny all actions until the user authenticates with MFA.

## 6. IAM Resource Pathing and Namespaces

In large organizations, you should use the `path` attribute to organize your identities.
- Path: `/engineering/`
- Path: `/ops/`
- Resulting ARN: `arn:aws:iam::123456789012:user/engineering/john.doe`
This allows you to write IAM policies that grant permissions to entire paths (folders) of users.

## 7. Lifecycle and Deletion Persistence

Deleting an IAM User in Terraform will fail if the user has active access keys, groups, or inline policies that are not managed by Terraform.
- **Clean Deletion:** Terraform's `aws_iam_user` resource handles the removal of policy attachments, but it is a best practice to keep all IAM-related resources (Keys, Policies, Group Memberships) within the same Terraform module to ensure clean resource destruction.

## 8. Troubleshooting IAM Provisioning

- **"Access Denied" during AssumeRole:** The Principal attempting to assume the role does not have the `sts:AssumeRole` permission in its own policy, OR the Role's Trust Policy does not list the Principal correctly.
- **"EntityAlreadyExists":** IAM names (Roles, Users, Groups) are global within the AWS account. If you attempt to create a role that already exists, Terraform will fail. You must either rename the role or import the existing one.

## 9. Summary

Identity is the perimeter of your cloud infrastructure. By prioritizing IAM Roles over Users and strictly enforcing Trust Relationships, you eliminate the risk of long-lived credential leakage. Terraform's declarative management of these identities ensures that your organizational hierarchy and access control patterns are versioned, peer-reviewed, and consistently applied, providing the necessary foundation for a secure, automated DevOps ecosystem.
