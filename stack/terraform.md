# Terraform (2025)

> **Last updated**: January 2026
> **Versions covered**: Terraform 1.9+, 1.10+, 1.11+, 1.12+, 1.13+
> **OpenTofu covered**: 1.8+
> **Purpose**: Infrastructure as Code (IaC) for provisioning and managing cloud resources

---

## Philosophy (2025-2026)

Terraform in 2025-2026 emphasizes **declarative infrastructure**, **immutable patterns**, and **GitOps workflows**. The ecosystem has matured with native testing, enhanced security features, and the emergence of OpenTofu as a fully compatible open-source fork.

**Key philosophical shifts:**
- **OpenTofu as viable alternative** — MPL 2.0 licensed fork with state encryption
- **GitOps-first workflows** — Atlantis, Terraform Cloud, or CI/CD for all changes
- **S3 native locking** — DynamoDB deprecated for state locking
- **Ephemeral values** — Secrets never stored in state (1.10+)
- **Provider-defined functions** — Extensible capabilities (1.8+)
- **Policy as code** — OPA/Sentinel/Checkov for compliance
- **Testing is standard** — Native `terraform test` and Terratest
- **Modules everywhere** — Reusable, versioned, documented

---

## TL;DR

- Use remote state with native S3 locking (not DynamoDB)
- Never commit secrets — use ephemeral variables or secret managers
- Always use modules for reusable infrastructure
- Pin provider and module versions exactly
- Run `terraform fmt`, `terraform validate`, and Checkov in CI
- Use workspaces sparingly — prefer directory-per-environment
- Add variable validation rules for all inputs
- Write tests with `terraform test` or Terratest
- Tag all resources consistently
- Use `-target` only for debugging, never in production
- Consider OpenTofu for vendor-neutral, open-source IaC

---

## Best Practices

### Project Structure (Recommended)

```
infrastructure/
├── modules/                    # Reusable modules
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── versions.tf
│   │   └── README.md
│   ├── compute/
│   └── database/
├── environments/               # Environment-specific configs
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── staging/
│   └── prod/
├── .terraform.lock.hcl         # Lock file (commit this!)
├── .tflint.hcl                 # Linting config
└── .pre-commit-config.yaml     # Pre-commit hooks
```

### Remote State Configuration (S3 Native Locking)

```hcl
# backend.tf — Terraform 1.10+ with native S3 locking
terraform {
  backend "s3" {
    bucket       = "mycompany-terraform-state"
    key          = "environments/prod/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true  # Native S3 locking (recommended)

    # Optional: Use assume_role for cross-account access
    assume_role = {
      role_arn = "arn:aws:iam::123456789012:role/TerraformStateAccess"
    }
  }
}

# Legacy DynamoDB locking (deprecated, migrate away)
# dynamodb_table = "terraform-locks"  # Don't use in new projects
```

### State Backend Bootstrap Module

```hcl
# modules/terraform-backend/main.tf
resource "aws_s3_bucket" "terraform_state" {
  bucket = var.bucket_name

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_kms_key" "terraform_state" {
  description             = "KMS key for Terraform state encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}
```

### Module Design Pattern

```hcl
# modules/vpc/versions.tf
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# modules/vpc/variables.tf
variable "name" {
  description = "Name prefix for all resources"
  type        = string

  validation {
    condition     = length(var.name) >= 3 && length(var.name) <= 24
    error_message = "Name must be between 3 and 24 characters."
  }

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]*$", var.name))
    error_message = "Name must start with a letter and contain only lowercase letters, numbers, and hyphens."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones required for HA."
  }
}

variable "tags" {
  description = "Tags to apply to all resources"
  type        = map(string)
  default     = {}
}

# modules/vpc/main.tf
locals {
  common_tags = merge(var.tags, {
    Environment = var.environment
    ManagedBy   = "terraform"
    Module      = "vpc"
  })
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.common_tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${var.name}-public-${var.availability_zones[count.index]}"
    Tier = "public"
  })
}

resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.cidr_block, 8, count.index + 100)
  availability_zone = var.availability_zones[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.name}-private-${var.availability_zones[count.index]}"
    Tier = "private"
  })
}

# modules/vpc/outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = aws_subnet.private[*].id
}
```

