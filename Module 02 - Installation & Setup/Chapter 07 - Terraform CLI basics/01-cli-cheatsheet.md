# Terraform CLI Cheat Sheet

## Overview

This file is your quick reference for all the Terraform CLI commands you will use in day-to-day DevOps work. Keep this handy — you will come back to it often.

Commands are organized by category so you can quickly find what you need.

---

## Core Workflow Commands

These are the commands you use on almost every Terraform project, every day.

### Initialize

```bash
# Initialize a working directory (download providers, set up backend)
terraform init

# Initialize and upgrade providers to newest version within constraints
terraform init -upgrade

# Initialize and migrate state to a new backend
terraform init -migrate-state

# Initialize but reconfigure the backend (useful when changing backends)
terraform init -reconfigure

# Initialize without configuring the backend (validation only)
terraform init -backend=false
```

---

### Plan

```bash
# Preview all changes
terraform plan

# Save the plan to a file (apply it later)
terraform plan -out=myplan.tfplan

# Plan with specific variable values
terraform plan -var="environment=production"
terraform plan -var="region=us-west-2" -var="instance_type=t3.medium"

# Plan using a variable file
terraform plan -var-file="production.tfvars"

# Plan for only one specific resource
terraform plan -target=aws_instance.web_server

# Preview a destroy (see what would be deleted)
terraform plan -destroy

# Plan and return exit code 2 if changes exist (useful in scripts)
terraform plan -detailed-exitcode

# Plan without refreshing real state (faster but may miss drift)
terraform plan -refresh=false
```

---

### Apply

```bash
# Apply changes (shows plan first, asks for confirmation)
terraform apply

# Apply without confirmation prompt (for CI/CD)
terraform apply -auto-approve

# Apply a saved plan file
terraform apply myplan.tfplan

# Apply with specific variable values
terraform apply -var="environment=production"
terraform apply -var-file="production.tfvars"

# Apply only a specific resource
terraform apply -target=aws_instance.web_server

# Apply but only refresh state (no resource changes)
terraform apply -refresh-only

# Apply with more parallel operations (default is 10)
terraform apply -parallelism=20
```

---

### Destroy

```bash
# Destroy all managed resources (asks for confirmation)
terraform destroy

# Destroy without confirmation prompt (for CI/CD)
terraform destroy -auto-approve

# Destroy only a specific resource
terraform destroy -target=aws_instance.web_server

# Destroy multiple specific resources
terraform destroy -target=aws_instance.web -target=aws_security_group.web_sg

# Preview what destroy would delete (without actually deleting)
terraform plan -destroy
terraform plan -destroy -out=destroy.tfplan
```

---

## Code Quality Commands

```bash
# Format all .tf files in current directory
terraform fmt

# Format recursively (all subdirectories and modules)
terraform fmt -recursive

# Check formatting without changing files (exit 0 = OK, exit 3 = needs formatting)
terraform fmt -check

# Check formatting recursively
terraform fmt -check -recursive

# Show what would change (diff view, no file changes)
terraform fmt -diff

# Validate the configuration (syntax, types, references)
terraform validate
```

---

## State Commands

These commands let you inspect and manage the Terraform state file.

```bash
# List all resources in state
terraform state list

# List resources matching a pattern
terraform state list aws_instance.*

# Show full details of a specific resource in state
terraform state show aws_instance.web_server

# Move a resource to a different state address (renaming in code)
terraform state mv aws_instance.web aws_instance.app_server

# Move a resource to a different state file
terraform state mv -state-out=other.tfstate aws_instance.web aws_instance.web

# Remove a resource from state without destroying it
terraform state rm aws_instance.web_server

# Remove all resources matching a pattern
terraform state rm 'aws_instance.*'

# Pull the current state (shows raw JSON)
terraform state pull

# Push a local state file to the remote backend
terraform state push terraform.tfstate
```

---

## Output Commands

```bash
# Show all output values
terraform output

# Show a specific output value
terraform output bucket_name

# Show output as raw value (no quotes, useful in scripts)
terraform output -raw bucket_name

# Show all outputs as JSON
terraform output -json

# Show outputs from a state file without running plan/apply
terraform output -state=terraform.tfstate
```

---

## Import Commands

Import existing AWS resources into Terraform management without recreating them.

```bash
# Import an existing resource (Terraform 1.4 and earlier)
terraform import aws_instance.web i-0a1b2c3d4e5f67890

# Import with variable values
terraform import -var="environment=production" aws_s3_bucket.app my-bucket-name

# Generate configuration for an existing resource (Terraform 1.5+)
terraform plan -generate-config-out=generated.tf
```

---

## Workspace Commands

Workspaces let you manage multiple environments (dev, staging, prod) from one configuration.

```bash
# List all workspaces
terraform workspace list

# Show the current workspace
terraform workspace show

# Create a new workspace
terraform workspace new staging

# Switch to an existing workspace
terraform workspace select production

# Delete a workspace (must be empty)
terraform workspace delete staging
```

---

## Provider Commands

```bash
# List all required providers and their versions
terraform providers

# Show provider schemas (all resources and their arguments) as JSON
terraform providers schema -json

# Lock providers for specific platforms (important for cross-platform teams)
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64
```

---

