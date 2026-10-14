# Fetching External State: The Terraform Data Block

Data sources allow a Terraform configuration to build upon information defined outside of Terraform, or defined by another separate Terraform configuration. Unlike resource blocks, which represent infrastructure objects that Terraform creates and manages, data blocks represent read-only operations that fetch data from a provider API or a local file. This chapter explores the mechanics of how data sources integrate external reality into your declarative code.

## 1. Syntax and Structure

A data block is declared using the `data` keyword, followed by the data source type and a local name.

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

### 1.1 Data Type
The first label, `aws_ami`, identifies the type of information to be fetched. Like resources, these are defined by the provider plugins.

### 1.2 Local Name
The second label, `ubuntu`, is used to reference the fetched data elsewhere in the module. The full address of a data source is `data.<TYPE>.<NAME>`. For example: `data.aws_ami.ubuntu.id`.

## 2. Arguments and Filtering

Data sources use arguments as filters to narrow down the search query to a specific object.

- **Required Arguments:** Most data sources require at least one identifier (like an `id`, `name`, or `arn`) or a set of filters to locate the data.
- **Filter Blocks:** Many providers use a nested `filter` block. These blocks often allow for wildcards (e.g., `*`) and multiple values.
- **Computed Attributes:** Once the data source is successfully queried, Terraform populates the "Computed" attributes. In the `aws_ami` example, these would include `id`, `architecture`, `description`, `root_device_name`, and more.

## 3. The Data Source Lifecycle

Understanding *when* Terraform reads data is critical for avoiding configuration errors.

### 3.1 Refresh During Plan
By default, Terraform attempts to fetch data sources during the `plan` phase.
- If the data source arguments are known (not dependent on a resource being created in the same plan), the data is fetched immediately.
- The resulting attributes are available to be used by other resources during the same plan.

### 3.2 Postponed Reading (Unknown Values)
If a data source depends on an attribute of a resource that hasn't been created yet (e.g., fetching a subnet that is being created in the current apply), Terraform marks the data source's attributes as `(known after apply)`.
- In this case, the data source is not read until the `apply` phase.
- This can sometimes lead to planning limitations, such as not being able to use the data source attributes in a `count` or `for_each` loop, as those require known values at plan time.

## 4. Use Cases for Data Sources

Data sources are the primary tool for interacting with "Brownfield" environments (infrastructure that exists but isn't managed by the current Terraform state).

### 4.1 Fetching Global Configuration
Fetching IDs of shared VPCs, subnets, or security groups that are managed by a centralized networking team.

### 4.2 Dynamic Lookups
Searching for the latest Amazon Machine Image (AMI) or the latest version of an RDS engine to ensure infrastructure stays up-to-date without hardcoding version strings.

### 4.3 Policy and Metadata
Reading the current AWS Account ID, Region, or Partition to make code more portable across different environments.
```hcl
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```

## 5. Referencing Data Source Attributes

Attributes of a data source are accessed using dot notation. Because data sources are read-only, you can only read attributes; you cannot modify them.

**Common Pattern:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  
  # Reference an existing subnet fetched via data source
  subnet_id     = data.aws_subnet.selected.id
}
```

## 6. Error Handling and "Missing Data"

If a data source query returns no results (e.g., a filter for an AMI that doesn't exist), Terraform will throw an error during the `plan` or `apply` phase. 

**Dealing with Multiple Results:** 
Some data sources (like `aws_ami` with `most_recent = true`) are designed to consolidate multiple results. Others will error if more than one result matches the filters. It is best practice to make filters as specific as possible to avoid ambiguity.

## 7. Data Sources and the State File

Data source results are stored in the `terraform.tfstate` file just like resource attributes. 
- This allows Terraform to track what was fetched.
- On the next `plan`, Terraform will re-read the data source and update the state if the external reality has changed (e.g., a newer "most recent" AMI is found), which may then trigger resource updates.

## 8. Summary of Data Source Characteristics

- **Read-Only:** They never create, update, or delete infrastructure.
- **Provider-Specific:** Each provider offers its own library of data sources.
- **Dynamic:** They enable "Live" infrastructure referencing rather than static hardcoding.
- **Dependency-Aware:** They participate in the dependency graph, allowing you to sequence data fetching after resource creation if necessary.

By effectively using data blocks, you can create Terraform configurations that are flexible, portable, and capable of integrating seamlessly with complex, pre-existing cloud environments.