### Ephemeral Variables for Secrets (Terraform 1.10+)

```hcl
# variables.tf
variable "database_password" {
  description = "Database password (ephemeral - not stored in state)"
  type        = string
  sensitive   = true
  ephemeral   = true  # New in 1.10 - never stored in state or plan
}

variable "api_key" {
  description = "API key for external service"
  type        = string
  sensitive   = true
  ephemeral   = true
}

# main.tf — Use with write-only attributes (1.11+)
resource "aws_db_instance" "main" {
  identifier     = "mydb"
  engine         = "postgres"
  engine_version = "16.1"
  instance_class = "db.t3.micro"

  username = "admin"
  # write-only: password is sent to API but never read back or stored
  password = var.database_password

  # ... other config
}
```

### Provider-Defined Functions (Terraform 1.8+)

```hcl
# Using provider-defined functions
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"  # Must support provider functions
    }
  }
}

locals {
  # Provider-defined function syntax
  arn_parts = provider::aws::arn_parse("arn:aws:s3:::my-bucket")

  # Build ARN using provider function
  custom_arn = provider::aws::arn_build({
    partition = "aws"
    service   = "s3"
    region    = ""
    account   = ""
    resource  = "my-bucket/*"
  })
}

# Use in validation
variable "bucket_arn" {
  type = string

  validation {
    condition     = can(provider::aws::arn_parse(var.bucket_arn))
    error_message = "Must be a valid ARN."
  }
}
```

### Using for_each with Maps

```hcl
# environments/prod/main.tf
locals {
  services = {
    api = {
      instance_type = "t3.medium"
      min_size      = 2
      max_size      = 10
      port          = 8080
    }
    worker = {
      instance_type = "t3.large"
      min_size      = 1
      max_size      = 5
      port          = null
    }
    scheduler = {
      instance_type = "t3.small"
      min_size      = 1
      max_size      = 1
      port          = null
    }
  }
}

module "service" {
  for_each = local.services
  source   = "../../modules/ecs-service"

  name          = each.key
  instance_type = each.value.instance_type
  min_size      = each.value.min_size
  max_size      = each.value.max_size
  port          = each.value.port

  environment = "prod"
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.private_subnet_ids
}
```

### Moved Blocks for Refactoring

```hcl
# When renaming resources or moving to modules
moved {
  from = aws_instance.web_server
  to   = aws_instance.app_server
}

moved {
  from = aws_instance.database
  to   = module.database.aws_instance.main
}

# Moving resources into for_each
moved {
  from = aws_subnet.private_a
  to   = aws_subnet.private["us-east-1a"]
}

moved {
  from = aws_subnet.private_b
  to   = aws_subnet.private["us-east-1b"]
}
```

### Import Blocks (Terraform 1.5+)

```hcl
# Import existing resources declaratively
import {
  to = aws_s3_bucket.existing
  id = "my-existing-bucket"
}

resource "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"

  # Run `terraform plan -generate-config-out=generated.tf`
  # to auto-generate the resource configuration
}

# Conditional import with for_each
import {
  for_each = var.import_existing_buckets ? toset(var.bucket_names) : []
  to       = aws_s3_bucket.imported[each.key]
  id       = each.value
}
```

### Native Testing (terraform test)

```hcl
# tests/vpc.tftest.hcl
variables {
  name               = "test"
  environment        = "dev"
  cidr_block         = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b"]
}

run "vpc_creation" {
  command = plan

  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block mismatch"
  }

  assert {
    condition     = aws_vpc.main.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }
}

run "subnet_count" {
  command = plan

  assert {
    condition     = length(aws_subnet.public) == 2
    error_message = "Expected 2 public subnets"
  }

  assert {
    condition     = length(aws_subnet.private) == 2
    error_message = "Expected 2 private subnets"
  }
}

# With mocking (Terraform 1.7+)
run "with_mocked_provider" {
  command = plan

  providers = {
    aws = aws.mock
  }

  override_data {
    target = data.aws_availability_zones.available
    values = {
      names = ["us-east-1a", "us-east-1b", "us-east-1c"]
    }
  }
}
```

