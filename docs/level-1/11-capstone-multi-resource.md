# 11 · Capstone — Multi-Resource Configuration

This capstone pulls together everything from Level 1 into one small but
complete configuration: variables, multiple resources with a real
dependency between them, a data source, and outputs — reasoned through
against documented provider behavior rather than run against a live cloud
account.

## The scenario

A "static site bucket" configuration: an S3 bucket for hosting static
files, versioning enabled on it, public access blocked (module 08's
pattern), and an output reporting the bucket's website endpoint — all
parameterized so the same configuration works for multiple environments.

## `variables.tf`

```hcl
variable "project_name" {
  description = "Short name used to build resource names"
  type        = string
}

variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "enable_versioning" {
  description = "Whether to enable S3 bucket versioning"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Common tags applied to every resource"
  type        = map(string)
  default     = {}
}
```

`validation` blocks (used here on `environment`) let a variable declare its
own acceptable values with a custom error message — Terraform rejects an
invalid value at plan time with that message, rather than letting a typo
like `"prd"` silently flow through to a resource name or tag.

## `main.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "random_id" "suffix" {
  byte_length = 4
}

locals {
  bucket_name = "${var.project_name}-${var.environment}-${random_id.suffix.hex}"

  common_tags = merge(var.tags, {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
  })
}

resource "aws_s3_bucket" "site" {
  bucket = local.bucket_name
  tags   = local.common_tags
}

resource "aws_s3_bucket_versioning" "site" {
  bucket = aws_s3_bucket.site.id

  versioning_configuration {
    status = var.enable_versioning ? "Enabled" : "Suspended"
  }
}

resource "aws_s3_bucket_public_access_block" "site" {
  bucket = aws_s3_bucket.site.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_website_configuration" "site" {
  bucket = aws_s3_bucket.site.id

  index_document {
    suffix = "index.html"
  }

  error_document {
    key = "error.html"
  }
}

data "aws_region" "current" {}
```

Every resource-to-resource reference here (`aws_s3_bucket.site.id` used
four times) is exactly module 05's implicit-dependency pattern: Terraform
infers that `aws_s3_bucket.site` must be created before any of the four
resources that reference its `id`, and orders operations accordingly — no
manual `depends_on` needed for any of these. `local.bucket_name`'s
ternary-free version deliberately reuses module 06's `merge()` pattern for
tags, and the `var.enable_versioning ? "Enabled" : "Suspended"` expression
is the conditional syntax previewed briefly in module 03 and covered fully
in Level 2.

## `outputs.tf`

```hcl
output "bucket_name" {
  description = "Name of the created S3 bucket"
  value       = aws_s3_bucket.site.id
}

output "bucket_arn" {
  description = "ARN of the created S3 bucket"
  value       = aws_s3_bucket.site.arn
}

output "website_endpoint" {
  description = "Static website hosting endpoint"
  value       = aws_s3_bucket_website_configuration.site.website_endpoint
}

output "region" {
  description = "AWS region this bucket was created in"
  value       = data.aws_region.current.name
}
```

## `dev.tfvars`

```hcl
project_name      = "acme-blog"
environment       = "dev"
enable_versioning = false

tags = {
  Team = "platform"
}
```

## Running it (workflow recap from modules 09–10)

```bash
terraform init
terraform fmt -check -recursive
terraform validate
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"
```

Expected plan shape, reasoned from the resource graph above: 5 resources to
add (`random_id.suffix`, `aws_s3_bucket.site`,
`aws_s3_bucket_versioning.site`, `aws_s3_bucket_public_access_block.site`,
`aws_s3_bucket_website_configuration.site`), with `data.aws_region.current`
read (not counted as an "add") during the same plan. `bucket_name` and
`bucket_arn` show `(known after apply)` for the same reason module 08's
example did — they depend on `random_id.suffix.hex`, unknown until that
resource is actually created.

## Exercise

Extend this capstone with a sixth argument in `variables.tf`:
`force_destroy_bucket` (type `bool`, default `false`), wire it into
`aws_s3_bucket.site` as the `force_destroy` argument (documented on the AWS
provider as allowing a non-empty bucket to be deleted by `terraform
destroy`), and add a `validation` block on `environment` becoming case
sensitive — verify that passing `"Prod"` (capitalized) is correctly
rejected by the existing `contains([...])` condition without changing it,
and explain in one sentence why case sensitivity is the correct default
behavior here.
