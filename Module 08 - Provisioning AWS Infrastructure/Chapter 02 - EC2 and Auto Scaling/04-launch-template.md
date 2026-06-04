# Blueprints for Success: Provisioning AWS Launch Templates

An AWS Launch Template is a powerful configuration blueprint that specifies how an EC2 instance should be provisioned. It consolidates all the parameters usually provided to a single `aws_instance` (like AMI ID, instance type, security groups, and user data) into a reusable entity. Launch Templates are the mandatory prerequisite for modern Auto Scaling Groups (ASG) and Amazon EKS worker nodes. In a professional DevOps environment, Launch Templates are preferred over the legacy "Launch Configurations" because they support versioning, advanced networking, and more granular resource control. This chapter provides a definitive exploration of the `aws_launch_template` resource and the technical patterns for versioned infrastructure deployment.

## 1. Why Move to Launch Templates?

For many years, AWS used "Launch Configurations." However, these were immutable; you could not edit an existing one, leading to an explosion of "MyConfig-v1", "MyConfig-v2" garbage in accounts. 

**Advantages of `aws_launch_template`:**
- **Versioning:** You can have multiple versions of the same template (e.g., v1 uses Amazon Linux 2, v2 uses Amazon Linux 2023).
- **Default/Latest:** You can tell your Auto Scaling Group to always use the `$Latest` or `$Default` version.
- **Support for Spot:** Better handling of mixed Spot and On-Demand fleets.
- **Granular Tagging:** Allows you to specify tags for the instance, the volumes, and the network interfaces independently.

## 2. Technical Anatomy of a Launch Template

The `aws_launch_template` resource is a deep object that covers almost every edge case of EC2 provisioning.

```hcl
resource "aws_launch_template" "app_v2" {
  name        = "production-app-template"
  description = "Template for high-performance application nodes"

  image_id      = data.aws_ami.latest.id
  instance_type = "t3.large"

  # Network Interface Configuration
  network_interfaces {
    associate_public_ip_address = false
    security_groups             = [aws_security_group.app_sg.id]
    delete_on_termination       = true
  }

  # Identity
  iam_instance_profile {
    name = aws_iam_instance_profile.app_profile.name
  }

  key_name = aws_key_pair.main.key_name

  # Bootstrapping logic
  user_data = base64encode(file("scripts/setup.sh"))
}
```

## 3. Managing Storage: Block Device Mappings

Launch templates provide sophisticated control over how storage is attached to instances initiated from the template.

```hcl
block_device_mappings {
  device_name = "/dev/xvda"
  ebs {
    volume_size           = 100
    volume_type           = "gp3"
    iops                  = 3000
    throughput            = 125
    encrypted             = true
    delete_on_termination = true
  }
}
```
**Pro Tip:** By specifying the `ebs` block inside the template, you ensure that every instance in your auto-scaling fleet uses encrypted, high-performance GP3 storage by default, satisfying organizational compliance requirements.

## 4. The Power of Versioning in Terraform

One of the greatest developer experiences in Terraform is managing template versions.

```hcl
resource "aws_launch_template" "web" {
  name_prefix   = "web-"
  # ... other params ...

  # Lifecycle rule to handle versioning
  lifecycle {
    create_before_destroy = true
  }
}
```

When you update an attribute (like the AMI ID), Terraform will update the `latest_version` of the template in AWS. If your Auto Scaling Group is configured to use the `latest` version, it will automatically start using the new AMI on its next scale-out event or during an Instance Refresh.

## 5. Metadata and Security Hardening (IMDSv2)

Security best practices mandate the use of Instance Metadata Service Version 2 (IMDSv2). You can enforce this globally at the template level.

```hcl
metadata_options {
  http_endpoint               = "enabled"
  http_tokens                 = "required" # Enforce v2
  http_put_response_hop_limit = 2
  instance_metadata_tags      = "enabled"
}
```
By setting `http_tokens = "required"`, any instance launched from this template will reject unauthenticated (v1) metadata requests, significantly reducing the risk of SSRF-based credential theft.

## 6. Tag Specification and Propagation

A common frustration in AWS is instances launching without tags. Use the `tag_specifications` block to solve this.

```hcl
tag_specifications {
  resource_type = "instance"
  tags = {
    Name        = "Fleet-Instance"
    Provisioner = "Terraform"
  }
}

tag_specifications {
  resource_type = "volume"
  tags = {
    Type = "App-Data"
  }
}
```
This ensures that the OS volumes, as well as the instance itself, are correctly tagged for billing and automation purposes from the moment of creation.

## 7. Performance and Advanced Features

### 7.1 Credit Specification
For burstable instances (T2/T3), you can ensure they never throttle by specifying the credit model.
```hcl
credit_specification {
  cpu_credits = "unlimited"
}
```

### 7.2 Capacity Reservations
If you have reserved instances or savings plans, you can link the launch template to a specific capacity reservation to ensure your instances are physically available during high-demand periods.

### 7.3 Spot Instance support
You can specify `instance_market_options` to request Spot instances, which can save up to 90% in costs for fault-tolerant workloads.

## 8. Troubleshooting Launch Template Failures

If an ASG fails to launch instances from a template:
1.  **AMI Validation:** Does the AMI ID exist in the region where the ASG is trying to launch?
2.  **User Data Encoding:** Ensure `user_data` is Base64 encoded. If you pass raw text, the launch will fail.
3.  **Subnet Mismatch:** Do the security groups in the template belong to the same VPC as the subnets in the ASG?
4.  **IAM Permissions:** Does the service linked role for Auto Scaling have permission to access the AMI and the KMS key used for volume encryption?

## 9. Summary

AWS Launch Templates are the sophisticated evolution of instance configuration. By consolidating networking, storage, identity, and security into a single versioned blueprint, you create an infrastructure specification that is both robust and flexible. Terraform's deep integration with launch templates allows for seamless version management and automated fleet updates, providing the necessary foundation for high-scale environments like Auto Scaling Groups and Kubernetes clusters.