### CI/CD with GitHub Actions

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  pull_request:
    paths:
      - 'infrastructure/**'
  push:
    branches:
      - main
    paths:
      - 'infrastructure/**'

env:
  TF_VERSION: "1.10.0"
  TF_WORKING_DIR: "infrastructure/environments/prod"

jobs:
  validate:
    name: Validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Format Check
        run: terraform fmt -check -recursive
        working-directory: infrastructure

      - name: Terraform Init
        run: terraform init -backend=false
        working-directory: ${{ env.TF_WORKING_DIR }}

      - name: Terraform Validate
        run: terraform validate
        working-directory: ${{ env.TF_WORKING_DIR }}

  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Checkov
        uses: bridgecrewio/checkov-action@v12
        with:
          directory: infrastructure
          framework: terraform
          soft_fail: false
          output_format: sarif
          output_file_path: checkov.sarif

      - name: Upload SARIF
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov.sarif

  plan:
    name: Plan
    runs-on: ubuntu-latest
    needs: [validate, security]
    if: github.event_name == 'pull_request'
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ env.TF_WORKING_DIR }}

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        working-directory: ${{ env.TF_WORKING_DIR }}
        continue-on-error: true

      - name: Comment PR
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            `;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

  apply:
    name: Apply
    runs-on: ubuntu-latest
    needs: [validate, security]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Terraform Init
        run: terraform init
        working-directory: ${{ env.TF_WORKING_DIR }}

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: ${{ env.TF_WORKING_DIR }}
```

### Pre-commit Hooks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_docs
        args:
          - --args=--config=.terraform-docs.yml
      - id: terraform_tflint
        args:
          - --args=--config=__GIT_WORKING_DIR__/.tflint.hcl
      - id: terraform_checkov
        args:
          - --args=--quiet
          - --args=--compact
```

### TFLint Configuration

```hcl
# .tflint.hcl
config {
  format = "compact"
  module = true
}

plugin "terraform" {
  enabled = true
  preset  = "recommended"
}

plugin "aws" {
  enabled = true
  version = "0.32.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}

rule "terraform_naming_convention" {
  enabled = true
  format  = "snake_case"
}

rule "terraform_documented_variables" {
  enabled = true
}

rule "terraform_documented_outputs" {
  enabled = true
}
```

---

## Anti-Patterns

### Not Using Remote State

**Why it's bad**: Local state can't be shared, easily lost, causes conflicts.

```hcl
# DON'T — Local state (default)
# No backend configuration = state stored locally

# DO — Remote state with locking
terraform {
  backend "s3" {
    bucket       = "company-terraform-state"
    key          = "prod/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

### Hardcoding Values

**Why it's bad**: Not reusable, hard to maintain, environment-specific.

```hcl
# DON'T — Hardcoded values
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.large"
  subnet_id     = "subnet-12345678"

  tags = {
    Name = "prod-web-server"
  }
}

# DO — Variables and data sources
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = merge(var.tags, {
    Name = "${var.environment}-web-server"
  })
}
```

### Secrets in State or Code

**Why it's bad**: State files contain secrets in plain text.

```hcl
# DON'T — Secrets in terraform.tfvars or variables
variable "db_password" {
  default = "SuperSecret123!"  # NEVER do this
}

# DON'T — Secrets readable from state
resource "random_password" "db" {
  length = 16
}

resource "aws_db_instance" "main" {
  password = random_password.db.result  # Stored in state!
}

# DO — Use ephemeral variables (1.10+) or external secrets
variable "db_password" {
  type      = string
  sensitive = true
  ephemeral = true  # Never stored in state
}

# DO — Reference secrets manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/database/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

### Not Pinning Versions

**Why it's bad**: Unexpected breaking changes, inconsistent environments.

```hcl
# DON'T — No version constraints
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}

# DO — Pin versions exactly or with constraints
terraform {
  required_version = ">= 1.10.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.40"  # Allows 5.40.x, not 5.41+
    }
  }
}

