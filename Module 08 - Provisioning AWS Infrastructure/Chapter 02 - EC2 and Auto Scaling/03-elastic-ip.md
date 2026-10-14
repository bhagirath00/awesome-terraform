# Static Networking: Provisioning AWS Elastic IPs (EIP)

An Elastic IP (EIP) is a reserved, static public IPv4 address designed for dynamic cloud computing. Unlike the ephemeral public IP addresses that AWS assigns to EC2 instances by default—which change whenever an instance is stopped and restarted—an EIP remains associated with your AWS account until you explicitly choose to release it. In a professional DevOps environment, EIPs are critical for resources that require a stable, predictable endpoint, such as NAT Gateways, Bastion hosts, and VPN endpoints. This chapter provides a deep dive into the `aws_eip` resource, the lifecycle management of static public IP addresses, and the cost implications of managing these scarce cloud resources.

## 1. Technical Mechanics of the `aws_eip` Resource

The `aws_eip` resource in Terraform creates an allocation of a public IPv4 address from Amazon's global pool and assigns it to your specific VPC.

```hcl
resource "aws_eip" "nat_static_ip" {
  domain = "vpc"
  
  tags = {
    Name        = "Primary-NAT-IP"
    Project     = "Infrastructure-Core"
  }

  # Critical dependency: The IGW must exist before an EIP can be used
  depends_on = [aws_internet_gateway.main]
}
```

### 1.1 The `domain` Argument
In modern AWS development, you must always set `domain = "vpc"`. This indicates that the IP address should be allocated for use within a Virtual Private Cloud. The alternative value, "standard" (for EC2-Classic), is deprecated for most modern AWS accounts.

## 2. Allocation vs. Association: Architectural Patterns

Terraform allows you to manage the relationship between an IP and a resource in two ways. Understanding the difference is vital for advanced automation.

### 2.1 Pattern A: Integrated Allocation
You can link the IP directly to an instance within the `aws_eip` block using the `instance` argument. This is simple but tightly couples the IP to a specific VM.

### 2.2 Pattern B: Decoupled Association (Recommended)
By using the standalone `aws_eip_association` resource, you separate the "Ownership" of the IP from its "Usage."

```hcl
resource "aws_eip_association" "eip_assoc" {
  instance_id   = aws_instance.web_server.id
  allocation_id = aws_eip.nat_static_ip.id
}
```

**Why Decouple?** 
Decoupling allows you to re-assign a static IP from a failing instance to a healthy recovery instance in a different Availability Zone without destroying and recreating the IP allocation itself. This minimizes downtime for external clients who have whitelisted that specific IP address.

## 3. The Scarcity Context and Pricing Models

IPv4 addresses are a finite global resource. AWS manages this scarcity through a specific pricing model designed to discourage waste.

- **The Active Usage Fee:** Historically, EIPs were free as long as they were associated with a running instance. However, as of February 1, 2024, AWS charges for all public IPv4 addresses (including EIPs), even when they are in use.
- **The Idle Penalty:** If you allocate an EIP but do not associate it with a running resource (or if the associated instance is stopped), AWS charges a significantly higher hourly "Idle" fee.
- **Terraform Implication:** Ensure that your `terraform destroy` command is part of your development teardown scripts to avoid accumulating "Hidden" costs for unreleased IP addresses.

## 4. Operational Dependency: The Internet Gateway (IGW)

A standard Elastic IP cannot function without an Internet Gateway.
- **The Race Condition:** If Terraform attempts to associate an EIP before the Internet Gateway is fully attached to the VPC, the AWS API will return a dependency error.
- **The Solution:** Always use the `depends_on = [aws_internet_gateway.<name>]` meta-argument in your EIP resource. This explicitly informs Terraform's graph engine to finalize the network's external connectivity before requesting the static IP.

## 5. Reverse DNS (PTR Records)

For specialized workloads like Mail Servers (SMTP), you often need to set up a PTR (Pointer) record for your EIP so that its reverse lookup matches your domain name.
- **Terraform Limit:** AWS does not expose the PTR record management directly through the standard EC2 API. Traditionally, you had to file a support ticket to set this. 
- **Modern Change:** AWS now allows you to manage reverse DNS for your own Elastic IP addresses in specific regions through the console or CLI, which can be scripted alongside your Terraform apply.

## 6. Limits, Quotas, and Scaling

By default, most AWS accounts are limited to **5 Elastic IPs per region**.
- **The Bottleneck:** If you are building a multi-region, multi-zone architecture with a NAT Gateway in every AZ across three regions, you will hit this limit immediately.
- **The Solution:** You must proactively request a "Service Quota Increase" through the AWS Console for the `Elastic IPs` limit. Terraform's `terraform plan` will not catch this limit—it will only fail during the `apply` stage.

## 7. Interaction with Network Interfaces (ENI)

While associating an EIP with an `instance_id` is common, it is technically more precise to associate it with a `network_interface_id`.
- This is the standard pattern when using an Elastic Load Balancer (ELB) that requires static IPs (Network Load Balancers).
- It is also how NAT Gateways are designed to function under the hood.

```hcl
resource "aws_eip" "nlb_ip" {
  domain = "vpc"
  network_interface = aws_network_interface.lb_interface.id
}
```

## 8. Summary

The Elastic IP is the stable heartbeat of an otherwise ephemeral cloud network. By mastering the distinction between allocation and association, managing the critical dependency on Internet Gateways, and being mindful of the data-transfer and idle-usage costs, you can build a cloud architecture that is reachable, reliable, and cost-effective. Terraform's declarative approach to managing these static networking components ensures that your external endpoints are always consistent, providing the predictable gateway needed for your users and automated service integrations.
