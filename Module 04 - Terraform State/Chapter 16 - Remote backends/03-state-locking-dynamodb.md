# State Locking with DynamoDB

## Introduction

State locking is the mechanism that prevents two Terraform operations from running simultaneously against the same state file. Without locking, two engineers running `terraform apply` at the same time can corrupt the state file — each reads the same state, makes different changes, and one overwrites the other's work.

DynamoDB is the locking backend used with the S3 state backend on AWS. When Terraform starts an operation that modifies state, it creates a lock entry in DynamoDB. When the operation finishes, it deletes the lock entry. Any other Terraform process trying to start at the same time sees the lock and waits or fails — preventing concurrent modifications.

---

## How DynamoDB Locking Works

The locking process step by step:

**Step 1 — Acquiring the Lock**

When you run `terraform apply` (or `terraform plan` in some cases), Terraform writes a lock entry to DynamoDB:

```json
{
  "LockID": "mycompany-terraform-state/production/terraform.tfstate",
  "Info": "{\"ID\":\"abc123\",\"Operation\":\"OperationTypeApply\",\"Who\":\"alice@laptop\",\"Version\":\"1.6.6\",\"Created\":\"2024-01-15T10:30:00Z\",\"Path\":\"mycompany-terraform-state/production/terraform.tfstate\"}"
}
```

The `LockID` is the S3 bucket name + key path. The `Info` field contains details about who holds the lock, what operation they are running, and when the lock was created.

**Step 2 — Running the Operation**

With the lock held, Terraform reads the state from S3, executes the plan, and writes the new state to S3. No one else can start a conflicting operation during this time.

**Step 3 — Releasing the Lock**

When the operation completes (successfully or with an error), Terraform deletes the lock entry from DynamoDB. The next operation can now acquire the lock.

---

## Creating the DynamoDB Table

The DynamoDB table for locking has a simple structure — just one attribute called `LockID` as the partition key.

### Using the AWS CLI

```bash
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

Key points:
- `AttributeName=LockID` — the attribute name must be exactly `LockID`
- `AttributeType=S` — it is a String type
- `billing-mode PAY_PER_REQUEST` — you only pay per request, no capacity provisioning needed. For a lock table that is rarely written to, this is the cheapest option

---

### Using Terraform (Bootstrap Config)

```hcl
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }

  lifecycle {
    prevent_destroy = true
  }

  tags = {
    Name      = "Terraform State Lock Table"
    ManagedBy = "Terraform"
  }
}
```

---

## Configuring Locking in the Backend

Add the `dynamodb_table` argument to your S3 backend configuration:

```hcl
terraform {
  backend "s3" {
    bucket         = "mycompany-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"   # enables locking
    encrypt        = true
  }
}
```

That is all the configuration needed. Once `dynamodb_table` is set, Terraform automatically uses DynamoDB for all locking operations.

---

## What Locking Looks Like in Practice

### When No Lock Exists

Normal operation — Terraform acquires the lock immediately and proceeds:

```bash
$ terraform apply

Acquiring state lock. This may take a few moments...

Terraform will perform the following actions:
  ...

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Releasing state lock. This may take a few moments...
```

---

### When a Lock Exists

If another process holds the lock, Terraform shows a detailed error:

```bash
$ terraform apply

Acquiring state lock. This may take a few moments...

╷
│ Error: Error acquiring the state lock
│
│ Error message: ConditionalCheckFailedException: The conditional request failed
│ Lock Info:
│   ID:        abc123def456
│   Path:      mycompany-terraform-state/production/terraform.tfstate
│   Operation: OperationTypeApply
│   Who:       alice@alices-laptop
│   Version:   1.6.6
│   Created:   2024-01-15 10:30:00.123456789 +0000 UTC
│   Info:
│
│
│ Terraform acquires a state lock to protect the state from being written
│ by multiple users at the same time. Please resolve the issue above and try
│ again. For most commands, you can disable locking with the "-lock=false"
│ flag, but this is not recommended.
╵
```

The error tells you:
- Who holds the lock (`alice@alices-laptop`)
- What operation they are running (`OperationTypeApply`)
- When the lock was created (`2024-01-15 10:30:00`)
- The lock ID (`abc123def456`)

This information helps you contact the right person and understand if the lock is stale.

---

## Which Operations Acquire Locks

Not all Terraform operations acquire a lock:

| Command | Acquires Lock? |
|---|---|
| `terraform apply` | Yes |
| `terraform destroy` | Yes |
| `terraform plan` | Yes (read lock) |
| `terraform apply -refresh-only` | Yes |
| `terraform init` | No |
| `terraform validate` | No |
| `terraform fmt` | No |
| `terraform output` | No |
| `terraform state list` | No |
| `terraform state show` | No |

`terraform plan` acquires a lock because it refreshes state by calling the cloud API — though it is a lighter lock than apply.

---

## Handling Stale Locks

A lock becomes **stale** when the process that created it dies or crashes without releasing it. For example:

- Your laptop loses power during a `terraform apply`
- A CI/CD pipeline job is killed mid-apply
- Terraform crashes due to a bug
- Network connection is lost during a remote operation

When a stale lock exists, no one can run Terraform until it is released.

### Identifying a Stale Lock

Look at the `Created` timestamp in the lock error message. If the lock was created hours or days ago and no one is actively running Terraform, it is likely stale.

You can also check the DynamoDB table directly:

```bash
# See all current locks in the table
aws dynamodb scan \
  --table-name terraform-state-lock \
  --region us-east-1
