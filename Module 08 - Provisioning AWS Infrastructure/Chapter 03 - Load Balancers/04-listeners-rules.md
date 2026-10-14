# Smart Routing: Provisioning AWS LB Listeners and Rules

Listeners and rules are the intelligence of the Application Load Balancer. While the LB defines the entry point and the Target Group defines the destination, the Listeners and Rules define the logic that connects them. They evaluate incoming requests based on hostnames, paths, headers, and other attributes to decide which target group should handle the traffic—or if the traffic should be redirected or blocked entirely. In a professional DevOps environment, this layer is used to implement microservices architectures, blue/green deployments, and forced SSL redirection. This chapter provides a definitive deep dive into the `aws_lb_listener` and `aws_lb_listener_rule` resources and the architectural patterns for sophisticated traffic routing.

## 1. The Listener: The Entry Guard

An ALB listener is a process that checks for connection requests using the protocol and port you configure.

### 1.1 The HTTP to HTTPS Redirect
In any production environment, you should never serve unencrypted traffic. A common pattern is to have a listener on port 80 that automatically redirects all traffic to port 443 (HTTPS).

```hcl
resource "aws_lb_listener" "http_redirect" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

### 1.2 The HTTPS Listener and SSL Termination
The HTTPS listener is where the Load Balancer performs SSL/TLS termination, decrypting the traffic before sending it to your backend.

```hcl
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = var.acm_certificate_arn

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "Access Denied: Use a valid hostname."
      status_code  = "403"
    }
  }
}
```

## 2. Listener Rules: The Routing Logic Engine

Rules are attached to listeners and allow you to route traffic based on the content of the HTTP request. Rules are evaluated in order of their **Priority**.

### 2.1 Host-Based Routing
Used for multi-tenant applications or microservices sharing a single LB.
- **Example:** `api.example.com` goes to the API target group, while `app.example.com` goes to the Web target group.

```hcl
resource "aws_lb_listener_rule" "api_routing" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 10

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    host_header {
      values = ["api.example.com"]
    }
  }
}
```

### 2.2 Path-Based Routing
Allows you to segment your application logic into different backends based on the URL path.
- **Example:** `/v1/*` goes to the legacy backend, while `/v2/*` goes to the new microservice.

```hcl
resource "aws_lb_listener_rule" "images" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 20

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.static_assets.arn
  }

  condition {
    path_pattern {
      values = ["/images/*", "/assets/*"]
    }
  }
}
```

## 3. Advanced Conditions: Headers and Client IPs

Modern ALBs can route based on more than just the URL.
- **HTTP Header:** Route traffic based on a custom header like `X-Feature-Gate: beta`. This is excellent for A/B testing.
- **Query String:** Route based on parameters in the URL (e.g., `?version=new`).
- **Source IP (CIDR):** Restrict access to specific paths (like `/admin`) to your corporate office's IP range.

## 4. Weighted Routing (Blue/Green Deployment)

Instead of sending 100% of traffic to one target group, you can specify weights to perform a gradual rollout of a new version.

```hcl
action {
  type = "forward"
  forward {
    target_group {
      arn    = aws_lb_target_group.blue.arn
      weight = 90
    }
    target_group {
      arn    = aws_lb_target_group.green.arn
      weight = 10
    }
    stickiness {
      enabled  = true
      duration = 600
    }
  }
}
```
Setting `weight` allows you to send only 10% of users to the new version ("Green") and verify its stability before committing to a 100% cutover.

## 5. Fixed Responses and Maintenance Pages

The ALB can return a static response without having any backend servers running.
- **Maintenance Mode:** If you are performing a database migration, you can update your default action to return a 503 status code with a "Down for Maintenance" message.
- **Pre-flight Checks:** Return a simple 200 OK for health check systems that query the Load Balancer itself.

## 6. Priority and Rule Evaluation Logic

- **Evaluation Order:** Rules are evaluated in numeric order from lowest to highest.
- **The Default Action:** Every listener MUST have a default action. This is the "Catch-All" that executes if no other rules match.
- **Overlapping Rules:** If a request matches both `*.example.com` and `/api/*`, the rule with the lower `priority` number wins.

## 7. Security Policy and TLS Versions

When creating an HTTPS listener, you must select an `ssl_policy`.
- **Best Practice:** Avoid "Legacy" policies that support TLS 1.0 or 1.1. Always use policies that enforce **TLS 1.2 or 1.3** and strong cipher suites (e.g., `ELBSecurityPolicy-TLS13-1-2-2021-06`).
- **SNI (Server Name Indication):** The ALB supports multiple SSL certificates on a single listener. You can use the `aws_lb_listener_certificate` resource to attach additional certificates for different hostnames.

## 8. Summary

Listeners and rules are where your networking infrastructure becomes "Application Aware." By leveraging host-based routing, path-based segmenting, and weighted traffic distribution, you transform a simple load balancer into a sophisticated orchestration tool for microservices and blue/green deployments. Terraform's declarative management of these rules provides a version-controlled, audit-friendly way to manage complex traffic patterns, ensuring that your users are always routed to the correct backend with the highest level of security and performance.
