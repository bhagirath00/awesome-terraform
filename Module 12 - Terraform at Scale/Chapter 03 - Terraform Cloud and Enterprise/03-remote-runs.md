# Decoupling Execution: The Remote Runs Model

A "Remote Run" is the execution of Terraform commands on a managed runner (in Terraform Cloud or Terraform Enterprise) instead of on a developer's local workstation. This is the cornerstone of professional Infrastructure as Code (IaC) governance. By moving execution to a centralized platform, you eliminate the "It works on my machine" problem, ensure that all plans are peer-reviewed, and keep sensitive cloud credentials isolated from individual laptops. This chapter provide a definitive deep dive into the technical workflow of remote runs, the different trigger modes, and the architectural benefits of "Streaming" logs to a global audit dashboard.

## 1. Technical Mechanics: The Workflow of a Remote Run

When you execute `terraform plan` in a local repository configured for a remote backend (TFC), the following sequence occurs:

1.  **Configuration Upload:** Terraform CLI bundles your local `.tf` files into a temporary tarball and uploads it to the TFC API.
2.  **Buffering:** TFC queues the run and allocates a "Runner" (a clean, ephemeral virtual environment).
3.  **Initialization:** The remote runner performs a `terraform init`, fetching the specific provider versions requested.
4.  **Streaming:** The output of the plan is streamed back to your terminal as if it were running locally, but the actual computation is happening in the cloud.

## 2. Trigger Modes: Three Ways to Work

### 2.1 VCS-Driven Runs (The GitOps Path)
The most common professional pattern. TFC monitors a Git repository (GitHub/GitLab). Whenever a PR is opened, TFC automatically triggers a "Speculative Plan." When the code is merged, TFC triggers the final "Apply."

### 2.2 CLI-Driven Runs
Best for iterative development. You write code locally and run `terraform plan`. TFC performs the computation and the state lock, but it doesn't require a Git commit. This allows for rapid testing while maintaining the security of remote execution.

### 2.3 API-Driven Runs
Used when TFC is integrated into a larger custom orchestration platform. You can trigger runs via a simple REST API call.

## 3. Speculative Plans: The Safety Net

A "Speculative Plan" is a special type of remote run that generates a plan but **cannot** be applied. 
- **Purpose:** Used on Pull Requests to let reviewers see the impact of a change without any risk of accidentally modifying the real cloud resources. 
- **The Gate:** Only after the code is merged to the main branch does the run become "Applyable."

## 4. Advantages of Remote Execution

1.  **Identity Isolation:** Your developers do not need `AdministratorAccess` tokens on their laptops. The TFC runner is the only entity that needs high-privilege credentials.
2.  **Auditability:** Every single execution is logged permanently in the TFC UI, including the HCL commit ID and the identity of the person who triggered it.
3.  **Shared Visibility:** As an architect, you can see all active runs across the entire company in one dashboard, enabling proactive management of resource conflicts.

## 5. Summary

Remote runs transform Terraform from a "Personal Tool" into an "Organizational Capability." By moving the execution tier to a managed, audited, and secure environment, you achieve a level of consistency and governance that is impossible with local execution. Terraform's ability to seamlessly stream these remote operations back to the developer's terminal provides the perfect balance of local performance and centralized security. Mastering the implementation of remote run workflows is a mandatory requirement for any senior DevOps professional responsible for the safety and scalability of enterprise-grade cloud foundations.
