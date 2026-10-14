# Provider Versioning

## Introduction

Provider versioning is one of those things that seems minor at first but becomes very important as you work on real-world Terraform projects. Getting versioning right prevents a whole class of frustrating bugs where your Terraform code suddenly breaks because a provider updated and changed something.

This file explains how provider versioning works, how to specify version constraints properly, how the lock file protects your team, and how to safely upgrade providers.

---

## Why Provider Versioning Matters

Imagine this scenario:

You write Terraform code that perfectly creates your infrastructure. Everything works. You commit it to Git and your CI/CD pipeline deploys it successfully.

Three months later, the AWS provider releases a new major version with some breaking changes. A team member runs `terraform init` on their laptop and gets the new provider version. Now when they run `terraform plan`, they see strange errors or unexpected changes — because the new provider version behaves differently.

This is the exact problem that provider versioning solves. By specifying which version of a provider to use, you ensure that:
- Everyone on your team uses the exact same provider version
- Your CI/CD pipeline uses the same version as your local machine
- Provider updates do not silently break your infrastructure code
- You control when to upgrade, not the provider release schedule

---

## Semantic Versioning (SemVer)

Terraform providers follow **Semantic Versioning** (SemVer), which uses the format: `MAJOR.MINOR.PATCH`

For example: `5.31.0`
- `5` — MAJOR version
- `31` — MINOR version
- `0` — PATCH version

**What each version change means:**

| Version type | What changed | Breaking changes? |
|---|---|---|
| PATCH (5.31.0 → 5.31.1) | Bug fixes | No |
| MINOR (5.31.0 → 5.32.0) | New features added | No |
| MAJOR (5.31.0 → 6.0.0) | Significant changes | Yes — possibly breaking |

Understanding this helps you choose the right version constraint for your situation.

---

## Version Constraint Syntax

Terraform uses a special syntax for specifying version constraints. Here are all the options:

### Exact version

```hcl
version = "5.31.0"
```

Only this exact version will be used. No updates, ever.

**Use when:** You need absolute reproducibility and are willing to manually update the version number when you want to upgrade.

---

### Greater than or equal

```hcl
version = ">= 5.0.0"
```

Any version that is 5.0.0 or newer. This includes 5.1.0, 5.31.0, 6.0.0, 7.0.0 — anything newer.

**Use when:** Rarely — too permissive. A new major version could break things.

---

### Less than

```hcl
version = "< 6.0.0"
```

Any version older than 6.0.0. Accepts 5.x, 4.x, etc.

**Use when:** Combine with `>=` to create a range.

---

### Range constraint

```hcl
version = ">= 5.0.0, < 6.0.0"
```

Any version in the 5.x range. Equivalent to `~> 5.0` (see below).

---

### Pessimistic constraint (`~>`) — Most Common

```hcl
version = "~> 5.0"
```

This is the most commonly used constraint. The `~>` operator allows updates to the rightmost version component only.

- `~> 5.0` — allows `5.x` (5.0, 5.1, 5.31) but NOT `6.0`
- `~> 5.31` — allows `5.31.x` (5.31.0, 5.31.1) but NOT `5.32.0`
- `~> 5.31.0` — allows only `5.31.x` patch updates but NOT `5.32.0`

**The rule:** everything to the left of the rightmost version component is locked. Only the rightmost component can increase.

```hcl
# Allows any 5.x version (most common for providers)
version = "~> 5.0"

# Allows only 5.31.x patch versions (more restrictive)
version = "~> 5.31.0"

# Allows only exactly 5.31.0 (same as = "5.31.0")
# Don't use ~> with three components unless you want patch-only updates
```

---

## Recommended Version Strategy

For most Terraform projects, use `~> MAJOR.MINOR`:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

This:
- Accepts any AWS provider version in the 5.x range
- Protects you from breaking changes in version 6.0
- Allows you to get bug fixes and new features within the 5.x range
- Is flexible enough that your team doesn't need to constantly update the constraint

---

## Terraform Version Constraint

You should also specify which version of Terraform itself is required. This goes in the `terraform` block:

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

This means anyone running this configuration must have Terraform 1.5.0 or newer. If they have an older version, Terraform will refuse to run and show an error message explaining why.

For stricter control:
```hcl
required_version = ">= 1.5.0, < 2.0.0"
```

---

## The Lock File — `.terraform.lock.hcl`

The lock file is one of Terraform's most important features for team collaboration.

When you run `terraform init`, Terraform:
1. Reads your version constraints from the `terraform` block
2. Finds the newest version that satisfies those constraints
3. Downloads that version
4. **Writes the exact version into `.terraform.lock.hcl`**

The lock file records the precise version selected, along with cryptographic hashes to verify the downloaded binary is legitimate.

### What the lock file looks like

