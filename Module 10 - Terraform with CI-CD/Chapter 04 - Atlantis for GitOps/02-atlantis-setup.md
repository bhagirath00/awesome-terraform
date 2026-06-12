# Building the GitOps Engine: Atlantis Setup and Configuration

Setting up Atlantis is the process of deploying a persistent automation server that acts as a bridge between your Git provider and your cloud infrastructure. Unlike ephemeral CI/CD runners, the Atlantis server must be stable, secure, and properly authenticated to perform high-privilege operations. In a professional DevOps environment, Atlantis is typically deployed within a Kubernetes cluster or as a containerized service in a private VPC. This chapter provide a definitive deep dive into the deployment methods, the technical prerequisites for Git integration, and the project-level configuration patterns needed for an enterprise-ready Atlantis installation.

## 1. Deployment Strategies

### 1.1 Kubernetes (The Standard)
For teams already running Kubernetes, the official Atlantis Helm chart is the preferred installation method. It handles persistent volume management (for checking out repositories) and the orchestration of the service and ingress.

### 1.2 Docker / AWS Fargate
For simpler environments, running the Atlantis Docker image on a managed container service like AWS Fargate is highly effective. This eliminates the overhead of managing a K8s cluster while providing a reliable, persistent endpoint for webhooks.

## 2. Technical Prerequisites for Git Integration

To function, Atlantis needs a "Handshake" with your Git provider (e.g., GitHub).

1.  **Service Account:** Create a dedicated "Bot" user in your Git provider (e.g., `company-atlantis-bot`). 
2.  **Access Token (PAT):** Generate a Personal Access Token for the bot with `repo` scopes. This allow Atlantis to clone repositories and post comments.
3.  **Webhook Secret:** A unique string used to verify that the webhook requests received by the Atlantis server actually came from GitHub and not a malicious actor.
4.  **HTTPS Endpoint:** GitHub and GitLab require webhooks to be sent over a secure HTTPS connection. You must configure an SSL certificate (via AWS ACM or Let's Encrypt) on your Load Balancer or Ingress.

## 3. Configuring the Atlantis Server

The server itself is configured using command-line flags or environment variables.

```bash
atlantis server \
  --atlantis-url="https://atlantis.company.com" \
  --gh-user="company-atlantis-bot" \
  --gh-token="${GH_TOKEN}" \
  --gh-webhook-secret="${GH_WEBHOOK_SECRET}" \
  --repo-allowlist="github.com/my-org/*" \
  --write-git-creds
```
### 3.1 The Repo Allowlist (Security Control)
This is a critical security flag. It ensures that Atlantis only responds to webhooks from specific, trusted repositories within your organization. Never use `*` in a production environment.

## 4. Project-Level Configuration: `atlantis.yaml`

While the server has global settings, you can define specific behaviors for individual repositories using an `atlantis.yaml` file located in the root of your project.

```yaml
version: 3
projects:
- name: networking-prod
  dir: environments/prod/networking
  terraform_version: v1.5.0
  autoplan:
    enabled: true
    when_modified: ["*.tf", "*.tfvars"]
  workflow: custom_security

workflows:
  custom_security:
    plan:
      steps:
      - run: tfsec .
      - init
      - plan:
          extra_args: ["-lock=false"]
```

### 4.1 Custom Workflows
Advanced teams use the `workflows` block to inject security scanning or custom scripts into the plan phase. In the example above, `tfsec` is executed before any Terraform logic, failing the plan if security vulnerabilities are found.

## 5. Persistent Storage and State

Atlantis stores a local clone of your repository while a PR is open. 
- **The Disk Risk:** If the Atlantis server restarts and lose its disk, you lose the local locks and the plan files. 
- **Solution:** Always use a Persistent Volume Claim (PVC) or an EBS volume to ensure that the `atlantis-data` directory survives server upgrades or failures.

## 6. Security Hardening

- **Network Isolation:** The Atlantis server should live in a **Private Subnet**. Use an Internal Load Balancer if you are using self-hosted runners, or a public Load Balancer with a restricted Security Group if receiving webhooks from the internet.
- **Provider Credentials:** Never bake AWS Access Keys into the Atlantis image. Use **IAM Roles for Service Accounts (IRSA)** or an EC2 Instance Profile to provide the Atlantis process with the permissions it needs.

## 7. Summary

Setting up Atlantis transforms your Git provider into a powerful infrastructure management console. By properly configuring service accounts, secure webhooks, and project-level `atlantis.yaml` files, you create an automation engine that is both highly powerful and strictly governed. Terraform's integration with Atlantis represents the pinnacle of "GitOps" maturity, ensuring that your organizational standards are automatically enforced and your deployment history is perfectly preserved in the collaborative heart of your development workflow.
