# String Interpolation

## Introduction

**String interpolation** is the ability to embed expressions inside strings. Instead of building strings by concatenating pieces with `+` or separate function calls, you write the expression directly inside the string using `${}` syntax.

String interpolation is one of the most frequently used features in Terraform — you will use it in almost every configuration you write, from naming resources to building connection strings to constructing URLs.

---

## Basic String Interpolation Syntax

The syntax is: `"text ${expression} more text"`

Everything inside `${}` is evaluated as an HCL expression. The result is converted to a string and inserted at that position:

```hcl
variable "environment" {
  default = "production"
}

variable "project" {
  default = "myapp"
}

locals {
  # Simple variable interpolation
  bucket_name = "${var.project}-${var.environment}-data"
  # Result: "myapp-production-data"

  # Interpolation in the middle of text
  log_path = "/var/log/${var.project}/app.log"
  # Result: "/var/log/myapp/app.log"

  # Multiple interpolations
  resource_name = "${var.project}-${var.environment}-${var.aws_region}"
  # Result: "myapp-production-us-east-1"
}
```

---

## What Can Go Inside `${}`

Any valid HCL expression can go inside `${}`. This includes:

### Variable References

```hcl
locals {
  name = "${var.project}-server"
  env  = "Running in ${var.environment}"
}
```

### Local References

```hcl
locals {
  prefix  = "${var.project}-${var.environment}"
  vpc_name = "${local.prefix}-vpc"       # local referencing another local
  sg_name  = "${local.prefix}-web-sg"
  db_name  = "${local.prefix}-database"
}
```

### Resource Attributes

```hcl
locals {
  # Build a URL using the load balancer's DNS name (only available after creation)
  app_url = "https://${aws_lb.main.dns_name}/api"

  # Build an S3 path using the bucket name
  config_path = "s3://${aws_s3_bucket.config.bucket}/settings.json"

  # Build an ARN reference
  db_secret_name = "${aws_db_instance.app.identifier}-credentials"
}
```

### Function Calls

```hcl
locals {
  # Call a function inside an interpolation
  upper_env   = "${upper(var.environment)}"        # "PRODUCTION"
  trimmed     = "${trimspace(var.project_name)}"   # trims whitespace

  # Combine functions with other text
  safe_name   = "${replace(lower(var.project), " ", "-")}"
  # Converts "My Project" → "my-project"
}
```

### Arithmetic Expressions

```hcl
locals {
  # Math inside interpolation
  next_port   = "${var.base_port + 1}"    # "8081" if base_port is 8080
  double_size = "${var.disk_size * 2}"    # "40" if disk_size is 20
}
```

### Conditional Expressions

```hcl
locals {
  # Ternary inside interpolation
  env_label   = "Environment: ${var.environment == "production" ? "PROD" : "NON-PROD"}"
  backup_msg  = "Backup retention: ${var.environment == "production" ? "30 days" : "7 days"}"
}
```

---

## Template Directives — `%{}` Syntax

Beyond simple value interpolation, HCL supports **template directives** using `%{}` syntax. These allow control flow (loops and conditionals) inside strings.

### `%{if}` Directive

```hcl
locals {
  # Conditionally add text to a string
  monitoring_note = "Server: ${var.server_name}%{if var.enable_monitoring} [monitoring enabled]%{endif}"
  # If monitoring is true: "Server: web-01 [monitoring enabled]"
  # If monitoring is false: "Server: web-01"
}
```

With an else clause:
```hcl
locals {
  env_type = "%{if var.environment == "production"}PRODUCTION%{else}NON-PRODUCTION%{endif}"
}
```

### `%{for}` Directive

Build strings by iterating over a list:

```hcl
locals {
  servers = ["web-01", "web-02", "web-03"]

  # Build a comma-separated list inside a string
  server_list = "%{for s in local.servers}${s}, %{endfor}"
  # Result: "web-01, web-02, web-03, " (note trailing comma)

  # Build a multi-line string (useful for user_data scripts)
  hosts_content = "%{for ip in local.server_ips}${ip} server\n%{endfor}"
}
```

A more practical use — building a bash script in `user_data`:

```hcl
locals {
  packages = ["nginx", "htop", "curl", "jq"]

  startup_script = <<-EOF
    #!/bin/bash
    apt-get update -y
    %{for pkg in local.packages}
    apt-get install -y ${pkg}
    %{endfor}
    systemctl start nginx
    echo "Setup complete"
  EOF
}

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  user_data     = local.startup_script
}
```

---

## Heredoc Strings

For multi-line strings — especially user_data scripts, policy documents, and configuration files — use the heredoc syntax:

### Standard Heredoc (`<<EOF`)

```hcl
resource "aws_instance" "web" {
  user_data = <<EOF
#!/bin/bash
echo "Hello from ${var.environment}" > /tmp/hello.txt
apt-get update
apt-get install -y nginx
systemctl start nginx
EOF
}
```

The content starts on the line after `<<EOF` and ends at the line containing only `EOF`. The marker can be any word — `EOF` is just convention.

### Indented Heredoc (`<<-EOF`)

