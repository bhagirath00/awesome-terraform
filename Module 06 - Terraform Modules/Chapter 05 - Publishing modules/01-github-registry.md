# Automated Publishing: GitHub and the Terraform Registry

The Terraform Registry is designed to work in tandem with Version Control Systems (VCS), with GitHub being the most common integration. Rather than manually uploading files, you connect your GitHub repository to the registry, allowing for automated indexing and version management. This chapter covers the technical setup for connecting GitHub to the registry and using GitHub Actions to automate the quality assurance and publishing process.

## 1. Connecting the Github Repository

The integration between GitHub and the Terraform Public Registry uses OAuth for authentication and Webhooks for event notification.

### 1.1 The Connection Process
1. **Authentication:** Log into the Terraform Registry using your GitHub identity.
2. **Authorization:** Grant the registry permissions to read your public repositories and install a webhook.
3. **Selection:** Choose the repository that follows the `terraform-<PROVIDER>-<NAME>` naming convention.
4. **Indexing:** The registry will scan your Git tags and automatically create versions for any valid SemVer tag it finds.

## 2. CI/CD with GitHub Actions

Before a module is published, it should pass a battery of automated tests. GitHub Actions is the standard tool for this infrastructure CI/CD.

### 2.1 The Standard "Lint and Validate" Workflow
A professional module should run the following steps on every Pull Request:

```yaml
name: "Terraform Module CI"
on:
  pull_request:
    branches: [ main ]

jobs:
  terraform:
    name: "Validate"
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v2

      - name: Format Check
        run: terraform fmt -check

      - name: Initialize
        run: terraform init

      - name: Validate
        run: terraform validate
```

## 3. Advanced Quality Assurance Tools

Beyond standard Terraform commands, several third-party tools are commonly used in GitHub Actions for higher-quality modules.

### 3.1 TFLint
While `terraform validate` checks for syntax, `tflint` checks for cloud-specific errors, such as using an invalid EC2 instance type for a specific region.
- **Action:** `terraform-linters/tflint-load-config-action`

### 3.2 Checkov or Tfsec
Static Analysis Security Testing (SAST) tools that scan your HCL for security vulnerabilities, such as unencrypted S3 buckets or open SSH ports.
- **Action:** `bridgecrewio/checkov-action`

### 3.3 Terraform-docs
Automatically generates the variable and output tables in your README based on your code. This ensures your documentation never drifts from your actual `variables.tf`.
- **Action:** `terraform-docs/gh-actions`

## 4. Automating the Release Process

The Terraform Registry only creates a new version when it detects a new Git Tag. You can automate this using a combination of GitHub Releases and specialized actions.

### 4.1 Automated Tagging
Using the "Semantic Release" pattern, you can have GitHub automatically tag your repository with `v1.1.0` if the merged PR contains a "feat:" prefix in its commits.
- **Benefits:** Eliminates manual version tracking and ensures that the version number accurately reflects the code change (SemVer).

## 5. Public vs. Private Registries

### 5.1 Public Registry
Requires the repository to be public and follow strict naming rules. It is globally searchable and accessible by any Terraform user.

### 5.2 Private Registry (HCP Terraform)
If you are developing modules for internal corporate use, you can connect private GitHub repositories to a private registry within HashiCorp Cloud Platform (HCP) Terraform. 
- **Entrants:** Only authorized users within your organization can see or use these modules.
- **Flexible Naming:** Private modules do not always require the `terraform-<PROVIDER>-<NAME>` naming convention, though it is still recommended for consistency.

## 6. Webhooks and Instant Indexing

Once a repository is connected, GitHub sends a webhook payload to the registry every time a new Release or Tag is created.
- **Low Latency:** Within seconds of pushing a tag to GitHub, the new version will appear on the Terraform Registry page.
- **Debugging:** If a version fails to appear, check the "Webhooks" tab in your GitHub repository settings to see if the registry returned a 200 OK response.

## 7. Using GitHub Actions for Testing (Terratest)

For critical modules, you should perform "Integration Testing." This involves using a GitHub Action to:
1. Authenticate with AWS/Azure/GCP using OIDC (OpenID Connect).
2. Run `go test` using the `Terratest` library.
3. Physically provision the resources, check them, and then destroy them.

**Note:** This can be expensive and should only be run on merges to the main branch or specific "Release Candidate" tags.

## 8. Summary

The integration of GitHub into the Terraform ecosystem transforms infrastructure development into a first-class software engineering process. By leveraging GitHub Actions for linting, security scanning, and documentation generation, you ensure that every module published to the registry meets the highest standards of quality. The combination of automated Git tagging and registry webhooks creates a seamless "Push to Publish" workflow that allows your infrastructure components to scale as fast as your application code.
