# Practical Implementation: Using the AWS VPC Module

The Virtual Private Cloud (VPC) is the fundamental networking layer for almost every AWS deployment. While you could build a VPC using individual resources (`aws_vpc`, `aws_subnet`, `aws_internet_gateway`, `aws_route_table`), the configuration overhead for a highly available, multi-AZ environment is significant. The community-standard `terraform-aws-modules/vpc/aws` module drastically simplifies this process. This chapter provides a deep dive into using this module to deploy production-grade networking.

## 1. The Basic Module Call

The VPC module allows you to define a complex network topology in a single block.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "main-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true

  tags = {
    Environment = "production"
    Team        = "infrastructure"
  }
}
```

## 2. Key Architecture Configuration

### 2.1 Multi-AZ Strategy
The `azs` variable defines which Availability Zones the module will use. The module automatically maps the `private_subnets` and `public_subnets` lists to these zones in order. 
- **Consistency Check:** The number of subnets in your lists must match the number of AZs you've specified to ensure an even distribution.

### 2.2 NAT Gateway Strategy
A NAT Gateway is required for resources in private subnets to reach the internet (e.g., for software updates). This module offers three common strategies:
- **`single_nat_gateway = true`**: All private subnets share one NAT Gateway. This is cost-effective but creates a single point of failure (if an AZ goes down, the NAT might be lost).
- **`one_nat_gateway_per_az = true`**: One NAT Gateway in every AZ. This is the production-standard for high availability.
- **`enable_nat_gateway = false`**: No NAT Gateway. Use this if your environment is isolated or uses VPC Endpoints exclusively.

## 3. Advanced Features and Gateways

### 3.1 VPC Endpoints (Interface and Gateway)
The module makes it easy to add VPC endpoints for services like S3, DynamoDB, or EC2. This keeps traffic within the AWS private network, increasing security and reducing NAT costs.

### 3.2 DNS and Hostnames
Always ensure `enable_dns_hostnames` and `enable_dns_support` are set to `true`. This allows EC2 instances to receive a public DNS hostname and perform name resolution within the VPC.

## 4. Consuming VPC Outputs

The real power of the VPC module is how you reference its generated data in subsequent resources.

```hcl
resource "aws_db_instance" "main" {
  # ...
  # Place the DB in the private subnets created by the module
  db_subnet_group_name = module.vpc.database_subnet_group
  vpc_security_group_ids = [aws_security_group.db_sg.id]
}

resource "aws_instance" "web" {
  # Place the EC2 in the first public subnet
  subnet_id = module.vpc.public_subnets[0]
}
```

**Commonly Used Outputs:**
- `vpc_id`: Required for Security Groups and Peering.
- `private_subnets`: A list of IDs for backend servers.
- `public_subnets`: A list of IDs for Load Balancers.
- `default_security_group_id`: The ID of the default SG created with the VPC.

## 5. Naming and Tagging Conventions

The module provides granular control over tags at every layer of the network.
- `public_subnet_tags`: Useful for Kubernetes to identify "Public" ElastiLoadBalancers (`kubernetes.io/role/elb = 1`).
- `private_subnet_tags`: Useful for Internal Kubernetes LoadBalancers (`kubernetes.io/role/internal-elb = 1`).
- `vpc_tags`: Overall metadata for the entire network stack.

## 6. Troubleshooting and Common Pitfalls

### 6.1 AZ Naming Errors
If you hardcode AZs like `"us-east-1a"`, your code will fail if you try to deploy to a region with different zone identifiers (like `eu-west-1a`).
- **Solution:** Use the `aws_availability_zones` data source to fetch AZs dynamically and pass them to the module.

### 6.2 CIDR Overlap
When connecting multiple VPCs via Peering or Transit Gateway, ensure your CIDR blocks do not overlap. The module does not check for external overlaps; this must be managed at the architectural level.

## 7. Migration from Custom VPC to Module

If you are currently managing VPC resources separately, "Importing" them into the module is complex because the module creates many child resources (Gateways, Route Tables, Subnets). It is often cleaner to deploy the new module-based VPC in parallel and migrate workloads rather than attempting a `terraform import` for such a large resource set.

## 8. Summary

The `terraform-aws-modules/vpc/aws` module is the industry standard for a reason: it encapsulates hundreds of lines of complex networking logic into a simple, high-level interface. By mastering its inputs—especially NAT Gateway strategies and CIDR planning—you can deploy world-class, highly available networking infrastructure in a fraction of the time it would take to build from scratch.
