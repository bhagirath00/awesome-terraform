# terraform state show

## Introduction

`terraform state show` displays the full details of a single resource in Terraform state. While `terraform state list` gives you a list of all resource addresses, `terraform state show` lets you drill into any one of them and see every attribute — both the ones you set in your configuration and the ones AWS assigned after the resource was created.

This command is one of your most important debugging tools. Whenever you need to know the current state of a specific resource — its ID, its ARN, its IP address, its configuration — `terraform state show` gives you that information instantly without having to open the AWS Console.

---

## Basic Usage

```bash
terraform state show <resource_address>
```

Examples:

```bash
terraform state show aws_instance.web_server
terraform state show aws_s3_bucket.app_data
terraform state show aws_security_group.web
terraform state show module.vpc.aws_vpc.main
```

---

## Reading the Output

Let's look at a real example. Here is the output for an EC2 instance:

```bash
terraform state show aws_instance.web_server
```

```hcl
# aws_instance.web_server:
resource "aws_instance" "web_server" {
    ami                                  = "ami-0c55b159cbfafe1f0"
    arn                                  = "arn:aws:ec2:us-east-1:123456789012:instance/i-0a1b2c3d4e5f67890"
    associate_public_ip_address          = true
    availability_zone                    = "us-east-1a"
    cpu_core_count                       = 1
    cpu_threads_per_core                 = 1
    disable_api_stop                     = false
    disable_api_termination              = false
    ebs_optimized                        = false
    get_password_data                    = false
    hibernation                          = false
    id                                   = "i-0a1b2c3d4e5f67890"
    instance_initiated_shutdown_behavior = "stop"
    instance_state                       = "running"
    instance_type                        = "t2.micro"
    ipv6_address_count                   = 0
    ipv6_addresses                       = []
    key_name                             = "my-key-pair"
    monitoring                           = false
    placement_partition_number           = 0
    primary_network_interface_id         = "eni-0abc123def456"
    private_dns                          = "ip-10-0-1-45.ec2.internal"
    private_ip                           = "10.0.1.45"
    public_dns                           = "ec2-54-23-45-67.compute-1.amazonaws.com"
    public_ip                            = "54.23.45.67"
    secondary_private_ips                = []
    security_groups                      = []
    source_dest_check                    = true
    subnet_id                            = "subnet-0abc123def"
    tags                                 = {
        "Environment" = "production"
        "ManagedBy"   = "Terraform"
        "Name"        = "web-server"
    }
    tags_all                             = {
        "Environment" = "production"
        "ManagedBy"   = "Terraform"
        "Name"        = "web-server"
    }
    tenancy                              = "default"
    user_data_replace_on_change          = false
    vpc_security_group_ids               = [
        "sg-0abc123def456",
    ]

    capacity_reservation_specification {
        capacity_reservation_preference = "open"
    }

    credit_specification {
        cpu_credits = "standard"
    }

    enclave_options {
        enabled = false
    }

    maintenance_options {
        auto_recovery = "default"
    }

    metadata_options {
        http_endpoint               = "enabled"
        http_put_response_hop_limit = 1
        http_tokens                 = "optional"
        instance_metadata_tags      = "disabled"
    }

    private_dns_name_options {
        enable_resource_name_dns_a_record    = false
        enable_resource_name_dns_aaaa_record = false
        hostname_type                        = "ip-name"
    }

    root_block_device {
        delete_on_termination = true
        device_name           = "/dev/xvda"
        encrypted             = false
        iops                  = 100
        tags                  = {}
        throughput            = 0
        volume_id             = "vol-0abc123def"
        volume_size           = 8
        volume_type           = "gp2"
    }
}
```

The output is formatted as valid HCL — it looks exactly like a resource block in your `.tf` files, making it very readable.

---

## What the Output Tells You

### Attributes You Set

These are the values you defined in your `.tf` configuration:

```hcl
ami           = "ami-0c55b159cbfafe1f0"
instance_type = "t2.micro"
key_name      = "my-key-pair"
tags          = { "Name" = "web-server", ... }
```

### Attributes AWS Assigned

These are the values AWS assigned after creating the resource — not in your configuration:

```hcl
id           = "i-0a1b2c3d4e5f67890"   # assigned by AWS
arn          = "arn:aws:ec2:..."         # assigned by AWS
public_ip    = "54.23.45.67"            # assigned by AWS
private_ip   = "10.0.1.45"             # assigned by AWS
private_dns  = "ip-10-0-1-45.ec2.internal"  # assigned by AWS
public_dns   = "ec2-54-23-45-67.compute-1.amazonaws.com"
availability_zone = "us-east-1a"        # assigned by AWS
```

