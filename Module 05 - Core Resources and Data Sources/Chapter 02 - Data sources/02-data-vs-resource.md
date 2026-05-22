# Architectural Comparison: Resources vs. Data Sources

In Terraform, the distinction between a `resource` block and a `data` block is one of the most fundamental concepts for designing modular and maintainable infrastructure. While they share similar syntax and are both provided by plugins, their roles in the lifecycle of an infrastructure stack are diametrically opposed. This chapter provides a deep dive into the technical and architectural differences between managed resources and external data sources.

## 1. Managed vs. Unmanaged Infrastructure

The primary difference lies in ownership and the "Source of Truth."

### 1.1 Managed Resources (`resource`)
A resource block represents an object that Terraform "owns." 
- **Ownership:** Terraform is responsible for the full CRUD (Create, Read, Update, Delete) lifecycle of this object.
- **Goal:** To move the current state of the infrastructure toward the "Desired State" defined in the configuration.
- **State Impact:** Every resource managed in a configuration results in a corresponding entry in the `terraform.tfstate` file, including sensitive metadata and current attribute values.

### 1.2 External Data Sources (`data`)
A data source represents an object that exists independently of the current Terraform configuration.
- **Consumption:** Terraform merely requests information about the object to use in other parts of the configuration.
- **Goal:** To "look up" information about the environment so that managed resources can be configured correctly.
- **State Impact:** While data source results are cached in the state file for performance, Terraform will never attempt to modify or delete the underlying object.

## 2. The CRUD Lifecycle Comparison

| Feature | Resource Block | Data Block |
| :--- | :--- | :--- |
| **Create** | Terraform creates the object if it doesn't exist. | No creation capability. |
| **Read** | Terraform reads the object to check for drift. | Terraform reads the object to fetch values. |
| **Update** | Terraform modifies the object to match config. | No update capability. |
| **Delete** | Terraform destroys the object if removed from config. | No destruction capability. |
| **Primary Use** | Defining what infrastructure should exist. | Querying what infrastructure already exists. |

## 3. Dependency Graph and Resolution

Terraform builds a Directed Acyclic Graph (DAG) to determine the order of operations. Resources and data sources are treated differently in this graph.

### 3.1 Resource Dependencies
Resources often depend on each other. If `Resource B` depends on `Resource A`, Terraform ensures `A` is fully created before initiating the creation of `B`. This is essential for physical dependencies (e.g., an EC2 instance needing a Subnet ID).

### 3.2 Data Source Dependencies
Data sources are usually prioritized. Terraform attempts to resolve data blocks as early as possible—often during the `refresh` phase before the `plan` is even generated. 
- If a data source depends on a managed resource *within the same module*, it creates a bridge between the managed and unmanaged worlds. 
- **Warning:** Data sources depending on resources can lead to planning failures if the resource's output isn't known until after the `apply` phase.

## 4. Use Case Scenarios

### 4.1 When to use a Resource
- You are building a new application stack from scratch.
- You are the primary administrator of the component (e.g., a Database role or a VPC).
- You want Terraform to alert you if someone manually changes the configuration in the cloud console (Drift Detection).

### 4.2 When to use a Data Source
- You need the ID of a VPC that was created by a different team or a different Terraform workspace.
- You want to find the ID of the latest "Gold Image" (AMI) published by your security team.
- You need to fetch the list of Availability Zones for the current region to distribute subnets.
- You need the AWS Account ID or current IAM user details for tagging purposes.

## 5. Decision Matrix: Resource or Data Source?

Imagine you need to configure an EC2 instance to use a specific Security Group.

**Scenario A: You manage the Security Group.**
You define a `resource "aws_security_group"`. You reference it as `aws_security_group.my_sg.id`. If you change the ingress rules in code, Terraform updates the group.

**Scenario B: You use a shared Corporate Security Group.**
You define a `data "aws_security_group"`. You fetch it by name or tag. You reference it as `data.aws_security_group.corp_sg.id`. If the corporate team changes the rules, your code simply "sees" the new rules on the next plan, but it will never try to change them back.

## 6. Technical Constraints and Error Handling

### 6.1 Unknown Values
- **Resources:** Can handle "Unknown" values during the plan phase. Terraform knows a resource *will* have an ID, even if it doesn't have it yet.
- **Data Sources:** If a data source is queried using a value that is "unknown" (because it comes from a resource not yet created), the entire data source result becomes unknown. This can block subsequent logic that relies on the data source, such as `count` or `for_each` loops.

### 6.2 Missing Objects
- **Resources:** If a resource is missing from the cloud, Terraform marks it for "Creation."
- **Data Sources:** If a data source query finds nothing, Terraform throws a fatal error and stops the plan. Data sources enforce that the external dependency *must* exist.

## 7. Performance and Cost

Every data source in your configuration represents at least one API call to the cloud provider during every `plan` and `refresh` operation. 
- **Large Stacks:** In modules with dozens of data sources, `terraform plan` can become slow as it waits for network responses.
- **Rate Limiting:** Excessive use of data sources can occasionally lead to API rate-limiting (Throttling) from cloud providers like AWS or Azure.
- **Optimization:** Use data sources only for information that truly cannot be passed via variables or managed as a resource.

## 8. Summary

The choice between a resource and a data source is a choice of **responsibility**. Use `resource` when your code is the authority on the object's existence. Use `data` when your code is a consumer of information provided by the outside world. Professional Terraform architectures leverage data sources to create "Loose Coupling" between different layers of infrastructure (Networking, Security, Application), allowing teams to work independently while remaining integrated.
