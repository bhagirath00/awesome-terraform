# The HashiCorp Terraform VS Code Extension

## Introduction

The **HashiCorp Terraform** extension for VS Code is the official extension maintained by HashiCorp. It transforms VS Code from a generic text editor into a full-featured Terraform IDE (Integrated Development Environment).

This extension is not optional for serious Terraform work — it catches errors as you type, completes code automatically, and helps you navigate large codebases efficiently. Every professional Terraform developer has it installed.

---

## Installing the Extension

### Method 1: VS Code Marketplace (GUI)

1. Open VS Code
2. Press `Ctrl+Shift+X` (or `Cmd+Shift+X` on Mac) to open Extensions
3. Search for "HashiCorp Terraform"
4. Look for the extension with the HashiCorp icon (orange diamond)
5. Click "Install"

**Extension ID:** `hashicorp.terraform`

### Method 2: Command Line

```bash
code --install-extension hashicorp.terraform
```

### Method 3: From VS Code Marketplace Website

Go to: https://marketplace.visualstudio.com/items?itemName=hashicorp.terraform
Click "Install" and follow the prompt to open VS Code.

---

## The Terraform Language Server

The extension works by running a background process called the **Terraform Language Server** (terraform-ls). This is a separate program that the extension communicates with to provide all its intelligent features.

When you install the extension and open a `.tf` file, VS Code:
1. Starts the language server in the background
2. The language server reads your Terraform configuration files
3. The language server provides information back to VS Code (completions, errors, hover info)

You usually don't need to interact with the language server directly — it runs automatically. But it's useful to know it exists so you understand why the extension needs `terraform init` to have been run in your project (the language server reads the downloaded provider schemas to know what arguments each resource accepts).

---

## Core Feature 1: Syntax Highlighting

The most basic feature — HCL code is colored by type:

- **Block types** (`resource`, `variable`, `output`, `provider`) — highlighted in one color
- **Resource types** (`"aws_instance"`, `"aws_s3_bucket"`) — highlighted in another color
- **Argument names** (`ami`, `instance_type`, `bucket`) — styled distinctly
- **String values** — colored differently from numbers and booleans
- **References** (`var.name`, `aws_instance.web.id`) — highlighted to stand out
- **Comments** — typically shown in gray/italic

This makes code much easier to read, especially in large files.

---

## Core Feature 2: Auto-Completion (IntelliSense)

This is the feature that saves the most time. As you type, the extension suggests:

### Block Type Completion

When you start typing a block type, the extension shows all available options:

```
res<cursor>
```

Suggestions appear:
- `resource`

Press Tab or Enter to complete.

---

### Resource Type Completion

After typing `resource "`, the extension suggests all available resource types from your installed providers:

```hcl
resource "<cursor>"
```

Start typing `aws_s3` and you see:
- `aws_s3_bucket`
- `aws_s3_bucket_acl`
- `aws_s3_bucket_cors_configuration`
- `aws_s3_bucket_lifecycle_configuration`
- `aws_s3_bucket_notification`
- `aws_s3_bucket_object`
- `aws_s3_bucket_policy`
- `aws_s3_bucket_versioning`
- ... and more

Select with arrow keys and press Tab to complete.

---

### Argument Name Completion

Inside a resource block, the extension knows every valid argument:

```hcl
resource "aws_instance" "web" {
  a<cursor>
}
```

Suggestions:
- `ami` (required)
- `associate_public_ip_address`
- `availability_zone`

You can see which arguments are required (marked with a star or label) and which are optional.

---

### Value Completion

For arguments with a known set of valid values, the extension suggests them:

```hcl
resource "aws_instance" "web" {
  instance_type = "<cursor>"
}
```

Suggestions:
- `t2.micro`
- `t2.small`
- `t2.medium`
- `t3.micro`
- `t3.small`
- ... (all valid instance types)

---

### Reference Completion

When referencing other resources, variables, or locals:

```hcl
resource "aws_subnet" "public" {
  vpc_id = aws_<cursor>
}
```

The extension suggests all resources of type `aws_*` that exist in your configuration.

---

## Core Feature 3: Hover Documentation

Hover your mouse over any argument name or resource type to see inline documentation — without leaving VS Code.

