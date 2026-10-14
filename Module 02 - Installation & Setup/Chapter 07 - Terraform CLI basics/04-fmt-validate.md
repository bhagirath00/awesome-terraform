# Terraform fmt and validate

## Introduction

Beyond the core workflow commands (init, plan, apply, destroy), Terraform provides two commands that help you write correct, clean, and consistent code: `terraform fmt` and `terraform validate`.

These might seem like small utility commands, but in real-world team environments they are essential. Most professional Terraform projects enforce both formatting and validation as part of their CI/CD pipeline — no code gets merged unless it passes both checks.

---

## `terraform fmt` — Code Formatter

### What it does

`terraform fmt` automatically formats your `.tf` and `.tfvars` files to follow the standard Terraform style. It fixes:

- Indentation (uses 2 spaces, not tabs)
- Alignment of `=` signs in argument blocks
- Line spacing within blocks
- Trailing whitespace

`terraform fmt` does NOT change what your code does — it only changes how it looks. It is a purely cosmetic operation.

---

### Why Consistent Formatting Matters

Imagine your team has 5 engineers, each with their own formatting preferences. One uses tabs, another uses 4 spaces, a third uses 2 spaces. Every time someone edits a file, Git shows hundreds of "changes" that are just whitespace differences. Code reviews become noise because you cannot see the actual logic changes.

`terraform fmt` solves this by enforcing one standard style for everyone. When everyone runs `terraform fmt`, all files look the same — regardless of each person's editor settings.

---

### Before and After Formatting

**Before formatting (inconsistent, hard to read):**

```hcl
resource "aws_instance" "web" {
ami = "ami-0c55b159cbfafe1f0"
  instance_type="t2.micro"
    tags={
    Name="web-server"
  Environment  =   "production"
    }
}
```

**After running `terraform fmt`:**

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}
```

Notice:
- Consistent 2-space indentation
- `=` signs are aligned within each block
- Proper spacing around `=`
- Tags block is properly formatted

---

### Running terraform fmt

**Format all files in the current directory:**
```bash
terraform fmt
```

This modifies the files in place. If any files were changed, it prints the names of the files it modified:

```
main.tf
variables.tf
```

If all files are already correctly formatted, it prints nothing.

**Format files recursively (including subdirectories and modules):**
```bash
terraform fmt -recursive
```

Very useful when your project has multiple modules.

**Check formatting without making changes:**
```bash
terraform fmt -check
```

Returns exit code `0` if files are formatted, exit code `3` if they need formatting. Does NOT modify any files.

This is the flag used in CI/CD pipelines:

```bash
terraform fmt -check
if [ $? -ne 0 ]; then
  echo "Code is not formatted. Please run: terraform fmt"
  exit 1
