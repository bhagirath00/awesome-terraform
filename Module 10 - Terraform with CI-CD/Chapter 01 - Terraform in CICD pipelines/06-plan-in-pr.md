# Visibility by Design: Rendering Terraform Plans in Pull Requests

The primary interface for infrastructure review in a modern DevOps organization is the Pull Request (PR). However, if the results of a `terraform plan` are hidden inside the logs of a CI/CD runner, reviewers are forced to switch contexts and dig through build logs to understand the impact of a code change. A professional pipeline eliminates this friction by automatically posting the execution plan directly as a comment into the PR. This chapter provide a definitive deep dive into the technical implementation of PR commenting, the use of collapsible markdown tags for readability, and the patterns for providing status-check feedback to the engineering team.

## 1. The Value of Contextual Feedback

When a developer modifies a Security Group or an IAM Policy, the risk is high. 
- **The Old Way:** Reviewer looks at the code, tries to mentally simulate the "Apply," and hopes for the best.
- **The Professional Way:** Reviewer looks at the code AND the plan output side-by-side. They see exactly which 3 rules are being added and which specific role is being deleted, all within the GitHub/GitLab interface.

## 2. Technical Implementation in GitHub Actions

To render a plan in a PR, you must capture the Terraform output and use the GitHub API to post it.

### 2.1 Capturing the Output
You must run Terraform with `-no-color` to ensure the output is pure text without ANSI escape codes, which would break the markdown rendering.

```bash
terraform plan -no-color -out=tfplan > plan.txt
```

### 2.2 Posting via `github-script`
Using the `actions/github-script` action allows you to interact with the PR metadata safely.

```yaml
- name: Post Plan to PR
  uses: actions/github-script@v6
  with:
    script: |
      const fs = require('fs');
      const plan = fs.readFileSync('plan.txt', 'utf8');
      const output = `#### Terraform Plan Summary 📖
      
      <details><summary>Show Detailed Plan</summary>
      
      \`\`\`hcl
      ${plan}
      \`\`\`
      
      </details>
      
      *Pushed by: @${context.actor}, Environment: Production*`;
      
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: output
      })
```

## 3. The "Collapsible" UI Pattern

Terraform plans can be thousands of lines long. Posting the full text directly would make the PR conversation unreadable.
- **The Solution:** Always wrap the plan output in HTML `<details>` and `<summary>` tags. This provides a clean interface where the reviewer only expands the plan when they are ready to analyze the details.

## 4. Status Checks and Gatekeeping

In addition to the comment, the pipeline should report a "Status Check" to the PR.
- **Pass:** The code is valid and the plan generated successfully.
- **Fail:** There is a syntax error or a provider conflict. 
By making the "Terraform Plan" check mandatory for merging, you ensure that no unvalidated code ever reaches the main branch.

## 5. Security Warning: Masking Sensitive Data

Even though you have the `sensitive` flag enabled, it is a professional best practice to ensure your CI/CD service role doesn't have permissions to print "Secrets" into the plan output. Always verify that your PR posting script is not accidentally bypassing your masking layer.

## 6. Summary

Rendering Terraform plans in Pull Requests is the single most effective way to improve the quality of infrastructure reviews. By bringing the cloud reality directly into the developer's discussion thread, you foster a culture of transparency, accountability, and collective ownership of the production environment. Terraform's machine-readable output and the flexibility of modern CI/CD scripts make this integration simple to implement yet incredibly powerful in its impact on organizational velocity and safety. Mastering these visibility patterns is a mandatory requirement for any engineer responsible for the maturity of an Infrastructure as Code ecosystem.