The `<<-` variant strips leading whitespace, allowing you to indent the heredoc content to match the surrounding code:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  user_data = <<-EOF
    #!/bin/bash
    # This script runs on first boot
    export ENVIRONMENT="${var.environment}"
    export PROJECT="${var.project}"
    export DB_HOST="${aws_db_instance.app.address}"

    apt-get update -y
    apt-get install -y nginx

    # Configure the application
    cat > /etc/app/config.json <<APPEOF
    {
      "environment": "${var.environment}",
      "db_host": "${aws_db_instance.app.address}",
      "db_name": "${var.db_name}"
    }
    APPEOF

    systemctl start nginx
    echo "Setup complete for ${var.project} in ${var.environment}"
  EOF
}
```

With `<<-EOF`, the leading 4 spaces are stripped from each line. This keeps the heredoc readable at its current indentation level.

---

## Common String Interpolation Patterns

### Resource Naming

```hcl
locals {
  prefix = "${var.project}-${var.environment}"
}

# Consistent naming throughout the configuration
resource "aws_vpc"            "main" { tags = { Name = "${local.prefix}-vpc" } }
resource "aws_lb"             "main" { name = "${local.prefix}-alb" }
resource "aws_db_instance"    "app"  { identifier = "${local.prefix}-db" }
resource "aws_s3_bucket"      "app"  { bucket = "${local.prefix}-app-data" }
resource "aws_security_group" "web"  { name = "${local.prefix}-web-sg" }
resource "aws_iam_role"       "app"  { name = "${local.prefix}-app-role" }
```

### Building Connection Strings

```hcl
locals {
  # PostgreSQL connection string
  postgres_url = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.app.endpoint}/${var.db_name}"

  # Redis connection string
  redis_url = "redis://${aws_elasticache_cluster.app.cache_nodes[0].address}:${aws_elasticache_cluster.app.cache_nodes[0].port}"

  # Application endpoint
  api_endpoint = "https://${aws_lb.main.dns_name}/api/v1"
}
```

### SSM Parameter Paths

```hcl
locals {
  ssm_prefix = "/${var.project}/${var.environment}"
}

resource "aws_ssm_parameter" "db_host" {
  name  = "${local.ssm_prefix}/database/host"
  type  = "String"
  value = aws_db_instance.app.address
}

resource "aws_ssm_parameter" "db_password" {
  name  = "${local.ssm_prefix}/database/password"
  type  = "SecureString"
  value = var.db_password
}

resource "aws_ssm_parameter" "api_key" {
  name  = "${local.ssm_prefix}/api/secret-key"
  type  = "SecureString"
  value = var.api_secret_key
}
```

### IAM Policy Documents

```hcl
data "aws_iam_policy_document" "s3_access" {
  statement {
    effect = "Allow"
    actions = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
    resources = [
      "${aws_s3_bucket.app.arn}/*",           # interpolation in ARN
      "${aws_s3_bucket.assets.arn}/*"
    ]
  }

  statement {
    effect    = "Allow"
    actions   = ["s3:ListBucket"]
    resources = [aws_s3_bucket.app.arn]
    condition {
      test     = "StringLike"
      variable = "s3:prefix"
      values   = ["${var.environment}/*"]     # interpolation in condition value
    }
  }
}
```

### CloudWatch Log Group Naming

```hcl
resource "aws_cloudwatch_log_group" "app" {
  name              = "/aws/ecs/${var.project}/${var.environment}/app"
  retention_in_days = var.environment == "production" ? 90 : 14
}

resource "aws_cloudwatch_log_group" "access" {
  name = "/aws/alb/${var.project}/${var.environment}/access"
}
```

---

## When NOT to Use String Interpolation

Sometimes interpolation is unnecessary. If the entire value is just a reference, you do not need `${}`:

```hcl
# UNNECESSARY — the entire value is just var.environment
tags = {
  Environment = "${var.environment}"  # redundant ${}
}

# BETTER — direct reference without interpolation
tags = {
  Environment = var.environment
}
```

```hcl
# UNNECESSARY
subnet_id = "${aws_subnet.public.id}"

# BETTER
subnet_id = aws_subnet.public.id
```

Use `${}` only when you are embedding the expression inside a larger string. If the whole value is just the expression, use it directly.

---

## Escaping the Dollar Sign

If you need a literal `${` in a string (not an interpolation), escape the dollar sign with `$$`:

```hcl
locals {
  # Shows literal "${HOSTNAME}" — not interpolated
  bash_command = "echo $${HOSTNAME}"
  # Result: echo ${HOSTNAME}

  # Useful for scripts that contain bash variable references
  startup_script = <<-EOF
    #!/bin/bash
    INSTANCE_ID=$$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
    echo "Instance: $${INSTANCE_ID}" >> /var/log/startup.log
    echo "Project: ${var.project}" >> /var/log/startup.log
  EOF
  # $${INSTANCE_ID} becomes ${INSTANCE_ID} (bash variable) in the output
  # ${var.project} becomes the Terraform variable value in the output
}
```

---

## Summary

- String interpolation uses `${}` to embed expressions inside strings
- Any HCL expression can go inside `${}` — variable references, locals, resource attributes, function calls, arithmetic, conditionals
- Template directives `%{if}` and `%{for}` allow control flow inside strings
- Heredoc syntax (`<<-EOF`) creates multi-line strings — very useful for user_data scripts and policy documents
- Common patterns: resource naming, connection strings, SSM parameter paths, IAM policies
- When the entire value is just a reference (not embedded in text), skip the `${}` and use the reference directly
- Use `$$` to include a literal `${` in a string (useful for bash scripts with variable references)

In the next file, we cover conditional expressions — the ternary operator and other ways to make decisions in your Terraform code.
