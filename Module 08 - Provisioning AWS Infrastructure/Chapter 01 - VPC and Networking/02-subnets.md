# Modern Network Segmentation: Provisioning AWS Subnets

Subnets are the logical subdivisions of a Virtual Private Cloud (VPC). They are the "rooms" within your VPC's infrastructure, designed to group resources based on their security profiles, accessibility requirements, and physical distribution across AWS Availability Zones. In professional Terraform engineering, subnets are not just created as static blocks; they are architected using dynamic functions to ensure scalability and high availability. This chapter provides a definitive deep dive into the `aws_subnet` resource, the calculation of CIDR ranges, and the strategic distribution of public and private tiers.

## 1. Architectural Principles of Subnet Provisioning

A subnet must reside entirely within a single Availability Zone (AZ). To ensure that your application can survive a data center failure, you must distribute your subnets across multiple AZs.

```hcl
resource "aws_subnet" "primary" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true

  tags = {
    Name        = "Public-Subnet-1a"
    Tier        = "Public"
    Environment = terraform.workspace
  }
}
```

## 2. The Distinction: Public vs. Private Subnets

It is a common technical misconception that a "Public" or "Private" subnet is a different type of AWS resource. In Reality, all subnets are created using the same `aws_subnet` resource. Their classification depends entirely on their **Route Table** and the `map_public_ip_on_launch` configuration.

### 2.1 Public Subnets
A subnet is "Public" if its route table has a destination for `0.0.0.0/0` (all internet traffic) pointing to an **Internet Gateway (IGW)**.
- **Onboarding Logic:** You typically set `map_public_ip_on_launch = true` for these subnets. This ensures that any EC2 instance launched here receives a public IP automatically.
- **Use Case:** Bastion hosts, Public-facing Load Balancers, and NAT Gateways.

### 2.2 Private Subnets
A subnet is "Private" if its route table does **not** have a direct route to an IGW. To reach the internet (for software updates or library downloads), instances in a private subnet must route their traffic to a **NAT Gateway** located in a public subnet.
- **Onboarding Logic:** `map_public_ip_on_launch` is always `false`.
- **Use Case:** Databases, Application microservices, and Internal caching layers.

## 3. Availability Zone Strategy (High Availability)

A single-AZ architecture is a single point of failure. Industry standards require spreading resources across at least two (and ideally three) AZs.

**The N+1 Deployment Pattern:**
- **Availability Zone A:** Public-A, Private-A.
- **Availability Zone B:** Public-B, Private-B.
- **Availability Zone C:** Public-C, Private-C.

In Terraform, you should use the `aws_availability_zones` data source to dynamically discover the zones available in your region, rather than hardcoding names like `us-east-1a`.

## 4. Subnet Sizing and CIDR Mathematics

Calculating subnet CIDR blocks manually is error-prone and hard to maintain as your VPC grows. Terraform provides the `cidrsubnet` function to automate this logic.

**Syntax:** `cidrsubnet(prefix, newbits, netnum)`
- **`prefix`**: The parent VPC CIDR (e.g., `10.0.0.0/16`).
- **`newbits`**: The number of bits to add to the mask (e.g., adding `8` bits to a `/16` creates a `/24`).
- **`netnum`**: The index of the subnet (e.g., `0` for the first subnet, `1` for the second).

```hcl
variable "public_subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = element(var.public_subnet_cidrs, count.index)
  availability_zone = element(data.aws_availability_zones.available.names, count.index)
}
```

## 5. The "5 Reserved IP" Limitation

Every AWS subnet has 5 IP addresses that are reserved and cannot be assigned to your resources. For a `/24` subnet (256 IPs), only 251 are usable.
1. **Network Address:** The first address in the CIDR block (e.g., `10.0.1.0`).
2. **VPC Router:** The second address (e.g., `10.0.1.1`).
3. **Amazon DNS:** The third address (e.g., `10.0.1.2`).
4. **Future Use:** The fourth address (e.g., `10.0.1.3`).
5. **Broadcast Address:** The last address in the block (e.g., `10.0.1.255`).

**Operational Impact:** When sizing small subnets (like a `/28` for a database), you only have 11 usable IPs. If you have a high-availability database cluster and several Lambda functions in that subnet, you could exhaust the address space unexpectedly.

## 6. IPv6 Networking in Subnets

As your VPC transitions to dual-stack networking, you must manage IPv6 prefixes.
- AWS provides a `/56` block for the VPC.
- Subnets are assigned a `/64` prefix from that block.

```hcl
resource "aws_subnet" "dual_stack" {
  vpc_id                          = aws_vpc.main.id
  cidr_block                      = "10.0.1.0/24"
  ipv6_cidr_block                 = cidrsubnet(aws_vpc.main.ipv6_cidr_block, 8, 1)
  assign_ipv6_address_on_creation = true
}
```

## 7. Tagging Standards for Automation

In modern DevOps, tags are more than just metadata; they are used for service discovery and automation.

### 7.1 Kubernetes Integration
If you are deploying Amazon EKS, your subnets **must** carry specific tags so that the Kubernetes AWS Cloud Provider can identify where to place load balancers.
- `kubernetes.io/cluster/<cluster-name> = shared`
- `kubernetes.io/role/elb = 1` (For public subnets)
- `kubernetes.io/role/internal-elb = 1` (For private subnets)

## 8. State and Lifecycle Management

A subnet cannot be deleted if it still has active Network Interfaces (ENIs).
- **The "Taint" Risk:** If you attempt to change the CIDR block of an existing subnet, Terraform will be forced to destroy the subnet and recreate it. This will fail if there are running instances or database nodes still inside that subnet.
- **Solution:** Plan your CIDR blocks carefully from day one. If you must change a CIDR, you generally have to create a new subnet, migrate the workloads, and then delete the old one.

## 9. Performance and Quotas

AWS imposes limits on the number of subnets per VPC (Standard limit: 200). While this is high, in massive multi-tenant architectures or projects with hundreds of micro-environments, you may need a quota increase. Furthermore, use the `aws_subnet` data source rather than the resource block when integrating with existing "Brownfield" networking provided by another team.

## 10. Summary

Designing and provisioning subnets is an exercise in both mathematical precision and architectural foresight. By classifying subnets into public and private tiers, distributing them across Availability Zones for high availability, and using Terraform's `cidrsubnet` and `count` functions to automate the deployment, you create a network that is both organized and resilient. Mastering the technical nuances of reserved IPs and automated tagging ensures that your network is not only functional but also ready for high-scale automation with services like EKS and Auto Scaling Groups.