## Graph Commands

Terraform can generate a visual dependency graph of your resources.

```bash
# Generate the dependency graph (outputs in DOT format)
terraform graph

# Generate graph and convert to PNG (requires graphviz)
terraform graph | dot -Tpng > graph.png

# Generate a destroy graph
terraform graph -type=plan-destroy
```

---

## Console Command

The Terraform console is an interactive shell for testing expressions and functions.

```bash
# Open the interactive console
terraform console
```

Inside the console:
```hcl
> 2 + 2
4

> "hello ${var.name}"
"hello world"

> length(["a", "b", "c"])
3

> upper("terraform")
"TERRAFORM"

> join(", ", ["us-east-1", "us-west-2"])
"us-east-1, us-west-2"

# Exit the console
> exit
```

The console is excellent for testing HCL expressions and functions before putting them in your real configuration.

---

## Get Command

```bash
# Download and update modules referenced in configuration
terraform get

# Download modules and update them to the latest version
terraform get -update
```

Note: `terraform init` also runs `terraform get` internally. You rarely need to run `terraform get` separately.

---

## Force Unlock Command

If a Terraform operation is interrupted, the state lock might not be released. This causes subsequent operations to fail with "state is already locked".

```bash
# Force unlock the state (get the lock ID from the error message)
terraform force-unlock LOCK_ID
```

**Use with caution.** Only force unlock if you are certain no other Terraform operation is actually running.

---

## Useful Flag Combinations

```bash
# Full CI/CD pipeline flow
terraform init
terraform fmt -check
terraform validate
terraform plan -out=plan.tfplan
terraform apply plan.tfplan

# Quick development check (format + validate before planning)
terraform fmt && terraform validate && terraform plan

# Apply to production with variable file and no prompt
terraform apply -var-file="prod.tfvars" -auto-approve

# Clean up dev environment completely
terraform destroy -auto-approve

# Debug: apply only one resource, see what happens
terraform apply -target=aws_instance.web_server

# Check if any changes exist (for use in scripts)
terraform plan -detailed-exitcode
if [ $? -eq 2 ]; then
  echo "Changes exist"
elif [ $? -eq 0 ]; then
  echo "No changes"
fi
```

---

## Environment Variables Reference

These environment variables are read by Terraform and the AWS provider. Set them to avoid passing flags repeatedly.

```bash
# AWS Authentication
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_SESSION_TOKEN="your-session-token"    # For temporary credentials
export AWS_DEFAULT_REGION="us-east-1"
export AWS_PROFILE="my-profile"                  # Use a named AWS CLI profile

# Terraform Variable Values (prefix: TF_VAR_)
# These set variable values - equivalent to -var="name=value"
export TF_VAR_environment="production"
export TF_VAR_instance_type="t3.medium"
export TF_VAR_region="us-east-1"

# Terraform Behavior
export TF_LOG="DEBUG"          # Enable debug logging (TRACE, DEBUG, INFO, WARN, ERROR)
export TF_LOG_PATH="tf.log"    # Write logs to a file
export TF_DATA_DIR=".terraform" # Directory for Terraform working files

# Terraform CLI Config
export TF_CLI_ARGS="-no-color"            # Disable color output (useful in CI/CD)
export TF_CLI_ARGS_plan="-var-file=prod.tfvars"  # Default args for specific command
export TF_CLI_ARGS_apply="-auto-approve"   # Auto-approve all applies (careful!)

# Terraform Cloud
export TF_TOKEN_app_terraform_io="your-tfc-token"
```

---

## Exit Codes Reference

Understanding exit codes is important for scripting and CI/CD.

| Command | Exit Code | Meaning |
|---|---|---|
| `terraform fmt -check` | 0 | All files are formatted correctly |
| `terraform fmt -check` | 3 | Files need formatting |
| `terraform validate` | 0 | Configuration is valid |
| `terraform validate` | 1 | Configuration has errors |
| `terraform plan` | 0 | Plan succeeded (changes may or may not exist) |
| `terraform plan` | 1 | Plan errored |
| `terraform plan -detailed-exitcode` | 0 | Plan succeeded, no changes |
| `terraform plan -detailed-exitcode` | 1 | Plan errored |
| `terraform plan -detailed-exitcode` | 2 | Plan succeeded, changes exist |
| `terraform apply` | 0 | Apply succeeded |
| `terraform apply` | 1 | Apply failed |

---

## Getting Help

```bash
# General help
terraform --help
terraform -help

# Help for a specific command
terraform plan --help
terraform apply --help
terraform state --help
terraform workspace --help

# Terraform version
terraform version

# Check for available updates
terraform version -check-tf-version
```

---

## Summary

This cheat sheet covers:
- Core workflow: `init`, `plan`, `apply`, `destroy`
- Code quality: `fmt`, `validate`
- State management: `state list`, `state show`, `state mv`, `state rm`
- Outputs: `output`
- Import: `import`
- Workspaces: `workspace list/new/select/delete`
- Debugging: `console`, `graph`, `TF_LOG`
- Key environment variables for AWS auth and Terraform behavior
- Exit codes for scripting and CI/CD

In the next chapter, we cover setting up VS Code and the Terraform extension to make your development experience much faster and more comfortable.
