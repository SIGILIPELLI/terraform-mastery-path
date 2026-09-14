# 01 · Large-Scale Module Design

Level 2 covered how modules work mechanically. At organizational scale —
dozens of teams, hundreds of module calls — module *design* decisions
compound: a module with the wrong interface gets copy-pasted-and-modified
by the tenth consumer rather than reused, defeating the entire point.

## Thin vs. thick modules

```hcl
# THIN — wraps one resource, adds little
module "bucket" {
  source = "./modules/s3-bucket-thin"
  name   = "acme-reports"
}
# barely saves anything over just writing the resource directly
```

```hcl
# THICK — an opinionated, composed unit
module "data_pipeline_storage" {
  source              = "./modules/data-pipeline-storage"
  name                = "acme-reports"
  retention_days      = 90
  enable_replication  = true
}
# internally: bucket + versioning + lifecycle rules + replication config +
# a KMS key + a bucket policy — an entire team's storage pattern in one call
```

A module wrapping a single resource with no added logic, defaults, or
validation is rarely worth the indirection — callers might as well use the
resource directly. The modules that get genuinely reused at scale
encapsulate a *decision* (how this org does encrypted, replicated,
lifecycle-managed storage), not just a resource type.

## Composition root vs. reusable module: separate concerns

```text
modules/
  s3-bucket/          # reusable, generic, no environment knowledge
  vpc/                # reusable, generic
platforms/
  data-team/          # a "composition root" - opinionated, wires several
                       # reusable modules together for one team's specific stack
    main.tf
```

The reusable `modules/` should have **zero** knowledge of any specific
team, environment name, or account — every environment-specific decision
belongs in the composition root that calls them (`platforms/data-team`),
never baked into the reusable module itself. Violating this is the most
common reason a module that started reusable stops being reusable: a
hardcoded team name or account ID creeping into `modules/s3-bucket` makes
every other caller either work around it or fork the module.

## Interface stability: variable/output contracts as an API

```hcl
variable "encryption" {
  description = "Encryption configuration."
  type = object({
    enabled    = bool
    kms_key_id = optional(string)
  })
  default = { enabled = true }
}
```

Grouping related arguments into one `object` variable (rather than
`encryption_enabled`, `encryption_kms_key_id` as two flat variables) gives
the module room to add more encryption-related fields later without
adding new top-level variables each time — an API-design habit borrowed
directly from software engineering, applied to a module's public
interface. `optional(string)` marks `kms_key_id` as not required even
though it's nested inside a required object.

## Validation blocks: catching bad inputs at `plan`, not at the provider

```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "retention_days" {
  type = number
  validation {
    condition     = var.retention_days >= 1 && var.retention_days <= 3650
    error_message = "retention_days must be between 1 and 3650."
  }
}
```

A `validation` block runs during `terraform plan`, before any provider is
even contacted — a typo'd environment name is rejected immediately with a
clear message, instead of surfacing later as a confusing provider-level
error (or worse, silently creating resources tagged with a nonsense
environment value).

## Worked example: a well-designed `vpc` module interface

```hcl
variable "name"       { type = string }
variable "cidr_block" {
  type = string
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "cidr_block must be a valid CIDR block."
  }
}
variable "az_count" {
  type    = number
  default = 2
  validation {
    condition     = var.az_count >= 1 && var.az_count <= 6
    error_message = "az_count must be between 1 and 6."
  }
}

output "vpc_id"            { value = aws_vpc.this.id }
output "public_subnet_ids" { value = [for s in aws_subnet.public : s.id] }
```

`can(cidrhost(var.cidr_block, 0))` is a common validation idiom: `can()`
wraps an expression and returns `true`/`false` for whether it would have
errored, letting you validate "is this string a syntactically valid CIDR
block" using the same `cidrhost` function the module needs anyway,
without writing a separate regex.

## How It Actually Works

**`validation` blocks are evaluated per-variable, during the same
graph-construction phase that resolves every other value, and they halt
the plan before any resource node is evaluated at all if they fail.**
Terraform Core treats a variable's validation condition as a dependent
node of that variable — it can only run once the variable's own value is
known, and a failing condition raises an error that stops graph evaluation
outright, meaning no `PlanResourceChange` RPC downstream of that variable
(directly or transitively) ever fires. This is mechanically identical to
why `terraform validate` catches type errors before any provider is
contacted (Level 1 module 09's "How It Actually Works") — variable
validation is simply a user-authored extension of that same early,
provider-free check.

**`optional()` in an object type constraint is resolved during type
conversion, not at the point a resource references the field — a caller
omitting an optional field gets that field set to `null` (or the declared
default) as part of coercing their input into the variable's declared
type, before any of the module's own resources evaluate.** This is why a
module can safely reference `var.encryption.kms_key_id` internally with a
`try()` or a conditional even when a caller never mentions
`kms_key_id` at all — by the time the module's own resources are
evaluated, the input has already been normalized to the full shape the
`object(...)` type declares, with missing optional fields filled in as
`null` uniformly, regardless of how sparsely different callers wrote
their input.

**Composition roots and reusable modules are architecturally identical to
Terraform Core — both are just root or child modules in the graph — but
the "reusable module has zero environment knowledge" discipline is a
purely human convention with no mechanical enforcement.** Nothing stops a
`modules/s3-bucket` resource from referencing a hardcoded account ID; the
module system provides no sandboxing to prevent it. The real-world cost of
violating this discipline shows up later, mechanically, exactly as module
02's version-pinning module described: once one team depends on a
specific (accidentally environment-coupled) behavior, bumping the module's
version to fix it becomes a breaking change for that team, which is why
the interface-design discipline in this module matters even though
Terraform itself has no opinion on it.

## Exercise

Take the flat `encryption_enabled`/`encryption_kms_key_id` pair from the
worked example's intro and rewrite it as an `optional()`-based nested
object variable named `encryption`, then write the `validation` block you
would add to reject `encryption.kms_key_id` being set while
`encryption.enabled` is `false` (a nonsensical combination) — using `can()`
or a direct boolean condition, your choice.