These are the values that appeared as `(known after apply)` in your original plan.

### Nested Blocks

Nested configuration blocks appear indented:

```hcl
root_block_device {
    delete_on_termination = true
    device_name           = "/dev/xvda"
    volume_id             = "vol-0abc123def"
    volume_size           = 8
    volume_type           = "gp2"
}
```

---

## Practical Use Cases

### Use Case 1: Get a Resource's ID or ARN

Before you can use AWS CLI commands or reference a resource from another tool, you often need its ID or ARN:

```bash
# Get the EC2 instance ID
terraform state show aws_instance.web_server | grep "^    id"
# id = "i-0a1b2c3d4e5f67890"

# Get the RDS endpoint
terraform state show aws_db_instance.app | grep "endpoint"
# endpoint = "myapp.abc123.us-east-1.rds.amazonaws.com:5432"

# Get the load balancer DNS name
terraform state show aws_lb.main | grep "dns_name"
# dns_name = "myapp-alb-1234567890.us-east-1.elb.amazonaws.com"
```

---

### Use Case 2: Find an IP Address

```bash
# Get public IP of a server
terraform state show aws_instance.bastion | grep "public_ip"
# public_ip = "54.23.45.67"

# SSH into it directly
ssh -i ~/.ssh/mykey.pem ec2-user@$(terraform state show aws_instance.bastion | grep "public_ip" | awk '{print $3}' | tr -d '"')
```

---

### Use Case 3: Verify Configuration After Apply

After a `terraform apply`, verify that the resource was configured correctly:

```bash
# Check if multi-AZ is enabled on RDS
terraform state show aws_db_instance.app | grep "multi_az"
# multi_az = true

# Check if deletion protection is on
terraform state show aws_db_instance.app | grep "deletion_protection"
# deletion_protection = true

# Check encryption is enabled
terraform state show aws_db_instance.app | grep "storage_encrypted"
# storage_encrypted = true
```

---

### Use Case 4: Investigate Unexpected Plan Changes

When `terraform plan` shows unexpected changes, use `terraform state show` to see the current state values and compare them to your configuration:

```bash
# Plan shows an unexpected change to the instance type
# Check what instance type is currently in state
terraform state show aws_instance.web | grep "instance_type"
# instance_type = "t3.small"

# Your config says t2.micro — someone changed it manually!
# Now you understand why the plan wants to change it back
```

---

### Use Case 5: Check Resource Dependencies

When troubleshooting dependency issues, `terraform state show` shows which subnet, VPC, and security groups an instance is using:

```bash
terraform state show aws_instance.web_server
# subnet_id              = "subnet-0abc123def"
# vpc_security_group_ids = ["sg-0abc123def456"]
```

This tells you exactly which networking resources it is connected to.

---

### Use Case 6: Viewing Module Resources

Resources inside modules need the full module path:

```bash
# First find the resource in the module
terraform state list | grep module.vpc

# Then show it
terraform state show 'module.vpc.aws_vpc.main'
# id         = "vpc-0abc123def"
# cidr_block = "10.0.0.0/16"
```

Always wrap module addresses in single quotes to prevent shell expansion of the brackets.

---

## Showing `count` and `for_each` Resources

For resources with `count` or `for_each`, specify the exact instance:

```bash
# Show the first web server instance
terraform state show 'aws_instance.web[0]'

# Show the second web server instance
terraform state show 'aws_instance.web[1]'

# Show a specific for_each subnet
terraform state show 'aws_subnet.public["us-east-1a"]'

# Show all instances one by one using a loop
for addr in $(terraform state list aws_instance.web); do
  echo "=== $addr ==="
  terraform state show "$addr" | grep "public_ip\|instance_type\|id "
done
```

---

## Getting State as JSON

For scripting and automation, get the state output as JSON:

```bash
terraform show -json | jq '.values.root_module.resources[] | select(.address == "aws_instance.web_server")'
```

This gives you the full resource details in JSON format, which you can parse with `jq` or any JSON processing tool.

---

## Summary

- `terraform state show <address>` displays all attributes of a specific resource in state
- The output is formatted as HCL — clean and readable
- It shows both the values you set in your config and the values AWS assigned after creation
- Use it to get resource IDs, ARNs, IP addresses, and endpoints without going to the AWS Console
- Use it to verify resource configuration after an apply
- Use it to investigate unexpected plan changes — compare state values to your config values
- For `count` and `for_each` resources, use the full address with index: `aws_instance.web[0]`
- For module resources, use the full path: `module.vpc.aws_vpc.main`
- Wrap addresses with special characters in single quotes to prevent shell expansion

In the next file, we cover `terraform state mv` — how to move and rename resources in state without destroying and recreating them.
