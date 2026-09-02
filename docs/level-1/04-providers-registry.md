# 04 · Providers & the Provider Registry

A **provider** is a plugin that translates HCL resource blocks into calls
against a specific API — AWS, Azure, GCP, Kubernetes, GitHub, Cloudflare,
Datadog, and hundreds more. This module covers configuring providers and
finding them on the registry.

## The `provider` block

Once `required_providers` (module 02) tells Terraform *which* provider
plugin to download, a `provider` block configures *how* to talk to it:

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
```

Provider configuration arguments (`region` here) are entirely defined by
that provider — the AWS provider also accepts `profile`, `assume_role`, and
others; the Azure provider (`azurerm`) instead expects `subscription_id` and
`features {}`; a Kubernetes provider expects a `config_path` or in-cluster
settings. Always check the specific provider's registry page for its schema.

## Where provider credentials come from

Terraform providers deliberately avoid hardcoding credentials in HCL.
Instead they read from the same sources the corresponding CLI/SDK already
uses:

```hcl
provider "aws" {
  region = "us-east-1"
  # no access_key / secret_key here — resolved from, in order:
  #   1. environment variables (AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY)
  #   2. shared credentials file (~/.aws/credentials)
  #   3. an IAM role attached to the compute environment running Terraform
}
```

```bash
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="us-east-1"
terraform plan
```

Putting literal credentials in a `provider` block is possible but strongly
discouraged — they'd end up committed to version control and also written
into the state file in plaintext (module 07 covers state's sensitivity in
more depth).

## The Terraform Registry

[registry.terraform.io](https://registry.terraform.io) is where providers
(and, as covered in Level 2, reusable modules) are published and versioned.
Each provider page documents:

- Its **resources** (things it can create/manage) and **data sources**
  (things it can look up) — the next module covers the distinction.
- The **argument reference** for each resource — required and optional
  arguments, their types, and defaults.
- **Import** instructions for bringing existing infrastructure under
  Terraform's management (Level 3 covers this).
- Guides for authentication and provider-specific configuration.

A provider's registry identity is `<namespace>/<name>`, e.g.
`hashicorp/aws`, `hashicorp/google`, `hashicorp/azurerm`,
`cloudflare/cloudflare`, `integrations/github`. The namespace tells you the
publisher — `hashicorp/*` providers are maintained by HashiCorp itself;
others are maintained by the named vendor or the community, with varying
support levels shown on their registry page (Official, Partner, Community).

## Multiple configurations of the same provider (aliases)

Sometimes a configuration needs to talk to the same provider more than
once with different settings — most commonly, two regions:

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_s3_bucket" "primary" {
  bucket = "acme-primary"
  # uses the default (unaliased) aws provider — us-east-1
}

resource "aws_s3_bucket" "replica" {
  provider = aws.west
  bucket   = "acme-replica"
}
```

Any resource that doesn't set a `provider` argument uses the default
(non-aliased) provider configuration for its provider.

## Multiple different providers together

A single configuration commonly uses several unrelated providers at once —
this is one of Terraform's core strengths over cloud-specific tools:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

provider "cloudflare" {
  api_token = var.cloudflare_api_token
}
```

This lets one `terraform apply` provision an AWS load balancer *and* point
a Cloudflare DNS record at it in the same run, with an implicit dependency
between them handled automatically if one references the other's attribute.

## Exercise

Look up the `hashicorp/random` provider's page on the Terraform Registry (or
recall module 02's example). Write the `terraform` and `provider` blocks
needed to use it, then write a second, aliased configuration of a
*different* hypothetical provider of your choosing, and explain in one
sentence when you'd reach for a provider alias versus just adding another
unaliased provider block.
