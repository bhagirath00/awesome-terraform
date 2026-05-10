# Installing Terraform on Linux

## Introduction

This file walks you through installing Terraform on a Linux system step by step. We will cover multiple installation methods so you can choose the one that works best for your environment.

Terraform is distributed as a single binary file — there is no complex installer, no dependencies to manage, and no runtime required. Once the binary is on your system and accessible from the terminal, Terraform is ready to use.

We will cover three installation methods:
1. **Using the official HashiCorp package repository** (recommended for most Linux distributions)
2. **Manual installation from a zip file** (works on any Linux system)
3. **Using tfenv** (recommended if you need to switch between Terraform versions)

---

## Prerequisites

Before installing Terraform, make sure you have:

- A Linux system (Ubuntu, Debian, CentOS, RHEL, Amazon Linux, etc.)
- Terminal access with sudo privileges
- Internet access to download packages
- `curl` or `wget` installed (usually available by default)

To check if curl is available:
```bash
curl --version
```

---

## Method 1: Official HashiCorp Package Repository (Recommended)

HashiCorp provides official package repositories for Debian/Ubuntu and RHEL/CentOS based systems. This method is recommended because:
- You get automatic updates when you run system updates
- The package is signed and verified by HashiCorp
- Installation is straightforward

### On Ubuntu / Debian

**Step 1: Install required dependencies**

```bash
sudo apt-get update
sudo apt-get install -y gnupg software-properties-common curl
```

**Step 2: Add HashiCorp's GPG key**

This key is used to verify that packages you download are genuine and haven't been tampered with.

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null
```

**Step 3: Verify the key fingerprint (optional but good practice)**

```bash
gpg --no-default-keyring \
  --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg \
  --fingerprint
