# Practical Implementation: Creating Environment-Specific `.tfvars`

The successful management of Development, Staging, and Production environments depends on the ability to provide distinct, validated configurations to the same base set of HCL code. By using environment-specific `.tfvars` files, you can scale resources, enable high-availability features, and toggle security controls without modifying the core logic. This chapter provides a technical walk-through of creating and managing these files for professional-grade deployments.

## 1. The HCL Foundation: `variables.tf`

Before creating `.tfvars` files, your `variables.tf` must define the variables with appropriate types and defaults.

```hcl
variable "environment" {
  type        = string
  description = "The deployment environment (e.g., dev, staging, prod)"
}

variable "instance_cfg" {
  type = object({
    type  = string
    count = number
  })
}

variable "enable_ha" {
  type    = bool
  default = false
}
```

## 2. Sample 1: The Development File (`dev.tfvars`)

In Dev, the priority is cost savings and developer iteration speed.

```hcl
# dev.tfvars
environment = "development"

instance_cfg = {
  type  = "t3.micro"
  count = 1
}

# Keep costs low by disabling multi-AZ and expensive monitoring
enable_ha = false

tags = {
  Owner       = "dev-team"
  CostCenter  = "research-101"
}
```

## 3. Sample 2: The Production File (`prod.tfvars`)

In Prod, the priority is absolute availability, performance, and compliance.

```hcl
# prod.tfvars
environment = "production"

instance_cfg = {
  type  = "m5.large"
  count = 6
}

# Enable redundancy and advanced monitoring
enable_ha = true

tags = {
  Owner       = "platform-ops"
  CostCenter  = "revenue-service"
  Compliance  = "SOC2-PCI"
}
```

## 4. Toggling Features with Booleans

Use boolean variables in your `.tfvars` to control complex infrastructure logic in your `main.tf`.

```hcl
# main.tf
resource "aws_db_instance" "main" {
  multi_az = var.enable_ha
  # ...
}
```
This pattern allows you to maintain one HCL file that "adapts" to its environment. Passing `enable_ha = true` in `prod.tfvars` automatically deploys a multi-AZ RDS instance, while `false` in `dev.tfvars` saves significant costs.

## 5. Propagation to Child Modules

Value flow follow a strict hierarchy: `Environment Files -> Root Module -> Child Modules`.

**Root Module (`main.tf`):**
```hcl
module "compute" {
  source = "./modules/ec2-cluster"
  
  # Forwarding values from the tfvars file
  cluster_size = var.instance_cfg.count
  vm_type      = var.instance_cfg.type
}
```
This "Pass-Through" pattern ensures that your child modules remain generic and agnostic of the environment, while the `.tfvars` files provide the "DNA" that defines the environment's character.

## 6. Automating the `-var-file` Injection

In a professional CI/CD pipeline, the variable file name is usually derived from the branch name or a pipeline variable.

**Jenkins / Bash Example:**
```bash
ENVIRONMENT=${GIT_BRANCH}
terraform plan -var-file="vars/${ENVIRONMENT}.tfvars"
```

## 7. The "Developer Override" Pattern

Sometimes a developer needs to override a single variable locally for testing without touching the shared `dev.tfvars`.
1. Create a file named `terraform.tfvars`. (This is automatically loaded and takes priority over defaults).
2. Create a file named `personal.auto.tfvars`. (This takes priority over `terraform.tfvars`).

**Caution:** Always ensure these files are in your `.gitignore` to prevent local testing values from being committed to the shared repository.

## 8. Validating Environment Consistency

A common failure point is a variable defined in `prod.tfvars` that hasn't been added to `dev.tfvars`.
- **Pre-apply Verification:** Run `terraform plan` for **all** environments in your CI pipeline whenever a change is made to the `variables.tf` file. This ensures that a missing variable in one environment doesn't break a future deployment.

## 9. Summary

Environment-specific variable files are the "Configuration Panel" for your infrastructure. By separating architectural logic from environmental parameters, you allow your codebase to be truly reusable across any number of deployment targets. Mastering the use of objects, booleans, and disciplined CI/CD injection ensures that your infrastructure is cost-effective in development and bulletproof in production.
