# 08 · A First Real Resource

This module walks through a complete, small configuration end to end — a
local file resource (needs no cloud account at all) and then the cloud
equivalent (an S3 bucket), reasoned through against documented provider
behavior. Neither example is run against a live cloud account in this
lesson — treat the AWS portion as a careful read-through to prepare you for
your own account, and verify current argument names against the provider's
registry page before applying anything for real.

## Starting with zero cloud dependencies: the `local` provider

The `hashicorp/local` provider manages files on the machine running
Terraform — perfect for learning the full resource lifecycle without
needing any credentials:

```hcl
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

resource "local_file" "greeting" {
  filename = "${path.module}/greeting.txt"
  content  = "Hello from Terraform!\n"
}
```

`path.module` is a built-in reference to the filesystem directory
containing the current module's `.tf` files — using it instead of a bare
relative path keeps the configuration correct regardless of what directory
`terraform` is invoked from.

Running this (`terraform init` then `terraform apply`) genuinely creates a
`greeting.txt` file with that content, tracked in state exactly like a cloud
resource would be — `terraform destroy` deletes the file again. It's a real,
safe way to see the full workflow before touching billable infrastructure.

## Reading it back with a data source

```hcl
data "local_file" "greeting_check" {
  filename = local_file.greeting.filename
}

output "file_contents" {
  value = data.local_file.greeting_check.content
}
```

This deliberately reuses module 05's resource/data-source distinction: the
`resource` block owns and can destroy the file; the `data` block only reads
whatever is there, tracking it via an implicit dependency (Terraform knows
to create the file before trying to read it, because the `data` block
references `local_file.greeting.filename`).

## The cloud equivalent: an S3 bucket, reasoned through

Structurally, an S3 bucket resource follows the exact same shape as the
`local_file` example — a `resource` block, arguments, and attributes you can
reference elsewhere:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "reports" {
  bucket = "acme-reports-2026-${random_id.suffix.hex}"

  tags = {
    Environment = "learning"
    ManagedBy   = "terraform"
  }
}

resource "random_id" "suffix" {
  byte_length = 4
}
```

Two things worth calling out, reasoned from documented AWS/Terraform
behavior rather than a live run:

- **S3 bucket names are globally unique across all of AWS**, not just your
  account — a hardcoded name like `acme-reports-2026` will very likely
  already be taken by someone else. Appending a `random_id` resource's
  `.hex` attribute is a common, documented pattern for guaranteeing
  uniqueness without hand-picking a name.
- The `aws_s3_bucket` resource on modern provider versions (v4+) intentionally
  keeps a minimal set of arguments (`bucket`, `bucket_prefix`, `tags`,
  `force_destroy`) — versioning, encryption, lifecycle rules, and public
  access blocking are each configured through **separate resources** (as
  seen in module 05's `aws_s3_bucket_versioning` example) rather than nested
  arguments on the bucket itself. Always check the provider's current
  registry page, since AWS provider major versions have changed this shape
  before.

## Blocking public access (a documented, commonly-required companion resource)

```hcl
resource "aws_s3_bucket_public_access_block" "reports" {
  bucket = aws_s3_bucket.reports.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

This is worth including even in a learning exercise: several cloud
providers' documented security baselines call for explicitly blocking
public access on storage resources rather than relying on defaults, and
seeing the pattern here (a second resource block, referencing the first via
`aws_s3_bucket.reports.id`) reinforces module 05's dependency-inference
point.

## Reading the plan output shape

If this were applied, `terraform plan` output would show, per documented
Terraform conventions:

```text
Terraform will perform the following actions:

  # random_id.suffix will be created
  + resource "random_id" "suffix" {
      + byte_length = 4
      + hex         = (known after apply)
      + id          = (known after apply)
    }

  # aws_s3_bucket.reports will be created
  + resource "aws_s3_bucket" "reports" {
      + bucket = (known after apply)
      + arn    = (known after apply)
      + id     = (known after apply)
      ...
    }

  # aws_s3_bucket_public_access_block.reports will be created
  + resource "aws_s3_bucket_public_access_block" "reports" {
      + bucket                  = (known after apply)
      + block_public_acls       = true
      ...
    }

Plan: 3 to add, 0 to change, 0 to destroy.
```

Note `bucket = (known after apply)` on `aws_s3_bucket.reports` — even though
part of its value (`acme-reports-2026-`) is a literal string you wrote, the
*whole* expression depends on `random_id.suffix.hex`, which isn't known
until `random_id.suffix` is actually created, so Terraform can't resolve the
full string at plan time.

## Exercise

Write out the full `local_file` example from this module (resource + data
source + output) in a scratch directory, run `terraform init` and
`terraform apply` for real (it's local-only, no account needed), inspect
`greeting.txt` on disk, then run `terraform destroy` and confirm the file is
gone. Separately, without applying it, write the S3 bucket + public-access-block
pair above from memory and explain in one sentence why `bucket` shows as
`(known after apply)` in the plan.
