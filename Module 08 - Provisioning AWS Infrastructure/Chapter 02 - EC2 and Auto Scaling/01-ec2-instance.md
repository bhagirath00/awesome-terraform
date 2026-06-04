# Core Compute Infrastructure: Provisioning AWS EC2 Instances

Amazon Elastic Compute Cloud (EC2) is the fundamental scalable compute engine of the AWS platform. While the industry is shifting toward containerization and serverless functions, the EC2 instance remains the versatile workhorse for persistent workloads, specialized legacy applications, and high-performance clusters. In an enterprise DevOps lifecycle, an EC2 instance is not a static server manually launched from a console; it is a version-controlled, immutable compute specification. This chapter provides an exhaustive exploration of the `aws_instance` resource, covering identity management, storage architecture, performance tuning, and the security mechanics of the modern cloud instance.

## 1. The Architectural Blueprint: The `aws_instance` Resource

The `aws_instance` resource is the primary abstraction for a virtual machine in Terraform. It provides hundreds of arguments that allow you to control every aspect of the VM's lifecycle.

```hcl
resource "aws_instance" "production_app" {
  ami           = data.aws_ami.latest_amazon_linux_2023.id
  instance_type = "t3.medium"
  
  # Network Topology
  subnet_id              = aws_subnet.private_subnet_a.id
  vpc_security_group_ids = [aws_security_group.app_security_group.id]
  
  # Identity and Access
  iam_instance_profile = aws_iam_instance_profile.app_instance_profile.name
  key_name             = aws_key_pair.operator_key.key_name

  # Provisioning Logic
  user_data            = file("${path.module}/scripts/bootstrap.sh")
  user_data_replace_on_change = true

  tags = {
    Name        = "Prod-App-Server"
    Tier        = "Application"
    CostCenter  = "Digital-Ops"
  }
}
```

## 2. AMI Management and Dynamic Sourcing

An Amazon Machine Image (AMI) contains the operating system and any pre-installed software.
- **The Golden Image Pattern:** Professional teams use Packer to build "Golden Images" (hardened, pre-configured images) and tag them with version numbers.
- **Dynamic Lookups:** Avoid hardcoding AMI IDs (e.g., `ami-0c55b159cbfafe1f0`). Use the `aws_ami` data source to find the latest image based on tags or owners. This ensures that your instances are always launched with the latest security patches.

```hcl
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}
```

## 3. Instance Families and Performance Sizing

AWS provides a vast library of instance types categorized into families. Selecting the right one is a balance of cost and performance.

- **General Purpose (T/M):** Best for balanced workloads and bursty traffic.
- **Compute Optimized (C):** High-performance processors for batch processing or scientific modeling.
- **Memory Optimized (R/X):** Fast performance for workloads which process large data sets in memory (databases).
- **Storage Optimized (I/D/H):** For high-speed local NVMe storage required by NoSQL databases.

### 3.1 The Burstable Performance Problem (T-Series)
T2 and T3 instances use a "Credit" system. If you exhaust your credits, the CPU will be throttled to a baseline level, often causing application outages. In Production, always enable `unlimited` mode via Terraform if using the T-series to prevent throttling during traffic spikes.

## 4. Storage Architecture: EBS Block Devices

Every EC2 instance requires at least one Elastic Block Store (EBS) volume for its operating system.

### 4.1 The Root Block Device
You can customize the root volume's size, type, and encryption directly in the `aws_instance` block.
```hcl
root_block_device {
  volume_size           = 50
  volume_type           = "gp3"
  iops                  = 3000
  throughput            = 125
  encrypted             = true
  delete_on_termination = true
}
```
**Engineering Tip:** Always use `gp3` instead of `gp2`. GP3 allows you to scale IOPS and throughput independently of the volume size, providing better performance for a lower cost.

### 4.2 EBS Volume Persistence
Volumes defined inside the `aws_instance` block are tied to the instance's lifecycle. For data that must survive the instance's deletion (like database files), use the standalone `aws_ebs_volume` resource and link it using `aws_volume_attachment`.

## 5. Bootstrapping with User Data and Cloud-Init

`user_data` allows you to pass a script that AWS runs during the first boot of the instance. This is the primary way to achieve "Self-Service Infrastructure."

- **Boilerplate Strategy:** Use `user_data` to install agents (CloudWatch, SSM), join an AD domain, or pull application binaries from S3.
- **Replayability:** In modern Terraform (1.3+), setting `user_data_replace_on_change = true` ensures that if you update the script, Terraform will automatically destroy and recreate the instance to apply the new logic.

## 6. Secure Identity with IAM Instance Profiles

Never hardcode AWS credentials (AK/SK) inside your application code or on an EC2 instance. 
- **The Profile Pattern:** Associate an IAM Role with your EC2 instance using an **IAM Instance Profile**.
- **Mechanics:** The instance's software can then securely request temporary credentials from the Instance Metadata Service (IMDS) to interact with other AWS services. This ensures that even if an instance is compromised, the attacker only has the scoped permissions of that specific role, and the credentials rotate automatically every few hours.

## 7. Advanced Networking: ENIs and IPs

### 7.1 Primary and Secondary interfaces
Each instance has a primary Elastic Network Interface (ENI). You can attach secondary ENIs to enable multi-homed network configurations (e.g., using one interface for management and another for application traffic).

### 7.2 Elastic IP Association
While you can request a public IP within the `aws_instance` resource, it is better to manage static IPs as a separate `aws_eip` resource to ensure they can be re-pointed to different instances during a disaster recovery event.

## 8. Hardening the Metadata Service (IMDSv2)

The Instance Metadata Service (IMDS) is a frequent target for attackers using Server-Side Request Forgery (SSRF). Version 1 of IMDS is vulnerable because it uses a simple GET request.
- **V2 Requirement:** Always enforce IMDSv2 in your Terraform code. This requires a session token and significantly hardens the instance against metadata exfiltration.

```hcl
metadata_options {
  http_endpoint = "enabled"
  http_tokens   = "required" # Enforce IMDSv2
  http_put_response_hop_limit = 1
}
```

## 9. Monitoring and Termination Protection

In a Production environment, you should protect critical servers from accidental deletion.
- **`disable_api_termination = true`**: Prevents the instance from being deleted via the AWS Console or API. To delete it, you must first disable this setting via Terraform.
- **Detailed Monitoring:** Setting `monitoring = true` enables 1-minute CloudWatch metrics (instead of the default 5-minute metrics), which is essential for real-time alerting.

## 10. Summary

Provisioning an EC2 instance is more than just launching a VM. It is a complex orchestration of storage sizing, network security, identity assumed roles, and automated bootstrapping script management. By leveraging dynamic AMI lookups, enforcing encrypted GP3 storage, and hardening the metadata service with IMDSv2, you create a compute environment that is resilient, secure, and ready for a fully automated DevOps pipeline. Terraform serves as the source of truth for this compute specification, ensuring that every server in your fleet is identical and easily reproducible across any region.