```hcl
# This file is maintained automatically by "terraform init".
# Manual edits may be lost in future updates.

provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abcdefghijklmnopqrstuvwxyz1234567890ABCDEFG=",
    "zh:1234567890abcdefghijklmnopqrstuvwxyz1234567890abcdefghijklmnop",
    "zh:abcdef1234567890abcdef1234567890abcdef1234567890abcdef12345678",
  ]
}
```

Key fields:
- `version` — the exact version that was selected and downloaded
- `constraints` — the constraint from your configuration
- `hashes` — cryptographic fingerprints of the provider binary for each supported platform

### Why you must commit the lock file to Git

The lock file ensures that everyone uses the exact same provider version. Without it:

- Developer A runs `terraform init` in January — gets AWS provider 5.28.0
- Developer B runs `terraform init` in February — gets AWS provider 5.31.0
- Both have different behaviors, potentially causing confusion

With the lock file committed:
- Both developers get exactly 5.31.0 (whatever was recorded in the lock file)
- The CI/CD pipeline also gets exactly 5.31.0
- Consistency is guaranteed

**Add this to your `.gitignore` to commit the right things:**

```gitignore
# Ignore Terraform state files (sensitive)
*.tfstate
*.tfstate.backup
.terraform/

# Do NOT ignore the lock file — commit it to Git!
# .terraform.lock.hcl  ← do not add this line
```

---

## Upgrading Providers

### Checking current versions

To see what provider versions are currently locked:

```bash
cat .terraform.lock.hcl
```

Or use:
```bash
terraform version
```

Output shows Terraform version and all provider versions:
```
Terraform v1.6.6
on linux_amd64
+ provider registry.terraform.io/hashicorp/aws v5.31.0
```

### Upgrading to a newer version within constraints

If your constraint is `~> 5.0` and AWS provider 5.35.0 was just released, you can upgrade:

```bash
terraform init -upgrade
```

This:
1. Finds the newest version matching your constraint
2. Downloads it
3. Updates `.terraform.lock.hcl` with the new version

After upgrading, run `terraform plan` to verify nothing broke.

### Upgrading to a new major version

If AWS provider 6.0.0 is released and you want to upgrade from 5.x:

1. **Read the upgrade guide** — major versions always have an upgrade guide in the provider's changelog
2. **Update your version constraint** in `main.tf` or `versions.tf`:
   ```hcl
   version = "~> 6.0"  # Changed from ~> 5.0
   ```
3. **Run `terraform init -upgrade`** to download the new version
4. **Review the plan carefully** — major upgrades can change how existing resources are handled
5. **Test in a non-production environment first**

---

## Managing Multiple Providers

When your configuration uses multiple providers, each one gets its own entry in the lock file:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.0"
    }
  }
}
```

After `terraform init`, the lock file records the exact version for each:

```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [...]
}

provider "registry.terraform.io/hashicorp/kubernetes" {
  version     = "2.24.0"
  constraints = "~> 2.0"
  hashes = [...]
}

provider "registry.terraform.io/hashicorp/helm" {
  version     = "2.12.1"
  constraints = "~> 2.0"
  hashes = [...]
}
```

---

## Platform Hashes in the Lock File

The lock file stores hashes for multiple platforms (Linux, macOS, Windows). This is important for teams where some developers are on Mac and others are on Linux.

When a new developer on a different platform runs `terraform init`, Terraform verifies that the provider binary for their platform matches the expected hash. This:
- Ensures the provider was not tampered with
- Ensures everyone is using the same trusted binary

If you start getting errors about missing hashes for your platform, you may need to add them:

```bash
terraform providers lock \
  -platform=linux_amd64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64
```

This adds hashes for all common platforms to the lock file.

---

## A Well-Structured versions.tf File

Many teams put all their version requirements in a dedicated `versions.tf` file to keep things organized:

```hcl
# versions.tf

terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

And the provider configuration stays in `providers.tf`:

```hcl
# providers.tf

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
    }
  }
}
```

Separating these concerns makes the code cleaner and easier to maintain.

---

## Summary

- Provider versioning prevents breaking changes from silently affecting your infrastructure
- Providers follow **Semantic Versioning** — MAJOR.MINOR.PATCH
- The `~> MAJOR.MINOR` constraint (e.g., `~> 5.0`) is the most common and recommended approach
- Always specify a `required_version` for Terraform itself as well
- The **lock file** (`.terraform.lock.hcl`) records exact versions — always commit it to Git
- Run `terraform init -upgrade` to upgrade providers within your stated constraints
- Upgrading to a new MAJOR version requires reading the upgrade guide and careful testing
- Separate your version requirements into a `versions.tf` file for cleaner organization

In the next chapter, we dive into the Terraform CLI — all the commands you will use every day and how to use them effectively.
