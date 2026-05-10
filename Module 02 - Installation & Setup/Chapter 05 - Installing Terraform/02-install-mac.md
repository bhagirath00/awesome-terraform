# Installing Terraform on macOS

## Introduction

Installing Terraform on macOS is straightforward. There are two main methods:

1. **Using Homebrew** (recommended — the easiest and most popular method on Mac)
2. **Manual installation from a zip file** (useful when you need a specific version or cannot use Homebrew)
3. **Using tfenv** (best when you need to switch between multiple Terraform versions)

Most macOS developers use Homebrew because it is already part of their workflow. If you do not have Homebrew installed yet, we will cover that first.

---

## Prerequisites

- macOS 10.15 (Catalina) or later
- Terminal access
- Internet connection

---

## Method 1: Using Homebrew (Recommended)

Homebrew is the most popular package manager for macOS. If you are a developer on a Mac, you almost certainly have it installed. If not, we will install it first.

### Step 1: Install Homebrew (if not already installed)

Open your Terminal and run:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Follow the on-screen instructions. The script will ask for your macOS password and will install the necessary tools (including Xcode Command Line Tools if needed).

After installation, verify Homebrew is working:

```bash
brew --version
```

You should see output like `Homebrew 4.x.x`.

### Step 2: Install the HashiCorp tap

A "tap" in Homebrew is an additional repository of packages. HashiCorp maintains their own tap with their official tools:

```bash
brew tap hashicorp/tap
```

### Step 3: Install Terraform

```bash
brew install hashicorp/tap/terraform
```

Homebrew will download and install Terraform along with any dependencies. This may take a minute or two.

### Step 4: Verify the installation

```bash
terraform version
```

Expected output:
```
Terraform v1.6.6
on darwin_amd64
```

(For Apple Silicon Macs, you will see `darwin_arm64` instead of `darwin_amd64`)

---

### Updating Terraform with Homebrew

When a new version of Terraform is released:

```bash
brew update
brew upgrade hashicorp/tap/terraform
```

Or to upgrade all Homebrew packages at once:

```bash
brew upgrade
```

---

## Method 2: Manual Installation from Zip File

Use this method when:
- You cannot use Homebrew
- You need a very specific version of Terraform
- You are setting up a script that automates the installation

### Step 1: Determine your Mac's architecture

Apple Silicon Macs (M1, M2, M3 chips) use ARM architecture. Intel Macs use AMD64 architecture. You need to download the correct binary.

```bash
uname -m
```

- If the output is `arm64` → you have an Apple Silicon Mac → download `darwin_arm64`
- If the output is `x86_64` → you have an Intel Mac → download `darwin_amd64`

### Step 2: Download Terraform

Go to https://releases.hashicorp.com/terraform/ and find the latest version, or use these commands:

**For Apple Silicon (M1/M2/M3):**
```bash
TF_VERSION="1.6.6"
curl -O https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_darwin_arm64.zip
```

**For Intel Mac:**
```bash
TF_VERSION="1.6.6"
curl -O https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_darwin_amd64.zip
```

### Step 3: Verify the checksum (recommended)

```bash
# Download the checksum file
curl -O https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_SHA256SUMS

# Verify (replace filename with your architecture)
shasum -a 256 -c terraform_${TF_VERSION}_SHA256SUMS --ignore-missing
```

You should see: `terraform_1.6.6_darwin_arm64.zip: OK`

### Step 4: Extract and install

```bash
# Extract the zip (replace filename with your architecture)
unzip terraform_${TF_VERSION}_darwin_arm64.zip

# Move the binary to a directory in your PATH
sudo mv terraform /usr/local/bin/

# Verify
terraform version
```

### Step 5: Clean up

```bash
rm terraform_${TF_VERSION}_darwin_arm64.zip
rm terraform_${TF_VERSION}_SHA256SUMS
```

---

### Handling macOS Security Warning

When you install Terraform manually (not through Homebrew), macOS may show a security warning when you first run it:

```
"terraform" cannot be opened because the developer cannot be verified.
```

To fix this:

**Option 1: Remove the quarantine attribute**
```bash
sudo xattr -d com.apple.quarantine /usr/local/bin/terraform
```

**Option 2: Allow it through System Settings**
1. Go to System Settings → Privacy & Security
2. Scroll down to the Security section
3. You will see a message about terraform being blocked
4. Click "Allow Anyway"
5. Run `terraform version` again and click "Open" in the dialog

