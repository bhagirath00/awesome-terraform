# VS Code Setup for Terraform

## Introduction

Visual Studio Code (VS Code) is the most popular code editor for DevOps work, and it has excellent support for Terraform. Setting it up properly will make writing Terraform code significantly faster, catch errors before you even run a command, and generally make your daily work more comfortable.

This file covers:
- Installing VS Code
- Essential settings for Terraform development
- Recommended general extensions
- Workspace and project configuration

The Terraform-specific extension is covered in the next file (`terraform-extension.md`).

---

## Why VS Code for Terraform?

You can write Terraform in any text editor — Notepad, Vim, Nano, Sublime Text — and it will work. But VS Code gives you:

- **Syntax highlighting** — HCL code is colored by type, making it easy to read
- **Auto-completion** — suggests resource types, argument names, and values as you type
- **Error highlighting** — shows mistakes in red before you run any commands
- **Integrated terminal** — run `terraform` commands without leaving the editor
- **Git integration** — see changes, commit, and push without leaving the editor
- **Remote development** — write code locally while running it on a remote server or in WSL

---

## Installing VS Code

### On Linux

**Method 1: Snap (easy, auto-updates)**
```bash
sudo snap install code --classic
```

**Method 2: Official package repository (Debian/Ubuntu)**
```bash
# Import Microsoft GPG key
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg

# Add repository
sudo sh -c 'echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'

# Install
sudo apt-get update
sudo apt-get install code
```

**Method 3: RPM (CentOS/RHEL/Fedora)**
```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo sh -c 'echo -e "[code]\nname=Visual Studio Code\nbaseurl=https://packages.microsoft.com/yumrepos/vscode\nenabled=1\ngpgcheck=1\ngpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/vscode.repo'
sudo yum check-update
sudo yum install code
```

---

### On macOS

**Using Homebrew (recommended):**
```bash
brew install --cask visual-studio-code
```

**Manual installation:**
1. Go to https://code.visualstudio.com/
2. Click "Download for Mac"
3. Open the downloaded `.zip` file
4. Drag `Visual Studio Code.app` to your Applications folder

**Adding `code` command to PATH (macOS):**
After installing, open VS Code and press `Cmd + Shift + P` to open the Command Palette. Type "shell command" and select:
`Shell Command: Install 'code' command in PATH`

Now you can open VS Code from any directory in your terminal:
```bash
code .          # Open current directory
code main.tf    # Open a specific file
```

---

### On Windows

**Using winget:**
```powershell
winget install Microsoft.VisualStudioCode
```

**Manual installation:**
1. Go to https://code.visualstudio.com/
2. Click "Download for Windows"
3. Run the installer
4. During installation, check "Add to PATH" to enable the `code` command in terminals

---

## Initial VS Code Configuration

### Opening the Settings

You can configure VS Code through:
- **GUI:** `File → Preferences → Settings` (or `Code → Settings` on Mac)
- **JSON file:** Press `Ctrl+Shift+P` (or `Cmd+Shift+P` on Mac), type "Open User Settings JSON"

The JSON file approach gives you more control and is easier to share with your team.

---

### Essential Settings for Terraform Development

Open your `settings.json` (User Settings) and add these settings:

```json
{
  // Editor basics
  "editor.fontSize": 14,
  "editor.tabSize": 2,
  "editor.insertSpaces": true,
  "editor.wordWrap": "on",
  "editor.minimap.enabled": true,
  "editor.renderWhitespace": "boundary",
  "editor.rulers": [120],

  // Auto-save (saves as you type — very helpful)
  "files.autoSave": "afterDelay",
  "files.autoSaveDelay": 1000,

  // Trim trailing whitespace on save
  "files.trimTrailingWhitespace": true,
  "files.insertFinalNewline": true,

  // Terraform-specific formatting
  "[terraform]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true,
    "editor.tabSize": 2
  },
  "[terraform-vars]": {
    "editor.defaultFormatter": "hashicorp.terraform",
    "editor.formatOnSave": true
  },

  // Terminal settings
  "terminal.integrated.fontSize": 13,
  "terminal.integrated.defaultProfile.linux": "bash",
  "terminal.integrated.defaultProfile.osx": "zsh",

  // Git settings
  "git.autofetch": true,
  "git.confirmSync": false,

  // File explorer
  "explorer.confirmDelete": true,
  "explorer.confirmDragAndDrop": false,

  // Search exclusions (ignore Terraform working directories)
  "search.exclude": {
    "**/.terraform": true,
    "**/*.tfstate": true,
    "**/*.tfstate.backup": true
  },

  // Files to hide in explorer (but still accessible)
  "files.exclude": {
    "**/.terraform": true
  }
}
```

