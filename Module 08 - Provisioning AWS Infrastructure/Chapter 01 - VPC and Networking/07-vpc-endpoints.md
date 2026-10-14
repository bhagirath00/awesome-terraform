# Private Connectivity: Provisioning AWS VPC Endpoints (PrivateLink)

VPC Endpoints allow you to connect your VPC privately to supported AWS services and VPC endpoint services powered by AWS PrivateLink. This bypasses the public internet, eliminating the need for an Internet Gateway, NAT Gateway, VPN, or Direct Connect. In a professional DevOps environment, VPC Endpoints are critical for increasing security, reducing data transfer costs, and meeting strict compliance standards. This chapter provides a deep dive into the two main types of VPC Endpoints—Gateway and Interface—and how to provision them using Terraform's `aws_vpc_endpoint` resource.

## 1. Gateway Endpoints vs. Interface Endpoints

Understanding the technical difference between these two types is essential for correct network design and cost management.

### 1.1 Gateway Endpoints
Gateway Endpoints are free of charge and work by adding prefix list routes to your VPC route tables.
- **Supported Services:** Amazon S3 and Amazon DynamoDB.
- **Connectivity:** No ENI (Network Interface) is created in your VPC. Traffic is routed directly to the service within the AWS network.

### 1.2 Interface Endpoints (PrivateLink)
Interface Endpoints have an hourly cost and a data processing fee.
- **Supported Services:** Most AWS services (SQS, SNS, Kinesis, ECR, etc.) and third-party SaaS services.
- **Connectivity:** Creates one or more Elastic Network Interfaces (ENIs) with private IP addresses in your subnets. Traffic is routed via DNS to these private IPs.

## 2. Provisioning S3 Gateway Endpoints

Gateway endpoints are the simplest to implement as they don't require security groups or specific subnets; they only require route table associations.

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.s3"
  service_type = "Gateway"

  # Automatically associate with these route tables
  route_table_ids = [aws_route_table.private.id, aws_route_table.public.id]

  tags = {
    Name = "S3-Gateway-Endpoint"
  }
}
```

## 3. Provisioning Interface Endpoints (PrivateLink)

Interface endpoints are more complex as they involve security groups and IP allocation.

```hcl
resource "aws_security_group" "vpc_endpoint_sg" {
  name   = "vpc-endpoint-sg"
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = [aws_vpc.main.cidr_block] # Allow access from within VPC
  }
}

resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.us-east-1.ecr.api"
  vpc_endpoint_type   = "Interface"
  
  # Deploy in multiple AZs for High Availability
  subnet_ids          = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids  = [aws_security_group.vpc_endpoint_sg.id]
  private_dns_enabled = true
}
```

## 4. The Power of Private DNS

When `private_dns_enabled = true` is set, AWS creates a private Route 53 zone that intercepts service DNS requests.
- **Standard URL:** `ecr.us-east-1.amazonaws.com`
- **Behavior:** Instead of resolving to a public IP, this URL will now resolve to the **Private IP** of the ENI created in your VPC.
- **Benefit:** Your application code doesn't need to change its API endpoint to take advantage of the private connection.

## 5. Security and Endpoint Policies

VPC Endpoints allow for a secondary layer of security called an **Endpoint Policy**. This is an IAM-style policy document that restricts what actions can be performed through the endpoint.

```hcl
resource "aws_vpc_endpoint_policy" "restrict_s3" {
  vpc_endpoint_id = aws_vpc_endpoint.s3.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action    = ["s3:GetObject", "s3:ListBucket"]
        Effect    = "Allow"
        Resource  = ["arn:aws:s3:::my-company-bucket/*"]
        Principal = "*" # Still subject to user's IAM permissions
      }
    ]
  })
}
```
Using Endpoint Policies allows you to enforce a "Data Perimeter" (e.g., ensuring your instances can only reach your own S3 buckets, not personal ones).

## 6. Cost Optimization Strategy (The "NAT Tax" Alternative)

The primary economic driver for VPC Endpoints is avoiding NAT Gateway data processing fees ($0.045/GB).
- **NAT GW cost:** $0.045/GB + $0.045/hour.
- **Interface Endpoint cost:** $0.01/GB + $0.01/hour.
For high-volume services like Kinesis or ECR (container image pulls), moving to Interface Endpoints can reduce networking costs by **70% or more**.

## 7. Operational Best Practices: Multi-AZ Deployment

Since Interface Endpoints create ENIs, they are subject to AZ failures.
- **High Availability:** Always provision Interface Endpoints in the same two or three Availability Zones where your application resides.
- **Health Checks:** AWS automatically handles the health and redundancy of the service behind the ENIs, so you don't need to manage Load Balancers for endpoints.

## 8. Troubleshooting Endpoint Connectivity

If a private instance cannot reach an AWS service through an endpoint:
1. **Security Group:** Is the Endpoint SG allowing port 443 from the instance's IP?
2. **DNS Hosting:** Is `enable_dns_support` and `enable_dns_hostnames` set to `true` in the VPC?
3. **IAM Permissions:** Does the IAM Role of the instance have permission to use the service? (Endpoint Policies don't grant permissions; they only restrict them).
4. **Subnet Routing:** For Interface endpoints, ensure the instance can reach the subnet where the Endpoint ENI lives.

## 9. Summary

VPC Endpoints are the final evolution of a secure cloud network. By keeping traffic completely within the AWS global backbone, you eliminate the security risks of public transit and the significant costs of NAT Gateway data processing. Terraform's unified `aws_vpc_endpoint` resource allows you to seamlessly integrate these sophisticated networking components into your architecture, ensuring that your data perimeters are strictly defined and consistently enforced.
