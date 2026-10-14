# terraform state pull and push

## Introduction

`terraform state pull` and `terraform state push` are commands that let you manually download and upload the Terraform state file. They give you direct access to the raw state data — useful for backups, migrations, advanced debugging, and manual state recovery.

While most state operations happen automatically through normal Terraform commands, pull and push are your escape hatch for situations that require direct state file manipulation.

---

## terraform state pull

`terraform state pull` downloads the current state from whichever backend is configured — local file, S3, Terraform Cloud, etc. — and prints it to stdout as JSON.

```bash
terraform state pull
```

The output is the complete state file in JSON format, printed to your terminal.

---

### Saving State to a File

The most common use is redirecting the output to a file:

```bash
# Save current state to a local file
terraform state pull > current-state.tfstate

# Save with a timestamp for versioned backups
terraform state pull > "state-backup-$(date +%Y%m%d-%H%M%S).tfstate"
```

This is valuable before any risky operation — always pull state before making manual changes.

---

### Inspecting State with jq

You can pipe the output to `jq` for analysis:

```bash
# See all resource addresses in state
terraform state pull | jq -r '.resources[].type + "." + .resources[].name'

# Count total resources
terraform state pull | jq '.resources | length'

# Find all EC2 instances and their IDs
terraform state pull | jq -r '
  .resources[]
  | select(.type == "aws_instance")
  | .name + ": " + .instances[0].attributes.id
'

# Get all output values
terraform state pull | jq '.outputs'

# Check the serial number (how many times state has been written)
terraform state pull | jq '.serial'

# Get the Terraform version that last wrote the state
terraform state pull | jq '.terraform_version'
```

---

### Use Cases for terraform state pull

**1. Creating a backup before risky operations**

```bash
# Always back up state before running state mv, state rm, or import
terraform state pull > "backup-before-refactor-$(date +%Y%m%d).tfstate"
terraform state mv aws_instance.web aws_instance.web_server
```

**2. Copying state between backends**

When migrating from one backend to another manually:

```bash
# Pull from old backend
terraform state pull > migrated-state.tfstate

# Switch to new backend configuration
# Edit versions.tf to point to new backend

# Push to new backend
terraform state push migrated-state.tfstate
```

**3. Inspecting state without Terraform installed**

Since `state pull` outputs raw JSON, you can inspect it with any JSON tool even on a machine without Terraform:

```bash
# Share state snapshot for debugging without sharing AWS access
terraform state pull > state-for-debugging.json
# Send to colleague who can inspect without needing AWS credentials
```

**4. Comparing states between environments**

```bash
# Pull both states and compare
terraform state pull > prod-state.json

# Switch to staging workspace or directory
cd ../staging
terraform state pull > staging-state.json

# Compare
diff <(jq -r '.resources[].type + "." + .resources[].name' prod-state.json | sort) \
     <(jq -r '.resources[].type + "." + .resources[].name' staging-state.json | sort)
```

---

## terraform state push

`terraform state push` uploads a local state file to the configured backend, overwriting whatever is currently stored there.

```bash
terraform state push <path-to-state-file>
```

```bash
# Push a local state file to the remote backend
terraform state push terraform.tfstate

# Push a backup state file
terraform state push state-backup-20240115.tfstate
```

---

### When to Use terraform state push

**1. Restoring from a backup**

If your current state is corrupted or incorrect, restore a known-good backup:

```bash
# Check the backup
cat state-backup-20240115.tfstate | jq '.serial'

# Verify it has the resources you expect
cat state-backup-20240115.tfstate | jq '.resources | length'

# Restore the backup
terraform state push state-backup-20240115.tfstate
```

**2. Completing a manual migration**

When moving state between backends and `terraform init -migrate-state` is not suitable:

```bash
# Pull from the source backend
cd old-project/
terraform state pull > state-to-migrate.tfstate

# Go to the new project/backend
cd ../new-project/
terraform init  # configure the new backend

# Push to the new backend
terraform state push ../old-project/state-to-migrate.tfstate
```

**3. Recovering a corrupted remote state**

If a remote state file becomes corrupted (rare but possible), push a known-good version:

```bash
terraform state push state-backup-before-corruption.tfstate
```

---

### The Serial Check — Safety Mechanism

`terraform state push` has an important safety check: it will refuse to push a state file if the serial number is less than or equal to the current state's serial.

This prevents you from accidentally overwriting a newer state with an older one.

```bash
terraform state push old-state.tfstate
```

If the push is rejected:
```
Failed to write state: cannot import state with serial 3 over newer state with serial 8
```

This means the state file you are pushing (serial 3) is older than the current remote state (serial 8). The push is blocked to protect you from losing newer changes.

