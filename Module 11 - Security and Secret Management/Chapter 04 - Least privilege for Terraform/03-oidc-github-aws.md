# Zero-Static-Secrets: OIDC for GitHub Actions and AWS

One of the most significant security advancements in the AWS ecosystem is the support for OpenID Connect (OIDC) identity federation. For years, the standard way to connect a CI/CD pipeline to AWS was to store a permanent `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as a secret in the build system. This created a major security vulnerability: if the build system was compromised, the keys could be stolen and used indefinitely. OIDC eliminates this risk by allowing GitHub Actions to negotiate short-lived, temporary security credentials for every project execution. This chapter provide a definitive deep dive into the technical handshake between GitHub and AWS, and the Terraform patterns for provisioning a secure, federated identity.

## 1. Technical Mechanics: The OIDC Handshake

OIDC is a "Trust" based architecture where AWS trusts the "Claims" made by GitHub.

1.  **Identity Provider:** You register GitHub as a trusted Identity Provider (IdP) in your AWS account.
2.  **Request:** When a GitHub Actions workflow starts, it requests a JWT (JSON Web Token) from GitHub's OIDC service.
3.  **Authentication:** The workflow sends this token to AWS.
4.  **Verification:** AWS verifies the token's signature against GitHub's public keys.
5.  **Authorization:** If the token's claims (e.g., "I am a workflow in repo 'my-org/infra-repo'") match the Trust Policy of an IAM Role, AWS returns temporary security credentials (session token).

## 2. Provisioning the OIDC Provider with Terraform

The first step is creating the trust anchor in your AWS account.

```hcl
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  
  # The thumbprint is the SHA1 hash of the CA certificate of the OIDC provider
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}
```

## 3. Creating the Federated IAM Role

The most critical part of OIDC security is the **Condition** block in the Trust Relationship. You must ensure that only your specific repositories can assume the role.

```hcl
resource "aws_iam_role" "github_actions" {
  name = "github-actions-oidc-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRoleWithWebIdentity"
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github.arn
        }
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
          }
          StringLike = {
            # THE SECURITY GUARDRAIL: Lock to your specific repo
            "token.actions.githubusercontent.com:sub": "repo:my-company/infra-repo:*"
          }
        }
      }
    ]
  })
}
```

## 4. Usage in GitHub Actions Workflow

With the OIDC role in place, your YAML workflow no longer needs any permanent AWS keys.

```yaml
permissions:
  id-token: write # Mandatory for OIDC
  contents: read

steps:
  - name: Configure AWS Credentials
    uses: aws-actions/configure-aws-credentials@v2
    with:
      role-to-assume: arn:aws:iam::1234567:role/github-actions-oidc-role
      aws-region: us-east-1
```

## 5. Security Benefits of the OIDC Pattern

1.  **Zero Static Secrets:** There are no AWS keys saved in GitHub. There is nothing for an attacker to "Steal" and use later.
2.  **Automatic Rotation:** Every workflow run gets a brand new set of keys that expire as soon as the job is finished.
3.  **Granular Context:** AWS knows specifically which repository, which branch, and even which specific environment triggered the request, allowing for extremely precise security auditing.

## 6. Summary

The transition to OIDC authentication is the "Final Level" of CI/CD security maturity. By replacing permanent access keys with dynamic, federated identity tokens, you eliminate one of the most dangerous and common attack vectors in cloud computing. Terraform's ability to orchestrate the OIDC provider, the federated roles, and the restrictive trust conditions ensures that your identity foundations are built on a "Zero-Trust" model that is both highly secure and fully automated. Mastering the OIDC handshake is a mandatory requirement for any engineer managing professional, enterprise-scale cloud foundations in a modern DevOps ecosystem.
