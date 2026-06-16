# Stealth Operations: Masking Secrets in Logs and Outputs

Data leakage through logs is one of the most common security vulnerabilities in Infrastructure as Code. Even with the `sensitive` flag enabled in HCL, mistakes in shell scripting or misconfigurations in CI/CD runners can cause passwords and keys to be printed in plain text. Masking is the technical process of identifying and replacing sensitive character strings with placeholders (like `***` or `<masked>`) before they are permanently written to a log file. This chapter provides a definitive deep dive into the multi-layered approach to secret masking, covering Terraform's internal logic, CI/CD provider integrations, and proactive log auditing strategies.

## 1. Layer 1: Terraform CLI Masking

Terraform provides native masking for any value marked with the `sensitive` attribute.

### 1.1 The "Sensitive Value" Placeholder
When Terraform detects that a value is sensitive, it intercepts the terminal write operation and replaces the string with `(sensitive value)`. This happens at the CLI level, ensuring that the raw value never touches the stdout or stderr of the process.

## 2. Layer 2: CI/CD Provider Masking (GitHub Actions)

Professional pipelines cannot rely solely on Terraform. If you print an environment variable that contains a secret using `echo`, Terraform's logic won't help you.

### 2.1 The `::add-mask::` Directive
In GitHub Actions, you can explicitly tell the runner to mask a string across the entire workflow execution.

```yaml
- name: Secure the Secret
  run: echo "::add-mask::${{ secrets.API_TOKEN }}"
```
Once this command is run, GitHub will find every instance of that specific API token in all subsequent job logs and replace it with `***`. This provides a global safety net for your infrastructure secrets.

## 3. The "Computed Secret" Challenge

Some resources (like a database or an IAM User) generate their own secrets (password/access key) during creation. Because these values are "Computed," they are not available to you as an input variable.

- **The Risk:** Terraform might display these values in the "Changes to be made" section of a plan if the provider doesn't explicitly mark them as sensitive in its schema.
- **The Solution:** Always check the provider documentation. For example, the `aws_iam_access_key` resource exports the `encrypted_secret` to help with this, but the raw `secret` string is often unavoidable. In these cases, use your CI/CD's masking capabilities proactively.

## 4. Post-Deployment Log Auditing

Even with masking, accidents happen. A mature DevOps organization implements "Log Hunting" to find and remediate leaks.

### 4.1 Automated Secret Scanners
Tools like **TruffleHog** and **Gitleaks** can be integrated into your pipeline to scan the `apply` logs for high-entropy strings or regex patterns matching AWS keys, SSH keys, or JWT tokens.
```bash
trufflehog --only-verified logs/apply.log
```
If a secret is found, the pipeline should fail, and the secret must be **Rotated Immediately**. Simply deleting the log is NOT enough; once a secret has appeared in a log, it must be considered compromised.

## 5. Controlling the "Log Perimeter"

The most effective masking strategy is to limit who can see the logs in the first place.
- **Access Control:** Only authorized DevOps engineers should have access to the full "Apply" logs of a production pipeline.
- **Retention:** Limit the retention period of CI/CD logs. If a log is only kept for 14 days, the duration of a potential leak's exposure is finite.

## 6. Summary

Secret masking is a layered defense strategy. By combining Terraform's internal sensitivity logic with CI/CD-level masking and proactive auditing with secret scanners, you create a robust environment where sensitive data is protected from accidental disclosure. While no system is perfect, the goal of masking is to ensure that even when a human makes a mistake, the automated guardrails are in place to prevent that mistake from becoming a security breach. Mastering these masking techniques is a non-negotiable requirement for managing professional, enterprise-scale cloud foundations.
