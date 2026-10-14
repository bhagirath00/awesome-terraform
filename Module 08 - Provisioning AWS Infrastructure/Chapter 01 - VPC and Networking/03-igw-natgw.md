# Edge Security and Connectivity: Internet and NAT Gateways

In an AWS Virtual Private Cloud (VPC), the Internet Gateway (IGW) and NAT Gateway (NAT-GW) are the specialized exit points that allow or restrict communication with the external internet. While an IGW provides bidirectional connectivity for public-facing resources, a NAT Gateway provides secure, outbound-only connectivity for private resources. In a professional DevOps environment, the placement and scaling of these gateways are critical for ensuring both security and operational cost-efficiency. This chapter provides a definitive exploration of provisioning these edge components, managing Elastic IP dependencies, and implementing high-availability strategies.

## 1. The Internet Gateway (IGW): The Public Front Door

The Internet Gateway is a horizontally scaled, redundant, and highly available VPC component that allows communication between your VPC and the internet.

```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "Primary-IGW"
    ManagedBy   = "Terraform"
  }
}
```

### 1.1 Technical Mechanics
Unlike traditional hardware routers, an IGW is a logical software attachment. It performs two main functions:
1. **Routing Target:** It acts as a target in your VPC route tables for any internet-bound traffic (typically the `0.0.0.0/0` destination).
2. **Network Address Translation:** For any resource with a public IPv4 address, the IGW performs a 1-to-1 NAT translation from the instance's private IP to its assigned public IP.

## 2. The NAT Gateway: Secure Outbound Access

A NAT (Network Address Translation) Gateway allows resources in a private subnet to connect to the internet or other AWS services but prevents the internet from initiating a connection with those resources.

### 2.1 The Elastic IP (EIP) Requirement
A NAT Gateway is a managed service that requires a static public IP address to function. AWS uses this IP to identify the outbound traffic from your private network.

```hcl
resource "aws_eip" "nat" {
  domain     = "vpc"
  depends_on = [aws_internet_gateway.main] # Best practice to avoid race conditions
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_a.id # MUST be placed in a Public Subnet

  tags = {
    Name        = "Primary-NAT-Gateway"
  }
}
```

### 2.2 Placement Logic
A NAT Gateway **must** be placed in a public subnet—defined as a subnet that has a route to the Internet Gateway. If you place a NAT Gateway in a private subnet, it will be unreachable from the external world, and your private instances will remain isolated.

## 3. High Availability (HA) Architectures

A single NAT Gateway is a single point of failure. If the Availability Zone where the NAT Gateway resides goes down, all private subnets that route through it will lose internet connectivity, even if they are in different AZs.

### 3.1 Strategy A: The Single NAT (Cost Optimized)
In Development or Staging environments, you typically create one NAT Gateway in one AZ and point all private route tables to it.
- **Monthly Cost:** ~$32 (Allocation) + Data charges.
- **Risk:** High. An AZ failure results in a complete network outage for all private instances.

### 3.2 Strategy B: NAT per AZ (Production Standard)
In Production, you should provision one NAT Gateway in every Availability Zone.
- **Architecture:** `Subnet-Private-A -> NAT-A`, `Subnet-Private-B -> NAT-B`.
- **Terraform Logic:** Use `count` or `for_each` based on your list of public subnets to automate this.

```hcl
resource "aws_nat_gateway" "multi_az" {
  count         = length(aws_subnet.public)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
}
```

## 4. Bandwidth and Scaling Limits

AWS NAT Gateways automatically scale their bandwidth from a baseline of 5 Gbps up to a maximum of 45 Gbps. 
- **Bursting:** If you require more than 45 Gbps of sustained throughput, you must distribute your workloads across multiple NAT Gateways (e.g., using more AZs or multiple VPCs).
- **Concurrency:** A NAT Gateway supports up to 55,000 concurrent connections to a single unique destination (IP/Port/Protocol). If your application maintains thousands of connections to a single external API, you may encounter "Port Exhaustion."

## 5. The "NAT Tax": Monitoring and Optimization

NAT Gateways carry a data processing charge (~$0.045 per GB). For data-intensive workloads (like moving terabytes of data between EC2 and S3), this "NAT Tax" can easily become the most expensive part of your AWS bill.

### 5.1 Optimization: VPC Endpoints
To avoid NAT charges, you should use **VPC Endpoints** for AWS services.
- **Gateway Endpoints:** For S3 and DynamoDB. These are free and provide a direct "Tunnel" from your VPC to the service, bypassing the NAT Gateway entirely.
- **Interface Endpoints (PrivateLink):** For most other AWS services (Kinesis, SNS, SQS). While they have a small hourly cost, they often eliminate the much higher NAT data processing fees.

## 6. IPv6 and the Egress-Only Internet Gateway

If your architecture is move toward IPv6, you don't need a NAT Gateway for outbound security. Instead, you use an **Egress-Only Internet Gateway**.
- **Function:** Allows outbound IPv6 traffic from your private instances but prevents inbound IPv6 connections.
- **Cost:** $0. It is a logical attachment like the standard IGW and does not require an Elastic IP.

## 7. Troubleshooting Connectivity Flows

If a private instance cannot reach the internet, verify the "Sensitivity Chain" in your Terraform code:
1. **The Instance:** Is the Security Group allowing outbound traffic on the required port?
2. **The Private Route Table:** Is there a route `0.0.0.0/0` -> `nat-gateway-id`?
3. **The NAT Gateway:** Is it in its "Available" state?
4. **The Public Route Table:** Does the subnet containing the NAT Gateway have a route `0.0.0.0/0` -> `igw-id`?

## 8. Summary

Internet and NAT Gateways are the gatekeepers of your network edge. By understanding the bidirectional nature of the IGW and the unidirectional, managed complexity of the NAT Gateway, you can build a VPC that is both highly accessible and strictly secure. Leveraging Terraform to automate the deployment of Elastic IPs and the conditional scaling of NAT Gateways across Availability Zones ensures that your connectivity is resilient to data center failures while remaining cost-optimized for non-production workloads.
