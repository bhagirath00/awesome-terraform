# Extreme Performance: Provisioning AWS Network Load Balancers (NLB)

The Network Load Balancer (NLB) operates at the fourth layer (Transport Layer) of the OSI model. It is designed to handle millions of requests per second while maintaining ultra-low latencies. Unlike the Application Load Balancer (ALB), which understands HTTP and path-based routing, the NLB only understands IP addresses, TCP/UDP ports, and protocols. In a professional DevOps environment, NLBs are used for high-performance workloads, static IP requirements, and protocols other than HTTP. This chapter explores the technical configuration of the `aws_lb` resource for Network types, the use of Elastic IPs for static load balancing, and the architectural differences between Layer 4 and Layer 7 distribution.

## 1. Technical Anatomy of the NLB

A Network Load Balancer is identified by the `load_balancer_type = "network"` argument.

```hcl
resource "aws_lb" "network" {
  name               = "prod-high-perf-nlb"
  internal           = false # Public-facing
  load_balancer_type = "network"
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_b.id]

  # NLBs do not use Security Groups (they rely on their targets' SGs)
  # EXCEPT in recent AWS updates where security groups are optional
}
```

## 2. The Static IP Advantage

The primary technical differentiator of the NLB is the support for **static IP addresses**.
- **The ALB Problem:** An ALB's IP addresses change dynamically as it scales. You must always use its DNS name.
- **The NLB Solution:** You can assign an Elastic IP (EIP) to each subnet of the NLB. This provides a set of fixed, immutable IP addresses.

```hcl
resource "aws_lb" "static_nlb" {
  name               = "static-ip-nlb"
  load_balancer_type = "network"

  subnet_mapping {
    subnet_id     = aws_subnet.public_a.id
    allocation_id = aws_eip.nlb_a.id
  }

  subnet_mapping {
    subnet_id     = aws_subnet.public_b.id
    allocation_id = aws_eip.nlb_b.id
  }
}
```

**Use Case:** Ideal for connecting with third-party partners who have strict firewall policies that only allow communication from specific, whitelisted IP addresses.

## 3. Preserving the Client Source IP

By default, an NLB preserves the source IP address of the client in the packet passed to the backend.
- **ALB Behavior:** The ALB replaces the source IP with its own and attaches the client's IP to the `X-Forwarded-For` header.
- **NLB Behavior:** The destination instance sees the packet as if it came directly from the client. This is essential for applications that perform IP-based rate limiting or geography-based security at the kernel/application level.

## 4. Protocols and Listeners

Because it is Layer 4, an NLB supports protocols that an ALB cannot handle.
- **TCP and UDP:** Supports raw data streams (e.g., for gaming, VoIP, or proprietary IoT protocols).
- **TLS Termination:** While it is Layer 4, the NLB can still perform TLS termination using the `TLS` protocol. This is known as "Secure TCP."

## 5. Health Checks and Target Groups

Health checks for an NLB are typically "TCP Ping" based.
- **The Check:** The NLB simply tries to open a TCP connection to the target port. If the connection is accepted, the target is considered healthy.
- **Speed:** NLB health checks are incredibly fast, allowing for sub-second failovers in high-density environments.

## 6. NLB vs ALB: Choosing the Right Tool

| Feature | ALB (Layer 7) | NLB (Layer 4) |
|---------|---------------|---------------|
| **Protocol** | HTTP / HTTPS / gRPC | TCP / UDP / TLS |
| **Routing** | Path, Host, Headers | Port, IP address |
| **Latencies** | Milliseconds | Microseconds |
| **Static IPs** | No | Yes |
| **Load Balancing** | Complex rules | Simple distribution |

## 7. Operational Best Practices: Multi-AZ Design

Just like the ALB, an NLB should be deployed in multiple Availability Zones. 
- **Zonal DNS:** AWS provides a DNS name for each AZ node of the NLB. If you need fine-grained control, you can route traffic to a specific data center.
- **Cross-Zone Load Balancing:** By default, Cross-Zone load balancing is **Disabled** on NLBs to reduce latency. This means an NLB node only sends traffic to targets in the same AZ. You can enable it via Terraform if you need even distribution across AZs.

```hcl
resource "aws_lb" "perf" {
  # ...
  enable_cross_zone_load_balancing = true
}
```

## 8. Summary

The Network Load Balancer is the high-performance architect's preferred choice for Layer 4 traffic and static IP requirements. While it lacks the message-level intelligence of the ALB, its ability to handle massive throughput with microsecond latency and its support for raw TCP/UDP streams makes it the indispensable tool for specialized cloud architectures. Terraform's ability to manage the delicate mapping of subnets to Elastic IPs and the cross-zone balancing configuration ensures that your network core is both rock-solid and infinitely scalable.
