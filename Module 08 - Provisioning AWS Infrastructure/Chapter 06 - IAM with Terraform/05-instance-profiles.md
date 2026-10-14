# Machine Identity: Provisioning IAM Instance Profiles for EC2

An IAM Instance Profile is a container for an IAM Role that you can use to pass role information to an EC2 instance when the instance starts. While IAM Roles define permissions for any entity (Human or Machine), an Instance Profile is a specific AWS abstraction required ONLY for EC2 instances to assume those roles. In a professional DevOps environment, Instance Profiles are the mandatory standard for granting servers access to other AWS services like S3, RDS, or CloudWatch, eliminating the need for hardcoded Access Keys. This chapter provides a definitive deep dive into the `aws_iam_instance_profile` resource and the technical workflow for implementing secure, machine-based authentication.

## 1. The Technical Workflow: Theory of Operation

When an EC2 instance is launched with an instance profile, it can securely request temporary security credentials from the Instance Metadata Service (IMDS).

1.  **Identity:** You create an IAM Role with a Trust Policy allowing `ec2.amazonaws.com`.
2.  **Permissions:** You attach IAM Policies to that Role.
3.  **The Bridge:** You create an IAM Instance Profile and place the Role inside it.
4.  **The Assignment:** You assign the Instance Profile to the EC2 instance in its launch specification.

## 2. Provisioning with Terraform

The implementation involves three interconnected resources.

```hcl
# 1. The Role
resource "aws_iam_role" "compute_role" {
  name = "web-server-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_trust.json
}

# 2. The Instance Profile (The Bridge)
resource "aws_iam_instance_profile" "compute_profile" {
  name = "web-server-instance-profile"
  role = aws_iam_role.compute_role.name
}

# 3. The EC2 Instance (The Assignment)
resource "aws_instance" "web" {
  ami                  = data.aws_ami.amazon_linux.id
  instance_type        = "t3.micro"
  iam_instance_profile = aws_iam_instance_profile.compute_profile.name
}
```

## 3. The "Container" Distinction: Role vs. Profile

A common point of confusion for developers is that in the AWS CLI or Console, you often see "Attach Role to Instance." However, under the hood, the API expects an **Instance Profile**.
- **Roles:** Are identities that can be assumed by anyone (Lambda, ECS, Humans).
- **Profiles:** Are specifically for EC2. A profile can typically only contain **one** IAM role.

## 4. Security Advantages of Instance Profiles

Using Instance Profiles is the baseline for AWS security compliance (e.g., SOC2, PCI).
- **Temporary Credentials:** AWS automatically rotates the Access Keys provided to the instance every few hours. This minimizes the "Blast Radius" if an instance is compromised.
- **Zero Secrets:** Your application code (using the AWS SDK) automatically finds and uses these temporary credentials. There are no configuration files to leak and no secrets to rotate manually.

## 5. Hardening with IMDSv2

The credentials provided by the instance profile are served via the Instance Metadata Service at `http://169.254.169.254`.
- **The Risk:** Version 1 of IMDS is vulnerable to SSRF (Server-Side Request Forgery). An attacker who can send a simple GET request to that IP can steal the role's temporary keys.
- **The Fix:** In your Terraform `aws_instance` or `aws_launch_template`, always enforce **IMDSv2**, which requires a session token and prevents most metadata exfiltration techniques.

## 6. Common Use Case: The CloudWatch Logging Agent

To allow your web servers to push logs to CloudWatch Logs, you must:
1.  Attach the `CloudWatchAgentServerPolicy` to your IAM Role.
2.  Assign the Role to the Instance Profile.
3.  Install the agent via User Data.
The agent will then "Just Work" without needing any AWS credentials provided in its config file.

## 7. Troubleshooting Instance Profile Issues

- **"Access Denied" from SDK:** Even if the instance profile is attached, ensure the **Trust Policy** on the Role allows `ec2.amazonaws.com`. If the trust policy is missing, the instance cannot "Assume" the identity.
- **"No Metadata Found":** If you have disabled the metadata service or have blocked the local IP `169.254.169.254` via the OS firewall (iptables), the SDK will be unable to retrieve its credentials.
- **Policy Propagation:** It can take up to 60 seconds for a new IAM policy attachment to propagate to the instance metadata service.

## 8. Summary

IAM Instance Profiles are the essential bridge for machine-to-machine security in AWS. By consolidating roles into profiles and assigning them to EC2 instances, you achieve a zero-secret security posture where credentials are provided dynamically and rotated automatically. Terraform's ability to orchestrate the entire chain—from the raw policy document to the running instance—ensures that your server fleet's identity and access control are always versioned, audited, and strictly enforced according to your organizational security guardrails.
