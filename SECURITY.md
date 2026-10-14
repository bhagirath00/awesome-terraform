# Security Policy

## Content Security & Attribution

This repository is designed for educational and professional reference purposes. The architectural guidance, HCL code snippets, and documentation provided here are compiled in accordance with industry best practices and official documentation.

We acknowledge and credit the following foundational references:
- **HashiCorp Terraform Documentation**: [developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform)
- **Terraform Registry**: [registry.terraform.io](https://registry.terraform.io/)
- **CIS Benchmarks for Cloud Providers**: [cisecurity.org](https://www.cisecurity.org/benchmark)

---

## Educational & Production Use

- Code snippets, CLI commands, and HCL configurations are intended for learning, experimentation, and testing in sandbox cloud environments.
- **State File Security**: Terraform state files (`.tfstate`) can contain sensitive values in plaintext (passwords, private keys, database connection strings). Always store production state in encrypted remote backends (e.g., AWS S3 with KMS encryption and DynamoDB state locking).
- Never commit real cloud access keys (`AWS_ACCESS_KEY_ID`), secrets, or `.tfvars` files with production values to version control.

---

## Supported Versions

We track actively maintained, modern Terraform releases:

| Terraform Version | Support Level |
| :--- | :--- |
| **1.8.x** | :white_check_mark: Supported |
| **1.7.x** | :white_check_mark: Supported |
| **1.6.x** | :white_check_mark: Supported |
| **< 1.5** | :x: Deprecated (Syntactic differences may apply) |

---

## Reporting Vulnerabilities & Content Issues

### Content & Configuration Inaccuracies
If you find any inaccuracies, broken links, deprecated provider arguments, or configuration errors in the documentation, please:
1. Open an issue on GitHub with a clear description and reproduction steps.
2. Submit a Pull Request following our [Contributing Guidelines](CONTRIBUTING.md).

### Security Vulnerabilities
If you discover a security vulnerability (such as exposed credentials, security anti-patterns in example code, or critical security flaws):
1. **Do NOT open a public issue.**
2. Send a report via GitHub Private Vulnerability Reporting or contact the maintainers directly.
3. Include details of the vulnerability, affected files, and suggested remediation steps.
4. Maintainers will review the report and coordinate a fix promptly.
