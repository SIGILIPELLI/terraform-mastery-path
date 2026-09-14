# 01 · Modules — Structure & Inputs/Outputs

Every `.tf` file you've written so far lives in the **root module** — the
directory you run `terraform` commands from. A **module** is just any
directory containing `.tf` files; the root module is a module too, and it
can *call* other modules to reuse configuration instead of copy-pasting it.

## Why modules exist

By Level 1's capstone you likely had one flat directory with every resource
in it. That's fine for a single environment, but the moment you need the
same set of resources twice (a dev VPC and a prod VPC, or ten near-identical
S3 buckets for ten teams), copy-pasting `.tf` files means every future fix
has to be repeated in every copy. A module packages a piece of
infrastructure as a reusable, parameterized unit — call it twice with
different inputs and get two independent copies of that infrastructure.

## Anatomy of a module

A conventional module directory:

```text
modules/
  s3-bucket/
    main.tf       # resources
    variables.tf  # input variables (the module's parameters)
    outputs.tf    # output values (what the module exposes to its caller)
    README.md     # optional, but expected in any team codebase
```

None of these filenames are special to Terraform — it loads and merges
*every* `.tf` file in a directory regardless of name. The split into
`main.tf` / `variables.tf` / `outputs.tf` is a strong community convention,
not a language requirement, and following it makes any module immediately
navigable to someone who has never seen it before.

```hcl
# modules/s3-bucket/variables.tf
variable "bucket_name" {
  description = "Globally-unique S3 bucket name."
  type        = string
}

variable "enable_versioning" {
  description = "Whether to enable object versioning on the bucket."
  type        = bool
  default     = false
}
```

```hcl
# modules/s3-bucket/main.tf
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_versioning" "this" {
  count  = var.enable_versioning ? 1 : 0
  bucket = aws_s3_bucket.this.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

```hcl
# modules/s3-bucket/outputs.tf
output "bucket_arn" {
  description = "ARN of the created bucket."
  value       = aws_s3_bucket.this.arn
}

output "bucket_id" {
  value = aws_s3_bucket.this.id
}
```

Inside a module, `var.bucket_name` and `var.enable_versioning` are its
**inputs** — exactly like Level 1's root-level variables, but here they're
the parameters a *caller* sets rather than values a human types at a
`plan`/`apply` prompt. `output` blocks are the module's **return values** —
the only things visible outside it.

## Calling a module from the root

```hcl
# root main.tf
module "reports_bucket" {
  source             = "./modules/s3-bucket"
  bucket_name        = "acme-reports-2026"
  enable_versioning  = true
}

module "logs_bucket" {
  source      = "./modules/s3-bucket"
  bucket_name = "acme-logs-2026"
  # enable_versioning omitted -> uses the module's default (false)
}

output "reports_arn" {
  value = module.reports_bucket.bucket_arn
}
```

Two `module` blocks calling the same `source` directory produce two
**completely independent** resources — `module.reports_bucket` and
`module.logs_bucket` are separate addresses in state, each with its own
`aws_s3_bucket.this` underneath. The module's internal resource name
(`this`) never collides between callers because the full state address is
prefixed by the module call's own label
(`module.reports_bucket.aws_s3_bucket.this` vs.
`module.logs_bucket.aws_s3_bucket.this`).

`source = "./modules/s3-bucket"` is a **local path** — the simplest kind of
source. Module 02 covers versioned sources (registry modules, Git tags) for
modules you don't own or want to pin to a specific release.

## Worked example: two environments, one module

```hcl
module "dev_bucket" {
  source      = "./modules/s3-bucket"
  bucket_name = "acme-dev-artifacts"
}

module "prod_bucket" {
  source            = "./modules/s3-bucket"
  bucket_name       = "acme-prod-artifacts"
  enable_versioning = true
}
```

Eighteen lines define two environments' storage, each independently
configurable, with the actual bucket logic (and any future fix to it)
living in exactly one place: `modules/s3-bucket/main.tf`.

## How It Actually Works

**A module call is a graph subgraph, not a function call.** When Terraform
Core parses configuration, each `module` block causes it to load the
target directory's `.tf` files and instantiate every resource inside them
as nodes in the *same* overall dependency graph the root module builds —
just namespaced under `module.<name>.`. There is no separate execution
pass per module and no runtime call stack; "calling" a module is a
compile-time expansion that produces more graph nodes, all planned and
applied by the same single graph walk described in module 09 of Level 1.
This is why a module's resources can depend on — and be depended on by —
resources anywhere else in the configuration: `aws_s3_bucket_versioning`
inside the module depends on `aws_s3_bucket.this` in the *same* module
instance via the implicit reference, and the root's
`module.reports_bucket.bucket_arn` output reference creates a graph edge
from whatever root-level resource uses it straight into that specific
module instance's `aws_s3_bucket.this`, without either side needing to
know the other is "in a module."

**Inputs and outputs are the only permitted edges across a module
boundary.** A resource inside `modules/s3-bucket` cannot reference
`var.some_root_variable` unless that value is explicitly passed in as one
of *this* module's own `variable` blocks — modules do not inherit the
caller's variable namespace. This isn't a convenience restriction; it's
what makes a module portable at all. Terraform enforces it by giving each
module instance its own separate variable/local evaluation scope during
graph construction, so `var.bucket_name` inside the module and any
same-named `var.bucket_name` in the root are simply unrelated symbols that
happen to share a name.

**State addresses encode the call, not the source.** `terraform state list`
after applying the example above prints
`module.dev_bucket.aws_s3_bucket.this` and
`module.prod_bucket.aws_s3_bucket.this` — the address is built from the
*module block's label* (`dev_bucket`), never from the `source` path or the
module's own internal resource name alone. That's precisely how two calls
to the identical `./modules/s3-bucket` source end up as two distinct,
independently-tracked resources instead of colliding: uniqueness comes
from the call site, not the module definition.

## Exercise

Take the `local_file` resource from Level 1 module 08's exercise and
extract it into a module at `./modules/local-file` with two variables
(`filename` and `content`) and one output (the file's full path via
`local_file.this.filename`). Then call that module twice from a root
`main.tf` with two different filenames, and write out what you'd expect
`terraform state list` to print for both instances — including the full
address prefix.
