# Professional Standards: Standard Module Structure

Developing a reusable module is an exercise in engineering discipline. To ensure a module is understandable by other engineers, compatible with automated testing tools, and easily published to a registry, it must follow the "Standard Module Structure." This chapter details the required files, directory organization, and technical conventions that define a production-grade Terraform module.

## 1. The minimal File Set

Every Terraform module, regardless of its complexity, should contain at least three files. This separation of concerns ensures that the "Contract" (the inputs and outputs) is clearly distinct from the "Implementation" (the resources).

### 1.1 `variables.tf` (The Contract: Inbound)
This file defines all inputs that the module requires. It should include:
- **Type Constraints:** Ensuring that a user cannot pass a string where a number is expected.
- **Default Values:** Providing sensible defaults where possible to reduce user friction.
- **Validation Blocks:** Enforcing architectural rules (e.g., "The instance type must be from the t3 family").
- **Descriptions:** Every variable must have a description, as this is used to generate documentation.

### 1.2 `outputs.tf` (The Contract: Outbound)
This file defines what data the module returns to the caller.
- **IDs and ARNs:** Always output the primary identifiers of the resources created.
- **Computed Data:** Output values that are only available after creation (like public IPs).
- **Attribute Sets:** Sometimes it is useful to output the entire resource object for flexibility.

### 1.3 `main.tf` (The Implementation)
This file contains the resource and data source blocks. For smaller modules, a single `main.tf` is sufficient. For larger modules, you may break this down further (e.g., `compute.tf`, `iam.tf`, `networking.tf`).

## 2. Advanced Foundation Files

As a module matures, additional files are required to ensure stability across different Terraform environments.

### 2.1 `versions.tf` (Requirement Constraints)
This file prevents the module from being used in unsupported environments. It typically contains two main blocks:
- **`required_version`**: Specifies which version of Terraform core is required (e.g., `~> 1.0`).
- **`required_providers`**: Specifies the provider versions the module has been tested with. This prevents breaking changes in a provider (e.g., the AWS provider move from v3 to v4) from breaking your module.

### 2.2 `locals.tf` (Internal Logic)
If a module requires complex string interpolation, tagging logic, or environment-specific calculations, these should be placed in `locals.tf`. This keeps `main.tf` clean and focuses on resource declaration.

## 3. Documentation: The `README.md`

In a professional module, the `README.md` is not just a help file; it is a critical technical component.
- **Registry Support:** The Terraform Public and Private Registries use the README to render the landing page for the module.
- **Terraform-docs:** Tools like `terraform-docs` can automatically parse your variables and outputs to generate a formatted table in the README.
- **LSP Integration:** Modern IDEs (like VS Code with the HashiCorp extension) use the README and variable descriptions to provide "IntelliSense" when a developer calls your module.

## 4. Sub-modules: Internal Organization

Sometimes a module is complex but doesn't warrant being broken into entirely separate repositories. In this case, use a `modules/` directory inside your main module.

**Structure:**
```text
my-aws-app/
├── main.tf
├── variables.tf
├── outputs.tf
├── modules/
│   ├── db/
│   │   ├── main.tf
│   │   └── variables.tf
│   └── networking/
│       ├── main.tf
│       └── variables.tf
```
This is known as a "Compound Module." The root logic in `my-aws-app` orchestrates the sub-modules found in the `modules/` folder.

## 5. Examples and the `examples/` Directory

A professional module is incomplete without usage examples. Place these in an `examples/` directory at the root of the module.
- Each example should be a runnable "Root Module" that calls your module.
- This serves two purposes: It shows users how to use the module, and it provides a target for automated testing (like `terratest` or `terraform test`).

**Example Structure:**
```text
examples/
├── basic-usage/
│   └── main.tf
├── custom-vpc/
│   └── main.tf
```

## 6. Naming Conventions for Resources

Inside a module, you should avoid generic names like `resource "aws_instance" "web"`. Instead, follow a standard like `resource "aws_instance" "this"` if the module only manages one instance of that type.
- This prevents the confusion of having `web_server.web` or `database.db`.
- The address `module.web_server.aws_instance.this` is clean and standard across the Terraform ecosystem.

## 7. The `.terraform` and `.terraform.lock.hcl` Files

When developing a module, these files should **never** be included in your source control (Git).
- **`.terraform/`**: This directory is created locally during `init`. It contains the provider binaries.
- **`.terraform.lock.hcl`**: This is for the *Root Module* to ensure consistent provider versions. If you are writing a child module, the consumer's root module will generate its own lock file.

## 8. Summary

The Standard Module Structure is more than just a naming convention; it is a framework for building predictable infrastructure. By strictly separating variables, outputs, and resources, and providing clear versions and examples, you ensure that your modules are not just "code that works," but "products" that other engineers can trust and use to accelerate their own development cycles.
