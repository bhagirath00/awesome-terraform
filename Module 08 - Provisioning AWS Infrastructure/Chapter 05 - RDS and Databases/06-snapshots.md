# Data Protection: Provisioning RDS DB Snapshots

DB Snapshots are point-in-time copies of your entire DB instance. They serve as the primary recovery mechanism for data loss, corruption, or catastrophic human error. In a professional DevOps environment, snapshots are used for disaster recovery, environment cloning, and compliance-mandated long-term data retention. This chapter explores the technical configuration of manual snapshots, the mechanics of automated backups, and the strategies for managing and sharing snapshots across AWS accounts using Terraform.

## 1. Automated Backups vs. Manual Snapshots

Understanding the difference between these two systems is critical for data durability planning.

- **Automated Backups:** Managed by AWS. They occur during your defined backup window and are automatically deleted after the `backup_retention_period` (up to 35 days). These are used for Point-in-Time Recovery (PITR).
- **Manual Snapshots:** Created by you (or Terraform). They persist indefinitely until you explicitly delete them. They are not affected by the deletion of the source database.

## 2. Provisioning Manual Snapshots with Terraform

The `aws_db_snapshot` resource creates a standalone backup of an existing database.

```hcl
resource "aws_db_snapshot" "pre_migration" {
  db_instance_identifier = aws_db_instance.main.identifier
  db_snapshot_identifier = "pre-schema-migration-snapshot"
}
```

## 3. Snapshot Security and Sharing

Snapshots inherit the encryption status of the source database. If your RDS instance is encrypted with a KMS key, the snapshot is also encrypted.

### 3.1 Cross-Account Copying
In an enterprise "Hub and Spoke" model, you often need to share a Production snapshot with a "Dev" account for testing purposes.
1.  **Authorize:** Modify the KMS key policy to allow the target account to use the key.
2.  **Share:** Use the `aws_db_snapshot_copy` resource to create a copy of the snapshot and share it with the target account ID.

## 4. Disaster Recovery: Cross-Region Copying

For extreme resiliency, you should copy your snapshots to a completely different AWS region (e.g., from `us-east-1` to `us-west-2`).

```hcl
resource "aws_db_snapshot_copy" "region_backup" {
  source_db_snapshot_identifier = aws_db_snapshot.pre_migration.db_snapshot_arn
  target_db_snapshot_identifier = "dr-backup-us-west-2"
  destination_region            = "us-west-2"
}
```

## 5. Restoration: Bringing Back the Data

To restore a database from a snapshot, you do not "Overwrite" the existing instance. Instead, you launch a **NEW** instance using the snapshot.

```hcl
resource "aws_db_instance" "restored" {
  instance_class      = "db.t3.medium"
  snapshot_identifier = aws_db_snapshot.pre_migration.id
  # Other settings...
}
```
**Operational Note:** After restoration, you must update your application's connection string to point to the endpoint of the NEW database instance.

## 6. Final Snapshots on Deletion

A professional Terraform configuration should always take a "Final Snapshot" whenever a database is destroyed.
- **`skip_final_snapshot = false`**: Forces the AWS API to take a backup before deleting.
- **`final_snapshot_identifier`**: Provides a specific name for this backup (e.g., `prod-db-final-backup-2023`).

## 7. Summary

Snapshots are your insurance policy against data loss. By combining the automated Point-in-Time Recovery of AWS with the declarative, versioned management of manual snapshots through Terraform, you create a data infrastructure that is virtually indestructible. Whether you are performing high-risk schema migrations, copying data across accounts for development, or implementing a global disaster recovery strategy, snapshots provide the reliable foundation needed for modern enterprise data management.