fi
```

**See what changes would be made without applying them:**
```bash
terraform fmt -diff
```

Shows a diff of what would change, similar to `git diff`:

```diff
--- old/main.tf
+++ new/main.tf
@@ -1,7 +1,7 @@
 resource "aws_instance" "web" {
-ami = "ami-0c55b159cbfafe1f0"
-  instance_type="t2.micro"
+  ami           = "ami-0c55b159cbfafe1f0"
+  instance_type = "t2.micro"
   tags = {
```

**Combine diff with check to see what's wrong without changing it:**
```bash
terraform fmt -check -diff
```

---

### Setting Up Auto-Format in Your Editor

Rather than remembering to run `terraform fmt` manually, set up your editor to format automatically on save.

**VS Code:**
Install the HashiCorp Terraform extension (covered in Chapter 8), then add to VS Code settings:
```json
{
  "[terraform]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "hashicorp.terraform"
  }
}
```

**Vim/Neovim:**
Add to your config:
```vim
autocmd BufWritePre *.tf execute ':!terraform fmt %'
```

**JetBrains IDEs (IntelliJ, GoLand, etc.):**
The HCL plugin supports auto-formatting on save through the "Reformat Code" action.

---

### Terraform fmt in CI/CD

Add a formatting check to your CI/CD pipeline so that unformatted code cannot be merged:

**GitHub Actions:**
```yaml
- name: Check Terraform formatting
  run: terraform fmt -check -recursive
```

**GitLab CI:**
```yaml
terraform-fmt:
  script:
    - terraform fmt -check -recursive
```

If a developer's PR includes unformatted code, the pipeline fails and they see a clear message to run `terraform fmt`.

---

## `terraform validate` — Configuration Validator

### What it does

`terraform validate` checks your Terraform configuration for:

- **Syntax errors** — missing brackets, incorrect block types, typos in keywords
- **Invalid arguments** — arguments that don't exist for a resource type
- **Type errors** — passing a string where a number is expected
- **Missing required arguments** — forgetting to set a required field
- **Invalid references** — referencing a resource or variable that doesn't exist

`terraform validate` does NOT check:
- Whether your credentials are valid
- Whether a resource with that name already exists in AWS
- Whether the specified AMI ID actually exists
- Whether you have permission to create the resource

It only checks the code itself — not the real world. This means it is very fast (no API calls) and requires no authentication.

---

### Running terraform validate

```bash
terraform validate
```

**Success output:**
```
Success! The configuration is valid.
```

**Error output examples:**

```
╷
│ Error: Missing required argument
│
│   on main.tf line 3, in resource "aws_instance" "web":
│    3: resource "aws_instance" "web" {
│
│ The argument "ami" is required, but no definition was found.
╵
```

```
╷
│ Error: Reference to undeclared resource
│
│   on main.tf line 12, in resource "aws_subnet" "public":
│   12:   vpc_id = aws_vpc.nonexistent.id
│
│ A managed resource "aws_vpc" "nonexistent" has not been declared in the
│ root module.
╵
```

```
╷
│ Error: Invalid expression
│
│   on main.tf line 5, in resource "aws_instance" "web":
│    5:   instance_type = t2.micro
│
│ An argument definition must end with a newline.
╵
```

---

### Common Errors Caught by Validate

**1. Typo in argument name:**
```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instnce_type  = "t2.micro"  # Typo: "instnce" instead of "instance"
}
```

```
Error: Unsupported argument
  An argument named "instnce_type" is not expected here.
  Did you mean "instance_type"?
```

Notice that Terraform even suggests the correct spelling.

**2. Wrong data type:**
```hcl
variable "port" {
  type = number
}

resource "aws_security_group_rule" "http" {
  from_port = var.port
  to_port   = var.port
  protocol  = 80  # Wrong: should be a string like "tcp"
}
```

**3. Reference to a non-existent resource:**
```hcl
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id  # Error if aws_vpc.main doesn't exist in this config
}
```

**4. Missing closing bracket:**
```hcl
resource "aws_instance" "web" {
  ami = "ami-0c55b159cbfafe1f0"
  tags = {
    Name = "web"
  # Missing closing } for tags block
}
```

---

### Validate Requires Init First

`terraform validate` reads provider schemas to know what arguments are valid for each resource type. This means you must run `terraform init` before `terraform validate` — the provider needs to be downloaded first.

```bash
# Must do this first
terraform init

# Then validate
terraform validate
```

If you run validate before init:
```
╷
│ Error: Could not load plugin
│
│ Plugin reinitialization required. Please run "terraform init".
╵
```

---

### Using Validate in the Development Workflow

Validate is most useful during development — run it frequently as you write code to catch errors early:

```bash
# Write some code...
# Then check it:
terraform validate

# Fix any errors...
# Check again:
terraform validate

# All good? Now plan:
terraform plan
```

This saves time because `terraform validate` is instant (no API calls), while `terraform plan` takes longer because it queries the cloud API.

---

### Validate in CI/CD

Add validate to your pipeline after init and before plan:

```yaml
# GitHub Actions example
steps:
  - name: Terraform Init
    run: terraform init

  - name: Terraform Format Check
    run: terraform fmt -check

  - name: Terraform Validate
    run: terraform validate

  - name: Terraform Plan
    run: terraform plan
```

The order is intentional:
1. `init` — set up providers (required for validate)
2. `fmt -check` — ensure formatting is correct (fast)
3. `validate` — ensure code is correct (fast, no API calls)
4. `plan` — check against real infrastructure (slower, requires credentials)

---

## The Difference Between fmt and validate

| | `terraform fmt` | `terraform validate` |
|---|---|---|
| **Purpose** | Fix code style/formatting | Check code correctness |
| **Changes files?** | Yes (or preview with `-check`) | No |
| **Catches logic errors?** | No | Yes |
| **Requires init?** | No | Yes |
| **Requires credentials?** | No | No |
| **Speed** | Very fast | Fast |
| **What it checks** | Indentation, alignment, spacing | Syntax, types, references, required args |

---

## Other Useful Code Quality Tools

### `terraform providers`

Shows which providers your configuration uses and their required versions:

```bash
terraform providers
```

Output:
```
Providers required by configuration:
.
└── provider[registry.terraform.io/hashicorp/aws] ~> 5.0
```

### `terraform providers schema`

Shows the complete schema for all providers — all available resources and their arguments:

```bash
terraform providers schema -json | jq .
```

This is very useful for understanding what arguments a resource supports without going to the documentation website.

---

## Pre-commit Hooks for Automatic Checks

For teams that want to automatically run `fmt` and `validate` before every Git commit, use **pre-commit hooks**.

Install the `pre-commit` tool and create a `.pre-commit-config.yaml` file:

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.86.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
```

Install the hooks:
```bash
pip install pre-commit
pre-commit install
```

Now every time a developer runs `git commit`, the hooks automatically:
1. Format all `.tf` files
2. Validate the configuration
3. Update any documentation

If any check fails, the commit is blocked until the issues are fixed. This ensures the repository always contains clean, valid code.

---

## Summary

- `terraform fmt` automatically formats your `.tf` files to follow the standard Terraform style — 2-space indentation, aligned `=` signs, consistent spacing
- Run `terraform fmt` before committing code, or set up your editor to format on save
- Use `terraform fmt -check` in CI/CD to enforce formatting standards
- `terraform validate` checks your code for syntax errors, type errors, invalid arguments, and broken references
- Validate does NOT check credentials or real-world AWS state — it only checks the code itself
- Always run `terraform init` before `terraform validate`
- In a CI/CD pipeline, the recommended order is: `init` → `fmt -check` → `validate` → `plan` → `apply`
- Pre-commit hooks can automate running these checks on every Git commit

In the next file, we put all the CLI commands together in a comprehensive cheat sheet for quick reference.
