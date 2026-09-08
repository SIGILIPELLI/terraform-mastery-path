# 07 · State Basics

Terraform's **state** is a JSON file recording what it believes exists and
how it maps to your configuration. It's arguably the single most important
concept to understand correctly before running Terraform against anything
that matters.

## What problem state solves

Terraform is declarative — you describe desired state, not steps. To know
*what to actually do* on each run, it needs to compare three things:

1. **Configuration** — what your `.tf` files say should exist.
2. **State** — what Terraform believes exists (from the last run).
3. **Real infrastructure** — what actually exists in the provider.

Without a record of #2, Terraform would have no way to know that
`aws_s3_bucket.reports` in your config corresponds to a specific bucket it
already created, versus a brand-new one it should create. State is that
record.

## Where state lives by default

Running `terraform apply` for the first time in a directory creates a
`terraform.tfstate` file right there:

```bash
terraform apply
ls
# main.tf  terraform.tfstate
```

This is called **local state** — fine for solo learning and experiments,
but risky for anything a team touches (module 03 in Level 2 covers **remote
state**, which solves the problems below by storing this file somewhere
shared and lockable, like an S3 bucket or Terraform Cloud).

## What's actually inside it

```json
{
  "version": 4,
  "terraform_version": "1.9.5",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_s3_bucket",
      "name": "reports",
      "instances": [
        {
          "attributes": {
            "id": "acme-reports-2026",
            "arn": "arn:aws:s3:::acme-reports-2026",
            "bucket": "acme-reports-2026",
            "region": "us-east-1"
          }
        }
      ]
    }
  ]
}
```

Every resource block gets an entry mapping its Terraform address
(`aws_s3_bucket.reports`) to every attribute the provider returned when it
was created — both the ones you set (`bucket`) and the ones the provider
computed (`arn`). This is exactly what lets `bucket_arn` from module 06's
output work without a separate lookup: Terraform already has the value
cached in state.

## Why state contains sensitive data — and why that matters

Because state records *every* attribute a resource has, including ones a
provider might return as secrets (a generated database password, a
connection string, a private key), **state files can contain plaintext
secrets even if you never wrote them into your `.tf` files.** This is the
single biggest reason state should never be committed to a public Git
repository, and why Level 2's remote state module treats access control to
the state backend as seriously as access control to the infrastructure
itself.

## `terraform plan`'s reconciliation, conceptually

On every `plan`, Terraform (conceptually):

1. Refreshes: reads the real infrastructure and updates its in-memory
   view of state to catch drift (something changed outside Terraform).
2. Compares refreshed state against configuration.
3. Computes the minimal set of create/update/destroy operations that would
   make reality match configuration.
4. Prints that as the plan, without changing anything yet.

```bash
terraform plan
# aws_s3_bucket.reports: Refreshing state... [id=acme-reports-2026]
#
# No changes. Your infrastructure matches the configuration.
```

If someone manually deleted that bucket in the AWS console, the next
`terraform plan` would refresh, notice the real bucket is gone, and propose
re-creating it — because the *configuration* still says it should exist.

## Inspecting state without editing the file by hand

```bash
terraform show
# shows the full current state in human-readable form

terraform state list
# aws_s3_bucket.reports
# aws_s3_bucket_versioning.reports

terraform state show aws_s3_bucket.reports
# shows just that one resource's attributes, in detail
```

Manually hand-editing `terraform.tfstate` in a text editor is strongly
discouraged — a malformed edit can desynchronize Terraform's understanding
of reality in ways that are hard to diagnose. `terraform state` subcommands
(`mv`, `rm`, `import` — covered in Level 3) exist precisely so state edits go
through Terraform's own validated code paths instead.

## Never delete a state file to "start clean"

A common early mistake: deleting `terraform.tfstate` when something looks
wrong. This doesn't touch real infrastructure at all — it just makes
Terraform forget everything it created. The next `terraform apply` sees an
empty state, believes nothing exists yet, and tries to create everything
again — which for resources with unique names (like an S3 bucket) fails
with a "already exists" error from the provider, and for resources without
naming conflicts, silently creates *duplicates* alongside the orphaned
originals.

## How It Actually Works: the refresh-diff-plan algorithm

This section goes one level deeper than "state tracks reality" into the
actual comparison algorithm `terraform plan` runs — described from
Terraform's documented internals, without a live provider in these lessons:

1. **Refresh (read real-world values).** For every resource address already
   in state, Terraform Core calls the provider's `ReadResource` RPC,
   passing the resource's last-known state as a hint. The provider queries
   the real API and returns the resource's *current* actual attributes. This
   produces what Terraform calls the "prior state" for planning — reality as
   of right now, not what was last written to the state file.
2. **Diff against desired configuration.** Terraform Core then walks the
   graph and, for each resource, calls `PlanResourceChange`, handing the
   provider three things: the prior state (from step 1), the proposed new
   state computed from your HCL plus any variable/interpolation values, and
   the raw config. The provider returns a "planned new state" plus which
   attributes, if any, force a destroy-and-recreate versus an in-place
   update — a distinction the provider's schema defines per-attribute
   (`ForceNew` in provider-SDK terms).
3. **Three-way comparison, not two.** Critically this is a *three-way*
   comparison — prior state, actual refreshed reality, and desired config —
   not just "old file vs. new file." This is exactly how Terraform detects
   **drift**: if refreshed reality differs from what state last recorded
   (someone manually changed a setting in the console), that shows up as
   part of the plan even when your `.tf` files haven't changed at all.
4. **Serialize into a plan file.** The full set of proposed actions (create/
   update/destroy/no-op per resource) is what `-out=tfplan` saves — a binary
   representation of exactly the RPC calls `apply` will replay, which is why
   an `apply` from a saved plan file skips refresh and re-planning entirely
   and is deterministic even if reality changed again in between.

The state file itself is therefore not the source of truth for "what's
real" — it's Terraform's *cache* of the last-known real values, used as an
optimization so `plan` doesn't have to guess every resource's identity from
scratch, and as the map from your resource addresses to the opaque IDs
providers use.

## Exercise

Without running anything against a real cloud, sketch what you'd expect
`terraform state list` to print after applying module 05's exercise
(a `data` block plus one `resource` block) — remember data sources are
**not** tracked the same way resources are in `state list`'s managed-resource
view. Then write, in your own words, why committing `terraform.tfstate` to
a public GitHub repository would be a security incident even if your `.tf`
files never contained a single secret.
