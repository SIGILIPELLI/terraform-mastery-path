# 03 · HCL Syntax Basics

HCL (HashiCorp Configuration Language) is the language Terraform
configurations are written in. It's built from a small number of building
blocks: blocks, arguments, and expressions.

## Blocks

A **block** is a container with a type, zero or more labels, and a body in
`{ }`:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c94855ba95c71c99"
  instance_type = "t3.micro"
}
```

- `resource` — the block type.
- `"aws_instance"` and `"web"` — labels (the resource type, then the local
  name Terraform uses to refer to this instance elsewhere in the config).
- Everything inside `{ }` — the block body, made of arguments and nested
  blocks.

Different block types take different numbers of labels:

```hcl
resource "aws_instance" "web" { }   # 2 labels: type, local name
provider "aws" { }                  # 1 label: provider name
variable "region" { }                # 1 label: variable name
terraform { }                        # 0 labels
```

## Arguments

An **argument** assigns a value to a name inside a block body:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c94855ba95c71c99"  # argument: ami
  instance_type = "t3.micro"               # argument: instance_type
}
```

The left-hand side is always an identifier; the right-hand side is an
**expression** — which can be a literal, a reference, a function call, or a
combination.

## Nested blocks

Some resources need repeated or structured sub-configuration expressed as a
**nested block** rather than a single argument:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c94855ba95c71c99"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }

  ebs_block_device {
    device_name = "/dev/sdh"
    volume_size = 20
  }
}
```

Here `tags` is an argument whose value happens to be a map; `ebs_block_device`
is a nested block (no `=`, its own `{ }`). Whether a given piece of a
resource's schema is an argument or a nested block is defined by that
resource's provider — always check the provider's documentation for the
exact shape.

## Comments

```hcl
# single-line comment (preferred style)
// also valid single-line comment
/* multi-line
   comment */
```

## Identifiers and naming rules

Block labels and argument names may contain letters, digits, underscores,
and hyphens, but must start with a letter or underscore:

```hcl
resource "aws_s3_bucket" "user_uploads" { }   # valid
resource "aws_s3_bucket" "user-uploads" { }   # valid (hyphen allowed)
# resource "aws_s3_bucket" "2024_uploads" { }  # invalid — starts with a digit
```

## Literal value types

HCL has a small set of primitive and collection types you'll use constantly:

```hcl
locals {
  is_enabled   = true                        # bool
  server_count = 3                           # number
  region_name  = "us-east-1"                 # string
  az_list      = ["us-east-1a", "us-east-1b"] # list(string)
  tags_map     = { Team = "platform", Env = "prod" } # map(string)
  nothing      = null                        # the null value
}
```

## String interpolation

Strings can embed expressions with `${ ... }`:

```hcl
locals {
  env    = "prod"
  region = "us-east-1"
  bucket_name = "acme-${local.env}-${local.region}"
  # => "acme-prod-us-east-1"
}
```

For a string that is *entirely* one expression (like `"${local.env}"` alone),
Terraform recommends writing the reference directly without interpolation
syntax — `local.env` instead of `"${local.env}"` — since as of Terraform 0.12+
the reference itself carries the correct type instead of always producing a
string.

## Multi-line strings (heredoc)

```hcl
locals {
  startup_script = <<-EOT
    #!/bin/bash
    echo "Booting..."
    systemctl start nginx
  EOT
}
```

The `<<-` variant (versus plain `<<`) allows the closing marker and content
to be indented to match surrounding code, with Terraform stripping the
common leading whitespace.

## Whitespace and formatting

HCL doesn't require significant indentation the way Python does, but
`terraform fmt` (module 10) enforces a canonical two-space style everywhere,
so most style questions are answered by just running it rather than
deciding by hand.

## Exercise

Write an HCL snippet (it doesn't need to reference a real provider) with:
a `locals` block defining an `environment` string and a `replica_count`
number, and a `resource` block with two labels where one argument
interpolates `local.environment` into a string and another argument
references `local.replica_count` directly (no interpolation braces). Run it
through `terraform fmt` mentally — is your indentation already two spaces
per nesting level?