**Hovering over a resource type:**
```
aws_instance
Resource: EC2 Instance
Provides an EC2 instance resource. This allows instances to be created, updated, and deleted.
[Learn more ↗]
```

**Hovering over an argument:**
```
instance_type
Type: string
Required: Yes
The instance type to use for the instance.
```

This means you rarely need to open a browser to look up documentation — the docs come to you.

---

## Core Feature 4: Inline Error Detection

The extension highlights errors in your code before you run any Terraform command.

**Red underlines** show errors:
- Missing required arguments
- Unknown argument names (typos)
- Type mismatches
- References to non-existent resources

**Yellow underlines** show warnings:
- Deprecated arguments
- Arguments that are set but have no effect

**Example:**

```hcl
resource "aws_instance" "web" {
  ami = "ami-0c55b159cbfafe1f0"
  # instance_type is missing — red underline on the resource block
}
```

You see the error immediately, highlighted in red in your code and listed in the Problems panel (`Ctrl+Shift+M`).

---

## Core Feature 5: Go to Definition

Press `F12` or `Ctrl+Click` on any reference to jump to its definition.

**Jumping to a variable definition:**
```hcl
resource "aws_instance" "web" {
  instance_type = var.instance_type  # Ctrl+Click here
}
```

VS Code jumps to:
```hcl
variable "instance_type" {   # ← Cursor lands here
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}
```

**Jumping to a resource definition:**
```hcl
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id  # Ctrl+Click on aws_vpc.main
}
```

VS Code jumps to the `aws_vpc "main"` resource block, even if it is in a different file.

This is extremely useful in large projects where your configuration is spread across many files.

---

## Core Feature 6: Find All References

Press `Shift+F12` on any resource or variable name to see everywhere it is used.

**Example:** Right-click on `aws_vpc.main` → "Find All References"

VS Code shows a panel listing every location in every file where `aws_vpc.main` is referenced. Click any result to jump to that location.

---

## Core Feature 7: Rename Symbol

Press `F2` on any resource name, variable name, or local name to rename it everywhere it is used.

**Example:** You have a variable named `region` that you want to rename to `aws_region`:
1. Click on the variable name `region` in `variables.tf`
2. Press `F2`
3. Type the new name `aws_region`
4. Press Enter

VS Code finds and renames every reference — in `variables.tf`, `providers.tf`, `main.tf`, everywhere — automatically and simultaneously.

---

## Core Feature 8: Code Formatting

The extension provides the formatter that VS Code uses when you press `Shift+Alt+F` (Format Document) or when format-on-save is enabled.

It runs `terraform fmt` in the background and applies the result to your open file.

Set up format-on-save (add to your VS Code settings):
```json
{
  "[terraform]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true
  },
  "[terraform-vars]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true
  }
}
```

---

## Core Feature 9: Document Symbols and Outline

The extension provides an "Outline" view (`Ctrl+Shift+O`) that shows all the blocks in your current file:

```
main.tf
├── resource "aws_vpc" "main"
├── resource "aws_subnet" "public"
├── resource "aws_subnet" "private"
├── resource "aws_internet_gateway" "main"
└── resource "aws_route_table" "public"
```

Click any item to jump directly to it. This is very helpful in large files with many resources.

---

## Core Feature 10: Module Navigation

When you reference a module:

```hcl
module "vpc" {
  source = "./modules/vpc"   # Ctrl+Click on the source path
}
```

`Ctrl+Click` on the source path opens the module directory or file. You can navigate into modules and back just like jumping to any other definition.

---

## Configuring the Extension

The extension has settings you can configure in your VS Code `settings.json`:

```json
{
  // Experimental features (some are very useful)
  "terraform.experimentalFeatures.prefillRequiredFields": true,
  "terraform.experimentalFeatures.validateOnSave": true,

  // Language server settings
  "terraform.languageServer.enable": true,

  // Terraform executable path (if not in PATH)
  "terraform.path": "/usr/local/bin/terraform",

  // Log level for the language server (for debugging extension issues)
  "terraform.languageServer.logLevel": "info"
}
```

### `prefillRequiredFields`

When enabled, after completing a resource type, the extension automatically inserts all required arguments as a skeleton:

```hcl
resource "aws_instance" "web" {
  ami           = ""  ← Cursor here, required fields pre-filled
  instance_type = ""
}
```

