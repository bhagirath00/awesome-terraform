# Local State

## Introduction

When you run `terraform apply` for the first time in a new project, Terraform automatically creates a file called `terraform.tfstate` in your current working directory. This is **local state** — the simplest and most straightforward way Terraform stores its state information.

Local state is the default. No configuration is needed. No setup is required. It just works out of the box. This makes it perfect for getting started with Terraform, learning, and working on solo projects.

This file explains exactly how local state works, what its directory structure looks like, how to work with it day to day, and — most importantly — where its limitations lie so you know when to move away from it.

---

## How Local State Works

When you initialize a Terraform project and run your first apply:

```bash
mkdir my-project && cd my-project
# Create your .tf files...
terraform init
terraform apply
```

Terraform creates the state file automatically in the same directory:

```
my-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── terraform.tfvars
├── .terraform/                    ← created by terraform init
│   ├── providers/
│   └── ...
├── .terraform.lock.hcl            ← provider lock file
├── terraform.tfstate              ← created after first apply
└── terraform.tfstate.backup       ← created after second apply onward
```

The state file lives right next to your configuration files. Every subsequent `terraform plan` reads from this file. Every `terraform apply` updates it.

---

## The Local Backend Configuration

When you use local state, you are using Terraform's **local backend**. This is the default backend — you don't need to declare it explicitly. But if you want to, you can declare it like this:

```hcl
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
```

The `path` argument tells Terraform where to store the state file. By default it is `terraform.tfstate` in the current directory. You can change this to any path:

```hcl
terraform {
  backend "local" {
    path = "/home/myuser/states/myproject.tfstate"
  }
}
```

In practice, you almost never need to declare the local backend explicitly — just leaving it out uses the default local state behavior.

---

## Working with Local State Day to Day

### First Apply — State is Created

```bash
terraform apply
```

After the first successful apply, `terraform.tfstate` is created containing all the resources Terraform just created. The file is created atomically — either it is written completely or not at all. Terraform never leaves a partially written state file.

---

### Subsequent Plans — State is Read

```bash
terraform plan
```

Terraform reads `terraform.tfstate` to know what currently exists, then compares with your configuration. The plan shows only what needs to change.

---

### The Backup File

Every time Terraform writes a new state, it first copies the current state to `terraform.tfstate.backup`:

```bash
# Before apply:
terraform.tfstate      ← contains serial 5

# After apply:
terraform.tfstate      ← contains serial 6 (new state)
terraform.tfstate.backup ← contains serial 5 (previous state)
```

Only one backup is kept. Each new apply overwrites the backup with whatever was in the state file before that apply.

If you accidentally run a destructive apply, you can try to recover using the backup:

```bash
# Restore previous state (use carefully)
cp terraform.tfstate.backup terraform.tfstate
terraform plan  # verify the restored state looks correct
```

---

### Viewing Local State

```bash
# List all resources in state
terraform state list

# Show details of a specific resource
terraform state show aws_instance.web_server

# Show everything in a readable format
terraform show

# Show everything as JSON (for scripts)
terraform show -json
```

---

## What Local State is Good For

### Learning and Experimentation

Local state is perfect when you are learning Terraform. You can:
- Create resources, apply, inspect the state file, destroy, and repeat
- Understand how state works without any extra setup
- Focus on learning HCL and Terraform concepts without infrastructure overhead

```bash
# Great learning workflow
terraform init
terraform apply    # create resources, inspect terraform.tfstate
terraform destroy  # clean up, state goes back to empty
```

---

### Personal Projects

If you are the only person working on a project — a personal website, a home lab, a side project — local state is completely fine:

- No teammates means no concurrency issues
- You control when changes are made
- Simplicity is an advantage, not a limitation

---

### Quick Prototyping

When you are trying out a new module, testing a new resource type, or building a proof of concept:

```bash
mkdir /tmp/tf-experiment && cd /tmp/tf-experiment
# Write some experimental .tf files
terraform init && terraform apply
# Test things out
terraform destroy
rm -rf /tmp/tf-experiment
```

Local state with a temporary directory is the fastest way to experiment.

---

### CI/CD with Ephemeral Environments

Some CI/CD patterns create entirely fresh infrastructure for each pipeline run and destroy it at the end. In this case, local state within the pipeline run is acceptable because:

- State only needs to persist for the duration of the pipeline
- Each pipeline run is independent
- No sharing of state between runs is needed

