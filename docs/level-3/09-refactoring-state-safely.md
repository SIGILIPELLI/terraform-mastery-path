# 09 · Refactoring State Safely

This module combines module 02's `state mv`/`moved` blocks and module 07's
graph model into a practical playbook for the refactors real teams do
regularly: splitting a monolithic configuration, extracting a resource
into a module, and safely renaming things — all without destroying and
recreating a single piece of real infrastructure.

## Refactor 1: extracting inline resources into a module

```hcl
# before: everything in root main.tf
resource "aws_s3_bucket" "reports" {
  bucket = "acme-reports-2026"
}

resource "aws_s3_bucket_versioning" "reports" {
  bucket = aws_s3_bucket.reports.id
  versioning_configuration { status = "Enabled" }
}
```

```hcl
# after: extracted into modules/s3-bucket, called from root
module "reports_bucket" {
  source            = "./modules/s3-bucket"
  bucket_name       = "acme-reports-2026"
  enable_versioning = true
}
```

```hcl
# the moved blocks that make this a no-op refactor
moved {
  from = aws_s3_bucket.reports
  to   = module.reports_bucket.aws_s3_bucket.this
}

moved {
  from = aws_s3_bucket_versioning.reports
  to   = module.reports_bucket.aws_s3_bucket_versioning.this
}
```

Without the `moved` blocks, this refactor would show as "destroy 2, create
2" — same resources, same real infrastructure, but now unnecessarily at
risk (an S3 bucket happens to survive a rename-that-looks-like-recreate
badly, but many resource types genuinely cannot be destroyed and
recreated without real downtime or data loss). The `moved` blocks make the
plan show **zero** changes for these two resources.

## Refactor 2: splitting one state into two

```bash
# 1. remove the resources that will move, from the SOURCE state,
#    into a temporary state file
terraform state mv -state-out=extracted.tfstate \
  aws_vpc.this aws_vpc.this

terraform state mv -state-out=extracted.tfstate \
  aws_subnet.public aws_subnet.public

# 2. push that temporary state into the DESTINATION configuration's backend
cd ../networking
terraform state push extracted.tfstate
```

Splitting a monolith into `networking` and `app` configurations (Level 3
module 06's directory-per-environment pattern applies just as well to
splitting by *domain*, not just by environment) means physically moving
state entries between two separate backends — `state mv` extracts them
locally, `state push` writes them into the new home. This is a rarer,
higher-stakes operation than the in-place `moved` refactor above, and
should always be preceded by a full backup of both states
(`terraform state pull > backup.tfstate`).

## Refactor 3: renaming across a provider version upgrade

```hcl
moved {
  from = aws_alb.web
  to   = aws_lb.web
}
```

`moved` isn't limited to your own renames — it's also how provider authors
document a resource-type rename across a major version (here, a
hypothetical case where a provider deprecates `aws_alb` in favor of
`aws_lb` as an alias). Checking a provider's upgrade guide for `moved`
blocks it recommends adding is standard practice before a major provider
version bump, precisely to avoid an upgrade silently proposing to
recreate every affected resource.

## The safety checklist before any state refactor

1. **Back up state first**: `terraform state pull > backup-$(date +%s).tfstate`.
2. **Run `terraform plan` after the refactor and read it in full** — the
   goal is always "0 to add, 0 to change, 0 to destroy" for a pure
   refactor; anything else means the `moved`/`state mv` mapping is
   incomplete or wrong.
3. **Never run a refactor and an unrelated feature change in the same
   commit** — if the plan shows an unexpected change, you want to know
   immediately whether it's the refactor or the feature causing it.
4. **Prefer `moved` blocks over ad hoc `state mv`** whenever the refactor
   is landing in shared/version-controlled configuration, so every
   teammate's next `plan` applies the same address mapping automatically.

## How It Actually Works

**A `moved` block is resolved by Terraform Core during graph construction,
strictly before the plan diff described in module 02 runs — it rewrites
the graph's own addressing scheme for that node before any comparison
against prior state happens.** Concretely: Terraform reads every `moved`
block in the configuration, builds a translation table (old address → new
address), and applies it to the state it loaded *before* computing the
diff against current configuration — so by the time the diff algorithm
runs, the state entry already appears under its new address, and the
diff naturally finds a perfect match with zero create/destroy. This is
exactly why moved blocks are safe to leave in configuration indefinitely
even after everyone has run the refactor once — a `moved` block whose
`from` address no longer exists in state simply has nothing to translate
and is a no-op, not an error.

**`state mv` across backends (`-state-out`, `state push`) works because
Terraform's state file is a self-contained, backend-agnostic JSON document
— nothing in it references "which backend it came from."** Moving a
resource's state entry to a different backend is therefore purely a data
operation: extract the JSON fragment, insert it into a different state
file, done. The receiving configuration's *next* `plan` treats those
entries exactly like any other pre-existing state it loaded normally — it
has no memory of, or dependency on, where they were state-managed before,
which is what makes splitting a monolith's state into two backends a
one-time, purely mechanical operation rather than something requiring
ongoing coordination between the two configurations afterward.

**The "0 to add, 0 to change, 0 to destroy" property that validates a
refactor is a direct consequence of the diff engine's address-based
identity from module 02** — since Terraform decides create/update/destroy
purely by comparing addresses (not by any deeper notion of "did the real
resource change"), a refactor is correct by definition exactly when it
produces zero diff, because that's the mechanical signal that every
resource address in the *before* state has a corresponding, unchanged
entry in the *after* configuration. This is also why the safety checklist
insists on reading the *full* plan, not just its summary line — a
refactor with one address mistranslated can still show a small,
easy-to-miss "1 to add, 1 to destroy" alongside dozens of correctly
zero-diffed resources.

## Exercise

You have `aws_instance.web` in your root configuration and want to extract
it (along with a newly-added `aws_eip.web`) into a new
`modules/web-server` module, calling it `module.web`. Write the two
`moved` blocks needed so that `terraform plan` after this refactor shows
zero changes for both resources, given the module's internal resource
names are `aws_instance.this` and `aws_eip.this`.