This is very helpful when you are learning — it shows you exactly what is required for each resource.

### `validateOnSave`

When enabled, the extension runs `terraform validate` every time you save a file. Errors appear immediately in the Problems panel.

---

## The Problems Panel

Press `Ctrl+Shift+M` to open the Problems panel. This shows all errors and warnings across all your open files.

```
PROBLEMS  OUTPUT  DEBUG CONSOLE  TERMINAL

● main.tf
  ✕ Missing required argument: "ami" (line 3)
  ⚠ Argument "engine_version" is deprecated (line 47)

● variables.tf
  ✕ Variable "instance_count" is declared but never used (line 12)
```

Click any problem to jump directly to that line in the file. This is the fastest way to find and fix errors in your configuration.

---

## Tips for Getting the Most Out of the Extension

### Tip 1: Always Run `terraform init` First

The extension reads provider schemas (the list of all resources and their arguments) from the downloaded providers in your `.terraform` directory. Without `terraform init`, the extension doesn't know what arguments `aws_instance` accepts, so completions and error detection won't work.

Always run `terraform init` when you start a new project or when someone else installs from a cloned repository.

### Tip 2: Use the Command Palette for Terraform Commands

Press `Ctrl+Shift+P` and type "Terraform" to see all available Terraform commands you can run directly from VS Code:
- `Terraform: init`
- `Terraform: validate`
- `Terraform: plan`
- `Terraform: Apply`

These run in the integrated terminal — you don't have to switch to the terminal manually.

### Tip 3: Keep the Extension Updated

The HashiCorp Terraform extension updates frequently with new features and bug fixes. Keep it updated through the VS Code Extensions panel.

### Tip 4: Use Outline View for Large Files

In large files with many resources, use the Outline view (`Ctrl+Shift+O`) to navigate. It is much faster than scrolling.

### Tip 5: Pair with Git Source Control View

Use VS Code's built-in Git Source Control view (`Ctrl+Shift+G`) to review your Terraform changes before committing. You can see exactly what changed in each `.tf` file as a diff.

---

## Troubleshooting Common Extension Issues

### Extension not providing completions

**Possible causes:**
- `terraform init` has not been run
- The language server crashed

**Fix:**
```bash
# Run terraform init in your project
terraform init

# In VS Code: Ctrl+Shift+P → "Developer: Reload Window"
```

### Completions are showing outdated resource types

**Fix:**
```bash
# Update providers
terraform init -upgrade

# Reload VS Code window
```

### Language server using wrong Terraform executable

If you have multiple versions of Terraform installed (e.g., via tfenv), set the path explicitly:

```json
{
  "terraform.path": "/home/username/.tfenv/shims/terraform"
}
```

### Extension showing errors that don't actually exist

Sometimes the extension gets out of sync. Fix with:
- `Ctrl+Shift+P` → "Terraform: restart language server"

---

## Summary

The HashiCorp Terraform extension for VS Code provides:

- **Syntax highlighting** — colors HCL code by type for readability
- **Auto-completion** — suggests resource types, argument names, and valid values as you type
- **Hover documentation** — shows docs for any resource or argument without leaving the editor
- **Inline error detection** — highlights mistakes before you run any command
- **Go to definition** — jump to any resource, variable, or module with Ctrl+Click
- **Find all references** — see everywhere a resource or variable is used
- **Rename symbol** — rename a resource or variable everywhere at once
- **Code formatting** — runs `terraform fmt` automatically on save
- **Document outline** — navigate large files with an overview of all blocks
- **Module navigation** — jump into module source files directly

To get the most from the extension:
- Always run `terraform init` before expecting completions to work
- Enable `prefillRequiredFields` to see required arguments automatically
- Enable `validateOnSave` for instant feedback
- Use the Problems panel (`Ctrl+Shift+M`) to see all errors at once

With VS Code properly set up and the Terraform extension configured, you have a professional Terraform development environment that catches mistakes early, speeds up writing, and helps you navigate large codebases with ease.

This completes Module 2 — Installation & Setup. You now have everything you need to start writing and running real Terraform configurations. In Module 3, we dive into HCL Fundamentals — the language features that make Terraform configurations powerful and flexible.
