# Horizontal Data Scaling: Provisioning RDS Read Replicas

RDS Read Replicas allow you to create one or more read-only copies of your primary database instance within the same AWS Region or in a different AWS Region. They are designed to offload read-intensive workloads from the primary database, increasing the overall throughput of your application. In a professional DevOps environment, read replicas are used for scaling web applications, running analytics/reporting queries without affecting production, and implementation of cross-region disaster recovery. This chapter provides a definitive deep dive into the `replicate_source_db` attribute and the strategies for managing distributed database read layers using Terraform.

## 1. Technical Mechanics of Read Replication

Unlike the synchronous standby in a Multi-AZ deployment (which is for High Availability), Read Replicas use **Asynchronous Replication**.

- **Workflow:** The primary instance sends database updates to the replicas. Because it is asynchronous, there is usually a small delay (milliseconds to seconds) known as "Replica Lag."
- **The Endpoint:** Each replica has its own unique DNS endpoint. Your application must be configured to send "Write" traffic to the primary and "Read" traffic to the replica(s).

## 2. Provisioning a Read Replica with Terraform

In Terraform, a read replica is just another `aws_db_instance` resource, but with the `replicate_source_db` attribute set to the ID or ARN of the primary instance.

```hcl
resource "aws_db_instance" "primary" {
  identifier = "prod-db-primary"
  # ... base config ...
  backup_retention_period = 7 # MUST be > 0 for replication
}

resource "aws_db_instance" "replica" {
  identifier             = "prod-db-replica-1"
  replicate_source_db    = aws_db_instance.primary.id
  instance_class         = "db.t3.medium"
  publicly_accessible    = false
  
  # Note: Most parameters (engine, storage, etc.) are inherited from the primary
}
```

## 3. The "Backup Requirement" Constraint

For RDS to enable replication, the primary database **must** have automated backups enabled.
- **Terraform Detail:** setting `backup_retention_period` to a value greater than `0` on the primary is a technical prerequisite. If it is set to `0`, the creation of the read replica will fail with an "InvalidParameterValue" error.

## 4. Cross-Region Read Replicas

You can place a read replica in a completely different AWS region. This is a common strategy for:
1.  **Global Performance:** Providing low-latency reads to users in another part of the world.
2.  **Disaster Recovery:** If your primary region goes down, you can promote the cross-region replica to become a standalone primary database in its own region.

```hcl
resource "aws_db_instance" "dr_replica" {
  provider            = aws.us_west_2 # Use a different provider alias
  identifier          = "prod-db-dr-replica"
  replicate_source_db = aws_db_instance.primary.arn # Use ARN for cross-region
  instance_class      = "db.t3.medium"
}
```

## 5. Promotion: From Replica to Primary

If the primary database fails or if you are performing a regional migration, you can "Promote" a read replica.
- **Promotion Process:** The replica stops following the primary, finishes processing any lagging transactions, and becomes a standard, standalone database instance.
- **Terraform Strategy:** In an automated DR scenario, you would update the Terraform state to remove the `replicate_source_db` attribute and provide a full configuration for the instance to maintain its state as the new "Primary."

## 6. Monitoring Replica Lag

High replica lag is a major risk for application data consistency.
- **CloudWatch Metrics:** Monitor the `ReplicaLag` metric. If it grows too high, it usually indicates that the replica's instance class is too small to keep up with the primary's write volume, or there is network congestion.
- **Scaling:** Professional teams often use different instance classes for replicas (e.g., a large primary for writes and multiple smaller replicas for reads).

## 7. Operational Best Practices: Multi-AZ Replicas

You can (and often should) enable Multi-AZ on your read replicas as well. This ensures that your read layer is just as resilient as your write layer, preventing a single AZ failure from reducing your read throughput by 50% or more.

## 8. Summary

RDS Read Replicas are the primary tool for scaling the most difficult part of the infrastructure: the data layer. By leveraging tiered replication (Local and Cross-Region) and ensuring that your primary has robust backup policies enabled, you build a database architecture that is both globally performant and ready for regional failover. Terraform's ability to chain these instances together through the `replicate_source_db` attribute ensures that your data scaling strategy is declarative, versioned, and easily reversible.
