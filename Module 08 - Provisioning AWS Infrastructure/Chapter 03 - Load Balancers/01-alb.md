# High-Availability Traffic Orchestration: Provisioning AWS ALBs

The Application Load Balancer (ALB) is a Layer 7 (HTTP/HTTPS) load balancing service provided by AWS. It is designed to handle modern application traffic, providing advanced routing features such as host-based routing, path-based routing, and support for HTTP/2 and WebSockets. In a professional DevOps environment, the ALB is the entry point for the majority of web-facing services. It acts as the traffic cop, ensuring that incoming requests are distributed across healthy targets while providing a centralized location for SSL termination and security policy enforcement. This chapter provides a definitive exploration of the `aws_lb` resource, its technical attributes, and the architectural patterns for scalable web infrastructure.

## 1. Technical Anatomy of the ALB

The `aws_lb` resource defines the physical load balancer entity. Unlike a NAT Gateway, an ALB is a managed fleet of AWS-managed instances (though these are hidden from you).

```hcl
resource "aws_lb" "main" {
  name               = "production-web-alb"
  internal           = false # Public-facing
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_sg.id]
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_b.id]

  enable_deletion_protection = true

  access_logs {
    bucket  = aws_s3_bucket.logs.id
    prefix  = "alb-logs"
    enabled = true
  }

  tags = {
    Environment = "Production"
  }
}
```

### 1.1 Public (External) vs. Internal ALBs
- **External (`internal = false`):** Has a public DNS name and is reachable from the internet. It must be placed in **Public** subnets.
- **Internal (`internal = true`):** Has a private DNS name and is only reachable from within your VPC or over a VPN/Peering connection. It should be placed in **Private** subnets.

## 2. Security Group Integration

The ALB is protected by its own Security Group.
- **Ingress:** Typically allows port 80 and 443 from `0.0.0.0/0`.
- **Egress:** Must allow traffic to the Target Group (e.g., port 8080 or 443) where your application servers are listening.

**Architectural Tip:** In a tiered security model, your application instances (EC2) should only allow traffic **from the ALB Security Group ID**, ensuring that the instances are never directly accessible from the internet.

## 3. High Availability and Subnet Distribution

An ALB must be associated with at least two subnets, each in a different Availability Zone (AZ).
- **Why?** Since the ALB is a high-availability service, it needs footings in multiple data centers. If one AZ experiences an outage, AWS automatically updates the DNS of the ALB to point only to the healthy nodes in the remaining zones.
- **Scaling:** Unlike classic load balancers, the ALB scales its throughput by automatically adding more nodes behind its DNS name. There is no IP address to whitelist; you must always use the DNS CNAME.

## 4. Operational Monitoring: Access Logs

A production ALB should always have access logs enabled.
- **Log Destination:** Logs are pushed to an S3 bucket in raw format.
- **Content:** Every request is logged, including the client IP, response time, status code, and target ID. This is the primary source of truth for debugging 5xx errors or identifying malicious scrapers.
- **Terraform Note:** The S3 bucket must have a specific bucket policy allowing the ELB service principal to write to it.

## 5. Connection Idle Timeouts

By default, an ALB closes connections if no data is transferred for 60 seconds.
- **Timeout Management:** If your application performs long-running tasks (like generating a large PDF or processing a heavy data upload), you may need to increase the `idle_timeout` attribute to prevent "Gateway Timeout" (504) errors.

## 6. WAF (Web Application Firewall) Integration

The ALB can be directly linked to an AWS WAF Web ACL.
- **Function:** WAF provides protection against SQL injection, Cross-Site Scripting (XSS), and DDoS attacks at the HTTP level.
- **Terraform Pattern:** Use the `aws_wafv2_web_acl_association` resource to link your ALB ARN to your Web ACL ARN.

## 7. Cost Considerations: The LCU Model

ALB pricing is based on Load Balancer Units (LCUs) in addition to the hourly fee. LCUs measure:
1.  **New Connections:** Number of new connections per second.
2.  **Active Connections:** Number of concurrent connections.
3.  **Processed Bytes:** Amount of data transferred.
4.  **Rule Evaluations:** Complexity of your routing rules.

**DevOps Strategy:** To keep costs low, consolidate multiple applications behind a single ALB using path-based or host-based routing instead of creating a new ALB for every microservice.

## 8. Troubleshooting ALB Failures

- **"Unhealthy Targets":** Check the Security Group between the ALB and the EC2 instances. If the ALB cannot reach the port defined in the target group, the health check will fail, and the ALB will return 503 (Service Unavailable).
- **"404 Not Found":** Usually indicates that the request did not match any of the rules in the ALB Listeners.
- **"504 Gateway Timeout":** The application instance received the request but failed to respond within the `idle_timeout` period.

## 9. Summary

The Application Load Balancer is the intelligent gateway of the modern cloud network. By mastering the configuration of public/private visibility, ensuring multi-AZ redundancy, and implementing robust logging and WAF protection, you create a traffic management layer that is both resilient and highly performant. Terraform's declarative management of the ALB ensures that your entrance into the cloud environment is consistently provisioned, audited, and ready to scale with the demands of your global user base.
