# The Human in the Loop: The Plan and Apply Workflow

A professional Infrastructure as Code (IaC) lifecycle should almost never use "Auto-Approve" in a production environment. Unlike application code—where a failed deploy can be reverted with a single command—infrastructure changes often involve the destruction of data volumes or network foundations that are difficult to recover. The "Plan and Apply" workflow institutionalizes a peer-review step, ensuring that every cloud modification is vetted by a human operator before it is executed. This chapter provide a definitive deep dive into the architecture of multi-stage pipelines, the use of GitHub Environments for approvals, and the technical patterns for artifact-based deployments.

## 1. The Separation of Concerns

The "Plan and Apply" workflow is divided into two distinct operational phases.

### 1.1 The Plan Phase (Continuous Integration)
Triggered by a Pull Request. Its goal is to provide a "Preview" of the impact.
- **Artifact:** The resulting binary `tfplan` file is uploaded as a build artifact.
- **Visibility:** The text summary (e.g., "3 to add, 1 to change, 0 to destroy") is posted as a PR comment.

### 1.2 The Apply Phase (Continuous Deployment)
Triggered by the merging of the PR. Its goal is to make the code a reality.
- **Requirement:** It must use the EXACT `tfplan` artifact generated in the previous step to prevent race conditions.

## 2. Implementing Approvals with GitHub Environments

GitHub Environments allow you to define "Protection Rules" that stop a workflow until a specific user or team clicks "Approve."

```yaml
jobs:
  apply:
    name: 'Terraform Apply'
    runs-on: ubuntu-latest
    environment: production # This triggers the approval requirement
    if: github.ref == 'refs/heads/main'
    
    steps:
      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: tfplan

      - name: Terraform Apply
        run: terraform apply tfplan
```

## 3. The Technical Safety of Artifacts

Why do we download a `tfplan` file instead of just running `terraform apply`?

- **The Problem:** In a high-velocity team, Engineer A might plan and approve. In the five minutes before they merge, Engineer B might manually change something in the cloud. If Engineer A just runs `terraform apply`, the results might be different from the plan they reviewed.
- **The Solution:** By applying the binary `tfplan` file, Terraform verifies that the cloud state hasn't changed since the plan was created. If a drift is detected, the apply will fail safely rather than executing an stale plan.

## 4. Post-Apply Verification and Testing

A truly professional pipeline doesn't end when Terraform says "Apply Complete." 
- **Smoke Tests:** Running a script to curl the health endpoint of the newly created Load Balancer.
- **Compliance Checks:** Running a tool like `inspec` or `terratest` to verify that the provisioned infrastructure actually meets the security requirements (e.g., checking if the database is truly private).

## 5. Concurrency Control

In GitHub Actions, you should use the `concurrency` property to ensure that only one "Apply" or "Plan" runs at a time for a specific environment.

```yaml
concurrency:
  group: terraform-${{ github.head_ref || github.run_id }}
  cancel-in-progress: true
```
This prevents the dreaded "State Lock" error that occurs when two pipelines attempt to write to the state file simultaneously.

## 6. Summary

The "Plan and Apply" workflow is the ultimate expression of "Change Management" in the cloud era. By strictly separating the generation of infrastructure plans from their execution, and by mandating human approval for production changes, you create a system that is both highly automated and highly reliable. Terraform's support for binary plan artifacts and GitHub's environmental protection rules provide the technical framework needed to manage complex, mission-critical cloud foundations with minimal risk and maximum transparency. Mastering this workflow is the hallmark of a senior DevOps engineer capable of governing global infrastructure at enterprise scale.
