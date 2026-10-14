# Data Integrity and Cost Control: S3 Versioning and Lifecycle Management

Managing data in Amazon S3 involves more than just storage; it requires a strategy for maintaining data integrity and optimizing costs as data ages. S3 Versioning protects your data from accidental deletion or overwrites by keeping multiple variants of an object in the same bucket. S3 Lifecycle Management automates the transition of objects between various storage tiers and their eventual expiration, ensuring you only pay for the performance level your data actually requires. This chapter provides a definitive exploration of the `aws_s3_bucket_versioning` and `aws_s3_bucket_lifecycle_configuration` resources, and the strategies for managing the full lifecycle of your cloud data.

## 1. S3 Versioning: The Safety Net

Versioning is a state-level configuration on a bucket that preserves every version of every object stored in it.

```hcl
resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.data_lake.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

### 1.1 How Versioning Works
- **The Version ID:** When versioning is enabled, every object is assigned a unique version ID.
- **Overwrites:** If you upload a new file over an existing one, the old file is kept as a "Non-current" version, and the new file becomes the "Current" version.
- **Deletes:** If you delete an object, AWS places a "Delete Marker" on it. The data is still there! To recover, you simply delete the Delete Marker or specify a previous Version ID in your request.

### 1.2 Suspending Versioning
You can "suspend" versioning at any time. This stops the creation of new versions but preserves all existing ones. You cannot truly "Disable" versioning once it has been enabled.

## 2. Lifecycle Management: Automating Cost Optimization

Lifecycle rules allow you to define a set of actions that S3 applies to a group of objects. These rules are processed by AWS in the background once per day.

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "data_retention" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    id     = "archive-old-logs"
    status = "Enabled"

    filter {
      prefix = "logs/"
    }

    # Transition current version to IA then Glacier
    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    # Automatically delete objects after 1 year
    expiration {
      days = 365
    }
  }
}
```

## 3. Managing Noncurrent Versions (The Cost Saver)

If versioning is enabled, a lifecycle rule that only uses `expiration` will simply create a Delete Marker, keep the old version, and **keep charging you for storage**. To actually save money, you must define `noncurrent_version_expiration`.

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "version_cleanup" {
  # ...
  rule {
    id     = "cleanup-old-versions"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 60
      storage_class   = "GLACIER"
    }

    noncurrent_version_expiration {
      noncurrent_days = 180 # Delete forever after 6 months
    }
  }
}
```

## 4. Cleaning Up Failed Multipart Uploads

When you upload large files (multi-GB), S3 breaks them into parts. If the upload is interrupted, those parts stay in your bucket, and AWS charges you for them even though they don't appear as objects.

**Professional Best Practice:** Every production bucket should have a rule to abort incomplete multipart uploads.

```hcl
rule {
  id     = "abort-failed-uploads"
  status = "Enabled"
  abort_incomplete_multipart_upload {
    days_after_initiation = 7
  }
}
```

## 5. Intelligent-Tiering: The Hands-off Approach

For data with unpredictable access patterns, manually writing transition rules can be difficult. 
- **The Service:** S3 Intelligent-Tiering automatically monitors access patterns and moves objects between a "Frequent" and "Infrequent" access tier.
- **Terraform Integration:** Use the `aws_s3_bucket_intelligent_tiering_configuration` resource to define which objects should be managed this way.

## 6. S3 Object Lock and Compliance

For highly regulated industries (Finance, Healthcare), you may be required to prevent data from being deleted by anyone—including the root account—for a specific period.
- **Object Lock:** Implements a WORM (Write Once, Read Many) model.
- **Retention Modes:** 
    - **Governance Mode:** Prevents deletion by most users, but specific users with high permissions can bypass it.
    - **Compliance Mode:** Deletion is impossible even for the Root account until the retention period ends. **Use with extreme caution.**

```hcl
resource "aws_s3_bucket" "immutable" {
  bucket = "company-immutable-archives"
  # This MUST be enabled at bucket creation
  object_lock_enabled = true
}
```

## 7. Operational Best Practices: MFA Delete

To prevent accidental mass-deletion, you can enable **MFA Delete**. 
- **Technicality:** This requires the bucket owner to provide an MFA code (from a physical token or app) to either change the versioning state or permanently delete a version.
- **Terraform Note:** This currently cannot be fully managed through the AWS provider alone and often requires a CLI-based administrative setup for the bucket owner.

## 8. Troubleshooting Lifecycle Rules

- **Lag Time:** Lifecycle actions are not instantaneous. They are processed in a 24-hour cycle. If you set a rule to delete after 1 day, it might take 48 hours for the storage metric to actually decrease.
- **Conflict of Interests:** Ensure that your `expiration` days are greater than your `transition` days. If you transition to Glacier at 30 days but expire at 15 days, the transition will never occur.
- **Filter Precision:** Use `filter { prefix = "xxx" }` carefully. A leading slash (e.g., `/logs/`) can sometimes prevent a match if your upload script doesn't include it.

## 9. Summary

Managing the time-series and storage-tier of your data is the hallmark of a mature DevOps engineer. By implementing S3 Versioning, you create an "Undo Button" for your most critical assets. By leveraging automated Lifecycle Configurations, you ensure that your data infrastructure remains cost-efficient as it grows to massive scale. Terraform's declarative management of these policies provides the necessary governance to ensure that your organization's data retention and cost-savings policies are perfectly enforced across every region in your cloud estate.
