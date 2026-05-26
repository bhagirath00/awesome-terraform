# Hierarchical Design: The Mechanics of Nested Modules

Nested modules occur when one child module calls another child module. This creates a vertical hierarchy of infrastructure logic, often referred to as "Onion Architecture." While nesting is a powerful tool for encapsulation and abstraction, it introduces technical complexities regarding variable propagation, output relaying, and debugging. This chapter explores the design patterns, risks, and best practices for managing multi-level module hierarchies.

## 1. The Logic of Nesting

The primary reason to nest modules is to create higher-level abstractions. For example, a platform team might want to offer a "Standard Application Stack" module.

**Hierarchy Example:**
1. **Root Module**: Deploys the application environment.
2. **"App Stack" Module**: Orchestrates the components.
   - Calls **"AutoScaling" Module**: Manages EC2 instances.
   - Calls **"Database" Module**: Manages RDS instances.
   - Calls **"Security" Module**: Manages IAM and Security Groups.

By nesting these components, the root module becomes very clean, as it only interacts with the top-level "App Stack" interface.

## 2. The Variable Propagation Problem

One of the most significant challenges in nested modules is the "Bubble Down" effect. If a resource in the deepest level (e.g., the Database) needs a specific parameter like `storage_size`, that parameter must be defined as a variable in every module above it.

- **Root Module** must have `db_storage_size`.
- **App Stack Module** must have `db_storage_size` and pass it to the Database module.
- **Database Module** finally uses the variable.

**Technical Impact:** This leads to "Boilerplate Fatigue." You end up maintaining identical variable definitions at every layer, increasing the risk of typos and making it harder to refactor the architecture.

## 3. The Output Relay Pattern

Communication in the opposite direction (from the deep resource back to the root) requires a "Relay" pattern. 

**Process:**
1. The **Database Module** outputs `db_endpoint`.
2. The **App Stack Module** must capture that output and create its own output: 
   `output "database_url" { value = module.database.db_endpoint }`.
3. The **Root Module** can finally access `module.app_stack.database_url`.

Each layer must explicitly choose to "expose" the information from the level below it. If one layer forgets to relay an output, the root module cannot see that data, even if the underlying resource was created successfully.

## 4. Provider Inheritance in Nested Stacks

Nested modules inherit their provider configuration from their immediate parent.
- If the **Root** defines an AWS provider for `us-east-1` and calls **Module A**, **Module A** uses that provider.
- If **Module A** calls **Module B**, **Module B** also uses that same provider configuration.

**Override Patterns:**
If you need a specific nested module to operate in a different region or with different credentials, you must pass the provider explicitly through every layer of the hierarchy using the `providers` map. This adds significant complexity to the orchestration logic.

## 5. The "Deep Nesting" Anti-pattern

A common architectural mistake is nesting too many levels deep. Most industry experts recommend a maximum depth of **two levels** (Root -> Component -> Utility).

**Risks of Deep Nesting:**
- **Hidden Complexity:** It becomes impossible for an engineer to know what resources will be created without clicking through five different folders.
- **Tightly Coupled Code:** Changing a variable type in the bottom layer requires updating every layer above it.
- **Slow initialization:** `terraform init` must traverse the entire tree to download modules, which can become slow if many levels of Git-sourced modules are involved.

## 6. Testing Challenges

Testing a nested module is significantly harder than testing a flat one.
- **Unit Testing:** To test the "Database" module, you can just pass dummy variables.
- **Integration Testing:** To test the "App Stack" module, you must ensure the "Database," "AutoScaling," and "Security" sub-modules all integrate correctly.
- **Mocking:** Terraform does not natively support "Mocking" child modules. To test a parent, you must actually provision all its children, which is time-consuming and expensive.

## 7. Best Practice: Composition over Nesting

Whenever possible, prefer "Composition" (Flat) over "Nesting" (Vertical).

**Comparison:**
- **Nested:** `Root -> Calls App Stack -> App Stack calls VPC and EC2`.
- **Composed (Better):** `Root -> Calls VPC Module` and `Root -> Calls EC2 Module`. 

In the composed model, the Root Module links the VPC output directly to the EC2 input. This eliminates the need for variable propagation and makes it much easier to swap out the VPC module for a different version without touching the EC2 logic.

## 8. Summary

Nested modules are a sophisticated solution for creating standardized infrastructure "Packages." They allow you to hide complex relationships behind a single, user-friendly interface. However, the costs—in terms of variable boilerplate, output relaying, and reduced visibility—are high. Use nesting sparingly, keep your hierarchies shallow, and always consider if a flat, composed model would achieve the same result with less maintenance overhead.
