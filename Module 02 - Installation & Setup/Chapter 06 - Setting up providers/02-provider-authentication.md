# Provider Authentication

## Introduction

For Terraform to create and manage AWS resources, it must be able to authenticate with your AWS account. AWS needs to know who is making the request and whether that person or system has permission to do so.

This file covers all the ways you can authenticate Terraform with AWS, from the simplest method for local development to the most secure methods used in production environments.

---

## How AWS Authentication Works

AWS uses a credential system based on **Access Keys**. An Access Key consists of two parts:

- **Access Key ID** — like a username (e.g., `AKIAIOSFODNN7EXAMPLE`)
- **Secret Access Key** — like a password (e.g., `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`)

When Terraform makes an API call to AWS, it signs the request with these credentials. AWS verifies the signature and checks if the associated IAM user or role has permission to perform the requested action.

---

## The Authentication Priority Order

Terraform (through the AWS provider) looks for credentials in a specific order. It tries each method and uses the first one it finds:

1. **Static credentials in the provider block** (least secure — avoid)
2. **Environment variables** (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
3. **Shared credentials file** (`~/.aws/credentials`)
4. **AWS CLI configuration** (`~/.aws/config`)
5. **IAM Instance Profile** (when running on an EC2 instance)
6. **IAM Role for Service Accounts** (when running in EKS/Kubernetes)
7. **IAM Role via AWS STS** (assume_role in provider config)

Understanding this order helps you troubleshoot authentication issues — if credentials are set in multiple places, Terraform uses the first valid one it finds.

---

## Method 1: Static Credentials in Provider Block (Avoid This)

You can put credentials directly in your `provider` block:

```hcl
# DO NOT DO THIS IN REAL PROJECTS
provider "aws" {
  region     = "us-east-1"
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}
```

**Why you should never do this:**
- If you commit this file to Git, your credentials are exposed to everyone who can see the repository
- If the repository is accidentally made public, your credentials are on the internet
- AWS actively scans GitHub and other platforms for exposed credentials and will notify you (and may suspend your account)
- There is no easy way to rotate credentials without changing your code

This method is shown only so you know it exists and to tell you not to use it. Always use one of the methods below instead.

---

## Method 2: Environment Variables (Good for Local Dev and CI/CD)

Set your credentials as environment variables. The AWS provider automatically reads these variables:

```bash
export AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE"
export AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
export AWS_DEFAULT_REGION="us-east-1"
```

Your `provider` block becomes simple:

```hcl
provider "aws" {
  region = "us-east-1"
  # No credentials here — they come from environment variables
}
```

**Advantages:**
- Credentials are not in your code or Git history
- Easy to change without touching your Terraform code
- Works perfectly in CI/CD pipelines (GitHub Actions, Jenkins, etc.)
- Different credentials can be set per terminal session

**Setting environment variables permanently:**

On Linux/macOS, add to your `~/.bashrc` or `~/.zshrc`:
```bash
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_DEFAULT_REGION="us-east-1"
```

**For a single terminal session only:**
```bash
# These credentials only exist in this terminal window
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
terraform apply
```

---

## Method 3: AWS CLI Configuration and Named Profiles (Best for Local Development)

The AWS CLI stores credentials in `~/.aws/credentials` and configuration in `~/.aws/config`. Terraform reads these files automatically.

### Setting Up AWS CLI Credentials

First, make sure the AWS CLI is installed:
```bash
aws --version
```

Then configure your credentials:
```bash
aws configure
```

You will be prompted to enter:
```
AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
Default region name [None]: us-east-1
Default output format [None]: json
```

This creates two files:

**`~/.aws/credentials`:**
```ini
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
```

**`~/.aws/config`:**
```ini
[default]
region = us-east-1
output = json
```

The `[default]` profile is used automatically by Terraform when no profile is specified.

---

### Using Named Profiles (Multiple AWS Accounts)

If you work with multiple AWS accounts (which is very common in real DevOps work), use **named profiles**:

```bash
# Configure a profile for your personal account
aws configure --profile personal

# Configure a profile for your work production account
aws configure --profile work-prod

# Configure a profile for your work staging account
aws configure --profile work-staging
```

This creates entries in `~/.aws/credentials`:

```ini
[default]
aws_access_key_id = AKIA...DEFAULT
aws_secret_access_key = ...

[personal]
aws_access_key_id = AKIA...PERSONAL
aws_secret_access_key = ...

[work-prod]
aws_access_key_id = AKIA...PROD
aws_secret_access_key = ...

[work-staging]
aws_access_key_id = AKIA...STAGING
aws_secret_access_key = ...
```

To use a specific profile in Terraform:

```hcl
provider "aws" {
  region  = "us-east-1"
  profile = "work-prod"
}
```

Or using an environment variable (does not require editing your `.tf` file):
```bash
export AWS_PROFILE="work-prod"
terraform plan
```

---

## Method 4: IAM Instance Profile (Best for EC2-Based CI/CD)

When Terraform runs on an **EC2 instance** (for example, in a self-hosted Jenkins server on EC2), you can attach an **IAM Instance Profile** to that EC2 instance. This gives the instance permission to make AWS API calls without any credentials files.

**How it works:**
1. Create an IAM role with the permissions Terraform needs
2. Create an instance profile and attach the role to it
3. Attach the instance profile to your EC2 instance
4. Terraform automatically detects and uses the instance profile credentials

**Advantages:**
- No credentials files anywhere on the server
- No credentials in environment variables
- AWS automatically rotates the temporary credentials
- Access can be revoked instantly by detaching the role

**Example IAM role for Terraform (Terraform configuration):**

```hcl
resource "aws_iam_role" "terraform_runner" {
  name = "terraform-runner-role"

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

resource "aws_iam_instance_profile" "terraform_runner" {
  name = "terraform-runner-profile"
  role = aws_iam_role.terraform_runner.name
}
```

When running on an EC2 instance with this profile, Terraform needs no credentials configuration at all:

```hcl
provider "aws" {
  region = "us-east-1"
  # Credentials come automatically from instance profile
}
```

---

## Method 5: OIDC / IAM Role for CI/CD Pipelines (Most Secure for CI/CD)

This is the most secure and modern way to authenticate Terraform in CI/CD pipelines like **GitHub Actions**. Instead of storing long-lived access keys, you use **OpenID Connect (OIDC)** to allow GitHub Actions to assume an IAM role temporarily.

**How it works:**
1. Configure AWS to trust GitHub's OIDC provider
2. Create an IAM role that GitHub Actions can assume
3. In your pipeline, GitHub Actions gets a short-lived token and exchanges it for AWS credentials
4. Credentials expire automatically after the pipeline job finishes

**Advantages:**
- No long-lived credentials stored anywhere (no access keys)
- Credentials are automatically scoped to the specific repository and branch
- If credentials are compromised, they expire in minutes (not indefinitely)
- This is the AWS-recommended approach for GitHub Actions

**Setting up OIDC with GitHub Actions (Terraform configuration):**

```hcl
# Create the OIDC provider in AWS
resource "aws_iam_openid_connect_provider" "github" {
  url = "https://token.actions.githubusercontent.com"

  client_id_list = ["sts.amazonaws.com"]

  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]
}

# Create the IAM role that GitHub Actions will assume
resource "aws_iam_role" "github_actions_terraform" {
  name = "github-actions-terraform"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Federated = aws_iam_openid_connect_provider.github.arn
        }
        Action = "sts:AssumeRoleWithWebIdentity"
        Condition = {
          StringEquals = {
            "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
          }
          StringLike = {
            # Only allow this specific repository
            "token.actions.githubusercontent.com:sub" = "repo:myorg/myrepo:*"
          }
        }
      }
    ]
  })
}
```

**GitHub Actions workflow using OIDC:**

```yaml
name: Terraform Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write   # Required for OIDC
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init

      - name: Terraform Plan
        run: terraform plan

      - name: Terraform Apply
        run: terraform apply -auto-approve
```

We cover CI/CD integration in detail in Module 10.

---

## Creating IAM Credentials for Terraform

To use Methods 2 or 3, you need an IAM user with access keys. Here is how to create them properly.

### Step 1: Create a dedicated IAM user for Terraform

Log into the AWS Console → IAM → Users → Create user

- User name: `terraform-user` (or your preference)
- Do NOT check "Enable console access" — this is a programmatic-only user

### Step 2: Attach permissions

Attach the permissions Terraform needs. For learning purposes, you can use `AdministratorAccess`. For production, use least-privilege permissions specific to what your Terraform code creates.

### Step 3: Create access keys

- Go to the user → Security credentials tab
- Click "Create access key"
- Select "Command Line Interface (CLI)"
- Download or copy the Access Key ID and Secret Access Key

**Important:** The Secret Access Key is only shown once. Copy it immediately.

### Step 4: Configure the AWS CLI

```bash
aws configure --profile terraform-user
# Enter your Access Key ID and Secret Access Key
```

### Step 5: Verify the credentials work

```bash
aws sts get-caller-identity --profile terraform-user
```

You should see your account ID, user ID, and ARN.

---

## Verifying Authentication is Working

After setting up authentication, test that Terraform can access your AWS account:

```bash
# Set your profile if using named profiles
export AWS_PROFILE="terraform-user"

# Create a simple test configuration
mkdir /tmp/auth-test && cd /tmp/auth-test

cat > main.tf << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "user_id" {
  value = data.aws_caller_identity.current.user_id
}
EOF

terraform init
terraform apply -auto-approve
```

If you see your AWS account ID and user ID in the output, authentication is working correctly.

Clean up:
```bash
cd ~ && rm -rf /tmp/auth-test
```

---

## Authentication Method Comparison

| Method | Security | Ease of Setup | Best Used In |
|---|---|---|---|
| Hardcoded in provider | Very Poor | Easy | Never |
| Environment variables | Good | Easy | Local dev, CI/CD with secrets |
| AWS CLI profile | Good | Easy | Local development |
| IAM Instance Profile | Excellent | Medium | EC2-based CI/CD (Jenkins) |
| OIDC (GitHub Actions) | Excellent | Medium | GitHub Actions, modern CI/CD |

---

## Summary

- AWS authentication works through Access Key ID and Secret Access Key credentials
- Terraform checks for credentials in a specific order — environment variables, then AWS CLI config, then instance profile
- **Never hardcode credentials** in your `.tf` files
- For local development, use **AWS CLI named profiles** — easy, secure, and supports multiple accounts
- For CI/CD on EC2, use **IAM Instance Profiles** — no credentials to manage at all
- For GitHub Actions, use **OIDC** — the most secure option with no long-lived credentials
- Always create a **dedicated IAM user or role** for Terraform with appropriate permissions
- Verify your authentication is working before writing extensive Terraform configurations

In the next file, we cover provider versioning — how to control exactly which version of the AWS provider Terraform uses.
