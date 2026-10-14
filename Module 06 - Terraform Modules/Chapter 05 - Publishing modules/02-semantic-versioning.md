# Precision Release Management: Semantic Versioning (SemVer)

When a module is published, the version number becomes its most important attribute. It acts as a technical guarantee between the module maintainer and the consumer about the impact of any changes. Following the Semantic Versioning (SemVer 2.0.0) standard is not just a best practice in the Terraform ecosystem; it is a requirement for the Public Registry and a critical component of professional CI/CD pipelines. This chapter explores how to apply SemVer specifically to infrastructure-as-code.

## 1. The SemVer Structure

A semantic version number consists of three parts separated by dots: `MAJOR.MINOR.PATCH` (e.g., `2.1.4`). 

- **MAJOR**: Incremented for incompatible API changes.
- **MINOR**: Incremented for adding functionality in a backward-compatible manner.
- **PATCH**: Incremented for backward-compatible bug fixes.

## 2. Defining MAJOR Changes in Terraform

In software development, a major change is a broken function call. In Terraform, a major change is anything that forces a resource to be destroyed and recreated unexpectedly, or anything that prevents code from compiling.

### 2.1 Breaking HCL Changes
- Renaming a required variable.
- Changing the type of a variable (e.g., from `string` to `list`).
- Removing an output that other modules might be relying on.

### 2.2 Breaking Infrastructure Changes
- Modifying an argument that has `ForceNew = true` in the provider schema (like changing the `name` of an S3 bucket or the `size` of a disk that doesn't support live resizing).
- Changing the logical structure of the module such that resource addresses change (e.g., moving resources into a `for_each` loop without using a `moved` block).

## 3. Defining MINOR Changes in Terraform

Minor changes add value without breaking the existing interface. A user should be able to upgrade a minor version without changing their own root module code.

### 3.1 Enhancements
- Adding a new optional variable with a default value.
- Adding a new output.
- Adding a new resource to the module that provides additional functionality (e.g., adding an optional CloudWatch alarm to an EC2 module).
- Adding a new `validation` block to a variable that narrows the permitted values but doesn't exclude currently valid ones.

## 4. Defining PATCH Changes in Terraform

Patch changes are internal. The user should not see any difference in the "Plan" output, or the difference should be purely cosmetic.

### 4.1 Maintenance
- Fixing a typo in a `description` field.
- Optimizing a `local` expression for better performance.
- Updating internal documentation in the `README.md`.
- Updating the `versions.tf` to allow a newer, backward-compatible provider version.

## 5. The Role of Git Tags

The Terraform Registry and Git-based source blocks use Git Tags to identify versions.

- **The `v` Prefix:** While SemVer doesn't require it, most DevOps tools expect tags to be prefixed with a `v` (e.g., `v1.0.0`).
- **Tag Immutability:** Once you push a tag like `v2.1.0`, you must **never** move it to a different commit. If a bug is found in that tag, you must fix it and release `v2.1.1`. Moving tags breaks the checksums in users' `.terraform.lock.hcl` files and ruins the reliability of the deployment.

## 6. Pre-release and Metadata

SemVer allows for suffixing version numbers to indicate the state of the release.
- **`v1.0.0-alpha.1`**: Very early testing.
- **`v1.0.0-beta.2`**: Feature complete but needs more validation.
- **`v1.0.0-rc.1`**: Release Candidate. Likely to become the stable version.

Terraform recognizes these suffixes and will not select them unless a user explicitly requests that exact version string.

## 7. Automated Versioning with Conventional Commits

Manually tracking version numbers is slow and prone to human error. Professional teams use **Conventional Commits** to automate the process.

**Commit Message Format:**
- `fix: correct typo in tag` -> Triggers a **PATCH** bump.
- `feat: add encryption support` -> Triggers a **MINOR** bump.
- `feat!: remove support for legacy networking` -> The `!` triggers a **MAJOR** bump.

By integrating tools like `semantic-release` into your GitHub Actions, every Merge to the `main` branch can automatically calculate the next version, generate a `CHANGELOG.md`, and push a new Git tag.

## 8. Summary

Semantic Versioning is the technical language of trust in the DevOps world. It allows module authors to innovate and evolve their code while giving consumers the confidence that their production environments won't break on the next `terraform init -upgrade`. By strictly adhering to MAJOR, MINOR, and PATCH definitions, you ensure that your infrastructure modules function as reliable, predictable components within a global automated ecosystem.
