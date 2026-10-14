# Managing Brownfield Infrastructure: terraform import

The `terraform import` command is the historical mechanism for bringing existing, manually created infrastructure under the management of Terraform. In most real-world scenarios, DevOps engineers don't start with a clean slate; they inherit "Brownfield" environments where VPCs, databases, and users were provisioned via the AWS Console. To achieve the benefits of version control and automation, these resources must be imported into the Terraform state file. This chapter provide a definitive deep dive into the terminal-based import workflow, the "State-Code Alignment" challenge, and the professional strategies for safe resource adoption.

## 1. Technical Mechanics: How CLI Import Works

`terraform import` is a CLI-only command that modifies your `terraform.tfstate` file directly. It does NOT generate HCL code.

```bash
terraform import aws_instance.web i-1234567890abcdef0
```

### 1.1 The Handshake
1.  **Preparation:** You must first create an empty resource block in your HCL code that matches the type and name you intend to import.
    ```hcl
    resource "aws_instance" "web" {}
    ```
2.  **Execution:** You run the import command, providing the resource address and the unique ID (e.g., instance ID, bucket name, or VPC ID).
3.  **Result:** Terraform queries the AWS API, fetches all attributes of the resource, and saves them into the state file under the address `aws_instance.web`.

## 2. The Challenge: State-Code Alignment

Once the resource is in the state, your work is only half finished. If you run `terraform plan` now, it will fail or show a massive destructive change because your HCL code block (`resource "aws_instance" "web" {}`) is empty, while the state contains dozens of attributes.

**The Iterative Alignment Process:**
1.  **Examine State:** Use `terraform show` or view the JSON state file to see the imported attributes.
2.  **Author HCL:** Copy the critical attributes (AMI, instance type, tags) from the state into your HCL block.
3.  **Refine Plan:** Run `terraform plan` repeatedly. Your goal is to reach a **"No-op" state**, where Terraform reports: `No changes. Your infrastructure matches the configuration.`

## 3. High-Value Import Scenarios

- **Root VPCs:** Importing the default VPC or a manually built core network to enable VPC Peering or endpoint configurations.
- **Legacy Databases:** Bringing long-running RDS instances under Terraform to manage their security groups and parameters safely.
- **S3 Data Lakes:** Importing petabyte-scale buckets to enforce lifecycle rules and public access blocks across a multi-account organization.

## 4. Operational Risks and Limitations

- **Resource Destabilization:** If you accidentally map the wrong ID to the wrong address, you can corrupt your state file. Always perform a backup of your `terraform.tfstate` before running any import command.
- **Manual Toil:** CLI import is notoriously tedious for large environments (e.g., importing 100 subnets one by one). 
- **Hidden Attributes:** Some provider attributes are "Computed" and cannot be hardcoded in HCL. Identifying these during the alignment process requires significant provider-specific knowledge.

## 5. The Evolution: Toward Declarative Import

While the CLI command is still functional, the introduction of the `import` block in Terraform 1.5 has revolutionized this workflow by making it declarative and capable of automatic code generation. (This is explored in the next chapter).

## 6. Summary

The `terraform import` CLI command is the foundational tool for infrastructure migration. By mastering the delicate balance of state updates and HCL alignment, you can transform a legacy, manually managed environment into a modern, automated system. While the manual authoring of attributes remains a challenge, the ability to adopt existing resources without triggering downtime is an essential skill for any DevOps professional responsible for the long-term governance of enterprise cloud environments.
