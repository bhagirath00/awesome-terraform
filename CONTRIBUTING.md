# Contributing to Awesome Terraform

Thank you for your interest in contributing to **Awesome Terraform**! We welcome contributions, bug fixes, enhancements, and architectural best practices from the community.

---

## How to Contribute

### Reporting Issues
If you identify an error in HCL configurations, deprecated provider syntax, broken links, or have suggestions for new architecture patterns:
1. Check existing issues to see if the topic has already been reported.
2. Open a new issue with a clear, descriptive title and explanation.
3. Include your Terraform version (e.g. 1.7+), provider versions (e.g. AWS 5.x), and steps to reproduce.

### Making Changes

1. **Fork** the repository to your GitHub account.
2. **Clone** your fork locally:
   ```bash
   git clone https://github.com/bhagirath00/awesome-terraform.git
   cd awesome-terraform
   ```
3. Create a feature **branch** for your changes:
   ```bash
   git checkout -b feature/your-feature-name
   ```
4. Make your changes adhering to the style guide below.
5. **Commit** your changes:
   ```bash
   git commit -m "feat(module-08): add aws transit gateway module examples"
   ```
6. **Push** to your fork:
   ```bash
   git push origin feature/your-feature-name
   ```
7. Open a **Pull Request** against the `main` branch.

---

## Style Guide

### HCL Code & Terraform Best Practices
- **Canonical Formatting**: Run `terraform fmt -recursive` on all `.tf` code blocks.
- **Validation**: Validate code blocks with `terraform validate` where feasible.
- **Provider Pinning**: Use standard version constraints (`~> 5.0`) in `required_providers`.
- **Sensitive Data**: Mark passwords, private keys, and tokens with `sensitive = true`. Never commit real credentials or `.tfstate` files.
- **Naming Conventions**: Use lowercase, underscore-separated names for Terraform identifiers (e.g., `resource "aws_vpc" "production_vpc"`).

### Documentation & Guides
- Use clear, professional, technical English.
- Include working CLI command snippets and sample expected plan/apply outputs.
- Include ASCII architectural flow diagrams or state progression graphs where helpful.
- Reference official HashiCorp documentation ([developer.hashicorp.com/terraform](https://developer.hashicorp.com/terraform)).
- Maintain sequential chapter numbering (`01-`, `02-`, etc.) to preserve directory sorting.

### Commit Messages
We follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:
- `feat:` – New chapter, guide, or feature addition
- `fix:` – Bug fix, HCL correction, or typo fix
- `docs:` – Documentation updates or formatting adjustments
- `refactor:` – Reorganizing chapters, updating naming, or streamlining code blocks

---

## Questions & Support

Feel free to open a GitHub Discussion or Issue for any questions.

Thank you for contributing to **Awesome Terraform**!