```

You should see a fingerprint that matches the one on HashiCorp's official website.

**Step 4: Add the HashiCorp repository**

```bash
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
```

**Step 5: Update the package list and install Terraform**

```bash
sudo apt-get update
sudo apt-get install terraform
```

**Step 6: Verify the installation**

```bash
terraform version
```

Expected output:
```
Terraform v1.6.6
on linux_amd64
```

---

### On CentOS / RHEL / Amazon Linux

**Step 1: Install yum-utils**

```bash
sudo yum install -y yum-utils
```

**Step 2: Add the HashiCorp repository**

```bash
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
```

**Step 3: Install Terraform**

```bash
sudo yum -y install terraform
```

**Step 4: Verify the installation**

```bash
terraform version
```

---

### On Fedora

```bash
sudo dnf install -y dnf-plugins-core
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/fedora/hashicorp.repo
sudo dnf -y install terraform
terraform version
```

---

## Method 2: Manual Installation from Zip File

This method works on any Linux distribution. It is useful when:
- You cannot add external package repositories
- You are working in a restricted environment
- You want to install a specific version

**Step 1: Find the latest version**

Go to https://releases.hashicorp.com/terraform/ to find the latest version, or use this command to get the latest version number:

```bash
LATEST=$(curl -s https://checkpoint-api.hashicorp.com/v1/check/terraform | grep -o '"current_version":"[^"]*"' | cut -d'"' -f4)
echo "Latest Terraform version: $LATEST"
```

**Step 2: Download the binary for your system**

For 64-bit Linux (most common):

```bash
# Set the version you want to install
TF_VERSION="1.6.6"

# Download the zip file
wget https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip
```

**Step 3: Download and verify the checksum (security best practice)**

```bash
# Download the checksum file
wget https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_SHA256SUMS

# Verify the downloaded zip matches the checksum
sha256sum -c terraform_${TF_VERSION}_SHA256SUMS --ignore-missing
```

You should see: `terraform_1.6.6_linux_amd64.zip: OK`

**Step 4: Extract the binary**

```bash
unzip terraform_${TF_VERSION}_linux_amd64.zip
```

This creates a single file called `terraform` in your current directory.

**Step 5: Move the binary to a directory in your PATH**

```bash
sudo mv terraform /usr/local/bin/
```

**Step 6: Set the correct permissions**

```bash
sudo chmod +x /usr/local/bin/terraform
```

**Step 7: Clean up the downloaded files**

```bash
rm terraform_${TF_VERSION}_linux_amd64.zip
rm terraform_${TF_VERSION}_SHA256SUMS
```

**Step 8: Verify the installation**

```bash
terraform version
```

---

## Method 3: Using tfenv (Terraform Version Manager)

**tfenv** is a tool that lets you install and switch between multiple versions of Terraform easily. This is very useful in real-world DevOps work where:
- Different projects use different Terraform versions
- You want to test a new version without removing the old one
- Your team uses a specific pinned version

**Step 1: Clone tfenv from GitHub**

```bash
git clone --depth=1 https://github.com/tfutils/tfenv.git ~/.tfenv
```

**Step 2: Add tfenv to your PATH**

For bash users:
```bash
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

For zsh users:
```bash
echo 'export PATH="$HOME/.tfenv/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

**Step 3: Verify tfenv is installed**

```bash
tfenv --version
```

**Step 4: Install a specific Terraform version**

```bash
# Install the latest version
tfenv install latest

# Install a specific version
tfenv install 1.6.6

# Install another version
tfenv install 1.5.7
```

**Step 5: Select which version to use**

```bash
# Use globally
tfenv use 1.6.6

# Check which version is active
terraform version
```

**Step 6: List installed versions**

```bash
tfenv list
```

Output:
```
* 1.6.6 (set by /home/username/.tfenv/version)
  1.5.7
```

**Step 7: Pin a version for a specific project**

Inside your project directory, create a `.terraform-version` file:

```bash
echo "1.6.6" > .terraform-version
```

When you are in this directory, tfenv will automatically use version 1.6.6 regardless of the global setting. This is very useful for teams — everyone uses the exact same version.

---

## Verifying Your Installation

After installing Terraform using any of the methods above, run these commands to confirm everything is working correctly.

**Check the version:**
```bash
terraform version
```

**Check the binary location:**
```bash
which terraform
```

Expected output: `/usr/local/bin/terraform` or `/usr/bin/terraform`

**Run a simple help command:**
```bash
terraform --help
```

You should see a list of all available Terraform commands.

**Test that Terraform can initialize a basic project:**

```bash
# Create a test directory
mkdir /tmp/tf-test && cd /tmp/tf-test

# Create a minimal configuration
cat > main.tf << 'EOF'
terraform {
  required_version = ">= 1.0"
}
EOF

# Initialize
terraform init

# Clean up
cd ~ && rm -rf /tmp/tf-test
```

If `terraform init` runs without errors, your installation is complete and working.

---

## Setting Up Shell Autocompletion

Terraform supports tab completion in bash and zsh. This saves a lot of typing when working with Terraform commands.

**For bash:**
```bash
terraform -install-autocomplete
source ~/.bashrc
```

**For zsh:**
```bash
terraform -install-autocomplete
source ~/.zshrc
```

After this, you can type `terraform pl` and press Tab to autocomplete to `terraform plan`.

---

## Keeping Terraform Updated

### If installed via package manager (apt/yum):

```bash
# Ubuntu/Debian
sudo apt-get update && sudo apt-get upgrade terraform

# CentOS/RHEL
sudo yum update terraform
```

### If installed manually:

Download the new version zip file, extract it, and replace the existing binary at `/usr/local/bin/terraform`.

### If using tfenv:

```bash
tfenv install latest
tfenv use latest
```

---

## Uninstalling Terraform

### If installed via package manager:

```bash
# Ubuntu/Debian
sudo apt-get remove terraform

# CentOS/RHEL
sudo yum remove terraform
```

### If installed manually:

```bash
sudo rm /usr/local/bin/terraform
```

### If installed via tfenv:

```bash
tfenv uninstall 1.6.6
# Or remove all of tfenv
rm -rf ~/.tfenv
```

---

## Common Installation Issues

### Issue: "command not found: terraform"

**Cause:** The terraform binary is not in your PATH.

**Fix:**
```bash
# Find where terraform is installed
find / -name "terraform" -type f 2>/dev/null

# Add its location to PATH (replace /path/to with actual path)
export PATH=$PATH:/path/to/terraform/directory

# Make it permanent
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
```

### Issue: "Permission denied"

**Cause:** The terraform binary is not executable.

**Fix:**
```bash
sudo chmod +x /usr/local/bin/terraform
```

### Issue: "cannot execute binary file: Exec format error"

**Cause:** You downloaded the wrong architecture binary (e.g., ARM binary on an x86 machine).

**Fix:** Check your system architecture and download the correct binary:
```bash
uname -m
# x86_64 → use linux_amd64
# aarch64 or arm64 → use linux_arm64
```

---

## Summary

- Terraform is a single binary — installation is straightforward on any Linux system
- The **HashiCorp package repository** method is recommended for Ubuntu/Debian and CentOS/RHEL — it gives you easy updates
- The **manual zip method** works on any Linux distribution when you cannot use package managers
- **tfenv** is the best option when you need to manage multiple Terraform versions across different projects
- Always verify your installation with `terraform version` after installing
- Set up shell autocompletion to make your daily Terraform work faster

In the next file, we cover installing Terraform on macOS.