# For modules, use exact versions
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.1"  # Exact version
}
```

### Using -target in Production

**Why it's bad**: Partial applies cause state drift and inconsistencies.

```bash
# DON'T — Targeting specific resources
terraform apply -target=aws_instance.web

# DO — Apply complete configurations
terraform apply

# If you need to break down changes, use smaller modules
# or separate state files per component
```

### Giant Monolithic Configurations

**Why it's bad**: Slow plans, blast radius too large, hard to maintain.

```hcl
# DON'T — Everything in one file/state
# main.tf with 500+ resources for entire infrastructure

# DO — Modular architecture with separate states
infrastructure/
├── networking/        # VPC, subnets, routes
│   └── main.tf
├── security/          # IAM, KMS, security groups
│   └── main.tf
├── compute/           # EC2, ECS, Lambda
│   └── main.tf
└── data/              # RDS, ElastiCache, S3
    └── main.tf
```

### Count vs for_each Misuse

**Why it's bad**: Count uses index, causing resource recreation on changes.

```hcl
# DON'T — Using count with dynamic lists
variable "subnet_names" {
  default = ["web", "app", "db"]
}

resource "aws_subnet" "main" {
  count      = length(var.subnet_names)
  cidr_block = cidrsubnet(var.vpc_cidr, 8, count.index)

  tags = {
    Name = var.subnet_names[count.index]
  }
}
# Problem: Removing "web" shifts all indexes, recreating subnets

# DO — Use for_each with maps or sets
resource "aws_subnet" "main" {
  for_each   = toset(var.subnet_names)
  cidr_block = cidrsubnet(var.vpc_cidr, 8, index(var.subnet_names, each.key))

  tags = {
    Name = each.key
  }
}
# Better: Removing "web" only destroys that subnet
```

### No Input Validation

**Why it's bad**: Errors discovered late in apply, unclear error messages.

```hcl
# DON'T — No validation
variable "environment" {
  type = string
}

# DO — Add validation rules
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_count" {
  type = number

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}
```

### Workspaces for Environment Separation

**Why it's bad**: Same backend, easy to mix up, hard to audit.

```hcl
# DON'T — Using workspaces for dev/staging/prod
terraform workspace select prod
terraform apply  # Which environment? Easy to mistake

# DO — Directory per environment with separate state
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── backend.tf      # Points to dev state
│   │   └── terraform.tfvars
│   ├── staging/
│   └── prod/
```

### Not Using Lifecycle Rules

**Why it's bad**: Accidental deletion of critical resources.

```hcl
# DON'T — No protection on critical resources
resource "aws_s3_bucket" "data" {
  bucket = "critical-data-bucket"
}

