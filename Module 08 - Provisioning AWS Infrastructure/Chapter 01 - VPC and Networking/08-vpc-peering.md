# Connecting Networks: Provisioning AWS VPC Peering

VPC Peering is a networking connection between two VPCs that enables you to route traffic between them using private IPv4 or IPv6 addresses. In a professional DevOps environment, peering is essential for multi-VPC architectures (e.g., connecting a "Shared Services" VPC containing CI/CD tools to a "Production" VPC). Peering is not transitive and requires mutual agreement between the requester and the accepter. This chapter explores the technical handshake of the `aws_vpc_peering_connection` resource, cross-account peering logic, and the critical routing updates required to enable traffic flow.

## 1. The Peering Lifecycle and Handshake

A VPC Peering connection is a three-stage process: Request, Acceptance, and Routing.

### 1.1 The Request (`aws_vpc_peering_connection`)
One VPC acts as the "Requester".

```hcl
resource "aws_vpc_peering_connection" "peer" {
  vpc_id        = aws_vpc.main.id # Requester
  peer_vpc_id   = aws_vpc.other.id # Accepter
  peer_owner_id = var.peer_account_id
  auto_accept   = false # Must be true for same-account/same-region only

  tags = {
    Name = "VPC-Peering-Main-to-Other"
  }
}
```

### 1.2 The Acceptance (`aws_vpc_peering_connection_accepter`)
If the VPCs are in different accounts or regions, the accepter must explicitly accept the request.

```hcl
resource "aws_vpc_peering_connection_accepter" "peer" {
  vpc_peering_connection_id = aws_vpc_peering_connection.peer.id
  auto_accept               = true

  tags = {
    Side = "Accepter"
  }
}
```

## 2. Technical Constraint: No Overlapping CIDRs

The most fundamental requirement for VPC Peering is that the two VPCs **must not have overlapping CIDR blocks**.
- If VPC A is `10.0.0.0/16` and VPC B is `10.0.0.0/24`, the peering connection will fail.
- **Why?** Since peering uses the VPC's internal routing, identical IP ranges create "Routing Ambiguity" that the AWS VPC router cannot resolve. Always plan your VPC ranges across your entire organization to be unique if you ever intent to peer them.

## 3. The "Non-Transitive" Nature of Peering

VPC Peering is NOT transitive. 
- If VPC A is peered with VPC B, and VPC B is peered with VPC C, **VPC A cannot talk to VPC C** through VPC B.
- To enable communication between A and C, you must create an explicit peering connection directly between A and C.
- for massive multi-VPC environments, this leads to a "Full Mesh" complexity disaster. In these cases, you should use **AWS Transit Gateway** instead of VPC Peering.

## 4. Routing Architecture for Peering

Creating the peering connection is only half the battle. You must manually add routes to the route tables of **both** VPCs.

### 4.1 Requester Route Table
```hcl
resource "aws_route" "requestor_route" {
  route_table_id            = aws_route_table.main.id
  destination_cidr_block    = aws_vpc.other.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.peer.id
}
```

### 4.2 Accepter Route Table
```hcl
resource "aws_route" "accepter_route" {
  route_table_id            = aws_route_table.other.id
  destination_cidr_block    = aws_vpc.main.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.peer.id
}
```

## 5. Security Group Logic in Peered Networks

Security Groups can reference Security Group IDs from a peered VPC, but only if the peering is within the **same region**.

```hcl
resource "aws_security_group_rule" "allow_peer_access" {
  type                     = "ingress"
  from_port                = 80
  to_port                  = 80
  protocol                 = "tcp"
  security_group_id        = aws_security_group.local.id
  source_security_group_id = "sg-xxxxxxx" # ID from the other VPC
}
```
If you are peering across regions (Inter-Region Peering), you cannot use Security Group references; you must use the CIDR block of the peer VPC in your security rules.

## 6. DNS Resolution over Peering

By default, an instance in VPC A cannot resolve the private DNS name of an instance in VPC B. To enable this, you must modify the peering connection options.

```hcl
resource "aws_vpc_peering_connection_options" "dns" {
  vpc_peering_connection_id = aws_vpc_peering_connection.peer.id

  requester {
    allow_remote_vpc_dns_resolution = true
  }

  accepter {
    allow_remote_vpc_dns_resolution = true
  }
}
```

## 7. Operational Best Practices: Automation and State

- **Side-by-Side Deletion:** When a peering connection is deleted, the routes associated with it enter a "Black Hole" state. Terraform is excellent at identifying and cleaning up these dangling routes.
- **Accepter Management:** If you are managing peering between two accounts that you own, you should use two different AWS provider aliases in the same Terraform configuration to handle the request and acceptance in a single `apply` cycle.

## 8. Summary

VPC Peering is the simplest and most performant way to link two virtual networks in AWS. By understanding the handshake process, strictly avoiding CIDR overlaps, and ensuring that route tables are updated on both sides of the connection, you can build a secure, interconnected infrastructure. While peering has limits regarding transitivity and "Mesh" complexity, it remains a fundamental tool for isolating environments while still allowing for shared service discovery and cross-environment communication managed through the precision of Terraform.
