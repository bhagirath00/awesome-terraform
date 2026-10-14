# Rapid Deployment: Using the AWS EC2 Instance Module

For individual virtual machines or small sets of specialized servers, the `terraform-aws-modules/ec2-instance/aws` module is the professional choice for standardized deployment. While a single `aws_instance` resource is simple, managing several instances with consistent naming, disk encryption, and IAM profiles becomes complex. This chapter demonstrates how to use the EC2 module to build reliable compute infrastructure that integrates seamlessly with your networking layer.

## 1. The Standard Instance Call

The module simplifies the declaration of an instance by providing high-level arguments for complex tasks like EBS volume management and IAM Role attachment.

```hcl
module "app_server" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "5.5.0"

  name = "app-prod-01"

  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.medium"
  key_name               = "deployer-key"
  monitoring             = true
  vpc_security_group_ids = [module.vpc.default_security_group_id]
  subnet_id              = module.vpc.private_subnets[0]

  tags = {
    Terraform   = "true"
    Environment = "prod"
  }
}
```

## 2. Managing Fleets with `for_each`

A common requirement is to deploy a fleet of identical or slightly varied servers. Since Terraform supports meta-arguments on modules, you can use `for_each` to iterate over the EC2 module.

```hcl
module "ec2_fleet" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  for_each = toset(["web-01", "web-02", "api-01"])

  name = each.key

  instance_type = "t3.small"
  subnet_id     = module.vpc.public_subnets[0]
  # ...
}
```

By using `for_each` on the module, you create three distinct instances in the state file: `module.ec2_fleet["web-01"]`, `module.ec2_fleet["web-02"]`, and so on.

## 3. Storage and Root Block Device

Managing disks is one of the most error-prone parts of EC2 management. The module provides a clean syntax for defining the root volume and additional EBS volumes.

```hcl
module "database_worker" {
  source = "terraform-aws-modules/ec2-instance/aws"

  # ...
  root_block_device = [
    {
      encrypted   = true
      volume_type = "gp3"
      volume_size = 50
    }
  ]
}
```

- **Encryption for Compliance:** Setting `encrypted = true` ensures that your data-at-rest meets security requirements (e.g., SOC2, HIPAA).
- **GP3 Optimization:** Using `gp3` allows you to scale IOPS and Throughput independently of storage size, which is more cost-effective than older `gp2` volumes.

## 4. IAM Integration

Instead of managing `aws_iam_instance_profile` separately, the module can handle the association if you provide an existing profile name.

```hcl
module "vault_server" {
  source = "terraform-aws-modules/ec2-instance/aws"

  # ...
  iam_instance_profile = "VaultServerRole"
  associate_public_ip_address = true
}
```

## 5. Integrating with VPC and Security Group Modules

In a professional "Tiers" architecture, the EC2 module consumes outputs from your networking modules.

```hcl
module "web_instance" {
  source = "terraform-aws-modules/ec2-instance/aws"

  name = "web-node"
  
  # Connect to the first public subnet from the VPC module
  subnet_id = module.vpc.public_subnets[0]
  
  # Use the security group ID from a security group module or resource
  vpc_security_group_ids = [module.web_sg.security_group_id]
}
```

## 6. Metadata Options (IMDSv2)

For enhanced security, modern AWS deployments should enforce **Instance Metadata Service Version 2 (IMDSv2)**, which is session-oriented and more secure than v1.

```hcl
metadata_options = {
  http_endpoint               = "enabled"
  http_tokens                 = "required" # Enforces IMDSv2
  http_put_response_hop_limit = 1
  instance_metadata_tags      = "enabled"
}
```

## 7. Outputs for Downstream Configuration

The EC2 module provides a wealth of outputs that are critical for further automation (e.g., in `local-exec` provisioners or DNS record creation).

- `id`: The unique instance identifier.
- `public_ip`: For bastion hosts or public servers.
- `private_ip`: For internal service communication.
- `primary_network_interface_id`: For attaching secondary IPs or networking rules.

## 8. Summary

The `terraform-aws-modules/ec2-instance/aws` module is more than just a wrapper around a single resource. It is a robust template for compute infrastructure that enforces standard naming, promotes security best practices (like encrypted root disks and IMDSv2), and integrates smoothly with other community modules. By using this module, you reduce the surface area for bugs and ensure that your server fleet is consistently configured across all AWS accounts and regions.