```

---

### Force Releasing a Stale Lock

Use `terraform force-unlock` to release a stale lock:

```bash
# Use the Lock ID from the error message
terraform force-unlock abc123def456
```

Terraform will confirm:

```
Do you really want to force-unlock?
  Terraform will remove the lock on the remote state.
  This will allow local Terraform commands to modify this state, even though it
  may be still be in use. Only 'yes' will be accepted to confirm.

  Enter a value: yes

Terraform state has been successfully unlocked!
```

**Important warnings about force-unlock:**

- Only use it when you are absolutely certain no other Terraform process is running
- If you force-unlock a lock that is still being held by an active apply, you could corrupt the state
- Always confirm with your team before force-unlocking
- The Lock ID in the command must exactly match the ID shown in the error message

---

### Manual Lock Removal from DynamoDB (Last Resort)

If `terraform force-unlock` does not work for some reason, you can manually delete the lock from DynamoDB:

```bash
aws dynamodb delete-item \
  --table-name terraform-state-lock \
  --key '{"LockID": {"S": "mycompany-terraform-state/production/terraform.tfstate"}}' \
  --region us-east-1
```

Use this only as a last resort. Prefer `terraform force-unlock`.

---

## Disabling Locking (Not Recommended)

You can temporarily disable locking with the `-lock=false` flag:

```bash
terraform plan -lock=false
terraform apply -lock=false
```

This skips both acquiring and releasing the lock. Use this only when:
- You are debugging a locking issue
- You are doing a read-only operation and absolutely certain no writes are happening
- You are working in a development environment alone

**Never use `-lock=false` in production environments or when working with a team.**

---

## Lock Timeout

By default, Terraform waits indefinitely to acquire a lock if another process holds it. You can set a timeout with `-lock-timeout`:

```bash
# Wait up to 5 minutes for the lock, then fail
terraform apply -lock-timeout=5m

# Wait up to 30 seconds
terraform apply -lock-timeout=30s
```

This is useful in CI/CD pipelines where you want the job to fail rather than wait indefinitely if a lock cannot be acquired:

```yaml
# GitHub Actions — fail after 5 minutes if lock cannot be acquired
- name: Terraform Apply
  run: terraform apply -auto-approve -lock-timeout=5m
```

---

## IAM Permissions for DynamoDB Locking

The IAM user or role running Terraform needs permissions for the DynamoDB table:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/terraform-state-lock"
    }
  ]
}
```

- `GetItem` — check if a lock exists before acquiring
- `PutItem` — create a lock entry
- `DeleteItem` — release a lock

---

## Multiple Projects, One Lock Table

You can use a single DynamoDB lock table for all your Terraform projects. The `LockID` combines the S3 bucket name and the key path, so each project gets its own unique lock entry:

```
Lock entries in DynamoDB:
mycompany-terraform-state/networking/production/terraform.tfstate
mycompany-terraform-state/networking/staging/terraform.tfstate
mycompany-terraform-state/applications/myapp/production/terraform.tfstate
mycompany-terraform-state/applications/myapp/staging/terraform.tfstate
```

Different projects can run simultaneously since they write to different lock IDs. Two applies for the same project and environment cannot run simultaneously.

---

## Monitoring Lock Table Activity

For production environments, set up a CloudWatch alarm to notify you if a lock exists for too long (indicating a possible stale lock):

```hcl
resource "aws_cloudwatch_metric_alarm" "stale_lock" {
  alarm_name          = "terraform-stale-lock-alarm"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = "1"
  metric_name         = "ItemCount"
  namespace           = "AWS/DynamoDB"
  period              = "3600"  # 1 hour
  statistic           = "Maximum"
  threshold           = "1"
  alarm_description   = "A Terraform state lock has been held for more than 1 hour"

  dimensions = {
    TableName = aws_dynamodb_table.terraform_locks.name
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

---

## Summary

- DynamoDB provides state locking for the S3 backend — preventing concurrent Terraform operations from corrupting state
- Create a DynamoDB table with a single `LockID` (String) hash key and `PAY_PER_REQUEST` billing
- Enable locking by adding `dynamodb_table = "terraform-state-lock"` to your S3 backend config
- When a lock exists, Terraform shows who holds it, what operation they are running, and when it was created
- Stale locks occur when a Terraform process crashes without releasing the lock — use `terraform force-unlock <LOCK_ID>` to release them
- Only force-unlock when you are certain no active apply is running — unlocking a live process can corrupt state
- Use `-lock-timeout=5m` in CI/CD to fail fast if a lock cannot be acquired
- One DynamoDB table can serve all your Terraform projects — each project gets its own lock ID based on the S3 key path

This completes Chapter 16 on Remote Backends. You now know how to set up the S3 backend, configure it cleanly across environments, and protect concurrent access with DynamoDB locking.

In Chapter 17, we cover all the Terraform state commands — the tools you use to inspect, modify, and manage state directly.
