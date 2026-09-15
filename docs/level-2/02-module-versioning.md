---
description: "Module Versioning — Module 01's source = './modules/s3-bucket' works because the module lives in the same repository. Once a module is shared across teams…"
---

# 02 · Module Versioning

Module 01's `source = "./modules/s3-bucket"` works because the module lives
in the same repository. Once a module is shared across teams or repos, you
need a **source** that points elsewhere and, critically, a **version**
pin — so that a change to the module doesn't silently change every caller
the next time someone runs `terraform init`.

## Source types

```hcl
# Local path — relative to the calling module
module "bucket" {
  source = "./modules/s3-bucket"
}

# Terraform Registry — namespace/name/provider
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.8.1"
}

# Git, pinned to a tag
module "bucket" {
  source = "git::https://github.com/acme/tf-modules.git//s3-bucket?ref=v2.3.0"
}

# Git, pinned to a specific commit (maximally reproducible)
module "bucket" {
  source = "git::https://github.com/acme/tf-modules.git//s3-bucket?ref=a1b2c3d"
}
```

The `//` in the Git URL separates the repository root from a **subdirectory**
within it — `tf-modules.git//s3-bucket` means "the `s3-bucket/` folder of
that repo," letting one Git repository host many independently-versioned
modules.

## The `version` constraint (registry modules only)

`version` is only meaningful for registry-sourced modules — Git and local
sources pin via `ref` or the path itself instead:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.8"   # >= 5.8.0, < 6.0.0
}
```

| Constraint | Meaning |
|---|---|
| `"5.8.1"` | exactly this version |
| `">= 5.8.0"` | this version or newer |
| `"~> 5.8"` | `>= 5.8.0`, `< 6.0.0` (allow patch/minor within the major) |
| `"~> 5.8.1"` | `>= 5.8.1`, `< 5.9.0` (allow only patch releases) |

`~>` ("pessimistic constraint") is the pattern used in almost every real
module call: it accepts bug-fix and non-breaking releases automatically
while refusing a major-version bump (which, by semantic-versioning
convention, may contain breaking input/output changes) until someone
deliberately widens the constraint.

## Worked example: pinning a shared module across two callers

```hcl
module "dev_bucket" {
  source  = "git::https://github.com/acme/tf-modules.git//s3-bucket?ref=v2.3.0"
  bucket_name = "acme-dev-artifacts"
}

module "prod_bucket" {
  source  = "git::https://github.com/acme/tf-modules.git//s3-bucket?ref=v2.1.0"
  bucket_name = "acme-prod-artifacts"
}
```

Nothing stops dev and prod from intentionally running *different* pinned
versions of the same module while its maintainers roll out a change —
prod stays on `v2.1.0` until the team has validated `v2.3.0` in dev.
Unpinned sources (a branch name, or no `ref` at all defaulting to the
default branch) make this impossible to guarantee: the module's behavior
could change under you on the next `terraform init` with zero diff in your
own repo's history.

## `.terraform.lock.hcl` locks providers, not modules

It's a common misconception that the provider lock file (Level 1 module 02)
also pins module versions — it doesn't. Module version/ref selections are
recorded only in your `.tf` source itself; there is no separate module lock
file. This is exactly why an explicit `version`/`ref` in the `module` block
is the *only* thing standing between you and a moving target.

## Upgrading deliberately

```bash
terraform init -upgrade
```

Re-resolves module sources (and provider versions, within their own
constraints) to the newest versions your constraints allow, and re-writes
the downloaded copies in `.terraform/modules/`. Running plain
`terraform init` again after a module is already downloaded does **not**
re-check for a newer version within a `~>` range — `-upgrade` is required
to actually move forward, which keeps `init` fast and deterministic day to
day.

## How It Actually Works

**Module source resolution happens entirely during `terraform init`, before
any graph is built.** Terraform Core parses every `module` block's `source`
and `version` arguments, resolves each to a concrete set of files (cloning
the Git ref, or querying the Terraform Registry's module API for the
version matching the constraint), and copies the result into
`.terraform/modules/<module-address>/`. The `plan`/`apply` graph-building
step described in module 01 then reads `.tf` files from that local cache —
it has no awareness of Git, HTTP, or the registry at all. This split is why
changing a `version` constraint requires re-running `init`: the graph
builder is working from whatever was materialized on disk during the last
`init`, not from the source string in your `.tf` file directly.

**Registry version resolution is a constraint-satisfaction query, not a
fetch of "latest."** For a registry source, `terraform init` calls the
registry's module-versions endpoint, gets back the full list of published
versions, and picks the *highest* version satisfying your constraint
string using semantic-versioning comparison rules — not the newest publish
timestamp. This is why `~> 5.8` deterministically resolves to the same
version across machines as long as no new matching version has been
published in between: it's a pure function of (constraint, currently
published version list), which is also why two people running `init` on
different days can get different results if the module author ships a new
patch release in between — the constraint didn't change, but the set of
versions satisfying it did.

**Git refs are resolved by clone-and-checkout, and commit-pinning is
strictly stronger than tag-pinning.** For `git::...?ref=X`, Terraform
shells out to a real `git clone`/`fetch` against that ref. A tag ref
(`v2.3.0`) is only as immutable as the remote repository's own tag-deletion
policy allows — Git tags can technically be force-moved by someone with
push access, which would silently change what `v2.3.0` resolves to on a
later fresh `init`. A commit SHA cannot be reassigned, which is why
maximally-reproducible pipelines (Level 4's CI/CD module) sometimes pin
modules by full commit hash rather than by tag, trading a little
readability for a guarantee no tag rename could ever provide.

## Exercise

Given the module call
`module "vpc" { source = "terraform-aws-modules/vpc/aws"; version = "~> 5.8" }`,
write out which of these hypothetical published versions Terraform would
resolve to, and why, if all of `5.7.9`, `5.8.0`, `5.8.4`, `5.9.0`, and
`6.0.0` are currently published: pick the one version `terraform init`
would select.