**To force push** (use with extreme caution):

```bash
terraform state push -force old-state.tfstate
```

The `-force` flag bypasses the serial check. Use this only when you are certain you want to overwrite newer state — for example, when deliberately rolling back to a previous state during incident recovery.

---

### Incrementing the Serial Before Pushing

When recovering state, you sometimes need to manually increment the serial number so the push succeeds without using `-force`. Edit the state file:

```bash
# Edit the state file
cat state-backup.tfstate | jq '.serial += 10' > state-to-push.tfstate

# Now push succeeds because the serial is higher
terraform state push state-to-push.tfstate
```

---

## A Complete Backup and Recovery Workflow

Here is a complete workflow for backing up state and recovering from an accident:

### Regular Backup (Run Before Risky Operations)

```bash
#!/bin/bash
# backup-state.sh

TIMESTAMP=$(date +%Y%m%d-%H%M%S)
BACKUP_FILE="state-backup-${TIMESTAMP}.tfstate"

echo "Creating state backup: $BACKUP_FILE"
terraform state pull > "$BACKUP_FILE"

# Verify backup is valid JSON
if jq empty "$BACKUP_FILE" 2>/dev/null; then
  echo "Backup created successfully"
  echo "Resources in backup: $(jq '.resources | length' $BACKUP_FILE)"
else
  echo "ERROR: Backup file is not valid JSON"
  exit 1
fi
```

---

### Recovery After Accident

```bash
# 1. Identify the most recent good backup
ls -la state-backup-*.tfstate

# 2. Inspect the backup to confirm it has the right resources
cat state-backup-20240115-103045.tfstate | jq '.resources | length'
cat state-backup-20240115-103045.tfstate | jq -r '.resources[].type + "." + .resources[].name'

# 3. Compare with current state
terraform state pull | jq '.serial'
cat state-backup-20240115-103045.tfstate | jq '.serial'

# 4. If backup is older (lower serial), increment before pushing
cat state-backup-20240115-103045.tfstate | jq '.serial += 100' > recovery-state.tfstate

# 5. Push the recovery state
terraform state push recovery-state.tfstate

# 6. Verify the recovery
terraform state list
terraform plan  # should show minimal or no changes if recovery was correct
```

---

## Using Pull and Push for State Manipulation

Sometimes you need to make changes to state that are not supported by normal Terraform commands. The pull-edit-push workflow lets you do this:

```bash
# 1. Pull current state
terraform state pull > state.json

# 2. Make your edits (use jq or a text editor — very carefully)
# Example: change a tag value that Terraform cannot change in-place
cat state.json | jq '
  .resources[] |= if .type == "aws_s3_bucket" and .name == "app"
  then .instances[0].attributes.tags.Environment = "production"
  else .
  end
' > modified-state.json

# 3. Increment the serial
cat modified-state.json | jq '.serial += 1' > final-state.json

# 4. Push the modified state
terraform state push final-state.json
```

**Warning:** Directly editing state JSON is risky and should only be done when absolutely necessary. Incorrect edits can corrupt state. Always back up before editing and verify with `terraform plan` after pushing.

---

## Automating State Backups in CI/CD

For production environments, automate state backups as part of your CI/CD pipeline:

```yaml
# GitHub Actions — backup state before every apply
- name: Backup Terraform State
  run: |
    terraform state pull > state-backup-${{ github.run_id }}.tfstate
    
- name: Upload State Backup as Artifact
  uses: actions/upload-artifact@v3
  with:
    name: terraform-state-backup
    path: state-backup-*.tfstate
    retention-days: 30

- name: Terraform Apply
  run: terraform apply -auto-approve
```

This gives you a backup of state before every apply, stored as a CI/CD artifact for 30 days.

---

## Summary

- `terraform state pull` downloads the current state from any backend and outputs it as JSON
- `terraform state push` uploads a local state file to the configured backend
- Use `state pull` for: creating backups, inspecting state with `jq`, manual migrations, and debugging
- Use `state push` for: restoring from backups, completing manual migrations, and recovering from corrupted state
- The serial number is a safety mechanism — `state push` fails if your file's serial is not newer than the current remote state
- Use `terraform state push -force` only when deliberately overwriting with an older state (incident recovery)
- Always increment the serial before pushing a modified state file
- Automate state backups in CI/CD pipelines for production safety

This completes Chapter 17 on State Commands. You now have a full toolkit for inspecting, moving, removing, and manually managing Terraform state.

In Chapter 18, we cover state best practices — patterns and guidelines for keeping your state organized, secure, and maintainable as your infrastructure grows.
