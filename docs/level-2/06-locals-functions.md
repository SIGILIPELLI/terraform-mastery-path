---
description: "Locals & Functions — Unlike variable, a local has no default, no type constraint, and cannot be set by a caller — it's computed once from other values…"
---

# 06 · Locals & Functions

**Local values** (`locals`) give a name to an expression you'd otherwise
repeat across a configuration. Terraform's **built-in functions** are what
you compute those expressions with — there is no way to define your own
function in HCL, so fluency with the built-in library matters more here
than in most languages.

## `locals`: named expressions, not variables

```hcl
locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_s3_bucket" "reports" {
  bucket = "${local.name_prefix}-reports"
  tags   = local.common_tags
}

resource "aws_s3_bucket" "logs" {
  bucket = "${local.name_prefix}-logs"
  tags   = local.common_tags
}
```

Unlike `variable`, a `local` has no `default`, no `type` constraint, and
cannot be set by a caller — it's computed once from other values
(variables, other locals, resource attributes) and referenced by
`local.<name>` wherever needed. The point isn't configurability; it's
avoiding writing `"${var.project}-${var.environment}"` five times and
having to fix all five if the naming scheme changes.

## String functions

```hcl
locals {
  bucket_name = lower(replace("${var.project} Reports", " ", "-"))
  # "Acme Reports" -> "acme-reports"

  truncated = substr(var.project, 0, 8)
  joined    = join(",", ["a", "b", "c"])   # "a,b,c"
  parts     = split(",", "a,b,c")          # ["a", "b", "c"]
}
```

## Collection functions

```hcl
locals {
  all_azs      = ["us-east-1a", "us-east-1b", "us-east-1c"]
  first_two    = slice(local.all_azs, 0, 2)
  az_count     = length(local.all_azs)
  has_prod_tag = contains(["prod", "staging"], var.environment)

  # merge: later maps win on key conflicts
  merged_tags = merge(
    { Project = var.project },
    { Environment = var.environment },
    var.extra_tags,
  )
}
```

`merge` is one of the most common functions in real configurations
precisely because of the worked example above: combining a module's own
baseline tags with whatever extra tags a caller supplies, without the
caller needing to know or repeat the baseline ones.

## Numeric and conditional functions

```hcl
locals {
  instance_count = max(1, min(var.desired_count, 10))
  # clamps desired_count into [1, 10]

  is_prod = var.environment == "prod"
}
```

## `for` expressions: transforming collections inline

```hcl
locals {
  az_map = { for idx, az in local.all_azs : az => idx }
  # { "us-east-1a" = 0, "us-east-1b" = 1, "us-east-1c" = 2 }

  upper_names = [for n in ["alice", "bob"] : upper(n)]
  # ["ALICE", "BOB"]

  prod_only = [for env in var.environments : env if env == "prod"]
}
```

A `for` expression is HCL's list/map comprehension — not a full
programming-language loop, but expressive enough to reshape one collection
into another (list to map, filtering, transforming each element) entirely
within a single expression, which is exactly what `count`/`for_each`
(module 08) need to consume as their input.

## Worked example: building tag maps and names from a few inputs

```hcl
variable "project"     { type = string }
variable "environment" { type = string }
variable "extra_tags"  {
  type    = map(string)
  default = {}
}

locals {
  name_prefix = "${var.project}-${var.environment}"
  common_tags = merge(
    {
      Project     = var.project
      Environment = var.environment
      ManagedBy   = "terraform"
    },
    var.extra_tags,
  )
}

resource "aws_s3_bucket" "this" {
  bucket = "${local.name_prefix}-data"
  tags   = local.common_tags
}
```

Three small variables expand, via `locals` and `merge`, into a consistent
naming and tagging scheme reusable across every resource in the file —
change the scheme once, in `locals`, and every resource referencing
`local.name_prefix`/`local.common_tags` picks it up.

## How It Actually Works

**A `local` is a graph node like any other, evaluated lazily and cached
per apply.** Terraform doesn't compute all `locals` up front in file
order — each `local.<name>` reference creates a dependency edge from
whatever uses it back to the `locals` block's expression, exactly like a
resource attribute reference. `local.merged_tags` above only gets
evaluated once its value is actually needed by a downstream node
(`aws_s3_bucket.this`'s `tags` argument), and its result is memoized for
the rest of that plan/apply — referencing it from ten different resources
evaluates the `merge()` call once, not ten times.

**Built-in functions are pure and evaluated entirely within Terraform
Core — never via a provider RPC.** `merge`, `lower`, `for` expressions,
and the rest of the function library run in Terraform's own Go runtime
during graph evaluation, with no network call and no provider involved at
all, which is why they work identically regardless of which providers a
configuration uses and why `terraform console` can evaluate them with zero
providers configured. This also means their results are always knowable
the moment their inputs are knowable — a `for` expression over a literal
list is resolved during plan, but a `for` expression over an attribute
that won't be known until after `apply` (an ID a provider assigns) stays
an "unknown value" in the plan output until that apply actually runs.

**`for` expressions compile to the same internal value-transformation
step whether they produce a list or a map** — the only difference is
syntax (`[for ... : ...]` vs `{for ... : ... => ...}`) controlling which
of Terraform's two composite-value constructors the expression evaluator
invokes; both walk the source collection exactly once, applying the body
expression (and optional `if` filter) per element, which is why a `for`
expression's cost scales linearly with collection size and never
re-scans previously-produced elements.

## Exercise

Write a `locals` block that takes a variable `var.environments` (a list of
strings like `["dev", "staging", "prod"]`) and produces a map from each
environment name to a boolean that's `true` only for `"prod"`, using a
`for` expression with `=>` — without writing an explicit `if`/`else`
anywhere.