```yaml
# Example: GitHub Actions with ephemeral environment
- name: Terraform Init
  run: terraform init

- name: Terraform Apply
  run: terraform apply -auto-approve

- name: Run Tests
  run: ./run-tests.sh

- name: Terraform Destroy
  run: terraform destroy -auto-approve
  if: always()  # always destroy even if tests fail
```

---

## Local State File Security

Even with local state, you need to be careful about security.

### Never Commit to Git

Add state files to `.gitignore` immediately when starting a project:

```gitignore
# .gitignore
*.tfstate
*.tfstate.backup
.terraform/
crash.log
```

State files can contain sensitive data — database passwords, API keys, private keys — stored in plain text. Committing state to Git exposes this data to anyone with repository access.

---

### File Permissions

The state file is created with default file permissions. On Linux/macOS, you might want to restrict access:

```bash
# Restrict state file to owner only
chmod 600 terraform.tfstate
chmod 600 terraform.tfstate.backup
```

---

### Encryption at Rest

Local state files are not encrypted by default. If your project handles sensitive data:
- Consider using remote state with encryption enabled (S3 with SSE)
- Or encrypt the state file manually if you must keep it local

---

## The Limitations of Local State

Local state works well within its intended use cases, but it has real limitations that become serious problems in team environments or production infrastructure.

### Limitation 1: No Sharing Between Team Members

The state file lives on one person's machine. If your colleague needs to run Terraform:

- They don't have your state file
- They would start with an empty state
- Terraform would try to create all resources again
- You would end up with duplicate infrastructure

There is no built-in way to share local state between team members.

**Workaround people try (and why they fail):**

Some teams try committing the state file to Git. This creates serious problems:
- Git merge conflicts on the state file are complex and dangerous
- The state file contains sensitive data (exposed to all repo access)
- Multiple people can still run apply simultaneously and overwrite each other's state

---

### Limitation 2: No State Locking

When two people run `terraform apply` at the same time against the same infrastructure, both are reading and writing the same state file. There is no mechanism to prevent this with local state.

What happens:
1. Person A reads state (serial 5)
2. Person B reads state (serial 5)
3. Person A applies, writes state (serial 6)
4. Person B applies, writes state (serial 6) — overwrites Person A's changes
5. State is now inconsistent — one person's changes are lost

With remote state (S3 + DynamoDB), a lock prevents this: only one apply can run at a time.

---

### Limitation 3: Risk of State Loss

If your laptop:
- Is stolen or breaks down
- Has a disk failure
- Gets accidentally wiped

Your state file is gone. Without state, Terraform has no record of what infrastructure exists. Recovering means manually importing every resource back into a new state file — a painful, time-consuming process.

Remote state stored in S3 has built-in durability (99.999999999%) and versioning, protecting you from this scenario.

---

### Limitation 4: No State History

The local backup only keeps one previous version. If you need to go back further — to see what state looked like a week ago, or to recover from a mistake made several applies ago — you cannot do that with local state.

Remote state in S3 with versioning enabled keeps every version of the state file. You can retrieve any previous version.

---

### Limitation 5: Not Suitable for Production

For production infrastructure, local state is simply not appropriate:
- Production infrastructure needs to be managed by the whole team
- Production state must be backed up and durable
- Production changes need locking to prevent concurrent modifications
- Production state must be auditable (who changed what and when)

---

## When to Move from Local to Remote State

Move to remote state when any of these are true:

- More than one person needs to run Terraform against the same infrastructure
- You are managing production or staging environments
- You need state history beyond one backup
- You need state locking to prevent concurrent applies
- You want automatic backups and durability guarantees

The good news: migrating from local to remote state is straightforward. Terraform handles the migration automatically when you configure a new backend and run `terraform init -migrate-state`.

---

## Summary

- Local state stores `terraform.tfstate` in your current working directory — the default with no setup required
- Terraform automatically creates a backup at `terraform.tfstate.backup` before every apply
- Local state is ideal for learning, personal projects, quick prototyping, and ephemeral CI/CD environments
- Never commit state files to Git — they contain sensitive data
- Local state has serious limitations for teams: no sharing, no locking, risk of loss, no history
- For any team environment or production infrastructure, move to remote state
- Migration from local to remote state is handled automatically by Terraform with `terraform init -migrate-state`

In the next file, we cover why remote state is essential for teams — and what specific problems it solves that local state cannot.
