# Installing Terraform on Windows

## Introduction

Terraform runs perfectly well on Windows. There are several ways to install it, and we will walk through each one clearly. We will also cover the recommended approach for DevOps engineers on Windows — using **Windows Subsystem for Linux (WSL)** — which gives you a real Linux environment inside Windows, making it much easier to work with Terraform and other DevOps tools.

Installation methods covered in this file:
1. **Using Chocolatey** (Windows package manager — recommended for Windows users)
2. **Using winget** (built-in Windows package manager)
3. **Manual installation** (download and set up manually)
4. **Using WSL (Windows Subsystem for Linux)** — strongly recommended for DevOps work

---

## Prerequisites

- Windows 10 (version 1903 or later) or Windows 11
- Administrator access for some installation methods
- Internet connection

---

## Method 1: Using Chocolatey (Recommended for Windows)

**Chocolatey** is the most popular package manager for Windows. It works similarly to `apt` on Ubuntu or `brew` on macOS. If you do DevOps work on Windows, Chocolatey is worth having.

### Step 1: Install Chocolatey

Open **PowerShell as Administrator** (right-click PowerShell → "Run as Administrator") and run:

```powershell
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
```

After installation, close and reopen PowerShell (as Administrator again), then verify:

```powershell
choco --version
```

### Step 2: Install Terraform

```powershell
choco install terraform
```

Chocolatey will download and install Terraform automatically.

### Step 3: Verify the installation

Close and reopen PowerShell (not as administrator this time), then:

```powershell
terraform version
```

Expected output:
```
Terraform v1.6.6
on windows_amd64
```

### Updating Terraform with Chocolatey

```powershell
choco upgrade terraform
```

---

## Method 2: Using winget (Built into Windows 10/11)

**winget** is Microsoft's official package manager, built into Windows 10 (version 1809+) and Windows 11. No installation required.

Open **Command Prompt** or **PowerShell** and run:

```powershell
winget install HashiCorp.Terraform
```

Verify:
```powershell
terraform version
```

To upgrade later:
```powershell
winget upgrade HashiCorp.Terraform
```

---

## Method 3: Manual Installation

Use this method when you cannot use a package manager or need a very specific version.

### Step 1: Download Terraform

1. Open your browser and go to: https://releases.hashicorp.com/terraform/
2. Click on the latest version (e.g., `1.6.6`)
3. Download the file: `terraform_1.6.6_windows_amd64.zip`

Or download directly from PowerShell:

```powershell
$TF_VERSION = "1.6.6"
Invoke-WebRequest -Uri "https://releases.hashicorp.com/terraform/$TF_VERSION/terraform_${TF_VERSION}_windows_amd64.zip" -OutFile "terraform_$TF_VERSION.zip"
```

### Step 2: Extract the zip file

```powershell
Expand-Archive -Path "terraform_$TF_VERSION.zip" -DestinationPath "C:\terraform"
```

This creates a folder `C:\terraform` containing the `terraform.exe` file.

### Step 3: Add Terraform to your PATH

This step is important — without it, Windows won't know where to find the `terraform` command.

**Option A: Through the GUI (easy)**
1. Press `Win + R`, type `sysdm.cpl`, press Enter
2. Go to the **Advanced** tab → click **Environment Variables**
3. Under **System variables**, find **Path** and click **Edit**
4. Click **New** and add: `C:\terraform`
5. Click **OK** on all dialogs
6. Close and reopen Command Prompt or PowerShell

**Option B: Through PowerShell (faster)**
```powershell
[System.Environment]::SetEnvironmentVariable(
  "Path",
  $env:Path + ";C:\terraform",
  [System.EnvironmentVariableTarget]::Machine
)
```

Close and reopen PowerShell after running this.

### Step 4: Verify the installation

```powershell
terraform version
```

### Step 5: Clean up

```powershell
Remove-Item "terraform_$TF_VERSION.zip"
```

---

## Method 4: Windows Subsystem for Linux (WSL) — Strongly Recommended

If you are doing serious DevOps work on Windows, **WSL** (Windows Subsystem for Linux) is the best approach. Here is why:

- Most DevOps tools (Terraform, kubectl, Helm, AWS CLI, Docker, etc.) are designed for Linux/macOS
- Shell scripts and bash commands in tutorials and documentation are written for Linux
- WSL gives you a full Ubuntu (or other Linux distro) environment inside Windows
- You can use the exact same commands and workflows as Linux DevOps engineers

### Step 1: Enable WSL

Open **PowerShell as Administrator** and run:

```powershell
wsl --install
```

This installs WSL2 with Ubuntu by default. Restart your computer when prompted.

If you already have WSL but want to make sure you have WSL2:
```powershell
wsl --set-default-version 2
```

### Step 2: Set up Ubuntu