---

## Method 3: Using tfenv (For Managing Multiple Versions)

tfenv is a Terraform version manager — it lets you install multiple versions and switch between them easily. This is very helpful in real-world DevOps work.

### Step 1: Install tfenv using Homebrew

```bash
brew install tfenv
```

### Step 2: List available Terraform versions

```bash
tfenv list-remote
```

This shows all available Terraform versions you can install.

### Step 3: Install Terraform versions

```bash
# Install the latest version
tfenv install latest

# Install a specific version
tfenv install 1.6.6

# Install an older version for a legacy project
tfenv install 1.5.7
```

### Step 4: Select which version to use

```bash
# Set globally (used everywhere unless overridden)
tfenv use 1.6.6

# Verify
terraform version
```

### Step 5: Pin a version per project

Inside a project directory, create a `.terraform-version` file:

```bash
cd my-project
echo "1.6.6" > .terraform-version
```

When you are inside this directory, tfenv automatically uses version 1.6.6. This is excellent for teams — everyone gets the same version when they enter the project directory.

### Step 6: Manage installed versions

```bash
# List locally installed versions
tfenv list

# Uninstall a version you no longer need
tfenv uninstall 1.5.7
```

---

## Verifying Your Installation

After installing Terraform, run these checks:

**Check the version:**
```bash
terraform version
```

**Check where Terraform is installed:**
```bash
which terraform
```

**Run a quick test:**
```bash
mkdir /tmp/tf-test && cd /tmp/tf-test

cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.0"
}
EOF

terraform init
cd ~ && rm -rf /tmp/tf-test
```

If everything runs without errors, your installation is complete.

---

## Setting Up Shell Autocompletion on macOS

Tab completion makes working with Terraform much faster.

**For bash (default on older macOS):**
```bash
terraform -install-autocomplete
source ~/.bash_profile
```

**For zsh (default on macOS Catalina and later):**
```bash
terraform -install-autocomplete
source ~/.zshrc
```

After this, you can type `terraform p` and press Tab to see all commands starting with `p` (plan, providers, etc.).

---

## Installing the AWS CLI on macOS (Needed for Next Chapter)

Since we will be working with AWS in the next chapter, let us also install the AWS CLI.

**Using Homebrew:**
```bash
brew install awscli
aws --version
```

**Using the official installer:**

Download the pkg installer from AWS:
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
aws --version
```

After installing the AWS CLI, configure it with your credentials:
```bash
aws configure
```

You will be prompted to enter:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-east-1`)
- Default output format (enter `json`)

We cover AWS credentials and authentication in detail in Chapter 6.

---

## Comparing Installation Methods on macOS

| Method | Pros | Cons | Best For |
|---|---|---|---|
| **Homebrew** | Easy, automatic updates, simple | Installs latest only by default | Most macOS developers |
| **Manual zip** | Full control over version, no extra tools needed | Manual updates, security warning | Specific version needs |
| **tfenv** | Manage multiple versions, per-project pinning | Extra tool to install | Teams, multi-project work |

---

## Common Issues on macOS

### Issue: "brew: command not found"

Homebrew is not installed. Follow the Homebrew installation step at the beginning of this file.

### Issue: Terraform binary shows wrong version after upgrade

The old binary might be cached. Run:
```bash
hash -r
terraform version
```

### Issue: "Permission denied" when moving to /usr/local/bin

```bash
# Use sudo
sudo mv terraform /usr/local/bin/

# Or install to user's local bin (no sudo needed)
mkdir -p ~/bin
mv terraform ~/bin/
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Issue: Apple Silicon compatibility

If you have an M1/M2/M3 Mac and installed the AMD64 binary by mistake, Terraform may run slowly through Rosetta 2 translation. Download and install the `darwin_arm64` binary instead.

---

## Summary

- **Homebrew** is the easiest and most recommended way to install Terraform on macOS
- **Manual installation** gives you control over the exact version and works without Homebrew
- **tfenv** is the best choice when managing multiple versions across different projects
- Always verify installation with `terraform version`
- On Apple Silicon Macs, make sure to download the `darwin_arm64` binary for native performance
- Set up shell autocompletion to speed up your daily Terraform work
- Install the AWS CLI alongside Terraform as you will need it for authentication in the next chapter

In the next file, we cover installing Terraform on Windows.
