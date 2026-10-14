# Cloud-Native Automation: GitHub Actions with Terraform

GitHub Actions has emerged as the preferred CI/CD platform for Terraform due to its deep integration with the source code repository and its powerful event-driven architecture. By using GitHub Actions, you can trigger infrastructure plans on every Pull Request and execute updates automatically upon a merge to the main branch. In a professional DevOps environment, this "Git-centric" approach ensures that infrastructure changes are treated with the same rigor as application code. This chapter provides a definitive deep dive into the structure of a GitHub Actions YAML workflow, the use of official HashiCorp actions, and the patterns for building a secure, automated deployment pipeline.

## 1. Technical Anatomy of a GitHub Actions Workflow

A Terraform workflow in GitHub is defined in a YAML file located in `.github/workflows/`. It consists of "Triggers," "Jobs," and "Steps."

### 1.1 The Triggers (Events)
A professional pipeline should distinguish between code reviews and physical deployments.
- **On Pull Request:** Triggers `terraform plan`.
- **On Push to Main:** Triggers `terraform apply`.

```yaml
on:
  pull_request:
    branches: [ main ]
  push:
    branches: [ main ]
```

## 2. The Job Environment and Runner Strategy

### 2.1 GitHub-Hosted vs. Self-Hosted Runners
- **GitHub-Hosted:** Convenient and maintenance-free. Best for standard deployments.
- **Self-Hosted:** Runners located inside your own AWS VPC. Best for projects that require private network access to internal resources or stricter security auditing.

### 2.2 Official HashiCorp Actions
Always use the verified HashiCorp actions to ensure compatibility and security.
- `hashicorp/setup-terraform`: Installs the Terraform CLI and configures authentications.

## 3. Implementation: A Production-Grade Workflow

A professional workflow should implement a "Fail Fast" strategy.

```yaml
name: 'Terraform-CI-CD'

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest
    
    defaults:
      run:
        shell: bash

    steps:
    # 1. Checkout the repository
    - name: Checkout
      uses: actions/checkout@v3

    # 2. Setup Terraform CLI
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        cli_config_token: ${{ secrets.TF_API_TOKEN }}

    # 3. Initialization
    - name: Terraform Init
      run: terraform init

    # 4. Format Check
    - name: Terraform Format
      run: terraform fmt -check

    # 5. The Plan (on PR)
    - name: Terraform Plan
      if: github.event_name == 'pull_request'
      run: terraform plan -no-color -input=false

    # 6. The Apply (on Merge/Push)
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve -input=false
```

## 4. OIDC Authentication: The Security Standard

The old way of securing GitHub Actions was by saving `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` as GitHub Secrets. 

**The Modern Solution: OpenID Connect (OIDC)**
OIDC allows GitHub Actions to request temporary, short-lived credentials from AWS without needing any permanent secrets.
- **How it works:** You create an IAM Role in AWS with a "Trust Relationship" that trusts the GitHub Actions OIDC provider. When the workflow runs, it "Assumes" the role.
- **Terraform resource:** Use the `aws_iam_openid_connect_provider` and `aws_iam_role` with specific `token.actions.githubusercontent.com` claims.

## 5. Commenting Plans on Pull Requests

To improve the developer experience (DX), a professional pipeline should post the output of `terraform plan` as a comment directly on the Pull Request.

```yaml
- name: Post Plan to PR
  uses: actions/github-script@v6
  if: github.event_name == 'pull_request'
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
    script: |
      const output = `#### Terraform Plan 📖\`${{ steps.plan.outcome }}\`
      <details><summary>Show Plan</summary>
      \`\`\`\n
      ${process.env.PLAN}
      \n\`\`\`
      </details>`;
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: output
      })
```

## 6. Matrix Builds: Multi-Environment Parallelism

If you manage multiple AWS regions or environments, you can use the `matrix` strategy to run Terraform in parallel across all of them.

```yaml
strategy:
  matrix:
    environment: [dev, staging, prod]
    region: [us-east-1, us-west-2]
```
This reduces the total deployment time and ensures that all environments are validated against the same code commit simultaneously.

## 7. Summary

GitHub Actions is more than just a task runner; it is a powerful platform for implementing "GitOps" principles. By combining event-driven triggers, secure OIDC authentication, and automated PR commenting, you transform your infrastructure development into a social, collaborative, and highly audited process. Terraform's deep compatibility with GitHub allows teams to scale their cloud operations with confidence, ensuring that every infrastructure change is visible, vetted, and perfectly executed in a clean, isolated environment.
