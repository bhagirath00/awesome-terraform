# Stateless Defense: Provisioning AWS Network ACLs (NACLs)

Network Access Control Lists (NACLs) are an optional layer of security for your VPC that act as a firewall for controlling traffic in and out of one or more subnets. Unlike Security Groups, which are stateful and operate at the instance/ENI level, NACLs are stateless and operate at the subnet boundary. In a high-security DevOps environment, NACLs are used as a "Defense in Depth" mechanism to enforce coarse-grained network policies across entire network tiers. This chapter provides a deep dive into the `aws_network_acl` and `aws_network_acl_rule` resources, their technical statelessness, and the best practices for their implementation.

## 1. The Core Attribute: Statelessness

The most critical technical distinction between a NACL and a Security Group is **Statelessness**.
- **The Requirement:** If you allow inbound traffic on port 80, you MUST also define an explicit outbound rule to allow the response traffic to leave the subnet.
- **Ephemeral Ports:** Because responses are not automatically tracked, you must open the ephemeral port range (typically 1024-65535) for outbound traffic to allow standard client-server communication to function.

## 2. NACL Resource Structure in Terraform

Similar to Security Groups, it is a professional best practice to define NACL rules as standalone resources rather than inline within the `aws_network_acl` block.

```hcl
resource "aws_network_acl" "main" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = [aws_subnet.public.id]

  tags = {
    Name = "Public-Tier-NACL"
  }
}

resource "aws_network_acl_rule" "allow_inbound_http" {
  network_acl_id = aws_network_acl.main.id
  rule_number    = 100
  egress         = false # Inbound rule
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 80
  to_port        = 80
}
```

## 3. Order of Evaluation: Rule Numbers

Security Groups don't have rule numbers; they simply merge all rules into a single "Allow" set. NACLs, however, use an ordered list of rules.
- **Evaluation:** Rules are processed starting with the lowest rule number. Once a match is found (either Allow or Deny), the evaluation stops.
- **Impact:** You can use NACLs to **explicitly deny** traffic from a specific IP range (e.g., blocking a known malicious botnet) by placing a "Deny" rule with a lower number than your "Allow All" rule.

## 4. The Default NACL and Security Hardening

When a VPC is created, it comes with a Default NACL.
- **Default State:** Allows all inbound and outbound traffic.
- **Standard Posture:** Because NACLs are complex to manage due to their statelessness, many teams leave the Default NACL as "Allow All" and rely on Security Groups for granular control.
- **Hardened Posture:** For highly regulated environments (PCI-DSS, HIPAA), you should use Terraform to replace the Default NACL with a "Deny All" configuration and only open the specific ports required for the tier.

## 5. Ephemeral Port Considerations

A common failure point in NACL configuration is forgetting the "Outbound Ephemeral" ports. 
- When an external client connects to your web server on port 443, the web server responds on a high-number port (1024-65535) chosen by the OS.
- If your NACL does not have an outbound rule for `1024-65535`, the response will be blocked at the subnet boundary, and the connection will time out, even if the Security Group allows it.

```hcl
resource "aws_network_acl_rule" "allow_outbound_ephemeral" {
  network_acl_id = aws_network_acl.main.id
  rule_number    = 200
  egress         = true # Outbound rule
  protocol       = "tcp"
  rule_action    = "allow"
  cidr_block     = "0.0.0.0/0"
  from_port      = 1024
  to_port        = 65535
}
```

## 6. NACLs as a "Blast Radius" Control

NACLs are excellent for preventing "Lateral Movement" within a VPC.
- If you have an "Internal" subnet and an "External" subnet, you can use a NACL on the internal subnet to strictly forbid any traffic originating from the external IP range, providing an extra layer of protection if a public-facing instance is compromised.

## 7. Operational Constraints

- **Subnet Association:** A subnet can only be associated with ONE NACL at a time. Associating a subnet with a new NACL automatically removes the previous association.
- **Quotas:** AWS limits the number of rules per NACL (Standard: 20 per direction). This limit is low because processing NACLs at the hardware level consumes VPC resources. If you need more than 20 rules, it is often a sign that you should be using Security Groups for that logic.

## 8. Summary

Network ACLs provide a robust, stateless security layer at the network edge. By understanding the importance of rule numbering, the necessity of ephemeral port management, and the technical distinction from stateful Security Groups, you can build a multi-layered "Defense in Depth" architecture. Terraform's standalone `aws_network_acl_rule` resources allow you to manage these complex rulesets with the precision and version control required for enterprise security compliance.