---

### Understanding the Key Settings

**`"editor.formatOnSave": true` (under `[terraform]`):**
This is the most important setting for Terraform work. Every time you save a `.tf` file, VS Code automatically runs `terraform fmt` on it. Your code is always properly formatted without you having to remember to run the command.

**`"editor.tabSize": 2`:**
Terraform uses 2-space indentation. Setting this ensures your Tab key inserts 2 spaces, matching the Terraform standard.

**`"editor.insertSpaces": true`:**
Inserts spaces when you press Tab, not actual tab characters. Terraform's formatter expects spaces.

**`"files.trimTrailingWhitespace": true`:**
Removes trailing spaces at the end of lines automatically. Keeps your code clean.

**`"search.exclude"` with `.terraform` and `*.tfstate`:**
Excludes the `.terraform` directory and state files from VS Code's search. Without this, searches would return results from inside provider binaries, which is never useful.

---

## The Integrated Terminal

One of VS Code's most useful features is the integrated terminal. Instead of switching between VS Code and a separate terminal window, you can run Terraform commands directly inside VS Code.

### Opening the Terminal

- **Keyboard shortcut:** `Ctrl + `` ` (backtick) on all platforms
- **Menu:** `View → Terminal`
- **Command Palette:** `Ctrl+Shift+P` → "Toggle Integrated Terminal"

### Using Multiple Terminal Panels

You can have multiple terminals open at once:
- Click the `+` button in the terminal panel to open a new terminal
- Use the dropdown to switch between terminals
- Split the terminal with the split button to see two terminals side by side

This is useful when you want to have one terminal running a long `terraform apply` while you continue editing in another.

### Running Terraform Commands in the Terminal

