# Remote Configuration: The `remote-exec` Provisioner

While `local-exec` runs commands on the machine where Terraform is executed, `remote-exec` executes commands on the newly created resource itself. This is typically achieved over SSH (for Linux) or WinRM (for Windows). It is a powerful tool for initial bootstrapping, service configuration, and smoke testing of virtual machines. This chapter covers the technical implementation, connection management, and security considerations of remote provisioning.

## 1. Requirement: The `connection` Block

Before a `remote-exec` provisioner can run, Terraform must know how to reach the machine. To do this, you must define a `connection` block at the resource level or inside the provisioner block.

```hcl
resource "aws_instance" "web" {
  # ... configuration ...

  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }

  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo amazon-linux-extras install nginx1 -y",
      "sudo systemctl start nginx"
    ]
  }
}
```

### 1.1 Connection Attributes
- **`type`**: Usually `ssh` or `winrm`.
- **`user`**: The username to log in with (depends on the OS image).
- **`host`**: The IP address or DNS name. Using `self.public_ip` or `self.private_ip` is standard practice.
- **`private_key`**: The content of the private key (often read using the `file()` function).

## 2. Syntax for `remote-exec`

There are two primary ways to specify what should be run:

### 2.1 Inline Commands (`inline`)
A list of command strings. Terraform runs them sequentially. This is best for simple, one-line operations like starting a service or creating a directory.

### 2.2 External Scripts (`scripts` and `script`)
- **`script`**: Takes a path to a single local script file. Terraform uploads it to the remote machine and executes it.
- **`scripts`**: Takes a list of paths to multiple script files to be executed in order.

## 3. Working with Bastion Hosts (Jump Boxes)

In secure environments, resources are often placed in private subnets with no direct internet access. To provision these, you must use a Bastion host. Terraform handles this natively within the `connection` block.

```hcl
connection {
  type        = "ssh"
  user        = "ec2-user"
  private_key = file("~/.ssh/id_rsa")
  host        = self.private_ip

  # Bastion configuration
  bastion_host        = "bastion.example.com"
  bastion_user        = "admin"
  bastion_private_key = file("~/.ssh/bastion_key")
}
```

## 4. Bootstrapping Scenarios

Professional DevOps teams use `remote-exec` for several critical tasks:
1. **Adding Nodes to a Cluster:** Running a `kubeadm join` command for a new Kubernetes worker node.
2. **Mounting Storage:** Running `mkfs` and `mount` commands for secondary EBS or Azure Disk volumes.
3. **Log Shipper Setup:** Installing and configuring agents like FluentBit or Datadog.
4. **Environment Hardening:** Disabling root login or applying CIS benchmarks via script.

## 5. Handling Failures and Timeouts

Remote execution is susceptible to network latency and service initialization delays.
- **Timeouts:** You can specify a `timeout` in the `connection` block (e.g., `timeout = "5m"`). If the machine isn't reachable within that window, Terraform fails.
- **Retries:** Terraform will attempt to connect several times before giving up.
- **Taint:** Just like `local-exec`, a non-zero exit code from a `remote-exec` script will taint the resource, causing it to be replaced on the next run.

## 6. Security Challenges

Using `remote-exec` introduces significant security overhead:
- **Keys in State:** The private key content, if passed as a variable or read from a file, may end up in the `terraform.tfstate` file in plain text.
- **SSH Connectivity:** You must open port 22 in security groups/firewalls, even if only temporarily, which increases the attack surface.
- **Credential Management:** Managing SSH keys across large teams and various environments is a complex operational task.

## 7. `remote-exec` vs. Cloud-Init (User Data)

One of the most common debates is whether to use `remote-exec` or a provider's native "bootstrapping" feature (like `user_data` in AWS or `custom_data` in Azure).

| Feature | `remote-exec` | Cloud-Init / User Data |
| :--- | :--- | :--- |
| **Execution** | Via SSH after boot. | Via cloud agent during boot. |
| **Visibility** | Terraform shows logs in real-time. | Logs are internal to the VM. |
| **Feedback Loop** | Terraform fails if script fails. | Resource is created even if script fails. |
| **Connectivity** | Requires SSH access. | No SSH access required. |

**Best Practice:** Use Cloud-Init for low-level OS configuration and `remote-exec` for application-specific logic that needs real-time feedback and validation within the Terraform workflow.

## 8. Summary

The `remote-exec` provisioner is a sophisticated tool for direct interaction with virtual resources. It allows for an interactive provisioning experience and precise error handling. However, due to its dependency on SSH/WinRM connectivity and its security implications, it should be used judiciously, often as a temporary bridge until more robust configuration management (like Ansible or Packer-baked images) is implemented.