After restart, Ubuntu will open automatically and ask you to create a username and password. This is your Linux user — choose something easy to remember.

### Step 3: Update Ubuntu

In the Ubuntu terminal:
```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### Step 4: Install Terraform inside WSL

Now follow the **Linux installation steps** from the `install-linux.md` file. Inside WSL, you are running a full Ubuntu system, so the Linux instructions apply exactly.

Quick version for Ubuntu in WSL:

```bash
# Install dependencies
sudo apt-get install -y gnupg software-properties-common curl wget

# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | \
  gpg --dearmor | \
  sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg > /dev/null

# Add repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install Terraform
sudo apt-get update && sudo apt-get install -y terraform

# Verify
terraform version
```

### Step 5: Access your Windows files from WSL

Your Windows drives are accessible inside WSL at `/mnt/`:
- `C:\` drive → `/mnt/c/`
- `D:\` drive → `/mnt/d/`

To navigate to your Windows Desktop from WSL:
```bash
cd /mnt/c/Users/YourUsername/Desktop
```

However, it is recommended to keep your Terraform project files in the Linux filesystem (`~/projects/`) rather than the Windows filesystem for better performance.

### Step 6: Using VS Code with WSL

VS Code has excellent WSL integration. To open VS Code from WSL:

```bash
code .
```

VS Code will open on Windows but will be connected to the WSL filesystem. This gives you the best of both worlds — a Windows GUI with a Linux backend.

---

## Recommended Terminal for Windows Terraform Work

Regardless of which installation method you use, we recommend installing **Windows Terminal** from the Microsoft Store. It supports multiple tabs (PowerShell, Command Prompt, WSL) in one window and is much better than the default terminals.

Install from PowerShell:
```powershell
winget install Microsoft.WindowsTerminal
```

---

## Installing the AWS CLI on Windows

You will need the AWS CLI for the next chapter on provider authentication.

**Using winget:**
```powershell
winget install Amazon.AWSCLI
```

**Manual installation:**
1. Download the installer from: https://awscli.amazonaws.com/AWSCLIV2.msi
2. Run the installer
3. Verify:
```powershell
aws --version
```

**Inside WSL:**
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

After installing, configure AWS credentials:
```bash
aws configure
```

Enter your:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-east-1`)
- Output format (`json`)

---

## Verifying Everything Works

After installation, run these checks:

**Check Terraform:**
```powershell
terraform version
```

**Quick initialization test:**
```powershell
mkdir C:\temp\tf-test
cd C:\temp\tf-test

# Create a simple config file
@"
terraform {
  required_version = ">= 1.0"
}
"@ | Out-File -FilePath main.tf -Encoding UTF8

terraform init
```

If `terraform init` completes successfully, your installation is working.

---

## Comparing Installation Methods on Windows

| Method | Ease | Auto-updates | Best For |
|---|---|---|---|
| **Chocolatey** | Easy | Yes (`choco upgrade`) | Windows power users |
| **winget** | Very easy | Yes (`winget upgrade`) | Quick setup, Windows 10/11 built-in |
| **Manual** | Medium | No (manual) | Specific version control |
| **WSL** | Medium setup, easy after | Yes (apt) | Serious DevOps work on Windows |

**Our recommendation:** If you plan to do serious DevOps work on Windows, set up WSL and do everything inside it. You will have a much smoother experience with all DevOps tools, not just Terraform.

---

## Common Issues on Windows

### Issue: "terraform is not recognized as an internal or external command"

**Cause:** Terraform's directory is not in your PATH.

**Fix:**
1. Find where terraform.exe is located
2. Add that directory to your System PATH (see Step 3 in the manual installation above)
3. Restart your terminal

### Issue: PowerShell execution policy error

```
File cannot be loaded because running scripts is disabled on this system
```

**Fix:**
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### Issue: "Access is denied" during installation

**Fix:** Run PowerShell or Command Prompt as Administrator (right-click → "Run as Administrator").

### Issue: Terraform works in Command Prompt but not in PowerShell (or vice versa)

**Fix:** Open a new terminal window after adding to PATH. Changes to environment variables don't apply to already-open terminal windows.

---

## Summary

- Windows has multiple good options for installing Terraform: Chocolatey, winget, and manual
- **Chocolatey** is the most convenient for Windows users who want easy updates
- **winget** is the simplest option since it is built into Windows 10/11
- **WSL (Windows Subsystem for Linux)** is the strongly recommended approach for serious DevOps work — it gives you a full Linux environment inside Windows
- Always verify installation with `terraform version`
- Install the AWS CLI alongside Terraform — you will need it for authentication in Chapter 6
- Use Windows Terminal for a better terminal experience on Windows

In the next chapter, we move to setting up providers — starting with how to configure and authenticate with AWS.
