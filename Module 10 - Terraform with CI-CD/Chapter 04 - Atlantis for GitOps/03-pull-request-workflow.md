# Collaborative Infrastructure: The Atlantis Pull Request Workflow

The Pull Request (PR) workflow is the primary interface for infrastructure management when using Atlantis. By shifting the execution of Terraform from the developer's laptop to a centralized, comment-driven system, Atlantis ensures that every change is transparent, peer-reviewed, and perfectly documented. In a professional DevOps environment, the PR serves as the "Audit Log" for the cloud foundation. This chapter provides a definitive deep dive into the day-to-day lifecycle of an infrastructure change, focusing on comment-based commands, locking mechanisms, and the final application of changes from the Git interface.

## 1. Step 1: The Initial Contribution

The workflow begins when a developer creates a new branch and pushes HCL changes to the remote repository.

- **Automatic Trigger:** As soon as the Pull Request is opened, Atlantis detects the changes in the Terraform directories.
- **Autoplan:** If configured, Atlantis automatically runs `terraform plan` for all modified projects and posts the output as a comment on the PR. This immediate feedback allows the developer to catch syntax errors or unintended impacts before any human reviewer even looks at the code.

## 2. Step 2: Comment-Driven Iteration

If the initial plan isn't quite right, or if the developer needs to pass specific variables, they can interact with Atlantis using special comments.

- **`atlantis plan`**: Re-runs the plan for all projects.
- **`atlantis plan -p networking-prod`**: Re-runs the plan only for a specific project.
- **`atlantis plan -- -var="size=large"`**: Passes extra arguments directly to the Terraform CLI.

**The Benefit:** This interaction is visible to everyone on the PR. Reviewers can see exactly how the plan evolved as the developer tuned the configuration.

## 3. Step 3: The PR Lock

When a plan is generated, Atlantis "Locks" that specific directory/project. 
- **Visualization:** A comment is posted stating: "Project networking-prod is locked by PR #123."
- **Conflict Prevention:** If another developer tries to open a PR modifying the same directory, Atlantis will refuse to run a plan, preventing two different PRs from trying to modify the same cloud resources simultaneously. This solves the "Merge Conflict" problem at the infrastructure level.

## 4. Step 4: Review and Approval

Atlantis can be configured to enforce strict governance rules before any changes are allowed.

- **Required Approvals:** The `atlantis apply` command can be blocked unless the PR has the required number of "Approve" votes from the team (using GitHub/GitLab native review features).
- **CI Success:** You can mandate that all other CI checks (e.g., unit tests or linting) must be green before an apply is permitted.

## 5. Step 5: The Execution (`atlantis apply`)

Once the code is reviewed and approved, the authorized engineer types the final command in the comment thread.

- **Command:** `atlantis apply`
- **Verification:** Atlantis performs a "Plan" one last time to ensure no drift has occurred, and then executes `terraform apply`.
- **Result:** The execution logs are posted back to the PR. If successful, Atlantis will automatically merge the PR (if configured) and delete the branch, ensuring that the "Main" branch always reflects the actual state of the cloud.

## 6. Handling Failures and Unlocking

If an apply fails (e.g., due to an AWS service being down), Atlantis keeps the lock in place to prevent others from making it worse.
- **Unlocking:** An administrator can type `atlantis unlock` or use the Atlantis UI to release the lock once the issue has been manually resolved.

## 7. Summary

The Atlantis Pull Request workflow is the gold standard for collaborative infrastructure engineering. By centralizing operations and using the PR as the single interface for planning, reviewing, and applying changes, you eliminate the fragmentation and risk of manual CLI-based deployments. Terraform's integration with Atlantis transforms the infrastructure team into a highly efficient, transparent, and audited unit, capable of managing complex global foundations with the same precision and peer-review rigor as modern software development teams.