# DO — Add lifecycle protection
resource "aws_s3_bucket" "data" {
  bucket = "critical-data-bucket"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_db_instance" "main" {
  identifier = "prod-database"

  lifecycle {
    prevent_destroy = true
    ignore_changes = [
      password,  # Managed externally
    ]
  }
}
```

---

## 2025-2026 Changelog

| Version | Date | Key Features |
|---------|------|--------------|
| 1.8 | Apr 2024 | Provider-defined functions, `provider::name::function()` syntax |
| 1.9 | Jul 2024 | Input variable validation improvements, enhanced `terraform test` |
| 1.10 | Nov 2024 | **Ephemeral values**, write-only attributes, S3 native locking, DynamoDB deprecated |
| 1.11 | Feb 2025 | Write-only attributes GA, enhanced ephemeral resources |
| 1.12 | May 2025 | Performance improvements, enhanced validation errors |
| 1.13 | Aug 2025 | `terraform stacks` command, file-level test variables, `rpcapi` GA |
| 1.14 | Dec 2025 | `terraform query` for listing resources, enhanced import workflow |

### OpenTofu Changelog

| Version | Date | Key Features |
|---------|------|--------------|
| 1.6 | Jan 2024 | Initial stable release (fork from Terraform 1.6) |
| 1.7 | Jun 2024 | State encryption, enhanced testing |
| 1.8 | Nov 2024 | Early variable evaluation in `terraform` block, module sources with variables |

### Key Breaking Changes

- **Terraform 1.10+**: DynamoDB locking deprecated; migrate to `use_lockfile = true`
- **Terraform 1.11+**: Some provider-specific behaviors changed
- **OpenTofu 1.7+**: State encryption format not backward compatible

---

## Quick Reference

### Commands

| Command | Purpose |
|---------|---------|
| `terraform init` | Initialize working directory |
| `terraform plan` | Preview changes |
| `terraform apply` | Apply changes |
| `terraform destroy` | Destroy infrastructure |
| `terraform fmt` | Format code |
| `terraform validate` | Validate configuration |
| `terraform state list` | List resources in state |
| `terraform state show <resource>` | Show resource details |
| `terraform import <resource> <id>` | Import existing resource |
| `terraform output` | Show outputs |
| `terraform test` | Run tests |
| `terraform console` | Interactive console |

### Meta-Arguments

| Argument | Purpose |
|----------|---------|
| `count` | Create multiple instances (use index) |
| `for_each` | Create instances from map/set (use key) |
| `depends_on` | Explicit dependency |
| `provider` | Select provider configuration |
| `lifecycle` | Resource lifecycle hooks |

### Lifecycle Arguments

| Argument | Purpose |
|----------|---------|
| `create_before_destroy` | Create replacement before destroying |
| `prevent_destroy` | Prevent accidental deletion |
| `ignore_changes` | Ignore attribute changes |
| `replace_triggered_by` | Force replacement on change |
| `precondition` | Validate before create |
| `postcondition` | Validate after create |

### Functions (Most Used)

| Function | Example | Purpose |
|----------|---------|---------|
| `length()` | `length(var.list)` | Count items |
| `lookup()` | `lookup(var.map, "key", "default")` | Map lookup with default |
| `merge()` | `merge(var.tags, local.tags)` | Merge maps |
| `concat()` | `concat(list1, list2)` | Combine lists |
| `coalesce()` | `coalesce(var.a, var.b)` | First non-null |
| `try()` | `try(var.obj.key, "default")` | Safe access |
| `can()` | `can(regex("^[a-z]+$", var.s))` | Test expression |
| `cidrsubnet()` | `cidrsubnet("10.0.0.0/16", 8, 1)` | Calculate subnet |
| `format()` | `format("%s-%s", var.a, var.b)` | String formatting |
| `join()` | `join(",", var.list)` | Join list to string |
| `split()` | `split(",", var.string)` | Split string to list |
| `toset()` | `toset(var.list)` | Convert to set |
| `tomap()` | `tomap(var.object)` | Convert to map |

### Terraform vs OpenTofu

| Feature | Terraform | OpenTofu |
|---------|-----------|----------|
| License | BSL 1.1 (source available) | MPL 2.0 (open source) |
| State encryption | External only | Native |
| S3 locking | Native (1.10+) | Both S3 and DynamoDB |
| Governance | HashiCorp | Linux Foundation |
| Provider ecosystem | Full | Full (compatible) |
| Early variable eval | No | Yes (1.8+) |

### Security Tools

| Tool | Purpose |
|------|---------|
| Checkov | Policy-as-code, CIS/PCI compliance |
| Trivy | Vulnerability and misconfiguration scanner |
| TFLint | Linting and best practices |
| Sentinel | HashiCorp policy framework |
| OPA/Conftest | Open Policy Agent for Terraform |

---

## Resources

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [OpenTofu Documentation](https://opentofu.org/docs/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [AWS Terraform Best Practices](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/introduction.html)
- [Terraform Module Registry](https://registry.terraform.io/)
- [Checkov Documentation](https://www.checkov.io/1.Welcome/Quick%20Start.html)
- [Terratest Documentation](https://terratest.gruntwork.io/)
- [Spacelift - Terraform Best Practices](https://spacelift.io/blog/terraform-best-practices-infrastructure-as-code-2025)
- [IaC Architecture Patterns](https://spacelift.io/blog/iac-architecture-patterns-terragrunt)
