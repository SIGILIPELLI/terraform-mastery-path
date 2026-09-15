---
description: "Moving & Importing Resources — Two operations edit state without touching real infrastructure at all: moving an address (renaming, restructuring into a…"
---

# 02 · Moving & Importing Resources

Two operations edit *state* without touching real infrastructure at all:
**moving** an address (renaming, restructuring into a module) and
**importing** a resource that already exists in the cloud but was never
created by Terraform. Both are precision tools for keeping state and
configuration honest without destroy/recreate cycles.

## `terraform state mv`: renaming without recreating

```bash
terraform state mv aws_instance.web aws_instance.app_server
```

If you rename a resource in your `.tf` file from `aws_instance.web` to
`aws_instance.app_server` without touching state, Terraform's diff sees
"a new resource `app_server` to create" and "the old resource `web` to
destroy" — because state only knows resources by their address, and a
renamed address looks like a completely different resource. `state mv`
updates the address stored in state directly, so the *next* plan sees the
already-existing resource simply matched to its new name, with **no**
create or destroy at all.

```bash
# moving a resource into a module during a refactor
terraform state mv aws_instance.web module.app.aws_instance.this
```

## The `moved` block: the declarative, version-controlled alternative

```hcl
moved {
  from = aws_instance.web
  to   = aws_instance.app_server
}
```

Where `state mv` is an imperative command run once by whoever does the
refactor, a `moved` block is committed alongside the renamed resource
itself — anyone who pulls the updated configuration and runs `terraform
plan` gets the same "no create/destroy, just an address update"
outcome automatically, without needing to know a manual `state mv` command
was ever necessary. This is strongly preferred for any refactor landing in
shared configuration, precisely because it makes the migration part of the
code review instead of a step someone has to remember to run out-of-band.

## `terraform import`: bringing existing infrastructure under management

```bash
terraform import aws_s3_bucket.reports acme-reports-2026
```

This tells Terraform "the resource address `aws_s3_bucket.reports` in your
configuration corresponds to the real bucket named `acme-reports-2026`" —
it reads that bucket's current attributes from the provider and writes
them into state, **but it does not generate the `.tf` configuration for
you.** You must have already written (or write immediately after) a
matching `resource "aws_s3_bucket" "reports" { ... }` block yourself, or
the very next `plan` will propose changing every attribute the real bucket
has that your (empty or mismatched) configuration doesn't specify.

## `import` blocks (declarative import, paired with generated config)

```hcl
import {
  to = aws_s3_bucket.reports
  id = "acme-reports-2026"
}
```

```bash
terraform plan -generate-config-out=generated.tf
```

The `import` block is the modern, declarative counterpart to
`terraform import`: committed to configuration, reviewable, and re-runnable.
Paired with `-generate-config-out`, `terraform plan` can even write a
best-effort `.tf` resource block for you from the real resource's current
attributes — a substantial time-saver for importing something with many
attributes, though the generated HCL is a starting point to review and
clean up, not guaranteed to be idiomatic or minimal.

## Worked example: bringing an unmanaged bucket under Terraform

```hcl
# 1. Write the import block
import {
  to = aws_s3_bucket.reports
  id = "acme-reports-2026"
}
```

```bash
# 2. Generate a starting configuration
terraform plan -generate-config-out=generated_reports.tf
```

```hcl
# 3. generated_reports.tf now contains something like:
resource "aws_s3_bucket" "reports" {
  bucket = "acme-reports-2026"
  # ... every other attribute the provider reported
}
```

```bash
# 4. Review, tidy, then apply — this "creates" nothing, since the
#    resource already exists; it only reconciles state with config.
terraform apply
```

## How It Actually Works

**State addresses are the only identity Terraform's diffing engine
knows — `state mv` and `moved` blocks work by rewriting that identity
directly, bypassing the create/destroy inference entirely.** When
Terraform computes a plan, it doesn't compare *resources* semantically
(noticing "this looks like the same S3 bucket, just renamed") — it
compares *addresses*. An address present in state but absent from
configuration is a destroy candidate; an address present in configuration
but absent from state is a create candidate. `state mv` (and `moved`,
which Terraform Core resolves during graph construction *before* diffing
even begins) short-circuits this entirely by making the "before" address
in state match the "after" address in configuration ahead of the
comparison — from the diff engine's point of view, nothing about that
resource's identity ever changed.

**`import` populates state via the exact same `ReadResource` provider RPC
used for the refresh step in every normal `plan`** — the only difference
is that Terraform doesn't yet have a prior state entry to hand the
provider as a hint; it's told the resource's real-world ID directly (by
you, or by an `import` block) and asks the provider to read that
specific ID's full current attribute set cold. This is precisely why
import can't generate your `.tf` configuration on its own without
`-generate-config-out`: the RPC only returns *state* (the provider's view
of current attributes), and someone (a human, or the config-generation
feature parsing the returned attributes into HCL syntax) still has to
produce the *configuration* half of the create/read pairing that every
other resource in your `.tf` files already has.

**`-generate-config-out` runs the normal plan/diff machinery in reverse
for exactly the imported resources** — instead of taking your
configuration and computing what changes are needed to reach it, it takes
the *state* just populated by import and serializes it back out as HCL,
using the same schema each provider already exposes for validating and
documenting its own resource types. This is why the generated
configuration can look unidiomatic (explicit values for attributes you'd
normally omit to accept a provider default) — it's a mechanical
schema-to-HCL dump of exactly what's currently true, not a human's
judgment about what's worth specifying explicitly.

## Exercise

You have an existing (unmanaged) `aws_dynamodb_table` named
`acme-sessions` that you want under Terraform without recreating it.
Write the `import` block you'd add to bring it in as
`aws_dynamodb_table.sessions`, and the exact `terraform plan` command that
would both perform the import and write a starting `.tf` configuration
file named `generated_sessions.tf`.
