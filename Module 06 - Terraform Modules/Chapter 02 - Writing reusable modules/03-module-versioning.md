# Reliability through Control: Module Versioning

In a large organization, multiple teams often depend on the same shared modules. If one team updates a module to add a new security feature, they might inadvertently break another team's deployment if those teams are all pointing to the latest "live" code. Module versioning is the technical solution to this coordination problem, ensuring that infrastructure deployments remain stable, predictable, and reproducible.

## 1. Why Versioning is Not Optional

Without versioning, every `terraform init` or `terraform apply` is a gamble. If the underlying module code changes on the disk or in the Git repository, your next apply will include those changes whether you wanted them or not.

**Controlled Release Cycle:**
Versioning allows module owners to release new features and security patches in a staged manner, allowing consumers to "opt-in" to the update only after they have tested it in their own development environments.

## 2. Versioning by Source Type

Terraform supports different versioning mechanisms depending on how you source the module.

### 2.1 Local Path Modules
Modules sourced via a local file path (e.g., `source = "./modules/vpc"`) **do not support versioning**. Terraform simply reads the files as they exist on the disk. This is useful for development but highly discouraged for sharing code across different projects.

### 2.2 Git and VCS Modules
When sourcing from Git (GitHub, GitLab, Bitbucket), you use the `ref` query parameter to specify a version. This can be a branch name, a commit SHA, or a Git tag.

```hcl
module "vpc" {
  # Using a specific Git Tag (Recommended)
  source = "git::https://github.com/org/vpc.git?ref=v1.2.0"
}
```
**Best Practice:** Always use Git Tags for production. Branches like `main` or `develop` are "moving targets" and can change without notice.

### 2.3 Terraform Registry Modules
The Terraform Registry (Public or Private) is the gold standard for module versioning. It uses a dedicated `version` argument and supports semantic versioning constraints.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
}
```

## 3. Semantic Versioning (SemVer) for IaC

Professional modules follow the SemVer 2.0.0 standard (`MAJOR.MINOR.PATCH`). In the context of infrastructure-as-code:

- **MAJOR (vX.0.0):** Breaking Changes. Any change that requires a manual state migration, changes a required variable, or causes a resource to be destroyed and recreated unexpectedly (e.g., changing a VPC's CIDR).
- **MINOR (v0.X.0):** New Features. Adding new optional variables, adding support for new attributes, or updating internal logic that doesn't affect the user.
- **PATCH (v0.0.X):** Bug Fixes. Correcting a typo in a tag, fixing a variable validation logic, or optimizing internal locals.

## 4. Understanding Version Constraints

Terraform uses operators to define which versions are acceptable for a root module.

- **`"v1.2.3"`**: Exact version. Only this specific version will be used.
- **`">= 1.2.3"`**: Any version greater than or equal to 1.2.3. (Risky, as it might pull in a major breaking change).
- **`"~> 1.2.3"`**: Pessimistic Constraint. Allows the patch version to increase (up to but not including 1.3.0).
- **`"~> 1.2"`**: Allows the minor version to increase (up to but not including 2.0.0). This is the balance between safety and automatic updates.

## 5. The Role of `terraform init -upgrade`

When you first run `terraform init`, Terraform fetches the latest version of a module that satisfies your constraints and records it in the `.terraform` directory. 

If a new version is released:
1. `terraform plan` will **not** automatically pull the new version if your constraints are already satisfied.
2. You must run `terraform init -upgrade`. This tells Terraform to re-evaluate the constraints and fetch the newest possible version.

## 6. Managing Module Releases

For module maintainers, the release process should be as automated as possible.

1. **Automation:** Use GitHub Actions or GitLab CI to run `terraform fmt` and `terraform validate` on every PR.
2. **Tagging:** Use automated tools like "Semantic Release" to automatically bump the version number based on commit messages.
3. **Changelog:** Always maintain a `CHANGELOG.md` file describing what changed in each version, especially documenting breaking changes in Major versions.

## 7. Strategy: Pinning for Production

A standard methodology for enterprise environments is "Strict Pinning."

- **Development:** Use `~> 2.0` to always get the latest minor improvements and bug fixes during the prototyping phase.
- **Staging/Production:** Use an exact version like `"2.5.1"`. This guarantees that the infrastructure you applied in your testing environment is the exact same code being applied in production, eliminating "Works on my machine" issues at the infrastructure level.

## 8. Summary

Versioning is the ultimate safety net of a modular infrastructure. It provides the legal "contract" between the module author and the module consumer. By mastering Git tagging, Semantic Versioning, and pessimistic constraints, you ensure that your platform can evolve without causing downtime for the teams that rely on your shared code.
