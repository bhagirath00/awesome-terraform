# Data-Driven Infrastructure: Provisioning Dynamic Resources

Dynamic resource creation is the pinnacle of Terraform automation. It allows you to move away from static, hardcoded resource blocks and instead generate infrastructure based on external data sources, configuration files, or complex logic. By combining the `count` and `for_each` meta-arguments with data transformation functions, you can build "Infrastructure Engines" that automatically respond to changes in your project requirements. This chapter provides a definitive deep dive into the patterns for data-driven resources, focusing on external file integration (JSON/YAML) and the orchestration of complex resource dependencies.

## 1. The Strategy: Decoupling Data from Code

In a professional DevOps environment, the "What" (the data) should be separated from the "How" (the Terraform code).
- **Static Pattern:** Defining every subnet manually in an `aws_subnet` resource.
- **Dynamic Pattern:** Defining the subnets in a JSON file and using a single `aws_subnet` block with `for_each` to provision them.

## 2. Pattern: JSON-Driven Resource Creation

JSON and YAML are the standard languages for configuration data. Terraform's `jsondecode()` and `yamldecode()` functions allow you to import this data as a map/list and use it directly in a loop.

**Example `config.json`:**
```json
{
  "web": { "type": "t3.micro", "public": true },
  "api": { "type": "t3.small", "public": false }
}
```

**Terraform Implementation:**
```hcl
locals {
  service_config = jsondecode(file("${path.module}/config.json"))
}

resource "aws_instance" "services" {
  for_each = local.service_config

  instance_type = each.value.type
  associate_public_ip_address = each.value.public
  
  tags = {
    Name = each.key
  }
}
```

## 3. Pattern: Building Complex Maps with for Expressions

Often, the data you receive from a data source is a list of objects, but `for_each` requires a map with unique keys. You can use the `for` expression to "Key" your data.

```hcl
data "aws_vpcs" "all" {}

locals {
  # Convert a list of VPC IDs into a map indexed by a friendly name
  vpc_map = {
    for id in data.aws_vpcs.all.ids : id => id
  }
}

resource "aws_flow_log" "logs" {
  for_each = local.vpc_map
  vpc_id   = each.value
  # ...
}
```

## 4. Scaling Modules Dynamically

Dynamic resource creation extends to modules. This allows you to deploy entire environment stacks identically based on a simple list of environment names.

```hcl
variable "environments" {
  type    = set(string)
  default = ["development", "staging"]
}

module "app_stack" {
  source   = "./modules/application"
  for_each = var.environments

  environment_name = each.key
}
```

## 5. Architectural Considerations: The Dependency Graph

When you create resources dynamically, the Terraform dependency graph becomes more complex. 
- **Implicit Dependencies:** Terraform automatically understands that if `aws_instance.web` depends on `aws_subnet.public`, and both use `for_each` over the same map, then each specific instance depends on its corresponding subnet.
- **Performance:** In massive deployments (e.g., 500 subnets), Terraform must calculate every specific resource address in the graph. This can significantly slow down `terraform plan` and increase memory consumption on CI/CD runners.

## 6. Constraints and Error Handling

- **Key Uniqueness:** If your dynamic logic generates two identical keys in a `for_each` map, Terraform will throw an error. Use the `try()` function or string concatenation to ensure keys are unique.
- **Empty Collections:** If your dynamic logic results in an empty map or list, Terraform will simply create zero resources. This is intended behavior and is useful for "Self-Scaling" infrastructure.

## 7. Summary

Dynamic resource creation transforms Terraform from a simple deployment script into a sophisticated infrastructure engine. By decoupling your configuration data from your HCL logic and mastering the use of `for` expressions and file-decoding functions, you can build modules that are infinitely reusable and highly scalable. While data-driven infrastructure requires more initial engineering effort and rigorous validation of input data, the long-term benefits in consistency and operational efficiency are the foundation of a modern, automated DevOps organization.
