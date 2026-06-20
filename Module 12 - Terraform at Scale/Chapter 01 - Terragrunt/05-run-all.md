# Global Orchestration: The run-all Command in Terragrunt

As an infrastructure project grows, managing individual modules one by one becomes an operational bottleneck. If you have 50 different microservices, manually running `terraform apply` on each is physically impossible. Terragrunt's `run-all` command is the industrial-scale solution for this problem, allowing you to orchestrate the execution of every module across your entire repository in a single, dependency-aware command. This chapter provide a definitive deep dive into the technical mechanics of the global graph engine, the performance tuning of parallel executions, and the critical security protocols for "Bulk" infrastructure operations.

## 1. Technical Mechanics: The Graph Engine

When you execute `terragrunt run-all plan`, Terragrunt doesn't just loop through folders. It performs a sophisticated analysis of your entire project:

1.  **Scanning:** It crawls the directory structure to find all `terragrunt.hcl` files.
2.  **Graph Construction:** It reads the `dependency` and `dependencies` blocks in each file to build a Directed Acyclic Graph (DAG) of the entire repository.
3.  **Ordering:** It determines the exact sequence required for a successful deployment (e.g., "VPC must finish before RDS starts").
4.  **Parallel Execution:** It executes as many modules as possible in parallel, limited only by the dependency constraints and your defined parallelism threads.

## 2. Performance Tuning: High-Velocity Orchestration

In an enterprise Monorepo, running a scan on 200+ modules can be resource-intensive. 
- **The `--terragrunt-parallelism` Flag:** By default, Terragrunt runs many operations in parallel. For massive environments, you can tune this to match your CI/CD runner's capabilities (e.g., `--terragrunt-parallelism 10` to limit concurrent AWS API calls).
- **The `--terragrunt-working-dir` Flag:** You can scope a `run-all` command to a specific subset of your hierarchy (e.g., running `run-all` only within the `prod/networking` folder).

## 3. Dependency Persistence and Propagation

`run-all` is particularly powerful because it handles the passing of outputs between modules automatically.
- **The Workflow:** If Module B depends on Module A, `run-all` will ensure Module A is applied, its outputs are cached in memory, and those outputs are injected into the inputs of Module B without either module ever touching a local disk or needing manual `terraform_remote_state` configuration.

## 4. The "Danger Zone": Destructive Operations at Scale

The most dangerous command in the DevOps lexicon is `terragrunt run-all destroy`.
- **The Risk:** A typo in the directory path could trigger the simultaneous destruction of your entire company's global footprint.
- **The Guardrail:** A professional pipeline should ALWAYS require a `--terragrunt-non-interactive` flag and a high-level manual approval before a `run-all destroy` is permitted.

## 5. Summary

The `run-all` command is what makes Terragrunt a true "Orchestration Engine" rather than a simple wrapper. By providing a dependency-intelligent way to manage hundreds of cloud modules at once, it enables platform engineering teams to treat an entire department's infrastructure as a single, manageable unit. Terraform's reliability combined with Terragrunt's graph awareness provides the technical foundation needed to deploy complex global foundations at enterprise scale. Mastering the orchestration of the global graph is a mandatory requirement for any senior DevOps professional responsible for the high-velocity growth and stability of world-class Infrastructure as Code environments.
