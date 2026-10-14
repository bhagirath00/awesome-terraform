# Securing Traffic: Managing SSL/TLS with AWS Certificate Manager (ACM)

In a modern cloud infrastructure, unencrypted HTTP traffic is a security risk and is often prohibited by organizational compliance standards. AWS Certificate Manager (ACM) provides a managed service for provisioning, managing, and deploying public and private SSL/TLS certificates. When integrated with an Application Load Balancer (ALB), ACM handles the complex cryptographical offloading and automatic certificate renewal. This chapter provides a definitive exploration of the `aws_acm_certificate` resource, the DNS validation handshake, and the technical patterns for attaching certificates to load balancers using Terraform.

## 1. The `aws_acm_certificate` Resource

Provisioning a certificate is an asynchronous process. You request a certificate for a specific domain name, and AWS waits for proof that you own that domain.

```hcl
resource "aws_acm_certificate" "cert" {
  domain_name       = "api.example.com"
  validation_method = "DNS"

  subject_alternative_names = [
    "app.example.com",
    "*.example.com"
  ]

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Environment = "Production"
  }
}
```

## 2. The Verification Handshake: DNS Validation

ACM requires proof of domain ownership. While "Email Validation" is an option, it is not automation-friendly. **DNS Validation** is the professional standard for DevOps workflows.

### 2.1 Generating the Validation Records
When you request a certificate with `validation_method = "DNS"`, ACM provides a set of CNAME records that must be added to your DNS provider (e.g., Route 53).

```hcl
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.cert.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.main.zone_id
}
```

### 2.2 The Validation Waiter (`aws_acm_certificate_validation`)
Terraform provides a specialized resource that "waits" for the certificate to become status `ISSUED`. This acts as a barrier in the dependency graph, ensuring that downstream resources (like the Load Balancer) don't try to use a pending certificate.

```hcl
resource "aws_acm_certificate_validation" "cert" {
  certificate_arn         = aws_acm_certificate.cert.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}
```

## 3. Attaching Certificates to Load Balancers

Once the certificate is validated, it can be attached to an ALB Listener.

```hcl
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.cert.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

## 4. Multi-Certificate Support (SNI)

A single ALB listener can serve multiple domain names (e.g., `site-a.com` and `site-b.com`). This is handled via **Server Name Indication (SNI)**.
- **Pattern:** Use the `aws_lb_listener_certificate` resource to attach additional certificates to an existing listener.

```hcl
resource "aws_lb_listener_certificate" "additional" {
  listener_arn    = aws_lb_listener.https.arn
  certificate_arn = aws_acm_certificate.secondary.arn
}
```

## 5. Automatic Renewals and Lifecycle

One of the greatest benefits of ACM is **Automatic Renewal**.
- **DNS Validation Advantage:** As long as the DNS validation records exist in Route 53, AWS will automatically renew the certificate 60 days before it expires. This eliminates the risk of human error leading to expired "Production" certificates.
- **The `create_before_destroy` Rule:** Always use this lifecycle rule. If you need to replace a certificate (e.g., to add a new SAN), Terraform will create the new one first, validate it, and then update the Load Balancer before deleting the old one, ensuring zero downtime.

## 6. Private Certificates (AWS Private CA)

In addition to public certificates, ACM can issue private certificates for internal service-to-service communication.
- **Use Case:** Encrypting traffic between an internal ALB and a backend microservice.
- **Terraform Integration:** The workflow is identical, but the `certificate_authority_arn` must be provided to a private ACM CA managed by your security team.

## 7. Troubleshooting ACM Deployment

- **"Validation Timed Out":** Ensure that your Route 53 zone ID is correct and that the CNAME records are publicly resolvable. If you are using a Private DNS zone, ACM cannot perform DNS validation.
- **"Invalid Certificate ARN":** Ensure you are passing the ARN from the `aws_acm_certificate_validation` resource, not the raw `aws_acm_certificate`. Passing the raw resource might result in an "In-Use" error if the certificate isn't yet in the `ISSUED` state.

## 8. Summary

AWS Certificate Manager is the security backbone of encrypted cloud communication. By automating the request, DNS validation, and attachment of SSL/TLS certificates through Terraform, you eliminate one of the most common causes of production downtime—expired certificates. Integrating ACM with your Application Load Balancers ensures that your user traffic is always encrypted using modern, high-strength cipher suites while significantly reducing the operational overhead of certificate lifecycle management.