```bash
# The terminal opens in your project directory automatically
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

---

## File Explorer and Project Organization

### Workspace Setup

For Terraform projects, open the entire project folder in VS Code:

```bash
# Open VS Code in the current directory
code .
```

VS Code will show all your `.tf` files in the Explorer sidebar on the left. You can click any file to open it.

### Multi-Root Workspaces

If you work on multiple Terraform projects at once, you can create a **workspace file** that includes all of them:

1. Open each project folder: `File → Add Folder to Workspace`
2. Save the workspace: `File → Save Workspace As`

This creates a `.code-workspace` file. Open it to get all your projects in one VS Code window.

Example `.code-workspace` file:
```json
{
  "folders": [
    {
      "name": "terraform-networking",
      "path": "../terraform-networking"
    },
    {
      "name": "terraform-compute",
      "path": "../terraform-compute"
    },
    {
      "name": "terraform-databases",
      "path": "../terraform-databases"
    }
  ],
  "settings": {
    "[terraform]": {
      "editor.formatOnSave": true
    }
  }
}
```

---

## Useful VS Code Keyboard Shortcuts for Terraform Work

Learning these shortcuts will speed up your work considerably.

| Action | Windows/Linux | macOS |
|---|---|---|
| Open file by name | `Ctrl+P` | `Cmd+P` |
| Command palette | `Ctrl+Shift+P` | `Cmd+Shift+P` |
| Open terminal | `Ctrl+`` ` | `Ctrl+`` ` |
| Open settings | `Ctrl+,` | `Cmd+,` |
| Find in file | `Ctrl+F` | `Cmd+F` |
| Find in all files | `Ctrl+Shift+F` | `Cmd+Shift+F` |
| Go to definition | `F12` | `F12` |
| Find all references | `Shift+F12` | `Shift+F12` |
| Format document | `Shift+Alt+F` | `Shift+Option+F` |
| Toggle sidebar | `Ctrl+B` | `Cmd+B` |
| Split editor | `Ctrl+\` | `Cmd+\` |
| Close editor tab | `Ctrl+W` | `Cmd+W` |
| Save file | `Ctrl+S` | `Cmd+S` |
| Multi-cursor | `Alt+Click` | `Option+Click` |
| Select all occurrences | `Ctrl+Shift+L` | `Cmd+Shift+L` |
| Comment line | `Ctrl+/` | `Cmd+/` |

---

## Recommended General Extensions

Beyond the Terraform-specific extension (covered in the next file), here are general extensions that improve the Terraform development experience:

### GitLens

Shows Git blame information, commit history, and file changes directly in the editor.

```
Extension ID: eamodio.gitlens
```

With GitLens, you can hover over any line and see who last changed it, when, and in which commit.

### Git Graph

Shows a visual graph of your Git history. Useful for understanding branch structure and commits.

```
Extension ID: mhutchie.git-graph
```

### DotENV

Syntax highlighting for `.env` files. Useful when you store Terraform variable values in `.env` files.

```
Extension ID: mikestead.dotenv
```

### YAML

Better YAML support — useful since Terraform projects often include GitHub Actions workflows, Kubernetes configs, and other YAML files.

```
Extension ID: redhat.vscode-yaml
```

### Better Comments

Allows you to write color-coded comments:

```hcl
# ! This is a warning comment (red)
# ? This is a question comment (blue)
# TODO: This is a todo comment (orange)
# * This is a highlighted comment (green)
```

```
Extension ID: aaron-bond.better-comments
```

### indent-rainbow

Colors different levels of indentation. Makes nested HCL blocks easier to read.

```
Extension ID: oderwat.indent-rainbow
```

---

## Installing Extensions

**Method 1: GUI (easiest)**
1. Click the Extensions icon in the left sidebar (or press `Ctrl+Shift+X`)
2. Search for the extension name
3. Click "Install"

**Method 2: Command line**
```bash
code --install-extension hashicorp.terraform
code --install-extension eamodio.gitlens
code --install-extension mhutchie.git-graph
code --install-extension redhat.vscode-yaml
```

**Sharing extensions with your team:**
Create a `.vscode/extensions.json` file in your project:

```json
{
  "recommendations": [
    "hashicorp.terraform",
    "eamodio.gitlens",
    "mhutchie.git-graph",
    "redhat.vscode-yaml",
    "aaron-bond.better-comments"
  ]
}
```

When a team member opens the project, VS Code automatically suggests installing these extensions.

---

## VS Code with WSL (Windows Users)

If you are using WSL on Windows, VS Code has excellent WSL integration through the **Remote - WSL** extension.

Install it:
```bash
code --install-extension ms-vscode-remote.remote-wsl
```

Or from WSL terminal, open any project in VS Code:
```bash
code .
```

VS Code opens on Windows but connects to WSL — you get a Windows GUI with Linux backend. All Terraform commands run inside WSL, all files are on the Linux filesystem, but the editor runs natively on Windows with full speed.

---

## Summary

- VS Code is the most popular editor for Terraform work and has excellent support through the HashiCorp extension
- Install VS Code using the package manager for your OS: Snap/apt on Linux, Homebrew on macOS, winget on Windows
- Configure `editor.formatOnSave: true` for Terraform files — this runs `terraform fmt` automatically on every save
- The integrated terminal lets you run Terraform commands without leaving the editor
- Essential settings include 2-space tab size, inserting spaces (not tabs), and excluding `.terraform` directories from search
- Recommended general extensions: GitLens, Git Graph, YAML, Better Comments, indent-rainbow
- Share your extension recommendations with your team through `.vscode/extensions.json`
- Windows users should use the Remote-WSL extension to connect VS Code to their WSL environment

In the next file, we cover the HashiCorp Terraform extension in detail — all its features and how to get the most out of it.
